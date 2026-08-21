# terminals-minimal

A deliberately small Helm chart that runs **Open WebUI Terminals** with as
little templating and as few moving parts as possible.

The official [`terminals`](../terminals) chart ships a Kubernetes **operator**,
a **CRD** (`terminals.openwebui.com`) and an **orchestrator**. This chart drops
the operator and the CRD entirely: it deploys **only the orchestrator**, running
with `TERMINALS_BACKEND=kubernetes`, so the orchestrator itself talks to the
Kubernetes API and spawns one Open Terminal pod per user on demand.

```
Open WebUI  ──HTTP (bearer)──▶  orchestrator (this chart)  ──creates──▶  per-user terminal pods
```

What gets installed:

| Resource | Purpose |
|----------|---------|
| Deployment + Service | the orchestrator Open WebUI connects to |
| ServiceAccount + **Role** + RoleBinding | lets the orchestrator create terminal pods **in its own namespace** (namespaced Role, no ClusterRole) |
| Secret | the shared bearer API key (auto-generated if not supplied) |
| Route *(optional)* | OpenShift Route, only if Open WebUI lives outside the cluster |

No operator, no CRD, no cluster-scoped RBAC.

## Install

```bash
helm install terminals ./charts/terminals-minimal \
  --namespace open-webui --create-namespace
```

On first install the chart generates a random API key. It is **preserved across
`helm upgrade`** (it is read back from the existing Secret), so the connection to
Open WebUI does not break on upgrades. To pin your own key:

```bash
helm install terminals ./charts/terminals-minimal -n open-webui \
  --set apiKey=$(openssl rand -hex 24)
```

## Connect it to your Open WebUI instance

Open WebUI discovers terminal servers through the `TERMINAL_SERVER_CONNECTIONS`
environment variable — a JSON array pointing at the orchestrator with the shared
key.

**1. Read the API key:**

```bash
kubectl -n open-webui get secret terminals-terminals-minimal-api-key \
  -o jsonpath='{.data.api-key}' | base64 -d ; echo
```

**2. Set this env var on your Open WebUI deployment** (same cluster → use the
in-cluster Service DNS name; substitute your `<API_KEY>`):

```json
[
  {
    "id": "terminals",
    "name": "Terminals",
    "enabled": true,
    "url": "http://terminals-terminals-minimal.open-webui.svc.cluster.local:8080",
    "key": "<API_KEY>",
    "auth_type": "bearer",
    "config": { "access_grants": [ { "principal_type": "user", "principal_id": "*", "permission": "read" } ] }
  }
]
```

As a one-liner for a raw Deployment:

```bash
kubectl -n open-webui set env deployment/open-webui \
  TERMINAL_SERVER_CONNECTIONS='[{"id":"terminals","name":"Terminals","enabled":true,"url":"http://terminals-terminals-minimal.open-webui.svc.cluster.local:8080","key":"<API_KEY>","auth_type":"bearer","config":{"access_grants":[{"principal_type":"user","principal_id":"*","permission":"read"}]}}]'
```

If you deploy Open WebUI with **its own Helm chart**, you do not need this at all —
set `terminals.enabled=true` there and it wires up the bundled subchart. This
chart is for the case where you want Terminals **standalone / minimal** and
connect it to an Open WebUI you already run.

Open a chat in Open WebUI afterwards and use the Terminal tool — watch the pods
appear:

```bash
kubectl -n open-webui get pods -l app.kubernetes.io/component=terminal -w
```

## OpenShift notes

This chart is built for OpenShift's **`restricted-v2`** SCC — no `anyuid`, no
extra SCC, no privileged pods.

* **No hard-coded UIDs.** Neither the orchestrator nor the spawned terminal pods
  set a numeric `runAsUser`/`fsGroup`, so OpenShift assigns them from the
  namespace's UID range. `restricted: true` (default) makes the orchestrator
  spawn terminal pods with `runAsNonRoot`, `allowPrivilegeEscalation: false`,
  `capabilities.drop: [ALL]` and the `RuntimeDefault` seccomp profile — exactly
  what `restricted-v2` requires.
* **Storage.** Leave `persistence.storageClass` empty to use the cluster's
  default StorageClass (recommended on OpenShift/ROSA/ARO).
* **External Open WebUI.** If Open WebUI runs outside this cluster, expose the
  orchestrator with a Route:

  ```bash
  helm upgrade terminals ./charts/terminals-minimal -n open-webui \
    --set route.enabled=true
  oc -n open-webui get route terminals-terminals-minimal -o jsonpath='{.spec.host}'
  ```

  Then use `https://<that-host>` as the `url` in `TERMINAL_SERVER_CONNECTIONS`.

## Values

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` / `image.tag` | `ghcr.io/open-webui/terminals` / `latest` | Orchestrator image |
| `terminalImage` | `ghcr.io/open-webui/open-terminal:latest` | Image for each spawned terminal pod |
| `service.type` / `service.port` | `ClusterIP` / `8080` | Orchestrator Service |
| `apiKey` | `""` (auto-generated) | Shared bearer key |
| `existingSecret` | `""` | Use an existing Secret with key `api-key` |
| `idleTimeoutMinutes` | `30` | Idle terminal cleanup (0 = never) |
| `persistence.enabled` / `.size` / `.storageClass` / `.mode` | `true` / `1Gi` / `""` / `per-user` | Per-terminal storage |
| `restricted` | `true` | OpenShift/PSA-restricted security context for spawned pods |
| `terminalLimits.cpu` / `.memory` / `.storage` | `""` | Optional hard caps per terminal |
| `resources` | requests 50m/128Mi, limits 500m/256Mi | Orchestrator pod resources |
| `route.enabled` / `.host` / `.tls` | `false` / `""` / `true` | Optional OpenShift Route |
| `nodeSelector` / `tolerations` / `affinity` | `{}` / `[]` / `{}` | Orchestrator scheduling |

> Resource names above assume the release is named `terminals`. They follow the
> pattern `<release>-terminals-minimal`.
