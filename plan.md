# 8-week plan

**Week 1 = 10 Aug 2026 · Week 8 = 28 Sept 2026**

The question these 8 weeks answer: **LibreChat + which combination of MCP servers brings DEs the most value?**

Weeks 1–5 find the answer. Weeks 6–8 write it up. The RFC is written *from* findings, not alongside them — a document written backwards from evidence beats one written forwards from hopes.

## Deliverables

| | Acceptance criterion | What it actually is | Status |
|---|---|---|---|
| 1 | Written inventory of DE chores, with time estimates and agent-suitability assessment | The profiling that drove the focus decision | ✅ Done |
| 2 | Enablement material delivered in markdown and presented | The RFCs, PoC write-ups and research | 🔄 Continuous |
| 3 | Working PoC covering at least one focus area | LibreChat + the chosen MCP server(s) running on stg, in DEs' hands against fabricated and real failures | 🔄 Continuous |

## Week by week

| Weeks | Focus | What I'm doing |
|---|---|---|
| **1–3**<br/>10–28 Aug | **Experiment** | • Try the different LibreChat + MCP server combinations across all three problem areas — troubleshooting, deploying, and the CPU/memory threshold work<br/>• Order: **kubectl first**, then judge whether k8sgpt earns its place on top, then Argo (read-only), then GitLab, then metrics<br/>• Agree the **scorecard and pass bars** with the team<br/>• Design **fabricated networking failures** — if nothing breaks on its own during the rollout, DEs have no reason to reach for the tool, so faults are injected to force the data points |
| **4–5**<br/>31 Aug – 11 Sep | **Roll out to DEs** | • DEs use the tool against the fabricated failures and whatever real ones arrive<br/>• One scorecard per issue<br/>• Target the less-experienced DEs — that's the group the value claim rests on |
| **6–8**<br/>14 Sep – 2 Oct | **Findings + write-up** | • Read the scorecards, derive the findings<br/>• Work out which combination actually brought the most value, and where it brought none<br/>• **Write the RFC** |

## What the three problem areas are

Each combination gets tested against all three — that is how we find out which one is worth having.

| | Problem |
|---|---|
| **Troubleshoot** | Argo sync failures, pods down / apps broken, Istio config (1.1.1–1.1.3) |
| **Deploy** | AI creates a branch, edits the scaffolding and values files, raises an MR. A human approves and syncs — **AI never syncs** (1.1.4) |
| **Threshold + remediate** | On OpenShift CPU/memory dashboards: determine what X is, then if X is exceeded, how to stop the bleeding and what the remediation steps are (2.2) |

## Also on the board

| | | |
|---|---|---|
| **RFC-X** · production security | **May never be written** | Security, RBAC and rollout — only if a tool passes the Day-0 thresholds |
| Grafana Kafka-throughput baseline (2.1) | **KIV** | Deliberately parked — not scoped, not written up |
