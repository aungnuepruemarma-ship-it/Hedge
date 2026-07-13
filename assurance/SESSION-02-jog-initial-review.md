# Engineering Assurance Review — Session 02: Initial Review of `jog` (Hermes Core)

Date: 2026-07-13
Subject: https://github.com/aungnuepruemarma-ship-it/Jog (cloned shallow at /workspace/jog)
Scope: Python package `hermes` v0.1.1 (Sprint 1 foundation), docs, tests.

## Confirmed Evidence

1. **Codebase**: ~2,628 lines of Python across `hermes/` (config, obs, events, state, core, policy, tools, plugins, surfaces). Zero third-party dependencies (`pyproject.toml` dependencies = []); requires Python >= 3.11.
2. **Tests**: `python3 -m unittest discover -s tests` → **106 tests, OK, ~6.8s, fully offline** on Python 3.11.15. Suites cover config, events/DLQ, state machine, scheduler CAS/lease, memory, router, gate (incl. workspace-jail traversal), plugins, checkpoints (tamper refusal), live-HTTP API, CLI, and a hardening suite.
3. **Governance artifacts**: 16 ADRs (004–019), 7 runbooks, sprint log, change-control/governance/incident-review docs, machine-readable `docs/ledger.json` with per-task evidence pointers.
4. **Honest stubs**: plugin sandbox `launch()` raises NotImplementedError by design; `research.fetch` labeled stub; stubs inventoried in ledger (S1-17, S1-18) and asserted by tests.
5. **Storage** (`state/storage.py`): parameterized SQL throughout, single-writer RLock, WAL for file DBs, rollback-on-exception transaction context; checkpoint rows carry sha256.
6. **API** (`surfaces/api.py`): binds 127.0.0.1 by default; JSON-only; no authentication of any kind.
7. **Ledger drift**: `ledger.json` records 93 tests; actual suite is 106 (ledger updated 2026-07-12 — stale by at least one change set).
8. **Missing**: no CI configuration (`.github/`, etc.), no LICENSE file, no lint/type-check configuration, no SECURITY.md. Shallow clone; commit history not yet examined.

## Findings

- **F-01 (Medium, security architecture — Confirmed by code read, not runtime-verified)**: In `PermissionGate.decide` (policy/__init__.py:56), session grants are checked *before* the workspace-jail rule and are keyed only on (actor, capability). A session grant of `fs.write` therefore permits writes to targets **outside the workspace**, silently overriding the jail (rule "session-grant" wins over "outside-workspace"). ADR-012's non-bypass intent is undermined for granted actors. Recommend: evaluate target scoping even for granted capabilities, or scope grants to (actor, capability, target-class).
- **F-02 (Medium, security architecture — Supported Inference)**: The local API exposes `POST /v1/tasks/{id}/approve` with no authentication. Any local process/user able to reach 127.0.0.1 can resolve WAITING(approval) tasks, converting the human-approval control into a same-host TOCTOU-free bypass. Acceptable only under a strict single-user-host trust model; that model is asserted in docs but not enforced (no socket peer check, no token).
- **F-03 (Low, governance)**: ledger.json test count (93) vs actual (106) — the machine-readable progress ledger is not regenerated automatically; drift undermines its authority as an assurance artifact.
- **F-04 (Low, delivery assurance)**: No CI pipeline in the repository; the green suite is only verifiable by local execution. No license or security policy files.

## Supported Inference

- Engineering maturity is high for a Sprint-1 codebase: layering is enforced by tests, error paths (DLQ, tamper refusal, hostile env) are tested, and stubs are explicit rather than hidden. The docs/ADR trail indicates deliberate change control.
- The stdlib-only constraint eliminates supply-chain exposure at the cost of re-implementing infrastructure (event bus, HTTP) that will carry its own defect risk as scope grows.

## Unknown (evidence requested for next session)

1. Commit history and contributor base (`git fetch --unshallow` or GitHub history review).
2. Whether the hardening suite covers F-01's grant-ordering case (test_hardening.py not yet read in full).
3. Deployment/operational context: is the API ever bound to non-loopback? Config allows host override?
4. Relationship between the `Hedge` repo (empty; holds this review) and `jog` — is Hedge intended only as the assurance ledger?

## Continuity

Next session: read `test_hardening.py`, `config.py` (host binding), `core/runtime.py` + `tools/` router ordering guarantees; unshallow history; verify F-01 with a targeted runtime probe.
