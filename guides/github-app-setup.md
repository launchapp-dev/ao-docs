# GitHub App Registration and Setup

This guide walks through installing the Animus GitHub App from scratch, configuring organisation or repository permissions, and completing the connection to Animus Cloud. By the end you will have GitHub events flowing into your AO project and a working webhook receiver.

For a reference of all GitHub App features (repository selector filters, delivery log, trigger syntax, posting check runs), see [GitHub App Integration](github-app.md).

---

## Prerequisites

**Animus Cloud account**

- You must be logged in to Animus Cloud. If you have not set up an account yet, complete [Cloud Deployment](cloud-deployment.md) first.
- Your role in the AO Cloud organisation must be **Admin** or higher. See [Cloud Dashboard — Team and Access](cloud-dashboard.md#team-and-access) to check your role.

**GitHub account**

- You must be an **Owner** of the GitHub organisation you want to connect, or the **owner** of your personal GitHub account.
- If you are installing on a GitHub organisation owned by someone else, ask them to complete this guide or to grant you Owner role first.

**AO project**

- You need at least one AO project to connect the app to. Create one with:

```bash
ao project create my-project
ao cloud push
```

---

## Step 1 — Open the Integration Settings

1. Log in to the Animus Cloud dashboard.
2. In the left navigation, click **Settings**.
3. Click **Integrations**.
4. Click the **GitHub** tab.

If no GitHub installations exist yet, the page shows an empty state with an **Install GitHub App** button.

---

## Step 2 — Start the GitHub App Installation

Click **Install GitHub App**. You are redirected to GitHub's app installation page.

GitHub asks you to choose where to install the app:

| Option | When to choose |
|---|---|
| **Your personal account** | You want to connect personal repositories (e.g. `your-username/repo`) |
| **An organisation** | You want to connect repositories belonging to a GitHub organisation |

Select the account and click **Install** (or **Install & Authorize** if prompted).

> Only install on accounts where you have Owner or Admin rights. If the organisation you need is not listed, ask an Owner to add you.

---

## Step 3 — Configure Repository Access

GitHub then asks which repositories the app should have access to:

| Mode | Behaviour |
|---|---|
| **All repositories** | Animus receives events from every current and future repository in the account |
| **Only select repositories** | Animus receives events only from the repositories you choose now |

**Recommendation:** Choose **Only select repositories** to start. You can add more repositories at any time without reinstalling the app.

Use the search box to find and select the repositories you want to connect. Click **Install & Authorize** when done.

---

## Step 4 — Review the Permissions

Before confirming, GitHub shows the permissions the Animus GitHub App requests:

| Permission | Level | Purpose |
|---|---|---|
| Contents | Read | Read repository files for workflow context |
| Pull requests | Read & Write | Open, update, and comment on pull requests |
| Issues | Read & Write | Create and update issues from workflow output |
| Checks | Read & Write | Publish check run results |
| Statuses | Read & Write | Post commit statuses |
| Webhooks | Read | Confirm webhook delivery health |
| Metadata | Read | Required by GitHub for all apps |

Animus does **not** request write access to repository contents, admin settings, deploy keys, or secrets.

Click **Install & Authorize** to confirm.

---

## Step 5 — Complete the Cloud Connection

After you confirm on GitHub, you are redirected back to the Animus Cloud dashboard. AO:

1. Receives the temporary `installation_id` from GitHub's redirect.
2. Exchanges it for a long-lived installation token.
3. Stores the mapping between the installation and your AO Cloud organisation.
4. Displays a success state on the **Settings → Integrations → GitHub** page.

The Integrations page now shows your installation:

| Field | Description |
|---|---|
| Account | The GitHub organisation or user account name |
| Installation ID | GitHub-assigned identifier (save this if you open a support ticket) |
| Status | `active`, `suspended`, or `revoked` |
| Installed at | Timestamp of the initial installation |
| Repositories | Number of repositories currently selected |

If the redirect does not complete (browser closed, session expired), return to **Settings → Integrations → GitHub** and click **Retry Installation**. GitHub reuses the existing `installation_id` for 10 minutes after the initial install.

---

## Step 6 — Activate Repositories in Your Project

The GitHub App installation grants AO access to repositories at the account level. You must also **activate** each repository within your AO project before events from that repository can trigger workflows.

### Activate via dashboard

1. Open your project in the Animus Cloud dashboard.
2. Go to **Settings → Repositories**.
3. The **Available Repositories** list shows all repositories accessible through your GitHub App installation.
4. Click **Activate** next to each repository you want to enable.

### Activate via CLI

```bash
# List all repositories accessible through your installations
ao cloud repo list

# Activate a repository for the current project
ao cloud repo activate my-org/my-repo
```

A repository must be both accessible (via the installation) and activated (within a project) for triggers to fire.

---

## Step 7 — Configure the Webhook

Animus Cloud manages the webhook receiver automatically — you do not need to create a webhook URL or generate a signing secret. When you installed the GitHub App, GitHub began forwarding events to AO's managed receiver endpoint.

To verify webhook delivery is working:

1. Go to **Settings → Integrations → GitHub → [your installation] → Deliveries**.
2. Make any change in a connected repository (push a commit, open an issue, etc.).
3. Refresh the Deliveries log. A new row should appear with status `accepted` or `filtered`.

| Status | Meaning |
|---|---|
| `accepted` | Signature valid; trigger evaluation ran |
| `filtered` | Valid event; no trigger matched (no trigger configured, or repository not selected) |
| `error` | Signature validation failed or internal error |

If the Deliveries log is empty after making a change, see [Troubleshooting](#troubleshooting) below.

---

## Step 8 — Write Your First Trigger

With the installation active and repositories connected, declare a trigger in `.ao/triggers.yaml` to dispatch a workflow when a GitHub event occurs:

```yaml
# .ao/triggers.yaml
triggers:
  - id: on-pr-opened
    kind: github
    github:
      installation: my-org          # GitHub account name from your installation
      repository: my-org/my-repo    # omit to match all repos in the installation
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
        head_sha: "{{ payload.pull_request.head.sha }}"
```

Push the file to your repository:

```bash
git add .ao/triggers.yaml
git commit -m "feat: add PR review trigger"
git push
```

AO picks up the change automatically. Confirm the trigger is registered:

```bash
ao trigger list
```

For full trigger field reference and supported event types, see [GitHub App Integration — Configuring Triggers](github-app.md#configuring-triggers-for-github-events).

---

## Changing Permissions After Installation

### Add or remove repositories

1. Go to **Settings → Integrations → GitHub** in the Animus Cloud dashboard.
2. Click the **edit** icon next to the installation.
3. Click **Configure on GitHub**.
4. Under **Repository access**, add or remove repositories.
5. Click **Save**. GitHub sends an `installation_repositories` event to AO and the dashboard updates within a few seconds.

You can also navigate directly to GitHub's installation settings:

```
https://github.com/settings/installations/<installation_id>
```

For organisations:

```
https://github.com/organizations/<org-name>/settings/installations/<installation_id>
```

### Expand to all repositories

If you started with selected repositories and want to switch to all repositories:

1. Open the GitHub installation settings page (link above).
2. Under **Repository access**, select **All repositories**.
3. Click **Save**.

### Suspend or revoke the installation

To temporarily pause event delivery without uninstalling, go to the GitHub installation settings and click **Suspend**.

To remove the integration entirely from Animus Cloud, go to **Settings → Integrations → GitHub** and click **Revoke**. This removes the stored token but does not uninstall the app from GitHub. To fully uninstall, also visit GitHub's installation settings and click **Uninstall**.

---

## Troubleshooting

### No entries in the Deliveries log

- Confirm the repository is listed under **Settings → Integrations → GitHub → [installation] → Repositories**.
- Check that the installation status is `active`, not `suspended`.
- On GitHub, navigate to the installation settings page and verify the app is still installed with the expected permissions.
- Wait 30–60 seconds after triggering the event — webhook delivery can be slightly delayed during high-traffic periods.

### Status is `filtered` for every event

- Verify that the repository is activated in your project (**Settings → Repositories**) and its status is `active`.
- Check that your trigger's `github.installation` value matches the GitHub account name exactly (case-sensitive).
- Ensure the `github.events` list includes the specific event and action you are testing (use `pull_request.opened`, not just `pull_request`, to target the opened action).
- If `github.repository` is set, it must match the `owner/repo` slug exactly.

### Installation redirect fails or times out

GitHub's OAuth redirect window is 10 minutes. If you see an error after being redirected back:

1. Return to **Settings → Integrations → GitHub**.
2. Click **Install GitHub App** again.
3. GitHub reuses the existing installation if one was already created; you will not create a duplicate.

### Redirect succeeds but status remains `pending`

If the dashboard shows a `pending` status for more than 30 seconds:

1. Hard-refresh the page (`Cmd+Shift+R` / `Ctrl+Shift+R`).
2. If the status does not update, click **Retry Installation** on the integrations page.
3. If the issue persists, check AO Cloud service status at [status.ao.dev](https://status.ao.dev).

### Trigger fires but workflow does not start

- Confirm the daemon is running: `ao daemon status` (local) or check the **Daemon** panel in the cloud dashboard.
- Run `ao trigger list` and check that the trigger status is `active`, not `paused`.
- Open **Settings → Integrations → GitHub → [installation] → Deliveries**, find the relevant delivery, and click through to the linked dispatch. The dispatch detail shows any startup errors.

### Cannot see the expected organisation in GitHub's install flow

You must be an Owner of the organisation. If the org is not listed, ask a current Owner to either install the app themselves or to grant you Owner role.

---

## Next Steps

- [GitHub App Integration](github-app.md) — Full reference: repository selector filters, delivery log, payload interpolation, check run posting
- [GitHub Checks](github-checks.md) — Publishing check run results from AO workflows
- [Event Triggers](event-triggers.md) — All trigger kinds: file watchers, webhooks, and GitHub events
- [Writing Custom Workflows](writing-workflows.md) — Authoring the workflows that triggers dispatch
