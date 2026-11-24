---
type: concept
tags: [software-delivery, ai-agents, lifecycle]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Deployment gates and tiered autonomy

A **deployment gate** is a checkpoint a change must pass before it reaches the next environment, most importantly production. With coding agents, the gates define how far an agent may go on its own. The governing principle: **"the agent may act up to the production gate and cannot pass it."** (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: an intern can draft, file, and prepare a contract for signature, but only a named manager signs it. (analogy: general knowledge)

## Details

### Tiered autonomy by environment

| Environment | What it is | What the agent may do |
| --- | --- | --- |
| Development | The engineer's own or a shared test environment | Deploy freely |
| Staging | A production-like rehearsal environment | "Somewhere in the middle" (the playbook doesn't specify) |
| Production | Real users | Prepare the release; a named release manager authorizes it; a hook enforces the gate |

(src: [[2026-11-24-ai-native-sdlc-playbook]]; "What it is" column: general knowledge)

### The gates

- **PR gate.** Anything the agent writes arrives as a pull request under branch protection. The agent has no route to push to main. Agentic review passes (bugs, security, compliance with `spec.md`/`plan.md`) run on every PR, but the findings neither approve nor block on their own. A code owner approves. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Approval hooks.** A [[agent-guardrails\|hook]] is a script that runs before the agent acts and can *allow*, *ask*, or *block*. Engineering leadership, with change management and compliance, lists the human gates that must survive (change-management sign-off, release authorization, edits to protected paths), and the platform engineer turns each into a hook. Team hooks live in the repo's `.claude/settings.json`; non-negotiable ones live in managed settings owned by the platform or IT admin, which engineers cannot switch off. A block should explain itself: the reason and the route to approval appear in the agent's output. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Example production gate: a `PreToolUse` hook on the Bash tool reads the command the agent is about to run; if it contains "deploy" and "production" and the `RELEASE_APPROVAL` environment variable is unset, the script exits with code 2, which blocks the action and feeds its stderr message to the agent. (src: [[2026-11-24-ai-native-sdlc-playbook]]; exit-code behaviour general knowledge; verified: [Claude Code hooks reference](https://code.claude.com/docs/en/hooks))
- Every allow/block decision is logged with a timestamp (to the OpenTelemetry export: a standard format for sending traces and logs to an observability system), so the time spent waiting at each gate is measurable. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Order matters: the PR review gate and approval hooks are prerequisites for CI/CD automation, "because the gates must exist before automation accelerates anything through them". (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The agent in the pipeline

- Adopt in steps. First, read-only judgment jobs with `claude -p` (non-interactive mode): triage a failed build (including whether the failure looks flaky or real), summarize a flaky test, draft the changelog. Then write jobs behind the existing gates: fix lint, update generated docs, address review comments via `@claude` mentions. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Agent jobs run sandboxed in containers, under a network policy, with short-lived scoped tokens and no production credentials by default. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Deploy, status, and rollback are exposed to the agent as tools through [[claude-code#Features the playbook uses|MCP]] (Model Context Protocol, a standard way to plug tools into an agent), scoped per environment. The agent's deployment powers are then an allowlist, not a shell script holding credentials. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Each non-interactive run acts under the agent's own identity, so the pipeline log separates what the agent did from what the engineer who triggered it did. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Success is measured by the share of pipeline failures triaged without paging a human (leading) and DORA measures (lagging), which CI and deployment tooling already emit. (src: [[2026-11-24-ai-native-sdlc-playbook]]) DORA (DevOps Research and Assessment) now uses five delivery metrics: change lead time, deployment frequency, failed deployment recovery time, change fail rate, and deployment rework rate (the fifth was added in 2024 to the original four keys). (general knowledge; verified: [DORA's software delivery performance metrics](https://dora.dev/guides/dora-metrics/))

### Rollback

- Rollback should be the most rehearsed path in the pipeline: a single command the agent can run, exercised regularly in staging. The automated maintenance loop calls it when production leaves its [[control-bands|control band]], so it must be proven in advance. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Pre-approved runbooks are the other gated route: at the highest autonomy tier, the agent may trigger only runbooks that were approved beforehand. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### GPU cloud bridge

General knowledge.

- In Kubernetes, a release is usually a rolling update of a [[kubernetes-deployment|Deployment]], and `kubectl rollout undo` rolls the Deployment back to its previous revision (old ReplicaSets are kept as revision history). It only reverts the Pod template, not separately changed ConfigMaps or Secrets. It is quick to run but still needs rehearsing: on GPU services the old version's Pods must find free GPUs and reload model weights (cold start), so a rollback can take minutes and needs spare capacity ([[kubernetes-scaling]]). (verified: [Kubernetes: Deployments, rolling back](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment))
- Control-plane rollouts across a GPU fleet usually go in waves (one cluster or region first) with gates between waves, the same "act up to the gate" pattern at a larger scale.

## Related

- [[agent-guardrails]]: the hooks, permissions, and sandbox the gates are built from.
- [[control-bands]]: the monitoring that triggers rollback and new work.
- [[artifact-chain]]: the PR as the audit record at the gate.
- [[kubernetes-deployment]]: rolling updates and rollback in Kubernetes.
- [[software-development-lifecycle]]: the deploy stage in context.

## Contradictions

- **What a hook's "ask" does.** The playbook says a hook can ask, "pausing the action until a specific person approves", which is what release gating needs. (src: [[2026-11-24-ai-native-sdlc-playbook]]) In Claude Code, a `PreToolUse` hook returning `"ask"` shows a permission prompt to the user running the session; in a non-interactive `-p` run, where no one can answer, the call is denied. Routing approval to a *named* person (e.g. a release manager) needs extra plumbing, as in the playbook's own example, which blocks (exit 2) unless a `RELEASE_APPROVAL` variable is set. (general knowledge; verified: [Claude Code hooks reference](https://code.claude.com/docs/en/hooks))

## Open questions

- How does the "agent prepares, human authorizes" model hold up for emergencies, when waiting for a release manager costs more than the risk of the change?
