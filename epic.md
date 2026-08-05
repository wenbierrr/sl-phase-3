# Problem

Duty engineers spend a significant share of their rotation on repetitive day-2 operations — monitoring, triage, and debugging cluster issues. LLM-based agents may be able to take on the execution layer of this work, freeing duty engineers for higher-leverage work: policy, guardrails, and design.

This issue covers evaluating whether that's viable for us, and if so, what a safe implementation looks like. The output is a benefit-vs-cost assessment the engineering team can act on.

# Scope

**In scope:** profiling the duty engineer workload, evaluating agent tooling against our operational constraints, and producing a PoC plus proposal.

**Out of scope:** LLM hosting (assume an endpoint is provided), model selection.

# Evaluation criteria

Any candidate solution is assessed against both sides of the ledger. Benefit is measured against the baseline established during profiling.

## Benefit

| Criterion | Question to answer |
|---|---|
| Coverage | What share of profiled duty chores can the agent handle, fully or partially? |
| Resolution quality | Does the agent reach correct outcomes, and how often does its output need rework or rollback? A wrong answer that looks right costs more than no answer. |
| Time to resolution | Does triage and diagnosis get faster, particularly for the long-tail issues that currently need escalation? |

## Cost and risk

| Criterion | Question to answer |
|---|---|
| Blast radius | What can the agent change? What guardrails constrain it, and how is execution vetted before it lands? |
| Security | Who can invoke the agent, and how is access controlled? |
| Auditability | Who ran what? How does duty engineer identity map to agent actions in the audit trail? |
| Cost | Order-of-magnitude infra and token cost at expected duty-rotation volume, set against the toil reduction above |
| Risk | What are the failure modes, and what's the recovery path? |
| Maintenance burden | What ongoing effort does the agent itself need — prompt upkeep, tool integration, drift as the platform changes? |
| Portability | How much does the solution couple us to a single platform? OpenShift-native options aren't excluded, but alternatives must be evaluated given our multi-cloud direction. |


# Tasks

1. **Duty workload profiling** (~4 weeks) — Inventory recurring duty engineer chores, estimate time spent on each, and assess which are tractable for agent assistance. Output: recommended focus areas and a baseline for benefit measurement.
2. **PoC and proposal** (~7 weeks) — Build against the recommended focus area; write up the proposal against the evaluation criteria above.
3. **Gen AI for Platform Engineers 101** (~1 week) — Team enablement session.

Total: ~12 weeks.

# Acceptance criteria

- [x] Written inventory of duty engineer chores with time estimates and agent-suitability assessment, presented to the team
- [ ] Enablement material delivered in markdown and presented
- [ ] Working PoC covering at least one