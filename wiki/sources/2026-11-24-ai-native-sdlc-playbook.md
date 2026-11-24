---
type: source
tags: [software-delivery, ai-agents, lifecycle]
sources: []
raw: raw/2026-11-24-ai-native-sdlc-playbook.md
url: https://claude.com/resources/articles/the-ai-native-sdlc-playbook
author: Louis Claxton (Anthropic Applied AI)
created: 2026-11-24
updated: 2026-11-24
---

# The AI-native SDLC playbook

Raw: [[raw/2026-11-24-ai-native-sdlc-playbook]] · Anthropic best-practices article, published 2026-08-21, clipped 2026-11-24 (a PDF version is linked from the raw file). The first source in the `software-delivery` and `ai-agents` areas.

**Read it as:** a vendor playbook. It shows how Anthropic recommends running [[claude-code]] across a whole engineering process, and its speed claims ("hours, not weeks") are the vendor's claims, not measurements. The control design (what is enforced, by whom, with what evidence) holds regardless of which agent is used. (framing: general knowledge)

## Summary

The [[software-development-lifecycle]] (SDLC) is the process that takes software from idea to production: plan, design, build, test, deploy, maintain. Coding agents now make the build stage fast, so the slow parts are the human-speed stages around it, and controls built for human-written code (line-by-line review, weekly change boards) can't keep up. The playbook's answer has three parts:
- an [[artifact-chain]]: each stage commits a file to git, and that commit starts the next stage;
- controls in two layers ([[agent-guardrails]]): advisory instructions (`CLAUDE.md`, skills) plus deterministic enforcement (hooks, permissions, sandbox, branch protection);
- [[deployment-gates]]: the agent may act up to the production gate and never past it.

The last stage closes the loop. A deterministic monitor watches production against [[control-bands]], and when a metric leaves its band, an agent diagnoses the problem and writes the next `intent.md`. Each incident also becomes a permanent [[evals|eval]] case. (src: [[2026-11-24-ai-native-sdlc-playbook]])

![[raw/assets/sdlc-bottleneck-shift.png]]
Two bars of the six stages. Before agents, Build is by far the longest stage. After agents, Build shrinks to a sliver, and Plan, Design, Test (review), Deploy (release), and Maintain keep their length. The time saved is labelled "cycle time reclaimed". (src: [[2026-11-24-ai-native-sdlc-playbook]])

![[raw/assets/sdlc-line-vs-loop.png]]
Left: the traditional SDLC as a straight line, where "one slow loop back is a new release cycle". Right: the AI-native SDLC as a circle of the same six stages around Claude, with "humans above the loop instigating, directing and governing". (src: [[2026-11-24-ai-native-sdlc-playbook]])

## Key points

### Why the process has to change

- When build is no longer the bottleneck, three things follow: the bottleneck moves to plan, review/test, and deploy, which still run at human speed; per-line human review can't keep up once agents write most of the diff; governance costs rise because exceptions still go through committees that meet weekly or monthly. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Security example: security teams are sized for human output. When agents multiply code output, either the review queue grows or code ships under-reviewed. A regulated organization can accept neither, so its checks must keep pace with the agents. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- PRDs (product requirement documents), estimation rituals, and security reviews existed to force alignment before weeks or months of coding work. That justification weakens when coding takes hours. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Other names for the same idea: agentic SDLC, AI SDLC, agentic software development. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Traditional vs AI-native, stage by stage

| Stage | Traditional | AI-native |
| --- | --- | --- |
| Plan | Requirements gathered by committee, written up by hand | Claude synthesizes pain points into `intent.md`, readable by humans and actionable by agents |
| Design | Analysts write the spec, designers parse it | Requirements and design in one agent session, constrained by policy written as skills, versioned in git |
| Build | Hand-written code and tests; docs written afterwards | Agent-generated code and tests; team knowledge kept in `CLAUDE.md` and skills |
| Test | QA gates at stage boundaries | Continuous evals during implementation |
| Deploy | Humans review every line; governance in review cycles | Layered agent review; humans review regulated and critical code; hooks as approval gates |
| Maintain | Humans watch production | Agents monitor; a breached control band becomes a new `intent.md` |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) Most organizations sit somewhere between the two columns.

### The artifact chain

- Each stage ends by committing an artifact, and the next stage starts by reading it: `intent.md` → `spec.md` → `plan.md` → diff and tests → PR with review findings → incident record. Details: [[artifact-chain]]. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Triggers: an accepted `intent.md` starts the design pass, an approved `spec.md` starts plan mode, a merged PR starts the pipeline, and a breached control band writes the next `intent.md`. Teams start by prompting each step by hand; the end state is a loop where each accepted artifact fires the next gate. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- The chain of commits is the audit trail: who asked for what, what the agent produced, who approved it. Humans stay accountable for every judgment call; their attention shifts to reviewing artifacts. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Existing tools (Jira, ServiceNow, Figma, change boards) stay. For each artifact, name one system as the source of truth: the repo, the legacy tool (synced through MCP), or at minimum two-way links between record ID and commit SHA. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The plays

The playbook calls each practice a "play". Every play lists what changes, prerequisites, infrastructure, steps, governance, and a leading and a lagging indicator. (src: [[2026-11-24-ai-native-sdlc-playbook]])

| Stage | Play | Core move | Measured by (leading / lagging) |
| --- | --- | --- | --- |
| Plan | Capture intent | Originator brainstorms with Claude, product owner reviews, `intent.md` committed | Time to committed intent / share accepted into design |
| Design | Requirements & design | One session turns `intent.md` into `spec.md` under policy skills, flagging conflicts | Intent→spec time / spec commits after plan starts |
| Build | Plan mode | Agent reads code read-only, writes a plan, engineer approves it as `plan.md` | First-pass merge share / rework cycles |
| Build | `CLAUDE.md` | One-page onboarding file the agent reads every session | Repeated mistakes / new joiner's time to first PR |
| Build | Skills | Policy as versioned instructions that load when relevant | Policy→skill time / review findings citing the policy |
| Build | Hooks (guardrails) | Scripts that block protected-path edits, run linters, keep secrets out | (none given) |
| Build | Parallel sessions & subagents | Several sessions in separate git worktrees; recurring jobs as subagents | Sessions per engineer / merged changes vs rework |
| Test | Feedback loop | The agent verifies its own work (tests, build, screenshot) before a human sees it | First-pass CI rate / review time, change failure rate |
| Test | Continuous evals | 20–50 real tasks re-run whenever the agent's configuration changes | Pass rate, incident→eval time / CI vs production regressions |
| Deploy | AI in PR review | Same review passes on every PR, findings ranked; `@claude` fixes comments | Time to first review / defects caught before merge |
| Deploy | Hooks as approval gates | A hook can allow, ask, or block, e.g. production deploys | Wait time per gate / gate violations in production |
| Deploy | CI/CD | Non-interactive agent in the pipeline, sandboxed, deploy tools via MCP | Failures triaged without paging / DORA metrics |
| Maintain | Closing the loop | Deterministic σ-bands trigger tiered agent response | Breach→intent time / findings that become fixes |
| Maintain | Recurring scans | Scheduled security scans (Claude Security); findings go through the review gate | Repos on schedule / scan vs production findings |
| Maintain | Claude Tag | Claude as an incident-channel member under its own identity | (none given) |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) DORA = DevOps Research and Assessment. Its delivery metrics were four keys (deployment frequency, lead time for changes, change failure rate, time to restore service) until 2024; it now uses five: change lead time, deployment frequency, failed deployment recovery time, change fail rate, deployment rework rate. The playbook only says "DORA measures". (metric names: general knowledge, dora.dev)

![[raw/assets/sdlc-play-dependencies.png]]
The adoption-order graph. The five plays with no prerequisites sit on top: capture intent, `CLAUDE.md`, feedback loop, hooks, plan mode. `CLAUDE.md` leads to skills, subagents, and evals; the feedback loop also leads to evals (and softly to subagents). Skills plus intent lead to requirements & design. Evals lead to PR review (skills softly too), then CI/CD (which also needs hooks), then closing the loop (which also needs intent). Plan mode has nothing after it. Dotted arrows are soft dependencies. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Controls: advisory vs enforced

- `CLAUDE.md` and skills make the right behaviour *likely*. Hooks, permission rules, the sandbox, and branch protection make violations close to impossible. A policy that must always hold needs the second layer behind the first. Details: [[agent-guardrails]]. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Separation of duties: the agent that wrote the code has no way to approve it. Approval comes from a human code owner through branch protection. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- For regulated enterprises, managed settings (pushed by IT, not editable by engineers) can lock down secrets, network egress, hooks, MCP servers, plugins, and the minimum Claude Code version. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- The agent's working rules are themselves tested: evals run on every change to `CLAUDE.md`, skills, or hooks ([[evals]]). (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Deploy and maintain

- "The agent may act up to the production gate and cannot pass it." Autonomy is tiered per environment: free in development, partial in staging, prepare-only in production, where a named release manager authorizes. Details: [[deployment-gates]]. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Rollback should be the most rehearsed path: one command the agent can run, exercised regularly in staging. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Detection stays deterministic: a script compares a metric with its rolling mean and standard deviation. At 1σ it only logs, at 2σ it invokes Claude read-only to diagnose, at 3σ Claude may open a PR or run a pre-approved runbook. Details: [[control-bands]]. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Scheduled security scans: a scan is valid only for one codebase and one model, and both go stale (code changes weekly; each model generation finds bugs the last one missed). So scans run on a schedule, and findings go through the same review gate as any change. (src: [[2026-11-24-ai-native-sdlc-playbook]])

![[raw/assets/sdlc-claude-tag-incident.png]]
A Slack incident channel. A human reports a 5xx rise after a deploy and tags @claude. Claude matches error signatures against the deploy diff, blames a new cache key that drops the tenant id, and asks before running the rollback rehearsed in staging that morning. After the human says "Go", Claude rolls back, confirms the 5xx rate is back inside its band, and writes the post-mortem to `lessons/2026-06-checkout-cache.md`. (src: [[2026-11-24-ai-native-sdlc-playbook]])

## Takeaways

- Speeding up one stage of a pipeline moves the bottleneck; it doesn't remove it. The gains depend on the stages that still run at human speed. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- A control is only as strong as its enforcement. An instruction an agent "should" follow is advice; a check that blocks the action is a control. Keep both, and know which is which. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Writing every stage's output to git as a file turns the process into data. You can trigger on it, measure it, and audit it. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Use the model for judgment and plain code for detection. A deterministic script decides *that* something is wrong; the agent helps decide *why* and *what to do*, inside gated routes. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- GPU cloud bridge: the example incident (a cache key that dropped the tenant id) is a tenancy bug. Shared caches must include the tenant in their keys, or one customer's data leaks to another. (general knowledge)

## Concepts & entities touched

[[software-development-lifecycle]] · [[artifact-chain]] · [[agent-guardrails]] · [[deployment-gates]] · [[control-bands]] · [[evals]] · [[claude-code]] · [[kubernetes-deployment]] · [[control-plane]]

## Questions raised

- What do the DORA metrics measure exactly, and how do teams collect them? (Next: the DORA research site or the *Accelerate* book.)
- What are the Western Electric rules, and how do they catch slow drift as well as spikes? (Next: an SPC / control-chart primer.)
- How do the playbook's controls apply to infrastructure changes, such as a control-plane rollout across a GPU fleet, where a "rollback" may mean draining Nodes?
- Agentic review and agent-written code both come from models. How independent are they really, and when is that "separation of duties" not enough?
- The companion article "how Anthropic secures its AI-native SDLC" covers how these controls compose at production scale.
