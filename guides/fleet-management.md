# Fleet Management

`ao fleet` is the multi-node coordination layer for AO. It extends the single-daemon model to a distributed set of machines, letting you register nodes, assign agents, scale capacity, and observe the entire fleet through a single CLI.

This guide walks through setup, everyday operations, and common operational patterns.

---

## Prerequisites

- AO installed on each machine that will act as a fleet node (see [Installation](../getting-started/installation.md))
- The AO daemon running on each node (`ao daemon start --autonomous`)
- Network connectivity between the machine running `ao fleet` commands and all node daemons

---

## Quick Start

### 1. Initialise the fleet

In your project directory, initialise fleet management:

```bash
ao fleet init
```

This creates `.ao/fleet.toml` and registers the local daemon as the first node (`primary`). To customise:

```bash
ao fleet init --pool-size 4 --node-name primary
```

### 2. Register additional nodes

For each remote machine running an AO daemon:

```bash
ao fleet node register --address 10.0.0.2:7700 --name worker-1
ao fleet node register --address 10.0.0.3:7700 --name worker-2 --tag backend
```

Verify all nodes are online:

```bash
ao fleet node list
```

### 3. Check fleet health

```bash
ao fleet health
```

A healthy fleet shows all nodes as `online` with available capacity.

### 4. View the status dashboard

```bash
ao fleet status
```

This shows the full picture: active agents, pool utilisation, queue depth, and recent errors across all nodes.

---

## Core Concepts

| Term | Description |
|------|-------------|
| **Fleet** | The set of registered nodes managed together |
| **Node** | A machine running an AO daemon, registered with the fleet |
| **Pool size** | Maximum concurrent agents on a single node |
| **Fleet capacity** | Sum of pool sizes across all online nodes |
| **Drain** | A node state where it finishes current work but accepts no new dispatches |
| **Schedule** | A cron or interval rule that automatically dispatches workflows |
| **Sync** | State reconciliation between the fleet coordinator and all nodes |

---

## Node Management

### Registering Nodes

Nodes are remote daemon instances. Register them by address:

```bash
# Register a plain worker
ao fleet node register --address 192.168.1.10:7700 --name worker-1

# Register with routing tags
ao fleet node register \
  --address 192.168.1.11:7700 \
  --name gpu-worker \
  --tag ml \
  --tag gpu \
  --pool-size 2
```

Tags let you route tasks to specific node types:

```bash
# Assign task to a node tagged "ml"
ao daemon config --set runner_scope=ml
```

### Removing Nodes

```bash
# Drain first (finish in-progress work), then remove
ao fleet node drain --id worker-1 --wait
ao fleet node remove --id worker-1 --confirmation yes
```

### Node Health and Connectivity

```bash
# Ping a node
ao fleet node ping --id worker-1

# Health metrics for one node
ao fleet node health --id worker-1

# All nodes at once
ao fleet health
```

---

## Pool Sizing

Pool size controls how many agents a node runs simultaneously. Right-sizing avoids both wasted capacity and resource contention.

### Setting Pool Sizes

```bash
# Set pool size for a single node
ao fleet pool set --node worker-1 --pool-size 6

# Set the fleet-wide default (applies to new nodes)
ao fleet pool set --pool-size 4
```

### Dynamic Scaling

Scale the entire fleet up or down in one command:

```bash
# Scale each node up by 2
ao fleet pool scale --delta +2

# Target a specific total fleet capacity
ao fleet pool scale --total-capacity 20
```

Check current utilisation before and after:

```bash
ao fleet pool stats
```

### Sizing Guidance

| Deployment | Pool Size per Node | Notes |
|---|---|---|
| Developer laptop | 2–4 | Preserve local resources |
| CI runner | 4–6 | Maximise throughput |
| Dedicated server | 6–12 | Full utilisation |
| GPU node (ML tasks) | 1–2 | Memory-bound; keep low |

---

## Running Parallel Agents

With multiple nodes registered and pool sizes configured, the fleet automatically distributes work. Tasks dispatched via `ao workflow run` or `ao daemon` are balanced across available nodes.

### Dispatching Work

```bash
# Daemon dispatches automatically as tasks become ready
ao daemon start --autonomous

# Manual dispatch to a specific workflow
ao workflow run --task-id TASK-001

# Check what's running across the fleet
ao fleet agent list
```

### Watching Active Agents

```bash
# All fleet agents
ao fleet agent list

# Agents on a specific node
ao fleet agent list --node worker-1

# Stream events as agents start and finish
ao fleet events stream
```

Example `ao fleet agent list` output:

```
┌──────────────────────┬──────────┬────────────────┬───────────┬──────────────┬──────────┐
│ Run ID               │ Task     │ Phase          │ Node      │ Model        │ Duration │
├──────────────────────┼──────────┼────────────────┼───────────┼──────────────┼──────────┤
│ abc12345             │ TASK-003 │ implementation │ primary   │ claude-sonnet│ 12m 34s  │
│ def67890             │ TASK-005 │ unit-test      │ worker-1  │ claude-haiku │ 5m 12s   │
│ ghi11223             │ TASK-007 │ research       │ worker-2  │ gemini-2.5   │ 3m 01s   │
└──────────────────────┴──────────┴────────────────┴───────────┴──────────────┴──────────┘
Fleet: 3/12 agents active (25%)
```

---

## Queue Management

The fleet queue controls dispatch order across all nodes.

```bash
# View the queue
ao fleet queue list

# Hold a task temporarily
ao fleet queue hold --subject-id TASK-010

# Release when ready
ao fleet queue release --subject-id TASK-010

# Check queue depth by node
ao fleet queue stats
```

To prioritise urgent work, combine with task priority:

```bash
ao task set-priority --id TASK-010 --priority critical
```

---

## Scheduling

Schedules dispatch workflows automatically on a cron or interval.

### Create a Nightly Schedule

```bash
ao fleet schedule create \
  --name nightly-tests \
  --cron "0 2 * * *" \
  --workflow-ref ci \
  --task-filter '{"tags": ["test"]}'
```

### Create an Interval Schedule

```bash
ao fleet schedule create \
  --name sync-check \
  --interval 30m \
  --workflow-ref health-check
```

### Manage Schedules

```bash
# List all schedules
ao fleet schedule list

# Disable a schedule
ao fleet schedule update --id sched-001 --enabled false

# Trigger manually (for testing)
ao fleet schedule trigger --id sched-001

# Remove a schedule
ao fleet schedule remove --id sched-001 --confirmation yes
```

---

## Maintenance Operations

### Draining a Node

Take a node out of rotation without dropping in-progress work:

```bash
# Start draining
ao fleet node drain --id worker-1

# Wait until all current agents finish
ao fleet node drain --id worker-1 --wait

# Verify no active agents remain
ao fleet agent list --node worker-1

# Take the machine offline, then resume later
ao fleet node resume --id worker-1
```

### Migrating Agents

Move a running agent to a different node (for example, before hardware maintenance):

```bash
# Find the run ID
ao fleet agent list --node worker-1

# Migrate to another node
ao fleet agent migrate --run-id abc12345 --to worker-2
```

### Updating Fleet Config

```bash
# View current config
ao fleet config get

# Update sync interval
ao fleet config set --sync-interval 15s

# Validate before applying a new config file
ao fleet config validate --file fleet-new.toml
ao fleet config import --file fleet-new.toml
```

---

## Observability

### Fleet Status Dashboard

```bash
ao fleet status
```

### Continuous Metrics

```bash
# Watch refreshing every 10 seconds
ao fleet metrics watch --interval 10
```

### Event History

```bash
# Recent events across the fleet
ao fleet events list --since 1h

# Stream live
ao fleet events stream

# Filter to one event type
ao fleet events list --type agent_failed
```

### Scripted Monitoring

```bash
#!/bin/bash
# fleet-health-check.sh

STATUS=$(ao fleet health --json | jq -r '.verdict')
if [ "$STATUS" != "healthy" ]; then
  echo "Fleet degraded: $STATUS"
  ao fleet health --json | jq '.nodes[] | select(.status != "healthy")'
  exit 1
fi
echo "Fleet healthy"
```

---

## Common Patterns

### Feature Factory

High-throughput parallel feature development across a fleet of nodes:

```bash
# 1. Configure fleet
ao fleet init --pool-size 4
ao fleet node register --address 10.0.0.2:7700 --name worker-1
ao fleet node register --address 10.0.0.3:7700 --name worker-2

# 2. Create and queue tasks
ao task create --title "Feature A" --priority high
ao task create --title "Feature B" --priority high
ao task create --title "Feature C" --priority high

# 3. Start daemon — dispatches up to 12 agents in parallel (3 nodes × pool 4)
ao daemon start --autonomous
```

### Staged Rollout

Use node tags to route tasks to specific environments:

```bash
# Tag nodes by environment
ao fleet node tag --id worker-1 --tag staging
ao fleet node tag --id worker-2 --tag production

# Tasks are routed by runner scope
ao daemon config --set runner_scope=staging
```

### Drain-and-Scale

Scale down during off-hours, scale back up for business hours:

```bash
# Evening: drain extra nodes
ao fleet node drain --id worker-1 --wait
ao fleet node drain --id worker-2 --wait

# Morning: resume
ao fleet node resume --id worker-1
ao fleet node resume --id worker-2
ao fleet pool scale --total-capacity 16
```

---

## Troubleshooting

### Node Unreachable

**Symptoms:** `ao fleet health` shows a node as `unreachable`.

```bash
# Test connectivity
ao fleet node ping --id worker-1

# Inspect last-seen timestamp
ao fleet node get --id worker-1 --json | jq '.last_seen'

# Check daemon on the remote machine
ssh worker-1 ao daemon status
```

**Fix:** Restart the daemon on the remote machine, then sync:

```bash
# On the remote machine
ao daemon start --autonomous

# From the fleet coordinator
ao fleet sync trigger --node worker-1
```

### Pool Exhaustion

**Symptoms:** Queue depth growing, `ao fleet pool stats` shows 100% utilisation on all nodes.

```bash
# Check what's running
ao fleet agent list

# Scale up if capacity is available
ao fleet pool scale --delta +2

# Or cancel stuck workflows
ao workflow list --status running
ao workflow cancel --id WF-XXX --confirmation yes
```

### Uneven Distribution

**Symptoms:** One node is overloaded while others are idle.

```bash
# Check per-node queue depth
ao fleet queue stats

# Evict agents from the overloaded node to re-balance
ao fleet agent evict --run-id <heavy-run-id>

# Verify redistribution
ao fleet pool stats
```

### Sync Lag

**Symptoms:** `ao fleet sync status` shows nodes are behind.

```bash
# Force sync
ao fleet sync trigger

# If a node is persistently behind, restart its daemon
ao fleet node drain --id lagging-node --wait
# (restart daemon on the remote machine)
ao fleet sync trigger --node lagging-node
```

---

## Related Documentation

- **[ao fleet CLI Reference](../reference/cli/fleet.md)** — Complete reference for all 51 commands with flags and examples
- **[Fleet Architecture](../architecture/fleet.md)** — How fleet coordination works internally
- **[Daemon Operations](daemon-operations.md)** — Single-daemon lifecycle and configuration
- **[Task Management](task-management.md)** — Creating and prioritising tasks
- **[Writing Workflows](writing-workflows.md)** — Defining workflow phases and agents
- **[Model Routing](model-routing.md)** — Per-phase model selection
