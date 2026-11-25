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

## Agent roles (Orca)

This session is the master node and runs `claude-opus-5-5`. It brainstorms, plans, and coordinates; it does not edit product code, and it does not merge.

- Dispatch implementation to Codex workers (`--agent codex --model gpt-6.1-terra --effort xhigh`), including workers in sibling repositories.
- Dispatch pull request review and merge to the OpenCode reviewer (`--agent opencode --model opencode-go/deepseek-v4.1-flash`). Only the reviewer merges, and only under the merge conditions in [AGENTS.md](AGENTS.md#merge-conditions).
- Commands, the delivery flow, and the merge conditions are in [AGENTS.md](AGENTS.md#agent-roles-orca). Auto mode does not grant permission to deploy or publish.

## Skill workflow (Claude)

- Before multi-step delivery, run the `ak-brainstorm` skill to lock the outcome, constraints, non-goals, and acceptance criteria, and to compare viable approaches.
- Then run the `ak-plan` skill to write a phased plan with acceptance criteria and verification steps. Hand the accepted plan to a Codex worker for implementation, then hand the worker's pull request to the OpenCode reviewer. The worker and reviewer skills are listed in [AGENTS.md](AGENTS.md#skill-workflow-codex).
