# Structure Steering — Driver Management Platform

> Sources: `docs/agent-guide.md` §1, §3, §5; the actual directory trees surveyed in the dmp-* repositories.

## Current actual state

The platform is split into independently versioned repositories; most are **scaffolds** whose leaf directories
contain only a `.gitkeep`. In this repository (dmp-docs) the files with real content are:
`docs/agent-guide.md`, `kiro-system-architecture-handoff.md`, `docs/schema.dbml`,
`docs/architecture/system-architecture.md` (+ `.drawio`/`.png`),
`README.md`, `AGENTS.md`, `CLAUDE.md`, and the specs under `.kiro/specs/`.
The monorepo's local CLI dispatcher (`services/agent-orchestrator/orchestrator.py` + README) and
`bootstrap-self-hosted.sh` were not carried into any dmp-* repository.
Scaffold directories are **intent**, not implemented code.

## Target multi-repo layout (docs/agent-guide.md §3, mapped to repositories)

```
dmp-web                  # React + Vite + TS (Admin); packages/{ui, api-client} — api-client generated from OpenAPI
dmp-mobile               # React Native + TS (Driver) [later]
dmp-api                  # FastAPI modular monolith (repo root: pyproject.toml, alembic.ini, alembic/versions, app/)
dmp-{event-consumer, data-processing, agent-orchestrator}  # DEFERRED / out of MVP
dmp-infra                # docker-compose.yml, docker/, kafka/, databases/, monitoring/, scripts/
dmp-docs                 # packages/contracts; data/{schemas, pipelines, seeds, notebooks};
                         # docs/{schema.dbml, agent-guide.md, architecture, decisions(ADR), api, data}; .kiro/
```

## `dmp-api` `app/` convention

```
app/
  main.py                 # FastAPI app factory
  core/                   # config, db session, security
  domains/<name>/         # <name>: accounts, areas, drivers, licenses, vehicles,
                          #          customers, bookings, leave, scheduling, payroll
```

Each `domains/<name>/` folder (strict):

- `router.py` — FastAPI routes; validate input, call service, map errors to HTTP.
- `service.py` — business rules; the only place rules live.
- `repository.py` — DB access; returns ORM/domain objects, no HTTP concerns.
- `models.py` — SQLAlchemy ORM models.
- `schemas.py` — Pydantic request/response models (NOT ORM).
- `tests/` — unit + integration tests for the domain.

Rules: routers never touch the DB directly; services never build HTTP responses; ORM is never serialized to clients.

## Reconciliation note

The current scaffold uses `dmp-api` `app/domains/{drivers,vehicles,assignments,auth}`, which diverges from §3.
When implementing: use `accounts`/`auth`, add `licenses/bookings/leave/scheduling/payroll`;
"assignments" actually belongs under `vehicles` (the `driver_vehicles` relationship).

## General rules

- Fill in by MVP order; do not spread half-finished domains.
- One migration per schema change; keep `docs/schema.dbml` **identical** to handoff Section 14.
- ADRs live in `docs/decisions` for: DB, Kafka-adoption decision, identity/authz, deployment, analytics.

## Naming hazard

"agent-orchestrator" is overloaded:
(a) the monorepo-era local CLI dispatcher `services/agent-orchestrator/orchestrator.py` (not carried into the multi-repo layout);
(b) the AO control-plane spec in `.kiro/specs/agent-orchestrator-multi-agent`, implemented in `dmp-agent-orchestrator`.
Keep the two concepts distinct; do not confuse ownership.
