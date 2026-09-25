# image-pull-secret-in-deployment


# Fixing `ImagePullBackOff`: Registry Secret Workflow

This is the four-command sequence used to diagnose and fix Docker registry authentication failures (`ImagePullBackOff` / `ErrImagePull` / "anonymous token 403") in Kubernetes. Run them in order, top to bottom, and only proceed to the next command if the previous one shows a problem.

---

## Background: why this happens

Kubernetes pulls container images the same way `docker login` + `docker pull` would. The credentials live in a Kubernetes **Secret** of type `kubernetes.io/dockerconfigjson`, referenced by a deployment's `imagePullSecrets`. If that secret doesn't exist, has the wrong password, or points at the wrong registry host, kubelet falls back to an **anonymous** pull — which a private registry rejects with `403 Forbidden`. The fix is almost always: find which secret the deployment expects, check whether it's correct, and if not, recreate it.

---

## Command 1: Set the registry password for this session

```bash
read -rs REGPASS && export REGPASS
```

**What it does:** Prompts you to type the registry password (`-s` means "silent" — it won't echo to the screen) and stores it in the shell variable `REGPASS`, then `export`s it so any command you run afterward in this terminal session can use `"$REGPASS"`.

**Why:** Keeps the password out of your shell history and off your screen. You only need to run this once per terminal session — it stays set until you close that terminal.

---

## Command 2: Find out which secret a deployment expects

```bash
kubectl -n <namespace> get deployment <deploymentName> -o jsonpath='{.spec.template.spec.imagePullSecrets}'
```

**What it does:** Reads the deployment's pod template and prints the name(s) of the `imagePullSecrets` it references, e.g. `[{"name":"regcred"}]` or `[{"name":"alphatree-registry"}]`.

**Why:** Different deployments (and different namespaces) sometimes reference different secret names — there's no single standard name across a cluster unless your team enforces one. You need this name before you can check or fix the right secret. Empty output means the deployment has **no** pull secret attached at all (a different problem — the deployment itself needs patching, not the secret).

**Placeholders:**
- `<namespace>` — e.g. `alphatree-qa`
- `<deploymentName>` — e.g. `alphatree-new-frontend`

---

## Command 3: Check what's actually inside that secret

```bash
kubectl -n <namespace> get secret <secretName> -o jsonpath='{.data.\.dockerconfigjson}' 2>/dev/null | base64 -d
```

**What it does:** Reads the secret's `.dockerconfigjson` field (which is base64-encoded inside the Kubernetes object) and decodes it back to plain JSON, e.g.:
```json
{"auths":{"repo.walkingtree.tech":{"username":"admin","password":"...","auth":"..."}}}
```

**Why:** Tells you exactly what's wrong, before you touch anything:
- **Empty output** → the secret doesn't exist in this namespace at all.
- **Wrong host** under `"auths"` (e.g. `repo.qritrim.com` or an `azurecr.io` address instead of `repo.walkingtree.tech`) → kubelet has no matching credentials for the actual registry, so it pulls anonymously and gets rejected.
- **Correct host, but you suspect a stale/expired password** → worth testing directly against the registry before recreating (see the note at the bottom).

**Note:** The `auth` field (a base64 of `username:password`) is what container runtimes actually use to authenticate — not the separate `username`/`password` fields shown alongside it. It's possible to see mismatched cosmetic `password` fields across secrets while `auth` is identical and still valid; don't assume a secret is broken just because `password` looks odd if `auth` decodes to the correct pair.

**Placeholders:**
- `<secretName>` — the exact name from Command 2's output, e.g. `regcred`, `alphatree-registry`, `dev2-alphatree-registry`

---

## Command 4: Recreate the secret with the correct credentials

```bash
kubectl -n <namespace> delete secret <secretName> --ignore-not-found; kubectl -n <namespace> create secret docker-registry <secretName> --docker-server=repo.walkingtree.tech --docker-username=admin --docker-password="$REGPASS"
```

**What it does:** Two commands chained with `;` (both run regardless of whether the first succeeds):
1. `delete secret ... --ignore-not-found` — removes the old/broken secret if it exists; does nothing (no error) if it doesn't.
2. `create secret docker-registry ...` — creates a fresh secret of the correct type, pointing at the right registry host, using the current `$REGPASS`.

**Why `delete` before `create`:** `kubectl create secret` fails if a secret with that name already exists — you can't just overwrite it in place. Deleting first (safely, with `--ignore-not-found`) guarantees the create step always succeeds whether or not the old one was there.

**Important:** This only fixes the secret's *contents*. It does **not**:
- Attach the secret to a deployment that doesn't already reference it (use `kubectl patch deployment` for that — see below).
- Force already-stuck pods to retry immediately. Kubernetes only re-checks credentials on the next scheduled pull attempt (which backs off over time, up to a few minutes). To force an immediate retry:
  ```bash
  kubectl -n <namespace> delete pod <podName> --force --grace-period=0
  ```
  The pod's controlling ReplicaSet/Deployment will recreate it right away, and it will pick up the now-correct secret.

**Placeholders:**
- `<namespace>`, `<secretName>` — same as Command 3
- `--docker-server` — change if a different registry host is involved
- `--docker-username` — change if the account isn't `admin`

---

## Quick reference: the full flow

```bash
# 1. Set the password once per terminal session
read -rs REGPASS && export REGPASS

# 2. Find out what secret this deployment expects
kubectl -n <namespace> get deployment <deploymentName> -o jsonpath='{.spec.template.spec.imagePullSecrets}'
# → [{"name":"<secretName>"}]

# 3. Check if that secret exists and is correct
kubectl -n <namespace> get secret <secretName> -o jsonpath='{.data.\.dockerconfigjson}' 2>/dev/null | base64 -d
# → (empty, or wrong host)

# 4. Recreate it correctly
kubectl -n <namespace> delete secret <secretName> --ignore-not-found; kubectl -n <namespace> create secret docker-registry <secretName> --docker-server=repo.walkingtree.tech --docker-username=admin --docker-password="$REGPASS"

# 5. (if needed) force the stuck pod to retry now
kubectl -n <namespace> delete pod <podName> --force --grace-period=0
```

---

## Things worth remembering

- **Secrets are per-namespace.** A working `regcred` in `qi` does not exist in `qibb`, `alphatree-dev`, or anywhere else — each namespace needs its own copy.
- **A `kubectl patch deployment` that adds/changes `imagePullSecrets` is a live-cluster change only.** If the deployment is managed by a Helm chart, GitOps tool (ArgoCD/Flux), or a CI/CD pipeline that re-applies manifests, the next deploy can silently revert your patch. Where possible, prefer fixing the *secret's contents* under the name the deployment already expects, rather than changing which secret name it points to.
- **Test credentials directly against the registry** if you're unsure whether the password itself is stale, independent of Kubernetes:
  ```bash
  curl -s -u admin:"$REGPASS" \
    "https://repo.walkingtree.tech/v2/token?scope=repository:<repo-path>:pull&service=repo.walkingtree.tech"
  ```
  A JSON `{"token": "..."}` response means the credential is valid; a 403 means the password itself is wrong.


### restart stuck pods (run in each cluster)
```bash
kubectl get pods -A --no-headers | awk '$4 ~ /ImagePullBackOff|ErrImagePull/ {print $1, $2}' | while read n p; do kubectl -n $n delete pod $p; done
kubectl get pods -A --no-headers | grep -Ei "ImagePull|ErrImage"
```
  
