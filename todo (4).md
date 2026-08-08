# Enterprise ERP: Incremental Complete-Delivery TODO (Precision Edition)

> This file supersedes any earlier version. It is written for an implementation model with **less
> judgment than a senior engineer**. Every ambiguity in the previous draft that could be
> misread as permission to simplify, substitute, or skip has been closed. Nothing in this
> revision changes scope, technology, or hardware — it only removes room for misinterpretation
> and adds numbers where the previous draft had adjectives.

## 0. Read this before touching any file

1. Implement tasks **strictly in numeric order** (1 → 33). Do not start task *N+1* until task
   *N*'s acceptance criteria all pass and are demonstrated by a command you actually ran.
2. This is **one complete production ERP**, not an MVP, demo, or optional roadmap. Every bullet
   under a task is in scope for that task — bullets are not "nice to have" extensions.
3. Start from a clean repository. Do not import scaffolding, boilerplate, or starter templates
   that bring extra dependencies, extra services, or a different architecture than Section 1.
4. Do not invent requirements outside this file, and do not remove or water down a requirement
   written here because it looks hard. If a requirement seems contradictory or physically
   impossible on the stack in Section 1, stop and report the conflict instead of silently
   dropping it.
5. Never delete or "clean up" a previously completed task's tests, migrations, or code to make a
   later task easier. Extend, don't erase.
6. **Task response format — for every task, report only:**
   - Files changed (paths only, no diffs unless asked).
   - One concise implementation note (2–5 sentences).
   - The exact commands you ran (verbatim, copy-pasteable).
   - The test result (pass/fail counts, not full logs).
   - The next task number.
   Do not reprint this specification. Do not narrate intentions ("I will now…") — report what
   was done.
7. Prefer existing helpers/interfaces already created in earlier tasks over inventing new
   abstractions. Before writing a new function, grep the repository for one that already does it.
8. **All system-level dependencies are already installed on the VM by the user** — see Section
   2.5. Never run `apt`/`apt-get`, never compile anything from source, never download a
   database/runtime binary, and never attempt to install, upgrade, or downgrade a system
   dependency. Never spend turns debugging a missing system package. If a Section 2.5
   verification command fails, or `pip`/`npm install` reports that a package needs to compile
   because no prebuilt wheel/binary exists for this platform, **stop immediately and output one
   short message asking the user to run a specific command in their own terminal** — name the
   exact package and the exact command. Do not attempt a workaround, a substitute package, or a
   different version. This is what keeps token spend on this project low: installation
   troubleshooting is a human's one-command job, not a model's multi-turn investigation.

## 1. Locked technology stack — do not add, remove, or substitute anything here

**Forbidden, under any justification:** Docker/containers/Kubernetes, a microservices split, a
message broker (RabbitMQ/Kafka/SQS/NATS/etc.), a NoSQL database, a BPM/workflow product, a
separate reporting/analytics warehouse, a separate search cluster (Elasticsearch/OpenSearch/
Meilisearch/etc.), and any frontend state-management framework (Redux, Zustand, Pinia,
MobX, XState, etc.) — Svelte's own reactivity is the only state layer.

**Required, exactly this and nothing else:**

| Layer | Technology | Notes for the implementation model |
|---|---|---|
| Frontend framework | SvelteKit 2 on Svelte 5 (runes API), TypeScript | Do not write Svelte 4 `export let` / stores-only style component code; use runes (`$state`, `$derived`, `$effect`). |
| Frontend components | Bits UI (headless primitives) + Tailwind CSS v4 | Install Tailwind v4 via the `@tailwindcss/vite` plugin and a single `@import "tailwindcss";` in `app.css`. Do not add `tailwind.config.js`-only v3-style setup. |
| Data grids / tables | AG Grid Community and/or TanStack Table | AG Grid **Community** edition only — never reference or import an Enterprise-only module (`RowGroupingModule`, `ServerSideRowModelModule`'s enterprise variant, pivoting, etc.). If a required grid behavior needs an AG Grid Enterprise module, implement it with TanStack Table instead. |
| Charts | Apache ECharts | Load ECharts core plus only the chart/renderer modules actually used (tree-shaken build), not the full bundle, to protect the memory budget in Section 2. |
| API framework | FastAPI | ASGI only; run under Uvicorn (see Section 2 for worker count). |
| Domain/application layers | Plain Python (no framework) | Domain code must not import `fastapi` or `sqlalchemy`; this is enforced by a forbidden-import test (Task 2). |
| ORM / persistence | SQLAlchemy 2.x (2.0-style `Mapped`/`mapped_column`, async engine) | Use Alembic for all schema migrations. Never hand-edit a database schema outside a migration. |
| Database | PostgreSQL 18 | PostgreSQL 18 is the current stable major release (first released September 2025, patch releases ongoing). Its async I/O subsystem, `uuidv7()`, virtual generated columns, and OAuth 2.0 authentication support are all usable. Pin the exact minor version in the deployment runbook (Task 32) and re-verify it is still a supported minor at deploy time — do not pin to a version older than 18.0. |
| Cache | Valkey | Valkey is the maintained, BSD-licensed, Linux-Foundation-governed fork used as the cache layer — treat it purely as a cache (metadata/lookup/session fragments per Task 31), never as a system of record. Pin a current stable release line at deployment time; do not use a release candidate. |
| Full-text search | PostgreSQL FTS | No external search engine. All search in Task 25 runs through Postgres `tsvector`/`tsquery` plus trigram/GIN indexes as needed. |
| Object storage | S3-compatible storage (any provider behind the S3 API) | Access only through the storage adapter interface built in Task 25; never hardcode a provider-specific SDK call outside that adapter. |
| Background jobs | ARQ (Redis/Valkey-backed asyncio job queue) | **Caution for the implementation model:** ARQ is in maintenance-only mode upstream (bug fixes only, no new features) as of 2026. This does not authorize switching to another queue library. Pin an exact ARQ version in the dependency manifest, write your own thin wrapper around its public API (enqueue, cron, retry, job status) so a future forced migration touches one module, and do not depend on any ARQ behavior that isn't documented in its stable API. Only if a specific job genuinely cannot be expressed in ARQ (documented in the implementation note for that task) may Celery be used for that one job, and only that one. |
| Auth | Username/password plus secure session or rotated JWT | No third-party auth-as-a-service product; OIDC/SAML/SCIM in Task 8/21 are protocol *interfaces* your code implements, not hosted services you depend on. |

**Initial deployment topology (fixed, do not change):** one SvelteKit/FastAPI process group
(reverse-proxied) plus one separate ARQ worker process, on a single 4 GB RAM VM, backed by
PostgreSQL, Valkey, and S3-compatible storage. No Docker, no container runtime, no
orchestrator, no second VM. If a task's acceptance criteria seem to require a second VM or a
container, you have misread the task — re-read Section 2 and the task's non-negotiable
cautions instead of adding infrastructure.

## 2. 4 GB RAM budget — hard ceilings, not suggestions

The VM has 4096 MB total RAM and no swap-backed elasticity to rely on. Every long-running
process gets an explicit ceiling. These numbers are the default configuration to ship; a task
may tighten them further under load testing (Task 31) but must never raise a ceiling without
recording the new total and confirming it still fits.

| Process | Ceiling | Configuration mechanism |
|---|---|---|
| Operating system + system daemons | 400 MB | N/A (baseline, do not tune) |
| PostgreSQL 18 | 1024 MB working set | `shared_buffers = 768MB`, `work_mem` sized so `work_mem * max_connections` stays under the remaining headroom (start at `work_mem = 4MB`), `max_connections = 40`, connection pooling at the application layer so FastAPI never opens more than ~20 live connections concurrently. |
| Valkey (cache only) | 256 MB | `maxmemory 256mb`, `maxmemory-policy allkeys-lru`. Never store data here that cannot be safely evicted. |
| FastAPI under Uvicorn | 400 MB | Exactly **2** Uvicorn worker processes (`--workers 2`) unless load testing in Task 31 proves 1 is sufficient and safer; document the chosen count and why. |
| ARQ worker | 300 MB | `max_jobs` (concurrent job slots) set low enough that peak concurrent job memory stays under this ceiling; start at 5 and justify any change with a measurement. |
| SvelteKit (Node adapter SSR process, if SSR is used) | 250 MB | Use `adapter-node`; if a static/CSR-only route set is sufficient for a given surface, prefer it over SSR to save memory, but never at the cost of a required accessibility or SEO behavior explicitly required elsewhere in this file. |
| Reverse proxy / TLS termination (Task 32) | 50 MB | Lightweight non-Docker reverse proxy (e.g., nginx or Caddy as a system service). |
| **Reserved headroom** (OS page cache, import/export spikes, PDF generation, backup jobs) | ≥ 1300 MB | Never design a feature that assumes it can consume this headroom continuously; spikes must be transient and bounded (see Task 31 export/import limits). |

Enforce these ceilings with real mechanisms, not documentation alone: systemd `MemoryMax=` per
service unit (Task 32), PostgreSQL and Valkey config files committed to the repository with
placeholder-free real values, and Uvicorn/ARQ startup flags checked into the process
definitions. Task 31's acceptance criteria require proving no feature exceeds this budget under
load, not just stating it doesn't.

## 2.5 Pre-installed environment — verify once, never install

The target VM is Debian 12 (Bookworm), kernel 6.6, 2 vCPU (AVX2 + AVX-512, x86_64), 4 GB RAM, no
swap, 10 GB root disk, Docker unavailable. The user has already installed every system-level
dependency below, using the versions shown, before handing this repository to the implementation
model. **Do this exactly once, at the start of Task 1, and never again:** run the verification
commands below, paste the raw output into `docs/implementation-baseline.md` under an
"Environment verification" heading, and proceed. Do not re-run this check at the start of every
later task — that wastes tokens on a fact that does not change during the project.

| Component | Installed as | Verification command | Expected |
|---|---|---|---|
| OS | Debian 12 Bookworm | `cat /etc/debian_version` | `12.x` |
| Python | CPython 3.11.2 (system) | `python3 --version` | `Python 3.11.2` |
| pip | 23.0.1 or newer (upgraded in venv) | `pip --version` | `pip 2x.x` inside `/opt/erp/venv` |
| PostgreSQL | 18.x server + client, via PGDG | `psql --version` and `pg_lsclusters` | `psql (PostgreSQL) 18.x` |
| Valkey | via `bookworm-backports` | `valkey-server --version` | `Valkey server v=8.x` or newer |
| Node.js | 22.x LTS, via NodeSource (build tooling only, not a runtime dependency of the shipped backend) | `node --version` | `v22.x` |
| npm | bundled with Node 22 | `npm --version` | `10.x` |
| nginx | Debian main repo | `nginx -v` | any current stable |
| Backend virtualenv | `/opt/erp/venv`, created from system Python 3.11.2 | `source /opt/erp/venv/bin/activate && python --version` | `Python 3.11.2` |
| Backend Python packages | pinned in `requirements.lock.txt`, installed from prebuilt wheels only (no compiler was needed) | `pip freeze` inside the venv | matches `requirements.lock.txt` |

Always develop and run the backend inside `/opt/erp/venv` (`source /opt/erp/venv/bin/activate`)
— never install a Python package into the system Python, and never create a second virtual
environment. Always read exact package versions from the committed `requirements.lock.txt` and
`frontend/package-lock.json` rather than assuming a version from memory; if a task needs a
capability not covered by an already-installed package, check whether an already-approved
package (Section 1) already provides it before asking the user to add a new one — adding a new
third-party package still requires the user to run the install command themselves per rule 8 in
Section 0.

## 3. Non-negotiable implementation cautions

- Implement and test the safe metadata-rule expression engine (Task 4) **before** enabling any
  metadata rule anywhere else in the system. Its grammar must preserve binary operators for
  expressions such as `qty * price`; cover arithmetic, comparison, boolean precedence, nulls,
  limits, and malicious input.
- PostgreSQL 18 and Valkey (Section 1) are the only product targets. Do not weaken the design —
  e.g., dropping a constraint, an index, or an RLS policy — because a local development machine
  happens to run an older substitute. If local dev uses an older version for convenience, the
  gap must be documented and the CI/staging environment must run the real target version.
- Never hardcode database URLs, usernames, passwords, JWT secrets, or storage paths anywhere in
  source, tests, migrations, or documentation. Use required environment variables or secret
  files, and use safe placeholder-only examples (e.g., `postgresql://CHANGE_ME`) in any
  committed example file.
- Do not declare RLS, audit, outbox, or accounting "complete" on the basis of Python classes
  alone. Each requires: a migration that creates the real database object, the actual database
  privilege/constraint enforcing it, an integration test that proves enforcement against a live
  database, and a failure/retry test where relevant.
- Never install, compile, upgrade, or downgrade PostgreSQL, Valkey, Node.js, Python, or any
  system package. All of it is pre-installed per Section 2.5. A missing or wrong-version system
  dependency is always reported to the user for a one-command fix, never worked around.

## 4. Global invariants (apply to every task from here on)

- **Clean Architecture:** UI/API layers contain no ERP decisions. Domain code imports neither
  FastAPI nor SQLAlchemy. Repositories only query/persist. Application services coordinate
  exactly one command each. Any behavior spanning more than one domain module runs through
  business-process/application orchestration — never a direct write from one module into
  another module's tables.
- **Aggregate shape:** every aggregate has an immutable UUID id, tenant/company scope, an
  explicit lifecycle state, an optimistic `version` column, created/changed actor+timestamp
  metadata, an audit policy, and a document number where applicable. APIs exchange command/read
  DTOs only — never raw ORM objects.
- **Mutation contract:** every mutation carries tenant, company, actor, correlation ID, request
  ID, locale, business date, and idempotency key; runs inside exactly one Unit-of-Work
  transaction; validates scope *before* loading any protected data; and returns a
  non-disclosing error (never "record exists but you can't see it") for inaccessible records.
- **Immutability of posted data:** posted journals and inventory movements are never edited or
  deleted. Correction happens only through an authorized reversal, return, credit, adjustment,
  or reclassification that links back to the original record.
- **Tenant scope:** all business data carries `tenant_id`; relevant data additionally carries
  company/business-unit/plant/warehouse/cost-center/profit-center/project/sales-organization
  scope as applicable. Tenant scope is enforced both in the database (RLS) and in every
  application-layer query — never rely on one layer alone.
- **Time and money:** store technical timestamps in UTC plus a separate business date and the
  company/site time zone for legally significant operations. Store all money and quantity values
  as `numeric`, never `float` or `double`. Centralize rounding, UOM conversion, zero-decimal
  currency handling, tax, FX, and allocation-residual rules in one shared module per concern —
  never re-implement rounding logic inline in a feature.

## Foundation and repair

1. [ ] Verify the pre-installed environment and create an implementation baseline.
   - Run every verification command in Section 2.5 and paste the raw output into
     `docs/implementation-baseline.md` under "Environment verification." If any command fails or
     shows an unexpected version, stop per rule 8 in Section 0 and ask the user to fix that one
     item — do not attempt to install or fix it yourself, and do not proceed to the rest of this
     task until it is resolved.
   - Create the backend/frontend/docs/test directory tree, dependency manifests (`requirements.lock.txt`
     copied from the venv per Section 2.5, and `frontend/package.json`/`package-lock.json`), editor/git
     ignore files. Add no business behavior in this task — no domain models, no endpoints, no UI routes.
   - In `docs/implementation-baseline.md`, also record: the project boundaries, the planned
     migration and test file locations, and an explicit statement that the system starts with
     zero implemented modules.
   - **Definition of done:** the environment verification output is captured; backend and
     frontend dependency manifests parse with their respective package managers; the baseline
     doc contains no credentials and accurately describes a clean starting point; you can point
     to the exact commit as "task 1 complete."

2. [ ] Stabilize project tooling and directory boundaries.
   - Create/keep `backend/app/{api,application,business_processes,business_objects,domains,
     workflow,rules,authorization,validation,repositories,metadata,events,audit,search,
     localization,extensions,background,storage,cache,core}`. Create the `frontend` SvelteKit
     structure only if it does not already exist from Task 1.
   - Configure, as real runnable commands (not just descriptions): formatting, linting, type
     checking, tests, coverage, a dependency/secret scan, and reproducible local commands for
     each. Domain-layer files (`business_objects`, `domains`, `rules`) must contain zero
     `import fastapi` / `import sqlalchemy` statements.
   - **Definition of done:** one single command runs all backend static checks; one single
     command runs all frontend static checks; a dedicated forbidden-import test fails if a
     domain file imports FastAPI or SQLAlchemy, and you demonstrate it failing on a deliberately
     broken example before removing that example.

3. [ ] Secure runtime configuration without changing the required stack.
   - Define typed settings objects for connecting to the already-installed services from Section
     2.5: PostgreSQL 18 URL, Valkey URL, S3 endpoint/bucket/credentials, session/JWT signing
     keys, allowed CORS origins, upload size limits, SMTP settings, and environment mode
     (dev/staging/prod). This task only reads connection details from environment
     variables/secrets — it never installs, starts, or configures the PostgreSQL/Valkey services
     themselves; that was done once by the user per Section 2.5. Every required secret causes a
     clean startup failure (not a stack trace) when absent.
   - Remove every hardcoded credential and default secret from the codebase. Add `.env.example`
     with placeholder values only (no real-looking secrets) and confirm `.env` itself is
     git-ignored. Never log a connection string, token, or password, including in debug mode.
   - **Definition of done:** starting the app without a required secret fails with a clear
     message and non-zero exit code; starting it with injected test settings succeeds; a secret
     scan of the repository (e.g., gitleaks or equivalent) reports zero findings.

4. [ ] Repair and test the safe metadata-rule expression engine.
   - Implement a typed, side-effect-free grammar: literals, field lookup, an explicit allow-list
     of lookup functions, arithmetic, comparison, boolean logic, dates, and an explicit
     allow-list of aggregate functions. Forbid Python `eval`/`exec`, raw SQL construction,
     filesystem access, network access, unbounded recursion, and unbounded loops at the parser
     level, not just by convention.
   - Fix binary operator parsing so operators are preserved explicitly through the
     grammar/transformer (the historical bug this task repairs). Write and pass tests for at
     least: `qty * price`, `qty + 1`, a comparison expression, boolean operator precedence, null
     handling in arithmetic and comparison, divide-by-zero, an unauthorized lookup function,
     a step/complexity limit being hit, and a deliberately malicious input string.
   - **Definition of done:** all listed tests pass; every evaluation returns both a value and a
     deterministic explanation/input hash suitable for audit; no code path evaluates raw Python
     against user-supplied expression text.

5. [ ] Complete transactional Unit of Work, idempotency, and optimistic concurrency.
   - Implement exactly one SQLAlchemy transaction per command; only documented savepoints are
     allowed inside it. Implement an idempotency ledger keyed on
     `(tenant, principal, endpoint_or_command, key, canonical_request_hash)` that replays the
     original response and status code for a repeated identical request, and rejects a repeated
     key whose payload hash differs.
   - Use a `version` predicate on every editable aggregate's update; on mismatch, return HTTP
     409 with the current version and the specific fields that changed. Use a short-lived
     `SELECT ... FOR UPDATE` only for number-series allocation, stock/on-hand/reservation rows,
     or other balance-critical rows — never as a general locking strategy.
   - **Definition of done:** an integration test proves replaying an identical command produces
     exactly one business effect; a test proves a replayed key with a changed payload is
     rejected; a test with two concurrent stale edits proves exactly one succeeds and the other
     returns 409.

6. [ ] Implement database schemas, roles, RLS, and initial migration.
   - Create PostgreSQL schemas: `core`, `master_data`, `finance`, `sales`, `purchasing`,
     `inventory`, `manufacturing`, `hr`, `workflow`, `reporting`, `audit`, `integration`.
   - Create these database roles: an owner role, a non-owner `NOBYPASSRLS` runtime role used by
     the application, a migration role, a backup role, and a break-glass role. Force RLS on
     every tenant-scoped table. Set the transaction-local tenant/company context only through
     trusted middleware (never trust a client-supplied header directly as the RLS predicate
     without validating it against the authenticated session). Reset that context explicitly at
     the start of every pooled connection use and every background job.
   - Add foreign keys, tenant-scoped uniqueness constraints, indexes, and sequences (including a
     dedicated audit sequence) as part of the same migration set, not deferred to a later task.
   - **Definition of done:** the Alembic initial migration creates all schemas, tables, RLS
     policies, grants, indexes, and seed data starting from an empty database; a test proves the
     runtime role's cross-tenant read, write, join, and search attempts all fail; a test proves
     only the migration role can execute migrations.

7. [ ] Finish append-only audit and transactional outbox at database level.
   - Audit rows capture: actor, impersonator (if any), timestamp, IP, user-agent, request ID,
     correlation ID, action, object, version, reason, workflow context, and a redacted
     before/after diff. The runtime role must be structurally unable to update or delete audit
     rows (a database privilege, not an application check). Maintain a hash chain over audit
     rows and periodic immutable/versioned checkpoints in S3-compatible storage.
   - Write the outbox row in the exact same database transaction as the business mutation that
     produced it. The consumer side implements event/consumer dedupe, a lease-and-reclaim
     mechanism for stuck rows, an aggregate-level ordering key, retry with backoff, a quarantine
     state for repeatedly failing rows, an authorized manual replay path, and an observable
     failure status (queryable, not buried in logs only).
   - **Definition of done:** a test proves a rolled-back transaction produces neither a business
     event nor an outbox row; a test proves a consumer crash-and-retry may redeliver a message
     but never produces a duplicate business effect; a test proves audit tampering fails when
     attempted as the runtime role.

8. [ ] Build identity, RBAC/ABAC, field security, and SoD engine.
   - Implement login, logout, password reset, account lockout, rate limiting, and either secure
     rotated sessions or rotated JWTs, plus CSRF protection and an MFA-ready design (interfaces
     present even if only one factor is wired up first). Implement OIDC/SAML and SCIM as
     interfaces your code exposes/consumes. Implement roles and grants that combine
     object/action/field scope with organization, own-record, and workflow-state constraints.
   - Support these actions wherever applicable: create, read, update, delete, submit, approve,
     reject, release, post, cancel, reverse, close, reopen, print, export, import, administer,
     audit-view. Enforce every one of them at the API layer, the query layer, lookups,
     attachments, exports, reports, WebSocket messages, background jobs, and integration
     endpoints — not only at the API layer.
   - Add configurable Segregation-of-Duties conflict rules for at least: vendor/bank-detail
     maintenance vs. payment execution, invoice/journal preparation vs. approval/posting, HR
     data changes vs. payroll processing, compensation/bank/termination changes, and privileged
     (admin/break-glass) access.
   - **Definition of done:** tests cover deny-before-load (never load then filter), field
     masking, owner-record scope, cross-company scope denial, export denial, self-approval
     denial, SoD prevent/warn/approved-exception paths, and that a sensitive action is always
     audited even when denied.

9. [ ] Build governed metadata, lifecycle, numbering, and localization packages.
   - Define versioned metadata artifacts for: object, field, relation, form/view, action,
     policy, lifecycle, number series, workflow, rule, report, localization, search,
     audit, import/export, and extension. Every artifact has states: draft, test, approved,
     published, superseded, retired.
   - Publication of a metadata artifact must validate references, types, layouts, policies, rule
     termination (no infinite loops), and dependencies; require reviewer/SoD-compliant approval;
     invalidate the versioned Valkey cache entry only after the publishing transaction commits;
     and permanently pin the exact published version that was in effect for any already-posted
     record (a posted record must always be able to show which metadata version applied to it).
   - Implement a shared document header/line model and typed relations for source, fulfillment,
     invoice, payment, return, credit, reversal, intercompany, and attachment links. Configure
     secure document numbering scoped by company/year/series/branch, allocated at a defined
     lifecycle point, and never reused even after cancellation.
   - **Definition of done:** tests cover publication failure cases and correct post-commit cache
     invalidation; a concurrency test proves no duplicate number is ever allocated; a test proves
     a historical record renders correctly against its pinned metadata, rule, workflow, tax, FX,
     and accounting configuration even after all of those have since changed.

10. [ ] Implement common master data complete with lifecycle and import-safe APIs.
    - Implement effective-dated, approval-governed master data for: legal entity, business unit,
      department, plant, warehouse/zone/bin, party/customer/vendor/contact, item/service/
      variant, UOM/conversion, currency/rate, tax registration/jurisdiction/code, payment/
      shipping/incoterm, bank account, chart of accounts/account/dimension, fiscal calendar,
      cost/profit center, project, work center, BOM/routing, price list, and reason code.
    - Item types are: stock, non-stock, service, asset, expense, kit, and variant, each with
      sales/procurement/planning/inventory/costing/tax/quality/serial-lot/shelf-life/
      substitute/status attributes as applicable. Party records support multiple roles,
      multiple addresses, multiple contacts, tax registration, protected payment/bank data, and
      credit/compliance status.
    - **Definition of done:** every master data type has create, change, approve, inactivate,
      read, list, and import routes; field-level security is enforced on each; audit and
      effective-date history are queryable; a test proves an invalid relation (e.g., a
      non-existent account) and use of an inactive master record are both rejected.

## Finance and operational domain slices

11. [ ] Implement finance ledger configuration and journal posting first.
    - Implement legal, management, tax, and adjustment/consolidation ledgers, plus optional
      parallel accounting principles. Implement chart-of-accounts hierarchy, account types,
      groups, normal balance, control accounts, dimensions and their valid combinations, fiscal
      calendar states (open, soft-close, hard-close, reopen, adjustment period), and currencies/
      rates.
    - Implement a versioned, approved accounting-event mapping: precedence order, posting keys,
      debit/credit accounts and dimensions, rounding/balancing accounts, an explicit error
      state, and a snapshot of the mapping used at posting time. Implement journal create,
      submit, approve, post, and reverse for: general, accrual, recurring, allocation, reclass,
      intercompany, opening, closing, and correction journals.
    - **Definition of done:** tests cover balanced posting, period-state enforcement, account and
      dimension validity, control-account protection, rounding, reversal, and posting
      concurrency; a trial balance API/report and a journal drill-down API/report both exist; a
      test proves a posted journal cannot be edited or deleted.

12. [ ] Implement budget, commitment, and account-reconciliation controls.
    - Build budget records scoped by scenario, version, period, account, dimension, and
      currency, tracking original, revised, committed, actual, and available amounts. Define an
      availability formula and wire document creation/consumption/reduction/release for
      requisitions, purchase orders, invoices, and journals; enforce configurable warn/block/
      approval-required/freeze/reforecast behavior.
    - Build an account reconciliation entity carrying balance, preparer, reviewer, evidence
      attachment, exception/aging tracking, certification, escalation, and a link to the period
      close process.
    - **Definition of done:** a budget-policy test exists for each source document type listed
      above; the reconciliation workflow, its audit trail, and its report all exist; a test
      proves an unresolved reconciliation exception blocks period close.

13. [ ] Implement AP, supplier invoice matching, payments, and cash.
    - Build AP invoice types: standard, non-PO, debit memo, prepayment, and withholding-tax
      cases, plus hold, dispute, and duplicate-invoice detection. Implement two-way, three-way,
      and service matching with configurable tolerance. Post AP, tax, expense, inventory, and
      asset entries through the mapping from Task 11.
    - Build the payment lifecycle: `proposed → approved → released → transmitted →
      acknowledged → confirmed/failed/voided`, with an independent preparer, approver, and
      releaser (no single user can occupy more than one of those roles for the same payment),
      configurable signatory limits, a hash of the transmitted payment file, a remediation path
      for rejected payments, a cancellation cutoff, a supplier bank-detail-change verification
      and cooling-off period, and remittance/positive-pay export support.
    - Build bank account and statement handling: import, continuity checking, period locking,
      auto-match within tolerance, manual match with a write-off/adjustment approval step,
      preparer/reviewer certification, inter-account transfers, current cash position, and a
      cash forecast.
    - **Definition of done:** tests cover duplicate-invoice detection, matching tolerance,
      payment-process SoD, bank-detail-change control, and reconciliation; AP aging, payment
      status, cash forecast, and GL-to-bank reconciliation reports all exist.

14. [ ] Implement AR, billing, credit, revenue, collections, and tax.
    - Build invoice, credit memo, debit memo, receipt, refund, unapplied-cash, allocation,
      write-off, dunning, collections case, statement, and aging records. Enforce credit
      exposure checks at order release, shipment, and billing, each with an approved-override
      path.
    - Build shipment-based, order-based, milestone, subscription, and manual billing, plus
      contract/obligation versions, standalone-selling-price allocation, variable
      consideration, returns, contract modification, financing-component policy, contract
      asset/liability tracking, revenue recognition/reversal/catch-up, and a billing-to-revenue
      reconciliation report.
    - Implement tax precedence, place-of-supply determination, registration checks,
      product/customer tax status, and VAT/GST/sales-and-use/withholding/reverse-charge/
      exemption/recoverability handling, tax point timing, tax rounding, evidence retention,
      and a tax-return lifecycle: draft → reviewed → filed → accepted → paid → amended, with
      period locking and reconciliation.
    - **Definition of done:** tests cover AR posting, payment application, credit-hold
      enforcement, revenue rounding, and edits attempted against a locked tax return (must be
      rejected); AR aging, customer statement, deferred revenue, and tax return/reconciliation
      reports all exist.

15. [ ] Implement assets, costing, intercompany, consolidation, and close.
    - Build asset class, capitalization threshold, construction-in-progress, component
      accounting, book, depreciation method, useful life, averaging convention, acquisition,
      capitalization, transfer, impairment, revaluation, depreciation run, reversal, disposal,
      physical verification, maintenance linkage, and GL reconciliation.
    - Build inventory valuation methods: standard cost, moving average, and FIFO layers, plus
      landed cost; and cost/profit center accounting with overhead allocation, allocation
      drivers, variance analysis, and profitability reporting.
    - Build intercompany transactions: counterpart identification, due-to/due-from tracking,
      transfer pricing, paired posting, FX handling, reciprocal matching, netting, settlement,
      mismatch handling, and elimination. Build consolidation: scope, ownership percentage,
      effective dates, opening balance carryforward, currency translation, translation rates,
      translation reserve, acquisition/disposal accounting, elimination entries, and a certified
      approval workflow.
    - Implement a close checklist with task dependencies, evidence attachment, period lock, and
      a reopen-approval path.
    - **Definition of done:** tests cover asset depreciation/disposal, intercompany elimination,
      and close/reopen; asset, cost, intercompany, and consolidated financial reports all exist.

16. [ ] Implement inventory ledger, availability, reservation, and movements.
    - Every inventory movement row is append-only and carries: item/variant, company, site,
      warehouse, location, bin, status, lot/serial, ownership, quantity, base quantity, value,
      cost layer, movement type, business date, source, destination, actor, and a link to any
      reversal. On-hand quantity is always derived/reconciled from movements — never stored as
      an independently editable field. No fractional stock beyond the item's defined precision;
      no negative stock unless explicitly and auditably overridden.
    - Implement receive, issue, ship, adjust, transfer, and reserve/release/allocate operations,
      each acquiring the necessary ledger/on-hand lock, supporting both hard and soft
      reservation, supporting reversal, and respecting a per item/site negative-stock override
      control.
    - **Definition of done:** concurrent reservation and movement tests prove no oversell and no
      unapproved negative stock occurs; the movement ledger is immutable and reversible;
      valuation and on-hand reconciliation reports both exist.

17. [ ] Implement warehouse execution, traceability, counts, and planning.
    - Add zones, bins, and stock statuses (available, quality-hold, blocked, consignment,
      damaged), lot/serial tracking with expiry and shelf life, FIFO/FEFO picking logic,
      replenishment, cycle counts, physical counts with freeze and variance-approval steps, and
      consignment/owned-stock/quarantine/landed-cost context on relevant records.
    - Implement inbound reference handling, receive, put-away, bin transfer, pick wave/task
      generation, pack/container assignment, shipment/load confirmation, a barcode-scanner
      input API, label generation, and proof of delivery. The scanner is an input method only —
      every scanner-driven workflow must remain fully usable from keyboard and touch as well.
    - Implement planning parameters (safety stock, reorder point, min-max, lead time, preferred
      supplier) and time-phased demand/supply netting with a run, an exceptions view, planned
      purchase orders, planned production orders, and an audited planner firm/release override.
    - **Definition of done:** tests cover bin-level accuracy, lot genealogy, expiry enforcement,
      count variance approval, pick/ship flow, replenishment triggers, and planning exceptions;
      the relevant operational reports all exist.

18. [ ] Implement purchasing and supplier lifecycle.
    - Build supplier onboarding, qualification, risk scoring, certificate tracking with
      expiry/requalification, and a supplier scorecard; requisition, catalog request, RFQ/RFP,
      bid, award, contract, blanket order, purchase order, change order, acknowledgement,
      receipt expectation, return, and corrective action.
    - Requisition-to-order flow must resolve preferred supplier, applicable contract or catalog,
      budget availability, and required approval/SoD before becoming a PO. PO supports partial
      receipt, partial invoicing, tolerance-based matching, cancellation/close, backorder,
      drop-ship, subcontract, and accrual. Receipt capture records accepted/rejected/quarantine
      quantities, lot/serial, inspection result, bin destination, and landed-cost input.
    - **Definition of done:** tests cover supplier certificate expiry, bid award, budget
      commitment, PO approval and change, partial receipt, return, and invoice-match exceptions;
      spend, contract, cycle-time, match-rate, supplier-performance, and risk reports all exist.

19. [ ] Implement CRM, pricing, sales order, delivery, and service.
    - Build party-360 view, contact, lead, opportunity, activity, campaign, competitor,
      pipeline stage, territory, consent tracking, and duplicate-merge handling; lead conversion
      must preserve all captured data. Build quote (with revisions), contract, blanket order,
      price list, pricing rule, discount, promotion, sales order, delivery request, return
      authorization, credit/debit/invoice request, and commission handling.
    - Pricing must persist the resolved currency, UOM, customer segment, item, quantity, date,
      contract, rule priority, discount stacking, tax, and any manual override, plus the
      individual components that produced the final price. Order lifecycle:
      `draft → submitted → credit/pricing/tax validated → approved/released → allocated →
      picked/packed/shipped → billed → paid/closed`, with partial fulfillment, backorder,
      substitution, cancellation, return, and credit all supported at the appropriate states.
    - Add service item, customer asset, case, entitlement, service order, labor, parts,
      resolution, communication log, and billing-reference tracking, exposed through a
      party-restricted API suitable for a future customer portal.
    - **Definition of done:** tests cover quote-to-order conversion, pricing rule precedence,
      credit hold enforcement, allocation, partial ship/bill, return/credit, and service SLA
      tracking; pipeline, backlog, on-time-delivery, margin, return, customer, and service
      reports all exist.

20. [ ] Implement manufacturing master data, planning, execution, quality, and maintenance.
    - Build item revisions, effective and approved BOM versions with alternates/substitutes,
      routing/operations, work center/resource/capacity/tooling, production versions,
      yield/scrap parameters, quality plans, and cost rollup. A production order must preserve
      the exact BOM/routing revision it was released against, and any change to that revision
      requires an approved, impact-reviewed change record.
    - Implement the production lifecycle: `planned → firm/release → issue/reserve → operation
      start/complete → labor/machine time capture → inspection → receipt → close/cancel`,
      supporting partial completion, backflush or manual material issue, co-products/
      by-products, scrap/rework, approved material substitution, genealogy tracking,
      subcontracting, and WIP/variance/cost accounting.
    - Implement MRP driven from forecast, sales orders, safety stock, open supply, lead times,
      and BOM structure, with every firm/release override logged. Implement quality plans, lot
      sampling, characteristics, specifications, results, dispositions, nonconformance, CAPA,
      certificates, and quarantine — nonconforming material must be blocked from consumption,
      shipment, and receipt at the system level, not by convention.
    - Implement equipment/location/meter tracking, preventive maintenance plans, work orders,
      labor/material/downtime logging, inspection, failure recording, and cost posting.
    - **Definition of done:** tests cover production execution, MRP output, quality
      block/CAPA enforcement, and maintenance work orders; schedule, capacity, cost, genealogy,
      recall, quality, and maintenance reports all exist.

21. [ ] Implement HR, time, expense, payroll interface, performance, and learning.
    - Build effective-dated Person, Worker, Employment, Assignment, Position, Job, Grade,
      Organization, Reporting-relationship, and Compensation records. Support contingent
      workers, legal employer assignment, concurrent and primary assignments, matrix/acting
      manager relationships, probation/suspension/rehire/transfer/secondment, FTE/vacancy/
      incumbent/budget tracking, and an audit trail for reorganizations.
    - Build candidate, application, interview, offer, pre-employment check, right-to-work
      verification, offer accept/reject, retention, and worker-conversion records; onboarding
      and offboarding including reason, date, final pay, asset return, access revocation,
      benefits, leave payout, legal document handling, exit interview, and rehire eligibility.
      Build leave accrual, proration, eligibility, carry-forward, expiry, negative-balance
      handling, cashout, partial-day leave, work schedules, holiday calendars, blackout periods,
      protected/medical leave, and retroactive adjustment; and time entry covering shift, clock
      in/out, breaks, overtime, time zone, rounding rules, missing-punch handling, delegated
      entry, period lock, retroactive correction, and payable-data export.
    - Build expense policy, receipt capture, mileage, per-diem, cost allocation, currency
      conversion, approval routing, AP reimbursement integration, and a corporate-card adapter
      interface. Build payroll provider mapping, mapping versioning, full and delta export
      modes, effective-dated export, pay calendar, input cutoff, input lock, retroactive/
      off-cycle/final-pay handling, import of results, reconciliation, rejection handling, GL
      journal creation, and access control; never claim built-in statutory payroll calculation
      that this system does not actually perform.
    - Build goals, performance review, rating, feedback, calibration, development planning, with
      draft/public visibility security and retention rules; and a learning catalog with
      assignment, mandatory-course tracking, certification, and LMS import. Implement OIDC/SAML/
      SCIM-driven joiner/mover/leaver provisioning: start/end-date provisioning, termination
      account disable, mover-triggered access review, a failure queue for provisioning errors,
      orphan-account reconciliation, break-glass access, and periodic recertification.
    - **Definition of done:** tests cover protected-field and manager-scope denial,
      effective-dated lifecycle transitions, leave/time/expense processing, payroll export/
      retroactive/reconciliation flows, a simulated joiner-mover-leaver failure, review/
      calibration, learning-certification expiry, and privacy-driven data retention; the
      relevant HR reports all exist.

## API, integration, UI, themes, and operations

22. [ ] Build FastAPI application wiring and core HTTP behavior.
    - Add a dependency container, settings loading, the database Unit-of-Work, identity/tenant
      context resolution, idempotency handling, authorization enforcement, error mapping,
      correlation-ID logging, OpenAPI tags, and health/readiness endpoints. Use `/api/v1` as the
      base path; JSON bodies; ISO 8601 dates/times; decimal values encoded as strings (never as
      floating point); typed request/response DTOs; and explicit command endpoints such as
      `POST /sales-orders/{id}:submit` rather than overloading generic PUT/PATCH for state
      transitions.
    - Implement pagination, filter and sort allow-lists (deny anything not explicitly
      allow-listed), field selection, locale handling, and RFC 9457 (`application/problem+json`)
      error bodies carrying a stable error code, field-level errors, the correlation ID,
      retryability, and — for 409s — the current version. The WebSocket channel only emits
      authenticated, scoped notification/job/assignment/change *signals*; the frontend always
      refetches actual state over REST rather than trusting WebSocket payloads as the source of
      truth.
    - **Definition of done:** integration tests cover login/context establishment, 401/403/404
      non-disclosure (a denied record looks identical to a nonexistent one), 409 conflict, 422
      validation, idempotency replay, and pagination; the OpenAPI schema validates.

23. [ ] Add domain routers in safe vertical order.
    - Add and test routers in exactly this sequence, one router group fully tested before the
      next begins: master data; workflow/metadata; finance (journals, budgets, AP, AR, cash,
      assets, close); inventory/warehouse/planning; purchasing; sales/CRM/service;
      manufacturing/quality/maintenance; HR; reports/search/admin.
    - Every router calls into DTO mapping and exactly one application command — never exposes an
      ORM object and never performs business calculation itself. Add, where applicable to the
      domain: an object-action route, a read/list route, an audit/history route, and an
      attachment route.
    - **Definition of done:** route contract tests exist for a router group before the next
      group's implementation begins; every protected route is proven to enforce
      permission/scope and correct idempotency behavior.

24. [ ] Implement ARQ worker, durable job behavior, and notification delivery.
    - The ARQ worker process handles: outbox dispatch, email/in-app notification delivery,
      metadata cache invalidation, search index maintenance, PDF generation, import/export
      processing, integration adapter calls, scheduled work, and backup verification. Job state,
      progress, error detail, retry count, correlation ID, and tenant scope are all persisted,
      not held only in worker memory.
    - Implement scheduling with correct business-day and time-zone behavior and a single-run
      lease so a scheduled job never runs twice concurrently. Finance, inventory, pricing, tax,
      and workflow *decisions* are never made asynchronously — the worker only executes and
      records the outcome of a decision already made synchronously in a command.
    - Respect the ARQ worker's 300 MB ceiling and `max_jobs` setting from Section 2.
    - **Definition of done:** tests cover job retry, lease reclaim after a crash, cancellation,
      progress reporting, permission checks on job-triggering endpoints, and that the scheduler
      never double-runs a job; a test proves slow background work never blocks an API request.

25. [ ] Implement attachments, imports/exports, search, and integration adapters.
    - The S3-compatible storage adapter stores only the object key in the database (never a
      full URL with embedded credentials); every object path is tenant-scoped; uploads are
      validated by type and size; an async malware scan quarantines suspicious files before
      they are servable; downloads use a short-lived authorized URL and are audited. A local
      development storage adapter may exist, but only behind the exact same interface.
    - Import uses a staged file, a mapping version, a preview step, per-row error reporting, a
      duplicate policy, an approval step where required, resumability, reconciliation against
      source, and retention of the original file. Export/report generation enforces role-based
      access, field-level masking, a row limit, an optional watermark, an audit entry, an
      optional approval step, and an expiring download link.
    - PostgreSQL FTS search must be permission-aware: it searches only authorized numbers,
      codes, names, references, and text; supports relevance ranking, exact match, prefix
      match, fuzzy match, and type/date/company filtering; highlights matches; never leaks a
      result count or suggestion for records the searching user cannot see; and is kept current
      through committed-event-driven indexing plus a rebuild/reconcile job.
    - Provide adapter contracts (interfaces plus at least one working implementation each) for:
      email, bank statement import, payment export, tax/e-invoice submission, identity provider
      (IdP), payroll export, shipping, and barcode input. Inbound webhook events use a signed
      envelope with schema, version, idempotency key, ordering, retry, replay, and audit.
    - **Definition of done:** tests cover storage tenant-scope enforcement, an import row error,
      an export permission denial, a search-leak attempt, and webhook replay/retry handling.

26. [ ] Scaffold the SvelteKit application shell and typed API client.
    - Create a typed API client usable from both server-side (`load` functions) and client-side
      code, session/auth handling, a root tenant/company context, an error boundary, WebSocket
      reconnect-and-refetch for notifications, route guards, dirty-form navigation protection,
      request cancellation/retry, and zero business decisions made in frontend code (all
      decisions come from the backend; the frontend only renders and calls commands).
    - Build the application shell: a 52px top bar; a sidebar that is 248px expanded and 64px
      collapsed; a global search/command palette; a context switcher (tenant/company); a
      breadcrumb trail; favorites/recent items; a task/approval inbox; a profile menu; and a
      contextual action bar. Persist each user's allowed grid/filter/context preferences on the
      server, keyed per user.
    - **Definition of done:** tests cover an authenticated route, a denied route, a context
      switch, an expired session, a deep-linked route, and an unsaved-change navigation prompt;
      the shell is fully keyboard-navigable.

27. [ ] Build metadata-driven forms and record workspaces.
    - Implement a field registry supporting: text, number/currency, date/time, select, an async
      secure lookup field, multi-select, checkbox/switch, rich text, attachment, status, and a
      calculated/read-only field type. Render fields according to metadata-defined section,
      tab, responsive breakpoint, label, help text, default value, required/read-only/hidden
      state, numeric precision, and field-level security rules — the same metadata that drives
      backend field security (Task 8) drives the frontend rendering.
    - Implement inline field-level and form-level summary error display, focus movement when a
      dynamic field's visibility or requiredness changes, server-side validation display,
      visual indication of a calculated field's data source/provenance, and
      attachments/comments/audit/document-flow/revision-compare/related-records panels plus an
      action toolbar on record workspaces.
    - Use these spacing tokens exactly: 4px for control padding, 6px for dialog/menu padding,
      and a maximum of 8px padding on item cards.
    - **Definition of done:** tests cover form rendering from metadata, field-level security
      enforcement, invalid input, conditional field visibility, focus movement, and a 409
      conflict's compare-refresh-reapply flow; a dirty-draft (unsaved changes) test passes.

28. [ ] Build grids, mobile operations, and accessibility behavior.
    - Grids use server-side paging and virtualization, a stable row ID across refetches,
      cancellable in-flight fetches, a column chooser, column resize/reorder/pin, per-column
      filter, grouping, aggregation, saved views, and a confirmation step before any bulk
      action. Define ARIA roles/attributes for row, column, selection, and filter state, and
      manage focus retention across data refresh. Never download an unbounded dataset to the
      browser — always page or export asynchronously (Task 25) instead.
    - Provide a narrow-screen record-card alternative to the grid for critical workflows, with a
      minimum 44px touch target. On mobile, a user must be able to complete: approval actions,
      lookups, barcode scan/capture, receive/pick/count operations, and quick status changes.
      At a 320 CSS-pixel viewport and 400% browser zoom, there is no page-level horizontal
      scroll except inside grids that are explicitly labelled as scrollable.
    - **Definition of done:** tests cover keyboard and screen-reader interaction with grid,
      filter, and selection state after a refresh; tablet and mobile critical workflows;
      right-to-left layout; reduced-motion preference; forced-colors mode; and offline/retry/
      partial-failure handling.

29. [ ] Build reporting, dashboards, print, and documents.
    - Implement report definitions with versioning, parameters, saved views, scheduling,
      sharing, ownership, and authorization checked at definition time, run time, delivery time,
      and download time against the *current* recipient's access (not the access they had when
      the report was scheduled). Support masking and revocation, a documented
      source-as-of/freshness statement, reconciliation to source, defined grain, defined
      rounding, retention policy, and a drill-down path from summary to detail. Output CSV,
      XLSX, and PDF asynchronously through the ARQ worker (Task 24).
    - Use Apache ECharts for trend, comparison, distribution, and composition visualizations,
      each keyboard-accessible, each with a textual insight summary, each backed by an
      accessible source data table with a download option, and each using direct data labels,
      markers, and line patterns in addition to color (never color alone) with drill-through
      support.
    - Create dashboards for: executive, finance (close/cash/working capital), sales, purchasing,
      inventory/warehouse, production/quality, HR, and task workload. Every KPI shown on any
      dashboard must reconcile to its source data — no dashboard-only derived numbers that
      cannot be traced back. Create versioned, HTML-first document templates with preview,
      render, print, download, email, and an issued-document state that is immutable once
      issued.
    - **Definition of done:** tests cover report-level RLS/masking/revocation, freshness display
      and error handling, chart-to-table and drill-through behavior, PDF output that is tagged,
      text-selectable, language-correct, in correct reading order, with proper table headers,
      and print-correct; a report-download audit test passes.

30. [ ] Implement the governed enterprise theme system.
    - Use semantic CSS variables exclusively — never a raw hex value in a component, in
      metadata, in AG Grid or ECharts options, in PDF templates, or in email templates. Use
      Tailwind v4's `@theme inline`, a root `data-theme` attribute, and a matching CSS
      `color-scheme`. Preference resolution order: explicit user setting → company default →
      tenant default → OS preference → Cloud Light fallback. Resolve the preference server-side
      and render it before hydration so there is no flash of the wrong theme.
    - Required semantic tokens: canvas, surfaces, borders, a text-hierarchy scale,
      primary/secondary, focus ring, success/warning/danger/info, disabled state, row states
      (selected/hover/striped), overlay, and shadow. Status, selection, required-field,
      overdue, and chart-series meaning must always be reinforced with text, an icon, a line
      style, or a marker — never color alone.
    - Implement these approved palettes exactly as specified (do not adjust any hex value):
      - Cloud Light: canvas `#F6F8FB`, surface `#FFFFFF`, text `#172033`, primary `#155EEF`,
        secondary `#7655D6`; semantic — success `#087443`, warning `#A15C00`, danger `#B42318`,
        info `#0E7490`.
      - Cloud Dark: canvas `#111827`, surface `#1F2937`, text `#F3F6FB`, primary `#78A6FF`.
      - Solar Yellow: canvas `#FFFDF6`, text `#2A2412`, primary `#8A5A00`, selected `#FFE9A3`.
      - Forest Ledger: canvas `#F6F8F5`, text `#173021`, primary `#167044`.
      - Graphite Night: canvas `#16181D`, surface `#20232A`, text `#F5F7FA`, primary `#7AA2FF`.
      - Chart series order (all themes): `#155EEF`, `#7655D6`, `#0E7490`, `#087443`, `#A15C00`,
        `#B42318`, `#526174`, `#E06C3A`.
    - A theme package is approved/versioned metadata carrying token values, a recorded contrast
      check, chart-series mapping, PDF template mapping, author, reviewer, effective date, and
      rollback evidence; issued legal documents pin the exact template and theme version used.
      No gradients, no decorative blobs, no large saturated color surfaces, and no
      red-green-only meaning anywhere.
    - **Definition of done:** WCAG 2.2 AA is met — 4.5:1 contrast for normal text, 3:1 for
      non-text/focus indicators; visual regression tests cover contrast, RTL layout, 320px
      viewport, 400% zoom, native dark mode, forced-colors mode, and print; any regression in
      these blocks theme publication.

31. [ ] Implement cache, performance, logging, metrics, and operational controls.
    - Cache only versioned, tenant/company-scoped metadata, lookups, permissions, navigation,
      localization strings, and dashboard fragments in Valkey, within the 256 MB ceiling from
      Section 2, invalidating strictly after the writing transaction commits. Never cache an
      unscoped or sensitive result set.
    - Enforce these performance targets: a simple read at P95 ≤ 500 ms; a command at P95 ≤ 1.5 s
      excluding asynchronous work; the first page of any list ≤ 1 s; a usable metadata-driven
      page ≤ 2 s; visible job progress updates within ≤ 5 s. Use indexes, `EXPLAIN` review,
      keyset pagination, and bounded exports to hit these targets; prohibit N+1 query patterns,
      unbounded `SELECT`s, large aggregations computed client-side, and any external HTTP call
      made from inside a database transaction.
    - Add redacted structured logs carrying correlation ID, actor, tenant, company, route or
      job/event name, duration, and status. Expose metrics and health/readiness for the API,
      database, outbox, cache, storage, authentication, security events, background jobs, and
      backups. Add alerting and dashboards for these. Implement a maintenance/read-only mode and
      a feature/configuration release-control mechanism (staged rollout, not a code branch).
    - **Definition of done:** query-plan and load tests exist for allocation, number-sequence
      allocation, posting, approval, import, and search; log redaction and health/metrics
      endpoints are tested; a documented measurement proves no single feature exceeds the 4 GB
      design budget in Section 2 under realistic load.

32. [ ] Implement backup, recovery, system service deployment, and runbooks without Docker.
    - Configure encrypted PostgreSQL backups with point-in-time recovery, off-host WAL
      archiving, versioned object storage for backup artifacts, daily backup retention of 35
      days and monthly retention of 13 months, plus legal-hold/retention support. Prove in
      testing: PostgreSQL RPO ≤ 15 minutes, attachment RPO ≤ 24 hours (tighter if versioning
      allows), and full restore RTO ≤ 8 hours. Explicitly document that there is no automatic
      failover for this single-VM topology.
    - Create non-Docker system service definitions (e.g., systemd units) for the reverse
      proxy/TLS termination, the FastAPI/Uvicorn process group, and the ARQ worker, each running
      as a least-privilege service user, with environment-file permissions locked down, graceful
      shutdown handling, and the resource/connection-pool/concurrency limits from Section 2
      encoded directly in the unit files (`MemoryMax=`, worker counts, pool sizes) — not left as
      documentation only.
    - Write runbooks for: job/outbox/integration failure, security incident response, data
      correction, period close, restore, capacity planning, deploy/rollback, and key rotation.
    - **Definition of done:** an automated restore rehearsal succeeds and is recorded, and a
      manual service start/stop/rollback drill is performed and its evidence recorded.

33. [ ] Complete the verification matrix and release gate.
    - Unit test invariants, calculations, rules, accounting logic, and authorization decisions.
      Integration test migrations, RLS, Unit-of-Work behavior, outbox delivery, background jobs,
      storage, and integration connectors. Contract test the API and webhook payloads. Run
      browser end-to-end tests for: quote-to-cash, procure-to-pay, receipt/transfer/count,
      production/quality, period close, HR lifecycle including leave/time/expense/payroll,
      a simulated joiner-mover-leaver failure, a sensitive-data access denial, calibration,
      learning-certification expiry, privacy-driven retention, and workflow escalation.
    - Test: concurrency and idempotency together, duplicate-event handling producing no
      duplicate business effect, reversal correctness, rounding/FX/tax edge cases, period-close
      enforcement, accessibility, performance against Section 2's budget, restore, security
      abuse cases, RLS leak attempts, attachments, reports, themes, RTL layout, and PDF output.
    - Produce these documents: architecture overview, module/process guides, data dictionary,
      metadata guide, OpenAPI/integration guide, roles/SoD matrix, configuration workbook,
      posting matrix, report/localization catalog, operations/deploy/runbooks (from Task 32),
      migration/upgrade guide, test/traceability matrix, known limitations, and admin/user
      help.
    - Create a traceability record for every task and feature in this file, recording: entity,
      command/API, route, permission, workflow/rule, accounting/inventory event, audit entry,
      report, test, and documentation reference.
    - **Final acceptance (the whole system, not one task):** a correctly scoped user can create,
      approve, process, reverse, find, report on, and audit every document type in this file
      without any direct database intervention; every recorded effect reconciles to its source,
      subledger, general ledger, and report; a posted record can always be reconstructed against
      its historical configuration; retrying any command never produces a duplicate business
      effect; and an unauthorized user can neither retrieve nor infer any restricted data
      through any surface — API, export, report, or search.
