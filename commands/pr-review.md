---
name: pr-review
description: Review a pull request — shows diff, CI status, and submits approve or request-changes. Usage: /pr-review <PR#> [owner/repo]
argument-hint: <PR#> [owner/repo]
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh pr checks:*)
---

Review the pull request given in $ARGUMENTS.

Parse $ARGUMENTS:
- First token is the PR number
- Second token (optional) is the repo in `owner/repo` format — if absent, use current repo

## Step 1 — Fetch PR metadata

```
gh pr view <PR#> [-R <repo>] --json number,title,body,author,headRefName,baseRefName,additions,deletions,changedFiles,reviewDecision,reviews,labels
```

Display: title, author, source → target branch, +additions / -deletions, changed files count, current review decision, any existing reviews.

## Step 2 — Show CI status

```
gh pr checks <PR#> [-R <repo>]
```

Summarise: which checks passed ✅, failed ❌, or are pending ⏳. Call out any failing checks by name.

## Step 3 — Show the diff

```
gh pr diff <PR#> [-R <repo>]
```

Read the diff and provide a concise code review covering:
- What this PR does (1–2 sentences)
- Any bugs, logic errors, or security issues found
- Missing tests or edge cases
- Naming / readability concerns (only if significant)

Keep the review focused and actionable. Flag blocking issues clearly.

## Step 4 — Prompt for action

Ask the user: "Approve, request changes, or leave a comment? (approve / changes / comment / skip)"

Wait for input, then run the appropriate command:

- **approve**: `gh pr review <PR#> [-R <repo>] --approve -b "<summary of what looks good>"`
- **changes**: `gh pr review <PR#> [-R <repo>] --request-changes -b "<specific blocking issues>"`
- **comment**: `gh pr review <PR#> [-R <repo>] --comment -b "<comment text>"`
- **skip**: do nothing

Confirm the action taken.
