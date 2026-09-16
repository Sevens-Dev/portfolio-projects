# Project 1: CustodyLedger

Shared context in `00-shared.md`. Sections below reference it as S1-S14 rather than repeating it.

## 1. Goal and final deliverable

A deployed, sole-authored web service that models a controlled-materials chain-of-custody regime as an append-only, tamper-evident ledger, and proves three properties with runnable tests:

1. **Tamper evidence.** Editing, deleting, or reordering any audit record causes `verify` to fail and to name the first bad record and the balances it invalidates.
2. **Exactly-once writes.** N concurrent requests sharing one idempotency key produce exactly one ledger row.
3. **Authorization that holds under attack.** A denial suite covering IDOR, horizontal and vertical escalation, and self-approval passes.

Final deliverable: public repo + live HTTPS URL with seeded demo accounts per role + a second implementation (Python) of the chain verifier that agrees with the first.

Headline README numbers: total test count, denial-suite case count, the storm result (`1000 concurrent → 1 row`), and `verify` detection on a mutated byte.

## 2. Existing state

Greenfield. Reusable knowledge only: React Hook Form + Zod from the developer's prior multi-step form project (same two libraries are used here, deliberately, so the earlier project reads as a progression). No TypeScript, Node, Fastify, or Postgres-administration experience. See S2.

## 3. Required architecture

Single repository, two deployables, one shared schema source.

```
custodyledger/
  server/        Node 22 + Fastify + node-postgres (`pg`)
  web/           Vite + React 18 + TypeScript
  shared/        Zod schemas + canonical-encoding spec, imported by both
  verifier/      Independent Python CLI (separate toolchain, uv)
  db/            Numbered .sql migrations, applied by a tiny runner script
  docs/adr/
```

Deliberate exclusions, each an ADR:

| Excluded | Reason |
| --- | --- |
| pnpm/npm workspaces | Path aliases in two `tsconfig.json` files achieve the same for two packages; workspace tooling is a day of setup for a solo repo |
| ORM (Drizzle, Prisma) | Raw SQL + numbered migrations. The developer already reads SQL; an ORM adds a learning curve and obscures the CHECK constraints that carry the project's main argument |
| TanStack Query | Two screens, `fetch` + `useState` is sufficient |
| Local Docker / Testcontainers | Neon for local dev, a `postgres:16` service container in CI. Removes Docker Desktop from the critical path (S3) |
| JWT | Server-side sessions in a `sessions` table. Revocation is trivial and the ADR explains the tradeoff |
| Ed25519 signatures, field-level AES-GCM | HMAC is sufficient for the threat model; more primitives means more ways to be subtly wrong |

**Trust model.** Single-tenant, one organization. Adversaries modeled: (a) a database administrator who edits or deletes history directly in SQL, (b) an authenticated handler who tries to act on another handler's lots or approve their own issuance, (c) a network client that replays a write. The HMAC key lives in the application environment, not the database; this is the specific property that makes (a) detectable, and the threat model must state plainly that an attacker with both database and application-environment access can forge a consistent chain.

## 4. Major components

### 4.1 Domain model

Physical regime being modeled: lots of controlled material are received into a magazine, issued to a licensed handler under a two-person rule, transferred between handlers, consumed, returned, and periodically reconciled.

Entities: `User`, `Role`, `Session`, `Lot`, `CustodyEvent`, `AuditRecord`, `IdempotencyKey`.

Lot lifecycle (exhaustively tested state machine):

```
open ──issue──▶ issued ──consume/return──▶ issued
  │                │
  │                └──begin_reconcile──▶ reconciling ──┬──balanced──▶ closed
  └──begin_reconcile──▶ reconciling                    └──unbalanced──▶ discrepancy
```

Invariant enforced at every transition: `on_hand = received − consumed − net_transferred_out`, and `on_hand >= 0`. A close is refused unless the ledger balances.

### 4.2 Audit chain

Each `CustodyEvent` write appends exactly one `AuditRecord` inside the same transaction.

```
AuditRecord {
  seq            bigint       -- gapless, per-ledger, assigned inside the transaction
  ts_ms          bigint       -- integer epoch ms, never a formatted timestamp
  actor_id       uuid
  event_type     text
  entity_type    text
  entity_id      uuid
  payload_sha256 bytea        -- SHA-256 of the canonical payload
  prev_hash      bytea        -- 32 zero bytes for seq = 1
  record_hmac    bytea        -- HMAC-SHA256(key, signed_bytes)
  signed_bytes   bytea        -- the exact bytes that were signed, stored
}
```

**Canonical encoding is specified in `SECURITY.md` before any code is written.** Use RFC 8785 JSON Canonicalization Scheme (npm `canonicalize`, PyPI `rfc8785`). `signed_bytes` is the JCS serialization of `{seq, ts_ms, actor_id, event_type, entity_type, entity_id, payload_sha256_hex, prev_hash_hex}` with keys in JCS order. Storing `signed_bytes` alongside the HMAC is deliberate: it makes the cross-language verifier a pure hash check rather than a re-serialization gamble, and it is the reason a schema addition cannot silently invalidate history.

Five canonical test vectors (input record → expected `signed_bytes` hex → expected HMAC hex with a fixed test key) are committed as JSON and consumed by both the TypeScript and Python test suites. These vectors are the contract between the two implementations.

### 4.3 Integrity commands

- `verify` — walks the chain from seq 1, recomputes each HMAC and each `prev_hash` link, stops at the first mismatch, reports its seq, its type (`mutated` | `deleted` | `reordered` | `chain_break`), and which lot balances depend on records at or after that point.
- `replay` — rebuilds every lot balance from the event log alone and compares to the materialized `lots` table. Any divergence is an error.

Both exist in the TypeScript server as CLI entry points and `replay` additionally as an admin-only HTTP endpoint. `verify` also exists independently in Python (§4.7).

### 4.4 Idempotency

`idempotency_keys(actor_id, key, body_sha256, response_status, response_body, created_at)` with `UNIQUE (actor_id, key)`.

Write path, inside one transaction:
1. `INSERT ... ON CONFLICT (actor_id, key) DO NOTHING RETURNING *`.
2. If a row was inserted, perform the write, store the response, commit.
3. If not, read the existing row. Same `body_sha256` → return the stored response. Different `body_sha256` → `409`.

This uses the unique constraint as the concurrency primitive. No advisory locks (an earlier draft used them; they were the wrong tool and the ADR records why).

### 4.5 Two-person rule

Issuance requires a requester and a distinct approver. Enforced in the schema, not in handler code:

```sql
ALTER TABLE issuances ADD CONSTRAINT two_person
  CHECK (approver_id IS NULL OR approver_id <> requester_id);
ALTER TABLE issuances ADD CONSTRAINT approved_issuances_complete
  CHECK (status <> 'issued' OR approver_id IS NOT NULL);
CREATE UNIQUE INDEX one_open_issuance_per_lot
  ON issuances (lot_id) WHERE status = 'pending';
```

A regression test demonstrates that an application-layer-only version of this rule can be defeated, and that the constraint version cannot. This test is the artifact; keep it.

### 4.6 Authorization

Role-based with object-level checks. Roles: `handler`, `keeper`, `auditor`, `admin`. A Fastify preHandler resolves the session; each route declares its required role **and** an ownership predicate evaluated against the specific row, not just the route.

The denial suite (target 30+ cases) is a first-class deliverable, organized by attack class: object reference (handler A reads/mutates handler B's lot), vertical escalation (handler performs keeper/admin actions), approval bypass (self-approval, approving without the keeper role, approving an already-issued lot), session handling (missing token, expired token, token for a deleted user), and idempotency abuse (replayed key with a different body).

### 4.7 Independent Python verifier

`verifier/` is a separate `uv` project, `mypy --strict`, pytest. It consumes an export (`GET /admin/export` → newline-delimited JSON of audit records plus lot snapshots) and independently:
- recomputes the chain from `signed_bytes` and the HMAC key,
- re-derives every balance,
- passes the five shared test vectors.

CI runs it against a freshly seeded database on every push. Purpose is threefold: it puts real, typed Python behind a target posting's "C, C++, Java or Python" essential qualification; it proves the canonical encoding is specified well enough for a second implementation; and it is the strongest available argument that the spec is not merely whatever the first implementation happened to emit.

### 4.8 Web client

Two screens only. (1) Custody operations: issue with approval, transfer, consume, return, reconcile. (2) Audit view: the chain with per-record verify status, and a control that runs `replay` and displays the diff. React 18 + TypeScript strict, React Hook Form + Zod resolvers against the shared schemas, plain `fetch`. No component library; hand-written CSS with tokens.

### 4.9 Second vocabulary

The same schema and code serve a second seeded demo: two-nurse controlled-substance counts in a skilled-nursing context, selectable by a seed flag. Costs roughly half a day (a seed file plus label strings) and makes the healthcare-adjacent target postings concrete rather than analogical. The README presents it as a second vocabulary over one ledger, not as a second product.

## 5. Data models and interfaces

### 5.1 Schema (abbreviated; full DDL in `db/001_init.sql`)

```sql
users(id uuid pk, email citext unique, password_hash text, role text, created_at timestamptz)
sessions(id uuid pk, user_id uuid fk, expires_at timestamptz, revoked_at timestamptz)
lots(id uuid pk, code text unique, material text, unit text,
     qty_received numeric(12,3), qty_on_hand numeric(12,3),
     status text, opened_at timestamptz, closed_at timestamptz)
issuances(id uuid pk, lot_id uuid fk, requester_id uuid fk, approver_id uuid,
          qty numeric(12,3), status text, created_at, approved_at)
custody_events(id uuid pk, lot_id uuid fk, type text, qty numeric(12,3),
               from_user uuid, to_user uuid, actor_id uuid, payload jsonb, created_at)
audit_records(seq bigserial pk, ts_ms bigint, actor_id uuid, event_type text,
              entity_type text, entity_id uuid, payload_sha256 bytea,
              prev_hash bytea, record_hmac bytea, signed_bytes bytea)
idempotency_keys(actor_id uuid, key text, body_sha256 bytea,
                 response_status int, response_body jsonb, created_at,
                 primary key (actor_id, key))
```

Quantities are `numeric`, never floating point. A test asserts that a float quantity is rejected at the schema boundary; this is one of the AI-introduced defect classes to watch for (S7).

`audit_records` has no `UPDATE` or `DELETE` grant for the application role. A migration creates a restricted role and the application connects as it.

### 5.2 HTTP interface

All routes under `/api`. Mutating routes require an `Idempotency-Key` header and return `201` with the created event plus its audit seq.

```
POST   /api/auth/login                     → session cookie
POST   /api/lots                           keeper        receive a lot
POST   /api/lots/:id/issuances             handler       request issuance
POST   /api/issuances/:id/approve          keeper        approve (two-person rule)
POST   /api/lots/:id/transfer              handler       to another handler
POST   /api/lots/:id/consume               handler
POST   /api/lots/:id/return                handler
POST   /api/lots/:id/reconcile             keeper        begin/complete reconciliation
GET    /api/lots, /api/lots/:id            role-scoped
GET    /api/audit?from=&to=                auditor|admin
POST   /api/admin/verify                   admin         runs chain verification
POST   /api/admin/replay                   admin         rebuild-and-compare
GET    /api/admin/export                   admin         NDJSON for the Python verifier
GET    /healthz                            public
```

OpenAPI is generated from the Zod schemas (`zod-to-openapi`) and committed, so the interface is reviewable without reading route code.

### 5.3 Error contract

RFC 9457 problem+json. Every error has a stable `type` slug. Authorization failures return `404` where leaking existence would be an information disclosure (handler B asking for handler A's lot), `403` where the resource's existence is already known to the caller. The denial suite asserts the distinction, and `SECURITY.md` explains it.

## 6. Dependencies

Server: `fastify`, `@fastify/cookie`, `@fastify/helmet`, `@fastify/rate-limit`, `pg`, `zod`, `canonicalize`, `@node-rs/argon2`, `pino`.
Test: `vitest`, `supertest` (or `fastify.inject`), `fast-check`.
Web: `react`, `react-dom`, `vite`, `react-hook-form`, `@hookform/resolvers`, `zod`.
Verifier: `pytest`, `mypy`, `rfc8785`.
Tooling: `typescript`, `eslint`, `prettier`, `k6` (CLI, not a package), `semgrep` (CI only).

Pin exact versions in a committed lockfile. No dependency is added after week 5 without deleting another.

## 7. Implementation phases

Each phase is one week at ~10 h. Ordering is by dependency; the cut order in §11 governs what is dropped if a week is lost.

**Week 1 — spec, scaffold, deploy path.**
Write `SECURITY.md` §canonical-encoding and the five test vectors *first*, by hand, before any code. Scaffold `server/` and `web/`. `db/001_init.sql` plus the migration runner. Provision Neon, connect. CI green with lint, typecheck, and one trivial test. Deploy a `/healthz` endpoint to the host so that deployment is proven on day one rather than discovered in week 8. Exit: a URL returns `{"ok":true}`, CI is green, the encoding spec is committed.

**Week 2 — ledger core and chain.**
Custody event writes (receive, transfer, consume, return) inside transactions. Audit record append with HMAC over JCS bytes. Idempotency via the unique constraint. `verify` and `replay` CLIs. Tests: mutate one byte → verify fails at the right seq; delete a record → detected; swap two records → detected; replay reproduces balances. **Tag `v0.1`** — the repo link goes on applications now. Note that `v0.1` deliberately has no authentication; the README at this tag says so.

**Week 3 — state machine and invariants.**
Lot lifecycle transitions with an exhaustive transition table test (every state × every event → expected state or rejection). Reconciliation that refuses an unbalanced close. `fast-check` properties: for any valid event sequence, `on_hand >= 0`; `received = consumed + returned + on_hand + net_out`; replaying the log reproduces the materialized state.

**Week 4 — identity and authorization.**
Argon2id password hashing, session table, cookie handling with `HttpOnly`/`Secure`/`SameSite=Strict`. RBAC preHandler plus per-route ownership predicates. Two-person rule constraints and the demonstration test from §4.5. The 30-case denial suite. Rate limiting and security headers.

**Week 5 — web client.**
The two screens. Zod schemas shared with the server. Form validation, error rendering from problem+json. Integration tests against the CI Postgres service container. **Tag `v0.5`** — the project is presentable from here even if weeks 6-8 are lost.

**Week 6 — independent verifier.**
`verifier/` Python package, `mypy --strict`, consuming `/admin/export`. Passes the five shared vectors. Wired into CI against a seeded database. This is the week most likely to surface an under-specified encoding; that discovery is the point.

**Week 7 — operations and measurement.**
k6 storm: 1000 concurrent requests, one idempotency key, assert exactly one row; publish p50/p95/p99 under a stated RPS with the S6 attributes. Structured logging, `RUNBOOK.md`, second seeded vocabulary (§4.9). Induce one real failure (connection-pool exhaustion under the storm is the likely one), diagnose it, fix it, and write `POSTMORTEM.md` with the fixing commit linked.

**Week 8 — evidence and close.**
`SECURITY.md` completed (data-flow diagram, trust boundaries, STRIDE table, each threat mapped to a mitigation *and* a test, accepted risks). Three ADRs minimum (§3 table). `AI-USAGE.md` per S7. README rewritten to S5. Demo video, 2-3 minutes. **Tag `v1.0`.**

Optional, in priority order, only after v1.0: recruit an outside PR reviewer (S14); one merged upstream PR to a dependency; an OWASP ZAP baseline scan with findings remediated.

## 8. Testing and validation

| Layer | Tool | What it must prove |
| --- | --- | --- |
| Unit | Vitest | Encoding, hashing, balance arithmetic, state transitions |
| Property | fast-check | Ledger invariants over arbitrary valid event sequences |
| Integration | supertest + CI Postgres | Full request path including transactions and constraints |
| Cross-language | pytest | The Python verifier agrees with the server on a real export and on the five vectors |
| Adversarial | Vitest | The 30-case denial suite |
| Tamper | Vitest | Mutate / delete / reorder each detected, at the correct seq |
| Load | k6 | Exactly-once under 1000-way concurrency; latency percentiles |
| Static | Semgrep, npm audit | CI gates; at least one commit that fixes a real finding |

Coverage is reported with a threshold enforced in CI. The threshold is a floor to prevent regression, not a target to optimize.

## 9. Deployment and setup

- Database: Neon free tier, one project, two branches (`main`, `dev`).
- API: Render or Fly.io, smallest instance (S14 open item). `HMAC_KEY`, `DATABASE_URL`, `SESSION_SECRET` as platform secrets.
- Web: static build served by the same origin to avoid CORS entirely (an ADR records this over a separate static host).
- Migrations run on deploy via a release command; the runner is idempotent and records applied versions in a `schema_migrations` table.
- Seeding: `npm run seed -- --vocabulary=explosives|nursing` creates demo users for each role with published credentials (the README states plainly that this is a demo instance with synthetic data).
- Key rotation is out of scope for v1.0 but must be named as an accepted risk in `SECURITY.md`, with the migration path sketched (`key_id` column reserved in `audit_records`).

## 10. Known risks and unresolved decisions

| Risk | Handling |
| --- | --- |
| Largest of the three projects and almost the entire stack is new | Week 1 proves deployment; `v0.1` at week 2 and `v0.5` at week 5 mean the resume value does not depend on finishing |
| Cryptography subtly wrong (signing the wrong bytes, unstable serialization) | Spec before code; store `signed_bytes`; five committed vectors; cross-language verification; library primitives only |
| Free-tier cold starts distort latency numbers | Run the storm three times, disclose instance size, state whether the first run was a cold start |
| Postgres `numeric` handling in JS (`pg` returns strings) | Decided deliberately: keep them as strings and use a decimal library or integer minor units. Do **not** let them become JS numbers. A test asserts this |
| Sole authorship does not satisfy one target posting's stated minimum | S14; either an outside reviewer or an upstream PR, scheduled post-v1.0 |
| Demo instance with zero users invites "it's a demo, not software" | Accept and state it. Optionally run a small pilot with former colleagues and tie two changelog entries to their feedback |

**Unresolved:** hosting provider (S14); whether `GET /admin/export` should stream (it should if the audit table exceeds ~50k rows, which the seed does not reach, so defer); whether to include the nursing vocabulary at all if week 7 is tight (it is the first thing cut after the optional items).

## 11. Cut order

If a week is lost, cut in this order and state the cut in the README rather than leaving a hollow claim: (1) nursing vocabulary, (2) demo video, (3) `POSTMORTEM.md` (only if no genuine failure occurred), (4) rate limiting and security headers, (5) the k6 storm's latency table (but never the exactly-once assertion).

Never cut: the canonical-encoding spec, the tamper tests, the denial suite, the Python verifier, the two-person constraint test.

## 12. Definition of done

S12, plus:
- `verify` detects all three tamper classes with a test per class, each naming the correct seq.
- The Python verifier and the TypeScript server agree on a real export in CI.
- 30+ denial cases pass, organized by attack class.
- The storm test asserts exactly one row from 1000 concurrent same-key requests, with the result committed.
- The live URL serves both screens with seeded per-role accounts.
- `SECURITY.md` maps every identified threat to both a mitigation and a test.
