---
name: to-pr
description: Draft a sentence-case PR title and concise "This PR..." description in my house style, present them for approval, then open the PR with `gh pr create` (or update it with `gh pr edit` if one already exists for the branch). Use when the user wants to open a PR, turn the current branch into a PR, or asks to run /to-pr.
dependencies: gh
---

## Purpose

Turn the current branch into a PR with a title and description written the way I actually
write them — short, starting with "This PR...", no padding, no boilerplate test-plan
checklist, no bot footer. Always show the draft before opening anything. If a PR already
exists for the branch, update its title/description in place instead of creating a new one.

## Step 1 — Scope the change and check for an existing PR

```bash
git branch --show-current
git status --short
git log main..HEAD --oneline
git diff main...HEAD --stat
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
gh pr view --json url,title,isDraft 2>/dev/null
```

If there are uncommitted changes, stop and ask whether to commit them first — don't fold
them into the PR silently, and don't commit anything without being asked.

If `gh pr view` finds a PR already open for this branch, this run is an **update**, not a
new PR: tell the user it already exists (show its URL and current title), and don't ask the
draft/ready question in Step 2 — there's nothing being created, and `gh pr edit` doesn't
touch that state. Otherwise (no existing PR), ask whether this should be a **draft** PR or
ready for review.

## Step 2 — Title

Short, sentence case (capitalise only the first word and proper nouns). Usually derived from
the branch name or the ticket it's named after — turn `bump-design-tokens-05-08` into
something like "Bump design tokens", not a literal slug dump. If the branch name is generic
(`fix`, `wip`, a raw ticket id) fall back to summarising the diff instead. In update mode,
it's fine to keep the existing title if the diff hasn't outgrown it — no need to force a
change for its own sake.

## Step 3 — Description

Read the full diff (`git diff main...HEAD`), not just the stat, so the summary is accurate —
skim the commit log too if there are several commits.

- Open with a couple of sentences starting with **"This PR..."**
- Summarise the changes — prose or a list, whichever reads better for the size of the change
- Call out anything unusual (a rename, a behaviour change, a workaround) — this is the part a
  reviewer would ask about if it went unmentioned
- Don't mention what the PR *doesn't* do, unless a reader would reasonably expect it (e.g.
  "no migration needed" when the diff touches a schema-adjacent area)
- No test-plan checklist, no "Generated with Claude Code" footer, no Co-authored-by trailer —
  just the title and body
- **Be concise.** Prioritise brevity over completeness — a sharp four-line description beats
  a padded ten-line one
- **Backtick every filename, path, and code identifier** — token/var names (`--radius-3`),
  file paths (`assets/css/tokens.css`), function/module names, glob patterns (`*.tokens.json`).
  Applies throughout the whole body, not just in a list of changes.

### Screenshots

If the diff is a visual change (CSS, a component template, anything LiveView-rendered), add
a `## Screenshot` (one image) or `## Screenshots` (more than one) section. `gh` has no way to
upload an image to a PR body, so always leave a `_TODO: screenshot_` placeholder under the
heading and create the PR anyway — don't ask the user to attach one in chat, and don't block
on it. In update mode, only add this section if one isn't already present in the existing PR
body worth preserving.

## Step 4 — Present for approval

Show the title, the full body exactly as it will appear (screenshot section included), and
whether it's a draft (new PR) or an update to the existing PR — **before** creating or
editing anything. Wait for explicit go-ahead; if the user asks for changes, fold them in and
re-show before proceeding.

## Step 5 — Push and open (or update)

Push the branch if it has no upstream yet, or has unpushed commits (confirm first — this is
the point where local work becomes visible to others).

For a new PR, create it with the approved title/body via a heredoc so formatting survives:

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)" [--draft]
```

For an update to an existing PR, edit it in place instead — don't pass `--draft` or any
ready-for-review flag, since `gh pr edit` doesn't touch that state and none was requested:

```bash
gh pr edit <number> --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)"
```

Report the PR URL back to the user.
