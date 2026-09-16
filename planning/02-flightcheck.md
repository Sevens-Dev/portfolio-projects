# Project 2: Flightcheck

Shared context in `00-shared.md` (S1-S14). Repository name: `flightcheck`.

## 1. Goal and final deliverable

A reproducible validation pipeline that ingests public PX4 flight logs, evaluates every flight against machine-readable system-level requirements whose thresholds are **derived from each vehicle's own configured parameters**, and emits a requirement-to-log traceability matrix with explicit coverage accounting plus a generated validation report.

The parameter-derived threshold design is the core idea and the answer to the strongest objection against this project. A verdict of FAIL against a threshold the developer invented for a stranger's hobby flight is meaningless. A verdict against the threshold that flight's own operator configured has a spec owner.

Final deliverable: public repo, one-command reproduction (`make all`) from raw logs to report, a static published report page, and a C++17 checker that independently agrees with the Python evaluator.

Headline README numbers: logs ingested, telemetry rows, ingest throughput, requirement count, coverage (evaluable vs not-evaluable per requirement), and Python-vs-C++ agreement (must be 100%).

## 2. Existing state

Greenfield. The developer has Python but none of the scientific stack, no DuckDB, no CMake/GoogleTest, no experience with flight data. See S2. This project carries the highest tool-learning load of the three, which is why it has a dedicated week 0.

## 3. Required architecture

A CLI-driven batch pipeline with a columnar analytical store. No service, no database server, no UI framework.

```
flightcheck/
  src/flightcheck/
    corpus/      log discovery, download, filtering, manifest
    ingest/      pyulog → normalized Parquet; data-quality checks
    params/      parameter extraction and threshold resolution
    metrics/     per-flight derived metrics
    requirements/ YAML loader, evaluator, traceability matrix
    report/      Jinja2 HTML + Matplotlib figures
    stats/       Wilson intervals, bootstrap, slicing (see S9)
    model/       scikit-learn baseline (week 6)
    cli.py       Typer entry points
  cpp/           C++17 checker: CMake + GoogleTest
  requirements/  requirements.yaml (the spec under test)
  tests/
  golden/        pinned log ids + expected metric/verdict outputs
  data/          git-ignored; corpus lives here
  Makefile
```

Data flow:

```
PX4 public log index ──filter──▶ corpus manifest (committed)
        │
        ▼ download (git-ignored)
    .ulg files ──pyulog──▶ Parquet (partitioned by log_id) ──▶ DuckDB views
        │                                                          │
        ├──parameters──▶ resolved thresholds ──┐                   │
        │                                      ▼                   ▼
        └──────────────────────────────▶ requirement evaluator ◀── metrics
                                               │
                        ┌──────────────────────┼──────────────────────┐
                        ▼                      ▼                      ▼
              traceability matrix       HTML report            C++ checker
                                                              (independent recompute)
```

**Storage choice:** Parquet on disk, queried through DuckDB. Rationale for the ADR: the working set is tens of millions of rows on a laptop; DuckDB reads Parquet directly with no server, and the columnar layout matches the access pattern (a few signals across many flights). A row-store alternative (SQLite) is benchmarked once in week 2 and the result is published, which turns a design choice into a measurement.

## 4. Major components

### 4.1 Corpus

Source: the public PX4 Flight Review log database. Logs are downloadable by UUID; a public index of logs with metadata is available from the same service.

**Verification required before week 1 commits to this** (flagged in §10): confirm the current index endpoint and download URL shape, confirm the licensing terms permit programmatic download and derived publication, and confirm that the metadata fields used for filtering (airframe type, flight mode, duration, firmware version) are present in the index rather than only inside each log.

Filter to a homogeneous slice:
- vehicle type: multicopter,
- contains a Mission-mode segment,
- duration between 60 s and 20 min,
- one firmware minor series for the first 50 logs; widened later with a topic-alias layer.

`corpus/manifest.json` commits the selected log UUIDs, their metadata, and a SHA-256 for each downloaded file. The raw logs are **not** committed. `make corpus` reproduces the download from the manifest. The manifest is what makes the project reproducible without redistributing data.

### 4.2 Ingest

`pyulog` parses `.ulg` into per-topic tables. Normalize the subset of topics actually used into Parquet, one directory per log.

Data-quality checks run during ingest and **report rather than drop**: timestamp monotonicity and gaps, duplicate timestamps, out-of-range values per signal, missing required topics, unit sanity. Each check writes counts to a per-log quality record; an ingest that silently discarded rows would invalidate every downstream count.

Topic and field names change across PX4 versions. An alias layer maps logical signal names to (topic, field) candidates per firmware range, resolved at ingest with the resolution recorded. Unresolvable signals mark the affected requirements not-evaluable for that log rather than failing the ingest.

### 4.3 Parameters and threshold resolution

Every ULog embeds the vehicle's full parameter set. Extract it into `parameters(log_id, name, value)`.

Threshold resolution turns a requirement's symbolic threshold into a concrete number per log:

```yaml
- id: REQ-BATT-002
  title: Battery voltage at disarm stays above the configured low threshold
  rationale: Landing below the operator's own low-battery threshold indicates
             the reserve was insufficient for the mission as flown.
  metric: battery_voltage_at_disarm
  threshold:
    expr: "BAT_N_CELLS * BAT_V_EMPTY"     # resolved from this log's parameters
  comparator: ">="
  requires_params: [BAT_N_CELLS, BAT_V_EMPTY]
  requires_signals: [battery_voltage, vehicle_status]
```

If a required parameter or signal is absent, the requirement is `not_evaluable` for that log, with the reason recorded. Coverage reporting depends on this distinction and it must never be collapsed into a pass or a fail.

Target 6-8 requirements, each evaluable from default-profile topics. Candidates: altitude hold error during mission cruise, battery voltage at disarm, horizontal distance from home versus the configured geofence, descent rate before disarm versus the land-speed parameter, GPS fix quality floor during mission, vibration RMS versus a fixed advisory level (the one requirement with a non-parameter threshold, documented as coming from public PX4 guidance).

### 4.4 Metrics

Five to eight per-flight scalars, each a pure function of the normalized signals, each unit-tested against a synthetic input with a hand-computed answer.

Examples: `altitude_error_rms_cruise`, `battery_voltage_at_disarm`, `vibration_rms_z` (FFT band power over a stated band), `gps_min_fix_type` / `gps_max_eph`, `descent_rate_pre_disarm_p95`, `max_distance_from_home`.

Signal processing is limited to what can be defended: a Butterworth low-pass with stated cutoff and order, an FFT with a stated window, percentiles over a stated time slice. No estimator internals, no controller analysis, no claims about navigation or control performance. The README states that the project validates against configured limits and does not analyze control design.

### 4.5 Requirement evaluator and traceability

For each (log, requirement): resolve thresholds, compute the metric, emit `pass | fail | not_evaluable` with the concrete threshold value, the metric value, and the reason when not evaluable.

The traceability matrix is the primary artifact: rows are requirements, columns are aggregate counts (`evaluable`, `pass`, `fail`, `not_evaluable` broken down by cause), plus a drill-down per log. Coverage gaps are printed prominently. A requirement evaluable on 4 of 300 logs is a finding about the corpus, and the report says so.

### 4.6 Golden-output regression

A pinned set of ~20 log UUIDs with committed expected outputs (metrics to a stated tolerance, verdicts exactly). CI re-runs them and fails on any change. This is what makes the pipeline a tested system rather than a script, and it is the artifact that most directly matches validation-team practice.

Float comparison uses explicit absolute and relative tolerances, declared per metric.

### 4.7 C++17 checker

`cpp/` builds a standalone binary that reads the exported per-flight metrics and resolved thresholds (Parquet is avoided here; the exporter writes a simple committed CSV/JSON contract) and recomputes the pass/fail verdicts, plus one metric end-to-end from raw samples so the binary is doing real work rather than re-comparing two numbers.

**Design note addressing a review objection:** a checker that only re-applies `value > threshold` is padding. The C++ side must own at least one metric computed from raw sample arrays (the descent-rate percentile is a good candidate: array traversal, windowing, and a selection algorithm, all hand-written), so the agreement test compares two independent computations.

CMake, GoogleTest via FetchContent, `-Wall -Wextra -Werror`, built in CI on `ubuntu-latest`. A timing comparison against the Python path is published under S6. Sanitizers are **not** claimed; the build flags are what they are.

### 4.8 Statistical slicing

Violation rate by bucket (altitude band, flight duration band, solar elevation computed from the log's GPS UTC time and position) with Wilson intervals and denominators shown (S9). Solar elevation is computed from a closed-form solar position formula, unit-tested against published almanac values for a few known times and places.

This section exists because one target posting's application form asks how the candidate would investigate false positives varying by condition. Answering it with executed code and intervals beats answering it with a paragraph.

### 4.9 Model baseline (week 6, bounded)

One scikit-learn logistic regression predicting whether a flight triggered a failsafe, using **only features computable before the failsafe event**. Target leakage is the named risk: whole-flight aggregates like battery voltage at disarm directly encode the low-battery failsafe they would predict. Features are computed over a window ending before the first failsafe transition, and the leakage analysis is written up whether or not it changes the result.

`GroupKFold` grouped by vehicle (multiple logs share airframes). Majority-class baseline reported alongside. A calibration curve and an explicit "what this must not be used for" section. One model, one feature set, no tuning loop.

### 4.10 Report and publication

Jinja2 HTML: scope, corpus provenance, method, requirement-by-requirement results with figures, coverage gaps, limitations. Matplotlib static figures committed. Published to GitHub Pages as a static artifact built from a committed DuckDB/Parquet summary, never re-ingesting at page-build time.

One root-cause memo (~2 pages): a real public log where a requirement failed, with timeline, hypothesis chain, the signals that confirmed it, the cause, and the regression test now guarding it in `golden/`.

## 5. Data models and interfaces

### 5.1 Normalized tables (Parquet, one directory per log)

```
samples_<signal>(log_id, t_us, value)          long form, one file per signal
events(log_id, t_us, type, subtype, detail)    mode changes, arming, failsafe
parameters(log_id, name, value_num, value_str)
quality(log_id, check, count, detail)
logmeta(log_id, airframe, sw_version, duration_s, start_utc, lat0, lon0)
```

### 5.2 Derived tables

```
metrics(log_id, metric, value, unit, window_start_us, window_end_us)
thresholds(log_id, req_id, resolved_value, expr, source_params jsonb)
verdicts(log_id, req_id, status, metric_value, threshold_value, reason)
```

### 5.3 CLI

```
flightcheck corpus build   --filter mission-multicopter --limit N
flightcheck ingest         --manifest corpus/manifest.json
flightcheck metrics
flightcheck validate       --requirements requirements/requirements.yaml
flightcheck matrix         --out reports/traceability.html
flightcheck report         --out reports/index.html
flightcheck slice          --by altitude_band,duration_band,solar_elevation
flightcheck model train    --target failsafe
flightcheck export-checks  --out export/checks.json   # feeds the C++ binary
```

`make all` runs the chain from manifest to report.

### 5.4 C++ interface contract

`export/checks.json` schema is committed and versioned. The C++ binary reads it, recomputes, and writes `export/cpp_verdicts.json`. A Python test asserts set-equality of verdicts and per-metric agreement within declared tolerance.

## 6. Dependencies

Python: `pyulog`, `pandas`, `numpy`, `scipy`, `duckdb`, `pyarrow`, `pyyaml`, `jinja2`, `matplotlib`, `typer`, `scikit-learn`, `requests`. Dev: `pytest`, `hypothesis`, `ruff`, `mypy`.
C++: CMake ≥ 3.20, GoogleTest (FetchContent), nlohmann/json (FetchContent).
Managed by `uv` with a committed lock.

`mypy` is scoped to the modules that are not Pandas-heavy (`requirements/`, `params/`, `stats/`, `cli`). Running `mypy` over DataFrame code is a known time sink and is explicitly out of scope; `ruff` covers the rest.

## 7. Implementation phases

**Week 0 — learning only, no deliverable.**
Five logs. `pyulog` → Pandas → one metric → one DuckDB query → one plot. Nothing committed to `main` except a scratch notebook converted to a script. Purpose is to discover the shape of ULog data before committing to an architecture. **Also in week 0:** verify the corpus source assumptions in §4.1 and record the findings in an ADR. If the public log index is unusable, the fallback is a smaller hand-curated corpus and the plan continues with reduced N.

**Week 1 — corpus and ingest.**
Manifest builder with filters. Download with checksums. `pyulog` ingest to Parquet for ~50 logs, one firmware series. Data-quality checks under pytest. First ingest throughput number (S6). Typer CLI skeleton. CI green with ruff + pytest.

**Week 2 — metrics and SQL.**
Five metrics with synthetic-input unit tests. Ten analytical SQL queries with tests. The DuckDB-vs-SQLite storage comparison, written up with a hypothesis stated first (S9). Static figures.

**Week 3 — requirements, the resume-critical week.**
`requirements.yaml` with 6-8 parameter-derived requirements. Threshold resolver. Evaluator with the three-valued verdict. Traceability matrix with coverage accounting. Jinja2 report. **Tag `v0.1`** — the repo link goes on applications now. This tag must contain the traceability matrix; it is the artifact the target postings' forms ask about.

**Week 4 — C++ checker.**
CMake + GoogleTest scaffold, CI job, the export contract, one metric computed from raw arrays in C++, the agreement test, the timing table. Budget the full week: this is a first C++ project with a new build system.

**Week 5 — regression suite and scale.**
Golden outputs over ~20 pinned logs with declared tolerances, wired into CI. Widen the corpus to several hundred logs across firmware versions via the alias layer; record what the widening broke. Root-cause memo on a real failing log, ending in a golden test. **Tag `v0.5`.**

**Week 6 — statistics and model.**
Wilson-interval slicing by altitude, duration, and solar elevation. Solar position function with almanac tests. Logistic-regression baseline with GroupKFold, majority baseline, calibration, leakage write-up.

**Week 7 — publication and close.**
GitHub Pages report. README as a validation report per S5. `make all` verified from a clean clone on a second machine. ADRs, `AI-USAGE.md`. **Tag `v1.0`.**

**Post-v1.0 only, time-boxed to four evenings:** a PX4 SITL feasibility spike. Can SITL run headless in WSL2, and can a script fly a short mission and produce a ULog the existing pipeline ingests? Kill-or-keep decision on evening four. If kept, it adds a scenario axis and a same-mission repeat-run regression comparison. If killed, SITL is listed as future work and no claim about simulation or scenario design is made anywhere.

## 8. Testing and validation

| Layer | Tool | What it must prove |
| --- | --- | --- |
| Unit | pytest | Every metric against a synthetic signal with a hand-computed answer |
| Unit | pytest | Threshold resolution, including the not-evaluable paths |
| Property | Hypothesis | Verdicts are total (every pair yields exactly one of three states); resolution is deterministic |
| Statistical | pytest | Wilson and bootstrap implementations against published/closed-form values; solar position against almanac values |
| Golden | pytest + CI | Pinned logs produce byte-stable verdicts and in-tolerance metrics |
| Cross-language | pytest + ctest | C++ and Python agree on all logs |
| Data quality | pytest | Injected defects (gaps, duplicates, out-of-range) are counted and reported, not dropped |
| Reproduction | manual, once | `make all` from a clean clone on a second machine |

## 9. Deployment and setup

No server. Distribution is the repo plus GitHub Pages.

Setup: `uv sync`, `make corpus`, `make all`. C++: `cmake -S cpp -B cpp/build && cmake --build cpp/build && ctest --test-dir cpp/build`.

CI does not download the corpus. It runs unit tests, the golden regression against a small committed fixture set (three tiny synthetic ULog-derived Parquet fixtures, committed, under 5 MB total), and the C++ build and tests. The heavy path is a manually triggered workflow.

## 10. Known risks and unresolved decisions

| Risk | Handling |
| --- | --- |
| **Corpus source assumptions unverified** | Week 0 gate (§4.1). Fallback: hand-curated smaller corpus. Do not start week 1 before this is settled |
| Log licensing for derived publication | Verify in week 0; publish only derived aggregates and metadata, never redistribute raw logs |
| Heterogeneous logs across firmware versions break field access | Alias layer; start with one firmware series; widening is deliberately in week 5, after the pipeline is stable |
| Many requirements turn out not-evaluable on public data | This is a *result*, not a failure, and coverage reporting is designed to surface it. If coverage is very low, the report leads with that finding |
| "This is PX4 Flight Review re-implemented" | It is not: Flight Review renders one log; this evaluates a corpus against requirements with traceability and regression. The README says this in the "What this is not" section |
| C++ week overruns | The checker is the compiled-language evidence, so it is not cut. If week 4 overruns, week 6 (model) is sacrificed instead |
| Model produces a leaky or noise-level result | Pre-registered (S9); report the leakage analysis and the null result |
| Solar elevation slicing looks arbitrary | It is motivated by a specific application-form question; the README states the motivation |

**Unresolved:** exact corpus source endpoints (week 0); final N for the corpus (depends on download bandwidth and disk); whether the vibration requirement's fixed advisory threshold belongs in a parameter-derived requirement set at all, or should be labeled as a different class of requirement (lean toward labeling it a separate class).

## 11. Cut order

(1) Model baseline, (2) solar-elevation slice, (3) corpus widening beyond one firmware series, (4) GitHub Pages publication (report still generated locally and committed).

Never cut: parameter-derived thresholds, three-valued verdicts with coverage accounting, the traceability matrix, the golden regression suite, the C++ checker.

## 12. Definition of done

S12, plus:
- 6-8 requirements evaluated across the corpus with per-requirement coverage published, including not-evaluable causes.
- Traceability matrix and HTML report generated by `make all` from a clean clone.
- Golden regression over pinned logs green in CI.
- C++ checker agrees with Python on 100% of logs, with the agreement asserted in CI and a timing table published.
- One root-cause memo whose regression test is in `golden/`.
- Every published rate carries a Wilson interval and its denominator.
