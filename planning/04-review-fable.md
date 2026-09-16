# Technical review of the three implementation plans

Reviewer: Fable 5.1. Date: 2026-09-16.
Inputs: `00-shared.md`, `01-custodyledger.md`, `02-flightcheck.md`, `03-fieldvoice.md`, the candidate profile, the "Three Projects, Thirty Postings" report, and the three rounds of skeptic verdicts.

Scope: problems only. Objections the plans already absorb (advisory locks, pipe-delimited encoding, SITL in week 0, pre-quantized downloads, NDT companion, fake-transport reconnect tests, etc.) are not restated. Where a fact was verified against a primary source during this review it is marked **verified**; where it was not, it is marked **verify**, with the check to run.

Severity legend: **blocking** = an implementer cannot produce the claimed artifact, or the claim would be false, until fixed; **important** = will cost a week or produce a wrong number; **minor** = cheap to fix, but an implementer would otherwise guess.

---

## Shared (`00-shared.md`)

**[Dependency and order] Hosting decision is unresolved but week 1 of CustodyLedger deploys to it**
- Where: S14 "Hosting provider for CustodyLedger"; `01` §7 week 1 ("Deploy a `/healthz` endpoint to the host").
- Problem: S14 leaves Render-vs-Fly open; CustodyLedger week 1 exit criterion requires a live URL. The decision has to be made before week 1 day 1, and the cost line in S3 depends on it. Render's free web service sleeps after inactivity (distorts the week-7 latency table); Fly has no free tier and bills small machines pay-as-you-go. **Verify** current Render Starter price and Fly shared-cpu-1x price and Fly's minimum-billing policy the week before CustodyLedger starts.
- Correction: add to S14 a decision deadline: "decided in the last week of the preceding project". Add the decision rule: if the k6 latency table is to be published, the host must not cold-start (rules out Render free); if it is cut (§11 item 5), Render free is acceptable and the README states "free instance, sleeps after 15 min".
- Severity: important

**[Incorrect assumption] "Neon for local dev" ignores Neon Free plan limits and scale-to-zero**
- Where: S3 "Docker Desktop is not required"; `01` §3 exclusions ("Neon for local dev, a `postgres:16` service container in CI"); `01` §9.
- Problem: **verified** Neon Free plan: 100 CU-hours per project per month, 0.5 GB storage per project, 10 branches per project, compute auto-suspends after 5 minutes of inactivity and this cannot be disabled. Running Vitest integration suites and fast-check properties against a Neon `dev` branch on every local run burns CU-hours, adds a 1-3 s cold start to every first query after idle, and makes the week-7 latency numbers depend on whether the DB was asleep. Docker is not the only alternative: Postgres 16 installs in WSL2 with one `apt` command.
- Correction: in S3 and `01` §3/§9 replace "Neon for local dev" with "Postgres 16 installed in WSL2 (`sudo apt install postgresql-16`) for local tests; Neon `main` for the deployed instance only". In `01` §7 week 7, add "warm the Neon compute with a request 60 s before each storm run and state that scale-to-zero is enabled". Keep the CI service container.
- Severity: important

**[Incorrect assumption] "A fresh clone on a different machine" is not available**
- Where: S12 second bullet; `02` §7 week 7 and §8 "Reproduction | manual, once | `make all` from a clean clone on a second machine".
- Problem: S2 states one Windows 11 laptop and no hardware budget. There is no second machine.
- Correction: define the reproduction check as either (a) a fresh WSL2 distro (`wsl --import` a clean Ubuntu 24.04 image, no dotfiles) or (b) a manually triggered GitHub Actions workflow (`workflow_dispatch`) that clones, runs the documented commands, and uploads the produced numbers as an artifact. State which in each README.
- Severity: minor

**[Conflict] Toolchain split between Windows host and WSL2 is inconsistent with the tooling the plans require**
- Where: S3 ("Node 22 LTS via `nvm-windows` or WSL `nvm`"), S4 (`Makefile`), `01` §6 (`k6` CLI, `semgrep` CI only).
- Problem: `make` does not exist on the Windows host; Semgrep has no native Windows build; the CustodyLedger plan expects `make all`/`make test` in every repo. Splitting Node between host and WSL2 also produces two `node_modules` trees with different native binaries (`@node-rs/argon2`).
- Correction: state in S3 that all three projects are developed inside WSL2 (VS Code Remote-WSL), including Node via WSL `nvm`; delete the `nvm-windows` option. Add S3 line: "Python 3.12 via `uv` also for `custodyledger/verifier/`" (currently S3 lists Python only for Flightcheck and FieldVoice).
- Severity: minor

**[Places where an implementer would guess] "Dependency audit step" and "no credential tracked" test are unnamed**
- Where: S8 last bullet; S4 last bullet.
- Correction: S8: Node `npm audit --audit-level=high`; Python `uv export --format requirements-txt | pip-audit -r /dev/stdin` (add `pip-audit` to dev deps). S4: run `gitleaks/gitleaks-action` in CI with the default ruleset, plus one unit test that asserts `git ls-files` contains no path matching `(^|/)\.env$|\.pem$|id_rsa|\.key$`.
- Severity: minor

**[Missing edge cases] S9 gives no acceptance vectors for Wilson/bootstrap, and no rule for paired comparisons**
- Where: S9 first, second and fourth bullets.
- Problem: (1) "Unit-test against published values" names no value. (2) Both projects compare conditions measured on the *same* items (same utterances with boosting on/off; same logs under two pipeline versions). Two independent percentile-bootstrap intervals that overlap is not a valid decision rule; the comparison must bootstrap the paired per-item difference. The plans' pre-registration templates never say this, so the headline "boosting cut error from X to Y" would be decided by an invalid rule.
- Correction: add to S9: "For any comparison of two conditions measured on the same items, the statistic is the mean paired difference; resample items with replacement and report the 95% percentile interval of the difference; the pre-registered decision rule is 'the interval excludes 0'." Add test vectors: Wilson 95% for x=1, n=10 is (0.0179, 0.4042); for x=0, n=20 the upper bound is z²/(n+z²) = 0.1611 with lower bound 0. Bootstrap test: n=1000 draws from N(0,1) with a fixed seed, B=10,000; each percentile endpoint must be within 0.1×SE of the t-interval endpoint (SE = s/√n).
- Severity: important

**[Missing requirements] S7 and S13 impose per-plan obligations that two plans do not meet**
- Where: S7 first bullet ("For each project the plan names these explicitly"); S13 last row ("Each plan names its single riskiest new tool and the fallback if it is not working by a stated date").
- Problem: `01` names no hand-written components and no riskiest tool with a date. `02` names the C++ metric as hand-written but no AI-USAGE list, and its "riskiest" item is a data source, not a tool; the C++ toolchain fallback has no date. `03` names the normalizer but no date for the llama.cpp fallback.
- Correction: see the per-project entries below (CustodyLedger "AI-USAGE list", Flightcheck "AI-USAGE list", and each plan's "riskiest tool" entry).
- Severity: important

---

## CustodyLedger (`01-custodyledger.md`)

**[Security] `verify` as specified does not detect edits to the columns the application actually reads**
- Where: §4.2 (`signed_bytes` rationale), §4.3 (`verify` "recomputes each HMAC and each `prev_hash` link").
- Problem: `verify` is described as HMAC(key, `signed_bytes`) == `record_hmac` plus the `prev_hash` link. A DBA who changes `actor_id`, `event_type`, `entity_id` or `payload_sha256` on a row, but leaves `signed_bytes` and `record_hmac` untouched, passes that check: the stored bytes were never modified. The same applies to a swap of all non-`seq` columns between two rows ("reordered"). Storing `signed_bytes` therefore does *not* by itself deliver tamper evidence over the row; it only makes re-serialization unnecessary. Likewise, editing `custody_events.payload` or `lots.qty_on_hand` is invisible to a chain walk that never looks at those tables.
- Correction: specify `verify` as, for each row in `seq` order: (1) parse `signed_bytes` as JSON; (2) assert each parsed field equals the corresponding column (`seq`, `ts_ms`, `actor_id`, `event_type`, `entity_type`, `entity_id`, `payload_sha256_hex`, `prev_hash_hex`) — mismatch is `mutated`; (3) HMAC check — mismatch is `mutated`; (4) `prev_hash` equals the hash of the previous row (definition below) — mismatch is `chain_break`; (5) look up the referenced `custody_events` row and assert `SHA256(JCS(payload)) == payload_sha256` — mismatch is `mutated` with `source=custody_events`. `replay` then covers `lots`/holdings. Write this decision procedure into `SECURITY.md` and add one test per step.
- Severity: blocking

**[Security] Deleting the newest record(s) is undetectable; the plan claims "deleting any audit record" is detected**
- Where: §1 property 1; §7 week 2 ("delete a record → detected"); report bullet 2 ("editing, deleting, or reordering any audit record fails `verify`").
- Problem: A chain walk detects a gap in the middle (seq discontinuity, `prev_hash` mismatch) but not truncation of the tail: rows 1..N-1 are a valid chain. The DBA adversary from §3 can therefore erase the most recent events.
- Correction: add a single-row table `ledger_head(id int primary key check (id=1), last_seq bigint, last_hash bytea, head_hmac bytea)` where `head_hmac = HMAC(key, JCS({seq:last_seq, hash:last_hash_hex}))`, updated in the same transaction as every append (this row also serialises appends — see next entry). `verify` starts by checking `head_hmac` and then that the table's max `seq` equals `last_seq`; mismatch is reported as `deleted` at `last_seq+1`. Add a test that deletes the last row and asserts detection. Name in `SECURITY.md` that the head row is the trust anchor and that an attacker with the key can rewrite it (already the stated accepted risk).
- Severity: blocking

**[Architectural] `seq` is declared gapless but the schema uses `bigserial`, and concurrent appends are unspecified**
- Where: §4.2 (`seq bigint -- gapless, per-ledger, assigned inside the transaction`); §5.1 (`audit_records(seq bigserial pk, ...)`).
- Problem: Postgres sequences are non-transactional; any rolled-back write (a CHECK violation, a 409, a crash) consumes a value and leaves a permanent gap, which `verify` will then report as `deleted`. Separately, `prev_hash` requires reading the previous record; two concurrent appends under READ COMMITTED both read the same predecessor, and nothing in the plan serialises them.
- Correction: drop `bigserial`. Assign `seq` inside the write transaction as `UPDATE ledger_head SET last_seq = last_seq + 1 RETURNING last_seq, last_hash` (the row lock serialises all appenders and hands back the predecessor hash), insert the audit row with that `seq`, then update `last_hash`/`head_hmac`. Add a test: run a write that fails a CHECK constraint, then a successful write, and assert `verify` is clean and `seq` is contiguous. Delete "per-ledger" (there is exactly one ledger; the nursing vocabulary is a seed, not a second ledger).
- Severity: blocking

**[Places where an implementer would guess] The signed object and `prev_hash` are not fully specified**
- Where: §4.2.
- Problem: `prev_hash` is never defined (hash of what?). The signed object's types are not fixed (`seq`/`ts_ms` as JSON numbers or strings?), there is no version field, hex case is unspecified, the HMAC key encoding and length are unspecified, and the "fixed test key" for vectors is unspecified. Adding a field later ("schema addition") will change JCS output, and nothing says how a v2 record is distinguished from a v1 record.
- Correction: specify in `SECURITY.md`: `prev_hash` = the previous row's `record_hmac` (32 bytes), zero bytes for `seq` 1. Signed object = `{"v":1,"seq":<int>,"ts_ms":<int>,"actor_id":"<uuid lowercase>","event_type":"<enum>","entity_type":"<enum>","entity_id":"<uuid>","payload_sha256":"<64 lowercase hex>","prev_hash":"<64 lowercase hex>"}`, JCS-serialised, UTF-8. `seq` and `ts_ms` are JSON integers (both < 2^53). `HMAC_KEY` is 32 bytes given as 64 lowercase hex characters; the test key is `000102...1f` (bytes 0x00..0x1f). Future fields go under `"v":2` and the verifier dispatches on `v`. The five vectors must include the full payload object (with at least one non-ASCII string, one nested object, one quantity as a decimal string, one `null`) so `payload_sha256` derivation is exercised, not just the outer object. Vectors are computed with `openssl dgst -sha256 -mac HMAC -macopt hexkey:<key>` over the JCS text produced by hand, so they are independent of both implementations.
- Severity: blocking

**[Incorrect assumption] node-postgres returns `bigint` as strings; JCS will serialise them differently from Python**
- Where: §10 risk row "Postgres `numeric` handling in JS (`pg` returns strings)"; §4.2.
- Problem: The plan covers `numeric` but not `int8`. `pg` returns OID 20 (`bigint`) as JS strings by default. If `seq` and `ts_ms` read back from the DB are placed into the signed object as strings, `canonicalize` emits `"seq":"42"` while the Python verifier (which receives integers from the export) emits `"seq":42`; HMACs will disagree and the week-6 cross-check fails for a non-cryptographic reason. Also, `canonicalize` throws on `BigInt`, and RFC 8785 requires numbers within ±2^53. **Verified** that PyPI `rfc8785.dumps()` returns `bytes` and raises `CanonicalizationError` subclasses on unserialisable values; it does not accept `Decimal`, so payload quantities must be strings on both sides.
- Correction: in `server/` set `pg.types.setTypeParser(20, v => Number(v))` (safe: `seq` and epoch-ms are far below 2^53) and keep `numeric` (OID 1700) as strings. State in `SECURITY.md`: "payload objects contain only strings, integers, booleans, null, arrays and objects; quantities are decimal strings such as `"12.500"`; no floats". Add a test that a payload containing a JS float is rejected before hashing.
- Severity: blocking

**[Missing requirements] The export does not carry what the Python verifier needs to "re-derive every balance"**
- Where: §4.7 (export = "audit records plus lot snapshots"; verifier "re-derives every balance"); §5.2 `GET /api/admin/export`.
- Problem: Audit records carry only `payload_sha256`, not the payload. Quantities, event types and counterparties live in `custody_events`. From audit rows plus lot snapshots the verifier cannot compute any balance, and cannot check that the events hash to the audit rows.
- Correction: define the NDJSON export as typed lines: `{"t":"head",...}`, `{"t":"audit",...}` (bytea fields as lowercase hex, `signed_bytes` as the UTF-8 string itself), `{"t":"event", id, lot_id, type, qty:"<decimal string>", from_user, to_user, actor_id, payload:{...}, audit_seq}`, `{"t":"lot",...}`, `{"t":"holding", lot_id, holder_id, qty}`. The verifier: checks head, walks the chain per the decision procedure above, hashes each event payload and matches it to its audit row, replays events into holdings, and compares to the exported holdings/lots. Make `export` also a CLI subcommand so the CI job can produce the file without HTTP: `node server/dist/cli.js export > export.ndjson && uv run --project verifier ledger-verify export.ndjson`.
- Severity: blocking

**[Architectural] Balances are modelled per lot only, so transfers between handlers are unrepresentable and the invariant is ambiguous**
- Where: §4.1 invariant (`on_hand = received − consumed − net_transferred_out`); §5.1 `lots(qty_received, qty_on_hand)`; §7 week 3 property (`received = consumed + returned + on_hand + net_out`).
- Problem: A transfer moves custody between handlers without changing the lot's total. `net_transferred_out` only has meaning per holder, but the schema has no per-holder quantity. The week-3 property double-counts `returned` (returned material is back in `on_hand`). `replay` therefore has nothing well-defined to rebuild for transfers, and the IDOR predicate "handler B's lot" has no data to evaluate against.
- Correction: add `holdings(lot_id uuid, holder_id uuid, qty numeric(12,3) not null check (qty >= 0), primary key (lot_id, holder_id))` where the magazine is a reserved holder row per lot. Event effects: receive: magazine += qty; issue (on approval): magazine −= qty, handler += qty; transfer: from −= qty, to += qty; consume: holder −= qty, `lots.qty_consumed` += qty; return: handler −= qty, magazine += qty. Invariants: for every lot, `sum(holdings.qty) + qty_consumed = qty_received`; every `holdings.qty >= 0`. `lots.qty_on_hand` is then a materialised `sum(holdings.qty)` that `replay` recomputes. Add `CHECK (qty_on_hand >= 0 AND qty_on_hand <= qty_received)` on `lots`. All decrements are conditional updates (`UPDATE holdings SET qty = qty - $1 WHERE ... AND qty >= $1 RETURNING`) so two concurrent consumes cannot overdraw; lock `holdings` rows in `(lot_id, holder_id)` order to avoid deadlock on transfers.
- Severity: blocking

**[Places where an implementer would guess] Lot state machine, issuance states and the reconcile endpoint are incomplete**
- Where: §4.1 diagram; §4.5; §5.2 (`POST /api/lots/:id/reconcile ... begin/complete`).
- Problem: The diagram has no `receive`, `transfer`, or "everything returned" transition; `discrepancy` has no exit; `issuances.status` values are never listed; the approve statement is not given, so "approving an already-issued lot" has no defined DB-level guard; `reconcile` is one route for two actions with no body schema. "Exhaustive transition table test" cannot be written from this.
- Correction: add the full table to §4.1 (states × events → next state or rejection code). Minimum: `open` × issue → `issued`; `issued` × transfer/consume/return → `issued`; `issued` × return-all (magazine holds everything) → `open`; `open|issued` × begin_reconcile → `reconciling`; `reconciling` × complete(balanced) → `closed`; `reconciling` × complete(unbalanced) → `discrepancy`; `discrepancy` × resolve(admin, note) → `reconciling`; `closed` × anything → rejected; all other pairs rejected. Issuance status enum: `pending | issued | rejected | cancelled`. Approve is exactly `UPDATE issuances SET approver_id=$approver, status='issued', approved_at=now() WHERE id=$id AND status='pending' RETURNING *` inside the same transaction as the holdings move; zero rows → 409 `issuance_not_pending`. Split reconcile into `POST /api/lots/:id/reconcile` (begin, body `{counted_qty}`) and `POST /api/lots/:id/reconcile/complete`; store `reconciliations(id, lot_id, counted_qty, ledger_qty, result, created_at)` with `CHECK (result <> 'balanced' OR counted_qty = ledger_qty)`.
- Severity: blocking

**[Security] The 404-vs-403 rule cannot be applied without a per-role visibility definition, and as written contradicts itself**
- Where: §5.3; §4.6.
- Problem: "404 where leaking existence would be an information disclosure (handler B asking for handler A's lot)" presumes handlers cannot see other handlers' lots. But `POST /api/lots/:id/issuances` lets a handler request issuance from a lot they do not yet hold, so lot existence must be visible to handlers (via `GET /api/lots`) before any issuance. If existence is visible, the plan's own rule says 403, not 404, for a transfer on someone else's holding. The suite would assert whichever the implementer picked.
- Correction: define visibility sets and a mapping table in `SECURITY.md`: handler may `GET` any lot with status `open` or `issued` (needed to request issuance) but may `GET` holdings only where `holder_id = self`; keeper/auditor/admin see everything. Then: role check first → 403 `forbidden_role` regardless of existence; object lookup within the caller's visibility set → 404 `not_found` if the object is outside it; ownership/state predicate failure on a visible object → 403 `forbidden_object`. Two-person CHECK violation (SQLSTATE 23514) → 409 `two_person_rule`. Missing/invalid `Idempotency-Key` → 400 `idempotency_key_required`. Commit the table of `type` slugs and status codes as the fixture the denial suite iterates over.
- Severity: important

**[Places where an implementer would guess] Ownership predicates and the 30 denial cases are not enumerated**
- Where: §4.6.
- Correction: list the predicate per mutating route: `transfer|consume|return`: `exists holdings where lot_id=:id and holder_id=session.user and qty>0`; `issuances` (request): lot visible and status in (`open`,`issued`); `approve`: role keeper and `issuances.requester_id <> session.user` (the CHECK is the backstop, not the only check); `reconcile`: role keeper. Enumerate the suite by class with counts: object reference 8, vertical escalation 8, approval bypass 6, session handling 5, idempotency abuse 3 = 30; each case is a row `(actor, route, target, expected_status, expected_type)` in a JSON fixture so the count on the README is `fixture.length`.
- Severity: important

**[Security or reliability] Rate limiting and the k6 storm are in direct conflict**
- Where: §7 week 4 (rate limiting), §7 week 7 (1000 concurrent requests from one client, one key).
- Problem: `@fastify/rate-limit` keys on IP by default. A 1000-request burst from one laptop will be answered mostly with 429, so the storm measures the rate limiter, not idempotency, and the "exactly one row" assertion becomes trivially true for the wrong reason.
- Correction: rate-limit only `POST /api/auth/login` (e.g. 10/min/IP) and apply a high per-session allowance elsewhere (e.g. 600/min); the storm authenticates once and reuses the session cookie; the k6 script asserts that no response is 429 and that exactly one response is 201 and the rest are 200-with-replay (same body), then runs `SELECT count(*) FROM custody_events WHERE ...` and asserts 1. Record this in the k6 script header.
- Severity: important

**[Security or reliability] Argon2id parameters and the storm's login pattern can exhaust a 256 MB instance**
- Where: §7 week 4; §7 week 7; §9.
- Problem: No Argon2id parameters are given. Default `@node-rs/argon2` memory cost is 19 MiB (`m=19456`); the common "64 MiB" recommendation would let ~4 concurrent logins consume the whole instance. Also unspecified: session TTL, cookie name, `Secure` on `localhost` (the cookie will not be set over plain HTTP in dev).
- Correction: specify Argon2id `m=19456 KiB, t=2, p=1` (OWASP minimum), state it in `SECURITY.md`; session TTL 12 h sliding, cookie `cl_session`, `Secure` only when `NODE_ENV=production`; login rate-limited as above; the storm uses one pre-established session.
- Severity: important

**[Security or reliability] Public write API with published demo credentials has no reset**
- Where: §9 seeding ("demo users for each role with published credentials").
- Problem: Anyone can write to the ledger; the ledger is append-only by design, so junk cannot be removed. Within a week the demo balances in the README screenshots will not match the live instance.
- Correction: add a scheduled reset: a GitHub Actions `schedule` workflow (daily) that runs the migration runner's `reset` command against the deployed DB (drop schema, migrate, seed) using repository secrets. State on the login page and README: "demo data resets nightly; the chain restarts at seq 1". Add `POST /api/admin/reset` guarded by admin role only if needed for the demo video.
- Severity: important

**[Places where an implementer would guess] Idempotency details missing**
- Where: §4.4, §5.2.
- Correction: `Idempotency-Key` must be a UUID v4 string (400 otherwise); required on every `POST /api/*` except `/auth/login`; stored `response_status` and `response_body` only for 2xx (any non-2xx aborts the transaction, which removes the key row, so retries re-execute); rows older than 24 h are purged by the nightly reset job; the replayed response carries header `Idempotent-Replayed: true`. Note in the ADR that under READ COMMITTED the second `INSERT ... ON CONFLICT DO NOTHING` blocks until the first transaction commits or rolls back, which is the property relied on; add a test with two concurrent same-key, different-body requests asserting one 201 and one 409.
- Severity: important

**[Places where an implementer would guess] `audit_records` cannot answer "which lot balances depend on records at or after that point"**
- Where: §4.3 `verify` output; §5.1 schema.
- Problem: For an approval record `entity_type='issuance'`, the lot is not on the audit row. `verify` cannot list affected lots without joining through tables that may themselves be tampered.
- Correction: add `lot_id uuid not null` to `audit_records` and to the signed object (`"lot_id"`). Define `verify`'s output as JSON: `{ok:false, first_bad_seq, class, detail, affected_lots:[...]}` where `affected_lots` = distinct `lot_id` over rows with `seq >= first_bad_seq`; exit code 2 on failure. Define the test operations per class and the expected `first_bad_seq`: mutate one byte of `signed_bytes` at k → k, `mutated`; `DELETE` row k (k < N) → k, `deleted`; delete row N → N, `deleted`; swap all non-`seq` columns of k and k+1 → k, `reordered`; replace `prev_hash` of k with random bytes and re-sign with the real key (simulating a key-holding attacker) → k, `chain_break`.
- Severity: important

**[Overstated claim] "Enforced in the schema, not in handler code" and resume bullet 3 overstate what the constraints do**
- Where: §4.5; report bullet 3 ("Two-person issuance and lot reconciliation enforced by database constraints").
- Problem: The CHECKs enforce distinctness and completeness; "the approver holds the keeper role" and "the issuance is still pending" are handler code (or would need a trigger). Nothing in the plan enforces reconciliation with a constraint at all.
- Correction: reword §4.5 to "the distinctness and completeness of the two-person rule are schema constraints; the approver's role is checked in the preHandler and covered by the denial suite". Either add the `reconciliations` CHECK from the state-machine entry above (then the bullet is true) or change the bullet to "Two-person issuance enforced by database constraints; lot reconciliation with an exhaustive state-machine suite".
- Severity: important

**[Testing gaps] Claims with no test that would catch them if false**
- Where: §5.1, §4.3, §4.4, §8.
- Correction: add tests for: (1) the application role cannot `UPDATE`/`DELETE` `audit_records` (expect SQLSTATE 42501); (2) `seq` stays contiguous after a rolled-back write; (3) tail deletion detected; (4) editing `custody_events.qty` directly is detected by `verify` (payload hash) and by `replay`; (5) concurrent same-key different-body → exactly one 201 and one 409; (6) a payload with a float is rejected; (7) the migration runner is idempotent (running twice applies nothing the second time); (8) session cookie flags (`HttpOnly; Secure; SameSite=Strict`) present in the login response in production mode.
- Severity: important

**[Dependency and order] Packages the plan uses but does not list; versions to pin**
- Where: §6; §5.2 ("`zod-to-openapi`"); §9 ("static build served by the same origin"); §8 ("coverage ... threshold enforced in CI").
- Correction: add `@fastify/static` (same-origin SPA serving), `@asteasolutions/zod-to-openapi` (the actual package name), `@vitest/coverage-v8`. Pin Zod major and `@hookform/resolvers` to a compatible pair (Zod 4 requires `@hookform/resolvers` ≥ 5; **verify** the exact pair on install). Set the coverage floor to a number (60% lines on `server/src`, raised only after v0.5).
- Severity: minor

**[Places where an implementer would guess] Storm and latency-table specification**
- Where: §7 week 7; S6.
- Correction: k6 script parameters: `vus: 1000, iterations: 1000` (one request each) against `POST /api/lots/:id/consume` with a shared `Idempotency-Key`, one session cookie; three runs, report which run was cold. Latency table: separate script, `constant-arrival-rate` at 20 and 50 RPS for 60 s with distinct keys, p50/p95/p99, and a statement of the `pg` pool size (set `max: 10` explicitly) so the reader can see that server-side queueing at the pool is the bottleneck.
- Severity: minor

**[Places where an implementer would guess] Remaining unspecified interfaces**
- Where: §5.2, §9.
- Correction: `GET /api/audit?from=&to=` takes `seq` bounds (integers), max 500 rows, ordered by `seq`; seeds create exactly one user per role (`handler_a`, `handler_b`, `keeper`, `auditor`, `admin`) plus three lots, with the password policy for seeds stated (fixed published passwords); migration files named `NNN_description.sql`, applied in a transaction each, with a SHA-256 recorded in `schema_migrations(version, checksum, applied_at)`. `HMAC_KEY` and `SESSION_SECRET` formats stated in `.env.example`.
- Severity: minor

**[Missing requirements] AI-USAGE hand-written list and riskiest-tool date are absent**
- Where: §7 week 8; S7; S13.
- Correction: name as hand-written and never generated: `server/src/ledger/append.ts` (audit append + head update), `server/src/ledger/verify.ts`, `server/src/ledger/replay.ts`, `server/src/auth/authorize.ts` (preHandler + predicates), `db/001_init.sql`, `verifier/src/ledger_verify/chain.py`. Riskiest new tool: Fastify + TypeScript strict on the write path; fallback date: end of week 2 — if the ledger core is not green, drop the React client from v0.5 (serve the audit view as server-rendered HTML) rather than slip v0.1.
- Severity: important

**[Week budget] Week 2 breaks first; week 4 breaks second**
- Where: §7.
- Problem: Week 2 asks a first-time TypeScript/Fastify/Postgres user for four transactional write paths, the HMAC chain with head anchoring, idempotency, two CLIs, and four tamper tests in ~10 h. Week 4 asks for password hashing, sessions, RBAC, ownership predicates, two-person constraints plus the bypass demonstration, a 30-case suite, rate limiting and helmet in ~10 h.
- Correction: Week 2 = `receive` + `consume` only, chain append, `verify` with the mutate and tail-delete tests, idempotency; tag `v0.1` on that. Week 3 = `transfer`/`return` + holdings, `replay`, delete/reorder tests, the transition table; move the fast-check properties to week 5 (integration week). Week 4 = auth, RBAC, two-person constraints, and 15 denial cases; the remaining 15 cases, rate limiting and helmet move to week 7 (rate limiting is already cut-order item 4).
- Severity: important

**[Conflict] The report's idempotency wording contradicts the plan**
- Where: report §"What you build" ("unique constraint on actor, key, and body hash"); plan §4.4 (`UNIQUE (actor_id, key)` with body hash compared).
- Problem: The plan is correct (a triple constraint would allow a second row for a different body). Any README or resume text copied from the report would describe a design that cannot produce the 409.
- Correction: the README, `SECURITY.md` and the resume bullet must say "unique constraint on (actor, key); body hash compared on conflict".
- Severity: minor

---

## Flightcheck (`02-flightcheck.md`)

**[Incorrect assumption] The named parameters are wrong or version-dependent**
- Where: §4.3 example (`BAT_N_CELLS * BAT_V_EMPTY`), §4.3 candidates ("descent rate before disarm versus the land-speed parameter"), report ("`BAT_LOW_THR`, `GF_MAX_HOR_DIST`, `MPC_XY_VEL_MAX`, `COM_DISARM_LAND`").
- Problem: (1) `BAT_N_CELLS` and `BAT_V_EMPTY` were renamed `BAT1_N_CELLS` and `BAT1_V_EMPTY` when multi-battery support landed (PX4 v1.11); most logs in the public index are newer than that, so the example requirement resolves to `not_evaluable` on almost every log. (2) `COM_DISARM_LAND` is the auto-disarm timeout after landing in seconds; it is not a threshold for anything the plan measures. The land-speed parameter is `MPC_LAND_SPEED` (m/s); `MPC_Z_VEL_MAX_DN` is the max descent speed above `MPC_LAND_ALT1`. (3) `BAT_LOW_THR` is a fraction of remaining capacity, comparable to `battery_status.remaining`, not to a voltage. (4) The alias layer in §4.2 covers topics and fields but says nothing about parameter names.
- Correction: extend the alias layer to parameters: `params/aliases.yaml` maps a logical name (`bat_n_cells`) to candidates `[BAT1_N_CELLS, BAT_N_CELLS]` in priority order, resolved per log with the chosen name recorded in `thresholds.source_params`. Rewrite the example requirement as `battery_remaining_at_disarm >= BAT_LOW_THR` with `requires_signals: [battery_remaining, landed]`, and keep a voltage variant only as a second requirement using `bat_n_cells * bat_v_empty`. Name the descent-rate threshold `MPC_LAND_SPEED` and define the window as the final descent below `MPC_LAND_ALT2` (see the metric-window entry). Remove `COM_DISARM_LAND` from every threshold list; it may be used only to locate the disarm event. **Verify** each final parameter name against the PX4 parameter reference for the firmware series chosen in week 0 and record the table in the ADR.
- Severity: blocking

**[Missing edge cases] Disabled or sentinel parameter values produce systematic false verdicts**
- Where: §4.3 threshold resolution ("If a required parameter or signal is absent, the requirement is `not_evaluable`").
- Problem: Presence is not sufficiency. `GF_MAX_HOR_DIST` defaults to 0 meaning "disabled"; resolving it to a threshold of 0 with comparator `<=` marks every flight FAIL. `BAT1_N_CELLS` may be 0 ("unknown"), giving a voltage threshold of 0 and PASS everywhere. `COM_DISARM_LAND` ≤ 0 is "disabled". None of these is "absent".
- Correction: add to the requirement schema `disabled_when: "<expr over params>"` (e.g. `GF_MAX_HOR_DIST <= 0`), evaluated before `expr`; when true the verdict is `not_evaluable` with cause `param_disabled`. Add a unit test per requirement for its sentinel. Define the cause taxonomy exhaustively: `param_missing`, `param_disabled`, `param_changed_in_flight`, `signal_missing`, `window_missing`, `insufficient_samples`, `quality_fail`.
- Severity: blocking

**[Missing edge cases] Parameters can change during a flight**
- Where: §4.3 ("Every ULog embeds the vehicle's full parameter set").
- Problem: **verified** pyulog exposes both `initial_parameters` (dict) and `changed_parameters` (list of `(timestamp, name, value)`). A parameter altered mid-flight (common with QGC-connected hobbyists) makes "the operator's threshold" ambiguous.
- Correction: resolve from `initial_parameters`; if any `requires_params` entry appears in `changed_parameters` with a different value, verdict is `not_evaluable` with cause `param_changed_in_flight`, and the count is reported in the matrix.
- Severity: important

**[Incorrect assumption / verify] Corpus source facts, now checked**
- Where: §4.1 ("Verification required before week 1"); §10.
- Problem/finding: **Verified** from `PX4/flight_review` `app/download_logs.py`: index is `https://review.px4.io/dbinfo` (JSON array), download is `https://review.px4.io/download?log=<log_id>`; the script filters on `mav_type`, `flight_modes` (log must contain all listed modes), `error_labels`, `rating`, `vehicle_uuid`, `vehicle_name`, `airframe_name`, `airframe_type`, `source`, `git_hash` (`ver_sw`), and has `--latest-per-vehicle`, `--max-num` (default 10; -1 requires confirmation above 100), and `--delay` (default 6 s between downloads "to respect server rate limits"). It does **not** filter on duration or firmware release; **verify** whether `dbinfo` entries carry `duration_s` and `ver_sw_release`; if not, the duration and firmware-series filters must run after download from the ULog header (`ULog(path, parse_header_only=True)` gives `msg_info_dict`). No data licence is published for uploaded logs; uploaders can delete logs.
- Correction: (1) manifest download must honour a 6 s delay (50 logs ≈ 5 min, 300 logs ≈ 30 min; runs unattended). (2) `make corpus` must tolerate `404` for deleted logs, write `corpus/missing.json`, and the README must say reproduction is "best effort against a public service". (3) ADR: raw logs are never redistributed; only per-log aggregates, metadata and the manifest are published; `vehicle_uuid` from `dbinfo` is the grouping key for `GroupKFold` (fallback `msg_info_dict['sys_uuid']`; if both absent, the log is its own group and the count is reported). (4) Mission mode filter = `nav_state` 3 (`AUTO_MISSION`) present in `flight_modes`; multicopter = `mav_type` in the quadrotor/hexarotor/octorotor set (list it).
- Severity: important

**[Incorrect assumption] Vibration metric feasibility depends on per-log IMU sample rate, which must be recorded**
- Where: §4.3 (vibration RMS requirement), §4.4 (`vibration_rms_z`, FFT band power).
- Problem: **verified** in current `logged_topics.cpp` that the default profile logs `sensor_combined` with no rate limit (full IMU rate), so on current firmware the FFT metric is feasible; the round-2 objection that it needs the high-rate profile is not correct for current firmware. But older firmware in the corpus logged at lower rates, and the plan states neither the band nor the minimum sample rate the band requires.
- Correction: at ingest, record `sample_rate_hz` per signal in `quality`; the vibration metric declares `band_hz: [10, 80]` (or whatever week 0 shows) and returns `insufficient_samples` when `sample_rate_hz < 2.5 × band_max`. Add `vehicle_imu_status.accel_vibration_metric` (default profile, 1 Hz) as an alias fallback signal so the requirement stays evaluable on logs without high-rate IMU. Cite the PX4 Flight Review vibration guidance page for the advisory level and label the requirement `class: advisory` (the §10 lean).
- Severity: important

**[Dependency and order] Golden suite "wired into CI" contradicts "CI does not download the corpus"**
- Where: §4.6 and §7 week 5 (~20 pinned logs, "CI re-runs them"); §9 ("CI runs ... the golden regression against a small committed fixture set (three tiny synthetic ULog-derived Parquet fixtures ...)").
- Problem: Two different golden suites are described. The 20-log suite cannot run in CI without the raw logs, and committing derived Parquet for 20 logs is likely 50-100 MB.
- Correction: define two tiers explicitly. `golden/ci/`: three committed Parquet fixtures (whitelisted signals only, ≤ 5 MB total), run on every push. `golden/corpus/`: 20 pinned `log_id`s with expected outputs, run by the manual `workflow_dispatch` workflow after `make corpus`, and locally before every tag. The README "golden regression suite fails CI when any verdict changes" refers to tier 1, and says so.
- Severity: important

**[Places where an implementer would guess] Alias layer, multi-instance and memory handling are unspecified**
- Where: §4.2; §5.1; §6.
- Correction: `ingest/aliases.yaml`: per logical signal, an ordered list of `{topic, field, min_version, max_version, scale}` entries; version from `ULog.get_version_info()` (**verified** signature `get_version_info(key_name='ver_sw_release')` returning `(major, minor, patch, type)`). Known renames to seed it: `vehicle_local_position_setpoint.z` → `trajectory_setpoint.z` (v1.13+); `vehicle_gps_position` → `sensor_gps` for raw GPS (v1.13+; `vehicle_gps_position` remains as the selected output); `sensor_gps.lat/lon` int32×1e-7 → `latitude_deg/longitude_deg` doubles (v1.15+); `alt` mm → `altitude_msl_m`. Multi-instance rule: always `get_dataset(name, multi_instance=0)` and record `multi_id` in `quality` when more instances exist. Memory: construct `ULog(path, message_name_filter_list=WHITELIST)` (**verified** constructor parameter) with the whitelist committed; ingest in a `multiprocessing.Pool(processes=max(1, cpu//2), maxtasksperchild=1)`.
- Severity: important

**[Places where an implementer would guess] Metric windows and definitions**
- Where: §4.4.
- Correction: state per metric: `altitude_error_rms_cruise`: within `nav_state==3` segments, drop the first and last 10 s of each segment, require ≥ 30 s remaining, RMS of `local_position.z − trajectory_setpoint.z`; `descent_rate_pre_land_p95`: `vehicle_local_position.vz` (NED, positive down) over the 5 s ending at the rising edge of `vehicle_land_detected.landed` — not at disarm, because the vehicle sits landed for `COM_DISARM_LAND` seconds with `vz≈0`, which would dilute the percentile — and only samples where `−z < MPC_LAND_ALT2` if the threshold is `MPC_LAND_SPEED`; percentile method fixed as NumPy `method='linear'` and implemented identically in C++; `max_distance_from_home`: horizontal norm of `local_position.(x,y) − home_position.(x,y)`; `gps_min_fix_type`/`gps_max_eph` from the aliased GPS signal; `battery_*_at_disarm` from `battery_status` instance 0, `voltage_filtered_v`, last sample before the disarm event (`actuator_armed.armed` falling edge). Butterworth: order 2, cutoff 5 Hz, `scipy.signal.sosfiltfilt`, stated per metric.
- Severity: important

**[Places where an implementer would guess] Requirement expression language and units**
- Where: §4.3 YAML.
- Correction: `expr` grammar = Python `ast` restricted to `Name`, numeric `Constant`, `BinOp` with `+ - * /`, `UnaryOp -`; anything else raises at load time (add a test that `__import__` and calls are rejected — the DatteBotYo dice evaluator is the pattern). Add `unit` to each requirement and to the parameter alias table; the resolver refuses a mismatch. `comparator` ∈ {`>=`, `<=`, `>`, `<`}. `requires_signals` uses logical names only (the example mixes `battery_voltage` with the topic name `vehicle_status`).
- Severity: important

**[Places where an implementer would guess] Table schemas and derived fields**
- Where: §5.1.
- Correction: `parameters(log_id, name, value_num double, value_type text)` — ULog parameters are only `int32` or `float32`, so `value_str` is dead; keep the type. `events.type` vocabulary: `nav_state_change`, `arming_change`, `landed_change`, `failsafe_change`, with `subtype` the new value. `quality.check` list with definitions: `ts_nonmonotonic`, `ts_gap_gt_1s`, `ts_duplicate`, `value_out_of_range` (range table committed per signal), `topic_missing`, `sample_rate_hz` (value, not a count), `param_changed`. `logmeta.start_utc` = first `time_utc_usec > 0` from the aliased GPS signal minus that sample's boot `timestamp`; `lat0/lon0` from `home_position`. Parquet layout: `data/parquet/log_id=<id>/samples_<signal>.parquet` with columns `(t_us uint64, value float64)` so DuckDB reads `read_parquet('data/parquet/*/samples_battery_voltage.parquet', hive_partitioning=true)`.
- Severity: important

**[Places where an implementer would guess] C++ contract and timing boundary**
- Where: §4.7, §5.4.
- Correction: commit `export/checks.schema.json`: `{version, logs:[{log_id, metrics:{name: value}, thresholds:[{req_id, metric, comparator, value|null, reason}], raw:{descent_vz:[...], land_edge_index}}]}` and `cpp_verdicts.json`: `[{log_id, req_id, status, metric_value}]`. Agreement test: verdict sets equal; `descent_rate_pre_land_p95` within `abs 1e-6`. Timing: both sides time only "read `checks.json` → write verdicts" (`hyperfine --warmup 3 --runs 20`), and the README states ULog parsing stays in Python; never call it a pipeline speedup.
- Severity: important

**[Places where an implementer would guess] Slicing and model definitions**
- Where: §4.8, §4.9.
- Correction: bucket edges committed in `stats/buckets.yaml`: altitude = max `−z` above home in `[0,30), [30,60), [60,120), [120,∞)` m; duration `[60,180), [180,600), [600,1200]` s; solar elevation `<0, [0,20), [20,45), ≥45` degrees; a bucket with n<20 prints `n=<k>, insufficient`. Model label = a rising edge of `vehicle_status.failsafe`; features computed over `[arming, min(first_failsafe, arming+60 s)]`; report class counts in the same sentence as precision/recall; `GroupKFold(n_splits=5)` by `vehicle_uuid`.
- Severity: important

**[Overstated claim] Resume bullet 3 commits to a failsafe memo the plan does not promise**
- Where: report bullet 3 ("root-cause memo on a public failsafe event"); plan §4.10 ("a real public log where a requirement failed").
- Correction: write the bullet after the memo subject is fixed; the plan should carry two alternative wordings ("public failsafe event" / "requirement violation in a public log") and forbid using the first unless `vehicle_status.failsafe` actually transitions in the chosen log.
- Severity: important

**[Overstated claim] "C++ checker that independently agrees with the Python evaluator"**
- Where: §1; report bullet 3.
- Correction: README and resume wording: "re-derives every verdict and independently recomputes one metric from raw samples". The 100% agreement figure is by construction for the verdict step and only meaningful for the raw-sample metric; say so under "What this is not".
- Severity: minor

**[Conflict] Report says Plotly on GitHub Pages; plan says Matplotlib static**
- Where: report week 7 ("Static Plotly report on GitHub Pages"); plan §4.10.
- Correction: pick one and align. Cheapest: keep Matplotlib for committed figures and add exactly one Plotly HTML page (the traceability matrix with per-log drill-down) since it is the same DataFrame; otherwise delete "Plotly" from the report text before it reaches a README or resume.
- Severity: minor

**[Testing gaps]**
- Where: §8.
- Correction: add (1) an alias-layer test with two synthetic fixtures whose topics differ by firmware version and assert the same logical signal resolves; (2) a sentinel test per requirement (`GF_MAX_HOR_DIST=0` → `param_disabled`); (3) `param_changed_in_flight`; (4) expression-evaluator rejection of calls/attributes; (5) a test that `quality.sample_rate_hz` gates the vibration metric; (6) a test that the manifest downloader records a missing log rather than failing.
- Severity: important

**[Places where an implementer would guess] DuckDB-vs-SQLite pre-registration**
- Where: §3, §7 week 2.
- Correction: pre-register: dataset = the 50-log slice; queries = three named queries from the ten (one full-scan aggregate, one per-log group-by, one time-window filter); statistic = median of 10 runs each, cold cache; hypothesis = "DuckDB is at least 5× faster on all three"; null wording written in advance.
- Severity: minor

**[Missing requirements] AI-USAGE list and riskiest-tool date**
- Where: §7 week 7; S7; S13.
- Correction: hand-written, never generated: `metrics/*.py`, `params/resolve.py`, `requirements/evaluate.py`, `stats/wilson.py`, `stats/bootstrap.py`, `cpp/src/descent.cpp`. Riskiest tool: CMake + GoogleTest on a first C++ project; fallback date: end of week 4 day 5 — if the CI job is not green, the checker ships as a single-file `g++` build with a `Makefile` target and GoogleTest is dropped from the claim.
- Severity: important

**[Week budget] Week 1 breaks first; week 5 breaks second**
- Where: §7.
- Problem: Week 1 combines the manifest builder, checksummed download (~5 min of waiting per 50 logs), first Parquet ingest, DQ checks under pytest, a throughput number, the Typer skeleton and CI — for someone who has never used Pandas or pyarrow. Week 5 combines the golden suite, corpus widening across firmware versions via a new alias layer, and a two-page RCA memo.
- Correction: Week 1 = manifest + download + ingest of 20 logs + timestamp-monotonicity check only + CI. Move the remaining DQ checks and the throughput measurement to week 2 (drop the SQL query count from ten to five to make room). Week 5 = golden suite (tier 1 and 2) + RCA memo; move corpus widening to week 6 and cut the model baseline (already cut-order item 1) unless week 5 finished early.
- Severity: important

**[Minor] Repository name collision**
- Where: §3 ("Repository name: `flightcheck`").
- Correction: "FlightCheck" is an existing commercial preflight product; a recruiter searching the name will find it first. `ulogcheck` (the round-3 name) or `px4-reqcheck` avoids that.
- Severity: minor

---

## FieldVoice (`03-fieldvoice.md`)

**[Incorrect assumption] DEMAND is not an industrial-noise dataset and its licence is stated inconsistently**
- Where: §4.1 ("a public industrial-noise dataset (DEMAND, Zenodo, CC BY-SA)"); report ("public DEMAND industrial-noise dataset").
- Problem: **verified** on the Zenodo record: DEMAND contains domestic (kitchen, living room, washing machine), nature (field, park, river), office (hallway, meeting, office), public (café, restaurant, station), transportation (bus, car, metro) and street (café, square, traffic) recordings, 16 channels each, at 16 and 48 kHz. There is no plant, machinery or industrial environment. The Zenodo rights field says "Creative Commons Attribution 4.0 International" while the record description says CC BY-SA 3.0 Unported. Calling it "industrial" in a README aimed at engineers is a factual error, and ShareAlike (if that is the governing licence) would attach to any published mixed audio.
- Correction: describe it as "public environmental noise recordings (DEMAND)"; choose and name the environments (`DWASHING`, `TMETRO`, `PSTATION`, `STRAFFIC` are the loudest broadband ones), channel `ch01`, 16 kHz set. State both licence strings and treat the stricter (CC BY-SA 3.0) as governing; never commit or publish mixed audio; the demo page, if any, plays only the clean recordings. If genuinely industrial noise matters to the story, add MIMII (machine sounds: pump, fan, valve, slider; CC BY-SA 4.0 — **verify**) as a second source in a later condition and cite it.
- Severity: important

**[Incorrect assumption / underspecified] Deepgram parameters that decide the headline number are not fixed**
- Where: §4.4; §7 week 3.
- Problem: **verified** from Deepgram's docs: `keyterm` is supported on Nova-3 (monolingual and multilingual) and Flux, including streaming; `keywords` is the feature for other models such as Nova-2; keyterms are limited to 500 tokens per request (recommended 20-50 terms). The plan is right about the split but leaves out the two options that change numeric-token accuracy more than boosting does: `smart_format` (formats numbers as digits) and `numerals`. If the grid runs with `smart_format=true`, Deepgram's own formatting competes with the project's normalizer and the "boosting" comparison is confounded.
- Correction: fix in config and in the pre-registration: `model=nova-3` and `model=nova-2`, `smart_format=false`, `numerals=false`, `punctuate=false`, `interim_results=true`, `endpointing=300`, `encoding=linear16`, `sample_rate=16000`, `channels=1`; the normalizer is applied identically to references and hypotheses. Keyterm list = the vocabulary list committed with the utterance script before recording (not extracted from transcripts), ≤ 50 terms. Add a unit test that the adapter emits `keyterm` for `nova-3` and `keywords` for `nova-2` and refuses other combinations.
- Severity: important

**[Missing edge cases] The comparison methodology is unpaired and the metrics are not fully defined**
- Where: §4.3, §4.7; S9.
- Problem: Boosting on/off and noise conditions are measured on the same 200 utterances; the plan bootstraps each condition separately. Corpus WER versus mean per-utterance WER are different numbers and the plan does not say which. "Numeric-token accuracy: of the tokens that carry a number, the fraction recovered exactly" needs an alignment (a hypothesis with an inserted token shifts every position).
- Correction: define: numeric token = a normalized token matching `^[A-Z]+-\d+$` or `^\d+\.\d+$` or `^\d+$`; align normalized reference and hypothesis with `jiwer.process_words(ref, hyp).alignments`; a reference numeric token counts as correct only if aligned to an equal hypothesis token (substitution/deletion = error); denominator = reference numeric tokens. WER = corpus-level (total edits / total reference words) with `jiwer.wer_standardize` as the transform on the raw text and the numeric normalizer applied afterwards, both stated. Every published comparison uses the paired bootstrap from S9 (difference per utterance, resample utterances). Pre-registration must also state the minimum detectable difference at N (≈ 600 numeric tokens gives a ±4 pp interval on a single condition; say it).
- Severity: blocking

**[Places where an implementer would guess] `confidence` has no source, and the write path has no target**
- Where: §4.6 ("high-confidence records are written without confirmation"); §5 (`Reading.confidence: float`).
- Correction: `confidence` = the minimum Deepgram per-word confidence over the words that produced the numeric tokens (available in streaming results as `words[].confidence`); for the regex baseline on text input, `confidence = 1.0` if all fields parsed else `0.0`; for the LLM extractor, the same STT-derived value (LLM logprobs are not used). Threshold `0.90`, in config. "Written" means appended to `records.jsonl` at the path given by `--out`; records routed to review go to `review.jsonl` with the reason. State this in the README sentence the plan already requires.
- Severity: important

**[Missing requirements] The validator has no reference data and no evaluation, but the resume bullet quotes a rejection rate**
- Where: §4.6 rules ("below nominal minus tolerance", "implausible change versus the previous reading"); report bullet 3 ("validator rejected <N>% of implausible records before write").
- Correction: add `data/locations.json` (synthetic; `location_id, nominal_in, tolerance_in, previous_reading_in, previous_date`) generated by a committed script with stated parameters (S10). Design 20-30 utterances in the script whose truth reading is implausible against that table, label them `implausible: true` in the manifest, and report the validator's rejection rate on them (with the Wilson interval) plus its false-rejection rate on plausible ones. Otherwise delete the clause from the bullet.
- Severity: important

**[Incorrect assumption] `llama-bench` does not measure what the plan publishes, and imatrix does not apply to Q8_0**
- Where: §4.8; §6; §7 week 5.
- Problem: `llama-bench` reports prompt-processing and generation tokens/s as mean ± SD over `-r` repetitions; it does not report time-to-first-token or peak memory, and it does not take the "median of 10" the plan promises. Importance matrices are used by k-quants and i-quants; `Q8_0` ignores the imatrix, so "with and without imatrix" at Q8_0 is a null by construction. `convert_hf_to_gguf.py` needs `torch` and `transformers` (`requirements/requirements-convert_hf_to_gguf.txt`), a multi-GB install the plan has not budgeted. "Peak working set" is Windows vocabulary; in WSL2 the measure is `VmHWM`.
- Correction: benchmark harness = `llama-server` (pinned release tag, built in WSL2 with `cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j`) plus a Python client that sends the fixed extraction prompt 10 times per configuration and records `timings.prompt_ms`, `timings.predicted_per_second`, and TTFT from the first streamed token; peak memory = `VmHWM` from `/proc/<pid>/status` after each run; report median and IQR. Grid: `Q4_K_M ± imatrix`, `Q6_K ± imatrix`, `Q8_0` (no imatrix), F16 baseline. Imatrix: `llama-imatrix -m model-f16.gguf -f calib.txt -o imatrix.gguf` with `calib.txt` built from **dev-split** hypothesis transcripts only (using held-out text would leak into the retention measurement). Budget 2 h in week 5 for the conversion environment (`uv venv` inside the llama.cpp tree, CPU torch).
- Severity: important

**[Places where an implementer would guess] Offline serving mechanism and the zero-outbound test**
- Where: §4.8 (FastAPI service; "zero outbound network calls (socket-level assertion)"); §6 ("LLM client per provider choice").
- Problem: The plan never says how Python talks to the model. If the service spawns `llama-server` and calls it over HTTP on loopback, the socket-level test must allow loopback or it will fail on the first request; if it uses `llama-cpp-python`, that is an unlisted dependency with a native build.
- Correction: `serve/` starts `llama-server --model <gguf> --port 8081 --host 127.0.0.1 --ctx-size 2048 --threads <n>` as a subprocess and the extractor calls its OpenAI-compatible `/v1/chat/completions` with `response_format: {type: "json_schema", json_schema: <pydantic schema>}` so output is grammar-constrained. The cloud extractor uses the same client shape against the chosen provider's OpenAI-compatible endpoint, which resolves the "LLM client per provider choice" item with one adapter. Zero-outbound test: `pytest-socket` with `--disable-socket --allow-hosts=127.0.0.1,::1` around the full request path, plus an assertion that `socket.getaddrinfo` is never called for a non-loopback host.
- Severity: important

**[Overstated claim] "Served fully offline" describes only extraction; STT stays in the cloud**
- Where: report bullet 4; plan §4.8 (already notes "Local STT is not in scope").
- Correction: bullet and README wording: "extraction served fully offline (STT remains cloud) with a zero-outbound-calls test". The plan's boundary statement is correct; the bullet is not.
- Severity: important

**[Places where an implementer would guess] Dataset composition, split and manifest**
- Where: §4.1; §5.
- Correction: state: each utterance is read by every speaker (200 scripts × 3 speakers = 600 files, ≈ 75 MB) or the 200 are divided among speakers (≈ 67 each, 25 MB); the size estimate implies the latter, but per-speaker results at N≈67 with a 30% held-out slice give ≈ 20 held-out items per speaker, below the S9 n<20 floor — choose the 600-file design and state ≈ 75 MB. Split: 70/30 stratified by speaker and by utterance category, seed 20260901, `split_hash = sha256` of the sorted `id:split` lines, asserted in a test. Manifest JSON schema committed as `data/manifest.schema.json`. Utterance categories with counts: plain reading 80, location-only 20, unit-switch 20, self-correction 30, implausible 25, indication-flag 25 (adjust, but commit the counts before recording).
- Severity: important

**[Data model] `Reading` conflates truth and prediction, and the field name contradicts the unit**
- Where: §5.
- Correction: rename `thickness_in` to `thickness` with `unit` alongside, or convert to inches at extraction and drop `unit` from the record; define a separate `ReadingTruth` (no `confidence`, `source_transcript`, `prompt_version`) for `Utterance.truth_reading`. Normalizer output formats fixed: readings always `0.312` with leading zero and three decimals preserved as spoken; location IDs uppercase `A-12`; units `in|mm`. Self-corrections ("no wait") are resolved by the normalizer, not the extractor, with the rule "last complete value wins", and the exhaustive table includes them.
- Severity: minor

**[Security or privacy] Volunteers' voice recordings need their own licence and consent for public redistribution**
- Where: §4.1 ("consent recorded in the repo"); S4 (`LICENSE` MIT).
- Problem: MIT covers the code. Voice is identifying data; "consent" for research use is not consent for a public repository under an open licence.
- Correction: add `data/LICENSE-DATA.md` (CC BY 4.0 for the recordings and manifest) and a consent template that names the repository URL, the licence, the pseudonym, and the right to request removal; commit signed copies with names redacted. Scripts must contain no names or places.
- Severity: important

**[Places where an implementer would guess] Grid runtime, concurrency and latency definitions**
- Where: §4.4, §4.7.
- Correction: 2 models × 2 boosting × 3 conditions = 12 conditions × 600 files × ~4 s file-paced ≈ 8 h sequential; run with a concurrency of 5 sessions (**verify** Deepgram's concurrent streaming-connection limit for the account tier) and state it. TTFT = first interim result minus first audio byte sent; final latency = last `is_final` result minus the `CloseStream` send time; both from file pacing, labelled "network + API, not microphone". Spend cap counts seconds from the manifest before the run and refuses if `seconds × rate > budget`; unit-test the refusal.
- Severity: minor

**[Places where an implementer would guess] Pre-registration template and extraction-eval inputs**
- Where: §4.7; §4.5.
- Correction: template fields: factors and levels, primary metric, paired-bootstrap decision rule, minimum meaningful difference, minimum detectable difference at N, the null sentence, the git commit of the code that will run. Extraction is evaluated on the hypotheses of one canonical STT condition (`nova-3`, keyterm on, quiet) and secondarily on `snr_low`; state this. Per-field F1: a field is TP when predicted equals truth after normalization, FP when predicted non-null and unequal, FN when truth non-null and predicted null; taxonomy assignment rules committed as a decision table.
- Severity: important

**[Dependency and order] Recording depends on volunteers whose availability is outside the schedule**
- Where: §7 week 1; §10 risk row.
- Correction: write the utterance script and consent form in the last week of the preceding project (a two-hour task) and book both recording sessions before FieldVoice week 1 starts; treat this as the permitted passive-time exception in S11 or amend S11 to allow it.
- Severity: minor

**[Missing requirements] Riskiest-tool date; Dockerfile contents**
- Where: §7 week 5-6; §9.
- Correction: riskiest tool = llama.cpp conversion path; fallback date = week 5 day 4 — if F16 conversion is not producing a GGUF that `llama-cli` loads, ship cloud-only v1.0 and state it. Dockerfile: `python:3.12-slim` + a pinned `llama-server` release binary; the model file is mounted at `/models` and never baked in; CI builds the image and runs the zero-outbound test with a stub model.
- Severity: minor

**[Testing gaps]**
- Where: §8.
- Correction: add tests for (1) spend-cap refusal; (2) model→parameter selection (`keyterm` vs `keywords`); (3) confidence routing at the threshold; (4) that every LLM output validates against the pydantic schema (grammar constraint holds); (5) normalizer self-correction cases; (6) that `imatrix` calibration text contains no held-out utterance ids; (7) the validator on the labelled implausible set.
- Severity: important

**[Week budget] Week 3 breaks first; week 5 breaks second**
- Where: §7.
- Problem: Week 3 stacks the streaming adapter, file pacing, latency marks, a reconnect/backoff state machine with fake-transport tests, the spend cap, the pre-registration, an 8-hour grid run, and result tables. Week 5 stacks the WSL2 build, a torch install, conversion, five quantizations, two imatrix passes, a benchmark harness, and F1 retention on the held-out split.
- Correction: Week 3 = adapter + pacing + spend cap + pre-registration + grid + tables + tag; move reconnect/backoff and latency marks to week 4 (they are not on the measurement path). Week 5 = build + convert + three plain quantizations + benchmark; drop imatrix pre-emptively (cut-order item 3) and move F1 retention into week 6 alongside the service.
- Severity: important

---

## Cross-project

**[Conflict] The report commits to an outside PR reviewer; all three plans make it optional or undecided**
- Where: report CustodyLedger week 8 and skeptic answer; S14; `01` §7 "Optional, after v1.0"; `02`/`03` no mention.
- Problem: The Stripe answer in the report rests on it; none of the plans schedule it. Leaving it "undecided" means it will not happen.
- Correction: decide in S14 now. If yes: in every plan, open the first reviewed PR by week 2 (a classmate or a Discord dev community) and treat "five reviewed PRs" as a v1.0 criterion; if no: delete the Stripe skeptic answer from the report and state plainly in the Stripe packet that the repos are sole-authored.
- Severity: important

**[Conflict] Report text contradicts the plans in six places; the plans are right in each**
- Where: report vs plans.
- Problem: (1) idempotency triple constraint (see CustodyLedger); (2) `COM_DISARM_LAND` as a threshold source (see Flightcheck); (3) Plotly on Pages vs Matplotlib; (4) "DEMAND industrial-noise dataset"; (5) "served fully offline"; (6) "root-cause memo on a public failsafe event". Any README, `SECURITY.md` or resume text drafted from the report will carry these.
- Correction: add to S5 or S12: "README and resume text are written from the plan, never from the report; the report is superseded where they differ", and correct the report's six phrases before it is reused.
- Severity: important

**[Duplicated effort] Shared statistics are re-implemented in two repos with no shared test vectors**
- Where: S9 ("They do not share code").
- Correction: keep separate implementations, but commit one `stats-vectors.json` (the Wilson and bootstrap cases from the Shared section above) to both repos with a note that it is copied verbatim; drift between copies is then visible.
- Severity: minor

**[Conflict] Week-0 "nothing committed to main" vs S4 "public from day 1, first commit is the scaffold"**
- Where: `02` §7 week 0; S4.
- Correction: week 0 begins with the scaffold commit (README stub, `pyproject.toml`, CI with one trivial test) on day 1; the scratch script lands in `notebooks/` in the same week. Either amend S4 to exempt week 0 or, better, keep S4 and make the scaffold the first hour of week 0.
- Severity: minor

**[Dependency and order] S11 gate and application windows are not checked in the plans**
- Where: S11 ("Start immediately if Zipline requisitions are still open"); each plan's "tag v0.1 → repo link goes on applications".
- Problem: Whether the Zipline requisitions are still open decides the whole ordering, and it is a five-minute check that no plan assigns to anyone or dates.
- Correction: add to S11: "checked on the day before the first scaffold commit; result recorded in `planning/00-shared.md` with the date".
- Severity: minor

---

## Resume-bullet supportability (report bullets vs plan deliverables)

- CustodyLedger bullet 2 ("deleting ... any audit record fails `verify`"): not supportable until the head anchor is added (tail deletion). Bullet 3 ("lot reconciliation enforced by database constraints"): not supportable as written; add the `reconciliations` CHECK or reword. Bullets 1 and 4: supportable.
- Flightcheck bullet 3 ("root-cause memo on a public failsafe event"): conditional on the memo's subject; keep two wordings. Bullet 4 (precision/recall, solar slicing): both are cut-order items 1 and 2; the clause must be deleted if cut. Bullets 1 and 2: supportable once the golden-suite tiers are defined.
- FieldVoice bullet 2 ("keyterm boosting cut numeric-token error from X to Y"): supportable only with the paired bootstrap and only if the interval excludes zero; the null wording must be pre-written. Bullet 3 ("validator rejected N% of implausible records"): not supportable without the labelled implausible set. Bullet 4 ("served fully offline"): overstated; qualify to extraction. Bullet 1: supportable.

## Budget verdict at ~10 h/week

- CustodyLedger: week 2 breaks first, week 4 second; moves listed above. Eight weeks hold only if v0.1 shrinks to two event types and the denial suite is split across weeks 4 and 7.
- Flightcheck: week 1 breaks first, week 5 second; the model baseline should be assumed cut from the start and reinstated only if week 5 finishes early.
- FieldVoice: week 3 breaks first, week 5 second; imatrix should be assumed cut from the start.

## Claims that would be false or overstated if the plans ship as written

1. "Editing, deleting, or reordering any audit record fails `verify`" (tail deletion; column edits).
2. "Two-person issuance and lot reconciliation enforced by database constraints" (role and pending-state checks are handler code; no reconciliation constraint exists).
3. "Thresholds derived from each vehicle's own configured parameters" using `BAT_N_CELLS`/`BAT_V_EMPTY` on post-v1.11 logs (resolves to nothing) or `GF_MAX_HOR_DIST` without a sentinel (fails everything).
4. "DEMAND industrial-noise dataset" (not industrial).
5. "Served fully offline" (extraction only).
6. "C++ checker independently agrees" (independent for one metric only).
