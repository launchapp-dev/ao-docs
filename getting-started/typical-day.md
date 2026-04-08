# A Typical Day Using Animus

This is the common end-to-end loop: define intent, generate requirements,
materialize tasks, and let the daemon execute task workflows.

## The Lifecycle

```mermaid
flowchart TB
    IDEA["Your Idea"]
    --> VISION["ao vision draft<br/>workflow_ref: ao.vision/draft"]
    --> REQS["ao requirements draft<br/>workflow_ref: ao.requirement/draft"]
    --> EXECUTE["ao requirements execute<br/>workflow_ref: ao.requirement/execute"]
    --> DAEMON["ao daemon start --autonomous"]

    DAEMON --> LOOP{"Daemon Tick"}
    LOOP -->|"Dispatch dequeued"| RUNNER["Spawn workflow-runner subprocess"]
    RUNNER --> PIPELINE["Project workflow<br/>often wraps ao.task/standard"]
    PIPELINE -->|"pass"| FACTS["Execution facts"]
    PIPELINE -->|"rework"| PIPELINE
    FACTS --> PROJECTORS["Subject adapters + projectors"]
    PROJECTORS --> DONE["Task / requirement state updated"]
```

## Typical Flow

### 1. Define intent

```bash
animus vision draft
animus requirements draft --include-codebase-scan
animus requirements refine --id REQ-001
```

Canonical refs for those commands are:

- `ao.vision/draft`
- `ao.requirement/draft`
- `ao.requirement/refine`

### 2. Turn requirements into work

```bash
animus requirements execute
```

This runs `ao.requirement/execute`, which plans and materializes task work
through Animus mutation surfaces.

### 3. Let task workflows run

```bash
animus daemon start --autonomous
```

Project-local task refs such as `standard-workflow` usually delegate to bundled
pack refs like `ao.task/standard`.

### 4. Watch the system

```bash
animus task stats
animus workflow list
animus daemon health
animus output tail
```

## Workflow Runs vs. Agent Runs

A **workflow run** (`WF-xxx`) is the full phase pipeline for a task. The daemon
dispatches it and `workflow-runner` executes each phase in sequence.

An **agent run** (`RUN-xxx`) is a single phase within that pipeline. Each phase
spawns one AI CLI process (claude, gemini, etc.) — that process is the agent run.
One workflow run produces many agent runs.

```
Workflow run (WF-001)
  └─ Phase: triage         → Agent run RUN-001  (verdict: advance)
  └─ Phase: research       → Agent run RUN-002  (verdict: advance)
  └─ Phase: implementation → Agent run RUN-003  (verdict: advance)
  └─ Phase: code-review    → Agent run RUN-004  (verdict: rework)
  └─ Phase: implementation → Agent run RUN-005  (verdict: advance)  ← rework retry
  └─ Phase: code-review    → Agent run RUN-006  (verdict: advance)
```

When you watch live output, you are watching an agent run:

```bash
animus output tail --run-id RUN-003
```

When you track overall progress, you are tracking the workflow run:

```bash
animus workflow get --id WF-001
animus workflow decisions --id WF-001
```

## What the Daemon Actually Does

The daemon:

- dequeues `SubjectDispatch` items
- checks capacity
- spawns runner subprocesses
- records execution facts

The daemon does not own task semantics, requirement semantics, or pack logic.

## Why This Matters

That split lets Animus support:

- bundled first-party packs such as `ao.task` and `ao.requirement`
- installed machine packs under `~/.ao/packs/`
- project overrides in `.ao/plugins/`
- subprocess-based Node and Python integrations

without expanding daemon responsibilities.
