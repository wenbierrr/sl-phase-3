# DE trial execution

## Table of contents

1. [How the DE trial is conducted](#how-the-de-trial-is-conducted) — how the trial is run
2. [DE trial rubric](#de-trial-rubric) — how each case is judged
3. [DE trial scorecard](#de-trial-scorecard) — what the DE fills in
4. [The pass bar](#the-pass-bar) — what the whole thing has to hit to pass

## How the DE trial is conducted

| Failure type | Scenarios | DEs each | Scorecards |
|---|---|---|---|
| Argo sync failure | 2 | 2 (Non-SF) | **4** |
| Argo synced, but workload down | 2 | 2 (Non-SF) | **4** |
| Kubernetes-native networking | 2 | 2 (Non-SF) | **4** |
| Istio — configure the ingress gateway to route traffic to Keycloak for authentication | 1 | 3 (Non-SF) | **3** |
| Istio — mesh networking fault | 1 | 2 (SF) | **2** |
| | **8** | | **17** |

**The 8 test cases**

| # | Type | Scenario | Tested on |
|---|---|---|---|
| 1 | Argo sync failure | `values.yaml` file path wrong | 2 Non-SF |
| 2 | Argo sync failure | Job auto-deletes after finishing → Argo still expects it → OutOfSync forever but nothing actually broken | 2 Non-SF |
| 3 | Workload down | Deployment pins `runAsUser`/`fsGroup` → OpenShift SCC rejects it | 2 Non-SF |
| 4 | Workload down | Liveness probe targets the wrong port → kubelet kills the container | 2 Non-SF |
| 5 | K8s networking | `default-deny-ingress` NetworkPolicy → connection times out | 2 Non-SF |
| 6 | K8s networking | Service selector doesn't match the pod labels → no endpoints → connection refused | 2 Non-SF |
| 7 | Istio config task | Configure the ingress gateway to route traffic to Keycloak | 3 Non-SF |
| 8 | Istio complex | mTLS enforced on an app (`PeerAuthentication: STRICT`), but a `DestinationRule` sets `tls.mode: DISABLE` for the same host → caller sends plaintext, server demands mTLS | 2 SF |



## DE trial rubric

Question to answer: **can the DE take the tool's answer and act on it, so that it can save time and unecessary escalations?**


| Criteria | Measures |
|---|---|
| **Resolution quality** | Was the answer correct and accurate? Did the DE have to constantly guide the troubleshooting? |
| **Time saved** | How much of the DE's time the tool saved? |
| **Actionability** | Could the DE act on the output as it is and prevent uneccessary escalations? |


Those three become the criteria. Each is graded 1–3, so a case records not just *whether* the tool fell short but *where*:

| Criteria | 1: No Go | 2: OK | 3: Good |
|---|---|---|---|
| **Resolution quality** | • Hallucination <br/> • Output by AI is wrong  | • Output by AI is correct but DE had to guide it after stating the task | • Output by AI is correct and DE don't need to guide it after stating the task  |
| **Time saved** | Saved no time (**0%**); the DE would have been just as fast alone | Saved some time (**0% < x ≤ 30%**) using it | Saved **> 30%** of the time using it |
| **Actionability** | • AI's output is useless <br/>• The case still needs an escalation | • AI's output is on the right track, but the DE has to edit it before using it<br/>• Closed without escalation | • AI's output can be used exactly as given, without editing <br/>• Closed without escalation |

## DE trial scorecard

One per case, filled in by the DE as soon as the case closes.

**Case:** `___`;  **DE:** `___`;  **Team:** ○ SF ○ Non-SF; **Sub-area:** ○ 1.1.1 ○ 1.1.2 ○ 1.1.3

| | |
|---|---|
| **1. Resolution quality** | ○ **1** — Useless <br/>○ **2** — Pointed me in the right direction, but I had to prompt further even AFTER I stated the task<br/>○ **3** — Exactly right, no prompt needed after stating the task |
| **2. Any Hallucination?** | ○ Yes  ○ No  |
| **3. Time saved** | ○ **1** — None, I'd have been just as fast alone<br/>○ **2** — Some, but less than 30%<br/>○ **3** — 30% or more |
| **4. Actionability** | ○ **1** — AI's output is unusable, or would make it worse<br/>○ **2** — Right track, but I had to edit AI's output before using it<br/>○ **3** — Used exactly as given by AI |
| **5. Without the tool, would you have escalated to Starforge?** | ○ Yes  ○ No |

## The pass bar

Scored across all **17** scorecards.

| Criteria | Across all 17 |
|---|---|
| **Resolution quality** | **≥ 12 Good** (71%), remainder OK, NO hallucinations |
| **Actionability** | **≥ 12 Good** (71%), remainder OK |
| **Time saved** | **≥ 12 Good** (71%) |

**For each type of failure, what is the min number of "Good" required?**

| Failure type | Cases | Good required |
|---|---|---|
| Argo sync failure | 4 | **≥ 3** |
| Argo synced, but workload down | 4 | **≥ 3** |
| Kubernetes-native networking | 4 | **≥ 3** |
| Istio — ingress gateway to Keycloak | 3 | **≥ 2** |
| Istio — complex mesh networking fault | 2 | **≥ 1** |

