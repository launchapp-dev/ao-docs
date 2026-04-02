# CLI Sync Protocol Reference

This document specifies the protocol used by `ao cloud push` and `ao cloud sync` to transfer project configuration from a local checkout to the AO cloud service. It is intended for users who need to automate deployments, debug push failures, or integrate AO cloud into custom tooling.

For the end-user deployment workflow see [Cloud Deployment](../guides/cloud-deployment.md). For the CLI flag reference see [ao cloud — CLI Reference](cli/cloud.md).

---

## Overview

The sync protocol is a single-round-trip HTTP upload. The CLI packages project metadata into a **sync bundle**, sends it to the cloud ingestion endpoint with an authenticated request, and receives a deployment receipt. No source code is included in the bundle.

```
┌─────────────────────────────────────────────────────────────────┐
│  ao cloud push                                                  │
│                                                                 │
│  1. Collect           2. Package          3. Upload             │
│  ─────────────────    ───────────────     ──────────────────    │
│  .ao/workflows/       gzipped tar         HTTPS POST           │
│  .ao/config.json      (bundle.tar.gz)     /v1/projects/{id}/   │
│  personas/                                deployments          │
│  packs (metadata)                                              │
│                                                                 │
│  4. Receive receipt                                             │
│  ────────────────                                               │
│  deployment_id        version             artifacts list        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Bundle Contents

A sync bundle is a gzipped tar archive (`application/x-tar+gzip`) containing the following paths, all relative to the project root:

| Path | Required | Description |
|---|---|---|
| `.ao/config.json` | Yes | Project identity and version |
| `.ao/workflows/*.yaml` | Yes (at least one) | Workflow definitions |
| `.ao/personas/*.yaml` | No | Agent persona overrides |
| `.ao/packs.yaml` | No | Pack dependency manifest |
| `.ao/env.yaml` | No | Non-secret environment variable overrides |

Source files, secrets, and the `.ao/state/` directory are never included. If a `.ao/sync-ignore` file exists in the project root, it is interpreted as a gitignore-format exclusion list applied to the above paths.

### `.ao/sync-ignore`

```
# Exclude environment-specific persona overrides
personas/local-*.yaml

# Exclude preview workflow definitions
workflows/preview-*.yaml
```

---

## Authentication

Every request to the ingestion API requires a Bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

The token is read from `~/.ao/cloud/credentials.json` (written by `ao cloud login`). For CI environments, set the `AO_CLOUD_TOKEN` environment variable; the CLI prefers the environment variable over the credentials file.

Token scopes required for push operations:

| Scope | Purpose |
|---|---|
| `deployments:write` | Create new deployments |
| `projects:read` | Resolve the project ID from the slug |

---

## Upload Endpoint

```
POST /v1/projects/{project_slug}/deployments
Host: api.ao.dev
Authorization: Bearer <token>
Content-Type: application/x-tar+gzip
X-AO-Env: production
X-AO-Bundle-Hash: <sha256-hex>
X-AO-CLI-Version: <semver>
```

### Path Parameters

| Parameter | Description |
|---|---|
| `project_slug` | URL-safe project identifier, taken from `.ao/config.json` → `project_name` |

### Request Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes | Bearer token (see Authentication above) |
| `Content-Type` | Yes | Must be `application/x-tar+gzip` |
| `X-AO-Env` | No | Target environment. Default: `production` |
| `X-AO-Bundle-Hash` | Yes | SHA-256 hex digest of the bundle body, for integrity verification |
| `X-AO-CLI-Version` | Yes | CLI version string, e.g. `0.5.0` |

### Request Body

The raw gzipped tar archive of the sync bundle described above.

---

## Response

### Success — 201 Created

```json
{
  "schema": "ao.cli.v1",
  "ok": true,
  "data": {
    "deployment_id": "dep_01HXYZ",
    "env": "production",
    "pushed_at": "2026-04-01T10:00:00Z",
    "artifacts": 12,
    "version": 7,
    "url": "https://app.ao.dev/acme/my-project/deployments/dep_01HXYZ"
  }
}
```

### Error Responses

| HTTP Status | Error Code | Cause |
|---|---|---|
| `400 Bad Request` | `invalid_bundle` | Bundle failed integrity check or is malformed |
| `400 Bad Request` | `missing_config` | `.ao/config.json` not found in bundle |
| `400 Bad Request` | `invalid_workflow` | A workflow YAML failed schema validation |
| `401 Unauthorized` | `unauthenticated` | Token missing or invalid |
| `403 Forbidden` | `insufficient_scope` | Token lacks `deployments:write` scope |
| `404 Not Found` | `project_not_found` | No project with the given slug in this organisation |
| `409 Conflict` | `env_locked` | Target environment is locked (e.g. a deploy is in progress) |
| `413 Content Too Large` | `bundle_too_large` | Bundle exceeds the 50 MB limit |
| `429 Too Many Requests` | `rate_limited` | Push rate limit exceeded (10 pushes per minute per project) |

All error responses follow the `ao.cli.v1` envelope:

```json
{
  "schema": "ao.cli.v1",
  "ok": false,
  "error": {
    "code": "invalid_workflow",
    "message": "Workflow 'nightly' references undefined phase 'deploy'",
    "details": {
      "workflow": "nightly",
      "phase": "deploy"
    }
  }
}
```

---

## Versioning

Each successful push increments the deployment version for the target environment. Versions are monotonically increasing integers scoped to `(project, environment)`. The `version` field in the response body reflects the new version number.

The cloud daemon always runs the configuration from the latest version for its environment. Rolling back to a previous deployment is not supported via CLI; contact support or use the dashboard's deployment history to reinstate an earlier bundle.

---

## Idempotency

Pushing the same bundle twice (same `X-AO-Bundle-Hash`) to the same environment within 60 seconds returns the existing deployment receipt rather than creating a new deployment. This window prevents duplicate deployments from transient retries.

If you intentionally want to force a re-push of identical content (e.g. to refresh environment variable bindings), use `ao cloud push --force`, which bypasses the idempotency check.

---

## Dry-Run Mode

When `ao cloud push --dry-run` is invoked, the CLI packages the bundle locally and sends it to the validation endpoint instead of the deployment endpoint:

```
POST /v1/projects/{project_slug}/deployments/validate
```

The request and response formats are identical to the live endpoint, except the response body contains `"dry_run": true` and no deployment record is created. Use this in CI to verify a push would succeed without side effects.

---

## Sync Events

After a successful push the cloud service emits a `deployment.created` event. If a daemon is running in the target environment, it receives this event via its persistent WebSocket connection to the cloud control plane and hot-reloads its configuration without restarting.

Event payload (WebSocket, JSON):

```json
{
  "event": "deployment.created",
  "deployment_id": "dep_01HXYZ",
  "env": "production",
  "version": 7,
  "ts": "2026-04-01T10:00:00Z"
}
```

The daemon acknowledges the event and enters a brief `reloading` state (visible in `ao cloud status`) while it validates and applies the new configuration.

---

## Rate Limits

| Limit | Value |
|---|---|
| Pushes per minute per project | 10 |
| Bundle size | 50 MB |
| Maximum artifacts in one bundle | 500 files |
| Concurrent pushes per organisation | 5 |

The `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers are present on every response.

---

## CLI Implementation Notes

The CLI build packages the sync protocol into the `orchestrator-cloud-sync` crate. Key entry points for contributors:

| Symbol | Location | Description |
|---|---|---|
| `SyncBundle::from_project` | `cloud-sync/src/bundle.rs` | Traverses `.ao/`, applies `.ao/sync-ignore`, builds tar |
| `CloudSyncClient::push` | `cloud-sync/src/client.rs` | Authenticates, uploads, parses receipt |
| `PushCommand::run` | `cli/src/commands/cloud/push.rs` | CLI adapter: dry-run flag, output formatting |

---

## Related

- [Cloud Deployment](../guides/cloud-deployment.md) — end-user push workflow
- [ao cloud — CLI Reference](cli/cloud.md) — `ao cloud push` flags
- [Configuration](configuration.md) — `.ao/config.json` schema
- [Workflow YAML Schema](workflow-yaml.md) — validated by the sync endpoint
- [JSON Envelope Contract](json-envelope.md) — response envelope format
