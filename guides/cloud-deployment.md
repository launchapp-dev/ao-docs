# Cloud Deployment Guide

Animus cloud lets you run the Animus daemon in a managed cloud environment — no server to provision, no daemon process to babysit locally. You push your project configuration once and the cloud handles scheduling, agent execution, and log storage.

For the full command reference see [ao cloud — CLI Reference](../reference/cli/cloud.md).

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

A browser window opens. Complete the OAuth flow and return to the terminal. The session is stored in `~/.ao/cloud/credentials.json`.

**CI / non-interactive environments:** generate a personal access token in the cloud dashboard and pass it directly:

```bash
animus cloud login --token $AO_CLOUD_TOKEN
```

---

## 2 — Push the Project

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

## 3 — Start the Cloud Daemon

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

## 4 — Check Status

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

## 5 — Stream Logs

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

## 6 — Stop the Cloud Daemon

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
STATUS=$(ao cloud status --json | jq -r '.daemon_status')
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

**`animus cloud login` hangs at browser prompt in a headless environment**

Pass `--token` to skip the browser flow:

```bash
animus cloud login --token $AO_CLOUD_TOKEN
```

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
