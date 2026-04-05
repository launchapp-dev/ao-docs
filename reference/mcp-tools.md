# MCP Tools Reference

All MCP tools exposed by `animus mcp serve`. These tools allow AI agents to interact with the Animus orchestrator over the Model Context Protocol. Each tool wraps an `ao` CLI command, accepting JSON input and returning structured results.

Every tool accepts an optional `project_root` parameter to override the default project root.

**Verified tool count: 73 tools across 8 categories** (plus 2 envelope schema types)

---

## Agent Control (3 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.agent.run` | Launch an AI agent to execute work | `tool`, `model`, `prompt`, `detach`, `task_id`, `input_json`, `run_id`, `runtime_contract_json`, `context_json`, `cwd`, `project_root`, `timeout_secs` |
| `ao.agent.control` | Control a running agent | `run_id`, `action`, `runner_scope` |
| `ao.agent.status` | Get status of an agent run | `run_id`, `runner_scope` |

---

## Daemon Management (11 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.daemon.start` | Start the Animus daemon | `interval_secs`, `autonomous`, `resume_interrupted`, `reconcile_stale`, `startup_cleanup`, `skip_runner`, `pool_size`, `max_tasks_per_tick`, `auto_merge`, `auto_pr`, `auto_commit_before_merge`, `auto_prune_worktrees_after_merge`, `auto_run_ready`, `stale_threshold_hours`, `idle_timeout_secs`, `phase_timeout_secs`, `runner_scope`, `project_root` |
| `ao.daemon.stop` | Stop the Animus daemon | `project_root` |
| `ao.daemon.status` | Get daemon status | `project_root` |
| `ao.daemon.health` | Check daemon health | `project_root` |
| `ao.daemon.pause` | Pause the daemon scheduler | `project_root` |
| `ao.daemon.resume` | Resume the daemon scheduler | `project_root` |
| `ao.daemon.events` | List recent daemon events | `limit`, `project_root` |
| `ao.daemon.agents` | List active daemon agents | `project_root` |
| `ao.daemon.logs` | Read daemon log file | `limit`, `search`, `project_root` |
| `ao.daemon.config` | Read daemon configuration | `project_root` |
| `ao.daemon.config-set` | Update daemon configuration | `pool_size`, `interval_secs`, `max_tasks_per_tick`, `auto_run_ready`, `stale_threshold_hours`, `phase_timeout_secs`, `idle_timeout_secs`, `auto_merge`, `auto_pr`, `auto_commit_before_merge`, `auto_prune_worktrees_after_merge`, `project_root` |

---

## Task Operations (20 tools)

### Query Tools (6)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.task.list` | List tasks with optional filters (status, priority, type, assignee, tags, linked requirements), plus sort and pagination hints | `status`, `priority`, `task_type`, `assignee_type`, `tag[]`, `risk`, `linked_requirement`, `linked_architecture_entity`, `search`, `limit`, `offset`, `max_tokens` |
| `ao.task.get` | Fetch a task by its ID | `id` |
| `ao.task.prioritized` | List tasks in priority order | `status`, `priority`, `assignee_type`, `search`, `limit`, `offset`, `max_tokens` |
| `ao.task.next` | Get the next task to work on | `project_root` |
| `ao.task.stats` | Get task statistics | `project_root` |
| `ao.task.history` | Get workflow dispatch history for a task | `id` |

### Mutation Tools (14)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.task.create` | Create a new task in Animus | `title`, `description`, `priority`, `task_type`, `linked_requirement[]`, `assignee` |
| `ao.task.update` | Update task fields | `id`, `title`, `description`, `priority`, `status`, `assignee`, `linked_architecture_entity[]`, `replace_linked_architecture_entities`, `input_json` |
| `ao.task.delete` | Delete a task from Animus | `id`, `confirm`, `dry_run` |
| `ao.task.status` | Update the status of a task | `id`, `status` |
| `ao.task.assign` | Assign a task to a user or agent | `id`, `assignee`, `assignee_type`, `agent_role`, `model` |
| `ao.task.pause` | Pause a running task | `id` |
| `ao.task.resume` | Resume a paused task | `id` |
| `ao.task.cancel` | Cancel a task | `id`, `confirm`, `dry_run` |
| `ao.task.set-priority` | Set task priority | `id`, `priority` |
| `ao.task.set-deadline` | Set or clear a task deadline | `id`, `deadline` |
| `ao.task.checklist-add` | Add a checklist item to a task | `id`, `description` |
| `ao.task.checklist-update` | Mark a checklist item complete or incomplete | `id`, `item_id`, `completed` |
| `ao.task.bulk-status` | Batch-update status for multiple tasks | `updates[]` (each: `id`, `status`), `on_error` |
| `ao.task.bulk-update` | Batch-update fields for multiple tasks | `updates[]` (each: `id` + fields), `on_error` |

---

## Workflow Operations (16 tools)

### Runtime Tools (11)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.workflow.run` | Run a workflow for a task | `task_id`, `requirement_id`, `title`, `description`, `workflow_ref`, `input_json` |
| `ao.workflow.run-multiple` | Run a workflow for multiple tasks in one call | `runs[]` (each: `task_id`, `workflow_ref`, `input_json`), `on_error` |
| `ao.workflow.execute` | Execute a workflow synchronously | `task_id`, `workflow_ref`, `phase`, `model`, `tool`, `phase_timeout_secs`, `input_json` |
| `ao.workflow.get` | Get workflow details by ID | `id` |
| `ao.workflow.list` | List workflows with optional filters | `status`, `workflow_ref`, `task_id`, `phase_id`, `search`, `limit`, `offset`, `max_tokens` |
| `ao.workflow.pause` | Pause a running workflow | `id`, `confirm`, `dry_run` |
| `ao.workflow.cancel` | Cancel a running workflow | `id`, `confirm`, `dry_run` |
| `ao.workflow.resume` | Resume a paused workflow | `id` |
| `ao.workflow.phase.approve` | Approve a gated workflow phase | `workflow_id`, `phase_id`, `feedback` |
| `ao.workflow.decisions` | List workflow decisions | `id`, `limit`, `offset`, `max_tokens` |
| `ao.workflow.checkpoints.list` | List workflow checkpoints | `id`, `limit`, `offset`, `max_tokens` |

### Definition & Config Tools (5)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.workflow.phases.list` | List workflow phase definitions | `project_root` |
| `ao.workflow.phases.get` | Get a workflow phase definition | `phase` |
| `ao.workflow.definitions.list` | List workflow definitions | `project_root` |
| `ao.workflow.config.get` | Read effective workflow config | `project_root` |
| `ao.workflow.config.validate` | Validate workflow config | `project_root` |

---

## Requirements (6 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.requirements.list` | List requirements with optional filters (status, priority, category, type, tags, linked task), plus sort and pagination hints | `limit`, `offset`, `max_tokens`, `status`, `priority`, `category`, `tag[]`, `type`, `search`, `sort` |
| `ao.requirements.get` | Get a requirement by its ID | `id` |
| `ao.requirements.create` | Create a requirement in Animus | `title`, `description`, `priority`, `category`, `type`, `acceptance_criterion[]`, `source`, `linked_requirement[]` |
| `ao.requirements.update` | Update a requirement in Animus | `id`, `title`, `description`, `priority`, `status`, `category`, `type`, `acceptance_criterion[]`, `replace_acceptance_criteria`, `source`, `linked_task_id[]` |
| `ao.requirements.delete` | Delete a requirement from Animus | `id` |
| `ao.requirements.refine` | Refine requirements with optional AI assistance | `id[]`, `focus`, `use_ai`, `model`, `tool`, `start_runner`, `input_json` |

---

## Queue Operations (7 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.queue.list` | List queued subject dispatches | `project_root` |
| `ao.queue.stats` | Show queue statistics | `project_root` |
| `ao.queue.enqueue` | Enqueue a subject dispatch | `task_id`, `requirement_id`, `title`, `description`, `workflow_ref`, `input_json` |
| `ao.queue.reorder` | Reorder queued subject dispatches | `subject_ids[]` |
| `ao.queue.hold` | Hold a queued subject dispatch | `subject_id` |
| `ao.queue.release` | Release a held queued subject dispatch | `subject_id` |
| `ao.queue.drop` | Drop (remove) a queued subject dispatch | `subject_id` |

---

## Output & Monitoring (6 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.output.run` | Get output for an agent run | `run_id`, `project_root` |
| `ao.output.tail` | Get the most recent output, error, or thinking events | `run_id`, `task_id`, `event_types[]`, `limit`, `project_root` |
| `ao.output.monitor` | Monitor output for a run, task, or phase | `run_id`, `task_id`, `phase_id`, `project_root` |
| `ao.output.jsonl` | Get JSONL log for an agent run | `run_id`, `entries`, `project_root` |
| `ao.output.artifacts` | Get artifacts for an execution | `execution_id`, `project_root` |
| `ao.output.phase-outputs` | Get persisted workflow phase outputs | `workflow_id`, `phase_id`, `project_root` |

---

## Runner (4 tools)

| Tool | Description | Key Parameters |
|---|---|---|
| `ao.runner.health` | Check runner process health | `project_root` |
| `ao.runner.orphans-detect` | Detect orphaned runner processes | `project_root` |
| `ao.runner.orphans-cleanup` | Clean up orphaned runner processes | `run_id[]`, `project_root` |
| `ao.runner.restart-stats` | Get runner restart statistics | `project_root` |

---

## List Tool Pagination

All list tools support pagination via these common parameters:

| Parameter | Type | Default | Max | Description |
|---|---|---|---|---|
| `limit` | integer | 25 | 200 | Maximum items to return |
| `offset` | integer | 0 | -- | Items to skip |
| `max_tokens` | integer | 3000 | 12000 | Token budget for response compaction (min: 256) |

List responses are wrapped in a guard envelope (`ao.mcp.list.result.v1`) that includes pagination metadata.

## Batch Tool Behavior

Batch tools (`ao.task.bulk-status`, `ao.task.bulk-update`, `ao.workflow.run-multiple`) accept an `on_error` parameter:

| Value | Behavior |
|---|---|
| `"continue"` | Process all items regardless of failures |
| `"stop"` | Stop processing after the first failure; remaining items are marked `"skipped"` |

Batch responses use the `ao.mcp.batch.result.v1` schema with a summary of succeeded/failed/skipped counts and per-item results.

Maximum batch size is 100 items per call.

See also: [JSON Envelope Contract](json-envelope.md), [CLI Command Surface](cli/index.md).
