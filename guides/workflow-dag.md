# Workflow DAG Visualization

The **DAG** (Directed Acyclic Graph) tab on the Workflow Detail page renders your workflow's phase graph as an interactive diagram. It gives an at-a-glance view of phase ordering, dependencies, and execution state — useful for debugging complex multi-phase workflows and for onboarding teammates to an unfamiliar workflow structure.

This feature was introduced in v57.

---

## Accessing the DAG View

1. Navigate to **Projects → [your project] → Workflows**.
2. Click a workflow name to open its **Workflow Detail** page.
3. Select the **DAG** tab in the tab bar (alongside Overview, Runs, and YAML).

The DAG tab is always available, whether or not the workflow is currently running.

---

## Reading the Graph

Each node in the graph represents a workflow phase. Directed edges represent ordering constraints: an edge from phase A to phase B means phase B will not start until phase A completes.

### Node Anatomy

Each node shows:

| Element | Description |
|---|---|
| Phase name | As declared in the workflow YAML `name` field |
| Agent persona | Icon and label of the assigned agent |
| Status badge | Live status when a run is active (see [Live Execution Mode](#live-execution-mode)) |
| Token estimate | Estimated token cost, shown for completed phases |

### Edge Types

| Edge style | Meaning |
|---|---|
| Solid arrow | Hard dependency — the downstream phase waits for the upstream phase to reach `done` |
| Dashed arrow | Soft dependency — the downstream phase receives output variables from the upstream phase but may start before it completes (parallel execution) |

### Layout

The graph is laid out top-to-bottom by default, with the first phase at the top and the final phase at the bottom. Parallel phases appear side-by-side at the same depth level.

Use the layout controls in the toolbar to switch between:

| Layout | Description |
|---|---|
| **Top-down** (default) | Phases flow from top to bottom in dependency order |
| **Left-right** | Phases flow left to right — useful for wide workflows with many parallel branches |
| **Force-directed** | Physics-based layout — phases spread out to minimise edge crossings; useful for inspecting densely connected graphs |

---

## Interacting with the Graph

### Zoom and Pan

- **Scroll** to zoom in and out.
- **Click and drag** on the background to pan.
- Click **Fit** in the toolbar to reset the viewport so all nodes are visible.
- Press **Cmd+0** (macOS) or **Ctrl+0** (Windows / Linux) to reset zoom to 100%.

### Node Inspection

Click any node to open an inline detail panel on the right side of the canvas:

| Panel section | Content |
|---|---|
| **Phase** | Name, agent persona, and YAML excerpt for this phase |
| **Inputs** | Variables expected as input (from upstream phases or the trigger payload) |
| **Outputs** | Variables produced by this phase and consumed by downstream phases |
| **Tools** | MCP tool names declared in the phase's `tools` list |
| **Latest run** | Status, duration, and token usage from the most recent execution (links to Run Detail) |

Click elsewhere on the canvas or press **Esc** to close the panel.

### Edge Inspection

Click any edge to highlight the upstream and downstream phases it connects and show the variable names passed along that edge in a tooltip.

---

## Live Execution Mode

When the workflow has an active run, the DAG tab automatically enters **live execution mode**:

- Nodes that have completed turn **green** (success) or **red** (failed).
- The currently executing node pulses with a blue ring.
- Pending nodes remain grey.
- A progress bar at the top of the canvas shows overall completion (phases done / total phases).

Live mode polls the run state every 3 seconds. To watch a specific run (rather than the most recent active run), select it from the **Run** dropdown in the toolbar.

### Status Colour Reference

| Colour | Phase status |
|---|---|
| Grey | `pending` — not yet started |
| Blue (pulsing) | `running` — currently executing |
| Green | `done` — completed successfully |
| Red | `failed` — exited with a non-zero status |
| Yellow | `cancelled` — cancelled before completion |
| Orange | `stale` — detected as abandoned after a daemon restart |

---

## Static vs. Live View

The **Run** dropdown in the toolbar controls what state is overlaid on the graph:

| Selection | Description |
|---|---|
| **(none)** | Static view — shows only structure from the workflow YAML; no status overlays |
| **Latest** (default) | Shows status from the most recent run, updated live if the run is active |
| **[specific run ID]** | Shows the final state of that completed run — useful for post-mortem inspection |

Switching to a specific run freezes the status overlay to that run's terminal state. No polling occurs for completed runs.

---

## Exporting the DAG

Click **Export** in the toolbar to download the current graph:

| Format | Description |
|---|---|
| **PNG** | Rasterised image at 2× device pixel ratio — suitable for sharing in documents or issues |
| **SVG** | Vector image — retains text and crisp edges at any scale |
| **JSON** | Raw graph data: nodes with phase metadata and edges with dependency type — useful for scripted analysis |

The export captures the current viewport. Use **Fit** before exporting to ensure all nodes are included.

---

## DAG and the YAML Tab

The DAG visualises the same structure as the **YAML** tab. The two tabs stay in sync: changes pushed via `ao cloud push` are reflected in both tabs on the next page load. There is no way to edit the YAML from the DAG tab directly — use the CLI to modify the workflow definition.

---

## Limitations

- Workflows with more than 50 phases may experience degraded layout performance in force-directed mode. Switch to top-down or left-right layout for large workflows.
- The DAG shows the phase graph as declared in the workflow YAML. Dynamic branching decisions made at runtime by agents are not reflected in the graph structure.
- Parallel phases that have a soft dependency but no explicit `after` ordering constraint will appear disconnected in the graph. Add an explicit `after` declaration in the YAML to make the relationship visible.

---

## Related

- [Writing Custom Workflows](writing-workflows.md) — declaring phases, dependencies, and parallel execution
- [Cloud Dashboard](cloud-dashboard.md) — the full dashboard guide including Run Detail
- [Workflow Run Timeline](cloud-dashboard.md#workflow-run-timeline) — Gantt-style timing view for a specific run
- [GitHub Checks](github-checks.md) — surfacing phase outcomes as GitHub Check Runs
