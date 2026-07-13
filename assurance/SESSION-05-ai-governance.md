# Engineering Assurance Review — Session 05: AI Governance Report

Date: 2026-07-13
Subject: `jog` / Hermes Core v0.1.1
Method: code evidence from Sessions 02–04 (all modules read, three runtime verifications), plus this session's full read of `hermes/state/memory.py` and ADR-005 (deterministic planning), ADR-007 (plugin isolation), ADR-010 (model routing). Evidence-only.

## Headline finding

**The platform contains no AI today.** Grep- and read-confirmed: there are no model calls, no provider adapters, no planner, and no prompt construction anywhere in the code. The "AI" layer exists as ADR commitments (005, 010) with explicit verification plans but zero implementation (ledger S1-16 "deferred"). What is reviewable now is the *governance chassis* the future AI will be bolted onto. This report therefore evaluates (a) the chassis as built, and (b) whether the ADR-committed AI controls are credible.

## Domain-by-domain

### Context isolation
- **Confirmed**: Tasks share one process, one workspace, one memory facade. Isolation between tasks is *state-machine* isolation (CAS-versioned records, per-task events/effects), not context isolation — any task's tool output lands in shared stores that any future task's `recall()` can read. WorkingMemory is a single global KV, not per-task.
- **ADR commitment**: ADR-005 pins planning context to content-hashed snapshots + memory watermarks — a real isolation design, unimplemented.

### Permission boundaries
- **Confirmed** (Session 04): capability taxonomy + workspace jail enforced at the gate, with the F-01 defect (session grants bypass the jail at gate level; builtin handlers' defensive re-resolve currently contains it) and the in-process caveat (boundaries bind tools routed through the router; nothing binds code sharing the process).

### Tool approval workflow
- **Confirmed working**: dangerous capabilities (shell.exec, net.fetch, plugin.spawn, browser.use, git.write, api.call) resolve to ask→deny headless; task parks WAITING(approval); `approve` grants the session and requeues; verified end-to-end via the API.
- **F-05 (Medium, newly registered; runtime-verified during run-skill work)**: the approval grant is in-memory per process, so via the CLI (`tasks approve` then `tasks run` = two processes) the approved task silently re-parks WAITING(approval). **The human-approval workflow is functionally broken on the CLI surface** — approval works only where one process spans approve+run (API server / direct invocation). No error, no log; it just re-asks. This also means approvals are not durable records: an approval is not written anywhere except the requeue event, so post-restart the approval evaporates.

### Memory lifecycle
- **Confirmed**: six-store facade (`state/memory.py`). Writes are explicit with provenance tags (`semantic.put(provenance=...)`); episodic memory is a rebuildable read model over the event log with a watermark; working memory is checkpointed and restored via recovery hook; retrieval is priority-ordered (working > semantic > episodic) with a budget.
- **Absent — Confirmed**: no retention, expiry, deletion API, or redaction on any store; notes and episodes are append-forever plaintext. No provenance *verification* (any caller may claim any provenance string). VectorIndex is in-memory only and silently empty after restart (semantic notes persist but their index does not — `remember()` indexes at write time; nothing re-indexes on boot). Embeddings backend honestly raises rather than degrading.

### Delegation controls
- **Confirmed**: there is no delegation. One serial worker, concurrency=1; tasks cannot spawn tasks (no tool exposes `register_task`); deps are declared at creation by the human surface. The worker protocol (claim/epoch/lease) is distributed-ready but inert. Delegation governance is therefore trivially satisfied today and entirely untested for tomorrow.

### Plugin governance
- **Confirmed**: manifest validation (required fields, host-API version negotiation, unknown-capability rejection), capability grants resolved through the gate at registration and recorded on the plugin record, refusals evented. Execution is impossible (`launch()` raises NotImplementedError — test-asserted), so plugin risk is currently zero by construction.
- **ADR-007 credibility check**: the sandbox design (subprocess, rlimits, scrubbed env, fs jail, quarantine on crash-loop) is specific and testable, and the ADR honestly states OS-primitive isolation is "containment, not a security boundary." Manifest `limits` (cpu_s, mem_mb) are parsed today but enforced by nothing.

### Execution governance
- **Confirmed**: single-door execution (router-only path to side effects), gate→durable decision event→ledger intent→execute→ledger completion→outcome event, verified ordering by tests; bounded retries; cooperative cancellation; offline enforcement blocks network tools ahead of the gate; shell.exec requires argv lists (no shell interpolation) and honors a config timeout.
- **Gaps** (Sessions 03–04): no resource limits beyond timeouts; heartbeat/lease gap allows double-execution of long effects; execution assumes but does not enforce single-threaded entry.

### Human approval
- See Tool approval workflow: present, evented, but (F-05) not durable and broken across CLI process boundaries; additionally `POST /v1/tasks/{id}/approve` is unauthenticated (F-02) — the human in "human approval" is any local process. No approval metadata (who, why, expiry) is captured beyond the requeue event.

### Audit logging
- **Confirmed — strongest domain**: every permission decision, tool intent, and outcome is a durable event appended *before* dispatch, carrying actor, trace_id, causation_id, task_id; effect ledger provides pre/post records around every side effect; DLQ preserves failed handler deliveries; `hermes policy` explains decisions dry-run.
- **Gaps**: log is tamperable by convention (no hash chain; ADR-018 unimplemented); approvals under-recorded (above); no retention policy (OP-02).

### Policy enforcement
- **Confirmed** (Session 04): enforced at the single router door, decisions pre-persisted; F-01 ordering defect; enforcement depth is in-process only until the ADR-007 sandbox exists. Policy itself is code, not data — the "default" bundle is hardcoded; named bundles are an extension point, so policy *change management* (who edits policy, review, versioning) has no mechanism yet.

### State persistence
- **Confirmed**: SQLite (WAL, single-writer, parameterized), CAS-versioned task docs, atomic hash-verified checkpoints with fallback restore, event-sourced read models rebuildable via `replay()`. Working memory survives restarts only via checkpoints; vector index does not survive at all; session grants do not survive (F-05 root cause).

### Risk containment
- **Confirmed containment that exists**: workspace jail (two layers), offline enforce flag, ask-gating of all outward-reaching capabilities, plugin execution disabled, bounded retries, tamper-refusing checkpoints, DLQ.
- **Containment that is claimed but absent**: plugin resource ceilings (parsed, unenforced), immutable history (ADR only), memory encryption (ADR-015, absent), non-loopback bind tripwire (absent), secret redaction (absent). Blast radius today = whatever the OS user can do inside (and — via F-01-class paths after approval — outside) the workspace.

## Governance maturity summary

| Domain | Maturity |
|---|---|
| Audit logging, execution governance, state persistence | **Built & tested** |
| Permission boundaries, tool approval, policy enforcement | Built; one verified defect each (F-01, F-05, in-process depth) |
| Memory lifecycle | Half-built: writes/provenance yes; retention/redaction/index-durability no |
| Plugin governance | Front door built; execution & limits deferred (safe: disabled) |
| Context isolation, delegation controls | Not built; trivially safe at concurrency=1, untested beyond |
| AI-specific governance (planning capture, model routing) | **Paper only** (ADR-005/010) — credible, test-specified, unimplemented |

## Supported Inference
- The project practices *governance-before-capability*: every dangerous capability is either gated, stubbed loud, or disabled, and the ADRs specify verification evidence before the features exist. This is the right ordering, and materially better than the industry norm of capability-first.
- The three defects that matter all share one root cause: **ephemeral in-memory authority state** (session grants) layered over durable everything-else. F-01 (grant scope), F-05 (grant lifetime), and the unauthenticated approve endpoint (F-02) would all be addressed by making approvals durable, target-scoped, attributed records in storage — one design change.

## Unknown
- Whether Sprint 2 will implement ADR-005 capture before or after the first live model call (ordering risk R11n acknowledged in the ADR itself).
- Behavior at concurrency >1 (explicitly deferred; no evidence).
- Git history/contributor base (still shallow-cloned; carried forward).

## Recommendations (AI-governance-specific, priority order)
1. Make approvals durable and scoped: persist (actor, capability, target-class, approver, expiry) in storage; fixes F-05, closes F-01's grant path, and gives F-02 something worth authenticating.
2. Land ADR-005's capture/replay *before* the first provider adapter — retrofit is the stated risk.
3. Add memory governance before models read memory: retention knobs, a delete/redact API, and boot-time vector re-index (silent empty index will otherwise degrade recall invisibly).
4. Enforce manifest limits at the same milestone as `launch()` — never ship execution ahead of ceilings.
5. Introduce policy-as-data (named bundles in config) so policy changes become reviewable, versioned artifacts.
