# ab-compare

**Validate what a change *does*, not just what it *edits*.** Run the same command
before and after a GitHub pull request, get a plain-language description of what
stayed the same and what changed, and — on approval — record it on the PR.

> **Skill-only — no Reactor.** This example runs in a single bounded OpenProse
> session as imperative **ProseScript** choreography (order matters). There is no
> world-model, no reconciler, and no receipts. It exercises the OpenProse
> *language / skill* layer, not the Reactor harness — a deliberately different
> shape from the reactor examples, and a gentler on-ramp.

## Why not just `diff` or a script?

Traditional A/B tooling (and `git diff`) tells you which *lines* changed. This
tells you what the *behavior* did — "the top-3 results are stable but B reorders
the tail and drops one error message" — over long, multi-step, messy output. That
semantic verbal diff is the LLM-native part, and it's the whole point. What
OpenProse adds over a bash one-off: the choreography is a **declarative,
inspectable, reusable contract** (sync → run A → run B → compare → gate →
publish), versioned in Markdown and re-runnable by anyone on any PR. It also runs
on **local, unsubmitted code with zero infra** — no CI, no submit required.

## The pipeline (one ordered ProseScript render)

```
base-ref ─▶ run-at-ref ─▶ output A ┐
                                    ├─▶ compare-outputs ─▶ report ─▶ [human gate] ─▶ publish-comparison ─▶ PR
head-ref ─▶ run-at-ref ─▶ output B ┘
```

| File | kind | Role |
| --- | --- | --- |
| `src/ab-compare.prose.md` | function | Orchestrator — the `### Execution` that drives the steps in order |
| `src/run-at-ref.prose.md` | function | Check out a ref in an **isolated `git worktree`**, run the command, capture output |
| `src/compare-outputs.prose.md` | function | The semantic **same/different** report (signal vs. run-to-run noise) |
| `src/publish-comparison.prose.md` | function | Human-gated `gh pr edit` writeback into the PR description |

## How this runs (skill, not Reactor)

There is no server and no compile step. **An agent with the OpenProse skill *is*
the runtime:** it reads `ab-compare.prose.md`, follows the `### Execution`
ProseScript in order, and performs each `call` itself — running the shell in
`run-at-ref`, reasoning out the diff in `compare-outputs`, asking you at the gate,
and running `gh` in `publish-comparison`. One bounded session; no world-model, no
reconciler, no receipts.

(Reactor is the *other* runtime — it compiles these contracts into a memoized DAG
that keeps a world-model true over time. That's overkill for a one-shot ordered
pipeline like this, which is why this example is skill-only.)

## Run it

In a session with the OpenProse skill active:

```
prose run ab-compare
  base-ref: <base sha/branch>
  head-ref: <head sha/branch>
  produce-command: "<your build/test/script>"
  pr: <optional PR number to publish to>
```

Get a PR's base/head SHAs with the GitHub CLI:

```sh
gh pr view <PR> --json baseRefOid,headRefOid \
  -q '"base-ref=\(.baseRefOid)\nhead-ref=\(.headRefOid)"'
```

The report is written to `ab-comparison.md` for you to read. Only if you pass a
`pr` **and** approve the report does `publish-comparison` touch the PR (it appends
an idempotent `## A/B comparison` section via `gh pr edit --body-file`).

`produce-command` is a **parameter** — you supply it per run, which is what makes
one contract serve any A/B. Nothing is hard-coded or "waiting."

### A worked example you can run right now (keyless, dogfoods OpenProse)

Point it at the reactor's offline compile check across two refs of an OpenProse
project — no model key, no spend:

```
prose run ab-compare
  base-ref: HEAD~1
  head-ref: HEAD
  produce-command: "reactor --offline compile --check"
```

Run inside an OpenProse project (e.g. the sibling `openprose-wiki-pilot/reactor-project`)
across two commits where the contracts differ, it compares the compiled
**contract-set fingerprint** before vs after. The output is terse, so the diff is
modest — this proves the *harness* runs end-to-end at zero cost. In real use,
`produce-command` is your build / test / script and both the outputs and the
semantic comparison are far richer.

> **`reactor` here is the *thing under test*, not the runtime.** It's the
> `produce-command` (the subject whose output we compare) — exactly like
> `npm test` would be. `ab-compare` itself still runs **skill-only**; no reactor
> executes the pipeline. (Using a reactor command as the subject is just
> dogfooding — A/B-testing OpenProse with OpenProse.)

## Sample run (real output)

A concrete run — A/B-testing OpenProse's own offline compile check across a change
to the sibling `openprose-wiki-pilot` project, base `5e30c1a` (original contracts)
vs `HEAD` (rewritten `Maintains` schemas). Skill-only, keyless, a few seconds:

```
prose run ab-compare
  base-ref: 5e30c1a
  head-ref: HEAD
  produce-command: "reactor --offline compile --check"
```

The live `ab-run-status.md` ledger ended at:

```
1. ✓ produce A (base)  → contract-set sha256:0059afb7…, model claude-haiku-4-5
2. ✓ produce B (head)  → contract-set sha256:e1e59892…, model claude-sonnet-4-6
3. ✓ compare A vs B    → ab-comparison.md
4. ✓ reviewer gate     → no PR, nothing to publish
5. — publish           (skipped)
```

…and `ab-comparison.md` said:

> **Verdict:** both refs compile offline to `STALE` at zero cost; the compiled
> **contract set changed** and the configured **compile model changed** — the
> check didn't regress, *what* would be compiled did.
>
> **Same:** both `STALE`, SDK `0.3.1`, `fresh=0 reused=0`, same 7-contract shape.
> **Different:** contract-set `sha256:0059afb7… → sha256:e1e59892…` (the
> `state/<node>.json` schema rework); compile model `haiku → sonnet`.
> **Noise (flagged, excluded):** the `state-dir` path differs only because each
> side ran in its own worktree — not a behavioral change.

This subject's output is deliberately terse — the sample shows the *shape*
(progress ledger, semantic same/different, signal-vs-noise), not a dramatic diff.
A real build/test subject produces far richer output.

## Notes & honest edges (for reviewers of this example)

- **Nondeterminism** is handled in `compare-outputs`'s strategies: it separates
  real code-driven changes from run-to-run noise (timestamps, ordering, ids) and
  flags ambiguous ones as questions rather than conclusions.
- **Worktree isolation** keeps your checkout (and unsubmitted work) untouched;
  A and B are pinned to SHAs so labels can't drift mid-run.
- **Cost:** the comparison reads both outputs into the model — keep produce
  outputs modest, or `focus` the comparison.
- **Sequential vs. parallel:** default is one worktree at a time; a runner with
  independent worktrees could wrap the two `run-at-ref` calls in a `parallel:`
  block.

- **Live progress:** the run maintains `ab-run-status.md` — a five-stage checklist
  (`○` not started · `⏳` running · `✓` done · `✗` failed) rewritten as each stage
  starts and settles, so you can see exactly where a long run is. This is the
  skill-only stand-in for what the Reactor's receipts/world-model give for free.

## Status

**Strawman for discussion** (Aug 2026) — a genericized version of a workflow used
in production to validate PRs (semantic before/after of long, multi-step jobs).
Proposed as a public OpenProse example: a skill-only, dev-workflow use case that
broadens the example library beyond the reactor/world-model set. A natural
dogfood is A/B-testing OpenProse itself — run an example before vs. after a change
and describe the behavioral delta.
