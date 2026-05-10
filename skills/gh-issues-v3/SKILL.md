---
name: gh-issues-v3
description: "Fetch GitHub issues with dependency-aware execution planning, spawn sub-agents to implement fixes respecting issue prerequisites, open PRs, then monitor and address PR review comments. Looks for ISSUES_EXECUTION_PLAN.md in the repo to determine dependencies and parallel execution order. Usage: /gh-issues-v3 [owner/repo] [--label bug] [--limit 5] [--milestone v1.0] [--assignee @me] [--fork user/repo] [--watch] [--interval 5] [--reviews-only] [--cron] [--dry-run] [--model glm-5] [--notify-channel -1002381931352]"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["curl", "git", "gh"] },
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
  echo '{"issues":{},"completed_groups":[]}' > "$EXECUTION_STATE_FILE"
fi
```

### Execution Plan Format (Quick Reference)

The execution plan is a YAML-like block inside a markdown file:

```markdown
# Issue Execution Plan

## Dependencies
- Issue 3 depends on: Issue 1
- Issue 5 depends on: Issues 1, 2
- Issue 7 depends on: Issue 4
- Issue 8 depends on: Issues 3, 4

## Parallel Groups
- Group "foundation": Issues 1, 2, 4 (can run in parallel)
- Group "core": Issues 3, 5, 7 (wait for their dependencies)
- Group "polish": Issue 8 (final phase)

## Constraints
- Max parallel: 4
- Review after each: true
```

**Full reference:** See `references/EXECUTION_PLAN.md`

### Dependency Resolution

1. **Load plan** → Build dependency graph
2. **Load state** → Check which issues are already completed (merged PRs)
3. **Determine eligible issues** → Issues with ALL dependencies satisfied
4. **Apply group constraints** → Only start issues in active groups
5. **Apply max_parallel** → Limit concurrent sub-agents

### State Tracking

After each run, update the execution state:

```json
{
  "issues": {
    "1": { "status": "merged", "pr": 99, "completed_at": "2026-05-09T21:00:00Z" },
    "2": { "status": "in_progress", "pr": 101, "started_at": "2026-05-09T21:30:00Z" },
    "3": { "status": "pending", "blocked_by": [1] },
    "5": { "status": "blocked", "blocked_by": [1, 2] }
  },
  "completed_groups": ["foundation"]
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

### Completion Detection

The skill checks for completion in this order:

1. **PR merged?** → Check if PR for this issue is merged
2. **PR open?** → Check if PR exists and is open
3. **Branch exists?** → Check if `fix/issue-{N}` branch exists
4. **In progress?** → Check claims file for active sub-agent

```bash
# Check if issue N is complete
check_issue_complete() {
  local repo="$1" issue_num="$2"
  
  # Check if PR is merged
  local merged_pr
  merged_pr=$(curl -s -H "Authorization: Bearer $CODER_TOKEN" \
    "https://api.github.com/repos/$repo/pulls?head=$(echo "$repo" | cut -d'/' -f1):fix/issue-$issue_num&state=all&per_page=1" | \
    jq -r '.[0] | select(.merged == true) | .number')
  
  if [ -n "$merged_pr" ]; then
    echo "merged:$merged_pr"
    return 0
  fi
  
  # Check if PR is open
  local open_pr
  open_pr=$(curl -s -H "Authorization: Bearer $CODER_TOKEN" \
    "https://api.github.com/repos/$repo/pulls?head=$(echo "$repo" | cut -d'/' -f1):fix/issue-$issue_num&state=open&per_page=1" | \
    jq -r '.[0].number // empty')
  
  if [ -n "$open_pr" ]; then
    echo "open:$open_pr"
    return 0
  fi
  
  # Check claims file
  local claims_file="$GH_ISSUES_DATA_DIR/gh-issues-claims.json"
  if [ -f "$claims_file" ]; then
    local claimed
    claimed=$(jq -r --arg key "$repo#$issue_num" 'has($key)' "$claims_file" 2>/dev/null || echo "false")
    if [ "$claimed" = "true" ]; then
      echo "in_progress"
      return 0
    fi
  fi
  
  echo "pending"
  return 0
}
```

---

## Phase 1 — Parse Arguments

Same as gh-issues Phase 1.

---

## Phase 2 — Fetch Issues

Same as gh-issues Phase 2, PLUS:

After fetching issues, **filter by execution plan** (if present):

1. For each fetched issue, check its current status via `check_issue_complete`
2. Update the execution state file with current statuses
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
- Show dependency tree:
  ```
  Issue #1 (foundation) [READY] — no dependencies
  Issue #2 (foundation) [READY] — no dependencies
  Issue #4 (foundation) [READY] — no dependencies
  Issue #3 (core) [BLOCKED] — waiting for: #1
  Issue #5 (core) [BLOCKED] — waiting for: #1, #2
  Issue #7 (core) [BLOCKED] — waiting for: #4
  Issue #8 (polish) [BLOCKED] — waiting for: #3, #4
  ```

---

## Phase 3 — Present & Confirm

Same as gh-issues Phase 3, but show dependency status:

```
| #   | Title                         | Group      | Status    | Dependencies |
| --- | ----------------------------- | ---------- | --------- | ------------ |
| 1   | Fix null pointer in parser    | foundation | Ready     | None         |
| 2   | Add retry logic for API calls | foundation | Ready     | None         |
| 4   | Update auth middleware        | foundation | Ready     | None         |
| 3   | Add user validation           | core       | Blocked   | #1           |
| 5   | Implement caching layer       | core       | Blocked   | #1, #2       |
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

## Phase 5 — Spawn Sub-agents (Dependency-Aware)

**Modified for execution plan support:**

### Cron Mode (`--cron` active):

1. **Load execution state**
2. **Refresh issue statuses** — call `check_issue_complete` for all known issues
3. **Identify ready issues** — dependencies satisfied, not in progress
4. **Apply group constraints** — only process issues in active (earliest incomplete) group
5. **Select next issue** — lowest-numbered ready issue in active group
6. **Spawn ONE sub-agent** for that issue (sequential within group, parallel across groups only if group is done)
7. **Update execution state** — mark as `in_progress`
8. **Exit** (fire and forget)

If `--watch` is active and no ready issues:
- Report: "All issues in current group complete. Waiting for PR reviews/merges before starting next group."
- Sleep for interval
- Re-check dependency status on wake

### Normal Mode (`--cron` NOT active):

Same as gh-issues Phase 5, but:
- Only spawn sub-agents for **Ready** issues
- Respect `max_parallel` limit
- Wait for all spawned agents in current group to complete before starting next group (if `review_after_each: true`)

---

## Phase 6 — PR Review Handler

Same as gh-issues Phase 6, PLUS:

After a PR is merged:
1. Update execution state: mark issue as `merged`
2. **Trigger dependency check** — if this unblocks other issues, they become `ready`
3. In watch mode: immediately check for newly-ready issues instead of waiting full interval

After a PR review is addressed and pushed:
- Update execution state with PR number
- If all issues in a group are `merged`, mark group as complete

---

## Watch Mode (Modified)

After presenting results:

1. Update execution state with current statuses
2. Determine if any new issues became `ready` (dependencies merged)
3. If new ready issues → process them immediately
4. If no ready issues → report: "Waiting for PR merges. Next dependency check in {interval} minutes..."
5. Sleep for interval
6. Re-check ALL issue statuses (not just new issues)
7. If dependencies became available → process ready issues
8. If `--reviews-only` → check for review comments on open PRs

---

## Execution Plan File Format

**File name:** `ISSUES_EXECUTION_PLAN.md` (repo root) or `.github/ISSUES_EXECUTION_PLAN.md`

**Structure:**

```markdown
# Issue Execution Plan

## Dependencies

Define which issues must be completed before others can start.

Syntax options (all equivalent):

### Option 1: List format
- Issue 3 depends on: Issue 1
- Issue 5 depends on: Issues 1, 2
- Issue 8 depends on: Issues 3, 4

### Option 2: YAML block
```yaml
dependencies:
  3: [1]
  5: [1, 2]
  8: [3, 4]
```

### Option 3: Arrow format
```
1 → 3 → 8
2 → 5
4 → 7 → 8
```

## Parallel Groups

Define execution phases. All issues in a group can run in parallel (respecting max_parallel and dependencies).

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
- Issue 15 depends on: Issue 12
- Issue 20 depends on: Issues 15, 18

## Groups
- foundation: [12, 18]
- api: [15]
- ui: [20]

## Constraints
- max_parallel: 3
```

**Complex example:**

```markdown
# Issue Execution Plan for NewmanZone/shopspark

## Dependencies
```yaml
dependencies:
  # Database layer must exist before API
  42: [37]        # #42 (schema) needs #37 (db config)
  
  # API needs auth and schema
  55: [42, 48]   # #55 (endpoints) needs schema (#42) and auth (#48)
  
  # Frontend needs API
  60: [55]        # #60 (UI) needs API (#55)
  61: [55]        # #61 (tests) needs API (#55)
  
  # Integration tests need everything
  70: [60, 61]    # #70 (e2e) needs UI and tests
```

## Groups
```yaml
groups:
  infra: [37, 48]           # Setup work
  data: [42]                # Database
  api: [55]                 # API layer
  frontend: [60, 61]        # Client work (parallel)
  testing: [70]             # Final validation
```

## Constraints
```yaml
max_parallel: 4
review_after_each: true
auto_merge: false
```

## Notes
- Group "frontend" can process #60 and #61 in parallel once #55 is done
- Group "testing" only starts after both frontend issues are merged
```

---

## State Files

### Execution State
`~/.openclaw/workspace/.gh-issues-data/gh-issues-execution-{repo-slug}.json`

Tracks per-issue status and group completion.

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

---

## Differences from gh-issues

| Feature | gh-issues | gh-issues-v3 |
|---------|-----------|--------------|
| Execution plan | ❌ No | ✅ Yes (ISSUES_EXECUTION_PLAN.md) |
| Dependency tracking | ❌ No | ✅ Yes (per-issue prerequisites) |
| Parallel groups | ❌ No | ✅ Yes (phased execution) |
| Completion detection | ❌ No | ✅ Yes (checks PR merge status) |
| Group-based processing | ❌ No | ✅ Yes (waits for group before next) |
| Max parallel control | Hardcoded 8 | Configurable in plan |

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
