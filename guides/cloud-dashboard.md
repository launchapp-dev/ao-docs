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

Press **Cmd+K** (macOS) or **Ctrl+K** (Windows / Linux) at any time to open the [command palette](#command-palette) for keyboard-driven navigation.

---

## Notification Bell

The notification bell icon in the top-right header displays a badge with the count of unread notifications. Clicking the bell opens a slide-out panel listing recent events in reverse chronological order.

Each entry in the panel shows:

- An event-type icon
- Message text (e.g. `Daemon stopped: my-project / production`)
- Relative timestamp (hover to see the absolute UTC time)
- A mark-as-read button

**Mark all as read** clears the badge immediately. Clicking any notification navigates to the relevant project, run, or deployment.

### Notification API

Notifications are also accessible over the cloud REST API, authenticated with an [API key](#api-key-management):

```
GET  /api/v1/notifications          List notifications (paginated)
POST /api/v1/notifications/read     Mark one or all as read
```

Example response from `GET /api/v1/notifications`:

```json
{
  "notifications": [
    {
      "id": "notif_01hx...",
      "type": "daemon_stopped",
      "project": "my-project",
      "environment": "production",
      "message": "Daemon stopped unexpectedly",
      "created_at": "2026-04-03T10:15:00Z",
      "read": false
    }
  ],
  "unread_count": 3,
  "next_cursor": "cur_01hy..."
}
```

Pass `?cursor=<next_cursor>` to fetch the next page. Notifications are retained for 30 days regardless of read status.

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
- **Live output** — streaming text output from the current phase (powered by SSE; see [SSE Workflow Streaming](#sse-workflow-streaming))
- **Artifacts** — files created or modified by the run, once the run completes
- **Token usage** — per-phase breakdown of input and output tokens

### History

The **History** tab shows completed runs. Filter by project, workflow, date range, or status. Completed run detail panels include the full phase timeline and all artifacts.

---

## SSE Workflow Streaming

Run Detail streams phase output using Server-Sent Events (SSE). The connection opens automatically when the panel loads and closes when the run reaches a terminal state.

**Stream endpoint:**

```
GET /api/v1/runs/{run_id}/stream
Authorization: Bearer <api-key>
Accept: text/event-stream
```

**Event types on the stream:**

| Event | Description |
|---|---|
| `phase_started` | New phase has begun; data includes `phase_name` and `phase_index` |
| `output` | A chunk of text output from the running phase |
| `phase_done` | Phase completed; data includes `status` (`done` or `failed`) |
| `run_done` | Terminal event — the connection closes after this |

Consumers can reconnect after a dropped connection by sending the `Last-Event-ID` header; the server replays all events since that ID. Events are buffered for 60 minutes after run completion.

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

## Audit Log

The Audit Log records every administrative action performed on the organisation. Navigate to **Settings → Audit Log** (requires Admin role or above).

### Events Logged

| Category | Events |
|---|---|
| Members | Invite sent, invite accepted, role changed, member removed |
| API Keys | Token created, token revoked |
| Webhooks | Endpoint created, endpoint updated, endpoint deleted |
| Projects | Project created, project deleted |
| Billing | Plan changed, payment method added, payment method removed |
| Settings | Organisation renamed, SSO configuration changed |

### Audit Log Table

| Column | Description |
|---|---|
| Timestamp | When the action occurred |
| Actor | Email address of the member who performed the action |
| Event | Machine-readable event type (e.g. `member.role_changed`) |
| Description | Human-readable summary |
| IP address | Source IP of the authenticated request |
| Target | The entity modified (member email, token label, project slug, etc.) |

### Filtering and Export

Use the filter bar to narrow results by actor email, event category, date range, or target. The default view shows the last 30 days.

Click **Export CSV** to download all columns including a `request_id` field for correlation with support tickets. Exported CSVs are not subject to retention limits.

**Retention by plan:** 30 days (Starter), 90 days (Pro), 1 year (Team and Enterprise).

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

### RBAC Route Guards

The dashboard enforces role-based access on every route. Navigating to a restricted route redirects to the nearest accessible parent with a "permission denied" toast.

| Route prefix | Minimum role |
|---|---|
| `/settings/billing` | Owner |
| `/settings/members` | Admin |
| `/settings/tokens` | Admin |
| `/settings/webhooks` | Admin |
| `/settings/audit-log` | Admin |
| `/projects/*/deployments` | Member |
| `/projects/*/agents` | Member |
| `/projects/*/logs` | Member |
| Everything else | Viewer |

Route guards are re-evaluated whenever the active organisation changes. Switching to an organisation where you hold a lower role immediately redirects away from any route you no longer have access to.

### Inviting Members

1. Go to **Settings → Members → Invite**.
2. Enter the invitee's email address and select a role.
3. Click **Send Invite**. The invitee receives an email with a link valid for 72 hours.

### API Key Management

Long-lived API keys for CLI authentication, CI/CD pipelines, and direct REST API access are managed at **Settings → Access Tokens**.

**Creating a key:**

1. Click **New Token**.
2. Enter a label (e.g. `ci-deploy`) and set an optional expiry date.
3. Select a **scope**:
   - `read` — GET endpoints only
   - `deploy` — read access plus `ao cloud push` and daemon start / stop operations
   - `admin` — full API access including member management (requires Owner role)
4. Copy the token — it is displayed only once. Store it in a secrets manager.

**Key list columns:**

| Column | Description |
|---|---|
| Label | Human-readable name |
| Scope | `read`, `deploy`, or `admin` |
| Last used | Timestamp of the most recent authenticated request, or "Never" |
| Expires | Expiry date, or "No expiry" |
| Created by | Member who created the token |

**Revoking a key:** Click **Revoke** next to any key. Revocation is immediate; in-flight requests authenticated with that key will still complete.

---

## Rate Limiting

The cloud REST API enforces rate limits per API key:

| Plan | Requests / minute | Burst |
|---|---|---|
| Starter | 30 | 10 |
| Pro | 120 | 30 |
| Team | 600 | 60 |
| Enterprise | Custom | Custom |

Exceeded limits return `429 Too Many Requests` with a `Retry-After` header indicating when the next request window opens. The response body includes:

```json
{
  "error": "rate_limit_exceeded",
  "retry_after": 12,
  "limit": 120,
  "remaining": 0,
  "reset_at": "2026-04-03T10:16:00Z"
}
```

SSE stream connections (`/api/v1/runs/{run_id}/stream`) count as one request against the rate limit at connection time and do not accrue additional charges while the stream is held open.

The dashboard UI operates under an internal service account and is not subject to per-key limits.

---

## Notifications

The **Settings → Notifications** page configures alerts for cloud events:

| Event | Description |
|---|---|
| Daemon stopped unexpectedly | Fires when daemon status changes to `error` or `stopped` without an explicit `ao cloud stop` |
| Deployment failed | Push or start operation returned an error |
| Agent run failed | An agent run exited with a non-zero status |
| Queue depth threshold | Queue depth exceeds a configurable value |

Notifications are delivered via email or webhook.

### Webhook Settings

Navigate to **Settings → Notifications → Webhooks** to configure webhook endpoints. Each endpoint has:

| Field | Description |
|---|---|
| **URL** | The HTTPS endpoint to POST events to |
| **Events** | Checkboxes for individual event types, or "All events" |
| **Secret** | Shared HMAC-SHA256 secret used to sign payloads |
| **Status** | `active` or `disabled` |
| **Last delivery** | Timestamp and HTTP status of the most recent attempt |

**Payload signature:** Every delivery includes an `X-AO-Signature` header containing `sha256=<hmac_hex>` computed over the raw request body using the webhook secret. Verify this value before processing the payload.

**Delivery history:** Click the delivery-history link on any endpoint to view recent attempts, including the HTTP status returned by your endpoint, the response body (truncated to 1 KB), and a **Redeliver** button that resends the most recent payload.

Webhook payloads follow the `ao.cli.v1` JSON envelope format.

---

## Command Palette

Press **Cmd+K** (macOS) or **Ctrl+K** (Windows / Linux) anywhere in the dashboard to open the command palette.

### Navigation Commands

| Command | Destination |
|---|---|
| `> Projects` | Projects overview |
| `> Agents: Live` | Live agent run table |
| `> Logs` | Log search |
| `> Audit Log` | Audit log |
| `> Billing` | Billing settings |
| `> Notifications` | Notification settings |
| `> Webhooks` | Webhook settings |

### Action Commands

| Command | Action |
|---|---|
| `> New Token` | Opens the new API key dialog |
| `> Invite Member` | Opens the member invite dialog |
| `> Switch Org` | Shows the organisation selector |

### Fuzzy Resource Search

Type any fragment of a project name, run ID, deployment ID, or member email to jump directly to that resource. Results update incrementally. The search scope is limited to the active organisation.

**Keyboard shortcuts within the palette:**

- `↑` / `↓` — navigate results
- `Enter` — select the highlighted result
- `Esc` — close the palette

On mobile, the command palette is accessible via the search icon in the top nav bar.

---

## Mobile Layout

The dashboard is fully responsive. Layout adapts at three breakpoints:

| Breakpoint | Viewport width | Layout |
|---|---|---|
| **Desktop** | ≥ 1024 px | Full sidebar always visible; multi-column tables |
| **Tablet** | 640–1023 px | Sidebar collapses to an icon-only rail; tables scroll horizontally |
| **Mobile** | < 640 px | Sidebar hidden behind a hamburger menu; tables replaced by card stacks |

At the mobile breakpoint:

- The notification bell, profile avatar, and hamburger menu appear in a top nav bar.
- Project cards replace the projects table. Each card shows the project name, daemon status badge, and quick-action buttons.
- Run Detail panels open as full-screen bottom sheets instead of side panels.
- The command palette opens via the search icon in the top nav bar.

---

## Related

- [Cloud Deployment](cloud-deployment.md) — CLI-driven deployment workflow
- [Cloud Billing](cloud-billing.md) — subscription plans, usage, and invoices
- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full CLI flag reference
- [Privacy & Data Policy](privacy.md) — what the dashboard stores and transmits
