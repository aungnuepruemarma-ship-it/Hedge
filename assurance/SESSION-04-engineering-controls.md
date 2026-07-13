# Engineering Assurance Review — Session 04: Engineering Controls Review

Date: 2026-07-13
Subject: `jog` / Hermes Core v0.1.1
Method: code read of all modules (Sessions 02–03), full read of tests/test_hardening.py, codebase-wide grep for crypto/auth/secret/TLS surface, and runtime verification of the F-01 gate-ordering finding. Only observable controls are evaluated; absence of a control is reported as absence, not speculated risk.

## Control-by-control assessment

### Identity
- **Observed**: Actors are free-form strings (`task:<id>`, `plugin:<name>`, `cli:explain`, `api`) attached to events, decisions, and ledger rows. This is *attribution labeling*, not identity: nothing verifies that a caller is who the string says.
- **Absent**: any principal model, user accounts, or caller verification. (Consistent with the single-user local trust model asserted in docs.)

### Authentication
- **Absent entirely — Confirmed**: grep across `hermes/` finds no token, password, key, or auth code of any kind. The HTTP API accepts any request on 127.0.0.1 (Session 02 F-02). The CLI is trusted by virtue of process ownership.

### Authorization
- **Observed**: the PermissionGate (ADR-012) is a real, tested control: capability taxonomy (10 capabilities), workspace jail for fs.* (symlink-aware via `Path.resolve()` + `is_relative_to`), default-ask for dangerous capabilities resolving to deny in headless mode, unknown capabilities denied. Router enforces gate-before-effect with the decision durably logged pre-execution. Plugin capability requests route through the same gate at registration.
- **Verified defect (F-01, upgraded from code-read to runtime-confirmed)**: session grants short-circuit *before* the workspace jail — `grant_session("task:x","fs.write")` then `decide(...,"/etc/passwd")` → **allow (rule: session-grant)**. Mitigating control also verified: builtin fs handlers re-resolve defensively and raised `PermissionError: path escapes workspace` on the actual write. Net: defense-in-depth held for builtin tools; **any future/plugin tool relying on the gate alone inherits a jail bypass after approval**. `test_hardening.py` does not cover this case.
- **Related observation**: `approve(task)` grants `(actor, capability)` for the tool in the task's intent; the grant is broader than the approved intent's arguments, but exposure is bounded because the task executes only its stored intent.

### Key lifecycle / Secret lifecycle / Certificate lifecycle
- **Absent entirely — Confirmed**: no keys, no secrets, no certificates exist anywhere in the codebase. Nothing to rotate, nothing to leak from the platform itself. `hashlib.sha256` is the only cryptographic primitive in use (integrity, not confidentiality). ADR-015 (encrypted memory) is a decision record for future work; no implementation observed.
- Consequence worth recording: there is also **no secret-redaction control** — if a future tool arg or note contains a secret, it will be persisted verbatim in the event log, effect ledger, and checkpoints, which are plaintext and immutable-by-convention.

### Transport security
- **Absent by scope — Confirmed**: API is plain HTTP, default-bound to 127.0.0.1 (`make_server` default; `serve --host` permits non-loopback binding with no compensating control). No TLS anywhere. The only transport is loopback.

### Storage protection
- **Observed**: SQLite with WAL, single-writer lock, parameterized SQL throughout (no injection surface found), corruption propagates rather than being absorbed. Checkpoints: atomic tmp→fsync→rename, sha256 recorded out-of-band in the DB, restore refuses hash/schema mismatch (tamper refusal tested in test_hardening.py; verified passing).
- **Absent**: encryption at rest, file permissions hardening (`.hermes/` created with default umask), and directory-fsync after checkpoint rename (Session 03 OP-08).

### Configuration management
- **Observed — strong**: single merged source (defaults→profile→TOML→env), fail-fast with *all* problems listed, unknown keys/sections rejected (typo detection), type coercion validated, config read-only after load (`__setattr__` raises), effective config inspectable (`/v1/config`, `hermes config`). Hostile-env behavior tested.
- **Defects**: two validated-but-unconsumed knobs (`observability.retention_mb`, `scheduler.heartbeat_s` — Session 03); config loader consumes *every* `HERMES_*` env var and hard-fails on strays (operationally brittle: an unrelated `HERMES_FOO` in the environment prevents startup — observed live in Session /run-skill work).

### Dependency governance
- **Observed — maximal by construction**: zero runtime dependencies (`dependencies = []`), stdlib only, offline test suite. Supply-chain surface is the Python interpreter and setuptools at build time. No SBOM, no lockfile — with nothing to lock, this is currently moot but becomes a gap the moment the first dependency lands. No license file governs the project itself (Session 02 F-04).

### Auditability
- **Observed — strong**: append-only event log written *before* dispatch; permission decisions, tool intents, and outcomes are all durable events with actor + trace_id + causation_id; effect ledger gives WAL-style intent/completion around every side effect; `hermes policy` provides decision explain (dry-run); DLQ preserves failed deliveries with error and attempt count.
- **Gaps**: append-only is convention, not mechanism — any process with file access can rewrite `hermes.db` (no hash chaining despite ADR-018 "immutable history" being on the books as a decision); event log is unbounded plaintext (retention gap OP-02); no clock integrity (all `time.time()`).

### Integrity
- **Observed**: checkpoint sha256 + schema versioning with loud refusal; `hermes verify` / `state/verify.py` detects merge damage (invalid states, running-without-lease, bogus wait reasons, missing deps, dependency cycles, terminal-with-lease) — all confirmed by the hardening suite; state machine is a single transition authority; CAS versioning on task records; epoch fencing rejects zombie writes.
- **Gaps**: no integrity protection on the event log or effect ledger themselves (only tasks and checkpoints are verified); verify is on-demand, not scheduled.

### Availability / Resilience
- **Observed**: bounded retries → ABANDONED terminal state; lease-expiry reclaim; DLQ with tested replay-and-recover flow (health degrades on non-empty DLQ, recovers after replay — test-verified); checkpoint fallback walks to newest *valid* snapshot; graceful degradation documented in 7 runbooks.
- **Gaps (Session 03, unchanged)**: no backoff, heartbeat dead code → long-effect double-execution window, unbounded resource growth, no request limits on the API, threaded API violates single-writer assumptions.

### Policy enforcement
- **Observed**: single-door enforcement — the router is the only path from intent to effect, and the gate is consulted on every invocation with the decision persisted before execution; plugins cannot execute at all yet (launch stubbed), so plugin policy bypass is currently impossible by construction. Offline enforcement (`offline.enforce`) blocks network-flagged tools ahead of the gate.
- **Gaps**: F-01 ordering defect (above); enforcement is in-process — any code sharing the process (e.g., a future in-process plugin) can call effects directly, bypassing the router; the non-bypass property therefore currently rests on the Sprint-2 subprocess sandbox actually landing.

## Summary judgment

| Domain | State |
|---|---|
| Authorization, auditability, integrity (task/checkpoint), config mgmt | **Present and genuinely tested** |
| Identity, authn, keys/secrets/certs, TLS, encryption at rest | **Absent by declared trust model** (single-user, local, offline) |
| Policy enforcement | Present with one verified ordering defect (F-01) and a process-boundary caveat |
| Dependency governance | Trivially strong now; no machinery for when it stops being trivial |

Supported Inference: the platform's controls are coherent *for its declared threat model* (local, single-user, offline). Every "absent" control above becomes a real gap at the first step outside that model — `serve --host 0.0.0.0`, the first plugin that executes, the first network tool, or the first secret that transits a tool arg. The codebase currently contains no guardrail that detects or resists that boundary being crossed (e.g., no warning on non-loopback bind).

## Actions recommended (priority order)
1. Fix F-01: evaluate workspace jail before session grants, or target-scope grants; add the missing hardening test.
2. Warn-or-refuse on `serve --host` ≠ loopback without an explicit `--i-know` flag (cheap boundary tripwire).
3. Decide ADR-018's fate: implement event-log hash chaining or downgrade the "immutable history" claim in docs.
4. Add secret-redaction hooks to event/ledger serialization before any network/LLM tools land (Sprint 2 items make this imminent).

## Unknown / carried forward
- Commit history still shallow (contributor base unexamined).
- Sprint-2 cross-check of docs/04-evolution.md + 09-foresight.md against OP-01/OP-02 (carried from Session 03).
- CLI approval process-boundary defect discovered during run-skill work (approve→run across CLI processes re-parks WAITING) — candidate finding F-05, to be formally registered next session.
