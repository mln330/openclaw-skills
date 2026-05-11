# Issue Execution Plan

## Format

This file uses a simple dependency graph format. Each issue is declared with its dependencies.
Issues with no dependencies can run in parallel. Issues with dependencies wait for their prerequisites.

## Syntax

```
# Issue execution plan for {owner/repo}
# Dependencies: issue_number: [prerequisite_issue_numbers]
# Groups: group_name: [issue_numbers]  # (optional - runs in parallel)

# Define dependencies
dependencies:
  3: [1]         # Issue 3 depends on issue 1 being completed
  5: [1, 2]      # Issue 5 depends on issues 1 AND 2
  7: [4]         # Issue 7 depends on issue 4
  8: [3, 4]      # Issue 8 depends on issues 3 AND 4

# Define parallel groups (optional)
groups:
  bootstrap: [1, 2, 4]    # These can all run in parallel
  core: [3, 5, 7]        # These wait for their dependencies
  polish: [8]            # Final phase

# Constraints
max_parallel: 4           # Maximum issues to process simultaneously
review_after_each: true   # Wait for PR review before starting dependents
```

## Rules

1. **Issues with no dependencies** → start immediately, in parallel (up to `max_parallel`)
2. **Issues with dependencies** → wait until ALL prerequisites are marked done
3. **"Done" means:** PR merged or closed, OR PR opened and marked `skip_review: true`
4. **Circular dependencies** → detected at load time, error reported, cycle broken by processing lowest-numbered first
5. **Missing issues** → if a dependency references an issue not in the fetched list, it's ignored (warning logged)
6. **No execution plan file** → falls back to sequential processing (oldest issue first)

## Status Tracking

The skill maintains state in `~/.openclaw/workspace/.gh-issues-data/{repo-slug}-execution-state.json`:

```json
{
  "issues": {
    "1": { "status": "merged", "pr": 99, "completed_at": "2026-05-09T21:00:00Z" },
    "2": { "status": "in_progress", "pr": 101, "started_at": "2026-05-09T21:30:00Z" },
    "3": { "status": "pending", "blocked_by": [1] }
  }
}
```

## Minimal Example

```yaml
dependencies:
  # Database schema must exist before API endpoints
  15: [12]   # Issue 15 (API) depends on issue 12 (schema)
  
  # Frontend needs both API and auth
  20: [15, 18]  # Issue 20 (frontend) depends on API (15) and auth (18)
  
groups:
  foundation: [12, 18]
  api: [15]
  ui: [20]

max_parallel: 3
```
