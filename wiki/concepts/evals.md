---
type: concept
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Evals (agent evaluations)

An **eval** is a test for an AI system's behaviour: a prompt (a task) plus checks that define an acceptable result. A suite of evals tells you whether an agent still does its work to the same standard after something changes: a new model, a rewritten prompt, an edited skill. The playbook calls continuous evals "the AI-native equivalent of stage-gate QA". (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: a regression test suite, but for the agent's instructions rather than the product's code. (analogy: general knowledge)

## Details

- **What to test.** The configuration that steers the agent (`CLAUDE.md`, skills, hooks) deserves the same regression testing as code, because changing it changes what the agent produces. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Building the suite.** Collect 20–50 real tasks from recent work with their accepted outcomes. Each becomes an eval: the prompt, plus checks such as tests pass, lint clean, behaviour unchanged, policy followed. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **When it runs.** Non-interactively in CI, on a schedule and on any change to the agent's configuration. Some teams prefer an offline run on a fixed cadence instead. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Gating.** A configuration change that drops the pass rate is reviewed before it merges; the pass-rate threshold is a merge check. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Growing it.** Every production incident gets an eval, written by the team that owned the incident, and stays as a regression test. So do vulnerability classes found by scheduled security scans. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Decay.** The suite is live: as models improve, cases that used to separate good from bad stop doing so, and new cases from monitoring must replace them. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Cost.** Each eval run calls the model, so it needs an API key with its own budget. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Measured by the pass rate over time, how long an incident takes to become an eval, and regressions caught in CI vs found in production. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Evals vs the agent's own checks

| | Feedback loop | Verifier subagent | Eval suite |
| --- | --- | --- | --- |
| Question | Does *this change* work? | Is this change done, judged with fresh eyes? | Does *the agent* still work well? |
| When | Throughout one task, as often as needed | Once, when the session thinks it is done | On config changes and on a schedule |
| Who runs it | The working session | A separate context window | CI |

(src: [[2026-11-24-ai-native-sdlc-playbook]]) The verifier runs in a fresh context so its verdict isn't coloured by the assumptions that produced the code. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### GPU cloud bridge

General knowledge: the same idea applies when a platform swaps the model or the [[inference-frameworks|inference framework]] version behind an endpoint. A fixed set of real prompts with expected outputs catches quality regressions that latency and error metrics miss, for example after [[inference-optimization|quantization]].

## Related

- [[agent-guardrails]]: the configuration evals regression-test.
- [[control-bands]]: incidents found by monitoring become evals.
- [[claude-code]]: `claude -p` runs evals non-interactively in CI.
- [[software-development-lifecycle]]: evals replace stage-gate QA in the test stage.
