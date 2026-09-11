# ADR-NN: Adopt LibreChat with the Kubernetes MCP server for DE cluster troubleshooting

| **Metadata**       | **Value**                                          |
|--------------------|-----------------------------------------------------|
| **Status**         | Proposed                                            |
| **Authors**        | @wenbierrr                                          |
| **Decision Date**  | 2026-09-11                                          |
| **Source**         | [rfc-1-ai-tool-evaluation.md](rfc-1-ai-tool-evaluation.md) |

---

## Context

The duty engineer changes every day, so whoever is on duty meets today's problem cold. About 90% of real DE issues are networking, where the symptom sits several hops from the cause, and the long-tail cases end up escalated to Starforge — costing time on both sides. RFC-1 settled the orchestrator as LibreChat backed by an MCP server, but deliberately left the server itself open, because LibreChat only supplies the interface and the persistence while the MCP server decides what the LLM can actually see of the cluster. That question has now been answered by a trial: 13 DEs, 3 scenarios each, against a catalogue of 14 injected faults spanning Argo sync failures, broken workloads and Istio mesh configuration. Each DE ran their first scenario without the tool and the next two with it, which gives a measured comparison rather than an estimated one.

## Decision

**Adopt LibreChat backed by Kubernetes MCP server. It can read objects in the cluster except Secrets — Istio and Argo included — so a fault can be followed wherever it leads.**

- **Every DE is supportive of it.** Asked how willing they would be to use it on their next duty rotation, **13 of 13 answered 5/5** — the top of the scale
- **It never sent a DE down the wrong path.** Across 26 runs it never named a cause that was the wrong root cause of a injected problem.
- **It helps the DEs who actually need help (usually less experienced DEs).** After each case, every DE was asked whether they could have solved it without the tool. Less-experienced DEs said **no in 14 of their 18 cases** and those are 14 escalations to Starforge that no longer need to happen.
- **Kubernetes MCP over k8sgpt.** k8sgpt ships no Istio analyzers, so every Istio scenario is invisible to it, and its fixed analyzers cannot chase a lead out of the namespace they found it in. The Kubernetes MCP server reads the whole API surface, which is what makes the multi-hop networking cases solvable.
- **The tool cannot write to the cluster.** LibreChat holds no cluster credential; the MCP server is the sole component with cluster access, bound to one read-only ClusterRole with Secrets excluded.

## Consequences

### Positive:

- **It saves time on time-consuming and challenging problems.** On every scenario that took 10+ minutes without AI, the tool cut **27–68%** off the time to root cause & fix.
- **It lifts a less-experienced DE to the level of a more-experienced DE (Starforge).** On the hardest injected problem scenario, a SF personnel solving without AI took 10 mins, whereas two less-experienced DEs using the AI tool took 4 and 6 mins respectively.
- **It solves the problems a DE would otherwise escalate to Starforge.** In three scenarios the DE ran out of time and never found the cause, whereas other DE using the AI tool found the root cause in ALL three scenarios.
- **Overtime, DEs will be more familiar and proficient with troubleshooting the cluster.** Every AI answer shows how it derived at the answer — the objects it read, the field that was wrong, and the fix. This allows the DE to see how the conclusion was reached and not just the conclusion. **Furthermore, this can be seen in a survey conducted as All DEs said it helps them quickly learn the mechanism of diff apps and troubleshoot them faster (assuming they do not outsource their understanding)**

### Trade-offs we accept:
- **The AI tool is blind to git.** When asked about configuration whose source lives in git, it does not know the repository structure and the exact details of the yaml files on gitlab.
- **Deskilling, raised by the DEs themselves.** Two of the four sub-5 learning scores were this concern, not a complaint about accuracy as engineers may skip building their own troubleshooting instinct and go straight to the tool.

### What this triggers:
**If adopted:**

- service design, installation, operations docs