---
type: entity
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Claude Code

**Claude Code** is Anthropic's agentic coding tool. It runs in a terminal (and in CI or on the web), reads a codebase, edits files, and runs commands to carry out a task, under rules the team and organization configure. The AI-native SDLC playbook is built around it. (src: [[2026-11-24-ai-native-sdlc-playbook]])

## Details

### Features the playbook uses

| Feature | What it is | Where it lives |
| --- | --- | --- |
| Plan mode | Read-only mode: Claude studies the code and writes a plan, and cannot edit files until the engineer accepts it | Session permission mode |
| Auto mode | Claude works through the approved plan without a per-edit prompt; the default for routine, well-tested, small-blast-radius work once guardrails mature | Session permission mode |
| `CLAUDE.md` | Project instructions read at the start of every session; `/init` generates a first draft | Repo root |
| Skills | Policy or procedure as a folder with a `SKILL.md`, loaded when relevant | `.claude/skills/<name>/`, or org-wide via plugins |
| Hooks | Scripts that run around agent actions and allow, ask, or block (e.g. a `PreToolUse` hook, which fires before a tool call, on `Bash`) | `.claude/settings.json`, or managed settings |
| Subagents | Scoped helpers inside one session with their own context window and tool limits (e.g. a verifier, a code simplifier, a researcher) | `.claude/agents/*.md` |
| Worktrees | Parallel sessions, each in its own git checkout and branch (`claude --worktree <name>`) | Git |
| Non-interactive mode | `claude -p "<prompt>"` for CI jobs and evals | CI runners |
| claude-code-action / Code Review | Runs Claude in GitHub CI; managed PR review service enabled by an admin | GitHub |
| MCP connectors | Tools and data from other systems (Jira, deploy tooling, browsers) | Settings / plugins |
| Managed settings | Org-wide settings engineers cannot override | MDM (mobile device management: IT's tool for pushing config to company machines) or admin console |
| OpenTelemetry export | Session traces and hook decisions sent to the org's observability stack (OpenTelemetry is the standard telemetry format) | Settings |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

- The playbook's "auto mode" wording mixes two current modes (general knowledge, per Claude Code's permission-modes docs, Nov 2026): `acceptEdits` auto-approves file edits and common filesystem commands only, while `auto` runs everything and has a second classifier model review each action instead of the human. `auto` is the built-in starting mode for interactive terminal and VS Code sessions on recent versions.

- **MCP (Model Context Protocol)** is an open protocol for connecting AI agents to external tools and data sources, so one integration works across agents. (general knowledge)
- Model access can go through Anthropic's API or through AWS Bedrock, Google Vertex, or Microsoft Foundry when traffic must stay on the organization's cloud agreement. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Parallel sessions: start with two or three; the ceiling is how many streams one person can review properly. Tasks that share files run in one session, one after another. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- How these features fit together as controls: [[agent-guardrails]]; as gates: [[deployment-gates]].

### Related Anthropic products mentioned

| Product | Role in the playbook |
| --- | --- |
| Claude Design (beta) | Product owner mocks a UI from `intent.md`, then exports it to Claude Code |
| Cowork / claude.ai | Lets non-engineers write `intent.md` and commit it through a connector |
| Claude Security (public beta) | Scheduled, hosted security scans of connected GitHub repos; findings validated and given a confidence rating; billed on consumption |
| Claude Tag (public beta) | Claude as a Slack channel member under its own identity; first responder on incidents |
| Agent SDK | Build a service around the agent, e.g. one that receives monitoring webhooks |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

### Bridge to this wiki

General knowledge: this wiki's `AGENTS.md` plays the role `CLAUDE.md` plays in a code repo. It is the always-read file of conventions and workflows for the agent that maintains the vault.

## Related

- [[agent-guardrails]]: `CLAUDE.md`, skills, hooks, permissions, sandbox.
- [[deployment-gates]]: Claude in CI/CD, up to the production gate.
- [[evals]]: testing Claude Code's configuration.
- [[artifact-chain]]: plan mode produces `plan.md`.
- [[software-development-lifecycle]]: the process it is used across.
