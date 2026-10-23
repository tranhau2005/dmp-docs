# DriverManagementPlatform Documentation contributor guidance

System specifications, architecture, schemas, data assets, and shared contracts.

**Authoritative agent guide:** [docs/agent-guide.md](docs/agent-guide.md) — pinned stack (§2), layout (§3), commands (§4), backend and migration-integrity rules (§5), MVP slice M1–M5 (§7), done-definition (§8), `[NEEDS-OWNER]` rules (§9), and the D-01..D-09 decision log (§10). Read it before writing code in any dmp-* repository.

### Documentation and data conventions

- Treat confirmed owner requirements as authoritative and mark unresolved business decisions explicitly.
- Keep shared API and event contracts aligned with dmp-api, dmp-web, dmp-mobile, dmp-event-consumer, and dmp-data-processing.
- Do not store sensitive or generated data here; use synthetic seed examples only.

Coordinate API, event, and schema changes in this repo (dmp-docs) with the affected sibling repositories. Keep secrets, local environment files, generated data, and real personal data out of commits.
