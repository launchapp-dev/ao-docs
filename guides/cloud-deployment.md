# Cloud Deployment Guide

Animus cloud lets you run the Animus daemon in a managed cloud environment — no server to provision, no daemon process to babysit locally. You push your project configuration once and the cloud handles scheduling, agent execution, and log storage.

For the full command reference see [animus cloud — CLI Reference](../reference/cli/cloud.md).

---

## Prerequisites

- Animus CLI installed (`animus version` should return `0.x.x` or later)
- An Animus cloud account — sign up at the Animus website
- Your project initialised locally (`animus setup` or an existing `.ao/` directory)

---

## 1 — Log In

Authenticate the CLI against your Animus cloud account:

```bash
animus cloud login
```

`ao cloud login` uses a **browser-based OAuth flow**: the CLI starts a local server on port `19823`, opens your system browser to Animus Cloud, completes GitHub OAuth, and captures the session token automatically.

```bash
# Default — opens system browser
ao cloud login

# Headless / remote — print URL, complete login from another device
ao cloud login --no-browser

# Self-hosted Animus Cloud
ao cloud login --server https://your-cloud.example.com
```

Credentials are stored in `~/.ao/cloud.json`. Re-run `ao cloud login` to refresh an expired session.

---

## 2 — Link the Project

Link the current project to an Animus Cloud project. This only needs to be done once per checkout:

```bash
animus cloud link
```

Animus reads the `origin` git remote, matches it against cloud projects in your organisation, and stores the link in `.ao/config.json`. All subsequent `push`, `deploy`, and `status` commands use the stored link automatically.

If auto-detection fails (no git remote, or ambiguous match), specify the project slug explicitly:

```bash
animus cloud link --project my-project
```

---

## 3 — Push the Project

Upload your project's workflow definitions, personas, and pack configuration to the cloud:

```bash
animus cloud push
```

`animus cloud push` packages `.ao/` metadata — it does **not** upload source code. Your repository stays local (or in your own VCS).

Validate what would be pushed without uploading:

```bash
animus cloud push --dry-run
```

Target a non-production environment:

```bash
animus cloud push --env staging
```

---

## 4 — Start the Cloud Daemon

Once the project is pushed, start the remote daemon:

```bash
animus cloud start
```

Use `--wait` to block until the daemon is fully running before returning:

```bash
animus cloud start --wait
```

The daemon is now active in the cloud and will pick up work as soon as tasks are ready.

---

## 5 — Check Status

Verify the deployment is healthy:

```bash
animus cloud status
```

Sample output:

```
deployment  dep_01HXYZ   pushed 2026-04-01 10:00 UTC
daemon      running      started 2026-04-01 10:05 UTC
active      2 agents     queue depth 0
region      us-east-1
```

Use `--json` for machine-readable output, e.g. in a deploy script:

```bash
animus cloud status --json | jq '.daemon_status'
```

---

## 6 — Stream Logs

Follow live output from the cloud daemon:

```bash
animus cloud logs --follow
```

Show the last 200 lines and then stream:

```bash
animus cloud logs --tail 200 --follow
```

Filter to errors only:

```bash
animus cloud logs --level error
```

Scope to a single agent run:

```bash
animus cloud logs --run-id run_1234
```

Fetch logs from the past hour as newline-delimited JSON:

```bash
animus cloud logs --since 1h --json
```

---

## 7 — Stop the Cloud Daemon

Gracefully drain active agents and stop:

```bash
animus cloud stop
```

Force-terminate immediately (skips drain):

```bash
animus cloud stop --force
```

---

## Common Workflows

### Deploy and Start in One Step

Use `animus cloud deploy` as a single command that runs `push` followed by `start`:

```bash
animus cloud deploy
animus cloud deploy --wait           # block until daemon reports running
```

Or manually:

```bash
animus cloud push && animus cloud start --wait
```

### Rolling Redeploy

Push updated configuration and restart the daemon:

```bash
animus cloud push
animus cloud stop --wait
animus cloud start --wait
```

### Check Daemon Health in CI

```bash
STATUS=$(animus cloud status --json | jq -r '.daemon_status')
if [ "$STATUS" != "running" ]; then
  echo "Cloud daemon is $STATUS — aborting" >&2
  exit 1
fi
```

### Tail Errors After a Deploy

```bash
animus cloud logs --follow --level error --since 10m
```

---

## Environments

Animus cloud supports isolated environments on the same project. Pass `--env` to any command:

| Environment | Use |
|---|---|
| `production` | Default. Live work dispatched to real agents. |
| `staging` | Pre-production validation. Shares no state with production. |
| `preview` | Ephemeral — created per branch or PR. |

Switching between environments:

```bash
animus cloud push --env staging
animus cloud start --env staging
animus cloud status --env staging
```

---

## Troubleshooting

**`ao cloud login` hangs or fails in a headless environment**

`ao cloud login` opens a system browser to complete GitHub OAuth. On headless servers (no browser available), use `--no-browser` to print the URL instead:

```bash
ao cloud login --no-browser
```

Copy the printed URL, open it in a browser on any device, complete GitHub login, and the CLI will receive the token automatically on port `19823`. Ensure port `19823` is reachable on your machine (not blocked by a firewall).

**`animus cloud push` fails with "no project found"**

Ensure you are running from a directory with an `.ao/` folder or use `--project-root`:

```bash
animus cloud push --project-root /path/to/project
```

**Daemon stuck in `starting`**

Check logs immediately after issuing `animus cloud start`:

```bash
animus cloud logs --since 5m --level warn
```

A missing model API key or misconfigured MCP server is a common cause. See [Troubleshooting](troubleshooting.md) for further diagnostics.

**`animus cloud stop` does not complete**

If the daemon is unresponsive, force-stop:

```bash
animus cloud stop --force
```

---

## Related

- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full flag and output reference
- [Daemon Operations](daemon-operations.md) — local daemon equivalent
- [Fleet Management](fleet-management.md) — multi-node distributed deployments
- [Privacy & Data Policy](privacy.md) — what Animus cloud does and does not store
