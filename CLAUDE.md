# DriverManagementPlatform Documentation

System specifications, architecture, schemas, data assets, and shared contracts.

## Repository map and collaboration

**Authoritative agent guide:** [docs/agent-guide.md](docs/agent-guide.md) — pinned stack (§2), layout (§3), commands (§4), backend and migration-integrity rules (§5), MVP slice M1–M5 (§7), done-definition (§8), business rules (§9), and the D-01..D-09 decision log (§10). Read it before writing code in any dmp-* repository.

This repo (dmp-docs) holds the Kiro specifications, architecture, schema, and shared contracts. Coordinate interface changes with the sibling repositories: dmp-api, dmp-web, dmp-mobile, dmp-agent-orchestrator, dmp-event-consumer, dmp-data-processing, dmp-infra. Treat this repository as independently versioned and keep cross-repository changes explicit.

### Documentation and data conventions

- Treat confirmed owner requirements as authoritative and mark unresolved business decisions explicitly.
- Keep shared API and event contracts aligned with dmp-api, dmp-web, dmp-mobile, dmp-event-consumer, and dmp-data-processing.
- Do not store sensitive or generated data here; use synthetic seed examples only.

## Commit and branch conventions

- Commits MUST follow Conventional Commits 1.0: `<type>[(<scope>)][!]: <description>`. Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Use an imperative, lowercase description with no trailing period; keep the header at 72 characters or fewer. A body/footer is optional; mark breaking changes with `!` or a `BREAKING CHANGE:` footer.
- Branches MUST follow `<type>/<short-kebab-description>`, using one of the same types (for example `feat/m2-schema-migrations`, `fix/auth-token-expiry`, `docs/update-readme`). Never commit directly to `main`; work through pull requests.
