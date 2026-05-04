---
name: gh-prs
description: "Poll GitHub PRs for review requests, new iterations, and failed CI checks. Spawn sub-agents to review code or fix failing CI. Avoids concurrency conflicts with gh-issues skill via workspace isolation."
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "🔍",
        "requires": { "bins": ["curl", "git", "node"] },
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

IMPORTANT — This skill uses GitHub Apps for authentication.

- `{REVIEWER_APP}` (default: tars-reviewer) — for review operations (posting comments, approvals)
- `{CODER_APP}` (default: tars-coder) — for code change operations (addressing comments, pushing fixes)

Both apps must be installed on the target repo. Tokens are fetched on-demand via:
```
node $HOME/.openclaw/workspace/scripts/get-gh-app-token.js <app-name>
```

The orchestrator phase uses `$REVIEWER_TOKEN` (read-only operations). Sub-agents use `$REVIEWER_TOKEN` (reviewer) or `$CODER_TOKEN` (fixer/coder) as documented in each phase.

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
| Flag | Default | Description |
|------|---------|-------------|
| --reviewer-app | tars-reviewer | GitHub App to use for review operations (tars-reviewer, tars-coder, or token) |
| --coder-app | tars-coder | GitHub App to use for code change operations (tars-coder, tars-reviewer, or token) |
| --workspace | auto | Workspace directory for git operations. `auto` = `/data/.clawdbot/gh-prs-workspace/{repo-slug}` |
| --notify-channel | _(none)_ | Channel ID to send summaries to |

**Derived values:**
- `SOURCE_REPO` = positional owner/repo
- `WORKSPACE_DIR` = --workspace value or auto-generated path
- `REPO_SLUG` = $(echo "{SOURCE_REPO}" | tr '[:upper:]' '[:lower:]' | tr '/' '-')
- `DATA_DIR` = if [ -d "/data/.clawdbot" ]; then echo "/data/.clawdbot"; else echo "$HOME/.openclaw/workspace/.gh-prs-data"; fi
- `STATE_FILE` = "$DATA_DIR/gh-prs-state-{REPO_SLUG}.json"
- `CLAIMS_FILE` = "$DATA_DIR/gh-prs-claims.json" (shared with gh-issues for cross-skill coordination)
- `REVIEWER_APP` = --reviewer-app value (default: tars-reviewer)
- `CODER_APP` = --coder-app value (default: tars-coder)
- `SKILLS_DIR` = $HOME/.openclaw/workspace/scripts
- `REVIEWER_AGENT` = --reviewer-agent value (default: coding)
- `REVIEWER_MODEL` = --reviewer-model value, or --model, or none
- `FIXER_AGENT` = --fixer-agent value (default: coding)
- `FIXER_MODEL` = --fixer-model value, or --model, or none

---

## Phase 2 — Resolve Token & Setup

**2.1 — Token Resolution (GitHub Apps):**

Resolve installation tokens for each app. Tokens are fetched fresh since they expire after 60 minutes.

```
# Unset any inherited PAT env vars to force App token generation
unset CODER_TOKEN GH_TOKEN GITHUB_TOKEN REVIEWER_TOKEN

SCRIPT_DIR="$HOME/.openclaw/workspace/scripts"

get_token() {
  APP_NAME="$1"
  TOKEN=$(node "$SCRIPT_DIR/get-gh-app-token.js" "$APP_NAME" 2>/dev/null)
  if [ $? -ne 0 ] || [ -z "$TOKEN" ]; then
    echo "ERROR: Failed to get GitHub App token for $APP_NAME" >&2
    return 1
  fi
  echo "$TOKEN"
}

REVIEWER_TOKEN=$(get_token "{REVIEWER_APP}")
CODER_TOKEN=$(get_token "{CODER_APP}")

export REVIEWER_TOKEN
export CODER_TOKEN
```

**Note:** `{REVIEWER_APP}` is used for read-only review operations (posting review comments, approvals). `{CODER_APP}` is used for operations that make code changes (addressing comments, pushing fixes, creating commits). Both apps must be installed on `{SOURCE_REPO}`.

**2.2 — Workspace Setup:**

Create workspace directory (for git operations, isolated from gh-issues):
```
mkdir -p "{WORKSPACE_DIR}"
```

Ensure workspace has a clone of the repo (shallow clone if missing):
```
if [ ! -d "{WORKSPACE_DIR}/.git" ]; then
  cd "{WORKSPACE_DIR}"
  git clone --depth 100 "https://x-access-token:$CODER_TOKEN@github.com/{SOURCE_REPO}.git" .
fi
```

**2.3 — State File Setup:**

Initialize state tracking file:
```
DATA_DIR="$HOME/.openclaw/workspace/.gh-prs-data"
if [ -d "/data/.clawdbot" ]; then DATA_DIR="/data/.clawdbot"; fi
REPO_SLUG=$(echo "{SOURCE_REPO}" | tr '[:upper:]' '[:lower:]' | tr '/' '-')
STATE_FILE="$DATA_DIR/gh-prs-state-$REPO_SLUG.json"
mkdir -p "$DATA_DIR"
if [ ! -f "$STATE_FILE" ]; then
  echo '{"processed_prs":{},"processed_checks":{},"last_poll":null}' > "$STATE_FILE"
fi
```

---

## Phase 3 — Quick SHA Check (Skip If Unchanged)

**CRITICAL OPTIMIZATION:** Before doing any LLM work, do a cheap API call to check if anything actually changed.

```bash
# Quick check: fetch only open PR numbers and their current HEAD SHAs
QUICK_PR_LIST=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/{SOURCE_REPO}/pulls?state=open&per_page=100" | \
  jq -r '.[] | "\(.number)|\(.head.sha)"' 2>/dev/null)

if [ -z "$QUICK_PR_LIST" ]; then
  echo "gh-prs: no open PRs found"
  exit 0
fi

# Compare against last known state
HAS_CHANGES=false
while IFS='|' read -r pr_num sha; do
  LAST_SHA=$(cat "$STATE_FILE" | jq -r ".processed_prs[\"$pr_num\"]?.last_sha // empty")
  if [ -z "$LAST_SHA" ] || [ "$sha" != "$LAST_SHA" ]; then
    HAS_CHANGES=true
    echo "PR #$pr_num: SHA changed or new (was: $LAST_SHA, now: $sha)"
  fi
done <<< "$QUICK_PR_LIST"

# Also check for new failed CI checks
FAILED_CHECK_COUNT=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/actions/runs?per_page=20&status=failure" | \
  jq '[.workflow_runs[] | select(.conclusion == "failure")] | length')

if [ "$FAILED_CHECK_COUNT" -gt 0 ]; then
  HAS_CHANGES=true
  echo "Found $FAILED_CHECK_COUNT failed action runs"
fi

if [ "$HAS_CHANGES" = "false" ]; then
  echo "gh-prs: no changes detected — exiting cheaply (no LLM calls)"
  exit 0
fi

echo "Changes detected — proceeding to full review..."
```

**Result:** This turns ~95% of cron polls into a cheap 2-API-call exit instead of spawning sub-agents.

---

## Phase 4 — Fetch PRs (Full Details)

Build GitHub API query for PRs:

```
QUERY_PARAMS=""
if [ "{state}" != "all" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&state={state}"
fi
if [ "{author}" = "@me" ]; then
  # Resolve current user first
  ME=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" https://api.github.com/user | jq -r '.login')
  QUERY_PARAMS="$QUERY_PARAMS&author=$ME"
elif [ -n "{author}" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&author={author}"
fi
if [ -n "{label}" ]; then
  QUERY_PARAMS="$QUERY_PARAMS&labels={label}"
fi

curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
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
PR_STATUS=$(cat "$STATE_FILE" | jq -r ".processed_prs[\"{pr_number}\"]?.status // empty")
if [ "$PR_STATUS" = "reviewed" ]; then
  # Already reviewed — check if new commits since last review
  LAST_SHA=$(cat "$STATE_FILE" | jq -r ".processed_prs[\"{pr_number}\"].last_sha // empty")
  if [ "$LAST_SHA" ] && [ "{head_sha}" != "$LAST_SHA" ]; then
    # New commits — needs re-review
    NEEDS_REREVIEW=true
  fi
  # else: same SHA, already reviewed, skip
elif [ "$PR_STATUS" = "reviewing" ] || [ "$PR_STATUS" = "fixing" ]; then
  # PR is currently being worked on by another agent — skip
  echo "PR #{pr_number} already in progress (status: $PR_STATUS), skipping"
elif [ "$PR_STATUS" = "" ] || [ "$PR_STATUS" = "null" ]; then
  # Never reviewed — needs review
  if [ "{pr_draft}" = "false" ]; then
    NEEDS_REVIEW=true
  fi
else
  # Any other status (e.g. from a failed previous run) — retry
  if [ "{pr_draft}" = "false" ]; then
    NEEDS_REVIEW=true
  fi
fi
```

**4.2 — PRs with new iterations (re-review needed):**

This is now handled in 4.1 above. When a PR has status `"reviewed"` but a new `head.sha` is detected, it gets marked `NEEDS_REREVIEW=true`. The re-review trigger is the SHA change detection in the `PR_STATUS = "reviewed"` block.

**4.3 — Failed CI checks:**

Fetch check runs for PR's HEAD commit:
```
curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/commits/{head_sha}/check-runs"
```

Also fetch via actions API:
```
curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/actions/runs?head_sha={head_sha}"
```

For each check/run:
- If `conclusion` is `failure` or `action_required`
- And check/run ID not in `processed_checks` state
- And no agent is currently working on this PR

Add to `FAILED_CHECKS` list.

**NOTE:** Comment addressing is handled by gh-issues skill for its issue-fix PRs. This skill does NOT address review comments — it only reviews code and fixes CI failures.

**4.4 — Check existing gh-prs comments for resolution:**

For each PR with previous gh-prs review comments:
```
# Get existing review comments from gh-prs
EXISTING_COMMENTS=$(cat "$STATE_FILE" | jq -r ".pr_comments[\"{pr_number}\"] // []")

if [ "$(echo "$EXISTING_COMMENTS" | jq 'length')" -gt 0 ]; then
  # Fetch current PR diff to check if commented lines changed
  CURRENT_DIFF=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
    -H "Accept: application/vnd.github.v3.diff" \
    "https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}")
  
  for comment in $(echo "$EXISTING_COMMENTS" | jq -c '.[]'); do
    COMMENT_ID=$(echo "$comment" | jq -r '.id')
    FILE_PATH=$(echo "$comment" | jq -r '.file')
    LINE_NUM=$(echo "$comment" | jq -r '.line')
    
    # Check if the line was modified in recent commits
    if ! echo "$CURRENT_DIFF" | grep -A5 -B5 "^@@.*$FILE_PATH" | grep "^+$LINE_NUM," >/dev/null; then
      # Line not in recent diff, likely fixed - mark for resolution
      echo "Comment $COMMENT_ID on $FILE_PATH:$LINE_NUM appears resolved"
    fi
  done
fi
```

**4.5 — Determine if approval is warranted:**

A PR can be approved when:
- All gh-prs review comments are resolved (fixed by author)
- No new critical issues found in re-review
- CI checks are passing
- PR is up-to-date with base branch

```
OPEN_COMMENTS=$(cat "$STATE_FILE" | jq -r ".pr_comments[\"{pr_number}\"] // [] | map(select(.status == \"open\")) | length")
if [ "$OPEN_COMMENTS" -eq 0 ] && [ "$CHECKS_PASSING" = "true" ] && [ "$BRANCH_UPTODATE" = "true" ]; then
  CAN_APPROVE=true
fi
```

**4.6 — Check if PR is out-of-date with base branch:**

GitHub shows "out-of-date" when the PR's base branch is behind the upstream base branch.

```
# Fetch PR details including mergeable_state
PR_DETAILS=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}")

MERGEABLE_STATE=$(echo "$PR_DETAILS" | jq -r '.mergeable_state')
BASE_REF=$(echo "$PR_DETAILS" | jq -r '.base.ref')
HEAD_SHA=$(echo "$PR_DETAILS" | jq -r '.head.sha')

# Check if behind by comparing commits
BEHIND_BY=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/{SOURCE_REPO}/compare/{BASE_REF}...{HEAD_SHA}" | \
  jq -r '.behind_by // 0')

if [ "$BEHIND_BY" -gt 0 ] && [ "$MERGEABLE_STATE" = "behind" ]; then
  NEEDS_REBASE=true
  echo "PR #{pr_number} is $BEHIND_BY commits behind $BASE_REF"
fi
```

If `NEEDS_REBASE` is true, add to `REBASE_NEEDED` list for a sub-agent to update the branch.

**4.7 — Check for merge conflicts:**

If `mergeable_state` is `dirty`, the PR has conflicts that need resolution:
```
if [ "$MERGEABLE_STATE" = "dirty" ]; then
  echo "PR #{pr_number} has merge conflicts - notify author"
  # Add to notification list but don't auto-fix (requires human decision)
fi
```**

**Concurrency Check — CRITICAL:**

Before adding any PR to action lists, verify no gh-issues sub-agent is working on related branches:

```
# Check gh-issues claims file for any active work on this repo
CLAIMS_FILE="$DATA_DIR/gh-issues-claims.json"
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
PR_CLAIMS_FILE="$DATA_DIR/gh-prs-claims.json"
if [ -f "$PR_CLAIMS_FILE" ]; then
  CUTOFF=$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-2H +%Y-%m-%dT%H:%M:%SZ)
  ACTIVE_PR_CLAIMS=$(cat "$PR_CLAIMS_FILE" | jq --arg repo "{SOURCE_REPO}" --arg cutoff "$CUTOFF" \
    '[to_entries[] | select(.value.repo == $repo and .value.expires > $cutoff)]')
  
  # CRITICAL FIX: If gh-prs already has an active claim on this PR, SKIP it
  for key in $(echo "$ACTIVE_PR_CLAIMS" | jq -r '.[].key'); do
    CLAIM_PR=$(echo "$key" | cut -d'#' -f2)
    if [ "$CLAIM_PR" = "{pr_number}" ]; then
      echo "⚠️  Skipping PR #{pr_number} — already claimed by gh-prs (key: $key)"
      remove_from_action_list {pr_number}
    fi
  done
fi
```


**Also check gh-prs state for PRs currently being reviewed:**

```
# Before adding a PR to the review list, check if it's already in-progress
PROCESSING_STATUS=$(cat "$STATE_FILE" | jq -r ".processed_prs[\"{pr_number}\"].status // empty")
if [ "$PROCESSING_STATUS" = "reviewing" ] || [ "$PROCESSING_STATUS" = "fixing" ]; then
  echo "⚠️  Skipping PR #{pr_number} — already in progress (status: $PROCESSING_STATUS)"
  remove_from_action_list {pr_number}
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
  ACTIVE_ISSUES_CLAIM=$(cat "$DATA_DIR/gh-issues-claims.json" 2>/dev/null | jq -r --arg key "{SOURCE_REPO}#{pr_number}" '.[$key] // empty')
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

Before spawning, write claim AND update state to prevent concurrent operations:
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
PR_CLAIMS_FILE="$DATA_DIR/gh-prs-claims.json"
if [ ! -f "$PR_CLAIMS_FILE" ]; then
  echo '{}' > "$PR_CLAIMS_FILE"
fi

# Add claim for this PR
jq --arg key "{SOURCE_REPO}#{pr_number}" --argjson val "$CLAIM_DATA" \
  '.[$key] = $val' "$PR_CLAIMS_FILE" > "${PR_CLAIMS_FILE}.tmp" && \
  mv "${PR_CLAIMS_FILE}.tmp" "$PR_CLAIMS_FILE"

# CRITICAL: Also mark as in_progress in state file BEFORE spawning
# This prevents re-queueing on the next poll even if cron fires while agent is running
STATE_STATUS="reviewing"
if [ "{action}" = "fix-checks" ]; then
  STATE_STATUS="fixing"
fi

jq --arg pr "{pr_number}" --arg status "$STATE_STATUS" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '.processed_prs[$pr] = {"status": $status, "started_at": $time}' \
  "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
```

**7.2 — Spawn Review Sub-agent (with comment tracking & approval):**

For each PR needing review:

```yaml
runtime: subagent
mode: run
task: |
  You are a code reviewer. Review PR #{pr_number} in {SOURCE_REPO}.
  
  ## Instructions
  1. CLONE: If not already cloned, shallow clone the repo to a temp location:
     git clone --depth 100 https://x-access-token:$REVIEWER_TOKEN@github.com/{SOURCE_REPO}.git /tmp/gh-prs-review-{pr_number}
  
  2. FETCH: Get PR details and diff:
     - PR info: curl -H "Authorization: Bearer $REVIEWER_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}
     - Files: curl -H "Authorization: Bearer $REVIEWER_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/files
     - Diff: curl -H "Authorization: Bearer $REVIEWER_TOKEN" -H "Accept: application/vnd.github.v3.diff" https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}
  
  3. ANALYZE: Review the code changes:
     - Check for bugs, security issues, performance problems
     - Verify test coverage for new code
     - Check code style and best practices
     - Look for missing error handling
     - Verify documentation updates if needed
  
  4. COMMENT: Post review comments using GitHub API and capture comment IDs:
     - For inline comments on specific lines:
       ```
       COMMENT_RESPONSE=$(curl -s -X POST -H "Authorization: Bearer $REVIEWER_TOKEN" \
         https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/comments \
         -d '{"commit_id":"{head_sha}","path":"file.js","line":42,"body":"comment"}')
       COMMENT_ID=$(echo "$COMMENT_RESPONSE" | jq -r '.id')
       ```
     - For general PR review:
       ```
       REVIEW_RESPONSE=$(curl -s -X POST -H "Authorization: Bearer $REVIEWER_TOKEN" \
         https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/reviews \
         -d '{"commit_id":"{head_sha}","body":"summary","event":"COMMENT","comments":[...]}')
       REVIEW_ID=$(echo "$REVIEW_RESPONSE" | jq -r '.id')
       ```
  
  5. APPROVAL: If {CAN_APPROVE} is set to true (all previous gh-prs comments resolved, no new critical issues, CI passing):
     - Submit approval:
       ```
       curl -s -X POST -H "Authorization: Bearer $REVIEWER_TOKEN" \
         https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/reviews \
         -d '{"commit_id":"{head_sha}","body":"All review comments addressed. LGTM!","event":"APPROVE"}'
       ```
  
  6. REPORT: Return a summary:
     - Number of issues found by category (critical, warning, suggestion)
     - Files reviewed
     - Lines of code changed
     - Review submitted: yes/no
     - Approval submitted: yes/no
     - **JSON comment tracking data** (for new comments created)

constraints:
  - If CAN_APPROVE: submit approval, else just leave comments
  - Be constructive and specific in feedback
  - Time limit: 30 minutes
  - MUST return JSON comment tracking data if new comments created
agentId: {REVIEWER_AGENT}
model: {REVIEWER_MODEL}
runTimeoutSeconds: 1800
cleanup: keep
```

**7.2.1 — Check for Comment Resolution Before Spawning:**

Before spawning reviewer, check if previous gh-prs comments were addressed:
```bash
# Fetch existing gh-prs comments from state
EXISTING_COMMENTS=$(cat "$STATE_FILE" | jq -r ".pr_comments[\"{pr_number}\"] // []")

if [ "$(echo "$EXISTING_COMMENTS" | jq 'length')" -gt 0 ]; then
  # Fetch current PR files to compare
  CURRENT_FILES=$(curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
    "https://api.github.com/repos/{SOURCE_REPO}/pulls/{pr_number}/files")
  
  RESOLVED_COUNT=0
  for comment in $(echo "$EXISTING_COMMENTS" | jq -c '.[]'); do
    COMMENT_ID=$(echo "$comment" | jq -r '.id')
    FILE_PATH=$(echo "$comment" | jq -r '.file')
    LINE_NUM=$(echo "$comment" | jq -r '.line')
    
    # Check if the file/line was modified in recent commits
    # If patch doesn't include this line anymore, mark as resolved
    if ! echo "$CURRENT_FILES" | jq -e --arg f "$FILE_PATH" '.[] | select(.filename == $f)' >/dev/null; then
      # File removed/changed significantly - mark resolved
      RESOLVED_COUNT=$((RESOLVED_COUNT + 1))
      # Update state to mark comment resolved
      jq --arg pr "{pr_number}" --arg cid "$COMMENT_ID" \
        '.pr_comments[$pr] |= map(if .id == $cid then .status = "resolved" else . end)' \
        "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
    fi
  done
  
  # Check if all comments resolved
  OPEN_COMMENTS=$(cat "$STATE_FILE" | jq -r ".pr_comments[\"{pr_number}\"] // [] | map(select(.status == \"open\")) | length")
  if [ "$OPEN_COMMENTS" -eq 0 ]; then
    CAN_APPROVE=true
  fi
fi
```

**7.2.2 — Update State with New Comments:**

After reviewer finishes, parse output for comment tracking JSON:
```bash
# Parse comment tracking from reviewer output
NEW_COMMENTS='{pr_number}__comment_json_from_output'
if [ -n "$NEW_COMMENTS" ] && [ "$NEW_COMMENTS" != "null" ]; then
  # Add status field to each comment
  COMMENTS_WITH_STATUS=$(echo "$NEW_COMMENTS" | jq '[.comments[] | . + {"status": "open"}]')
  jq --arg pr "{pr_number}" --argjson data "$COMMENTS_WITH_STATUS" \
    '.pr_comments[$pr] = $data' "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
fi

# Update processed_prs
jq --arg pr "{pr_number}" --arg sha "{head_sha}" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '.processed_prs[$pr] = {"status": "reviewed", "last_sha": $sha, "reviewed_at": $time}' \
  "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
```

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
     git clone --depth 100 https://x-access-token:$CODER_TOKEN@github.com/{SOURCE_REPO}.git /tmp/gh-prs-checks-{pr_number}
     cd /tmp/gh-prs-checks-{pr_number}
     # Configure git identity for TARSCoder App commits
     git config --global user.name "TARSCoder"
     git config --global user.email "3558889+tars-coder@users.noreply.github.com"
     git config --global --add safe.directory /tmp/gh-prs-checks-{pr_number}
  
  2. FETCH: Get PR branch and check details:
     - git fetch origin pull/{pr_number}/head:pr-branch
     - git checkout pr-branch
     - Get failed checks: curl -H "Authorization: Bearer $CODER_TOKEN" https://api.github.com/repos/{SOURCE_REPO}/commits/{head_sha}/check-runs
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
     git remote set-url origin https://x-access-token:$CODER_TOKEN@github.com/{SOURCE_REPO}.git
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

**7.4 — Spawn Rebase Sub-agent (for out-of-date PRs):**

For each PR that needs to be updated with latest base branch changes:

```yaml
runtime: subagent
mode: run
task: |
  You are updating a PR branch with the latest changes from the base branch.
  
  ## PR Details
  - PR #{pr_number} in {SOURCE_REPO}
  - PR branch: {head_ref}
  - Base branch: {base_ref}
  - Commits behind: {behind_by}
  
  ## Instructions
  1. CLONE: Set up workspace:
     git clone --depth 100 https://x-access-token:$CODER_TOKEN@github.com/{SOURCE_REPO}.git /tmp/gh-prs-rebase-{pr_number}
     cd /tmp/gh-prs-rebase-{pr_number}
     # Configure git identity for TARSCoder App commits
     git config --global user.name "TARSCoder"
     git config --global user.email "3558889+tars-coder@users.noreply.github.com"
     git config --global --add safe.directory /tmp/gh-prs-rebase-{pr_number}
  
  2. FETCH and CHECKOUT PR branch:
     git fetch origin pull/{pr_number}/head:{head_ref}
     git checkout {head_ref}
  
  3. FETCH base branch:
     git fetch origin {base_ref}:{base_ref}
  
  4. REBASE or MERGE (prefer rebase for clean history):
     Option A - Rebase (preferred):
       git rebase origin/{base_ref}
       
     Option B - Merge (if rebase has conflicts):
       git merge origin/{base_ref} -m "Merge {base_ref} into {head_ref}"
  
  5. PUSH: Update the PR branch:
     git remote set-url origin https://x-access-token:$CODER_TOKEN@github.com/{SOURCE_REPO}.git
     git push origin HEAD:{head_ref} --force-with-lease
  
  6. REPORT: Success/failure and method used (rebase vs merge)

constraints:
  - Prefer rebase for clean history
  - If conflicts: abort rebase, use merge instead
  - Never force push without --force-with-lease
  - Time limit: 15 minutes
agentId: {FIXER_AGENT}
model: {FIXER_MODEL}
runTimeoutSeconds: 900
cleanup: keep
```

**After rebase:** Update state to mark PR as rebased and trigger re-review since new commits were added:
```bash
jq --arg pr "{pr_number}" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '.rebased_prs[$pr] = {"rebased_at": $time}' "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
```

**Cron mode behavior:**
- Spawn all agents without waiting
- Exit immediately
- State will be updated on next run

---

## Phase 8 — Update State & Cleanup

**8.1 — Update State File:**

```bash
STATE_FILE="$DATA_DIR/gh-prs-state-$REPO_SLUG.json"

# Update processed PRs — preserve last_sha if already set, update status
for pr in $REVIEWED_PRS; do
  # Get existing sha if any
  EXISTING_SHA=$(jq -r ".processed_prs[\"$pr\"]?.last_sha // empty" "$STATE_FILE")
  if [ -n "$EXISTING_SHA" ] && [ "$EXISTING_SHA" != "null" ]; then
    jq --arg pr "$pr" --arg sha "$EXISTING_SHA" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '.processed_prs[$pr] = {"status": "reviewed", "last_sha": $sha, "reviewed_at": $time}' \
      "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
  else
    jq --arg pr "$pr" --arg sha "{head_sha}" --arg time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '.processed_prs[$pr] = {"status": "reviewed", "last_sha": $sha, "reviewed_at": $time}' \
      "$STATE_FILE" > tmp.json && mv tmp.json "$STATE_FILE"
  fi
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
PR_CLAIMS_FILE="$DATA_DIR/gh-prs-claims.json"
for pr in $PROCESSED_PRS; do
  jq --arg key "{SOURCE_REPO}#$pr" 'del(.[$key])' "$PR_CLAIMS_FILE" > tmp.json && mv tmp.json "$PR_CLAIMS_FILE"
done
```

**8.3 — Release Workspace Lock:**
```
rm -f "{WORKSPACE_DIR}/.gh-prs-lock"
```

**8.4 — Tally Counts for Idle Detection:**
Before Phase 9, count what happened this run so the notification gate can skip on idle:
```
REVIEW_COUNT=$(echo "$PROCESSED_PRS" | grep -c '.' || echo "0")
FIX_COUNT=$(echo "$FIXED_CHECKS" | grep -c '.' || echo "0")
ERROR_COUNT=$(echo "$FAILED_ERRORS" | grep -c '.' || echo "0")
# Export for use in Phase 9 idle check
export REVIEW_COUNT FIX_COUNT ERROR_COUNT
```

---

## Phase 9 — Results Summary

**Skip output on idle polls (no notifications by default):**
```
if [ "$REVIEW_COUNT" = "0" ] && [ "$FIX_COUNT" = "0" ] && [ "$ERROR_COUNT" = "0" ]; then
  echo "gh-prs: no action needed — all PRs current"
  exit 0
fi

# Notify on approvals and CI fixes (only when there's real action)
if [ "$REVIEW_COUNT" -gt "0" ] || [ "$FIX_COUNT" -gt "0" ]; then
  # Sub-agent should have already posted approvals/fixes to GitHub.
  # Send a brief notification to the channel so Mike knows action is needed.
  NOTIFY_SUMMARY="gh-prs: $REVIEW_COUNT review(s), $FIX_COUNT CI fix(es) — review needed"
fi
```

Otherwise, present full summary:

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

1. **Workspace Separation:** gh-prs uses `$DATA_DIR/gh-prs-workspace/` while gh-issues uses the main workspace
2. **Claim Checking:** Before any operation, check `$DATA_DIR/gh-issues-claims.json` for active claims
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
curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/owner/repo/pulls?state=open&sort=updated"

# Get PR review requests
curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/owner/repo/pulls/55/requested_reviewers"

# Get PR diff
curl -s -H "Authorization: Bearer $REVIEWER_TOKEN" \
  -H "Accept: application/vnd.github.v3.diff" \
  "https://api.github.com/repos/owner/repo/pulls/55"

# Post review comment
curl -X POST -H "Authorization: Bearer $REVIEWER_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/owner/repo/pulls/55/comments" \
  -d '{"commit_id":"abc123","path":"file.js","line":10,"body":"Nice work!"}'

# Rerun failed checks
curl -X POST -H "Authorization: Bearer $REVIEWER_TOKEN" \
  "https://api.github.com/repos/owner/repo/actions/runs/{run_id}/rerun-failed-jobs"
```

---

## Notes

- **Comment addressing is intentionally excluded** — gh-issues skill handles review comments on its `fix/issue-*` PRs
- gh-prs only reviews code and fixes CI failures
- Always specify `--repo owner/repo` when not in a git directory
- **App permissions:** Both TARSReviewer and TARSCoder GitHub Apps must be installed on the target repo. TARSReviewer needs `pull_requests: write` (for reviews/comments) and `contents: read`. TARSCoder needs `contents: write` (for pushes) and `pull_requests: write` (for commenting).
- Rate limits: 5000 requests/hour for authenticated users
- In fork scenarios, PR branches are on the fork, but comments go to the upstream PR
