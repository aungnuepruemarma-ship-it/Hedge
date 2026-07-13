# Engineering Assurance Review — Session 03: Operational Assurance Report (Runtime Behavior)

Date: 2026-07-13
Subject: `jog` / Hermes Core v0.1.1 @ /workspace/jog
Method: full read of obs.py, events.py, core/scheduler.py, core/runtime.py, state/checkpoints.py, state/ledger.py, state/storage.py, config.py, tools/, surfaces/context.py+api.py, PLUS a live behavioral probe (real HTTP server, failing task, approval-gated task, shell timeout injection).

## 1. Confirmed Evidence — what works as claimed

- **Structured logging**: JSON lines to stderr with ts/level/logger/msg/trace_id; optional file handler; exception capture. Trace IDs propagate via contextvars (`obs.trace()`).
- **Event spine**: every event durably appended to SQLite *before* dispatch; bounded redelivery (default 3) then dead-letter with error + attempts; DLQ replay door exists (`retry_dead_letter`) and is surfaced in the CLI (`deadletters`).
- **Health checks are real probes** (`context.health()`): storage event count, episodic watermark, checkpoint count, DLQ (non-empty DLQ **degrades** health — good incident-driven design), metrics counter count. Probes can't crash the process.
- **Error handling discipline**: tool exceptions → `status=error` results, never crashes (verified live: injected `TimeoutExpired` became a clean error result, ledger completion `ok:false` written, no pending intents leaked).
- **Timeouts**: `shell.exec` honors `execution.shell_timeout_s` (verified live via env override, killed at 3.0s); `test.run` has a hardcoded 300s timeout.
- **Retry logic**: bounded (`max_retries`, default 2); failed task observed live going FAILED→READY→FAILED→ABANDONED (attempts=3) in 5 ticks; attempts counted once per failure; ABANDONED is a proper terminal state.
- **Crash recovery**: lease-expiry reclaim (RUNNING→READY, attempts+1, epoch fencing against zombies); effect ledger (INTENT-before-effect, COMPLETION-after) gives resume-without-duplicate-work; checkpoint write path is serialize→tmp→fsync→rename with sha256, restore refuses hash/schema mismatch, `restore_latest_valid()` degrades loudly to the previous valid snapshot.
- **Backpressure primitive**: `run_until_idle(max_ticks)` bounds each scheduling burst; API caps it via `max_ticks`.

## 2. Findings — operational weaknesses

- **OP-01 (High) Metrics are unexportable and volatile.** No `/v1/metrics` endpoint exists (verified live: 404). Health exposes only the *number* of counters, not their values. Counters are in-process only and lost on restart. There is no way for any external monitor to read `tools.error`, `events.dead_lettered`, etc. → the entire metrics system is currently write-only.
- **OP-02 (High) No retention or compaction anywhere.** `observability.retention_mb` is validated by config but consumed by no code (grep-confirmed dead knob). The SQLite event log, episodes, effects, and checkpoint files all grow without bound; no pruning, no VACUUM, no disk-space health probe. Long-running deployment ⇒ unbounded disk growth with no early warning.
- **OP-03 (Medium) Retries have no backoff.** Failed tasks are re-queued on the next tick (verified: 3 attempts inside one synchronous `/v1/run`). Deterministic failures burn all retries in milliseconds; transient failures (e.g., future network tools) get no time to clear. No jitter, no retry-delay config.
- **OP-04 (Medium) Heartbeat is dead code; long effects can outlive their lease.** `scheduler.heartbeat_s` is validated but never consumed; `SerialWorker.heartbeat()` is never called by the runtime. A tool execution longer than `lease_ttl_s` (120s — and `shell_timeout_s` is user-configurable above that) leaves a RUNNING task with an expired lease; the next tick reclaims and re-executes. The effect ledger only shields *completed successful* effects, so an in-flight long effect can be double-executed.
- **OP-05 (Medium) Concurrency assumption vs. threaded API.** The event bus and runtime assume single-threaded emission ("per-task order follows from single-threaded emission"), but the API is a `ThreadingHTTPServer`: two concurrent `POST /v1/run` (or run + approve) execute the tick loop in parallel with no lock. CAS on the task store limits corruption, but event ordering, `_effects_since_checkpoint`, and the serial-worker assumption are all violated. No API request-serialization guard exists.
- **OP-06 (Medium) Health is blind to stuck work.** Verified live: a task parked WAITING(approval) and another ABANDONED, yet `/v1/health` = "ok". No probe for: oldest WAITING age, ABANDONED count, READY-queue depth/age, lease near-expiry, or event-log/episodic watermark *lag alarm* (lag is exposed as a number, never evaluated — the ledger itself defers this to Sprint 2).
- **OP-07 (Low) No API input bounds.** `Content-Length` is trusted and read fully into memory; no request-size cap, no rate limiting, unbounded thread creation per connection — a local misbehaving client can exhaust memory/threads (consistent with F-02 trust-model caveat from Session 02).
- **OP-08 (Low) Checkpoint durability gap + no pruning.** File contents are fsynced but the *directory* is not after rename, so a crash can lose the rename on some filesystems; checkpoint files accumulate forever (auto-checkpoint every 10 effects) with no retention policy.
- **OP-09 (Low) Logging defaults leave no trail.** `logging.file` defaults to "" — production logs go to stderr only; if the process is not run under a supervisor capturing stderr, forensic evidence is lost. `prod` profile changes nothing but `json: true` (already the default).
- **OP-10 (Info) Effect-identity collision.** `effect_id = sha256(task, tool, args)`: a task legitimately needing the same tool+args twice will have the second invocation replay-skipped. Acceptable at Sprint-1 scope; will bite once plans contain repeated idempotent-looking steps (e.g., `test.run` twice).

## 3. Missing telemetry / monitoring gaps (consolidated blind-spot list)

| Signal | Status |
|---|---|
| Counter values externally readable | **Missing** (OP-01) |
| Latency/duration histograms (tool exec, tick, API) | Missing entirely |
| Task-age / queue-depth gauges | Missing (OP-06) |
| Disk usage / DB size / checkpoint-dir size | Missing (OP-02) |
| Watermark-lag alarm | Missing (acknowledged: Sprint 2) |
| WAITING(approval) aging alert | Missing (OP-06) |
| Lease-expiry / zombie-fencing counter | Missing (reclaims emit events but no metric) |
| Metrics persistence across restart | Missing (OP-01) |
| Log retention/rotation | Missing (OP-02, OP-09) |

## 4. Supported Inference

- The observability design is honest (no faked signals) but **consumption-side incomplete**: signals are produced and stored, yet almost nothing can be *watched* from outside the process. Operationally the system is auditable after the fact (event log, ledger) but not monitorable in real time.
- Recovery machinery (leases, ledger, checkpoints, DLQ replay) is ahead of typical Sprint-1 maturity; the dominant operational risk is not crash-recovery but **silent degradation**: disk growth, stuck approvals, and double-execution of long effects would all pass health checks today.

## 5. Unknown / evidence requested

1. Intended deployment mode (supervised service vs. CLI bursts) — determines severity of OP-05/OP-09.
2. Whether Sprint 2 plans (ADR-009/011 follow-ups) already cover OP-01/OP-02 — cross-check `docs/04-evolution.md` and `docs/09-foresight.md` next session.
3. `test_hardening.py` coverage map vs. these findings (still unread in full; carried over from Session 02 along with F-01 runtime verification).

## 6. Priority recommendations

1. Add `GET /v1/metrics` (values, not counts) and persist counters or derive them from the event log on startup (fixes OP-01 cheaply — the event log already holds the truth).
2. Implement retention: size-triggered event/episode pruning behind the existing `observability.retention_mb` knob + checkpoint keep-last-N (OP-02, OP-08).
3. Wire heartbeats into `_execute` for long effects, or clamp `shell_timeout_s < lease_ttl_s` at config validation (OP-04).
4. Serialize runtime entry (a runtime-level mutex) until true concurrency lands (OP-05).
5. Add health probes: oldest-WAITING age, DB file size, checkpoint-dir size, watermark lag threshold (OP-06).
