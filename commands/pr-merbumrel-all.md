# /pr-merbumrel-all

Orchestrate merge → version bump → release for multiple repositories sequentially with non-interactive mode.

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

For each repository, execute the `/pr-merbumrel` workflow sequentially:
1. Find open bump/dependency PRs (dependabot, renovate, or title contains "bump"/"update deps"/"chore(deps)")
2. Merge each PR with `--no-confirm` (non-interactive)
3. Bump the semver version with `--no-confirm`
4. Trigger a GitHub release automatically

## Implementation

Parse `$ARGUMENTS`:
- All tokens except the last are repository names (owner/repo format)
- Last token may be a bump type (--patch, --minor, --major) or a repo name
  - If it looks like `--patch|--minor|--major`, use that; default is `patch`
  - Otherwise, treat it as a repo name

For each repo in sequence:
1. **Find bump PRs**: `gh pr list -R <repo> --state open --json number,title,author,headRefName` and filter for dependabot/renovate or title matching bump/deps
2. **Merge PRs**: For each PR, invoke `/pr-merge <PR#> <repo> --no-confirm`
3. **Bump version**: Invoke `/bump-version <bump_type> --no-confirm` (auto-creates release)
4. Report results (succeeded/failed count per repo)

## Why sequential, not parallel?

Background agents have permission constraints. Executing in the main session ensures all operations complete without permission issues, while `--no-confirm` keeps everything non-interactive.
