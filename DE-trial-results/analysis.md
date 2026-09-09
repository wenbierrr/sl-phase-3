# DE trial — results analysis

From `DE Trial — Facilitator Record (Responses).xlsx` (39 runs) and `DE Trial — Scorecard (Responses).xlsx` (13 DEs), against the design in [DE-trial-execution.md](../DE-trial-execution.md). The roster was followed exactly — 13 DEs × 3 runs, one per family, one without AI and two with, all three Istio baselines SF.

> **No pass bar was ever agreed** — the design still reads *"To be agreed"*. Nothing here is a pass or a fail. Setting a bar now, against data that already exists, is the exact failure the design was built to prevent.

---

# Insights derived

**Insights 1–5 are the case for adopting LibreChat + Kubernetes MCP. 6–11 are the supporting detail and the costs that come with it.**

## 1. It reaches the right answer nearly every time — and never reached a wrong one

**Evidence** — root cause found in **25 of 26 AI runs (96%)**, against **9 of 13 (69%)** without AI. Three cases that went **unresolved** without AI were resolved with it: A4, K5, K6.

**No case was scored 1 or 2 on resolution quality**, where 1 is *"wrong or made up — named a cause that was not the cause"*. The expensive failure mode — a confident wrong answer that sends a DE down the wrong path — **did not occur once in 26 runs**.

## 2. A less-experienced DE with the tool outperformed a platform specialist without it

**Evidence** — **I4** is the hardest scenario in the catalogue: a three-namespace `exportTo` visibility fault, the symptom two hops from the cause, with a correct-looking `AuthorizationPolicy` sitting in the path as a decoy. An **SF platform specialist needed 10 minutes**. Two non-SF DEs with the tool needed **4 and 6 minutes** — **60% and 40% faster than the specialist**.

This is the levelling-up claim the whole trial was designed around, and it held on the one scenario hard enough to test it.

## 3. It solved problems the less-experienced DEs could not have solved alone

**Evidence** — asked *"without the AI, would you have been able to solve this?"*, **14 of 18 non-SF cases (78%) said no**. **0 of 6 SF cases said no** — every platform specialist said they could have managed alone.

The value falls exactly where the epic said it should: on the DEs who don't already know where to look. **Winner point 3, answered.**

## 4. On the problems that actually consume DE time, it cut time by 27–68%

**Evidence** — sorted by how long each scenario took *without* AI:

| Baseline | Savings with AI |
|---|---|
| A2 · 11.0 min | **+64%, +68%, +27%, +64%** |
| I4 · 10.0 min | **+60%, +40%** |
| I5 · 3.5 min | 0%, −14% |
| I3 · 3.0 min | +33%, 0%, −67% |
| K1 · 2.0 min | 0% |
| K3 · 2.0 min | 0% |
| K7 · 0.5 min | −300% |

**Every scenario with a 10+ minute baseline produced a large saving.** Scenarios under 4 minutes produced none — but a fault a DE already spots in two minutes is not where a rotation's time goes, and reading any answer costs about as long as spotting it did. K7's −300% is 2 minutes against 0.5: measurement noise wearing a percentage.

> The tool adds nothing to problems a DE already solves in two minutes. On problems that take ten minutes or defeat them entirely, it cut the time by 27–68%, or produced an answer where there had been none.

**Do not quote a single blended mean across all 18 runs** — it averages the real effect against a measurement floor and understates both.

## 5. Every DE would use it, on every rotation

**Evidence** — asked *"how willing would you be to use this on your next duty rotation?"*, **13 of 13 answered 5/5** — the top of the scale, meaning "would use it every rotation". **Unanimous**, including both DEs who complained about verbosity. There is no adoption risk to manage.

## 6. Its answers are usable as written, not just correct

**Evidence** — **Actionability 5/5 in 23 of 26 cases** ("fix used exactly as given"); the other three scored 4 ("minor edits"). **Resolution quality 5/5 in 23 of 26** ("correct after a single prompt"). Nothing scored below 3 on either. The DE applies the fix rather than rewriting it.

## 7. The single AI failure was a prompt problem, not a tool problem

**Evidence** — the only unresolved AI run was DE-07 on I4. That DE wrote:

> "the prompt needs to provide more context... if not the AI will be stuck in a loop and will be unable to triage. prompt used which made it take very long: *'i have an issue with namespace i4-core ... my pod is stuck at crashloopbackoff'*"

That prompt is also **factually wrong about I4** — nothing in `i4-core` is in CrashLoopBackOff; `order-router` is `1/2 Ready`. The tool was reasoning from a false premise it had no instruction to check. Addressed since by the `serverInstructions` rewrite, which now makes it confirm real pod state first.

## 8. The one hallucination was caused by having no repository access

**Evidence** — **1 of 26 runs** (DE-09, I3). Facilitator note:

> "Hallucinated because it could not see Gitlab, therefore assumed that wtv is helm deployed follows the exact same scaffolding as gitlab"

The same DE independently wrote: *"there is no access to gitlab to troubleshoot potential issues in yaml file for configuration issues."*

The Kubernetes MCP server sees cluster state and nothing else. Asked about configuration whose source lives in git, the model filled the gap by inventing repository structure. **This is the named cost of GitLab MCP going untested**, and belongs in the ADR as an accepted trade-off. It did *not* produce a wrong root cause — I3 was still resolved, quality 5.

## 9. Verbosity was the main complaint, and it appears to have been fixed mid-trial

**Evidence** — both complaints came on Mon/Tue, both compliments on Wed/Thu:

> "slightly verbose. Could get straight to the point." — DE-01 *(Mon)*
> "talk too much... i need to scroll and skimp through to find where the solution is" — DE-12 *(Tue)*
> "the model response is short enough and concise with actionable easy to find due to the bold **Fix**" — DE-03 *(Thu)*

That timing matches the `serverInstructions` rewrite landing mid-week — **but the config version in force per run was not recorded**, so this is a pattern, not a proven cause. See [open questions](#open-questions).

## 10. The DEs' own biggest worry is deskilling, not accuracy

**Evidence** — two of the four sub-5 Learning scores are this concern:

> "Engineer may lose initial engineering instinct build up... with the use of AI, many a times i would imagine engineers will just skip this engineering troubleshooting steps and go straight to AI." — DE-04

> *(on DE-12's 3)* "because it will make her more lazy and not learn"

Neither is a complaint about the tool being wrong. It belongs in the ADR as a named trade-off with a mitigation.

## 11. Two smaller gaps, both cheap to fix

**Evidence** —

> "Cross environment does not work well (e.g command given were for bash, asked for a powershell command but the command did not work) but the solution was correct." — DE-13

> "It would be nice to have a comparable view, such as current config and change made config." — DE-10

Both are `serverInstructions` changes, not architecture.

---

# Supporting data

## Resolution

| | Resolved | Unresolved | Rate |
|---|---|---|---|
| **Without AI** (15 min box) | 9 | 4 | 69% |
| **With AI** (10 min box) | 25 | 1 | **96%** |

Unresolved without AI: K6 (DE-12), K5 (DE-11), A2 (DE-06), A4 (DE-08). Unresolved with AI: I4 (DE-07).

Unpaired across all scenarios, mean time-to-root-cause fell from **5.4 → 3.1 min**, median **3.5 → 3.0**. Directional only — different scenarios, different people.

## Time saved, per run

Measured against the without-AI run of the same scenario. Excluded per the design: the 6 SF with-AI runs (confounded), and I1/I2 (no baseline exists). **n = 18.**

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

**The 30-minute assumption was unnecessary.** The rubric already grades *"resolved a case that went unresolved without AI"* as 5, so the three rescued runs grade without inventing a baseline. Drop it from the write-up rather than defending it.

## Levelling-up — non-SF with AI vs SF baseline

| Scenario | SF baseline | Non-SF with AI | Result |
|---|---|---|---|
| **I4** | DE-02, 10.0 min | DE-05 **4.0**, DE-10 **6.0**, DE-07 *unresolved* | **2 of 3 beat the SF DE** |
| I3 | DE-01, 3.0 min | DE-04 2.0, DE-06 3.0, DE-09 5.0 | 1 faster, 1 equal, 1 slower |
| I5 | DE-03, 3.5 min | DE-11 3.5, DE-08 4.0 | 1 equal, 1 slower |

I3 and I5 had 3.0 and 3.5-minute SF baselines, so they are floor-effect cases carrying no signal. **Of the three scenarios meant to carry the levelling-up claim, only I4 had the headroom to test it.**

## Scorecard distributions

| Criterion | Distribution | Mean |
|---|---|---|
| Resolution quality (n=26) | 23 × **5**, 3 × 3 | 4.77 |
| Actionability (n=26) | 23 × **5**, 3 × 4 | 4.88 |
| Learning (n=13) | 9 × 5, 2 × 4, 2 × 3 | 4.54 |
| Willingness to use (n=13) | **13 × 5** | **5.00** |

All three resolution-quality 3s and all three actionability 4s came from a non-SF DE's **third** scenario. For non-SF DEs the third scenario is always Istio, so *hardest family* and *last run of the session* are perfectly confounded — this data cannot separate them.

## Against the proposed bars

Drafted in the since-deleted RFC-2 and **never agreed**. Reproduced because this is now their only record:

| Proposed bar | Result | |
|---|---|---|
| Sample size n ≥ 8 | 26 AI cases | ✅ |
| Root cause found ≥ 70% | 96% | ✅ |
| **Wrong** root cause ≤ 1, none costing >15 min | 0 wrong | ✅ |
| **Hallucination = 0** | **1** | ❌ |
| Less-experienced DE says useful ≥ 4 of every 5 | 14 of 18 = **78%** | ❌ *by 2 points* |
| Escalation avoided ≥ 30% | — | removed by the supervisor |
| Reached a raised MR ≥ 1 case | — | out of trial scope (1.1.4) |

Two miss, one by a whisker. **Neither is a fail, because neither was agreed in advance** — and 78% vs 80% on n=18 is one DE answer either way.

---

# Corrections to the trial document

| # | Issue |
|---|---|
| 1 | **n is 18, not 20.** The design computes 26 − 6 SF = 20, but I1 and I2 are separately excluded for having no baseline, and both were run by non-SF DEs so they sit outside the 6. Correct: 26 − 6 − 2 = **18** |
| 2 | **A2 has two baselines** — DE-05 at 11.0 min and DE-06 unresolved. The design doesn't cover this. This analysis uses the observed 11.0, the conservative choice; using 30 would inflate all four A2 savings |
| 3 | **The 30-minute assumption is not needed** — the rubric's own grade-5 clause covers rescued cases |
| 4 | **DE-12's comment contradicts the recorded scores.** The comment says *"3 for willingness"*, but the form records Learning 3 / Willingness 5. The reasoning is about learning, so the recorded values are probably right — confirm with the DE |
| 5 | **DE-10 left both "could you have solved this" fields blank**, so that measure is n=24, not 26 |

# Limits

State these wherever the figures are quoted.

- **Injected faults are cleaner than organic ones**, and the injector knew every answer.
- **Seven of fourteen scenarios had baselines under 4 minutes** — too fast to detect a saving. Time saved was effectively measured on half the catalogue.
- **n = 18 for time saved, 26 for resolution, 13 for the per-DE scores.** Every percentage moves several points on one changed answer.
- **Ten of the fourteen scenarios had never run on a live cluster** before the trial, including the whole Argo family.
- **It ran on CRC**, with hub and stg collapsed onto one cluster. The placement half of 1.1.1 is still unanswered.
- **Drill urgency is not incident urgency**, and a 10-minute box is not a rotation.
- **1.1.4 and 2.2 were not tested.** Nothing here speaks to the deploy flow or the CPU/memory threshold work.

# Open questions

1. **Which runs used which `serverInstructions`?** The block was rewritten mid-week. If the Mon/Tue cohort ran the old one, insight 9 is confirmed and the trial contains two tool configurations rather than one. Settle before publishing any figure.
2. **Is a pass bar being set at all?** If yes, it must be acknowledged as post-hoc. Reporting the figures without a verdict is more defensible and matches what the design intended.
3. **DE-12's Learning/Willingness values** — confirm which field the 3 belongs to.
