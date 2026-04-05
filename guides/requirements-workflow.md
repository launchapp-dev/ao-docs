# Requirements Workflow Guide

Animus provides a structured pipeline for turning project ideas into executable tasks: **Vision -> Requirements -> Tasks**. Each stage narrows scope and adds specificity until work is ready for agents to implement.

## Planning Hierarchy

```
Vision
  |
  v
Requirements (REQ-001, REQ-002, ...)
  |
  v
Tasks (TASK-001, TASK-002, ...)
  |
  v
Workflows (phases: implement, test, review, ...)
```

## Step 1: Draft a Vision

The vision document captures the high-level goals and complexity assessment for your project or initiative.

```bash
animus vision draft
```

This generates a vision document using AI analysis of your project context. The output includes a complexity assessment that informs downstream planning.

Refine the vision iteratively:

```bash
animus vision refine
```

Read the current vision:

```bash
animus vision get
```

## Step 2: Draft Requirements

Requirements bridge the gap between vision and tasks. Draft them with an optional codebase scan so the AI understands existing code:

```bash
animus requirements draft --include-codebase-scan
```

This produces a set of requirements (REQ-001, REQ-002, etc.) with priorities and acceptance criteria.

### Requirement Priorities

Requirements use MoSCoW prioritization:

| Priority | Meaning |
|----------|---------|
| **Must** | Non-negotiable for the current milestone |
| **Should** | Important but not blocking |
| **Could** | Nice to have if time allows |
| **Won't** | Explicitly out of scope for now |

### Requirement Statuses

```
Draft --> Refined --> Planned --> In-Progress --> Done
```

- **Draft** -- Initial AI-generated or manually created requirement
- **Refined** -- Acceptance criteria sharpened and validated
- **Planned** -- Decomposed into tasks
- **In-Progress** -- Linked tasks are being worked on
- **Done** -- All linked tasks completed

## Step 3: Refine Requirements

Sharpen acceptance criteria for specific requirements:

```bash
animus requirements refine --requirement-ids REQ-001
```

Refine multiple at once:

```bash
animus requirements refine --requirement-ids REQ-001 REQ-002 REQ-003
```

Refinement uses AI to analyze the codebase and produce concrete, testable acceptance criteria for each requirement.

## Step 4: Execute Requirements into Tasks

Decompose requirements into actionable tasks:

```bash
animus requirements execute --requirement-ids REQ-001 REQ-002 REQ-003 REQ-004 REQ-005
```

This creates tasks linked to each requirement, with:

- Titles and descriptions derived from the requirement
- Acceptance criteria mapped to task checklists
- Priority inherited from the requirement
- Dependencies inferred between related tasks

## Managing Requirements

List all requirements:

```bash
animus requirements list
```

Get a specific requirement:

```bash
animus requirements get --id REQ-001
```

Create a requirement manually:

```bash
animus requirements create --title "User authentication" --priority must
```

Update a requirement:

```bash
animus requirements update --id REQ-001 --status refined
```

Delete a requirement:

```bash
animus requirements delete --id REQ-001
```

## Requirement Graph

Requirements can have relationships (dependencies, parent-child). View the graph:

```bash
animus requirements graph get
```

## Recommendations

Run an automated scan for improvement recommendations:

```bash
animus requirements recommendations scan
animus requirements recommendations list
animus requirements recommendations apply --id REC-001
```

## How Requirements Link to Tasks

When you run `animus requirements execute`, each requirement produces one or more tasks. These tasks carry a reference back to their parent requirement. As tasks complete, the requirement status updates accordingly.

You can also use the planning facade for a streamlined experience:

```bash
animus planning vision draft
animus planning requirements draft
animus planning requirements refine --requirement-ids REQ-001
animus planning requirements execute --requirement-ids REQ-001
```

The `planning` commands mirror the top-level `vision` and `requirements` commands but are grouped for discoverability.
