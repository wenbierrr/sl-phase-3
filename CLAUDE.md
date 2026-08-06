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

### The three clear winner points

These are the yardstick a combination is justified against. They shape what gets built and measured — design input, not a write-up step.

| | Winner point |
|---|---|
| **1** | **Prompt, then prepare the deployment** — the AI takes a change all the way to a raised MR; a human approves and syncs. See the deploy flow below. |
| **2** | **Show the blind spots of the dashboards.** Two distinct halves: <br/>**2a** — Red Hat OpenShift dashboards (CPU / memory utilisation). **In scope.**<br/>**2b** — the Grafana dashboard recording **Kafka records arriving into the topic per second**. **KIV** — not in any RFC, don't write it up. |
| **3** | **A less-experienced DE says it was useful** — not a senior DE, not the author. The value is lifting someone who doesn't already know where to look, so experience level gets recorded per pilot case or the criterion is unmeasurable. |

**Don't force the winner points into the architecture.** Same rule that applied to the tool evaluation: if a combination doesn't naturally address a point, that is the finding — record it. A contorted claim that a combination "sort of" satisfies a point is worth less than a clean no.

### The deploy flow — where AI stops

**AI never triggers an Argo sync.** This is settled, not a trade-off to re-weigh.

```
AI     → ask the DE the necessary questions
       → create a branch
       → edit the scaffolding, values files and application files
       → raise the MR, set the reviewer
       ── stops here ──
HUMAN  → approve the MR
       → first time this app is onboarded?  create the App of Apps, then manually sync
       → already onboarded?                 manually Argo sync
```

The AI's entire deploy contribution is a branch, some file edits and an MR. Everything downstream is a person.

`epic.md` holds the formal scope and acceptance criteria. Read it before planning work. Out of scope there: LLM hosting (assume an endpoint is provided) and model selection — don't build hosting infrastructure or benchmark models against each other.

## Platform context

- Kubernetes, with **Argo CD** as the CD tool.
- **Starforge** is the org's platform team. DEs escalate to them for platform-level changes (e.g. Istio configuration).
- Work happens on the **staging cluster**. Istio is in play.
- **~90% of real DE issues are networking.** This is why the pilot's fault catalogue is networking-weighted, and why Istio-layer visibility matters in the MCP server choice.
- **The agent never writes to the cluster, and never triggers an Argo sync.** A standing rule. Write-capable Kubernetes MCP servers exist; their write flags stay off. Everything that lands goes through git, an MR, and a human.

**Reaching the OpenShift dashboard metrics** (winner point 2a) — the facts, so nobody re-researches them:

- Observe → Dashboards in the OCP console reads the **Thanos Querier**.
- Programmatic access is the `thanos-querier` route in `openshift-monitoring`: `oc get routes -n openshift-monitoring thanos-querier`.
- **Bearer token auth only**, via a ServiceAccount bound to the **`cluster-monitoring-view`** ClusterRole. Only `/api` paths are reachable on the route.
- Red Hat advises no more than **one query per 30 seconds** — design any polling around that.
- It speaks the Prometheus query API, so any Prometheus-compatible MCP server works pointed at it.

Two verified facts worth not re-deriving:

- **k8sgpt's MCP server exposes 12 tools and none of them write cluster state** ([MCP.md](https://github.com/k8sgpt-ai/k8sgpt/blob/main/MCP.md)). It also cannot reach metrics.
- **k8sgpt ships no Istio analyzers.** Sub-area 1.1.3 likely needs a separate read-only Istio MCP server.

## The RFC series

**There were once six planned RFCs. There are now two, plus a maybe** — the supervisor's call: all three problem areas ride on the *same* architecture, so splitting by problem would produce documents that each re-argue it and none that carries enough weight on its own. Don't split them back apart.

See [plan.md](plan.md) for the schedule.

| | Question | State |
|---|---|---|
| [rfc-1-ai-tool-evaluation.md](rfc-1-ai-tool-evaluation.md) | Which tool architecture? K8sGPT vs kubectl-ai vs kagent vs LibreChat + an MCP server | **Done.** Verdict: **LibreChat + MCP server** |
| [rfc-2-mcp-server-eval-for-librechat.md](rfc-2-mcp-server-eval-for-librechat.md) | **LibreChat + which combination of MCP servers brings DEs the most value?** Covers troubleshooting (1.1.1–1.1.3), deploy (1.1.4) and the CPU/memory threshold work (2.2), plus the DE rollout | **Active** |
| RFC-X | Production security, RBAC wiring, rollout | **May never be written.** Only if a tool passes its Day-0 thresholds — deliberately not numbered, because this is R&D and not a delivery pipeline |

### How the work actually runs: answers first, RFC last

**Weeks 1–5 find out whether LibreChat + X is useful and in which combination. Weeks 6–8 write it up.** A document written backwards from findings is better than one written forwards from hopes.

So: before week 6, "write the RFC" is *not* the task. The task is running experiments and collecting scorecards. Don't let a future session start drafting conclusions it doesn't have evidence for yet.

### House style for the RFC — read `example-of-rfc/example-rfc.md` first

The supervisor's note: the RFC should be **sharper, carrying only what decision-makers need**. [`example-of-rfc/example-rfc.md`](example-of-rfc/example-rfc.md) (EVAL-01) is the model. It makes a complete adopt/reject case in **99 lines**, almost entirely in tables. The current RFC-2 draft is three times that length and mostly prose — that gap is the work.

**The skeleton:**

| § | What goes in it |
|---|---|
| Metadata table | **Status (the verdict) first**, services evaluated, related docs, created, last reviewed. A decision-maker sees Go/No-Go in the first five lines |
| Change log | Version, date, what changed, author. Each entry says whether the **verdict changed** — that's what tells a returning reader to re-read or not |
| 1. The question this eval answers | **One sentence, answerable yes/no.** Nothing else |
| 2. How the candidate works | Comparison table, one row per aspect, incumbent vs candidate. Bold the candidate's distinctive mechanism. No prose |
| 3. Requirements | `R1…Rn`, each tagged `[Must-Have]` / `[Nice-to-Have]`, each with a **use case column giving the real situation that makes it a requirement** |
| 4. Scorecard | Per requirement: verdict, finding, workaround. The heart of the document |
| 5. Strengths the requirements don't capture | The "good to have" — but each one ends **"Becomes a requirement if/when ⟨condition⟩"** |
| 6. Verdict | The call, a short paragraph of reasoning, **Revisit if:** ⟨condition⟩, **Decision date:** |
| Appendix: Evidence | Requirement ID → link → **what the link confirms**. Not a URL dump |

**The rules that make it sharp:**

- **Fixed scoring vocabulary, defined at the top of §4.** `Met` = confirmed · `Gap` = confirmed shortfall · `Unverified` = untested, **or a `[Must-Have]` with no magnitude** · `N/A`. That last clause matters most for us: *a must-have without a number is Unverified, not Met.*
- **Quantify every finding or mark it Unverified.** The example writes "~2–3 min vs ~5s locally (~25–35x)", "3 of 4 repair primitives", "0 supported hook points". Never "slower" or "limited".
- **Label workarounds `Accepted` / `Not accepted`.** An **Accepted** workaround must state **what would let it be dropped** — otherwise it's an unbounded concession.
- **Say what you didn't test.** Two of the example's eight requirements are `Unverified` and it names exactly what's unchecked. That's what makes the rest credible.
- **Tables carry the argument; prose only in the verdict.**
- **Requirement IDs are traceable** — the same `R3` appears in requirements, scorecard, strengths and evidence, so any claim can be walked back to its source.
- **Everything gets a revisit condition.** The doc is built to be re-read later, not filed.

**One adaptation for our case.** EVAL-01 scores *one* candidate against requirements. We're comparing *several* LibreChat + MCP server combinations, so §4 needs a column per combination rather than a single verdict column. Requirements come from the [three winner points](#the-three-clear-winner-points), tagged Must-Have, with the DE problem sub-areas as their use cases.

Two things that are easy to get wrong:

- **RFC-2 has an order of investigation, not a menu.** kubectl MCP first → then judge whether k8sgpt earns its place on top → then Argo MCP (read-only, for 1.1.1 diagnosis only) → then GitLab MCP → then a Prometheus-compatible MCP server. This is the reverse of how RFC-2's options section currently reads; the document catches up when it's written properly.
- **Argo MCP is not a deploy path.** It exists for read-only 1.1.1 diagnosis. The deploy capability is GitLab MCP raising an MR — see the deploy flow above.

Two things about RFC-1 that are easy to get wrong:

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
2. Incident response
   2.1 Monitoring dashboards                             AI-tractable — KIV
   2.2 Escalating incidents to Starforge                 partly — in scope
```

### 1.1.4 — deploying via Argo, behind an approval gate

Different in kind from 1.1.1–1.1.3: those diagnose a deployment that broke, this one *performs* the deployment. AI drafts the Argo change as an **MR**; a human approves it before anything reaches Red Hat OCP.

The approval gate is the point, not a formality — it is what makes an AI-authored deployment acceptable at all, because a person verifies correctness before it lands.

This is **winner point 1**, and the full flow is in [the deploy flow](#the-deploy-flow--where-ai-stops) above. It is the same mechanism whether the change is routine deployment work or a fix found while troubleshooting — don't build it twice.

### 2.1 — monitoring dashboards · KIV

**What DEs check today**, on a scheduled interval, to detect and report anomalies:

| Dashboard | Shows | |
|---|---|---|
| 4 × OpenShift (hub, stg, prd, prd2) | CPU and memory utilisation per app, and for the whole cluster | → drives **2.2**, in scope |
| Grafana (Thanos + Prometheus + ACM) | **Kafka records arriving into the topic per second** — data flow from SP1 (proxy cluster) into hub, stg, prd and prd2 | **KIV** |

**The Grafana side is winner point 2b and is KIV.** A DE looking at it has no way to know what normal throughput is *at this time of day, on this day of week*, so a baseline derived from past data would reduce false positives and false negatives. **Deliberately not in any RFC — don't write it up, don't scope work against it.** Recorded here only so the opportunity isn't lost.

*If it is ever picked up:* deriving that baseline is statistical work — percentiles, seasonality, variance — not something an LLM does by reading graphs, and months of series will not fit in a context window. Compute the baseline from Thanos with queries, then use AI to explain deviations. Check Thanos retention first.

### 2.2 — escalating to Starforge · in scope

Two halves that differ on whether AI helps:

- **The handoff itself is not AI-tractable.** Deciding to escalate, and the conversation with Starforge, are a human decision and a human conversation.
- **The remediation Starforge then performs is.** On an app's CPU/memory spike: the immediate steps that stop the bleeding, then the persistent fix so the same incident does not recur. Separately, whether CPU/memory could be better allocated across apps — **with justification**, since a reallocation recommendation nobody can defend is unusable.

**Sequencing for 2.2: define X first.** There is a prior step that has to land before any of the remediation work means anything — **determining what X is**, the threshold that separates normal from a spike. Only once X exists can you answer "it has gone above/below X, so here is how to stop the bleeding and here are the remediation steps". Work that comes before tooling.

The spike work sits here rather than under 2.1 because Starforge is the team that stops the bleeding and implements the persistent fix. 2.1 is where the anomaly is *detected*; 2.2 is where it is *acted on*.

This is **winner point 2a**, scoped to the **Red Hat OpenShift console** CPU and memory dashboards. Metrics access is documented under [Platform context](#platform-context).

### Scope

**In scope for RFC-2:** all of 1.1 (1.1.1–1.1.4) and 2.2. One architecture is being tested against all of it — that is the point of a single RFC.

**KIV, not in scope:** 2.1, the Grafana Kafka-throughput side. Don't scope work against it or write it up.

Don't build generic "handles everything" tooling. A combination that convincingly handles Argo sync failures beats one that shallowly touches every sub-area — and a clean "this combination doesn't help here" is a better finding than a stretched claim that it does.

## Current status

Against the epic's acceptance criteria:

- ✅ **Written inventory of DE chores** with time estimates and agent-suitability assessment — done and presented. This drove the focus decision above.
- 🔨 **Enablement material in markdown, presented** — **this means the RFCs, PoC write-ups and research**, not a training session. It is the highest-priority live deliverable and it accumulates continuously; every RFC written is progress against it.
- 🔄 **Working PoC** covering at least one focus area — **continuous.** Concretely this is LibreChat plus the chosen MCP server(s) running on stg and in DEs' hands, exercised against fabricated and real failures. RFC-1's evaluation is complete; 3 stg cases have been run against LibreChat + K8sGPT (2 with root cause found and no hallucination, 1 still open).

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

**Promotion** — the boundary is concrete: **RFC-2's Day-0 thresholds**, agreed by the team *before* any pilot data exists, scored at the end of the pilot week. Passing them means proceeding to RFC-X (production security, RBAC wiring, rollout) — not going to production directly. RFC-X is where the full criteria list gets worked through deliberately and in writing.

Two things about those thresholds: they include a bar for **reaching a raised MR end-to-end** (one clean case), because that is the supervisor's first winner point; and **DE-estimated time saved is deliberately not among them**, because an estimate must not gate a decision.

The gate is a real filter, not a formality. A tool can be impressive in exploration and still fail here — unbounded blast radius, no way to map DE identity into the audit trail, or a maintenance burden the team can't carry are all sufficient reasons to reject it. Fixing the thresholds before the data exists is what stops them being quietly relaxed to fit a result.

Practical consequence: don't let a spike drift toward production without passing the gate, and don't silently apply gate-level engineering to something still being explored. A failed gate that names its elimination factor is a valid answer to the epic's question, not a failed project.

Two criteria are worth watching even during exploration, because they affect whether a finding is trustworthy at all:

- **Resolution quality** — a confidently wrong answer is worse than no answer. Record wrong-but-plausible outputs explicitly; they are the finding, not noise to smooth over.
- **Portability** — multi-cloud is the org's stated direction. OpenShift-native approaches may be explored, but note where a result depends on OpenShift specifics so the decision isn't made on a false assumption of transferability.

## Deliverable formats

Markdown prose documents in the repo — for enablement material, inventories, and the final proposal alike. Not slides, not external docs.
