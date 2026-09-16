# Implementation Handoff: Three Portfolio Projects

This document is the single source of truth for implementing Flightcheck, CustodyLedger, and FieldVoice. It is self-contained: a coding session needs only this file plus the codebase it produces. Do not consult any other planning document. README and resume text for all three projects are written from this document only.

---

## 1. Shared System Context

Referenced throughout as **§1.x**. Not repeated in the per-project sections.

### 1.1 Purpose and evaluation standard

Three portfolio repositories serve as verifiable software evidence on internship applications. Every design decision is subordinate to one test: a hiring engineer opens the repo and can re-run a claim. A shipped, tested, measured, smaller artifact beats a larger unfinished one.

Cluster targets:

| Project | Primary cluster | Language evidence |
|---|---|---|
| Flightcheck | Data / validation engineering | Python, SQL, C++17 |
| CustodyLedger | Product web + application security | TypeScript (primary), Python (verifier) |
| FieldVoice | Applied AI / inference | Python |

### 1.2 Developer constraints (hard inputs)

- **Capacity: ~10 hours/week.** Full-time field job (Explosive Technician, plant outage schedules, some long shift days) plus continuous 8-week online CS terms. Plans assuming 20 h/week are rejected.
- **Sequential, not parallel.** Exactly one project in active development at a time. A project is finished (`v1.0` tagged) before the next begins. The one permitted exception is passive unattended time (a corpus download, a long fuzz run, a benchmark sweep, or preparing FieldVoice's recording materials during CustodyLedger's final week — see §6).
- **Existing skills:** Python (intermediate, async, MySQL), C++ (beginner; one Unreal Engine actor), JavaScript/React (beginner; one multi-step form using React Hook Form + Zod), SQL (fundamentals, MS SQL Server), Linux (user level), Git (basic).
- **Absent skills, treated as new learning with real cost:** TypeScript, Node backend, Fastify, Postgres administration, Pandas, NumPy, SciPy, DuckDB, scikit-learn, pyulog, CMake, GoogleTest, GitHub Actions, Docker, FastAPI, llama.cpp, property-based testing, k6.
- **No hardware budget.** No microcontrollers, drones, Raspberry Pi, or paid GPU. One Windows 11 Home laptop.
- **No employer data, ever.** No site names, procedures, forms, vocabularies, audio, or readings from any current or former employer. All domain data in all three projects is synthetic or public, and every README states this explicitly.

### 1.3 Environment baseline

**All development for all three projects happens inside WSL2 Ubuntu 24.04**, including Node (via WSL `nvm`, not `nvm-windows`) and Python. This is a hard requirement, not a preference: `make`, GoogleTest's build chain, and Semgrep have no usable native Windows path, and splitting Node between the Windows host and WSL2 produces two `node_modules` trees with incompatible native binaries (e.g. `@node-rs/argon2`). VS Code with the WSL Remote extension is the editor for all three repos.

- Git + GitHub, public repos, the developer's existing personal account (§1.4).
- GitHub Actions, free tier, `ubuntu-latest` runners.
- Node 22 LTS via WSL `nvm`, for CustodyLedger's `server/` and `web/`.
- Python 3.12 via `uv` for environment and lock management, for Flightcheck, FieldVoice, **and CustodyLedger's `verifier/`** (all Python subprojects use `uv`, not venv+pip). A `uv.lock` is committed in each.
- Docker Desktop is **not** required for local development in any project. FieldVoice publishes a Dockerfile that CI builds; it is never required for local work. CustodyLedger uses a native local Postgres install (§1.3.1) and a CI-only Postgres service container; it does not use Docker locally at all.

**1.3.1 CustodyLedger local database.** Do not use a Neon branch for local development or for running the Vitest integration suite / `fast-check` properties. Neon's free plan auto-suspends compute after 5 minutes of idle (cannot be disabled), caps usage at 100 CU-hours/project/month and 0.5 GB storage, and a cold start adds 1-3 s to the first query after idle — this would make local test runs flaky and would make any latency number depend on whether Neon happened to be asleep. Install Postgres 16 natively in WSL2 for all local dev and test runs:

```
sudo apt install postgresql-16
```

Neon is used only for the deployed instance's `main` branch. CI continues to use a `postgres:16` GitHub Actions service container (unchanged). Before every k6 storm or latency run against the deployed instance, send one warm-up request 60 s ahead of the run and disclose in the README that the instance uses scale-to-zero.

**Cost ceiling: under $10/month total.** Neon free tier (CustodyLedger deployed DB), the CustodyLedger hosting decision in §11 governs the API host cost, Deepgram free credit then pay-as-you-go (FieldVoice; budget $20 one-time, actual spend published per §1.6), GitHub Actions free minutes, GitHub Pages free. Any design exceeding this is wrong.

### 1.4 Repository conventions (identical across all three)

Every repo:

```
README.md              # see §1.5
LICENSE                # MIT (code)
AI-USAGE.md            # see §1.7
docs/adr/NNN-title.md  # 3+ architecture decision records
.github/workflows/ci.yml
Makefile               # `make all`, `make test`, `make bench` where applicable
```

- **GitHub account: the developer's existing personal account. This is decided and final, not open.** Consequence: the account profile is part of the evidence, not just the three repos. Before the first `v0.1` tag of the first project (Flightcheck, §11), pin the three portfolio repos on the profile, and audit the profile so it does not lead with abandoned or unrepresentative work (archive, unpin, or otherwise de-emphasize prior repos that would undercut the portfolio). A profile README is optional; pinning is not.
- **Public from day 1.** First commit is the scaffold, not a finished feature — this includes Flightcheck's week 0: the scaffold commit (README stub, `pyproject.toml`, CI with one trivial test) is the first commit of week 0, day 1; the scratch/learning script lands in `notebooks/` later the same week. Commit history is evidence; a single squashed "initial commit" of a finished project is a negative signal.
- **Branch + PR workflow for every unit of work**, even solo. Each PR has a written description stating what changed and why. Branch protection on `main` requiring CI green.
- **Tags are milestones, not releases.** `v0.1` is the earliest point where the repository link is worth putting on an application. `v0.5` is the "presentable even if the remaining weeks are lost" point. `v1.0` is the definition of done (§8).
- **Commit messages:** imperative subject under 72 chars. No AI attribution trailers in commits for these repos (personal portfolio work; AI usage is disclosed once, in `AI-USAGE.md`).
- **No secrets in the repo.** `.env.example` committed, `.env` git-ignored. CI uses repository secrets. `gitleaks/gitleaks-action` runs in CI with the default ruleset, plus one unit test per repo asserting `git ls-files` contains no path matching `(^|/)\.env$|\.pem$|id_rsa|\.key$`.

### 1.5 README contract

The header must let a reviewer decide in 10 seconds whether to keep reading. Required order:

1. One-sentence description leading with engineering nouns, not domain nouns.
2. A single line of hard numbers: test count, CI status badge, and the project's headline measurement.
3. Live URL (CustodyLedger) or one-command run instruction (Flightcheck, FieldVoice).
4. "What this is not" — 2 to 4 bullets bounding the claim (see §10 for the mandatory bullets per project).
5. Architecture diagram (committed SVG or a Mermaid block).
6. Reproduction: exact commands, from clone to the numbers in item 2.

Everything else goes below the fold.

### 1.6 Measurement contract

Applies to every number in a README, resume bullet, or benchmark table. A number without these attributes is deleted, not published:

- Hardware stated: CPU model, RAM, and whether the run was on the Windows host, WSL2, a VPS, or a CI runner.
- Run count stated, and the statistic named (median of N, p50/p95/p99, mean ± SD). Single-run numbers are not published.
- The exact command that produces it, committed and runnable.
- Tool version pinned (compiler + flags, Python + library versions, Node version).
- For latency percentiles: the load generator's configuration and whether the client or the server is the bottleneck.
- WSL2 and shared-CPU cloud instances have unstable timing. Either pin cores (`taskset`) and disclose, or move the final number to a quieter machine and disclose which.

Any statistic with a confidence interval must have the interval computed by a function unit-tested against a closed-form or published example — the exact vectors are in §1.9.

### 1.7 AI-USAGE.md contract

All three projects are built with AI assistance; concealing it is dishonest and wastes a signal several target postings explicitly reward. Required contents, per project (the hand-written lists are given in each project's §D "Definition of done" area and must not be generated):

- Which components are hand-written and can be explained line by line (list given per project below — non-negotiable, never generated).
- Which components were AI-assisted (scaffolding, boilerplate, docs, test fixtures).
- At least two specific defects introduced by AI assistance, each with the failing test that caught it and a link to the fixing commit. If fewer than two occur naturally, the tests are too weak — do not pad the file.
- One case where an AI suggestion was rejected on correctness grounds, with the reasoning.

### 1.8 CI contract

Every repo's `ci.yml` runs on push and PR, green before merge. Minimum jobs:

- Lint + format check.
- Type check: TypeScript `tsc --noEmit`; Python `mypy` scoped per project (Flightcheck: `requirements/`, `params/`, `stats/`, `cli` only — DataFrame-heavy modules are out of scope for `mypy` and covered by `ruff` instead; FieldVoice: `mypy --strict` over the whole package; CustodyLedger `verifier/`: `mypy --strict`).
- Unit tests.
- Dependency audit: Node `npm audit --audit-level=high`; Python `uv export --format requirements-txt | pip-audit -r /dev/stdin` (add `pip-audit` to dev dependencies in every Python subproject).
- Secret scan: `gitleaks/gitleaks-action`.
- The project's characteristic correctness gate: CustodyLedger tamper-detection tests; Flightcheck golden-output regression (tier 1, §fc.7.5); FieldVoice evaluation smoke run against committed fixtures.

CI runs in under 10 minutes. Long benchmarks, corpus downloads, and model inference are manual (`workflow_dispatch`) or scheduled workflows, never on the PR path.

### 1.9 Shared statistical rules (Flightcheck + FieldVoice)

The two projects do not share code (separate repos, separate primary languages of use for this component — both are Python here, but the implementations are kept independent), but they share these rules and one committed data file.

**Percentile bootstrap**, for continuous metrics measured independently per condition (WER, numeric-token accuracy, F1, timing): resample the observations with replacement, B=10,000, take the 2.5th and 97.5th percentiles of the resampled statistic. Implement directly (~20 lines).

**Wilson score interval**, for binomial proportions (violation rates, flag rates by bucket, pass/fail rates). Never use the normal approximation.

**Paired-bootstrap rule for any comparison of two conditions measured on the same items** (boosting on/off on the same 200 utterances; DuckDB vs SQLite on the same 50-log slice used only as a timing comparison, not applicable here since that's not paired-by-item — applies specifically wherever the same item is measured twice, e.g. boosting on/off per utterance, or two pipeline versions over the same logs): the statistic is the **mean paired difference**. Resample **items** (not raw observations) with replacement, B=10,000, and report the 95% percentile interval of the resampled mean difference. **The pre-registered decision rule is: the interval excludes 0.** Two independently computed condition-level intervals that merely fail to overlap is not a valid decision procedure and must not be used.

**Exact test vectors (unit-test both implementations against these before any published number depends on them):**

- Wilson 95% CI, `x=1, n=10` → `(0.0179, 0.4042)`.
- Wilson 95% CI, `x=0, n=20` → lower bound `0`, upper bound `z²/(n+z²) = 0.1611` (z = 1.96).
- Percentile bootstrap: draw n=1000 samples from N(0,1) with a fixed seed, B=10,000. Each of the two percentile-bootstrap endpoints must fall within `0.1 × SE` of the corresponding t-interval endpoint, where `SE = s/√n` (s = sample standard deviation).

Commit these three vectors verbatim as `stats-vectors.json` in **both** Flightcheck and FieldVoice repos, with a comment noting the file is a verbatim copy shared across both repos — this makes any future drift between the two implementations visible instead of silent.

**Every bucketed statistic reports its denominator.** A rate with n < 20 is displayed with the count and marked as insufficient, never plotted as a bare point estimate.

**Pre-registration for any comparison that will be published.** Before running it, commit a file stating: the hypothesis, the metric, the decision rule (§1.9 paired-bootstrap rule where applicable), the minimum difference considered meaningful, the minimum detectable difference given N, and the sentence that will be published if the result is null. A null result reported honestly is a stronger signal than a positive result found by searching, and is the intended fallback outcome for several of the comparisons below — not a failure mode.

### 1.10 Domain disclosure rules

Flightcheck and FieldVoice use the developer's industrial domain (ultrasonic thickness inspection, PX4 flight operations, controlled-materials custody) as problem space. Rules:

- Domain knowledge explains **why** the software is designed as it is. It is never presented as software experience.
- Never describe NDT inspection, QA/QC, or regulatory compliance work as software testing, software validation, or security engineering — not in a README, a commit, or a resume bullet.
- Rules encoded from public standards (remaining-life, next-inspection conventions, PX4 parameter conventions) are described as "a simplified public convention," never as compliance with API 570/510, ATF, OSHA, NFPA, or any PX4/vehicle certification standard.
- All synthetic data generators state their parameters and are committed, so a reviewer can regenerate the dataset.

### 1.11 Report/plan supersession rule

README, `SECURITY.md`, and resume text for all three projects are written from this document only. No earlier planning material or review commentary is a valid source for published wording — this document already incorporates every correction to that material.

---

## 2. Project 1: Flightcheck

Sequenced first (§11). Repository name: **`px4-reqcheck`** (not `flightcheck` — "FlightCheck" is an existing commercial preflight product; a recruiter searching the name would find that product first).

### 2.1 Goal and final deliverable

A reproducible validation pipeline that ingests public PX4 flight logs, evaluates every flight against machine-readable system-level requirements whose thresholds are **derived from each vehicle's own configured parameters**, and emits a requirement-to-log traceability matrix with explicit coverage accounting plus a generated validation report.

The parameter-derived threshold design is the core idea: a verdict against a threshold the developer invented for a stranger's hobby flight is meaningless; a verdict against the threshold that flight's own operator configured has a spec owner.

Final deliverable: public repo, one-command reproduction (`make all`) from raw logs to report, a static published report page, and a C++17 checker that re-derives every verdict and independently recomputes one metric from raw samples, agreeing with the Python evaluator.

Headline README numbers: logs ingested, telemetry rows, ingest throughput, requirement count, coverage (evaluable vs not-evaluable per requirement, with cause breakdown), and Python-vs-C++ verdict agreement (must be 100%, with the scope of that claim stated — see §10).

### 2.2 Architecture

CLI-driven batch pipeline with a columnar analytical store. No service, no database server, no UI framework.

```
px4-reqcheck/
  src/px4reqcheck/
    corpus/       log discovery, download, filtering, manifest
    ingest/       pyulog -> normalized Parquet; data-quality checks; aliases.yaml
    params/       parameter extraction, threshold resolution, params/aliases.yaml
    metrics/      per-flight derived metrics
    requirements/ YAML loader, evaluator, traceability matrix
    report/       Jinja2 HTML + Matplotlib figures + one Plotly drill-down page
    stats/        Wilson intervals, bootstrap, slicing (see §1.9)
    model/        scikit-learn baseline (week 6, cut-order item 1)
    cli.py        Typer entry points
  cpp/            C++17 checker: CMake + GoogleTest
  requirements/   requirements.yaml (the spec under test)
  tests/
  golden/
    ci/           three committed synthetic Parquet fixtures, <=5MB total
    corpus/       ~20 pinned log_ids + expected outputs (not committed data, just ids+expectations)
  data/           git-ignored; corpus lives here
  Makefile
```

Data flow:

```
PX4 Flight Review log index --filter--> corpus manifest (committed)
        |
        v download (git-ignored, checksum-verified)
    .ulg files --pyulog--> Parquet (per-log dir) --> DuckDB views
        |                                                    |
        +--params (aliased)--> resolved thresholds --+       |
        |                                             v       v
        +---------------------------------> requirement evaluator <-- metrics
                                                    |
                        +---------------------------+---------------------------+
                        v                            v                            v
              traceability matrix              HTML + Plotly report      C++ checker (independent recompute)
```

**Storage choice:** Parquet on disk, queried through DuckDB. Rationale (ADR): the working set is tens of millions of rows on a laptop; DuckDB reads Parquet with no server; columnar layout matches the access pattern (a few signals across many flights). A row-store alternative (SQLite) is benchmarked once (§2.7 week 2), pre-registered exactly as follows: dataset = the 20-log week-1 slice; queries = three named queries selected from the five implemented that week (one full-scan aggregate, one per-log group-by, one time-window filter); statistic = median of 10 runs each, cold cache; hypothesis = "DuckDB is at least 5x faster than SQLite on all three queries"; the null-result sentence is written before running and published verbatim if the hypothesis does not hold.

### 2.3 Components

**2.3.1 Corpus.** Source: PX4 Flight Review public log database. **Verified facts** (from `PX4/flight_review` `app/download_logs.py`): index endpoint `https://review.px4.io/dbinfo` (JSON array); download endpoint `https://review.px4.io/download?log=<log_id>`; the reference script filters on `mav_type`, `flight_modes` (log must contain all listed modes), `error_labels`, `rating`, `vehicle_uuid`, `vehicle_name`, `airframe_name`, `airframe_type`, `source`, `git_hash` (`ver_sw`); it supports `--latest-per-vehicle`, `--max-num` (default 10; values above 100 require confirmation), and `--delay` (default 6 s between downloads, to respect server rate limits). It does **not** filter on duration or firmware release in `dbinfo`. There is no published data licence for uploaded logs, and uploaders can delete logs at any time.

**Open verification — before requirements.yaml is finalized, week 0:** confirm whether `dbinfo` entries carry `duration_s` and `ver_sw_release`. Method: fetch `https://review.px4.io/dbinfo` and inspect the keys of one entry. If absent, apply the duration and firmware-series filters after download, from the ULog header (`ULog(path, parse_header_only=True).msg_info_dict`), not from the index.

Filter to a homogeneous slice: vehicle type = multicopter (`mav_type` in the quadrotor/hexarotor/octorotor set — enumerate the exact `mav_type` integer values used, in the ADR, from the MAVLink `MAV_TYPE` enum, at week 0); contains a Mission-mode segment (`nav_state` value 3, `AUTO_MISSION`, present in the log's `flight_modes`/`vehicle_status.nav_state` transitions); duration between 60 s and 20 min; one firmware minor series for the first 20 logs (reduced from an earlier 50-log target, see §2.7 week 1), widened later via the alias layer (week 6).

`corpus/manifest.json` commits selected log UUIDs, their metadata, and a SHA-256 per downloaded file. Raw logs are **never** committed or redistributed — only per-log aggregates, metadata, and the manifest are published (ADR). `make corpus` reproduces the download from the manifest, honoring the 6 s inter-request delay (20 logs ≈ 2 min; a later 300-log widening ≈ 30 min; this runs unattended, the one permitted parallel-time exception per §1.2). It must tolerate `404` for logs a submitter has since deleted: write missing UUIDs to `corpus/missing.json` and continue. The README states reproduction is "best effort against a public service."

`vehicle_uuid` from `dbinfo` (fallback: `msg_info_dict['sys_uuid']` from the ULog header; if both are absent, the log is its own group and this is reported as a count) is the grouping key used later for `GroupKFold` (§2.3.9).

**2.3.2 Ingest.** `pyulog` parses `.ulg` into per-topic tables, normalized into Parquet, one directory per log. Construct with a committed message-name whitelist to bound memory: `ULog(path, message_name_filter_list=WHITELIST)` (verified constructor parameter). Ingest runs in `multiprocessing.Pool(processes=max(1, cpu_count()//2), maxtasksperchild=1)`.

Data-quality checks run during ingest and **report rather than drop**: timestamp monotonicity, timestamp gaps > 1 s, duplicate timestamps, out-of-range values per signal (range table committed per signal), missing required topics, per-signal sample rate. Each check writes counts to a per-log `quality` record.

Multi-instance topics: always read `get_dataset(name, multi_instance=0)`; record `multi_id` in `quality` when more instances exist.

Topic/field aliasing: `ingest/aliases.yaml` maps a logical signal name to an ordered list of `{topic, field, min_version, max_version, scale}` candidates, resolved per log using `ULog.get_version_info(key_name='ver_sw_release')` (verified signature, returns `(major, minor, patch, type)`), with the resolved candidate recorded per log. Seed it with these known renames:

| Logical signal | Pre-rename | Post-rename | Version boundary |
|---|---|---|---|
| trajectory setpoint z | `vehicle_local_position_setpoint.z` | `trajectory_setpoint.z` | v1.13+ |
| raw GPS | `vehicle_gps_position` | `sensor_gps` | v1.13+ (kept as selected output name `vehicle_gps_position`) |
| GPS lat/lon | `sensor_gps.lat/lon` (int32 x1e-7) | `latitude_deg/longitude_deg` (double) | v1.15+ |
| altitude | `alt` (mm) | `altitude_msl_m` | — |

Unresolvable signals mark the affected requirements `not_evaluable` for that log (cause `signal_missing`), not an ingest failure.

**2.3.3 Parameters and threshold resolution.** Extract the full parameter set into `parameters(log_id, name, value_num double, value_type text)` (ULog parameters are `int32` or `float32` only — no `value_str` column). Resolve thresholds **from `initial_parameters`**. If a `requires_params` entry also appears in `changed_parameters` (verified: pyulog exposes both `initial_parameters` dict and `changed_parameters` list of `(timestamp, name, value)`) with a different value than initial, the verdict is `not_evaluable`, cause `param_changed_in_flight`, and the count is reported in the matrix — a mid-flight parameter change makes "the operator's configured threshold" ambiguous.

Parameter names are themselves aliased the same way topics are (`params/aliases.yaml`): logical name -> ordered candidate list, e.g. `bat_n_cells: [BAT1_N_CELLS, BAT_N_CELLS]`, resolved per log with the chosen name recorded in `thresholds.source_params`. **This is required, not optional**: `BAT_N_CELLS`/`BAT_V_EMPTY` were renamed `BAT1_N_CELLS`/`BAT1_V_EMPTY` at multi-battery support (PX4 v1.11); most logs in the public index postdate this, so an unaliased requirement using the old names resolves to `not_evaluable` on nearly every log.

**Open verification — week 0/before requirements.yaml is finalized:** cross-reference every final parameter name against the PX4 parameter reference for the chosen firmware minor series (from the current PX4 docs firmware parameter reference page); record the resulting name table in the ADR that documents the alias seed.

Sentinel/disabled values are not the same as an absent parameter and must not silently resolve to a threshold that is always true or always false. Add `disabled_when: "<expr over params>"` to the requirement schema, evaluated before `expr`; when true, verdict is `not_evaluable`, cause `param_disabled`. Known sentinels to guard explicitly, each with its own unit test:

- `GF_MAX_HOR_DIST == 0` → geofence disabled (else a threshold of 0 with `<=` fails every flight).
- `BAT1_N_CELLS == 0` (or `BAT_N_CELLS == 0`) → "unknown" cell count (else a voltage threshold of 0 passes every flight).
- `COM_DISARM_LAND <= 0` → auto-disarm-after-land disabled. (`COM_DISARM_LAND` is a timeout in seconds after landing before auto-disarm; it is never used as a threshold value for any requirement — it may only be used to help locate the disarm event window.)

Full `not_evaluable` cause taxonomy (exhaustive, used everywhere a verdict can be non-committal): `param_missing`, `param_disabled`, `param_changed_in_flight`, `signal_missing`, `window_missing`, `insufficient_samples`, `quality_fail`.

Requirement expression grammar (`expr` field): Python `ast`, restricted at load time to `Name`, numeric `Constant`, `BinOp` with `{+, -, *, /}`, `UnaryOp -`. Anything else (calls, attribute access, imports, comprehensions) raises at load time; unit-test that `__import__(...)` and arbitrary calls are rejected. `comparator` in `{>=, <=, >, <}`. `requires_signals` uses logical names only, never raw topic names (e.g. `battery_voltage`, not `vehicle_status`).

Every requirement and every parameter-alias entry states a `unit`; the resolver refuses on a unit mismatch.

Target 6-8 requirements. Concrete corrected set (replacing the earlier draft that used stale/wrong parameter names):

```yaml
- id: REQ-BATT-002-REMAIN
  title: Battery remaining fraction at disarm stays at or above the operator's configured low threshold
  metric: battery_remaining_at_disarm
  threshold: { expr: "BAT_LOW_THR" }        # BAT_LOW_THR is a fraction of remaining capacity
  comparator: ">="
  requires_params: [bat_low_thr]
  requires_signals: [battery_remaining, landed]

- id: REQ-BATT-003-VOLT
  title: Battery voltage at disarm stays above the pack's configured empty-cell voltage floor
  metric: battery_voltage_at_disarm
  threshold: { expr: "bat_n_cells * bat_v_empty" }
  disabled_when: "bat_n_cells == 0"
  comparator: ">="
  requires_params: [bat_n_cells, bat_v_empty]
  requires_signals: [battery_voltage, landed]

- id: REQ-GEOFENCE-001
  title: Maximum horizontal distance from home stays within the configured geofence radius
  metric: max_distance_from_home
  threshold: { expr: "GF_MAX_HOR_DIST" }
  disabled_when: "GF_MAX_HOR_DIST == 0"
  comparator: "<="
  requires_params: [GF_MAX_HOR_DIST]
  requires_signals: [local_position, home_position]

- id: REQ-LAND-001
  title: Pre-landing descent rate (95th percentile) stays within the configured land speed
  metric: descent_rate_pre_land_p95
  threshold: { expr: "MPC_LAND_SPEED" }
  comparator: "<="
  requires_params: [MPC_LAND_SPEED, MPC_LAND_ALT2]
  requires_signals: [local_position_vz, land_detected]

- id: REQ-GPS-001
  title: GPS fix quality does not drop below the minimum acceptable fix type during the mission segment
  metric: gps_min_fix_type
  threshold: { expr: "3" }
  comparator: ">="
  requires_params: []
  requires_signals: [gps]

- id: REQ-CRUISE-001
  title: Altitude-hold RMS error during mission cruise stays within a bounded envelope
  metric: altitude_error_rms_cruise
  threshold: { expr: "1.5" }
  comparator: "<="
  requires_params: []
  requires_signals: [local_position, trajectory_setpoint, nav_state]

- id: REQ-VIB-001
  title: Z-axis vibration RMS stays within public PX4 advisory guidance
  class: advisory   # not parameter-derived; sourced from public PX4 Flight Review vibration guidance, cited in the ADR
  metric: vibration_rms_z
  threshold: { expr: "0.3" }
  comparator: "<="
  requires_params: []
  requires_signals: [sensor_combined_or_imu_status]
```

**2.3.4 Metrics.** Five to eight per-flight scalars, each a pure function of normalized signals, each unit-tested against a synthetic input with a hand-computed answer.

Exact windows/definitions (fixing prior ambiguity):

- `altitude_error_rms_cruise`: within `nav_state==3` (AUTO_MISSION) segments, drop the first and last 10 s of each segment, require >=30 s remaining, RMS of `local_position.z - trajectory_setpoint.z`.
- `descent_rate_pre_land_p95`: `vehicle_local_position.vz` (NED, positive down) over the 5 s ending at the **rising edge of `vehicle_land_detected.landed`** (not at disarm — the vehicle sits landed for `COM_DISARM_LAND` seconds with `vz~0` first, which would dilute the percentile); restrict to samples where `-z < MPC_LAND_ALT2` if compared against `MPC_LAND_SPEED`. Percentile method fixed as NumPy `method='linear'`, implemented identically in C++.
- `max_distance_from_home`: horizontal norm of `local_position.(x,y) - home_position.(x,y)`.
- `gps_min_fix_type` / `gps_max_eph`: from the aliased GPS signal.
- `battery_*_at_disarm`: `battery_status` instance 0, `voltage_filtered_v` (voltage) or `remaining` (fraction), last sample before the disarm event, defined as the falling edge of `actuator_armed.armed`.
- `vibration_rms_z`: FFT band power, band `[10, 80]` Hz (confirmed feasible in week 0), `scipy.signal` Butterworth low-pass order 2, cutoff 5 Hz, `scipy.signal.sosfiltfilt`, stated per metric where filtering applies. **Verified**: current `logged_topics.cpp` logs `sensor_combined` at full IMU rate with no default rate limit, so this is feasible on current firmware without the high-rate logging profile; older firmware in the corpus may log at a lower rate. Record `sample_rate_hz` per signal in `quality` at ingest; the metric returns `insufficient_samples` (not a silent zero) when `sample_rate_hz < 2.5 x band_max`. Add `vehicle_imu_status.accel_vibration_metric` (default-profile, 1 Hz) as an alias fallback so the requirement stays evaluable without high-rate IMU logging.

No estimator internals, no controller analysis, no claims about navigation or control performance. The README states plainly that the project validates against configured limits, not control design.

**2.3.5 Requirement evaluator and traceability.** For each (log, requirement): resolve thresholds (applying `disabled_when` first), compute the metric, emit `pass | fail | not_evaluable` with the concrete threshold value, the metric value, and the cause (from the taxonomy in §2.3.3) when not evaluable. Verdicts are total: every (log, requirement) pair yields exactly one of the three states (property-tested, §2.7 week 3 / §2.6).

Traceability matrix: rows are requirements; columns are aggregate counts (`evaluable`, `pass`, `fail`, `not_evaluable` broken down by cause); plus a per-log drill-down. Coverage gaps are printed prominently — a requirement evaluable on 4 of 300 logs is a finding about the corpus, and the report leads with it if coverage overall is low.

**2.3.6 Golden-output regression — two explicit tiers** (a single "20 pinned logs run in CI" design is impossible, because CI does not download the corpus):

- `golden/ci/`: three committed synthetic Parquet fixtures (whitelisted signals only, <=5 MB total). Runs on every push; this is the CI correctness gate (§1.8).
- `golden/corpus/`: ~20 pinned real `log_id`s with committed expected outputs (metrics to a stated absolute+relative tolerance per metric, verdicts exact). Runs via a manually triggered `workflow_dispatch` workflow after `make corpus`, and locally before every tag.

The README states which tier "fails CI when any verdict changes" refers to (tier 1).

**2.3.7 C++17 checker.** `cpp/` builds a standalone binary reading exported per-flight metrics/thresholds and recomputing pass/fail verdicts, **plus one metric computed end-to-end from raw sample arrays in C++** (the descent-rate p95: array traversal, windowing, and a hand-written selection algorithm) so the agreement test compares two independent computations, not a re-application of `value > threshold`.

CMake >=3.20, GoogleTest via FetchContent, nlohmann/json via FetchContent, `-Wall -Wextra -Werror`, built in CI on `ubuntu-latest`. Sanitizers are **not** claimed.

Contract files (both schemas committed and versioned):

`export/checks.schema.json`:
```json
{"version": 1, "logs": [{"log_id": "...", "metrics": {"<name>": 0.0}, "thresholds": [{"req_id": "...", "metric": "...", "comparator": ">=", "value": 0.0, "reason": null}], "raw": {"descent_vz": [0.0], "land_edge_index": 0}}]}
```

`export/cpp_verdicts.json`:
```json
[{"log_id": "...", "req_id": "...", "status": "pass", "metric_value": 0.0}]
```

Agreement test (Python): verdict sets are equal between `verdicts` (Python) and `cpp_verdicts.json`; `descent_rate_pre_land_p95` agrees within absolute 1e-6.

Timing: both sides are timed only from "read `checks.json`" to "write verdicts" (`hyperfine --warmup 3 --runs 20`); ULog parsing stays in Python and is never included. The README never calls this a pipeline speedup.

**Corrected claim wording:** "re-derives every verdict and independently recomputes one metric from raw samples" — not "independently agrees on everything." The 100% agreement figure is by construction for the verdict-recompute step and is meaningful, on independent computation, only for the raw-sample metric; state this under "What this is not."

**2.3.8 Statistical slicing.** Violation rate by bucket, Wilson intervals with denominators shown (§1.9). Bucket edges committed in `stats/buckets.yaml`:

- Altitude (max `-z` above home): `[0,30), [30,60), [60,120), [120,inf)` m.
- Duration: `[60,180), [180,600), [600,1200]` s.
- Solar elevation: `<0, [0,20), [20,45), >=45` degrees, computed from the log's GPS UTC time and position via a closed-form solar-position formula, unit-tested against published almanac values for a few known times/places.

A bucket with n<20 prints `n=<k>, insufficient`, never a bare rate.

Motivation stated in the README: one target posting's application form asks how the candidate would investigate false positives varying by condition; this section answers it with executed code and intervals rather than a paragraph.

**2.3.9 Model baseline (cut-order item 1 — assumed cut from the start).** One scikit-learn logistic regression predicting whether a flight triggers a failsafe (label = a rising edge of `vehicle_status.failsafe`), using **only features computable over `[arming, min(first_failsafe, arming+60s)]`** — whole-flight aggregates like battery-at-disarm directly encode the low-battery failsafe they would predict, and using them is the named leakage risk. `GroupKFold(n_splits=5)` grouped by `vehicle_uuid`. Majority-class baseline reported alongside; class counts stated in the same sentence as precision/recall. A calibration curve and an explicit "what this must not be used for" section. One model, one feature set, no tuning loop. Only attempted if week 5 finishes early (§2.7).

**2.3.10 Report and publication.** Jinja2 HTML: scope, corpus provenance, method, requirement-by-requirement results with figures, coverage gaps, limitations. Matplotlib static figures committed, **plus exactly one Plotly HTML page** for the traceability matrix with per-log drill-down (same DataFrame; do not add a second plotting library beyond this one page — earlier drafts inconsistently said "Plotly report" vs "Matplotlib static," this resolves it). Published to GitHub Pages as a static artifact built from a committed DuckDB/Parquet summary, never re-ingesting at page-build time. GitHub Pages publication is cut-order item 4 (report still generated locally and committed if cut).

One root-cause memo (~2 pages): a real public log where a requirement failed, with timeline, hypothesis chain, the signals that confirmed it, the cause, and the regression test now guarding it in `golden/corpus/`. **Do not commit to "a public failsafe event" in any resume or README wording before the memo's actual subject is fixed** — a requirement violation that is not a failsafe event is an equally valid subject. Use "requirement violation in a public log" as the default wording; upgrade to "public failsafe event" only if `vehicle_status.failsafe` actually transitions in the chosen log.

### 2.4 Data models and interfaces

**Normalized tables (Parquet, one directory per log):**

```
samples_<signal>(log_id, t_us uint64, value float64)   long form, one file per signal
events(log_id, t_us, type, subtype, detail)            type in {nav_state_change, arming_change, landed_change, failsafe_change}; subtype = new value
parameters(log_id, name, value_num double, value_type text)
quality(log_id, check, count, detail)                  check in {ts_nonmonotonic, ts_gap_gt_1s, ts_duplicate, value_out_of_range, topic_missing, sample_rate_hz, param_changed}
logmeta(log_id, airframe, sw_version, duration_s, start_utc, lat0, lon0)
```

`logmeta.start_utc` = first `time_utc_usec > 0` from the aliased GPS signal, minus that sample's boot `timestamp`. `lat0/lon0` from `home_position`. Parquet layout: `data/parquet/log_id=<id>/samples_<signal>.parquet`, read via `duckdb`'s `read_parquet('data/parquet/*/samples_battery_voltage.parquet', hive_partitioning=true)`.

**Derived tables:**

```
metrics(log_id, metric, value, unit, window_start_us, window_end_us)
thresholds(log_id, req_id, resolved_value, expr, source_params jsonb)
verdicts(log_id, req_id, status, metric_value, threshold_value, reason)
```

**CLI:**

```
px4reqcheck corpus build   --filter mission-multicopter --limit N
px4reqcheck ingest         --manifest corpus/manifest.json
px4reqcheck metrics
px4reqcheck validate       --requirements requirements/requirements.yaml
px4reqcheck matrix         --out reports/traceability.html
px4reqcheck report         --out reports/index.html
px4reqcheck slice          --by altitude_band,duration_band,solar_elevation
px4reqcheck model train    --target failsafe
px4reqcheck export-checks  --out export/checks.json
```

`make all` runs manifest -> report end to end.

### 2.5 Dependencies

Python: `pyulog`, `pandas`, `numpy`, `scipy`, `duckdb`, `pyarrow`, `pyyaml`, `jinja2`, `matplotlib`, `plotly` (one page only), `typer`, `scikit-learn`, `requests`. Dev: `pytest`, `hypothesis`, `ruff`, `mypy`, `pip-audit`.
C++: CMake >=3.20, GoogleTest (FetchContent), nlohmann/json (FetchContent).
Managed by `uv` with a committed lock. `mypy` scoped per §1.8.

### 2.6 Testing

| Layer | Tool | Proves |
|---|---|---|
| Unit | pytest | Every metric against a synthetic signal with a hand-computed answer |
| Unit | pytest | Threshold resolution, including `not_evaluable` paths and every sentinel (`GF_MAX_HOR_DIST=0`, `BAT1_N_CELLS=0`, `COM_DISARM_LAND<=0`) |
| Unit | pytest | `param_changed_in_flight` detection |
| Unit | pytest | Expression-evaluator rejects calls/attributes/imports |
| Unit | pytest | Alias layer: two synthetic fixtures whose topics differ by firmware version resolve to the same logical signal |
| Unit | pytest | `quality.sample_rate_hz` gates the vibration metric (`insufficient_samples` below threshold) |
| Unit | pytest | Manifest downloader records a missing log (404) rather than failing the run |
| Property | Hypothesis | Verdicts are total (every pair yields exactly one of three states); threshold resolution is deterministic |
| Statistical | pytest | Wilson and bootstrap implementations against §1.9 vectors; solar position against published almanac values |
| Golden (tier 1) | pytest + CI | Three committed synthetic fixtures produce byte-stable verdicts, every push |
| Golden (tier 2) | pytest, `workflow_dispatch` | ~20 pinned real logs produce in-tolerance metrics and exact verdicts |
| Cross-language | pytest + ctest | C++ and Python agree on all logs per §2.3.7 |
| Data quality | pytest | Injected defects (gaps, duplicates, out-of-range) are counted and reported, never dropped |
| Reproduction | manual/`workflow_dispatch`, once | `make all` from a clean environment (see §9) |

### 2.7 Phased implementation (7 weeks + week 0 spike; re-budgeted)

**Week 0 — learning spike, no feature deliverable.** Scaffold commit day 1 (README stub, `pyproject.toml`, CI with one trivial test) per §1.4. Five logs by hand: `pyulog` -> Pandas -> one metric -> one DuckDB query -> one plot; lands in `notebooks/` later in the week, not on `main` as a feature. **Also this week:** verify the `dbinfo` field question (§2.3.1), enumerate the exact `mav_type` values for the multicopter filter, verify PX4 parameter names for the chosen firmware series (§2.3.3), and confirm no licence terms block programmatic download + derived publication. Record all findings in an ADR. Exit criterion: ADR committed with the verified/verify-pending facts above; scaffold CI green.

**Week 1 — corpus and initial ingest.** Manifest builder with filters; download with checksums, the 6 s delay, and 404 tolerance (`corpus/missing.json`). Ingest **20 logs** (reduced scope), one firmware series. **Only** the timestamp-monotonicity data-quality check runs this week (the rest move to week 2). Typer CLI skeleton. CI green (ruff + pytest). Exit criterion: 20 logs ingested to Parquet, monotonicity check passing/reporting, CI green.

**Week 2 — remaining metrics, DQ, and SQL.** Five to eight metrics with synthetic-input unit tests. Remaining data-quality checks (gaps, duplicates, out-of-range, missing topics, sample rate). First ingest-throughput number (§1.6-compliant). **Five** analytical SQL queries with tests (reduced from ten, to make room). DuckDB-vs-SQLite comparison, hypothesis pre-registered first (§2.9). Static figures. Exit criterion: all metrics unit-tested, DQ checks complete and reporting (not dropping), throughput number published, DuckDB-vs-SQLite result committed with its pre-registration.

**Week 3 — requirements, the resume-critical week.** `requirements.yaml` with the 7 corrected requirements (§2.3.3), alias layer for parameters, threshold resolver with `disabled_when`, evaluator with the full three-valued verdict and cause taxonomy, traceability matrix with coverage accounting, Jinja2 report. **Tag `v0.1`** — repo link goes on applications now; this tag must contain the traceability matrix. Exit criterion: `v0.1` tagged; matrix and report generate from `make all`; every requirement has a passing sentinel/disabled-value test.

**Week 4 — C++ checker.** CMake + GoogleTest scaffold, CI job, `export/checks.json` and `cpp_verdicts.json` contracts, `descent_rate_pre_land_p95` computed from raw arrays in C++, the agreement test, the timing table. **Riskiest tool: CMake + GoogleTest on a first C++ project. Fallback date: end of week 4, day 5** — if the CI job is not green by then, ship the checker as a single-file `g++` build driven by a `Makefile` target, and drop GoogleTest from the claim (state this substitution in the ADR if it happens). Exit criterion: C++ binary builds in CI; agreement test green; timing table published.

**Week 5 — golden suite and root-cause memo.** Golden suite tier 1 (CI, committed synthetic fixtures) and tier 2 (`workflow_dispatch`, ~20 pinned real logs) wired up with declared per-metric tolerances. Root-cause memo on a real failing log ending in a golden test (use "requirement violation in a public log" wording unless a failsafe transition is confirmed present). **Tag `v0.5`.** Exit criterion: both golden tiers green; memo committed with its regression test in `golden/corpus/`.

**Week 6 — corpus widening and statistics (model baseline only if ahead of schedule).** Widen the corpus to several hundred logs across firmware versions via the alias layer (moved here from week 5); record what the widening broke. Wilson-interval slicing by altitude, duration, solar elevation (§2.3.8), with the almanac unit tests. **Model baseline (§2.3.9) only attempted if this week finishes with time remaining** — it is cut-order item 1 and assumed cut by default. Exit criterion: widened corpus ingested; slicing report generated with denominators on every bucket.

**Week 7 — publication and close.** GitHub Pages report (cut-order item 4 if time is short — report still generated and committed locally). README as a validation report per §1.5. `make all` verified via the reproduction method in §9. Three ADRs minimum. `AI-USAGE.md` per §1.7 with the hand-written list from §2.9. **Tag `v1.0`.**

**Post-`v1.0` only, time-boxed to four evenings:** a PX4 SITL feasibility spike — can SITL run headless in WSL2, and can a script fly a short mission producing a ULog the pipeline ingests? Kill-or-keep decision on evening four. If kept, adds a scenario axis and a same-mission repeat-run regression. If killed, list as future work; make no claim about simulation or scenario design anywhere.

### 2.8 Deployment and setup

No server. Distribution is the repo plus GitHub Pages.

Setup: `uv sync`, `make corpus`, `make all`. C++: `cmake -S cpp -B cpp/build && cmake --build cpp/build && ctest --test-dir cpp/build`.

CI does not download the corpus. It runs unit tests, the tier-1 golden regression against the three committed synthetic fixtures (<=5 MB total), and the C++ build+tests. The heavy path (corpus download, tier-2 golden, corpus widening) is a manually triggered workflow.

### 2.9 Risks, AI-USAGE hand-written list, riskiest tool

| Risk | Handling |
|---|---|
| Corpus source assumptions | Week 0 gate (§2.7); fallback: hand-curated smaller corpus |
| Log licensing for derived publication | Verified in week 0 (no published licence found); publish only derived aggregates/metadata, never raw logs |
| Heterogeneous logs across firmware versions | Alias layer for topics, fields, **and parameters**; start with one firmware series; widening deliberately in week 6 |
| Many requirements not-evaluable on public data | This is a *result*; coverage reporting surfaces it; if coverage is very low, the report leads with that finding |
| "This is PX4 Flight Review re-implemented" | It is not: Flight Review renders one log; this evaluates a corpus against requirements with traceability and regression. Stated in "What this is not" |
| C++ week overruns | Checker is not cut; if week 4 overruns, week 6 model baseline is sacrificed instead (already the default) |
| Model produces a leaky or noise-level result | Pre-registered (§1.9); report the leakage analysis and the null result |
| Solar elevation slicing looks arbitrary | Motivated by a specific application-form question; README states the motivation |

**AI-USAGE.md hand-written list (never generated):** `metrics/*.py`, `params/resolve.py`, `requirements/evaluate.py`, `stats/wilson.py`, `stats/bootstrap.py`, `cpp/src/descent.cpp`.

**Riskiest new tool:** CMake + GoogleTest on a first C++ project. **Fallback date: end of week 4, day 5** — substitute a single-file `g++` build with a `Makefile` target if CI is not green by then.

### 2.10 Cut order

(1) Model baseline, (2) solar-elevation slice, (3) corpus widening beyond one firmware series, (4) GitHub Pages publication (report still generated locally and committed).

**Never cut:** parameter-derived thresholds, three-valued verdicts with coverage accounting, the traceability matrix, the golden regression suite (both tiers), the C++ checker.

### 2.11 Definition of done

- 6-8 requirements evaluated across the corpus with per-requirement coverage published, including every `not_evaluable` cause.
- Traceability matrix and HTML+Plotly report generated by `make all` from a clean environment (§9 reproduction method).
- Golden regression green in CI (tier 1) and via `workflow_dispatch` (tier 2).
- C++ checker's verdict-recompute agrees with Python on 100% of logs, asserted in CI; the raw-sample metric (`descent_rate_pre_land_p95`) is independently computed in C++ and agrees within 1e-6; a timing table is published; the README states the agreement claim's exact scope (§2.3.7).
- One root-cause memo whose regression test is in `golden/corpus/`.
- Every published rate carries a Wilson interval and its denominator (§1.9).
- No requirement threshold ever silently resolves through a disabled/sentinel parameter value (tested per sentinel, §2.6).

---

## 3. Project 2: CustodyLedger

Sequenced second, after Flightcheck `v1.0` (§11).

### 3.1 Goal and final deliverable

A deployed, sole-authored web service modeling a controlled-materials chain-of-custody regime as an append-only, tamper-evident ledger, proving three properties with runnable tests:

1. **Tamper evidence.** Editing any tracked column of an audit record (not just its raw bytes), deleting a record anywhere in the chain **including the most recent record**, or reordering records causes `verify` to fail and to name the first bad record, its class, and the lot balances it invalidates.
2. **Exactly-once writes.** N concurrent requests sharing one idempotency key produce exactly one ledger row.
3. **Authorization that holds under attack.** A 30-case denial suite covering IDOR, horizontal and vertical escalation, and self-approval passes.

Final deliverable: public repo + live HTTPS URL with seeded demo accounts per role + a second implementation (Python) of the chain verifier that agrees with the first on a real export.

Headline README numbers: total test count, denial-suite case count (30), the storm result (`1000 concurrent -> 1 row`), and `verify` detection on a mutated byte, a deleted middle record, and a deleted tail record (three distinct, separately tested claims — see §3.4.2).

### 3.2 Architecture

Single repository, two deployables, one shared schema source.

```
custodyledger/
  server/        Node 22 + Fastify + node-postgres (pg)
  web/           Vite + React 18 + TypeScript
  shared/        Zod schemas + canonical-encoding spec, imported by both
  verifier/      Independent Python CLI (uv), mypy --strict
  db/            Numbered .sql migrations, applied by a tiny idempotent runner
  docs/adr/
```

Deliberate exclusions (each an ADR):

| Excluded | Reason |
|---|---|
| pnpm/npm workspaces | Path aliases in two `tsconfig.json` files suffice for two packages |
| ORM (Drizzle, Prisma) | Raw SQL + numbered migrations; an ORM obscures the CHECK constraints that carry the project's argument |
| TanStack Query | Two screens; `fetch` + `useState` suffices |
| Local Docker / Testcontainers | Native Postgres 16 in WSL2 for local dev (§1.3.1), a `postgres:16` CI service container |
| JWT | Server-side sessions in a `sessions` table; revocation is trivial |
| Ed25519 signatures, field-level AES-GCM | HMAC suffices for the threat model; fewer primitives, fewer subtle-wrong ways |

**Trust model.** Single-tenant, one organization. Adversaries: (a) a database administrator who edits or deletes history directly in SQL; (b) an authenticated handler who acts on another handler's lots or approves their own issuance; (c) a network client that replays a write. The HMAC key lives in the application environment, not the database — this is the specific property that makes (a) detectable. State plainly in `SECURITY.md` that an attacker with both database and application-environment access can forge a consistent chain; this is an accepted risk, not a gap.

### 3.3 Components

**3.3.1 Domain model.** Lots of controlled material are received into a magazine, issued to a licensed handler under a two-person rule, transferred between handlers, consumed, returned, and periodically reconciled.

Entities: `User`, `Role`, `Session`, `Lot`, `Holding`, `Issuance`, `CustodyEvent`, `AuditRecord`, `LedgerHead`, `Reconciliation`, `IdempotencyKey`.

**Full lot state machine** (exhaustive transition table test iterates this table — every state x every event yields exactly one of {next-state, rejection-code}):

| State | Event | Result |
|---|---|---|
| `open` | `issue` | -> `issued` |
| `open` | `begin_reconcile` | -> `reconciling` |
| `issued` | `transfer` \| `consume` \| `return` | -> `issued` |
| `issued` | return-all (magazine holds 100% again) | -> `open` |
| `issued` | `begin_reconcile` | -> `reconciling` |
| `open` \| `issued` | `begin_reconcile` | -> `reconciling` |
| `reconciling` | `complete(balanced)` | -> `closed` |
| `reconciling` | `complete(unbalanced)` | -> `discrepancy` |
| `discrepancy` | `resolve(admin, note)` | -> `reconciling` |
| `closed` | anything | rejected, `lot_closed` |
| all other (state, event) pairs | — | rejected |

Invariant, holdings-based (replacing the earlier per-lot-only model, which could not represent transfers): for every lot, `sum(holdings.qty) + qty_consumed = qty_received`; every `holdings.qty >= 0`. `lots.qty_on_hand` is a materialized `sum(holdings.qty)` that `replay` recomputes and compares. `CHECK (qty_on_hand >= 0 AND qty_on_hand <= qty_received)` on `lots`.

Issuance status enum: `pending | issued | rejected | cancelled`. Approve is exactly:
```sql
UPDATE issuances SET approver_id=$approver, status='issued', approved_at=now()
WHERE id=$id AND status='pending' RETURNING *;
-- zero rows => 409 issuance_not_pending
```
(inside the same transaction as the holdings move for that issuance).

Reconcile is two routes, not one: `POST /api/lots/:id/reconcile` (begin, body `{counted_qty}`) and `POST /api/lots/:id/reconcile/complete`. `reconciliations(id, lot_id, counted_qty, ledger_qty, result, created_at)` with `CHECK (result <> 'balanced' OR counted_qty = ledger_qty)` — this is the schema-level enforcement that makes "lot reconciliation enforced by database constraints" a true claim (§3.9).

**3.3.2 Audit chain and tamper evidence.** Each `CustodyEvent` write appends exactly one `AuditRecord` inside the same transaction.

```
audit_records(
  seq            bigint,      -- NOT bigserial; assigned inside the transaction (see below)
  ts_ms          bigint,      -- integer epoch ms, never a formatted timestamp
  actor_id       uuid,
  event_type     text,
  entity_type    text,
  entity_id      uuid,
  lot_id         uuid not null,   -- required so verify() can report affected lots
  payload_sha256 bytea,
  prev_hash      bytea,       -- previous row's record_hmac (32 bytes); 32 zero bytes for seq=1
  record_hmac    bytea,       -- HMAC-SHA256(key, signed_bytes)
  signed_bytes   bytea        -- exact bytes signed, stored verbatim
)
```

**Canonical encoding (fully specified, `SECURITY.md`, written before any code):** RFC 8785 JSON Canonicalization Scheme (npm `canonicalize`, PyPI `rfc8785`). Signed object, version 1:

```json
{"v":1,"seq":<int>,"ts_ms":<int>,"actor_id":"<uuid lowercase>","event_type":"<enum>","entity_type":"<enum>","entity_id":"<uuid>","lot_id":"<uuid>","payload_sha256":"<64 lowercase hex>","prev_hash":"<64 lowercase hex>"}
```

JCS-serialized, UTF-8; JCS itself fixes member order, so no manual key ordering is needed. `seq` and `ts_ms` are JSON integers, both < 2^53. `HMAC_KEY` is 32 bytes, given as 64 lowercase hex characters in `.env`. **The fixed test key for all committed vectors is bytes `0x00..0x1f`, i.e. hex `000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f`.** Future schema fields go under `"v":2`; the verifier dispatches on `v`. Payload objects (the `custody_events.payload` that hashes into `payload_sha256`) contain only strings, integers, booleans, null, arrays, and objects — **no floats, ever** (quantities are decimal strings, e.g. `"12.500"`). A test asserts a payload containing a JS float is rejected before hashing.

**Five committed test vectors** (JSON, consumed by both TypeScript and Python suites): each vector is a full record including a realistic `payload` object, so `payload_sha256` derivation is exercised, not just the outer signed object. Across the five vectors, include at least one non-ASCII string, one nested object, one quantity as a decimal string, and one `null` field. Compute vectors independently of both implementations: `openssl dgst -sha256 -mac HMAC -macopt hexkey:<key>` over the JCS text produced by hand. Each vector records: input record -> `signed_bytes` hex -> `record_hmac` hex.

**bigint/serialization correctness (Postgres `int8` returns as JS strings by default):** in `server/`, set `pg.types.setTypeParser(20, v => Number(v))` for OID 20 (`bigint`) — safe here because `seq` and epoch-ms values are far below 2^53 — while keeping `numeric` (OID 1700) as strings. Without this, `canonicalize` would serialize `"seq":"42"` from the JS side while the Python verifier (receiving integers from the export) emits `"seq":42`, producing HMAC disagreements that are a serialization bug, not a real tamper signal.

**Tamper-evidence anchor (fixes: tail deletion was previously undetectable).** A chain walk alone detects a gap in the middle but not truncation of the tail (rows `1..N-1` remain a valid chain even if the true latest rows are deleted). Add a single-row anchor table:

```sql
ledger_head(
  id int primary key check (id = 1),
  last_seq bigint not null,
  last_hash bytea not null,     -- = the last audit_records.record_hmac
  head_hmac bytea not null      -- HMAC(key, JCS({"seq": last_seq, "hash": "<last_hash hex>"}))
);
```
Initialize with `last_seq=0`, `last_hash` = 32 zero bytes.

**`seq` assignment (fixes: `bigserial` is non-transactional and would leave permanent gaps on any rolled-back write; also fixes unspecified concurrent-append ordering).** Do not use `bigserial`. Inside the write transaction:
```sql
UPDATE ledger_head SET last_seq = last_seq + 1 RETURNING last_seq, last_hash;
-- returns the NEW last_seq and the OLD last_hash (this UPDATE does not touch last_hash),
-- which is exactly the predecessor's hash needed for prev_hash.
-- The row lock this UPDATE takes serializes all concurrent appenders.
```
Build the signed object with `seq = <returned last_seq>`, `prev_hash = <returned last_hash>`; compute `record_hmac`; `INSERT` the audit row; then:
```sql
UPDATE ledger_head SET last_hash = $record_hmac, head_hmac = $new_head_hmac WHERE id = 1;
```
There is exactly one ledger — delete "per-ledger" language; the nursing vocabulary (§3.3.9) is a seed, not a second ledger.

**`verify` decision procedure (fixes: as originally specified, `verify` never re-checked the row's own columns against `signed_bytes`, so a DBA could edit `actor_id`, `event_type`, `entity_id`, `payload_sha256`, or swap non-`seq` columns between two rows, without failing verification).** For each row in `seq` order:

0. Check `head_hmac` against recomputation from `ledger_head`; then confirm `max(seq)` in `audit_records` equals `ledger_head.last_seq` — mismatch is reported as `deleted` at `last_seq + 1` (this is what catches tail deletion).
1. Parse `signed_bytes` as JSON.
2. Assert each parsed field equals the corresponding column (`seq`, `ts_ms`, `actor_id`, `event_type`, `entity_type`, `entity_id`, `lot_id`, `payload_sha256` hex, `prev_hash` hex) — mismatch is `mutated`.
3. HMAC check (`HMAC(key, signed_bytes) == record_hmac`) — mismatch is `mutated`.
4. `prev_hash` equals the previous row's `record_hmac` — mismatch is `chain_break`.
5. Look up the referenced `custody_events` row and assert `SHA256(JCS(payload)) == payload_sha256` — mismatch is `mutated`, `source=custody_events` (this is what catches a DBA editing `custody_events.payload` directly, which the original design left invisible to a chain walk).

`replay` separately rebuilds `holdings`/`lots` from the event log and compares to the materialized tables (this is what catches a DBA editing `lots.qty_on_hand` directly).

`verify` output: `{ok: false, first_bad_seq, class, detail, affected_lots: [...]}` where `class` in `{mutated, deleted, reordered, chain_break}`, `affected_lots` = distinct `lot_id` over rows with `seq >= first_bad_seq`, exit code 2 on failure. Test operations and expected results, one test per row:

| Operation | Expected `first_bad_seq` | Expected class |
|---|---|---|
| Mutate one byte of `signed_bytes` at row k | k | `mutated` |
| `DELETE` row k, k<N | k | `deleted` |
| `DELETE` row N (the tail) | N | `deleted` |
| Swap all non-`seq` columns of rows k and k+1 | k | `reordered` |
| Replace `prev_hash` of row k with random bytes, re-signed with the real key (simulating a key-holding attacker) | k | `chain_break` |
| Edit `custody_events.payload` directly for the event behind row k | k | `mutated`, `source=custody_events` |
| Edit `lots.qty_on_hand` directly | — (caught by `replay`, not `verify`) | divergence reported |

`audit_records` has no `UPDATE`/`DELETE` grant for the application role; a migration creates a restricted role and the application connects as it (tested: expect SQLSTATE 42501).

**3.3.3 Idempotency.** `idempotency_keys(actor_id, key, body_sha256, response_status, response_body, created_at)`, `UNIQUE (actor_id, key)`. `Idempotency-Key` header must be a UUID v4 string (400 otherwise); required on every `POST /api/*` except `/auth/login`. Write path, one transaction:

1. `INSERT ... ON CONFLICT (actor_id, key) DO NOTHING RETURNING *`.
2. Row inserted -> perform the write, store the response **only for 2xx** (any non-2xx aborts the transaction, removing the key row, so retries re-execute), commit.
3. Not inserted -> read existing row. Same `body_sha256` -> return the stored response with header `Idempotent-Replayed: true`. Different `body_sha256` -> `409`.

Under READ COMMITTED, the second concurrent `INSERT ... ON CONFLICT DO NOTHING` blocks until the first transaction commits or rolls back — this is the concurrency property the design relies on (state it in the ADR). Rows older than 24 h are purged by the nightly reset job (§3.7). Test: two concurrent same-key, different-body requests assert exactly one 201 and one 409. **README/`SECURITY.md`/resume wording: "unique constraint on `(actor, key)`; body hash compared on conflict"** — never "unique constraint on actor, key, and body hash" (a triple-column constraint would allow a second row for a different body and could not produce the 409).

**3.3.4 Two-person rule.** Schema-enforced distinctness/completeness:
```sql
ALTER TABLE issuances ADD CONSTRAINT two_person
  CHECK (approver_id IS NULL OR approver_id <> requester_id);
ALTER TABLE issuances ADD CONSTRAINT approved_issuances_complete
  CHECK (status <> 'issued' OR approver_id IS NOT NULL);
CREATE UNIQUE INDEX one_open_issuance_per_lot
  ON issuances (lot_id) WHERE status = 'pending';
```
**Corrected claim wording:** "the distinctness and completeness of the two-person rule are schema constraints; the approver's role is checked in the preHandler and covered by the denial suite; lot reconciliation is enforced by the `reconciliations` CHECK constraint in §3.3.1" — do not claim the CHECKs alone enforce "approver holds keeper role" or "issuance still pending" (those are handler-code checks, backstopped by the CHECK, not replaced by it). A regression test demonstrates an application-layer-only version of the two-person rule can be defeated and the constraint version cannot; keep this test — it is the artifact.

**3.3.5 Authorization.** Role-based with object-level checks. Roles: `handler`, `keeper`, `auditor`, `admin`. A Fastify preHandler resolves the session; each route declares its required role **and** an ownership predicate evaluated against the specific row.

**Visibility sets (fixes: the 404-vs-403 rule was self-contradicting because handler visibility was never defined):**

| Role | Can `GET` lots | Can `GET` holdings |
|---|---|---|
| `handler` | any lot with status `open` or `issued` (needed to request issuance) | only where `holder_id = self` |
| `keeper` / `auditor` / `admin` | all | all |

Error-mapping table (commit as the fixture the denial suite iterates over — `type` slugs and status codes):

| Condition | Status | `type` |
|---|---|---|
| Role check fails | 403 | `forbidden_role` |
| Object outside caller's visibility set | 404 | `not_found` |
| Ownership/state predicate fails on a visible object | 403 | `forbidden_object` |
| Two-person CHECK violation (SQLSTATE 23514) | 409 | `two_person_rule` |
| Missing/invalid `Idempotency-Key` | 400 | `idempotency_key_required` |
| Issuance not pending on approve | 409 | `issuance_not_pending` |
| Idempotency replay, different body | 409 | `idempotency_conflict` |

RFC 9457 problem+json for every error.

**Ownership predicates per mutating route:**

| Route | Predicate |
|---|---|
| `transfer` \| `consume` \| `return` | `exists holdings where lot_id=:id and holder_id=session.user and qty>0` |
| `issuances` (request) | lot visible and `status in (open, issued)` |
| `approve` | role `keeper` and `issuances.requester_id <> session.user` (the CHECK is a backstop, not the only check) |
| `reconcile` (begin/complete) | role `keeper` |

**30-case denial suite**, fixture rows `(actor, route, target, expected_status, expected_type)`, README count = `fixture.length`:

| Class | Count |
|---|---|
| Object reference (IDOR) | 8 |
| Vertical escalation | 8 |
| Approval bypass (self-approval, wrong role, already-issued lot) | 6 |
| Session handling (missing/expired/deleted-user token) | 5 |
| Idempotency abuse (replayed key, different body) | 3 |
| **Total** | **30** |

**3.3.6 Rate limiting vs. the k6 storm (fixes: a naive per-IP rate limit would make the 1000-concurrent storm measure the rate limiter, not idempotency).** Rate-limit **only** `POST /api/auth/login` (10/min/IP). Apply a high per-session allowance elsewhere (600/min). The storm authenticates once and reuses one session cookie; the k6 script asserts no response is 429, exactly one response is 201, the rest are 200-with-`Idempotent-Replayed`, then `SELECT count(*) FROM custody_events WHERE ...` asserts exactly 1.

**3.3.7 Auth parameters (fixes: unspecified Argon2id cost could exhaust a small instance under concurrent logins).** Argon2id: `m=19456 KiB, t=2, p=1` (OWASP minimum; default `@node-rs/argon2` memory cost of 19 MiB is what this pins, avoiding an unspecified "64 MiB" recommendation that would let ~4 concurrent logins exhaust a 256 MB instance). Session TTL 12 h sliding, cookie name `cl_session`, `HttpOnly; SameSite=Strict`, `Secure` only when `NODE_ENV=production` (so it still works over plain HTTP in local dev).

**3.3.8 Independent Python verifier.** `verifier/` is a separate `uv` project, `mypy --strict`, pytest. Consumes the export defined below and independently: recomputes the chain per the §3.3.2 decision procedure, re-derives every balance via holdings, passes the five shared vectors. CI runs it against a freshly seeded database on every push, via a CLI export path (not HTTP), so the check does not depend on the live server:
```
node server/dist/cli.js export > export.ndjson
uv run --project verifier ledger-verify export.ndjson
```

**Export schema (fixes: the original export of "audit records plus lot snapshots" could not reconstruct any balance, since quantities/event types live in `custody_events`).** NDJSON, typed lines:
```
{"t":"head", "last_seq":..., "last_hash_hex":..., "head_hmac_hex":...}
{"t":"audit", "seq":..., "ts_ms":..., "actor_id":..., "event_type":..., "entity_type":..., "entity_id":..., "lot_id":..., "payload_sha256_hex":..., "prev_hash_hex":..., "record_hmac_hex":..., "signed_bytes":"<utf-8 string>"}
{"t":"event", "id":..., "lot_id":..., "type":..., "qty":"<decimal string>", "from_user":..., "to_user":..., "actor_id":..., "payload":{...}, "audit_seq":...}
{"t":"lot", "id":..., "code":..., "qty_received":"<decimal string>", "qty_consumed":"<decimal string>", "status":...}
{"t":"holding", "lot_id":..., "holder_id":..., "qty":"<decimal string>"}
```
The verifier: checks the head, walks the chain, hashes each event payload and matches it to its audit row, replays events into holdings, and compares to the exported holdings/lots.

**3.3.9 Web client.** Two screens: (1) custody operations (issue with approval, transfer, consume, return, reconcile); (2) audit view (chain with per-record verify status, and a control that runs `replay` and displays the diff). React 18 + TypeScript strict, React Hook Form + Zod resolvers against the shared schemas, plain `fetch`. No component library; hand-written CSS with tokens.

**3.3.10 Second vocabulary.** The same schema/code serve a second seeded demo (two-nurse controlled-substance counts, skilled-nursing context), selectable by a seed flag. README presents it as a second vocabulary over one ledger, never a second product. Cut-order item 1 (§3.10) if week 7 is tight.

### 3.4 Data models and interfaces

**3.4.1 Schema** (full DDL in `db/001_init.sql`; abbreviated here — combine with §3.3.1's `reconciliations` and §3.3.2's `ledger_head`/`audit_records`):

```sql
users(id uuid pk, email citext unique, password_hash text, role text, created_at timestamptz)
sessions(id uuid pk, user_id uuid fk, expires_at timestamptz, revoked_at timestamptz)
lots(id uuid pk, code text unique, material text, unit text,
     qty_received numeric(12,3), qty_consumed numeric(12,3), qty_on_hand numeric(12,3),
     status text, opened_at timestamptz, closed_at timestamptz,
     check (qty_on_hand >= 0 and qty_on_hand <= qty_received))
holdings(lot_id uuid fk, holder_id uuid fk, qty numeric(12,3) not null check (qty >= 0),
     primary key (lot_id, holder_id))          -- the magazine is a reserved holder row per lot
issuances(id uuid pk, lot_id uuid fk, requester_id uuid fk, approver_id uuid,
          qty numeric(12,3), status text check (status in ('pending','issued','rejected','cancelled')),
          created_at, approved_at,
          constraint two_person check (approver_id is null or approver_id <> requester_id),
          constraint approved_issuances_complete check (status <> 'issued' or approver_id is not null))
create unique index one_open_issuance_per_lot on issuances (lot_id) where status = 'pending';
reconciliations(id uuid pk, lot_id uuid fk, counted_qty numeric(12,3), ledger_qty numeric(12,3),
     result text, created_at,
     check (result <> 'balanced' or counted_qty = ledger_qty))
custody_events(id uuid pk, lot_id uuid fk, type text, qty numeric(12,3),
               from_user uuid, to_user uuid, actor_id uuid, payload jsonb, created_at)
ledger_head(id int primary key check (id=1), last_seq bigint not null,
            last_hash bytea not null, head_hmac bytea not null)
audit_records(seq bigint primary key,     -- NOT bigserial; assigned per §3.3.2
              ts_ms bigint, actor_id uuid, event_type text, entity_type text, entity_id uuid,
              lot_id uuid not null,
              payload_sha256 bytea, prev_hash bytea, record_hmac bytea, signed_bytes bytea)
idempotency_keys(actor_id uuid, key text, body_sha256 bytea,
                 response_status int, response_body jsonb, created_at,
                 primary key (actor_id, key))
schema_migrations(version text primary key, checksum text, applied_at timestamptz)
```

All decrements on `holdings` are conditional updates (`UPDATE holdings SET qty = qty - $1 WHERE lot_id=$2 AND holder_id=$3 AND qty >= $1 RETURNING *`) so two concurrent consumes cannot overdraw; lock `holdings` rows in `(lot_id, holder_id)` order to avoid deadlock on transfers. Event effects: receive: magazine += qty; issue (on approval): magazine -= qty, handler += qty; transfer: from -= qty, to += qty; consume: holder -= qty, `lots.qty_consumed` += qty; return: handler -= qty, magazine += qty.

Quantities are `numeric`, never floating point; a test asserts a float quantity is rejected at the schema boundary.

Migration files named `NNN_description.sql`, applied in a transaction each, SHA-256 recorded in `schema_migrations`. The runner is idempotent (a test runs it twice and asserts the second run applies nothing).

### 3.5 HTTP interface

All routes under `/api`. Mutating routes require `Idempotency-Key`, return `201` with the created event plus its audit `seq`.

```
POST   /api/auth/login                     -> session cookie
POST   /api/lots                           keeper        receive a lot
POST   /api/lots/:id/issuances             handler       request issuance
POST   /api/issuances/:id/approve          keeper        approve (two-person rule)
POST   /api/lots/:id/transfer              handler       to another handler
POST   /api/lots/:id/consume               handler
POST   /api/lots/:id/return                handler
POST   /api/lots/:id/reconcile             keeper        begin reconciliation, body {counted_qty}
POST   /api/lots/:id/reconcile/complete    keeper        complete reconciliation
GET    /api/lots, /api/lots/:id            role-scoped (visibility sets, §3.3.5)
GET    /api/audit?from=&to=                auditor|admin   seq bounds (integers), max 500 rows, ordered by seq
POST   /api/admin/verify                   admin         runs chain verification
POST   /api/admin/replay                   admin         rebuild-and-compare
GET    /api/admin/export                   admin         NDJSON per §3.3.8 (streaming deferred; not needed until audit table exceeds ~50k rows, which v1.0 seed data does not reach)
POST   /api/admin/reset                    admin         optional, only if needed for the demo video; nightly reset (§3.7) is the primary mechanism
GET    /healthz                            public
```

OpenAPI generated from Zod schemas via `@asteasolutions/zod-to-openapi` (correct package name) and committed.

### 3.6 Dependencies

Server: `fastify`, `@fastify/cookie`, `@fastify/helmet`, `@fastify/rate-limit`, `@fastify/static` (same-origin SPA serving), `pg`, `zod`, `canonicalize`, `@node-rs/argon2`, `pino`.
Test: `vitest`, `@vitest/coverage-v8`, `supertest` (or `fastify.inject`), `fast-check`.
Web: `react`, `react-dom`, `vite`, `react-hook-form`, `@hookform/resolvers`, `zod`.
Verifier: `pytest`, `mypy`, `rfc8785`, `pip-audit`.
Tooling: `typescript`, `eslint`, `prettier`, `k6` (CLI, not a package), `semgrep` (CI only), `gitleaks` (CI only).

**Open verification — week 1 setup:** confirm the compatible Zod major / `@hookform/resolvers` major pairing (Zod 4 requires `@hookform/resolvers` >= 5) by running `npm install` and checking for peer-dependency warnings before writing any form code.

Pin exact versions in a committed lockfile. Coverage floor: 60% lines on `server/src`, enforced in CI as a regression floor (not a target), raised only after `v0.5`. No dependency added after week 5 without deleting another.

### 3.7 Deployment and setup

- Database: Neon free tier for the deployed `main` branch only (never for local dev/test — §1.3.1); a `dev` branch may exist but is not part of the local test loop.
- API host: **decision required before this project starts — see §11 unresolved-decision entry.** `HMAC_KEY`, `DATABASE_URL`, `SESSION_SECRET` as platform secrets.
- Web: static build served by the same origin (`@fastify/static`) to avoid CORS entirely (ADR records this over a separate static host).
- Migrations run on deploy via a release command; idempotent runner records applied versions in `schema_migrations`.
- Seeding: `npm run seed -- --vocabulary=explosives|nursing` creates demo users per role with published credentials: `handler_a`, `handler_b`, `keeper`, `auditor`, `admin`, plus three lots. README states plainly this is a demo instance with synthetic data and published fixed passwords.
- **Nightly reset (fixes: a public write API with an append-only ledger and no reset means demo state degrades within a week):** a GitHub Actions `schedule` workflow (daily) runs the migration runner's `reset` command against the deployed DB (drop schema, migrate, seed) using repository secrets. State on the login page and README: "demo data resets nightly; the chain restarts at seq 1."
- Key rotation is out of scope for `v1.0`, named as an accepted risk in `SECURITY.md`, with a `key_id` column reserved in `audit_records` for the future migration path.

### 3.8 Testing

| Layer | Tool | Proves |
|---|---|---|
| Unit | Vitest | Encoding, hashing, balance arithmetic, state transitions |
| Property | fast-check | Ledger invariants (`sum(holdings.qty)+qty_consumed=qty_received`; `holdings.qty>=0`) over arbitrary valid event sequences |
| Integration | supertest + CI Postgres | Full request path including transactions and constraints |
| Cross-language | pytest | Python verifier agrees with the server on a real export and on the five vectors |
| Adversarial | Vitest | 30-case denial suite |
| Tamper | Vitest | Every row of the §3.3.2 test-operations table, each naming the correct `first_bad_seq` and `class` |
| Load | k6 | Exactly-once under 1000-way concurrency (§3.3.6 script); latency percentiles |
| Static | Semgrep, `npm audit --audit-level=high`, `pip-audit`, `gitleaks` | CI gates; at least one commit fixes a real finding |

Additional required tests (fixes: claims with nothing that would catch them if false): application role cannot `UPDATE`/`DELETE` `audit_records` (SQLSTATE 42501); `seq` stays contiguous after a rolled-back write (a CHECK-violating write followed by a successful write, `verify` clean, `seq` contiguous); tail deletion detected (already in the table above); editing `custody_events.qty` directly is detected by both `verify` (payload hash) and `replay`; concurrent same-key different-body -> exactly one 201 and one 409; a payload with a float is rejected pre-hash; migration runner is idempotent (second run applies nothing); session cookie flags (`HttpOnly; Secure; SameSite=Strict`) present in production-mode login response.

Coverage floor per §3.6, a regression guard, not an optimization target.

### 3.9 Phased implementation (8 weeks; re-budgeted)

**Week 1 — spec, scaffold, deploy path.** Write `SECURITY.md` canonical-encoding section and the five test vectors *first*, by hand, per §3.3.2, before any code. Scaffold `server/` and `web/`. `db/001_init.sql` (full schema from §3.4.1, including `holdings`, `ledger_head`, `reconciliations` from day one — designing them in later is what caused the original architectural gaps) plus the idempotent migration runner. Provision Neon `main`; install Postgres 16 in WSL2 for local (§1.3.1). CI green: lint, typecheck, one trivial test. Deploy `/healthz` to the host decided in §11. Exit criterion: a URL returns `{"ok":true}`; CI green; encoding spec + 5 vectors committed.

**Week 2 — ledger core, chain, `v0.1` (re-scoped: `receive` + `consume` only).** `receive` and `consume` event writes inside transactions. Audit append with HMAC over JCS bytes, `ledger_head` anchor, `seq` assignment per §3.3.2 (not `bigserial`). Idempotency via the unique constraint. `verify` implementing the full 6-step decision procedure (§3.3.2). Tests: mutate one byte -> `mutated` at correct seq; delete a middle record -> `deleted`; **delete the tail record -> `deleted` at `last_seq+1`** (this is the head-anchor test — do not defer it). **Tag `v0.1`.** The `v0.1` README states plainly there is no authentication yet and only two event types exist.

**Week 3 — transfers, holdings, state machine.** `transfer`/`return` events plus the `holdings` model (§3.3.1/§3.4.1). `replay` rebuilding holdings/lots from the event log. Delete/reorder tamper tests (remaining rows of the §3.3.2 table). Full lot-state transition table test (every state x every event). *(fast-check properties moved to week 5, alongside integration tests, where a real Postgres is already wired up for both.)*

**Week 4 — identity and authorization (first half of the denial suite).** Argon2id per §3.3.7, session table, cookie handling. RBAC preHandler plus ownership predicates (§3.3.5). Two-person rule constraints plus the bypass-demonstration test (§3.3.4). `reconciliations` CHECK (already in schema from week 1; wire the routes). **Cases 1-15 of the denial suite** (object-reference class, 8 cases; vertical-escalation class, first 7 of 8 cases) — the fixture is ordered by class per §3.3.5's table, and this week implements the first 15 rows of it.

**Week 5 — web client, integration, properties. Tag `v0.5`.** The two screens; Zod schemas shared with the server; form validation; error rendering from problem+json. Integration tests against the CI Postgres service container. `fast-check` properties (moved from week 3): for any valid event sequence, `sum(holdings.qty)+qty_consumed=qty_received`; `holdings.qty>=0`; replaying the log reproduces materialized state. **Tag `v0.5`** — presentable even if weeks 6-8 are lost.

**Week 6 — independent verifier.** `verifier/` Python package, `mypy --strict`, consuming `/admin/export` (or the CLI export path). Full export schema per §3.3.8. Passes the five shared vectors. Wired into CI against a seeded database via the CLI export path. This week is most likely to surface an under-specified encoding detail; that discovery is expected and is itself evidence for `AI-USAGE.md`.

**Week 7 — operations and measurement (remaining denial cases, rate limiting, storm).** **Cases 16-30 of the denial suite** (remaining 1 vertical-escalation case; approval-bypass, 6; session-handling, 5; idempotency-abuse, 3). Rate limiting (`/auth/login` only, §3.3.6) and `@fastify/helmet`. k6 storm: 1000 concurrent requests (`vus: 1000, iterations: 1`, one shared `Idempotency-Key`, one pre-established session cookie, against `POST /api/lots/:id/consume`), asserting exactly one row and no 429s; run three times, disclose which run was cold, warm the Neon compute 60 s ahead of each run (§1.3.1). Separate latency script: `constant-arrival-rate` at 20 and 50 RPS for 60 s with distinct keys, p50/p95/p99, `pg` pool `max: 10` stated explicitly (so pool queueing, not the app, is the visible bottleneck). Structured logging, `RUNBOOK.md`, second seeded vocabulary (§3.3.10, cut-order item 1). Nightly reset workflow (§3.7) live. Induce one real failure (connection-pool exhaustion under the storm is the likely one), diagnose, fix, write `POSTMORTEM.md` linked to the fixing commit.

**Week 8 — evidence and close.** `SECURITY.md` completed: data-flow diagram, trust boundaries, STRIDE table, every threat mapped to a mitigation **and** a test, accepted risks (including the DBA-with-app-env-access risk and key rotation). Three ADRs minimum (§3.2 table + others). `AI-USAGE.md` per §1.7 with the hand-written list from §3.11. README rewritten to §1.5. Demo video, 2-3 minutes. **Tag `v1.0`.**

**Optional, after `v1.0`:** recruit an outside PR reviewer, or one merged upstream PR to a dependency (§7); an OWASP ZAP baseline scan with findings remediated.

### 3.10 Cut order

(1) Nursing vocabulary, (2) demo video, (3) `POSTMORTEM.md` (only if no genuine failure occurred), (4) rate limiting and security headers, (5) the k6 storm's latency table (**never** the exactly-once assertion).

**Never cut:** the canonical-encoding spec, the tamper tests (including the tail-deletion/head-anchor test), the denial suite, the Python verifier, the two-person constraint test, the `reconciliations` CHECK.

### 3.11 Risks, AI-USAGE hand-written list, riskiest tool

| Risk | Handling |
|---|---|
| Largest of the three projects; almost the entire stack is new | `v0.1` at week 2 and `v0.5` at week 5 mean resume value does not depend on finishing |
| Cryptography subtly wrong | Spec before code; store `signed_bytes`; five committed vectors; cross-language verification; the full 6-step `verify` procedure; library primitives only |
| Free-tier cold starts distort latency numbers | Warm the Neon compute 60s ahead of every storm/latency run; run the storm three times; disclose instance size and which run was cold |
| Postgres `numeric`/`bigint` handling in JS | Decided deliberately: `numeric` stays a string; `bigint` (OID 20) is parsed to `Number` (safe below 2^53); a test asserts this |
| Sole authorship vs. Stripe-style multi-person requirement | §11 unresolved decision (PR reviewer) |
| Demo instance with zero users | Accept and state it; optionally run a small pilot with former colleagues and tie two changelog entries to their feedback |

**AI-USAGE.md hand-written list (never generated):** `server/src/ledger/append.ts` (audit append + head update), `server/src/ledger/verify.ts`, `server/src/ledger/replay.ts`, `server/src/auth/authorize.ts` (preHandler + predicates), `db/001_init.sql`, `verifier/src/ledger_verify/chain.py`.

**Riskiest new tool:** Fastify + TypeScript strict on the write path. **Fallback date: end of week 2** — if the ledger core is not green by then, drop the React client from the `v0.5` scope (serve the audit view as server-rendered HTML) rather than slip `v0.1`.

### 3.12 Definition of done

- `verify` detects all tamper classes from the §3.3.2 test-operations table, including tail deletion, each naming the correct `first_bad_seq` and `class`.
- The Python verifier and the TypeScript server agree on a real export and on the five vectors, in CI.
- 30 denial cases pass, organized by class, counted from the fixture (`fixture.length === 30`).
- The storm test asserts exactly one row from 1000 concurrent same-key requests, with the result committed and no 429s.
- The live URL serves both screens with seeded per-role accounts; nightly reset keeps demo state coherent.
- `SECURITY.md` maps every identified threat to both a mitigation and a test, including the DBA-with-app-env-access accepted risk.
- The `reconciliations` CHECK exists, making "two-person issuance and lot reconciliation enforced by database constraints" a true statement as scoped in §3.3.4.

---

## 4. Project 3: FieldVoice

Sequenced third, after CustodyLedger `v1.0` (§11). Repository name: `fieldvoice`.

### 4.1 Goal and final deliverable

A Python pipeline converting spoken inspection readings into schema-validated records, whose center of gravity is **measurement, not the feature**: an evaluation harness with pre-registered hypotheses, confidence intervals, a baseline the model had to beat, and a held-out split the prompt never saw.

Final deliverable: public repo, four-command CLI, a committed and checksummed evaluation dataset, published result tables with intervals, and an offline mode running a self-quantized local model with a benchmark.

Headline README numbers: utterance count and speakers, numeric-token error rate with and without keyterm boosting (paired-bootstrap interval), extraction F1 versus a regex baseline on a frozen split, and local-model throughput and memory across quantization levels.

Motivating use case, stated as motivation only, never as deployed practice: readings taken gloved, hands occupied, at height, under hearing protection, often with no connectivity, transcribed hours later from paper — every transcription step is an opportunity to transpose a digit.

**Corrected claim boundary (fixes: "served fully offline" previously implied an end-to-end offline pipeline):** the offline mode covers **extraction only**; STT remains cloud-based in every mode. State this explicitly wherever the offline mode is described. Correct wording: "extraction served fully offline (STT remains cloud) with a zero-outbound-calls test."

### 4.2 Architecture

A library plus a thin CLI, with a swappable provider boundary at the speech and extraction layers so cloud and local paths are the same code path with different adapters.

```
fieldvoice/
  src/fieldvoice/
    audio/       loading, resampling, active-speech detection, noise mixing
    stt/         base.py (Protocol), deepgram.py, offline.py
    normalize/   spoken-number and unit normalization (hand-written, §4.9)
    extract/     base.py (Protocol), regex_baseline.py, llm.py, schema.py
    validate/    physical plausibility rules
    eval/        metrics, bootstrap, grid runner, report emitter
    serve/       FastAPI app (offline mode; subprocess-managed llama-server)
    cli.py
  data/
    utterances/  wav files (committed; ~75 MB, see §4.3.1), manifest.json
    locations.json          synthetic reference table for the validator (§4.3.6)
    noise/       git-ignored; fetch script + checksums
    LICENSE-DATA.md         CC BY 4.0 for recordings + manifest; consent template
  eval/
    preregistration/   committed before each grid run
    results/           committed result tables and plots
  tests/
  Dockerfile
```

Provider boundary:

```
audio --> SttProvider --transcript--> Extractor --> Reading (pydantic) --> Validator --> record | review
             |  deepgram                  |  regex_baseline
             +- offline (local, extraction only)  +- llm (cloud or local, same client shape)
```

Every stage is independently evaluable: the harness holds one stage fixed and varies another, and extraction is evaluated on *real STT output*, never on clean reference text.

### 4.3 Components

**4.3.1 Evaluation dataset (the most important deliverable, built first).**

Composition (fixes: original size estimate implied dividing 200 utterances among 3 speakers, which leaves ~20 held-out items per speaker — below the §1.9 n<20 floor for per-speaker reporting): **each of the 200 utterance scripts is read by all 3 speakers**, 600 files total, ~75 MB (200 x 3 x ~4 s x 16 kHz x 16-bit). Speakers: the developer plus two volunteers, written consent recorded in the repo.

Utterance categories with counts (commit before recording): plain reading 80, location-only 20, unit-switch 20, self-correction 30, implausible 25 (used by the validator evaluation, §4.3.6), indication-flag 25. Self-corrections follow the pattern "CML A-12, point three one two, no wait, point three two one" and are resolved by the **normalizer**, not the extractor, with the rule "last complete value wins" (included in the exhaustive normalizer test table).

Vocabulary is synthetic and generic, derived from public terminology, never from an employer's forms or nomenclature (§1.10). Recorded with any consumer recorder at 16 kHz mono WAV.

Three conditions: quiet, and two mixed-noise levels using **public environmental noise recordings (DEMAND, Zenodo)** — corrected description: DEMAND is domestic/nature/office/public/transportation/street recordings (kitchen, living room, café, station, bus, traffic, etc.); it contains **no plant or industrial machinery**, and must never be called "industrial noise" in any README or resume text. Use the loudest broadband environments: `DWASHING`, `TMETRO`, `PSTATION`, `STRAFFIC`, channel `ch01`, 16 kHz set. **Licensing:** the Zenodo record states two different licence strings (CC BY 4.0 in the rights field, CC BY-SA 3.0 in the description); treat the stricter, **CC BY-SA 3.0, as governing**. Never commit or publish mixed audio (only the clean recordings may ever be published or demoed). If a genuinely industrial noise source is wanted later, MIMII (machine sounds: pump, fan, valve, slider) is a candidate second source — **open verification if pursued: confirm its licence (believed CC BY-SA 4.0) on the Zenodo record before using it.**

Mixing at stated SNRs computed as RMS over active-speech segments (not the whole file); the mixing function is unit-tested by round-tripping a known-SNR construction.

`manifest.json`: per-utterance verbatim transcript, structured ground-truth record, SHA-256 per file, speaker, condition, split. **Split**: 70/30, stratified by speaker and utterance category, seed `20260901`, `split_hash = sha256` of the sorted `id:split` lines, asserted unchanged by a test. `data/manifest.schema.json` is committed. **Split assignment is frozen before any prompt work; the held-out split is never used during prompt development.**

**Cross-project scheduling note:** because two volunteer speakers are needed, write the utterance script and consent form, and book both recording sessions, during the **last week of the preceding project (CustodyLedger week 8)** — this is the permitted passive-time exception to the sequential-projects rule (§1.2/§6).

**4.3.2 Spoken-number normalization.** Deterministic, table-driven, hand-written (never AI-generated — §4.9), handling digit-sequence reading, decimal words, "oh"/"zero", fractional forms, unit words, location identifiers (`letter-dash-number`), and self-correction resolution ("last complete value wins"). Exhaustive unit-test table. Output formats fixed: readings always as `0.312` (leading zero, three decimals as spoken); location IDs uppercase `A-12`; units `in|mm`.

**4.3.3 Metrics.**

- **WER** via `jiwer`, **corpus-level** (total edits / total reference words), `jiwer.wer_standardize` as the transform on raw text, with the numeric normalizer applied afterward — both steps stated explicitly (this resolves the ambiguity between corpus-level and mean-per-utterance WER).
- **Numeric-token accuracy** (headline metric): numeric token = a normalized token matching `^[A-Z]+-\d+$` or `^\d+\.\d+$` or `^\d+$`. Align normalized reference and hypothesis via `jiwer.process_words(ref, hyp).alignments`; a reference numeric token counts correct only if aligned to an equal hypothesis token (substitution or deletion = error); denominator = reference numeric tokens.
- **Per-field extraction F1** with a per-field error taxonomy (decimal word confusion, location-ID normalization, unit mismatch, hallucinated field, dropped field). Decision table: a field is TP when predicted equals truth after normalization, FP when predicted non-null and unequal, FN when truth non-null and predicted null.
- All three metrics use the **paired bootstrap** (§1.9) for any boosting-on/off or condition comparison — never two independently-computed condition intervals compared by overlap.
- Minimum detectable difference stated in the pre-registration before running: with ~600 numeric tokens (200 utterances read by 3 speakers, quiet condition), a single condition's interval is approximately +/-4 percentage points; state this explicitly.

**4.3.4 Speech provider.** Deepgram streaming over WebSocket, keyterm boosting for the domain vocabulary. **Verified:** `keyterm` is supported on Nova-3 (mono/multilingual) and Flux, including streaming; `keywords` is the corresponding feature on Nova-2 and earlier; keyterms are limited to 500 tokens per request (recommended 20-50 terms).

**Fixed configuration (fixes: leaving `smart_format`/`numerals` unfixed would let Deepgram's own number formatting compete with the project's normalizer and confound the boosting comparison):**
```
model=nova-3 (and, separately, model=nova-2)
smart_format=false
numerals=false
punctuate=false
interim_results=true
endpointing=300
encoding=linear16
sample_rate=16000
channels=1
```
The normalizer is applied identically to references and hypotheses. Keyterm list = the vocabulary list committed with the utterance script **before recording** (never extracted from transcripts after the fact), <=50 terms. Unit test: the adapter emits `keyterm` for `nova-3` and `keywords` for `nova-2`, and refuses any other model/parameter combination.

Streaming is exercised file-paced (audio fed at wall-clock rate) so TTFT and interim-to-final latencies are meaningful. TTFT = first interim result minus first audio byte sent; final latency = last `is_final` result minus the `CloseStream` send time; both labeled "network + API, not microphone." A reconnect-and-backoff state machine is unit-tested against a **fake transport object**, not a mock server (building a protocol-accurate mock Deepgram server is explicitly out of scope — a known scope trap). One manually induced disconnect during a live run is recorded in the README as a qualitative check.

Grid runtime: 2 models x 2 boosting x 3 conditions = 12 conditions x 600 files x ~4 s file-paced ~ 8 h sequential; run at a concurrency of 5 sessions. **Open verification — before the week 3 grid run:** confirm Deepgram's concurrent streaming-connection limit for the account tier (check the account dashboard/docs); set grid concurrency = `min(5, verified limit)`.

Cost control: entire dataset ~15 minutes of audio per pass; a full grid run costs a few dollars. A hard spend cap counts seconds from the manifest before sending and refuses above a configured budget (unit-tested); the README publishes actual spend.

**4.3.5 Extraction.** Two extractors behind one `Extractor` Protocol:

1. **Regex baseline**, deterministic, built from the normalizer — it exists so the LLM has to earn its place; if the LLM does not beat it meaningfully, that is a publishable finding.
2. **LLM structured extraction**: a pydantic model as schema, JSON output, prompt and few-shot examples versioned in the repo with the prompt hash recorded in every result row.

Both evaluated on **real STT hypothesis transcripts**, never reference text — the measured F1 includes upstream error propagation, which is the honest number. Extraction is evaluated primarily on one canonical STT condition (`nova-3`, keyterm on, quiet) and secondarily on `snr_low`; state this in the pre-registration.

**4.3.6 Physical validation.** Rules independent of the model: reading below nominal minus tolerance, reading above nominal, implausible change versus the previous reading for the same location, unit mismatch, unknown location identifier. Violations route to review, never silently corrected.

**Reference data (fixes: the validator had no ground truth to check against, yet a resume bullet quoted a rejection rate):** `data/locations.json`, synthetic, `{location_id, nominal_in, tolerance_in, previous_reading_in, previous_date}`, generated by a committed script with stated parameters (§1.10). The 25 "implausible" utterances from §4.3.1 have `implausible: true` in the manifest with a truth reading that is implausible against this table. The validator's rejection rate on the labelled-implausible set, and its false-rejection rate on the plausible set, are both published with Wilson intervals (§1.9). **Do not publish a rejection-rate resume bullet without this labelled set.**

`confidence` (fixes: previously had no defined source): the minimum Deepgram per-word confidence over the words that produced the numeric tokens (`words[].confidence` in streaming results); for the regex baseline on text input, `1.0` if all fields parsed else `0.0`; for the LLM extractor, the same STT-derived value (LLM logprobs are not used). Threshold `0.90`, in config.

`--confirm` flag requires explicit reviewer sign-off before any record is written; **the README states clearly that high-confidence records (confidence >= 0.90) are written without confirmation when the flag is absent** — an earlier description overstated this as always requiring confirmation. "Written" = appended to `records.jsonl` at the path given by `--out`; below-threshold records go to `review.jsonl` with the reason.

**4.3.7 Pre-registered evaluation grid.** Before the main run, commit `eval/preregistration/<date>-<name>.md` with: factors and levels, primary metric, the **paired-bootstrap decision rule** (§1.9: interval excludes 0), minimum difference considered meaningful, minimum detectable difference at N (§4.3.3), the null sentence, and the git commit that will run. The grid runner emits a tidy per-utterance-x-condition result table plus aggregate tables with intervals, regenerated by one command from the committed manifest. **A null result is an expected, publishable outcome** — keyterm boosting may not move numeric accuracy measurably; quantization-level differences may be within noise; the pre-registration converts either into evidence of method rather than an embarrassment.

**4.3.8 Offline mode.** The differentiator for applied-AI target postings whose stated duties are profiling, benchmarking, and quantization on constrained devices.

- Convert Qwen2.5-1.5B-Instruct (Apache-2.0; **open verification at week 5 day 1: re-check the license has not changed**; if changed or incompatible, substitute the nearest Apache-2.0 or MIT small instruct model, <=2B params) from safetensors to F16 GGUF via llama.cpp's `convert_hf_to_gguf.py` (needs `torch`+`transformers` per `requirements-convert_hf_to_gguf.txt` — a multi-GB install; **budget 2 h in week 5** for this environment, built as a separate `uv venv` inside the llama.cpp tree, CPU-only torch). Quantize to **three plain levels only for `v1.0`: Q4_K_M, Q6_K, Q8_0** (imatrix variants are cut-order item 3, dropped pre-emptively per §4.10 — do not attempt "with and without imatrix" as originally scoped: **`Q8_0` does not use an importance matrix at all**, so "with/without imatrix at Q8_0" is a null by construction and must not appear as a planned comparison). The developer performs the conversion and quantization; pre-quantized downloads are not the claim and would not survive scrutiny.
- **Benchmark harness (fixes: `llama-bench` reports only mean+-SD tokens/s over `-r` repetitions — it reports neither TTFT nor peak memory, and cannot produce "median of 10"):** use `llama-server` (pinned release tag, built in WSL2: `cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j`) plus a Python client sending the fixed extraction prompt 10 times per configuration, recording `timings.prompt_ms`, `timings.predicted_per_second`, and TTFT from the first streamed token. Peak memory = `VmHWM` from `/proc/<pid>/status` after each run (Windows "peak working set" is the wrong vocabulary in WSL2). Report median and IQR, hardware and build flags stated (§1.6). Grid: `Q4_K_M`, `Q6_K`, `Q8_0`, plus F16 baseline.
- Report extraction-F1 retention per quantization level on the held-out split, with intervals — **evaluated in week 6, alongside the offline service** (moved from week 5 to make room for the build/convert/benchmark work).
- One genuine tuning experiment: thread-count sweep and prompt-cache reuse, before/after table, explanation of the binding constraint.
- FastAPI service exposing the offline extraction path, with a **socket-level** zero-outbound-calls test.

**Serving mechanism (fixes: the plan never specified how Python talks to the model, leaving the socket test's scope ambiguous):** `serve/` starts `llama-server --model <gguf> --port 8081 --host 127.0.0.1 --ctx-size 2048 --threads <n>` as a subprocess; the extractor calls its OpenAI-compatible `/v1/chat/completions` with `response_format: {type: "json_schema", json_schema: <pydantic schema>}` for grammar-constrained output. The cloud extractor uses the same client shape against the chosen provider's OpenAI-compatible endpoint (this single adapter shape resolves "LLM client per provider choice"). Zero-outbound test: `pytest-socket` with `--disable-socket --allow-hosts=127.0.0.1,::1` around the full request path, plus an assertion that `socket.getaddrinfo` is never called for a non-loopback host.

Local STT is **not** in scope; the offline mode covers extraction only (§4.1). Docker image is cut-order item 1.

**4.3.9 CLI.**

```
fieldvoice eval-stt      --grid eval/preregistration/<file>.md
fieldvoice extract       --input <wav|txt> [--provider deepgram|offline] [--confirm]
fieldvoice validate      --records <jsonl>
fieldvoice eval-extract  --split held_out --extractor regex|llm --quant <level>
fieldvoice serve         --offline
```
Recording uses any external recorder; there is no `record` command. Live microphone capture is a single documented demo script, not tested library surface.

### 4.4 Data models and interfaces

```python
class ReadingTruth(BaseModel):          # ground truth only, no prediction metadata
    location_id: str
    thickness: Decimal                  # Decimal, never float; unit-agnostic field name (fixes name/unit contradiction)
    unit: Literal["in", "mm"]
    indication: bool

class Reading(BaseModel):               # a prediction
    location_id: str
    thickness: Decimal
    unit: Literal["in", "mm"]
    indication: bool
    confidence: float                   # per §4.3.6
    source_transcript: str
    prompt_version: str | None

class Utterance(BaseModel):
    id: str; path: Path; sha256: str
    speaker: str; condition: Literal["quiet", "snr_high", "snr_low"]
    split: Literal["dev", "held_out"]
    truth_transcript: str; truth_reading: ReadingTruth
    implausible: bool                   # true for the 25 labelled-implausible utterances, §4.3.6

class SttProvider(Protocol):
    async def transcribe(self, audio: AudioSource, *, keyterms: list[str]) -> SttResult: ...

class Extractor(Protocol):
    def extract(self, transcript: str) -> ExtractionResult: ...
```

`SttResult` carries hypothesis text, word timings when available, latency marks (first byte, first interim, final). `ExtractionResult` carries the parsed `Reading` or a structured failure, plus token counts and wall time. Result rows are JSONL with full provenance: manifest hash, split hash, prompt hash, model id, quantization level, git commit.

### 4.5 Dependencies

`pydantic`, `httpx` + `websockets` (Deepgram streaming), `jiwer`, `numpy`, `soundfile`, `scipy` (resampling/filtering only), `fastapi`, `uvicorn`, `typer`, `matplotlib`, `python-dotenv`, `pytest-socket`. LLM client: one adapter shape used for both cloud and local (§4.3.8), provider chosen per §11. Dev: `pytest`, `pytest-asyncio`, `hypothesis`, `ruff`, `mypy`, `pip-audit`.

External, not Python packages: llama.cpp built in WSL2 (`llama-server`, `llama-quantize`, `llama-imatrix`, `convert_hf_to_gguf.py`), the DEMAND noise dataset (fetch script with checksums; not committed).

`mypy --strict` applies to the whole package (no DataFrame-heavy code here, unlike Flightcheck).

### 4.6 Testing

| Layer | Tool | Proves |
|---|---|---|
| Unit | pytest | Normalizer over the exhaustive spoken-number table, including self-correction resolution |
| Unit | pytest | Bootstrap against §1.9 closed-form vectors; WER against hand-computed examples |
| Unit | pytest | Noise mixing achieves the requested SNR on a constructed signal |
| Unit | pytest (fake transport) | Reconnect/backoff state machine transitions |
| Unit | pytest | Spend-cap refusal above budget |
| Unit | pytest | Model->parameter selection (`keyterm` for nova-3, `keywords` for nova-2; other combos refused) |
| Unit | pytest | Confidence routing at the 0.90 threshold |
| Unit | pytest | Every LLM output validates against the pydantic schema (grammar constraint holds) |
| Unit | pytest | `imatrix` calibration text (if pursued) contains no held-out utterance ids |
| Unit | pytest | Validator against the 25 labelled-implausible utterances, plus false-rejection rate on plausible ones |
| Contract | pytest | Manifest integrity: every file's SHA-256 matches; split hash unchanged |
| Integration | pytest | End-to-end on three committed fixture utterances with a recorded provider response |
| Privacy | pytest (`pytest-socket`) | Offline service makes zero outbound socket connections (loopback only) |
| Reproduction | manual | Every published table regenerates from the manifest with one command |

Provider responses for integration tests are recorded fixtures; CI never calls a paid API.

### 4.7 Phased implementation (7 weeks; re-budgeted)

**Week 1 — dataset and normalizer.** Utterance script covering the vocabulary and malformed/self-correction cases (script + consent form already drafted during CustodyLedger week 8, per §4.3.1's cross-project note). Record three speakers x 200 utterances = 600 files. Trim, resample, checksum, write the manifest, freeze the 70/30 stratified split (seed `20260901`). Implement the normalizer with its exhaustive test table. CI green. No model call this week, deliberately.

**Week 2 — metrics and noise.** WER (corpus-level) and numeric-token accuracy implementations, with alignment per §4.3.3. Percentile bootstrap with its closed-form unit test (§1.9). Active-speech RMS noise mixing with a round-trip test, using the corrected DEMAND environments (`DWASHING`, `TMETRO`, `PSTATION`, `STRAFFIC`, `ch01`, 16 kHz) and both licence strings recorded, CC BY-SA 3.0 treated as governing. Fetch script for DEMAND with checksums. First result: baseline numeric accuracy on quiet audio using a single default provider configuration.

**Week 3 — streaming provider, pre-registration, grid (re-scoped: reconnect/backoff and latency marks moved out).** Deepgram adapter with the fixed configuration (§4.3.4), keyterm list committed pre-recording, spend cap. Write and commit the pre-registration (§4.3.7) including the paired-bootstrap decision rule and the minimum-detectable-difference statement. Run the grid (verify the concurrency limit first, §4.3.4). Publish tables with paired-bootstrap intervals. **Tag `v0.1`** — this already answers the Deepgram application form's question with measured numbers, the earliest resume value in this project.

**Week 4 — extraction, validation, and the moved streaming work.** Regex baseline from the normalizer. Pydantic schema (`Reading`/`ReadingTruth` split, §4.4). LLM extractor with versioned prompts. Evaluate both on real STT hypotheses over the frozen held-out split (canonical condition per §4.3.5). Per-field error taxonomy. Physical-plausibility validator against `data/locations.json` and the 25 labelled-implausible utterances (§4.3.6), with `--confirm`. **Reconnect/backoff state machine with fake-transport tests and latency marks (moved from week 3, since they are not on the measurement path).** 80+ tests green. **Tag `v0.5`.**

**Week 5 — quantization (re-scoped: imatrix and F1-retention moved out).** Build `llama-server` in WSL2. Convert the chosen model to F16 GGUF (budget 2 h for the conversion environment). Quantize to **three plain levels only** (`Q4_K_M`, `Q6_K`, `Q8_0`) — no imatrix. Benchmark harness per §4.3.8 (median of 10, TTFT, `VmHWM`, hardware stated). **Riskiest new tool: the llama.cpp conversion path. Fallback date: week 5, day 4** — if F16 conversion does not produce a GGUF that `llama-cli` loads, ship cloud-only for `v1.0` and state offline mode as future work; do not substitute pre-quantized downloads and call it quantization work.

**Week 6 — offline service, tuning, and moved F1 retention.** FastAPI offline path (subprocess `llama-server`, OpenAI-compatible endpoint, `json_schema`-constrained output). Zero-outbound-calls socket-level test. Thread-count sweep and prompt-cache experiment with a before/after table and binding-constraint analysis. **Extraction-F1 retention per quantization level on the held-out split, with intervals (moved from week 5).** Dockerfile, built in CI (cut-order item 1 if time is short — the image itself, not the offline CLI path).

**Week 7 — publication and close.** README as an evaluation report per §1.5, with the corrected claim boundaries from §4.1 (offline = extraction only) and §4.3.6 (validator rejection rate only if the labelled set exists). Result plots. Demo script and a short recorded walkthrough. Three ADRs minimum. `AI-USAGE.md` per §1.7 with the hand-written list from §4.9. Draft the application-form answer from the published numbers. **Tag `v1.0`.**

**Post-`v1.0` stretches, not part of the estimate, not claimed before they exist:** a LangGraph-style agentic orchestration layer; a LoRA fine-tune of the extractor with a proper comparison against the prompted baseline.

### 4.8 Deployment and setup

No hosted deployment. `uv sync`; `DEEPGRAM_API_KEY` in `.env` for cloud paths; `make fetch-noise` for the noise dataset; WSL2 with a built `llama-server` and a converted model for offline paths (documented, with the exact conversion/quantization commands from week 5).

Dockerfile packages the offline path only (`python:3.12-slim` + a pinned `llama-server` release binary; the model file is mounted at `/models`, never baked in). CI builds the image and runs the zero-outbound test with a stub model; publishing the image is optional (cut-order item 1).

### 4.9 Risks, AI-USAGE hand-written list, riskiest tool

| Risk | Handling |
|---|---|
| Null results across the board (boosting/quantization within noise) | Pre-registration with the null sentence written in advance (§1.9); report it with the power/minimum-detectable-difference limitation stated. This is the single most likely outcome and the plan survives it |
| Two volunteer speakers may not materialize | Fall back to one speaker, state it, report per-condition (not per-speaker) results; do not fabricate speaker diversity — **decision deadline: end of week 1; decision rule: if fewer than 2 volunteers recorded by then, drop per-speaker reporting from every downstream table for this project** |
| Quantization differences need more than 60 held-out cases to detect | State the detectable effect size given N in the pre-registration before running |
| llama.cpp build/conversion fails in WSL2 | Week 5, day 4 fallback (§4.7): cloud-only `v1.0`, offline mode as future work |
| API cost overrun | Hard spend cap in the runner; published actual spend |
| Volunteer consent and voice data | Written consent committed; `data/LICENSE-DATA.md` (CC BY 4.0 for recordings/manifest, distinct from the code's MIT licence); speakers identified by pseudonym; no names/places in scripts |
| Scope creep into agents or fine-tuning | Both post-`v1.0`, not claimed until they exist |
| DEMAND mislabeled as industrial | Corrected in §4.3.1: "public environmental noise recordings"; CC BY-SA 3.0 treated as governing; never publish mixed audio |

**AI-USAGE.md hand-written list (never generated):** the spoken-number normalizer (`normalize/*.py`) — small, plausible-looking, subtly wrong if AI-written, and the foundation under every published number.

**Riskiest new tool:** llama.cpp conversion path. **Fallback date: week 5, day 4** (§4.7).

### 4.10 Cut order

(1) Docker image, (2) thread/prompt-cache tuning experiment, (3) imatrix variants (already excluded from the `v1.0` plan by default — never scheduled, not merely cut), (4) the third noise condition, (5) the offline FastAPI service (keep the CLI offline path and the benchmark).

**Never cut:** the checksummed manifest with a frozen split, the normalizer test table, paired-bootstrap intervals, the regex baseline, the pre-registration.

### 4.11 Definition of done

- Committed, checksummed, split-frozen dataset (600 files, 3 speakers) with consent records and `LICENSE-DATA.md`.
- Every published metric carries a paired-bootstrap interval (where the comparison is paired) and its N.
- The regex baseline and the LLM are both evaluated on real STT output over the same frozen held-out split.
- A pre-registration file exists for every published comparison, committed before the run, reported against its stated decision rule.
- Self-performed quantization at three plain levels (Q4_K_M, Q6_K, Q8_0) with a benchmark meeting §1.6 and F1 retention per level.
- The offline service passes the socket-level zero-outbound-calls test.
- Published actual API spend.
- README states the offline-mode boundary as "extraction only, STT remains cloud" and the validator rejection-rate claim only appears if backed by the labelled-implausible-set evaluation.

---

## 5. Cross-Project Dependencies and Interfaces

- **`stats-vectors.json`**: the three §1.9 test vectors (two Wilson, one bootstrap), committed verbatim to both Flightcheck and FieldVoice, each copy carrying a comment that it is a shared, independently-copied file — this is how drift between the two independent implementations becomes visible instead of silent. No other code is shared between the two repos.
- **FieldVoice's recording dependency on CustodyLedger's schedule**: FieldVoice needs 2 volunteer speakers with booked sessions before its week 1 starts. The utterance script and consent form are written, and both sessions are booked, during **CustodyLedger's week 8** (its final week) — the one scheduled passive-time exception to "exactly one project active" (§1.2).
- **CustodyLedger's hosting decision depends on Flightcheck's timeline**: the Render-vs-Fly.io check (§11) is performed during **Flightcheck's final week (week 7)**, so the decision is ready before CustodyLedger's week 1 deploy step.
- **GitHub account/profile pinning** applies once, before Flightcheck's `v0.1` tag (the first tag of the first project), and is not repeated per project.
- **The outside-PR-reviewer decision (§11)** is cross-cutting: if resolved "yes," every project (starting with Flightcheck week 2) opens its first reviewed PR by that project's week 2 and tracks toward 5 reviewed PRs as a `v1.0` criterion; if "no," no project schedules it and application materials state sole authorship plainly.
- **No shared runtime, database, or service exists between the three projects.** Each is a fully independent repository; the only artifacts crossing project boundaries are the schedule/decision dependencies above and the copied `stats-vectors.json`.

---

## 6. Implementation Order

Sequential, one project active at a time (§1.2). Total: 22 active weeks plus a 1-week learning spike before Flightcheck.

| Order | Project | Weeks | Gate to start |
|---|---|---|---|
| 1 | Flightcheck (`px4-reqcheck`) | week 0 + 7 | See pre-start checks below |
| 2 | CustodyLedger | 8 | Flightcheck `v1.0` tagged |
| 3 | FieldVoice | 7 | CustodyLedger `v1.0` tagged |

Swap rule: swap order 2 and 3 if applied-AI-postings (Deepgram, HP IQ AML) become the priority. Never overlap two projects; the only permitted exception is passive unattended time (a corpus download, a long fuzz run, a benchmark sweep, or FieldVoice's recording prep during CustodyLedger's week 8).

**Resume update points:** each project's `v0.1` tag is the moment its repository URL goes onto the resume and into application GitHub fields — do not wait for `v1.0`.

### Pre-start checks that gate ordering (who, when)

| Check | Gates | When | Who |
|---|---|---|---|
| Zipline requisitions still open (decides whether Flightcheck or CustodyLedger goes first) | Project order | The day before the first scaffold commit | Developer; result recorded in this document's revision history or a dated note alongside it |
| PR-reviewer decision (§11) | Whether every project schedules PR-review milestones | Before Flightcheck week 1 starts | Developer |
| Flightcheck §2.3.1 corpus-source verification (dbinfo fields, licensing, `mav_type` values, parameter names) | Flightcheck week 1 start | Flightcheck week 0 | Developer |
| CustodyLedger hosting decision (Render vs Fly.io, §11) | CustodyLedger week 1 deploy step | Flightcheck's final week (week 7) | Developer |
| FieldVoice utterance script, consent forms, recording sessions booked | FieldVoice week 1 start | CustodyLedger's final week (week 8) | Developer + 2 volunteers |
| FieldVoice Deepgram concurrency-limit check | FieldVoice week 3 grid run | FieldVoice week 3, before the grid | Developer |
| Zod4/`@hookform/resolvers`5 pairing check | CustodyLedger week 1 | CustodyLedger week 1 setup | Developer |

---

## 7. Acceptance Criteria (per project, testable)

**Flightcheck:** (1) `make all` from a clean environment (§9) produces the traceability matrix and report without manual steps. (2) All 7 requirements return one of `pass|fail|not_evaluable` for every ingested log, with `not_evaluable` causes drawn only from the fixed taxonomy (§2.3.3). (3) Tier-1 golden regression is green in CI on every push; tier-2 golden regression is green on `workflow_dispatch`. (4) The C++ checker's verdict set equals the Python verdict set on 100% of processed logs, asserted in CI; `descent_rate_pre_land_p95` agrees within absolute 1e-6 between the two independent computations. (5) Every published rate has a Wilson interval and a stated denominator; any bucket with n<20 is marked insufficient, never plotted as a point. (6) No sentinel/disabled parameter value (`GF_MAX_HOR_DIST=0`, `BAT1_N_CELLS=0`, `COM_DISARM_LAND<=0`) ever produces a `pass` or `fail` verdict — each is a passing unit test asserting `not_evaluable`/`param_disabled`.

**CustodyLedger:** (1) Each row of the §3.3.2 tamper-test table passes with the exact stated `first_bad_seq`/`class`, including tail deletion. (2) 1000 concurrent same-idempotency-key requests against the deployed instance produce exactly 1 row and zero HTTP 429s. (3) The 30-row denial-suite fixture passes in full, and the README's "30" is asserted equal to `fixture.length` by a test. (4) The Python verifier and the TypeScript server independently agree on a real seeded database's export and on all 5 committed vectors, in CI. (5) The live URL is reachable over HTTPS and serves both screens with the 5 seeded per-role accounts working. (6) `SECURITY.md`'s threat table has zero rows without both a mitigation and a linked test.

**FieldVoice:** (1) `manifest.json`'s split hash test passes (split has not changed since freeze). (2) The paired-bootstrap interval for keyterm boosting on numeric-token accuracy is computed and published whether or not it excludes 0, with the pre-registration's null sentence used verbatim if it does not. (3) Regex-baseline and LLM extraction F1 are both computed on real STT hypotheses (never reference text) over the identical frozen held-out split. (4) The offline service's zero-outbound-calls test passes at the socket level (not a mocked client). (5) Benchmark table reports median-of-10 throughput, TTFT, and `VmHWM` for `Q4_K_M`, `Q6_K`, `Q8_0`, and F16, each with hardware and build flags stated. (6) Actual Deepgram API spend is published and is less than the configured hard cap.

---

## 8. Testing Requirements

Per-project test tables are in §2.6 (Flightcheck), §3.8 (CustodyLedger), §4.6 (FieldVoice) — each already includes every test the technical review added, not only the tests in the original plan drafts. Shared requirements across all three:

- No number is published without a test proving the function that computed its interval matches §1.9's vectors.
- Every "claim that would be false if the code shipped as originally drafted" (§10) has at least one test that fails if the claim is false.
- CI (§1.8) is green on every push and PR to `main`; branch protection requires it.
- Coverage floors are regression floors, not optimization targets, and are stated as literal numbers per project (CustodyLedger: 60% lines on `server/src`).

---

## 9. Environment and Setup Requirements

- **All development in all three repos happens inside WSL2 Ubuntu 24.04**, via VS Code Remote-WSL (§1.3). Do not develop Node or run `make`/Semgrep-dependent workflows from the Windows host directly.
- Node 22 LTS via WSL `nvm` (CustodyLedger only).
- Python 3.12 via `uv`, with a committed `uv.lock`, in Flightcheck, FieldVoice, and CustodyLedger's `verifier/`.
- Postgres 16 installed natively in WSL2 (`sudo apt install postgresql-16`) for CustodyLedger local dev/test; Neon `main` branch is the deployed instance only.
- Docker is used only for FieldVoice's Dockerfile, built by CI; not required locally in any project.
- llama.cpp (`llama-server`, `llama-quantize`, `llama-imatrix`, `convert_hf_to_gguf.py`) built from source in WSL2, for FieldVoice week 5 only.
- GitHub Actions, `ubuntu-latest`, free tier, for all CI in all three repos.
- **Reproduction-check method (fixes: the original plans assumed a "second machine" that does not exist — there is one laptop):** for any plan step that says "clean clone on a second machine," use **either** (a) a freshly imported WSL2 distro (`wsl --import` a clean Ubuntu 24.04 image with no dotfiles) **or** (b) a manually triggered GitHub Actions `workflow_dispatch` workflow that clones, runs the documented commands, and uploads the produced numbers as an artifact. State which method was used in each README.
- Cost ceiling: under $10/month total across all three projects (§1.3).

---

## 10. Constraints and Invariants

### 10.1 Claims that must never be made (the corrected wording is the only wording that may appear anywhere — README, `SECURITY.md`, ADRs, resume, application forms)

| Never say | Say instead |
|---|---|
| "Editing, deleting, or reordering **any** audit record fails `verify`" without qualification | "`verify` detects mutation of any tracked column, deletion anywhere in the chain including the most recent record, and reordering, via the 6-step decision procedure and the head-anchor check (§3.3.2)" |
| "Two-person issuance **and lot reconciliation** enforced by database constraints" as if role/pending-state checks were also constraints | "The distinctness and completeness of the two-person rule, and the balanced/unbalanced integrity of reconciliation, are schema constraints; the approver's role and issuance-pending state are checked in application code and covered by the denial suite" (§3.3.4) |
| "Thresholds derived from each vehicle's own configured parameters" using `BAT_N_CELLS`/`BAT_V_EMPTY` without the `BAT1_` alias, or `GF_MAX_HOR_DIST` without the disabled-sentinel guard | State the aliased parameter names actually used (§2.3.3) and that every sentinel value is guarded and tested |
| "DEMAND industrial-noise dataset" | "Public environmental noise recordings (DEMAND)" — never "industrial" (§4.3.1) |
| "Served fully offline" (FieldVoice) | "Extraction served fully offline (STT remains cloud) with a zero-outbound-calls test" |
| "C++ checker independently agrees with the Python evaluator" unqualified | "Re-derives every verdict and independently recomputes one metric (`descent_rate_pre_land_p95`) from raw samples; the 100% agreement figure is by construction for the verdict-recompute step and independently meaningful only for that one metric" (§2.3.7) |
| "Root-cause memo on a public failsafe event" before the memo's subject is confirmed | "Requirement violation in a public log," upgraded to "public failsafe event" only once `vehicle_status.failsafe` is confirmed to transition in the chosen log |
| "Validator rejected N% of implausible records" without the labelled-implausible dataset | Only publish this once `data/locations.json` and the 25 labelled-implausible utterances exist and back the number (§4.3.6) |
| "Unique constraint on actor, key, and body hash" (CustodyLedger idempotency) | "Unique constraint on `(actor, key)`; body hash compared on conflict" (§3.3.3) |
| Any description of NDT inspection, QA/QC, or regulatory compliance work as software testing, software validation, or security engineering | Domain knowledge explains design rationale only, never claimed as software experience (§1.10) |
| Any rule from a public standard described as compliance with API 570/510, ATF, OSHA, or NFPA | "A simplified public convention" (§1.10) |

### 10.2 Other invariants

- No employer data (site names, procedures, forms, vocabularies, audio, readings) in any of the three projects, ever (§1.2).
- Quantities are `numeric`/`Decimal`, never float, in both CustodyLedger and FieldVoice; each has a dedicated test asserting a float is rejected.
- CustodyLedger's `audit_records` table has no `UPDATE`/`DELETE` grant for the application role.
- FieldVoice's held-out split is frozen before prompt work begins and is never touched during development; a test asserts the split hash is unchanged.
- Flightcheck never redistributes raw PX4 logs; only manifests, metadata, and derived aggregates are published.
- FieldVoice never publishes or commits noise-mixed audio (CC BY-SA 3.0 ShareAlike risk); only clean recordings may be published or demoed.
- **README and resume text for all three projects are written from this document only** (§1.11) — not from any other source, prior draft, or external commentary.
- Every published number carries the full measurement-contract attribute set (§1.6); a number missing any attribute is deleted, not published with a caveat.
- Every comparison of two conditions measured on the same items uses the paired-bootstrap rule (§1.9); independent per-condition intervals compared by overlap are never used as a decision procedure.

---

## 11. Unresolved Decisions

| Decision | Deadline | Decision rule |
|---|---|---|
| **Outside PR reviewer vs. sole authorship** (affects the "multi-person feedback loop" requirement some target postings state) | Before Flightcheck week 1 starts | If pursuing: open the first reviewed PR (a classmate or a developer-community reviewer) by week 2 of *each* project; treat 5 total reviewed PRs across the three projects as a `v1.0` criterion. If not pursuing: state sole authorship plainly in any application material referencing a multi-person requirement; do not claim an outside review that did not happen. |
| **CustodyLedger hosting provider (Render vs. Fly.io)** | Last week of Flightcheck (week 7) | Check current Render Starter price, Fly.io shared-cpu-1x price, and Fly's minimum-billing policy. If the k6 latency table (§3.9 week 7) is to be published: the host must not cold-start under load — this rules out Render's free tier; use Fly.io's smallest paid instance or Render's paid Starter, whichever is cheaper at check time. If the latency table is cut (cut-order item 5, §3.10): Render's free tier is acceptable, and the README must disclose "free instance, sleeps after 15 min idle." |
| **Flightcheck final corpus N** (depends on download bandwidth/disk, gated by the 6 s per-request delay) | End of Flightcheck week 1 | Ingest as many logs as complete within the ~10 h weekly budget during week 1, minimum 20 (the rebudgeted week-1 target); widen toward several hundred in week 6 only if the alias layer and bandwidth allow it. |
| **FieldVoice cloud LLM provider for extraction** | FieldVoice week 4 start | Choose based on cost and structured-output (`json_schema`) support at check time; either is acceptable since the prompt/schema and client adapter shape (§4.3.8) are portable. Document the choice in an ADR. |
| **FieldVoice offline model** | FieldVoice week 5, day 1 | Qwen2.5-1.5B-Instruct is the reference choice (Apache-2.0). Re-verify the licence has not changed; if changed or incompatible, substitute the nearest available Apache-2.0 or MIT instruct model at <=2B parameters and record the substitution in an ADR. |
| **FieldVoice per-speaker reporting** | FieldVoice week 1 end | If both volunteers recorded, report per-speaker results in addition to per-condition. If fewer than 2 volunteers recorded, drop per-speaker reporting entirely from every downstream table and state the fallback plainly — never fabricate speaker diversity. |
| **FieldVoice Deepgram concurrent-streaming-connection limit** | FieldVoice week 3, before the grid run | Check the account dashboard/docs at the current plan tier; set grid concurrency to `min(5, verified limit)`. |
| **CustodyLedger `GET /admin/export` streaming** | Deferred; revisit only if the audit table exceeds ~50k rows | Not needed for `v1.0` seed data volume; implement streaming only if this threshold is crossed. |
| **CustodyLedger nursing vocabulary inclusion** | CustodyLedger week 6 checkpoint | It is cut-order item 1 (§3.10); include only if week 7 is on schedule at the week 6 checkpoint. |
| **FieldVoice MIMII as a second (genuinely industrial) noise source** | Only if pursued, before first use | Verify MIMII's licence (believed CC BY-SA 4.0) on its Zenodo record before use; this is optional future work, not part of the `v1.0` scope in §4. |
