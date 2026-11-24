---
type: concept
tags: [software-delivery, ai-agents]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Artifact chain

An **artifact chain** is a way of running the [[software-development-lifecycle]] where every stage ends by committing a file to git, and the next stage starts by reading that file. Humans and agents can both read and act on the same file, the commit can trigger the next step automatically, and the sequence of commits is the audit trail. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: a relay race where the baton is a written note, and every handoff is photographed with a timestamp. (analogy: general knowledge)

## Details

### The chain

| Artifact | Written by | Approved by | Its acceptance triggers |
| --- | --- | --- | --- |
| `intent.md`: problem, proposed outcome, affected users/systems, constraints, open questions | The originator with Claude, or a monitoring agent | Product owner (merge = accept, closed review = reject) | The requirements & design pass |
| `spec.md`: requirements and design, with flagged policy conflicts | Claude, constrained by policy skills | Product owner; tech lead for higher-risk changes | Plan mode |
| `plan.md`: files that change, order of work, risks, proof (tests) | Claude in plan mode, interviewed by the engineer | Engineer; tech lead/architect for higher risk | Implementation |
| Diff and tests | Claude | Code owner, through PR review | The CI/CD pipeline on merge |
| PR with review findings | Agentic review passes + humans | Code owner (branch protection) | Deploy, up to the production gate |
| Incident record / new `intent.md` | Agent after a [[control-bands\|control-band]] breach | Service owner triages | The next loop |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

- Early stages use markdown because a product owner and an agent can both read and edit the same file. From build onward, the artifact is code and its records. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- `spec.md` is committed next to `intent.md`: together they record what was asked for and what was decided. When implementation departs from the plan, `plan.md` is updated in the same commit (a hook can enforce this). PR review checks the diff against `spec.md` and `plan.md`. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- The design-stage log includes the prompt that produced the spec and the versions of the skills in force, so the policy applied to a decision can be reconstructed. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Non-engineers don't need to learn git. A connector lets Claude commit their markdown files from the chat app. For one product, the simplest home is an `intent/` folder in the product repo; a separate intent repo pays off only when intent spans many repos. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Adoption is gradual. Teams first prompt each step by hand. The end state is a loop where each accepted artifact fires the next gate, and humans spend their attention at the gates. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Measuring with git timestamps

Because every handoff is a commit, process metrics come straight from git: intent→spec time, spec→plan time, spec commits made after the plan started (rework), and how often the merged diff still matches `plan.md`. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Coexisting with existing tools

Organizations already keep these records in Jira, ServiceNow, requirements tools, Figma, and change boards, which auditors accept. For each artifact, pick one source of truth:

| Option | Authoritative record | How the other side stays in sync |
| --- | --- | --- |
| Repo is the source of truth | The markdown files | The legacy tool links to commits |
| Legacy tool is the source of truth | Jira / ServiceNow / requirements tool | Claude reads the record at session start and writes back through MCP |
| Linkage (minimum bar) | Both (accepted as two sources) | Every artifact notes the record ID; every record holds the commit SHA |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

### Bridge to other areas

General knowledge.

- This is the same idea as Kubernetes' declarative desired state ([[kubernetes-scaling]], [[control-plane]]): write down what should be true, let a controller act on it, and keep the written record as the truth. Teams that keep Kubernetes desired state in git and let a controller apply it call this **GitOps**.
- This wiki works the same way: `raw/` sources, wiki pages, and `log.md` are committed files, and the schema in `AGENTS.md` tells the agent what to do with them.

## Related

- [[software-development-lifecycle]]: the stages the chain connects.
- [[agent-guardrails]]: the controls that apply while each artifact is produced.
- [[deployment-gates]]: where the chain meets production.
- [[control-bands]]: how production writes the next `intent.md`.
- [[claude-code]]: plan mode, which produces `plan.md`.
