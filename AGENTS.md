# DriverManagementPlatform Documentation contributor guidance

System specifications, architecture, schemas, data assets, and shared contracts.

**Authoritative agent guide:** [docs/agent-guide.md](docs/agent-guide.md) — pinned stack (§2), layout (§3), commands (§4), backend and migration-integrity rules (§5), MVP slice M1–M5 (§7), done-definition (§8), business rules (§9), and the D-01..D-09 decision log (§10). Read it before writing code in any dmp-* repository.

### Documentation and data conventions

- Treat confirmed owner requirements as authoritative and mark unresolved business decisions explicitly.
- Keep shared API and event contracts aligned with dmp-api, dmp-web, dmp-mobile, dmp-event-consumer, and dmp-data-processing.
- Do not store sensitive or generated data here; use synthetic seed examples only.

Coordinate API, event, and schema changes in this repo (dmp-docs) with the affected sibling repositories. Keep secrets, local environment files, generated data, and real personal data out of commits.

## Agent roles (Orca)

Development is orchestrated through Orca with these roles:

| Role | Agent | Model / mode |
|---|---|---|
| Orchestrator | Claude (`claude`) | `claude-opus-5-5` |
| Worker | Codex (`codex`) | `gpt-6.1-terra`, effort `xhigh` |
| Reviewer | Codex (`codex`) | `gpt-6.1-terra`, effort `xhigh` |

All three run in auto mode, without per-command permission prompts. Enable it in Orca desktop Settings > Agents. This repository does not commit permission-bypass configuration. Do not switch agent or model on quota or access errors; report the exact error and wait for a decision.

Dispatch a worker from a branch named `<type>/<short-description>`:

```sh
orca orchestration worker-start --spec "<task with acceptance criteria>" \
  --worktree current \
  --agent codex --model gpt-6.1-terra --effort xhigh \
  --task-title "<short title>"
```

When the worker finishes, dispatch a second Codex worker to review the same worktree (`current` or that worktree's `path:`), then the orchestrator reviews the diff and checks itself:

```sh
orca orchestration worker-start --worktree current \
  --agent codex --model gpt-6.1-terra --effort xhigh \
  --task-title "review <short title>" \
  --spec "Review the diff of this branch against origin/main for bugs, regressions and AGENTS.md violations. Report findings with file:line; do not edit files."
```

To dispatch into another repository registered in Orca (`orca repo list`), create a worktree with a conventional branch, then start the worker on that path. Do not use `--worktree new-top-level`; it picks a branch name outside the convention.

```sh
orca worktree create --repo name:dmp-web --name feat-driver-search \
  --branch feat/driver-search --base-branch origin/main --json
orca orchestration worker-start \
  --worktree path:/home/haukun/orca/workspaces/dmp-web/feat-driver-search \
  --agent codex --model gpt-6.1-terra --effort xhigh \
  --task-title "driver search UI" \
  --spec "<task, owned files, acceptance criteria, checks to run>"
```

The worker follows that repository's `AGENTS.md`. For work that spans repositories, start one worker per repository, state the shared API, event, or schema contract in each spec, and pass `--deps '["<task_id>"]'` when a worker needs another worker's result first.

Auto mode does not grant permission to merge, deploy, or publish. Merge only with explicit owner approval.
