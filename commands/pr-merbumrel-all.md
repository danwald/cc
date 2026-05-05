# /pr-merbumrel-all

Run `/pr-merbumrel` for multiple repositories in parallel using background agents.

## Usage

```
/pr-merbumrel-all <repo1> <repo2> <repo3> ... [--patch|--minor|--major]
```

## Examples

```bash
/pr-merbumrel-all danwald/pickemOdder danwald/bulk-note          # patch (default)
/pr-merbumrel-all danwald/butterfly danwald/saxtract --minor     # minor bump
/pr-merbumrel-all owner/repo1 owner/repo2 owner/repo3 --major    # major bump
```

## What it does

1. Parses the repo list and bump type (patch/minor/major, default: patch)
2. Spawns a background Agent for each repo
3. Each agent runs `/pr-merbumrel <repo> <bump-type>` independently
4. Reports completion status for each repo

## Implementation

When invoked, Claude parses arguments and spawns parallel agents using the Agent tool with `run_in_background=True`, each running `/pr-merbumrel` for its assigned repo. This avoids sequential blocking when orchestrating across multiple repositories.

For each repo argument:
1. Extract repo name (owner/repo format)
2. Create Agent task: `Agent(description=f"Run pr-merbumrel for {repo}", prompt=f"/pr-merbumrel {repo} {bump_type}", run_in_background=True)`
3. Report progress and results once all agents complete
