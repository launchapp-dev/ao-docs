# Daemon Operations Quick Reference

Essential commands for starting, stopping, monitoring, and troubleshooting the AO daemon. Keep this reference handy for daily operations.

## Quick Reference

```bash
ao daemon start --autonomous    # Start background daemon
ao daemon status                # Check if running
ao daemon health                # Detailed health
ao daemon pause                 # Pause scheduling
ao daemon resume                # Resume scheduling
ao daemon stop                  # Stop daemon
```

---

## Starting the Daemon

### Background Mode (Production)

Start as a detached background process:

```bash
ao daemon start --autonomous
```

What happens:
- Forks into background
- Writes PID to `.ao/daemon.pid`
- Logs to `.ao/daemon.log`
- Returns immediately

### With Options

```bash
# Autonomous mode (runs tasks automatically)
ao daemon start --autonomous

# With custom interval (seconds)
ao daemon start --autonomous --interval-secs 10

# With concurrency limit
ao daemon start --autonomous --pool-size 4
```

### Foreground Mode (Debugging)

Run in foreground for immediate visibility:

```bash
ao daemon run
```

Use this when:
- Debugging startup issues
- Watching real-time activity
- Developing workflow configurations

Press `Ctrl+C` to stop.

---

## Checking Daemon Status

### Basic Status

```bash
ao daemon status
```

Output includes:
- Running: yes/no
- PID (if running)
- Uptime
- Current state (running/paused)

### Detailed Health

```bash
ao daemon health
```

Shows:
- Process health
- Memory usage
- Active agents count
- Queue depth
- Capacity metrics

### List Active Agents

```bash
ao daemon agents
```

Shows agents currently managed by this daemon.

---

## Controlling the Daemon

### Pause Scheduling

Temporarily stop picking up new tasks:

```bash
ao daemon pause
```

Effects:
- In-progress work continues
- No new tasks dispatched
- Daemon process stays running

Use when:
- Making manual changes
- Debugging a specific workflow
- Temporary maintenance

### Resume Scheduling

```bash
ao daemon resume
```

Returns to normal operation.

### Stop the Daemon

Graceful shutdown:

```bash
ao daemon stop
```

What happens:
- Stops accepting new work
- Waits for in-progress phases (with timeout)
- Cleans up resources
- Removes PID file

### Force Stop

If graceful stop hangs:

```bash
# Find the PID
ao daemon status

# Kill manually (last resort)
kill -9 <PID>

# Clean up
rm .ao/daemon.pid
```

---

## Monitoring

### View Logs

```bash
# Recent logs
ao daemon logs

# With limit
ao daemon logs --limit 100

# Search for errors
ao daemon logs --search "error"

# Tail in real-time
tail -f .ao/daemon.log
```

### Stream Events

Watch daemon activity in real-time:

```bash
ao daemon events
```

Shows:
- Workflow dispatches
- Phase completions
- Verdicts
- State changes

With limit:

```bash
ao daemon events --limit 50
```

### Clear Logs

When logs grow too large:

```bash
ao daemon clear-logs
```

Rotation happens automatically at 10MB.

---

## Configuration

### View Configuration

```bash
ao daemon config
```

Shows all daemon settings.

### Key Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `auto_merge` | Auto-merge PRs after workflow | false |
| `auto_pr` | Auto-create PRs for work | false |
| `pool_size` | Max concurrent workflows | 4 |
| `interval_secs` | Seconds between ticks | 5 |
| `auto_run_ready` | Auto-dispatch ready tasks | true |
| `stale_threshold_hours` | Hours before task is stale | 24 |
| `phase_timeout_secs` | Timeout per phase | 3600 |
| `idle_timeout_secs` | Idle shutdown (0 = never) | 0 |

### Update Configuration

```bash
# Set specific values
ao daemon config --set pool_size=8
ao daemon config --set auto_merge=true
ao daemon config --set interval_secs=10
```

Or use the MCP tool:

```bash
# Via daemon config-set (if available)
ao daemon config-set --pool-size 8 --auto-merge true
```

---

## Runner Management

The runner is a separate process that spawns agent CLIs.

### Check Runner Health

```bash
ao runner health
```

Shows:
- Runner status
- Active processes
- Capacity

### Detect Orphaned Processes

Sometimes processes get orphaned:

```bash
ao runner orphans detect
```

### Clean Up Orphans

```bash
ao runner orphans cleanup
```

### Restart Statistics

View runner restart history:

```bash
ao runner restart-stats
```

High restart counts may indicate configuration issues.

---

## Common Operations

### Daily Startup

```bash
# Check if already running
ao daemon status

# Start if not
ao daemon start --autonomous

# Verify
ao daemon health
```

### Pause for Manual Work

```bash
# Pause daemon
ao daemon pause

# Do your work...
vim src/important.rs

# Resume
ao daemon resume
```

### End of Day

```bash
# Check what's running
ao workflow list --status running

# If tasks in progress, let them finish or pause
ao daemon pause

# Or stop entirely
ao daemon stop
```

### Debug Session

```bash
# Stop background daemon
ao daemon stop

# Run in foreground
ao daemon run

# Watch output, identify issues
# Press Ctrl+C when done

# Restart background
ao daemon start --autonomous
```

---

## Troubleshooting

### Daemon Won't Start

```bash
# Check for existing instance
ao daemon status

# Check logs
ao daemon logs

# Try foreground mode for errors
ao daemon run

# Check environment
ao doctor
```

Common causes:
- Already running
- Invalid configuration
- Missing API keys
- Port conflicts

### Tasks Not Being Picked Up

```bash
# Check daemon running
ao daemon status

# Check not paused
ao daemon status | grep paused

# Check queue
ao queue list
ao queue stats

# Check task status
ao task list --status ready

# Check capacity
ao daemon health
```

### High Memory Usage

```bash
# Check health
ao daemon health

# Clear old logs
ao daemon clear-logs

# Restart daemon
ao daemon stop
ao daemon start --autonomous
```

### Runner Issues

```bash
# Check runner
ao runner health

# Look for orphans
ao runner orphans detect

# Clean up
ao runner orphans cleanup

# Check restart stats
ao runner restart-stats
```

### Workflow Stuck

```bash
# Check workflow state
ao workflow list --status running

# Get details
ao workflow get --id WF-XXX

# Check if agent is running
ao daemon agents

# Check logs
ao daemon logs | grep WF-XXX

# May need to cancel
ao workflow cancel --id WF-XXX --confirmation yes
```

---

## Log Analysis

### Find Errors

```bash
# Recent errors
ao daemon logs | grep -i error

# Count errors today
grep "$(date +%Y-%m-%d)" .ao/daemon.log | grep -c error
```

### Track Workflow Dispatches

```bash
# All dispatches
grep "workflow_dispatched" .ao/daemon.log

# Today's dispatches
grep "workflow_dispatched" .ao/daemon.log | grep "$(date +%Y-%m-%d)"
```

### Phase Completions

```bash
# Completed phases
grep "phase_complete" .ao/daemon.log | tail -20
```

### Daemon Restarts

```bash
# Startup events
grep "daemon_startup" .ao/daemon.log
```

---

## Health Checks

### Quick Health

```bash
ao daemon health
```

### Full Diagnostics

```bash
# Environment
ao doctor

# Daemon status
ao daemon status

# Runner health
ao runner health

# Model availability
ao model status

# Recent errors
ao errors list --limit 10
```

### Automated Health Check Script

```bash
#!/bin/bash
# health-check.sh

echo "=== Daemon Status ==="
ao daemon status

echo -e "\n=== Runner Health ==="
ao runner health

echo -e "\n=== Queue Stats ==="
ao queue stats

echo -e "\n=== Task Stats ==="
ao task stats

echo -e "\n=== Recent Errors ==="
ao errors list --limit 5

echo -e "\n=== Model Status ==="
ao model status
```

---

## Best Practices

### Starting the Daemon

1. Check if already running: `ao daemon status`
2. Start with appropriate options: `ao daemon start --autonomous`
3. Verify startup: `ao daemon health`

### Daily Operations

1. Morning: Check status and health
2. Throughout day: Monitor with `ao daemon events`
3. End of day: Review task progress, pause or stop as needed

### Before Maintenance

1. Pause daemon: `ao daemon pause`
2. Wait for in-progress work: `ao workflow list --status running`
3. Make changes
4. Resume: `ao daemon resume`

### Regular Maintenance

1. Weekly: Clear old logs
2. As needed: Check for orphans
3. Periodically: Review restart statistics

---

## Environment Variables

### Daemon Behavior

| Variable | Effect |
|----------|--------|
| `AO_DAEMON_INTERVAL` | Tick interval (seconds) |
| `AO_DAEMON_POOL_SIZE` | Max concurrent workflows |
| `AO_ALLOW_NON_EDITING_PHASE_TOOL` | Allow non-write tools for phases |

### API Keys

Required for agent execution:

| Variable | For Tool |
|----------|----------|
| `ANTHROPIC_API_KEY` | claude |
| `OPENAI_API_KEY` | codex, oai-runner |
| `GEMINI_API_KEY` | gemini |

---

## Process Management

### PID File

Location: `.ao/daemon.pid`

```bash
# Read PID
cat .ao/daemon.pid

# Check if process exists
ps -p $(cat .ao/daemon.pid)
```

### Log Files

| File | Purpose |
|------|---------|
| `.ao/daemon.log` | Current log |
| `.ao/daemon.log.1` | Rotated log (at 10MB) |

---

## Related Documentation

- [CLI Cheat Sheet](cli-cheat-sheet.md) -- Quick command reference
- [Daemon Operations Guide](../guides/daemon-operations.md) -- Comprehensive daemon docs
- [Troubleshooting Guide](../guides/troubleshooting.md) -- Common issues and fixes
- [Workflow Execution](workflow-execution.md) -- Running and monitoring workflows
