# Product Steering — Driver Management Platform

> Sources: `docs/agent-guide.md`, `kiro-system-architecture-handoff.md`, `docs/schema.dbml`.
> This is the product-direction document. On conflict with older descriptions, this file + the DBML + docs/agent-guide.md win.

Owner-confirmed business rules (source revision 2026-10-27): [docs/business-rules.md](../../docs/business-rules.md).

## What the product is

A platform to manage the driver-workforce lifecycle for a **single organization**, covering:

- Accounts & access control (accounts).
- Area catalog (areas).
- Driver profiles, employment history, driving licenses (drivers, employment_history, driver_licenses).
- Vehicles and driver–vehicle assignment (vehicles, driver_vehicles).
- Customers and bookings (customers, bookings).
- Leave / approval (leave_requests, leave_actions).
- Work schedule & timekeeping (work_schedule, activity_sessions).
- Income policy & payroll (income_policies, payroll_periods, payrolls, payroll_adjustments).
- Three named reports: monthly driver status, workforce composition, payroll PDF.

## Actors (from the data model only)

- **ADMIN** — administrative account. Permission matrix: **Resolved by bao-cao.tex**, see [business rules](../../docs/business-rules.md#permission-matrix).
- **DRIVER** — account optionally linked 1:1 to a driver profile.
- **CUSTOMER** — account optionally linked 1:1 to a customer profile.

Do not invent manager/dispatcher/team-leader roles — the ERD has none
(the DBML explicitly states there is no team leader or driver-team table).

## Scope discipline

- **MVP = docs/agent-guide.md §7 (M1–M5):** scaffold + health, schema + migrations, accounts + auth,
  drivers + licenses, vehicles + assignment. Build the synchronous path, then **stop for review**.
- **Out of MVP:** Kafka, event-consumer, data-processing, analytics lakehouse (D-04 = No for now).
- **Resolved by bao-cao.tex:** booking, leave, status mapping, timekeeping, payroll, reports, deletion and the permission matrix follow [business rules](../../docs/business-rules.md). Q1–Q10 are decided; see [owner decisions](../../docs/business-rules.md#owner-decisions) (OWNER-APPROVED: Q1 through OWNER-APPROVED: Q10).
- **Single-organization only.** No multi-tenancy (the ERD has no tenant table).

## Source-of-truth hierarchy

1. `bao-cao.tex` / [business rules](../../docs/business-rules.md) for business rules and `docs/schema.dbml` for structure; `docs/agent-guide.md` for conventions and MVP.
2. `docs/architecture/system-architecture.md` — **target architecture / roadmap** (NOT the MVP).
3. `.kiro/specs/_archived/driver-management-dashboard` — **non-authoritative** (a different, simpler model with Viewer/Manager roles). Treat as archived.

## Non-functional intent (unquantified — `[NEEDS-OWNER]` for concrete targets)

PII protection, auditability, availability, and backup/restore all matter; no SLA/SLO
numbers are confirmed. Do not invent them.

## Open decision log (from docs/agent-guide.md §10 / handoff §10)

| ID | Decision | Current default | Status |
|---|---|---|---|
| D-01 | MVP scope | M1–M5 | ASSUMPTION |
| D-02 | Multi-tenant? | Single-org, no tenant table | ASSUMPTION |
| D-03 | DB ops / backup | dev compose only | NEEDS-OWNER (prod) |
| D-04 | Kafka in MVP? | No | ASSUMPTION |
| D-05 | Booking/leave/timekeeping/assignment lifecycle | [Business rules](../../docs/business-rules.md) | Resolved by bao-cao.tex |
| D-06 | Identity / authz model | JWT + role claim | ASSUMPTION |
| D-07 | Cloud/region/topology | undecided | NEEDS-OWNER |
| D-08 | Report definitions | [Reports](../../docs/business-rules.md#reports) | Resolved by bao-cao.tex |
| D-09 | AI/agent capabilities | none in MVP | ASSUMPTION |
