# Workflow Execution Reference

How to run, monitor, pause, resume, and cancel workflows. This guide covers phases, decisions, and the verdict system.

## What Is a Workflow?

A workflow is a defined sequence of phases that execute to complete a task. Each phase has:
- An agent that executes (Claude, Codex, Gemini, etc.)
- A directive (what to do)
- Success criteria (what "done" means)

Workflows are defined in YAML and can be project-specific or from installed packs.

---

## Running Workflows

### Async Execution (Daemon)

The standard way to run workflows—non-blocking, managed by the daemon:

```bash
animus workflow run --task-id TASK-001
```

This:
1. Creates a workflow record
2. Queues it for the daemon
3. Returns immediately
4. Daemon executes when capacity allows

### Sync Execution (Blocking)

Run a workflow and wait for completion:

```bash
animus workflow execute --task-id TASK-001
```

This blocks your terminal until the workflow completes. Useful for:
- Debugging
- CI/CD pipelines
- Scripts that need completion status

### Specify Workflow Type

Use a specific workflow definition:

```bash
animus workflow run --task-id TASK-001 --workflow-ref hotfix-workflow
animus workflow run --task-id TASK-001 --workflow-ref standard-workflow
```

### With Input Data

Pass structured input to the workflow:

```bash
animus workflow run --task-id TASK-001 --input-json '{"priority": "urgent", "skip_tests": false}'
```

---

## Monitoring Workflows

### List Workflows

```bash
# All workflows
animus workflow list

# Filter by status
animus workflow list --status running
animus workflow list --status completed
animus workflow list --status failed
animus workflow list --status paused

# Filter by task
animus workflow list --task-id TASK-001

# Filter by workflow type
animus workflow list --workflow-ref standard-workflow
```

### Get Workflow Details

```bash
animus workflow get --id WF-ABC123
```

This shows:
- Current phase
- Phase history
- Status
- Start/end times
- Task linkage

### View Decisions

Workflows make decisions as they execute (advance, rework, skip, fail):

```bash
animus workflow decisions --id WF-ABC123
```

Each decision includes:
- Which phase made it
- What was decided
- Why (reasoning)
- Timestamp

### View Checkpoints

Checkpoints are saved workflow states at phase boundaries:

```bash
animus workflow checkpoints list --id WF-ABC123
```

Useful for:
- Recovery after failures
- Auditing phase transitions
- Debugging unexpected behavior

### Monitor Output

Watch live output from a running workflow:

```bash
# Stream output
animus output monitor --run-id RUN-XYZ789

# Recent output
animus output tail --run-id RUN-XYZ789 --limit 100

# Full output
animus output run --run-id RUN-XYZ789
```

---

## Workflow States

### Status Values

| Status | Meaning |
|--------|---------|
| `pending` | Created, not yet started |
| `running` | Currently executing |
| `paused` | Temporarily halted |
| `completed` | Successfully finished |
| `failed` | Terminated with error |
| `cancelled` | Manually cancelled |

### Phase Verdicts

Within a running workflow, each phase returns a verdict:

| Verdict | Effect |
|---------|--------|
| `advance` | Move to next phase |
| `rework` | Return to a previous phase |
| `skip` | Skip remaining phases |
| `fail` | Terminate workflow as failed |

---

## Controlling Workflows

### Pause a Workflow

Temporarily halt execution:

```bash
animus workflow pause --id WF-ABC123
```

Effects:
- Current phase continues to completion
- No new phases start
- Workflow state is preserved

### Resume a Paused Workflow

```bash
animus workflow resume --id WF-ABC123
```

Continues from where it left off.

### Cancel a Workflow

Permanently terminate:

```bash
animus workflow cancel --id WF-ABC123
```

This requires confirmation:

```bash
animus workflow cancel --id WF-ABC123 --confirmation yes
```

Effects:
- All execution stops
- Task returns to previous status
- Cannot be resumed

### Approve Gate Phases

Some workflows have manual approval gates:

```bash
# Check for pending gates
animus workflow get --id WF-ABC123

# Approve the gate
animus workflow phase approve --workflow-id WF-ABC123
```

With feedback:

```bash
animus workflow phase approve --workflow-id WF-ABC123 --feedback "Looks good, proceed"
```

Specify which phase:

```bash
animus workflow phase approve --workflow-id WF-ABC123 --phase-id po-review
```

---

## Phase Types

### Standard Phases

Agent-executed phases that do work:

```yaml
phases:
  - id: implementation
    agent_role: implementer
    directive: "Implement the feature described in the task"
```

### Command Phases

Execute shell commands:

```yaml
phases:
  - id: build
    type: command
    command: "cargo build --release"
```

### Gate Phases

Require manual approval:

```yaml
phases:
  - id: po-review
    type: gate
    approvers: ["product-owner"]
```

### Sub-Workflow Phases

Delegate to another workflow:

```yaml
phases:
  - id: subtask
    workflow_ref: ao.task/standard
```

---

## Workflow Definitions

### List Available Workflows

```bash
animus workflow definitions list
```

Shows all workflow definitions available to the project.

### View Phase Definitions

```bash
animus workflow phases list
animus workflow phases get --phase implementation
```

### Check Configuration

```bash
# View workflow config
animus workflow config get

# Validate configuration
animus workflow config validate
```

### Update Definitions

```bash
# Create or update a workflow definition
animus workflow definitions upsert --input-json '{
  "id": "custom-workflow",
  "name": "Custom Workflow",
  "phases": [
    {"id": "plan", "agent_role": "planner"},
    {"id": "implement", "agent_role": "implementer"},
    {"id": "test", "agent_role": "tester"}
  ]
}'
```

---

## Common Patterns

### Run and Monitor

```bash
# Start workflow
animus workflow run --task-id TASK-001

# Get the workflow ID from output
WF_ID=$(ao workflow list --task-id TASK-001 --json | jq -r '.data[0].id')

# Monitor until completion
while animus workflow get --id "$WF_ID" --json | jq -e '.data.status == "running"' > /dev/null; do
  echo "Still running..."
  sleep 5
done

echo "Workflow completed"
animus workflow get --id "$WF_ID"
```

### Debug Failed Workflow

```bash
# Find the failed workflow
animus workflow list --status failed --limit 5

# Get details
animus workflow get --id WF-FAILED

# Check decisions
animus workflow decisions --id WF-FAILED

# View output
animus output run --run-id <run-id>

# Check for errors in output
animus output tail --run-id <run-id> --event-types '["stderr"]'
```

### Rerun Failed Task

```bash
# Cancel if still running
animus workflow cancel --id WF-ABC123 --confirmation yes

# Reset task status
animus task status --id TASK-001 --status ready

# Rerun
animus workflow run --task-id TASK-001
```

### Wait for Gate Approval

```bash
# Check for pending gates
animus workflow get --id WF-ABC123 | grep -A5 "pending"

# As approver, approve the gate
animus workflow phase approve --workflow-id WF-ABC123 --phase-id po-review --feedback "Approved"
```

---

## Execution Facts

As workflows execute, they generate "facts" that are projected back to task state.

### Types of Facts

| Fact | Effect |
|------|--------|
| Task status change | Updates task status field |
| Checklist update | Marks checklist items complete |
| Decision recorded | Stored in workflow history |
| Artifact created | Files available via `animus output artifacts` |

### View Artifacts

```bash
# List artifacts from execution
animus output artifacts --execution-id EXEC-001

# Download artifact
animus output download --execution-id EXEC-001 --artifact-id ART-001
```

---

## Workflow Configuration

### Agent Runtime Config

Control which models and tools are used:

```bash
# View current config
animus workflow agent-runtime get

# Update config
animus workflow agent-runtime set --input-json '{
  "agents": {
    "default": {
      "model": "claude-sonnet-4-6",
      "tool": "claude"
    },
    "research": {
      "model": "gemini-2.0-flash",
      "tool": "gemini"
    }
  }
}'
```

### State Machine Config

Control verdict routing and phase transitions:

```bash
# View state machine
animus workflow state-machine get

# Validate
animus workflow state-machine validate
```

---

## Best Practices

### Choosing Async vs Sync

Use **async** (`animus workflow run`) when:
- You want the daemon to manage execution
- Multiple tasks can run in parallel
- You don't need immediate results

Use **sync** (`animus workflow execute`) when:
- You're in a CI/CD pipeline
- You need the result before proceeding
- Debugging a specific task

### Monitoring Running Workflows

```bash
# Quick check
animus workflow list --status running

# Detailed monitoring
watch -n 5 'ao workflow list --status running --json | jq ".data | length"'
```

### Handling Failures

1. **Check the workflow**: `animus workflow get --id WF-XXX`
2. **Review decisions**: `animus workflow decisions --id WF-XXX`
3. **Check output**: `animus output run --run-id RUN-XXX`
4. **Fix the issue** (update code, fix config, etc.)
5. **Rerun**: Reset task and run again

### Using Gates Effectively

- Place gates after risky phases
- Include clear criteria for approvers
- Provide feedback when approving/rejecting
- Don't overuse—too many gates slow progress

---

## Troubleshooting

### Workflow Stuck at Phase

```bash
# Check current phase
animus workflow get --id WF-ABC123

# Is it a gate? Approve it
animus workflow phase approve --workflow-id WF-ABC123

# Is it stuck running? Check agent output
animus output tail --run-id <run-id>

# Still stuck? Cancel and retry
animus workflow cancel --id WF-ABC123 --confirmation yes
```

### Workflow Won't Start

```bash
# Check daemon status
animus daemon status

# Check runner health
animus runner health

# Check task status
animus task get --id TASK-001

# Task must be "ready" for workflow to start
animus task status --id TASK-001 --status ready
```

### Unexpected Rework Loop

Workflows can rework when phases fail checks:

```bash
# Check decisions
animus workflow decisions --id WF-ABC123

# Look for "rework" verdicts and their reasons
# Common causes:
# - Tests failing
# - Lint errors
# - Missing documentation
```

---

## Related Documentation

- [CLI Cheat Sheet](cli-cheat-sheet.md) -- Quick command reference
- [Workflows Concept](../concepts/workflows.md) -- How workflows work
- [Writing Workflows](../guides/writing-workflows.md) -- Creating custom workflows
- [Task Lifecycle](task-lifecycle.md) -- How tasks and workflows interact
- [Daemon Quick Reference](daemon-quick-ref.md) -- Managing the daemon
