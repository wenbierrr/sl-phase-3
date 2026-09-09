# <Service/Tool> Service Design

| **Metadata**      | **Value**                                  |
|--------------------|--------------------------------------------|
| **Status**         | <Current / Superseded by `<doc>` — pick one, delete the other> |
| **Owners**         | @name1, @name2                             |
| **Created**        | YYYY-MM-DD                                 |
| **Last Reviewed**  | YYYY-MM-DD                                 |

## Change log

<Track document revisions, not product versions — the point is to show the reader that this doc has kept up with the live system. Update Last Reviewed above whenever you confirm this doc still matches reality, even with no content change.>

| Version | Date       | Description                | Author |
|---------|------------|-----------------------------|--------|
| 1.0.0   | YYYY-MM-DD | Initial release             | @me    |

---

**What belongs here:** the architecture as it runs today, or as targeted by an approved epic in scope — components, interfaces, dependencies, known limitations. Not speculative or unapproved ideas — that's `rfc/`'s job (or `adr/`'s, once decided). Update it as part of the epic that changes it, even before every piece is live — git history covers what it looked like before.

Sections 1–6 and 9 are required. If one genuinely doesn't apply, write `Not applicable — <one line why>` rather than deleting the heading. The rest may be deleted when empty — for a simple tool, most of §7–11 may be one line each, and that's fine.

Not sure this belongs in `design/`? → [documentation-guide.md §3.4](../../team-guidelines/documentation-guide.md#34-docsdesign)

## 1. What this is, and why we run it

<What is this, and what problem does it solve? State what it does, not why we chose it over alternatives — that's the ADR's job; link it here.>

## 2. Key concepts

<Only terms the rest of this doc depends on that aren't general Kubernetes knowledge — upstream vocabulary specific to this tool (e.g. operator/operand, WAL, PITR), one short paragraph each. Needs more than that? It's a `knowledge/` 101 — link it instead. Put every term here once — no second "Key Terms" block later in the doc.>

## 3. Architecture at a glance

<The C4 container diagram — source `.drawio` and exported image live in this component or domain's `assets/` (`repo-structure.md`), referenced from here, not a separate `docs/diagram/` directory.

A bare image isn't documentation: add a short paragraph naming what to notice — trust/security boundaries, the one path data takes through the system, whatever isn't obvious from the picture alone.>

## 4. Components

<Every component that actually runs — controller, sidecar, operand. For each: what it does, workload kind (Deployment/StatefulSet/DaemonSet/CR), namespace, built vs. pulled from upstream.

A component the reader can't locate in the cluster from this section alone is not documented.>

## 5. How it works

<The runtime path tying §4's components together, as an ordered sequence — trigger, steps, state changes. Mechanism, not rationale.

Good in-repo example: Trivy Operator's "Scan Sequence" — trigger → image identification → scanner pod spin-up → scan → CRD write-back → spin-down.>

## 6. Supported use cases and patterns

<What this actually supports — and just as importantly, what it explicitly doesn't, so a reader doesn't assume a capability exists. State the boundary, not just the happy path.

Good in-repo example: "Exposing a PostgreSQL port out-of-cluster, regardless of which port, is not supported from a manageability and security perspective.">

## 7. Interfaces & Dependencies

### Interfaces

<How this is interacted with — CRs/manifests, CLI, API, console/GUI. For each: who uses it (platform engineer vs. consumer), and for what. Link API docs if they exist.>

### Dependencies and connectivity

<External dependencies required for full functionality but not deployed by this component or domain (e.g. LDAP federation, an S3 endpoint, a container registry), and any connectivity required outside the cluster.

For each: what breaks if it's unavailable — "requires S3" alone isn't enough; say what happens to this service when S3 is down.>

## 8. Capacity & Availability

### Resources and scale

<CPU/memory/storage requirements, and GPU if applicable. Prefer measured numbers (`report/` benchmarks) over estimates — link the report rather than restating it here, so the two can't drift.>

### Availability and data protection

<HA shape (replica count, failover mechanism), replication mode (sync/async), and backup/restore posture (mechanism, RTO/RPO if known). If none of these apply, say so explicitly rather than omitting the section — "no HA; single replica" is a real answer.>

## 9. Standards & Security Posture

### Standards baseline

<State whether this service meets the Baseline in `team-guidelines/service-standards.md` (don't copy the tables here — link them), and which Nice-to-haves it's adopted, if any. Gaps against Baseline aren't optional — link the tracking issue that closes them.>

Baseline: <Met / gap — if gap, which checks, and the issue tracking the fix>
Nice to have adopted: <list, or "none">

### Security posture

<CVE/SAST/CIS/hardening state — link the relevant `report/` doc(s) rather than restating findings here.>

## 10. Deployment shape

<Chart topology — single chart, umbrella + subcharts, GitOps vs. Helm-applied. The shape, not the procedure — "how to install" belongs in `deployment/`.>

## 11. History, Limitations & Roadmap

### Decisions that shaped this

<Link the ADRs behind the current architecture, one line each on what each one settled. If you're explaining *why* a choice was made in more than a link and a sentence, that content belongs in an ADR, not here.>

| ADR | What it settled |
|---|---|
| [ADR-NN](../adr/NN-slug.md) | ... |

### Limitations and known gaps

<Known limitations, and any tech debt. Don't keep a separate tech-debt file — it goes stale independently of this doc; fold it in here.>

### Future considerations

<The only place forward-looking content belongs in this doc. Each item should link an RFC or issue if one exists, or be a single line if it's just a note to future-self.>