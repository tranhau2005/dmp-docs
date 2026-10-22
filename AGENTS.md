# DriverManagementPlatform Documentation contributor guidance

System specifications, architecture, schemas, data assets, and shared contracts.

### Documentation and data conventions

- Treat confirmed owner requirements as authoritative and mark unresolved business decisions explicitly.
- Keep shared API and event contracts aligned with dmp-api, dmp-web, dmp-mobile, dmp-event-consumer, and dmp-data-processing.
- Do not store sensitive or generated data here; use synthetic seed examples only.

Coordinate API, event, and schema changes with [dmp-docs](https://github.com/tranhau2005/dmp-docs) and affected sibling repositories. Keep secrets, local environment files, generated data, and real personal data out of commits.
