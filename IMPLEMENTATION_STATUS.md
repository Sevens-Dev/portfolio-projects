# Implementation status

Last updated: 2026-09-19

This file is informational. `HANDOFF.md` remains authoritative.

## Flightcheck (`px4-reqcheck`)

- Current phase: handoff phase Week 5
- Repository URL: https://github.com/Sevens-Dev/px4-reqcheck
- Current branch: `main`
- Latest merged PR or commit: PR #15, `Publish C++ checker timing evidence` (`9ca3ddc`); annotated tag `v0.1`
- Completed exit criteria: Weeks 0-4; deterministic 20-log checksum-pinned corpus; normalized Parquet and complete data-quality reporting; seven corrected requirements with sentinel-safe parameter-derived thresholds and total three-valued evaluation; committed traceability report; byte-for-byte clean Ubuntu 24.04 reproduction; CMake/GoogleTest C++17 checker builds in CI; versioned exchange schemas; C++ independently recomputes the raw-sample landing descent p95; 140/140 cross-language verdict agreement; 20-run checker timing table with retained raw hosted-run evidence
- Tests currently passing: 59 pytest tests; 5 CTest/GoogleTest tests; Ruff format/check; scoped strict mypy; dependency audit; gitleaks; clean real-corpus `make all` reproduction and C++ agreement
- Documented cut-order decisions: model baseline remains assumed cut by default; no never-cut item removed
- Unresolved external blockers:
  - Local execution host is Ubuntu 26.04, not the preferred WSL2 Ubuntu 24.04 development environment; clean measurement/reproduction work uses the permitted Ubuntu 24.04 `workflow_dispatch` method
  - Outside-reviewer decision: sole authorship; no outside-review milestone is claimed or scheduled
- Current milestone: `v0.1`
- Next unit of work: implement the three-fixture synthetic golden CI tier, real-corpus golden expectations/workflow with per-metric tolerances, and a root-cause memo ending in a regression test; tag `v0.5` only after both tiers pass

## CustodyLedger (`custodyledger`)

- Current phase: not started; blocked by Flightcheck `v1.0` sequencing gate
- Repository URL: https://github.com/Sevens-Dev/custodyledger
- Current branch: not applicable
- Latest merged PR or commit: none
- Completed exit criteria: none
- Tests currently passing: none
- Documented cut-order decisions: none
- Unresolved external blockers: Flightcheck has not reached `v1.0`; hosting decision is deferred to Flightcheck Week 7
- Current milestone: none
- Next unit of work: wait for Flightcheck `v1.0`

## FieldVoice (`fieldvoice`)

- Current phase: not started; blocked by CustodyLedger `v1.0` sequencing gate
- Repository URL: https://github.com/Sevens-Dev/fieldvoice
- Current branch: not applicable
- Latest merged PR or commit: none
- Completed exit criteria: none
- Tests currently passing: none
- Documented cut-order decisions: none
- Unresolved external blockers: CustodyLedger has not reached `v1.0`; volunteer recordings and paid-provider credentials do not exist in this workspace
- Current milestone: none
- Next unit of work: wait for CustodyLedger `v1.0`, except for the explicitly permitted final-week recording preparation
