# DE trial — results analysis

> **Question:** Does the AI tool measurably help a duty engineer? *(trial design: [DE-trial-modality.md](DE-trial-modality.md))*
>
> **Answer: Yes. It helps most on the hardest problems and for less-experienced DEs, which is where escalations to Starforge come from.**

13 DEs (10 non-SF, 3 SF) each solved one injected fault without AI (15-min timebox), then two with AI (10-min timebox): 39 runs across Argo, k8s-native and Istio. </br> This analysis is based on two spreadsheets: `DE Trial — Facilitator Record (Responses).xlsx`, the facilitator's record of every run (resolved, time to root cause, whether the AI said anything untrue), and `DE Trial — Scorecard (Responses).xlsx`, each DE's rating on the tool and comments.

---

## At a glance

| Hypothesis (it = the AI) | Result | Verdict |
|---|---|---|
| It reaches the right answer | 25 of 26 runs found the root cause | **Yes** |
| Its answer is right without being guided there | Resolution quality 5/5 in 23 of 26 runs; the other 3 scored 3/5 (right direction, after some prompting) | **Yes** |
| Its output is usable as it stands | Actionability 5/5 in 23 of 26; the rest 4/5 | **Yes** |
| It does not hallucinate | 1 untrue statement in 26 runs; never a wrong root cause | **Yes**, with one gap: it cannot see git |
| It finds root causes the DE otherwise misses | All 4 scenarios a DE couldn't solve without AI were solved with AI promptly | **Yes** |
| It solves what the DE could not have solved alone | Non-SF DEs said "could not have" on 16 of their 20 AI runs | **Yes** |
| It makes DEs faster | **Hard scenarios** (10+ min without AI): 27–68% faster<br>**Easy scenarios** (under 4 min without AI): no gain | **Yes, on hard problems only** |
| It levels DEs up | Non-SF DEs with AI matched or beat an SF engineer without AI on 5 of 8 Istio runs | **Yes** |
| DEs would actually use it | All 13 DEs rated their willingness to use it on their next duty rotation - 5/5 | **Yes** |

---

## Key Findings

### 1. Its weak runs came from the input, not its reasoning

**It never named a wrong root cause.** Its three weaker runs, all on Istio, came from a DE's false premise, config it cannot see in git, and an explanation the DE could not follow. Without AI, only 9 of 13 runs found the root cause, even with 5 more minutes. *([Appendix A](#a--accuracy))*

### 2. Its value is with less-experienced DEs, most of all on Istio

**Non-SF DEs said they could not have solved any of their 10 Istio runs without it.** On I4, the hardest scenario, an SF engineer took 10 minutes alone, while two non-SF DEs with the tool took 4 and 6. They also expect it to help them learn: the learning forecast averaged 4.54/5. *([Appendix B](#b--how-much-the-tool-helps-less-experienced-des))*

### 3. On hard problems, it gets a DE to the fix in minutes

**In 4 scenarios, a DE working alone ran out of time at 15 minutes. With AI, every run on those scenarios was solved: 10 of 10, in 1–8 minutes.** Where DEs did finish alone but needed 10+ minutes, the tool cut **27–68%** off the time on 6 of 7 runs; the seventh was the false-premise miss. On problems a DE already solves in under 4 minutes it adds nothing, and costs at most 2 minutes. *([Appendix C](#c--time-saved))*

---

## Where it fell short, and the fix

| Gap | What happened | Fix |
|---|---|---|
| **Accepts a wrong premise** | Told a non-existent pod was in CrashLoopBackOff, it searched for that pod until time ran out *(DE-07, DE-11)* | Guide DEs to describe symptoms, not a diagnosis |
| **Cannot see git** | Guessed the repo structure behind a Helm release *(DE-09)* | [System prompt](#f--system-prompt) §3: say only what it read; when the cluster cannot tell it something, ask instead of guessing |
| **Too verbose** | DEs had to scroll to find the fix *(DE-01, DE-12)* | [System prompt](#f--system-prompt) §4: Define a answer format (Root cause, Evidence in at most 3 bullets, Fix, Confidence)|
| **No before/after view** | A DE asked to see current and proposed config side by side *(DE-10)* | [System prompt](#f--system-prompt) §4: the fix is a complete YAML file with the changed lines marked |

Treat these results as indicative: the sample is small and most scores are self-reported ([Appendix E](#e--limits-of-this-trial)).

---

# Appendix

## A · Accuracy

| | Resolved | Unresolved | Rate |
|---|---|---|---|
| **Without AI** (15-min timebox) | 9 | 4 | 69% |
| **With AI** (10-min timebox) | 25 | 1 | **96%** |

| DE-scored, n = 26 | 5/5 | Below 5 |
|---|---|---|
| **Resolution quality:** *correct after a single prompt* | 23 | 3 × 3/5: *right direction, after repeated prompting* |
| **Actionability:** *fix applied exactly as written* | 23 | 3 × 4/5: *minor edits* |

No run scored 1 or 2. Every score below 5 was on Istio. The three runs that scored 3/5 for quality:

| Run | What happened |
|---|---|
| **DE-07 · I4**, the only unresolved AI run | A false premise. The DE opened with *"my pod is stuck at CrashLoopBackOff"*, but no pod was. The tool searched for that pod for the whole timebox. |
| **DE-09 · I3**, the only untrue statement | It assumed the Helm-deployed app followed the same structure as the repo in GitLab, which it cannot see. It still found the root cause. |
| **DE-12 · I1** | The diagnosis was right, but the DE could not follow the Istio explanation and kept prompting until the tool gave a fix ready to copy and paste. |

## B · How much the tool helps less-experienced DEs

After each AI run, DEs were asked whether they could have solved it without the tool:

| | Said "could not have" |
|---|---|
| Non-SF, **Istio** runs | **10 of 10** |
| Non-SF, Argo and k8s runs | 6 of 10 |
| **Non-SF, all runs** | **16 of 20** |
| SF, all runs | 0 of 6 |

On Istio, non-SF DEs with AI were compared with an SF engineer without it:

| Scenario | SF without AI | Non-SF with AI |
|---|---|---|
| I3 | 3 mins | 2 mins · 3 mins · 5 mins |
| I4 | 10 mins | **4 mins · 6 mins** · *unresolved* |
| I5 | 3 mins 30 secs | 3 mins 30 secs · 4 mins |

5 of 8 runs matched or beat SF, 2 were within 2 minutes, and 1 was the false-premise run. I4 is an `exportTo` visibility fault spanning three namespaces, two hops from its symptom, with a correct-looking AuthorizationPolicy in the path as a decoy.

**Learning forecast** (will it help you learn JPE apps faster?): 9 × 5, 2 × 4, 2 × 3; mean 4.54.

## C · Time saved

With AI, 22 of 25 resolved runs took 4 mins or less, with a median of 3 mins. Without AI, times ranged from 30 secs to unresolved at 15 mins.

Four scenarios went unresolved without AI; with AI, all 10 runs on them were solved:

| Scenario | Without AI (15-min timebox) | With AI (10-min timebox) |
|---|---|---|
| **A4:** sync says only "hook failed"; the reason is in a PreSync Job's pod log | DE-08 *unresolved* | DE-13 **1 min** · DE-03 (SF) 2 mins |
| **K5:** Service `targetPort` names a port the container does not expose | DE-11 *unresolved* | DE-02 (SF) 2 mins · DE-08 **3 mins** |
| **K6:** NetworkPolicy omits `namespaceSelector` | DE-12 *unresolved* | DE-03 (SF) 3 mins · DE-05 **4 mins** |
| **A2:** a self-deleting Job keeps the app permanently OutOfSync | DE-06 *unresolved* (DE-05 solved it in 11 mins) | DE-09 **3 mins 30 secs** · DE-11 **4 mins** · DE-12 **4 mins** · DE-10 **8 mins** |

Each run is graded against the [rubric](DE-trial-modality.md#d--scoring-rubric); n = 18, with exclusions as set out in the [trial design](DE-trial-modality.md#e--how-time-saved-is-worked-out). A2 has two baselines because it is a common issue: DE-05 solved it in 11 mins and DE-06 ran out of time. Savings use 11 mins.

| Scenario | DE | With AI | Baseline | Saving | Grade |
|---|---|---|---|---|---|
| A4 | 13 | 1 min | *unresolved* (DE-08) | — | **5** *(rescued)* |
| K5 | 08 | 3 mins | *unresolved* (DE-11) | — | **5** *(rescued)* |
| K6 | 05 | 4 mins | *unresolved* (DE-12) | — | **5** *(rescued)* |
| A2 | 09 | 3 mins 30 secs | 11 mins (DE-05) | +68% | 4 |
| A2 | 12 | 4 mins | 11 mins (DE-05) | +64% | 4 |
| A2 | 11 | 4 mins | 11 mins (DE-05) | +64% | 4 |
| I4 | 05 | 4 mins | 10 mins (DE-02) | +60% | 4 |
| I4 | 10 | 6 mins | 10 mins (DE-02) | +40% | 3 |
| I3 | 04 | 2 mins | 3 mins (DE-01) | +33% | 3 |
| A2 | 10 | 8 mins | 11 mins (DE-05) | +27% | 3 |
| I3 | 06 | 3 mins | 3 mins (DE-01) | 0% | 1 |
| I5 | 11 | 3 mins 30 secs | 3 mins 30 secs (DE-03) | 0% | 1 |
| K1 | 04 | 2 mins | 2 mins (DE-09) | 0% | 1 |
| K3 | 07 | 2 mins | 2 mins (DE-10) | 0% | 1 |
| I5 | 08 | 4 mins | 3 mins 30 secs (DE-03) | −14% | 1 |
| I3 | 09 | 5 mins | 3 mins (DE-01) | −67% | 1 |
| K7 | 06 | 2 mins | 30 secs (DE-13) | −300% | 1 |
| I4 | 07 | *unresolved* | 10 mins (DE-02) | — | 1 |

- **Hard (A2, I4):** every resolved AI run was 27–68% faster. The unresolved I4 run is the false-premise run in Appendix A.
- **Easy (baseline under 4 mins):** of 8 runs, 1 was a minute faster, 4 took the same time and 3 were 30 secs to 2 mins slower. The negative percentages are small in absolute terms.

## D · Scorecard

| Criterion | Distribution | Mean |
|---|---|---|
| Resolution quality (n = 26) | 23 × **5**, 3 × 3 | 4.77 |
| Actionability (n = 26) | 23 × **5**, 3 × 4 | 4.88 |
| Learning *(forecast)* (n = 13) | 9 × 5, 2 × 4, 2 × 3 | 4.54 |
| Willingness to use (n = 13) | **13 × 5** | **5.00** |


## E · Limits of this trial

- **Small sample.** 13 DEs, and most scenarios have a single baseline, so each saving compares one person's time with another's.
- **Self-reported scores.** DEs scored quality, actionability, "could not have solved", learning and willingness straight after using the tool, so expect a positive lean.
- **Timeboxes favour the baseline.** AI runs had 10 minutes against 15, so the 96% vs 69% resolution gap understates the tool's advantage.
- **Clean faults.** Each scenario has a single injected root cause; real incidents can have several.

## F · System prompt

Revised after the trial to address the gaps in [Where it fell short, and the fix](#where-it-fell-short-and-the-fix).

```markdown
You help duty engineers (DEs) troubleshoot an OpenShift cluster. The DE
meets today's problem cold. Be concise and exact — never at the cost of
accuracy. Ask a follow-up whenever you are unsure, the information is
ambiguous, or the answer would change your next step.

Read-only. Never write to the cluster and never apply a fix yourself.

## 1. Work inside one namespace, and scope every call to it

Get one namespace — or one Argo Application name — before any tool call.

- Named -> use it. Do not check it exists first.
- Not named -> ask "Which namespace is the workload in?", then stop and
  wait. Never hunt for it with `namespaces_list`, `pods_list` or an
  unscoped `events_list`: those scan the whole cluster and are the
  slowest calls available to you.
- Argo CD -> anchor on the Application name: `resources_get`
  (`argoproj.io/v1alpha1`, `Application`, namespace `openshift-gitops`).

Then pass `namespace` on every tool that accepts it:

- `pods_list_in_namespace`, never `pods_list`.
- `events_list` with `fieldSelector: "type=Warning"`.
- `resources_get` when you know the name, `resources_list` otherwise.
- `pods_log` with `tail: 50`; `container` in mesh namespaces (pods carry
  an `istio-proxy` sidecar); `previous: true` after a restart.
- Never `configuration_*` or `nodes_*` unless the DE asks about cluster
  config or nodes.

## 2. Procedure

1. `pods_list_in_namespace` — the real state.
2. `events_list` — `type=Warning`.
3. The failing object: `pods_get`, then `pods_log`.
4. Only if 1-3 do not explain it, the request path: the Service it
   calls, the NetworkPolicy over it, its Istio config
   (`networking.istio.io`, `security.istio.io`).
5. Another namespace only when evidence names it — a Service FQDN, a
   policy selector, a VirtualService host. Say what sent you there.

**Stop the moment you can name the cause.** Never re-fetch, and never
call to confirm what you already know.

## 3. Say only what you read

The DE's description is their interpretation, not a fact — work from
what step 1 actually shows. Every Evidence line must come from a tool
result you received.

- Never assume what a container listens on, what a field is set to, or
  what a policy permits. Read it.
- If the fix turns on intent the cluster cannot tell you — which subset
  is correct, which port the app should serve — ask instead of guessing.
- No cause found? Say so, and what you ruled out. Never invent one.

## 4. Answer in this format, nothing else

**Root cause** — 1-2 sentences. Exact object, field, value.
**Evidence** — max 3 bullets, each naming the tool and object it came from.
**Fix** — the exact change, as a command or a complete valid YAML file
(never a fragment), with the changed lines marked.
**Confidence** — `Confirmed` (nothing assumed) or `Probable` (something inferred).
```