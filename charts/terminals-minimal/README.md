# terminals-minimal

Open WebUI **Terminals** as plain, static Kubernetes manifests — **no Helm
templating** (`{{ }}`) anywhere. Every value is written literally, so you can
read exactly what will be created and apply it with either `kubectl` or `helm`.

It deploys **only the orchestrator** (`TERMINALS_BACKEND=kubernetes`): the
orchestrator talks to the Kubernetes API itself and spawns one Open Terminal pod
per user. There is **no operator and no CRD**.

```
Open WebUI ──HTTP (bearer)──▶ orchestrator (this chart) ──creates──▶ per-user terminal pods
```

## Files

```
templates/secret.yaml       # shared bearer API key
templates/rbac.yaml         # ServiceAccount + namespaced Role + RoleBinding
templates/deployment.yaml   # the orchestrator
templates/service.yaml      # ClusterIP Service Open WebUI connects to
templates/NOTES.txt         # printed by `helm install` (ignored by kubectl)
optional-route.yaml         # OpenShift Route — apply only if OWUI is off-cluster
```

Everything is hard-wired to name **`openwebui-terminals`** in namespace
**`open-webui`**. To use a different namespace, find/replace it first:

```bash
grep -rl 'open-webui' charts/terminals-minimal | xargs sed -i 's/open-webui/YOUR_NS/g'
```

## Install

Create the namespace, then apply the manifests. Pick either tool:

```bash
kubectl create namespace open-webui

# with kubectl (NOTES.txt is skipped automatically):
kubectl apply -f charts/terminals-minimal/templates/

# ...or with helm (same static objects):
helm install terminals ./charts/terminals-minimal -n open-webui
```

> **Set your own API key** before (or right after) installing. Edit
> `templates/secret.yaml` and replace the `api-key` value, e.g. with
> `openssl rand -hex 24`. The shipped value is a placeholder default.

## Connect it to your Open WebUI instance

Open WebUI discovers terminal servers through the `TERMINAL_SERVER_CONNECTIONS`
environment variable — a JSON array pointing at the orchestrator with the key.

**1. Read the key:**

```bash
kubectl -n open-webui get secret openwebui-terminals-api-key \
  -o jsonpath='{.data.api-key}' | base64 -d ; echo
```

**2. Set the env var on your Open WebUI deployment** (same cluster → use the
in-cluster Service DNS; substitute `<API_KEY>`):

```bash
kubectl -n open-webui set env deployment/open-webui \
  TERMINAL_SERVER_CONNECTIONS='[{"id":"terminals","name":"Terminals","enabled":true,"url":"http://openwebui-terminals.open-webui.svc.cluster.local:8080","key":"<API_KEY>","auth_type":"bearer","config":{"access_grants":[{"principal_type":"user","principal_id":"*","permission":"read"}]}}]'
```

Restart Open WebUI, open a chat, and use the Terminal tool. Watch pods appear:

```bash
kubectl -n open-webui get pods -l app.kubernetes.io/component=terminal -w
```

## OpenShift notes

Built for OpenShift's **`restricted-v2`** SCC — no `anyuid`, no privileged pods.

* **No hard-coded UIDs.** Neither the orchestrator nor the spawned pods set a
  numeric `runAsUser`/`fsGroup`, so OpenShift assigns them from the namespace
  range. `TERMINALS_KUBERNETES_RESTRICTED=true` makes the spawned pods use
  `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`
  and `RuntimeDefault` seccomp — exactly what `restricted-v2` requires.
* **Storage.** `TERMINALS_KUBERNETES_STORAGE_CLASS` is left unset so terminal
  PVCs use the cluster's default StorageClass. Uncomment it in
  `templates/deployment.yaml` to pin a class.
* **External Open WebUI.** If Open WebUI runs outside this cluster, apply the
  Route and use `https://<route-host>` as the `url` above:

  ```bash
  oc apply -f charts/terminals-minimal/optional-route.yaml
  oc -n open-webui get route openwebui-terminals -o jsonpath='{.spec.host}' ; echo
  ```

## Changing settings

Because there is no templating, you configure this by **editing the YAML**:

| Want to change | Edit |
|----------------|------|
| API key | `templates/secret.yaml` → `stringData.api-key` |
| Terminal image / idle timeout / storage size | `templates/deployment.yaml` env vars |
| Pin a StorageClass | uncomment `TERMINALS_KUBERNETES_STORAGE_CLASS` |
| Namespace / names | find-replace `open-webui` / `openwebui-terminals` |
| Expose externally | apply `optional-route.yaml` |
