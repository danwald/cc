---
name: release-create
description: Create a GitHub release with auto-generated changelog. Usage: /release-create <version> [owner/repo] [--pre]
argument-hint: <version> [owner/repo] [--pre]
allowed-tools: Bash(gh release list:*), Bash(gh release create:*), Bash(gh repo view:*)
---

Create a GitHub release for the version in $ARGUMENTS.

Parse $ARGUMENTS:
- Token 1: version string — accept with or without leading `v` (normalise to `v<version>` for the tag)
- Token 2: repo as `owner/repo` (optional)
- Flag `--pre` anywhere in args: mark as pre-release

## Step 1 — Check latest release

```
gh release list [-R <repo>] --limit 3 --json tagName,publishedAt,isPrerelease
```

Show the last 3 releases so the user can confirm the version bump makes sense.

## Step 2 — Confirm

Show a plan:
```
New release tag:  v<version>
Repo:             <repo or "current">
Pre-release:      yes/no
Changelog:        auto-generated from merged PRs
Target:           main (default branch)
```

Ask: "Create this release? (yes / no)"  Wait for confirmation.

## Step 3 — Create the release

```
gh release create "v<version>" \
  [-R <repo>] \
  --title "v<version>" \
  --generate-notes \
  --target main \
  [--prerelease]
```

`--generate-notes` automatically builds the changelog from merged PRs since the previous release — no manual editing needed.

## Step 4 — Confirm

Print the release URL. Tell the user they can view the auto-generated changelog on GitHub and edit it if needed.
