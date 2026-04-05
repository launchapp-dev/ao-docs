# OpenAPI / Swagger UI

The Animus local web server ships an interactive OpenAPI explorer at `/api/docs`. When the web server is running you can browse every REST endpoint, read request and response schemas, and execute requests directly from the browser — no separate API client required.

For a general overview of the web server see [Web Dashboard](../guides/web-dashboard.md). For the raw JSON envelope format see [JSON Envelope Contract](json-envelope.md).

---

## Accessing the UI

Start the web server (default port `3000`):

```bash
animus web serve
```

Then open the Swagger UI in your browser:

```
http://localhost:3000/api/docs
```

Pass a custom port when the default is occupied:

```bash
animus web serve --port 8080
# UI is at http://localhost:8080/api/docs
```

`animus web open` opens the dashboard root. Navigate to `/api/docs` manually or append it to the URL printed by the serve command.

---

## Raw OpenAPI Spec

The machine-readable OpenAPI 3.1 specification is served alongside the UI:

```
GET http://localhost:3000/api/docs/openapi.json
```

Use this URL with any OpenAPI-compatible toolchain — code generators, Postman, Insomnia, or your own documentation pipeline:

```bash
# Save a local copy
curl http://localhost:3000/api/docs/openapi.json -o ao-openapi.json

# Generate a TypeScript client (example using openapi-ts)
npx @hey-api/openapi-ts -i ao-openapi.json -o src/client -c @hey-api/client-fetch
```

---

## Authentication

The REST API accepts a bearer token issued by `animus cloud login` or generated in the cloud dashboard.

In the Swagger UI, click **Authorize** (padlock icon) at the top-right and enter:

```
Bearer <your-token>
```

For local development the daemon may run without authentication when `auth.enabled` is `false` in the project configuration. Check `animus config show` for the active setting.

---

## Endpoint Groups

The OpenAPI spec organises endpoints into the following tags. Each tag maps to a section in the Swagger UI sidebar.

| Tag | Path prefix | Description |
|---|---|---|
| **Tasks** | `/api/v1/tasks` | CRUD operations for tasks |
| **Workflows** | `/api/v1/workflows` | Workflow listing, detail, and phase history |
| **Requirements** | `/api/v1/requirements` | Requirement records and status transitions |
| **Daemon** | `/api/v1/daemon` | Daemon health, status, and control signals |
| **Events** | `/api/v1/events` | Server-Sent Events stream for real-time updates |

---

## Using the Try-It-Out Feature

1. Expand any endpoint entry in the Swagger UI.
2. Click **Try it out**.
3. Fill in path parameters, query parameters, and (for `POST`/`PATCH`) the request body.
4. Click **Execute**.

The UI shows the raw `curl` command, the full HTTP response code, response headers, and the JSON body. Responses follow the standard `ao.cli.v1` envelope:

```json
{
  "schema": "ao.cli.v1",
  "ok": true,
  "data": { ... }
}
```

Error responses use the same envelope with `"ok": false` and an `error` object. See [JSON Envelope Contract](json-envelope.md) for the complete schema.

---

## Filtering and Pagination

List endpoints (`GET /api/v1/tasks`, `GET /api/v1/workflows`, etc.) accept query parameters documented in the OpenAPI spec. Common parameters:

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by status value (see [Status Values](status-values.md)) |
| `priority` | string | Filter tasks by priority (`low`, `medium`, `high`, `critical`) |
| `limit` | integer | Maximum number of results (default: `50`, max: `200`) |
| `offset` | integer | Pagination offset |

Example:

```
GET /api/v1/tasks?status=in-progress&priority=high&limit=20
```

---

## Cloud vs. Local

| | Local web server | Animus Cloud |
|---|---|---|
| URL | `http://localhost:<port>/api/docs` | `https://app.ao.dev/api/docs` |
| Auth | Optional (config-controlled) | Bearer token required |
| Scope | Single project on local daemon | Organisation-wide across all projects |
| Spec URL | `http://localhost:<port>/api/docs/openapi.json` | `https://app.ao.dev/api/docs/openapi.json` |

The cloud dashboard exposes the same OpenAPI UI and spec for the cloud REST API, which extends the local surface with multi-project and organisation-scoped endpoints.

---

## Related

- [Web Dashboard Guide](../guides/web-dashboard.md) — starting the web server, dashboard features, SSE events
- [JSON Envelope Contract](json-envelope.md) — request/response envelope schema
- [Status Values & Enums](status-values.md) — accepted `status` values for filtering
- [Cloud Dashboard Guide](../guides/cloud-dashboard.md) — cloud-hosted web UI at `app.ao.dev`
- [Cloud CLI Reference](cli/cloud.md) — `animus cloud` commands for deployment and daemon control
