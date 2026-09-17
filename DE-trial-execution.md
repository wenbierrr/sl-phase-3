# DE trial execution

**Does the AI tool measurably help a duty engineer?** 13 DEs × 3 scenarios = 39 runs. Each DE solves one problem without AI before meeting two with the tool, so every comparison is measured, not estimated.

## Table of contents

1. [Design principles](#design-principles) — the rules the trial is built on
2. [What the trial has to answer](#what-the-trial-has-to-answer) — each hypothesis and the figure behind it
3. [How the trial is scored](#how-the-trial-is-scored)
4. [Appendix](#appendix) — test cases, roster, forms, full rubric, worked example

---

## Design principles

| | Principle | Why |
|---|---|---|
| **1** | **Minimal overlaps** — 14 scenarios, 13 DEs | To cover as many different test cases as possible|
| **2** | **3 runs per DE** — 1 without AI (15 min), 2 with AI (10 min) | The run without AI is the baseline its AI runs are measured against |
| **3** | **35–40 min per session** | Has to fit around a duty rotation |
| **4** | **No non-SF DE does Istio without AI** | Without AI they would spend the time learning Istio. It also puts every Istio baseline in SF hands, so Istio measures a less-experienced DE with the AI tool against an experienced one without it |
| **5** | **One scenario per problem family per DE** | Every DE will tackle problems from Argo, k8s-native and Istio |

13 DEs — 10 non-SF, 3 SF.

| Problem family | Covers | Scenarios | Runs |
|---|---|---|---|
| Argo | class 1 — always OutOfSync, un-synced | 4 | 13 |
| k8s-native | class 2 (workload won't spin up) + class 3 (pod up, app broken) | 5 | 13 |
| Istio | class 4 | 5 | 13 |
| | | **14** | **39** |

## What the trial has to answer

| Hypothesis (It = the AI) | Evidenced by |
|---|---|
| It reaches the right answer | Root cause checked against the answer key |
| **It does not hallucinate** | Anything it stated that was not true |
| It makes DEs faster | Time to root cause, with AI vs without |
| It finds root causes the DE otherwise misses | Root cause found inside the timebox |
| **It solves what the DE could not have solved alone** | "Without AI, I wouldn't have been able to solve this" |
| Its answer is right without being guided there | Resolution quality, 1–5 |
| Its output is usable as it stands | Actionability, 1–5 |
| It levels DEs up — they learn JPE apps faster | Learning *(forecast)*, 1–5 |
| DEs would actually use it | Willingness to use next rotation, 1–5 |

## How the trial is scored

Five criteria, 1–5. Descriptors in [Appendix D](#d--scoring-rubric), forms in [Appendix C](#c--facilitator-record-and-de-scorecard), Time saved computation in [Appendix E](#e--how-time-saved-is-worked-out).

| Criterion | Scored by |
|---|---|
| Resolution quality | DE |
| Actionability | DE |
| **Time saved** | computed |
| Learning *(forecast)* | DE |
| Willingness to use | DE |



---

# Appendix

## A · Test cases

### Argo — 4 scenarios

| ID | Presents as | Injected fault | without AI / with AI |
|---|---|---|---|
| **A1** | Never syncs; `ComparisonError` | `valueFiles` path under `$values` names a file that does not exist | 1 / 1 |
| **A2** | Permanently OutOfSync, **every pod healthy** | A `Job` with `ttlSecondsAfterFinished` deletes itself on completion; Argo keeps comparing it as drift | 2 / 4 |
| **A3** | Never syncs; revision won't resolve | `targetRevision` names a branch that does not exist | 1 / 1 |
| **A4** | Sync stuck; Argo says only **"hook failed"** | A PreSync `Job` that must run before the deploy fails — the actual reason exists only in that Job's pod log | 1 / 2 |

### k8s-native — 5 scenarios

| ID | Class | Presents as | Injected fault | without AI / with AI |
|---|---|---|---|---|
| **K1** | 2 | Pod Running but **never Ready**; rollout never completes, no restarts, nothing in the logs | Readiness probe targets a port the container does not serve | 1 / 1 |
| **K7** | 2 | Pod stuck Pending; `persistentvolumeclaim not found` | The overlay sets `persistence.enabled: false` while the Deployment template mounts the volume unconditionally, so the PVC is never created | 1 / 1 |
| **K3** | 3 | Pods healthy, page blank / 503 | The Route exposes the **wrong Service** — `spec.to.name` points at a Service fronting a different app | 1 / 2 |
| **K5** | 3 | Connection refused — **but the Service has endpoints and the pod is Ready** | Service `targetPort` names a port the container does not expose | 1 / 2 |
| **K6** | 3 | Callers in one namespace work, callers in another are refused | An allow-NetworkPolicy looks correct but omits `namespaceSelector`, so it only matches same-namespace pods | 1 / 2 |

### Istio — 5 scenarios

| ID | Presents as | Injected fault | without AI / with AI |
|---|---|---|---|
| **I1** | Connection reset between two services | Sidecar not injected (namespace label missing) against `PeerAuthentication: STRICT` | 0 / 1 |
| **I2** | RBAC: access denied | `AuthorizationPolicy` denies the caller | 0 / 1 |
| **I3** | 503 from the mesh | `VirtualService` routes to a subset the `DestinationRule` never defines | 1 / 3 |
| **I4** | Caller fails; **a correct-looking AuthorizationPolicy sits in the path as a decoy** | Three-namespace `exportTo` visibility fault; the symptom is two hops from the cause | 1 / 3 |
| **I5** | Connection fails; **both configs are individually correct** | `PeerAuthentication: STRICT` on the server, `DestinationRule` sets `tls.mode: DISABLE` for the same host | 1 / 2 |


## B · The roster

How the 39 runs are allocated:

| | Without AI (15 min) | With AI (10 min) | Total |
|---|---|---|---|
| Argo | 5 (non-SF) | 8 | 13 |
| k8s | 5 (non-SF) | 8 | 13 |
| Istio | **3 (SF only)** | 10 | 13 |
| | **13** | **26** | **39** |

The 13 DEs fall into three groups, by which family they take without AI:

| Group | DEs | Without AI | With AI | With AI |
|---|---|---|---|---|
| SF | 3 | Istio | Argo | k8s |
| Non-SF, group A | 5 | Argo | k8s | Istio |
| Non-SF, group B | 5 | k8s | Argo | Istio |


| Day | DE | Team | **1 — without AI** (15 min) | **2 — with AI** (10 min) | **3 — with AI** (10 min) |
|---|---|---|---|---|---|
| **Mon** | 01 (yong jiun) | SF | **I3** | A1 | K3 |
| | 05 (jeremy) | Non-SF | **A2** | K6 | I4 |
| | 13 (nicholas) | Non-SF | **K7** | A4 | I2 |
| **Tue** | 02 (favian) | SF | **I4** | A3 | K5 |
| | 04 (anthony) | Non-SF | **A1** | K1 | I3 |
| | 12 (claudia) | Non-SF | **K6** | A2 | I1 |
| **Wed** | 03 (zhi wen) | SF | **I5** | A4 | K6 |
| | 07 (melvin) | Non-SF | **A3** | K3 | I4 |
| | 09 (jia qing) | Non-SF | **K1** | A2 | I3 |
| **Thu** | 06 (clement) | Non-SF | **A2** | K7 | I3 |
| | 08 (jiong) | Non-SF | **A4** | k5 | I5 |
| **Fri** | 10 (jeng) | Non-SF | **K3** | A2 | I4 |
| | 11 (joash) | Non-SF | **K5** | A2 | I5 |


### The 8 levelling-up comparisons

Each is a non-SF DE, with AI, on a scenario whose baseline is an SF DE without AI:

| Scenario | SF baseline (no AI) | Non-SF runs with AI | Direct comparisons |
|---|---|---|---|
| **I3** | DE-01 | DE-04, DE-06, DE-09 | 3 |
| **I4** | DE-02 | DE-05, DE-07, DE-10 | 3 |
| **I5** | DE-03 | DE-08, DE-11 | 2 |
| | | | **8** |

## C · Facilitator record and DE scorecard

### What the facilitator records

| Field | Recorded as |
|---|---|
| Scenario ID | A1 … I5 |
| DE · Team | 01–13 · SF / Non-SF |
| Mode | without AI / with AI |
| Resolved? | Y / N |
| Time to root cause | minutes, or *unresolved* |
| Did AI Hallucinate? | Y / N |

### DE scorecard

One form per DE, filled once at the end.

**DE:** `___`;  **Team:** ○ SF ○ Non-SF

**Part A — one column per scenario**

| | S1 (no AI) | S2 (AI) | S3 (AI) |
|---|---|---|---|
| Scenario ID | | | |
| **Without AI, I wouldn't have been able to solve this** | — | Y / N | Y / N |
| **Resolution quality** (1–5)<br/>**1** wrong or made up · **5** correct after 1 prompt, the one telling it what's wrong | — | | |
| **Actionability** (1–5)<br/>**1** unusable · **5** fix used exactly as given | — | | |

**Part B — about the AI overall**

| | |
|---|---|
| **Learning** (1–5)<br/>With this tool, do you foresee it helping you **learn faster** with regards to apps within **JPE** (including troubleshooting if needed)? | **1** no, it definitely will not help me learn faster - **5** yes, it definitely will help me learn faster |
| **Willingness to use** (1–5)<br/>How willing would you be to use this on your **next duty rotation**? | **1** would not use it - **5** would use it every subsequent DE day  |
| **Any areas the AI fall short?** | *(free text)* |

## D · Scoring rubric

| Criteria | 1: No Go | 2: Poor | 3: OK | 4: Good | 5: Excellent |
|---|---|---|---|---|---|
| **Resolution quality** | Wrong or made up — named a cause that was not the cause, or said something untrue about the cluster | Nothing untrue, but too vague to act on; the DE got there themselves | Right direction, but only after repeated prompting | Correct, after more than one prompt | Correct after a single prompt — the one stating what is wrong |
| **Time saved** *(computed)* | No saving, or slower | >0–25% | >25–50% | >50–75% | >75%, or resolved a case that went unresolved without AI |
| **Actionability** | Unusable, or following it would have made things worse | Pointed at the right area but gave no fix the DE could use | Gave a fix the DE had to substantially rewrite | Gave a fix needing minor edits | Gave a fix the DE applied exactly as written |
| **Learning** *(forecast)* | No — it definitely will not help me learn JPE apps any faster | Unlikely — marginal at best; I would still learn them the same way | Possibly — faster on some parts of JPE, not on others | Likely — it would speed up how quickly I get to grips with JPE apps | Yes — it definitely will help me learn JPE apps, and troubleshoot them, faster |
| **Willingness to use** | Would not use it | Would use it reluctantly | Would use it occasionally | Would use it on most rotations | Would use it every rotation |

## E · How time saved is worked out

**Saving = (baseline − time with AI) ÷ baseline.** The baseline is the same scenario's run without AI.

| Who | Mode | Time | Saving | Grade |
|---|---|---|---|---|
| DE-12 | without AI | 12 min | — | K5's baseline |
| DE-02 | with AI | 5 min | (12−5) ÷ 12 = **58%** | 4 |
| DE-07 | with AI | 7 min | (12−7) ÷ 12 = **42%** | 3 |

**Two sets of runs get no time-saved grade.** Both still count for resolution quality, actionability and hallucination.

- **I1 and I2** — nobody ran them without AI, so there is no baseline.
- **The 6 SF runs with AI** — DE-01 (A1, K3), DE-02 (A3, K5), DE-03 (A4, K6). All six of those baselines are non-SF, so a faster time is due to the extra experience.

**If the DE without AI ran out of time**, no percentage is computed. The rubric grades it 5 — *"resolved a case that went unresolved without AI"*.
