# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Reading order = the document's flow:** what this repo holds → what the project is → the problems being solved → the architecture and its rules → what is running where → where the work stands → how findings become a decision → the RFCs that carry it all.

## Repository state

**Mostly documents, plus three vendored upstream repos.** The tracked content is `epic.md`, `plan.md`, `DE-trial-execution.md`, the RFC series and `README.md` — no build system or test suite of our own, so no build/lint/test commands to document.

Alongside them sit three upstream projects, cloned in place, each tracked by its own `.git` and excluded via `.gitignore` so `git add .` cannot turn them into gitlinks:

| Directory | Upstream at | Local changes |
|---|---|---|
| `LibreChat/` | `chart-2.0.7-66-g8fcb77fe6` | `helm/librechat/values-sf.yaml`, plus chart edits for an optional `/app/uploads` volume |
| `kubernetes-mcp-server/` | `v0.0.63-31-g9c6ef49` | `charts/kubernetes-mcp-server/values-sf.yaml`, `Dockerfile` patched for a containerd CVE |
| `gitlab-mcp/` | `zereight/gitlab-mcp` `v2.1.46-3-g926d42c` | none yet — cloned 14 Aug 2026, untested |

**The `-sf.yaml` files and the `serverInstructions` prompts inside them are the PoC's tuned configuration** — the thing weeks 1–2 exist to produce. They currently live only as uncommitted edits inside ignored directories, so a re-clone loses them. Treat them as deliverables, not scratch.

`miscellaneous/` holds environment deliverables: the Istio-on-CRC install guide and `gitlab-ce.yaml` — the single-container GitLab manifest the PoC runs on (see [PoC environment status](#poc-environment-status)).

`test-cases/` holds the RCA benchmark — 9 fault charts and their answer keys: `README.md` covers tc1–tc4 (core Kubernetes), `README-istio.md` covers tc5–tc8 (Istio, including a three-namespace case), `README-argo.md` covers tc9 (Argo CD, delivered through GitLab rather than `helm install`). **The answer keys must stay off the cluster.** tc1–tc4's chart names name their own root cause, which a Secret read recovers; tc5–tc8 are deliberately uninformative for that reason. **tc9 goes further: its chart source is read by the GitLab MCP arm, so the chart itself carries no explanatory comment and no values key naming the remedy** — see its chart-hygiene rules before editing it.

**`test-cases/` and `miscellaneous/` are gitignored but are *not* git repos.** Unlike the vendored projects they have no `.git` of their own, so the 9 fault charts, every answer key, `gitlab-ce.yaml` and the Istio guide are versioned nowhere. A `git init` in each closes it.

Deployment is by `helm install`/`upgrade` with `-f <chart>/values-sf.yaml`; the subchart `.tgz` files under `LibreChat/helm/librechat/charts/` are vendored deliberately — do not run `helm dependency update`.

Do not infer a stack or framework beyond this. Nothing else has been chosen.

## What this project is

R&D. The goal is to answer a question, not to ship a system: **can AI measurably make duty engineers' (DE) lives better?** The audience for the answer is the supervisor, who uses it to make a go/no-go decision for bringing AI capability into cluster.

Everything here is research, PoCs, and pilot tests in service of that decision. Nothing is intended for production. Direct consequence: **optimise for learning per unit of effort, and for evidence that would change the decision.**

`epic.md` holds the formal scope and acceptance criteria — read it before planning work. Out of scope there: LLM hosting (assume an endpoint is provided) and model selection.

**Quantify what is useful; for everything rejected, name the elimination factor.** A rejection with a specific named reason carries the same decision weight as the winner — it tells the supervisor what was ruled out and why, so the same ground isn't re-covered later. RFC-1's verdict list is the template:

> - **K8sGPT alone** — no conversational follow-up
> - **kubectl-ai** — no GUI
> - **kagent** — adopting it means standing up an entire agentic platform inside the cluster (CRDs, controller, UI, Postgres) to install, run and keep upgrading; the autonomy it buys does not justify that permanent cost

"Didn't work out" is not an elimination factor. "We'd be running a whole agentic platform in-cluster forever, and the autonomy isn't worth it" is.

### The three clear winner points

These are the yardstick a combination is justified against. They shape what gets built and measured — design input, not a write-up step.

| | Winner point |
|---|---|
| **1** | **Prompt, then prepare the deployment** — the AI drafts the exact change; a human commits it, raises the MR, approves and syncs. Scored on whether the drafted change is correct and directly usable — the AI does not reach a raised MR itself (git writes ruled out 11 Aug 2026; see [the standing rules](#standing-rules--the-ai-is-read-only-everywhere)). |
| **2** | **Show the blind spots of the dashboards.** Two distinct halves: <br/>**2a** — Red Hat OpenShift dashboards (CPU / memory utilisation). **In scope.**<br/>**2b** — the Grafana dashboard recording **Kafka records arriving into the topic per second**. **KIV** — not in any RFC, don't write it up. |
| **3** | **A less-experienced DE says it was useful** — not a senior DE, not the author. The value is lifting someone who doesn't already know where to look, so experience level gets recorded per pilot case or the criterion is unmeasurable. |

**Don't force the winner points into the architecture.** If a combination doesn't naturally address a point, that is the finding — record it. A contorted claim that a combination "sort of" satisfies a point is worth less than a clean no.

## The DE problem space

**The duty engineer changes every day.** Nobody accumulates context across a rotation — whoever is on duty meets today's problem cold. That is a large part of why an assistant might help at all, and it is why the pilot measures whether a *less-experienced* DE finds the tool useful rather than whether an expert does.

**~90% of real DE issues are networking.** This is why the pilot's fault catalogue is networking-weighted, and why Istio-layer visibility matters in the MCP server choice.

The team profiled DE work and split it into two areas. Every leaf is marked for whether AI can help with it:

```
1. DE troubleshooting complexities                       ← current focus
   1.1 D-2-D deployment                                  ← active target
       1.1.1 Workload cannot be synced on Argo           AI-tractable
       1.1.2 Workload synced but pod down /              AI-tractable
             app not working as intended
       1.1.3 Additional configs Starforge must make      AI-tractable
             (e.g. Istio)
       1.1.4 Deploying to the clusters via Argo — AI      partly — AI drafts,
             drafts the change, a human commits it,       human commits
             raises the MR, approves and syncs
2. Incident response
   2.1 Monitoring dashboards                             AI-tractable — KIV
   2.2 Escalating incidents to Starforge                 partly — in scope
```

### 1.1.4 — deploying via Argo, with the AI stopping at the draft

Different in kind from 1.1.1–1.1.3: those diagnose a deployment that broke, this one *prepares* a deployment. The AI drafts; a human commits, raises the MR and approves before anything reaches Red Hat OCP — the full mechanics are in [the deploy flow](#the-deploy-flow--where-ai-stops).

**1.1.4 is only partly AI-tractable.** The judgement — which file, which field, what value, correct YAML — is the AI's; the mechanics are the DE's. Don't present it as end-to-end automation.

This is **winner point 1**. It is the same mechanism whether the change is routine deployment work or a fix found while troubleshooting — don't build it twice.

**Go at the existing-app case first.** It is one file in one project, and it is what DEs actually do most days; new-app onboarding is four file locations across two projects. The pass bar is *one clean case where the AI names the right file and produces a change the DE applies unmodified* — the existing-app path clears it for a fraction of the build.

**"Unmodified" is what makes this measurable.** The only honest test of the drafted change is whether the DE had to correct it. Record, per case: did it name the right file · was the diff applied as-is, edited, or discarded · did the resulting MR pass review. A change the DE had to rewrite is a fail, however plausible it looked.

### 2.1 — monitoring dashboards · KIV

**What DEs check today**, on a scheduled interval, to detect and report anomalies:

| Dashboard | Shows | |
|---|---|---|
| 4 × OpenShift (hub, stg, prd, prd2) | CPU and memory utilisation per app, and for the whole cluster | → drives **2.2**, in scope |
| Grafana (Thanos + Prometheus + ACM) | **Kafka records arriving into the topic per second** — data flow from SP1 (proxy cluster) into hub, stg, prd and prd2 | **KIV** |

**The Grafana side is winner point 2b and is KIV.** A DE looking at it has no way to know what normal throughput is *at this time of day, on this day of week*, so a baseline derived from past data would reduce false positives and negatives. **Deliberately not in any RFC — don't write it up, don't scope work against it.** Recorded here only so the opportunity isn't lost.

*If it is ever picked up:* deriving that baseline is statistical work — percentiles, seasonality, variance — not something an LLM does by reading graphs, and months of series will not fit in a context window. Compute the baseline from Thanos with queries, then use AI to explain deviations. Check Thanos retention first.

### 2.2 — escalating to Starforge · in scope

Two halves that differ on whether AI helps:

- **The handoff itself is not AI-tractable.** Deciding to escalate, and the conversation with Starforge, are a human decision and a human conversation.
- **The remediation Starforge then performs is.** On an app's CPU/memory spike: the immediate steps that stop the bleeding, then the persistent fix so the same incident does not recur. Separately, whether CPU/memory could be better allocated across apps — **with justification**, since a reallocation recommendation nobody can defend is unusable.

**Sequencing for 2.2: define X first** — the threshold that separates normal from a spike. Only once X exists can you answer "it has gone above X, so here is how to stop the bleeding and here are the remediation steps". Work that comes before tooling.

The spike work sits here rather than under 2.1 because Starforge is the team that stops the bleeding and implements the persistent fix. 2.1 is where the anomaly is *detected*; 2.2 is where it is *acted on*. This is **winner point 2a**, scoped to the **Red Hat OpenShift console** CPU and memory dashboards; metrics access facts are under [Reaching the dashboard metrics](#reaching-the-openshift-dashboard-metrics-winner-point-2a).

### Scope

**In scope for RFC-2:** all of 1.1 (1.1.1–1.1.4) and 2.2. One architecture is being tested against all of it — that is the point of a single RFC. **KIV, not in scope:** 2.1, the Grafana Kafka-throughput side.

**The DE trial is narrower than RFC-2 (Aug 2026): it covers 1.1.1–1.1.3 only.** 1.1.4 is tested by the author personally, not by DEs; 2.2 is not in the trial at all. Never present the trial's result as evidence for either.

Don't build generic "handles everything" tooling. A combination that convincingly handles Argo sync failures beats one that shallowly touches every sub-area — and a clean "this combination doesn't help here" is a better finding than a stretched claim that it does.

## Architecture

RFC-1 settled the **orchestrator**: LibreChat. What sits behind it is the **MCP server** — never "the backend" — and *which* MCP servers is exactly RFC-2's open question. ("MCP client" is still correct where the protocol role is meant, e.g. LibreChat's MCP support.) The org is on GitLab: say **MR**, not PR.

### Standing rules — the AI is read-only everywhere

- **The agent never writes to the cluster, never triggers an Argo sync, and never writes to git.** A standing rule (extended to git 11 Aug 2026), not a trade-off to re-weigh. Write-capable Kubernetes MCP servers exist; their write flags stay off. The GitLab token is read-only. Everything that lands is typed by a human.
- **The gate is at the credential, not at the merge.** The org does not accept LibreChat holding a credential that can write to GitLab at all. Worth stating to stakeholders this way: an AI-authored commit is not something a reviewer has to catch — it cannot exist. The question "what if the AI commits something bad" has no attack surface rather than a mitigation.
- **Read-only is not the same as safe — scope the read too.** k8sgpt's ClusterRole as deployed is `apiGroups: ['*'] / resources: ['*']` with get/list/watch, which **includes every Secret in the cluster** (verified). With a public inference endpoint in play, a secret read reaches chat, MongoDB conversation history and a third-party API. Prefer OpenShift's `cluster-reader`, which reads pods, nodes, PVs and NetworkPolicies cluster-wide but **excludes secrets** (verified). A prompt instruction saying "never read secrets" is advisory, not a control — it belongs in RBAC.
- **On hub the wildcard role is a blocker, not a to-do (14 Aug 2026).** ACM generates Argo's cluster Secrets in `openshift-gitops`, and each holds a bearer token with near-admin rights on a spoke. A wildcard-read server on hub can therefore read **prd's cluster credentials** into a chat window. That is categorically worse than app secrets: tighten the role before anything runs on a real hub.
- **Cluster-wide read also means the model wanders (observed 13 Aug 2026).** Asked about one namespace, it listed others and volunteered findings about an unrelated one. Constrained by `serverInstructions` for now — follow the request path, never browse — but the durable fix is RBAC.

### The deploy flow — where AI stops

**The AI drafts, the human commits.** The AI reads the repos, works out exactly which file and which field, and outputs a ready-to-apply change — file path, the diff, a commit message and an MR description, as chat output only. The DE creates the branch, applies it and raises the MR. The value that survives the read-only rule is the part DEs actually find hard: knowing *which* of four file locations to touch, what the current value is, and what correct YAML looks like. What moves to the human is mechanical — branch, commit, click.

**The repo topology.** The org is on GitLab, organised as one **Group** holding multiple **Projects**. Two carry the deploy flow. The GitHub template repos — [helm-charts](https://github.com/wenbierrr/helm-charts), [argohub](https://github.com/wenbierrr/argohub) — are the authoritative scaffolding spec, and **recreating them on the in-cluster GitLab is completed** (see [PoC environment status](#poc-environment-status)).

| Project | Path | Holds |
|---|---|---|
| **helm-charts** | `charts/<team>/<app>/` | the chart itself — `Chart.yaml`, `values.yaml`, `templates/` |
| **argohub** | `apps/<team>/<app>/base/values.yaml` | values common to every cluster |
| | `apps/<team>/<app>/overlay/<cluster>/values.yaml` | per-cluster values — **this is where `image.tag` lives** |
| | `argocd/application/apps/<team>/<app>/app-<cluster>.yaml` | one Argo `Application` CRD per cluster |

**How the two projects join up — the multi-source `$values` pattern.** Each Application declares two sources: the chart from helm-charts, and argohub as a bare `ref: values` source contributing no manifests. The values files are addressed through that ref, base first and overlay second, so the overlay wins:

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

**The environment is airgapped.** Charts are dragged and dropped in; nothing pulls from an external chart repo. Assume the same of everything else the PoC needs — MCP server images, chart dependencies, the inference endpoint. (Mirroring the demo app's own container image is *not* a PoC concern.)

Two cases, and the distinction matters because only one is cheap enough to demonstrate end to end.

**New app onboarding** — spans both projects, so two MRs:

```
AI     → ask the DE the necessary questions
       → READ both repos to establish layout and conventions
       → draft, as chat output only:
         helm-charts:  the chart under charts/<team>/<app>
         argohub:      apps/<team>/<app>/base/values.yaml
                       apps/<team>/<app>/overlay/<cluster>/values.yaml
                       argocd/application/apps/<team>/<app>/app-<cluster>.yaml
                       (one per target cluster)
       ── stops here ──
HUMAN  → create both branches, apply the drafted files, raise both MRs
       → approve both MRs
       → create the App of Apps by hand, then manually sync
```

**The App of Apps is always created by a human, never by the AI.** The AI's contribution stops at the suggestion of how to change or create Application CRDs that an AOA later sweeps up.

Its shape is a real choice, and the layout is not neutral between them. **One AOA per app across all clusters** falls out naturally — point it at `argocd/application/apps/<team>/<app>/` and it picks up every `app-<cluster>.yaml` beneath. **One AOA per cluster** cannot be expressed as a directory at all, because cluster lives in the filename; it needs a `directory.include` glob such as `app-*-stg.yaml`. Establish which shape is in use before designing anything against it.

**Existing app change** — the common case, and it touches **one project, one file**:

```
DE     → a team asks to bump a container image version, or change the chart
AI     → ask which, and to what version
       → READ the repo to locate the exact file and its current value
         image bump   → argohub:     apps/<team>/<app>/overlay/<cluster>/values.yaml
         chart change → helm-charts: charts/<team>/<app>/
       → output in chat: file path + diff + commit message + MR description
       ── stops here ──
HUMAN  → branch, apply the diff, raise the MR
       → approve the MR
       → Argo GUI → DIFF → manually sync
```

An image bump never touches helm-charts, and a chart change never touches argohub. Neither needs the cross-project coordination that new-app onboarding does.

**The DIFF-and-sync step is where 1.1.1 surfaces.** A faulty chart, or Argo objecting to something else, throws its error at exactly this moment. So the deploy flow (1.1.4) and *workload cannot be synced* (1.1.1) are one workflow observed at two moments, not two separate problems — a single injected fault can exercise both.

### Placement and cluster topology

| Cluster | Runs |
|---|---|
| **hub** | **GitLab and Argo CD** |
| **stg, prd, prd2** | the workloads — the manifests the Helm charts render. **Istio troubleshooting is mainly needed here**, not on hub |

**Starforge** is the org's platform team; DEs escalate to them for platform-level changes (e.g. Istio configuration).

**How hub reaches the spokes — and what that puts within reach of a hub-side reader.** One Argo CD instance, on hub; spokes run none. Registration is done by **RHACM, not `argocd cluster add`**: each spoke joins as a `ManagedCluster`, a `ManagedClusterSet` is bound into `openshift-gitops`, a `Placement` selects eligible clusters, and a `GitOpsCluster` ties that Placement to Argo, which generates the cluster Secrets. Hub self-deployment is disabled (`cluster.inClusterEnabled: "false"`), so every workload Application must name a spoke.

Three consequences that decide what a hub-side Kubernetes MCP server can diagnose:

- **Every `Application` object lives in `openshift-gitops` on hub**; only `spec.destination` differs. So one server on hub reads sync state for *every* spoke — but that is Argo's record of the spoke, never the spoke itself.
- **Env separation is by branch, not just path** — prd tracks `HEAD`/main, stg tracks a team branch `{cluster}/team/app_name`. Expect unresolvable-`targetRevision` to be a frequent 1.1.1 cause.
- **The `default` AppProject is stripped of all rights**, with per-cluster AppProjects restricting destinations and repos. A guardrail design like this *generates* sync failures as its normal mode — "destination is not permitted in project" will be common, and both the error and the AppProject are readable on hub.

**LibreChat and the MCP servers are portable — they can be deployed in any cluster.** Nothing pins them to hub, and portability is one of the epic's evaluation criteria — a point in the architecture's favour. Placement follows from what each server talks to:

- Cluster troubleshooting (1.1.2) and Starforge config issues (1.1.3) live in stg → deploy LibreChat + Kubernetes MCP (or whichever server proves useful) **on stg**.
- Argo sync issues (1.1.1) and app-onboarding guidance (GitLab as source of truth) live on hub → deploy LibreChat + whichever server proves useful **on hub**.

**One LibreChat per cluster. Fan-out will never happen.** A single LibreChat with near-identical tool surfaces registered for stg *and* prd is ruled out by design, not left untested. If stg needs LibreChat + X, deploy it in stg; same for prd and hub. Don't design against a multi-cluster tool surface, and don't score tool-selection-across-clusters in RFC-2.

**The planned topology** — two independent deployments, each serving the sub-areas its servers can actually reach:

| Where | LibreChat + | Serves | State |
|---|---|---|---|
| **stg** | Kubernetes MCP | 1.1.2 pod down / app broken · 1.1.3 Istio | baseline runs pass tc1–tc8 |
| **hub** | Kubernetes MCP · **candidate: GitLab MCP** · a Prometheus-compatible MCP | 1.1.4 deploy · 1.1.1 Argo sync · 2.2 CPU/memory thresholds | **being PoC'd — not chosen** |

**Which servers go on hub is the open question, and it is decided by a cost-benefit comparison presented to stakeholders, not by this file.** Candidate combinations are Kubernetes MCP alone and + GitLab MCP; the capability differences between them belong in RFC-2's matrix. Nothing here is a commitment.

**Hub needs its own Kubernetes MCP server** whichever way that lands — a second deployment reading hub's API, not a second tool surface on one LibreChat. That is not fan-out: each instance reads exactly one cluster.

The hub instance is cross-cluster *by nature*, and that does not contradict the no-fan-out rule: Argo and ACM's Thanos are already **aggregators** presenting one unified surface, not N identical ones.

**The open question this topology creates — 1.1.1.** Three classes hide under "sync failure", and they differ on whether Argo's error text on hub is an explanation or a pointer:

| Class | Cause | Evidence | Hub-only enough? |
|---|---|---|---|
| **Manifest generation fails** — wrong `valueFiles` path, `ref: values` misconfigured, wrong chart path, `targetRevision` won't resolve, Helm template error | GitLab, on hub | `ComparisonError` on hub, self-explaining | **Yes** |
| **Apply rejected by the target cluster** — namespace missing, admission webhook denies, immutable field, CRD absent, Argo's SA lacks RBAC there | stg | Error text on hub; the offending object in stg | Partly |
| **Sync hook fails** — a PreSync Job | stg, where the Job runs | Hub says only "hook failed"; the reason is a pod log in stg | No |

**The first class is the biggest slice, and it is hub-complete.** The `$values` pattern gives it the surface — four addressable paths across two projects, cluster in the filename, chart pinned to `targetRevision: main`. Each of those fails at manifest generation, where the error is on hub and needs nothing from stg.

**With Argo MCP excluded, 1.1.1 is diagnosed by the Kubernetes MCP server reading the `Application` CR** — `status.conditions`, `status.sync`, `status.resources[]`. That covers the first class outright, since the error text is on hub and self-explaining.

**Classify before scoring.** DEs report Synced-but-Degraded as "Argo is red", but manifests that applied cleanly with a pod that won't start is 1.1.2, not 1.1.1.

**The PoC collapses this onto one cluster.** There aren't the resources for separate stg and prd — which is what the reference `Application` CRDs already do with `destination.server: https://kubernetes.default.svc`. Keep the `overlay/<cluster>/` directory shape anyway, because it is the org's real structure. The single cluster models the co-located case honestly: an MCP server sitting alongside the workloads it reads is exactly the arrangement stg and prd would use.

**What it cannot settle is the placement half of 1.1.1.** With `kubernetes.default.svc`, hub *is* stg — no boundary, no second credential, no hop that could fail — so a hub-placed server sees stg state for free and a pass there says nothing about the org's real split. Placement waits for a real spoke; don't record a CRC pass as having answered it.

### GitLab MCP — the read-side of the deploy flow

- **GitLab MCP is a read-only server, and its job is grounding.** It no longer creates branches, commits files or raises MRs — those tools stay unused and the token must not carry the scope to call them. **A read-only token (`read_api` / `read_repository`) is the whole credential.** It lets the AI ground its drafted change in the real repo — find the right file among four candidate locations, read the current `image.tag`, follow existing conventions. Without it the AI guesses paths from memory, which is exactly where a wrong-but-plausible answer comes from. `project_id` is a per-call parameter, so one server reaches both `helm-charts` and `argohub` — do **not** add a second, generic git MCP server alongside it.
- **The working candidate is the community server** [`zereight/gitlab-mcp`](https://github.com/zereight/gitlab-mcp): it has `get_file_contents`, `get_repository_tree` and `list_branches`, supports `GITLAB_PERMISSION_MODE=readonly`, points at self-hosted instances via `GITLAB_API_URL`, and needs no GitLab Duo. Untested against our repos — that is the next PoC.
- **GitLab MCP is evaluated by the author in tandem with the DE trial, not inside it.** The trial covers 1.1.1–1.1.3; GitLab MCP's job is 1.1.4. Its output is the cost-benefit analysis, not a scorecard.

### Reaching the OpenShift dashboard metrics (winner point 2a)

The facts, so nobody re-researches them:

- Observe → Dashboards in the OCP console reads the **Thanos Querier**.
- Programmatic access is the `thanos-querier` route in `openshift-monitoring`: `oc get routes -n openshift-monitoring thanos-querier`.
- **Bearer token auth only**, via a ServiceAccount bound to the **`cluster-monitoring-view`** ClusterRole. Only `/api` paths are reachable on the route.
- Red Hat advises no more than **one query per 30 seconds** — design any polling around that.
- It speaks the Prometheus query API, so any Prometheus-compatible MCP server works pointed at it.
- **ACM aggregates spoke metrics onto hub.** An `observability-endpoint-operator` on each managed cluster pushes that cluster's OCP Prometheus data to a Thanos store on hub, so **one Prometheus MCP server on hub can serve hub, stg, prd and prd2**. **Verify first that ACM Observability is actually enabled, and that container CPU/memory are in its forwarded-metrics allowlist** — ACM forwards a curated set by default. If it isn't enabled, fall back to one server per cluster pointed at that cluster's own `thanos-querier`.

### Verified RBAC and server facts — don't re-derive

- **The `view` ClusterRole does not cover Istio CRDs — verified, all seven return `no`.** An MCP server bound only to `view` gets 403 on VirtualServices, DestinationRules, AuthorizationPolicies and PeerAuthentications, which reads exactly like "kubectl cannot diagnose mesh faults" when it is a permissions bug. Bind an additional minimal read-only ClusterRole over `networking.istio.io`, `security.istio.io`, `telemetry.istio.io` and `extensions.istio.io`. Do **not** bind Istio's own `istio-reader-clusterrole-istio-system`: it grants cluster-wide `secrets` reads and carries `create`/`delete` on `serviceexports`. Run this check on every cluster before any 1.1.3 case.
- **CRC currently sidesteps this (12 Aug 2026):** the Kubernetes MCP server binds a custom wildcard ClusterRole — `get/list/watch` on `*/*`, defined in its `values-sf.yaml` — verified to read all Istio kinds and to refuse writes. It also reads Secrets: acceptable for CRC exploration only; tighten to an enumerated-core-group-minus-secrets role before any DE rollout.
- **k8sgpt's MCP server exposes 12 tools and none write cluster state** ([MCP.md](https://github.com/k8sgpt-ai/k8sgpt/blob/main/MCP.md)). It also cannot reach metrics.
- **k8sgpt ships no Istio analyzers.** Confirmed by tc5–tc8: it cannot see any of them. (A separate Istio MCP server turned out not to be needed — the Kubernetes MCP server reads Istio CRDs fine under the wildcard role.)
- **Argo MCP's tool surface — verified 14 Aug 2026, kept as the basis for its exclusion.** [`mcp-for-argocd`](https://github.com/argoproj-labs/mcp-for-argocd) exposes 14 tools, **five of them writes** (`create/update/delete_application`, `sync_application`, `run_resource_action`). The reach it buys from hub is the same reach a spoke-side LibreChat has natively, and narrower — Argo sees only what it manages, so out-of-tree causes (a webhook, a quota, a NetworkPolicy no Application owns) are invisible to it either way. **That is the elimination factor for RFC-2.**

## PoC environment status

Two environments: **stg** (org cluster — LibreChat + k8sgpt) and **CRC** (local OpenShift). On CRC, **GitLab, Argo CD and Istio are all up (13 Aug 2026) — 1.1.1 and 1.1.4 are unblocked**, alongside LibreChat, k8sgpt and the Kubernetes MCP server.

- **Istio (11 Aug 2026):** upstream 1.23.2 via `istioctl`, matching stg's version, using `--set profile=openshift` (the `openshift` profile was only removed in 1.24, so the newer `values.global.platform=openshift` form does not apply here).
- **Argo CD (12 Aug 2026):** OpenShift GitOps operator v1.21.2; default instance in `openshift-gitops` — exactly the namespace the scaffolding's Application CRDs declare, so they land untouched. The default instance only acts in namespaces labelled `argocd.argoproj.io/managed-by=openshift-gitops`; `stg` and `prd` exist and carry the label. Admin password: secret `openshift-gitops-cluster`.
- **GitLab (12 Aug 2026):** single-container GitLab CE (`gitlab/gitlab-ce:latest`, 19.x line) via `miscellaneous/gitlab-ce.yaml` — ns `gitlab`, anyuid SCC for its SA, 3 PVCs, plain-HTTP Route `gitlab.apps-crc.testing`, root credential in secret `gitlab-root`. Two boot fixes are baked into the manifest and must not be dropped: kubelet probe sources sit in `monitoring_whitelist` (outside it GitLab hides `/-/readiness` behind a 404 and the pod restarts forever) and a 20-min startupProbe budget. Known trade of plain HTTP: the Web IDE cannot authenticate (OAuth/PKCE needs a secure context) — use the single-file editor.
- **GitLab Operator — tried and eliminated (12 Aug 2026).**
- **Scaffolding recreated in-cluster (13 Aug 2026):** GitLab group `starforge` (public) with projects `helm-charts` and `argohub` pushed from the GitHub templates; argohub's Application CRDs' `repoURL`s point at the in-cluster GitLab. Hand-made AOA `hello-app-aoa` → `hello-app-demo-stg`/`-prd` both Synced/Healthy, nginx pods running in `stg`/`prd`. **The GitLab → Argo → cluster chain is verified end to end.**

## Current status

Against the epic's acceptance criteria:

- 🔨 **Trial of the proposed solution by DE** — designed and agreed; runs weeks 3–5. Detail in [DE-trial-execution.md](DE-trial-execution.md).
- 🔨 **RFC on choice of tools** — [RFC-1](rfc-1-ai-tool-evaluation.md) done (orchestrator); [RFC-2](rfc-2-mcp-server-eval-for-librechat.md) active, written weeks 6–8 from the trial's scorecards.
- ⏸️ **`service-design` of the solution, if the ADR goes ahead** — production security, RBAC, rollout. Same artifact this file calls RFC-X. Conditional; may never be written.

**Where the runs stand (14 Aug 2026):**

- **The benchmark is 9 cases.** tc1–tc4 core Kubernetes (OOM · liveness-probe port · Service selector · default-deny-ingress); tc5–tc7 Istio (no sidecar vs STRICT mTLS · AuthorizationPolicy deny · undefined routing subset); **tc8** a three-namespace `exportTo` visibility fault where the symptom is two hops from the cause and a correct AuthorizationPolicy sits in the path as a decoy; **tc9** an Argo 1.1.1 case — a self-deleting `Job` compared as permanent drift. **tc9 is written but not yet run.**
- **tc9 is the first case built to discriminate between server combinations.** tc1–tc8 were all passed by LibreChat + Kubernetes MCP alone, so every matrix column reads the same and they separate nothing. tc9 is gradeable by reach: the Kubernetes MCP arm gets the Application CR and the deleted Job's surviving events but never the Job's spec; the GitLab MCP arm gets the chart source and a correctly-hooked counterpart Job as a worked example. (Its third arm was Argo MCP, now excluded.) It also inverts the usual failure mode — **the correct answer includes "no workload is broken"**, so inventing a fault to explain a red Application is the thing being caught.
- **LibreChat + Kubernetes MCP passed all 8 first try** — root cause and suggested fix correct, no hallucinations, and with **no `serverInstructions` in force at all**.
- **k8sgpt eliminated.** What LibreChat + k8sgpt can troubleshoot is a proper subset of what LibreChat + Kubernetes MCP can: it ships no Istio analyzers, so tc5–tc8 are invisible to it, and it needs 45 lines of tuning to handle the four it can see. **Elimination factor: no reliability advantage on the shared cases, no reach on the rest, and a prompt to maintain forever.**
- **Still baseline confidence, not evidence** — injected faults, injector knows the answers, single-fault namespaces. tc8's run was also **contaminated**: tc5–tc7 were live in the same cluster and the wildcard read role let the model read tc7, which is the case tc8 is built to resemble. Re-run it isolated before citing it.

**Supervisor steer (Aug 2026), and it outranks more fabricated cases:** unit-test-style cases only establish baseline confidence. What convinces stakeholders is the tool against **real stg issues**, with the fault catalogue derived from **incidents that actually recur on our clusters** — then replicated on stg with DEs using it. Mining that incident history is research work needing no cluster, so it runs in parallel with everything else.

**The DE trial, agreed Aug 2026** — full detail in [DE-trial-execution.md](DE-trial-execution.md):

- **13 DEs, 14 scenarios, 39 attempts** — 13 without AI and **26 with AI**, covering 1.1.1–1.1.3. Each DE runs 3 scenarios, one per family (Argo · k8s-native · Istio), in a 35–40 minute box.
- **The first scenario is run without AI (15 min), the other two with AI (10 min each).** This is what finally gives the project a **prospective baseline**: the same scenario is run both ways by different DEs, so time saved is *computed*, not estimated. It supersedes the banded DE-estimate below.
- **It runs on the author's laptop (CRC), not the airgapped stg** — hub and stg collapsed onto one cluster, which is what lets Argo scenarios back in. A CRC pass still cannot settle the placement half of 1.1.1.
- **No non-SF DE attempts Istio without AI.** With 3 SF DEs, that fixes the whole allocation: 5/5/3 without AI across Argo/k8s/Istio, and **20 of the 26 AI scorecards come from non-SF DEs**.
- **Two records, because ground truth and experience come from different people.** The facilitator records the timings, whether the root cause was reached, and whether the AI hallucinated; the DE fills one scorecard at the end covering all three scenarios.
- **Rubric: Resolution quality · Time saved · Actionability · Learning · Willingness to use**, each graded **1–5** (No Go / Poor / OK / Good / Excellent). The scorecard anchors only the endpoints; all five levels are defined in the rubric table.
- **`Non-SF` is the agreed proxy for "less experienced"** — SF are the platform specialists. Winner point 3 is scored on that basis, and every figure is split SF against non-SF.
- **The pass bar is deliberately not yet set** — to be agreed before the trial starts, and before any data exists.

See [plan.md](plan.md) for the 8-week schedule. When the active work changes, update this section and plan.md together.

## Measuring benefit — there is no historical baseline

`epic.md` says benefit is measured against a baseline established during profiling. **That baseline does not exist for D-2-D deployment troubleshooting.** So: never present a before/after time saving as though it were measured, and don't back-fill a plausible "usually takes N hours" figure. A fabricated baseline would make the whole benefit case unsound — and it's exactly the kind of thing that survives into a decision memo unchallenged.

Workable substitutes, in order of strength:

- **Capture the baseline prospectively.** During a pilot, run a real D-2-D failure both ways — DE without AI, and DE with the agent — and time both. Small n is fine; say what n was.
- **Count outcomes instead of minutes.** Did the agent identify the actual root cause? Could the DE have solved it without the agent at all? Countable without historical data, and maps directly onto what the supervisor cares about.
- **Structured DE judgement.** DEs who ran the pilot say whether it helped and where. Weak evidence, honest when labelled as such.

Whichever is used, state the method and its limits alongside the result.

**This bites concretely on RFC-2's pilot scorecard.** Two fields sit on opposite sides of the line:

- *"Roughly how long without the tool?"* — was a **DE estimate**, and the failure mode was quiet: an estimate hardening into a measured figure between scorecard and decision memo. **Superseded (Aug 2026)** — the DE trial now runs each DE's first scenario without AI, so almost every scenario has a real without-AI time from another DE and time saved is **computed, not asked** (I1 and I2 are the exceptions; they fall back to the Istio average). The question is off the scorecard entirely. Don't reinstate it.
- *"Without AI, would you have been able to solve this?"* — needs no baseline, is countable, and maps straight onto the toil the epic set out to reduce. Lead with this one. **Do not add an escalation-avoided question alongside it (supervisor's call, Aug 2026):** not having to escalate is a *consequence* of being able to solve the problem, so asking both counts the same effect twice.

**The stg runs are not evidence — the pilot is.** Conflating them is the easiest way to overstate the case.

- **Hands-on runs on stg** build a feel for the tool and produce the tuned configuration. Few cases, no baseline, and whoever runs them already knows the root cause. Useful for filling the comparison matrix and deciding which MCP servers are needed. **Never cite them as evidence of value.**
- **The pilot** puts the tuned tool in front of DEs who don't know the answer, scored against bars agreed in advance. That is what the decision rests on.

**The pilot forces its own data points.** Faults are injected from a networking-weighted catalogue rather than waiting for organic failures — a one-week window won't reliably produce enough. The answer key stays sealed; the injector doesn't score. Two consequences:

- **Percentages need n.** Fix the sample size before agreeing any percentage bar; at n=3 a percentage is noise. RFC-2 proposes n ≥ 8, reported as counts alongside percentages.
- **Injected faults are cleaner than real ones**, and the injector knows what the tool can see. RFC-2 states this and the other limits explicitly. Don't let the pilot get described as proof.

**Each DE case is timeboxed — 15 minutes without AI, 10 minutes with AI**, announced up front so it reads as a rule of the exercise, not the DE giving up. Two consequences for the numbers:

- **Time-to-root-cause becomes censored data.** A case that hits its timebox did not take that long — it did not finish. Record it as **unresolved**, and never average it with completed times; averaging censored runs silently flatters the tool. Where a *without-AI* run is censored, the trial assumes a baseline of **30 minutes**, on the reasoning that the DE would have escalated and the case would have run on. **Treat that as an assumption, not a measurement** — the only observed fact is that it took longer than 15 minutes, so a 30-minute baseline inflates the computed saving rather than understating it. Label it as an assumption wherever the figure is quoted, and say what the saving looks like against the observed 15-minute floor as well.
- **Unresolved is not the same as `no root cause found`.** The first says the tool was too slow to be worth reaching for; the second says it was wrong or empty. Only the second bears on resolution quality. The timebox also caps what a *wrong* root cause can cost — re-read that pass bar when it is agreed rather than carrying over a figure set against unbounded sessions.

## The evaluation criteria are a promotion gate

`epic.md` lists criteria covering blast radius, security, auditability, cost, risk, maintenance burden, and portability. These are **a gate, not a running constraint** — two modes of work, and the criteria bind in only one:

**Exploration** — the RFC-1/RFC-2 work: trying tools, spiking, running stg cases. The criteria do not apply. Don't build RBAC, audit trails, or platform abstraction here. Hack freely against stg.

**Promotion** — the boundary is concrete: **RFC-2's Day-0 thresholds**, agreed by the team *before* any pilot data exists, scored at the end of the pilot week. Passing means proceeding to RFC-X (production security, RBAC wiring, rollout) — not to production directly. Two notes: the drafted-change bar (winner point 1) is assessed **by the author personally, not by the DE trial**, and **time saved *is* among the thresholds since Aug 2026** — the supervisor's call, now measured from the with-AI/without-AI runs rather than estimated by the DE.

The gate is a real filter: a tool can be impressive in exploration and still fail on unbounded blast radius, no DE-identity mapping in the audit trail, or an uncarryable maintenance burden. Fixing the thresholds before the data exists is what stops them being quietly relaxed to fit a result. Don't let a spike drift toward production without passing the gate, and don't silently apply gate-level engineering to something still being explored. A failed gate that names its elimination factor is a valid answer to the epic's question, not a failed project.

Two criteria are worth watching even during exploration, because they affect whether a finding is trustworthy at all:

- **Resolution quality** — a confidently wrong answer is worse than no answer. Record wrong-but-plausible outputs explicitly; they are the finding, not noise.
- **Portability** — multi-cloud is the org's stated direction. OpenShift-native approaches may be explored, but note where a result depends on OpenShift specifics so the decision isn't made on a false assumption of transferability.

## The RFC series

**There were once six planned RFCs. There are now two, plus a maybe** — the supervisor's call: all three problem areas ride on the *same* architecture, so splitting by problem would produce documents that each re-argue it. Don't split them back apart.

| | Question | State |
|---|---|---|
| [rfc-1-ai-tool-evaluation.md](rfc-1-ai-tool-evaluation.md) | Which tool architecture? K8sGPT vs kubectl-ai vs kagent vs LibreChat + an MCP server | **Done.** Verdict: **LibreChat + MCP server** |
| [rfc-2-mcp-server-eval-for-librechat.md](rfc-2-mcp-server-eval-for-librechat.md) | **LibreChat + which combination of MCP servers brings DEs the most value?** Covers troubleshooting (1.1.1–1.1.3), deploy (1.1.4), the CPU/memory threshold work (2.2), and the DE rollout | **Active** |
| RFC-X | Production security, RBAC wiring, rollout | **May never be written.** Only if a tool passes its Day-0 thresholds — deliberately not numbered, because this is R&D, not a delivery pipeline |

**Answers first, RFC last.** Weeks 1–5 find out whether LibreChat + X is useful and in which combination; weeks 6–8 write it up. Before week 6, "write the RFC" is *not* the task — running experiments and collecting scorecards is. Don't let a future session draft conclusions it doesn't have evidence for.

**RFC-2 has an order of investigation, not a menu:** k8sgpt first → then Kubernetes MCP, which superseded it → then GitLab MCP → then a Prometheus-compatible MCP server. **Argo MCP was dropped from the order (Aug 2026)** and never tested.

**Progress (Aug 2026):** kubectl MCP done · k8sgpt eliminated · **Argo MCP excluded** — not tested, not in the trial. RFC-2 needs a named elimination factor for it and its matrix column closes.

**The plan from here:** bring the Kubernetes MCP server into the **airgapped environment**, then run the DE trial on it. **In tandem**, the author evaluates GitLab MCP and produces the cost-benefit analysis. GitLab MCP is not in the DE trial — it reaches 1.1.4, which the trial excludes.

Two things about RFC-1 that are easy to get wrong:

- **RFC-1 settled the orchestrator, not the MCP server.** Its scoring paired LibreChat with k8sgpt as a reference implementation — cells marked *(k8sgpt ref.)*. Don't read them as a decision that k8sgpt is the MCP server; that is RFC-2's question.
- **RFC-1's appendix preserves the kagent Collector/Diagnostician design, but kagent was eliminated.** The appendix is kept to show the research was done.

### House style for the RFCs

**RFC-1 is the template. RFC-2 mirrors it.** Structure: ToC → Motivation → Scope → Brief overview table → Options considered → Comparison matrix.

- **Motivation: two lines, then the question** — a paragraph opening *"The question this RFC answers: …"* which also names what follows in the next RFC.
- **Brief overview table** compares candidates on the same rows: why it was built, maturity, what it adds, what is lost.
- **Options are numbered** (`Option 1 — …`), and the numbers carry into the matrix column headers and the verdict list.
- **Comparison matrix** splits each driver into **Information** (the fact) and **Evaluation metric** (the judgement). Cells stay **blank until tested** — a plausible guess defeats the point.
- **Verdict is a numbered list with a named elimination factor per option**, in option order.

**`example-of-rfc/example-rfc.md` is a source of craft, not a skeleton.** It was tried as a template and rejected — don't restructure the RFCs around its metadata block, change log, `R1…Rn` requirements or Met/Gap/Unverified scorecard. What *is* worth stealing:

- **Quantify or say untested** — "~2–3 min vs ~5s locally (~25–35x)", never "slower".
- **Name what you didn't test** — marking the gaps is what makes the rest credible.
- **Tables carry the argument; prose only in the verdict.**
- **Everything gets a revisit condition** — "becomes a requirement if…", "re-check on first tagged release".
- **A workaround you accept must state what would let it be dropped**, or it's an unbounded concession.

## Deliverable formats

Markdown prose documents in the repo — for enablement material, inventories, and the final proposal alike. Not slides, not external docs.
