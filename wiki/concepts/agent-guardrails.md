---
type: concept
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Agent guardrails: advisory vs enforced controls

**Agent guardrails** are the controls that keep a coding agent inside an organization's rules. They come in two layers. **Advisory** controls are instructions the agent reads and usually follows (`CLAUDE.md`, skills). **Deterministic** controls are code that runs on every matching action and allows or blocks it regardless of what the agent intends (hooks, permission rules, the sandbox, branch protection). The skill makes violations rare; the hook makes them close to impossible. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: the employee handbook vs the locked door. (analogy: general knowledge)

## Details

### The layers

| Control | Layer | What it does | Example |
| --- | --- | --- | --- |
| `CLAUDE.md` | Advisory | One-page onboarding file read at every session start: commands, conventions, architecture, common mistakes | "Money is always BigDecimal, never double" |
| Skill | Advisory | Versioned policy instructions that load when the task matches their description | API security standard: auth on every endpoint, no PII in logs |
| Hook | Deterministic | Script that runs before or after an action and can **allow**, **ask** (wait for a named human), or **block** | Block `deploy … production` without a release authorization |
| Permission rules | Deterministic | Allow/deny lists for tools, files, and commands | Deny reading `.env*`; allow `make test` |
| Sandbox | Deterministic (OS level) | Filesystem and network isolation for shell commands | Network allowlist of two domains; deny `~/.ssh` |
| Branch protection | Deterministic | Merges need a code owner's approval (and usually passing checks: general knowledge) | The agent cannot push to main |
| Managed settings | Deterministic | Org-wide settings pushed by IT that engineers cannot override | Only managed hooks and [[claude-code#Features the playbook uses\|MCP]] servers (connectors to other tools) may run |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

### Advisory controls

- `CLAUDE.md` turns team knowledge that lived in people's heads into a file. Rules: start from `/init`, cut it to what a new joiner needs on day one, check it into git so changes get reviewed like code, keep it under a page (all of it uses context every session), and "when Claude makes a mistake twice, the correction goes into `CLAUDE.md`". (src: [[2026-11-24-ai-native-sdlc-playbook]])
- A skill is a folder with a `SKILL.md`, whose frontmatter says when it triggers and whose body says what to do. Write a skill for institutional knowledge that must be applied consistently; keep everything else in `CLAUDE.md` or the prompt. When the policy changes, the skill changes, the policy owner signs off, and engineers get it in their next session. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Advisory controls are checked indirectly: if PR review keeps finding violations of a policy that has a skill, either the skill isn't triggering or its text has drifted from the policy. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Deterministic controls

- Any policy that has to hold without exception needs a deterministic control behind its skill. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Build-time hooks fire on almost every action, so they must be fast and scoped to the changed file: block edits to protected paths (generated code, a frozen package), run the formatter and linter after edits, keep credentials out of the diff. Heavy checks such as the full test suite belong at commit or PR time. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- A hook that asks a human belongs at the deploy gate, not during build. During build it would put a person on the critical path of every parallel session. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- In Claude Code, a hook exiting with code 2 blocks the action, and its message goes to the agent. A block should explain itself: the reason and the route to approval. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Protecting the feedback loop: when the agent is fixing code, a hook blocks it from editing the test files, so it cannot weaken the check on its own work. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Separation of duties: the agent that wrote a change has no way to approve it. Approval comes from a human through branch protection. Each non-interactive run acts under the agent's own identity, so logs separate what the agent did from what the engineer who triggered it did. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Managed settings for a regulated enterprise

The playbook's example locks down a fleet of engineer machines. What each key buys:

| Setting | Effect |
| --- | --- |
| `permissions.deny` | Keeps secrets out of the agent's context; blocks network tools (WebFetch, curl, wget) |
| `permissions.allow` | Pre-approves the safe inner loop (git, build, test, lint), so a long deny list doesn't cause prompt fatigue |
| `disableBypassPermissionsMode`, `allowManagedPermissionRulesOnly` | No engineer, project file, or flag can widen the rules |
| `sandbox` + `network.allowedDomains` | OS-level egress allowlist; closes the gap where a shell command reaches the network despite a tool-level deny |
| `failIfUnavailable`, `allowUnsandboxedCommands: false` | Refuses to start without a sandbox; a command that fails inside cannot retry outside |
| `sandbox.credentials` | Denies shell reads of `~/.ssh`, `~/.aws/credentials`; strips named secrets (e.g. `GITHUB_TOKEN`) from the environment |
| `allowManagedHooksOnly` | Only the organization's hooks run |
| `disableSideloadFlags`, `strictKnownMarketplaces` | Every skill, agent, hook, and MCP server comes from the approved plugin marketplace |
| `allowManagedMcpServersOnly` | The agent's tool surface is an allowlist owned by the platform team |
| `requiredMinimumVersion` | Refuses to run a version older than the one the organization assessed |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) Every deny trades away capability, so this is a starting point to tailor to the repo's data classification, not a template to copy. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Testing the guardrails

The configuration that steers the agent (`CLAUDE.md`, skills, hooks) is regression-tested like code: an eval suite runs on every change to it, and a change that drops the pass rate is reviewed before merge ([[evals]]). (src: [[2026-11-24-ai-native-sdlc-playbook]])

### GPU cloud bridge

General knowledge.

- Same split in a control plane: a documented quota is advice; a quota the API rejects requests over is a control. Tenancy and billing rules need the enforced kind, checked on every request.
- Defense in depth applies to both: tool-level denies, OS-level sandboxing, and network allowlists cover each other's gaps, as tenant isolation does across Kubernetes namespaces, network policy, and node separation.

## Related

- [[claude-code]]: where these controls are configured.
- [[deployment-gates]]: hooks used as human approval gates.
- [[evals]]: regression tests for the guardrail configuration.
- [[artifact-chain]]: the files the controls apply to.
- [[software-development-lifecycle]]: why per-line human review stopped scaling.

## Open questions

- How independent is an agentic reviewer from the agent that wrote the code when both use the same model family? The playbook doesn't address correlated blind spots.
