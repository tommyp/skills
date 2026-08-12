---
name: chunk-commits
description: Split the working tree's uncommitted changes into a sequence of small, logical, reviewable commits, grouped by concern rather than by file, each following the repo's commit-message style. Use when there's a pile of uncommitted work that should land as more than one commit.
dependencies: git
---

## Purpose

Turn a messy pile of uncommitted changes into a clean sequence of atomic commits —
the history a careful engineer would have produced by committing as they went. Group by
*concern*, not by file type: a component, its own tests, and its own docs/story wiring
belong in one commit together; a stray unrelated edit picked up along the way gets its
own commit (or gets flagged to drop).

This skill only reorganizes **uncommitted working-tree changes** into new commits. It
does not rewrite existing history (no rebase/amend of commits that already exist) — that
is a different, riskier operation and out of scope here.

## Step 1 — Survey the change

```bash
git status
git diff --stat
git diff --cached --stat
git log --oneline -20
```

If `git status` is clean, say so and stop. `git log` tells you this repo's
commit-message tone (imperative mood, typical length, capitalization) — match it later.

Capture the full, exact diff before touching anything — **`git diff HEAD` alone silently
omits untracked (new) files**, so if `git status` shows any `??` entries, stage
everything first to get a complete capture, then unstage again (safe, non-destructive —
`git reset` only unstages, it never touches working-tree content):

```bash
git add -A
git diff --cached HEAD > /tmp/chunk-commits-original.diff
git reset
```

If there are no untracked files, `git diff HEAD > /tmp/chunk-commits-original.diff` alone
is sufficient. This capture is the ground truth you'll verify the final result against in
Step 6 — get it right, since Step 6 does a byte-for-byte comparison against it.

## Step 2 — Read the actual diffs, not just the stat

Read the full content of the capture from Step 1 (or the relevant per-file diffs) to
understand *what* changed and *why* — not just which files. For any file that mixes more
than one concern (e.g. a real change plus an incidental, unrelated edit elsewhere in the
same file), identify the exact hunks that belong to each concern before proposing chunks.

## Step 3 — Propose the chunking plan, then stop

Group the changes into an ordered list of chunks. Each chunk becomes one commit.

Grouping heuristics:

- **One logical concern per commit.** Implementation, its own tests, and its own
  docs/story/config wiring stay together — don't mechanically split "all `.ex` files"
  from "all test files."
- **Split apart genuinely unrelated changes** bundled into the same diff (incidental
  reformatting, an unrelated one-line fix picked up along the way). These get their own
  small commit, or get flagged as noise worth dropping/reverting instead of committing.
- **Group by meaning, not by file boundary.** Never split a file's related hunks across
  commits just to keep "one file per commit"; conversely, never force unrelated hunks in
  the same file into the same commit just because they share a file.
- **Order for a coherent narrative.** Foundational/config changes before the feature that
  needs them; each commit should ideally leave the tree in a working, buildable state.
- **Keep messages very concise — a single short subject line, no body.** These PRs get
  squashed on merge, so the PR description is what carries the full "why"; the intermediate
  commit messages just need to be scannable while reviewing the sequence, e.g. `Add select
  component`, `Wire up typecheck in CI`. Skip prose, skip bullet lists, skip a
  Co-Authored-By trailer.

Present the plan before touching git state: for each chunk, list the files/hunks it will
include, a one-line rationale, and a draft commit message. Chunking calls are subjective
and mildly annoying to unpick after the fact — get sign-off (or adjustments) before
staging anything. If the user says to just proceed without reviewing the plan, skip the
wait and move straight to Step 4.

## Step 4 — Stage each chunk precisely

Start from a clean index — this is safe and non-destructive, it only unstages, it never
touches working-tree content:

```bash
git reset
```

Then, per chunk, in the agreed order:

- **Whole files** (new, deleted, or entirely one concern): `git add <path>`.
- **A file whose hunks split across chunks**: get its patch with `git diff -- <path>`,
  write a copy containing only the hunks for the *current* chunk (drop the rest — hunk
  headers stay valid as long as whole hunks are removed rather than partial-hunk lines),
  then apply just that patch to the index:

  ```bash
  git apply --cached /path/to/chunk.patch
  ```

  If a single hunk genuinely interleaves two concerns line-by-line (not just adjacent
  hunks), don't silently force a split — flag it to the user and either commit it whole
  under the more significant concern or ask them to confirm the exact line-level split.

- **Before committing**, self-check with `git diff --cached --stat` (should show exactly
  the intended files for this chunk) and `git diff --stat` (should show only what's left
  for later chunks). If anything looks off, stop and re-plan rather than commit blind.
- **Never stage anything that looks like a secret** (`.env`, credentials, keys) even if it
  was part of the original diff — flag it to the user instead.

## Step 5 — Commit each chunk

One commit per chunk, in the planned order, with a single-line subject and nothing
else — no body, no Co-Authored-By trailer (these commits get squashed on merge, so the
PR description carries the "why," not the individual commit messages). Never amend,
never force-push, never skip hooks (`--no-verify`). If a pre-commit hook fails, fix the
underlying issue, re-stage, and create a new commit rather than retrying with hooks
bypassed.

Repeat Step 4 → Step 5 until the working tree is clean.

## Step 6 — Verify and report

```bash
git log --oneline <original-HEAD>..HEAD
git status
git diff <original-HEAD> HEAD > /tmp/chunk-commits-result.diff
diff /tmp/chunk-commits-original.diff /tmp/chunk-commits-result.diff
```

The last `diff` should be empty — proof that the sequence of new commits reproduces
*exactly* the original uncommitted change, with nothing dropped, duplicated, or
corrupted in the split. Report the final commit list and confirm the tree is clean. Stop
there — this skill never pushes; that's a separate, explicit ask.
