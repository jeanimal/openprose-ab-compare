# A/B comparison

- **subject** (`produce-command`): `reactor --offline compile --check`
- **A** = base `5e30c1a` (original contracts) · **B** = head `7a0ce3b` (rewritten contracts)

**Verdict:** both refs compile offline to a `STALE` result at zero cost; the
compiled **contract set changed** and the configured **compile model changed**.
The check itself didn't regress — the difference is in *what* would be compiled,
exactly as expected for a contract-schema change.

## Same
- Both report `status STALE` — a fresh worktree has no `.reactor` cache to reuse.
- Same SDK: `@openprose/reactor@0.3.1`.
- Both cost `fresh=0 reused=0` (offline; no model calls).
- Same project shape: the 7-contract OpenProse set (a gateway + responsibilities).

## Different
- **Contract-set fingerprint:** `sha256:0059afb7…` → `sha256:e1e59892…`. The
  `### Maintains` schemas were rewritten between these refs (the `state/<node>.json`
  structured-backing rework), so the compiled contract set is materially different
  — a real, intended change.
- **Configured compile model:** `claude-haiku-4-5` → `claude-sonnet-4-6` (a
  `reactor.yml` change; haiku had been mis-generating canonicalizers).

## Noise (flagged, not a real difference)
- `state-dir` differs: `/tmp/ab-base/reactor-project/.reactor` vs
  `/tmp/ab-head/reactor-project/.reactor`. That's just an artifact of running each
  side in its own worktree — **not** a behavioral change, so it's excluded from
  the verdict.
