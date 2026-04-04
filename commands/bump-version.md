Bump the project version using `uv version --bump`, create an annotated git tag with release notes, push it, and optionally create a GitHub release.

## Arguments

`$ARGUMENTS` is the bump type: `patch` (default), `minor`, or `major`.

## Steps

1. Determine bump type from `$ARGUMENTS`. If empty or not provided, use `patch`.
2. Run `uv version --bump <type> --dry-run` to preview the new version without writing it.
3. Find the previous tag with `git describe --tags --abbrev=0` and collect commit messages since that tag with `git log <previous_tag>..HEAD --oneline`. Use these as the default release notes (formatted as a bullet list), prefixed with `Release v<new_version>`. Ask the user for release notes, showing this default.
4. Ask the user whether to create a GitHub release after pushing the tag (yes/no, default: yes).
5. Confirm with the user: show the bump type, new version, release notes, and whether a GitHub release will be created, then ask "Proceed? (yes/no)".
6. If confirmed:
   a. Run `uv version --bump <type>` to write the new version.
   b. Stage and commit the version change: `git add -u && git commit -m "Bump version to v<new_version>"`.
   c. Create an annotated git tag on the latest commit: `git tag -a v<new_version> -m "<release_notes>"`.
   d. Push the commit and tag: `git push origin HEAD && git push origin v<new_version>`.
   e. If the user opted into a GitHub release, run: `gh release create v<new_version> --title "v<new_version>" --notes-from-tag`.
7. Report the new version, confirm the tag was pushed, and (if applicable) show the GitHub release URL.

## Rules

- Only accept `patch`, `minor`, or `major` as bump type. Reject anything else with a clear error.
- Stop if `uv version --bump` fails (e.g., no `pyproject.toml`).
- Do not proceed past step 5 without explicit user confirmation.
