# Quick Start

This guide takes you from a fresh repository to Animus-managed workflows running in
that project.

**⏱️ Total estimated time: ~6 minutes**

> **Autonomous mode is the golden path.**
> Run `animus daemon start --autonomous` and let the daemon dispatch tasks continuously.
> Manual `animus workflow run` is for one-off debugging and CI pipelines — not day-to-day operation.

## Timeline Overview

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌────────────────┐    ┌─────────────┐    ┌──────────────┐
│  Initialize │───▶│ Draft Vision │───▶│ Generate Reqs   │───▶│ Materialize    │───▶│   Start     │───▶│   Monitor    │
│   (~1 min)  │    │  (~1 min)    │    │   (~2 min)      │    │ Tasks (~1 min) │    │Daemon (~30s)│    │  (~30 sec)   │
└─────────────┘    └──────────────┘    └─────────────────┘    └────────────────┘    └─────────────┘    └──────────────┘
       │                   │                     │                      │                   │                  │
       └───────────────────┴─────────────────────┴──────────────────────┴───────────────────┴──────────────────┘
                                                               │
                                                        Autonomous PR Flow
```

For the fastest path to your first autonomous PR in **5 minutes**, see the **[First Autonomous PR Guide (5 min)](../guides/first-autonomous-pr.md)**.

## 1. Initialize the Project

**⏱️ Time: ~1 min**

```bash
cd /path/to/your/project
animus setup
animus pack list
```

`animus setup` creates the project-local `.ao/` scaffold. `animus pack list` shows the
bundled and active pack inventory available to the project.

## 2. Draft a Vision

**⏱️ Time: ~1 min**

```bash
animus vision draft
```

This resolves the canonical workflow ref `ao.vision/draft` and saves the vision
artifact through Animus-managed state.

## 3. Generate Requirements

**⏱️ Time: ~2 min**

```bash
animus requirements draft --include-codebase-scan
```

This resolves `ao.requirement/draft`. Requirement planning is now described as
a workflow surface, with compatibility aliases for the older `builtin/*` refs.

## 4. Materialize Tasks

**⏱️ Time: ~1 min**

```bash
animus requirements execute
```

This resolves `ao.requirement/execute`. The workflow creates or updates tasks
through Animus mutation surfaces such as `ao.task.create`, not through daemon-owned
business logic.

## 5. Start the Daemon

**⏱️ Time: ~30 sec**

```bash
animus daemon start --autonomous
```

The daemon handles queueing, capacity, and subprocess supervision only. Task and
requirement behavior comes from workflows, packs, MCP, and subject adapters.

## 6. Monitor Progress

**⏱️ Time: ~30 sec**

```bash
animus task stats
animus daemon status
animus workflow list
animus output tail
animus status
```

## What Happens Next

Project-local workflows such as `standard-workflow` typically wrap bundled pack
refs like `ao.task/standard`. As work completes, execution facts are projected
back onto Animus task and requirement state.

## Workflow Runs vs. Agent Runs

These two terms appear throughout the docs and CLI output. They are different
levels of the same hierarchy:

| Term | What it is | CLI record | Typical command |
|------|-----------|-----------|-----------------|
| **Workflow run** | A full phase pipeline for one task (`WF-xxx`) | `animus workflow list` | Started by daemon or `animus workflow run` |
| **Agent run** | A single phase execution within a workflow (`RUN-xxx`) | `animus output tail --run-id` | Spawned automatically by `workflow-runner` |

A single workflow run produces multiple agent runs — one per phase
(triage, research, implementation, code-review, …). When a reviewer
returns `rework`, the implementation phase agent is spawned again as a
new agent run within the same workflow run.

Use this to navigate the CLI:

```bash
# See the workflow pipeline level
animus workflow list
animus workflow get --id WF-001
animus workflow decisions --id WF-001

# Drill into a specific agent run (phase execution)
animus output tail --run-id RUN-001
animus output run --run-id RUN-001
```

## Common Pitfalls

### Task not picked up by the daemon

The daemon only dispatches tasks in `ready` status. Check:

```bash
animus task get --id TASK-001   # status must be "ready"
animus daemon status             # daemon must be running
```

If the task is stuck in `paused` or `blocked` state from a previous
interrupted run, reset it:

```bash
animus task status --id TASK-001 --status ready
```

Never hand-edit `.ao/*.json` files to fix this — the command above clears all
blocking metadata (`paused`, `blocked_at`, `blocked_reason`).

### `CLAUDECODE` environment variable blocks agents

Running the daemon inside a Claude Code session fails because Claude Code sets
`CLAUDECODE=1` in the environment and the `claude` CLI refuses to launch inside
another Claude Code session.

Fix: unset the variable before starting the daemon:

```bash
unset CLAUDECODE
animus daemon start --autonomous
# or:
env -u CLAUDECODE animus daemon start --autonomous
```

### Unexpected model routing

If all phases are using the same model instead of routing research to gemini,
check the agent runtime config:

```bash
animus workflow agent-runtime get
```

Explicit `model` and `tool` fields in `.ao/state/agent-runtime-config.v2.json`
override compiled defaults. Set them to `null` to restore defaults:

```bash
animus workflow agent-runtime set --input-json '{"agents":{"default":{"model":null,"tool":null}}}'
```

### Rework loops are a feature, not a bug

A workflow that cycles between `implementation` and `code-review` is working
correctly. The reviewer found an issue and sent work back. To see why:

```bash
animus workflow decisions --id WF-001
```

Each `rework` verdict includes the reviewer's reason. Address the root cause
rather than cancelling the workflow.

### Diagnose anything else with `animus doctor`

```bash
animus doctor
animus doctor --fix   # attempt automatic remediation
```

See the full [Troubleshooting Guide](../guides/troubleshooting.md) for
additional scenarios.

## Next Steps

- [Project Setup](project-setup.md)
- [A Typical Day](typical-day.md)
- [Workflows](../concepts/workflows.md)
- [Troubleshooting](../guides/troubleshooting.md)
