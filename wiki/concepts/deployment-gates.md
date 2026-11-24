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
| Staging | A production-like rehearsal environment | In between: set per organization |
| Production | Real users | Prepare the release; a named release manager authorizes it; a hook enforces the gate |

(src: [[2026-11-24-ai-native-sdlc-playbook]]; "What it is" column: general knowledge)

### The gates

- **PR gate.** Anything the agent writes arrives as a pull request under branch protection. The agent has no route to push to main. Agentic review passes (bugs, security, compliance with `spec.md`/`plan.md`) run on every PR, but the findings neither approve nor block on their own. A code owner approves. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Approval hooks.** A hook can *ask*: pause the action until a specific person approves. Leadership, change management, and compliance list the human gates that must survive (change sign-off, release authorization, edits to protected paths), and the platform engineer turns each into a hook. Team hooks live in the repo's `.claude/settings.json`; non-negotiable ones live in managed settings that engineers cannot turn off. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Example production gate: a script reads the command the agent is about to run; if it contains "deploy" and "production" and no `RELEASE_APPROVAL` is set, it exits with code 2, which blocks the action and tells the agent why. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Every allow/block decision is logged with a timestamp (to the OpenTelemetry export), so the time spent waiting at each gate is measurable. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Order matters: the PR review gate and approval hooks are prerequisites for CI/CD automation, "because the gates must exist before automation accelerates anything through them". (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The agent in the pipeline

- Adopt in steps. First, read-only judgment jobs with `claude -p` (non-interactive mode): triage a failed build, judge whether a test failure is flaky, draft the changelog. Then write jobs behind the existing gates: fix lint, update generated docs, address review comments. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Agent jobs run sandboxed in containers, under a network policy, with short-lived scoped tokens and no production credentials by default. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Deploy, status, and rollback are exposed to the agent as tools through MCP, scoped per environment. The agent's deployment powers are then an allowlist, not a shell script holding credentials. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Success is measured with the DORA metrics, which CI and deployment tooling already emit. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Rollback

- Rollback should be the most rehearsed path in the pipeline: a single command the agent can run, exercised regularly in staging. The automated maintenance loop calls it when production leaves its [[control-bands|control band]], so it must be proven in advance. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Pre-approved runbooks are the other gated route: at the highest autonomy tier, the agent may trigger only runbooks that were approved beforehand. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### GPU cloud bridge

General knowledge.

- In Kubernetes, a release is usually a rolling update of a [[kubernetes-deployment|Deployment]], and rollback means returning to the previous ReplicaSet (`kubectl rollout undo`). It is quick to run but still needs rehearsing: on GPU services the old version's Pods must find free GPUs and reload model weights (cold start), so a rollback can take minutes and needs spare capacity ([[kubernetes-scaling]]).
- Control-plane rollouts across a GPU fleet usually go in waves (one cluster or region first) with gates between waves, the same "act up to the gate" pattern at a larger scale.

## Related

- [[agent-guardrails]]: the hooks, permissions, and sandbox the gates are built from.
- [[control-bands]]: the monitoring that triggers rollback and new work.
- [[artifact-chain]]: the PR as the audit record at the gate.
- [[kubernetes-deployment]]: rolling updates and rollback in Kubernetes.
- [[software-development-lifecycle]]: the deploy stage in context.

## Open questions

- How does the "agent prepares, human authorizes" model hold up for emergencies, when waiting for a release manager costs more than the risk of the change?
