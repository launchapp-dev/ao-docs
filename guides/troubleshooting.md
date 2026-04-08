# Troubleshooting Guide

Common issues and their fixes when working with Animus.

## Environment Diagnostics

Run the built-in doctor command first:

```bash
animus doctor
```

This checks for required tools, API keys, configuration files, and common misconfigurations. Use `--fix` to attempt automatic repairs:

```bash
animus doctor --fix
```

## Daemon Won't Start

**Symptoms**: `animus daemon start --autonomous` exits immediately or `animus daemon status` shows not running.

**Steps**:

1. Check daemon status:
   ```bash
   animus daemon status
   ```

2. Read the daemon log:
   ```bash
   animus daemon logs
   ```
   Or read the log file directly:
   ```bash
   cat .ao/daemon.log
   ```

3. Try running in foreground for immediate error output:
   ```bash
   animus daemon run
   ```

4. Check if another daemon instance is already running (port conflict).

## CLAUDECODE Environment Variable Blocking claude CLI

**Symptoms**: Inside a Claude Code session, the daemon fails to start agents with an error like "Cannot be launched inside another Claude Code session".

**Cause**: Claude Code sets `CLAUDECODE=1` in the environment. The `claude` CLI refuses to run when it detects this variable.

**Fix**: Unset the variable before starting the daemon:

```bash
unset CLAUDECODE
animus daemon start --autonomous
```

Or use an env prefix:

```bash
env -u CLAUDECODE animus daemon start --autonomous
```

## Agent Runtime Config Overriding Compiled Defaults

**Symptoms**: The daemon uses unexpected models. For example, all phases use the same model instead of routing research to gemini.

**Cause**: The agent runtime config at `.ao/state/agent-runtime-config.v2.json` has explicit `model` and `tool` fields that override the compiled routing table.

**Fix**: Set the fields to `null` to let compiled defaults take over:

```bash
animus workflow agent-runtime get    # Inspect current config
```

Then update:

```bash
animus workflow agent-runtime set --input-json '{"agents":{"default":{"model":null,"tool":null}}}'
```

Or read the config cascade documentation in the [Model Routing Guide](model-routing.md).

## Paused / Ghost Task State

**Symptoms**: A task shows as blocked or paused but should be ready. The daemon skips it.

**Cause**: When tasks are blocked, `paused=true` is set internally. The daemon skips paused tasks during scheduling. Direct JSON edits or incomplete status transitions can leave ghost state.

**Fix**: Always reset via `animus task status` which clears all blocking metadata:

```bash
animus task status --id TASK-XXX --status ready
```

This clears `paused`, `blocked_at`, `blocked_reason`, and `blocked_by`. Never hand-edit `.ao/*.json` files.

## Runner Connection Issues

**Symptoms**: Workflows fail with runner connection errors. Agents do not start.

**Steps**:

1. Check runner health:
   ```bash
   animus runner health
   ```

2. Detect orphaned runner processes:
   ```bash
   animus runner orphans detect
   ```

3. Clean up orphans if found:
   ```bash
   animus runner orphans cleanup
   ```

4. Check restart statistics:
   ```bash
   animus runner restart-stats
   ```

5. Verify API keys are set:
   ```bash
   animus model status
   ```

## Build Cache Stale

**Symptoms**: After editing protocol types or model routing, `cargo build` does not pick up changes.

**Cause**: Cargo's incremental compilation may not detect changes in certain files, particularly in the `protocol` crate.

**Fix**: Touch the changed file to force recompilation:

```bash
touch crates/protocol/src/model_routing.rs
cargo build -p orchestrator-cli
```

## Daemon Log Location

The daemon writes logs to `.ao/daemon.log` relative to the project root. Log rotation occurs at 10MB, with the rotated file at `.ao/daemon.log.1`.

Read logs:

```bash
animus daemon logs
```

Clear logs:

```bash
animus daemon clear-logs
```

Tail in real time:

```bash
tail -f .ao/daemon.log
```

## Workflow Stuck or Failed

**Steps**:

1. List workflows to find the problematic one:
   ```bash
   animus workflow list
   ```

2. Inspect the workflow:
   ```bash
   animus workflow get --id WF-001
   ```

3. Check decisions made during execution:
   ```bash
   animus workflow decisions --id WF-001
   ```

4. View the agent output:
   ```bash
   animus output run --id RUN-001
   ```

5. If the workflow is stuck, you can cancel and retry:
   ```bash
   animus workflow cancel --id WF-001
   animus task status --id TASK-XXX --status ready
   ```

## Missing API Keys

**Symptoms**: Agents fail to start with authentication errors.

**Check**:

```bash
animus model status
```

Required keys by tool:

| Tool | Environment Variable |
|------|---------------------|
| `claude` | `ANTHROPIC_API_KEY` |
| `codex` | `OPENAI_API_KEY` |
| `gemini` | `GEMINI_API_KEY` or `GOOGLE_API_KEY` |
| `oai-runner` | `MINIMAX_API_KEY`, `ZAI_API_KEY`, or `OPENAI_API_KEY` |

## Gemini Redirected on Write Phases

**Symptoms**: Research phases that use gemini get redirected to claude even though they do not need write access.

**Cause**: `enforce_write_capable_phase_target` redirects non-write-capable tools by default.

**Fix**:

```bash
export AO_ALLOW_NON_EDITING_PHASE_TOOL=true
```

---

## Agent-Runner Pitfalls

### Runner Exits Immediately After Launch

**Symptoms**: `animus runner health` shows unhealthy immediately after daemon start. Agent phases never begin.

**Cause**: The runner binary may not be on `PATH`, the socket file from a previous run was not cleaned up, or the runner received a conflicting port binding.

**Steps**:

1. Confirm the runner binary is reachable:
   ```bash
   which animus-runner
   ```
2. Check for a stale socket or PID file:
   ```bash
   ls -la .ao/run/
   ```
   Delete stale files if a previous run crashed:
   ```bash
   rm -f .ao/run/runner.sock .ao/run/runner.pid
   ```
3. Restart the daemon:
   ```bash
   animus daemon restart
   ```

### Orphaned Runner Processes Accumulating

**Symptoms**: Repeated agent failures, increasing memory usage, `animus runner restart-stats` shows a high restart count.

**Cause**: Each failed workflow phase can leave a runner process behind if the daemon loses contact with it before the cleanup signal is sent (crash, SIGKILL, or network interruption).

**Steps**:

1. List orphans:
   ```bash
   animus runner orphans detect
   ```
2. Clean them up:
   ```bash
   animus runner orphans cleanup
   ```
3. If the cleanup command itself hangs, find and kill the processes manually:
   ```bash
   pgrep -a animus-runner
   kill <PID>
   ```

### Agent Phase Never Starts (Queued Forever)

**Symptoms**: A workflow shows its phase as `pending` or `ready` indefinitely. No agent output appears.

**Cause**: The runner's concurrency slot is saturated by another long-running phase, or the daemon's internal queue is paused.

**Steps**:

1. Check how many phases are actively running:
   ```bash
   animus runner health
   ```
2. Check the daemon queue state:
   ```bash
   animus daemon status
   ```
3. If the daemon is in drain mode, resume it:
   ```bash
   animus daemon resume
   ```
4. If a previous phase is stuck, cancel its workflow and free the slot:
   ```bash
   animus workflow cancel --id WF-XXX
   ```

### Runner Uses Wrong Model After Config Change

**Symptoms**: An agent runs with a stale model even after updating the agent runtime config.

**Cause**: The runner caches its agent configuration at launch time. Changes to `.ao/state/agent-runtime-config.v2.json` are not picked up until the runner restarts.

**Fix**: Restart the daemon after any agent runtime config change:

```bash
animus daemon restart
```

---

## Decision Contract Pitfalls

Decision contracts are the JSON payload every phase must emit. When the payload is malformed or missing required fields, the workflow engine cannot route the phase and treats it as a failure.

### Phase Fails with "Missing required field: verdict"

**Symptoms**: A phase ends and the workflow immediately marks it failed. The daemon log contains a validation error mentioning `verdict`.

**Cause**: The agent's output did not include a `verdict` field, or it was nested under an unexpected key. The phase decision envelope must include `verdict` at the top level.

**Required envelope fields**:

| Field | Type | Description |
|-------|------|-------------|
| `verdict` | string | `advance`, `rework`, `skip`, or `fail` |
| `reason` | string | Short explanation for the verdict |
| `confidence` | number | 0.0–1.0 confidence in the decision |
| `risk` | string | `low`, `medium`, or `high` |
| `evidence` | array | Concrete supporting evidence |

**Fix**: Update the phase's `system_prompt` to instruct the agent to emit all required fields. See [Phase Contracts](../architecture/phase-contracts.md) for the full schema.

### Unknown Verdict Value Silently Ignored

**Symptoms**: A phase finishes but the workflow does not advance, and no error is logged.

**Cause**: An unrecognised verdict string (e.g. `"approved"`, `"complete"`) is deserialized as the internal `Unknown` variant. The workflow engine has no routing rule for it and drops the phase.

**Fix**: Use only the four accepted values: `advance`, `rework`, `skip`, `fail`. Check the phase output:

```bash
animus output run --id RUN-XXX
```

### Phase-Local Fields Ignored

**Symptoms**: A custom field added to the phase decision (e.g. `skip_reason`) is never used for routing or downstream context injection.

**Cause**: Phase-local fields must be declared in the YAML `phase_definitions` block for the runtime to recognise and persist them. Undeclared fields are parsed but not stored as structured artifacts.

**Fix**: Declare each phase-local field under `phase_definitions` in your workflow YAML:

```yaml
phase_definitions:
  triage:
    emits: decision
    fields:
      skip_reason:
        type: string
        required: false
        description: "Reason for skipping: already_done, duplicate, no_longer_valid, or out_of_scope."
      recommended_task_status:
        type: string
        required: false
        enum: [done, cancelled]
        description: "Suggested terminal status when verdict is skip."
```

### Confidence Value Out of Range

**Symptoms**: Workflow validation rejects the phase output with a range error.

**Cause**: `confidence` must be a float between `0.0` and `1.0`. Agents occasionally emit values like `0.95%` (string) or `95` (integer percentage).

**Fix**: Ensure your system prompt explicitly states that `confidence` must be a decimal between 0 and 1, not a percentage. Example prompt language:

> "Emit confidence as a decimal between 0.0 and 1.0. Do not use percentages."

---

## Rework Loop Pitfalls

### Workflow Fails After Hitting the Rework Limit

**Symptoms**: A workflow terminates with "max rework attempts exhausted" before the code is ready.

**Cause**: The default `max_rework_attempts` for the implementation phase is typically 3. If a reviewer repeatedly returns `rework` without the engineer resolving the root cause, the counter is exhausted and the workflow fails.

**Common root causes**:
- The reviewer's feedback is vague and the engineer cannot act on it.
- The system prompt for the engineer lacks sufficient context about the codebase.
- A test command in the workflow always fails due to an environmental problem unrelated to the code change.

**Steps**:

1. Inspect the rework history for the workflow:
   ```bash
   animus workflow decisions --id WF-XXX
   ```
2. Read the agent output for the last failed implementation phase to understand what the engineer attempted:
   ```bash
   animus output run --id RUN-XXX
   ```
3. If the environment is the issue (e.g. flaky tests), fix the environment first, then re-queue:
   ```bash
   animus task status --id TASK-XXX --status ready
   ```
4. If the rework limit is too low for complex tasks, raise it in your workflow YAML:
   ```yaml
   phases:
     - id: implementation
       agent: senior-engineer
       max_rework_attempts: 5
   ```

### Rework Loop Never Terminates (No Limit Set)

**Symptoms**: A workflow phase cycles indefinitely between implementation and review. The daemon log shows the same phases repeating.

**Cause**: `max_rework_attempts` was not set, and the workflow YAML routes `rework` back to `implementation` unconditionally without a guard.

**Fix**: Always set `max_rework_attempts` on phases that can receive rework:

```yaml
phases:
  - id: implementation
    agent: senior-engineer
    max_rework_attempts: 3
  - id: code-review
    agent: code-reviewer
    on_verdict:
      rework: { target: implementation }
      advance: { target: testing }
```

Without `max_rework_attempts`, the workflow engine will not automatically break the cycle.

### Rework Context Not Passed to the Engineer

**Symptoms**: The engineer agent repeats the same fix attempt that the reviewer already rejected. The reviewer's specific feedback does not appear in the engineer's phase.

**Cause**: The workflow is not configured to inject prior phase output into the rework phase. The engineer starts the rework phase with only its original context.

**Fix**: Verify that the workflow is using the standard `ao.task/standard` pipeline, which injects reviewer feedback automatically. For custom workflows, add a `context_from` directive to pull in the previous phase decision:

```yaml
phases:
  - id: implementation
    agent: senior-engineer
    context_from: [code-review]
```

### Rework Counter Resets Unexpectedly

**Symptoms**: A phase that should have exhausted its rework limit continues executing past the configured maximum.

**Cause**: Cancelling and re-queuing a task resets the workflow, including rework counters, because a new workflow execution is created. This is expected behavior — the counter tracks attempts within a single workflow run, not across all historical runs.

If you need to prevent re-queuing after repeated failures, set the task to `on-hold` while you diagnose the issue:

```bash
animus task status --id TASK-XXX --status on-hold --reason "Investigating repeated rework failures"
```

---

## Task Status Transition Pitfalls

### Task Stuck in `in-progress` After Workflow Cancellation

**Symptoms**: A workflow is cancelled but the associated task remains `in-progress`. The daemon does not pick it up again.

**Cause**: Cancelling a workflow does not automatically revert the task status. The task was moved to `in-progress` when the workflow started, and cancellation only affects the workflow record.

**Fix**: Manually reset the task status after cancelling:

```bash
animus workflow cancel --id WF-XXX
animus task status --id TASK-XXX --status ready
```

To cancel and reset in one step using the task command:

```bash
animus task cancel-workflow --id TASK-XXX --requeue
```

### Task Skipped by Daemon After Manual Status Edit

**Symptoms**: You edited a task's status directly in `.ao/tasks.json`, but the daemon still skips it.

**Cause**: The `.ao/*.json` files are the authoritative store, but the daemon holds a cached in-memory view. Direct file edits do not notify the daemon to refresh its view. Additionally, manual JSON edits can leave residual fields (`paused`, `blocked_at`, `blocked_by`) that the daemon checks independently of the `status` field.

**Fix**: Always use the CLI to change task status:

```bash
animus task status --id TASK-XXX --status ready
```

This writes through the daemon's update path, clears all blocking metadata, and flushes the daemon's cache. Never hand-edit `.ao/*.json` files.

### `blocked` Status Not Cleared After Dependency Resolves

**Symptoms**: A task was blocked on a dependency that is now `done`, but the blocked task remains in `blocked` status and is not picked up.

**Cause**: Animus does not automatically unblock tasks when their dependencies complete. The transition from `blocked` to `ready` must be performed explicitly.

**Fix**: Once the blocking dependency is resolved, unblock the task:

```bash
animus task status --id TASK-XXX --status ready
```

To find all currently blocked tasks:

```bash
animus task list --status blocked
```

### Task Status Mismatch Between CLI and Dashboard

**Symptoms**: The CLI reports a task as `ready`, but the web dashboard shows it as `in-progress` (or vice versa).

**Cause**: The dashboard reads from the same `.ao/` state files, but it may be displaying a cached view. A reload is usually sufficient. If the mismatch persists, the daemon's in-memory state diverged from disk — typically caused by a crash during a write.

**Steps**:

1. Reload the dashboard.
2. Run:
   ```bash
   animus task get --id TASK-XXX
   ```
   to confirm the CLI-reported status.
3. If the status is genuinely wrong, reset it:
   ```bash
   animus task status --id TASK-XXX --status <correct-status>
   ```
4. If the daemon is in an inconsistent state, restart it:
   ```bash
   animus daemon restart
   ```

### Transition to `done` Rejected by the Daemon

**Symptoms**: Running `animus task status --id TASK-XXX --status done` returns an error like "invalid transition from blocked".

**Cause**: The task state machine enforces valid transitions. Moving directly from `blocked` or `on-hold` to `done` is not allowed — the task must pass through `in-progress` or `ready` first.

**Valid transition paths**:

```
backlog  → ready → in-progress → done
backlog  → ready → in-progress → blocked → ready → in-progress → done
in-progress → cancelled
```

**Fix**: Reset the task to `ready` first, then mark it done:

```bash
animus task status --id TASK-XXX --status ready
animus task status --id TASK-XXX --status done
```
