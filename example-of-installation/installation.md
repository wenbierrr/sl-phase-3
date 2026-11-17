# Installing Filestash

Deploy Filestash on OpenShift via ArgoCD/GitOps (recommended) or on a VM via Podman.

> **Istio users:** If your OpenShift cluster uses Istio for ingress or egress, you will also need the [Istio appendix](#appendix-istio-configuration-optional). Read it before you start so you can prepare the TLS certificate and ServiceEntry ahead of time.

---

## Onboarding Process

Before deploying Filestash, complete the following steps with the Starforge team:

1. **Register the requirement** — Submit a request through Starforge to register a new
   Filestash instance. Include:
   - Target namespace and owning team
   - Backend mode required (`local` or `s3`)
   - Expected storage size (PVC)
   - Use case / justification

2. **Obtain approval** — The Starforge team reviews the request and confirms resource
   availability.

3. **Update the tracking spreadsheet** — Once approved, record the instance in the
   **Spring Planning Excel Sheet, stardrive migration tab**:
   - Namespace, team, backend mode, storage class, PVC size
   - Planned deployment date and responsible engineer

4. **Proceed with deployment** — Only after approval and spreadsheet update, continue
   to the deployment steps below.

---

## Which path should I follow?

**1. Are you deploying on OpenShift?**

| Answer | Next step |
|--------|-----------|
| Yes | Continue below (§1–§3) |
| No (VM / bastion host) | Go to [§4 VM / Podman](#4-vm--podman) |

**2. Which storage backend do you need?**

| If you need… | Set `backend` to | Go to |
|--------------|------------------|-------|
| A shared filesystem that all users browse (most common) | `local` | [§1 Local Backend](#1-openshift--local-backend) |
| S3-compatible object storage (Pure FlashBlade, MinIO, AWS) | `s3` | [§2 S3 Backend](#2-openshift--s3-backend) |
| Both at the same time | `both` | [§3 Both Backends](#3-openshift--both-backends) |

---

## ArgoCD / GitOps Deployment Workflow

All OpenShift deployments follow the standard Starforge ArgoCD onboarding process. The steps below are a summary. For full details, refer to **`starlab-operations/starforge-documentation/day-to-day/argo.md`**.

### Step 1 — Upload the Helm chart

Push the Filestash Helm chart (from `services/stardrive/helm/`) to the **`helm-charts`** repository in GitLab.

### Step 2 — Add overlay values to ArgoHub

Add the overlay values file for your backend mode to **`starlab-operations/argohub`**:

- Local backend: use `helm/overlays/values-local.yaml` as the base, customise for your namespace (storage class, PVC size)
- S3 backend: use `helm/overlays/values-s3.yaml`, ensure the S3 Secret is pre-created in the target namespace
- Both backends: combine settings from both overlays into a single values file

See the backend-specific sections below ([§1](#1-openshift--local-backend), [§2](#2-openshift--s3-backend), [§3](#3-openshift--both-backends)) for what to configure in the overlay.

### Step 3 — Create the Application CRD

Create the ArgoCD Application manifest in the **`argocd/application`** folder. The Application should reference the Helm chart from Step 1 and the overlay values from Step 2.

### Step 4 — Create a merge request

Create a merge request (MR) in GitLab for the changes from Steps 1–3.

| Environment | MR requirement |
|-------------|---------------|
| **Production** | MR must be merged to `main` |
| **Staging (first deployment)** | MR must be merged to `main` |
| **Staging (subsequent updates)** | MR to `main` is not required |

### Step 5 — Create the App of Apps (AoA) in ArgoCD

In the ArgoCD UI, create the AoA entry and copy the `aoa.md` content from Step 3.

### Step 6 — Diff and sync

1. **Diff & sync the AoA** — The AoA sync should show the Application CRD being created
2. **Diff the Application** — Press **Diff** first and review all changes before syncing
3. **Second-person validation** — Have a second person review and approve the diff
4. **Sync the Application** — Once approved, sync to deploy Filestash to the target namespace

After sync completes, go to [§5 Post-install](#5-post-install) for admin setup.

> For the complete ArgoCD onboarding walkthrough with screenshots and troubleshooting, refer to **`starlab-operations/starforge-documentation/day-to-day/argo.md`**.

---

## Prerequisites

**All OpenShift deployments:**
- Access to GitLab and the ArgoCD UI
- `oc` CLI for verification
- Namespace created in the target cluster
- Familiarity with the ArgoCD onboarding process (see `argo.md`)

**For S3 backend:**
- An S3-compatible endpoint: AWS S3, MinIO, Pure FlashBlade, or similar
- Access key ID and secret access key for that endpoint
- S3 Secret pre-created in the target namespace before syncing

> **Running on an Istio-enabled cluster?** You will also need to apply Gateway, VirtualService, AuthorizationPolicy, and optionally egress resources. Prepare the TLS certificate Secret before starting. See [Appendix: Istio Configuration](#appendix-istio-configuration-optional).

> **Note on PVCs:** A state PVC (`<release>-pvc`, 5 Gi) is **always created** by the chart regardless of backend mode. It stores Filestash config, the internal database, and session state. PVCs are annotated with `helm.sh/resource-policy: keep`, so they survive `helm uninstall` and ArgoCD app deletion. See [PVC Lifecycle & Cleanup](#pvc-lifecycle--cleanup) for manual cleanup steps.

---

## §1 OpenShift — Local Backend

This is the most common deployment. Users browse a shared filesystem mounted from a PersistentVolume.

**What the chart creates:**

| Resource | Name | Purpose |
|----------|------|---------|
| PVC (state) | `<release>-pvc` (5 Gi) | Config, DB, sessions |
| PVC (data) | `<release>-local-data` (100 Gi default) | User files |

### Configure the overlay

Use `helm/overlays/values-local.yaml` as the starting point. Customise for your namespace:

```yaml
backend: local

localMount:
  mountPath: /mnt/data
  persistence:
    existingClaim: ""                  # Set to reuse an existing PVC
    storageClassName: "<your-sc>"      # REQUIRED: set to your storage class
    accessMode: ReadWriteMany
    size: 100Gi
```

### Deploy

Follow the [ArgoCD / GitOps Deployment Workflow](#argocd--gitops-deployment-workflow) above. In Step 2, add this overlay to ArgoHub.

> If your cluster uses Istio, also apply the manifests in the [Istio appendix](#appendix-istio-configuration-optional) before proceeding.

Then go to [§5 Post-install](#5-post-install).

---

## §2 OpenShift — S3 Backend

**What the chart creates:**

| Resource | Name | Purpose |
|----------|------|---------|
| PVC (state) | `<release>-pvc` (5 Gi) | Config, DB, sessions |

No local-data PVC is created.

### Create the S3 credentials Secret

The Secret must exist in the target namespace **before** the ArgoCD Application is synced.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-secret
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: xxx
  AWS_SECRET_ACCESS_KEY: xxx
  AWS_ENDPOINT_URL: xxx
```

Or via CLI:

```bash
oc create secret generic s3-secret \
  --from-literal=AWS_ACCESS_KEY_ID=<key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<secret> \
  --from-literal=AWS_ENDPOINT_URL=https://<s3-endpoint> \
  -n <namespace>
```

### Self-signed or private-CA HTTPS endpoints (optional)

If your S3 endpoint (`AWS_ENDPOINT_URL`) uses a certificate signed by a private CA or a self-signed cert, the Filestash container will fail to connect with an `x509: certificate signed by unknown authority` error. Fix this by mounting your CA cert over the container's Alpine trust store (`/etc/ssl/certs/ca-certificates.crt`).

> **Note:** The PEM file you provide replaces the container's full system trust store. If the container also needs to reach other standard HTTPS endpoints, concatenate your custom CA with the Alpine default bundle before creating the ConfigMap.

**Step 1 — Create a ConfigMap with the PEM-encoded CA cert:**

```bash
oc create configmap s3-ca-bundle \
  --from-file=ca-certificates.crt=./ca-bundle.pem \
  -n <namespace>
```

> The ConfigMap must exist in the same namespace as the Filestash deployment before installing/upgrading. The key name must be `ca-certificates.crt`.

**Step 2 — Enable `caBundle` in your overlay values:**

```yaml
caBundle:
  enabled: true
  configMapName: s3-ca-bundle   # must match the ConfigMap name above
```

The chart mounts the ConfigMap key `ca-certificates.crt` directly over `/etc/ssl/certs/ca-certificates.crt` (read-only), so Go's TLS stack trusts your custom CA without any additional environment variables.

### Configure the overlay

Use `helm/overlays/values-s3.yaml` as the starting point:

```yaml
backend: s3

secret:
  enable: true
  name: s3-secret

# Add this block if your endpoint uses a self-signed cert (see above):
# caBundle:
#   enabled: true
#   configMapName: s3-ca-bundle
```

### Deploy

Follow the [ArgoCD / GitOps Deployment Workflow](#argocd--gitops-deployment-workflow) above. In Step 2, add this overlay to ArgoHub.

> If your cluster uses Istio, also apply the manifests in the [Istio appendix](#appendix-istio-configuration-optional) before proceeding.

Then go to [§5 Post-install](#5-post-install).

---

## §3 OpenShift — Both Backends

S3 and local storage active simultaneously. Users see a backend selector in the UI.

> **Note:** This mode is not commonly used today. It is available for namespaces that require both S3 and local filesystem access.

**What the chart creates:**

| Resource | Name | Purpose |
|----------|------|---------|
| PVC (state) | `<release>-pvc` (5 Gi) | Config, DB, sessions |
| PVC (data) | `<release>-local-data` (100 Gi default) | User files |

### Prerequisites

1. Create the S3 Secret as described in [§2](#create-the-s3-credentials-secret)
2. If your S3 endpoint uses a self-signed or private-CA cert, also follow the [CA bundle steps in §2](#self-signed-or-private-ca-https-endpoints-optional) before deploying.
3. Prepare a combined overlay values file with both local and S3 settings:

```yaml
backend: both

secret:
  enable: true
  name: s3-secret

localMount:
  mountPath: /mnt/data
  persistence:
    existingClaim: ""
    storageClassName: "<your-sc>"
    accessMode: ReadWriteMany
    size: 100Gi
```

### Deploy

Follow the [ArgoCD / GitOps Deployment Workflow](#argocd--gitops-deployment-workflow) above. In Step 2, add the combined overlay to ArgoHub.

> If your cluster uses Istio, also apply the manifests in the [Istio appendix](#appendix-istio-configuration-optional) before proceeding.

Then go to [§5 Post-install](#5-post-install).

---

## §4 VM / Podman

For deploying on a bastion or RHEL host using Podman. See [installing-filestash-vm.md](./installing-filestash-vm.md) for the full step-by-step walkthrough (directory setup, systemd service, and OAuth2-Proxy auth option).

The key per-backend differences are:

| Backend | `/etc/filestash/.env` | Bind mounts in `run-filestash.sh` |
|---------|----------------------|----------------------------------|
| `s3` only | Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL` | State volume + CA cert mount (if FlashBlade) |
| `local` only | Leave `AWS_*` variables empty or omit them | State volume + `/mnt/data` host path |
| `both` | Set all `AWS_*` variables | State volume + `/mnt/data` + CA cert mount |

After starting the container, Filestash is available at `http://<host-ip>:8334`. Then follow [§5 Post-install](#5-post-install).

---

## §5 Post-install

**1. Get the route URL (OpenShift only):**

```bash
oc get route -n <namespace>
```

**2. Complete admin setup:**

Follow [admin-setup.md](./admin-setup.md) to:
- Set the one-time admin password at `<host>/admin`
- **(Managed mode)** Verify pre-seeded backend connections in **Admin > Connections** so users see a CONNECT-only form
- **(Self-service mode)** Skip connection configuration — users will enter their own credentials at login
- **(All modes)** Set `upload_chunk_size` to `128` in **Admin > Settings** — required so large uploads survive ingress disruptions

**3. Optional — Istio ingress RBAC and secure egress:**

If your cluster uses Istio for ingress control or requires HTTP to HTTPS upgrade on egress to Pure FlashBlade, see [Appendix: Istio Configuration](#appendix-istio-configuration-optional) below.

---

## PVC Lifecycle & Cleanup

Both PVCs created by this chart are annotated with `helm.sh/resource-policy: keep`. This means:

- PVCs **survive** `helm uninstall` and ArgoCD app deletion
- This is intentional — it prevents accidental data loss on upgrade or redeployment
- PVCs must be **manually deleted** when the Filestash instance is permanently decommissioned

### Cleanup steps

After confirming that Filestash data is no longer needed:

```bash
NAMESPACE="<namespace>"
RELEASE="<release-name>"

# Delete the state PVC
oc delete pvc/${RELEASE}-pvc -n $NAMESPACE

# Delete the local data PVC (if backend was local or both)
oc delete pvc/${RELEASE}-local-data -n $NAMESPACE
```

If you plan to reuse the PVC data with a new release, set `persistence.existingClaim` or `localMount.persistence.existingClaim` in the overlay values file instead of deleting the PVCs.

---

## Appendix: Istio Configuration (optional)

Istio is used for RBAC on ingress and for HTTP to HTTPS upgrade on egress to Pure FlashBlade. This configuration applies to all backend modes on OpenShift. It is not required for basic operation or VM deployments.

### A.1 Ingress RBAC

#### Prerequisites
- A Secret containing the TLS cert for the Filestash hostname must exist.

**Ingress Gateway**
```yaml
- hosts:
    - filestash.svc.example.com
  port:
    name: filestash
    number: 443
    protocol: HTTPS
  tls:
    credentialName: filestash-cert
    mode: SIMPLE
```

**VirtualService**
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: filestash-ingress-vs
  namespace: filestash
spec:
  hosts:
    - filestash.svc.example.com
  gateways:
    - istio-system/istio-ingressgateway
  http:
    - match:
        - uri:
            prefix: /
      retries:
        attempts: 0   # Istio defaults to 2, which makes the gateway buffer each upload (~250 MiB)
      route:
        - destination:
            host: filestash.filestash.svc.cluster.local
            port:
              number: 8334
```

**AuthorizationPolicy**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-filestash-authz
  namespace: filestash
spec:
  selector:
    matchLabels:
      app: filestash
  action: ALLOW
  rules:
    - to:
        - operation:
            hosts:
              - filestash.svc.example.com
            notPaths:
              - /admin/*
            paths:
              - /*
      when:
        - key: request.auth.claims[iss]
          values:
            - https://sso.example.com
        - key: request.auth.claims[role]
          values:
            - superadmin
            - filestash-reader
    - to:
        - operation:
            hosts:
              - filestash.svc.example.com
            paths:
              - /admin/*
      when:
        - key: request.auth.claims[iss]
          values:
            - https://sso.example.com
        - key: request.auth.claims[role]
          values:
            - superadmin
```

### A.2 Secure Egress (S3 mode only)

This section applies when using the S3 backend with Pure FlashBlade. Traffic from the Filestash pod is routed through the egress gateway, where TLS origination upgrades it from HTTP to HTTPS.

**Egress Gateway**
```yaml
- hosts:
    - s3.example.com
  port:
    name: flashblade
    number: 80
    protocol: HTTP
```

**ServiceEntry**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: pureflash-external
spec:
  hosts:
  - s3.example.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

**VirtualService**
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: filestash-egress-vs
  namespace: filestash
spec:
  hosts:
    - s3.example.com
  gateways:
    - istio-system/istio-egressgateway
    - mesh
  http:
    - match:
        - gateways:
            - mesh
          port: 80
      route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            port:
              number: 80
    - match:
        - gateways:
            - istio-system/istio-egressgateway
          port: 80
      route:
        - destination:
            host: s3.example.com
            port:
              number: 443
```

**DestinationRule**
```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: originate-tls-to-flashblade
spec:
  host: s3.example.com
  trafficPolicy:
    portLevelSettings:
      - port:
          number: 443
        tls:
          credentialName: jpe-ca
          mode: SIMPLE
```
