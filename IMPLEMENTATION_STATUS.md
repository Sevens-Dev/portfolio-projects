# Implementation status

Last updated: 2026-09-16

This file is informational. `HANDOFF.md` remains authoritative.

## Flightcheck (`px4-reqcheck`)

- Current phase: handoff phase Week 0
- Repository URL: pending GitHub CLI re-authorization; local repository at `../px4-reqcheck`
- Current branch: `main`
- Latest merged PR or commit: `459c34c Scaffold Python package and CI` (local)
- Completed exit criteria: scaffold committed; pinned Python 3.12 environment resolves; Ruff, mypy, and the scaffold test pass locally
- Tests currently passing: 1 pytest test; Ruff format/check; strict mypy package check
- Documented cut-order decisions: model baseline remains assumed cut by default; no never-cut item removed
- Unresolved external blockers:
  - GitHub CLI credentials are invalid until the device authorization flow is approved
  - Execution host is Ubuntu 26.04, not the required WSL2 Ubuntu 24.04; no clean-environment or final benchmark claim can be made here
  - Outside-reviewer decision has not been supplied, so no reviewed-PR criterion is claimed
- Current milestone: none
- Next unit of work: complete the Week 0 source-verification ADR and learning spike, then create the public repository and open the first PR-sized branch when authentication is restored

## CustodyLedger (`custodyledger`)

- Current phase: not started; blocked by Flightcheck `v1.0` sequencing gate
- Repository URL: not created
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
- Repository URL: not created
- Current branch: not applicable
- Latest merged PR or commit: none
- Completed exit criteria: none
- Tests currently passing: none
- Documented cut-order decisions: none
- Unresolved external blockers: CustodyLedger has not reached `v1.0`; volunteer recordings and paid-provider credentials do not exist in this workspace
- Current milestone: none
- Next unit of work: wait for CustodyLedger `v1.0`, except for the explicitly permitted final-week recording preparation
