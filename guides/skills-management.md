# Skills Management Guide

Skills are reusable capability modules that enhance agent behavior in AO workflows. They inject prompt fragments, model/tool policy overrides, MCP attachments, timeout overrides, and capability flags into agent execution contexts.

This guide covers how to discover, install, list, and use skills in your AO workflows and agent personas.

---

## What Are Skills?

Skills are modular, reusable packages that define specialized agent capabilities. Instead of duplicating system prompts and configuration across multiple agent definitions, you can define a skill once and attach it to any agent or phase.

Skills can provide:

| Capability | Description |
|------------|-------------|
| **Prompt fragments** | Additional context or instructions injected into the agent's system prompt |
| **Model/tool overrides** | Policy changes for which models or tools the agent can use |
| **MCP attachments** | Automatic attachment of MCP servers when the skill is active |
| **Timeout overrides** | Custom timeout settings for long-running operations |
| **Launch args/env** | Additional CLI arguments or environment variables |
| **Capability flags** | Boolean flags that enable/disable specific behaviors |

---

## Skill Discovery

### Searching the Skill Catalog

Use `ao skill search` to discover available skills from configured registries:

```bash
# Search for skills by keyword
ao skill search "code-review"

# Search for skills by category
ao skill search "security"

# List all available skills
ao skill search ""
```

### Skill Registries

Skills are published to registries. Manage your registries with:

```bash
# List configured registries
ao pack registry list

# Add a new registry
ao pack registry add https://github.com/example/ao-skills-registry.git

# Sync a registry for latest catalog
ao pack registry sync <registry-name>
```

The official AO skills are available at [github.com/launchapp-dev/ao-skills](https://github.com/launchapp-dev/ao-skills).

---

## Installing Skills

### Install from Registry

Install a skill by name from a configured registry:

```bash
# Install a specific skill
ao skill install code-review

# Install multiple skills
ao skill install code-review security-review test-strategy
```

### Install from Local Path

Install a skill from a local directory:

```bash
# Install from local path
ao skill install ./path/to/my-skill
```

### Skill Installation Location

Installed skills are stored in AO's state directory and become available for reference in workflow YAML files.

---

## Listing Installed Skills

View all installed skills:

```bash
# List all installed skills
ao skill list
```

The output shows:
- Skill name and version
- Description
- Capabilities provided
- Source (registry or local)

---

## Updating Skills

Update skills to their latest versions:

```bash
# Update all installed skills
ao skill update

# Update a specific skill
ao skill update code-review
```

---

## Publishing Skills

If you've created a custom skill, publish it to a registry:

```bash
# Publish a skill version
ao skill publish ./my-skill --version 1.0.0
```

### Skill Package Structure

A skill package typically includes:

```
my-skill/
├── skill.yaml          # Skill definition
├── prompts/
│   └── system.md       # Prompt fragments
└── README.md           # Documentation
```

The `skill.yaml` defines the skill's capabilities:

```yaml
name: code-review
version: 1.0.0
description: "Code review capabilities for quality and correctness"
category: review

provides:
  prompts:
    - path: prompts/system.md
      inject: append      # append, prepend, or replace
  
  capabilities:
    can_review: true
    can_update_checklists: true
  
  mcp_servers:
    - ao
  
  timeout_override: 1800  # 30 minutes for large reviews
```

---

## Using Skills in Agent Personas

Reference skills in your agent definitions within `.ao/workflows.yaml`:

```yaml
agents:
  code-reviewer:
    description: "Senior code reviewer focused on quality and correctness"
    system_prompt: |
      You are a senior code reviewer with 15 years of experience.
      Focus on correctness, maintainability, and best practices.
    model: claude-sonnet-4-6
    tool: claude
    mcp_servers:
      - ao
    skills:
      - code-review
      - git-operations
```

When this agent runs, the `code-review` and `git-operations` skills are resolved and their capabilities are injected into the execution context.

### Multiple Skills

Attach multiple skills to combine capabilities:

```yaml
agents:
  security-reviewer:
    description: "Security engineer for vulnerability assessment"
    system_prompt: |
      You are a senior security engineer...
    model: claude-sonnet-4-6
    tool: claude
    mcp_servers:
      - ao
    skills:
      - security-review
      - threat-modeling
      - owasp-top-10
      - vulnerability-assessment
```

---

## Using Skills in Phases

Skills can also be attached directly to phases for phase-specific capabilities:

```yaml
phases:
  implementation:
    mode: agent
    agent: default
    directive: "Implement the change."
    skills:
      - implementation
      - code-review    # Pre-emptive review awareness
  
  security-scan:
    mode: agent
    agent: default
    directive: "Perform security analysis"
    skills:
      - security-review
      - sast-scanning
```

Phase-level skills are validated during configuration load. At runtime, they inject their defined capabilities into the phase execution context.

---

## Using Skills in Workflows

Reference agents with skills in your workflow definitions:

```yaml
agents:
  reviewer:
    system_prompt: |
      You are a senior code reviewer...
    model: claude-sonnet-4-6
    tool: claude
    skills:
      - code-review
      - git-operations

  implementer:
    model: claude-sonnet-4-6
    tool: claude
    skills:
      - implementation
      - testing

workflows:
  - id: reviewed-implementation
    name: "Reviewed Implementation"
    description: "Implement with code review gate"
    phases:
      - implementation:
          agent: implementer
      - code-review:
          agent: reviewer
          on_verdict:
            rework:
              target: implementation
      - testing
```

---

## Skill Resolution Order

When skills are attached to both agents and phases, they are merged in this order:

1. **Base agent skills** -- Skills defined on the agent profile
2. **Phase skills** -- Skills defined on the phase (can override or extend)
3. **Runtime skills** -- Skills injected at dispatch time

Later sources can override earlier ones for conflicting settings.

---

## Built-in vs. Custom Skills

### Built-in Skills

AO may bundle certain skills that are always available:

| Skill | Description |
|-------|-------------|
| `implementation` | Core implementation capabilities |
| `code-review` | Code review and quality analysis |
| `testing` | Test strategy and execution |
| `requirements-analysis` | Requirements refinement |

### Custom Skills

Create custom skills for project-specific capabilities:

1. **Create the skill directory**:

   ```bash
   mkdir -p .ao/skills/my-custom-skill/prompts
   ```

2. **Define the skill** (`.ao/skills/my-custom-skill/skill.yaml`):

   ```yaml
   name: my-custom-skill
   version: 1.0.0
   description: "Project-specific capabilities"
   category: custom
   
   provides:
     prompts:
       - path: prompts/system.md
         inject: append
     capabilities:
       custom_flag: true
   ```

3. **Add prompt fragments** (`.ao/skills/my-custom-skill/prompts/system.md`):

   ```markdown
   When working on this project:
   - Follow the project's coding standards in docs/coding-standards.md
   - Use the custom test framework defined in tests/
   - All PRs must update the CHANGELOG.md
   ```

4. **Reference in workflows**:

   ```yaml
   agents:
     default:
       model: claude-sonnet-4-6
       tool: claude
       skills:
         - my-custom-skill
   ```

---

## CLI Reference

| Command | Description |
|---------|-------------|
| `ao skill search <query>` | Search skill catalog |
| `ao skill install <name>` | Install skill from registry or path |
| `ao skill list` | List installed skills |
| `ao skill update [name]` | Update skills to latest versions |
| `ao skill publish <path>` | Publish skill to registry |

---

## Best Practices

1. **Prefer skills over duplicated prompts** -- If you find yourself copying the same instructions across multiple agents, extract them into a skill.

2. **Keep skills focused** -- Each skill should provide a cohesive set of related capabilities. Avoid "kitchen sink" skills.

3. **Version your skills** -- Use semantic versioning for custom skills, especially when sharing across teams.

4. **Document skill behavior** -- Include a README in your skill packages explaining what the skill provides and how to use it.

5. **Test skill combinations** -- When using multiple skills together, verify they don't have conflicting configurations.

6. **Use phase skills sparingly** -- Most capabilities should be attached to agents. Only use phase-level skills for phase-specific overrides.

7. **Validate after installation** -- Run `ao workflow config validate` after installing new skills to ensure they integrate correctly.

---

## See Also

- **[Agent Persona Cookbook](agent-personas.md)** -- Ready-to-use agent templates with skill references
- **[Writing Custom Workflows](writing-workflows.md)** -- Complete workflow YAML reference
- **[Workflow YAML Schema](../reference/workflow-yaml.md)** -- Full schema documentation including skills fields
- **[MCP Tools Reference](../reference/mcp-tools.md)** -- All available AO tools
- **[CLI Command Reference](../reference/cli/index.md)** -- Complete CLI surface including skill commands
