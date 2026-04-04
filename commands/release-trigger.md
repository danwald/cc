---
name: release-trigger
description: Trigger a release via GitHub Actions workflow_dispatch. Usage: /release-trigger <version> [owner/repo] [workflow-filename]
argument-hint: <version> [owner/repo] [workflow-filename]
allowed-tools: Bash(gh workflow list:*), Bash(gh workflow run:*), Bash(gh run list:*), Bash(gh run watch:*)
---

Trigger the release workflow for $ARGUMENTS.

Parse $ARGUMENTS:
- Token 1: version (required)
- Token 2: repo as `owner/repo` (optional)
- Token 3: workflow filename e.g. `release.yml` (optional — auto-detect if omitted)

## Step 1 — Find the release workflow

```
gh workflow list [-R <repo>] --json name,path,state
```

If a workflow filename was given, use it. Otherwise, find the first workflow whose name matches `release` (case-insensitive). Show the matched workflow name and path to the user.

If no match found, list all available workflows and ask the user to specify one.

## Step 2 — Inspect workflow inputs (if any)

```
gh workflow view <workflow-path> [-R <repo>]
```

Check if the workflow accepts a `version` input. If it does, pass it as `-f version=<version>`. If it accepts different input names, adapt accordingly — show the user what inputs will be sent.

## Step 3 — Confirm

Show:
```
Workflow:  <name> (<path>)
Repo:      <repo or "current">
Ref:       main
Inputs:    version=<version> (or none)
```

Ask: "Trigger this workflow? (yes / no)"  Wait for confirmation.

## Step 4 — Trigger

```
gh workflow run <workflow-path> \
  [-R <repo>] \
  --ref main \
  [-f version=<version>]
```

## Step 5 — Watch (optional)

After triggering, ask: "Watch the run live? (yes / no)"

If yes:
```
gh run list [-R <repo>] --workflow <workflow-path> --limit 1 --json databaseId -q '.[0].databaseId'
gh run watch <run-id> [-R <repo>]
```

If no, just print the Actions URL so the user can monitor it manually:
`https://github.com/<repo>/actions`
