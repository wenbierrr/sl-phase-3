# Documentation Guide

This file establishes the team process for how documentation should be structured for any tool or service. Built on [Diátaxis](https://diataxis.fr/): every directory maps to exactly one **audience** (who reads it) and one **intent** (what they're doing). A doc serving two audiences, or answering two intents, is two docs.

This file answers **where and which** — not **how**. Templates for each type live in `template/doc/`.

## Doc Lifecycle

```text
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 DOCUMENTATION LIFECYCLE                                 │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                             │
                     1. DECISIONAL PIPELINE  │ (Proposing & Deciding)
                                             ▼
               ┌────────────────────────────────────────────────────────┐
               │                        1. RFC                          │
               │         "What should we change and why?"               │
               └────────────────────────────┬───────────────────────────┘
                                             │
                                             ▼
               ┌────────────────────────────────────────────────────────┐
               │                        2. ADR                          │
               │                 "What did we decide?"                  │
               └────────────────────────────┬───────────────────────────┘
                                             │
                                             ├────────────────────────────────────────┐
                                             │                                        │
                      2. SYSTEM STATE PIPELINE │ (What IS)       3. OPERATIONAL PIPELINE │ (What to DO)
                                             ▼                                        ▼
               ┌────────────────────────────────────────┐             ┌───────────────────────────┐
               │           3. SERVICE DESIGN            │             │       DEPLOYMENT /        │
               │       "How does it run right now?"     │             │       OPERATIONS          │
               └────────────────────┬───────────────────┘             │ "How to install/run/fix?" │
                                    │                                 └─────────────┬─────────────┘
                                    │                                               │
               ┌────────────────────┴────────────────────┐             ┌────────────┴──────────────┐
               │               USER GUIDE                │             │          RECIPES          │
               │   "How do consumers consume it?"        │             │   "Runnable configs"      │
               └─────────────────────────────────────────┘             └───────────────────────────┘

 ─── STANDALONE SATELLITES ────────────────────────────────────────────────────────────────
  • KNOWLEDGE (101s / Spikes / General literacy)
  • REPORT    (Automated evidence / Scans / Benchmarks)
```

---

## 1. Directory Overview

| Directory | Audience | Intent | Purpose |
|---|---|---|---|
| `docs/knowledge/` | ALL | Learning & Researching | 101s, spikes, workshops, comparisons |
| `docs/rfc/` | PLT | Proposing & Deciding | Propose an architecture change |
| `docs/adr/` | PLT | Proposing & Deciding | Record a decision |
| `docs/design/` | PLT | System State | Current architecture |
| `docs/deployment/` | PLT | Operating & Maintaining | Install, upgrade, or decommission |
| `docs/recipes/` | PLT | Operating & Maintaining | Sample Helm values & manifests |
| `docs/operations/` | SRE | Operating & Maintaining | Runbooks, no version change |
| `docs/report/` | PLT | System State | CVE scans & benchmark results |
| `docs/user-guide/` | EXT | System State | Consumer datasheet & contract |

**Audience legend:**

- **EXT** — external consumer teams (own namespace + a filed service request, nothing more)
- **SRE** — on-call platform engineers
- **PLT** — Starforge platform engineers (cluster-admin)
- **ALL** — anyone, generic information

---

## 2. Resolving Ambiguity

Most directories are unambiguous once you know the audience and intent — nobody confuses a user guide with an ADR. These two clusters aren't:

**Knowledge vs RFC.** Is this general literacy, or are we weighing a real decision right now?
- Explains how something works, or surveys or compares options with no active decision behind it → `knowledge/`. Even a full comparison with tradeoffs belongs here if nothing is currently being decided — e.g. "Istio Ambient vs Sidecar: how they work" written for team literacy.
- Compares candidates to solve a real, active platform problem (there's a live decision on the table) → `rfc/`, even if the document never says "propose" and even if the outcome is to keep the status quo. "We recommend X" driven by a live decision is an RFC wearing a softer verb, not a factsheet.
- Wording doesn't decide this, and neither does whose alternative got rejected — a vendor product or our own design can both end up in `rfc/`. The signal to check is whether there's an active initiative or problem this comparison is deciding.

**Deployment vs Recipes vs Operations.** All three touch "making the service work," which is why they blur together.
- End state differs from the start (image tag, chart version, component set) → `deployment/`.
- No version change, but a Duty Engineer could be paged into it with no warning → `operations/`.
- Neither — it's the runnable config artifact (Helm values, manifests, CRDs) itself → `recipes/`. A deployment or operations doc *references* a recipe; it doesn't duplicate it.

`docs/deployment/` may hold files flat regardless of how many action kinds are present. `<action-kind>/`
subfolders (`installation/`, `upgrade/`, `migration/`, ...) are optional — reach for them when a directory
has enough files that flat browsing gets unwieldy, not as a rule triggered by file count or kind.
`platform/openshift/` is the reference for the grouped shape once you want it.

### Naming Conventions

| Directory | Naming Pattern |
|---|---|
| `docs/knowledge/` | `<topic>-101.md`, `<question>.md`, `<topic>-workshop.md`, `<topic>-comparison.md`, `external-<source>-<topic>.md` |
| `docs/rfc/` | `NN-<slug>.md` |
| `docs/adr/` | `NN-<slug>.md` |
| `docs/design/` | `service-design.md` (one per component, domain, or process); `<component>-design.md` for extras |
| `docs/deployment/` | `<action-kind>/<outcome-slug>.md` (e.g. `installation/clean-install.md`, `upgrade/16-to-17.md`) |
| `docs/recipes/` | `<artifact-kind>/<outcome-slug>.<ext>` (e.g. `helm-values/backup-using-s3-barman.yaml`) |
| `docs/operations/` | `<symptom-or-task>.md` |
| `docs/report/` | unnumbered; `YYYY-MM-DD-<slug>` only if the run date is itself meaningful |
| `docs/user-guide/` | `info-sheet.md`, `integration-guide.md` |

- **kebab-case, lowercase everywhere** — pattern `^[a-z0-9][a-z0-9.-]*\.md$`.
- **ISO dates (`YYYY-MM-DD`) everywhere.**
- **Directory names from a closed list only** — `docs`, `knowledge`, `rfc`, `adr`, `design`, `deployment`, `recipes`, `operations`, `report`, `user-guide`, `plans`. Don't invent names or guess plurality. `plans/` is valid only in `processes/<process>/docs/` (see `team-guidelines/repo-structure.md`) — not in a component's or domain's `docs/`.
- **No letter suffixes** (`03a-`) — use a full number plus a `Related: NN-<slug>.md` line instead.
- **No typed prefixes** (`RFC01-`) — the directory already carries the type.

---

## 3. Per-Directory Specs

### 3.1 `docs/knowledge/`
- **What it is:**
  - 101s/explainers, spikes, workshops — general domain literacy
  - Landscape surveys and comparisons with no active decision behind them
  - No live platform problem being decided
- **What it is not:**
  - Executed against a real cluster → `operations/`
  - Compares candidates to solve a real, active platform problem → `rfc/` (see §2), regardless of wording
  - Third-party content without the `external-` prefix and a provenance block

### 3.2 `docs/rfc/`
- **What it is:**
  - A proposal to change **our own** architecture
  - Names at least one alternative rejected, and why
  - Must conclude in an ADR once a decision is reached — an RFC that never resolves is incomplete, not simply long-running
- **What it is not:**
  - No rejected alternative → `knowledge/`
  - A factsheet or landscape survey with no active decision behind it → `knowledge/` (see §2)
  - Describes what already is, not what should change → `design/` or `report/`
  - A sequence the reader is meant to execute → `deployment/` or `operations/` (see §2)
  - Has `Milestone`/`Epic`/`Sprint`/`Phase N` headings → a GitHub Epic, not this repo
  - Contains a `## Decision` heading → extract to `adr/`, mark the RFC `Superseded by ADR-NN`

### 3.3 `docs/adr/`
- **What it is:**
  - One decision, one page — Context / Decision / Consequences
  - Usually the conclusion of an RFC — but doesn't require one; a small or clear-cut decision can go straight to an ADR with no RFC first
- **What it is not:**
  - Options analysis → belongs in the RFC/knowledge doc this ADR points back to
  - Longer than one page → that's a design doc

### 3.4 `docs/design/`
- **What it is:**
  - Live-state architecture — HA or not, standards baseline met, interfaces, dependencies, known limitations
  - Exported diagram images live in the piece of work's `assets/` (`repo-structure.md`), referenced from here; the editable source (`.drawio`) sits at `docs/` root — never a `docs/` subdirectory
- **What it is not:**
  - Anything conditional ("we would") → `rfc/`
  - A decision and its rationale → `adr/`
  - Commands to run → `deployment/` or `operations/`
  - A standalone tech-debt register → fold into a Limitations subsection

### 3.5 `docs/deployment/`
- **What it is:**
  - Install, upgrade, migration, decommission, airgap deltas, mirroring
  - End state differs from the start state (see §2)
- **What it is not:**
  - Symptom-driven remediation with no version change → `operations/`
  - The rationale for why this shape was chosen → `design/` or `adr/`

### 3.6 `docs/recipes/`
- **What it is:**
  - Runnable config demonstrating one outcome — Helm values, manifests/CRDs, kustomize overlays
  - Organized by artifact kind, one file per outcome (see §2)
- **What it is not:**
  - A fill-in-the-blank template → `template/doc/`
  - Prose steps or rationale for when/why to use it → `deployment/` or `operations/`, which reference it rather than duplicate it
  - A vague filename like `sample.yaml` — name the outcome instead

### 3.7 `docs/operations/`
- **What it is:**
  - Runbooks — symptom → diagnosis → remediation
  - Day-2 CLI, monitoring/alert response, scheduled checks, tool administration
  - No version change, but a Duty Engineer could be paged into it with no warning (see §2)
- **What it is not:**
  - A version/chart/component change → `deployment/`
  - Something a *consumer* is meant to run themselves → `user-guide/`
  - Pure learning material with no action attached → `knowledge/`

### 3.8 `docs/report/`
- **What it is:**
  - Generated evidence about our own actual deployment — scans, benchmarks, CVE justifications
- **What it is not:**
  - A comparison of third-party products for adoption → `knowledge/` (or `rfc/` if there's an active decision behind it)
  - An image with no prose stating what was measured — add the paragraph or move the image to the piece of work's `assets/`

### 3.9 `docs/user-guide/` — the External tier
- **What it is:**
  - Decided by audience, not bucket — any component may carry one when the content is genuinely written for consumers, `services/` and `platform-tools/` alike (e.g. a platform-tool consumers integrate with, like an SSO provider)
  - Exactly two shapes: `info-sheet.md` (a service datasheet) and `integration-guide.md` (an interface contract)
  - Zero-privilege rule: every instruction must be executable by someone whose only access is their own namespace plus a filed service request
- **What it is not:**
  - Written for the engineer operating the tool, not the consumer → `operations/` (or `knowledge/` if internal enablement) — regardless of bucket, this content is never a user guide

---

## Appendix: Example Layout (`services/postgres-k8`)

```
services/postgres-k8/
├─ docs/
│  ├─ knowledge/
│  │  ├─ postgres-replication-101.md
│  │  ├─ logical-vs-physical-replication-comparison.md
│  │  └─ connection-pooling-workshop.md
│  ├─ rfc/
│  │  ├─ 01-choosing-cnpg-operator.md
│  │  └─ 02-adopt-barman-cloud-backup-plugin.md
│  ├─ adr/
│  │  ├─ 01-cnpg-operator-selected.md
│  │  ├─ 02-barman-cloud-backup-plugin-selected.md
│  │  └─ 03-use-pgbouncer-for-connection-pooling.md
│  ├─ design/
│  │  └─ service-design.md
│  ├─ deployment/
│  │  ├─ installation/
│  │  │  └─ clean-install.md
│  │  └─ upgrade/
│  │     └─ 16-to-17.md
│  ├─ recipes/
│  │  ├─ helm-values/
│  │  │  ├─ sample-prod-deployment.yaml
│  │  │  ├─ sample-staging-deployment.yaml
│  │  │  └─ backup-using-s3-barman.yaml
│  │  └─ manifests/
│  │     └─ sample-manual-backup.yaml
│  ├─ operations/
│  │  ├─ creating-a-new-db-instance.md
│  │  └─ restoring-from-backup.md
│  ├─ report/
│  │  ├─ 2026-06-15-trivy-cve-scan.md
│  │  └─ benchmarking-nfs-vs-block.md
│  └─ user-guide/
│     ├─ info-sheet.md
│     └─ integration-guide.md
└─ helm/
   └─ ...                         # the Helm chart itself
```