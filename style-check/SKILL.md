---
name: style-check
description: Check the lines a branch or working tree adds against the project's style guides, for the rules Credo cannot enforce, and fix what is clear-cut. Use when the user asks for a style check, asks whether code follows the style guide, or before committing or opening a PR in an Elixir/Phoenix project.
---

# Style check

Credo, `mix format` and Dialyzer catch what they can. This skill covers the rest: rules that live only in prose in the project's style guides, plus a few recurring problems the guides don't name. Run it over new code before it is committed or reviewed.

## Arguments

- `$ARGUMENTS`: optional git range, such as `main...HEAD` or `abc123..def456`. With no range, check the branch against `main` plus any uncommitted changes (`git diff main`).

## Step 1: Scope

```bash
git diff <range> --stat
git diff <range> -U0
```

Keep `.ex`, `.exs`, `.heex`, `.css`, `.ts` and `.md` files. Stop and say so if none changed.

Read each changed file in full for context, but **flag only lines the diff adds or changes**. Older code in the same file is out of scope, even where it breaks a rule. If old code sets a local pattern that new code follows, mention it once and move on.

## Step 2: Load the rules

Read these before looking at any code. The project's text wins over this checklist wherever they disagree.

- `CLAUDE.md` and `AGENTS.md`, including the files they `@`-include
- every file in `docs/style_guides/`
- `docs/design_system.md` or similar, if the diff touches components or CSS
- `.credo.exs`, so you know which rules Credo already enforces and can skip them

A rule in the checklist below that the project's guides don't state does not apply. Skip it.

## Step 3: Check

Go file by file. These are the rules Credo can't see.

### Function heads and destructuring

- A struct argument is matched by name in the head (`def area(%Site{} = site)`), and its fields are destructured in the body. This applies to multi-line anonymous functions too. A one-line `fn %SiteView{id: id} -> id end` is fine.
- Fields are destructured only in the head when the match picks the clause.
- Related fields come out in one destructuring expression, with nested struct patterns for associations. Dot access (`site.id`, `assigns.x`, `socket.assigns.y`) outside HEEx is a violation. In HEEx, `@foo.bar` is fine and `@foo.bar.baz` is not.
- One function with a `case` is preferred over several clauses when it picks behavior from a mode or value. A public function with several clauses needs an empty head that has the `@doc` and `@spec`.
- `do:` only on a definition that fits on one line.
- Function heads and calls on one line where they fit. Three arguments or fewer, unless the values are unrelated.

### Names and values

- A variable name reused in a nested `fn` or block for a different kind of data (`sites` as `Site` structs outside, `SiteView` structs inside). Rename one.
- A struct field passed explicitly as its own `defstruct` default, especially when it travels through a map of facts just to reach the struct. Drop it.
- Plain ids passed between functions. Pass the struct, typed `id_only()`, and unwrap ids only at the boundary.
- Struct updates name the struct: `%Card{card | map_areas: areas}`, never `%{card | ...}`.
- One- and two-letter names, truncations and acronyms (`pref`, `m2m`). American spelling.
- Unordered groups sorted alphabetically: list literals, type unions, `Ecto.Enum` values, `defstruct` and `@enforce_keys` keys, map and keyword literals, CSS declarations and rules.
- `for` only with several generators. A single generator is `Enum.map` or `Enum.each`.
- `{:noreply, assign(socket, ...)}`: bind the socket first, return the tuple after.
- Multi-value returns as maps or structs, not tuples.

### Types and docs

- Every public function has `@doc` then `@spec`. No private function has either.
- `@moduledoc` and `@doc` are heredocs.
- Typespecs name precise types: domain structs, map shapes, value unions. No `term()`. `any()` and `map()` only where the value really is open.
- A function returning tagged tuples returns them on every path, or bare `:ok`/`:error`.
- Keyword options have a named type listing each key, and an `## Options` section in the `@doc`.
- Ecto `t()` describes a valid persisted record: required fields are not `| nil`. Creation changesets build the struct; update changesets take `t()`.
- A doc lists conditions but hides the rule that joins them. "One area per site that has a boundary or a point" hides that the point is a fallback. Write the rule: "the site's boundary, or its point when it has none".
- A doc claims more or less than the code does: the wrong arity, an attr name that doesn't exist, a value the code rejects.
- Docs and comments describe the code as it stands. Flag history and change narration ("now", "no longer", "was", "rather than the old way") and reasons for the design that belong in the PR description.
- Comments that repeat the code. Comments on standard Phoenix or LiveView idioms.

### Phoenix and HEEx

- `:if` and `:for` attributes, never `<%= if %>`, `<%= for %>`, `cond` or `case` blocks. Several branches become a function component with one `case`.
- Class lists built in the function body and assigned, not inline in the template.
- `attr` and `slot` declarations come before `@doc`, which comes right before the function. Every `attr` has `doc:`, enumerated ones have `values:`, and `:rest` is last.
- Booleans end in `?`, except an attr rendered as an HTML boolean attribute of the same name.
- Key elements have a DOM id, and tests select by those ids.
- A `phx-hook` element has an id, and `phx-update="ignore"` covers only the DOM the hook owns.

### Ecto and queries

- Queries composed from the schema's query module, one concern per function, never ad hoc `from` at call sites.
- Binding names spelled out: `membership`, not `m`.
- SQL in lowercase, in fragments, migrations and raw queries.
- Boolean fields end in `?` with `source:`. Migrations use `:text`.
- Public context functions take the scope first.

### CSS

- Colors only through semantic tokens. Spacing, radii and strokes through tokens where one fits the intent, otherwise a literal `rem`.
- `rem`, not `px`, except borders. No nesting.
- Declarations sorted alphabetically within each rule, and rules sorted by selector.
- Modifiers are flat single-dash classes (`.badge-subtle`), not BEM `--`.

### Tests

- No generated tests (loops, `unquote`, module attributes per case). Enumerate values with `Enum.each` inside one test.
- Repeated setup goes in named `setup :helper` functions.
- A multi-line target is asserted whole, not with stacked `=~` checks.
- LiveView and component assertions use `LazyHTML` selectors or `has_element?`, not substrings of raw HTML.
- No `Process.sleep` or `Process.alive?`.
- A new rule or guard has a test that fails without it.

## Step 4: Report and fix

List every finding as `path:line`, the rule, and one sentence on what is wrong. Then:

- **Clear-cut** (the guide states the rule and there is one obvious fix): fix it.
- **Judgement call** (several fixes, a rename that ripples outward, or a rule the old code in the file already breaks): report it and ask.

After fixing, run:

```bash
mix format
mix credo --strict <changed files>
mix test <test files for the changed modules>
```

A fix can trip a Credo rule. Breaking a pipe up, for example, can leave a multi-line pipeline that starts with a function call. Loop until all three are clean.

End with a one-line count: fixed, needs your call, and anything skipped because older code sets the pattern.
