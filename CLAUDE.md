# CLAUDE.md

Guidance for Claude Code working in this repository.

## Repository state

**Documents, plus three vendored upstream repos.** Tracked content is `epic.md`, `DE-trial-execution.md`, `rfc-1-ai-tool-evaluation.md`, `README.md`. No build system or test suite of our own.

| Directory | Holds | Tracked? |
|---|---|---|
| `LibreChat/` | upstream `chart-2.0.7-66-g8fcb77fe6`. Local: `helm/librechat/values-sf.yaml` + chart edits for an optional `/app/uploads` volume | own `.git`, gitignored |
| `kubernetes-mcp-server/` | upstream `v0.0.63-31-g9c6ef49`. Local: `charts/kubernetes-mcp-server/values-sf.yaml`, `Dockerfile` patched for a containerd CVE | own `.git`, gitignored |
| `gitlab-mcp/` | `zereight/gitlab-mcp` `v2.1.46-3-g926d42c` — **backlogged, untested** | own `.git`, gitignored |
| `test-cases/` | the 14-scenario fault catalogue + answer keys | **gitignored, no `.git` — versioned nowhere** |
| `miscellaneous/` | Istio-on-CRC install guide, `gitlab-ce.yaml`, `verify-mcp-rbac.sh` | **gitignored, no `.git` — versioned nowhere** |
| `DE-trial-results/` | the two response spreadsheets + [`analysis.md`](DE-trial-results/analysis.md) | untracked, not ignored |
| `team-guidelines/` | `documentation-guide.md` — the org's doc process (Diátaxis; RFC → ADR → service design) | untracked, not ignored |
| `templates/` | `RFC-template.md`, `ADR-template.md`, `svc_design-template.md` | untracked, not ignored |
| `example-of-rfc/`, `example-of-adr/` | worked examples, craft reference only | untracked |

**The `-sf.yaml` files and the `serverInstructions` prompts inside them are the PoC's tuned configuration — deliverables, not scratch.** They exist only as uncommitted edits inside ignored directories, so a re-clone loses them.

Deploy with `helm install`/`upgrade -f <chart>/values-sf.yaml`. The subchart `.tgz` files under `LibreChat/helm/librechat/charts/` are vendored deliberately — **do not run `helm dependency update`**.

## What this project is

R&D. The goal is to answer a question, not ship a system: **can AI measurably make duty engineers' (DE) lives better?** The audience is the supervisor, who uses the answer for a go/no-go. Nothing here is intended for production — **optimise for evidence that would change the decision.**

`epic.md` holds the formal scope. Out of scope there: LLM hosting (assume an endpoint) and model selection.

**Quantify what is useful; for everything rejected, name the elimination factor.** A rejection with a named reason carries the same decision weight as the winner. "Didn't work out" is not an elimination factor; *"we'd be running a whole agentic platform in-cluster forever, and the autonomy isn't worth it"* is.

**The three winner points**, and what the trial actually reached:

| | Winner point | State |
|---|---|---|
| **1** | **Prompt, then prepare the deployment** — AI drafts the exact change; a human commits, raises the MR, approves, syncs | **Backlogged** with GitLab MCP |
| **2a** | **Show the blind spots of the OpenShift CPU/memory dashboards** — define the threshold, then how to stop the bleeding and stop it recurring | **Untested** — needs a Prometheus MCP server, never deployed |
| **3** | **A less-experienced DE says it was useful** — not a senior DE, not the author | **Answered** — see [Where the work stands](#where-the-work-stands) |

**Don't force the winner points into the architecture.** If a combination doesn't naturally address one, that is the finding — record it. A clean no beats a contorted "sort of".

## The DE problem space

**The duty engineer changes every day** — whoever is on duty meets today's problem cold. That is much of why an assistant might help, and why winner point 3 measures a *less-experienced* DE. **~90% of real DE issues are networking**, which is why the fault catalogue is networking-weighted and why Istio visibility drove the MCP server choice.

```
1. DE troubleshooting complexities                       ← current focus
   1.1 D-2-D deployment
       1.1.1 Workload cannot be synced on Argo           AI-tractable · in trial
       1.1.2 Workload synced but pod down / app broken   AI-tractable · in trial
       1.1.3 Additional configs Starforge must make      AI-tractable · in trial
             (e.g. Istio)
       1.1.4 Deploying via Argo — AI drafts the change,  BACKLOGGED
             a human commits, raises the MR, syncs
2. Incident response
   2.1 Monitoring dashboards                             KIV — do not write up
   2.2 Escalating incidents to Starforge                 partly — untested
```

**1.1.4 · backlogged (Sep 2026).** Deferred with GitLab MCP by the supervisor. The durable facts, so they aren't re-derived: the org is on GitLab, one Group with projects **helm-charts** (`charts/<team>/<app>/`) and **argohub** (`apps/<team>/<app>/base|overlay/<cluster>/values.yaml`, plus `argocd/application/.../app-<cluster>.yaml`). They join via the **multi-source `$values` pattern** — the Application declares the chart source plus argohub as a bare `ref: values`, base then overlay. Two consequences: the chart is consumed from a git path at `targetRevision: main`, so a chart change lands the moment its MR merges; and **cluster is encoded in the filename, not the directory**, which decides what shape an App of Apps can take. An image bump touches argohub only; a chart change touches helm-charts only.

**2.1 · KIV.** The Grafana Kafka-throughput dashboard (records/sec from SP1 into hub/stg/prd/prd2). **Deliberately in no document — don't write it up or scope work against it.** If ever picked up: the baseline is statistical work (percentiles, seasonality) computed from Thanos, not something an LLM does by reading graphs.

**2.2 · partly tractable, untested.** The handoff to Starforge is human. The remediation is tractable — stop-the-bleeding steps, then the persistent fix, plus CPU/memory reallocation **with justification**. **Sequencing: define the threshold X first**; only then can "it has gone above X, so do Y" be answered.

**Classify before scoring.** DEs report Synced-but-Degraded as "Argo is red", but manifests that applied cleanly with a pod that won't start is 1.1.2, not 1.1.1.

## Architecture

RFC-1 settled the **orchestrator**: LibreChat. Behind it sits the **MCP server** — never "the backend". The org is on GitLab: say **MR**, not PR.

### Standing rules — the AI is read-only everywhere

- **The agent never writes to the cluster, never triggers an Argo sync, and never writes to git.** A standing rule (extended to git 11 Aug 2026), not a trade-off to re-weigh. Write flags stay off; the GitLab token is read-only. Everything that lands is typed by a human.
- **Read-only is not safe — scope the read too.** For kubernetes MCP server cluster role assigned, make sure it cannot read secrets. In tandem, make sure it has sufficient RBAC to perform whatever cluster troubleshooting and solve Argo application sync issues. 
- **On hub a wildcard role is a blocker, not a to-do.** ACM generates Argo's cluster Secrets in `openshift-gitops`, each holding a near-admin bearer token for a spoke — so a wildcard reader on hub can read **prd's credentials** into a chat window.
- **Never let it run cluster-wide queries.** `pods_list`, `events_list` and `resources_list` all default to **every** namespace, and a live cluster carries 60+ namespaces and projects — so one unscoped call is slow and expensive, and returns almost nothing relevant when the question is "why is this pod not running". It also makes the model wander: asked about one namespace, it volunteered findings about another (observed 13 Aug 2026). **Unless the DE explicitly asks to scan the whole cluster, every call must be namespace-scoped** — enforced by rules 1 and 2 of the Kubernetes MCP `serverInstructions`, which require a namespace before any tool call and `namespace` passed on every tool that accepts it.

### Verified facts — don't re-derive
- **The Kubernetes MCP server runs `kubernetes-mcp-server-read-no-secrets`** — core group enumerated minus Secrets, Istio/Argo/OpenShift/metrics readable, `pods/exec`, `pods/attach`, `pods/portforward` and `nodes/proxy` omitted, plus `config.read_only: true`. Live-verified 9 Sep 2026 by `miscellaneous/verify-mcp-rbac.sh` — **78/78 pass**, and `GET /api/v1/secrets` as the SA returns **403** against a valid-token control; re-run it on every new cluster.
- **k8sgpt ships no Istio analyzers**
- **`nodes/proxy` cannot be scoped by path** — allowing `/stats/summary` also allows `/exec`, `/attach` and `/portForward` on every node, so it stays omitted and `nodes_stats_summary` is correctly denied. **Node state is unaffected**: conditions, capacity, allocatable, taints and cordon status all come from `nodes`/`nodes/status`, and live usage from `metrics.k8s.io`.

### Placement and topology

| Cluster | Runs |
|---|---|
| **hub** | GitLab and Argo CD |
| **stg, prd, prd2** | the workloads. **Istio troubleshooting is mainly needed here**, not on hub |

**Starforge** is the org's platform team; DEs escalate to them for platform-level changes.

**One LibreChat on hub, all four MCP servers beside it — NOT CONFIRMED, needs an RFC then an ADR.** DEs open one URL and pick the cluster in LibreChat's MCP picker: `kubernetes-mcp-hub`, `-stg`, `-prd`, `-prd2`. **The alternative:** a LibreChat + MCP server pair installed on every cluster.

**The picker is the cluster selector — the model never infers which cluster it is on.** Tools are namespaced `<tool>_mcp_<serverName>`, and **only ticked servers' tools *and* `serverInstructions` reach the model** ([`agents/load.ts`](LibreChat/packages/api/src/agents/load.ts), [`agents/context.ts`](LibreChat/packages/api/src/agents/context.ts)), so an unticked cluster is invisible.

**If the ADR decides on all four MCP servers on hub:** hub's own server is a ClusterIP call and already works; `-stg`, `-prd` and `-prd2` each reach their cluster by kubeconfig, so hub holds one spoke credential each — scoped to `kubernetes-mcp-server-read-no-secrets` there.

Spokes join via **RHACM, not `argocd cluster add`**, and hub self-deployment is off. Two consequences:

- **Every `Application` lives in `openshift-gitops` on hub** — one hub-side server reads sync state for every spoke, but that is Argo's record, never the spoke itself.
- **Env separation is by branch** — prd tracks main, stg a team branch. Expect unresolvable `targetRevision` as a common 1.1.1 cause.

**1.1.1 splits three ways**, differing on whether hub's error explains or only points:

| Class | Where the evidence is | Hub-complete? |
|---|---|---|
| Manifest generation fails — bad `valueFiles`, unresolvable `targetRevision`, template error | `ComparisonError` on hub, self-explaining | **Yes**, and the biggest slice |
| Apply rejected by the target | error on hub, object in stg | Partly |
| Sync hook fails | hub says only "hook failed"; the reason is a pod log in stg | No |

The Kubernetes MCP server deployed on HUB covers the first class outright from the `Application` CR's `status.conditions`, `status.sync` and `status.resources[]`.

**The PoC collapses hub and stg onto one CRC cluster**, which is what lets Argo scenarios in at all — but it cannot settle the placement half of 1.1.1, since `kubernetes.default.svc` means no boundary and no second credential to fail.

## PoC environment

CRC (local OpenShift) on the author's laptop: 16 vCPU / 64 GB. GitLab, Argo CD and Istio all up since 13 Aug 2026, alongside LibreChat and the Kubernetes MCP server.

- **Istio (11 Aug 2026):** upstream 1.23.2 via `istioctl`, matching stg, `--set profile=openshift` (the profile was removed in 1.24, so the newer `values.global.platform=openshift` form does not apply).
- **Argo CD (12 Aug 2026):** OpenShift GitOps operator v1.21.2, default instance in `openshift-gitops` — the namespace the scaffolding's Application CRDs already declare. It acts only in namespaces labelled `argocd.argoproj.io/managed-by=openshift-gitops`. Admin password: secret `openshift-gitops-cluster`.
- **GitLab (12 Aug 2026):** single-container GitLab CE via `miscellaneous/gitlab-ce.yaml` — ns `gitlab`, anyuid SCC, 3 PVCs, plain-HTTP Route `gitlab.apps-crc.testing`, root credential in secret `gitlab-root`. **Two boot fixes are baked in and must not be dropped:** kubelet probe sources in `monitoring_whitelist` (outside it GitLab hides `/-/readiness` behind a 404 and the pod restarts forever), and a 20-min startupProbe budget. The GitLab Operator was tried and eliminated.
- **Scaffolding (13 Aug 2026):** GitLab group `starforge` with projects `helm-charts` and `argohub`, pushed from the GitHub templates, `repoURL`s pointing at the in-cluster GitLab. **The GitLab → Argo → cluster chain is verified end to end.**

## Where the work stands

**The DE trial is complete (Sep 2026).** 13 DEs × 3 scenarios = 39 runs, 13 without AI and 26 with, covering 1.1.1–1.1.3 against a 14-scenario injected-fault catalogue. Design in [DE-trial-execution.md](DE-trial-execution.md); raw data and full working in [DE-trial-results/analysis.md](DE-trial-results/analysis.md).

| | Result |
|---|---|
| Root cause reached | **96% with AI (25/26)** vs 69% without (9/13) |
| Hallucinations | **1 of 26** — caused by no repository visibility; the model invented chart scaffolding it could not read |
| Willingness to use next rotation | **13 of 13 scored 5/5** |
| Non-SF DEs who could not have solved it alone | **16 of 20 (80%)**; 0 of 6 for SF |
| Time saved | **7 of 14 comparable runs saved time**, 4 no change, 3 slower |

**The time figures need their caveat every time they are quoted.** Savings appear only where the without-AI baseline exceeded ~6 minutes: scenarios with 10+ minute baselines saved 27–68%, scenarios under 4 minutes saved nothing. That is a **floor effect** — when a DE solves something unaided in two minutes there is no time for a tool to save. **Never report a single blended mean**; it averages a real effect with a measurement floor and understates both.

**No pass bar was ever agreed** — the design never set one. Report the figures; do not set a bar now against data that already exists, and do not reverse-engineer one the data happens to clear.


## Deliverables

Two documents, per the supervisor (Sep 2026). Follow `templates/` and `team-guidelines/documentation-guide.md` for both.

| | Question | State |
|---|---|---|
| [rfc-1-ai-tool-evaluation.md](rfc-1-ai-tool-evaluation.md) | Which tool architecture? K8sGPT vs kubectl-ai vs kagent vs LibreChat + an MCP server | **Done.** Verdict: LibreChat + MCP server |
| **ADR** | Record the decision to adopt **LibreChat + Kubernetes MCP server**, evidenced by the DE trial | **Done.** |
| `service-design` | Production security, RBAC wiring, rollout | **Next** |

**The ADR carries three things:** the decision and the trial evidence behind it (willingness 13/13, 96% resolution, the non-SF split); why Kubernetes MCP over k8sgpt — it reaches strictly more, since k8sgpt has no Istio analyzers and needs 45 lines of tuning for the cases it can see; and, as an accepted trade-off, that **GitLab MCP was never tested**, which is also the named cause of the trial's single hallucination.

**Argo MCP is not under consideration and is not to be mentioned.**

**Format: Markdown documents committed to this repo.** Not slides, not Google Docs.

**`templates/` gives the structure; `example-of-rfc/` and `example-of-adr/` show how to write a document concisely, non long-winded, and straight to the point.**

**Two things about RFC-1 that are easy to get wrong:** it settled the orchestrator, not the MCP server — its matrix paired LibreChat with k8sgpt as a *reference implementation*, and cells marked *(k8sgpt ref.)* would change with a different server. And its appendix preserves the kagent Collector/Diagnostician design even though kagent was eliminated, kept to show the research was done.
