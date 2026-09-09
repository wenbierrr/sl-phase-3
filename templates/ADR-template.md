# ADR-NN: <Decision stated as an imperative — e.g. "Adopt Kyverno as the Kubernetes Policy Engine">

| **Metadata**       | **Value**                                          |
|--------------------|-----------------------------------------------------|
| **Status**         | <Proposed / Approved / Rejected / Superseded by ADR-NN — pick one, delete the rest> |
| **Authors**        | @name1, @name2                                      |
| **Decision Date**  | YYYY-MM-DD                                          |
| **Source**         | <Link to the RFC / evaluation / knowledge doc holding the options analysis, if one exists> |

---

**One page.** An ADR records what was decided and why, *at the time*. When it stops being true, supersede it with a new ADR — don't edit this one after Decision Date (that's what a **Last Updated** field would invite, which is why this template has none). Options analysis stays in the Source doc — link it, don't inline it.

Not sure this belongs in `adr/`? → [documentation-guide.md §3.3](../../team-guidelines/documentation-guide.md#33-docsadr)

## Context

<≈5 sentences. The forces and constraints that were true at decision time — what problem needed solving, what options existed, what mattered when weighing them. Link the Source doc for the full comparison; don't reproduce it here.>

## Decision

<One bold sentence stating the decision, then at most five bullets on why.>

Good in-repo example:
> **Adopt Kyverno as the policy engine.**
>
> - Policies are defined as Kubernetes Custom Resources — no separate language to learn.
> - A large library of community-maintained policies lets us adapt rather than build from scratch.

## Consequences

<≈12 bullets total across the three subsections below.>

### Positive

<What gets better as a direct result of this decision.>

### Trade-offs we accept

<Costs we are knowingly taking on — not hypothetical risks, but things that are now true and won't change unless this ADR is superseded.>

### What this triggers

<Follow-on work this decision creates — a PoC, a baseline policy set, a new 101 doc, etc.>