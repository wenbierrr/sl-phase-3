# 8-week plan

**Week 1 = 10 Aug 2026 · Week 8 ends 2 Oct 2026**

The question these 8 weeks answer: **LibreChat + which combination of MCP servers brings DEs the most value?**

## The plan

| Weeks | Focus | What lands |
|---|---|---|
| **1–2**<br/>10–21 Aug | **Experiment** | Run the LibreChat + MCP server combinations against all three problem areas. Order: **kubectl → k8sgpt only if it earns its place → GitLab → metrics**.<br/>Agree the [rubric](#de-trial-rubric) and pass bars with the team. Build the injected fault catalogue — nothing breaks on cue, so the data points are forced |
| **3–5**<br/>24 Aug – 11 Sep | **DE trial** | DEs run the tuned tool against injected faults and whatever real failures arrive. One [rubric](#de-trial-rubric) scorecard per case, aimed at the **less-experienced DEs** — that is the group the value claim rests on.<br/>*This is where evidence starts.* |
| **6–8**<br/>14 Sep – 2 Oct | **RFC + service design** | **[RFC-2](rfc-2-mcp-server-eval-for-librechat.md)**, written from the scorecards: which combination brought value, and where it brought none.<br/>Then **`service-design`** — production security, RBAC, rollout — **only if the trial passes its Day-0 thresholds and the ADR goes ahead.** |

All three acceptance criteria sit in weeks 3–8: the DE trial, the RFC, and the conditional service design. [RFC-1](rfc-1-ai-tool-evaluation.md) already settled the orchestrator.

## What gets tested

Every combination is tested against all three — that is how we find which one is worth having.

| | Problem | |
|---|---|---|
| **Troubleshoot** | Argo sync failures · pods down / apps broken · Istio config | 1.1.1–1.1.3 |
| **Deploy** | The AI drafts the exact change — file path, diff, commit message, MR description — as chat output. A human commits, raises the MR, approves and syncs. **The AI never writes to git or the cluster.** | 1.1.4 |
| **Threshold + remediate** | OpenShift CPU/memory dashboards: determine what X is, then how to stop the bleeding and what the persistent fix is | 2.2 |

## DE trial rubric

The epic asks whether an agent can take over the DE's triage and debugging work. Case by case, that comes down to one question: **can the DE take the tool's answer and act on it, so that it can save time and unecessary escalations?**

Three things have to be true for that, and each is separate from the others:

| Criteria | Measures |
|---|---|
| **Resolution quality** | Was the answer correct and accurate? Did the DE have to constantly guide the troubleshooting? |
| **Time saved** | How much of the DE's time the tool saved? |
| **Actionability** | Could the DE act on the output as it is and prevent uneccessary escalations? |


Those three become the criteria. Each is graded 1–3, so a case records not just *whether* the tool fell short but *where*:

| Criteria | 1: No Go | 2: OK | 3: Good |
|---|---|---|---|
| **Resolution quality** | • Wrong answer<br/>• Made something up<br/>• No answer at all<br/>*(however much the DE steered it)* | • Right area, but not the exact object or value<br/>• Or exactly right, but only because the DE steered it — told it where to look, corrected its wrong turns | • Exactly right — the object, field and value match the "answer key"<br/>• The AI found its own way there |
| **Time saved** | Saved no time (**0%**); the DE would have been just as fast alone | Saved some time (**0% ≤ x ≤ 30%**) using it | Saved **≥ 30%** of the time using it |
| **Actionability** | • No usable fix, or a fix that would make things worse<br/>• The case still needs an escalation | • Fix is on the right track, but the DE has to edit AI's output before using it<br/>• Closed without escalation | • Fix can be used exactly as given<br/>• Closed with **no escalation and no edits to AI's output** |

## The pass bar

A case passes only if it scores:

| Criteria | Minimum |
|---|---|
| **Resolution quality** | **3** |
| **Actionability** | **3** |
| Time saved | 2 |

## DE trial scorecard

One per case. Part 1 is filled in straight after the case; Part 2 waits until the answer key is opened, since correctness cannot be graded against a sealed key.

**Case:** ID · date · DE · **experience (junior / mid / senior)** · sub-area (1.1.1–1.1.4 / 2.2) · MCP servers in force

**Part 1 — the DE, right after the case**

| # | Question | Answer |
|---|---|---|
| 1 | Outcome | resolved · **unresolved at 30 min** |
| 2 | Minutes to root cause *(resolved only)* | ___ |
| 3 | **Time saved** — vs handling it without the tool | 0% = **1** · <30% = **2** · ≥ 30% = **3** |
| 4 | Did you have to steer it — tell it where to look, or correct a wrong turn? | Y / N |
| 5 | Did it show evidence you could check yourself? | Y / N |
| 6 | Without the tool, would you have escalated to SF? | Y / N |
| 7 | Did you still need SF? | Y / N |
| 8 | Anything it said that was made up or wrong? *(one line)* | |

**Part 2 — the DE plus a reviewer who is not the injector, after the key is opened**

| # | Question | Answer |
|---|---|---|
| 9 | **Resolution quality** — if 1, wrong answer (`1-W`) or no answer (`1-N`)? | 1-W · 1-N · 2 · 3 |
| 10 | **Actionability** | 1 · 2 · 3 |
| 11 | *(deploy cases)* Was the diff applied unchanged? | Y / N |

**Q6 and Q7 together are what make "prevented an escalation" countable** — Q7 alone cannot tell a save from a case that was never going to SF. Q3 is banded rather than a precise figure because it is a DE estimate with no measured baseline behind it. Q4 feeds Resolution quality: it is the DE's on-the-spot record of steering, which the reviewer checks against the chat transcript when grading Q9.

