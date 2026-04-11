---
name: pr-merbumrel
description: Merge open bump/dependency PRs, bump the semver version (patch by default), then trigger a release. Usage: /pr-merbumrel [owner/repo] [patch|minor|major]
argument-hint: [owner/repo] [patch|minor|major]
allowed-tools: Bash(gh pr list:*), Bash(gh repo view:*)
---

Orchestrate merge → version bump → release for open bump/dependency PRs in $ARGUMENTS.

Parse $ARGUMENTS:
- Token 1: repo as `owner/repo` (optional — auto-detect via `gh repo view --json nameWithOwner -q .nameWithOwner`)
- Token 2: bump type — `patch` (default), `minor`, or `major`

---

## Step 1 — Find open bump PRs

```bash
gh pr list -R <repo> --state open --json number,title,author,headRefName
```

Filter to PRs where author login is `app/dependabot` or `renovate`, or title matches `bump`, `update deps`, `chore(deps)` (case-insensitive).

Show the matching PRs in a table. If none are found, stop and tell the user.

---

## Step 2 — Merge each bump PR

For each matching PR number, invoke `/pr-merge <PR#> <repo> --no-confirm`.

Stop if any merge fails before proceeding to the next step.

---

## Step 3 — Bump version

Invoke `/bump-version <bump-type> --no-confirm`.

---

## Step 4 — Create release

`/bump-version --no-confirm` will automatically create a GitHub release.
