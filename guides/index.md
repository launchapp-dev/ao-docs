# Guides

Practical walkthroughs for day-to-day AO operations.

## Getting Started

- **[First Autonomous PR in 5 Minutes](first-autonomous-pr.md)** -- The fastest path from installation to your first autonomous pull request.

## Planning and Requirements

- **[Requirements Workflow](requirements-workflow.md)** -- From vision drafting through requirement decomposition into actionable tasks.

## Task and Execution

- **[ao now — Work Inbox](ao-now.md)** -- Your command-line work inbox: next task, active workflows, blocked and stale items at a glance.
- **[Task Management](task-management.md)** -- Creating, prioritizing, assigning, and tracking tasks through their full lifecycle.
- **[Writing Custom Workflows](writing-workflows.md)** -- YAML workflow authoring: agents, phases, pipelines, MCP servers, and post-success hooks.
- **[Event Triggers: File Watchers and Webhooks](event-triggers.md)** -- Dispatch workflows automatically when files change or when an HTTP webhook fires.
- **[Self-Hosting Workflow](self-hosting.md)** -- How AO tracks its own development through `ao` commands.

## Operations

- **[Daemon Operations](daemon-operations.md)** -- Starting, stopping, pausing, configuring, and monitoring the autonomous daemon.
- **[Pack Management](pack-management.md)** -- Installing, listing, configuring, and troubleshooting AO plugin packs.
- **[Fleet Management](fleet-management.md)** -- Getting started with `ao fleet`: registering nodes, sizing pools, running parallel agents, scheduling, and observability across a distributed fleet.
- **[Cloud Deployment](cloud-deployment.md)** -- Logging in, pushing project configuration, and managing the cloud-hosted daemon with `ao cloud`.
- **[Cloud Dashboard](cloud-dashboard.md)** -- Navigating the React web app at app.ao.dev: projects, agent monitoring, dark mode, webhook delivery, template gallery, and team access.
- **[Workflow DAG Visualization](workflow-dag.md)** -- Interactive phase graph in the dashboard: reading node and edge types, live execution overlay, critical-path highlighting, and export options.
- **[Cloud Daemon Management](cloud-daemon-management.md)** -- Starting, stopping, restarting, and sizing cloud daemon instances from the dashboard: auto-restart policies, drain mode, queue visibility, and the daemon health timeline.
- **[Cloud Billing](cloud-billing.md)** -- Subscription plans, usage metering, per-project cost breakdown, Stripe payment methods, and invoices.
- **[AO Cloud Beta Signup](cloud-beta-signup.md)** -- Joining the beta, onboarding wizard walkthrough, included features, and limitations.
- **[Demo Mode](demo-mode.md)** -- Exploring AO Cloud with a pre-populated sandbox environment — no account or API keys required — and running a local demo with `ao demo start`.
- **[Service Status Page](status-page.md)** -- Checking AO Cloud service health at status.ao.dev: component status, incident history, uptime metrics, email/RSS subscriptions, and the status embed widget.
- **[Model Routing](model-routing.md)** -- How AO selects models and tools per phase, and how to override defaults.
- **[Web Dashboard](web-dashboard.md)** -- Launching the local web UI, navigating boards, and using the REST API.

## Integrations

- **[GitHub App Registration and Setup](github-app-setup.md)** -- Step-by-step guide to installing the Animus GitHub App, configuring org/repo permissions, completing the Animus Cloud connection, and writing your first trigger.
- **[GitHub App Integration](github-app.md)** -- Full reference: repository selector filters, delivery log, payload interpolation, managed webhook receiver, and posting check runs back to GitHub.
- **[GitHub Checks](github-checks.md)** -- Creating and updating GitHub Check Runs from AO workflows: automatic check runs, manual tool control, annotations, re-run support, and the dashboard check runs panel.

## MCP & Agent Integration

- **[Working with AO via MCP Tools](agents.md)** -- Complete guide to all 73 MCP tools: JSON examples, common workflows, sequencing tips, pagination, and batch operations.
- **[Agent Persona Cookbook](agent-personas.md)** -- Ready-to-use agent persona templates: code reviewer, requirements analyst, architect, QA engineer, security reviewer, documentation writer.
- **[Skills Management](skills-management.md)** -- How to discover, install, and use agent skills. Referencing skills in personas and workflows.
- **[Skill Marketplace](skill-marketplace.md)** -- Browsing the skill catalog, managing registries, authoring skill packages, and publishing to a registry.

## Privacy & Security

- **[Privacy & Data Policy](privacy.md)** -- AO's data guarantees, AI provider data policies, local-model operation, and how AO differs from cloud AI assistants that collect code for training.
- **[Security Headers](security-headers.md)** -- Content-Security-Policy, HSTS, X-Frame-Options, and other HTTP security headers applied to the AO cloud dashboard in production.
- **[Custom Error Pages](custom-error-pages.md)** -- Branded 404 and 500 error pages: what they show, how they behave in a single-page application, and self-hosted web server configuration.
- **[SEO and Site Metadata](seo-and-site-metadata.md)** -- HTML meta tags, Open Graph, Twitter Card, canonical URLs, favicon set, and manifest.json configuration.

## Infrastructure

- **[CI/CD](ci-cd.md)** -- CI workflows, release pipelines, build commands, and test targets.
- **[Troubleshooting](troubleshooting.md)** -- Common issues, diagnostics, and fixes.

## Tutorials & Quick References

For hands-on tutorials and quick-reference guides, see **[Tutorials](../tutorials/)**:

- **[CLI Cheat Sheet](../tutorials/cli-cheat-sheet.md)** -- One-page quick reference for all essential commands
- **[Common Command Sequences](../tutorials/common-sequences.md)** -- Everyday command patterns and one-liners
- **[Task Lifecycle Walkthrough](../tutorials/task-lifecycle.md)** -- Follow a task from creation to completion
- **[Workflow Execution Reference](../tutorials/workflow-execution.md)** -- Running and monitoring workflows
- **[Daemon Operations Quick Reference](../tutorials/daemon-quick-ref.md)** -- Essential daemon commands
