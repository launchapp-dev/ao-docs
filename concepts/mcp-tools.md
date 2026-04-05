# MCP Integration

## What MCP Is

MCP is Animus's tool boundary. Agents and workflows use MCP to read and mutate
state, and packs can contribute additional MCP server descriptors without
teaching the daemon new behavior.

## Animus's Core MCP Surface

Animus ships an MCP server:

```bash
animus mcp serve
```

It exposes Animus mutation and query tools such as:

- `ao.task.*`
- `ao.requirements.*`
- `ao.workflow.*`
- `ao.daemon.*`

Many of those tools are now conceptually owned by bundled first-party packs
such as `ao.task` and `ao.requirement`, even though they are exposed through
the Animus MCP server.

## MCP Server Transports

Workflow YAML supports two transport types for `mcp_servers` entries.

**stdio** (default) — Animus spawns the server as a local subprocess and
communicates over stdin/stdout. Use for any MCP server distributed as a CLI
package:

```yaml
mcp_servers:
  ao:
    command: ao
    args: ["mcp", "serve"]
```

**HTTP/SSE** — Animus connects to a running remote MCP server over HTTP. Use
`transport: http` and supply the server `url`. Animus does not manage the server
process; it must already be reachable at the given URL. Optional `headers` let
you pass authentication tokens using `${VAR}` interpolation:

```yaml
mcp_servers:
  remote-api:
    transport: http
    url: "https://mcp.example.com/sse"
    headers:
      Authorization: "Bearer ${REMOTE_API_TOKEN}"
```

Both transport types support the `tools` allowlist field to restrict which tool
names are exposed to agents.

See [Workflow YAML Schema](../reference/workflow-yaml.md) for the full field
reference.

## Pack-Owned MCP Descriptors

Packs can also ship MCP descriptors under pack assets. Animus loads those
descriptors, namespaces the resulting server ids by pack id, and makes them
available to workflows and phases.

Examples:

- `ao.requirement/github-sync`
- `vendor.crm/runtime`

Pack-owned MCP behavior stays outside the daemon. The daemon only supervises the
processes and records execution facts.

## Workflow-Level Usage

Project YAML can reference MCP servers directly, and pack overlays can inject
phase bindings and default server sets.

Key rules:

- project YAML defines repo-specific MCP servers
- pack overlays can contribute namespaced MCP servers
- agents and phases only see explicitly allowed tools
- Animus state mutations should go through MCP or CLI mutation surfaces, not direct
  file edits

## Why This Boundary Exists

Tool-driven mutation keeps Animus auditable and composable:

- state changes flow through validated surfaces
- external integrations remain process-based
- packs can add behavior without changing daemon-core

See [Workflows](./workflows.md) and [How Animus Works](./how-ao-works.md) for how
MCP fits into execution.
