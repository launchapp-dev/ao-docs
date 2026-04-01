# ao fleet — CLI Reference

Complete reference for all 51 `ao fleet` commands. Fleet management provides node registration, agent assignment, pool sizing, scheduling, and observability across a distributed set of AO daemon instances.

For global flags (`--json`, `--project-root`) see [Global Flags](global-flags.md). For exit codes see [Exit Codes](exit-codes.md).

---

## Command Tree

```
ao fleet
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

### `ao fleet status`

Fleet-wide status dashboard. Aggregates daemon status, node counts, active agents, queue depth, and recent errors from all registered nodes.

```bash
ao fleet status
ao fleet status --json
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

### `ao fleet health`

Aggregated health check across all registered nodes. Returns per-node health alongside a fleet-level summary verdict (`healthy`, `degraded`, `critical`).

```bash
ao fleet health
ao fleet health --json
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

### `ao fleet info`

Fleet configuration summary — pool limits, node count, schedule count, and sync state.

```bash
ao fleet info
ao fleet info --json
```

---

### `ao fleet init`

Initialise fleet management for the current project. Creates `.ao/fleet.toml` with sensible defaults and registers the local daemon as the first node.

```bash
ao fleet init
ao fleet init --pool-size 4 --node-name primary
```

| Flag | Description |
|---|---|
| `--pool-size <N>` | Initial per-node pool size (default: `4`) |
| `--node-name <NAME>` | Label for the local node (default: hostname) |
| `--yes` | Skip interactive confirmation |

---

## node — Node Lifecycle and Routing

### `ao fleet node list`

List all registered nodes with their status and tags.

```bash
ao fleet node list
ao fleet node list --tag backend
ao fleet node list --status online
ao fleet node list --json
```

| Flag | Description |
|---|---|
| `--tag <TAG>` | Filter by tag |
| `--status <STATUS>` | Filter by status (`online`, `offline`, `draining`) |

---

### `ao fleet node get`

Get detailed information about a single node.

```bash
ao fleet node get --id node-a
ao fleet node get --id node-a --json
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node ID |

---

### `ao fleet node register`

Register a new remote node. The node must be running an AO daemon accessible at the given address.

```bash
ao fleet node register --address 10.0.0.2:7700 --name worker-1
ao fleet node register --address 10.0.0.3:7700 --name worker-2 --tag backend --tag gpu
```

| Flag | Description |
|---|---|
| `--address <HOST:PORT>` | **Required.** Daemon address (host and port) |
| `--name <NAME>` | Human-readable label |
| `--tag <TAG>` | Tag for routing (repeatable) |
| `--pool-size <N>` | Override pool size for this node |

---

### `ao fleet node remove`

Remove a node from the fleet. Requires `--confirmation yes` to prevent accidental removal of active nodes.

```bash
ao fleet node remove --id worker-1 --confirmation yes
ao fleet node remove --id worker-1 --dry-run
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to remove |
| `--confirmation <yes\|no>` | **Required for execution.** Confirm removal |
| `--drain-first` | Drain the node before removing |
| `--dry-run` | Preview effects without removing |

---

### `ao fleet node status`

Node running status — whether the daemon process is alive and scheduling.

```bash
ao fleet node status --id worker-1
ao fleet node status --id worker-1 --json
```

---

### `ao fleet node health`

Full health metrics for a specific node.

```bash
ao fleet node health --id worker-1
ao fleet node health --id worker-1 --json
```

---

### `ao fleet node ping`

Test TCP connectivity to a node's daemon.

```bash
ao fleet node ping --id worker-1
ao fleet node ping --id worker-1 --timeout 5
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to ping |
| `--timeout <SECS>` | Connection timeout in seconds (default: `3`) |

Returns exit code `0` on success, `1` on unreachable. Safe to use in scripts.

---

### `ao fleet node tag`

Add one or more tags to a node. Tags are used for task routing (`--runner-scope`).

```bash
ao fleet node tag --id worker-1 --tag backend
ao fleet node tag --id worker-2 --tag frontend --tag gpu
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to tag |
| `--tag <TAG>` | Tag to add (repeatable) |

---

### `ao fleet node untag`

Remove a tag from a node.

```bash
ao fleet node untag --id worker-1 --tag backend
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to modify |
| `--tag <TAG>` | **Required.** Tag to remove |

---

### `ao fleet node drain`

Put a node in drain mode. The daemon stops accepting new work; in-progress agents run to completion.

```bash
ao fleet node drain --id worker-1
ao fleet node drain --id worker-1 --wait
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to drain |
| `--wait` | Block until all active agents finish |

Drain mode is useful before taking a node offline for maintenance.

---

### `ao fleet node resume`

Resume a drained node — re-enable work dispatch.

```bash
ao fleet node resume --id worker-1
```

| Flag | Description |
|---|---|
| `--id <NODE_ID>` | **Required.** Node to resume |

---

## agent — Agent Assignment and Control

### `ao fleet agent list`

List all agent runs across the fleet with their node assignments.

```bash
ao fleet agent list
ao fleet agent list --node worker-1
ao fleet agent list --status running
ao fleet agent list --json
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Filter by node |
| `--status <STATUS>` | Filter by status (`running`, `paused`, `terminated`) |

---

### `ao fleet agent get`

Get full details for a specific agent run, including node assignment, task, current phase, and runtime.

```bash
ao fleet agent get --run-id abc12345
ao fleet agent get --run-id abc12345 --json
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run ID |

---

### `ao fleet agent assign`

Assign an agent run to a specific node. The run must be in a queued or paused state.

```bash
ao fleet agent assign --run-id abc12345 --node worker-2
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run to assign |
| `--node <NODE_ID>` | **Required.** Target node |

---

### `ao fleet agent evict`

Evict an agent run from its current node. The run re-enters the dispatch queue and will be picked up by the next available node.

```bash
ao fleet agent evict --run-id abc12345
ao fleet agent evict --run-id abc12345 --reason "node maintenance"
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Agent run to evict |
| `--reason <TEXT>` | Eviction reason recorded in history |

---

### `ao fleet agent migrate`

Migrate an agent run directly from one node to another without re-queuing. The run checkpoints its state before migration.

```bash
ao fleet agent migrate --run-id abc12345 --to worker-3
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to migrate |
| `--to <NODE_ID>` | **Required.** Destination node |

---

### `ao fleet agent pause`

Pause a running agent. The agent checkpoints at the next phase boundary.

```bash
ao fleet agent pause --run-id abc12345
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to pause |

---

### `ao fleet agent resume`

Resume a paused agent run from its last checkpoint.

```bash
ao fleet agent resume --run-id abc12345
```

| Flag | Description |
|---|---|
| `--run-id <RUN_ID>` | **Required.** Run to resume |

---

## pool — Pool Sizing and Capacity

### `ao fleet pool get`

Read the pool configuration for all nodes or a specific node.

```bash
ao fleet pool get
ao fleet pool get --node worker-1
ao fleet pool get --json
```

---

### `ao fleet pool set`

Update pool size limits for a node or the entire fleet.

```bash
# Set pool size for one node
ao fleet pool set --node worker-1 --pool-size 6

# Set fleet-wide default
ao fleet pool set --pool-size 4

# Set max per-node pool size
ao fleet pool set --max-pool-size 8
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Target node (omit for fleet-wide default) |
| `--pool-size <N>` | New pool size |
| `--max-pool-size <N>` | Maximum pool size allowed |

---

### `ao fleet pool stats`

Pool utilization statistics across all nodes.

```bash
ao fleet pool stats
ao fleet pool stats --node worker-1
ao fleet pool stats --json
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

### `ao fleet pool scale`

Dynamically scale pool capacity across the fleet. Applies a scaling strategy and recalculates per-node pool sizes.

```bash
# Scale to a total fleet capacity of 16
ao fleet pool scale --total-capacity 16

# Scale up by N per node
ao fleet pool scale --delta +2

# Scale down by N per node (no-op if below minimum)
ao fleet pool scale --delta -1
```

| Flag | Description |
|---|---|
| `--total-capacity <N>` | Target total fleet capacity (distributed evenly) |
| `--delta <+N\|-N>` | Per-node capacity change |
| `--min-per-node <N>` | Never scale below this value per node (default: `1`) |

---

### `ao fleet pool reset`

Reset pool configuration to project defaults (`.ao/fleet.toml` values).

```bash
ao fleet pool reset
ao fleet pool reset --node worker-1
```

---

## queue — Distributed Dispatch Queue

### `ao fleet queue list`

List queued dispatches across all nodes, or for a specific node.

```bash
ao fleet queue list
ao fleet queue list --node worker-1
ao fleet queue list --status held
ao fleet queue list --json
```

---

### `ao fleet queue stats`

Aggregate queue statistics across the fleet.

```bash
ao fleet queue stats
ao fleet queue stats --json
```

| Output field | Description |
|---|---|
| `total` | Total queued dispatches |
| `pending` | Ready to dispatch |
| `held` | Manually held |
| `per_node` | Breakdown by node |

---

### `ao fleet queue hold`

Hold a queued dispatch — it stays in the queue but is skipped during dispatch until released.

```bash
ao fleet queue hold --subject-id TASK-005
ao fleet queue hold --subject-id TASK-005 --node worker-1
```

| Flag | Description |
|---|---|
| `--subject-id <ID>` | **Required.** Subject to hold |
| `--node <NODE_ID>` | Node queue to target (defaults to whichever node holds the subject) |

---

### `ao fleet queue release`

Release a held dispatch — resume normal dispatch eligibility.

```bash
ao fleet queue release --subject-id TASK-005
```

---

### `ao fleet queue drain`

Remove all queued dispatches. Destructive — requires `--confirmation yes`.

```bash
ao fleet queue drain --confirmation yes
ao fleet queue drain --node worker-2 --confirmation yes
ao fleet queue drain --dry-run
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Drain one node's queue (omit for entire fleet) |
| `--confirmation <yes\|no>` | **Required for execution** |
| `--dry-run` | Preview how many items would be removed |

---

## schedule — Automated Work Scheduling

### `ao fleet schedule list`

List all configured schedules.

```bash
ao fleet schedule list
ao fleet schedule list --status active
ao fleet schedule list --json
```

---

### `ao fleet schedule get`

Get details for a specific schedule.

```bash
ao fleet schedule get --id sched-001
ao fleet schedule get --id sched-001 --json
```

---

### `ao fleet schedule create`

Create a new schedule. Schedules dispatch a workflow on a cron expression or interval.

```bash
# Run nightly at 02:00
ao fleet schedule create \
  --name nightly-tests \
  --cron "0 2 * * *" \
  --workflow-ref ci \
  --task-filter '{"tags": ["test"]}'

# Run every 30 minutes
ao fleet schedule create \
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

### `ao fleet schedule update`

Update an existing schedule.

```bash
ao fleet schedule update --id sched-001 --cron "0 3 * * *"
ao fleet schedule update --id sched-001 --enabled false
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

### `ao fleet schedule remove`

Remove a schedule. Requires `--confirmation yes`.

```bash
ao fleet schedule remove --id sched-001 --confirmation yes
```

---

### `ao fleet schedule trigger`

Manually trigger a schedule immediately, regardless of its next scheduled time.

```bash
ao fleet schedule trigger --id sched-001
ao fleet schedule trigger --id sched-001 --dry-run
```

| Flag | Description |
|---|---|
| `--id <SCHED_ID>` | **Required.** Schedule to trigger |
| `--dry-run` | Show which tasks would be dispatched |

---

## config — Fleet Configuration

### `ao fleet config get`

Read the fleet configuration from `.ao/fleet.toml`.

```bash
ao fleet config get
ao fleet config get --json
```

---

### `ao fleet config set`

Update one or more fleet configuration values.

```bash
ao fleet config set --pool-size 6
ao fleet config set --sync-interval 30s
ao fleet config set --auto-drain-on-error true
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

### `ao fleet config validate`

Validate the fleet configuration file. Reports any structural or semantic errors.

```bash
ao fleet config validate
ao fleet config validate --file /path/to/fleet.toml
```

---

### `ao fleet config export`

Export the fleet configuration to a file.

```bash
ao fleet config export --out fleet-backup.toml
ao fleet config export --out fleet-backup.json --format json
```

| Flag | Description |
|---|---|
| `--out <PATH>` | **Required.** Output path |
| `--format <toml\|json>` | Output format (default: `toml`) |

---

### `ao fleet config import`

Import fleet configuration from a file, replacing current settings.

```bash
ao fleet config import --file fleet-backup.toml
ao fleet config import --file fleet-backup.toml --dry-run
```

| Flag | Description |
|---|---|
| `--file <PATH>` | **Required.** Config file to import |
| `--dry-run` | Validate without applying |

---

## sync — State Synchronisation

### `ao fleet sync status`

Show synchronisation status across all nodes — which nodes are in sync, which are behind, and when the last sync completed.

```bash
ao fleet sync status
ao fleet sync status --json
```

---

### `ao fleet sync trigger`

Force immediate state synchronisation across all nodes.

```bash
ao fleet sync trigger
ao fleet sync trigger --node worker-2
```

| Flag | Description |
|---|---|
| `--node <NODE_ID>` | Sync a single node only |

---

### `ao fleet sync cancel`

Cancel an in-progress sync operation.

```bash
ao fleet sync cancel
```

---

## events — Fleet Event Log

### `ao fleet events list`

List recent events from across the fleet. Events include node registrations, agent starts/stops, pool changes, and errors.

```bash
ao fleet events list
ao fleet events list --limit 50
ao fleet events list --node worker-1
ao fleet events list --type agent_started
ao fleet events list --since 1h
ao fleet events list --json
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

### `ao fleet events stream`

Stream live fleet events to stdout. Press Ctrl+C to stop.

```bash
ao fleet events stream
ao fleet events stream --node worker-1
ao fleet events stream --type agent_started --type agent_completed
```

---

### `ao fleet events clear`

Clear the fleet event history. Does not affect running agents.

```bash
ao fleet events clear
ao fleet events clear --older-than 7d
```

| Flag | Description |
|---|---|
| `--older-than <DURATION>` | Clear only events older than this (e.g. `7d`, `30d`) |

---

## metrics — Fleet Observability

### `ao fleet metrics get`

Get a point-in-time metrics snapshot across the fleet.

```bash
ao fleet metrics get
ao fleet metrics get --node worker-1
ao fleet metrics get --json
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

### `ao fleet metrics watch`

Watch fleet metrics continuously, refreshing every N seconds.

```bash
ao fleet metrics watch
ao fleet metrics watch --interval 10
ao fleet metrics watch --node worker-1 --interval 5
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
