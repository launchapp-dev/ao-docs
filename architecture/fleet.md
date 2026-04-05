# Fleet Architecture

`animus fleet` adds a coordination layer on top of one or more single-node Animus daemons. This document describes how fleet state is represented, how nodes communicate, how work is distributed, and how the system recovers from failures.

---

## Overview

A fleet is a named set of **nodes**. Each node is an independent Animus daemon instance with its own task store, queue, and agent runner. The fleet coordinator — a process that runs alongside the primary node's daemon — maintains a registry of all nodes, synchronises shared state, and provides the aggregated view that `animus fleet` commands expose.

```
┌─────────────────────────────────────────────────────┐
│                   Fleet Coordinator                 │
│   (runs alongside primary daemon; writes fleet.db)  │
│                                                     │
│  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  Registry  │  │  Scheduler  │  │   Metrics   │  │
│  │  (nodes,   │  │ (schedules, │  │ (aggregated │  │
│  │   tags)    │  │  balancer)  │  │  snapshots) │  │
│  └────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────┬───────────────────────────┘
                          │ HTTP (localhost or mTLS)
         ┌────────────────┼───────────────┐
         ▼                ▼               ▼
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Node A    │  │  Node B    │  │  Node C    │
  │ (primary)  │  │ (worker-1) │  │ (worker-2) │
  │            │  │            │  │            │
  │  daemon    │  │  daemon    │  │  daemon    │
  │  runner    │  │  runner    │  │  runner    │
  │  store     │  │  store     │  │  store     │
  └────────────┘  └────────────┘  └────────────┘
```

---

## Components

### Fleet Coordinator

The coordinator is embedded in the `orchestrator-daemon-runtime` crate and activates when fleet mode is enabled (`.ao/fleet.toml` present). It runs in the same process as the primary daemon and exposes an internal API used by the `animus fleet` CLI commands.

Responsibilities:
- Maintain the node registry (`fleet.db`)
- Perform periodic health pings to all registered nodes
- Collect and aggregate metrics
- Evaluate and fire schedules
- Route dispatch decisions to the least-loaded eligible node

### Node Agents

Each node runs an unmodified Animus daemon. The daemon exposes the same JSON-over-HTTP API used for local MCP operations. The fleet coordinator connects to this API to:
- Query health and status
- Push dispatch instructions
- Read queue depth and agent counts
- Manage pool size

Nodes do not communicate with each other. All coordination flows through the coordinator.

### Fleet Store (`fleet.db`)

A SQLite database in `.ao/fleet.db` on the primary node. Tables:

| Table | Description |
|---|---|
| `nodes` | Registered nodes (id, address, tags, pool size, drain state) |
| `node_health` | Most recent health snapshot per node |
| `schedules` | Configured cron/interval schedules |
| `schedule_runs` | Schedule execution history |
| `fleet_events` | Event log (node and agent lifecycle events) |
| `sync_state` | Per-node sync cursor |

All writes to `fleet.db` are serialised through a Tokio mutex to prevent races between the coordinator's background tasks.

---

## Node Communication

### Transport

The coordinator communicates with nodes using the same HTTP API that `animus mcp serve` exposes — the same JSON-RPC-style envelope used for local MCP tool calls. No additional protocol is introduced.

For local deployments, communication is over `127.0.0.1`. For remote nodes, the connection uses plain HTTP by default; mTLS can be configured via `fleet.toml` for production multi-machine deployments.

### Heartbeat and Health Pings

The coordinator sends a lightweight `daemon.health` probe to each node at the configured `heartbeat_interval` (default: `10s`). The response is stored in `node_health` and used to compute the fleet health verdict:

| Condition | Verdict |
|---|---|
| All nodes `online` | `healthy` |
| Any node `unreachable`, all others `online` | `degraded` |
| Majority of nodes `unreachable` | `critical` |

A node is marked `unreachable` after three consecutive missed heartbeats.

### Sync

State sync keeps the fleet coordinator's view of each node consistent with the node's actual state (active agents, queue depth, pool configuration). Sync is not a distributed commit protocol — the node's local state is always authoritative for its own work.

The sync cycle:
1. Coordinator polls each online node for its current `daemon.status`, `daemon.agents`, and `queue.stats` payloads.
2. Results are written to `fleet.db` with a timestamp.
3. `animus fleet sync status` reports which nodes have been updated within the last sync window.

Sync interval is configurable (`sync_interval`, default: `30s`).

---

## Work Distribution

### Dispatch Flow

When a task is ready for dispatch, the coordinator's scheduler:

1. Reads the fleet node list, filtering to `online` and not `draining` nodes.
2. Filters by tag if the task has a `runner_scope` set.
3. Sorts by available capacity (pool size minus active agents).
4. Selects the node with the most available capacity (least-loaded-first).
5. Sends a `workflow.run` instruction to the selected node's daemon API.
6. Records the dispatch in `fleet_events`.

If no eligible node has available capacity, the dispatch waits in the fleet queue.

```
Task Ready
    │
    ▼
Filter: online, not draining, tag match
    │
    ▼
Sort by: available capacity (desc)
    │
    ▼
Node selected?
  ├── Yes → dispatch → record in fleet_events
  └── No  → enqueue in fleet queue (retry on next scheduler tick)
```

### Pool Enforcement

Each node enforces its own pool limit locally. The coordinator's dispatch decision is based on the last-synced capacity snapshot, which means there is a window (up to `sync_interval`) during which the coordinator may over-dispatch to a node. Nodes refuse dispatches that would exceed their pool limit, and the coordinator re-queues the rejected dispatch.

This is a deliberate trade-off: the coordinator does not acquire a distributed lock per dispatch, keeping the hot path simple and fast.

### Queue Priority

Dispatch priority is determined by task priority (`critical` > `high` > `normal` > `low`). Within the same priority tier, FIFO order is preserved. Manual reordering via `animus fleet queue hold` / `animus fleet queue release` applies at the fleet queue level before the node-level queue.

---

## Agent Lifecycle on a Node

```
Dispatch Instruction Received
          │
          ▼
    Node Queue (local)
          │
          ▼
  Capacity Available?
    ├── Yes → spawn agent-runner process
    └── No  → wait in local queue
          │
          ▼
   Agent Run (isolated worktree)
    ├── Phase 1
    ├── Phase 2 (checkpoint written after each phase)
    └── Phase N
          │
          ▼
   Completion / Failure
          │
          ▼
   Result written to node store
   Event emitted → picked up by coordinator on next sync
```

Checkpoints are stored in the node's local store. If the coordinator requests a migration (`animus fleet agent migrate`), the checkpoint is read, serialised, and re-dispatched to the destination node. The destination node resumes execution from the checkpoint.

---

## Failure Handling

### Node Loss

When a node becomes unreachable:

1. The coordinator marks it `unreachable` after three missed heartbeats.
2. If `evict_on_node_loss = true` (default: `false`), the coordinator re-queues all dispatches that were assigned to the lost node.
3. If `evict_on_node_loss = false`, those dispatches remain in a `node_lost` state until the node comes back online or the operator manually evicts them with `animus fleet agent evict`.

### Node Recovery

When a previously unreachable node responds to a heartbeat:

1. The coordinator marks it `online`.
2. A sync is triggered immediately to reconcile its state.
3. If the node was draining before it went offline, it resumes in drain mode.
4. Normal dispatch eligibility is restored after a successful sync (unless the node is still draining).

### Coordinator Restart

The coordinator is stateless in memory — all durable state lives in `fleet.db`. On restart it reads the registry and health table, re-establishes heartbeat timers for all registered nodes, and resumes from the last sync cursor.

In-flight dispatches that were sent before the coordinator restarted are not lost; they are running on their respective nodes and will be picked up on the next sync.

---

## Security Model

- **Local deployments:** Communication between the coordinator and nodes on the same machine is over the loopback interface; no authentication is required.
- **Remote nodes:** Configure `[tls]` in `fleet.toml` to enable mTLS. Each node presents a certificate; the coordinator validates it against a trusted CA.
- **Fleet store:** `fleet.db` is readable only by the user that owns the Animus process. It does not contain task content, only fleet topology and event history.

---

## Configuration Reference

Fleet configuration lives in `.ao/fleet.toml`. Key fields:

```toml
[fleet]
name = "my-fleet"
pool_size = 4            # default per-node pool size
max_pool_size = 8        # hard ceiling per node
sync_interval = "30s"
heartbeat_interval = "10s"
evict_on_node_loss = false
auto_drain_on_error = false

[tls]
enabled = false
# ca_cert = "/path/to/ca.pem"
# client_cert = "/path/to/client.pem"
# client_key = "/path/to/client.key"
```

Nodes inherit `pool_size` unless overridden at registration time (`animus fleet node register --pool-size`).

---

## Crate Responsibilities

Fleet functionality spans several crates in the workspace:

| Crate | Role |
|---|---|
| `orchestrator-daemon-runtime` | Hosts the fleet coordinator and scheduler; owns `fleet.db` |
| `orchestrator-core` | Provides domain types for nodes, agents, and events |
| `protocol` | Wire types for fleet API requests and responses |
| `orchestrator-cli` | Implements all `animus fleet` CLI commands |

There is no dedicated fleet crate — the coordinator is a subsystem within the existing daemon runtime, activated by the presence of `fleet.toml`.

---

## Related

- [Fleet Management Guide](../guides/fleet-management.md) — Getting started and operational patterns
- [ao fleet CLI Reference](../reference/cli/fleet.md) — All 51 commands with flags and examples
- [Subject Dispatch Daemon](subject-dispatch-daemon.md) — Single-node dispatch architecture
- [ServiceHub Pattern](service-hub.md) — Dependency injection used by the coordinator
- [Crate Map](crate-map.md) — Full crate dependency overview
