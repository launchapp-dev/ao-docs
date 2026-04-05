# Workflows

## Everything Is a Workflow

In Animus, every autonomous operation resolves through a `workflow_ref`. The CLI,
web API, daemon queue, and MCP surfaces all emit a
[SubjectDispatch](./subject-dispatch.md) that points at a workflow definition,
and `workflow-runner` executes the resulting phase plan.

The daemon does not own domain behavior. Workflow behavior comes from bundled
kernel workflows, bundled first-party packs, installed packs, and project-local
overrides.

## Workflow Sources

Animus currently resolves workflows from these sources:

| Source | Typical Refs | What It Owns |
|---|---|---|
| Bundled kernel workflows | `ao.vision/draft`, `ao.vision/refine` | Core planning workflows that still ship with Animus directly |
| Bundled first-party packs | `ao.task/standard`, `ao.requirement/draft`, `ao.requirement/execute` | Task, requirement, review, and QA behavior shipped as pack overlays |
| Installed machine packs | `vendor.pack/ref` | Shared packs installed under `~/.ao/packs/<pack-id>/<version>/` |
| Project pack overrides | `vendor.pack/ref` | Per-project overrides under `.ao/plugins/<pack-id>/` |
| Project-local ad hoc YAML | `standard-workflow`, `incident-response` | Repository-specific workflows in `.ao/workflows.yaml` or `.ao/workflows/*.yaml` |

### Resolution Order

1. Project pack overrides in `.ao/plugins/<pack-id>/`
2. Project-local YAML in `.ao/workflows.yaml` and `.ao/workflows/*.yaml`
3. Installed packs in `~/.ao/packs/<pack-id>/<version>/`
4. Bundled sources embedded in Animus

This means a project can override a bundled or installed workflow without
teaching the daemon any new behavior.

## Canonical Workflow Refs

Pack-qualified refs are now the canonical surface:

| Surface | Canonical Ref | Notes |
|---|---|---|
| `animus vision draft` | `ao.vision/draft` | Legacy alias `builtin/vision-draft` still works |
| `animus vision refine` | `ao.vision/refine` | Legacy alias `builtin/vision-refine` still works |
| `animus requirements draft` | `ao.requirement/draft` | Legacy alias `builtin/requirements-draft` still works |
| `animus requirements refine` | `ao.requirement/refine` | Legacy alias `builtin/requirements-refine` still works |
| `animus requirements execute` | `ao.requirement/execute` | Legacy alias `builtin/requirements-execute` still works |
| Default task workflow | `ao.task/standard` | Project-local refs such as `standard-workflow` can wrap it |

The first-party pack boundary is currently most visible in task, requirement,
review, and QA behavior. For example, task routing and task execution phases now
flow through the bundled `ao.task` and `ao.review` packs instead of living in
the kernel baseline.

## Bundled First-Party Packs

Animus ships with bundled manifests under
`crates/orchestrator-config/config/bundled-packs/`. Today those bundled packs
include:

- `ao.task` for task workflows and task-owned runtime overlays
- `ao.requirement` for requirement planning and execution flows
- `ao.review` for review, QA, and command-phase runtime overlays

These packs can contribute:

- workflow overlays
- phase catalog entries
- runtime overlays
- MCP server descriptors
- runtime requirements
- permissions and secrets policy

## Pack Operations

Operators can inspect and control which packs are active for a project:

```bash
animus pack list
animus pack inspect --pack-id ao.task
animus pack install --path /tmp/vendor.pack --activate
animus pack pin --pack-id ao.task --version =0.1.0
```

Project-specific pack selections are stored in
`.ao/state/pack-selection.v1.json`. Pack override content lives in
`.ao/plugins/`.

## Project-Local Workflow Composition

Project YAML usually wraps canonical pack refs instead of redefining domain
logic:

```yaml
workflows:
  - id: standard-workflow
    name: Standard Workflow
    description: Repository default delivery workflow
    phases:
      - workflow_ref: ao.task/standard

  - id: hotfix-workflow
    name: Hotfix Workflow
    description: Fast-track workflow for urgent fixes
    phases:
      - workflow_ref: ao.task/quick-fix
```

That keeps repository customization local while task and requirement semantics
stay owned by the relevant pack.

## Supported Features

Workflow definitions can combine:

- ordered phase execution
- verdict routing (`advance`, `rework`, `skip`, `fail`)
- sub-workflow composition
- command phases
- manual approval phases
- per-phase MCP bindings
- post-success merge and PR behavior
- pack-owned runtime overlays and policy checks

See [Writing Workflows](../guides/writing-workflows.md) for authoring guidance
and [Subject Dispatch](./subject-dispatch.md) for how workflow refs reach the
runner.
