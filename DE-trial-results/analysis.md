# DE trial — results analysis


>From `DE Trial — Facilitator Record (Responses).xlsx` (39 runs) and `DE Trial — Scorecard (Responses).xlsx` (13 DEs). Every hypothesis in [DE-trial-execution.md](../DE-trial-execution.md) is answered below, **ranked by weight rather than by the order the design lists them.**
---

## 1. It finds root causes the DE otherwise misses

**96% with AI (25 of 26) against 69% without (9 of 13).** 

> Note: The 1 miss with AI was a false premise where the DE prompted *"my pod is stuck at crashloopbackoff in X namespace"* when none was. Thus, the AI tool tried to keep finding that pod for the whole duration of the timebox. 

**Four runs without AI were unrersolved within 15 min timebox. Every one of those scenarios was then solved with the tool, in 1–4 minutes:**

| Scenario | Without AI | With AI |
|---|---|---|
| A4 | DE-08 — *unresolved* | DE-13 **1.0** |
| K5 | DE-11 — *unresolved* | DE-02 **2.0**, DE-08 **3.0** |
| K6 | DE-12 — *unresolved* | DE-03 **3.0**, DE-05 **4.0** |
| A2 | DE-06 — *unresolved* | four runs, **3.5–8.0** |

These are the cases that would have otherwise be an escalation to Starforge.

## 2. It solves what the DE could not have solved alone

After every run with the tool, the DE was asked: *"without the AI, would you have been able to solve this?"*

| | Said no |
|---|---|
| **Non-SF** | **16 of 20** |
| SF | 0 of 6 |


## 3. It makes DEs faster

Sorted by how long each scenario took *without* AI:

| Baseline | Savings with AI |
|---|---|
| A2 · 11.0 min | **+68%, +64%, +64%, +27%** |
| I4 · 10.0 min | **+60%, +40%** *(one run unresolved — the false premise above)* |
| I5 · 3.5 min | 0%, −14% |
| I3 · 3.0 min | +33%, 0%, −67% |
| K1 · 2.0 min | 0% |
| K3 · 2.0 min | 0% |
| K7 · 0.5 min | −300% |

**Both 10+ minute baselines produced large savings; nothing under 4 minutes produced any.**

The below test cases are not included because:

1. The baseline DE never finished, so there is no time to subtract from — graded 5 as a rescue instead:

   | Scenario | Without AI | With AI |
   |---|---|---|
   | A4 | DE-08 *unresolved* | DE-13 **1.0** |
   | K5 | DE-11 *unresolved* | DE-08 **3.0** |
   | K6 | DE-12 *unresolved* | DE-05 **4.0** |

2. A baseline exists, but the only run with AI was an SF DE — excluded, because the time saved is due to experience:

   | Scenario | Baseline | Only run with AI |
   |---|---|---|
   | A1 | DE-04, 11.0 min | DE-01 (**SF**), 0.5 min |
   | A3 | DE-07, 6.0 min | DE-02 (**SF**), 3.0 min |

3. I1 and I2 are the remaining two of the 14 where nobody ran them without AI.


## 4. DEs would actually use it

**13 of 13 answered 5/5** — "would use it every rotation"

## 5. Its answer is right without being guided there

**Resolution quality 5/5 in 23 of 26** — *"correct after a single prompt"*. The other three scored 3. **Nothing scored 1 or 2**, so the expensive failure — a confident wrong answer that sends a DE down the wrong path — never occurred.

## 6. It reaches the right answer

**25 of 26 runs with AI found the root cause.**

The 1 failure with AI was a false premise where the DE prompted "my pod is stuck at crashloopbackoff in X namespace" when none was. Thus, the AI tool tried to keep finding that pod for the whole duration of the timebox.

## 7. Its output is usable as it stands

**Actionability 5/5 in 23 of 26** — *"fix used exactly as given"*. The other three scored 4, *"minor edits"*. Nothing below 4.

## 8. It levels DEs up — they learn JPE apps faster

**9 of 13 answered 5, 2 answered 4, 2 answered 3.** Mean **4.54**, nothing below 3.

## 9. It does not hallucinate

**1 of 26 runs.** DE-09 on I3. Facilitator note:

> "Hallucinated because it could not see Gitlab, therefore assumed that wtv is helm deployed follows the exact same scaffolding as gitlab"

**The cause was missing git visibility, not the model.** The Kubernetes MCP server sees cluster state and nothing else, so asked about config whose source lives in git, it invented the repository structure. It did not produce a wrong root cause — I3 was still resolved, quality 5.

---

# Supporting data

## Resolution

| | Resolved | Unresolved | Rate |
|---|---|---|---|
| **Without AI** (15 min box) | 9 | 4 | 69% |
| **With AI** (10 min box) | 25 | 1 | **96%** |

Unresolved without AI: K6 (DE-12), K5 (DE-11), A2 (DE-06), A4 (DE-08). Unresolved with AI: I4 (DE-07).

## Time saved, per run

Measured against the without-AI run of the same scenario. Two sets are excluded per the design — **n = 18**: I1 and I2, which nobody ran without AI; and the 6 SF runs with AI, whose baselines are all non-SF.

| Scenario | DE | With AI | Baseline | Saving | Grade |
|---|---|---|---|---|---|
| A4 | 13 | 1.0 | *unresolved* | — | **5** *(rescued)* |
| K5 | 08 | 3.0 | *unresolved* | — | **5** *(rescued)* |
| K6 | 05 | 4.0 | *unresolved* | — | **5** *(rescued)* |
| A2 | 09 | 3.5 | 11.0 | **+68%** | 4 |
| A2 | 12 | 4.0 | 11.0 | **+64%** | 4 |
| A2 | 11 | 4.0 | 11.0 | **+64%** | 4 |
| I4 | 05 | 4.0 | 10.0 | **+60%** | 4 |
| I4 | 10 | 6.0 | 10.0 | +40% | 3 |
| I3 | 04 | 2.0 | 3.0 | +33% | 3 |
| A2 | 10 | 8.0 | 11.0 | +27% | 3 |
| I3 | 06 | 3.0 | 3.0 | 0% | 1 |
| I5 | 11 | 3.5 | 3.5 | 0% | 1 |
| K1 | 04 | 2.0 | 2.0 | 0% | 1 |
| K3 | 07 | 2.0 | 2.0 | 0% | 1 |
| I5 | 08 | 4.0 | 3.5 | −14% | 1 |
| I3 | 09 | 5.0 | 3.0 | −67% | 1 |
| K7 | 06 | 2.0 | 0.5 | −300% | 1 |
| I4 | 07 | *unresolved* | 10.0 | — | 1 |

**Grades:** 3 × 5 · 4 × 4 · 3 × 3 · 0 × 2 · 8 × 1. Mean 2.7.

## Scorecard distributions

| Criterion | Distribution | Mean |
|---|---|---|
| Resolution quality (n=26) | 23 × **5**, 3 × 3 | 4.77 |
| Actionability (n=26) | 23 × **5**, 3 × 4 | 4.88 |
| Learning (n=13) | 9 × 5, 2 × 4, 2 × 3 | 4.54 |
| Willingness to use (n=13) | **13 × 5** | **5.00** |

