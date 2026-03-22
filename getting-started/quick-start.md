# Quick Start

This guide takes you from a fresh repository to AO-managed workflows running in
that project.

**⏱️ Total estimated time: ~6 minutes**

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
ao setup
ao pack list
```

`ao setup` creates the project-local `.ao/` scaffold. `ao pack list` shows the
bundled and active pack inventory available to the project.

## 2. Draft a Vision

**⏱️ Time: ~1 min**

```bash
ao vision draft
```

This resolves the canonical workflow ref `ao.vision/draft` and saves the vision
artifact through AO-managed state.

## 3. Generate Requirements

**⏱️ Time: ~2 min**

```bash
ao requirements draft --include-codebase-scan
```

This resolves `ao.requirement/draft`. Requirement planning is now described as
a workflow surface, with compatibility aliases for the older `builtin/*` refs.

## 4. Materialize Tasks

**⏱️ Time: ~1 min**

```bash
ao requirements execute
```

This resolves `ao.requirement/execute`. The workflow creates or updates tasks
through AO mutation surfaces such as `ao.task.create`, not through daemon-owned
business logic.

## 5. Start the Daemon

**⏱️ Time: ~30 sec**

```bash
ao daemon start --autonomous
```

The daemon handles queueing, capacity, and subprocess supervision only. Task and
requirement behavior comes from workflows, packs, MCP, and subject adapters.

## 6. Monitor Progress

**⏱️ Time: ~30 sec**

```bash
ao task stats
ao daemon status
ao workflow list
ao output tail
ao status
```

## What Happens Next

Project-local workflows such as `standard-workflow` typically wrap bundled pack
refs like `ao.task/standard`. As work completes, execution facts are projected
back onto AO task and requirement state.

## Next Steps

- [Project Setup](project-setup.md)
- [A Typical Day](typical-day.md)
- [Workflows](../concepts/workflows.md)
