# 8-week plan

**Week 1 = 10 Aug 2026 · Week 8 ends 2 Oct 2026**

The question these 8 weeks answer: **LibreChat + which combination of MCP servers brings DEs the most value?**

## The plan

| Weeks | Focus | What lands |
|---|---|---|
| **1–2**<br/>10–21 Aug | **Experiment** | Run the LibreChat + MCP server combinations against all three problem areas. Order: **kubectl → k8sgpt only if it earns its place → GitLab → metrics**.<br/>Agree the [rubric](DE-trial-execution.md#de-trial-rubric) and pass bars with the team. Build the injected fault catalogue — nothing breaks on cue, so the data points are forced |
| **3–5**<br/>24 Aug – 11 Sep | **DE trial** | 13 DEs run 3 injected faults each in a 35-minute box — **the first unaided (15 min), the other two with AI (10 min each)**, which is what makes time saved measurable rather than estimated. One [rubric](DE-trial-execution.md#de-trial-rubric) scorecard per DE, aimed at the **less-experienced DEs** — that is the group the value claim rests on.<br/>*This is where evidence starts.* |
| **6–8**<br/>14 Sep – 2 Oct | **RFC + service design** | **[RFC-2](rfc-2-mcp-server-eval-for-librechat.md)**, written from the scorecards: which combination brought value, and where it brought none.<br/>Then **`service-design`** — production security, RBAC, rollout — **only if the trial passes its Day-0 thresholds and the ADR goes ahead.** |

All three acceptance criteria sit in weeks 3–8: the DE trial, the RFC, and the conditional service design. [RFC-1](rfc-1-ai-tool-evaluation.md) already settled the orchestrator.

## What gets tested

Every combination is tested against all three — that is how we find which one is worth having.

| | Problem | |
|---|---|---|
| **Troubleshoot** | Argo sync failures · pods down / apps broken · Istio config | 1.1.1–1.1.3 |
| **Deploy** | The AI drafts the exact change — file path, diff, commit message, MR description — as chat output. A human commits, raises the MR, approves and syncs. **The AI never writes to git or the cluster.** | 1.1.4 |
| **Threshold + remediate** | OpenShift CPU/memory dashboards: determine what X is, then how to stop the bleeding and what the persistent fix is | 2.2 |

## How the trial is run

The rubric, the pass bar and the scorecard DEs fill in per case live in **[DE-trial-execution.md](DE-trial-execution.md)**.
