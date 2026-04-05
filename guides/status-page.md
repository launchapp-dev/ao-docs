# Service Status Page

AO publishes a public status page at `status.ao.dev` that tracks the availability and performance of all AO Cloud services in real time. Use the status page to check whether an incident is in progress before filing a support ticket or debugging a local issue.

The status page was introduced in v45.

---

## Accessing the Status Page

Visit `status.ao.dev` in any browser — no authentication required. The page is hosted independently of the main AO Cloud infrastructure so it remains accessible during partial outages.

---

## Service Components

The status page tracks each AO Cloud service component independently. A single degraded component does not necessarily affect all users.

| Component | Description |
|---|---|
| **API** | The cloud REST API (`api.ao.dev`) — powering CLI commands and dashboard data fetches |
| **Dashboard** | The web application at `app.ao.dev` |
| **Daemon Host** | The compute layer that runs cloud-hosted daemon instances |
| **Trigger Engine** | GitHub webhook receiver and trigger evaluation service |
| **Artifact Storage** | Deployment artifact storage and retrieval |
| **Log Ingestion** | Structured log ingestion pipeline |
| **Billing** | Stripe-backed billing and metering service |
| **GitHub App** | GitHub App webhook delivery and check run posting |

### Status Indicators

Each component displays one of four statuses:

| Status | Colour | Meaning |
|---|---|---|
| **Operational** | Green | All systems normal |
| **Degraded Performance** | Yellow | The service is available but slower than usual, or a subset of requests are failing |
| **Partial Outage** | Orange | A significant fraction of requests are affected |
| **Major Outage** | Red | The service is unavailable or completely unresponsive |
| **Maintenance** | Blue | Scheduled maintenance is in progress; the service may be intermittently unavailable |

The overall system status at the top of the page reflects the worst status across all components.

---

## Incidents

When a component degrades below **Operational**, AO opens an incident. Each incident has a public incident page with:

- **Title** — a short description (e.g. `Elevated API error rate`)
- **Impact** — which components are affected
- **Timeline** — timestamped updates from the AO engineering team, from detection through resolution
- **Current status** — `Investigating`, `Identified`, `Monitoring`, or `Resolved`

### Incident Lifecycle

| Status | Meaning |
|---|---|
| `Investigating` | The team has detected the issue and is diagnosing the cause |
| `Identified` | The root cause is known; a fix is being deployed |
| `Monitoring` | The fix is deployed; the team is watching metrics before resolving |
| `Resolved` | All affected components have returned to operational status |

Resolved incidents remain visible on the status page for 90 days and in the incident history indefinitely.

---

## Incident History

Click **Incident History** at the bottom of the status page to view all past incidents. The history table shows:

| Column | Description |
|---|---|
| Date | Start date of the incident |
| Title | Incident summary |
| Duration | Time from first impact to resolution |
| Components | Affected components |
| Impact | Worst severity level reached |

Click any incident row to read the full timeline and post-mortem (where available). Significant incidents include a linked post-mortem document explaining the root cause, contributing factors, and remediation steps taken.

---

## Uptime Metrics

Below each component, the status page shows a 90-day uptime bar:

- Each day is represented by a coloured segment.
- Hover a segment to see the date and uptime percentage for that day.
- The rolling average uptime percentage for the last 90 days is displayed next to the component name.

Overall system uptime (the percentage of minutes in the last 90 days when all components were operational) appears in the header.

---

## Subscribing to Updates

### Email Subscriptions

Click **Subscribe to updates** on the status page. Enter your email address to receive notifications for:

- New incidents (immediate)
- Incident status updates (as the team posts them)
- Incident resolutions
- Scheduled maintenance windows (72 hours in advance)

Unsubscribe at any time via the link in any notification email.

### RSS / Atom Feed

The status page publishes an Atom feed of all incident updates:

```
https://status.ao.dev/feed.atom
```

Subscribe with any RSS reader to receive updates without email.

### Webhook Notifications

For programmatic consumption, configure a status webhook at **Settings → Notifications** in the AO Cloud dashboard. Toggle **Service status events** to receive a POST request to your endpoint whenever a component status changes or an incident is created or updated.

Status webhook payload:

```json
{
  "event": "incident.created",
  "incident": {
    "id": "inc_01j9...",
    "title": "Elevated API error rate",
    "status": "investigating",
    "impact": "partial_outage",
    "components": ["api"],
    "started_at": "2026-04-03T09:42:00Z",
    "url": "https://status.ao.dev/incidents/inc_01j9..."
  }
}
```

The payload follows the `ao.cli.v1` JSON envelope format. See [Cloud Dashboard → Notifications](cloud-dashboard.md#notifications) for signature verification details.

---

## Scheduled Maintenance

Scheduled maintenance windows are announced at least 72 hours in advance. During a scheduled window:

- Affected components show the **Maintenance** (blue) status.
- The dashboard displays a banner at the top: **Scheduled maintenance in progress — some features may be unavailable.**
- No incident is opened unless the maintenance runs longer than planned or causes unexpected degradation.

Maintenance windows are listed on the status page under **Upcoming Maintenance** before they begin.

---

## Embedding the Status Widget

Add a live status indicator to your internal tools or dashboards by embedding the AO status widget:

```html
<script
  src="https://status.ao.dev/embed.js"
  data-style="badge"
  data-link="true">
</script>
```

**Widget options:**

| Attribute | Values | Description |
|---|---|---|
| `data-style` | `badge`, `bar`, `dot` | Visual style of the widget |
| `data-link` | `true`, `false` | Whether clicking the widget opens `status.ao.dev` |
| `data-components` | comma-separated component IDs | Limit the widget to specific components |
| `data-theme` | `light`, `dark`, `auto` | Colour theme (default: `auto`) |

The widget script is served from the independently hosted status infrastructure and does not depend on `app.ao.dev` availability.

---

## API

The status page exposes a read-only JSON API for integration into monitoring tools:

```
GET https://status.ao.dev/api/v1/status
```

Response:

```json
{
  "overall": "operational",
  "components": [
    {
      "id": "api",
      "name": "API",
      "status": "operational",
      "uptime_90d": 99.97
    },
    {
      "id": "daemon-host",
      "name": "Daemon Host",
      "status": "operational",
      "uptime_90d": 99.91
    }
  ],
  "active_incidents": [],
  "updated_at": "2026-04-04T08:00:00Z"
}
```

```
GET https://status.ao.dev/api/v1/incidents?status=active
GET https://status.ao.dev/api/v1/incidents/{incident_id}
```

All endpoints return JSON and require no authentication. The API is rate-limited to 60 requests per minute per IP address.

---

## Related

- [Cloud Dashboard](cloud-dashboard.md) — notification settings including status webhooks
- [Troubleshooting](troubleshooting.md) — diagnosing local issues distinct from service incidents
- [Cloud Deployment](cloud-deployment.md) — deploying to AO Cloud
