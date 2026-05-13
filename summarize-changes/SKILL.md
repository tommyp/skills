---
name: summarize-changes
description: Summarize the changes between this branch and the main branch, and flag anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff --stat main...HEAD`

## Instructions

Summarize the changes above, then list any risks you notice such as missing error handling, hardcoded values, or tests that need updating. If the diff is empty, say there are no changes.
