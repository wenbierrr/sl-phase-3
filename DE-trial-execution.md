# DE trial execution

## Table of contents

1. [How the DE trial is conducted](#how-the-de-trial-is-conducted) — how the trial is run
   - [Who takes part](#who-takes-part)
   - [The session](#the-session)
   - [How scenarios are allocated](#how-scenarios-are-allocated)
   - [The roster — who runs what](#the-roster--who-runs-what)
   - [Argo — 4 scenarios](#argo--4-scenarios)
   - [k8s-native — 5 scenarios](#k8s-native--5-scenarios)
   - [Istio — 5 scenarios](#istio--5-scenarios)
2. [What the trial has to produce](#what-the-trial-has-to-produce) — the claims, and the figure behind each
3. [Facilitator record](#facilitator-record) — what the author records
4. [DE trial scorecard](#de-trial-scorecard) — what the DE fills in
5. [DE trial rubric](#de-trial-rubric) — how each case is judged
6. [The pass bar](#the-pass-bar) — what the whole thing has to hit to pass

## How the DE trial is conducted

### Who takes part

13 DEs — approximately **10 non-SF, 3 SF**. Each runs **3 scenarios, one per family**, in a 35-minute box.

**14 scenarios, so DEs cannot really share answers.**

| Family | Covers | Scenarios | Attempts |
|---|---|---|---|
| Argo | class 1 — always OutOfSync, un-synced | 4 | 13 |
| k8s-native | class 2 (workload won't spin up) + class 3 (pod up, app broken) | 5 | 13 |
| Istio | class 4 | 5 | 13 |
| | | **14** | **39** |

### The session

| Step | Min |
|---|---|
| Brief | 5 |
| **Scenario 1 — unaided** | **15** |
| **Scenario 2 — with AI** | **10** |
| **Scenario 3 — with AI** | **10** |
| Scorecard, covering all three | 5 |

Each scenario is recorded simply as **resolved or not** — one flag for the unaided scenario, one for each AI scenario.

### How scenarios are allocated

No non-SF DE attempts Istio unaided. Only the first scenario is unaided, so every unaided Istio run falls to an SF DE.

| | Unaided (15 min) | With AI (10 min) | Total |
|---|---|---|---|
| Argo | 5 (non-SF) | 8 | 13 |
| k8s | 5 (non-SF) | 8 | 13 |
| Istio | **3 (SF only)** | 10 | 13 |
| | **13** | **26** | **39** |

The 13 DEs fall into three groups, by which family they take unaided:

| Group | DEs | Unaided | With AI | With AI |
|---|---|---|---|---|
| SF | 3 | Istio | Argo | k8s |
| Non-SF, group A | 5 | Argo | k8s | Istio |
| Non-SF, group B | 5 | k8s | Argo | Istio |

All 10 non-SF DEs get an AI-assisted Istio run. **20 of the 26 AI scorecards come from non-SF DEs.**

The Istio baseline is set by SF unaided and compared against non-SF with AI, so it is not like-for-like.

### The roster — who runs what

Ordered by day, with the three SF DEs on Tue, Wed and Fri.

| Day | DE | Team | **1 — unaided** (15 min) | **2 — with AI** (10 min) | **3 — with AI** (10 min) |
|---|---|---|---|---|---|
| **Mon** | 05 | Non-SF | **A2** | K7 | I1 |
| | 07 | Non-SF | **A3** | K5 | I2 |
| | 11 | Non-SF | **K3** | A2 | I4 |
| **Tue** | 01 | SF | **I3** | A2 | K3 |
| | 04 | Non-SF | **A1** | K1 | I1 |
| | 12 | Non-SF | **K5** | A3 | I5 |
| **Wed** | 02 | SF | **I4** | A2 | K5 |
| | 09 | Non-SF | **K1** | A1 | I3 |
| | 13 | Non-SF | **K6** | A4 | I5 |
| **Thu** | 06 | Non-SF | **A2** | K3 | I2 |
| | 08 | Non-SF | **A4** | K6 | I3 |
| **Fri** | 03 | SF | **I5** | A4 | K6 |
| | 10 | Non-SF | **K7** | A2 | I4 |

**The week contains exactly one same-day overlap: DE-05 and DE-11 both meet A2 on Monday.** Every other pair sharing a day has no scenario in common.

One collision is unavoidable rather than sloppy: A2 is run by six DEs across five days, so two of them must land together. This is the least damaging version of it — the two DEs share only that one scenario, and they meet it in different modes (05 unaided, 11 with AI), so neither is grading the same thing. Run them at opposite ends of Monday.

**Day loading is 3 / 3 / 3 / 2 / 2.** If the SF days move, the overlap count moves with them and the roster needs re-solving rather than nudging.

Attempt counts in the catalogues below are **(unaided / with AI)** and follow directly from this roster.

### Argo — 4 scenarios

| ID | Presents as | Injected fault | What it tests | u / AI |
|---|---|---|---|---|
| **A1** | Never syncs; `ComparisonError` | `valueFiles` path under `$values` names a file that does not exist | Reading the Application's own error | 1 / 1 |
| **A2** | Permanently OutOfSync, **every pod healthy** | A `Job` with `ttlSecondsAfterFinished` deletes itself on completion; Argo keeps comparing it as drift | **The correct answer is "nothing is broken"** — catches inventing a fault to explain a red app | 2 / 4 |
| **A3** | Never syncs; revision won't resolve | `targetRevision` names a branch that does not exist | Argo resolves git before it touches the cluster | 1 / 1 |
| **A4** | Sync stuck; Argo says only **"hook failed"** | A PreSync `Job` that must run before the deploy fails — the actual reason exists only in that Job's pod log | Forces the DE off the Argo UI and into the cluster; the error text on Argo is a pointer, not an explanation | 1 / 2 |

A2 is **tc9**, written and never run.

**Reserves:** the AppProject forbidding the Application's destination; and a manifest deleted from git that stays in the cluster because prune is off.

### k8s-native — 5 scenarios

| ID | Class | Presents as | Injected fault | What it tests | u / AI |
|---|---|---|---|---|---|
| **K1** | 2 | Pod Running but **never Ready**; rollout never completes, no restarts, nothing in the logs | Readiness probe targets a port the container does not serve | A pod that looks fine and is not — no crash to follow, so the probe definition is the only lead | 1 / 1 |
| **K7** | 2 | Pod stuck Pending; `persistentvolumeclaim not found` | The overlay sets `persistence.enabled: false` while the Deployment template mounts the volume unconditionally, so the PVC is never created | Tracing a runtime symptom back through the template to the values flag that drives it | 1 / 1 |
| **K3** | 3 | Pods healthy, page blank / 503 | The Route exposes the **wrong Service** — `spec.to.name` points at a Service fronting a different app | The fault is above the pod layer: everything the DE checks first is green | 1 / 2 |
| **K5** | 3 | Connection refused — **but the Service has endpoints and the pod is Ready** | Service `targetPort` names a port the container does not expose | **Kills "endpoints exist, so the Service is fine"** | 1 / 2 |
| **K6** | 3 | Callers in one namespace work, callers in another are refused | An allow-NetworkPolicy looks correct but omits `namespaceSelector`, so it only matches same-namespace pods | NetworkPolicies are additive; a rule that looks allowed may not be | 1 / 2 |

Every k8s scenario carries its own unaided baseline.

**Reserves:** the Service selector matching no pods (**tc3**); a PVC naming a `storageClassName` that does not exist; **K2**, a PVC that binds but never attaches (held back because a single-node CRC may not reproduce it cleanly); a second Deployment whose pods match the Service selector, splitting traffic so failures are **intermittent**; and a ConfigMap updated in git but consumed as env vars, so running pods hold the old value.

### Istio — 5 scenarios

| ID | Presents as | Injected fault | What it tests | u / AI |
|---|---|---|---|---|
| **I1** | Connection reset between two services | Sidecar not injected (namespace label missing) against `PeerAuthentication: STRICT` | Baseline mesh case | 0 / 2 |
| **I2** | RBAC: access denied | `AuthorizationPolicy` denies the caller | Reading mesh policy, not just kube objects | 0 / 2 |
| **I3** | 503 from the mesh | `VirtualService` routes to a subset the `DestinationRule` never defines | The VS → DR chain | 1 / 2 |
| **I4** | Caller fails; **a correct-looking AuthorizationPolicy sits in the path as a decoy** | Three-namespace `exportTo` visibility fault; the symptom is two hops from the cause | Punishes stopping at the first plausible culprit | 1 / 2 |
| **I5** | Connection fails; **both configs are individually correct** | `PeerAuthentication: STRICT` on the server, `DestinationRule` sets `tls.mode: DISABLE` for the same host | Two valid objects that are invalid together — no single object is wrong | 1 / 2 |

I1–I4 are **tc5–tc8**. The three unaided slots land on I3, I4 and I5, so the specialist baseline sits where it is most informative.

**Reserve:** a `VirtualService` whose catch-all `match` is ordered before the specific route, so the specific rule never fires.



## What the trial has to produce

Each row runs the whole way across: the claim, the question that evidences it, who can answer that question, and the figure that comes out at the end.

| Claim to be defended | Evidenced by | From | The figure it produces |
|---|---|---|---|
| It reaches the right answer | Root cause checked against the sealed key | Facilitator | "19 correct, 5 partial, 2 wrong" |
| **It does not hallucinate** | Anything it stated that was not true | Facilitator | "1 of 26 cases" |
| It makes DEs faster | Time to root cause, aided against unaided | Facilitator | Calculated — see [How Time saved is worked out](#how-time-saved-is-worked-out) |
| It finds root causes the DE otherwise misses | Was the root cause found inside the timebox | Facilitator | "21 of 26 (81%) with AI, 6 of 13 (46%) without" |
| **It solves what the DE could not have solved alone** | "Without AI, I wouldn't have been able to solve this" | DE | "14 of 26 (54%) answered yes, 12 of them non-SF" |
| Its answer is right without having to be dragged there | Resolution quality, 1–5 | DE | "19 of 26 at 4 or 5" |
| Its output can be used as it stands | Actionability, 1–5 | DE | "20 of 26 at 4 or 5" |
| It levels DEs up — they came away knowing more | "Did it help you learn things you didn't know?" 0–10 | DE | Graded 1–5 — "10 of 13 at 4 or 5" |
| DEs would actually use it | Willingness to use next rotation, 0–10 | DE | Graded 1–5 — "11 of 13 at 4 or 5" |
| Where it does not help | "Where did it fall short?" | DE | Recurring themes |

**Every row is also split SF against non-SF**, since the value claim rests on the non-specialists.

Nothing is collected that is not on this table, and the three kinds of figure carry different weight:

- **Counts** — correct against the key, hallucinations, couldn't-have-solved-alone, root causes found aided against unaided. Nothing has to be known beforehand to make sense of a count, so these are the strongest evidence here. Lead with them.
- **The measured time comparison** — real data rather than opinion, but the aided and unaided runs are different people and the numbers are small.
- **The self-reported grades** — learning and willingness are what DEs *say*. Report them; do not rest the case on them.

## Facilitator record

One row per scenario, filled by the author while observing. Correctness and hallucination both need the sealed key, so neither can be asked of the DE.

| Field | Recorded as |
|---|---|
| Scenario ID | A1 … I5 |
| DE · Team | 01–13 · SF / Non-SF |
| Mode | unaided / with AI |
| Resolved? | Y / N |
| Time to root cause | minutes, or *unresolved* |
| Root cause vs key | correct / partial / wrong |
| Hallucination? | Y / N |

## DE trial scorecard

One form per DE, filled once at the end, ~5 minutes.

**DE:** `___`;  **Team:** ○ SF ○ Non-SF

**Part A — one column per scenario**

| | S1 (no AI) | S2 (AI) | S3 (AI) |
|---|---|---|---|
| Scenario ID | | | |
| Did you find the root cause in the time? | Y / N | Y / N | Y / N |
| **Without AI, I wouldn't have been able to solve this** | — | Y / N | Y / N |
| **Resolution quality** (1–5)<br/>**1** wrong or made up · **2** vague, I got there myself · **3** right only after repeated prompting · **4** correct after one or two prompts · **5** correct first time | — | | |
| **Actionability** (1–5)<br/>**1** unusable · **2** right area, no usable fix · **3** fix needed substantial rewriting · **4** fix needed minor edits · **5** fix used exactly as given | — | | |

**Part B — about the AI overall**

| | |
|---|---|
| Did the AI tool help you **learn things you didn't know**? | `0` not at all — `10` a great deal |
| How willing would you be to use this on your **next duty rotation**? | `0` not at all — `10` definitely |
| **Where did it fall short?** | *(free text)* |

Two things are deliberately absent. **Time saved** is not asked, because the unaided first scenario means it can be computed instead of guessed. **Hallucination** is not asked, because spotting it needs the answer key, so it sits on the facilitator record.

## DE trial rubric

Five criteria graded 1–5. The two 0–10 answers reach 1–5 by banding: `0–2 → 1 · 3–4 → 2 · 5–6 → 3 · 7–8 → 4 · 9–10 → 5`.

| Criteria | 1: No Go | 2: Poor | 3: OK | 4: Good | 5: Excellent | n |
|---|---|---|---|---|---|---|
| **Resolution quality** | Wrong or made up — named a cause that was not the cause, or said something untrue about the cluster | Nothing untrue, but too vague to act on; the DE got there themselves | Right direction, but only after repeated prompting once the task had been stated | Correct, after one or two follow-up prompts | Correct on its first answer, no follow-up needed once the task was stated | 26 |
| **Time saved** *(computed)* | No saving, or slower — including any case left unresolved at 10 min | >0–25% | >25–50% | >50–75% | >75%, or resolved a case that went unresolved unaided | 26 |
| **Actionability** | Unusable, or following it would have made things worse | Pointed at the right area but gave no fix the DE could use | Gave a fix the DE had to substantially rewrite | Gave a fix needing minor edits | Gave a fix the DE applied exactly as written | 26 |
| **Learning** | Learned nothing I didn't already know | Picked up a minor detail | Learned something useful about this cluster | Learned something I will use on future cases | Learned something that changes how I would troubleshoot | 13 |
| **Willingness to use** | Would not use it | Would use it reluctantly | Would use it occasionally | Would use it on most rotations | Would use it every rotation | 13 |

A DE grading Resolution quality 5 on a case the facilitator marked *wrong* against the key is worth flagging separately: the DE was convinced by an answer that was not right.

### How Time saved is worked out

Every scenario is run **both ways** by different DEs. The unaided run is what the AI runs are measured against.

*Worked example — scenario K5:*

| Who | Mode | Time | Saving | Grade |
|---|---|---|---|---|
| DE-12 | unaided | 12 min | — | this is K5's baseline |
| DE-02 | with AI | 5 min | (12−5) ÷ 12 = **58%** | 4 |
| DE-07 | with AI | 7 min | (12−7) ÷ 12 = **42%** | 3 |

Two cases need a rule:

- **I1 and I2 have no unaided run** — there were 13 unaided slots for 14 scenarios, and Istio's three went to I3, I4 and I5. For these two, use the middle unaided time across the Istio scenarios as the baseline.
- **The unaided DE ran out of time** — baseline = 15 min.

## The pass bar

To be agreed before the trial starts.

