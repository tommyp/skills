---
name: pr-comments
description: Review unresolved PR comments, evaluate each on its merits, and apply fixes that are warranted — pushing back (in chat) when a comment is wrong.
dependencies: gh
---

## Instructions

Ask for edit permissions if you don't have them already.

Fetch review threads via the GraphQL API so you get the `isResolved` flag
(the REST `/pulls/:n/comments` endpoint does **not** expose resolution
state). Use:

```bash
gh api graphql -F owner=OWNER -F repo=REPO -F pr=PR_NUMBER -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            isOutdated
            comments(first: 20) {
              nodes { id author { login } body path line }
            }
          }
        }
      }
    }
  }'
```

**Ignore any thread where `isResolved` is `true`.** Resolved threads are
done — re-raising them wastes the user's time and can undo intentional
decisions. Process only `isResolved: false` threads.

For each remaining comment, evaluate it on its merits — do NOT default
to compliance. Automated reviewers (Copilot, CodeRabbit, etc.) and
humans alike can be wrong, overly defensive, or unaware of project
context.

For each comment, classify into one of:

- **apply** — the comment is correct and worth fixing. Make the change and
  summarise what you changed.
- **reject** — the comment is wrong, misleading, or its suggestion would
  make the code worse. Explain your reasoning to the user and wait for
  their call before doing anything else with it.
- **ambiguous** — you can see arguments either way. Ask the user.

### Signals that justify rejecting a comment

- **Typespec/schema vs. production reality.** A field is `t() | nil` in
  the typespec but in practice always populated. Defending against the
  nil case would add dead code.
- **Local convention overrides.** The project's style guides or
  established patterns already cover this; the suggestion conflicts.
- **Misreads the code.** Reviewer didn't trace the call path, missed a
  guard upstream, or the proposed change would break existing callers.
- **Nitpick with no behaviour change.** Stylistic preference that adds
  noise without changing behaviour, and isn't backed by a project rule.

When in doubt, surface to the user with your reasoning — don't silently
apply, and don't silently ignore.

### Important

**Never post replies on GitHub.** All reasoning, agreement, and pushback
goes to the user in chat only. The user will decide whether and how to
respond on the PR themselves.

If there are no unresolved comments, say so.

## Output format

For each comment, report in chat:

- Comment author + a one-line summary of what they're saying
- Your verdict: apply / reject / ambiguous
- Your reasoning (especially important for reject and ambiguous)
- For apply: what you changed

End with a one-line summary of the overall outcome (e.g. "Applied 3, rejected 1, 1 needs your input").
