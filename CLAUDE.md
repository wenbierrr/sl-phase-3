# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

**Mostly documents, plus two vendored upstream repos.** The tracked content is `epic.md`, `plan.md`, the RFC series and `README.md` — no build system or test suite of our own, so no build/lint/test commands to document.

Alongside them sit two upstream projects, cloned in place, each tracked by its own `.git` and excluded via `.gitignore` so `git add .` cannot turn them into gitlinks:

| Directory | Upstream at | Local changes |
|---|---|---|
| `LibreChat/` | `chart-2.0.7-66-g8fcb77fe6` | `helm/librechat/values-sf.yaml`, plus chart edits for an optional `/app/uploads` volume |
| `kubernetes-mcp-server/` | `v0.0.63-31-g9c6ef49` | `charts/kubernetes-mcp-server/values-sf.yaml`, `Dockerfile` patched for a containerd CVE |

**The `-sf.yaml` files and the k8sgpt `serverInstructions` prompt inside them are the PoC's tuned configuration** — the thing weeks 1–3 exist to produce. They currently live only as uncommitted edits inside ignored directories, so a re-clone loses them. Treat them as deliverables, not scratch.

Deployment is by `helm install`/`upgrade` with `-f <chart>/values-sf.yaml`; the subchart `.tgz` files under `LibreChat/helm/librechat/charts/` are vendored deliberately — do not run `helm dependency update`.

Do not infer a stack or framework beyond this. Nothing else has been chosen.

## What this project is

R&D. The goal is to answer a question, not to ship a system: **can AI measurably make duty engineers' (DE) lives better?** The audience for the answer is the supervisor, who uses it to make a go/no-go decision for bringing AI capability into cluster.

Everything here is research, PoCs, and pilot tests in service of that decision. Nothing is intended for production.

This has a direct consequence for how work should be done: **optimise for learning per unit of effort, and for evidence that would change the decision.**

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

**The repo topology.** The org is on GitLab, organised as one **Group** holding multiple **Projects**, each with its own repository. Two of them carry the deploy flow. **GitLab is not installed yet.** These two GitHub repos are the authoritative scaffolding template — [helm-charts](https://github.com/wenbierrr/helm-charts), [argohub](https://github.com/wenbierrr/argohub) — to be recreated as Projects inside the Group once GitLab is up on OpenShift. Treat their layout as the spec, not as a repo to migrate.

| Project | Path | Holds |
|---|---|---|
| **helm-charts** | `charts/<team>/<app>/` | the chart itself — `Chart.yaml`, `values.yaml`, `templates/` |
| **argohub** | `apps/<team>/<app>/base/values.yaml` | values common to every cluster |
| | `apps/<team>/<app>/overlay/<cluster>/values.yaml` | per-cluster values — **this is where `image.tag` lives** |
| | `argocd/application/apps/<team>/<app>/app-<cluster>.yaml` | one Argo `Application` CRD per cluster |

**How the two projects join up — the multi-source `$values` pattern.** Each Application declares two sources: the chart from helm-charts, and argohub as a bare `ref: values` source contributing no manifests. The values files are then addressed through that ref, base first and overlay second, so the overlay wins:

```yaml
sources:
  - repoURL: <helm-charts>
    path: charts/demo-team/hello-app
    targetRevision: main
    helm:
      valueFiles:
        - $values/apps/demo-team/hello-app/base/values.yaml
        - $values/apps/demo-team/hello-app/overlay/demo-stg/values.yaml
  - repoURL: <argohub>
    targetRevision: main
    ref: values
```

Two consequences that decide what an AI has to edit. **The chart is consumed from a git path, not a chart version** — `targetRevision: main` means a chart change lands the moment its MR merges, with no version to bump. And **cluster is encoded in the filename, not the directory** (`app-demo-stg.yaml`, `app-demo-prd.yaml`), which is what makes the App of Apps shape matter below.

**The environment is airgapped.** Charts are dragged and dropped in; nothing pulls from an external chart repo. Assume the same of everything else the PoC needs — MCP server images, chart dependencies, the inference endpoint. Nothing resolves from the public internet. (Mirroring the demo app's own container image is *not* a PoC concern — don't spend effort there.)

Two cases, and the distinction matters because only one is cheap enough to demonstrate end to end.

**New app onboarding** — spans both projects, so two MRs:

```
AI     → ask the DE the necessary questions
       → helm-charts:  branch → add the chart under charts/<team>/<app>
                              → MR to main
       → argohub:      branch → apps/<team>/<app>/base/values.yaml
                              → apps/<team>/<app>/overlay/<cluster>/values.yaml
                              → argocd/application/apps/<team>/<app>/app-<cluster>.yaml
                                (one per target cluster)
                              → MR to main
       ── stops here ──
HUMAN  → approve both MRs
       → create the App of Apps by hand, then manually sync
```

**The App of Apps is always created by a human, never by the AI, and it does not live in argohub.** The AI's contribution stops at the Application CRDs that an AOA later sweeps up.

Its shape is a real choice, and the layout is not neutral between them. **One AOA per app across all clusters** falls out naturally — point it at `argocd/application/apps/<team>/<app>/` and it picks up every `app-<cluster>.yaml` beneath. **One AOA per cluster** cannot be expressed as a directory at all, because cluster lives in the filename; it needs a `directory.include` glob such as `app-*-stg.yaml`. Establish which shape is in use before designing anything against it.

**Existing app change** — the common case, and it touches **one project, one file**:

```
DE     → a team asks to bump a container image version, or change the chart
AI     → ask which, and to what version — the DE may give the value, or
         choose to edit the file themselves
       → image bump  → argohub:     apps/<team>/<app>/overlay/<cluster>/values.yaml
         chart change → helm-charts: charts/<team>/<app>/
       → raise the MR
       ── stops here ──
HUMAN  → approve the MR
       → Argo GUI → DIFF → manually sync
```

An image bump never touches helm-charts, and a chart change never touches argohub. Neither needs the cross-project coordination that new-app onboarding does.

**The DIFF-and-sync step is where 1.1.1 surfaces.** A faulty chart, or Argo objecting to something else, throws its error at exactly this moment. So the deploy flow (1.1.4) and *workload cannot be synced* (1.1.1) are one workflow observed at two moments, not two separate problems — a single injected fault can exercise both.

The AI's entire deploy contribution is a branch, some file edits and an MR. Everything downstream is a person.

`epic.md` holds the formal scope and acceptance criteria. Read it before planning work. Out of scope there: LLM hosting (assume an endpoint is provided) and model selection — don't build hosting infrastructure or benchmark models against each other.

## Platform context

- Kubernetes, with **Argo CD** as the CD tool.
- **Starforge** is the org's platform team. DEs escalate to them for platform-level changes (e.g. Istio configuration).
- Istio is in play.

**Cluster topology.**

| Cluster | Runs |
|---|---|
| **hub** | **GitLab and Argo CD** |
| **stg, prd, prd2** | the workloads — the manifests the Helm charts render. **Istio troubleshooting is mainly needed here**, not on hub |

**LibreChat and the MCP servers are portable — they can be deployed in any cluster.** They are not hub-resident, and nothing about the design pins them there. That is worth stating plainly because portability is one of the epic's evaluation criteria, and this is a point in the architecture's favour.

Placement follows from what each server talks to, not from where the tooling "belongs":

- **Cluster-scoped servers** — Kubernetes, k8sgpt, and an Istio server if one is adopted — read the cluster they run in, through an in-cluster ServiceAccount. To troubleshoot stg, run one in stg. Since Istio work is concentrated in stg and prd, that is where an Istio server goes.
- **API-client servers** — Argo and GitLab — hold a token and talk to hub over the network. They run anywhere with a route to hub; where they sit is irrelevant.

**The PoC runs on one cluster.** There aren't the resources for separate stg and prd, so everything collapses in-cluster — which is what the reference `Application` CRDs already do with `destination.server: https://kubernetes.default.svc`. Keep the `overlay/<cluster>/` directory shape anyway, because it is the org's real structure; just expect one overlay in the PoC.

That single cluster models the co-located case honestly: an MCP server sitting alongside the workloads it reads is exactly the arrangement stg and prd would use. **The one thing it leaves untested is fan-out** — one LibreChat with a Kubernetes MCP server registered for stg *and* another for prd, where the LLM has two near-identical tool surfaces and has to pick the right cluster. That is a tool-selection question, and it belongs in the tool-selection row of RFC-2's matrix rather than being treated as a reach problem.
- **~90% of real DE issues are networking.** This is why the pilot's fault catalogue is networking-weighted, and why Istio-layer visibility matters in the MCP server choice.
- **The agent never writes to the cluster, and never triggers an Argo sync.** A standing rule. Write-capable Kubernetes MCP servers exist; their write flags stay off. Everything that lands goes through git, an MR, and a human.
- **GitLab, Argo CD and Istio are not yet in the PoC environment.** LibreChat, k8sgpt and the Kubernetes MCP server are up; the other three have to be stood up on OpenShift before 1.1.1, 1.1.3 and 1.1.4 can be exercised at all. They gate three of the five sub-areas and both of the in-scope winner points that are not metrics — treat standing them up as experiment work, not setup overhead.
- **Check the GitLab version before committing to the deploy path.** The official GitLab MCP server ships in-product from **18.5**. On an older airgapped instance winner point 1 has no supported server behind it, which changes the plan rather than the schedule — find out early.

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

### House style for the RFCs

**RFC-1 is the template. RFC-2 mirrors it.** Structure is: ToC → Motivation → Scope → Brief overview table → Options considered → Comparison matrix. Keep new RFCs in that shape.

- **Motivation: two lines, then the question.** A short statement of the problem, then a separate paragraph opening *"The question this RFC answers: …"* which also names what follows in the next RFC. Both RFCs do this — match it.
- **Brief overview table** compares candidates on the same rows: why it was built, maturity, what it adds, what is lost.
- **Options are numbered** (`Option 1 — …`), and the numbers carry into the comparison-matrix column headers and the verdict list so a reader can walk between them.
- **Comparison matrix** splits each decision driver into **Information** (the fact) and **Evaluation metric** (the judgement drawn from it). Cells stay **blank until tested** — a plausible guess defeats the point.
- **Verdict is a numbered list with a named elimination factor per option**, in the same order as the options.

**`example-of-rfc/example-rfc.md` is a source of craft, not a skeleton.** It was tried as a template and rejected — don't restructure the RFCs around its metadata block, change log, `R1…Rn` requirements or Met/Gap/Unverified scorecard. What *is* worth stealing is how it stays sharp in 99 lines:

- **Quantify or say untested.** It writes "~2–3 min vs ~5s locally (~25–35x)", "3 of 4 repair primitives", "0 supported hook points" — never "slower" or "limited".
- **Name what you didn't test.** Explicitly marking the gaps is what makes the rest credible.
- **Tables carry the argument; prose only in the verdict.**
- **Everything gets a revisit condition** — "becomes a requirement if…", "re-check on first tagged release". The document is built to be re-read, not filed.
- **A workaround you accept must state what would let it be dropped**, or it's an unbounded concession.

Two things that are easy to get wrong:

- **RFC-2 has an order of investigation, not a menu.** kubectl MCP first → then judge whether k8sgpt earns its place on top → then Argo MCP (read-only, for 1.1.1 diagnosis only) → then GitLab MCP → then a Prometheus-compatible MCP server.
- **Argo MCP is not a deploy path.** It exists for read-only 1.1.1 diagnosis. The deploy capability is GitLab MCP raising an MR — see the deploy flow above.

Two things about RFC-1 that are easy to get wrong:

- **RFC-1 settled the orchestrator, not the MCP server.** Its scoring paired LibreChat with k8sgpt as a reference implementation, and those cells are marked *(k8sgpt ref.)*. Don't read them as a decision that k8sgpt is the MCP server — that is exactly what RFC-2 is for.
- **RFC-1's appendix preserves the kagent Collector/Diagnostician design, but kagent was eliminated.** The appendix is kept just to show that we did our research.

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

**Go at the existing-app case first.** It is one file in one project, and it is what DEs actually do most days; new-app onboarding is four file locations across two projects and two MRs. The pass bar is *one clean case reaching a raised MR* — the existing-app path clears it for a fraction of the build, and the new-app path can be attempted afterwards with nothing riding on it.

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
