# ADR-NN: Adopt the Network Observability Operator to identify the source and destination of outbound traffic

| **Metadata**       | **Value**                                          |
|--------------------|-----------------------------------------------------|
| **Status**         | Proposed                                            |
| **Authors**        | @wenbierrr                                          |
| **Decision Date**  | 2026-09-08                                          |
| **Source**         | [noo-101.md](noo-101.md)                            |

---

## Context

Outbound network volume from the cluster is rising and the cause cannot be identified. Every pod is source-NAT'd to its node address before the traffic leaves, so firewall logs, proxy logs and node-level metrics all attribute at best to "some workload on node X". What is needed is the ability to name both ends of outbound traffic: which workload sent it, and where it went. With both ends named, a spike stops being something to wait out and becomes something to act on — a workload found to be sending traffic we do not need can be scaled down on our side, reducing the volume leaving the cluster.

## Decision

**Adopt NOO. It names the source of outbound traffic down to the workload, and the destination down to a named subnet — not to an address.**

- **Source is solved.** eBPF captures flows in-kernel, before source-NAT rewrites the address, so traffic carries namespace, owner name and owner kind.
- **Destination is solved with a caveat.** It resolves to a *subnet label*, not an IP, so a destination is identifiable only if its CIDR has been mapped to a name by hand. Unmapped external traffic is still detected and measured, but anonymous.
- **Both ends arrive on the same record.** Source and destination are labels on a single metric series, so a spike can be traced from the workload that produced it through to where it went — not left as two separate facts with no way to join them.
- **It runs on infrastructure already present** — the platform Prometheus. No Loki, no Kafka, no object storage.

## Consequences

### Positive:

- **Spikes stop being guesswork.** The workload driving an outbound spike can be named while it is still happening, from the console or from PromQL.
- **Accurate enough to act on.** Byte counts tracked ground truth to within 3% on the evaluation cluster — the gap is TCP/IP header overhead, not error.
- **No burden on service owners.** Capture is in-kernel: no sidecars, no instrumentation, no redeploys, nothing to coordinate with app teams.

### Trade-offs we accept:

- **No requests per second, no per-request size.** NOO counts connections, not requests — ten keep-alive requests on one connection are one flow. No configuration changes this.
- **Destination naming is manual.** The CIDR-to-name map is **maintained by hand** and drifts unless someone owns it.
- **Sampling trades exactness for cost.** The default captures 1-in-50 packets — sound for ranking and trend, not for exact totals. `sampling: 1` is exact but raises agent CPU and memory.
- **Without Loki** there are no individual flow records, no per-pod filtering and no packet-drop statistics. "Which workload is sending the most out of the cluster" is answerable; "show me the exact connection that failed at 14:02" is not.
- **Without Kafka** there is no buffer — in-flight flows are lost, not queued if the pipeline restarts or falls behind.
- **Retention is the cluster-wide Prometheus setting**
- **Proxies and `hostNetwork` hide the workload.** Traffic through an Istio egress gateway or an OpenShift Route is named as the gateway or router; a `hostNetwork` pod sends using the node's own IP address, so it shows up as the node instead of as a workload.

### What this triggers:

STARFORGE is able to deploy NOO quickly whenever there is a need to identify who is generating outbound network traffic.

**If NOO is brought in:**

- Mirror the images. 
- Map the destinations manually. The CIDR blocks of known external endpoints must be identified and given names by hand in the `FlowCollector` CR, under `customLabels`.