---
name: pre-review
description: Pre-review the current branch before colleagues review the PR. Checks the diff against the recurring feedback my reviewers actually leave — style-guide adherence, CSS token layering, accessibility, correctness traps, internal consistency, naming, and scope. Use before opening a PR or requesting review.
dependencies: gh
---

## Purpose

Catch the things my colleagues would flag *before* they have to. The checklist below is
distilled from real review comments left on my PRs (by @slashdotdash, @Copilot,
@ijdickinson, @josef-bolt, @violet-hall-verna). Apply it as a reviewer would: report
findings, don't silently rewrite.

## Step 1 — Scope the diff

```bash
git diff main...HEAD --stat
git diff main...HEAD --name-only
```

If the diff is empty, stop and say so.

## Step 2 — Load the conventions

Read `CLAUDE.md`, `AGENTS.md`, everything under `docs/style_guides/`, and `.credo.exs`.
These are the primary source of truth — reviewers cite them by link. Also read the **full**
changed files, not just the hunks.

## Step 3 — Review against the recurring feedback

Go file by file. Flag anything matching the patterns below. Each is something a reviewer
has actually pushed back on.

### Style-guide adherence (most common — @slashdotdash)

- **Component `@doc` ordering**: for `Phoenix.Component` function components, `@doc` must
  come *after* the `attr`/`slot` definitions and immediately *before* the `@spec`/function
  head. Not at the top.
- **Aliases for internal modules**: any `Hyphae.*`/`HyphaeWeb.*` module referenced by full
  namespace inline should have an `alias`. This applies to typespecs too
  (`Rendered.t()` not `Phoenix.LiveView.Rendered.t()`) and even to sample code inside docs
  and ADRs — example code must not violate the style guide.
- **Mix task namespacing**: Hyphae Mix tasks live under `Mix.Tasks.Hyphae.*` so they get the
  `hyphae.` prefix and group together in `mix help`. File named `hyphae.<task>.ex`.
- **British English** spelling in all naming.
- **`:for` / `:if` over block forms** in HEEx (also an internal-consistency point — see below).
- **Destructuring over struct/map dot access**: `field.errors`, `assigns.id`, `socket.assigns.x`
  etc. outside HEEx must be destructured instead (`%FormField{errors: errors} = field`,
  `%{id: id} = assigns`). The one exception is *inside* HEEx templates, where non-nested
  `@foo.bar` is allowed — but nested access (`@foo.bar.baz`) still isn't; extract it into an
  assign before the template. Look especially at FormField-resolution clauses
  (`field.name`/`field.value`/`field.errors`) and any `socket.assigns.*` reads — these are
  plain Elixir function bodies, not templates, so the exception doesn't apply.
- **Boolean attrs and assigns get a `?` suffix**, no exceptions for "it matches the HTML
  attribute name" (e.g. `attr :show_count?, :boolean`, not `:show_count`, even though the
  rendered HTML has no `?`). Check both the `attr` declaration and any local assign/variable
  built from it (`show_denominator?`, not `show_denominator`).
- **Batch multiple `assign/2` calls into one**: when several keys are being set on the socket
  or assigns in sequence, combine them into a single `assign(socket, key1: val1, key2: val2,
  ...)` rather than chaining separate `assign` calls.
- **`@doc` examples must use real attr names.** If an example shows `error={true}` but the
  component actually defines `attr :error?`, copying the example produces broken/no-op code.
  Check every example snippet against the actual `attr`/`slot` names in the same file.
- Check the diff against the Credo rule summary in the Elixir style guide.

### CSS token layering & formatting (colocated CSS — @slashdotdash, @Copilot, @ijdickinson)

Applies to any `<style :type={Phoenix.LiveView.ColocatedHook}>`-style colocated CSS or
`assets/css/*.css`. This is the most-repeated line of feedback on component PRs — check it
even when nothing else looks wrong.

- **Components must never reference primitives directly** (`--stroke-1`, `--radius-m`, any
  numbered/sized scale value). Primitives are the one tier allowed to be a scale; components
  must go through a semantic/intent token. If a needed semantic token doesn't exist yet,
  that's a real gap to raise, not a reason to reach for the primitive.
  - **Exception — typography and spacing primitives are allowed directly**: the
    `--font-family-*`, `--font-size-*`, `--font-weight-*`, and `--space-*` scales are the
    established direct-use layer for components (every sibling component uses them). Do not
    flag a component for referencing these.
- **No hard-coded values where a token should exist**: literal `rem` sizes, literal
  `font-weight: 600`, etc. in component styles — flag and ask whether a semantic var covers
  it, even if none currently exists (worth raising rather than silently hard-coding).
- **No BEM-style class names** in colocated CSS (`--` modifiers, `__` elements, e.g.
  `.alert--no-title`, `.icon__error`). Flat single-dash names instead (`.alert-no-title`,
  `.icon-error`) — see [[feedback_css-class-and-ordering-conventions]].
- **Colocated CSS should read as if Prettier-formatted**: multi-line rule bodies (one
  declaration per line, blank line between rules), not compressed single-line blocks. Flag
  dense one-line CSS rules the same way you'd flag unformatted Elixir.
- **`:has()` / pseudo-class rules**: verify the property actually changes on the element the
  user is interacting with. E.g. a `.checkbox:has(.input:disabled)` rule that sets `cursor`
  on the `<label>` does nothing if an absolutely-positioned inner element still has its own
  `cursor: pointer` and sits on top.
- If touching the scoped-CSS transform itself (`colocated_scoped_css.ex` or similar): check
  the selector-matching regex doesn't also rewrite keyframe selectors (`from`/`to`/`0%`) or
  at-rules (`@font-face`) into invalid output.
- **Custom-drawn item styling shouldn't duplicate a `<style>` block per rendered item**: a
  private renderer invoked once per list entry (e.g. a `radio_option/1` called once per
  radio) must not embed its own `<style :type={ColocatedScopedCSS}>` — hoist the CSS into the
  single style block owned by the parent/collection component and keep the per-item renderer
  purely structural (classes only).
- **Prefer CSS-driven checked/active state over a server-computed class** for custom
  radio/checkbox-like controls: toggle the visual indicator with a sibling `:checked`
  selector (e.g. `.input:checked ~ .ring ...`) so it responds instantly to a click rather
  than only after a LiveView round-trip, and stays consistent with how sibling components
  (e.g. `Checkbox`) already do it.
- **Generic slot-adjacent class names leak into slot content**: names like
  `.title`/`.content`/`.footer` cascade into whatever markup a caller passes into a slot.
  Prefix with the component name instead (`.card-title`, `.list-row-content`) — this has come
  up on more than one component PR.

### Accessibility (@Copilot)

- **Decorative icons need `aria-hidden`** (or equivalent) so they aren't announced —
  especially icons that duplicate adjacent visible text.
- **Meaningful icons need a text alternative**: an icon that's the *only* indicator of
  severity/state (error/warning/info/success) should have accessible text conveying that
  meaning, not just a visual colour/shape difference.
- **`role="alert"` is assertive** — reserve it for genuinely urgent/error variants. Use
  `role="status"` (polite) for neutral/info/success variants so screen readers aren't
  interrupted for routine content.
- **Don't use `<label>` as a bare layout wrapper** (no `for=`, no label text). It confuses
  assistive tech and blocks composing a real `<label for=...>` later. Use a neutral `<div>`
  wrapper instead.
- **`:rest, :global` must actually be spread onto the element it documents.** Declaring the
  attr isn't enough — if the wrapper meant to receive `aria-label`/etc. is missing `{@rest}`,
  global attributes silently do nothing. Check every render path a component has, especially
  a secondary one (e.g. the dot-only branch of a component that also has a labelled branch)
  — it's easy to wire `rest` into the primary path and forget the alternate one.
- **Text-less/icon-only/dot-only variants need a doc example and Storybook story that
  actually demonstrates the `aria-label` pattern**, not just the capability to accept one. A
  story with no accessible name modelled invites callers to skip it too.

### Correctness traps (@Copilot, @ijdickinson)

- **Validation set vs. actual clauses**: if code validates input against a whitelist
  (e.g. `MapSet.member?`) but the handling function only has clauses for a subset, valid
  input can pass validation then hit a fallback exception. The validation set must match
  the handled cases.
- **Raising before the intended error**: e.g. `String.to_existing_atom/1` raises its own
  unhelpful error before the component's nice `unknown X` message can fire. Rescue and
  re-raise in the expected format.
- **Normalise input before branching on it**: e.g. comparing `method == "get"` fails to
  match `method={:get}` or `"GET"`, silently falling through to the wrong branch. Convert
  with `to_string/1` + `String.downcase/1` (or similar) before comparison whenever the type
  or case of the input isn't guaranteed by a typespec.
- **Boolean/checkbox form fields need the hidden-input pairing**: an unchecked HTML checkbox
  submits no parameter at all, so a boolean field's component must render a hidden input
  (when `name` is set) plus an explicit checkbox `value`, matching the convention already
  used elsewhere in the codebase (e.g. `Input`). Don't let a checkbox component silently
  drop `false` from form params.
- **Don't casually reimplement a Phoenix helper's safety net**: a hand-rolled equivalent of
  `Phoenix.Component.link/1` (or similar) that skips its href sanitisation (blocking
  `javascript:` URIs, etc.) is a latent XSS gap the instant a caller passes a non-literal
  URL. If reimplementing for a good reason (e.g. scoped-CSS attribute needs), call out what
  safety behaviour is being dropped.
- **`<button>` defaults to `type="submit"`**: any button not meant to submit a form needs an
  explicit `type="button"` attr (as a real `attr` with a default, not passed through `rest`)
  to avoid accidental form submission.
- **Undeclared slots**: a component calling `render_slot(@inner_block)` (or any named slot)
  must declare it with `slot`, so callers get compile-time validation and docs pick it up.
- **Attrs that are effectively required should be `required: true`**: if a component always
  renders the attr's value with no fallback (e.g. `name` on an input that always renders
  `<input name={@name}>`), mark it required so a missing value is a compile-time HEEx error,
  not a runtime `nil` rendering.
- **Docs/contract mismatch**: a moduledoc or comment claiming broader support than the code
  allows (e.g. "supports every HTML input type" when `attr :type` restricts to a handful of
  values) will mislead the next person who extends it. Either the doc or the restriction is
  wrong — flag the gap.
- **Prefer compile-time safety over magic strings**: a `name="foo"` string that becomes a
  runtime exception on typo is worse than one-function-per-variant giving a compile-time
  error (and free `@doc`). Flag magic-string dispatch where a typed alternative fits.
- **Docs describing future behaviour as present**: ADRs/READMEs that say CI "fails" on
  something not yet implemented should be worded as a normative decision ("CI must…"), not
  current fact. Watch broken links (branch vs. PR) and product-name accuracy
  ("GitHub Actions workflow" not "a GitHub Action").
- **`value || fallback` doesn't treat `""` as absent** — only `nil` and `false` are falsy in
  Elixir, so an explicitly empty string wins over the fallback. Anywhere a caller-supplied
  attr is meant to fall back to a derived value (e.g. a `field`-derived `id`/`name`), guard
  against blank string too (`if value in [nil, ""], do: fallback, else: value`). This also
  affects derived/concatenated ids (`"#{id}-trigger"`): a nil/blank base id silently produces
  a malformed id like `"-trigger"` that can collide across instances — worth an explicit
  check (or a required attr, see above) rather than trusting the fallback alone.
- **Booleans pulled from a slot attribute map (`Map.get(option, :checked)`) come back `nil`
  when omitted, not `false`.** Normalise to an explicit boolean before using it in
  `&&`/`class`/`:if` logic so downstream comparisons stay boolean rather than `nil`-vs-truthy.
- **Scope `phx-update="ignore"` to only the DOM a hook actually owns.** Ignoring the whole
  control blocks LiveView from patching anything inside it after mount — including
  `value`/`errors`/`disabled`/`loading`-driven changes a form-validation cycle needs to apply.
  If part of the ignored subtree must still react to server state, either narrow the ignored
  region or reconcile in `updated()` (see [[project_select-updated-hook-reconciliation]]).
- **Guard colocated-hook keyboard/pointer handlers against empty collections.** A hook that
  opens a listbox/menu and immediately highlights index `0` (or acts on the "active" index)
  will throw if the option list is empty and the index is left at `-1`. Check length before
  acting on an index.

### Test quality (@slashdotdash, @Copilot)

- **Chained `=~` assertions can be pointless**: if a later assertion's string is a superset
  of an earlier one (`assert html =~ "5"` followed by `assert html =~ "500"`), the first
  passes trivially regardless of what it's meant to check. Prefer one exact match on the
  full rendered string (heredoc `==`), or LazyHTML selectors that target the specific
  element/attribute, over stacked substring checks — see the multi-line assertion rule in
  the [Elixir style guide](../../../docs/style_guides/elixir_style_guide.md).
  Related: [[feedback_no-nested-function-calls]].
- **A substring match doesn't prove *where* the text landed**: `assert html =~ "Hello
  world"` passes if that string appears *anywhere* in the output — it doesn't confirm it's
  actually bound to the attribute/element under test (e.g. the textarea's `value`). Use a
  LazyHTML selector against the specific attribute instead.

### Internal consistency (@ijdickinson, @Copilot)

- Inline `<%= for %>` loops that could be `:for` attributes — convert for consistency
  unless the loop body isn't a single tag (then a wrapping element / inline form is the
  documented reason; note it).
- Inline `<%= if %>`/`cond` conditions in HEEx that could be extracted to a helper function
  — same principle as `:for`/`:if`, applies to conditions too, not just loops.
- Several near-identical calls that vary only by one enum-like value (e.g. five separate
  `<.icon>` calls, one per alert variant) — suggest collapsing into a single helper function
  that pattern-matches on the variant instead.
- Missing `alias` for a module used in a story/test/module. Note the `List` collision
  escape hatch: `alias HyphaeWeb.Components.List, as: ComponentsList`.
- `describe` blocks should name the module being tested, not just the function
  (`describe "ColocatedScopedCSS.transform/4"`, not `describe "transform/4"`) — makes it
  clear where the tested code lives without checking the top of the file.
- **Avoid recursive self-dispatch across a multi-clause component**: don't derive assigns in
  one clause then recurse back into the same public multi-clause function just to reach the
  "real" render clause. Factor the shared render body into a private function (or a single
  clause with an internal `case`) that every public clause calls directly instead.
- **Prefer the semantic element over a generic `<div>`/`<p>` when one fits the content's
  role**: a footer-like block should be a `<footer>`, a card/panel heading should render as an
  actual heading element (default `h2`, but let the caller override the level via an attr)
  rather than a styled `<p>`, and an outer container should weigh `<section>` vs `<article>`
  vs a plain wrapper based on whether the content is independently distributable (`article`)
  or just a thematic grouping (`section`).
- Storybook `:ai_mode` (or similar) variation-group wrappers should reuse the flex+gap layout
  other component stories already apply for grouped variations, not an unstyled wrapper div.

### Naming & semantics (@violet-hall-verna, @ijdickinson)

- **No literal colour names in the token/semantic layer** (`--accent-lime`,
  `--surface-cream`). A token that means "lime" breaks the moment a theme makes it
  non-lime. Flag semantic names that encode a concrete value.
- **Near-duplicate tokens** resolving to the same value (token proliferation) — question
  whether both are needed.
- **T-shirt size abbreviations should be consistent**: if the design system has settled on
  abbreviated sizes (`s`/`m`/`l`/`xl`), don't reintroduce spelled-out variants
  (`large`/`small`) in a new component — check sibling components for the current
  convention before naming a new `size` attr's values.
- Variant/token names worth a second look when they encode an implementation detail rather
  than intent (e.g. `ghost` vs. a more descriptive `text-only`) — soft, non-blocking, but
  worth surfacing as a question.

### Explanatory comments (@josef-bolt)

- Non-obvious code (compiler-warning workarounds, business-rule edge cases, "why not the
  obvious thing") should carry a brief comment explaining *why*. Flag clever code that a
  future reader would stumble on. (Balance against the project rule to avoid superfluous
  comments — only where the reason is genuinely non-obvious.) This applies doubly to
  regex-based transforms (e.g. a CSS selector-scoping regex) that are otherwise
  indecipherable on their own.

### Scope (@slashdotdash)

- Flag anything that expands MVP scope where it could be reduced (e.g. supporting eight
  theme×mode states when one theme + AI/non-AI modes would do). Not a blocker, but worth
  surfacing as "is this needed for MVP?" — reviewers ask this.

## Step 4 — Report

**Write the report as visible chat text in this reply — always, no exceptions.** A
structured tool call (e.g. `ReportFindings`) is not a substitute: its output isn't
rendered as a numbered list the user can see and reference, so calling it alone leaves
the report effectively invisible. If such a tool exists and something else requires
calling it, still restate the same numbered list as markdown text in the same reply.

Group findings by file, most important first. Number every finding sequentially across the
*entire* report (1, 2, 3, ... — not restarting per file or per severity), so the user can
refer back to one by number alone ("fix 2 and 5"). For each:

- **Number** — `#1`, `#2`, ... in report order
- **File:line** — clickable `path:line`
- **Severity** — blocker / should-fix / nit / question
- **What & why** — the issue and which convention or reviewer pattern it maps to
- **Suggested fix** — concise, ideally a `suggestion`-style snippet

End with a short verdict: is this ready for colleague review, or are there blockers to
clear first. Don't apply fixes unless asked — this is a review pass. If the user then wants
them applied, resolve their request by number (e.g. "fix 1 and 2" means findings #1 and #2
from the report just given), and run `mix format` and `mix credo --strict` on touched Elixir
afterward.
