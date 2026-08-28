**Golden rule**: When unsure about implementation details or requirements, ALWAYS consult the developer rather than making assumptions.

---

## Non-negotiable golden rules

| #:  | AI _may_ do                                                                                                                                                                       | AI _must NOT_ do                                                                                                                                      |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| G-0 | Whenever unsure about something that's related to the project, ask the developer for clarification before making changes.                                                         | ❌ Write changes or use tools when you are not sure about something project specific, or if you don't have context for a particular feature/decision. |
| G-1 | Generate code **and tests** inside relevant source and test directories                                                                                                           | ❌ Modify `SPEC.md` without explicit approval (humans own product specs).                                                                              |
| G-2 | Add/update **`AIDEV-NOTE:` anchor comments** near non-trivial edited code.                                                                                                        | ❌ Delete or mangle existing `AIDEV-` comments.                                                                                                       |
| G-3 | Follow lint/style configs (`pyproject.toml`, `.ruff.toml`, `.pre-commit-config.yaml`). Use the project's configured linter, if available, instead of manually re-formatting code. | ❌ Re-format code to any other style.                                                                                                                 |
| G-4 | For changes >300 LOC or >3 files, **ask for confirmation**.                                                                                                                       | ❌ Refactor large modules without human guidance.                                                                                                     |
| G-5 | Stay within the current task context. Inform the dev if it'd be better to start afresh.                                                                                           | ❌ Continue work from a prior prompt after "new task" – start a fresh session.                                                                        |

---

## Project setup & coding standards

Python and TypeScript/frontend conventions, tooling, and commands are documented in `~/.claude/skills/setup-project/references/`:

- [Python](references/python.md) — uv, ruff, mypy, pytest, Ward, error handling, agents-api expressions
- [TypeScript / Frontend](references/typescript.md) — Node.js, ESLint, Prettier, Vitest, Next.js/React patterns
- [Common patterns](references/common-patterns.md) — git, CI/CD, versioning, environment variables, security

Claude Code users can also pull these in via the `setup-project` skill; other harnesses should read the files at that path directly.

---

## Anchor comments

Add specially formatted comments throughout the codebase, where appropriate, for yourself as inline knowledge that can be easily `grep`ped for.

### Guidelines

- Use `AIDEV-NOTE:`, `AIDEV-TODO:`, or `AIDEV-QUESTION:` (all-caps prefix) for comments aimed at AI and developers.
- Keep them concise (≤ 120 chars).
- **Important:** Before scanning files, always first try to **locate existing anchors** `AIDEV-*` in relevant subdirectories.
- **Update relevant anchors** when modifying associated code.
- **Do not remove `AIDEV-NOTE`s** without explicit human instruction.
- Make sure to add relevant anchor comments, whenever a file or piece of code is:
  - too long, or
  - too complex, or
  - very important, or
  - confusing, or
  - could have a bug unrelated to the task you are currently working on.

Example:

```python
# AIDEV-NOTE: perf-hot-path; avoid extra allocations (see ADR-24)
async def render_feed(...):
    ...
```

---

## Commit discipline

- **Granular commits**: One logical change per commit.
- **Tag AI-generated commits**: e.g., `feat: optimise feed query [AI]`.
- **Clear commit messages**: Explain the _why_; link to issues/ADRs if architectural.
- **Default to a `git worktree`** for AI-driven changes rather than working directly on the checked-out branch (e.g., `git worktree add ../wip-foo -b wip-foo`). Skip this only for trivial, single-line fixes applied at the user's explicit request.
- **Review AI-generated code**: Never merge code you don't understand.
- Make atomic commits for each change to code where atomic commits are one commit per logical change, no more, no less and each commit should be a self-contained unit that could be reverted independently without breaking anything else.

---

## Directory-Specific AGENTS.md Files

- **Always check for `AGENTS.md` files in specific directories** before working on code within them. These files contain targeted context.
- If a directory's `AGENTS.md` is outdated or incorrect, **update it**.
- If you make significant changes to a directory's structure, patterns, or critical implementation details, **document these in its `AGENTS.md`**.
- If a directory lacks a `AGENTS.md` but contains complex logic or patterns worth documenting for AI/humans, **suggest creating one**.

---

## Common pitfalls

- Large AI refactors in a single commit (makes `git bisect` difficult).
- Fully delegating test or spec authorship to AI without human review (can lead to false confidence).

Language-specific pitfalls (wrong CWD, `src/` layout, pytest vs ward syntax, etc.) are documented in the `setup-project` skill references.

---

## Meta: Guidelines for updating AGENTS.md files

### Elements that would be helpful to add

1. **Decision flowchart**: A simple decision tree for "when to use X vs Y" for key architectural choices would guide my recommendations.
2. **Reference links**: Links to key files or implementation examples that demonstrate best practices.
3. **Domain-specific terminology**: A small glossary of project-specific terms would help me understand domain language correctly.
4. **Versioning conventions**: How the project handles versioning, both for APIs and internal components.

### Format preferences

1. **Consistent syntax highlighting**: Ensure all code blocks have proper language tags (`python`, `bash`, etc.).
2. **Hierarchical organization**: Consider using hierarchical numbering for subsections to make referencing easier.
3. **Tabular format for key facts**: The tables are very helpful - more structured data in tabular format would be valuable.
4. **Keywords or tags**: Adding semantic markers (like `#performance` or `#security`) to certain sections would help me quickly locate relevant guidance.

---

---

## AI Assistant Workflow: Step-by-Step Methodology

When responding to user instructions, the AI assistant (Claude, Cursor, GPT, etc.) should follow this process to ensure clarity, correctness, and maintainability:

1. **Consult Relevant Guidance**: When the user gives an instruction, consult the relevant instructions from `AGENTS.md` files (both root and directory-specific) for the request.
2. **Clarify Ambiguities**: Based on what you could gather, see if there's any need for clarifications. If so, ask the user targeted questions before proceeding.
3. **Break Down & Plan**: Break down the task at hand and chalk out a rough plan for carrying it out, referencing project conventions and best practices.
4. **Trivial Tasks**: If the plan/request is trivial, go ahead and get started immediately.
5. **Non-Trivial Tasks**: Otherwise, present the plan to the user for review and iterate based on their feedback.
6. **Track Progress**: Use a to-do list (internally, or optionally in a `TODOS.md` file) to keep track of your progress on multi-step or complex tasks.
7. **If Stuck, Re-plan**: If you get stuck or blocked, return to step 3 to re-evaluate and adjust your plan.
8. **Update Documentation**: Once the user's request is fulfilled, update relevant anchor comments (`AIDEV-NOTE`, etc.) and `AGENTS.md` files in the files and directories you touched.
9. **User Review**: After completing the task, ask the user to review what you've done, and repeat the process as needed.
10. **Session Boundaries**: If the user's request isn't directly related to the current context and can be safely started in a fresh session, suggest starting from scratch to avoid context confusion.

See also: [RTK.md](RTK.md) for additional repo-specific reference material, if present.
