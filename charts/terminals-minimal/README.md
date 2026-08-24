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
optional-limitrange.yaml    # LimitRange — apply only if the namespace quota needs it
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

Use the **literal** key value — do **not** use `"key":"$(TERMINALS_API_KEY)"`
unless you also define a `TERMINALS_API_KEY` env var on the Open WebUI pod, or
the unexpanded string is sent and every request is rejected.

Restart Open WebUI, open a chat, and use the Terminal tool. Watch pods appear:

```bash
kubectl -n open-webui get pods -l app.kubernetes.io/component=terminal -w
```

### 401 on `/ports`, `/api/terminals`, `/files/*`

The orchestrator runs in API-key mode: every proxied endpoint needs
`Authorization: Bearer <key>` matching the Secret. A 401 means the `key` in
`TERMINAL_SERVER_CONNECTIONS` doesn't match. Confirm the orchestrator accepts
the Secret's key (prints `200`; without the header it prints `401`):

```bash
KEY=$(kubectl -n open-webui get secret openwebui-terminals-api-key -o jsonpath='{.data.api-key}' | base64 -d)
kubectl -n open-webui run keytest --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $KEY" -H "X-User-Id: selftest" \
  http://openwebui-terminals.open-webui.svc.cluster.local:8080/ports
```

If that prints `200`, the orchestrator is fine — fix the `key` on the Open WebUI
side to match.

### 403 `must specify limits.cpu/limits.memory` when a terminal spawns

```
pods "terminal-xxxx" is forbidden: failed quota: ...: must specify
limits.cpu for: open-terminal; limits.memory for: open-terminal
```

The namespace has a **ResourceQuota that requires resource limits**, but the
orchestrator spawns terminal pods without any. `TERMINALS_MAX_CPU`/
`TERMINALS_MAX_MEMORY` only *cap* limits that already exist — they don't add
them. Supply defaults with the bundled **LimitRange**, which injects
limits/requests into containers that omit them before the quota is checked:

```bash
oc apply -n open-webui -f charts/terminals-minimal/optional-limitrange.yaml
```

Match `-n` to the namespace where terminals run, and tune the numbers to the
namespace's per-container quota max. It only affects containers that don't set
their own limits, so Open WebUI's pods are untouched.

### Terminal pod crashes: `mkdir: cannot create directory '//.local'`

The `open-terminal` image runs as its build user (uid 1000, `HOME=/home/user`)
and sets no `ENV HOME`. Under the default `restricted-v2` SCC, OpenShift runs the
pod as an arbitrary UID with no passwd entry, so `$HOME` is empty and the
entrypoint fails writing to `/home/user`.

The deployment pins the terminal to uid/fsGroup **1000** (via
`TERMINALS_KUBERNETES_POD_SECURITY_CONTEXT`) so `HOME` resolves and the
`/home/user` volume is writable. That fixed UID is only admitted if the terminal
pods' ServiceAccount is allowed to request it — grant the `nonroot-v2` SCC to the
namespace's **default** ServiceAccount (one-time, cluster-admin):

```bash
oc adm policy add-scc-to-user nonroot-v2 -z default -n open-webui
```

`nonroot-v2` still forbids root — it only lets the pod choose a non-root UID. If
you can't grant an SCC, or the image's build UID isn't 1000 (check with `id` in a
terminal), tell me and I'll adjust. The looser alternative is `anyuid` with
`TERMINALS_KUBERNETES_RESTRICTED=false`, which runs the image exactly as it does
under Docker.

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
