---
name: pr-list
description: List open pull requests for one or more GitHub repos. Usage: /pr-list owner/repo [owner/repo2 ...]
argument-hint: <owner/repo> [owner/repo2 ...]
allowed-tools: Bash(gh pr list:*), Bash(gh repo view:*)
---

List open PRs for each repo passed as arguments.

Arguments: $ARGUMENTS

For each repo in $ARGUMENTS (space-separated), run:

```
gh pr list -R <repo> --state open \
  --json number,title,author,headRefName,createdAt,reviewDecision,statusCheckRollup \
  --template '{{range .}}  #{{.number}}  {{.title}}
    author: {{.author.login}}  branch: {{.headRefName}}  review: {{.reviewDecision}}
{{end}}'
```

If no arguments are given, run for the current repo (no `-R` flag).

Present the results in a clear, readable table grouping PRs by repo. If a repo has no open PRs, say so explicitly. After listing, summarise total open PR count across all repos.
