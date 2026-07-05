# 2026-07-05 - PolicyEvaluator Context Snapshot Deep-Copy Stragglers

PR: #4 (fix/ci-pyo3-resolve-config)

Follow-up to #3023.

## What changed and why

#3023 established that `audit_entry["context_snapshot"]` must be a
`copy.deepcopy` of the evaluation context, so that later mutation of the
caller's context dict cannot silently rewrite what the audit trail says was
evaluated. That change covered eight snapshot sites but missed two:

- `evaluator.py` flat-path YAML-match branch (was line 253): stored the live
  `context` reference.
- `evaluator.py` folder-scoped YAML-match branch (was line 386): same.

Both now use `copy.deepcopy(context)`, identical to the other eight sites.

This audit doc also covers the two build/tooling fixes shipped in the same PR
(pyo3-build-config `resolve-config` feature + exact 0.29.0 pin, and the
dep-confusion/spell allowlisting of `tzdata`), which are CI-infrastructure
changes with no runtime behavior.

## Threat model impact

Audit-trail integrity fix; no new attack surface.

- Before: a caller (or any code holding the same context dict) could mutate
  the dict after a YAML-match decision and retroactively change the recorded
  `context_snapshot` — an audit-tamper primitive on exactly the two branches
  that skipped the copy. An attacker able to influence post-decision context
  mutation could make the audit log disagree with what was actually evaluated.
- After: all ten snapshot sites store an isolated deep copy; the audit entry
  is immutable with respect to caller-side mutation.
- No changes to decision logic, defaults, backends, identity, trust, or
  cryptographic code. The deny/allow outcome for every input is unchanged.

Dependency note: `pyo3-build-config` is a build-time-only dependency of the
vendored native SDK (compile-time config resolution; not linked into the
produced artifact). Pinned to the exact published version 0.29.0
(2026-06-11 on crates.io, past the 7-day cooling-off window), aligned with
the pyo3 0.29 bump from #3002.

## Test coverage for security-relevant behavior

- `tests/test_folder_governance.py::TestContextSnapshotIsolation` — the
  pre-existing isolation suite from #3023 covers both fixed branches:
  `test_flat_yaml_match_isolated_from_top_level_mutation` (failing before this
  change, passing after) and the folder-scoped/nested siblings. Full file:
  31 passed locally; docker-compose-test exercises the same suite in CI.
- No new tests were needed: the failing test that motivated this fix already
  encodes the guarantee; the fix makes it pass.
