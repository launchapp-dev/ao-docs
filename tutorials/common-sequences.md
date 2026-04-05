# Common Command Sequences

Practical command sequences and one-liners for everyday Animus operations. Use these as starting points for your workflows and scripts.

## Project Initialization

### New Project Setup

Complete initialization from empty repository to running daemon:

```bash
# Navigate to project
cd /path/to/your/project

# Initialize Animus
animus setup

# Verify packs are available
animus pack list

# Draft vision and requirements
animus vision draft
animus requirements draft --include-codebase-scan

# Refine specific requirements
animus requirements refine --requirement-ids REQ-001 REQ-002

# Generate tasks from requirements
animus requirements execute --requirement-ids REQ-001 REQ-002 REQ-003

# Start autonomous execution
animus daemon start --autonomous
```

### Quick Status Check

```bash
animus status && ao task stats && animus daemon health
```

---

## Daily Operations

### Morning Check-in

Start your day with a comprehensive status overview:

```bash
# Full status dashboard
animus status

# Task statistics
animus task stats

# What's the daemon doing?
animus daemon status

# What's coming up next?
animus task prioritized --limit 10

# Any errors recently?
animus errors list --limit 5
```

### End-of-Day Summary

```bash
# What completed today?
animus task list --status done

# What's still in progress?
animus task list --status in-progress

# Workflow activity
animus workflow list --limit 20

# Daemon health
animus daemon health
```

---

## Task Management Patterns

### Create and Start a Task

```bash
# Create task
animus task create --title "Implement OAuth login" --type feature --priority high

# Get the task ID from output, then start it
animus task status --id TASK-042 --status ready

# Assign to an agent
animus task assign --id TASK-042 --type agent --model claude-sonnet-4-6

# Or assign to yourself
animus task assign --id TASK-042 --type human --assignee "$(whoami)"
```

### Bulk Status Updates

Move multiple tasks to ready:

```bash
for id in TASK-001 TASK-002 TASK-003; do
  animus task status --id "$id" --status ready
done
```

### Task with Full Details

```bash
# Get comprehensive task info
animus task get --id TASK-001

# View its history
animus task history --id TASK-001

# Check linked requirements
animus requirements get --id REQ-001
```

### Set Up Task Dependencies

Create a dependency chain:

```bash
# TASK-002 depends on TASK-001
animus task dependency-add --id TASK-002 --depends-on TASK-001

# TASK-003 depends on TASK-002
animus task dependency-add --id TASK-003 --depends-on TASK-002

# Verify the chain
animus task get --id TASK-003  # Shows dependencies
```

### Task with Checklist

```bash
# Create task
animus task create --title "Add password reset flow"

# Add checklist items
animus task checklist-add --id TASK-050 --item "Design reset token schema"
animus task checklist-add --id TASK-050 --item "Implement /reset-password endpoint"
animus task checklist-add --id TASK-050 --item "Send reset email"
animus task checklist-add --id TASK-050 --item "Add rate limiting"
animus task checklist-add --id TASK-050 --item "Write integration tests"

# Mark items complete as you go
animus task checklist-update --id TASK-050 --index 0 --done true
animus task checklist-update --id TASK-050 --index 1 --done true
```

---

## Daemon Operations

### Start and Verify

```bash
# Start autonomous daemon
animus daemon start --autonomous

# Wait a moment, then verify
sleep 2 && animus daemon status

# Check health
animus daemon health

# Watch initial activity
animus daemon events --limit 20
```

### Pause for Manual Changes

When you need to make changes without daemon interference:

```bash
# Pause scheduler
animus daemon pause

# Verify paused
animus daemon status

# Make your changes...
vim src/important-file.rs

# Resume when done
animus daemon resume
```

### Debug Daemon Issues

```bash
# Check if running
animus daemon status

# If not running, try foreground mode
animus daemon run

# Check logs for errors
animus daemon logs | grep -i error

# Check runner health
animus runner health

# Look for orphaned processes
animus runner orphans detect
```

### Clean Restart

```bash
# Stop daemon
animus daemon stop

# Clear old logs
animus daemon clear-logs

# Clean up any orphans
animus runner orphans cleanup

# Start fresh
animus daemon start --autonomous
```

---

## Workflow Patterns

### Run Single Task Workflow

```bash
# Async (daemon handles it)
animus workflow run --task-id TASK-001

# Sync (blocks until done)
animus workflow execute --task-id TASK-001
```

### Run with Custom Workflow

```bash
# Use a specific workflow definition
animus workflow run --task-id TASK-001 --workflow-ref hotfix-workflow
```

### Monitor Running Workflow

```bash
# List running workflows
animus workflow list --status running

# Get details
animus workflow get --id WF-ABC123

# Watch output
animus output monitor --run-id RUN-XYZ789

# Check decisions
animus workflow decisions --id WF-ABC123
```

### Handle Stuck Workflow

```bash
# Check workflow state
animus workflow get --id WF-STUCK

# If paused at a gate, approve it
animus workflow phase approve --workflow-id WF-STUCK

# If truly stuck, cancel and retry
animus workflow cancel --id WF-STUCK

# Reset task to ready
animus task status --id TASK-XXX --status ready

# Re-run
animus workflow run --task-id TASK-XXX
```

---

## Queue Operations

### Queue Tasks for Execution

```bash
# Queue multiple tasks in order
animus queue enqueue --task-id TASK-001
animus queue enqueue --task-id TASK-002
animus queue enqueue --task-id TASK-003

# Check queue
animus queue list

# Reorder if needed
animus queue reorder --subject-ids TASK-003 TASK-001 TASK-002
```

### Manage Queue Items

```bash
# Hold a task (don't dispatch yet)
animus queue hold --subject-id TASK-001

# Release it
animus queue release --subject-id TASK-001

# Remove from queue entirely
animus queue drop --subject-id TASK-001
```

---

## Git and Worktrees

### Worktree Workflow

```bash
# List existing worktrees
animus git worktree list

# Create worktree for a task
animus git worktree create --task-id TASK-001

# Work happens in the worktree...

# Sync changes
animus git worktree sync --task-id TASK-001

# Remove when done
animus git worktree remove --task-id TASK-001
```

### Clean Up Old Worktrees

```bash
# Prune worktrees for completed/cancelled tasks
animus git worktree prune
```

---

## Monitoring & Debugging

### Watch Daemon Activity

```bash
# Stream daemon events
animus daemon events

# Or tail the log file
tail -f .ao/daemon.log
```

### Check Model Status

```bash
# All models and API keys
animus model status

# Specific model availability
animus model availability
```

### Error Investigation

```bash
# List recent errors
animus errors list --limit 10

# Get error details
animus errors get --id ERR-001

# Error statistics
animus errors stats
```

### Output Inspection

```bash
# Get run output
animus output run --id RUN-001

# Stream live output
animus output monitor --run-id RUN-001

# Get recent output events
animus output tail --run-id RUN-001 --limit 100

# Structured logs
animus output jsonl --run-id RUN-001
```

---

## JSON Output for Scripting

### Extract Task IDs

```bash
# All ready task IDs
animus task list --status ready --json | jq -r '.data[].id'

# High priority tasks
animus task list --priority high --json | jq '.data[] | {id, title, status}'
```

### Count Workflows

```bash
# Running workflows count
animus workflow list --status running --json | jq '.data | length'

# Failed today
animus workflow list --status failed --json | jq '.data | length'
```

### Export Task List

```bash
# Export to JSON file
animus task list --json > tasks-backup.json

# Export to CSV (with jq)
animus task list --json | jq -r '["id","title","status","priority"], (.data[] | [.id, .title, .status, .priority]) | @csv' > tasks.csv
```

### Check Multiple Projects

```bash
# List all projects
animus project list --json | jq -r '.data[].id'

# Check status of each
for project in $(ao project list --json | jq -r '.data[].id'); do
  echo "=== $project ==="
  animus task stats --project-root "$HOME/projects/$project"
done
```

---

## One-Liners

### Quick Counts

```bash
# Count tasks by status
animus task stats

# Count running workflows
animus workflow list --status running --json | jq '.data | length'

# Count queue depth
animus queue stats
```

### Quick Filters

```bash
# My assigned tasks
animus task list --assignee "$(whoami)"

# Critical tasks
animus task list --priority critical

# Blocked tasks
animus task list --status blocked
```

### Quick Actions

```bash
# Get next task to work on
animus task next

# What's the daemon doing?
animus daemon status && animus daemon health

# Is everything healthy?
animus doctor
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
animus daemon run

# Check all diagnostics
animus doctor --fix
```

---

## Recovery Patterns

### Reset Stuck Task

```bash
# Cancel any running workflow
animus workflow list --status running --json | jq -r '.data[] | select(.task_id == "TASK-001") | .id' | xargs -I {} ao workflow cancel --id {}

# Reset task status
animus task status --id TASK-001 --status ready

# Re-run
animus workflow run --task-id TASK-001
```

### Clean Slate

```bash
# Stop everything
animus daemon stop

# Clean up orphans
animus runner orphans cleanup

# Clear logs
animus daemon clear-logs

# Verify environment
animus doctor --fix

# Start fresh
animus daemon start --autonomous
```

---

## Related Documentation

- [CLI Cheat Sheet](cli-cheat-sheet.md) -- Quick reference for all commands
- [Task Lifecycle Walkthrough](task-lifecycle.md) -- Detailed task management
- [Daemon Quick Reference](daemon-quick-ref.md) -- Daemon operations
- [Troubleshooting Guide](../guides/troubleshooting.md) -- Debugging common issues
