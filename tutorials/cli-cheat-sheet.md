# AO CLI Cheat Sheet

Quick reference for the most essential AO commands. Organized by category with common flags and examples.

## Global Flags

```bash
--json                    # Machine-readable JSON output
--project-root <PATH>     # Override project root directory
PROJECT_ROOT=/path        # Alternative via environment variable
```

---

## Quick Status

```bash
ao status                 # Unified project dashboard
ao version                # Show installed version
ao doctor                 # Environment diagnostics
ao doctor --fix           # Auto-fix common issues
```

---

## Daemon Operations

### Lifecycle

```bash
ao daemon start --autonomous    # Start background daemon
ao daemon run                   # Run in foreground (debugging)
ao daemon stop                  # Stop daemon gracefully
ao daemon status                # Check if running
ao daemon health                # Detailed health metrics
```

### Scheduler Control

```bash
ao daemon pause                 # Pause scheduling (in-progress continues)
ao daemon resume                # Resume scheduling
```

### Monitoring

```bash
ao daemon logs                  # Read daemon logs
ao daemon events                # Stream event history
ao daemon agents                # List active agents
ao daemon config                # View/update automation settings
```

---

## Task Management

### Creating Tasks

```bash
ao task create --title "Add feature X"                    # Basic task
ao task create --title "Fix bug" --type bugfix --priority high
ao task create --title "Docs" --type docs --priority low
```

**Task Types**: `feature`, `bugfix`, `hotfix`, `refactor`, `docs`, `test`, `chore`, `experiment`

**Priorities**: `critical`, `high`, `medium`, `low`

### Querying Tasks

```bash
ao task list                           # All tasks
ao task list --status ready            # Filter by status
ao task list --priority high           # Filter by priority
ao task list --type feature            # Filter by type
ao task prioritized                    # Sorted by priority
ao task next                           # Next ready task
ao task stats                          # Statistics summary
ao task get --id TASK-001              # Single task details
```

### Task Status

```bash
ao task status --id TASK-001 --status ready
ao task status --id TASK-001 --status in-progress
ao task status --id TASK-001 --status done
ao task status --id TASK-001 --status blocked --reason "Waiting on X"
```

**Statuses**: `backlog`, `todo`, `ready`, `in_progress`, `blocked`, `on_hold`, `done`, `cancelled`

### Assignment

```bash
ao task assign --id TASK-001 --type agent --model claude-sonnet-4-6
ao task assign --id TASK-001 --type human --assignee "alice"
```

### Priority & Deadlines

```bash
ao task set-priority --id TASK-001 --priority critical
ao task set-deadline --id TASK-001 --deadline "2025-03-15"
ao task set-deadline --id TASK-001 --clear    # Remove deadline
ao task rebalance-priority                    # Rebalance all by budget
```

### Dependencies

```bash
ao task dependency-add --id TASK-002 --depends-on TASK-001
ao task dependency-remove --id TASK-002 --depends-on TASK-001
```

### Checklists

```bash
ao task checklist-add --id TASK-001 --item "Write unit tests"
ao task checklist-update --id TASK-001 --index 0 --done true
```

### Control

```bash
ao task pause --id TASK-001          # Prevent scheduling
ao task resume --id TASK-001         # Resume scheduling
ao task cancel --id TASK-001         # Cancel (needs confirmation)
ao task history --id TASK-001        # View workflow history
```

---

## Workflow Operations

### Running Workflows

```bash
ao workflow run --task-id TASK-001                    # Async via daemon
ao workflow run --task-id TASK-001 --workflow-ref custom-workflow
ao workflow execute --task-id TASK-001                # Sync, blocking
```

### Monitoring

```bash
ao workflow list                           # All workflows
ao workflow list --status running          # Running only
ao workflow get --id WF-001                # Workflow details
ao workflow decisions --id WF-001          # Decision history
ao workflow checkpoints list --id WF-001   # Saved checkpoints
```

### Control

```bash
ao workflow pause --id WF-001              # Pause execution
ao workflow resume --id WF-001             # Resume paused workflow
ao workflow cancel --id WF-001             # Cancel permanently
ao workflow phase approve --workflow-id WF-001   # Approve gate
```

### Configuration

```bash
ao workflow config get                     # View workflow config
ao workflow config validate                # Validate config
ao workflow phases list                    # List phase definitions
ao workflow definitions list               # List workflow definitions
```

---

## Requirements

### Planning

```bash
ao vision draft                           # Draft project vision
ao vision refine                          # Refine vision
ao vision get                             # Read current vision
```

### Requirements Management

```bash
ao requirements draft --include-codebase-scan
ao requirements list
ao requirements get --id REQ-001
ao requirements refine --requirement-ids REQ-001 REQ-002
ao requirements execute --requirement-ids REQ-001 REQ-002
ao requirements update --id REQ-001 --status refined
ao requirements delete --id REQ-001
```

**Priorities (MoSCoW)**: `must`, `should`, `could`, `wont`

---

## Queue Operations

```bash
ao queue list                    # List queued dispatches
ao queue stats                   # Queue statistics
ao queue enqueue --task-id TASK-001
ao queue hold --subject-id TASK-001      # Hold from dispatch
ao queue release --subject-id TASK-001   # Release held item
ao queue drop --subject-id TASK-001      # Remove from queue
ao queue reorder --subject-ids TASK-003 TASK-001 TASK-002
```

---

## Output & Monitoring

```bash
ao output run --id RUN-001              # Read run output
ao output monitor --run-id RUN-001      # Stream live output
ao output tail --run-id RUN-001         # Recent output
ao output artifacts --execution-id EXEC-001
ao output jsonl --run-id RUN-001        # Structured logs
```

---

## Git Operations

```bash
ao git status                           # Repository status
ao git branches                         # List branches
ao git commit                           # Commit changes
ao git push                             # Push branch
ao git pull                             # Pull branch
```

### Worktrees

```bash
ao git worktree list                    # List worktrees
ao git worktree create --task-id TASK-001
ao git worktree remove --task-id TASK-001
ao git worktree prune                   # Clean up task worktrees
ao git worktree sync --task-id TASK-001 # Pull + push
```

---

## Model & Runner

```bash
ao model status                         # Model + API key status
ao model availability                   # Check model availability
ao model validate                       # Validate model selection
ao runner health                        # Runner health
ao runner orphans detect                # Find orphaned processes
ao runner orphans cleanup               # Clean up orphans
ao runner restart-stats                 # Restart history
```

---

## Packs

```bash
ao pack list                            # List packs
ao pack inspect --pack-id ao.task
ao pack install --path /path/to/pack
ao pack pin --pack-id ao.task --version =0.1.0
ao pack search <query>
```

---

## Project Management

```bash
ao project list                         # List projects
ao project active                       # Show active project
ao project get --id my-project
ao project create --name "New Project"
ao project load --id my-project         # Set as active
```

---

## Web UI

```bash
ao web serve                            # Start web server
ao web open                             # Open in browser
```

---

## MCP Server

```bash
ao mcp serve                            # Start MCP server
```

---

## TUI (Terminal UI)

```bash
ao tui                                  # Interactive terminal UI
```

---

## Common Patterns

### Start New Project

```bash
cd /path/to/project
ao setup
ao pack list
ao vision draft
ao requirements draft --include-codebase-scan
ao requirements execute
ao daemon start --autonomous
```

### Daily Check-in

```bash
ao status                    # Overall status
ao task stats                # Task summary
ao daemon health             # Daemon health
ao task prioritized          # What's next
```

### Debug Stuck Workflow

```bash
ao daemon status             # Is daemon running?
ao daemon logs               # Check for errors
ao workflow list --status running
ao workflow get --id WF-XXX
ao workflow decisions --id WF-XXX
ao output run --id RUN-XXX
```

### Pause for Manual Work

```bash
ao daemon pause
# Make your changes...
ao daemon resume
```

### JSON Output for Scripts

```bash
ao task list --json | jq '.data[] | select(.status == "ready") | .id'
ao workflow list --json --status running | jq '.data | length'
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
