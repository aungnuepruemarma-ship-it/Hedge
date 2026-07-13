# Engineering Assurance Review — Session 06: Independent Review Board

Date: 2026-07-13
Charge: treat Sessions 01–05 reports as evidence; reconstruct the complete posture; challenge assumptions; find contradictions, unsupported conclusions, and missing evidence.
New evidence gathered this session: **full git history** (`git fetch --unshallow`), full read of `docs/06-arb-review.md` (the repository's own prior review-board report), headers of `docs/05-incident-review.md`, greps of `docs/04-evolution.md`/`09-foresight.md`, and code verification of three claims taken from the internal ARB before relying on them.

## 1. Provenance — the single most important new fact

**Confirmed**: the entire `jog` repository is **10 commits over two days (2026-07-12/13), all authored by "Claude"**, structured as a scripted exercise ("Turn 2" … "Turn 9"). Consequences:

- The "incidents" cited throughout the code ("incident Day 6/13/14") are **not operational history**. `05-incident-review.md` states this itself (assumption T5-A0: Days 1–5 are analyzed against a *hypothetical* hosted deployment). Sessions 03–04 cited "incident-driven design" as a maturity signal; that signal must be downgraded — the incidents were authored, not experienced.
- Session 02's inference "engineering maturity is high for a Sprint-1 codebase," supported partly by the docs/ADR volume, is weakened: the docs were generated in the same authoring process as the code and are **not independent corroboration**. The code-level evidence (tests pass, invariants hold) stands; the process-level evidence does not.
- The review's framing premise — "a large production platform" — is **contradicted by evidence**. The repository's own governance chain concludes: ARB verdict "Not Production Ready" (Turn 6), governance verdict "delay Version 2.0" (Turn 7), change board "limited preview allowed" (Turn 8). This is a two-day-old pre-production prototype with unusually complete paperwork.

## 2. Prior-work failure in Sessions 02–05 (self-finding)

**Confirmed**: `docs/06-arb-review.md` — present in the repo since before this review began — contains a 13-item findings register (repo-F-00…F-12) that **anticipates most of Sessions 02–05's findings**:

| This review found | Already in repo ARB | Delta |
|---|---|---|
| F-05 CLI approval process boundary (S05, "newly registered") | repo-F-01, rated **Critical**, with repro | We under-rated it (Medium) and presented it as a discovery |
| OP-04 heartbeat dead code (S03) | repo-F-03 | Same finding |
| OP-01 metrics unreachable (S03) | repo ARB area review: "metrics unreachable via API" | Same |
| F-01 grant scoping (S02/S04) | repo-F-11 (args-scope) — related but narrower; the jail-ordering bypass appears **novel to this review** | Partial overlap |
| S05 vector index empty after restart | repo-F-05 | Same finding |
| S02 F-02 unauthenticated approve/bind | repo-F-02 (adds: the non-loopback guard was *prescribed by the team's own incident review and never implemented*) | Repo version is stronger |

Verdict on our own process: Sessions 02–05 **did not read** `docs/05–09` despite listing them in Session 02's file inventory, and therefore re-derived known findings and mislabeled at least one as new. Independent re-derivation has confirmation value, but the reports' claims of novelty were unsupported. Additionally, our F-numbering collides with the repo's own register — future sessions must namespace (HG-F-xx vs repo-F-xx).

## 3. Findings the internal ARB has that we missed (verified this session before adoption)

- **repo-F-04 (verified)**: `ledger.pending_intents()` — the "unknown outcome, must re-verify on resume" half of the R3 crash-recovery story — is consumed **only by the verifier**, never by the runtime (grep: single call site, `state/verify.py:92`). Session 03 praised "resume-without-duplicate-work"; that praise was **half-unsupported**: skip-completed exists, re-verify-unknown does not.
- **repo-F-06 (verified)**: `obs.trace()` has **zero production call sites** — every log line and event ships `trace_id: "-"`. This was visible in our own Session 03 probe output and we did not flag it. ADR-011's trace propagation is built but never bound.
- **repo-F-07**: no execution owner — the only drivers of long-running work are a synchronous CLI command and a synchronous HTTP handler. Consistent with our OP-05 but more fundamental.
- **repo-F-10 (verified)**: `scheduler.concurrency` validates as ≥0 — `0` silently schedules nothing; values >1 are accepted although the runtime is not thread-safe.

## 4. Contradictions and unsupported conclusions in Sessions 01–05

1. **S04 "defense-in-depth held" for F-01 is overstated.** The containment evidence covers only the builtin **fs** handlers' re-resolve. `shell.exec` after approval has *no jail at all* (only `cwd=workspace`; an approved command can touch any path the OS user can). The correct statement: F-01's gate-level bypass is contained for fs.read/fs.write/repo.inspect and **uncontained for every other capability**, which is precisely repo-F-11.
2. **S03 "recovery machinery … ahead of typical Sprint-1 maturity"** — partially unsupported per repo-F-04 (resume re-verification missing) and §1 (the "incident-driven" pedigree is authored).
3. **S02 ledger drift (93 vs 106)** — now *explained* by history: the Sprint-1 commit shipped 93 tests; Turn 5 hardening added 13; `ledger.json`'s suite field was not regenerated. Drift confirmed, mechanism evidenced, severity unchanged (Low).
4. **S05 "no AI today"** — independently corroborated by repo-F-00 (a prior submission claimed "Sprint 2 complete"; the repo ARB refuted it from repository state). Conclusion stands, now double-sourced.
5. **S01–S05 relationship assumption** — we assumed Hedge is "the assurance ledger" for Jog. Still **Unknown**: no artifact in either repo states their relationship. This remains an open evidence request, now three sessions old.

## 5. Reconstructed engineering posture (evidence-based, consolidated)

- **What it is**: a two-day-old, single-author (AI-authored), stdlib-only Python prototype of a local-first agent runtime, ~2.6k LOC, 106 passing offline tests, with an exceptional written governance trail that is *part of the artifact itself*, not external oversight.
- **What genuinely holds** (multiply verified, code-level): single-door effect routing with pre-effect durable audit events; workspace jail for fs builtins (two layers); tamper-refusing checkpoints with valid-fallback restore; DLQ with tested replay; CAS/epoch state discipline; fail-fast config; honest stubs.
- **What is claimed-but-unwired** (the dominant posture pattern): heartbeats, resume re-verification, trace propagation, retention, metrics export, plugin limits, immutable history, non-loopback guard, durable approvals. The repo's own ARB named the disease: *"paper features vs wired features [are] not distinguished anywhere machine-readable."*
- **The load-bearing risk cluster** (unchanged from S05, now upgraded in confidence): ephemeral in-memory authority (session grants) — repo rates the CLI manifestation **Critical**; it breaks the product's primary surface.
- **Production readiness**: Not production ready — by this review's evidence *and* the repository's own three-stage governance verdict. Any "production platform" characterization is unsupported.

## 6. Missing evidence (consolidated register)

| # | Missing | Blocks |
|---|---|---|
| M-1 | Hedge↔Jog relationship statement | Scope validity of the entire review |
| M-2 | Any human authorship/review evidence (all commits are "Claude") | Governance: no human accountability point exists in the record |
| M-3 | CI execution evidence (no CI config; tests verified only by this review's local runs) | Delivery assurance |
| M-4 | Concurrency >1 behavior (zero tests) | repo-F-10 / OP-05 severity |
| M-5 | Kill-window exactly-once test (deferred in repo's own list) | R3 crash-recovery claim |
| M-6 | License/ownership (no LICENSE file) | Legal/governance |

## 7. Recommended further investigation (priority order)

1. **Resolve M-1/M-2 with the owner** — one paragraph from a human stating what Hedge is, what Jog is, and who is accountable, committed to either repo.
2. **Adopt the repo's findings register as the canonical one** and merge ours into it (namespaced), eliminating the numbering collision; track wired-vs-paper status per feature machine-readably (extends `ledger.json`).
3. **Kill-window test** (M-5): kill -9 between ledger intent and completion, restart, prove exactly-once — this single test would convert the largest unsupported reliability claim into evidence.
4. **Concurrency clamp**: reject `concurrency != 1` in config until thread-safety exists (one-line fix; removes M-4's urgency).
5. Re-verify the F-01 jail-ordering bypass against the repo's F-11 fix when the durable-approvals change lands — they must be fixed together or the fix is incomplete.

## 8. Conclusions (evidence-based only)

- The strongest prior conclusions survive the challenge: audit spine quality, honest-stub discipline, the ephemeral-authority root cause, and "no AI exists yet."
- The weakest prior conclusions are the *maturity* inferences that leaned on the authored documentation trail; these are withdrawn and replaced by: **the artifact demonstrates disciplined design, unverified process**.
- This review board's own predecessor sessions failed to consume available evidence (docs 05–09) before reporting; that is now corrected, and the internal ARB's register is confirmed as substantially accurate against code.
