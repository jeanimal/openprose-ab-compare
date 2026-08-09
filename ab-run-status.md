# ab-compare — run status

- **subject** (`produce-command`): `reactor --offline compile --check`
- **A** = base `5e30c1a` (original contracts)
- **B** = head `HEAD` (rewritten contracts)

1. ✓ produce A (base)  → contract-set `sha256:0059afb7…`, model `claude-haiku-4-5`
2. ✓ produce B (head)  → contract-set `sha256:e1e59892…`, model `claude-sonnet-4-6`
3. ✓ compare A vs B    → `ab-comparison.md` (fingerprint + model changed; state-dir = noise)
4. ✓ reviewer gate     → no PR passed, so nothing to publish
5. — publish to PR     _(skipped — no PR in this demo)_

**done** · report: `ab-comparison.md`
