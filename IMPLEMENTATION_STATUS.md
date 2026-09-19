# Implementation status

Last updated: 2026-09-19

This file is informational. `HANDOFF.md` remains authoritative.

## Flightcheck (`px4-reqcheck`)

- Current phase: handoff phase Week 6
- Repository URL: https://github.com/Sevens-Dev/px4-reqcheck
- Current branch: `main`
- Latest merged PR or commit: PR #17, `Add real-corpus golden regression tier` (`893155d`); annotated tag `v0.5`
- Completed exit criteria: Weeks 0-5; deterministic 20-log checksum-pinned corpus; normalized Parquet and complete data-quality reporting; seven corrected requirements with sentinel-safe parameter-derived thresholds and total three-valued evaluation; committed traceability report; byte-for-byte clean Ubuntu 24.04 reproduction; CMake/GoogleTest C++17 checker builds in CI; versioned exchange schemas; C++ independently recomputes the raw-sample landing descent p95; 140/140 cross-language verdict agreement; 20-run checker timing evidence; three-fixture synthetic golden CI tier; 140-pair real-corpus golden tier green in hosted workflow 35450552601; root-cause memo and guarded landing-rate regression
- Tests currently passing: 60 ordinary pytest tests; 2 real-corpus golden tests; 5 CTest/GoogleTest tests; Ruff format/check; scoped strict mypy; dependency audit; gitleaks; clean real-corpus `make all` reproduction and C++ agreement
- Documented cut-order decisions: model baseline remains assumed cut by default; no never-cut item removed
- Unresolved external blockers:
  - Local execution host is Ubuntu 26.04, not the preferred WSL2 Ubuntu 24.04 development environment; clean measurement/reproduction work uses the permitted Ubuntu 24.04 `workflow_dispatch` method
  - Outside-reviewer decision: sole authorship; no outside-review milestone is claimed or scheduled
- Current milestone: `v0.5`
- Next unit of work: widen the corpus across firmware versions as bandwidth permits, record alias-layer breakage, and implement denominator-aware Wilson slicing by altitude, duration, and solar elevation; keep the model baseline cut unless core Week 6 work finishes early

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
