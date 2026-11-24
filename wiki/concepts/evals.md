---
type: concept
tags: [ai-agents, software-delivery]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Evals (agent evaluations)

An **eval** is a test for an AI system's behaviour: a prompt (a task) plus the checks that define an acceptable result. A suite of evals tells you whether an agent still does its work to the same standard after something changes: a new model, a rewritten prompt, an edited skill. The playbook calls evals "the AI-native equivalent of stage-gate QA" (a QA sign-off between stages). (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: a regression test suite, but for the agent's instructions rather than the product's code. (analogy: general knowledge)

## Details

- **What to test.** The configuration that steers the agent (`CLAUDE.md`, skills, hooks) deserves the same regression testing as code, because changing it changes what the agent produces. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Building the suite.** A platform engineer collects 20–50 real tasks from recent work with their expected or accepted outcomes. Each becomes an eval: the prompt, plus checks such as tests pass, lint clean, behaviour unchanged, policy followed. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **When it runs.** Non-interactively in CI, on a schedule and on any change to `CLAUDE.md`, skills, or hooks. Depending on the use case, some teams run evals offline on a set cadence instead of on every change. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Gating.** A skill or other configuration change that drops the pass rate gets reviewed before it merges; the pass-rate threshold is enforced as a merge check, and the team that owns the change approves it. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Growing it.** Every production incident gets an eval, written by the team that owned the incident, and stays as a regression test. When a fix for a vulnerability found by a scheduled security scan reaches production, an eval for that vulnerability class is added too. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Decay.** The suite is live: as models improve, cases that used to discriminate (separate good from bad output) stop doing so, and new cases arising from monitoring must be added. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Prerequisites and cost.** A `CLAUDE.md` and a feedback loop first; then CI that can run Claude Code non-interactively and an API key with budget for eval runs. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- **Measuring it.** Leading: the pass rate over time and how long an incident takes to become a permanent eval. Lagging: regressions caught in CI vs regressions found in production. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The playbook's example workflow

A GitHub Actions workflow (`.github/workflows/agent-evals.yml`) runs on pull requests that touch `CLAUDE.md` or `.claude/**` and nightly (`cron: '0 2 * * *'`, 02:00 UTC). For each `evals/*.json` file it runs `claude -p "<prompt>" --allowedTools "Read,Edit,Bash(make test)" --output-format json` and passes the result to a `check.sh` script. (src: [[2026-11-24-ai-native-sdlc-playbook]]) The `--allowedTools` list matters: a `claude -p` run with no permission mode starts in Manual (`default`) mode, where tool calls that would need approval are denied unless an allow rule, hook, or auto mode approves them. (general knowledge; verified: [Choose a permission mode](https://code.claude.com/docs/en/permission-modes), [Run Claude Code programmatically](https://code.claude.com/docs/en/headless))

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
