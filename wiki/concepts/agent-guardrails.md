---
type: concept
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Agent guardrails: advisory vs enforced controls

**Agent guardrails** are the controls that keep a coding agent inside an organization's rules. They come in two layers. **Advisory** controls are instructions the agent reads and usually follows (`CLAUDE.md`, skills); nothing forces a session to comply. **Deterministic** controls are code or configuration that is checked on every matching action and allows or blocks it regardless of what the agent intends (hooks, permission rules, the sandbox, branch protection). In the playbook's words, the skill makes violations rare and the hook makes them close to impossible. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: the employee handbook vs the locked door. (analogy: general knowledge)

## Details

### The layers

| Control | Layer | What it does | Example |
| --- | --- | --- | --- |
| `CLAUDE.md` | Advisory | Short onboarding file the agent reads at the start of every session: commands, conventions, architecture, common mistakes | "Money is always BigDecimal, never double" |
| Skill | Advisory | Versioned policy instructions that load when the task matches the skill's description | API security standard: auth on every endpoint, no PII in logs |
| Hook | Deterministic | Script that runs at a point in the agent's work (e.g. before a tool call) and can **allow**, **ask** (show a confirmation prompt), or **block** the action | Block `deploy … production` without a release authorization |
| Permission rules | Deterministic | Allow / ask / deny lists for tools, files, and commands | Deny reading `.env*`; allow `make test` |
| Sandbox | Deterministic (OS level) | Filesystem and network isolation for the shell commands the agent runs | Network allowlist of two domains; deny reads of `~/.ssh` |
| Branch protection | Deterministic | Merges need a code owner's approval (and usually passing checks: general knowledge) | The agent cannot push to main |
| Managed settings | Deterministic | Org-wide settings deployed by IT or the platform team that engineers cannot override | Only the organization's hooks run; MCP servers (connectors to other tools) limited to an allowlist |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) Where each control is configured in Claude Code: [[claude-code#Features the playbook uses]].

### Advisory controls

- `CLAUDE.md` turns team knowledge that lived in people's heads into a file. Rules: start from `/init`, cut it to what a new joiner needs on day one, check it into git so changes get reviewed like code, keep it under a page (Claude reads all of it every session, so stale lines waste context), and "when Claude makes a mistake twice, the correction goes into `CLAUDE.md`". (src: [[2026-11-24-ai-native-sdlc-playbook]]) Anthropic's docs give a concrete size target: under 200 lines per file, because longer files use more context and reduce adherence. (general knowledge; verified: [How Claude remembers your project](https://code.claude.com/docs/en/memory))
- A skill is a folder with a `SKILL.md`, whose frontmatter says when it triggers and whose body says what to do. Write a skill for institutional knowledge that must be applied consistently; keep everything else in `CLAUDE.md` or the prompt. When the policy changes, the skill changes, the policy owner signs off, and engineers pick up the new version in their next session. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Advisory controls are checked indirectly: if PR review keeps finding violations of a policy that has a skill, either the skill isn't triggering or its text has drifted from the policy. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Deterministic controls

- A policy that has to hold without exception needs something deterministic behind its skill: a hook that blocks the action, or a review pass that re-checks the policy at the PR. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Build-time hooks run on every action that matches them, and most build actions are file edits and shell commands, so build is where hooks fire most often. They must be fast and scoped to the changed file: block edits to protected paths (generated code, a frozen package), run the formatter and linter after edits, keep credentials out of the diff. Heavy checks such as the full test suite belong at commit or PR time. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- A hook that asks a human belongs at the deploy gate, not during build: during build it would put a person on the critical path of every parallel session. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- How a Claude Code hook blocks: a `PreToolUse` hook (runs before a tool call) that exits with code 2 blocks the call, and its stderr message is shown to Claude. Instead of an exit code, the hook can print JSON with `permissionDecision` set to `allow`, `deny`, `ask`, or `defer`. `ask` shows the normal permission prompt to whoever is running the session; in a non-interactive `claude -p` run, where nobody can answer, the call is denied. (general knowledge; verified: [Hooks reference](https://code.claude.com/docs/en/hooks)) See Contradictions for the playbook's "named approver" reading of `ask`.
- A block should explain itself: the reason and the route to approval appear in Claude's output. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Protecting the feedback loop: when the agent is fixing code, a hook blocks it from editing the test files, so it cannot weaken the check on its own work. The fallback is rejecting, in review, any fix that touches a test. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Separation of duties: the agent that wrote a change has no way to approve it. Approval comes from a human through branch protection. Each non-interactive run acts under the agent's own identity, so pipeline logs separate what the agent did from what the engineer who triggered it did. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Defense in depth between tools and the shell: permission rules govern Claude's built-in tools (Read, Edit, WebFetch, …), while the sandbox governs shell commands and their child processes. A sandbox network allowlist does not limit WebFetch, and a tool-level deny on WebFetch does not stop `curl` in a shell, so the two cover each other's gaps. The sandbox uses Seatbelt on macOS and bubblewrap on Linux/WSL2; on native Windows commands run unsandboxed. (general knowledge; verified: [Sandboxing](https://code.claude.com/docs/en/sandboxing))

### Managed settings for a regulated enterprise

The playbook's worked example locks down a fleet of engineer machines. What each key buys (src: [[2026-11-24-ai-native-sdlc-playbook]]), with how Claude Code's settings reference defines it (general knowledge; verified: [Settings reference](https://code.claude.com/docs/en/settings-reference)):

| Setting | Effect |
| --- | --- |
| `permissions.deny` | Keeps secrets out of the agent's context (`Read(.env*)`, `Read(./secrets/**)`); blocks network tools (`WebFetch`, `Bash(curl *)`, `Bash(wget *)`) |
| `permissions.allow` | Pre-approves the safe inner loop (git, build, test, lint), so a long deny list doesn't cause prompt fatigue |
| `permissions.disableBypassPermissionsMode: "disable"` | Nobody can enter `bypassPermissions` mode; the `--dangerously-skip-permissions` flag is rejected |
| `allowManagedPermissionRulesOnly` | Permission rules come only from managed settings; allow/ask/deny rules in user, project, and local files and `--allowedTools` are ignored. With the line above: no engineer, project file, or flag can widen the rules |
| `sandbox.enabled` + `sandbox.network.allowedDomains` | OS-level egress allowlist for shell commands; closes the gap where a shell command reaches the network despite a tool-level deny |
| `sandbox.failIfUnavailable`, `sandbox.allowUnsandboxedCommands: false` | Claude Code exits at startup if the sandbox can't start; a command blocked inside the sandbox cannot be retried outside it |
| `sandbox.credentials` | `deny` entries block sandboxed reads of `~/.ssh` and `~/.aws/credentials` (readable by default) and remove named variables (e.g. `GITHUB_TOKEN`) from sandboxed commands' environment |
| `allowManagedHooksOnly` | Only managed hooks run (plus hooks from plugins that managed settings force-enable, and Agent SDK in-process hooks); user, project, and local hooks are blocked |
| `disableSideloadFlags` | Rejects the `--plugin-dir`, `--plugin-url`, `--agents`, and `--mcp-config` CLI flags, which could otherwise bypass the marketplace allowlist for one run |
| `strictKnownMarketplaces` | Plugins can be added and installed only from the listed marketplace sources (the playbook lists one approved GitHub repo) |
| `allowManagedMcpServersOnly` | Only the managed MCP allowlist (`allowedMcpServers`) applies; allowlists in user, project, and local settings are ignored. The agent's tool surface becomes an allowlist owned by the platform team |
| `requiredMinimumVersion` | Claude Code refuses to start below the version the organization assessed (checked at startup only) |

Every deny trades away capability, so this is a starting point to tailor to the repo's data classification, not a template to copy. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Where managed settings are delivered from: [[claude-code#Settings files and precedence]].

### Testing the guardrails

The configuration that steers the agent (`CLAUDE.md`, skills, hooks) is regression-tested like code: an eval suite runs on every change to it, and a change that drops the pass rate is reviewed before merge ([[evals]]). (src: [[2026-11-24-ai-native-sdlc-playbook]])

### GPU cloud bridge

General knowledge.

- Same split in a [[control-plane]]: a documented quota is advice; a quota the API rejects requests over is a control. Tenancy and billing rules need the enforced kind, checked on every request.
- Defense in depth applies to both: tool-level denies, OS-level sandboxing, and network allowlists cover each other's gaps, as tenant isolation does across Kubernetes namespaces, network policy, and node separation.

## Related

- [[claude-code]]: where these controls are configured.
- [[deployment-gates]]: hooks used as human approval gates.
- [[evals]]: regression tests for the guardrail configuration.
- [[artifact-chain]]: the files the controls apply to.
- [[software-development-lifecycle]]: why per-line human review stopped scaling.

## Contradictions

- **What a hook's "ask" does.** The playbook says a hook "can also ask, pausing the action until a specific person approves, which is what release gating needs". (src: [[2026-11-24-ai-native-sdlc-playbook]]) In Claude Code, `ask` only shows the permission prompt to the user running the session, and in unattended `claude -p` runs it is treated as a denial. Gating on a *named* approver has to be built into the hook's own logic; the playbook's own example script does this by blocking (exit 2) unless a `RELEASE_APPROVAL` variable is set, not by asking. (general knowledge; verified: [Hooks reference: PreToolUse decision control](https://code.claude.com/docs/en/hooks))
- **Does the example lock down every skill, agent, hook, and MCP server?** The playbook says `disableSideloadFlags` plus `strictKnownMarketplaces` mean every skill, agent, hook, and MCP server on an engineer's machine arrived through the approved plugin marketplace, "never from a home directory". (src: [[2026-11-24-ai-native-sdlc-playbook]]) The settings reference says otherwise: those two keys only restrict marketplace sources and reject sideload CLI flags. Users can still add MCP servers with `claude mcp add` or `.mcp.json`, and skills and agents in `~/.claude/` and `.claude/` still load. Blocking user and project skills, agents, hooks, and MCP servers takes `strictPluginOnlyCustomization`, which the example does not set. Hooks are covered separately by `allowManagedHooksOnly`. (general knowledge; verified: [Settings reference: strictPluginOnlyCustomization, disableSideloadFlags](https://code.claude.com/docs/en/settings-reference))
- **MCP allowlist with no list.** The example sets `allowManagedMcpServersOnly: true` but defines no managed `allowedMcpServers`. The docs pair the two keys: the flag only makes the managed allowlist the one that applies, and an unset allowlist allows every server. (general knowledge; verified: [Managed MCP](https://code.claude.com/docs/en/managed-mcp), [Settings reference: allowedMcpServers](https://code.claude.com/docs/en/settings-reference))

## Open questions

- How independent is an agentic reviewer from the agent that wrote the code when both use the same model family? The playbook doesn't address correlated blind spots.
