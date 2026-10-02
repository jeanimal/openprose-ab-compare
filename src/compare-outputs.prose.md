---
name: compare-outputs
kind: function
version: 0.1.0
---

# Compare Outputs

### Description

Given the two captured outputs, write a human-readable description of what is the
**same** and what is **different** — in plain language, about behavior the reader
cares about, not a raw byte diff. This is the step traditional A/B tooling can't
do: a semantic, verbal account of the delta.

In OpenProse's executor/evaluator terms, this is the **evaluator**: it assesses
the delta between the two results and records its findings in `report` — the
inspectable evidence, not a separate verdict value.

### Parameters

- `output-a`: the "before" output (from the base ref)
- `output-b`: the "after" output (from the head ref)
- `focus`: optional — what the reviewer most wants compared (e.g. "ranking
  order", "latency tail", "error messages"); when absent, compare broadly

### Returns

- `report`: Markdown with three parts — a one-line verdict, a **Same** section,
  and a **Different** section; every difference cites concrete evidence from the
  outputs

### Shape

- `self`: read both outputs, group observations into same / different, and render
  `report` from the structured findings
- `prohibited`: inventing differences not present in the outputs, or emitting a
  raw line diff in place of a behavioral description

### Strategies

- Separate SIGNAL from NOISE: distinguish a real change caused by the code from
  run-to-run nondeterminism (timestamps, ordering, random ids, wall-clock). Flag
  anything you suspect is noise rather than a true difference.
- When a difference is ambiguous, state it as a question for the reviewer, not a
  conclusion.
- Prefer concrete, quotable evidence over adjectives.
