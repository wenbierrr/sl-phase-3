# DE trial execution

## Table of contents

1. [How the DE trial is conducted](#how-the-de-trial-is-conducted) — how the trial is run
   - [Who takes part](#who-takes-part)
   - [The session](#the-session-35-40-mins-timebox)
   - [How scenarios are allocated](#how-scenarios-are-allocated)
   - [The roster — who runs what](#the-roster--who-runs-what)
   - [Argo — 4 scenarios](#argo--4-scenarios)
   - [k8s-native — 5 scenarios](#k8s-native--5-scenarios)
   - [Istio — 5 scenarios](#istio--5-scenarios)
2. [What the trial has to answer with regards to the AI tool](#what-the-trial-has-to-answer-with-regards-to-the-ai-tool) — the hypothesis, and the figure behind 
3. [Facilitator record](#facilitator-record) — what the author records
4. [DE trial scorecard](#de-trial-scorecard) — what the DE fills in
5. [DE trial rubric](#de-trial-rubric) — how each case is judged
6. [The pass bar](#the-pass-bar) — what the whole thing has to hit to pass

## How the DE trial is conducted

### Who takes part

13 DEs — approximately **10 non-SF, 3 SF**. 

**14 scenarios, so DEs cannot really share answers.**

| Problem Family | Covers | Scenarios | Attempts |
|---|---|---|---|
| Argo | class 1 — always OutOfSync, un-synced | 4 | 13 |
| k8s-native | class 2 (workload won't spin up) + class 3 (pod up, app broken) | 5 | 13 |
| Istio | class 4 | 5 | 13 |
| | | **14** | **39** |

### The session (35-40 mins timebox)

| Step | Min |
|---|---|
| Brief | 5 |
| **Scenario 1 — without AI** | **15** |
| **Scenario 2 — with AI** | **10** |
| **Scenario 3 — with AI** | **10** |


### How scenarios are allocated

No non-SF DE attempts Istio without AI, so every Istio scenario (without AI) falls to an SF personel.

| | Without AI (15 min) | With AI (10 min) | Total |
|---|---|---|---|
| Argo | 5 (non-SF) | 8 | 13 |
| k8s | 5 (non-SF) | 8 | 13 |
| Istio | **3 (SF only)** | 10 | 13 |
| | **13** | **26** | **39** |

The 13 DEs fall into three groups, by which family of problem they take without AI:

| Group | DEs | Without AI | With AI | With AI |
|---|---|---|---|---|
| SF | 3 | Istio | Argo | k8s |
| Non-SF, group A | 5 | Argo | k8s | Istio |
| Non-SF, group B | 5 | k8s | Argo | Istio |

All 10 non-SF DEs get an AI-assisted Istio run. As a result, the comparison for Istio problems is **SF (without AI) vs Non-SF (with AI)** — the claim the whole benefit case rests on: *a less-experienced DE with the tool performs like an experienced one without it.*

**That comparison is deliberately conservative.** The less-experienced DE has to overcome the experience gap before any saving appears, so a saving that survives is hard to argue with.

**It also has a hard ceiling. 3 SF DEs × 1 without-AI run each = 3 SF baselines**, and all three must be Istio, because rule 3 means Istio gets no baseline any other way. So only three scenarios can ever carry the claim. The roster below concentrates the non-SF AI-assisted Istio runs onto exactly those three, which is what raises the count.


### The roster — who runs what

Ordered by day, with the three SF DEs on Mon, Tue and Wed so each Istio baseline is set before most of the AI-assisted runs of the same scenario.

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

**The week contains exactly one same-day overlap: DE-10 and DE-11 both meet A2 on Friday.** That is the minimum available — six DEs carry A2 across five days, so by pigeonhole one day must hold two.

#### What this roster buys

**8 direct leveling-up comparisons**, up from 6. Each is a non-SF DE, with AI, on a scenario whose baseline is an SF DE without AI:

| Scenario | SF baseline (no AI) | Non-SF runs with AI | Direct comparisons |
|---|---|---|---|
| **I3** | DE-01 | DE-04, DE-06, DE-09 | 3 |
| **I4** | DE-02 | DE-05, DE-07, DE-10 | 3 |
| **I5** | DE-03 | DE-08, DE-11 | 2 |
| | | | **8** |



#### Rules — adhered and not

| | Rule | Status |
|---|---|---|
| **1** | Minimal overlaps | ✅ **Adhered, and provably minimal.** One same-day overlap (DE-10 / DE-11 on A2, Friday). Cannot be reduced without lowering A2's four AI-assisted runs |
| **2** | Every DE does 3 runs — 1 without AI (15 min), 2 with AI (10 min) | ✅ **Adhered.** 13 × 1 = 13 without AI, 13 × 2 = 26 with AI, 39 total. Every DE also still takes one scenario per family, and no DE meets the same scenario twice |
| **3** | No non-SF does Istio without AI | ✅ **Adhered.** All three Istio baselines are SF — DE-01 (I3), DE-02 (I4), DE-03 (I5). No non-SF appears in an Istio without-AI slot |

**Two things this roster does *not* achieve, both structural rather than oversights:**

1. **The 6 SF AI-assisted runs stay confounded.** Each SF DE owes two AI-assisted runs, and under rule 3 those must be Argo and k8s — families where every baseline is non-SF. So DE-01 (A1, K3), DE-02 (A3, K5) and DE-03 (A4, K6) are all *SF-with-AI measured against a non-SF baseline*, where experience and tool push the same way and inflate the saving. **Exclude these 6 from the time-saved computation** — see [How Time saved is worked out](#how-time-saved-is-worked-out). They still count for resolution quality, actionability, hallucination and every DE-scored field.
2. **Leveling-up cannot reach Argo or k8s.** It needs an SF baseline, all three are spent on Istio, and moving one would leave an Istio scenario with no baseline at all. Argo and k8s can therefore only answer the within-level question — *does the tool make the same population faster?* **Do not present an Argo or k8s result as evidence that AI lifts a less-experienced DE to expert level.**

**The cost of the change:** I1 and I2 drop from 2 AI-assisted runs to 1 each. They keep their anti-sharing role and still yield resolution-quality, actionability and hallucination data — they just contribute less to the timing figures, which they were never able to ground anyway.


### Argo — 4 scenarios

| ID | Presents as | Injected fault | number of ppl who will do it without AI / with AI |
|---|---|---|---|
| **A1** | Never syncs; `ComparisonError` | `valueFiles` path under `$values` names a file that does not exist | 1 / 1 |
| **A2** | Permanently OutOfSync, **every pod healthy** | A `Job` with `ttlSecondsAfterFinished` deletes itself on completion; Argo keeps comparing it as drift | 2 / 4 |
| **A3** | Never syncs; revision won't resolve | `targetRevision` names a branch that does not exist | 1 / 1 |
| **A4** | Sync stuck; Argo says only **"hook failed"** | A PreSync `Job` that must run before the deploy fails — the actual reason exists only in that Job's pod log | 1 / 2 |



### k8s-native — 5 scenarios

| ID | Class | Presents as | Injected fault | number of ppl who will do it without AI / with AI |
|---|---|---|---|---|
| **K1** | 2 | Pod Running but **never Ready**; rollout never completes, no restarts, nothing in the logs | Readiness probe targets a port the container does not serve | 1 / 1 |
| **K7** | 2 | Pod stuck Pending; `persistentvolumeclaim not found` | The overlay sets `persistence.enabled: false` while the Deployment template mounts the volume unconditionally, so the PVC is never created | 1 / 1 |
| **K3** | 3 | Pods healthy, page blank / 503 | The Route exposes the **wrong Service** — `spec.to.name` points at a Service fronting a different app | 1 / 2 |
| **K5** | 3 | Connection refused — **but the Service has endpoints and the pod is Ready** | Service `targetPort` names a port the container does not expose | 1 / 2 |
| **K6** | 3 | Callers in one namespace work, callers in another are refused | An allow-NetworkPolicy looks correct but omits `namespaceSelector`, so it only matches same-namespace pods | 1 / 2 |


### Istio — 5 scenarios

| ID | Presents as | Injected fault | number of ppl who will do it without AI / with AI |
|---|---|---|---|
| **I1** | Connection reset between two services | Sidecar not injected (namespace label missing) against `PeerAuthentication: STRICT` | 0 / 1 |
| **I2** | RBAC: access denied | `AuthorizationPolicy` denies the caller | 0 / 1 |
| **I3** | 503 from the mesh | `VirtualService` routes to a subset the `DestinationRule` never defines | 1 / 3 |
| **I4** | Caller fails; **a correct-looking AuthorizationPolicy sits in the path as a decoy** | Three-namespace `exportTo` visibility fault; the symptom is two hops from the cause | 1 / 3 |
| **I5** | Connection fails; **both configs are individually correct** | `PeerAuthentication: STRICT` on the server, `DestinationRule` sets `tls.mode: DISABLE` for the same host | 1 / 2 |

**I3, I4 and I5 carry the leveling-up claim** — each has an SF baseline, and its AI-assisted runs are all non-SF. **I1 and I2 have no baseline of their own** and are scored on resolution quality, actionability and hallucination only.




## What the trial has to answer with regards to the AI tool


| Hypothesis (It refers to AI) | Evidenced by | From | Example figure it produces | Data points |
|---|---|---|---|---|
| It reaches the right answer | Root cause checked against answer | Facilitator | "X correct, Y wrong" | 26 AI cases |
| **It does not hallucinate** | Anything it stated that was not true | Facilitator | "1 of 26 cases hallucinated" | 26 AI cases |
| It makes DEs faster | Time to root cause, with AI VS without AI| Facilitator | Calculated in terms of % — see [How Time saved is worked out](#how-time-saved-is-worked-out) | **20** with AI vs 13 without (6 SF AI runs excluded) |
| It finds root causes the DE otherwise misses | Was the root cause found inside the timebox | Facilitator | "21 of 26 (81%) with AI, 6 of 13 (46%) without" | 26 with AI vs 13 without |
| **It solves what the DE could not have solved alone** | "Without AI, I wouldn't have been able to solve this" | DE | "14 of 26 (54%) answered yes, 12 of them non-SF" | 26 AI cases |
| Its answer is right without having to be guided there | Resolution quality, 1–5 | DE | "19 of 26 ans 5/5" | 26 AI cases |
| Its output can be used as it stands | Actionability, 1–5 | DE | "19 of 26 ans 5/5" | 26 AI cases |
| DEs expect it to level them up — to learn JPE apps faster | Learning *(forecast)*, 1–5 | DE | "5 of 13 ans 5/5" | 13 — one per DE |
| DEs would actually use it | Willingness to use next rotation, 1–5 | DE | "6 of 13 ans 5/5" | 13 — one per DE |


## Facilitator record

| Field | Recorded as |
|---|---|
| Scenario ID | A1 … I5 |
| DE · Team | 01–13 · SF / Non-SF |
| Mode | without AI / with AI |
| Resolved? | Y / N |
| Time to root cause | minutes, or *unresolved* |
| Did AI Hallucinate? | Y / N |

## DE trial scorecard

One form per DE, filled once at the end, ~2 minutes.

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



## DE trial rubric

Five criteria, each graded 1–5.

| Criteria | 1: No Go | 2: Poor | 3: OK | 4: Good | 5: Excellent | n |
|---|---|---|---|---|---|---|
| **Resolution quality** | Wrong or made up — named a cause that was not the cause, or said something untrue about the cluster | Nothing untrue, but too vague to act on; the DE got there themselves | Right direction, but only after repeated prompting | Correct, after more than one prompt | Correct after a single prompt — the one stating what is wrong | 26 |
| **Time saved** *(computed)* | No saving, or slower | >0–25% | >25–50% | >50–75% | >75%, or resolved a case that went unresolved without AI | 20 |
| **Actionability** | Unusable, or following it would have made things worse | Pointed at the right area but gave no fix the DE could use | Gave a fix the DE had to substantially rewrite | Gave a fix needing minor edits | Gave a fix the DE applied exactly as written | 26 |
| **Learning** *(forecast)* | No — it definitely will not help me learn JPE apps any faster | Unlikely — marginal at best; I would still learn them the same way | Possibly — faster on some parts of JPE, not on others | Likely — it would speed up how quickly I get to grips with JPE apps | Yes — it definitely will help me learn JPE apps, and troubleshoot them, faster | 13 |
| **Willingness to use** | Would not use it | Would use it reluctantly | Would use it occasionally | Would use it on most rotations | Would use it every rotation | 13 |


### How Time saved is worked out

*Worked example — scenario K5:*

| Who | Mode | Time | Saving | Grade |
|---|---|---|---|---|
| DE-12 | without AI | 12 min | — | this is K5's baseline |
| DE-02 | with AI | 5 min | (12−5) ÷ 12 = **58%** | 4 |
| DE-07 | with AI | 7 min | (12−7) ÷ 12 = **42%** | 3 |

Three cases to note:

- **I1 and I2 are never run without AI**, since there will only be 3 SF personel. Their single AI-assisted run each is **not given a time-saved grade** — the old fallback of averaging the other Istio scenarios borrowed a baseline from a different fault, which is not a measurement. Score them on resolution quality, actionability and hallucination only.
- **The 6 SF AI-assisted runs are excluded** — DE-01 (A1, K3), DE-02 (A3, K5), DE-03 (A4, K6). Each is an SF DE with AI measured against a non-SF baseline, so experience and tool push the same way and the saving is overstated. **Time saved is therefore reported on n = 20, not 26.** State the n alongside the figure.
- **If the DE without AI ran out of time**, it means time to resolution will be >15 min, and we will assume a time of 30 mins considering how the DE will need to escalate. **That 30 is an assumption, not a measurement** — the only observed fact is "longer than 15 minutes", and 30 inflates the computed saving. Label it as an assumption wherever quoted, and report the saving against the observed 15-minute floor as well.

**The 8 comparisons that carry the benefit case** are the non-SF AI-assisted runs on I3, I4 and I5, each measured against that scenario's SF baseline. Report these separately from the within-level Argo and k8s figures — they answer a different question, and it is the one the supervisor cares about.

## The pass bar

To be agreed

