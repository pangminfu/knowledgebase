---
type: concept
tags: [software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Software development lifecycle (SDLC)

The **software development lifecycle (SDLC)** is the process that takes software from an idea to running in production, and keeps it running. Most organizations use some version of six stages: plan, design, build, test, deploy, maintain. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Phase names and counts vary between sources (e.g. AWS uses plan, design, implement, test, deploy, maintain; others split out requirements analysis), but the sequence is the same. (general knowledge; verified: [AWS: What is SDLC?](https://aws.amazon.com/what-is/sdlc/)) Analogy: building a house. Someone decides what's needed (plan), an architect draws it (design), builders build it (build), an inspector checks it (test), the owners move in (deploy), and repairs follow for years (maintain). (analogy: general knowledge)

## Details

### The six stages

| Stage | What happens | Traditional owner | Output |
| --- | --- | --- | --- |
| Plan | Decide what problem to solve and why | Product manager | Requirements |
| Design | Decide how the solution will look and work | Technical architect | Spec / design |
| Build | Write the code and its tests | Engineers | Code changes |
| Test | Check that it works and nothing else broke | QA team ("at regulated enterprises") | Test results, sign-off |
| Deploy | Release it to users | Release team | A running new version |
| Maintain | Monitor what is running, fix what breaks | Operations | Incidents, fixes |

(src: [[2026-11-24-ai-native-sdlc-playbook]]; "Output" column: general knowledge)

- Traditionally each stage is a separate phase owned by a different role, and work moves between phases through documents, tickets, and sign-offs. The process is heavy on purpose: it creates accountability and control at each step. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- That process was designed for a time when build (writing and implementing code) was the slowest, most expensive stage. PRDs (product requirement documents), estimation rituals, and product security reviews existed to force alignment before weeks, months, or quarters of development work. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Its controls also assume every step is done by a human, for example a reviewer reading every line of a change. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The AI-native SDLC

- When coding agents shrink build to hours, the bottleneck moves to the stages around it (mainly plan, review/test, and deploy), which still run at human speed. Per-line review stops scaling, and governance costs rise because exceptions still route through committees that meet weekly or monthly. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- The **AI-native** (or agentic) SDLC keeps the old control objectives but enforces them differently. The line becomes a loop with an agent at every stage. Each stage commits a file that starts the next ([[artifact-chain]]). Controls are enforced as the agent acts ([[agent-guardrails]], [[deployment-gates]]). Production monitoring feeds new work back into Plan ([[control-bands]]). (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Humans stay accountable for every judgment call. Their work shifts from doing each stage to reviewing what the agent produced and flagged. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Stage-by-stage comparison table and the full list of plays: [[2026-11-24-ai-native-sdlc-playbook]].

![[raw/assets/sdlc-line-vs-loop.png]]
Traditional SDLC as a line (a loop back means a new release cycle) vs the AI-native SDLC as a loop of the same six stages around the agent, with humans above the loop. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Beginner glossary

General knowledge, except where cited.

| Term | Meaning |
| --- | --- |
| **git / repository (repo)** | Version control: a full history of every change to a set of files. The repo is one project's history. |
| **commit** | One saved change in git, with author, timestamp, and message. |
| **branch** | A separate line of changes, merged back into the main line (`main`) when ready. |
| **diff** | The exact lines a change adds and removes. |
| **pull request (PR)** | A request to merge a branch into main. It is where review, discussion, and automated checks happen. |
| **code review** | Another person (or, now, an agent) reads a PR before it merges. |
| **code owner** | A person or team named (on GitHub, in a `CODEOWNERS` file) as responsible for part of the code. Their approval is required only if the repo's branch protection turns that rule on. (verified: [GitHub: About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)) |
| **branch protection** | Repo rules that block merging into main without required approvals and passing checks. |
| **CI/CD** | Continuous integration / continuous delivery (or deployment): automation that builds and tests every change (CI), keeps it always releasable (continuous delivery), or releases every passing change automatically (continuous deployment). The automation runs as a *pipeline* of steps. |
| **environment** | A place the software runs: development, staging (a production-like rehearsal), production (real users). |
| **rollback** | Returning production to the previous working version. |
| **runbook** | Written step-by-step procedure for an operational task, such as a rollback. |
| **incident / post-mortem** | A production problem / the written analysis of what happened and how to prevent it. |
| **blast radius** | How much can break if a change goes wrong. The playbook lists a small blast radius (with a tight `spec.md` and code the tests already cover) as the condition for making auto-accept the default for routine work (src: [[2026-11-24-ai-native-sdlc-playbook]]). |

## Related

- [[artifact-chain]]: the files that carry work between stages.
- [[agent-guardrails]]: how controls change when an agent writes the code.
- [[deployment-gates]]: the deploy stage's controls and rollback.
- [[control-bands]]: how the maintain stage feeds back into plan.
- [[evals]]: the AI-native form of the test stage's QA gate.
- [[claude-code]]: the coding agent the playbook builds the process around.
- [[control-plane]]: a GPU cloud's control plane is itself software shipped through an SDLC.

## Open questions

- Only one source so far, written by a coding-agent vendor. A neutral source on the traditional SDLC (waterfall vs agile, DevOps, DORA metrics) would anchor this page.
