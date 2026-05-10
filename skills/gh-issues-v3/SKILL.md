---
name: gh-issues-v3
description: "Fetch GitHub issues with dependency-aware execution planning, spawn sub-agents to implement fixes respecting issue prerequisites, open PRs, then monitor and address PR review comments. Looks for ISSUES_EXECUTION_PLAN.md in the repo to determine dependencies and parallel execution order. Usage: /gh-issues-v3 [owner/repo] [--label bug] [--limit 5] [--milestone v1.0] [--assignee @me] [--fork user/repo] [--watch] [--interval 5] [--reviews-only] [--cron] [--dry-run] [--model glm-5] [--notify-channel -1002381931352]"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["curl", "git", "gh", "jq"] },
        "primaryEnv": "CODER_TOKEN",
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gh",
              "bins": ["gh"],
              "label": "Install GitHub CLI (brew)",
            },
          ],
      },
  }
---

# gh-issues-v3 — Dependency-Aware Auto-fix GitHub Issues

You are an orchestrator that respects issue dependencies. Follow these phases exactly.

**Core concept:** Before processing issues, the skill checks for an `ISSUES_EXECUTION_PLAN.md` file in the repo root (or `.github/`). This file defines which issues depend on others. The skill only processes issues whose prerequisites are complete (PR merged), and respects parallel execution groups.

**Performance design:** This skill is optimized for cron execution with tight time budgets (2-5 minutes). It uses batched GraphQL queries, parallel status checks, and fire-and-forget subagent spawning to avoid timeouts.

---

## Phase 0 — Load Execution Plan

**Before fetching issues**, check for the execution plan file:

```bash
# Check repo root first, then .github/
REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null)"
PLAN_FILE=""
if [ -n "$REPO_ROOT" ]; then
  if [ -f "$REPO_ROOT/ISSUES_EXECUTION_PLAN.md" ]; then
    PLAN_FILE="$REPO_ROOT/ISSUES_EXECUTION_PLAN.md"
  elif [ -f "$REPO_ROOT/.github/ISSUES_EXECUTION_PLAN.md" ]; then
    PLAN_FILE="$REPO_ROOT/.github/ISSUES_EXECUTION_PLAN.md"
  fi
fi
```

**If found:** Parse the execution plan and load it into memory. See [Execution Plan Format](references/EXECUTION_PLAN.md) for full syntax.

**If not found:** Log "No execution plan found, proceeding with sequential processing" and continue normally (all issues eligible, processed in order).

**Execution Plan State Directory:**
```bash
GH_ISSUES_DATA_DIR="$HOME/.openclaw/workspace/.gh-issues-data"
if [ -d "/data/.clawdbot" ]; then GH_ISSUES_DATA_DIR="/data/.clawdbot"; fi
mkdir -p "$GH_ISSUES_DATA_DIR"
```

**Per-repo execution state file:**
```bash
REPO_SLUG=$(echo "{SOURCE_REPO}" | tr '[:upper:]' '[:lower:]' | tr '/' '-')
EXECUTION_STATE_FILE="$GH_ISSUES_DATA_DIR/gh-issues-execution-$REPO_SLUG.json"
if [ ! -f "$EXECUTION_STATE_FILE" ]; then
  echo '{"issues":{},"completed_groups":[],"external_prs":{},"last_run_ms":0}' > "$EXECUTION_STATE_FILE"
fi
```

### Execution Plan Format (Quick Reference)

The execution plan is a YAML-like block inside a markdown file:

```markdown
# Issue Execution Plan

## Dependencies
```yaml
dependencies:
  3: [1]         # Issue 3 depends on issue 1 being completed
  5: [1, 2]      # Issue 5 depends on issues 1 AND 2
  7: [4]         # Issue 7 depends on issue 4
  8: [3, 4]      # Issue 8 depends on issues 3 AND 4
```

## Parallel Groups
```yaml
groups:
  foundation: [1, 2, 4]    # These can all run in parallel
  core: [3, 5, 7]         # These wait for their dependencies
  polish: [8]             # Final phase
```

## External PRs
```yaml
external_prs:
  26: "feat/app-shell-dashboard"  # PR #26 blocks group "frontend_core"
```

## Constraints
```yaml
max_parallel: 4
review_after_each: true
auto_merge: false
```

## Notes
- Use groups to organize work into logical phases
- Dependencies within a group are still respected
- Issues not in any group run in the "default" group (first)
- max_parallel defaults to 8 (subagents.maxConcurrent)
```

**Full reference:** See `references/EXECUTION_PLAN.md`

### Dependency Resolution

1. **Load plan** → Build dependency graph from YAML
2. **Load state** → Read execution state file
3. **Batch refresh statuses** → Use GraphQL to check all issue/PR statuses in one query (see Batch Status Check below)
4. **Update state** → Merge new statuses into state file
5. **Determine eligible issues** → Issues with ALL dependencies satisfied (status = `merged`)
6. **Apply group constraints** → Only start issues in earliest incomplete group
7. **Apply max_parallel** → Limit concurrent sub-agents

### Batch Status Check (Performance Critical)

Instead of N sequential API calls, use a single GraphQL query:

```bash
# Build GraphQL query for all tracked issues + external PRs
build_status_query() {
  local repo="$1"
  local owner=$(echo "$repo" | cut -d'/' -f1)
  local name=$(echo "$repo" | cut -d'/' -f2)
  
  # Read all issue numbers from state file
  local issue_nums
  issue_nums=$(jq -r '.issues | keys[]' "$EXECUTION_STATE_FILE" 2>/dev/null | sort -u | tr '\n' ' ')
  
  # Read external PR numbers
  local ext_prs
  ext_prs=$(jq -r '.external_prs | keys[]' "$EXECUTION_STATE_FILE" 2>/dev/null | sort -u | tr '\n' ' ')
  
  # Build query fragments
  local query="{"
  
  # Add issue fragments
  for num in $issue_nums; do
    query="${query} issue${num}: issue(number: ${num}) { state closedAt number title }"
  done
  
  # Add PR fragments for fix/issue-{N} branches
  for num in $issue_nums; do
    query="${query} pr${num}: pullRequests(headRefName: \"fix/issue-${num}\", states: [OPEN, MERGED], first: 1) { nodes { number state merged mergeable } }"
  done
  
  # Add external PR fragments
  for pr in $ext_prs; do
    query="${query} extpr${pr}: pullRequest(number: ${pr}) { number state merged mergeable headRefName }"
  done
  
  query="${query} }"
  
  echo "$query"
}

# Execute batch query
run_batch_status_check() {
  local repo="$1"
  local query=$(build_status_query "$repo")
  
  # Escape for JSON
  local json_query=$(echo "$query" | jq -Rs '{query: .}')
  
  local result
  result=$(curl -s -X POST \
    -H "Authorization: Bearer $CODER_TOKEN" \
    -H "Content-Type: application/json" \
    "https://api.github.com/graphql" \
    -d "$json_query" 2>/dev/null)
  
  echo "$result" | jq -r '
    def issue_status($num):
      .data["issue\($num)"] as $i |
      if $i.state == "CLOSED" then "merged"
      elif $i.closedAt != null then "merged"
      else "open" end;
    
    def pr_status($num):
      .data["pr\($num)"].nodes[0] as $p |
      if $p == null then "no_pr"
      elif $p.merged then "merged:\($p.number)"
      elif $p.state == "OPEN" then "open:\($p.number)"
      else "closed" end;
    
    # Output: issue_number status pr_status
    to_entries[] | select(.key | startswith("issue")) |
    (.key | ltrimstr("issue")) as $num |
    "\($num) \(issue_status($num)) \(pr_status($num))"
  ' 2>/dev/null || echo ""
}
```

This reduces 25+ API calls to **1 GraphQL query** (~1-2 seconds total).

### External PR Support

Some projects have pre-existing PRs that block issues but are NOT generated by the issue skill. Mark them in the plan:

```yaml
external_prs:
  26: "feat/app-shell-dashboard"
```

The skill will:
1. Check if PR #26 is merged using GraphQL
2. If merged → unblock dependent groups
3. If open → dependent groups stay blocked
4. Store status in state: `"external_prs": {"26": {"status": "open", "head_ref": "feat/app-shell-dashboard"}}`

### State Tracking

After each run, update the execution state:

```json
{
  "issues": {
    "1": { "status": "merged", "pr": 99, "completed_at": "2026-05-09T21:00:00Z", "last_checked_ms": 1746825600000 },
    "2": { "status": "in_progress", "pr": 101, "started_at": "2026-05-09T21:30:00Z", "last_checked_ms": 1746825600000 },
    "3": { "status": "pending", "blocked_by": [1] },
    "5": { "status": "blocked", "blocked_by": [1, 2] }
  },
  "completed_groups": ["foundation"],
  "external_prs": {
    "26": { "status": "open", "head_ref": "feat/app-shell-dashboard", "last_checked_ms": 1746825600000 }
  },
  "last_run_ms": 1746825600000,
  "run_count": 42
}
```

**Status values:**
- `pending` — not started, waiting for dependencies
- `ready` — dependencies satisfied, ready to process
- `in_progress` — sub-agent spawned
- `pr_opened` — PR created, waiting for review/merge
- `merged` — PR merged, issue complete
- `failed` — sub-agent failed, will retry on next run
- `skipped` — skipped (duplicate, existing PR, etc.)

**Smart caching:** Only re-check issues whose status changed or whose `last_checked_ms` is older than 1 hour. This avoids redundant API calls on every cron run.

---

## Phase 1 — Parse Arguments

Same as gh-issues Phase 1.

---

## Phase 2 — Fetch Issues

Same as gh-issues Phase 2, PLUS:

After fetching issues, **filter by execution plan** (if present):

1. For each fetched issue, check its current status via **batch GraphQL query** (not sequential calls)
2. **Smart update:** Only update status if changed or stale (>1 hour since last check)
3. **Determine eligible issues:**
   - Issue has no dependencies defined → eligible
   - Issue has dependencies → ALL must be `merged` status
   - Issue is in a group → previous groups must have all issues `merged`
4. **Sort eligible issues:**
   - First: issues in earliest incomplete group
   - Within group: by issue number (ascending)
   - Ungrouped issues: by number
5. **Limit to max_parallel** (or `subagents.maxConcurrent` if not set in plan)

If `--dry-run` is active:
- Show dependency tree with batch-queried statuses:
  ```
  Issue #1 (foundation) [MERGED #99] ✅
  Issue #2 (foundation) [OPEN #101] 🔄
  Issue #4 (foundation) [READY] ⏳
  Issue #3 (core) [BLOCKED] — waiting for: #1
  Issue #5 (core) [BLOCKED] — waiting for: #1, #2
  ```

---

## Phase 3 — Present & Confirm

Same as gh-issues Phase 3, but show dependency status:

```
| #   | Title                         | Group      | Status    | Dependencies |
| --- | ----------------------------- | ---------- | --------- | ------------ |
| 1   | Fix null pointer in parser    | foundation | Merged #99| None         |
| 2   | Add retry logic for API calls | foundation | Open #101 | None         |
| 4   | Update auth middleware        | foundation | Ready     | None         |
| 3   | Add user validation           | core       | Blocked   | #1           |
```

If `--yes` is active, only process issues marked **Ready**.

---

## Phase 4 — Pre-flight Checks

Same as gh-issues Phase 4, PLUS:

**Additional check: Verify dependency prerequisites are met**

Before processing an issue, re-verify all its dependencies are `merged`. If any dependency is not merged:
- Skip the issue: "Skipping #{N} — dependency #{dep} not yet complete"
- Update its status to `blocked` in execution state
- If all remaining issues are blocked, report: "All issues blocked by dependencies. Waiting for PRs to be merged."
- If in watch mode, schedule next check for when dependencies might complete

---

## Phase 5 — Spawn Sub-agents (Dependency-Aware, Fire-and-Forget)

**CRITICAL:** This phase must complete within 60-90 seconds to avoid cron timeout. Use **fire-and-forget** subagent spawning — don't wait for results.

### Cron Mode (`--cron` active):

1. **Load execution state** (from file, already batch-queried in Phase 0)
2. **Identify ready issues** — dependencies satisfied, not in progress
3. **Apply group constraints** — only process issues in active (earliest incomplete) group
4. **Select next issue** — lowest-numbered ready issue in active group
5. **Spawn sub-agent** via gateway API (async, non-blocking)
   - Use `sessions_spawn` with `runTimeoutSeconds: 3600` and `mode: "run"`
   - Immediately update state: `status: "in_progress"`
   - **DO NOT WAIT** for sub-agent to complete
6. **Exit** (fire and forget)

If `--watch` is active and no ready issues:
- Report: "All issues in current group complete. Waiting for PR reviews/merges before starting next group."
- Sleep for interval
- On wake: **batch re-check** all dependency statuses (1 GraphQL query)
- If dependencies became available → process ready issues

### Normal Mode (`--cron` NOT active):

Same as gh-issues Phase 5, but:
- Only spawn sub-agents for **Ready** issues
- Respect `max_parallel` limit
- Wait for all spawned agents in current group to complete before starting next group (if `review_after_each: true`)

### Fire-and-Forget Spawn

```bash
spawn_fixer_fire_and_forget() {
  local repo="$1" issue_num="$2" issue_title="$3"
  
  # Build task
  local task="Implement Issue #$issue_num for $repo: $issue_title

Rules:
- One issue per PR
- PR title format: feat(scope): short description
- PR body must reference: closes #$issue_num
- If a dependency is missing, stop and leave a comment explaining the blocker
- Run tests before pushing
- Ensure CI passes"

  # Spawn via gateway API (async)
  local json_payload
  json_payload=$(jq -n \
    --arg task "$task" \
    --arg model "${MODEL:-ollama/glm-5.1:cloud}" \
    '{
      runtime: "subagent",
      mode: "run",
      task: $task,
      model: $model,
      runTimeoutSeconds: 3600,
      cleanup: "keep",
      lightContext: true
    }')
  
  # Fire and forget
  curl -s -X POST \
    -H "Content-Type: application/json" \
    -H "X-Gateway-Token: $GATEWAY_TOKEN" \
    "$GATEWAY_URL/api/v1/sessions/spawn" \
    -d "$json_payload" > /dev/null 2>&1 &
  
  # Store PID for tracking
  local spawn_pid=$!
  
  # Update state immediately
  jq --arg num "$issue_num" \
     --arg now "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
     --arg pid "$spawn_pid" \
     '.issues[$num].status = "in_progress" |
      .issues[$num].started_at = $now |
      .issues[$num].spawn_pid = $pid' \
     "$EXECUTION_STATE_FILE" > "$EXECUTION_STATE_FILE.tmp" && \
     mv "$EXECUTION_STATE_FILE.tmp" "$EXECUTION_STATE_FILE"
  
  log "Spawned fixer for issue #$issue_num (PID: $spawn_pid)"
}
```

**Why fire-and-forget?**
- The orchestrator's job is to **decide what to do**, not **do it**
- Sub-agents run for 30-60 minutes — cron timeout is 2-5 minutes
- The sub-agent updates GitHub directly (creates PR, pushes commits)
- Next cron run will see the new PR and update state accordingly

---

## Phase 6 — PR Review Handler

Same as gh-issues Phase 6, PLUS:

After a PR is merged:
1. Update execution state: mark issue as `merged`
2. **Trigger dependency check** — batch query to see if this unblocks other issues
3. In watch mode: immediately check for newly-ready issues instead of waiting full interval

After a PR review is addressed and pushed:
- Update execution state with PR number
- If all issues in a group are `merged`, mark group as complete
- Log: "Group 'foundation' complete — {N} issues merged"

---

## Watch Mode (Modified)

After presenting results:

1. Update execution state with current statuses
2. Determine if any new issues became `ready` (dependencies merged)
3. If new ready issues → process them immediately
4. If no ready issues → report: "Waiting for PR merges. Next dependency check in {interval} minutes..."
5. Sleep for interval
6. **Batch re-check ALL statuses** (1 GraphQL query, not N sequential calls)
7. If dependencies became available → process ready issues
8. If `--reviews-only` → check for review comments on open PRs

---

## Execution Plan File Format

**File name:** `ISSUES_EXECUTION_PLAN.md` (repo root) or `.github/ISSUES_EXECUTION_PLAN.md`

**Structure:**

```markdown
# Issue Execution Plan

## Dependencies

```yaml
dependencies:
  3: [1]         # Issue 3 depends on issue 1
  5: [1, 2]      # Issue 5 depends on issues 1 AND 2
  8: [3, 4]      # Issue 8 depends on issues 3 AND 4
```

## External PRs

Pre-existing PRs that block issues but are not generated by the issue skill.

```yaml
external_prs:
  26: "feat/app-shell-dashboard"   # PR #26 blocks group "frontend_core"
```

## Parallel Groups

```yaml
groups:
  foundation: [1, 2, 4]    # Run first
  core: [3, 5, 7]           # Run after foundation
  polish: [8]               # Run last
```

## Constraints

```yaml
max_parallel: 4               # Max concurrent sub-agents
review_after_each: true      # Wait for PR review before starting next group
auto_merge: false            # Auto-merge PRs when checks pass
```

## Notes

- Use groups to organize work into logical phases
- Dependencies within a group are still respected
- Issues not in any group run in the "default" group (first)
- max_parallel defaults to 8 (subagents.maxConcurrent)
```

**Minimal example:**

```markdown
# Issue Execution Plan

## Dependencies
```yaml
dependencies:
  15: [12]
  20: [15, 18]
```

## External PRs
```yaml
external_prs:
  26: "feat/app-shell-dashboard"
```

## Groups
```yaml
groups:
  foundation: [12, 18]
  api: [15]
  ui: [20]
```

## Constraints
```yaml
max_parallel: 3
```
```

---

## State Files

### Execution State
`~/.openclaw/workspace/.gh-issues-data/gh-issues-execution-{repo-slug}.json`

Tracks per-issue status, group completion, external PR statuses, and run metadata.

### Claims File
`~/.openclaw/workspace/.gh-issues-data/gh-issues-claims.json`

Same as gh-issues claims file. Keys are `{owner/repo}#{issue_number}`.

### Cursor File
`~/.openclaw/workspace/.gh-issues-data/gh-issues-cursor-{repo-slug}.json`

Same as gh-issues cursor file. Tracks `last_processed` and `in_progress`.

---

## Error Handling

### Circular Dependencies

Detected during plan parsing. Report:
> "Execution plan contains circular dependency: #3 → #5 → #3. Breaking cycle by processing lowest-numbered issue (#3) first."

### Missing Dependencies

If a dependency references an issue not in the fetched list:
> "Warning: Issue #5 depends on #99, which was not found. Treating #99 as already complete."

### Invalid Plan Format

If the plan file cannot be parsed:
> "Warning: Could not parse ISSUES_EXECUTION_PLAN.md. Falling back to sequential processing."

### Plan File Not Found

> "No ISSUES_EXECUTION_PLAN.md found. Processing issues sequentially (oldest first)."

### Timeout Prevention

The skill is designed to complete Phase 0-5 within 90 seconds:
- Batch GraphQL queries (1 call vs N calls)
- Smart caching (skip re-checking unchanged issues)
- Fire-and-forget subagent spawning (no waiting)
- Parallel status checks for external PRs

If a run exceeds 90 seconds, the orchestrator logs a warning and exits cleanly, leaving state intact for the next run.

---

## Differences from gh-issues

| Feature | gh-issues | gh-issues-v3 |
|---------|-----------|--------------|
| Execution plan | ❌ No | ✅ Yes (ISSUES_EXECUTION_PLAN.md) |
| Dependency tracking | ❌ No | ✅ Yes (per-issue prerequisites) |
| External PR support | ❌ No | ✅ Yes (pre-existing PRs as blockers) |
| Parallel groups | ❌ No | ✅ Yes (phased execution) |
| Completion detection | ❌ No | ✅ Yes (checks PR merge status) |
| Group-based processing | ❌ No | ✅ Yes (waits for group before next) |
| Max parallel control | Hardcoded 8 | Configurable in plan |
| Batch API queries | ❌ No | ✅ Yes (GraphQL) |
| Smart caching | ❌ No | ✅ Yes (skip stale re-checks) |
| Fire-and-forget | ❌ No | ✅ Yes (avoids cron timeout) |
| Timeout prevention | ❌ No | ✅ Yes (90-second target) |

## Usage Examples

```bash
# Check for execution plan and process ready issues
/gh-issues-v3 NewmanZone/shopspark --limit 10

# Cron mode with dependency awareness
/gh-issues-v3 NewmanZone/shopspark --cron --limit 5

# Reviews only (skip issue processing)
/gh-issues-v3 NewmanZone/shopspark --reviews-only

# Watch mode with dependency checks
/gh-issues-v3 NewmanZone/shopspark --watch --interval 10

# Dry run to see dependency tree
/gh-issues-v3 NewmanZone/shopspark --dry-run
```

## Migration from gh-issues

1. Add `ISSUES_EXECUTION_PLAN.md` to your repo (optional — skill works without it)
2. Change command from `/gh-issues` to `/gh-issues-v3`
3. All other flags work identically
4. No execution plan → behaves like gh-issues (sequential)
5. With plan → respects dependencies and groups
