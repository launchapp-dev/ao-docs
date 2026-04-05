# Cloud Dashboard Guide

The Animus cloud dashboard is a React web application hosted at `app.ao.dev`. It provides a browser-based view of all your cloud projects: deployment status, live agent activity, log browsing, and billing — without requiring the CLI.

For CLI-based cloud management see [Cloud Deployment](cloud-deployment.md). For billing details see [Cloud Billing](cloud-billing.md).

---

## Accessing the Dashboard

Navigate to `app.ao.dev` and sign in with the same credentials used by `animus cloud login`. After authentication you land on the **Projects** overview.

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

## Workflow Template Gallery

The **Template Gallery** at `app.ao.dev/templates` provides a curated library of pre-built workflow definitions that can be deployed into any of your projects with a single click.

### Browsing the Gallery

Templates are organised into categories in the left filter panel:

| Category | Examples |
|---|---|
| **Development** | Code review, automated PR description, dependency audit |
| **Quality** | Test generation, linting fix, coverage report |
| **Documentation** | API docs generation, changelog draft, README refresh |
| **Operations** | Incident triage, alert summarisation, runbook execution |
| **Data** | ETL pipeline, data validation, schema migration |
| **Custom** | Templates published by your organisation |

Type in the search box to filter by keyword. Results update as you type and match against template name, description, and tags.

### Template Card

Each card in the gallery shows:

- Template name and one-line description
- Author (Animus team or organisation name for custom templates)
- Category tags
- Star count (community favourites)
- **Preview** button — opens a read-only view of the workflow YAML
- **Deploy** button — starts the deployment flow

### Deploying a Template

1. Click **Deploy** on any template card.
2. A modal asks you to select:
   - **Target project** — choose an existing project or create a new one inline.
   - **Environment** — `production`, `staging`, or `preview`.
   - **Variable overrides** — any `{{ variable }}` placeholders in the template YAML are listed with editable fields; defaults from the template are pre-filled.
3. Click **Deploy**. The dashboard calls `animus cloud push` internally with the resolved YAML and selected target.
4. On success, you are taken directly to the **Workflow Detail** page for the newly deployed workflow.

Deployed templates behave identically to workflows you push manually with the CLI. The source template is noted in the workflow metadata (`template_id` field) but does not create a dependency — you can edit the YAML freely after deployment.

### Featured Templates

The gallery highlights a rotating **Featured** row at the top of the main grid. Featured templates are curated by the Animus team based on community star counts and recent additions. They rotate weekly.

### Template Versions

Each template has a version history. The gallery shows the latest stable version by default. To deploy an older version:

1. Click **Preview** on any template card.
2. Click the version selector in the preview modal header.
3. Choose the version you want.
4. Click **Deploy this version**.

Deployed templates record the `template_id` and `template_version` in the workflow metadata. When a newer template version is published, the dashboard shows a **Template update available** banner on the affected workflow's detail page. Upgrading is opt-in — existing deployments continue running the version they were deployed with.

### Importing Templates

Import a template directly from a GitHub URL or a local YAML file:

**From GitHub:**
1. Navigate to **Settings → Templates → Import**.
2. Paste the raw GitHub URL to a workflow YAML file (e.g. `https://raw.githubusercontent.com/org/repo/main/.ao/workflows/example.yaml`).
3. Click **Import**. Animus fetches the file, validates the YAML, and adds it to your organisation's Custom templates.

**From file:**
1. Navigate to **Settings → Templates → Import**.
2. Click **Upload file** and select a `.yaml` file.
3. Fill in the name and description fields (pre-populated from the YAML `name` field if present).
4. Click **Save**.

Imported templates appear in the **Custom** category and are private to your organisation.

### Publishing Custom Templates

Organisation templates are available only to members of your organisation. To publish a custom template:

1. Navigate to **Settings → Templates → Publish**.
2. Paste or upload the workflow YAML.
3. Fill in the name, description, category tags, and variable placeholder descriptions.
4. Click **Publish**. The template appears in the gallery under the **Custom** category for all members.

Requires **Admin** role or above to publish. Members can deploy but not publish or delete organisation templates.

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

## Workflow Detail Page

Click any workflow name in the active workflow table (or in the Agents → History run list) to open the **Workflow Detail** page for that workflow definition.

### Overview Tab

The Overview tab shows static metadata from the workflow YAML:

| Field | Description |
|---|---|
| Name | Workflow definition name (matches the YAML `name` field) |
| Phases | Ordered list of phase names and their assigned agent persona |
| Triggers | File watchers and webhook triggers attached to this workflow |
| Packs | Pack dependencies declared in the workflow |
| Last updated | Timestamp of the most recent deployment that included this workflow |

### Runs Tab

The Runs tab lists all historical and live agent runs dispatched from this workflow definition:

| Column | Description |
|---|---|
| Run ID | Unique identifier; click to open [Run Detail](#run-detail) |
| Status | `running`, `done`, `failed`, or `cancelled` |
| Started | Absolute timestamp |
| Duration | Wall-clock run time |
| Phases completed | Count of phases that reached `done` status |
| Tokens | Total token usage across all phases |

Filter the runs table by status, date range, or environment using the toolbar above the table.

### YAML Tab

The YAML tab renders the current workflow definition as syntax-highlighted YAML. The definition shown reflects the latest active deployment; use the deployment selector in the toolbar to compare against an earlier version.

---

## Project Settings

Per-project settings are managed at **Settings → Project** inside any project's detail view. Requires **Member** role or above.

### General

| Setting | Description |
|---|---|
| Display name | Human-readable label shown in the dashboard (independent of the project slug) |
| Default environment | The environment opened by default when viewing the project (`production`, `staging`, or `preview`) |
| Description | Optional free-text notes |

### Log Retention

| Setting | Description |
|---|---|
| Log retention window | How long raw log lines are stored (7 days default; up to 90 days on Pro, 1 year on Team/Enterprise) |
| Structured export | Enables automatic nightly export of log lines to the configured S3-compatible bucket |

Changing the retention window takes effect for new log lines only. Existing lines are not retroactively purged or extended.

### Environment Variables

Runtime environment variables injected into every agent run for this project. Variables are stored encrypted at rest and are not visible after initial entry — only the variable name is shown in the list.

| Action | Description |
|---|---|
| **Add variable** | Opens an inline form: key, value, optional description |
| **Edit** | Re-enters the value for an existing key |
| **Delete** | Removes the variable; takes effect on the next run |

### Danger Zone

| Action | Description |
|---|---|
| **Pause project** | Stops the daemon and prevents new runs from starting; deployments and data are preserved |
| **Delete project** | Permanently removes all deployments, artifacts, logs, and run history; requires typing the project slug to confirm |

---

## Deployments

The Deployments page lists every `animus cloud push` event across all projects. Columns:

| Column | Description |
|---|---|
| Deployment ID | Unique identifier (matches the `deployment_id` returned by `animus cloud push --json`) |
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
| Run ID | Unique run identifier (use with `animus cloud logs --run-id`) |
| Project | Source project |
| Workflow | Workflow definition name |
| Phase | Currently executing phase |
| Started | Duration since the run started |
| Tokens | Estimated token usage for the current run |

### Run Detail

Click a run ID to open the Run Detail panel:

- **Phase timeline** — sequential list of phases with status icons (`pending`, `running`, `done`, `failed`)
- **Run timeline** — Gantt-style timing chart showing phase durations and overlap (see [Workflow Run Timeline](#workflow-run-timeline))
- **Live output** — streaming text output from the current phase (powered by SSE; see [SSE Workflow Streaming](#sse-workflow-streaming))
- **Artifacts** — files created or modified by the run, once the run completes
- **Token usage** — per-phase breakdown of input and output tokens

### History

The **History** tab shows completed runs. Filter by project, workflow, date range, or status. Completed run detail panels include the full phase timeline and all artifacts.

---

## Workflow Run Timeline

The **Run Timeline** tab inside any Run Detail panel shows a Gantt-style chart of phase execution. Each phase is rendered as a horizontal bar spanning its wall-clock start and end times. The timeline was introduced in v58.

### Reading the Timeline

| Visual element | Meaning |
|---|---|
| Horizontal bar | A single phase; bar width represents elapsed duration |
| Bar colour | Phase status: blue (running), green (done), red (failed), yellow (cancelled) |
| Grey region | Time between the run starting and the first phase beginning (queue wait) |
| Overlapping bars | Phases executing in parallel |
| Tick marks | Elapsed time from run start in seconds |

Hover over any bar to see a tooltip with:

- Phase name
- Absolute start and end timestamps (UTC)
- Wall-clock duration
- Token usage (input / output)

### Zooming and Panning

- **Scroll** horizontally to pan through the timeline.
- **Pinch** (trackpad) or use the **+** / **−** buttons to zoom in or out.
- Click **Fit** to reset the viewport so the entire run fits on screen.

For very long runs (more than 60 minutes), the timeline defaults to a compressed view where each tick represents 1 minute. Zoom in to see second-level granularity.

### Live Mode

When the run is still active, the timeline updates every 3 seconds:

- Running phases extend their bars in real time.
- New phases appear as they start.
- A **Now** marker (a vertical red line) tracks the current wall-clock time.

The timeline stops auto-updating once the run reaches a terminal state (`done`, `failed`, or `cancelled`).

### Critical Path

Click **Show critical path** in the toolbar to overlay the longest dependency chain through the phase graph. The critical path bars are outlined in orange. Phases not on the critical path are dimmed. Use this view to identify which phases are the bottleneck for total run duration.

### Exporting

Click **Export PNG** to download the current timeline view as a rasterised image at 2× device pixel ratio. The export captures the full run span (not just the visible viewport).

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

Click **Export** (top-right of the results panel) to download the filtered results as newline-delimited JSON. The format matches `animus cloud logs --json` output.

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
   - `deploy` — read access plus `animus cloud push` and daemon start / stop operations
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
| Daemon stopped unexpectedly | Fires when daemon status changes to `error` or `stopped` without an explicit `animus cloud stop` |
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

**Payload signature:** Every delivery includes an `X-Animus-Signature` header containing `sha256=<hmac_hex>` computed over the raw request body using the webhook secret. Verify this value before processing the payload.

**Delivery history:** Click the delivery-history link on any endpoint to view recent attempts, including the HTTP status returned by your endpoint, the response body (truncated to 1 KB), and a **Redeliver** button that resends the most recent payload.

Webhook payloads follow the `ao.cli.v1` JSON envelope format.

### Webhook Delivery Mechanics

Each event triggers an immediate HTTP POST to all matching active endpoints. Delivery is best-effort with automatic retries on failure.

**Retry schedule:**

| Attempt | Delay after previous attempt |
|---|---|
| 1 (initial) | Immediately |
| 2 | 30 seconds |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After five failed attempts the delivery is marked **failed** and no further retries are scheduled. A failed delivery is visible in the endpoint's delivery history with status `failed`. Use the **Redeliver** button to trigger a manual retry at any time.

A delivery is considered successful when the endpoint returns any `2xx` HTTP status code within 10 seconds. Non-`2xx` responses (including `3xx` redirects) and connection timeouts count as failures.

**Delivery timeout:** 10 seconds per attempt. Responses that arrive after this window are discarded and the attempt is counted as failed.

**Delivery headers sent with every request:**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-Animus-Event` | Event type string (e.g. `daemon.stopped`) |
| `X-Animus-Delivery` | Unique delivery ID (stable across retries for the same event) |
| `X-Animus-Signature` | `sha256=<hmac_hex>` — HMAC-SHA256 of the raw request body |
| `User-Agent` | `ao-cloud-webhook/1` |

### Webhook Delivery Log

The delivery log for each endpoint is accessible at **Settings → Notifications → Webhooks → [endpoint] → Deliveries**. The table shows:

| Column | Description |
|---|---|
| Delivery ID | Stable across retries; use for deduplication |
| Event | Event type that triggered the delivery |
| Attempt | Attempt number (1–5) |
| Status | `success`, `failed`, or `pending` |
| HTTP status | The HTTP response code returned by your endpoint, or `—` on timeout |
| Duration | Response time in milliseconds |
| Timestamp | When this attempt was made |

Delivery records are retained for 30 days. Click any row to see the full request body and response body (truncated to 4 KB).

---

## Activity Feed

The **Activity Feed** at the bottom of each project's detail page shows a reverse-chronological stream of events that affected the project — triggers fired, dispatches created, runs started and completed, daemon state changes, and deployments pushed. It provides a quick at-a-glance history without opening the full Logs or Audit Log pages.

### Feed Events

| Event type | Description |
|---|---|
| `trigger.fired` | A file-watch, webhook, or GitHub trigger matched and created a dispatch |
| `trigger.filtered` | A trigger matched the event kind but the `filter.expr` condition was false |
| `dispatch.queued` | A dispatch entered the run queue |
| `run.started` | An agent run transitioned to `running` |
| `run.completed` | An agent run finished with status `done` |
| `run.failed` | An agent run finished with status `failed` |
| `run.cancelled` | An agent run was cancelled manually |
| `daemon.started` | The cloud daemon started |
| `daemon.stopped` | The cloud daemon stopped (expected or unexpected) |
| `deployment.pushed` | A new deployment was received via `animus cloud push` |

### Feed Controls

The feed defaults to the last 100 events for the active environment. Use the controls above the feed to adjust the view:

| Control | Description |
|---|---|
| **Environment selector** | Switch between `production`, `staging`, and `preview` feeds |
| **Event filter** | Show all events or filter to a specific event type category (`triggers`, `runs`, `daemon`, `deployments`) |
| **Auto-refresh toggle** | When on (default), the feed polls for new events every 10 seconds and prepends them without a full page reload |
| **Load more** | Fetches the next 100 events older than the oldest currently displayed |

### Feed Entry Detail

Click any feed entry to expand it. The expanded view shows:

- Full event payload as formatted JSON
- For `run.*` events: a link to the Run Detail panel
- For `trigger.fired` events: the trigger ID and dispatch ID
- For `deployment.pushed` events: the deployment ID and artifact count

### Activity Feed API

The feed is available over the REST API:

```
GET /api/v1/projects/{project}/activity
Authorization: Bearer <api-key>
```

Query parameters:

| Parameter | Description |
|---|---|
| `environment` | Filter to `production`, `staging`, or `preview`. Defaults to `production`. |
| `type` | Comma-separated list of event type prefixes to include (e.g. `run,daemon`). |
| `limit` | Number of events to return (default: `100`, max: `500`). |
| `before` | ISO 8601 timestamp — return only events older than this time. |

Example response:

```json
{
  "events": [
    {
      "id": "evt_01hx...",
      "type": "run.completed",
      "project": "my-project",
      "environment": "production",
      "timestamp": "2026-04-04T09:30:00Z",
      "data": {
        "run_id": "run_01hy...",
        "workflow": "pr-review",
        "status": "done",
        "duration_seconds": 47
      }
    }
  ],
  "next_before": "2026-04-04T09:25:00Z"
}
```

Pass `?before=<next_before>` to paginate backwards through the feed.

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

## Dark Mode

The dashboard ships with a light theme by default and a full dark theme that inverts surface, border, and text colours. Syntax-highlighted YAML, log lines, and code blocks use separate dark-variant palettes optimised for readability.

### Toggling the Theme

Click the **sun / moon icon** in the top-right header to switch between light and dark mode. The preference is stored in `localStorage` under the key `ao.theme` and persists across sessions in the same browser.

To let the browser choose automatically based on the operating-system colour preference, select **System** from the theme picker dropdown (accessible by long-pressing the toggle on desktop, or via **Settings → Appearance → Theme**).

| Setting | Description |
|---|---|
| **Light** | White surfaces, dark text, blue accent colours |
| **Dark** | Near-black surfaces (`#0d1117`), off-white text, lighter blue accents |
| **System** | Follows `prefers-color-scheme` media query — switches automatically when the OS theme changes |

### Appearance Settings

Navigate to **Settings → Appearance** for additional display preferences:

| Preference | Options | Default |
|---|---|---|
| Theme | Light, Dark, System | System |
| Font size | Small (13 px), Medium (14 px), Large (16 px) | Medium |
| Reduced motion | Off, On | Off (respects OS `prefers-reduced-motion`) |
| Dense tables | Off, On | Off — normal row height; On reduces row padding for more data on screen |

Appearance settings are per-browser and are not synchronised across devices or stored in the cloud.

---

## Loading States

The dashboard uses skeleton screens to indicate that content is loading. Each skeleton mirrors the shape of the content it replaces so the layout does not shift when data arrives.

| Location | Skeleton description |
|---|---|
| Projects list | Three card-shaped skeletons with a status-badge placeholder and two lines of text |
| Run Detail panel | Phase timeline items as grey bars; output area as a blinking cursor line |
| Workflow Detail YAML | Full-height code block with alternating short and long grey lines |
| Audit log table | Six rows with cells of varying widths |
| Notification panel | Three notification-shaped rows with an icon circle and two text lines each |

Skeletons are dismissed as soon as the first data payload arrives from the API; partial data is displayed immediately rather than waiting for the full response.

---

## Empty States

When a list or table has no data to show, the dashboard renders an illustrated empty state with a short explanation and a primary action button.

| Page / section | Illustration | Primary action |
|---|---|---|
| Projects (no projects yet) | Server with a blinking light | **Push your first project** (links to [Cloud Deployment](cloud-deployment.md)) |
| Agents → Live (no active runs) | Robot sitting idle | **Start daemon** (opens a project selector then calls `animus cloud start`) |
| Agents → History (no completed runs) | Timeline with no entries | **View workflows** (navigates to the Workflow Detail page) |
| Audit Log (no events in range) | Magnifying glass over an empty page | **Expand date range** (resets the date filter to the last 30 days) |
| Notifications panel (all read) | Bell with a checkmark | No action — confirming all notifications are cleared |
| API keys (none created) | Key on a table | **Create token** (opens the new API key dialog) |
| Webhooks (none configured) | Plug with no socket | **Add endpoint** (opens the new webhook form) |

---

## Error Boundaries

Each major dashboard section is wrapped in a React error boundary. If a component throws an unhandled error, the boundary catches it and renders a contained error panel without crashing the entire application.

### Error Panel Contents

- A short title (e.g. "Could not load run detail")
- The error message — sanitised before display
- A **Retry** button that remounts the failed component
- A **Report issue** link that prefills a GitHub issue with the error message, current URL, and a browser info summary

### Scope of Boundaries

| Boundary | Scope |
|---|---|
| Root boundary | Catches errors that escape all inner boundaries; shows a full-page error screen |
| Sidebar | Isolates sidebar rendering failures from the main content area |
| Each dashboard section | Projects, Agents, Logs, Audit Log, Billing, Notifications — each is individually bounded |
| Run Detail panel | The SSE stream and phase timeline are bounded separately so a stream error does not remove the static phase list |
| Modals | Each dialog is bounded so a modal crash does not close the underlying page |

Error details are also sent to the cloud error-tracking service with the authenticated user's organisation ID for triage. No source code or personal data beyond the organisation ID is transmitted.

---

## Related

- [Cloud Deployment](cloud-deployment.md) — CLI-driven deployment workflow
- [Cloud Billing](cloud-billing.md) — subscription plans, usage, and invoices
- [Workflow DAG Visualization](workflow-dag.md) — interactive phase graph for workflow structure and live execution state
- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full CLI flag reference
- [Privacy & Data Policy](privacy.md) — what the dashboard stores and transmits
