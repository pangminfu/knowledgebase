---
type: entity
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Claude Code

**Claude Code** is Anthropic's agentic coding tool: it reads a codebase, edits files, and runs commands to carry out a task, under rules the team and organization configure. It runs in the terminal, in IDE extensions (VS Code, JetBrains), in a desktop app, and on the web, and non-interactively in CI. (general knowledge; verified: [Claude Code overview](https://code.claude.com/docs/en/overview)) The AI-native SDLC playbook is built around it. (src: [[2026-11-24-ai-native-sdlc-playbook]])

## Details

### Features the playbook uses

| Feature | What it is | Where it lives |
| --- | --- | --- |
| Plan mode | Claude reads code (and may run exploratory shell commands) and writes a plan; it doesn't edit source files until the engineer approves the plan | Permission mode `plan` |
| "Auto mode" / auto-accept | After the plan is approved, Claude applies changes without a per-edit prompt; the playbook's default for routine, well-tested, small-blast-radius work once guardrails mature | Permission mode (see below for which one) |
| `CLAUDE.md` | Project instructions loaded at the start of every session; `/init` generates a first draft | `./CLAUDE.md` or `./.claude/CLAUDE.md` in the repo; personal `~/.claude/CLAUDE.md` |
| Skills | Policy or procedure as a folder with a `SKILL.md`; the description tells Claude when to load it | `.claude/skills/<name>/` (project), `~/.claude/skills/` (personal), or org-wide via plugins or the managed settings directory |
| Hooks | Scripts that run at points in the agent loop and can allow, ask, or block (e.g. a `PreToolUse` hook, which fires before a tool call, matched on `Bash`) | `.claude/settings.json` (team, in git), `~/.claude/settings.json`, managed settings, plugins, or skill/subagent frontmatter |
| Subagents | Scoped helpers inside one session, each with its own context window, system prompt, and tool limits (e.g. a verifier, a code simplifier, a researcher) | `.claude/agents/*.md` (project) or `~/.claude/agents/` (personal) |
| Worktrees | Parallel sessions, each in its own git checkout and branch: `claude --worktree <name>` creates `.claude/worktrees/<name>/` on branch `worktree-<name>` | Git |
| Non-interactive mode | `claude -p "<prompt>"` for CI jobs and evals | CI runners |
| Code Review / claude-code-action | Code Review: Anthropic-hosted PR review for Team and Enterprise plans, turned on by an organization Owner; findings don't approve or block a PR by themselves. claude-code-action: GitHub Action that runs Claude Code in your own workflows and answers `@claude` mentions | GitHub |
| MCP connectors | Tools and data from other systems (Jira, deploy tooling, browsers) | Settings, `.mcp.json`, plugins, or managed MCP config |
| Managed settings | Org-wide settings engineers cannot override | See [[#Settings files and precedence]] |
| OpenTelemetry export | Metrics, events (including every tool permission decision and its source, e.g. a hook), and optional traces sent to the org's observability stack (OpenTelemetry is the standard telemetry format) | Settings / environment variables |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) Locations, flags, and plan/availability details checked against the docs. (general knowledge; verified: [Permission modes](https://code.claude.com/docs/en/permission-modes), [Memory](https://code.claude.com/docs/en/memory), [Skills](https://code.claude.com/docs/en/skills), [Hooks](https://code.claude.com/docs/en/hooks), [Subagents](https://code.claude.com/docs/en/sub-agents), [Worktrees](https://code.claude.com/docs/en/worktrees), [Code Review](https://code.claude.com/docs/en/code-review), [GitHub Actions](https://code.claude.com/docs/en/github-actions), [Monitoring](https://code.claude.com/docs/en/monitoring-usage))

- **MCP (Model Context Protocol)** is an open-source standard for connecting AI tools to external tools, data sources, and APIs, so one integration works across agents. (general knowledge; verified: [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp))
- Model access can go through Anthropic's API or through Amazon Bedrock, Google Vertex, or Microsoft Foundry when traffic must stay on the organization's cloud agreement. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Google now calls Vertex AI "Google Cloud's Agent Platform". (general knowledge; verified: [Enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations))
- Parallel sessions: start with two or three; the ceiling is how many streams one person can review properly. Tasks that share files run in one session, one after another. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- How these features fit together as controls: [[agent-guardrails]]; as gates: [[deployment-gates]].

### Permission modes

A permission mode sets what Claude may do without asking first. Switch with `Shift+Tab` in the CLI or start with `--permission-mode <mode>`. (general knowledge; verified: [Choose a permission mode](https://code.claude.com/docs/en/permission-modes))

| Mode | Runs without asking |
| --- | --- |
| `default` (shown as "Manual") | File reads in the working directories; prompts before most edits, shell commands, and network requests |
| `acceptEdits` | Reads, file edits, and common filesystem commands (`mkdir`, `touch`, `mv`, `cp`, …) inside the working directory |
| `plan` | Reads and exploratory commands; no source edits until the plan is approved |
| `auto` | Everything without routine prompts; a separate classifier model reviews actions before they run and blocks risky ones |
| `dontAsk` | Only pre-approved tools; anything that would prompt is denied (for locked-down CI) |
| `bypassPermissions` | Everything; meant only for isolated containers/VMs. Deny rules still apply |

- Starting mode: `auto` is the built-in starting mode for interactive terminal and VS Code sessions on Claude Code v2.1.283 or later; `claude -p` and Agent SDK runs start in `default`, where calls needing approval are denied unless allowed. (general knowledge; verified: [Choose a permission mode](https://code.claude.com/docs/en/permission-modes))
- The playbook's "auto mode" / "auto-accept" maps onto two current modes: `acceptEdits` auto-approves file edits and common filesystem commands only, while `auto` approves everything subject to the classifier. When approving a plan, the option reads "Yes, and use auto mode", or "Yes, auto-accept edits" when auto mode is unavailable. (general knowledge; verified: [Choose a permission mode](https://code.claude.com/docs/en/permission-modes))

### Settings files and precedence

When the same key is set in several places, the highest level wins (list keys such as `permissions.allow` merge instead): (general knowledge; verified: [Settings](https://code.claude.com/docs/en/settings))

1. **Managed settings**: deployed by the organization; nothing below can override them.
2. **Command line**: `--settings <file-or-json>` for one session.
3. **Project local**: `.claude/settings.local.json` (personal, not committed).
4. **Shared project**: `.claude/settings.json` (committed to git; where team hooks go per the playbook).
5. **User**: `~/.claude/settings.json`.

Managed settings can be delivered three ways (general knowledge; verified: [Managed settings](https://code.claude.com/docs/en/managed-settings)):

| Mechanism | Where the policy lives |
| --- | --- |
| Server-managed | The claude.ai admin console; fetched at startup and polled hourly |
| MDM (mobile device management: IT's tool for pushing config to company machines) / OS policy | macOS configuration profile (`com.anthropic.claudecode` domain); Windows registry `HKLM\SOFTWARE\Policies\ClaudeCode` |
| File-based | `managed-settings.json` in `/Library/Application Support/ClaudeCode/` (macOS), `/etc/claude-code/` (Linux, WSL), `C:\Program Files\ClaudeCode\` (Windows) |

The playbook names "MDM or the admin console" for its regulated-enterprise example. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Related Anthropic products mentioned

| Product | Role in the playbook | Status check |
| --- | --- | --- |
| Claude Design | Product owner mocks a UI from `intent.md`, iterates, then hands it off to Claude Code to build | Hand-off to Claude Code confirmed; playbook (Aug 2026) says beta, the product page (Nov 2026) lists it with every plan and no beta label ([Claude Design](https://claude.com/product/design)) |
| Cowork / claude.ai | Lets non-engineers write `intent.md` and commit it through a version-control connector | Cowork exists as a Claude product ([Cowork](https://claude.com/product/cowork)) |
| Claude Security | Hosted security scans of connected GitHub repos on Claude Mythos 5; each finding validated and given a confidence rating; patches reviewed in Claude Code on the web; billed on consumption | Public beta for Claude Enterprise; scans with Claude Mythos, validates findings, suggests patches ([Claude Security](https://claude.com/product/claude-security)). Consumption billing not checked online |
| Claude Tag | Claude as a member of Slack channels under its own identity; first responder on incidents | Public beta, in Slack, on Team and Enterprise plans ([Claude Tag docs](https://claude.com/docs/claude-tag/overview)) |
| Agent SDK | Build a service around the agent, e.g. one that receives monitoring webhooks | The same agent loop and tools as Claude Code, as a CLI and Python/TypeScript packages ([Run Claude Code programmatically](https://code.claude.com/docs/en/headless)) |

(src: [[2026-11-24-ai-native-sdlc-playbook]]; status column: general knowledge, verified at the linked pages)

### Bridge to this wiki

This wiki's `AGENTS.md` plays the role `CLAUDE.md` plays in a code repo: the always-read file of conventions and workflows for the agent that maintains the vault. (general knowledge) Claude Code itself loads `AGENTS.md` when a directory has no `CLAUDE.md` or `CLAUDE.local.md`. (general knowledge; verified: [Memory: AGENTS.md](https://code.claude.com/docs/en/memory))

## Related

- [[agent-guardrails]]: `CLAUDE.md`, skills, hooks, permissions, sandbox, managed settings.
- [[deployment-gates]]: Claude in CI/CD, up to the production gate.
- [[evals]]: testing Claude Code's configuration with `claude -p` in CI.
- [[artifact-chain]]: plan mode produces `plan.md`.
- [[software-development-lifecycle]]: the process it is used across.

## Contradictions

- **Does Claude Security need a scan schedule?** The playbook says to "set a schedule per project", with weekly as a sensible default. (src: [[2026-11-24-ai-native-sdlc-playbook]]) The product page (Nov 2026) answers "Do I need to schedule scans? No": after a baseline scan, Claude keeps scanning changes regularly based on their risk and complexity. (general knowledge; verified: [Claude Security](https://claude.com/product/claude-security)) The product may have changed since the playbook was published (Aug 2026).
