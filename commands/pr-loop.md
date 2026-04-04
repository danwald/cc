---
name: pr-loop
description: Run a PR or release operation across multiple repos in one go. Usage: /pr-loop <command> <owner/repo1> <owner/repo2> ... — commands: list, merge <PR#>, release <version> [patch|minor|major]
argument-hint: <list|merge <PR#>|release <version>> <owner/repo> [owner/repo2 ...]
allowed-tools: Bash(gh pr list:*), Bash(gh pr view:*), Bash(gh pr checks:*), Bash(gh pr merge:*), Bash(gh release list:*), Bash(gh release create:*), Bash(gh workflow list:*), Bash(gh workflow run:*)
---

Run a GitHub operation across multiple repos.

$ARGUMENTS format: `<command> <repos...>`

## Parse arguments

- Token 1: sub-command — `list`, `merge`, or `release`
- For `merge`: token 2 is PR number, remaining tokens are repos
- For `release`: token 2 is version, token 3 (optional) is bump type (`patch`/`minor`/`major`), remaining tokens are repos
- For `list`: all remaining tokens are repos

---

## Sub-command: `list`

For each repo, run:
```
gh pr list -R <repo> --state open --json number,title,author,headRefName,reviewDecision
```

Show a grouped summary — one section per repo, clearly labelled. End with total open PR count.

---

## Sub-command: `merge <PR#> <repos...>`

**This is a destructive multi-repo operation. Always confirm before executing.**

For each repo:
1. Verify the PR exists and is open: `gh pr view <PR#> -R <repo> --json state,mergeable,reviewDecision`
2. Check CI: `gh pr checks <PR#> -R <repo>`

Show a pre-merge table:
```
Repo               PR#   Mergeable   CI        Review
myorg/service-a    #12   ✅          ✅ pass   APPROVED
myorg/service-b    #12   ✅          ⏳ pending REVIEW_REQUIRED
myorg/service-c    #12   ❌ CONFLICT —         —
```

Flag any repos that cannot be merged (conflicts, failing CI, changes requested). Ask:
"Merge on the repos that are ready? (yes / no / list repos to skip)"

On confirmation, merge only the ready repos:
```
gh pr merge <PR#> -R <repo> --squash --delete-branch
```

Summarise results — merged / skipped / failed.

---

## Sub-command: `release <version> [bump-type] <repos...>`

If version is a bump type keyword (`patch`/`minor`/`major`) rather than a version number, compute the next version per-repo using:
```
gh release list -R <repo> --limit 1 --json tagName -q '.[0].tagName'
```

Then apply the bump to get the per-repo next version.

Show a pre-release table:
```
Repo               Current   Next      Workflow found?
myorg/service-a    v1.2.3    v1.2.4    ✅ release.yml
myorg/service-b    v0.9.1    v0.9.2    ✅ release.yml
myorg/service-c    (none)    v0.0.1    ❌ (will use gh release create)
```

Ask: "Release all? (yes / no)"  Wait for explicit confirmation.

For each repo:
- If a release workflow exists: `gh workflow run release.yml -R <repo> --ref main -f version=<next>`
- Otherwise: `gh release create "v<next>" -R <repo> --title "v<next>" --generate-notes --target main`

Summarise: released / failed per repo.
