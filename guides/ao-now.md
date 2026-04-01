# ao now — Work Inbox

`ao now` is your command-line work inbox. It shows you exactly what to focus on: the next ready task, any workflows currently running, and any items that are blocked or stale. Run it at the start of your session or whenever you want a quick situational snapshot.

## Basic Usage

```bash
ao now
```

Example output:

```
AO Now/Inbox Surface
Generated At: 2026-03-31T09:15:00Z

Next Task
  id: TASK-042
  title: Fix auth token expiry edge case
  priority: High
  status: Backlog

Active Workflows
  - id=3a521769 task_id=TASK-038 task_title=Refactor cache layer phase=implement-rust
  - id=b4880e66 task_id=TASK-040 task_title=Update API docs phase=update-docs

Blocked Items
  none

Stale Items (>7 days in progress)
  none
```

## What Each Section Means

| Section | Description |
|---|---|
| **Next Task** | The highest-priority backlog task that is ready to work on. This is what the daemon would dispatch next. |
| **Active Workflows** | Workflows currently running, with their task association and current phase. |
| **Blocked Items** | Tasks or workflows waiting on an external dependency, approval, or manual input. |
| **Stale Items** | Tasks or workflows that have been in-progress for more than 7 days without progress. |

## Machine-Readable Output

Add `--json` to get structured output safe for scripting or dashboards:

```bash
ao now --json
```

```json
{
  "schema": "ao.cli.v1",
  "ok": true,
  "data": {
    "schema": "ao.now.v1",
    "generated_at": "2026-03-31T09:15:00Z",
    "next_task": {
      "id": "TASK-042",
      "title": "Fix auth token expiry edge case",
      "priority": "High",
      "status": "Backlog",
      "linked_requirements": []
    },
    "active_workflows": [
      {
        "id": "3a521769-...",
        "task_id": "TASK-038",
        "task_title": "Refactor cache layer",
        "status": "Running",
        "current_phase": "implement-rust"
      }
    ],
    "blocked_items": [],
    "stale_items": []
  }
}
```

The inner data schema is `ao.now.v1`. See [JSON Envelope Contract](../reference/json-envelope.md) for envelope details.

## Common Workflows

**Start your day:**

```bash
ao now
```

Check what's next, confirm the daemon is making progress, and spot anything blocked or stale before diving into work.

**Script against it:**

```bash
NEXT=$(ao now --json | jq -r '.data.next_task.id')
echo "Working on $NEXT"
```

**Check daemon health alongside `ao now`:**

```bash
ao daemon status
ao now
```

`daemon status` tells you if the scheduler is running; `ao now` tells you what it's working on.

## Flags

| Flag | Description |
|---|---|
| `--json` | Machine-readable JSON using the `ao.cli.v1` envelope |
| `--project-root <PATH>` | Override project root (also reads `PROJECT_ROOT` env) |
| `-h, --help` | Print help |

## Related Commands

- [`ao daemon status`](daemon-operations.md) — Check if the autonomous scheduler is running
- [`ao task list`](task-management.md) — Full task list with filtering
- [`ao task next`](task-management.md) — Get just the next ready task (no workflow context)
- [`ao errors list`](../reference/cli/index.md) — See recent errors from autonomous runs
