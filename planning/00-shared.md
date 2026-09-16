# Shared context for all three projects

Referenced by `01-custodyledger.md`, `02-flightcheck.md`, `03-fieldvoice.md`. Nothing in this file is repeated in those plans.

## S1. Purpose

Three portfolio repositories whose purpose is to serve as verifiable software evidence on internship applications. Every design decision is subordinate to one test: a hiring engineer opens the repo and can re-run a claim. Feature richness is not a goal. A shipped, tested, measured, smaller artifact beats a larger unfinished one.

Derived from an analysis of 30 internship postings the developer has already applied to. Cluster targets:

| Project | Primary cluster | Language evidence |
| --- | --- | --- |
| CustodyLedger | Product web + application security | TypeScript (primary), Python (verifier) |
| Flightcheck | Data / validation engineering | Python, SQL, C++17 |
| FieldVoice | Applied AI / inference | Python |

## S2. Developer constraints (hard inputs, not negotiable)

- **Capacity: ~10 hours/week.** Full-time field job (Explosive Technician, plant outage schedules, some long shift days) plus continuous 8-week online CS terms. Plans that assume 20 h/week have already been rejected twice in review.
- **Sequential, not parallel.** Exactly one project is in active development at a time. A project is finished (v1.0 tagged) before the next begins.
- **Existing skills:** Python (intermediate, async, MySQL), C++ (beginner; one Unreal Engine actor), JavaScript/React (beginner; one multi-step form using React Hook Form + Zod), SQL (fundamentals, MS SQL Server), Linux (user level), Git (basic).
- **Absent skills, must be treated as new learning with real cost:** TypeScript, Node backend, Fastify, Postgres administration, Pandas, NumPy, SciPy, DuckDB, scikit-learn, pyulog, CMake, GoogleTest, GitHub Actions, Docker, FastAPI, llama.cpp, property-based testing, k6.
- **No hardware budget assumed.** No microcontrollers, no drones, no Raspberry Pi, no paid GPU. One Windows 11 Home laptop.
- **No employer data, ever.** No site names, procedures, forms, vocabularies, audio, or readings from Groome Industrial Services, SCF Lewis & Clark Marine, or Team Industrial Services. All domain data in all three projects is synthetic or public, and every README says so explicitly.

## S3. Environment baseline

Common to all three unless a plan overrides:

- Windows 11 Home host. WSL2 Ubuntu 24.04 for anything requiring Linux (C++ builds, llama.cpp builds, shell tooling). Assume WSL2 is installed but nothing inside it is.
- Git + GitHub, public repos, personal account.
- GitHub Actions, free tier, `ubuntu-latest` runners.
- Editor: VS Code with the WSL remote extension.
- Node 22 LTS (CustodyLedger only) via `nvm-windows` or WSL `nvm`.
- Python 3.12 (Flightcheck, FieldVoice) via `uv` for environment and lock management. `uv` is chosen over venv+pip for reproducibility and speed; a `uv.lock` is committed.
- Docker Desktop is **not** required by any v1.0. Anything that would need local Docker uses a hosted service or a CI service container instead. Docker appears only in FieldVoice as a published Dockerfile that CI builds (never required for local development).

**Cost ceiling: under $10/month total.** Neon free tier (CustodyLedger DB), Render or Fly.io smallest paid instance if the free tier sleeps (~$0-7/mo), Deepgram free credit then pay-as-you-go (FieldVoice; budget $20 one-time), GitHub Actions free minutes, GitHub Pages free. Any design that exceeds this is wrong.

## S4. Repository conventions (identical across all three)

Every repo:

```
README.md              # see S5
LICENSE                # MIT
AI-USAGE.md            # see S7
docs/adr/NNN-title.md  # 3+ architecture decision records
.github/workflows/ci.yml
Makefile               # or justfile; `make all`, `make test`, `make bench` where applicable
```

- **Account: the developer's existing personal GitHub account** (decided 2026-09-16). Consequence: the account profile is part of the evidence, not just the three repos. Before the first `v0.1` tag, pin the portfolio repos, and make sure the profile does not lead with abandoned or unrepresentative work. A profile README is optional; pinning is not.
- **Public from day 1.** First commit is the scaffold, not a finished feature. Commit history is evidence; a single squashed "initial commit" of a finished project is a negative signal.
- **Branch + PR workflow for every unit of work**, even solo. Each PR has a written description stating what changed and why. Branch protection on `main` requiring CI green.
- **Tags are milestones, not releases.** `v0.1` is defined per project as the earliest point where the repository link is worth putting on an application. `v0.5` is the "presentable even if the remaining weeks are lost" point. `v1.0` is the definition of done.
- **Commit messages:** imperative subject under 72 chars. No AI attribution trailers in commits for these repos (they are personal portfolio work; AI usage is disclosed once, in `AI-USAGE.md`, which is the more credible form).
- **No secrets in the repo.** `.env.example` committed, `.env` git-ignored. CI uses repository secrets. One test asserts that no file matching a credential pattern is tracked.

## S5. README contract

The README header must let a reviewer decide in 10 seconds whether to keep reading. Required order:

1. One-sentence description leading with engineering nouns, not domain nouns.
2. A single line of hard numbers: test count, CI status badge, and the project's headline measurement (see the per-project definition).
3. Live URL or one-command run instruction.
4. "What this is not" — 2 to 4 bullets bounding the claim.
5. Architecture diagram (a committed SVG or a Mermaid block).
6. Reproduction: exact commands, from clone to the numbers in item 2.

Everything else goes below the fold.

## S6. Measurement contract

Applies to every number that appears in a README, a resume bullet, or a benchmark table. A number without these attributes must be deleted rather than published.

- Hardware stated: CPU model, RAM, and whether the run was on the Windows host, WSL2, a VPS, or a CI runner.
- Run count stated, and the statistic named (median of N, p50/p95/p99, mean ± SD). Single-run numbers are not published.
- The exact command that produces it, committed and runnable.
- Tool version pinned (compiler + flags, Python + library versions, Node version).
- For latency percentiles: the load generator's configuration and whether the client or the server is the bottleneck.
- WSL2 and shared-CPU cloud instances have unstable timing. Either pin cores (`taskset`) and disclose, or move the final number to a quieter machine and disclose which.

Any statistic with a confidence interval must have the interval computed by a function that is itself unit-tested against a closed-form or published example (see S9).

## S7. AI-USAGE.md contract

All three projects are built with AI assistance. Concealing it is both dishonest and a wasted opportunity: five of the target postings explicitly reward demonstrated judgment when using AI tools. Required contents:

- Which components are hand-written and can be explained line by line. For each project the plan names these explicitly; they are non-negotiable and must not be generated.
- Which components were AI-assisted (scaffolding, boilerplate, docs, test fixtures).
- At least two specific defects introduced by AI assistance, each with the failing test that caught it and a link to the fixing commit. If fewer than two occur naturally, that is a signal the tests are too weak, not that the file should be padded.
- One case where an AI suggestion was rejected on correctness grounds, with the reasoning.

## S8. CI contract

Every repo's `ci.yml` runs on push and PR and must be green before merge. Minimum jobs:

- Lint + format check.
- Type check (TypeScript `tsc --noEmit`; Python `mypy` on the modules where it is cheap — see per-project scoping).
- Unit tests.
- The project's characteristic correctness gate (per project: tamper-detection tests; golden-output regression; evaluation smoke run).
- A dependency audit step.

CI must run in under 10 minutes. Long benchmarks, corpus downloads, and model inference are manual or scheduled workflows, never on the PR path.

## S9. Shared statistical requirements (Flightcheck + FieldVoice)

Both projects publish interval estimates. They do **not** share code (separate repos, separate languages of use), but they share these rules:

- **Percentile bootstrap** for continuous metrics (WER, numeric-token accuracy, F1). Implement it directly (~20 lines: resample with replacement B=10,000, take the 2.5th and 97.5th percentiles). Unit-test the implementation against a case with a known analytic answer (e.g. the CI of a sample mean from a large normal sample against the t-interval) and assert agreement within tolerance.
- **Wilson score interval** for binomial proportions (violation rates, flag rates by bucket). Do not use the normal approximation; it is wrong at the small bucket sizes these projects produce. Unit-test against published values.
- **Every bucketed statistic reports its denominator.** A rate with n < 20 is displayed with the count and marked as insufficient, not plotted as a point estimate.
- **Pre-registration for any comparison.** Before running a comparison that will be published, write the hypothesis, the metric, the decision rule, and what a null result will be reported as, into a committed file. A null result reported honestly is a stronger signal than a positive result found by searching. This is the single most transferable habit from the developer's NDT background and should be visible in the repo.

## S10. Domain disclosure rules

Two projects use the developer's industrial domain (ultrasonic thickness inspection, controlled-materials custody) as their problem space. This is the differentiator and must be handled precisely.

- Domain knowledge explains **why** the software is designed as it is. It is never presented as software experience.
- Never describe NDT inspection, QA/QC, or regulatory compliance work as software testing, software validation, or security engineering. Not in a README, not in a commit, not in a resume bullet.
- Rules encoded from public standards (e.g. remaining-life and next-inspection conventions) are described as "a simplified public convention," never as compliance with API 570/510, ATF, OSHA, or NFPA.
- All synthetic data generators state their parameters and are committed, so a reviewer can regenerate the dataset.

## S11. Cross-project sequencing

Total: 22 active weeks, plus a 1-week learning spike before Flightcheck.

| Order | Project | Weeks | Gate to start |
| --- | --- | --- | --- |
| 1 | Flightcheck | 0 + 7 | Start immediately if Zipline requisitions are still open; otherwise swap with CustodyLedger |
| 2 | CustodyLedger | 8 | Flightcheck v1.0 tagged |
| 3 | FieldVoice | 7 | CustodyLedger v1.0 tagged |

Swap 2 and 3 if the applied-AI postings (Deepgram, HP IQ AML) become the priority. Never overlap two projects. The one permitted exception is passive time: a corpus download, a long fuzz run, or a benchmark sweep may run unattended while the developer is not at the machine.

**Resume update points:** each project's `v0.1` tag is the moment its repository URL goes onto the resume and into application GitHub fields. Do not wait for `v1.0`.

## S12. Definition of done, shared elements

A project is done when, in addition to its own criteria:

- `v1.0` is tagged and CI is green on that tag.
- A fresh clone on a different machine reproduces every published number using only the documented commands.
- README satisfies S5; every number satisfies S6; `AI-USAGE.md` satisfies S7.
- The resume bullets for the project are written, each under 30 words, each with its `<N>` placeholders replaced by measured values or the clause deleted.
- At least three ADRs exist recording decisions where a reasonable alternative was rejected.

## S13. Known shared risks

| Risk | Mitigation |
| --- | --- |
| Capacity collapses during a plant outage or exam week | Every plan front-loads its resume-critical artifact before week 4 and defines an explicit cut order. Losing weeks 6-8 must not invalidate the tagged milestone. |
| Toolchain setup consumes the first week | Week 1 of each plan is deliberately scoped to setup plus one thin vertical slice, with no user-visible feature required. |
| Scope creep from "one more feature" | Each plan has a "what to avoid" list. Items on it are refused, not deferred. |
| Numbers turn out unimpressive or null | S9 pre-registration. A published null result with a correct method is the intended fallback, not a failure. |
| Learning a tool takes longer than budgeted | Each plan names its single riskiest new tool and the fallback if it is not working by a stated date. |

## S14. Unresolved decisions (shared)

- **Whether to pursue outside PR reviewers.** Stripe's posting states a minimum requirement of multi-person projects demonstrating a feedback loop, which sole-authored repos do not satisfy. Options: recruit a reviewer from a developer community, or contribute one merged upstream PR to a dependency of each project. The latter is cheaper and is scheduled as an optional item in each plan. Not decided.
- **Hosting provider for CustodyLedger** (Render vs Fly.io) pending a check of which currently offers a free or sub-$5 tier that does not cold-start badly enough to distort latency numbers.
