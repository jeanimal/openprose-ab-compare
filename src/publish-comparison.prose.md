---
name: publish-comparison
kind: function
version: 0.1.0
---

# Publish Comparison

### Description

The human-gated final step: after the reviewer has read the comparison and
approved it, record it on the pull request so the change's *behavioral* effect
lives alongside its code diff. Never runs without approval.

### Parameters

- `report`: the approved comparison Markdown from `compare-outputs`
- `pr`: the pull request to update (number or URL)

### Returns

- `published`: the URL of the updated PR, or `null` with a reason if not published

### Shape

- `self`: write `report` into the PR description under a clear
  `## A/B comparison` heading via `gh pr edit <pr> --body-file`, preserving the
  rest of the existing description
- `prohibited`: publishing without reviewer approval, overwriting the existing PR
  description, or posting to any PR other than `pr`

### Strategies

- Idempotent: if an `## A/B comparison` section already exists, replace it rather
  than appending a second one.
