# Cloud Getting Started Tutorial

This tutorial walks you from a new Animus account through your first cloud deployment and live agent run. By the end you will have a project running in the cloud dashboard and understand how the CLI and browser interface work together.

**Prerequisites:** Animus installed locally (`animus version` works). If you haven't installed Animus yet, see [Installation](../getting-started/installation.md) first.

---

## Step 1 — Sign Up or Accept an Invite

**New accounts:** Navigate to `app.ao.dev` and click **Sign Up**. Enter your email and create a password, then verify your email address.

**Invited to an existing organisation:** Click the link in the invite email. You will be asked to set a password and then land in the organisation you were invited to.

After sign-in you arrive at the **Projects** overview. If you just created an account it will be empty — that is expected.

---

## Step 2 — Authenticate the CLI

The CLI and dashboard share the same credentials. Run:

```bash
animus cloud login
```

Your browser opens the `app.ao.dev` authentication page. After you confirm, the CLI stores a refresh token in your system keychain. Subsequent CLI commands use this token automatically.

Verify the connection:

```bash
animus cloud status
```

Expected output:

```
Authenticated as: you@example.com
Organisation:     your-org
Plan:             Starter
```

---

## Step 3 — Initialise a Local Project

If you do not have an existing Animus project, create one now:

```bash
mkdir my-cloud-project && cd my-cloud-project
animus setup
```

`animus setup` creates the `.ao/` scaffold with a default workflow definition. You can use any existing Animus project — the cloud deployment step works the same way.

---

## Step 4 — Push to the Cloud

Deploy your project to the cloud with:

```bash
animus cloud push --env production
```

Animus packages your workflow definitions and persona configs, uploads them to the cloud, and starts a cloud daemon. Output looks like:

```
Packaging artifacts…       done
Uploading to cloud…        done (3 artifacts)
Starting daemon…           done

Deployment ID: dep_01hx…
Dashboard:     https://app.ao.dev/projects/my-cloud-project
```

Open the dashboard link. You should see your project listed with a **running** status badge.

---

## Step 5 — Explore the Dashboard

With the project running, take a quick tour:

**Projects view** — your project card shows the daemon status, last push time, and quick-action buttons (Start / Stop / Push).

**Workflow Detail** — click the project name, then click any workflow in the active workflow table. The Overview tab shows the phases and triggers from your YAML; the YAML tab shows the syntax-highlighted definition.

**Agents → Live** — any agent runs currently in progress appear here. The table refreshes every 5 seconds.

**Notification bell** — the bell in the top-right header shows unread events. Daemon start and stop events appear here automatically.

**Cmd+K** — press **Cmd+K** (macOS) or **Ctrl+K** (Windows / Linux) at any time to jump to any section by typing its name.

---

## Step 6 — Trigger an Agent Run

From the CLI, dispatch a workflow run against your cloud project:

```bash
animus cloud run --workflow my-workflow --env production
```

Switch back to the dashboard. Within a few seconds the run appears in **Agents → Live**. Click the run ID to open the Run Detail panel:

- The **phase timeline** shows each phase with a status icon.
- The **live output** area streams phase output in real time via SSE — no manual refresh needed.

When the run finishes it moves to **Agents → History**. The completed panel shows the full phase timeline, all artifacts created, and a per-phase token usage breakdown.

---

## Step 7 — Check Costs and Usage

Navigate to **Settings → Billing → Usage** to see compute minutes consumed by this run. The usage tab updates approximately every 15 minutes.

For a per-model breakdown of AI token costs, go to **Settings → Billing → Cost Breakdown** and change the **Group by** selector to **Model**. This shows how token spend is distributed across models like `claude-sonnet-4-6` and `claude-opus-4-6`.

---

## Step 8 — Set Up Notifications (optional)

To receive alerts when a daemon stops unexpectedly or a run fails:

1. Go to **Settings → Notifications**.
2. Enable **Daemon stopped unexpectedly** and **Agent run failed**.
3. Add your email (pre-filled from your account) or configure a webhook endpoint under **Settings → Notifications → Webhooks**.

Webhook payloads are signed with HMAC-SHA256; see [Cloud Dashboard Guide — Webhook Settings](../guides/cloud-dashboard.md#webhook-settings) for signature verification details.

---

## Next Steps

| Goal | Where to go |
|---|---|
| Understand deployment environments | [Cloud Deployment](../guides/cloud-deployment.md) |
| Browse pre-built workflow templates | [Workflow Template Gallery](../guides/cloud-dashboard.md#workflow-template-gallery) |
| Manage team access | [Cloud Dashboard — Team and Access](../guides/cloud-dashboard.md#team-and-access) |
| Monitor logs across runs | [Cloud Dashboard — Logs](../guides/cloud-dashboard.md#logs) |
| Automate pushes in CI/CD | [CI/CD Guide](../guides/ci-cd.md) |
| Understand billing and model costs | [Cloud Billing Guide](../guides/cloud-billing.md) |

---

## Related

- [Cloud Dashboard Guide](../guides/cloud-dashboard.md) — complete reference for every dashboard section
- [Cloud Deployment](../guides/cloud-deployment.md) — full `animus cloud push` and environment management reference
- [Cloud Billing Guide](../guides/cloud-billing.md) — plans, usage metrics, and cost breakdown
- [Quick Start](../getting-started/quick-start.md) — local-only Animus setup without cloud
