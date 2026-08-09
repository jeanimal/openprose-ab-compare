---
name: run-at-ref
kind: function
version: 0.1.0
---

# Run at Ref

### Description

Produce one side of an A/B comparison: check out a git ref in an isolated
worktree, run the caller's produce-command there, and capture the result —
deciding nothing about A vs B. Commands are expected to be long-running and
multi-step; wait for completion and capture the whole output.

### Parameters

- `ref`: the git ref to evaluate (branch, tag, or SHA — e.g. a PR's base or head)
- `produce-command`: the shell command whose output is the thing to compare
  (a build, a test run, a script, a query — whatever "output" means here)

### Returns

- `output`: the captured stdout / stderr / named artifacts the command produced
- `resolved-ref`: the exact SHA `ref` resolved to, for labeling and reproducibility

### Shape

- `self`: `git rev-parse` the ref to a SHA, create a throwaway `git worktree` at
  that SHA, run `produce-command` inside it, wait for it to finish, capture the
  output, then remove the worktree
- `prohibited`: touching the caller's checkout, committing, pushing, or running
  the command anywhere but the isolated worktree

### Strategies

- Pin `resolved-ref` up front so A and B are labeled by SHA, not by a branch name
  that could move mid-run.
- Capture a non-zero exit as part of `output` rather than aborting — "it stopped
  building at this ref" is itself a comparison result.
