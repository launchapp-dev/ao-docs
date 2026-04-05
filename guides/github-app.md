# GitHub App Integration

Animus's GitHub App integration connects your GitHub repositories to Animus Cloud directly — no manual webhook secrets, no personal access tokens. After installation, Animus receives GitHub events (push, pull request, issue, and more) via a managed webhook receiver and can act on them through [event triggers](event-triggers.md).

This guide covers the three components shipped in v38:

1. **Installation flow** — authorising the GitHub App on your organisation or personal account.
2. **Repository selector** — choosing which repositories Animus can access.
3. **Webhook receiver** — how incoming GitHub events are validated, routed, and converted to workflow dispatches.

---

## Prerequisites

- An Animus Cloud account. See [Cloud Deployment](cloud-deployment.md) to get started.
- **Admin** role or above on the Animus Cloud organisation. See [Team and Access](cloud-dashboard.md#team-and-access).
- Owner or admin permissions on the target GitHub organisation or personal account.

---

## Installation Flow

### 1. Start the Installation

Navigate to **Settings → Integrations → GitHub** in the Animus Cloud dashboard. Click **Install GitHub App**.

You are redirected to GitHub's app installation page. GitHub prompts you to:

1. Choose an account — your personal GitHub account or any GitHub organisation you administer.
2. Select the installation scope (covered in [Repository Selector](#repository-selector) below).
3. Confirm the requested permissions.

**Permissions requested by the Animus GitHub App:**

| Permission | Scope | Reason |
|---|---|---|
| Contents | Read | Read repository files for workflow context |
| Pull requests | Read & Write | Create, update, and comment on pull requests |
| Issues | Read & Write | Create and update issues from workflow output |
| Checks | Read & Write | Publish check runs for workflow results |
| Statuses | Read & Write | Post commit statuses |
| Webhooks | Read | Confirm webhook delivery |
| Metadata | Read | List repositories (required by GitHub) |

The app does **not** request write access to repository contents, administration settings, or secrets.

### 2. Complete the Installation

After you confirm the permissions, GitHub redirects back to the Animus Cloud dashboard with a temporary `installation_id`. Animus exchanges this for an installation token, stores the mapping, and shows a success state on the Integrations page.

If the redirect fails or times out, return to **Settings → Integrations → GitHub** and click **Retry Installation**. The retry link reuses the existing `installation_id` if GitHub already recorded the installation.

### 3. Verify the Installation

The Integrations page lists each GitHub installation with:

| Field | Description |
|---|---|
| Account | The GitHub organisation or user account name |
| Installation ID | GitHub-assigned identifier (useful for support tickets) |
| Status | `active`, `suspended`, or `revoked` |
| Installed at | Timestamp of the initial installation |
| Repositories | Number of repositories currently selected |

A status of `suspended` means a GitHub organisation owner suspended the app installation on the GitHub side. No events are delivered while suspended. To restore, re-authorise via **Settings → Integrations → GitHub → Resume**.

---

## Repository Selector

The repository selector controls which repositories Animus can receive events from and access via the GitHub API. Limiting scope here minimises the blast radius if credentials are compromised.

### Selecting Repositories During Installation

During the GitHub installation flow, choose one of two modes:

| Mode | Description |
|---|---|
| **All repositories** | Animus receives events from every repository in the account now and any added in the future |
| **Selected repositories** | Animus receives events only from the repositories you explicitly choose |

**Recommended:** Start with selected repositories. You can expand the selection later.

### Changing Repository Access After Installation

1. Go to **Settings → Integrations → GitHub** in the Animus Cloud dashboard.
2. Click the **edit** icon next to the installation you want to update.
3. Click **Configure on GitHub**. You are taken to the GitHub app settings page for this installation.
4. Under **Repository access**, add or remove repositories.
5. Click **Save**. GitHub sends an `installation_repositories` event to Animus; the dashboard updates within a few seconds.

Alternatively, update access directly on GitHub at:
`https://github.com/settings/installations/<installation_id>`

### Repository List in the Dashboard

After installation, the **Settings → Integrations → GitHub → [installation] → Repositories** tab shows the repositories Animus currently has access to:

| Column | Description |
|---|---|
| Repository | `owner/repo` slug; click to open on GitHub |
| Visibility | `public` or `private` |
| Language | Primary language detected by GitHub |
| Topics | Up to three GitHub topic tags |
| Added | When the repository was added to the installation |
| Events | Count of webhook events received in the last 7 days |
| Triggers | Count of `.ao/triggers.yaml` webhook triggers bound to this repository |

### Searching and Filtering Repositories

The repository list includes a search and filter bar (added in v39):

- **Search box** — type any fragment of a repository name for instant filtering. Matches on the `owner/repo` slug, not the display name.
- **Language filter** — select one or more programming languages to show only repositories with that primary language.
- **Topic filter** — enter a GitHub topic tag (e.g. `microservice`, `frontend`) to filter by topic. Repositories with no topics are shown when the filter is empty.
- **Visibility toggle** — show `public`, `private`, or all repositories.
- **Sort** — order by `Name`, `Added`, or `Events (last 7 days)` in ascending or descending order.

Filters are combined with AND logic. The active filter state is preserved in the URL so you can bookmark a filtered view or share it with a teammate.

### Bulk Repository Management

Select multiple repositories using the checkboxes on the left of each row. With one or more repositories selected, a context toolbar appears with:

| Action | Description |
|---|---|
| **Remove from installation** | Removes the selected repositories from this installation; Animus stops receiving events from them |
| **View triggers** | Opens a combined trigger list scoped to the selected repositories |
| **Export** | Downloads the selected repository list as CSV |

Removing repositories from Animus does not uninstall the GitHub App — it removes them from the installation's repository access list only. Re-add them at any time via **Configure on GitHub**.

---

## Repository Activation

The GitHub App installation grants Animus access to a set of repositories at the account level. The **repository activation** step is a per-project control that determines which of those accessible repositories are actively monitored within a specific Animus Cloud project. A repository must be both accessible (via the installation) and activated (within a project) before triggers bound to it can fire.

### Activating a Repository

1. Open your project in the Animus Cloud dashboard.
2. Navigate to **Settings → Repositories**.
3. The **Available Repositories** list shows all repositories accessible through your GitHub App installations. Repositories not yet activated show a greyed-out **Activate** toggle.
4. Click **Activate** next to the repository you want to enable. Animus registers the repository with the project and begins routing matching GitHub events to the project's trigger engine.

Only one project can be the primary handler for a given repository-event combination, but the same repository can be activated in multiple projects. Events are routed to all projects where the repository is activated and a matching trigger exists.

### Deactivating a Repository

1. Navigate to **Settings → Repositories** within the project.
2. Click **Deactivate** next to the activated repository.
3. Confirm the action. Animus stops routing new events from this repository to the project's trigger engine. In-progress runs are not interrupted.

Deactivating does not affect other projects that have the same repository activated.

### Activation Status

The **Settings → Repositories** table shows:

| Column | Description |
|---|---|
| Repository | `owner/repo` slug |
| Installation | Which GitHub App installation provides access |
| Status | `active` (routing events) or `inactive` (not routing events) |
| Triggers | Number of triggers in this project bound to this repository |
| Events (7 d) | GitHub events received and routed to this project in the last 7 days |
| Activated at | When the repository was activated in this project |

### Activating via CLI

```bash
# List repositories available for activation in the current project
animus cloud repo list

# Activate a repository
animus cloud repo activate my-org/my-repo

# Deactivate a repository
animus cloud repo deactivate my-org/my-repo
```

The `animus cloud repo list` command shows both accessible and activated repositories:

```
REPOSITORY              INSTALLATION   STATUS    TRIGGERS   EVENTS (7D)
my-org/my-repo          my-org         active    3          47
my-org/other-repo       my-org         inactive  0          —
my-org/third-repo       my-org         active    1          12
```

---

## Webhook Receiver

Animus Cloud runs a managed webhook receiver that GitHub posts events to. You do not need to manage a webhook URL or signing secret — the GitHub App integration handles this automatically.

### How Events Are Delivered

1. A GitHub event occurs (push, pull request opened, issue commented, etc.).
2. GitHub posts a signed webhook payload to Animus's receiver endpoint.
3. Animus validates the payload signature using the installation's webhook secret (managed internally; never exposed to users).
4. Animus parses the event, looks up the matching Animus Cloud organisation and project by `installation_id`, and emits an internal `github.*` event.
5. The trigger engine evaluates `.ao/triggers.yaml` entries in each project that has the originating repository configured.
6. Matching triggers create a `SubjectDispatch` entry, which the daemon picks up as a normal workflow run.

### Supported GitHub Event Types

| GitHub event | Animus event name | Description |
|---|---|---|
| `push` | `github.push` | Commits pushed to any branch |
| `pull_request` | `github.pull_request` | PR opened, synchronised, closed, or reopened |
| `pull_request_review` | `github.pull_request_review` | Review submitted |
| `pull_request_review_comment` | `github.pull_request_review_comment` | Inline comment added |
| `issues` | `github.issues` | Issue opened, closed, edited, or labelled |
| `issue_comment` | `github.issue_comment` | Comment added to an issue or PR |
| `check_run` | `github.check_run` | Check run created or completed |
| `status` | `github.status` | Commit status updated |
| `release` | `github.release` | Release published or edited |
| `workflow_run` | `github.workflow_run` | GitHub Actions workflow completed |

Events for repositories not in the installation's selected set are silently dropped before trigger evaluation.

### Configuring Triggers for GitHub Events

Declare a `github` trigger in `.ao/triggers.yaml`:

```yaml
# .ao/triggers.yaml
triggers:
  - id: on-pr-opened
    kind: github
    github:
      installation: my-org          # GitHub account name from the installation
      repository: my-org/my-repo    # optional — omit to match all repos
      events:
        - pull_request.opened
        - pull_request.synchronize
    dispatch:
      workflow_ref: ao.task/standard
      subject:
        kind: ao.task
        id: "pr-{{ payload.pull_request.number }}"
        title: "Review PR #{{ payload.pull_request.number }}: {{ payload.pull_request.title }}"
      vars:
        pr_number: "{{ payload.pull_request.number }}"
        base_branch: "{{ payload.pull_request.base.ref }}"
        head_sha: "{{ payload.pull_request.head.sha }}"
```

### GitHub Trigger Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique trigger identifier |
| `kind` | string | yes | Must be `github` |
| `github.installation` | string | yes | GitHub account name (org or user) from the installed app |
| `github.repository` | string | no | `owner/repo` slug; omit to match all repos in the installation |
| `github.events` | string[] | yes | One or more `<event>.<action>` strings (e.g. `push`, `pull_request.opened`) |
| `github.branches` | string[] | no | For push events — glob patterns matching the ref (e.g. `main`, `release/*`) |
| `dispatch.workflow_ref` | string | yes | Workflow to execute |
| `dispatch.subject` | SubjectRef | yes | Subject identity; supports `{{ payload.* }}` interpolation |
| `dispatch.vars` | map\<string, string\> | no | Variables passed to the workflow; supports `{{ payload.* }}` interpolation |
| `dispatch.priority` | string | no | Queue priority hint |

### Payload Interpolation

`{{ payload.* }}` expressions in `dispatch.subject` and `dispatch.vars` are evaluated against the raw GitHub webhook payload. The payload structure matches the [GitHub Webhooks documentation](https://docs.github.com/en/webhooks/webhook-events-and-payloads) for each event type.

Examples:

| Expression | GitHub event | Value |
|---|---|---|
| `{{ payload.ref }}` | `push` | `refs/heads/main` |
| `{{ payload.pull_request.number }}` | `pull_request` | `42` |
| `{{ payload.pull_request.head.sha }}` | `pull_request` | `abc123def...` |
| `{{ payload.issue.title }}` | `issues` | `Bug: login fails on Safari` |
| `{{ payload.sender.login }}` | any | GitHub username of the actor |
| `{{ payload.repository.full_name }}` | any | `owner/repo` |

Missing keys are replaced with an empty string. Nested paths use dot notation.

### Branch Filtering

For `push` events, use `github.branches` to restrict which branches trigger a dispatch:

```yaml
triggers:
  - id: on-main-push
    kind: github
    github:
      installation: my-org
      repository: my-org/my-repo
      events:
        - push
      branches:
        - main
        - release/*
    dispatch:
      workflow_ref: ao.task/standard
      subject:
        kind: ao.task
        id: "post-push-{{ payload.after }}"
        title: "Post-push review on {{ payload.ref }}"
```

Branch patterns use the same glob syntax as [file watcher paths](event-triggers.md#glob-pattern-rules).

---

## Webhook Delivery Log

Every inbound GitHub webhook is logged in the Animus Cloud dashboard at **Settings → Integrations → GitHub → [installation] → Deliveries**.

| Column | Description |
|---|---|
| Delivery ID | GitHub's unique delivery identifier |
| Event | GitHub event type (e.g. `pull_request`) |
| Action | Event action sub-type (e.g. `opened`) |
| Repository | Source repository (`owner/repo`) |
| Status | `accepted`, `filtered`, or `error` |
| Dispatches | Number of workflow dispatches triggered by this event |
| Received at | Timestamp |

**Status values:**

| Status | Meaning |
|---|---|
| `accepted` | Signature valid; trigger evaluation ran |
| `filtered` | Signature valid; no trigger matched (repository not selected or no matching trigger) |
| `error` | Signature validation failed or an internal error occurred |

Click any delivery row to see the full payload (truncated to 16 KB for display) and the list of dispatches created. Click a dispatch ID to open the corresponding Run Detail panel.

---

## Posting Results Back to GitHub

Workflows triggered by GitHub events can write check runs, commit statuses, or PR comments back to GitHub using the `github` MCP tool pack (installed separately via `animus pack install github`).

Example workflow phase posting a check run:

```yaml
# .ao/workflows/pr-review.yaml
name: pr-review
phases:
  - name: review
    agent: code-reviewer
    tools:
      - github.create_check_run
      - github.update_check_run
    vars:
      repo: "{{ vars.pr_repo }}"
      sha: "{{ vars.head_sha }}"
      pr_number: "{{ vars.pr_number }}"
```

The `github` pack uses the installation token for the repository's installation — no additional credentials are required. Token rotation is handled automatically by the Animus Cloud backend.

---

## Revoking the Integration

To remove the GitHub App integration from your Animus Cloud organisation:

1. Go to **Settings → Integrations → GitHub**.
2. Click **Revoke** next to the installation.
3. Confirm the action. Animus deletes the installation token and stops accepting events for this installation.

This does **not** uninstall the GitHub App from your GitHub account. To fully uninstall, also go to `https://github.com/settings/installations` (or your organisation's equivalent) and click **Uninstall** next to the Animus app.

---

## Troubleshooting

### No events appearing in the Delivery Log

- Confirm the repository is listed under **Settings → Integrations → GitHub → [installation] → Repositories**.
- Check that the GitHub App installation status is `active` (not `suspended`).
- On GitHub, go to `https://github.com/settings/installations/<id>` and verify the app is still installed with the expected permissions.

### Trigger fires but workflow does not start

- Run `animus trigger list` and confirm the trigger status is `active`, not `paused`.
- Check `animus daemon status` — the daemon must be running to process dispatches.
- Review the dispatch in **Settings → Integrations → GitHub → [installation] → Deliveries** and click through to the Run Detail to see any startup errors.

### `filtered` status for all deliveries

- The trigger's `github.repository` field must exactly match the repository's `owner/repo` slug (case-sensitive).
- The `github.events` list must include the specific event and action. Use `pull_request.opened` rather than `pull_request` if you want to limit to the opened action.
- If `github.branches` is set, verify the push ref matches one of the patterns.

### Installation redirect fails

GitHub's OAuth redirect has a 10-minute window. If the redirect URL expires, start the installation again from **Settings → Integrations → GitHub → Install GitHub App**.

---

## Related

- [Event Triggers: File Watchers and Webhooks](event-triggers.md) — general trigger documentation
- [Writing Custom Workflows](writing-workflows.md) — authoring workflows dispatched by triggers
- [Cloud Dashboard](cloud-dashboard.md) — dashboard navigation and webhook delivery settings
- [Pack Management](pack-management.md) — installing the `github` tool pack
- [Subject Dispatch](../concepts/subject-dispatch.md) — how all triggers produce a unified work envelope
