# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

**This is a documents repo, not a code repo.** It holds `epic.md`, the RFC series, and `README.md` — no source code, dependency manifest, build system, or test suite, and therefore no build/lint/test commands to document. Only `README.md` is tracked in git; everything else is currently untracked, with no `.gitignore`.

The PoC work itself (LibreChat and MCP server deployments) lives on the staging cluster as Helm releases, not in this repo. If artifacts do land here — a values file, a patched Dockerfile, an agent prompt — add the commands to this section then.

Do not infer a stack, framework, or directory layout from this file. Nothing has been chosen.

## What this project is

R&D. The goal is to answer a question, not to ship a system: **can AI measurably make duty engineers' (DE) lives better?** The audience for the answer is the supervisor, who uses it to make a go/no-go decision for bringing AI capability into cluster.

Everything here is research, PoCs, and pilot tests in service of that decision. Nothing is intended for production.

This has a direct consequence for how work should be done: **optimise for learning per unit of effort, and for evidence that would change the decision.** Prefer the throwaway spike that answers the question this week over the durable implementation that answers it next month. Hardcoding, manual steps, narrow happy paths, and scripts that only work on one cluster are all acceptable when they buy a faster answer — say so in the write-up rather than engineering them away.

**Quantify what is useful; for everything rejected, name the elimination factor.** A rejection with a specific named reason carries the same decision weight as the winner — it tells the supervisor what was ruled out and why, so the same ground isn't re-covered later. RFC-1's verdict list is the template:

> - **K8sGPT alone** — no conversational follow-up
> - **kubectl-ai** — no GUI
> - **kagent** — adopting it means standing up an entire agentic platform inside the cluster (CRDs, controller, UI, Postgres) to install, run and keep upgrading; the autonomy it buys does not justify that permanent cost

"Didn't work out" is not an elimination factor. "We'd be running a whole agentic platform in-cluster forever, and the autonomy isn't worth it" is.

### What would convince the decision-makers

Two outcomes count as a clear win. They shape what gets built and measured — treat them as design input, not as a write-up step:

1. **It can prompt, then deploy** — diagnosis through to the fix landing, driven from the chat. **Via Argo sync or an MR only** — the agent never writes to the cluster (see Platform context). The MR route is preferred: no cluster access at all, and its blast radius is a merge request someone has to approve.
2. **A less-experienced DE says it was useful** — not a senior DE, not the author. The value proposition is lifting someone who doesn't already know where to look, so experience level gets recorded per pilot case or the criterion is unmeasurable.

A third candidate — **"show the blind spots of the dashboards"** — was considered and **routed elsewhere, not dropped.** It turned out to be two distinct problems, now mapped as area 2.1 and 2.2 with an RFC each (RFC-5 and RFC-6). They need Prometheus/Thanos metrics rather than cluster object state, so they don't belong to the current track. Don't fold them back into RFC-2.

`epic.md` holds the formal scope and acceptance criteria. Read it before planning work. Out of scope there: LLM hosting (assume an endpoint is provided) and model selection — don't build hosting infrastructure or benchmark models against each other.

## Platform context

- Kubernetes, with **Argo CD** as the CD tool.
- **Starforge** is the org's platform team. DEs escalate to them for platform-level changes (e.g. Istio configuration).
- Work happens on the **staging cluster**. Istio is in play.
- **~90% of real DE issues are networking.** This is why the pilot's fault catalogue is networking-weighted, and why Istio-layer visibility matters in the MCP server choice.
- **The agent never writes to the cluster.** A standing rule, not a trade-off to re-weigh. Write-capable Kubernetes MCP servers exist; their write flags stay off. Anything that lands goes through git and Argo CD.

Two verified facts worth not re-deriving:

- **k8sgpt's MCP server exposes 12 tools and none of them write cluster state** ([MCP.md](https://github.com/k8sgpt-ai/k8sgpt/blob/main/MCP.md)). Any deploy capability must come from the Argo MCP server or an MR — not from k8sgpt, and not by enabling writes on a Kubernetes MCP server.
- **k8sgpt ships no Istio analyzers.** Sub-area 1.1.3 likely needs a separate read-only Istio MCP server.

## The RFC series

Each RFC answers one question and hands the next one on. Read the relevant RFC before starting work — they carry the decisions and the evidence.

**The active track — D-2-D troubleshooting (area 1.1):**

| | Question | State |
|---|---|---|
| [rfc-1-ai-tool-evaluation.md](rfc-1-ai-tool-evaluation.md) | Which tool architecture? K8sGPT vs kubectl-ai vs kagent vs LibreChat + an MCP server | **Done.** Verdict: **LibreChat + MCP server** — web GUI, persisted multi-user history, no disqualifying gap |
| [rfc-2-mcp-server-evaluation.md](rfc-2-mcp-server-evaluation.md) | Which MCP server backs LibreChat? k8sgpt / kubernetes / both / + Argo. Then hands-on runs on stg to tune it, then a DE pilot | **Active** |
| RFC-3 (not written) | Production security, RBAC wiring, rollout | Only if the pilot passes its Day-0 thresholds |

**Planned — one RFC per problem. These are independent of RFC-3, not sequential after it** (numbering is provisional):

| | Question | Area |
|---|---|---|
| RFC-4 (not written) | Deploying to the clusters via Argo by raising an MR, behind a human approval gate | 1.1.4 |
| RFC-5 (not written) | A ~90-day traffic baseline, to reduce the occurrence of DE false positives and false negatives | 2.1 |
| RFC-6 (not written) | On an app's CPU/memory spike: stop the bleeding, persistent fix, and a justified allocation review | 2.2 |

RFC-5 and RFC-6 are separate because they solve different problems for different consumers — one makes sense of past data for the DE, the other tells Starforge what to do about a live spike.

**RFC-1's shortlist does not serve RFC-5 or RFC-6.** k8sgpt and the Kubernetes MCP servers read cluster object state, not Prometheus/Thanos metrics, so those RFCs start from a fresh tool evaluation. [HolmesGPT](https://github.com/HolmesGPT/holmesgpt) — alert-driven investigation over Prometheus/Grafana/AlertManager — is the obvious first candidate to look at.

Two things about the active track that are easy to get wrong:

- **RFC-1 settled the orchestrator, not the MCP server.** Its scoring paired LibreChat with k8sgpt as a reference implementation, and those cells are marked *(k8sgpt ref.)*. Don't read them as a decision that k8sgpt is the MCP server — that is exactly what RFC-2 is for.
- **RFC-1's appendix preserves the kagent Collector/Diagnostician design, but kagent was eliminated.** The appendix is kept because the evidence-bundle schema, success criteria, and CrashLoopBackOff test-case table remain useful reference. It is not a live plan. Do not resurrect it as the architecture; if something in it is worth reusing, port the idea into the LibreChat + MCP design deliberately.

**Terminology:** LibreChat is the **orchestrator**; what sits behind it is the **MCP server** — never "the backend". ("MCP client" is still correct where the protocol role is what's meant, e.g. describing LibreChat's MCP support.) The org is on GitLab: say **MR**, not PR.

## The DE problem space

**The duty engineer changes every day.** Nobody accumulates context across a rotation — whoever is on duty meets today's problem cold. That is a large part of why an assistant might help at all, and it is why the pilot measures whether a *less-experienced* DE finds the tool useful rather than whether an expert does.

The team profiled DE work and split it into two areas. Every leaf is marked for whether AI can help with it:

```
1. DE troubleshooting complexities                       ← current focus
   1.1 D-2-D deployment                                  ← active target
       1.1.1 Workload cannot be synced on Argo           AI-tractable
       1.1.2 Workload synced but pod down /              AI-tractable
             app not working as intended
       1.1.3 Additional configs Starforge must make      AI-tractable
             (e.g. Istio)
       1.1.4 Deploying to the clusters via Argo, by      AI-tractable
             raising an MR behind a human approval gate
2. Incident response                                     ← mapped, not active
   2.1 Monitoring dashboards                             AI-tractable
   2.2 Escalating incidents to Starforge                 partly — see below
```

### 1.1.4 — deploying via Argo, behind an approval gate

Different in kind from 1.1.1–1.1.3: those diagnose a deployment that broke, this one *performs* the deployment. AI drafts the Argo change as an **MR**; a human approves it before anything reaches Red Hat OCP.

The approval gate is the point, not a formality — it is what makes an AI-authored deployment acceptable at all, because a person verifies correctness before it lands.

Note the overlap with RFC-2's "prompt then deploy" criterion: **same MR-plus-gate mechanism, different job.** RFC-2 deploys a fix found while troubleshooting; this is routine deployment work. Don't build the mechanism twice.

### 2.1 — monitoring dashboards

**What DEs check today**, on a scheduled interval, to detect and report anomalies:

| Dashboard | Shows |
|---|---|
| 4 × OpenShift (hub, stg, prd, prd2) | CPU and memory utilisation per app, and for the whole cluster |
| Grafana (Thanos + Prometheus + ACM) | Data flow from SP1 (proxy cluster) into hub, stg, prd and prd2 |

**AI opportunity — the data-driven side.** A DE looking at Grafana has no way to know what normal traffic looks like *at this time of day, on this day of week*. Making sense of ~90 days of past traffic and deriving a baseline from it would **reduce the occurrence of false positives and false negatives**.

**One design constraint, because the obvious approach fails.** Deriving that baseline is statistical work — percentiles, time-of-day and day-of-week seasonality, variance — not something an LLM does by reading graphs, and 90 days of series will not fit in a context window. The workable shape is: compute baselines from Thanos with queries and statistics, then use AI to explain deviations and describe "normal" in plain language. **Check first that Thanos retention actually holds 90 days** before designing against it.

### 2.2 — escalating to Starforge

Two halves that differ on whether AI helps:

- **The handoff itself is not AI-tractable.** Deciding to escalate, and the conversation with Starforge, are a human decision and a human conversation.
- **The remediation Starforge then performs is.** On an app's CPU/memory spike: the immediate steps that stop the bleeding, then the persistent fix so the same incident does not recur. Separately, whether CPU/memory could be better allocated across apps — **with justification**, since a reallocation recommendation nobody can defend is unusable.

The spike work sits here rather than under 2.1 because Starforge is the team that stops the bleeding and implements the persistent fix. 2.1 is where the anomaly is *detected*; 2.2 is where it is *acted on*.

### Scope

Area 1 was chosen over area 2 by team decision, and within it **D-2-D deployment is the active target**. Scope PoC work to 1.1 unless told otherwise.

Area 2 is mapped, not off-limits to think about — the detail above exists so the opportunities are on record. But it isn't current work, and it needs different tooling from 1.1 (see the RFC series). Don't build generic "handles everything" tooling either; a PoC that convincingly handles Argo sync failures beats one that shallowly touches all seven sub-areas.

## Current status

Against the epic's acceptance criteria:

- ✅ **Written inventory of DE chores** with time estimates and agent-suitability assessment — done and presented. This drove the focus decision above.
- 🔨 **Enablement material in markdown, presented** — **this means the RFCs, PoC write-ups and research**, not a training session. It is the highest-priority live deliverable and it accumulates continuously; every RFC written is progress against it.
- 🔨 **Working PoC** covering at least one focus area — conditional on a tool proving promising. RFC-1's evaluation is complete; 3 stg cases have been run against LibreChat + K8sGPT (2 with root cause found and no hallucination, 1 still open). RFC-2 and the DE pilot are the remaining work.

**Don't confuse the second criterion with "Gen AI for Platform Engineers 101".** That session is epic task 3 and is currently parked. The enablement *material* is the written body of work in this repo.

The ordering matters: **criterion 2 outranks criterion 3**, and criterion 3 is conditional. If the PoC stalls, the RFCs still land and the deliverable is still met — so don't sacrifice the written work to keep a PoC moving.

See [plan.md](plan.md) for the 8-week schedule. When the active work changes, update this section and plan.md together.

## Measuring benefit — there is no historical baseline

`epic.md` says benefit is measured against a baseline established during profiling. **That baseline does not exist for D-2-D deployment troubleshooting.** There is no past data on how long these troubleshooting sessions actually took.

So: never present a before/after time saving as though it were measured, and don't back-fill a plausible-sounding "usually takes N hours" figure to compare against. A fabricated baseline would make the whole benefit case unsound, and it's exactly the kind of thing that survives into a decision memo unchallenged.

Workable substitutes, roughly in order of strength:

- **Capture the baseline prospectively.** During a pilot, run a real D-2-D failure both ways — DE unaided, and DE with the agent — and time both. Small n is fine here; this is R&D, not a study. Say what n was.
- **Count outcomes instead of minutes.** Did the agent identify the actual root cause? Would it have avoided an escalation to Starforge? These are countable without any historical data and map directly onto what the supervisor cares about.
- **Structured DE judgement.** Have DEs who ran the pilot say whether it helped and where. Weak evidence on its own, honest when labelled as such.

Whichever is used, state the method and its limits alongside the result.

**This bites concretely on RFC-2's pilot scorecard.** Two of its fields sit on opposite sides of the line:

- *"Roughly how long without the tool?"* — a **DE estimate**. It must stay labelled that way everywhere it is reported or aggregated, and it is deliberately excluded from the pass bars: an estimate must not gate a decision. The failure mode is quiet — an estimate that hardens into a measured figure somewhere between the scorecard and the decision memo.
- *"Without the tool, would this have been escalated?"* — needs no baseline, is countable, and maps straight onto the toil the epic set out to reduce. Lead with this one.

**The stg runs are not evidence — the pilot is.** Keep these two apart; conflating them is the easiest way to overstate the case.

- **Hands-on runs on stg** build a baseline feel for how the tool behaves and produce a tuned configuration. Few cases, no baseline, and whoever runs them already knows the root cause — so the session gets steered in ways a real DE's would not be. Useful for filling the comparison matrix and deciding which MCP servers are needed. **Never cite them as evidence of value.**
- **The pilot** puts the tuned tool in front of DEs who don't know the answer, and scores it against bars agreed in advance. That is what the decision rests on.

**The pilot forces its own data points.** Rather than waiting for organic stg failures — a one-week window won't reliably produce enough, and the mix would be whatever chance supplies — faults are injected from a networking-weighted catalogue, with the answer key sealed and the injector separate from the scorer. Two consequences to hold onto:

- **Percentages need n.** Fix the sample size before agreeing any percentage bar; at n=3 a percentage is noise. RFC-2 proposes n ≥ 8, reported as counts alongside percentages.
- **Injected faults are cleaner than real ones**, and the injector knows what the tool can see. RFC-2 states this and the other limits explicitly. Don't let the pilot get described as proof.

## The evaluation criteria are a promotion gate

`epic.md` lists criteria covering blast radius, security, auditability, cost, risk, maintenance burden, and portability. These are **a gate, not a running constraint.** There are two modes of work, and the criteria bind in only one of them:

**Exploration** — the RFC-1 and RFC-2 work: trying tools out, spiking, running stg cases, seeing what an agent can do on a D-2-D deployment failure. The criteria do not apply. Don't build RBAC, audit trails, or platform abstraction here. Hack freely against stg.

**Promotion** — the boundary is concrete: **RFC-2's Day-0 thresholds**, agreed by the team *before* any pilot data exists, scored at the end of the pilot week. Passing them means proceeding to RFC-3 (production security, RBAC wiring, rollout) — not going to production directly. RFC-3 is where the full criteria list gets worked through deliberately and in writing.

Two things about those thresholds: they include a bar for reaching a **deployable fix end-to-end** (one clean Argo sync or MR), because that is the supervisor's first winner criterion; and **DE-estimated time saved is deliberately not among them**, because an estimate must not gate a decision.

The gate is a real filter, not a formality. A tool can be impressive in exploration and still fail here — unbounded blast radius, no way to map DE identity into the audit trail, or a maintenance burden the team can't carry are all sufficient reasons to reject it. Fixing the thresholds before the data exists is what stops them being quietly relaxed to fit a result.

Practical consequence: don't let a spike drift toward production without passing the gate, and don't silently apply gate-level engineering to something still being explored. A failed gate that names its elimination factor is a valid answer to the epic's question, not a failed project.

Two criteria are worth watching even during exploration, because they affect whether a finding is trustworthy at all:

- **Resolution quality** — a confidently wrong answer is worse than no answer. Record wrong-but-plausible outputs explicitly; they are the finding, not noise to smooth over.
- **Portability** — multi-cloud is the org's stated direction. OpenShift-native approaches may be explored, but note where a result depends on OpenShift specifics so the decision isn't made on a false assumption of transferability.

## Deliverable formats

Markdown prose documents in the repo — for enablement material, inventories, and the final proposal alike. Not slides, not external docs.
