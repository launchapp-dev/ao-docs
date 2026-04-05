# animus fleet — CLI Reference

Complete reference for all 51 `animus fleet` commands. Fleet management provides node registration, agent assignment, pool sizing, scheduling, and observability across a distributed set of Animus daemon instances.

For global flags (`--json`, `--project-root`) see [Global Flags](global-flags.md). For exit codes see [Exit Codes](exit-codes.md).

---

## Command Tree

```
animus fleet
├── status                    Fleet-wide status dashboard
├── health                    Aggregated health across all nodes
├── info                      Fleet configuration summary
├── init                      Initialize fleet for the current project
│
├── node                      Node lifecycle and routing
│   ├── list                  List all registered nodes
│   ├── get                   Get node details by ID
│   ├── register              Register a new node
│   ├── remove                Remove a node from the fleet (--confirmation)
│   ├── status                Node running status
│   ├── health                Node health check
│   ├── ping                  Test connectivity to a node
│   ├── tag                   Add tag(s) to a node
│   ├── untag                 Remove tag(s) from a node
│   ├── drain                 Drain a node — reject new work
│   └── resume                Resume a drained node
│
├── agent                     Agent assignment and control
│   ├── list                  List all fleet agents
│   ├── get                   Get agent details by run ID
│   ├── assign                Assign agent to a specific node
│   ├── evict                 Evict agent from its current node
│   ├── migrate               Move agent to another node
│   ├── pause                 Pause a running agent
│   └── resume                Resume a paused agent
│
├── pool                      Pool sizing and capacity
│   ├── get                   Get pool configuration
│   ├── set                   Update pool size or limits
│   ├── stats                 Pool utilization statistics
│   ├── scale                 Scale pool capacity dynamically
│   └── reset                 Reset pool to project defaults
│
├── queue                     Distributed dispatch queue
│   ├── list                  List queued dispatches
│   ├── stats                 Queue statistics
│   ├── hold                  Hold a queued dispatch
│   ├── release               Release a held dispatch
│   └── drain                 Drain all queued dispatches (--confirmation)
│
├── schedule                  Automated work scheduling
│   ├── list                  List configured schedules
│   ├── get                   Get schedule details by ID
│   ├── create                Create a schedule
│   ├── update                Update a schedule
│   ├── remove                Remove a schedule (--confirmation)
│   └── trigger               Manually trigger a schedule
│
├── config                    Fleet configuration
│   ├── get                   Read fleet configuration
│   ├── set                   Update fleet configuration
│   ├── validate              Validate configuration
│   ├── export                Export config to a file
│   └── import                Import config from a file
│
├── sync                      State synchronisation across nodes
│   ├── status                Sync status across all nodes
│   ├── trigger               Force immediate synchronisation
│   └── cancel                Cancel an in-progress sync
│
├── events                    Fleet event log
│   ├── list                  List recent fleet events
│   ├── stream                Stream live events
│   └── clear                 Clear event history
│
└── metrics                   Fleet observability
    ├── get                   Get metrics snapshot
    └── watch                 Watch metrics continuously
```

---

## Top-Level Commands

### `animus fleet status`

Fleet-wide status dashboard. Aggregates daemon status, node counts, active agents, queue depth, and recent errors from all registered nodes.

```bash
animus fleet status
animus fleet status --json
```

**Output fields:**

| Field | Description |
|---|---|
| `nodes_total` | Total registered nodes |
| `nodes_online` | Nodes that responded to last health ping |
| `nodes_draining` | Nodes currently draining |
| `agents_active` | Total active agent runs across the fleet |
| `agents_paused` | Paused agent runs |
| `queue_depth` | Total dispatches waiting |
| `pool_utilization` | Fleet-wide pool usage percentage |
| `last_activity` | Timestamp of most recent event |

---

### `animus fleet health`

Aggregated health check across all registered nodes. Returns per-node health alongside a fleet-level summary verdict (`healthy`, `degraded`, `critical`).

```bash
animus fleet health
animus fleet health --json
```

```json
{
  "verdict": "degraded",
  "nodes": [
    { "node_id": "node-a", "status": "healthy", "uptime_secs": 86400, "active_agents": 3 },
    { "node_id": "node-b", "status": "unreachable", "last_seen": "2024-01-15T09:00:00Z" }
  ],
  "total_active_agents": 3,
  "pool_utilization_pct": 37
}
```

---

### `animus fleet info`

Fleet configuration summary — pool limits, node count, schedule count, and sync state.

```bash
animus fleet info
animus fleet info --json
```

---

### `animus fleet init`

Initialise fleet management for the current project. Creates `.ao/fleet.toml` with sensible defaults and registers the local daemon as the first node.

```bash
animus fleet init
animus fleet init --pool-size 4 --node-name primary
```

| Flag | Description |
|---|---|
| `--pool-size <N>` | Initial per-node pool size (default: `4`) |
| `--node-name <NAME>` | Label for the local node (default: hostname) |
| `--yes` | Skip interactive confirmation |

---

## node — Node Lifecycle and Routing

### `animus fleet node list`

List all registered nodes with their status and tags.

```bash
animus fleet node list
animus fleet node list --tag backend
animus fleet node list --status online
animus fleet node list --json
```

| Flag | Description |
|---|---|
| `--tag <TAG>` | Filter by tag |
| `--status <STATUS>` | Filter by status (`online`, `offline`, `draining`) |

---

### `animus fleet node get`

Get detailed information about a single node.

```bash
animus fleet node get --id node-a
animus fleet node get --id node-a --json
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node ID |

---

### `animus fleet node register`

Register a new remote node. The node must be running an Animus daemon accessible at the given address.

```bash
animus fleet node register --address 10.0.0.2:7700 --name worker-1
animus fleet node register --address 10.0.0.3:7700 --name worker-2 --tag backend --tag gpu
```

| Flag | Description |
|---|---|
| `--address <HOST:PORT>` | **Required.** Daemon address (host and port) |
| `--name <NAME>` | Human-readable label |
| `--tag <TAG>` | Tag for routing (repeatable) |
| `--pool-size <N>` | Override pool size for this node |

---

### `animus fleet node remove`

Remove a node from the fleet. Requires `--confirmation yes` to prevent accidental removal of active nodes.

```bash
animus fleet node remove --id worker-1 --confirmation yes
animus fleet node remove --id worker-1 --dry-run
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to remove |
| `--confirmation <yes\|no>` | **Required for execution.** Confirm removal |
| `--drain-first` | Drain the node before removing |
| `--dry-run` | Preview effects without removing |

---

### `animus fleet node status`

Node running status — whether the daemon process is alive and scheduling.

```bash
animus fleet node status --id worker-1
animus fleet node status --id worker-1 --json
```

---

### `animus fleet node health`

Full health metrics for a specific node.

```bash
animus fleet node health --id worker-1
animus fleet node health --id worker-1 --json
```

---

### `animus fleet node ping`

Test TCP connectivity to a node's daemon.

```bash
animus fleet node ping --id worker-1
animus fleet node ping --id worker-1 --timeout 5
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to ping |
| `--timeout <SECS>` | Connection timeout in seconds (default: `3`) |

Returns exit code `0` on success, `1` on unreachable. Safe to use in scripts.

---

### `animus fleet node tag`

Add one or more tags to a node. Tags are used for task routing (`--runner-scope`).

```bash
animus fleet node tag --id worker-1 --tag backend
animus fleet node tag --id worker-2 --tag frontend --tag gpu
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to tag |
| `--tag <TAG>` | Tag to add (repeatable) |

---

### `animus fleet node untag`

Remove a tag from a node.

```bash
animus fleet node untag --id worker-1 --tag backend
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to modify |
| `--tag <TAG>` | **Required.** Tag to remove |

---

### `animus fleet node drain`

Put a node in drain mode. The daemon stops accepting new work; in-progress agents run to completion.

```bash
animus fleet node drain --id worker-1
animus fleet node drain --id worker-1 --wait
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to drain |
| `--wait` | Block until all active agents finish |

Drain mode is useful before taking a node offline for maintenance.

---

### `animus fleet node resume`

Resume a drained node — re-enable work dispatch.

```bash
animus fleet node resume --id worker-1
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to resume |

---

## agent — Agent Assignment and Control

### `animus fleet agent list`

List all agent runs across the fleet with their node assignments.

```bash
animus fleet agent list
animus fleet agent list --node worker-1
animus fleet agent list --status running
animus fleet agent list --json
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Filter by node |
| `--status <STATUS>` | Filter by status (`running`, `paused`, `terminated`) |

---

### `animus fleet agent get`

Get full details for a specific agent run, including node assignment, task, current phase, and runtime.

```bash
animus fleet agent get --run-id abc12345
animus fleet agent get --run-id abc12345 --json
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run ID |

---

### `animus fleet agent assign`

Assign an agent run to a specific node. The run must be in a queued or paused state.

```bash
animus fleet agent assign --run-id abc12345 --node worker-2
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run to assign |
| `--node <NODE_ID>` | **Required.** Target node |

---

### `animus fleet agent evict`

Evict an agent run from its current node. The run re-enters the dispatch queue and will be picked up by the next available node.

```bash
animus fleet agent evict --run-id abc12345
animus fleet agent evict --run-id abc12345 --reason "node maintenance"
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run to evict |
| `--reason <TEXT>` | Eviction reason recorded in history |

---

### `animus fleet agent migrate`

Migrate an agent run directly from one node to another without re-queuing. The run checkpoints its state before migration.

```bash
animus fleet agent migrate --run-id abc12345 --to worker-3
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to migrate |
| `--to <NODE_ID>` | **Required.** Destination node |

---

### `animus fleet agent pause`

Pause a running agent. The agent checkpoints at the next phase boundary.

```bash
animus fleet agent pause --run-id abc12345
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to pause |

---

### `animus fleet agent resume`

Resume a paused agent run from its last checkpoint.

```bash
animus fleet agent resume --run-id abc12345
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to resume |

---

## pool — Pool Sizing and Capacity

### `animus fleet pool get`

Read the pool configuration for all nodes or a specific node.

```bash
animus fleet pool get
animus fleet pool get --node worker-1
animus fleet pool get --json
```

---

### `animus fleet pool set`

Update pool size limits for a node or the entire fleet.

```bash
# Set pool size for one node
animus fleet pool set --node worker-1 --pool-size 6

# Set fleet-wide default
animus fleet pool set --pool-size 4

# Set max per-node pool size
animus fleet pool set --max-pool-size 8
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Target node (omit for fleet-wide default) |
| `--pool-size <N>` | New pool size |
| `--max-pool-size <N>` | Maximum pool size allowed |

---

### `animus fleet pool stats`

Pool utilization statistics across all nodes.

```bash
animus fleet pool stats
animus fleet pool stats --node worker-1
animus fleet pool stats --json
```

Example output:

```
┌──────────┬────────┬────────┬────────────┬────────────────┐
│ Node     │ Active │ Pool   │ Util %     │ Queue Depth    │
├──────────┼────────┼────────┼────────────┼────────────────┤
│ primary  │ 3      │ 4      │ 75%        │ 2              │
│ worker-1 │ 1      │ 4      │ 25%        │ 0              │
│ worker-2 │ 4      │ 4      │ 100%       │ 5              │
└──────────┴────────┴────────┴────────────┴────────────────┘
Fleet: 8/12 agents active (66%), queue depth: 7
```

---

### `animus fleet pool scale`

Dynamically scale pool capacity across the fleet. Applies a scaling strategy and recalculates per-node pool sizes.

```bash
# Scale to a total fleet capacity of 16
animus fleet pool scale --total-capacity 16

# Scale up by N per node
animus fleet pool scale --delta +2

# Scale down by N per node (no-op if below minimum)
animus fleet pool scale --delta -1
```

| Flag | Description |
|---|---|
| `--total-capacity <N>` | Target total fleet capacity (distributed evenly) |
| `--delta <+N\|-N>` | Per-node capacity change |
| `--min-per-node <N>` | Never scale below this value per node (default: `1`) |

---

### `animus fleet pool reset`

Reset pool configuration to project defaults (`.ao/fleet.toml` values).

```bash
animus fleet pool reset
animus fleet pool reset --node worker-1
```

---

## queue — Distributed Dispatch Queue

### `animus fleet queue list`

List queued dispatches across all nodes, or for a specific node.

```bash
animus fleet queue list
animus fleet queue list --node worker-1
animus fleet queue list --status held
animus fleet queue list --json
```

---

### `animus fleet queue stats`

Aggregate queue statistics across the fleet.

```bash
animus fleet queue stats
animus fleet queue stats --json
```

| Output field | Description |
|---|---|
| `total` | Total queued dispatches |
| `pending` | Ready to dispatch |
| `held` | Manually held |
| `per_node` | Breakdown by node |

---

### `animus fleet queue hold`

Hold a queued dispatch — it stays in the queue but is skipped during dispatch until released.

```bash
animus fleet queue hold --subject-id TASK-005
animus fleet queue hold --subject-id TASK-005 --node worker-1
```

| Flag | Description |
|---|---|
| `--subject-id <ID>` | **Required.** Subject to hold |
| `--node <NODE_ID>` | Node queue to target (defaults to whichever node holds the subject) |

---

### `animus fleet queue release`

Release a held dispatch — resume normal dispatch eligibility.

```bash
animus fleet queue release --subject-id TASK-005
```

---

### `animus fleet queue drain`

Remove all queued dispatches. Destructive — requires `--confirmation yes`.

```bash
animus fleet queue drain --confirmation yes
animus fleet queue drain --node worker-2 --confirmation yes
animus fleet queue drain --dry-run
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Drain one node's queue (omit for entire fleet) |
| `--confirmation <yes\|no>` | **Required for execution** |
| `--dry-run` | Preview how many items would be removed |

---

## schedule — Automated Work Scheduling

### `animus fleet schedule list`

List all configured schedules.

```bash
animus fleet schedule list
animus fleet schedule list --status active
animus fleet schedule list --json
```

---

### `animus fleet schedule get`

Get details for a specific schedule.

```bash
animus fleet schedule get --id sched-001
animus fleet schedule get --id sched-001 --json
```

---

### `animus fleet schedule create`

Create a new schedule. Schedules dispatch a workflow on a cron expression or interval.

```bash
# Run nightly at 02:00
animus fleet schedule create \
  --name nightly-tests \
  --cron "0 2 * * *" \
  --workflow-ref ci \
  --task-filter '{"tags": ["test"]}'

# Run every 30 minutes
animus fleet schedule create \
  --name frequent-sync \
  --interval 30m \
  --workflow-ref quick
```

| Flag | Description |
|---|---|
| `--name <NAME>` | **Required.** Schedule name |
| `--cron <EXPR>` | Cron expression (5-field, UTC) |
| `--interval <DURATION>` | Interval (e.g. `15m`, `1h`, `6h`) |
| `--workflow-ref <REF>` | Workflow definition reference |
| `--task-filter <JSON>` | Filter expression for which tasks to include |
| `--node <NODE_ID>` | Pin schedule to a specific node |
| `--enabled` | Start enabled (default: `true`) |

Exactly one of `--cron` or `--interval` is required.

---

### `animus fleet schedule update`

Update an existing schedule.

```bash
animus fleet schedule update --id sched-001 --cron "0 3 * * *"
animus fleet schedule update --id sched-001 --enabled false
```

| Flag | Description |
|---|---|
| `--id <SCHED_ID>` | **Required.** Schedule to update |
| `--cron <EXPR>` | New cron expression |
| `--interval <DURATION>` | New interval |
| `--workflow-ref <REF>` | New workflow reference |
| `--task-filter <JSON>` | New task filter |
| `--enabled <true\|false>` | Enable or disable |

---

### `animus fleet schedule remove`

Remove a schedule. Requires `--confirmation yes`.

```bash
animus fleet schedule remove --id sched-001 --confirmation yes
```

---

### `animus fleet schedule trigger`

Manually trigger a schedule immediately, regardless of its next scheduled time.

```bash
animus fleet schedule trigger --id sched-001
animus fleet schedule trigger --id sched-001 --dry-run
```

| Flag | Description |
|---|---|
| `--id <SCHED_ID>` | **Required.** Schedule to trigger |
| `--dry-run` | Show which tasks would be dispatched |

---

## config — Fleet Configuration

### `animus fleet config get`

Read the fleet configuration from `.ao/fleet.toml`.

```bash
animus fleet config get
animus fleet config get --json
```

---

### `animus fleet config set`

Update one or more fleet configuration values.

```bash
animus fleet config set --pool-size 6
animus fleet config set --sync-interval 30s
animus fleet config set --auto-drain-on-error true
```

| Key | Type | Description |
|---|---|---|
| `pool_size` | integer | Default per-node pool size |
| `max_pool_size` | integer | Maximum per-node pool size |
| `sync_interval` | duration | How often nodes sync state |
| `heartbeat_interval` | duration | Node health ping interval |
| `auto_drain_on_error` | boolean | Drain node after repeated errors |
| `evict_on_node_loss` | boolean | Re-queue agents from unreachable nodes |

---

### `animus fleet config validate`

Validate the fleet configuration file. Reports any structural or semantic errors.

```bash
animus fleet config validate
animus fleet config validate --file /path/to/fleet.toml
```

---

### `animus fleet config export`

Export the fleet configuration to a file.

```bash
animus fleet config export --out fleet-backup.toml
animus fleet config export --out fleet-backup.json --format json
```

| Flag | Description |
|---|---|
| `--out <PATH>` | **Required.** Output path |
| `--format <toml\|json>` | Output format (default: `toml`) |

---

### `animus fleet config import`

Import fleet configuration from a file, replacing current settings.

```bash
animus fleet config import --file fleet-backup.toml
animus fleet config import --file fleet-backup.toml --dry-run
```

| Flag | Description |
|---|---|
| `--file <PATH>` | **Required.** Config file to import |
| `--dry-run` | Validate without applying |

---

## sync — State Synchronisation

### `animus fleet sync status`

Show synchronisation status across all nodes — which nodes are in sync, which are behind, and when the last sync completed.

```bash
animus fleet sync status
animus fleet sync status --json
```

---

### `animus fleet sync trigger`

Force immediate state synchronisation across all nodes.

```bash
animus fleet sync trigger
animus fleet sync trigger --node worker-2
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Sync a single node only |

---

### `animus fleet sync cancel`

Cancel an in-progress sync operation.

```bash
animus fleet sync cancel
```

---

## events — Fleet Event Log

### `animus fleet events list`

List recent events from across the fleet. Events include node registrations, agent starts/stops, pool changes, and errors.

```bash
animus fleet events list
animus fleet events list --limit 50
animus fleet events list --node worker-1
animus fleet events list --type agent_started
animus fleet events list --since 1h
animus fleet events list --json
```

| Flag | Description |
|---|---|
| `--limit <N>` | Maximum events to return (default: `100`) |
| `--node <NODE_ID>` | Filter to one node |
| `--type <EVENT_TYPE>` | Filter by event type |
| `--since <DURATION>` | Events within this window (e.g. `30m`, `2h`) |

**Common event types:**

| Type | Description |
|---|---|
| `node_registered` | Node added to fleet |
| `node_removed` | Node removed |
| `node_drained` | Node entered drain mode |
| `node_resumed` | Node exited drain mode |
| `agent_started` | Agent run began |
| `agent_completed` | Agent run finished |
| `agent_failed` | Agent run errored |
| `agent_evicted` | Agent evicted from node |
| `agent_migrated` | Agent moved between nodes |
| `pool_scaled` | Pool size changed |
| `schedule_triggered` | Schedule fired |
| `sync_completed` | State sync finished |

---

### `animus fleet events stream`

Stream live fleet events to stdout. Press Ctrl+C to stop.

```bash
animus fleet events stream
animus fleet events stream --node worker-1
animus fleet events stream --type agent_started --type agent_completed
```

---

### `animus fleet events clear`

Clear the fleet event history. Does not affect running agents.

```bash
animus fleet events clear
animus fleet events clear --older-than 7d
```

| Flag | Description |
|---|---|
| `--older-than <DURATION>` | Clear only events older than this (e.g. `7d`, `30d`) |

---

## metrics — Fleet Observability

### `animus fleet metrics get`

Get a point-in-time metrics snapshot across the fleet.

```bash
animus fleet metrics get
animus fleet metrics get --node worker-1
animus fleet metrics get --json
```

**Output fields:**

| Metric | Description |
|---|---|
| `fleet_agent_count` | Total active agents |
| `fleet_pool_utilization_pct` | Pool usage percentage |
| `fleet_queue_depth` | Total queued dispatches |
| `fleet_error_rate_1h` | Errors per hour |
| `fleet_throughput_24h` | Agent runs completed in last 24 hours |
| `per_node` | Per-node breakdown of above metrics |

---

### `animus fleet metrics watch`

Watch fleet metrics continuously, refreshing every N seconds.

```bash
animus fleet metrics watch
animus fleet metrics watch --interval 10
animus fleet metrics watch --node worker-1 --interval 5
```

| Flag | Description |
|---|---|
| `--interval <SECS>` | Refresh interval in seconds (default: `5`) |
| `--node <NODE_ID>` | Scope to one node |

Press Ctrl+C to stop.

---

## Summary

| Group | Commands |
|---|---|
| Top-level | 4 |
| `node` | 11 |
| `agent` | 7 |
| `pool` | 5 |
| `queue` | 5 |
| `schedule` | 6 |
| `config` | 5 |
| `sync` | 3 |
| `events` | 3 |
| `metrics` | 2 |
| **Total** | **51** |

---

## Related

- [Fleet Management Guide](../../guides/fleet-management.md) — Getting started and operational patterns
- [Fleet Architecture](../../architecture/fleet.md) — How fleet coordination works internally
- [Daemon Operations Guide](../../guides/daemon-operations.md) — Single-daemon management
- [Queue Management](../../guides/daemon-operations.md#queue) — Low-level queue commands
- [Global Flags](global-flags.md) — Flags available on every command
