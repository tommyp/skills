---
name: boy-scout
description: Review files touched by the current branch for pre-existing improvements (bugs, DRY opportunities, style violations) and offer to fix each or create a ticket.
dependencies: gh
---

## Instructions

Identify pre-existing improvements in files touched by the current branch — things that were already wrong or suboptimal before this branch's changes. These are "boy scout" opportunities: leave the code better than you found it.

### Step 1 — Determine the diff scope

```bash
git diff main...HEAD --name-only
```

If there are no changed files, stop and say so.

### Step 2 — Read project style guides and conventions

Read the project's style guides (if they exist) and CLAUDE.md so you know what to check against. Also read `.credo.exs` if present for Credo rules.

### Step 3 — Analyse each changed file

For each file in the diff:

1. Read the **full file** (not just the diff) — boy-scout improvements are about pre-existing issues, not just the lines changed in this PR.
2. Look for:
   - **Bugs** — logic errors, missing edge cases, race conditions, incorrect assumptions
   - **DRY violations** — duplicated logic that could be extracted into a shared function
   - **Style guide violations** — things that violate the project's documented conventions but predate this branch
   - **Dead code** — unused functions, unreachable branches, stale comments
   - **Naming issues** — unclear or inconsistent naming that hurts readability
   - **Missing docs** — public functions without `@doc`/`@spec` where the project requires them
   - **Simplification** — overly complex code that could be written more clearly

3. **Exclude** anything that:
   - Was introduced by this branch (that's a regular code review concern, not boy-scout)
   - Would require significant refactoring beyond the scope of a single PR
   - Is a matter of pure taste with no backing convention

### Step 4 — Present findings

For each issue found, report:

- **File and line** — path and line number(s)
- **Category** — bug / DRY / style / dead-code / naming / docs / simplification
- **Description** — what's wrong and why it matters
- **Suggested fix** — brief description of the fix

Then ask the user what to do with each issue. Present options:

1. **Fix now** — apply the fix in the current working tree (ship it in this PR)
2. **Create ticket** — create a GitHub issue describing the improvement
3. **Skip** — ignore it

### Step 5 — Execute choices

For items marked "fix now":
- Apply the fix
- Run `mix format` on changed Elixir files
- Run `mix credo --strict` to verify no new violations introduced

For items marked "create ticket":

First, determine which issue tracker to use. Check your memory for a project-level preference (e.g. "project uses Linear" or "project uses GitHub Issues"). If no memory exists, ask the user which tracker they use — then save their answer to memory so you don't ask again.

#### Supported trackers

**Linear**:
- Create via `linear issue create` CLI or ask the user to provide their team/project identifiers if needed
- Title: `<brief description>`
- Description: file path, line numbers, category, description of the issue and suggested fix

**GitHub Issues**:
- Create via `gh issue create`
- Title: `<brief description>`
- Body: file path, line numbers, category, description of the issue and suggested fix

### Notes

- Prioritise high-value findings. A real bug or a clear style violation is worth reporting; a marginal naming quibble is not.
- Batch related issues in the same file together when presenting.
- If there are many issues (>10), group by category and present the highest-value ones first, then ask if the user wants to see more.
- Don't run the full test suite unless the user asks — just format and credo as a sanity check after fixes.
