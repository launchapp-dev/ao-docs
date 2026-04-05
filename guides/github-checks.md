# GitHub Checks

AO Cloud can create and update GitHub Check Runs for every workflow dispatch that originates from a GitHub event. This gives your team live pass/fail feedback directly on pull requests and commits — without writing any custom reporting code.

This guide covers the GitHub Checks integration shipped in v41: the check run lifecycle, dashboard visibility, configuration, and the `github.checks` MCP tool surface.

---

## Prerequisites

- The [GitHub App Integration](github-app.md) installed and at least one repository connected.
- A workflow triggered by a `github` trigger kind. Check runs are attached to the `head_sha` of the triggering event.
- The `github` MCP tool pack installed: `ao pack install github`.

---

## How Check Runs Work

A GitHub Check Run represents a single unit of CI feedback on a specific commit SHA. AO maps each workflow phase to a check run so that reviewers can see phase-level progress in the GitHub UI.

**Lifecycle of a check run created by AO:**

1. Workflow dispatch begins. AO creates a check run with status `in_progress` and the `head_sha` from the trigger payload.
2. Each phase start updates the check run's `output.summary` with the current phase name and elapsed time.
3. On phase completion the run's `output.text` is updated with a summary of the phase output.
4. When the workflow finishes, AO sets the check run status to `completed` with a conclusion of `success`, `failure`, or `cancelled`.

If the daemon stops mid-run (e.g. a crash), AO sets a `stale` annotation on the check run on next daemon start and creates a new run for any re-queued dispatch.

---

## Automatic Check Runs

When the `github_checks` flag is enabled on a project, AO creates a check run automatically for every workflow dispatched by a `github` trigger — no additional workflow YAML required.

### Enabling Automatic Check Runs

1. In the AO Cloud dashboard, go to **Settings → Integrations → GitHub → [installation] → Check Runs**.
2. Toggle **Automatic check runs** on.
3. Select the **check suite name** that will appear in GitHub (default: `AO`).
4. Click **Save**.

Or enable via the CLI:

```bash
ao cloud project config --set github_checks.auto=true
ao cloud project config --set github_checks.suite_name="AO"
```

When enabled, the check suite name appears as a header in the **Checks** tab of every pull request, with child check runs named after the workflow definition (e.g. `AO / pr-review`).

### Disabling for Specific Workflows

To suppress automatic check run creation for a particular workflow, add the `github_checks: false` key to the workflow YAML:

```yaml
# .ao/workflows/internal-cleanup.yaml
name: internal-cleanup
github_checks: false
phases:
  - name: cleanup
    agent: operator
```

---

## Manual Check Run Control

For fine-grained control — custom names, annotations, per-step conclusions — use the `github.checks` MCP tools directly inside a phase.

### Available Tools

| Tool | Description |
|---|---|
| `github.checks.create` | Create a new check run in `in_progress` state |
| `github.checks.update` | Update status, conclusion, output title, summary, or text |
| `github.checks.annotate` | Add file-level annotations (errors, warnings, notices) |
| `github.checks.complete` | Set the final conclusion and close the check run |

### Example: Phase-Level Check Run

```yaml
# .ao/workflows/pr-review.yaml
name: pr-review
phases:
  - name: lint
    agent: code-reviewer
    tools:
      - github.checks.create
      - github.checks.update
      - github.checks.annotate
      - github.checks.complete
    vars:
      repo: "{{ vars.pr_repo }}"
      sha: "{{ vars.head_sha }}"
      check_name: "AO / lint"
```

The agent uses `github.checks.create` at the start of the phase with the name and SHA, then calls `github.checks.annotate` for each issue found, and finally `github.checks.complete` with `success` or `failure`.

### Annotation Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `path` | string | yes | File path relative to repository root |
| `start_line` | integer | yes | First line of the annotated range |
| `end_line` | integer | no | Last line (defaults to `start_line`) |
| `annotation_level` | string | yes | `notice`, `warning`, or `failure` |
| `message` | string | yes | Annotation message text |
| `title` | string | no | Short label for the annotation |
| `raw_details` | string | no | Full context (shown in an expandable section) |

### Example: Annotating Lint Issues

```yaml
# Agent instructions excerpt
After running the linter, call github.checks.annotate for each violation:
- path: the file that contains the violation
- start_line: the line number
- annotation_level: "failure" for errors, "warning" for style issues
- message: the linter message
Then call github.checks.complete with conclusion "failure" if any failures exist,
"success" otherwise.
```

---

## Dashboard: Check Runs Panel

Navigate to **Settings → Integrations → GitHub → [installation] → Check Runs** to see all check runs AO has posted, across all repositories in the installation.

### Table Columns

| Column | Description |
|---|---|
| Repository | `owner/repo` |
| SHA | Commit hash (first 7 characters; click to open on GitHub) |
| Check name | Suite and run name (e.g. `AO / pr-review`) |
| Workflow | AO workflow definition that created the run |
| Status | `in_progress`, `completed`, or `stale` |
| Conclusion | `success`, `failure`, `cancelled`, or `—` while in progress |
| Duration | Wall-clock time from creation to completion |
| Created | Timestamp |

Use the filter bar to narrow by repository, status, or conclusion. Click any row to see the full check run output and a link to the corresponding AO run detail.

### Check Run Status Badges

The dashboard surfaces check run status in two additional places:

- **Project detail → Active workflows table**: A GitHub icon with a coloured badge appears next to any run that has an associated check run.
- **Run detail panel**: A **GitHub Check** section shows the check run name, status, and a direct link to the check run on GitHub.

---

## Re-running Failed Checks

When a check run concludes as `failure`, a **Re-run** button appears on the GitHub checks tab (powered by the GitHub App's check re-run webhook). Clicking it sends a `check_run.rerequested` event to AO, which creates a new dispatch for the same workflow and subject.

The re-run reuses the original `head_sha` and `vars` from the triggering event. To change input variables, dispatch a new run manually via `ao workflow run` or the dashboard.

---

## Troubleshooting

### Check run not appearing on the pull request

- Confirm the `head_sha` in the trigger payload matches the PR's head commit. Use `{{ payload.pull_request.head.sha }}` in the trigger's `dispatch.vars`.
- Verify the GitHub App has `Checks: Read & Write` permission. Revoke and reinstall if the permission is missing.
- Check **Settings → Integrations → GitHub → [installation] → Check Runs** for any `error` status rows; the row detail shows the GitHub API error message.

### Check run stuck in `in_progress`

- The daemon may have stopped before the workflow completed. On daemon restart AO detects orphaned check runs (those open for more than 2 hours with no update) and marks them `stale`.
- The stale check run is visible in the dashboard with status `stale`. A re-run from GitHub or the dashboard creates a fresh run.

### Annotations not appearing

- Annotations are only rendered by GitHub when the check run conclusion is `failure`, `success`, or `neutral` — not while `in_progress`.
- Ensure `github.checks.complete` is called after all `github.checks.annotate` calls.

---

## Related

- [GitHub App Integration](github-app.md) — installing the app and configuring webhook triggers
- [Event Triggers: File Watchers and Webhooks](event-triggers.md) — how GitHub triggers produce dispatches
- [Pack Management](pack-management.md) — installing the `github` tool pack
- [Writing Custom Workflows](writing-workflows.md) — authoring workflows that use check run tools
