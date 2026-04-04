# Cloud Billing Guide

AO cloud billing is managed through the dashboard at `app.ao.dev/settings/billing`. Subscriptions are processed by Stripe. You can view your current plan, monitor usage, download invoices, and update payment methods without leaving the browser.

For access to the billing page you must be an **Owner** on the organisation. Admins and below see the plan name but cannot make changes.

---

## Plans

| Plan | Agents | Compute minutes / month | Included storage | Price |
|---|---|---|---|---|
| **Starter** | 1 | 1 000 | 1 GB | Free |
| **Pro** | 5 | 10 000 | 10 GB | Paid monthly |
| **Team** | 20 | 50 000 | 50 GB | Paid monthly |
| **Enterprise** | Unlimited | Custom | Custom | Annual contract |

All paid plans include:

- Unlimited projects and deployments
- All cloud environments (`production`, `staging`, `preview`)
- Priority support queue
- Audit log retention (90 days for Pro, 1 year for Team and Enterprise)

**Overage pricing** applies when compute minutes or storage exceed the plan allocation. Overages are billed at the end of each billing cycle and shown as a line item on the invoice.

---

## Changing Plans

### Upgrading via Stripe Checkout

Upgrades use a Stripe-hosted Checkout session:

1. Navigate to **Settings → Billing → Plan**.
2. Click **Change Plan** and select the new plan from the selector.
3. Click **Upgrade**. The dashboard creates a Stripe Checkout session and redirects you to the Stripe-hosted page.
4. On the Stripe page, confirm the plan and enter or select a payment method.
5. After a successful payment, Stripe redirects back to `app.ao.dev/settings/billing`. The dashboard polls for session completion and displays a confirmation banner once the plan is active.

Upgrades take effect immediately. The charge is prorated: unused time on the old plan is credited against the new plan charge.

### Downgrading

Downgrades do not go through Stripe Checkout. Select the lower plan in the in-app selector and confirm in the modal. The downgrade takes effect at the start of the next billing cycle; your current plan remains active at full capacity until then.

---

## Usage

The **Usage** tab at **Settings → Billing → Usage** shows the current billing cycle's consumption:

| Metric | Description |
|---|---|
| Compute minutes | Total wall-clock time across all agent runs |
| Storage | Deployment artifact storage, log storage, and audit log storage combined |
| Agents active | Peak concurrent agents seen during the cycle |
| Runs | Total completed agent runs |

Usage updates approximately every 15 minutes. The bar charts show daily consumption against the plan limit.

### Compute Minute Accounting

A compute minute is accrued for each minute — or fraction thereof — that an agent run is in the `running` state. Runs in `starting`, `stopping`, or `failed` states do not accrue compute time.

### Storage Accounting

Storage is sampled daily and averaged across the billing cycle. The following contribute to storage:

- Deployment artifacts pushed with `ao cloud push`
- Log lines retained beyond the default 7-day window (configurable per project in **Settings → Project**)
- Audit log entries

---

## Cost Breakdown

The **Cost Breakdown** tab at **Settings → Billing → Cost Breakdown** shows how charges accumulate at a granular level — by project, environment, and agent — for the current and past billing cycles.

### Project Cost Table

| Column | Description |
|---|---|
| Project | Project name and slug |
| Environment | `production`, `staging`, or `preview` |
| Compute minutes | Minutes consumed during the selected cycle |
| Compute cost | Charge for compute minutes at the plan rate or overage rate |
| Storage GB-days | Average storage sampled daily, in GB-days |
| Storage cost | Charge for storage at the plan rate or overage rate |
| Total | Sum of compute and storage charges for this project |

### Agent Cost Rows

Expanding any project row reveals a per-agent breakdown:

| Column | Description |
|---|---|
| Workflow | Workflow definition name |
| Runs | Number of completed agent runs |
| Total minutes | Sum of agent run wall-clock times |
| Avg minutes / run | Mean run duration |
| Compute cost | Proportional compute charge allocated to this workflow |

### Date Range Selector

The cost breakdown defaults to the current billing cycle. Use the cycle selector (top-right) to view any prior month. Cross-cycle comparisons are not currently supported; export both cycles to CSV and diff offline.

### Export

Click **Export CSV** to download the full cost breakdown as a flat file with one row per project-environment pair. The CSV includes all columns from the project cost table plus the `billing_cycle_start` and `billing_cycle_end` date fields.

### Cost Breakdown API

The cost breakdown data is also available over the REST API:

```
GET /api/v1/billing/cost-breakdown
Authorization: Bearer <api-key>
```

Optional query parameters:

| Parameter | Description |
|---|---|
| `cycle` | Billing cycle in `YYYY-MM` format. Defaults to the current cycle. |
| `project` | Filter to a specific project slug. |
| `group_by` | `project` (default) or `workflow` |

Example response:

```json
{
  "cycle": "2026-04",
  "currency": "usd",
  "total_compute_minutes": 4820,
  "total_compute_cost_cents": 482,
  "total_storage_gb_days": 12.4,
  "total_storage_cost_cents": 6,
  "items": [
    {
      "project": "my-project",
      "environment": "production",
      "compute_minutes": 3100,
      "compute_cost_cents": 310,
      "storage_gb_days": 8.2,
      "storage_cost_cents": 4,
      "total_cost_cents": 314
    }
  ]
}
```

---

## Usage Metering API

Usage data is available in real time over the REST API, authenticated with an [API key](cloud-dashboard.md#api-key-management) with at least `read` scope.

### Current Cycle Usage

```
GET /api/v1/billing/usage
Authorization: Bearer <api-key>
```

Returns the current billing cycle's aggregate usage:

```json
{
  "cycle_start": "2026-04-01T00:00:00Z",
  "cycle_end": "2026-04-30T23:59:59Z",
  "compute_minutes_used": 4820,
  "compute_minutes_limit": 10000,
  "compute_minutes_overage": 0,
  "storage_gb_used": 3.1,
  "storage_gb_limit": 10,
  "agents_active_peak": 3,
  "agents_limit": 5,
  "runs_total": 214,
  "updated_at": "2026-04-04T09:45:00Z"
}
```

### Daily Usage Series

```
GET /api/v1/billing/usage/daily
Authorization: Bearer <api-key>
```

Returns one entry per calendar day in the current billing cycle, useful for plotting usage charts:

```json
{
  "cycle_start": "2026-04-01T00:00:00Z",
  "days": [
    {
      "date": "2026-04-01",
      "compute_minutes": 220,
      "storage_gb_snapshot": 2.9,
      "runs": 11
    },
    {
      "date": "2026-04-02",
      "compute_minutes": 195,
      "storage_gb_snapshot": 3.0,
      "runs": 9
    }
  ]
}
```

Pass `?cycle=YYYY-MM` to query a previous billing cycle. Data is available for all cycles since account creation.

---

## Payment Methods

Navigate to **Settings → Billing → Payment** to manage cards.

### Adding a Card

1. Click **Add Payment Method**.
2. The Stripe Elements payment form loads inline. Enter the card number, expiry date, and CVC.
3. Click **Save**. Stripe validates the card with a $0 authorisation before storing it.

Card numbers, CVVs, and expiry dates are handled entirely by Stripe's PCI-DSS Level 1 infrastructure and are never transmitted to AO servers. The AO backend receives only a Stripe payment method ID.

### Setting the Default Card

If multiple payment methods are on file, click **Set as Default** next to the card to use for the next billing cycle.

### Removing a Card

Click **Remove** next to a card. You cannot remove the default card while an active paid subscription is running — set another card as default first.

---

## Invoices

The **Invoices** tab at **Settings → Billing → Invoices** lists all invoices in reverse chronological order:

| Column | Description |
|---|---|
| Invoice number | Stripe invoice ID |
| Billing period | Start and end dates for the cycle |
| Plan charge | Fixed plan fee for the period |
| Overage | Additional compute or storage charges |
| Total | Sum of all line items |
| Status | `paid`, `open`, or `void` |
| PDF | Download link |

Click an invoice row to see the full line-item breakdown including overage detail (per-minute rates, storage GB-days).

### Failed Payments

If a payment fails, the dashboard shows a banner and sends an email to the billing contact. AO retries the charge automatically on days 3, 7, and 14 after the initial failure. If the charge has not succeeded by day 21 the subscription is paused and cloud daemons are stopped.

To resolve a failed payment, update the payment method and click **Retry Payment** in the banner.

---

## Billing Portal

For advanced billing operations — tax ID management, billing address updates, and VAT configuration — click **Open Billing Portal** in **Settings → Billing**. This opens a Stripe-hosted portal in a new tab. Changes made in the portal are reflected in the AO dashboard within a few minutes.

---

## Cancelling a Subscription

1. Navigate to **Settings → Billing → Plan**.
2. Click **Cancel Subscription**.
3. Confirm in the modal. A brief summary shows what will be lost (e.g. "Your 4 additional agents will be removed").

Cancellation takes effect at the end of the current billing period. Until then the plan remains active at full capacity. After cancellation the organisation reverts to the Starter plan; projects and data are retained.

---

## Enterprise Billing

Enterprise customers receive invoices directly from Anthropic (the company behind AO) rather than through Stripe. Contact your account representative for:

- Custom compute allocations
- On-premises or private-cloud deployment options
- Volume discount structures
- Purchase order and NET payment terms
- Multi-year prepay

---

## Receipts and Tax

Invoices include a Stripe-generated PDF suitable for expense reports. If your organisation requires tax documentation beyond what Stripe provides, open the **Billing Portal** to add a tax ID or VAT number. Once configured, the tax ID appears on all future invoices.

---

## Related

- [Cloud Dashboard Guide](cloud-dashboard.md) — navigating the full dashboard UI
- [Cloud Deployment](cloud-deployment.md) — deploying projects with the CLI
- [Privacy & Data Policy](privacy.md) — what AO cloud stores and transmits
