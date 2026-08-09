---
name: ab-compare
kind: function
version: 0.1.0
---

# A/B Compare

### Description

Compare what a change *does*, not just what it *edits*: run the same command
before and after a change, produce a plain-language account of what stayed the
same and what changed, and — once the reviewer approves — record that account on
the pull request.

**Skill-only — no Reactor.** This runs in a single bounded OpenProse session as
imperative ProseScript choreography (order matters: A, then B, then compare, then
a human gate). There is no world-model, no reconciler, and no receipts — this is
the language/skill layer, not the reactor harness.

### Parameters

- `base-ref`: the "before" ref — the PR's base (see README for the one-liner that
  reads it from a PR)
- `head-ref`: the "after" ref — the PR's head
- `produce-command`: the shell command whose output is the thing to compare
- `focus`: optional — what to compare most closely (passed through to the diff)
- `pr`: optional — the PR to record the result on; omit to only produce the report

### Returns

- `report`: the same/different comparison Markdown (also written to
  `ab-comparison.md` for review)

### Invariants

- A is `base-ref`, B is `head-ref`; each is pinned to a SHA so the result is
  reproducible and correctly labeled.
- Each side runs in its own isolated `git worktree`; the reviewer's working tree
  and any unsubmitted changes are never touched.
- The PR is modified only after explicit reviewer approval of `report`.

### Progress

Maintain a live status ledger at `ab-run-status.md` so the reviewer can always
see where the run is (skill-only has no reactor receipts, so the pipeline keeps
its own). Rewrite it whenever a stage changes state — markers:
`○` not started · `⏳` running · `✓` done · `✗` failed.

```
1. ○ produce A (base)
2. ○ produce B (head)
3. ○ compare A vs B
4. ○ reviewer gate
5. ○ publish to PR
```

Flip a stage to `⏳` immediately **before** starting it and to `✓`/`✗`
immediately **after** it settles, so a long-running produce step reads as
visibly in-flight rather than a black box.

### Execution

```prose
# Each stage flips its `ab-run-status.md` marker to ⏳ before and ✓/✗ after.

# Stage 1 — produce A (base). Sequential by default; a runner with independent
# worktrees could wrap stages 1 and 2 in a `parallel:` block.
let a = call run-at-ref
  ref: base-ref
  produce-command: produce-command

# Stage 2 — produce B (head).
let b = call run-at-ref
  ref: head-ref
  produce-command: produce-command

# Stage 3 — the verbal same/different description (the point of the exercise).
let comparison = call compare-outputs
  output-a: a.output
  output-b: b.output
  focus: focus

# Stage 4 — reviewer gate: the report is written to `ab-comparison.md` for review.
# Stage 5 — publish, only if a PR is given AND the reviewer approves.
if pr is set and the reviewer approves comparison.report for publication:
  call publish-comparison
    report: comparison.report
    pr: pr

return comparison.report
```
