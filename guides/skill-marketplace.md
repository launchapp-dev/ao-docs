# Skill Marketplace

The Animus skill marketplace is a decentralized catalog of reusable agent capability modules. Skills are distributed through **registries** — Git repositories that index skill packages — and installed locally via the `animus skill` CLI.

This guide covers: browsing the marketplace, installing skills, managing registries, authoring skill packages, and publishing to a registry.

---

## How the Marketplace Works

Skills are published to **pack registries**: Git repositories that serve as catalogs. Each registry contains a manifest listing available skills with their names, versions, descriptions, and download URLs.

```
Registry (Git repo)
└── catalog.yaml          # Index of all published skills
    ├── code-review 1.2.0
    ├── security-review 0.9.1
    └── test-strategy 2.0.0
```

When you run `animus skill install <name>`, Animus:

1. Queries all configured registries for a skill matching that name
2. Resolves the latest compatible version (or a pinned one)
3. Downloads and unpacks the skill package
4. Stores it in `~/.ao/skills/<name>/<version>/`

---

## Browsing Available Skills

Search the catalog across all configured registries:

```bash
# Search by keyword
animus skill search "code-review"

# Search by category
animus skill search "security"

# List everything in the catalog
animus skill search ""
```

Example output:

```
SKILL                  VERSION  REGISTRY          DESCRIPTION
code-review            1.2.0    ao-official       Code review and quality analysis
code-review-strict     1.0.0    ao-official       Strict style-enforcing code review
security-review        0.9.1    ao-official       Security vulnerability assessment
test-strategy          2.0.0    ao-official       Test planning and execution
requirements-analysis  1.1.0    ao-official       Requirements decomposition
```

---

## Installing Skills

### Install from a Registry

Install a skill by name. Animus resolves it from your configured registries:

```bash
animus skill install code-review
```

Install multiple skills in one command:

```bash
animus skill install code-review security-review test-strategy
```

### Install a Pinned Version

Append `@<version>` to install a specific version:

```bash
animus skill install code-review@1.1.0
```

Pinning versions ensures reproducible agent behavior across team members and CI environments.

### Install from a Local Path

During development, install a skill directly from a local directory:

```bash
animus skill install ./path/to/my-skill
```

This copies the skill into `~/.ao/skills/` so it is available project-wide.

### Verify Installation

List all installed skills to confirm:

```bash
animus skill list
```

```
SKILL            VERSION  SOURCE          DESCRIPTION
code-review      1.2.0    ao-official     Code review and quality analysis
security-review  0.9.1    ao-official     Security vulnerability assessment
my-custom-skill  0.1.0    local           Project-specific capabilities
```

### Update Installed Skills

Update a single skill to its latest registry version:

```bash
animus skill update code-review
```

Update all installed skills at once:

```bash
animus skill update
```

---

## Managing Registries

Registries are Git repositories that host skill catalogs. You can configure multiple registries — Animus searches all of them when resolving installs.

### List Configured Registries

```bash
animus pack registry list
```

```
NAME          URL                                                   LAST SYNCED
ao-official   https://github.com/launchapp-dev/ao-skills.git       2 hours ago
team-skills   https://github.com/myorg/ao-skills-internal.git      1 day ago
```

### Add a Registry

```bash
animus pack registry add https://github.com/myorg/ao-skills-internal.git
```

Give the registry a friendly name with `--name`:

```bash
animus pack registry add https://github.com/myorg/ao-skills-internal.git --name team-skills
```

Animus clones the registry locally on add and indexes its catalog.

### Sync a Registry

Pull the latest catalog from a registry without reinstalling skills:

```bash
animus pack registry sync team-skills
```

Sync all registries at once:

```bash
animus pack registry sync
```

Run this periodically to pick up newly published skills.

### Remove a Registry

```bash
animus pack registry remove team-skills
```

This removes the registry from your configuration. Already-installed skills from that registry are not affected.

---

## Authoring a Skill Package

A skill package is a directory with a `skill.yaml` manifest and optional supporting files.

### Directory Layout

```
my-skill/
├── skill.yaml          # Required: skill manifest
├── prompts/
│   ├── system.md       # System prompt fragment
│   └── context.md      # Optional additional context
└── README.md           # Marketplace description (shown in search results)
```

### Skill Manifest (`skill.yaml`)

The manifest declares the skill's identity and what capabilities it injects:

```yaml
name: my-skill
version: 1.0.0
description: "Short description shown in catalog search results"
category: review           # Used for categorized search

# Semantic version constraint for animus daemon compatibility
ao_version: ">=0.8.0"

provides:
  # Prompt fragments injected into the agent's system prompt
  prompts:
    - path: prompts/system.md
      inject: append         # append | prepend | replace

  # Boolean capability flags available in workflow conditionals
  capabilities:
    can_review: true
    can_update_checklists: true

  # MCP servers automatically attached when this skill is active
  mcp_servers:
    - ao

  # Override the default agent timeout (seconds)
  timeout_override: 1800

  # Additional environment variables injected at runtime
  env:
    REVIEW_STRICTNESS: high
```

#### `provides` Fields

| Field | Type | Description |
|-------|------|-------------|
| `prompts` | list | Prompt fragments to inject. Each entry has `path` and `inject` (append/prepend/replace). |
| `capabilities` | map | Boolean flags set on the execution context. |
| `mcp_servers` | list | MCP server names attached for the duration of the skill. |
| `timeout_override` | integer | Agent timeout in seconds. Overrides the agent default. |
| `env` | map | Environment variables merged into the agent's runtime env. |

### Writing Prompt Fragments

Keep prompt fragments focused. They are merged with the agent's own system prompt, so avoid repeating instructions the agent already has.

```markdown
<!-- prompts/system.md -->

When performing code review:
- Check for off-by-one errors in loop bounds and slice indices.
- Flag any use of `eval()` or dynamic code execution.
- Verify error return values are handled, not silenced.
- Ensure public functions have doc comments.
```

---

## Publishing a Skill

### 1. Prepare and Validate

Before publishing, validate the manifest structure:

```bash
animus skill publish ./my-skill --dry-run
```

This checks that `skill.yaml` is valid, all referenced prompt files exist, and the version does not already exist in the target registry.

### 2. Publish to a Registry

```bash
animus skill publish ./my-skill --version 1.0.0
```

By default, Animus publishes to the first writable registry in your list. Target a specific registry:

```bash
animus skill publish ./my-skill --version 1.0.0 --registry team-skills
```

Publishing:
1. Packages the skill directory into a versioned archive
2. Pushes the archive to the registry
3. Updates the registry's `catalog.yaml` with the new entry

### 3. Verify Publication

After publishing, sync the registry and confirm the skill appears:

```bash
animus pack registry sync team-skills
animus skill search "my-skill"
```

### Versioning Guidelines

Follow [Semantic Versioning](https://semver.org/):

| Change type | Version bump |
|-------------|-------------|
| Bug fix or prompt tweak | Patch (`1.0.0` → `1.0.1`) |
| New capability or field added | Minor (`1.0.0` → `1.1.0`) |
| Breaking change (removed capability, renamed field) | Major (`1.0.0` → `2.0.0`) |

---

## Hosting a Private Registry

To distribute skills within a team, host a Git repository that follows the registry layout:

```
my-skills-registry/
├── catalog.yaml          # Index file — updated by animus skill publish
└── packages/
    ├── my-skill-1.0.0.tar.gz
    └── another-skill-2.1.0.tar.gz
```

Animus manages `catalog.yaml` automatically when you publish. The only requirement is that team members have read access to the repository.

```bash
# Each team member adds the private registry once
animus pack registry add git@github.com:myorg/ao-skills-registry.git --name team-skills

# Then installs skills normally
animus skill install my-skill
```

---

## CLI Reference

### `animus skill` Commands

| Command | Description |
|---------|-------------|
| `animus skill search <query>` | Search skill catalog across all registries |
| `animus skill install <name>[@version]` | Install skill from registry or local path |
| `animus skill install <path>` | Install skill from a local directory |
| `animus skill list` | List installed skills |
| `animus skill update [name]` | Update one or all skills to latest registry version |
| `animus skill publish <path>` | Publish skill to a registry |
| `animus skill publish <path> --dry-run` | Validate without publishing |

### `animus pack registry` Commands

| Command | Description |
|---------|-------------|
| `animus pack registry list` | List all configured registries and sync status |
| `animus pack registry add <url>` | Add a registry (clones and indexes on add) |
| `animus pack registry add <url> --name <name>` | Add with a friendly name |
| `animus pack registry remove <name>` | Remove a registry |
| `animus pack registry sync [name]` | Pull latest catalog; omit name to sync all |

---

## Troubleshooting

**Skill not found after `animus skill search`**

The registry catalog may be stale. Sync it:

```bash
animus pack registry sync
```

If the skill still does not appear, verify the registry URL with `animus pack registry list` and confirm the skill is published to that registry.

---

**`animus skill install` fails with "version already installed"**

The skill is already at the requested version. Use `animus skill update` to refresh it, or uninstall and reinstall if you need to reset local state.

---

**`animus skill publish` fails with "registry not writable"**

Your configured registries may be read-only (e.g., the official Animus registry). Target your own registry explicitly:

```bash
animus skill publish ./my-skill --version 1.0.0 --registry team-skills
```

---

**Skill installed but agent does not pick it up**

Confirm the skill name in `skill.yaml` matches what you reference in your workflow YAML:

```yaml
agents:
  default:
    skills:
      - my-skill   # must match `name:` in skill.yaml exactly
```

Run `animus workflow config validate` to catch resolution errors before dispatch.

---

## See Also

- **[Skills Management](skills-management.md)** — Using skills in agent personas, phases, and workflows
- **[Pack Management](pack-management.md)** — Installing and configuring Animus plugin packs
- **[Agent Persona Cookbook](agent-personas.md)** — Ready-to-use agent templates with skill references
- **[Workflow YAML Schema](../reference/workflow-yaml.md)** — Full schema including `skills` fields
- **[CLI Command Reference](../reference/cli/index.md)** — Complete CLI surface
