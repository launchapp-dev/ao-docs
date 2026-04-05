# Daemon Operations Quick Reference

Essential commands for starting, stopping, monitoring, and troubleshooting the Animus daemon. Keep this reference handy for daily operations.

## Quick Reference

```bash
animus daemon start --autonomous    # Start background daemon
animus daemon status                # Check if running
animus daemon health                # Detailed health
animus daemon pause                 # Pause scheduling
animus daemon resume                # Resume scheduling
animus daemon stop                  # Stop daemon
```

---

## Starting the Daemon

### Background Mode (Production)

Start as a detached background process:

```bash
animus daemon start --autonomous
```

What happens:
- Forks into background
- Writes PID to `.ao/daemon.pid`
- Logs to `.ao/daemon.log`
- Returns immediately

### With Options

```bash
# Autonomous mode (runs tasks automatically)
animus daemon start --autonomous

# With custom interval (seconds)
animus daemon start --autonomous --interval-secs 10

# With concurrency limit
animus daemon start --autonomous --pool-size 4
```

### Foreground Mode (Debugging)

Run in foreground for immediate visibility:

```bash
animus daemon run
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
animus daemon status
```

Output includes:
- Running: yes/no
- PID (if running)
- Uptime
- Current state (running/paused)

### Detailed Health

```bash
animus daemon health
```

Shows:
- Process health
- Memory usage
- Active agents count
- Queue depth
- Capacity metrics

### List Active Agents

```bash
animus daemon agents
```

Shows agents currently managed by this daemon.

---

## Controlling the Daemon

### Pause Scheduling

Temporarily stop picking up new tasks:

```bash
animus daemon pause
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
animus daemon resume
```

Returns to normal operation.

### Stop the Daemon

Graceful shutdown:

```bash
animus daemon stop
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
animus daemon status

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
animus daemon logs

# With limit
animus daemon logs --limit 100

# Search for errors
animus daemon logs --search "error"

# Tail in real-time
tail -f .ao/daemon.log
```

### Stream Events

Watch daemon activity in real-time:

```bash
animus daemon events
```

Shows:
- Workflow dispatches
- Phase completions
- Verdicts
- State changes

With limit:

```bash
animus daemon events --limit 50
```

### Clear Logs

When logs grow too large:

```bash
animus daemon clear-logs
```

Rotation happens automatically at 10MB.

---

## Configuration

### View Configuration

```bash
animus daemon config
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
animus daemon config --set pool_size=8
animus daemon config --set auto_merge=true
animus daemon config --set interval_secs=10
```

Or use the MCP tool:

```bash
# Via daemon config-set (if available)
animus daemon config-set --pool-size 8 --auto-merge true
```

---

## Runner Management

The runner is a separate process that spawns agent CLIs.

### Check Runner Health

```bash
animus runner health
```

Shows:
- Runner status
- Active processes
- Capacity

### Detect Orphaned Processes

Sometimes processes get orphaned:

```bash
animus runner orphans detect
```

### Clean Up Orphans

```bash
animus runner orphans cleanup
```

### Restart Statistics

View runner restart history:

```bash
animus runner restart-stats
```

High restart counts may indicate configuration issues.

---

## Common Operations

### Daily Startup

```bash
# Check if already running
animus daemon status

# Start if not
animus daemon start --autonomous

# Verify
animus daemon health
```

### Pause for Manual Work

```bash
# Pause daemon
animus daemon pause

# Do your work...
vim src/important.rs

# Resume
animus daemon resume
```

### End of Day

```bash
# Check what's running
animus workflow list --status running

# If tasks in progress, let them finish or pause
animus daemon pause

# Or stop entirely
animus daemon stop
```

### Debug Session

```bash
# Stop background daemon
animus daemon stop

# Run in foreground
animus daemon run

# Watch output, identify issues
# Press Ctrl+C when done

# Restart background
animus daemon start --autonomous
```

---

## Troubleshooting

### Daemon Won't Start

```bash
# Check for existing instance
animus daemon status

# Check logs
animus daemon logs

# Try foreground mode for errors
animus daemon run

# Check environment
animus doctor
```

Common causes:
- Already running
- Invalid configuration
- Missing API keys
- Port conflicts

### Tasks Not Being Picked Up

```bash
# Check daemon running
animus daemon status

# Check not paused
animus daemon status | grep paused

# Check queue
animus queue list
animus queue stats

# Check task status
animus task list --status ready

# Check capacity
animus daemon health
```

### High Memory Usage

```bash
# Check health
animus daemon health

# Clear old logs
animus daemon clear-logs

# Restart daemon
animus daemon stop
animus daemon start --autonomous
```

### Runner Issues

```bash
# Check runner
animus runner health

# Look for orphans
animus runner orphans detect

# Clean up
animus runner orphans cleanup

# Check restart stats
animus runner restart-stats
```

### Workflow Stuck

```bash
# Check workflow state
animus workflow list --status running

# Get details
animus workflow get --id WF-XXX

# Check if agent is running
animus daemon agents

# Check logs
animus daemon logs | grep WF-XXX

# May need to cancel
animus workflow cancel --id WF-XXX --confirmation yes
```

---

## Log Analysis

### Find Errors

```bash
# Recent errors
animus daemon logs | grep -i error

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
animus daemon health
```

### Full Diagnostics

```bash
# Environment
animus doctor

# Daemon status
animus daemon status

# Runner health
animus runner health

# Model availability
animus model status

# Recent errors
animus errors list --limit 10
```

### Automated Health Check Script

```bash
#!/bin/bash
# health-check.sh

echo "=== Daemon Status ==="
animus daemon status

echo -e "\n=== Runner Health ==="
animus runner health

echo -e "\n=== Queue Stats ==="
animus queue stats

echo -e "\n=== Task Stats ==="
animus task stats

echo -e "\n=== Recent Errors ==="
animus errors list --limit 5

echo -e "\n=== Model Status ==="
animus model status
```

---

## Best Practices

### Starting the Daemon

1. Check if already running: `animus daemon status`
2. Start with appropriate options: `animus daemon start --autonomous`
3. Verify startup: `animus daemon health`

### Daily Operations

1. Morning: Check status and health
2. Throughout day: Monitor with `animus daemon events`
3. End of day: Review task progress, pause or stop as needed

### Before Maintenance

1. Pause daemon: `animus daemon pause`
2. Wait for in-progress work: `animus workflow list --status running`
3. Make changes
4. Resume: `animus daemon resume`

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
