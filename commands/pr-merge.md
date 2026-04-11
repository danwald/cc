---
name: pr-merge
description: Safely merge a pull request after verifying CI and approvals. Usage: /pr-merge <PR#> [owner/repo] [squash|merge|rebase]
argument-hint: <PR#> [owner/repo] [squash|merge|rebase]
allowed-tools: Bash(gh pr view:*), Bash(gh pr checks:*), Bash(gh pr merge:*), Bash(gh repo view:*), Bash(git checkout:*), Bash(git pull:*)
---

Merge the pull request in $ARGUMENTS safely.

Parse $ARGUMENTS:
- Token 1: PR number (required)
- Token 2: repo as `owner/repo` (optional — omit `-R` if not provided)
- Token 3: merge strategy — `merge` (default), `squash`, or `rebase`
- Flag `--no-confirm`: if present, skip the confirmation prompt in Step 3 and proceed directly to merge

## Step 1 — Verify PR state

```
gh pr view <PR#> [-R <repo>] --json number,title,state,mergeable,reviewDecision,statusCheckRollup,reviews
```

Show: title, merge status, review decision, and overall CI result.

**Stop and abort if:**
- `state` is not `OPEN`
- `mergeable` is `CONFLICTING` — tell the user to resolve conflicts first
- `reviewDecision` is `CHANGES_REQUESTED` — tell the user which reviewer blocked it

## Step 2 — Check CI

```
gh pr checks <PR#> [-R <repo>]
```

**Stop and abort if any required check is failing.** List the failing checks by name.

If all checks pass (or there are no required checks), proceed.

## Step 3 — Confirm merge

Show a summary:
```
PR #<n>: <title>
Strategy: <merge|squash|rebase>
Repo: <repo>
```

If `--no-confirm` was passed, skip this step and proceed. Otherwise ask: "Confirm merge? (yes / no)" and wait for explicit confirmation before proceeding.

## Step 4 — Merge

```
gh pr merge <PR#> [-R <repo>] --<strategy> --delete-branch
```

## Step 5 — Sync local (if in the repo directory)

```
gh repo view [-R <repo>] --json defaultBranchRef -q .defaultBranchRef.name
git checkout <default-branch>
git pull
```

Only run the git commands if the working directory appears to be the target repo. If not, skip and tell the user to pull manually.

Confirm success with the PR URL and merged status.
