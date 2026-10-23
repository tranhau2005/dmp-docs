<!--
Provenance: copied from AGENTS.md in the former monorepo
tranhau2005/DriverManagementPlatform, branch ao/drivermanagementplatform-10/docs-specs
(commit 11603c2). Sections 0, 1 and 3 have since been updated for the multi-repo layout;
all other sections are unchanged.
-->

> **Note:** This guide originated as `AGENTS.md` in the DriverManagementPlatform monorepo. It is now the
> **authoritative agent guide for the multi-repo layout** and lives in this repository (dmp-docs).
> Section numbers are kept stable so existing references such as `docs/agent-guide.md §7` still resolve.
> Where the text says "this AGENTS.md" or "this file", read it as this guide. Sections 4-10 still use some
> monorepo paths (`services/api`, `apps/web`, root `docker-compose.yml`); translate them with the mapping table in Section 3.

---

# AGENTS.md — DriverManagementPlatform

> **Audience:** autonomous coding agents (Codex, Claude, Gemini, Kiro, etc.).
> **Purpose:** everything you need to vibe code this project without blocking on questions.
> Read this file top to bottom before writing code. When a rule here conflicts with your defaults, this file wins.

---

## 0. TL\;DR for agents

- This is a **driver management platform**: accounts, drivers, vehicles, customers, bookings, leave/approval, timekeeping, and payroll.
- Stack: **FastAPI + PostgreSQL** backend (modular monolith), **React** web, **React Native** mobile. **Multi-repo**: independently versioned `dmp-*` repositories checked out side by side (Section 3).
- The schema in `dmp-docs/docs/schema.dbml` / Appendix A is the **authoritative data model**. Do not rename tables, columns, roles, or statuses.
- Build the **MVP slice in Section 7 first**. It is fully specified - you can build it end to end without asking anyone.
- Anything marked **[ASSUMPTION]** is a reasonable default chosen so you can proceed. It is safe to build on, but leave a `# ASSUMPTION:` code comment so a human can confirm later.
- Anything marked **[NEEDS-OWNER]** is a business rule that is *not* decided. Do **not** invent it. Stub it, raise `NotImplementedError`, and add a `# NEEDS-OWNER:` comment. See Section 9.

---

## 1. Current repository state

The platform is split into independently versioned repositories (layout and path mapping in Section 3). As of now:

- **dmp-api** has the **M1 scaffold** (Section 7): `pyproject.toml` + `uv.lock` (Python 3.12), FastAPI app factory `app/main.py` with `GET /health` -> `{"status":"ok"}`, `app/core/` (`config.py` reading `DATABASE_URL`, async SQLAlchemy session in `db.py`, typed domain errors + a single exception handler in `errors.py`), async Alembic setup (`alembic.ini`, `alembic/env.py`, empty `alembic/versions/`), and `tests/test_main.py`. Domain folders under `app/domains/` are empty placeholders. **No models or migrations yet** - M2 onward is not built.
- **dmp-infra** has `docker-compose.yml` with a `db` service (PostgreSQL 16, database `dmp`, bound to `127.0.0.1:5432`). `docker/`, `databases/`, `kafka/`, `monitoring/`, `scripts/` are empty placeholders.
- **dmp-docs** (this repository) holds this guide, `docs/schema.dbml`, `docs/architecture/`, `kiro-system-architecture-handoff.md` (reference, not instructions), the Kiro workspace `.kiro/` (`specs/`, `steering/`), and placeholder `data/` and `packages/contracts/` folders.
- **dmp-web**, **dmp-mobile**, **dmp-agent-orchestrator**, **dmp-event-consumer**, **dmp-data-processing** are scaffolds without application code. Treat empty directories as **intent**, not implemented code.
- **DriverManagementPlatform** is now the hub repository: a README that links the repositories and documents the side-by-side checkout. It contains no application code.

Build on the existing dmp-api scaffold; do not re-scaffold it. Continue with M2 in Section 7.

> WARNING: Do not confuse this with `dmp-docs/.kiro/specs/driver-management-dashboard/`. That spec describes a *different, simpler* system (roles `Viewer`/`Manager`, a trimmed model). It is **not authoritative** for this platform. Use only this guide and Appendix A.

---

## 2. Tech stack (pinned)

| Layer | Choice | Version target | Notes |
|---|---|---|---|
| Backend language | Python | 3.12 | |
| Backend framework | FastAPI | latest 0.11x | async routers |
| ASGI server | Uvicorn | latest | |
| ORM | SQLAlchemy | 2.x (typed) | models never returned directly as API responses |
| Migrations | Alembic | latest | every schema change = one migration |
| Validation/schemas | Pydantic | v2 | request/response models separate from ORM |
| Database | PostgreSQL | 16 | source of truth = Appendix A DBML |
| Package manager (py) | uv | latest | `uv` for installs/locking |
| Web | React + TypeScript + Vite | React 18 | |
| Mobile | React Native + TypeScript | latest stable | **[ASSUMPTION]** Expo-managed |
| Backend tests | pytest + pytest-asyncio | latest | |
| Lint/format (py) | ruff | latest | lint + format |
| Lint/format (ts) | eslint + prettier | latest | |
| Container | Docker + docker-compose | latest | dev Postgres via compose |

**[ASSUMPTION]** Kafka, event-consumer, data-processing, and agent-orchestrator services are **out of MVP scope**. Build the synchronous API path first. Do not add Kafka until a [NEEDS-OWNER] async use case is confirmed.

---

## 3. Repository layout (multi-repo)

Clone every repository **side by side** in one parent directory; cross-repository relative paths depend on it:

```
<parent>/
  DriverManagementPlatform/     # hub: README linking all repositories; no application code
  dmp-api/                      # FastAPI backend (modular monolith)
    pyproject.toml  uv.lock  alembic.ini  .env.example
    alembic/versions/
    app/
      main.py                   # FastAPI app factory
      core/                     # config, db session, errors (security later)
      domains/<domain>/         # see per-domain convention below
    tests/                      # pytest testpaths
  dmp-web/                      # React + Vite + TS (admin web)
    packages/ui/                # shared UI components
    packages/api-client/        # typed client generated from the dmp-api OpenAPI
  dmp-mobile/                   # React Native + TS   [later]
  dmp-agent-orchestrator/       # out of MVP (D-09)
  dmp-event-consumer/           # out of MVP (D-04)
  dmp-data-processing/          # out of MVP
  dmp-infra/
    docker-compose.yml          # dev postgres (service `db`)
    docker/ databases/ kafka/ monitoring/ scripts/
  dmp-docs/
    docs/schema.dbml            # copy of Appendix A - keep in sync
    docs/agent-guide.md         # this guide
    docs/architecture/ docs/decisions/ docs/api/ docs/data/
    .kiro/specs/  .kiro/steering/
    data/  packages/contracts/
    kiro-system-architecture-handoff.md
```

Cross-repository wiring:

- dmp-api starts its dev database with dmp-infra's Compose file: `docker compose -f ../dmp-infra/docker-compose.yml up -d db` from the dmp-api root (or `docker compose up -d db` from the dmp-infra root).
- When dmp-infra gains an API build context, it targets `../dmp-api`.
- Specs live only in `dmp-docs/.kiro/specs`; shared API/event contracts live in `dmp-docs/packages/contracts`. Coordinate contract changes with every affected repository.

### Monorepo path -> repository mapping

| Former monorepo path | Multi-repo location |
|---|---|
| `AGENTS.md` | `dmp-docs/docs/agent-guide.md` (each repository also has its own short `AGENTS.md`/`CLAUDE.md`) |
| `README.md` (root) | `DriverManagementPlatform/README.md` (hub); each repository has its own `README.md` |
| `docker-compose.yml` (root) | `dmp-infra/docker-compose.yml` |
| `infra/*`, scripts | `dmp-infra/` |
| `docs/` (incl. `docs/schema.dbml`) | `dmp-docs/docs/` |
| `data/`, `packages/contracts` | `dmp-docs/data/`, `dmp-docs/packages/contracts/` |
| `.kiro/` (specs, steering) | `dmp-docs/.kiro/` |
| `kiro-system-architecture-handoff.md` | `dmp-docs/kiro-system-architecture-handoff.md` |
| `services/api/` (`pyproject.toml`, `alembic.ini`, `alembic/versions/`, `app/...`) | `dmp-api/` repository root (same relative paths) |
| `services/api/app/tests/` | `dmp-api/tests/` |
| `apps/web/` | `dmp-web/` repository root |
| `packages/ui`, `packages/api-client` | `dmp-web/packages/ui`, `dmp-web/packages/api-client` |
| `apps/mobile/` | `dmp-mobile/` |
| `services/agent-orchestrator/` | `dmp-agent-orchestrator/` |
| `services/event-consumer/` | `dmp-event-consumer/` |
| `services/data-processing/` | `dmp-data-processing/` |

Commands in Section 4 that say "run from services/api" run from the `dmp-api` root; "run from apps/web" runs from the
`dmp-web` root; `docker compose up -d db` runs from the `dmp-infra` root (or with `-f ../dmp-infra/docker-compose.yml` from `dmp-api`).

### dmp-api domains

Target domains under `dmp-api/app/domains/`: `accounts/ areas/ drivers/ licenses/ vehicles/ customers/ bookings/ leave/ scheduling/ payroll/`.

The current M1 scaffold diverges: it has empty placeholder domains `auth/ drivers/ vehicles/ assignments/`, each split into
`models/ repositories/ routers/ schemas/ services/` sub-packages, plus empty `app/database/`, `app/dependencies/`,
`app/infrastructure/`, `app/routers/`. Tests currently live in the root `tests/` folder (pytest `testpaths`). When implementing,
follow the convention below; `assignments` (the `driver_vehicles` relationship) belongs with `vehicles`.

**Per-domain module convention (strict).** Each `domains/<name>/` folder contains:

- `router.py` - FastAPI routes; validate input, call service, map errors to HTTP.
- `service.py` - business rules; the only place rules live.
- `repository.py` - DB access; returns ORM/domain objects, no HTTP concerns.
- `models.py` - SQLAlchemy ORM models.
- `schemas.py` - Pydantic request/response models (NOT ORM).
- `tests/` - unit + integration tests for the domain.

Rules: routers never touch the DB directly; services never build HTTP responses; **ORM models are never serialized to clients** - always map to a Pydantic schema.

---

## 4. Setup and commands (make these work, then keep them working)

Agents must ensure these commands succeed. If a command does not exist yet, create the config that makes it work. Start the DB, apply migrations, then run tests. Do not run the backend dev server or any watch mode as a blocking step in automation; a human runs those manually.

```bash
# dev database
docker compose up -d db                 # Postgres 16 on localhost:5432

# backend (run from services/api)
uv sync                                  # install deps
uv run alembic upgrade head              # apply migrations
uv run pytest                            # tests must pass before a task is done
uv run ruff check . && uv run ruff format .
# backend dev server (run manually in a terminal, not in automation)

# web (run from apps/web)
npm install
npm run build
npm run lint
# web dev server (run manually in a terminal)
```

**[ASSUMPTION]** Local dev DB URL: `postgresql+psycopg://postgres:postgres@localhost:5432/dmp`. Read it from the `DATABASE_URL` env var; never hardcode secrets.

---

## 5. Backend conventions

- **Layering:** router -> service -> repository. One direction only.
- **Schemas vs models:** every endpoint takes a Pydantic request model and returns a Pydantic response model. Never return a SQLAlchemy object.
- **Errors:** services raise typed domain exceptions (`NotFound`, `Conflict`, `ValidationError`, `Forbidden`); a single FastAPI exception handler maps them to HTTP (404/409/422/403). Do not scatter `HTTPException` through services.
- **Money:** all amounts are `numeric(14,0)` -> integer minor units. **[ASSUMPTION]** currency is VND with 0 decimal places. Mark with `# ASSUMPTION:` and confirm.
- **Time:** store `timestamptz` in UTC. `month` columns are **always the first day of the month** - enforce with a CHECK constraint in the migration.
- **Auth:** **[ASSUMPTION]** JWT bearer tokens; `password_hash` via argon2 or bcrypt. Never store raw passwords. Never log tokens or PII.
- **PII:** `drivers` holds DOB, phone, address, citizen_id, emergency_contact. Restrict read access by role, never log these fields, and add an audit entry on read/change of sensitive fields. **[NEEDS-OWNER]** exact masking/retention rules.
- **Attribution:** `changed_by`, `decided_by`, `actor_id`, `created_by` always reference the acting `accounts.id`. Populate them on every write that has such a column.

### Data-integrity rules to encode in migrations (not just the app)

These close real gaps in the raw DBML. Add them as Alembic migrations with CHECK/constraint SQL:

1. `accounts`: CHECK that `driver_id` and `customer_id` are not both set, and that the set link matches `role` (DRIVER -> driver_id, CUSTOMER -> customer_id, ADMIN -> neither). **[ASSUMPTION]**
2. `employment_history` and `driver_vehicles`: prevent **overlapping validity intervals** per driver. Use a PostgreSQL `EXCLUDE` constraint (btree_gist) or enforce in the service inside a serializable transaction. **[ASSUMPTION]** overlaps are not allowed.
3. Enum/CHECK constraints for every status/role/type column, using the allowed values in the DBML notes (account role/status, employment status, booking status, leave type/status, leave action, vehicle type, adjustment type).
4. `*.month` columns: CHECK `date_trunc('month', month) = month`.
5. FK delete behavior: **[ASSUMPTION]** use `ON DELETE RESTRICT` everywhere, plus soft-archive via `archived_at` on `drivers`/`customers`. Do not hard-delete PII. **[NEEDS-OWNER]** confirm.

---

## 6. Frontend conventions

- React + TS + Vite; function components + hooks.
- Typed API client aligned to the backend OpenAPI; no `any` on API boundaries.
- Keep business rules on the server. The web app renders state and surfaces server errors; it does not re-implement payroll/booking logic.
- Role-aware UI: hide/disable actions the current role cannot perform, but **never** rely on the client for authorization - the API enforces it.

---

## 7. MVP slice - BUILD THIS FIRST (fully specified, no owner input needed)

Deliver these in order. Each is independently shippable and testable.

**M1. Project scaffold + health check**
- Create the layout in Section 3, `docker-compose.yml` with Postgres 16, a FastAPI app factory, and `GET /health` returning `{"status":"ok"}`.
- Acceptance: the DB starts, the API serves `GET /health` as 200, and `uv run pytest` runs (at least one trivial test).

**M2. Database schema + migrations**
- Translate **all** of Appendix A into SQLAlchemy models + an initial Alembic migration, including the integrity rules in Section 5.
- Acceptance: `uv run alembic upgrade head` creates every table and constraint; a round-trip test inserts and reads one `account` and one `driver`.

**M3. Accounts + auth**
- `POST /auth/login` (username + password -> JWT), `GET /auth/me`, password hashing, role on the token.
- CRUD for `accounts` restricted to `ADMIN`. Enforce the account/role/link CHECK.
- Acceptance: login succeeds/fails correctly; a non-ADMIN token gets 403 on account management; passwords are hashed and never returned.

**M4. Drivers + licenses (core domain)**
- CRUD `drivers`; list with pagination + filter by `area_id` and status; manage `driver_licenses` (unique `license_number`).
- Record an `employment_history` row on status change with `changed_by` + reason; reject overlapping intervals.
- Acceptance: create/read/update a driver; add a license; a status change writes history; overlapping history is rejected; PII fields are not logged.

**M5. Vehicles + assignment**
- CRUD `vehicles`; assign a driver to a vehicle over a validity interval via `driver_vehicles`; reject overlapping active assignments.
- Acceptance: assigning, then re-assigning closes the prior interval; an overlap is rejected.

> Stop after M5 and ask for review. M6+ (bookings lifecycle, leave approval, timekeeping aggregation, payroll calculation, reports/PDF) depend on **[NEEDS-OWNER]** business rules - see Section 9. Build CRUD/storage for those tables, but do not invent their lifecycle or formulas.

---

## 8. Testing and done-definition

A task is **done** only when:

- `uv run pytest` passes (backend) and `npm run build` + `npm run lint` pass (web).
- New endpoints have at least one happy-path test, one auth/permission test, and one validation/error test.
- Any constraint added in a migration has a test proving it rejects bad data.
- No PII or secrets appear in logs or test fixtures (use fake data).
- Every `[ASSUMPTION]` you relied on has a `# ASSUMPTION:` comment at the point of use.

---

## 9. Business rules you must NOT invent ([NEEDS-OWNER])

Stub these; raise `NotImplementedError`; add a `# NEEDS-OWNER:` comment. Do not guess.

- **Booking lifecycle:** assignment, state transitions (PENDING -> ASSIGNED -> IN_PROGRESS -> COMPLETED/CANCELLED), fare calculation, currency, cancellation fees, concurrency.
- **Leave/approval:** who may approve which request type, date limits, and the exact effect of approval on `employment_history` and schedules.
- **Timekeeping:** who or what creates `activity_sessions` (app? booking? device?), overnight/overlap/correction rules, and how sessions aggregate against `work_schedule.planned_minutes`.
- **Payroll formula:** how `net_amount` is derived from base/trip/bonus/allowance/deduction; the source of `completed_bookings`; rounding; recalculation; period locking; who may finalize; PDF template.
- **Reports:** exact metrics, filters, permissions, and templates for the three named reports.
- **Permission matrix:** anything beyond "ADMIN manages, DRIVER/CUSTOMER see their own" is unconfirmed.
- **Multi-tenancy:** the ERD has **no** organization/tenant table. Treat the system as **single-organization**. Do not add tenancy.

If you need one of these to proceed, implement storage/CRUD only, leave the rule as a stub, and surface it in your summary.

---

## 10. Open decisions (carry forward)

| ID | Decision | Default taken here | Status |
|---|---|---|---|
| D-01 | MVP scope | Section 7 (M1-M5) | [ASSUMPTION] |
| D-02 | Multi-tenant? | Single-org, no tenant table | [ASSUMPTION] |
| D-03 | DB ops / backup | dev compose only | [NEEDS-OWNER] prod |
| D-04 | Kafka in MVP? | No | [ASSUMPTION] |
| D-05 | Booking/leave/timekeeping/assignment lifecycle | storage only, rules stubbed | [NEEDS-OWNER] |
| D-06 | Identity / authz model | JWT + role claim | [ASSUMPTION] |
| D-07 | Cloud/region/topology | undecided | [NEEDS-OWNER] |
| D-08 | Report definitions | undecided | [NEEDS-OWNER] |
| D-09 | AI/agent capabilities | none in MVP | [ASSUMPTION] |

---

## Appendix A: Authoritative schema (DBML)

The canonical copy lives at `docs/schema.dbml`. Keep it identical to the DBML in `kiro-system-architecture-handoff.md` Section 14. Table names, column names, roles (`ADMIN`/`DRIVER`/`CUSTOMER`), statuses, and cardinalities are **fixed** - propose changes via a migration + comment, never silently.

Key invariants to preserve from the DBML:

- Every table has a `bigint` identity PK.
- Unique: `accounts.username`, `drivers.driver_code`, `drivers.citizen_id`, `driver_licenses.license_number`, `vehicles.plate_number`, `bookings.booking_code`, `customers.phone`, `activity_sessions.source_key`.
- `work_schedule (driver_id, work_date)` unique; `income_policies (driver_id, month)` unique; `payrolls (period_id, driver_id)` unique; `payroll_periods.month` unique; `payrolls.policy_id` unique.
- `accounts` links to at most one of driver/customer (see Section 5 rule 1).
- `*.month` = first day of the month.

> Known schema question (flag, do not silently fix): `payrolls.policy_id` is UNIQUE and FKs to `income_policies`, while `income_policies` is keyed by `(driver_id, month)` and `payrolls` by `(period_id, driver_id)`. If a payroll period's month can differ from the policy month, this 1:1 is ambiguous. Treat as **[NEEDS-OWNER]**.
