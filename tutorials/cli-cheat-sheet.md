# Animus CLI Cheat Sheet

Quick reference for the most essential Animus commands. Organized by category with common flags and examples.

## Global Flags

```bash
--json                    # Machine-readable JSON output
--project-root <PATH>     # Override project root directory
PROJECT_ROOT=/path        # Alternative via environment variable
```

---

## Quick Status

```bash
animus status                 # Unified project dashboard
animus version                # Show installed version
animus doctor                 # Environment diagnostics
animus doctor --fix           # Auto-fix common issues
```

---

## Daemon Operations

### Lifecycle

```bash
animus daemon start --autonomous    # Start background daemon
animus daemon run                   # Run in foreground (debugging)
animus daemon stop                  # Stop daemon gracefully
animus daemon status                # Check if running
animus daemon health                # Detailed health metrics
```

### Scheduler Control

```bash
animus daemon pause                 # Pause scheduling (in-progress continues)
animus daemon resume                # Resume scheduling
```

### Monitoring

```bash
animus daemon logs                  # Read daemon logs
animus daemon events                # Stream event history
animus daemon agents                # List active agents
animus daemon config                # View/update automation settings
```

---

## Task Management

### Creating Tasks

```bash
animus task create --title "Add feature X"                    # Basic task
animus task create --title "Fix bug" --type bugfix --priority high
animus task create --title "Docs" --type docs --priority low
```

**Task Types**: `feature`, `bugfix`, `hotfix`, `refactor`, `docs`, `test`, `chore`, `experiment`

**Priorities**: `critical`, `high`, `medium`, `low`

### Querying Tasks

```bash
animus task list                           # All tasks
animus task list --status ready            # Filter by status
animus task list --priority high           # Filter by priority
animus task list --type feature            # Filter by type
animus task prioritized                    # Sorted by priority
animus task next                           # Next ready task
animus task stats                          # Statistics summary
animus task get --id TASK-001              # Single task details
```

### Task Status

```bash
animus task status --id TASK-001 --status ready
animus task status --id TASK-001 --status in-progress
animus task status --id TASK-001 --status done
animus task status --id TASK-001 --status blocked --reason "Waiting on X"
```

**Statuses**: `backlog`, `todo`, `ready`, `in_progress`, `blocked`, `on_hold`, `done`, `cancelled`

### Assignment

```bash
animus task assign --id TASK-001 --type agent --model claude-sonnet-4-6
animus task assign --id TASK-001 --type human --assignee "alice"
```

### Priority & Deadlines

```bash
animus task set-priority --id TASK-001 --priority critical
animus task set-deadline --id TASK-001 --deadline "2025-03-15"
animus task set-deadline --id TASK-001 --clear    # Remove deadline
animus task rebalance-priority                    # Rebalance all by budget
```

### Dependencies

```bash
animus task dependency-add --id TASK-002 --depends-on TASK-001
animus task dependency-remove --id TASK-002 --depends-on TASK-001
```

### Checklists

```bash
animus task checklist-add --id TASK-001 --item "Write unit tests"
animus task checklist-update --id TASK-001 --index 0 --done true
```

### Control

```bash
animus task pause --id TASK-001          # Prevent scheduling
animus task resume --id TASK-001         # Resume scheduling
animus task cancel --id TASK-001         # Cancel (needs confirmation)
animus task history --id TASK-001        # View workflow history
```

---

## Workflow Operations

### Running Workflows

```bash
animus workflow run --task-id TASK-001                    # Async via daemon
animus workflow run --task-id TASK-001 --workflow-ref custom-workflow
animus workflow execute --task-id TASK-001                # Sync, blocking
```

### Monitoring

```bash
animus workflow list                           # All workflows
animus workflow list --status running          # Running only
animus workflow get --id WF-001                # Workflow details
animus workflow decisions --id WF-001          # Decision history
animus workflow checkpoints list --id WF-001   # Saved checkpoints
```

### Control

```bash
animus workflow pause --id WF-001              # Pause execution
animus workflow resume --id WF-001             # Resume paused workflow
animus workflow cancel --id WF-001             # Cancel permanently
animus workflow phase approve --workflow-id WF-001   # Approve gate
```

### Configuration

```bash
animus workflow config get                     # View workflow config
animus workflow config validate                # Validate config
animus workflow phases list                    # List phase definitions
animus workflow definitions list               # List workflow definitions
```

---

## Requirements

### Planning

```bash
animus vision draft                           # Draft project vision
animus vision refine                          # Refine vision
animus vision get                             # Read current vision
```

### Requirements Management

```bash
animus requirements draft --include-codebase-scan
animus requirements list
animus requirements get --id REQ-001
animus requirements refine --requirement-ids REQ-001 REQ-002
animus requirements execute --requirement-ids REQ-001 REQ-002
animus requirements update --id REQ-001 --status refined
animus requirements delete --id REQ-001
```

**Priorities (MoSCoW)**: `must`, `should`, `could`, `wont`

---

## Queue Operations

```bash
animus queue list                    # List queued dispatches
animus queue stats                   # Queue statistics
animus queue enqueue --task-id TASK-001
animus queue hold --subject-id TASK-001      # Hold from dispatch
animus queue release --subject-id TASK-001   # Release held item
animus queue drop --subject-id TASK-001      # Remove from queue
animus queue reorder --subject-ids TASK-003 TASK-001 TASK-002
```

---

## Output & Monitoring

```bash
animus output run --id RUN-001              # Read run output
animus output monitor --run-id RUN-001      # Stream live output
animus output tail --run-id RUN-001         # Recent output
animus output artifacts --execution-id EXEC-001
animus output jsonl --run-id RUN-001        # Structured logs
```

---

## Git Operations

```bash
animus git status                           # Repository status
animus git branches                         # List branches
animus git commit                           # Commit changes
animus git push                             # Push branch
animus git pull                             # Pull branch
```

### Worktrees

```bash
animus git worktree list                    # List worktrees
animus git worktree create --task-id TASK-001
animus git worktree remove --task-id TASK-001
animus git worktree prune                   # Clean up task worktrees
animus git worktree sync --task-id TASK-001 # Pull + push
```

---

## Model & Runner

```bash
animus model status                         # Model + API key status
animus model availability                   # Check model availability
animus model validate                       # Validate model selection
animus runner health                        # Runner health
animus runner orphans detect                # Find orphaned processes
animus runner orphans cleanup               # Clean up orphans
animus runner restart-stats                 # Restart history
```

---

## Packs

```bash
animus pack list                            # List packs
animus pack inspect --pack-id ao.task
animus pack install --path /path/to/pack
animus pack pin --pack-id ao.task --version =0.1.0
animus pack search <query>
```

---

## Project Management

```bash
animus project list                         # List projects
animus project active                       # Show active project
animus project get --id my-project
animus project create --name "New Project"
animus project load --id my-project         # Set as active
```

---

## Web UI

```bash
animus web serve                            # Start web server
animus web open                             # Open in browser
```

---

## Cloud Deployment

```bash
animus cloud login                          # Authenticate (browser OAuth)
animus cloud login --token $AO_CLOUD_TOKEN  # Authenticate with PAT (CI)
animus cloud push                           # Push project config to cloud
animus cloud push --env staging             # Push to staging environment
animus cloud push --dry-run                 # Validate without uploading
animus cloud start                          # Start cloud-hosted daemon
animus cloud start --wait                   # Block until daemon is running
animus cloud stop                           # Graceful drain and stop
animus cloud stop --force                   # Immediate termination
animus cloud status                         # Show deployment + daemon status
animus cloud status --json                  # Machine-readable status
animus cloud logs                           # Read recent log history
animus cloud logs --follow                  # Stream logs in real time
animus cloud logs --tail 200 --follow       # Tail 200 lines then stream
animus cloud logs --level error             # Filter to errors only
animus cloud logs --run-id run_1234         # Scope to one agent run
animus cloud logs --since 1h --json         # Logs as newline-delimited JSON
```

---

## MCP Server

```bash
animus mcp serve                            # Start MCP server
```

---

## TUI (Terminal UI)

```bash
animus tui                                  # Interactive terminal UI
```

---

## Common Patterns

### Start New Project

```bash
cd /path/to/project
animus setup
animus pack list
animus vision draft
animus requirements draft --include-codebase-scan
animus requirements execute
animus daemon start --autonomous
```

### Daily Check-in

```bash
animus status                    # Overall status
animus task stats                # Task summary
animus daemon health             # Daemon health
animus task prioritized          # What's next
```

### Debug Stuck Workflow

```bash
animus daemon status             # Is daemon running?
animus daemon logs               # Check for errors
animus workflow list --status running
animus workflow get --id WF-XXX
animus workflow decisions --id WF-XXX
animus output run --id RUN-XXX
```

### Pause for Manual Work

```bash
animus daemon pause
# Make your changes...
animus daemon resume
```

### JSON Output for Scripts

```bash
animus task list --json | jq '.data[] | select(.status == "ready") | .id'
animus workflow list --json --status running | jq '.data | length'
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | Not found |
| 4 | Permission denied |
| 5 | Already exists |
| 6 | Conflict / precondition failed |

See [Exit Codes](../reference/cli/exit-codes.md) for complete reference.

---

## Keyboard Shortcuts (TUI)

| Key | Action |
|-----|--------|
| `?` | Help |
| `q` | Quit / Go back |
| `j` / `↓` | Move down |
| `k` / `↑` | Move up |
| `Enter` | Select |
| `Tab` | Next pane |
| `r` | Refresh |
| `p` | Pause/Resume daemon |
