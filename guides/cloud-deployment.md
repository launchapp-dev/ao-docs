# Cloud Deployment Guide

AO cloud lets you run the AO daemon in a managed cloud environment — no server to provision, no daemon process to babysit locally. You push your project configuration once and the cloud handles scheduling, agent execution, and log storage.

For the full command reference see [ao cloud — CLI Reference](../reference/cli/cloud.md).

---

## Prerequisites

- AO CLI installed (`ao version` should return `0.x.x` or later)
- An AO cloud account — sign up at the AO website
- Your project initialised locally (`ao setup` or an existing `.ao/` directory)

---

## 1 — Log In

Authenticate the CLI against your AO cloud account:

```bash
ao cloud login
```

A browser window opens. Complete the OAuth flow and return to the terminal. The session is stored in `~/.ao/cloud/credentials.json`.

**CI / non-interactive environments:** generate a personal access token in the cloud dashboard and pass it directly:

```bash
ao cloud login --token $AO_CLOUD_TOKEN
```

---

## 2 — Push the Project

Upload your project's workflow definitions, personas, and pack configuration to the cloud:

```bash
ao cloud push
```

`ao cloud push` packages `.ao/` metadata — it does **not** upload source code. Your repository stays local (or in your own VCS).

Validate what would be pushed without uploading:

```bash
ao cloud push --dry-run
```

Target a non-production environment:

```bash
ao cloud push --env staging
```

---

## 3 — Start the Cloud Daemon

Once the project is pushed, start the remote daemon:

```bash
ao cloud start
```

Use `--wait` to block until the daemon is fully running before returning:

```bash
ao cloud start --wait
```

The daemon is now active in the cloud and will pick up work as soon as tasks are ready.

---

## 4 — Check Status

Verify the deployment is healthy:

```bash
ao cloud status
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
ao cloud status --json | jq '.daemon_status'
```

---

## 5 — Stream Logs

Follow live output from the cloud daemon:

```bash
ao cloud logs --follow
```

Show the last 200 lines and then stream:

```bash
ao cloud logs --tail 200 --follow
```

Filter to errors only:

```bash
ao cloud logs --level error
```

Scope to a single agent run:

```bash
ao cloud logs --run-id run_1234
```

Fetch logs from the past hour as newline-delimited JSON:

```bash
ao cloud logs --since 1h --json
```

---

## 6 — Stop the Cloud Daemon

Gracefully drain active agents and stop:

```bash
ao cloud stop
```

Force-terminate immediately (skips drain):

```bash
ao cloud stop --force
```

---

## Common Workflows

### Deploy and Start in One Step

```bash
ao cloud push && ao cloud start --wait
```

### Rolling Redeploy

Push updated configuration and restart the daemon:

```bash
ao cloud push
ao cloud stop --wait
ao cloud start --wait
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
ao cloud logs --follow --level error --since 10m
```

---

## Environments

AO cloud supports isolated environments on the same project. Pass `--env` to any command:

| Environment | Use |
|---|---|
| `production` | Default. Live work dispatched to real agents. |
| `staging` | Pre-production validation. Shares no state with production. |
| `preview` | Ephemeral — created per branch or PR. |

Switching between environments:

```bash
ao cloud push --env staging
ao cloud start --env staging
ao cloud status --env staging
```

---

## Troubleshooting

**`ao cloud login` hangs at browser prompt in a headless environment**

Pass `--token` to skip the browser flow:

```bash
ao cloud login --token $AO_CLOUD_TOKEN
```

**`ao cloud push` fails with "no project found"**

Ensure you are running from a directory with an `.ao/` folder or use `--project-root`:

```bash
ao cloud push --project-root /path/to/project
```

**Daemon stuck in `starting`**

Check logs immediately after issuing `ao cloud start`:

```bash
ao cloud logs --since 5m --level warn
```

A missing model API key or misconfigured MCP server is a common cause. See [Troubleshooting](troubleshooting.md) for further diagnostics.

**`ao cloud stop` does not complete**

If the daemon is unresponsive, force-stop:

```bash
ao cloud stop --force
```

---

## Related

- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full flag and output reference
- [Daemon Operations](daemon-operations.md) — local daemon equivalent
- [Fleet Management](fleet-management.md) — multi-node distributed deployments
- [Privacy & Data Policy](privacy.md) — what AO cloud does and does not store
