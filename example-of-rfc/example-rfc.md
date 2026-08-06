# EVAL-01: OCI Resource Manager vs Terraform CLI

| **Metadata**    | **Value**                                                        |
|------------------|---------------------------------------------------------------------|
| **Status**       | No-Go |
| **Service(s) evaluated** | OCI Resource Manager |
| **Replacing**    | N/A — execution-model shift, not an on-prem replacement |
| **Related**      | none yet |
| **Created**      | 2026-08-04 |
| **Last Reviewed**| 2026-08-05 |

## Change log

| Version | Date       | Changes                   | Author |
|---------|------------|------------------------------|--------|
| 1.0.0   | 2026-08-04 | Initial evaluation opened | @leonlnj |
| 1.1.0   | 2026-08-05 | Added R1's clearing condition, per documentation-guide.md §3.10's new verdict vocabulary — an Accepted workaround must state what would let it be dropped. Verdict unchanged. | @leonlnj |

---

## 1. The question this eval answers

Can OCI Resource Manager (ORM) replace the Terraform CLI as the execution engine for `terraform/`,
with no loss of interactive state-repair capability?

## 2. How the candidate works

| Aspect | Terraform CLI | OCI Resource Manager |
|---|---|---|
| Config storage location | git working dir, engineer's laptop | **Stack** — HCL uploaded as a snapshot, or linked to a Git repo/path |
| State storage location | `tfstate-<env>` Object Storage bucket, native locking, 1 `backend "oci"` block per layer (`terraform/CLAUDE.md` "Remote state backend") | hosted and locked by ORM per Stack — no backend block to configure |
| Execution environment | engineer's laptop, synchronous | **Job** — queued, asynchronous plan/apply on Oracle-managed compute |
| Authenticates as | `~/.oci/config` profile (`config_file_profile`) — the engineer's own identity | **Resource principal** — an OCI-managed identity bound to the Stack, not the engineer |
| Debugging & output access | local terminal stdout, live | OCI Console job history, or `oci resource-manager job get-job-logs` — pulled after the run completes |
| Network access | engineer's laptop or self-hosted CI runner — needs its own network path (VPN/bastion) to reach private resources | **Private Endpoint** — an ORM-managed interface into your VCN/subnet; a Job reaches private resources with no self-hosted runner |

## 3. Requirements

| ID | Requirement | Use case |
|---|---|---|
| R1 | [Must-Have] Each area authenticates as the principal its layer needs, without granting hub-scoped credentials identity-domain-admin rights | `DEFAULT` for `starx-dev-hub/*`, `STARX_DEV_IDENTITY` for identity/workload layers (`terraform/CLAUDE.md`) |
| R2 | [Must-Have] Two engineers applying the same layer concurrently cannot corrupt its state | more than one engineer already touches the dev tenancy |
| R3 | [Must-Have] State can be repaired in place — import, move, remove, inspect — without destroying and recreating the resource | stale backend `key` from a copied layer; a resource created out-of-band |
| R4 | [Must-Have] `terraform.tfvars` stays committed and reviewable in git, holding identifiers only | PR review is today's only config-change audit trail |
| R5 | [Nice-to-Have] A syntax/type error surfaces without waiting on a queued remote job | authoring a new layer's HCL |
| R6 | [Nice-to-Have] A module resolves via a relative path from more than one layer without copy-pasting | shared modules reused across layers/environments |
| R7 | [Nice-to-Have] A layer applies via CI with IAM-gated approval, not only from a laptop | near-term next step for `terraform/` |
| R8 | [Nice-to-Have] A third-party linter/policy tool runs against a layer's plan in the same pipeline | adding `tflint`/`checkov` later without switching engines |

## 4. Scorecard

Met = confirmed · Gap = confirmed shortfall · Unverified = untested, or a `[Must-Have]` row with no
magnitude · N/A = doesn't apply.

| ID | ORM | Finding | Workaround |
|---|---|---|---|
| R1 | Gap | One resource principal per Stack — the DEFAULT/STARX_DEV_IDENTITY split needs 1 Stack per principal, 2 minimum today, growing with each future principal | **Accepted** — access shifts to per-Stack IAM, isolation guarantee survives. Clears only if the DEFAULT/STARX_DEV_IDENTITY split is ever consolidated to one principal — until then, Stack-per-principal is inherent to ORM's model, not a temporary gap |
| R2 | Met | Manages state and locking natively per Stack (§2's `State storage location` row) — same guarantee, no gap | — |
| R3 | Gap | No command-line interface into a Job — only `import {}` blocks exist, no `state mv`/`rm`/`console` equivalent. 3 of 4 local repair primitives have no ORM equivalent; a bad entry means hand-editing a downloaded state file, or destroy/recreate | **Not accepted** — no safe substitute for stateful resources (domain users, DBs) |
| R4 | Unverified | Whether ORM honors a committed `tfvars` as-is, or forces re-declaring every variable as a Stack Variable (losing PR visibility), is unchecked | — |
| R5 | Gap | `plan`/`validate` run only as a queued Job — ~2–3 min to first log line vs ~5s locally (~25–35x), no live tail | No workaround — trade is an auditable job record for iteration speed |
| R6 | Unverified | Whether a relative `../../modules/<name>` source resolves inside a server-managed Stack working dir is untested | — |
| R7 | Met | Stacks are natively Git-triggerable, and apply rights are an IAM policy statement — no self-built gating layer needed | — |
| R8 | Gap | ORM Jobs run in a managed environment with 0 supported hook points to inject an external binary (linter/policy tool) before/after `plan` — a third-party tool can't share the same job | Run lint as a separate stage, gate ORM's apply manually — two disconnected stages instead of one |

## 5. Strengths the requirements don't capture

- **Zero infrastructure overhead** — ORM hosts state/locking per Stack, removing the `tfstate-<env>`
  bucket and per-layer `backend "oci"` block this repo maintains across 3 envs. Becomes a requirement
  if env/tenancy count grows enough that backend-bootstrap toil becomes the bottleneck.
- **Native OCI IAM integration** — apply rights become one IAM policy statement instead of a
  distributed API key per engineer; offboarding is revoking a grant, not rotating a key. Becomes a
  requirement if a credential-rotation SLA is ever stated.
- **Private network access** — a Job can reach private resources via a Private Endpoint (§2), with no
  self-hosted runner needed inside that network. Becomes a requirement the day a layer applies against
  a no-public-endpoint resource.
- **Drift detection** — a `detect-drift` job exists on-demand (Console/CLI); no native scheduler is
  documented — today nothing runs it at all, so this is still strictly ahead of the status quo, just
  not automatic. Becomes a requirement if unmonitored drift on a Must-Have resource causes an incident.

## 6. Verdict

**Verdict:** No-Go until governance and centralisation is prioritised over speed and convenience, then it calls for a re-evaluation.

Using ORM requires using both terraform and ORM. Using ORM is an additional complexity and has poor DevX as each layer that are created as a stack, needs to be deployed and debugged as a job. While it provide centralised goverance and convenience factors for networking and drift, the cost does not outweigh the benefits for the current state of terraform/ and the current team size.

**Revisit if:** governance and centralisation get prioritised over speed and convenience.

**Decision date:** 2026-08-04

## Appendix: Evidence

| ID | Evidence |
|---|---|
| R1 | [OCI IAM policy reference — `orm-stacks`/`orm-jobs`](https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/resourcemanagerpolicyreference.htm) — confirms per-Stack IAM policy is the real access-control unit |
| R3 | [Creating an Import Job](https://docs.oracle.com/iaas/Content/ResourceManager/Tasks/create-job-import.htm) — confirms the only state-repair path is uploading a full replacement state file, not `state mv`/`rm` |
| R7 | [Creating a Stack from Git](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/Tasks/create-stack-from-csp-git.htm) — confirms Git-linked stacks are a real, supported source |
| §5 strength | [Managing Private Endpoints](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/Tasks/private-endpoints.htm) — confirms private-VCN job execution |
| §5 strength | [Detecting Drift in a Stack](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/Tasks/detect-drift.htm) — confirms on-demand only, no native scheduler found |