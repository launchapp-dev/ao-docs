# Cloud Dashboard Guide

The AO cloud dashboard is a React web application hosted at `app.ao.dev`. It provides a browser-based view of all your cloud projects: deployment status, live agent activity, log browsing, and billing — without requiring the CLI.

For CLI-based cloud management see [Cloud Deployment](cloud-deployment.md). For billing details see [Cloud Billing](cloud-billing.md).

---

## Accessing the Dashboard

Navigate to `app.ao.dev` and sign in with the same credentials used by `ao cloud login`. After authentication you land on the **Projects** overview.

If your account belongs to multiple organisations, a selector in the top-left corner switches context. All data — projects, deployments, logs — is scoped to the active organisation.

---

## Navigation Structure

The left sidebar contains five top-level sections:

| Section | Purpose |
|---|---|
| **Projects** | List of all projects in the active organisation |
| **Deployments** | Deployment history and configuration details |
| **Agents** | Live and historical agent run activity |
| **Logs** | Full-text log search across all runs |
| **Billing** | Subscription plan, usage metrics, invoices, and payment methods |

---

## Projects

The Projects page shows every project pushed to the organisation. Each row displays:

- Project name and slug
- Active environment (`production`, `staging`, or `preview`)
- Daemon status badge (`running`, `stopped`, `error`, `starting`)
- Last push timestamp
- Quick-action buttons: **Start**, **Stop**, **Push** (opens a confirmation modal)

Click a project name to open the **Project Detail** view, which shows:

- Deployment timeline — a chronological list of pushes with artifact counts
- Environment tabs — switch between `production`, `staging`, and any active `preview` environments
- Daemon uptime graph — last 30 days of uptime as a stacked bar chart
- Active workflow table — workflows currently executing, with phase progress bars

---

## Deployments

The Deployments page lists every `ao cloud push` event across all projects. Columns:

| Column | Description |
|---|---|
| Deployment ID | Unique identifier (matches the `deployment_id` returned by `ao cloud push --json`) |
| Project | Project the deployment belongs to |
| Environment | `production`, `staging`, or `preview` |
| Artifacts | Number of uploaded workflow definitions, personas, and pack configs |
| Pushed At | Timestamp of the push |
| Status | `active` (currently live) or `superseded` (replaced by a later push) |

Click a deployment to view its artifact manifest and a diff against the previous deployment.

---

## Agents

The Agents page shows agent runs across all projects in real time.

### Live Runs

The **Live** tab lists every agent run with status `starting` or `running`. The table auto-refreshes every 5 seconds:

| Column | Description |
|---|---|
| Run ID | Unique run identifier (use with `ao cloud logs --run-id`) |
| Project | Source project |
| Workflow | Workflow definition name |
| Phase | Currently executing phase |
| Started | Duration since the run started |
| Tokens | Estimated token usage for the current run |

### Run Detail

Click a run ID to open the Run Detail panel:

- **Phase timeline** — sequential list of phases with status icons (`pending`, `running`, `done`, `failed`)
- **Live output** — streaming text output from the current phase (powered by SSE)
- **Artifacts** — files created or modified by the run, once the run completes
- **Token usage** — per-phase breakdown of input and output tokens

### History

The **History** tab shows completed runs. Filter by project, workflow, date range, or status. Completed run detail panels include the full phase timeline and all artifacts.

---

## Logs

The Logs page provides full-text search across all log lines emitted by the cloud daemon and agent runs.

### Search Bar

Type any string to filter log lines. Results update as you type. The search scope defaults to the last 24 hours; adjust with the time-range picker.

### Filters

| Filter | Options |
|---|---|
| Project | One or more projects |
| Environment | `production`, `staging`, `preview` |
| Level | `debug`, `info`, `warn`, `error` |
| Run ID | Scope to a specific agent run |
| Component | `daemon`, `scheduler`, `runner`, `agent` |

### Log Line Format

Each log line is displayed with:

- Timestamp (your browser's local timezone; hover to see UTC)
- Level badge (colour-coded)
- Component label
- Run ID link (if applicable — click to open Run Detail)
- Message text

### Export

Click **Export** (top-right of the results panel) to download the filtered results as newline-delimited JSON. The format matches `ao cloud logs --json` output.

---

## Team and Access

### Members

Navigate to **Settings → Members** to manage access. Roles:

| Role | Permissions |
|---|---|
| **Owner** | Full access: billing, member management, project creation and deletion |
| **Admin** | All project operations; cannot manage billing or remove owners |
| **Member** | Read and write access to projects and deployments; cannot manage members |
| **Viewer** | Read-only access to projects, deployments, and logs |

### Inviting Members

1. Go to **Settings → Members → Invite**.
2. Enter the invitee's email address and select a role.
3. Click **Send Invite**. The invitee receives an email with a link that is valid for 72 hours.

### Personal Access Tokens

Tokens for CI environments and the `ao cloud login --token` flag are managed at **Settings → Access Tokens**:

1. Click **New Token**.
2. Enter a label (e.g. `ci-deploy`) and set an optional expiry date.
3. Copy the token — it is shown only once.
4. To revoke a token, click the **×** next to it in the token list.

---

## Notifications

The **Settings → Notifications** page configures alerts for cloud events:

| Event | Description |
|---|---|
| Daemon stopped unexpectedly | Fires when daemon status changes to `error` or `stopped` without an explicit `ao cloud stop` |
| Deployment failed | Push or start operation returned an error |
| Agent run failed | An agent run exited with a non-zero status |
| Queue depth threshold | Queue depth exceeds a configurable value |

Notifications can be delivered via email or webhook. Webhook payloads follow the standard `ao.cli.v1` JSON envelope.

---

## Related

- [Cloud Deployment](cloud-deployment.md) — CLI-driven deployment workflow
- [Cloud Billing](cloud-billing.md) — subscription plans, usage, and invoices
- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full CLI flag reference
- [Privacy & Data Policy](privacy.md) — what the dashboard stores and transmits
