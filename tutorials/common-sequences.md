# Common Command Sequences

Practical command sequences and one-liners for everyday AO operations. Use these as starting points for your workflows and scripts.

## Project Initialization

### New Project Setup

Complete initialization from empty repository to running daemon:

```bash
# Navigate to project
cd /path/to/your/project

# Initialize AO
ao setup

# Verify packs are available
ao pack list

# Draft vision and requirements
ao vision draft
ao requirements draft --include-codebase-scan

# Refine specific requirements
ao requirements refine --requirement-ids REQ-001 REQ-002

# Generate tasks from requirements
ao requirements execute --requirement-ids REQ-001 REQ-002 REQ-003

# Start autonomous execution
ao daemon start --autonomous
```

### Quick Status Check

```bash
ao status && ao task stats && ao daemon health
```

---

## Daily Operations

### Morning Check-in

Start your day with a comprehensive status overview:

```bash
# Full status dashboard
ao status

# Task statistics
ao task stats

# What's the daemon doing?
ao daemon status

# What's coming up next?
ao task prioritized --limit 10

# Any errors recently?
ao errors list --limit 5
```

### End-of-Day Summary

```bash
# What completed today?
ao task list --status done

# What's still in progress?
ao task list --status in-progress

# Workflow activity
ao workflow list --limit 20

# Daemon health
ao daemon health
```

---

## Task Management Patterns

### Create and Start a Task

```bash
# Create task
ao task create --title "Implement OAuth login" --type feature --priority high

# Get the task ID from output, then start it
ao task status --id TASK-042 --status ready

# Assign to an agent
ao task assign --id TASK-042 --type agent --model claude-sonnet-4-6

# Or assign to yourself
ao task assign --id TASK-042 --type human --assignee "$(whoami)"
```

### Bulk Status Updates

Move multiple tasks to ready:

```bash
for id in TASK-001 TASK-002 TASK-003; do
  ao task status --id "$id" --status ready
done
```

### Task with Full Details

```bash
# Get comprehensive task info
ao task get --id TASK-001

# View its history
ao task history --id TASK-001

# Check linked requirements
ao requirements get --id REQ-001
```

### Set Up Task Dependencies

Create a dependency chain:

```bash
# TASK-002 depends on TASK-001
ao task dependency-add --id TASK-002 --depends-on TASK-001

# TASK-003 depends on TASK-002
ao task dependency-add --id TASK-003 --depends-on TASK-002

# Verify the chain
ao task get --id TASK-003  # Shows dependencies
```

### Task with Checklist

```bash
# Create task
ao task create --title "Add password reset flow"

# Add checklist items
ao task checklist-add --id TASK-050 --item "Design reset token schema"
ao task checklist-add --id TASK-050 --item "Implement /reset-password endpoint"
ao task checklist-add --id TASK-050 --item "Send reset email"
ao task checklist-add --id TASK-050 --item "Add rate limiting"
ao task checklist-add --id TASK-050 --item "Write integration tests"

# Mark items complete as you go
ao task checklist-update --id TASK-050 --index 0 --done true
ao task checklist-update --id TASK-050 --index 1 --done true
```

---

## Daemon Operations

### Start and Verify

```bash
# Start autonomous daemon
ao daemon start --autonomous

# Wait a moment, then verify
sleep 2 && ao daemon status

# Check health
ao daemon health

# Watch initial activity
ao daemon events --limit 20
```

### Pause for Manual Changes

When you need to make changes without daemon interference:

```bash
# Pause scheduler
ao daemon pause

# Verify paused
ao daemon status

# Make your changes...
vim src/important-file.rs

# Resume when done
ao daemon resume
```

### Debug Daemon Issues

```bash
# Check if running
ao daemon status

# If not running, try foreground mode
ao daemon run

# Check logs for errors
ao daemon logs | grep -i error

# Check runner health
ao runner health

# Look for orphaned processes
ao runner orphans detect
```

### Clean Restart

```bash
# Stop daemon
ao daemon stop

# Clear old logs
ao daemon clear-logs

# Clean up any orphans
ao runner orphans cleanup

# Start fresh
ao daemon start --autonomous
```

---

## Workflow Patterns

### Run Single Task Workflow

```bash
# Async (daemon handles it)
ao workflow run --task-id TASK-001

# Sync (blocks until done)
ao workflow execute --task-id TASK-001
```

### Run with Custom Workflow

```bash
# Use a specific workflow definition
ao workflow run --task-id TASK-001 --workflow-ref hotfix-workflow
```

### Monitor Running Workflow

```bash
# List running workflows
ao workflow list --status running

# Get details
ao workflow get --id WF-ABC123

# Watch output
ao output monitor --run-id RUN-XYZ789

# Check decisions
ao workflow decisions --id WF-ABC123
```

### Handle Stuck Workflow

```bash
# Check workflow state
ao workflow get --id WF-STUCK

# If paused at a gate, approve it
ao workflow phase approve --workflow-id WF-STUCK

# If truly stuck, cancel and retry
ao workflow cancel --id WF-STUCK

# Reset task to ready
ao task status --id TASK-XXX --status ready

# Re-run
ao workflow run --task-id TASK-XXX
```

---

## Queue Operations

### Queue Tasks for Execution

```bash
# Queue multiple tasks in order
ao queue enqueue --task-id TASK-001
ao queue enqueue --task-id TASK-002
ao queue enqueue --task-id TASK-003

# Check queue
ao queue list

# Reorder if needed
ao queue reorder --subject-ids TASK-003 TASK-001 TASK-002
```

### Manage Queue Items

```bash
# Hold a task (don't dispatch yet)
ao queue hold --subject-id TASK-001

# Release it
ao queue release --subject-id TASK-001

# Remove from queue entirely
ao queue drop --subject-id TASK-001
```

---

## Git and Worktrees

### Worktree Workflow

```bash
# List existing worktrees
ao git worktree list

# Create worktree for a task
ao git worktree create --task-id TASK-001

# Work happens in the worktree...

# Sync changes
ao git worktree sync --task-id TASK-001

# Remove when done
ao git worktree remove --task-id TASK-001
```

### Clean Up Old Worktrees

```bash
# Prune worktrees for completed/cancelled tasks
ao git worktree prune
```

---

## Monitoring & Debugging

### Watch Daemon Activity

```bash
# Stream daemon events
ao daemon events

# Or tail the log file
tail -f .ao/daemon.log
```

### Check Model Status

```bash
# All models and API keys
ao model status

# Specific model availability
ao model availability
```

### Error Investigation

```bash
# List recent errors
ao errors list --limit 10

# Get error details
ao errors get --id ERR-001

# Error statistics
ao errors stats
```

### Output Inspection

```bash
# Get run output
ao output run --id RUN-001

# Stream live output
ao output monitor --run-id RUN-001

# Get recent output events
ao output tail --run-id RUN-001 --limit 100

# Structured logs
ao output jsonl --run-id RUN-001
```

---

## JSON Output for Scripting

### Extract Task IDs

```bash
# All ready task IDs
ao task list --status ready --json | jq -r '.data[].id'

# High priority tasks
ao task list --priority high --json | jq '.data[] | {id, title, status}'
```

### Count Workflows

```bash
# Running workflows count
ao workflow list --status running --json | jq '.data | length'

# Failed today
ao workflow list --status failed --json | jq '.data | length'
```

### Export Task List

```bash
# Export to JSON file
ao task list --json > tasks-backup.json

# Export to CSV (with jq)
ao task list --json | jq -r '["id","title","status","priority"], (.data[] | [.id, .title, .status, .priority]) | @csv' > tasks.csv
```

### Check Multiple Projects

```bash
# List all projects
ao project list --json | jq -r '.data[].id'

# Check status of each
for project in $(ao project list --json | jq -r '.data[].id'); do
  echo "=== $project ==="
  ao task stats --project-root "$HOME/projects/$project"
done
```

---

## One-Liners

### Quick Counts

```bash
# Count tasks by status
ao task stats

# Count running workflows
ao workflow list --status running --json | jq '.data | length'

# Count queue depth
ao queue stats
```

### Quick Filters

```bash
# My assigned tasks
ao task list --assignee "$(whoami)"

# Critical tasks
ao task list --priority critical

# Blocked tasks
ao task list --status blocked
```

### Quick Actions

```bash
# Get next task to work on
ao task next

# What's the daemon doing?
ao daemon status && ao daemon health

# Is everything healthy?
ao doctor
```

### Log Inspection

```bash
# Errors in daemon log
grep -i error .ao/daemon.log | tail -20

# Today's workflow dispatches
grep "workflow_dispatched" .ao/daemon.log | grep "$(date +%Y-%m-%d)"
```

---

## Environment Setup

### Set API Keys

```bash
# Add to ~/.bashrc or ~/.zshrc
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export GEMINI_API_KEY="..."
```

### Set Default Project

```bash
# Option 1: Environment variable
export PROJECT_ROOT=/path/to/project

# Option 2: Always use --project-root
alias ao='ao --project-root /path/to/project'
```

### Debug Mode

```bash
# Run daemon in foreground for debugging
ao daemon run

# Check all diagnostics
ao doctor --fix
```

---

## Recovery Patterns

### Reset Stuck Task

```bash
# Cancel any running workflow
ao workflow list --status running --json | jq -r '.data[] | select(.task_id == "TASK-001") | .id' | xargs -I {} ao workflow cancel --id {}

# Reset task status
ao task status --id TASK-001 --status ready

# Re-run
ao workflow run --task-id TASK-001
```

### Clean Slate

```bash
# Stop everything
ao daemon stop

# Clean up orphans
ao runner orphans cleanup

# Clear logs
ao daemon clear-logs

# Verify environment
ao doctor --fix

# Start fresh
ao daemon start --autonomous
```

---

## Related Documentation

- [CLI Cheat Sheet](cli-cheat-sheet.md) -- Quick reference for all commands
- [Task Lifecycle Walkthrough](task-lifecycle.md) -- Detailed task management
- [Daemon Quick Reference](daemon-quick-ref.md) -- Daemon operations
- [Troubleshooting Guide](../guides/troubleshooting.md) -- Debugging common issues
