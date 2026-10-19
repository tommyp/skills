---
name: to-pr
description: Draft a sentence-case PR title and a direct summary description in my house style, present them for approval, then open the PR with `gh pr create` (or update it with `gh pr edit` if one already exists for the branch). Takes an optional detail level, `concise` (default) or `verbose`. Use when the user wants to open a PR, turn the current branch into a PR, or asks to run /to-pr.
argument-hint: "[concise|verbose]"
dependencies: gh
---

## Purpose

Turn the current branch into a PR with a title and description written the way I write them.
Short, opens with what the change does, no padding, no test-plan checklist, no bot footer.
Always show the draft before opening anything. If a PR already exists for the branch, update
its title and description in place.

## Detail level

The argument is $ARGUMENTS. It selects one of two levels.

**concise** is the default when no argument is given. One short paragraph of plain English,
no bullets and no headings, however many concerns the change has. It's a high-level summary:
what the change lets someone do, and the one or two guarantees that matter most. Three or four
plain sentences, written for someone who will read the diff for the detail. So no walk-through
of the mechanism, no step-by-step of a pipeline, and no module or function names unless the
reviewer must go to one. If a sentence needs a comma-separated list to fit, it's too detailed
for this level. Anything the reviewer must do or decide (a migration to run, a deliberate
design seam) may follow as one short second paragraph.

**verbose** is the structured form for a change a reviewer can't take in from a paragraph,
such as a new subsystem or several interacting invariants. Open with a one-line statement of
what the change does. Then give each distinct behaviour its own bullet, with the reasoning
behind non-obvious decisions, the invariants the code now guarantees (lock ordering,
idempotency, what a retry may and may not do), and anything a reviewer should look at closely.
Never a file-by-file replay of the diff.

Every other rule in this file applies to both levels: the same opening, register, exclusions
and output format. Verbose adds structure and explanation of the design. It doesn't add tests,
documentation, future work or narrative. If the argument is anything other than `concise` or
`verbose`, say so in the lead-in to the draft and use concise.

## Step 1: Scope the change and check for an existing PR

```bash
git branch --show-current
git status --short
git log main..HEAD --oneline
git diff main...HEAD --stat
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
gh pr view --json url,title,isDraft 2>/dev/null
```

If there are uncommitted changes, stop and ask whether to commit them first. Don't fold them
into the PR silently, and don't commit anything without being asked.

If `gh pr view` finds a PR already open for this branch, this run is an **update**: tell the
user it already exists (show its URL and current title), and skip the draft/ready question.
Nothing is being created, and `gh pr edit` doesn't touch that state. With no existing PR, ask
whether this should be a **draft** PR or ready for review.

## Step 2: Title

A short noun phrase naming what the PR delivers, in sentence case (capitalise only the first
word and proper nouns): "Modal component", "Projects list view", "Target editing". No leading
verb, so "Modal component", not "Add a Modal component".

Derive it from the branch name or its ticket when that names the feature:
`tommy/hyphae-688-add-a-modal-component` becomes "Modal component". If the branch name is
generic (`fix`, `wip`, a raw ticket id), summarise the diff. In update mode, keep the existing
title if it already has this form and the diff hasn't outgrown it.

## Step 3: Description

Read the full diff (`git diff main...HEAD`), not just the stat, and skim the commit log if
there are several commits.

The description summarises what changed, stated directly. The detail level sets its shape and
how much of the why it gives. No background narrative.

- Open with a plain affirmative sentence saying what the change does: "Adds a `Modal` design
  system component built on a native `<dialog>`." or "Enables auditing on the organisations
  table." No "This PR..." preamble
- Describe behaviour and intent. Never replay the diff file by file
- Mention anything the reviewer must do or decide: migrations to run, config to set, open
  questions. State a deliberate design seam as a decision with its reason, never as a question
- Leave out tests, verification and how the change was checked. They're mandatory, so
  mentioning them is noise. The one exception is a fact the reviewer must act on, such as a
  deliberate coverage gap or tests that need a special command, and it gets one sentence at
  most. When in doubt, omit
- Leave out documentation changes for the same reason, unless they make up most of the change
  or the reviewer particularly needs to know about the docs themselves. When in doubt, omit
- Don't mention what the PR doesn't do, unless a reader would reasonably expect it ("no
  migration needed" when the diff touches a schema-adjacent area)
- Don't invent ticket references, reviewers or checklists. Include a ticket link only if the
  branch name or conversation provides one
- Don't cite design documents. Never the `.plans/` directory, which is local-only and
  git-excluded so reviewers can't read it, and not architecture decision records either. The
  summary stands alone
- No "Generated with Claude Code" footer and no Co-authored-by trailer. Just the title and body
- Sentence case for any headings, British English spelling
- Backtick every filename, path and code identifier: token and variable names (`--radius-3`),
  file paths (`assets/css/tokens.css`), function and module names, glob patterns
  (`*.tokens.json`)

### Verbose lists

- Separate the item from its detail with ` - `, not a full stop. The item is usually a
  function, module or token name, so it reads as a label:
  `` `Modal.open/2`, `Modal.close/2` - JS commands that dispatch events to a `.Modal` hook ``
- Call out anything unusual (a rename, a behaviour change, a workaround) in a list introduced
  with `Notes:`, never "Things a reviewer might ask about" or similar

### Screenshots

If the diff is a visual change (CSS, a component template, anything LiveView-rendered), add a
`## Screenshot` (one image) or `## Screenshots` (more than one) section. This is the one
heading the concise level allows. In update mode, only add this section if the existing PR body
doesn't already have one worth keeping.

Capture each changed Storybook variation with headless Chrome against the dev server on port
4000 (never start it). The checked-out branch is what renders, so switch to the PR's branch
and `mix compile` first. Size the window tight to the content and render at 2x:

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --hide-scrollbars --virtual-time-budget=5000 --window-size=<w>,<h> \
  --force-device-scale-factor=2 --screenshot=<scratchpad>/<name>.png \
  "http://localhost:4000/storybook/iframe/components/<category>/<component>?variation_id=<id>"
```

Read each PNG back before using it. An unstyled render means the CSS hadn't rebuilt yet, so
take it again.

Under the heading, reference each image by its local path with alt text,
`![Detail list with five rows](./detail_list.png)`, and pass the file with `--attach` to
`gh pr create` or `gh pr edit`, run from the directory the paths are relative to. `gh` uploads
the file and rewrites the local reference to the uploaded URL in place. `--attach` needs `gh`
2.99.0 or later. If `gh --version` is older and `brew upgrade gh` fails, download the release
binary with `gh release download --repo cli/cli --pattern '*macOS_arm64.zip'` into the
scratchpad and use that. If no screenshot can be captured, fall back to a `_TODO: screenshot_`
placeholder and create the PR anyway.

### Unslop pass

Run the `unslop` skill over the title and body and apply what it finds, before showing the
draft in Step 4. Never present a draft that hasn't been through it. The tells that keep
surviving into these descriptions are rhetorical inversions ("Gone are the..."), reassurance
filler ("Nothing breaks today", "worth a look"), em dashes, incidental detail the diff stat
already shows, and sentences that cram two unrelated points together.

## Step 4: Present for approval

Show the detail level used, the title, the full body exactly as it will appear (screenshot
section included, with the local image paths and the PNGs read back so the user can see them),
and whether it's a draft (new PR) or an update to the existing PR. Do this **before** creating
or editing anything. Wait for explicit go-ahead. If the user asks for changes, fold them in and
re-show before proceeding.

## Step 5: Push and open (or update)

Push the branch if it has no upstream yet, or has unpushed commits. Confirm first, since this
is the point where local work becomes visible to others.

For a new PR, create it with the approved title and body via a heredoc so formatting survives:

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)" [--draft] [--attach ./<name>.png ...]
```

For an update to an existing PR, edit it in place. Don't pass `--draft` or any
ready-for-review flag, since `gh pr edit` doesn't touch that state and none was requested:

```bash
gh pr edit <number> --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)" [--attach ./<name>.png ...]
```

Report the PR URL back to the user.
