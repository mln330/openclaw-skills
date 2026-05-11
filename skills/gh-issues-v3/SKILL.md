---
name: gh-issues-v3
description: "Fetch GitHub issues with dependency-aware execution planning, spawn sub-agents to implement fixes respecting issue prerequisites, open PRs. Looks for ISSUES_EXECUTION_PLAN.md in the repo to determine dependencies and parallel execution order. Usage: /gh-issues-v3 [owner/repo] [--label bug] [--limit 5] [--cron] [--dry-run] [--model glm-5]"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["curl", "git", "gh", "jq"] },
        "primaryEnv": "CODER_TOKEN",
      },
  }
---

# gh-issues-v3 — Dependency-Aware Auto-fix

Respects `ISSUES_EXECUTION_PLAN.md` for issue dependencies and parallel groups. Designed for cron execution with <60s runtime.

## Quick Reference

**State file:** `~/.openclaw/workspace/.gh-issues-data/gh-issues-execution-{owner-repo}.json`

**Execution plan format (minimal):**
```yaml
dependencies:
  3: [1]         # Issue 3 depends on issue 1
  5: [1, 2]
external_prs:
  26: "feat/app-shell"  # PR blocks group
groups:
  foundation: [1, 2, 4]  # Run first
  core: [3, 5, 7]        # Run after
max_parallel: 4
```

## Phase 1 — Setup (5s)

```bash
REPO="{SOURCE_REPO}"
OWNER=$(echo "$REPO" | cut -d'/' -f1)
NAME=$(echo "$REPO" | cut -d'/' -f2)
LIMIT="{limit:-5}"
LABEL="{label}"
MILESTONE="{milestone}"
DRY_RUN="{dry_run}"
MODEL="${MODEL:-ollama/glm-5.1:cloud}"

# Auth
eval $(gh auth status -t 2>/dev/null | grep token | cut -f2)
GATEWAY_URL="${GATEWAY_URL:-http://127.0.0.1:18789}"
GATEWAY_TOKEN="${GATEWAY_TOKEN}"

# State directory
STATE_DIR="$HOME/.openclaw/workspace/.gh-issues-data"
mkdir -p "$STATE_DIR"
REPO_SLUG=$(echo "$REPO" | tr '[:upper:]' '[:lower:]' | tr '/' '-')
STATE_FILE="$STATE_DIR/gh-issues-execution-$REPO_SLUG.json"

# Init state if missing
[ -f "$STATE_FILE" ] || echo '{"issues":{},"groups":{},"last_updated":""}' > "$STATE_FILE"
```

## Phase 2 — Load Plan (5s)

```bash
# Fetch execution plan from repo
curl -s -H "Authorization: token $CODER_TOKEN" \
  "https://raw.githubusercontent.com/$REPO/main/ISSUES_EXECUTION_PLAN.md" \
  -o /tmp/plan.md 2>/dev/null || true

# Parse YAML block from markdown
PLAN=$(cat /tmp/plan.md 2>/dev/null | sed -n '/^```yaml$/,/^```$/p' | grep -v '^```')
DEPENDENCIES=$(echo "$PLAN" | yq -oj '.dependencies // {}' 2>/dev/null || echo '{}')
GROUPS=$(echo "$PLAN" | yq -oj '.groups // {}' 2>/dev/null || echo '{}')
EXTERNAL_PRS=$(echo "$PLAN" | yq -oj '.external_prs // {}' 2>/dev/null || echo '{}')
MAX_PARALLEL=$(echo "$PLAN" | yq '.max_parallel // 4' 2>/dev/null || echo "4")
```

## Phase 3 — Batch Status Check (15s)

```bash
# Single GraphQL query for ALL issues + their PRs
query='
query($owner: String!, $name: String!, $limit: Int!) {
  repository(owner: $owner, name: $name) {
    issues(first: $limit, states: OPEN'"${LABEL:+, labels: [\"$LABEL\"]}"') {
      nodes {
        number title body url
        labels(first: 10) { nodes { name } }
        timelineItems(first: 20, itemTypes: CROSS_REFERENCED_EVENT) {
          nodes {
            ... on CrossReferencedEvent {
              source { ... on PullRequest { number state merged title url } }
            }
          }
        }
      }
    }
  }
}'

resp=$(curl -s --max-time 10 -X POST \
  -H "Authorization: bearer $CODER_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.github.com/graphql" \
  -d "{\"query\": $(echo "$query" | jq -Rs .), \"variables\": {\"owner\":\"$OWNER\",\"name\":\"$NAME\",\"limit\":$LIMIT}}")

# Extract issue array
ISSUES=$(echo "$resp" | jq '.data.repository.issues.nodes // []')
COUNT=$(echo "$ISSUES" | jq 'length')
```

## Phase 4 — Determine Ready Issues (10s)

```bash
# Build dependency status map from state + PR data
READY_ISSUES="[]"
for i in $(seq 0 $((COUNT-1))); do
  issue=$(echo "$ISSUES" | jq ".[$i]")
  num=$(echo "$issue" | jq '.number')
  title=$(echo "$issue" | jq -r '.title')
  
  # Check if issue has linked PR
  pr_num=$(echo "$issue" | jq '.timelineItems.nodes[] | select(.source.__typename == "PullRequest") | .source.number // empty' | head -1)
  pr_merged=$(echo "$issue" | jq '.timelineItems.nodes[] | select(.source.__typename == "PullRequest") | .source.merged // false' | head -1)
  
  # Check dependencies from plan
  deps=$(echo "$DEPENDENCIES" | jq -r --arg n "$num" '.[$n] // [] | @json')
  deps_met=true
  if [ "$deps" != "[]" ]; then
    for dep in $(echo "$deps" | jq -r '.[]'); do
      dep_status=$(jq -r --arg d "$dep" '.issues[$d].status // "unknown"' "$STATE_FILE")
      [ "$dep_status" = "merged" ] || deps_met=false
    done
  fi
  
  # Determine status
  if [ "$pr_merged" = "true" ]; then
    status="merged"
    jq --arg n "$num" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '.issues[$n] = {"status":"merged","merged_at":$now}' \
      "$STATE_FILE" > "$STATE_FILE.tmp" && mv "$STATE_FILE.tmp" "$STATE_FILE"
  elif [ -n "$pr_num" ]; then
    status="in_review"
  elif [ "$deps_met" = "true" ]; then
    status="ready"
    # Add to ready list
    READY_ISSUES=$(echo "$READY_ISSUES" | jq --argjson i "$issue" '. + [$i]')
  else
    status="blocked"
  fi
  
  # Update state
  jq --arg n "$num" --arg s "$status" --arg t "$title" \
    '.issues[$n] = {"status":$s,"title":$t}' \
    "$STATE_FILE" > "$STATE_FILE.tmp" && mv "$STATE_FILE.tmp" "$STATE_FILE"
done

READY_COUNT=$(echo "$READY_ISSUES" | jq 'length')
```

## Phase 5 — Spawn Fixers (20s max)

```bash
# Limit to max_parallel
TO_SPAWN=$(echo "$READY_ISSUES" | jq "[.[] | select(.status == \"ready\")][:($MAX_PARALLEL)]")
SPAWN_COUNT=$(echo "$TO_SPAWN" | jq 'length')

# Truly async spawn (no waiting)
spawn_async() {
  local repo="$1" num="$2" title="$3"
  local payload=$(jq -n \
    --arg task "Implement Issue #$num for $repo: $title

Rules:
- One issue per PR
- PR title: feat(scope): short description
- PR body: closes #$num
- Run tests before pushing
- Ensure CI passes" \
    --arg model "$MODEL" \
    '{runtime:"subagent",mode:"run",task:$task,model:$model,runTimeoutSeconds:3600,cleanup:"keep",lightContext:true}')
  
  # Fire curl in background, no wait
  (curl -s --max-time 3 -X POST \
    -H "Content-Type: application/json" \
    -H "X-Gateway-Token: $GATEWAY_TOKEN" \
    "$GATEWAY_URL/api/v1/sessions/spawn" \
    -d "$payload" > /dev/null 2>&1) &
  
  # Update state immediately (don't wait for response)
  jq --arg n "$num" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    '.issues[$n].status = "in_progress" | .issues[$n].started_at = $now' \
    "$STATE_FILE" > "$STATE_FILE.tmp" && mv "$STATE_FILE.tmp" "$STATE_FILE"
  
  echo "Spawned #$num"
}

# Spawn all in parallel
for i in $(seq 0 $((SPAWN_COUNT-1))); do
  issue=$(echo "$TO_SPAWN" | jq ".[$i]")
  num=$(echo "$issue" | jq '.number')
  title=$(echo "$issue" | jq -r '.title')
  spawn_async "$REPO" "$num" "$title"
done

# Wait max 2s for background curls
timeout 2 wait 2>/dev/null || true
```

## Phase 6 — Report (5s)

```bash
# Summary output
jq -n \
  --arg repo "$REPO" \
  --argjson total "$COUNT" \
  --argjson ready "$READY_COUNT" \
  --argjson spawned "$SPAWN_COUNT" \
  '{
    repository: $repo,
    total_issues: $total,
    ready_issues: $ready,
    spawned: $spawned,
    status: ($spawned | if . > 0 then "spawned" elif ($ready | . > 0) then "blocked_by_parallel_limit" else "all_blocked_or_done" end)
  }'
```

## Cron Mode

When `--cron` is passed:
1. Skip interactive confirmation
2. Reduce log verbosity
3. Exit immediately after spawning (don't wait)

## Error Handling

- GraphQL timeout → retry once with shorter limit
- Spawn failure → mark issue as `failed`, retry next cron
- Missing execution plan → log warning, process all issues sequentially
