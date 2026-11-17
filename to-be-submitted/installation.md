# Installing the Kubernetes MCP server

Deploy the Kubernetes MCP server on OpenShift via Argo CD/GitOps, read-only and rate-limited at the kube-apiserver. For what each piece is and why, see the [service design](../design/service-design.md).

| | |
|---|---|
| **End State** | One Deployment and one read-only ClusterRole, plus an APF rate limit. LibreChat is installed separately |

---

## 1. Prerequisites

1. LibreChat is installed: see its [installation docs]().
2. The Kubernetes MCP server image is in the image registry (Quay).
3. The upstream Helm chart (`charts/kubernetes-mcp-server` in [containers/kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)) is in the `helm-charts` repository.

## 2. Configure the overlay file

Copy the [recipe](../recipes/values-sf.yaml) and change these values for your cluster:

| Field |
|---|
| `image.registry` |
| `image.repository` | 
| `image.version` | 


Leave everything else as it is. In particular, `config.read_only: true` and `rbac.extraClusterRoles` keep the server read-only.


## 3. Add the API rate limit (APF)

Without this, the MCP server shares the built-in `workload-low` priority level with every other ServiceAccount in the cluster, so a looping or prompt-injected LLM could starve operators and controllers of API access. This gives it its own priority level with a hard cap.

### How it limits

APF limits **concurrency** (requests in flight at once), not requests per second:

1. Up to **16** requests run at once (the figure on CRC; see *Sizing*).
2. Up to **20** more wait in a queue.
3. A request gets **HTTP 429** if the queue is full, or if it waits about 15 seconds without a free slot. The MCP server retries 429s, so a DE first sees slower answers and only gets a tool error under a sustained flood.

Watches and pod logs are long-running requests and aren't counted. The MCP server also throttles itself first (5 requests/s, burst 10 by default); APF is the cap that holds even if that is raised or bypassed.

### The manifest

Add this to the chart as `templates/apf.yaml`, so Argo CD deploys and removes it together with the server:

```yaml
{{- $sa := include "kubernetes-mcp-server.serviceAccountName" . }}
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: {{ $sa }}
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 1
    borrowingLimitPercent: 0
    lendablePercent: 0
    limitResponse:
      type: Queue
      queuing:
        queues: 1
        handSize: 1
        queueLengthLimit: 20
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: {{ $sa }}
spec:
  priorityLevelConfiguration:
    name: {{ $sa }}
  matchingPrecedence: 8000
  distinguisherMethod:
    type: ByUser
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: {{ $sa }}
            namespace: {{ .Release.Namespace }}
      resourceRules:
        - verbs: ["*"]
          apiGroups: ["*"]
          resources: ["*"]
          namespaces: ["*"]
          clusterScope: true
      nonResourceRules:
        - verbs: ["*"]
          nonResourceURLs: ["*"]
```

| Field | Why |
|---|---|
| `nominalConcurrencyShares: 1` | The smallest share, so the fewest concurrent requests |
| `borrowingLimitPercent: 0` | A hard cap: it can't borrow idle capacity from other levels |
| `lendablePercent: 0` | Keeps its own capacity. It is too small to be worth lending |
| `queueLengthLimit: 20` | A legitimate burst waits instead of failing. Overflow gets 429 |
| `queues: 1`, `handSize: 1` | One ServiceAccount is one flow, so one queue is enough |
| `matchingPrecedence: 8000` | Below 9000, so it wins over the built-in `service-accounts` FlowSchema |

### Sizing

The cap depends on each cluster's API server settings:

```
concurrent requests = ceil((max-requests-inflight + max-mutating-requests-inflight) × shares ÷ total shares of all priority levels)
CRC:                = ceil((3000 + 1000) × 1 ÷ 256) = 16
```

Check the real value on every cluster after deploying (§6). On a cluster with several control-plane nodes, the limit applies **per kube-apiserver instance**.

## 4. Deploy via Argo CD

Follow the standard Starforge ArgoCD onboarding process: **`starlab-operations/starforge-documentation/day-to-day/argo.md`**. In short:

1. Add `templates/apf.yaml` (§3) to the chart in the **`helm-charts`** repository.
2. Add the overlay values from §2 to **`starlab-operations/argohub`**.
3. Create the Application in the **`argocd/application`** folder, with:
   - Destination namespace **`k8-mcp-server`**
   - Helm release name **`kubernetes-mcp-server`**. It becomes the ServiceAccount and Service name, which the rate limit and LibreChat's MCP URL (§5) rely on.
4. Raise an MR, then create the App of Apps entry.
5. **Diff**, have a **second person review it**, then **sync**.

> ⚠️ The chart creates cluster-scoped objects: a ClusterRole, a ClusterRoleBinding, a FlowSchema and a PriorityLevelConfiguration. The Argo CD project must allow them, or the sync fails with "not permitted in project".

## 5. Connect LibreChat

Add the server to `librechat.configYamlContent` in LibreChat's values in git, then sync LibreChat:

```yaml
mcpSettings:
  allowedAddresses:          # LibreChat blocks in-cluster MCP addresses unless listed here
    - "kubernetes-mcp-server.k8-mcp-server.svc.cluster.local:8080"
mcpServers:
  kubernetes:
    type: streamable-http
    url: "http://kubernetes-mcp-server.k8-mcp-server.svc.cluster.local:8080/mcp"
    timeout: 120000
    serverInstructions: |
      # The system prompt — see below
```

`serverInstructions` is the system prompt: it sets how the LLM investigates and the format it answers in. Its rules are summarised in [service design §5](../design/service-design.md#the-system-prompt). LibreChat restarts automatically when `configYamlContent` changes.

## 6. Verify

```bash
SA=system:serviceaccount:k8-mcp-server:kubernetes-mcp-server

oc get pods -n k8-mcp-server
oc auth can-i list secrets -A --as=$SA
oc auth can-i delete pods  -A --as=$SA
oc auth can-i list pods    -A --as=$SA
oc get flowschema kubernetes-mcp-server -o jsonpath='{.metadata.uid}'; echo
oc get namespaces --as=$SA -v=8 2>&1 | grep -i x-kubernetes-pf-flowschema-uid
oc get --raw /metrics | grep 'apiserver_flowcontrol_nominal_limit_seats{.*priority_level="kubernetes-mcp-server"'
```

| Check | Want to see |
|---|---|
| Pod | `1/1 Running` |
| `list secrets` / `delete pods` | `no` / `no` |
| `list pods` | `yes` |
| FlowSchema UID | The same UID on both lines: the server's requests land in its own priority level |
| `nominal_limit_seats` | This cluster's concurrency cap (16 on CRC) |

Then, in LibreChat, ask: *"List the pods in namespace k8-mcp-server."* If the answer names the MCP server's own pod, the whole chain works.

Full proof, including requests made with the real token and a flood test that shows the 429s: `verify-mcp-rbac.sh` and `verify-mcp-apf.sh`.

## 7. Change or remove

| To | Do |
|---|---|
| Change configuration, RBAC or the image | Edit the chart or overlay in git, raise an MR, then diff, review and sync. A change made in the console shows as **OutOfSync** in Argo CD |
| Grant read access to a new API group | Add a rule under `rbac.extraClusterRoles[0].rules`, sync, then re-run `verify-mcp-rbac.sh` |
| Remove the rate limit only | Delete `templates/apf.yaml` and sync with **prune**. The ServiceAccount falls back to `workload-low` immediately |
| Uninstall | Remove the `kubernetes` entry from LibreChat's config first, then delete the Application |

---
 🗓️ *Last Updated: 17/11/2026*
 ⚠️ *Tested on: OpenShift Local (CRC), kubernetes-mcp-server chart 0.1.0 (upstream commit `597120c`).*
