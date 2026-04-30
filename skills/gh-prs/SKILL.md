---
name: gh-prs
description: "Poll GitHub PRs for review requests, new iterations, and failed CI checks. Spawn sub-agents to review code or fix failing CI. Avoids concurrency conflicts with gh-issues skill via workspace isolation."
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "🔍",
        "requires": { "bins": ["curl", "git", "gh"] },
        "primaryEnv": "GH_TOKEN",
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gh",
              "bins": ["gh"],
              "label": "Install GitHub CLI (brew)",
            },
            {
              "id": "apt",
              "kind": "apt",
              "package": "gh",
              "bins": ["gh"],
              "label": "Install GitHub CLI (apt)",
            },
          ],
      },
  }
---

# gh-prs — Automated GitHub PR Review & CI Failure Handler

You are an orchestrator for PR automation. Follow these phases exactly.

IMPORTANT — This skill uses curl + GitHub REST API exclusively. GH_TOKEN is already injected by OpenClaw. Pass it as a Bearer token:

```
curl -s -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" ...
```

---

## Phase 1 — Parse Arguments

Parse arguments provided after `/gh-prs`.

**Positional:**
- `owner/repo` — optional. Source repo to monitor. If omitted, detect from git remote:
  ```
  git remote get-url origin
  ```
  Extract owner/repo from URL:
  - HTTPS: `https://github.com/owner/repo.git` → owner/repo
  - SSH: `git@github.com:owner/repo.git` → owner/repo
  
  If not in a git repo, stop with error asking user to specify owner/repo.

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| --limit | 10 | Max PRs to fetch per poll |
| --state | open | PR state: open, closed, all |
| --author | _(none)_ | Filter by PR author (`@me` for self) |
| --label | _(none)_ | Filter by label |
| --review-requests | false | Only fetch PRs requesting my review |
| --watch | false | Keep polling for new activity |
| --interval | 5 | Minutes between polls (only with --watch) |
| --dry-run | false | Fetch and display only — no sub-agents |
| --yes | false | Skip confirmation and auto-process |
| --cron | false | Cron-safe mode: spawn agents, exit without waiting |
| --model | _(none)_ | Default model for all sub-agents (overridden by specific flags below) |
| --reviewer-agent | coding | Agent ID for code review tasks |
| --reviewer-model | _(inherits --model)_ | Model for reviewer sub-agent (e.g., `kimi-k2.6`) |
| --fixer-agent | coding | Agent ID for CI failure fix tasks |
| --fixer-model | _(inherits --model)_ | Model for fixer sub-agent (e.g., `minimax-2.5`) |
| --workspace | auto | Workspace directory for git operations. `auto` = `/data/.clawdbot/gh-prs-workspace/{repo-slug}` |
| --notify-channel | _(none)_ | Channel ID to send summaries to |

**Derived values:**
- `SOURCE_REPO` = positional owner/repo
- `WORKSPACE_DIR` = --workspace value or auto-generated path
- `STATE_FILE` = `/data/.clawdbot/gh-prs-state-{REPO_SLUG}.json`
- `CLAIMS_FILE` = `/data/.clawdbot/gh-prs-claims.json` (shared with gh-issues for cross-skill coordination)
- `REVIEWER_AGENT` = --reviewer-agent value (default: coding)
- `REVIEWER_MODEL` = --reviewer-model value, or --model, or none
- `FIXER_AGENT` = --fixer-agent value (default: coding)
- `FIXER_MODEL` = --fixer-model value, or --model, or none

---

## Phase 2 — Resolve Token & Setup

**2.1 — Token Resolution:**

Check environment:
```
echo $GH_TOKEN
```

If empty, read from config:
```
CONFIG_PATH="${OPENCLAW_CONFIG_PATH:-${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/openclaw.json}"
cat "$CONFIG_PATH" | jq -r '.skills.entries["gh-prs"].apiKey // empty'
```

Export for subsequent commands:
```
export GH_TOKEN="<token>"
```

**2.2 — Workspace Setup:**

Create workspace directory (for git operations, isolated from gh-issues):
```
mkdir -p "{WORKSPACE_DIR}"
```

Ensure workspace has a clone of the repo (shallow clone if missing):
```
if [ ! -d "{WORKSPACE_DIR}/.git" ]; then
  cd "{WORKSPACE_DIR}"
  git clone --depth 100 "https://x-access-token:$GH_TOKEN@github.com/{SOURCE_REPO}.git" .
fi
```

**2.3 — State File Setup:**

Initialize state tracking file:
```
STATE_FILE="/data/.clawdbot/gh-prs-state-{REPO_SLUG}.json"
if [ ! -f "$STATE_FILE" ]; then
  mkdir -p /data/.clawdbot
  echo '{"processed_prs":{},"processed_checks":{},"last_poll":null}' > "$STATE_FILE"
fi
```

---

## Phase 3 — Fetch PRs

Build GitHub API query for PRs:

```
QUERY_PARAMS=""
if [ "{state}" != "all" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&state={state}"
fi
if [ "{author}" = "@me" ]; then
  # Resolve current user first
  ME=$(curl -s -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/user | jq -r '.login')
  QUERY_PARAMS="$QUERY_PARAMS&author=$ME"
elif [ -n "{author}" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&author={author}"
fi
if [ -n "{label}" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&labels={label}"
fi

curl -s -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/{SOURCE_REPO}/pulls?per_page={limit}${QUERY_PARAMS}"
```

**Filter for review requests (if --review-requests):**
After fetching, filter PRs where `requested_reviewers` contains current user.

**Error handling:**
- HTTP 401/403 → "GitHub authentication failed. Check your apiKey."
- Empty array → "No PRs found matching filters."

Parse response and extract for each PR:
- `number`, `title`, `body`, `head.sha`, `head.ref`, `base.ref`, `user.login`
- `created_at`, `updated_at`
- `requested_reviewers` (array)
- `draft`, `mergeable`, `mergeable_state`

---

## Phase 4 — Identify Actionable Items

For each fetched PR, determine what action is needed:

**4.1 — New PRs needing review:**

Check if PR number is in `processed_prs` state with status `"reviewed"`:
```
cat "$STATE_FILE" | jq -r ".processed_prs[\"{pr_number}\"] // empty"
```

If missing or status is not "reviewed", and PR is not draft → needs review.

**4.2 — PRs with new iterations (re-review needed):**

Compare `head.sha` with last reviewed SHA in state:
```
LAST_SHA=$(cat "$STATE_FILE" | jq -r ".processed_prs[\"{pr_number}\"].last_sha // empty")
if [ "$LAST_SHA" ] && [ "{head_sha}" != "$LAST_SHA" ]; then
  # Check if author is not self (avoid re-reviewing our own updates)
  if [ "{pr_author}" != "$CURRENT_USER" ]; then
    NEEDS_REREVIEW=true
  fi
fi
```

**4.3 — Failed CI checks:**

Fetch check runs for PR's HEAD commit:
```
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/commits/{head_sha}/check-runs"
```

Also fetch via actions API:
```
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/actions/runs?head_sha={head_sha}"
```

For each check/run:
- If `conclusion` is `failure` or `action_required`
- And check/run ID not in `processed_checks` state
- And no agent is currently working on this PR

Add to `FAILED_CHECKS` list.

**NOTE: Comment addressing is handled by gh-issues skill for its issue-fix PRs. This skill does NOT address review comments — it only reviews code and fixes CI failures.**

**Concurrency Check — CRITICAL:**

Before adding any PR to action lists, verify no gh-issues sub-agent is working on related branches:

```
# Check gh-issues claims file for any active work on this repo
CLAIMS_FILE="/data/.clawdbot/gh-issues-claims.json"
if [ -f "$CLAIMS_FILE" ]; then
  # Check for active claims on this repo
  CUTOFF=$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-2H +%Y-%m-%dT%H:%M:%SZ)
  ACTIVE_CLAIMS=$(cat "$CLAIMS_FILE" | jq --arg repo "{SOURCE_REPO}" --arg cutoff "$CUTOFF" \
    '[to_entries[] | select(.value.repo == $repo and .value.expires > $cutoff)]')
  
  if [ "$(echo "$ACTIVE_CLAIMS" | jq 'length')" -gt 0 ]; then
    echo "⚠️  gh-issues has active claims on this repo. Checking for branch overlap..."
    # Check if any gh-issues PR affects same files or is the same PR
    for key in $(echo "$ACTIVE_CLAIMS" | jq -r '.[].key'); do
      echo "Active gh-issues work: $key"
    done
  fi
fi
```

Also check gh-prs own claims:
```
PR_CLAIMS_FILE="/data/.clawdbot/gh-prs-claims.json"
if [ -f "$PR_CLAIMS_FILE" ]; then
  CUTOFF=$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-2H +%Y-%m-%dT%H:%M:%SZ)
  ACTIVE_PR_CLAIMS=$(cat "$PR_CLAIMS_FILE" | jq --arg repo "{SOURCE_REPO}" --arg cutoff "$CUTOFF" \
    '[to_entries[] | select(.value.repo == $repo and .value.expires > $cutoff)]')
fi
```

**Note on gh-issues PRs:**
PRs with `fix/issue-*` branches are created by gh-issues skill, but gh-issues only **addresses review comments** on them — it does NOT:
- Review the code (that's our job)
- Fix CI failures (that's also our job)

So we DO process these PRs for code review and CI fixes. We just don't address comments on them.

---

## Phase 5 — Present & Confirm

Display markdown summary:

```
## PR Activity Summary for {SOURCE_REPO}

### PRs Needing Review: {count}
| # | Title | Author | Status |
|---|-------|--------|--------|
| 55 | Fix memory leak | @alice | New |
| 42 | Add feature X | @bob | Updated since last review |

### Failed CI Checks: {count}
| PR | Check | Status |
|----|-------|--------|
| 55 | test/unit | ❌ failure |
| 42 | lint | ❌ failure |

### Skipped (gh-issues managed): {count}
| PR | Branch |
|----|--------|
| 60 | fix/issue-7 |

### Skipped (Concurrency): {count}
| PR | Reason |
|----|--------|
| 61 | gh-issues agent active |
```

If `--dry-run`: Display and stop.

If `--yes`: Auto-process all.

Otherwise, ask user:
- "all" — process everything
- "review-only" — only review new/updated PRs
- "checks-only" — only fix failed checks
- Comma-separated PR numbers — process only those
- "cancel" — abort

---

## Phase 6 — Pre-flight Checks

**6.1 — Workspace Isolation Check:**

Ensure workspace is not currently in use by another gh-prs process:
```
LOCK_FILE="{WORKSPACE_DIR}/.gh-prs-lock"
if [ -f "$LOCK_FILE" ]; then
  LOCK_PID=$(cat "$LOCK_FILE")
  if ps -p "$LOCK_PID" > /dev/null 2>&1; then
    echo "Workspace locked by process $LOCK_PID. Waiting or skipping..."
    # In --cron mode, skip; otherwise wait with timeout
  else
    # Stale lock, remove it
    rm -f "$LOCK_FILE"
  fi
fi
# Create lock
echo $$ > "$LOCK_FILE"
```

**6.2 — Skip gh-issues managed PRs ONLY for comment addressing:**

We already removed comment addressing from this skill. For code review and CI fixes, we process ALL PRs including `fix/issue-*` branches.

The only time we skip `fix/issue-*` PRs is if gh-issues has an ACTIVE claim on that specific PR (meaning it's currently addressing comments). Check claims file:
```
if [ -f "$PR_CLAIMS_FILE" ]; then
  # Check if gh-issues has a claim on this specific PR
  ACTIVE_ISSUES_CLAIM=$(cat "/data/.clawdbot/gh-issues-claims.json" 2>/dev/null | jq -r --arg key "{SOURCE_REPO}#{pr_number}" '.[$key] // empty')
  if [ -n "$ACTIVE_ISSUES_CLAIM" ]; then
    echo "Skipping #{pr_number} — gh-issues is actively addressing comments on this PR"
    remove_from_action_list $pr_number
  fi
fi
```

**6.3 — Git Sync:**
```
cd "{WORKSPACE_DIR}"
git checkout main || git checkout master
git pull origin $(git rev-parse --abbrev-ref HEAD)
```

---

## Phase 7 — Claim & Spawn Sub-agents

**7.1 — Claim Management:**

Before spawning, write claim to prevent concurrent operations:
```
CLAIM_DATA=$(cat <<EOF
{
  "repo": "{SOURCE_REPO}",
  "pr_number": {pr_number},
  "action": "{review|fix-checks}",
  "claimed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "expires": "$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v+2H +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
)

# Update claims file
PR_CLAIMS_FILE="/data/.clawdbot/gh-prs-claims.json"
if [ ! -f "$PR_CLAIMS_FILE" ]; then
  echo '{}' > "$PR_CLAIMS_FILE"
fi

# Add claim for this PR
jq --arg key "{SOURCE_REPO}#{pr_number}" --argjson val "$CLAIM_DATA" \
  '.[$key] = $val' "$PR_CLAIMS_FILE" > "${PR_CLAIMS_FILE}.tmp" && \
  mv "${PR_CLAIMS_FILE}.tmp" "$PR_CLAIMS_FILE"
```

**7.2 — Spawn Review Sub-agent:**

For each PR needing review:

```yaml
runtime: subagent
mode: run
task: |
  You are a code reviewer. Review PR #{pr_number} in {SOURCE_REPO}.
  
  ## Instructions
  1. CLONE: If not already cloned, shallow clone the repo to a temp location:
     git clone --depth 100 https://x-access-token:$GH_TOKEN@github.com/{SOURCE_REPO}.git /tmp/gh-prs-review-{pr_number}
  
  2. FETCH: Get PR details and diff:
     - PR info: curl -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}
     - Files: curl -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/files
     - Diff: curl -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github.v3.diff" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}
  
  3. ANALYZE: Review the code changes:
     - Check for bugs, security issues, performance problems
     - Verify test coverage for new code
     - Check code style and best practices
     - Look for missing error handling
     - Verify documentation updates if needed
  
  4. COMMENT: Post review comments using GitHub API:
     - For inline comments on specific lines:
       curl -X POST -H "Authorization: Bearer $GH_TOKEN" \
         https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/comments \
         -d '{"commit_id":"{head_sha}","path":"file.js","line":42,"body":"comment"}'
     - For general PR review:
       curl -X POST -H "Authorization: Bearer $GH_TOKEN" \
         https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/reviews \
         -d '{"commit_id":"{head_sha}","body":"summary","event":"COMMENT","comments":[...]}'
  
  5. REPORT: Return a summary:
     - Number of issues found by category (critical, warning, suggestion)
     - Files reviewed
     - Lines of code changed
     - Review submitted: yes/no

constraints:
  - Do NOT approve or request changes — just leave comments
  - Be constructive and specific in feedback
  - Time limit: 30 minutes
agentId: {REVIEWER_AGENT}
model: {REVIEWER_MODEL}
runTimeoutSeconds: 1800
cleanup: keep
```

**Note on model specification:** Include `model: {REVIEWER_MODEL}` in spawn config only if REVIEWER_MODEL is set (not empty). If empty, omit the model field to use default.

**7.3 — Spawn Failed Checks Sub-agent:**

For each PR with failed checks:

```yaml
runtime: subagent
mode: run
task: |
  You are fixing failed CI checks on PR #{pr_number} in {SOURCE_REPO}.
  
  ## Context
  This PR has failing checks. Your job is to diagnose and fix them.
  
  ## Instructions
  1. CLONE: Set up workspace:
     git clone --depth 100 https://x-access-token:$GH_TOKEN@github.com/{SOURCE_REPO}.git /tmp/gh-prs-checks-{pr_number}
     cd /tmp/gh-prs-checks-{pr_number}
  
  2. FETCH: Get PR branch and check details:
     - git fetch origin pull/{pr_number}/head:pr-branch
     - git checkout pr-branch
     - Get failed checks: curl -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/commits/{head_sha}/check-runs
     - Get check output/logs if available
  
  3. DIAGNOSE: Identify root causes:
     - Read failing test output
     - Check for lint errors
     - Look for build failures
     - Check for dependency issues
  
  4. FIX: Make necessary changes:
     - Fix code causing test failures
     - Fix lint/formatting issues
     - Update dependencies if needed
     - Add missing files
  
  5. VERIFY: Run checks locally if possible:
     - npm test, pytest, etc.
     - eslint, prettier, etc.
  
  6. PUSH: Push fixes to PR branch:
     git add .
     git commit -m "ci: fix failing checks - {description}"
     git remote set-url origin https://x-access-token:$GH_TOKEN@github.com/{SOURCE_REPO}.git
     git push origin HEAD:{pr_branch}
  
  7. REPORT: Summary of what was fixed and how

constraints:
  - Only fix what's needed for checks to pass
  - Don't change functionality unless tests require it
  - Keep changes minimal
  - Time limit: 60 minutes
  - CAN touch fix/issue-* branches for CI fixes only (not for new features)
agentId: {FIXER_AGENT}
model: {FIXER_MODEL}
runTimeoutSeconds: 3600
cleanup: keep
```

**Note on model specification:** Include `model: {FIXER_MODEL}` in spawn config only if FIXER_MODEL is set. If empty, omit to use default.

**Cron mode behavior:**
- Spawn all agents without waiting
- Exit immediately
- State will be updated on next run

---

## Phase 8 — Update State & Cleanup

**8.1 — Update State File:**

```bash
STATE_FILE="/data/.clawdbot/gh-prs-state-{REPO_SLUG}.json"

# Update processed PRs
for pr in $REVIEWED_PRS; do
  jq --arg pr "$pr" --arg sha "{head_sha}" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    '.processed_prs[$pr] = {"status": "reviewed", "last_sha": $sha, "reviewed_at": $time}' "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
done

# Update processed checks
for check_id in $FIXED_CHECKS; do
  jq --arg id "$check_id" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    '.processed_checks[$id] = {"fixed_at": $time}' "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
done

# Update last poll timestamp
jq --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" '.last_poll = $time' "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
```

**8.2 — Release Claims:**

```bash
PR_CLAIMS_FILE="/data/.clawdbot/gh-prs-claims.json"
for pr in $PROCESSED_PRS; do
  jq --arg key "{SOURCE_REPO}#$pr" 'del(.[$key])' "$PR_CLAIMS_FILE" > tmp.json && mv tmp.json "$PR_CLAIMS_FILE"
done
```

**8.3 — Release Workspace Lock:**
```
rm -f "{WORKSPACE_DIR}/.gh-prs-lock"
```

---

## Phase 9 — Results Summary

Present final summary:

```
## gh-prs Results for {SOURCE_REPO}

### Code Reviews: {count}
| PR | Status | Issues Found |
|----|--------|--------------|
| 55 | ✅ Reviewed | 2 warnings, 1 suggestion |
| 42 | ✅ Re-reviewed (new commit) | 0 issues |

### Checks Fixed: {count}
| PR | Fixed | Remaining |
|----|-------|-----------|
| 38 | test/unit | lint (still failing) |

### Skipped (gh-issues managed): {count}
| PR | Branch |
|----|--------|
| 60 | fix/issue-7 |

### Skipped (Other): {count}
| PR | Reason |
|----|--------|
| 61 | gh-issues agent active |

Total time: {duration}
```

---

## Phase 10 — Watch Mode Loop

If `--watch` is active:

1. Add all processed items to tracking sets
2. Display: "Next poll in {interval} minutes... (say 'stop' to end)"
3. Sleep for {interval} minutes
4. Go back to Phase 3 — Fetch PRs
5. Continue until user says "stop"

On stop, present cumulative summary of all activity.

---

## Cross-Skill Coordination

**Avoiding Conflicts with gh-issues:**

1. **Workspace Separation:** gh-prs uses `/data/.clawdbot/gh-prs-workspace/` while gh-issues uses the main workspace
2. **Claim Checking:** Before any operation, check `/data/.clawdbot/gh-issues-claims.json` for active claims
3. **Branch Pattern Exclusion:** **NEVER touch branches matching `fix/issue-*`** — these are reserved for gh-issues skill
4. **File Locking:** Use process-level locks to prevent concurrent git operations

**Claim Expiration:**
- All claims expire after 2 hours
- Expired claims are cleaned up on each run
- Stale git locks are detected and removed

---

## State File Schema

```json
{
  "processed_prs": {
    "55": {
      "status": "reviewed",
      "last_sha": "abc123...",
      "reviewed_at": "2026-04-30T12:00:00Z",
      "iterations_reviewed": ["abc123", "def456"]
    }
  },
  "processed_checks": {
    "check_67890": {
      "pr_number": 55,
      "fixed_at": "2026-04-30T13:00:00Z"
    }
  },
  "last_poll": "2026-04-30T14:00:00Z"
}
```

---

## Common Commands Reference

```bash
# Quick check for open PRs needing review
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/owner/repo/pulls?state=open&sort=updated"

# Get PR review requests
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/owner/repo/pulls/55/requested_reviewers"

# Get PR diff
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github.v3.diff" \
  "https://api.github.com/repos/owner/repo/pulls/55"

# Post review comment
curl -X POST -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/owner/repo/pulls/55/comments" \
  -d '{"commit_id":"abc123","path":"file.js","line":10,"body":"Nice work!"}'

# Rerun failed checks
curl -X POST -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/owner/repo/actions/runs/{run_id}/rerun-failed-jobs"
```

---

## Notes

- **Comment addressing is intentionally excluded** — gh-issues skill handles review comments on its `fix/issue-*` PRs
- gh-prs only reviews code and fixes CI failures
- Always specify `--repo owner/repo` when not in a git directory
- GH_TOKEN must have `repo` and `pull_requests:write` scope for reviews
- Rate limits: 5000 requests/hour for authenticated users
- In fork scenarios, PR branches are on the fork, but comments go to the upstream PR
