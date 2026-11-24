---
type: concept
tags: [software-delivery, lifecycle, ai-agents]
sources: [2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-24
updated: 2026-11-24
---

# Control bands

A **control band** is the normal range of a metric, computed from its own recent history: typically the rolling mean plus or minus some multiple of the standard deviation (σ). A value outside the band is a statistically unusual event worth a response. In the AI-native SDLC, a deterministic script watches the bands, and the size of the breach decides how much an agent may do. (src: [[2026-11-24-ai-native-sdlc-playbook]]) Analogy: a thermostat with three alarm levels, where the furthest from normal unlocks the strongest response. (analogy: general knowledge)

## Details

### The statistics, for beginners

General knowledge.

- **Mean:** the average value. **Standard deviation (σ, "sigma"):** how far values typically sit from the mean. **Rolling window:** compute both over the last N days, so the baseline follows normal change.
- For roughly normally distributed data, about 68% of points fall within 1σ of the mean, 95% within 2σ, and 99.7% within 3σ. A 3σ point is rare enough to be a real signal.
- **Western Electric (WECO) rules** come from industrial quality control (the Western Electric *Statistical Quality Control Handbook*, 1956). They flag a process as out of control on: (1) any one point beyond 3σ; (2) 2 of the last 3 points beyond 2σ on the same side; (3) 4 of the last 5 points beyond 1σ on the same side; (4) 8 consecutive points on the same side of the mean. Rules 2–4 catch slow drift and shifts, not just spikes. (verified: [NIST/SEMATECH e-Handbook, 6.3.2 (WECO rules)](https://www.itl.nist.gov/div898/handbook/pmc/section3/pmc32.htm))

### Tiered response

| Breach | Action | What the agent may do |
| --- | --- | --- |
| 1σ | Log only | Nothing |
| 2σ | Diagnose | Run read-only (e.g. `Read`, `Grep`, `gh run view`) and report |
| 3σ | Propose | Open a PR into the review gate, or trigger a pre-approved runbook (e.g. roll back a deploy) |

(src: [[2026-11-24-ai-native-sdlc-playbook]])

The tiers live in version-controlled config:

```yaml
metric: ci_test_failure_rate
baseline: rolling_30d
rules: western_electric
tiers:
  1sigma: { action: log }
  2sigma: { action: diagnose,
            tools: "Read,Grep,Bash(gh run view *)" }
  3sigma: { action: propose,
            routes: [pull_request, runbook:rollback-deploy] }
```

(src: [[2026-11-24-ai-native-sdlc-playbook]])

### How the loop works

1. Pick one metric with a stable rolling baseline, such as CI test failure rate, post-deploy 5xx (server error) rate, or PR cycle time. (src: [[2026-11-24-ai-native-sdlc-playbook]])
2. Write the detection script: rolling mean and σ plus Western Electric (or similar) rules. It is version-controlled, unit-tested, and **fully deterministic, with no model involved**. (src: [[2026-11-24-ai-native-sdlc-playbook]])
3. Trigger the agent from a scheduled GitHub or GitLab workflow, a webhook from the existing monitoring stack, or a cron job inside the network. Claude runs stateless, as a non-interactive step on a CI runner or as an Agent SDK service in a sandboxed container, so a loop can begin and end without anyone starting it. (src: [[2026-11-24-ai-native-sdlc-playbook]])
4. The agent writes its diagnosis as an `intent.md` in the Plan-stage format (the anomaly and its evidence, proposed outcome, affected systems, open questions), which enters the [[artifact-chain]] like any other intent. (src: [[2026-11-24-ai-native-sdlc-playbook]])
5. The service owner or on-call engineer triages the queue, routing product-facing findings to the product owner: fix now, schedule, or dismiss. Dismissals tune the bands to cut noise. (src: [[2026-11-24-ai-native-sdlc-playbook]])
6. When a fix ships, the incident becomes a permanent [[evals|eval]] case. (src: [[2026-11-24-ai-native-sdlc-playbook]])

- Examples: CI failure rate past 3σ → quarantine the flaky test or open a revert PR, and the review gate decides. Post-deploy 5xx past 3σ with a deployment in the window → trigger the existing rollback pipeline. PR cycle time trips a drift rule → write a report for engineering leadership, which shows the same machinery works for process metrics. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Governance: tier boundaries come from version-controlled config, with permissions and managed settings denying production access; invocations, findings, and triage decisions are logged with timestamps; runbooks the agent may trigger are approved in advance. (src: [[2026-11-24-ai-native-sdlc-playbook]])
- Measured by time from breach to an `intent.md` in the triage queue, the share of findings that become merged fixes, and whether repeat incidents of the same class fall. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### Why detection stays deterministic

The split is deliberate: plain code decides *that* something is wrong, reproducibly and testably, and the model is called only after a breach, with the tier limiting what it may do. (src: [[2026-11-24-ai-native-sdlc-playbook]]) This keeps the alerting cheap, explainable, and free of model variance. (general knowledge)

### GPU cloud bridge

General knowledge.

- Candidate metrics for an inference platform: per-endpoint error rate, time-to-first-token latency, queue depth, Pods stuck Pending ([[kubernetes-scheduling]]), GPU hardware error counts per Node.
- Band choice is a metric-design problem, like choosing a utilization metric: a band on GPU-Util can stay "normal" while useful throughput collapses ([[gpu-utilization-metrics]]).
- Inference traffic often has daily and weekly cycles, so a plain rolling mean flags every morning ramp. Bands on such metrics need seasonality-aware baselines.

## Related

- [[deployment-gates]]: rollback and runbooks are the 3σ routes.
- [[artifact-chain]]: the breach writes the next `intent.md`.
- [[evals]]: every incident becomes a regression case.
- [[gpu-utilization-metrics]]: picking metrics that mean what you think.
- [[software-development-lifecycle]]: the maintain stage that closes the loop.

## Open questions

- How do teams band metrics with strong daily cycles or near-zero baselines, where mean ± σ is a poor model?
