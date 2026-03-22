# Fleet Management Guide

Managing multiple AI agents working in parallel is a core capability of AO. This guide covers running parallel agents via the daemon, configuring concurrency limits, monitoring fleet health, and coordinating multi-agent work.

## Overview

AO's daemon manages a fleet of AI agents that can work on multiple tasks simultaneously. Each agent runs in an isolated worktree, executes workflow phases, and reports progress back to the orchestrator.

Key concepts:

| Term | Description |
|------|-------------|
| **Fleet** | The collection of agents currently managed by the daemon |
| **Pool Size** | Maximum number of concurrent agents/workflows |
| **Capacity** | Available slots in the pool for new work |
| **Worktree** | Isolated Git checkout where an agent makes changes |

---

## Running Parallel Agents

### Starting the Daemon for Parallel Work

Start the daemon with concurrency configured:

```bash
# Start with default pool size (4)
ao daemon start --autonomous

# Start with custom pool size
ao daemon start --autonomous --pool-size 8
```

The daemon will automatically pick up ready tasks up to the pool size limit.

### Configuring Pool Size

Pool size controls how many agents can run simultaneously. Set it at startup or update dynamically:

```bash
# Set at startup
ao daemon start --autonomous --pool-size 6

# Update while running
ao daemon config --set pool_size=6
```

Choose pool size based on:

| Factor | Recommendation |
|--------|----------------|
| **CPU cores** | 1-2 agents per core |
| **Memory** | ~2-4GB per agent |
| **API rate limits** | Stay under provider limits |
| **Task independence** | Higher for isolated tasks |

### Monitoring Active Agents

View currently running agents:

```bash
ao daemon agents
```

Output shows:
- Agent run IDs
- Associated task IDs
- Current phase
- Model/tool being used
- Runtime duration

Example output:

```
┌──────────────────────────────────────┬──────────┬───────────────┬─────────────┬──────────┐
│ Run ID                               │ Task     │ Phase         │ Model       │ Duration │
├──────────────────────────────────────┼──────────┼───────────────┼─────────────┼──────────┤
│ abc12345-1234-5678-9abc-def012345678 │ TASK-003 │ implementation│ claude-4    │ 12m 34s  │
│ def67890-2345-6789-0def-123456789abc │ TASK-005 │ unit-test     │ gemini-2.5  │ 5m 12s   │
└──────────────────────────────────────┴──────────┴───────────────┴─────────────┴──────────┘

Pool: 2/4 agents active (50% utilization)
```

---

## Monitoring Fleet Health

### Daemon Health Check

Get comprehensive health metrics:

```bash
ao daemon health
```

Returns:
- Process status and uptime
- Memory usage
- Active agent count
- Pool capacity
- Queue depth
- Runner health status

### Via MCP Tools

For programmatic access, use the MCP tools:

```json
// ao.daemon.health — detailed metrics
{}

// ao.daemon.agents — list active agents
{}

// ao.daemon.status — basic running state
{}
```

Example health response:

```json
{
  "running": true,
  "paused": false,
  "uptime_secs": 3600,
  "active_agents": 3,
  "pool_size": 4,
  "available_capacity": 1,
  "queue_depth": 5,
  "runner_healthy": true
}
```

### Capacity Planning

Monitor capacity to ensure smooth operations:

```bash
# Check current capacity
ao daemon health | grep capacity

# View queue depth
ao queue stats

# List queued dispatches
ao queue list
```

Capacity indicators:

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Pool utilization | 25-75% | 75-90% | >90% or 0% |
| Queue depth | < pool_size | pool_size to 2x | > 2x pool_size |
| Available capacity | > 0 | 1 | 0 |

### Real-Time Monitoring

Stream daemon events to watch fleet activity:

```bash
ao daemon events
```

Or tail logs for detailed activity:

```bash
tail -f .ao/daemon.log
```

---

## Scaling Strategies

### Vertical Scaling (Increase Pool Size)

Add more capacity to the existing daemon:

```bash
# Increase pool size
ao daemon config --set pool_size=8

# Verify change
ao daemon config
```

Best for:
- Temporary workload spikes
- Single-machine deployments
- Quick capacity adjustments

### Horizontal Scaling (Multiple Daemons)

For large workloads, run multiple daemon instances across different machines:

**Setup:**

1. Clone the repository on each machine
2. Configure each daemon with its own scope:

```bash
# Machine 1 — handles frontend tasks
ao daemon start --autonomous --runner-scope frontend --pool-size 4

# Machine 2 — handles backend tasks  
ao daemon start --autonomous --runner-scope backend --pool-size 4
```

3. Use task assignment to route work:

```bash
ao task assign --id TASK-001 --type agent --agent-role frontend
```

### Scaling by Task Type

Allocate pool capacity by task characteristics:

| Task Type | Pool Allocation | Rationale |
|-----------|-----------------|-----------|
| Quick fixes | 1-2 slots | Fast turnaround, low resource |
| Features | 2-4 slots | Medium duration, standard resources |
| Refactoring | 1-2 slots | Long duration, careful changes |
| Research | 2-3 slots | Can run parallel, no writes |

### Auto-Scaling Patterns

Implement dynamic scaling based on queue depth:

```bash
#!/bin/bash
# scale-fleet.sh

QUEUE_DEPTH=$(ao queue stats --json | jq '.pending')
CURRENT_POOL=$(ao daemon config --json | jq '.pool_size')

if [ "$QUEUE_DEPTH" -gt 10 ] && [ "$CURRENT_POOL" -lt 8 ]; then
  echo "Scaling up: queue=$QUEUE_DEPTH, pool=$((CURRENT_POOL + 2))"
  ao daemon config --set pool_size=$((CURRENT_POOL + 2))
elif [ "$QUEUE_DEPTH" -lt 3 ] && [ "$CURRENT_POOL" -gt 4 ]; then
  echo "Scaling down: queue=$QUEUE_DEPTH, pool=$((CURRENT_POOL - 1))"
  ao daemon config --set pool_size=$((CURRENT_POOL - 1))
fi
```

---

## Coordinating Multi-Agent Work

### Task Dependencies

Use dependencies to coordinate work order:

```bash
# TASK-002 depends on TASK-001
ao task dependency-add --id TASK-002 --depends-on TASK-001
```

The daemon respects dependencies when dispatching — TASK-002 won't start until TASK-001 completes.

### Queue Management

Control dispatch order with the queue:

```bash
# View current queue
ao queue list

# Reorder priorities
ao queue reorder --subject-ids TASK-005 TASK-003 TASK-001

# Hold a task temporarily
ao queue hold --subject-id TASK-003

# Release when ready
ao queue release --subject-id TASK-003
```

### Batch Dispatch

Dispatch multiple workflows at once:

```bash
# Via CLI
ao workflow run --task-id TASK-001 &
ao workflow run --task-id TASK-002 &
ao workflow run --task-id TASK-003 &

# Or use batch dispatch (MCP)
```

Via MCP:

```json
// ao.workflow.run-multiple
{
  "runs": [
    { "task_id": "TASK-001" },
    { "task_id": "TASK-002", "workflow_ref": "quick" },
    { "task_id": "TASK-003" }
  ],
  "on_error": "continue"
}
```

### Model Diversity

Run different models for different task types:

```yaml
# In workflows.yaml
agents:
  code-agent:
    model: claude-sonnet-4-6
    tool: claude
  
  research-agent:
    model: gemini-2.5-flash
    tool: gemini
  
  review-agent:
    model: claude-opus-4-6
    tool: claude
```

Assign tasks to specific agent profiles:

```bash
ao task assign --id TASK-001 --type agent --model claude-opus-4-6
```

### Worktree Isolation

Each agent works in an isolated worktree. View active worktrees:

```bash
ao worktree list
```

Worktrees prevent conflicts when agents modify the same files:

```
.ao/worktrees/
├── task-task-001-abc123/    # Agent working on TASK-001
├── task-task-002-def456/    # Agent working on TASK-002
└── task-task-003-ghi789/    # Agent working on TASK-003
```

---

## Workload Distribution Patterns

### Feature Factory Pattern

High-throughput feature development:

```bash
# Configure for parallel feature work
ao daemon config --set pool_size=6
ao daemon config --set auto_pr=true
ao daemon config --set auto_merge=true

# Create multiple ready tasks
ao task create --title "Feature A" --type feature --priority high
ao task create --title "Feature B" --type feature --priority high
ao task create --title "Feature C" --type feature --priority high

# Mark all ready
ao task bulk-status --updates '[
  {"id": "TASK-001", "status": "ready"},
  {"id": "TASK-002", "status": "ready"},
  {"id": "TASK-003", "status": "ready"}
]'

# Daemon dispatches up to 6 in parallel
```

### Pipeline Pattern

Sequential phases across parallel tasks:

```
Task Queue:  [TASK-001] [TASK-002] [TASK-003]
                 ↓          ↓          ↓
Phase 1:     research   research   research    (3 agents parallel)
                 ↓          ↓          ↓
Phase 2:     implement  implement  implement   (3 agents parallel)
                 ↓          ↓          ↓
Phase 3:     review     review     review      (3 agents parallel)
```

Each phase runs in parallel across tasks, but phases within a task are sequential.

### Specialized Fleet Pattern

Assign specific agent types to task categories:

```bash
# Create specialized tasks
ao task create --title "API endpoint" --type feature --tag api
ao task create --title "UI component" --type feature --tag frontend
ao task create --title "Database migration" --type feature --tag database

# Assign to specialized models
ao task assign --id TASK-001 --type agent --model claude-sonnet-4-6  # API work
ao task assign --id TASK-002 --type agent --model gemini-3.1-pro    # UI work
ao task assign --id TASK-003 --type agent --model claude-opus-4-6   # DB work
```

---

## Health Monitoring Dashboard

Create a simple monitoring script:

```bash
#!/bin/bash
# fleet-status.sh — Fleet health dashboard

echo "╔══════════════════════════════════════════════════════════════╗"
echo "║                    AO FLEET STATUS                           ║"
echo "╚══════════════════════════════════════════════════════════════╝"

# Daemon status
echo -e "\n📊 Daemon Status:"
ao daemon status

# Health metrics
echo -e "\n🏥 Health Metrics:"
ao daemon health

# Active agents
echo -e "\n🤖 Active Agents:"
ao daemon agents

# Queue stats
echo -e "\n📋 Queue Statistics:"
ao queue stats

# Task distribution
echo -e "\n📈 Task Distribution:"
ao task stats

# Runner health
echo -e "\n⚙️  Runner Status:"
ao runner health

echo -e "\n══════════════════════════════════════════════════════════════"
```

Run periodically:

```bash
watch -n 10 ./fleet-status.sh
```

---

## Troubleshooting Fleet Issues

### All Agents Stuck

**Symptoms:** No progress on any task, agents showing long durations

**Diagnosis:**

```bash
# Check daemon health
ao daemon health

# Check runner
ao runner health

# Look for errors in logs
ao daemon logs --search "error"
```

**Fixes:**

```bash
# Check for orphans
ao runner orphans detect
ao runner orphans cleanup

# Restart daemon
ao daemon stop
ao daemon start --autonomous
```

### Pool Exhaustion

**Symptoms:** Queue backing up, no available capacity

**Diagnosis:**

```bash
# Check queue depth vs pool size
ao queue stats
ao daemon config | grep pool_size

# See what's running
ao daemon agents
```

**Fixes:**

```bash
# Increase pool size
ao daemon config --set pool_size=8

# Cancel stuck workflows
ao workflow list --status running
ao workflow cancel --id WF-XXX --confirmation yes
```

### Agent Conflicts

**Symptoms:** Merge conflicts, file collisions between agents

**Diagnosis:**

```bash
# Check worktrees
ao worktree list

# See which agents touch which files
ao daemon agents --verbose
```

**Fixes:**

```bash
# Use task dependencies to sequence work
ao task dependency-add --id TASK-002 --depends-on TASK-001

# Reduce pool size for high-conflict work
ao daemon config --set pool_size=2
```

### Uneven Distribution

**Symptoms:** Some tasks stuck while others complete quickly

**Diagnosis:**

```bash
# Check task priorities
ao task prioritized

# Check queue ordering
ao queue list
```

**Fixes:**

```bash
# Reorder queue
ao queue reorder --subject-ids TASK-005 TASK-001 TASK-003

# Adjust priorities
ao task set-priority --id TASK-005 --priority critical
```

---

## Best Practices

### Right-Sizing Your Fleet

| Scenario | Pool Size | Rationale |
|----------|-----------|-----------|
| Development machine | 2-4 | Preserve local resources |
| CI/CD pipeline | 4-8 | Maximize throughput |
| Dedicated server | 8-16 | Full utilization |
| Large codebase | 4-6 | Balance speed vs conflicts |

### Resource Management

1. **Monitor memory:** Each agent uses 2-4GB
2. **Watch API limits:** Stay under rate limits
3. **Balance I/O:** Multiple agents read/write simultaneously
4. **Clean up worktrees:** Prune after merges

```bash
# Clean up completed worktrees
ao worktree prune
```

### Operational Guidelines

1. **Start small:** Begin with pool_size=2, scale up as needed
2. **Monitor first:** Watch `ao daemon events` before increasing load
3. **Use dependencies:** Sequence related work to avoid conflicts
4. **Regular cleanup:** Clear logs and prune worktrees weekly

---

## MCP Tools Reference

For programmatic fleet management:

### Fleet Monitoring

```json
// ao.daemon.health — comprehensive metrics
{}

// ao.daemon.agents — list active agents
{}

// ao.daemon.status — basic state
{}

// ao.daemon.events — recent activity
{ "limit": 50 }
```

### Configuration

```json
// ao.daemon.config — read settings
{}

// ao.daemon.config-set — update settings
{
  "pool_size": 8,
  "interval_secs": 5,
  "phase_timeout_secs": 3600
}
```

### Queue Control

```json
// ao.queue.list — view queue
{}

// ao.queue.stats — aggregate counts
{}

// ao.queue.reorder — set dispatch order
{ "subject_ids": ["TASK-003", "TASK-001", "TASK-002"] }

// ao.queue.hold — pause dispatch
{ "subject_id": "TASK-001" }

// ao.queue.release — resume dispatch
{ "subject_id": "TASK-001" }
```

### Batch Operations

```json
// ao.task.bulk-status — update multiple tasks
{
  "updates": [
    { "id": "TASK-001", "status": "ready" },
    { "id": "TASK-002", "status": "ready" }
  ]
}

// ao.workflow.run-multiple — dispatch multiple workflows
{
  "runs": [
    { "task_id": "TASK-001" },
    { "task_id": "TASK-002" }
  ]
}
```

---

## Related Documentation

- **[Daemon Operations](daemon-operations.md)** — Comprehensive daemon management
- **[Task Management](task-management.md)** — Creating and managing tasks
- **[Writing Workflows](writing-workflows.md)** — Defining workflow phases
- **[Model Routing](model-routing.md)** — How models are selected per phase
- **[CLI Cheat Sheet](../tutorials/cli-cheat-sheet.md)** — Quick command reference
- **[Daemon Quick Reference](../tutorials/daemon-quick-ref.md)** — Essential daemon commands
