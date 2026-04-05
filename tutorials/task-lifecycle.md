# Task Lifecycle Walkthrough

Follow a task from creation to completion. This tutorial covers status transitions, assignments, dependencies, checklists, and how tasks interact with the daemon.

## Overview

Every task in Animus goes through a defined lifecycle:

```
Created → Backlog → Ready → In-Progress → Done
                      ↓           ↓
                   Blocked    Cancelled
                      ↓
                   On-Hold
```

Let's walk through each stage with practical examples.

---

## Stage 1: Task Creation

### Basic Creation

Create a new task with minimal information:

```bash
animus task create --title "Implement user profile page"
```

Output shows the new task ID (e.g., `TASK-042`).

### Creation with Full Details

```bash
animus task create \
  --title "Implement password reset" \
  --type feature \
  --priority high \
  --description "Add password reset functionality with email verification and token expiration."
```

**Task Types**:

| Type | Use When |
|------|----------|
| `feature` | Adding new functionality |
| `bugfix` | Fixing a known defect |
| `hotfix` | Urgent production fix |
| `refactor` | Restructuring without behavior change |
| `docs` | Documentation updates |
| `test` | Test coverage additions |
| `chore` | Maintenance, dependencies, CI |
| `experiment` | Exploratory or spike work |

**Priorities**:

| Priority | Use When |
|----------|----------|
| `critical` | Blocking everything else |
| `high` | Important, work soon |
| `medium` | Normal priority |
| `low` | Nice to have, can wait |

### Link to Requirements

Tasks are often created from requirements. If you've run `animus requirements execute`, tasks are auto-created. You can also link manually:

```bash
animus task update --id TASK-042 --linked-requirement REQ-001
```

### Verify Creation

```bash
animus task get --id TASK-042
```

This shows all task details including:
- Title and description
- Status (starts as `backlog`)
- Priority
- Assigned requirements
- Dependencies
- Checklist items

---

## Stage 2: Backlog to Ready

New tasks start in `backlog` status. Before work can begin, move them to `ready`.

### Move to Ready

```bash
animus task status --id TASK-042 --status ready
```

### What "Ready" Means

A task is `ready` when:
- It has a clear title and description
- Dependencies are satisfied (or none exist)
- It's not blocked
- It's ready for assignment

The daemon only picks up tasks with status `ready`.

### Check Task Position

See where this task sits in the priority queue:

```bash
animus task prioritized
```

This shows all ready tasks sorted by priority, then by dependency order.

---

## Stage 3: Assignment

Assign tasks to specify who (or what) will do the work.

### Assign to Agent

```bash
animus task assign --id TASK-042 --type agent --model claude-sonnet-4-6
```

This tells the daemon to use a Claude agent with Sonnet 4.6 when executing this task.

### Assign to Human

```bash
animus task assign --id TASK-042 --type human --assignee "alice@example.com"
```

Human-assigned tasks won't be picked up by the daemon automatically.

### Check Assignment

```bash
animus task get --id TASK-042 | grep -A5 "assignee"
```

### Unassign

To clear assignment:

```bash
animus task update --id TASK-042 --assignee ""
```

---

## Stage 4: Dependencies

Dependencies ensure tasks execute in the correct order.

### Add Dependency

TASK-043 should wait for TASK-042:

```bash
animus task dependency-add --id TASK-043 --depends-on TASK-042
```

Now TASK-043 cannot move to `in-progress` until TASK-042 is `done`.

### Check Dependencies

```bash
animus task get --id TASK-043
```

Look for the `dependencies` field.

### Dependency Chain

Create a sequence:

```bash
# Setup task
animus task create --title "Create database schema" --type feature
# Returns TASK-050

# Implementation task
animus task create --title "Implement API endpoints" --type feature
# Returns TASK-051

# Test task
animus task create --title "Write integration tests" --type test
# Returns TASK-052

# Set up chain: TASK-050 → TASK-051 → TASK-052
animus task dependency-add --id TASK-051 --depends-on TASK-050
animus task dependency-add --id TASK-052 --depends-on TASK-051

# Verify
animus task get --id TASK-052
```

### Remove Dependency

```bash
animus task dependency-remove --id TASK-043 --depends-on TASK-042
```

---

## Stage 5: Checklist Items

Checklists track subtasks and acceptance criteria within a task.

### Add Checklist Items

```bash
animus task checklist-add --id TASK-042 --item "Create password reset token table"
animus task checklist-add --id TASK-042 --item "Implement /auth/reset-request endpoint"
animus task checklist-add --id TASK-042 --item "Send reset email with token link"
animus task checklist-add --id TASK-042 --item "Implement /auth/reset-password endpoint"
animus task checklist-add --id TASK-042 --item "Add rate limiting (5 requests/hour)"
animus task checklist-add --id TASK-042 --item "Write unit tests"
animus task checklist-add --id TASK-042 --item "Write integration tests"
```

### View Checklist

```bash
animus task get --id TASK-042
```

The checklist appears in the task details.

### Mark Items Complete

As work progresses:

```bash
animus task checklist-update --id TASK-042 --index 0 --done true
animus task checklist-update --id TASK-042 --index 1 --done true
```

Indices are 0-based in the order items were added.

### Why Checklists Matter

- **Agents** use checklists during PO review phases to verify acceptance criteria
- **Humans** use them to track progress within a task
- **Workflows** can reference checklist completion for decisions

---

## Stage 6: In-Progress

When work begins (either by daemon or human), the task moves to `in-progress`.

### Manual Transition

```bash
animus task status --id TASK-042 --status in-progress
```

### Daemon Transition

When the daemon picks up a ready task, it automatically:
1. Sets status to `in-progress`
2. Creates a git worktree
3. Dispatches the workflow
4. Monitors execution

### Monitor In-Progress Tasks

```bash
# List all in-progress
animus task list --status in-progress

# Check workflow status
animus workflow list --status running

# Watch daemon activity
animus daemon events
```

---

## Stage 7: Blocking and Pausing

Sometimes tasks get stuck.

### Block a Task

```bash
animus task status --id TASK-042 --status blocked --reason "Waiting on API credentials from ops team"
```

Blocked tasks:
- Are not picked up by the daemon
- Show up in blocked task queries
- Have a clear reason for the blockage

### Unblock a Task

```bash
animus task status --id TASK-042 --status ready
```

This clears the `blocked_at`, `blocked_reason`, and `blocked_by` fields automatically.

### Pause a Task

Pause prevents scheduling without marking as blocked:

```bash
animus task pause --id TASK-042
```

### Resume a Paused Task

```bash
animus task resume --id TASK-042
```

### Query Blocked/Paused

```bash
animus task list --status blocked
animus task list --status on_hold
```

---

## Stage 8: Completion

When work is done, mark the task complete.

### Mark as Done

```bash
animus task status --id TASK-042 --status done
```

### What Happens on Completion

1. **Dependencies unblock**: Any tasks depending on this one can now proceed
2. **Worktree cleanup**: The daemon may clean up the git worktree
3. **Linked requirements update**: Parent requirement status may change
4. **Metrics update**: Task stats reflect the completion

### Verify Completion

```bash
animus task get --id TASK-042
```

Check that:
- Status is `done`
- All checklist items are complete
- Dependencies are resolved

---

## Stage 9: Cancellation

Sometimes tasks are no longer needed.

### Cancel a Task

```bash
animus task cancel --id TASK-042
```

This requires confirmation:

```bash
animus task cancel --id TASK-042 --confirmation yes
```

### When to Cancel

- Requirements changed
- Task is no longer relevant
- Duplicate of another task
- Decided not to implement

### Cancelled vs Done

- `done`: Work completed successfully
- `cancelled`: Work abandoned without completion

Both are "terminal" states—tasks don't automatically leave them.

---

## Full Example: Feature Implementation

Let's trace a complete feature from start to finish.

### Step 1: Create the Task

```bash
animus task create \
  --title "Add two-factor authentication" \
  --type feature \
  --priority high \
  --description "Implement TOTP-based 2FA with QR code setup and recovery codes."
# Returns: TASK-100
```

### Step 2: Add Checklist

```bash
animus task checklist-add --id TASK-100 --item "Add TOTP secret column to users table"
animus task checklist-add --id TASK-100 --item "Create /auth/2fa/setup endpoint"
animus task checklist-add --id TASK-100 --item "Generate QR code for authenticator apps"
animus task checklist-add --id TASK-100 --item "Create /auth/2fa/verify endpoint"
animus task checklist-add --id TASK-100 --item "Add recovery code generation"
animus task checklist-add --id TASK-100 --item "Update login flow to check 2FA"
animus task checklist-add --id TASK-100 --item "Write unit tests"
animus task checklist-add --id TASK-100 --item "Write integration tests"
animus task checklist-add --id TASK-100 --item "Update user documentation"
```

### Step 3: Set Up Dependencies

```bash
# This task depends on the user model refactor
animus task dependency-add --id TASK-100 --depends-on TASK-095
```

### Step 4: Move to Ready

```bash
# After TASK-095 is done
animus task status --id TASK-100 --status ready
```

### Step 5: Assign

```bash
animus task assign --id TASK-100 --type agent --model claude-sonnet-4-6
```

### Step 6: Start Daemon (if not running)

```bash
animus daemon start --autonomous
```

### Step 7: Monitor Progress

```bash
# Check task status
animus task get --id TASK-100

# Watch workflow
animus workflow list --status running

# Stream output
animus output monitor --run-id <run-id>
```

### Step 8: Handle Blocking (if needed)

If the task gets blocked:

```bash
animus task status --id TASK-100 --status blocked --reason "Need TOTP library approval"

# After approval
animus task status --id TASK-100 --status ready
```

### Step 9: Verify Completion

```bash
# Check task is done
animus task get --id TASK-100

# Verify all checklist items
# (should show all complete)

# Check linked requirement updated
animus requirements get --id REQ-050
```

---

## Task History

View everything that happened to a task:

```bash
animus task history --id TASK-100
```

This shows:
- Workflow dispatches
- Status changes
- Phase completions
- Decisions made

---

## Querying Tasks

### By Status

```bash
animus task list --status ready
animus task list --status in-progress
animus task list --status blocked
animus task list --status done
```

### By Priority

```bash
animus task list --priority critical
animus task list --priority high
```

### By Type

```bash
animus task list --type feature
animus task list --type bugfix
```

### By Assignee

```bash
animus task list --assignee "alice@example.com"
```

### Combined Filters

```bash
animus task list --status ready --priority high --type feature
```

### JSON Output for Processing

```bash
animus task list --status ready --json | jq '.data[] | {id, title, priority}'
```

---

## Best Practices

### Writing Good Titles

✅ Good:
- "Add password reset with email verification"
- "Fix race condition in order processing"
- "Update API docs for v2 endpoints"

❌ Bad:
- "Fix bug"
- "Update stuff"
- "Do the thing"

### Writing Good Descriptions

Include:
- **What**: What needs to be done
- **Why**: Context and motivation
- **How**: Implementation hints if relevant
- **Acceptance**: What "done" looks like

### Setting Priorities

- **Critical**: Only for true blockers
- **High**: Important work for current sprint
- **Medium**: Default for most tasks
- **Low**: Backlog items, nice-to-haves

### Using Checklists

- Add items for each major piece of work
- Include testing and documentation
- Update as you discover new requirements
- Use for acceptance criteria tracking

---

## Related Documentation

- [CLI Cheat Sheet](cli-cheat-sheet.md) -- Quick command reference
- [Common Sequences](common-sequences.md) -- Everyday command patterns
- [Task Management Guide](../guides/task-management.md) -- Comprehensive task docs
- [Workflows](../concepts/workflows.md) -- How tasks execute
