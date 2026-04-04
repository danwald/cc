---
name: semver-bump
description: Bump the project version in pyproject.toml using uv version, then commit the change directly to the remote branch via gh api — no git pull required. Usage: /semver-bump [major|minor|patch] [owner/repo] [branch]
argument-hint: [major|minor|patch] [owner/repo] [branch]
allowed-tools: Bash(uv version:*), Bash(gh api:*), Bash(gh repo view:*), Bash(base64:*), Bash(cat:*)
---

Bump the project version using `uv version` and commit `pyproject.toml` directly
to the remote branch via the GitHub Contents API — no `git clone` or `git pull` needed.

Parse $ARGUMENTS:
- Token 1: bump type — `major`, `minor`, or `patch` (default: `patch` if omitted)
- Token 2: repo as `owner/repo` (optional — auto-detect from `gh repo view` if omitted)
- Token 3: branch to commit to (optional — defaults to repo's default branch)

---

## Step 1 — Resolve repo and branch

If repo not provided:
```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If branch not provided:
```bash
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
```

---

## Step 2 — Show current version and preview bump

```bash
uv version
uv version --bump <type> --dry-run
```

Show the user:
```
Repo:     <owner/repo>
Branch:   <branch>
Current:  <current>
Bump:     <type>
Next:     <next>
```

Ask: "Apply bump and commit to <branch>? (yes / no)"  Wait for confirmation.

---

## Step 3 — Apply the bump locally

```bash
uv version --bump <type>
```

This writes the new version into `pyproject.toml`. The `uv.lock` file may also
be updated — but only commit `pyproject.toml` to the remote (lock file changes
belong in a proper PR).

---

## Step 4 — Get the current file SHA from the remote

The GitHub Contents API requires the blob SHA of the existing file to safely
update it (prevents overwriting concurrent changes):

```bash
gh api repos/<owner>/<repo>/contents/pyproject.toml \
  --jq .sha
```

Store this as `REMOTE_SHA`.

---

## Step 5 — Commit pyproject.toml directly via gh api

```bash
gh api --method PUT repos/<owner>/<repo>/contents/pyproject.toml \
  --field message="chore: bump version to <next-version>" \
  --field branch="<branch>" \
  --field sha="<REMOTE_SHA>" \
  --field content=@<(base64 pyproject.toml)
```

The `--field content=@<(base64 pyproject.toml)` reads the base64-encoded file
content via process substitution — avoids shell argument length limits on large files.

---

## Step 6 — Confirm

Print the commit URL from the API response (`.commit.html_url`). Then suggest:
- `/release-create <next-version>` — create a GitHub release directly
- `/release-trigger <next-version>` — trigger your release workflow via `workflow_dispatch`

---

## Error handling

- **SHA mismatch / 409 conflict**: someone else pushed to the branch between
  step 4 and step 5. Re-fetch the SHA and retry once.
- **404 on pyproject.toml**: the file path may differ — check with
  `gh api repos/<owner>/<repo>/contents/ --jq '.[].name'`
- **422 Unprocessable**: branch protection may require PRs; in that case,
  advise the user to merge via a PR instead of committing directly.

---

## Note on bump types

`uv version` supports: `major`, `minor`, `patch`, `alpha`, `beta`, `rc`, `post`,
`dev`, and `stable` (promotes a pre-release to stable). Mention this if the user
appears to be managing pre-releases.
