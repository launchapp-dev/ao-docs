# Demo Mode

Demo mode lets you explore Animus Cloud with a fully populated sandbox environment — no GitHub account, no API keys, and no billing information required. It is the fastest way to see what the dashboard looks like with real-looking data before committing to an account.

Demo mode was introduced in v43.

---

## Accessing Demo Mode

Navigate to `app.ao.dev/demo`. No sign-in is required. The dashboard loads immediately in a shared, read-only sandbox environment pre-populated with sample projects, agent runs, and workflow definitions.

Alternatively, click **Try the demo** on the marketing homepage or the sign-in page.

---

## What Demo Mode Includes

The demo environment contains:

| Section | Demo content |
|---|---|
| **Projects** | Three sample projects: `api-service`, `frontend-app`, `data-pipeline` |
| **Daemon** | Two daemons in `running` status (one per project) |
| **Agents** | Five active runs and a history of 30 completed runs across the projects |
| **Logs** | Realistic structured log lines for the last 72 hours |
| **Workflows** | Four workflow definitions: `pr-review`, `test-gen`, `deploy-staging`, `incident-triage` |
| **Template Gallery** | Full template library with preview mode enabled |
| **Billing** | Sample Pro plan with illustrative usage charts |
| **Audit Log** | 14 days of sample audit events |
| **GitHub Checks** | Sample check runs linked to a mock repository |

All data is synthetic and refreshes every 24 hours to a consistent baseline. Timestamps are relative to the current time so that charts and log views always look live.

---

## Limitations of Demo Mode

Demo mode is intentionally restricted:

| Action | Available in demo? |
|---|---|
| Browse all dashboard sections | Yes |
| View workflow YAML | Yes (read-only) |
| Preview template gallery entries | Yes |
| Deploy a template | No |
| Start or stop a daemon | No |
| Push a deployment | No |
| Create or revoke API keys | No |
| Invite members | No |
| Change billing plan | No |
| View real log output | No (all data is synthetic) |

Attempting a restricted action shows a modal explaining the limitation and offering a direct link to sign up.

---

## Demo Mode Banners

While in demo mode, a persistent banner at the top of every page reads:

> **You are viewing a demo environment.** Data is synthetic and read-only. [Sign up free →]

The banner cannot be dismissed. It does not appear once you sign in to a real account.

---

## Converting to a Real Account

Click **Sign up free** in any demo mode prompt, the persistent banner, or the sign-in link in the top-right corner.

The sign-up flow takes you through:

1. **Email and password** — or continue with GitHub OAuth.
2. **Organisation name** — sets your `app.ao.dev/<org>` URL slug.
3. **Onboarding wizard** — guides you through creating your first project, installing the CLI, and pushing your initial deployment.

Your demo session is not migrated; the onboarding wizard starts you with an empty organisation. See [Cloud Getting Started](../tutorials/cloud-getting-started.md) for the full setup walkthrough.

---

## Running a Local Demo

For offline exploration or a self-hosted evaluation, you can run a local demo instance:

```bash
animus demo start
```

This starts a local server at `http://localhost:4200` pre-loaded with the same synthetic dataset used by the hosted demo. The local demo supports all read and write operations but operates entirely in memory — data does not persist between restarts.

Stop the local demo with:

```bash
animus demo stop
```

The local demo requires Animus version 0.43.0 or later. It does not require an Animus Cloud account or internet access.

### Local Demo Flags

| Flag | Default | Description |
|---|---|---|
| `--port` | `4200` | Port to listen on |
| `--seed` | `demo` | Seed string for the synthetic data generator — change to get different project names and run IDs |
| `--no-open` | `false` | Skip auto-opening the browser |

---

## Demo Mode and the CLI

The `animus demo` command group manages local demo sessions:

```bash
animus demo start            # Start local demo server
animus demo stop             # Stop local demo server
animus demo status           # Show whether the local demo is running
animus demo reset            # Wipe in-memory state and reinitialise from seed
```

The local demo exposes the same REST API as Animus Cloud, so you can point `animus cloud login` at it for CLI exploration:

```bash
animus cloud login --host http://localhost:4200
```

All `animus cloud` commands — `push`, `start`, `stop`, `logs` — work against the local demo. Changes are in-memory only.

---

## Related

- [Cloud Getting Started](../tutorials/cloud-getting-started.md) — end-to-end walkthrough for real accounts
- [Cloud Dashboard](cloud-dashboard.md) — full dashboard feature reference
- [Animus Cloud Beta Signup](cloud-beta-signup.md) — joining the cloud beta
- [Cloud Deployment](cloud-deployment.md) — pushing your first real project
