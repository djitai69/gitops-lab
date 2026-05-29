# gitops-lab — Break & Fix

A hands-on GitOps lab for learning [Argo CD](https://argo-cd.readthedocs.io/). You deploy a simple nginx app from Git, then deliberately **break** it in a series of exercises and use Argo CD + `kubectl` to **diagnose and fix** each failure. The goal is to build intuition for how GitOps reconciliation behaves when manifests are wrong.

## What you'll learn

- How Argo CD reconciles a Git repo against a live cluster (Sync status vs. Health status).
- How to read Argo CD sync errors and Kubernetes events to find a root cause.
- Common real-world failure modes: bad image references, invalid manifests, and **immutable fields**.

## Repo layout

```
argocd/
  nginx-app.yaml          # Argo CD Application (points at apps/nginx)
apps/nginx/
  deployment.yaml         # nginx Deployment (2 replicas)
  service.yaml            # ClusterIP Service
  kustomization.yaml      # Kustomize entrypoint, namespace: default
```

The `Application` in `argocd/nginx-app.yaml` syncs `apps/nginx` with an **automated** sync policy (`prune: true`, `selfHeal: true`) into the `default` namespace.

## Prerequisites

- A Kubernetes cluster — this lab was built with [kind](https://kind.sigs.k8s.io/) (`kind create cluster --name argocd-lab`).
- `kubectl`
- The `argocd` CLI

## Setup

1. **Install Argo CD** into the cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl rollout status deploy/argocd-server -n argocd
   ```

2. **Point the Application at your fork.** If you forked this repo, edit `argocd/nginx-app.yaml` and change `spec.source.repoURL` to your fork's URL. Then register it:
   ```bash
   kubectl apply -f argocd/nginx-app.yaml
   ```

3. **Access the UI / CLI.** Port-forward the Argo CD server and log in:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # UI: https://localhost:8080  (self-signed cert — accept the warning)

   PW=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d)
   argocd login localhost:8080 --username admin --password "$PW" --insecure
   ```

   > Tip: to keep the port-forward running across reboots, run it as a systemd user service
   > (`~/.config/systemd/user/argocd-portforward.service` with `Restart=always`) and enable
   > `loginctl enable-linger $USER`.

4. **Confirm the baseline is healthy** before breaking anything:
   ```bash
   argocd app get nginx
   # Sync Status: Synced   |   Health Status: Healthy
   kubectl get pods -n default -l app=nginx   # 2 pods Running
   ```

## How each exercise works

For every stage you:
1. Edit a manifest under `apps/nginx/` to introduce the break, then `git commit` and `git push`.
2. Let Argo CD auto-sync (or run `argocd app sync nginx`).
3. **Observe** the failure in `argocd app get nginx` and `kubectl`.
4. **Diagnose** the root cause from the error message.
5. Revert/fix the manifest, commit, push, and confirm it returns to **Synced / Healthy**.

The git history of this repo contains worked examples of each break and its fix — use `git log -p apps/nginx/deployment.yaml` if you get stuck.

---

## Stage 1 — Bad image & invalid manifest

**Break:** in `apps/nginx/deployment.yaml`, point the container at a non-existent image and give it an invalid port:
```yaml
      containers:
      - name: nginx
        image: nonexistent:broken
        ports:
        - containerPort: "not-a-number"   # must be an integer
```

**Observe / diagnose:**
- The string `containerPort` makes the manifest invalid — Argo CD reports a sync error, or Kubernetes rejects the object. Run `argocd app get nginx` and read the condition message.
- Once the port is a valid integer but the image is still `nonexistent:broken`, the Deployment applies but pods get stuck in `ImagePullBackOff`:
  ```bash
  kubectl get pods -n default -l app=nginx
  kubectl describe pod -n default -l app=nginx   # Events show the pull failure
  ```
- App shows **Synced** (Git matches cluster) but **Degraded** (pods can't run) — a key GitOps distinction: *sync* and *health* are separate.

**Fix:** restore a real image and an integer port:
```yaml
        image: nginx:stable
        ports:
        - containerPort: 80
```

**Lesson:** Argo CD can faithfully sync a manifest that still produces unhealthy pods. Always check **Health**, not just **Sync**.

---

## Stage 2 — Mutating an immutable field (the selector trap)

**Break:** in `apps/nginx/deployment.yaml`, change only the selector so it no longer matches the pod template labels:
```yaml
spec:
  selector:
    matchLabels:
      app: changed          # template labels are still app: nginx
```

**Observe / diagnose:**
- The sync **fails at apply time**. `argocd app get nginx` (or the UI) shows the error. Two things are wrong:
  1. `spec.selector` no longer matches `spec.template.metadata.labels` (`selector does not match template labels`).
  2. Even if you also changed the template labels, a Deployment's `spec.selector` is **immutable** — Kubernetes rejects an in-place change with `field is immutable`.

**Fix:** set the selector back to match the template:
```yaml
  selector:
    matchLabels:
      app: nginx
```

**Lesson:** Some fields can't be changed via `kubectl apply` (which is what Argo CD does by default). A revert in Git only recovers cleanly if the bad value **never reached the cluster**. If an immutable change *did* apply, you must either delete & recreate the resource, or tell Argo CD to recreate it instead of patch:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Replace=true
```

---

## Cleanup

```bash
argocd app delete nginx
kind delete cluster --name argocd-lab
```

## Notes

- The `argocd-initial-admin-secret` is a bootstrap credential. For anything beyond a local lab, change the admin password (`argocd account update-password`) and delete the secret.
- `selfHeal: true` means Argo CD will revert manual `kubectl edit` changes back to Git — to experiment with drift, temporarily disable self-heal or expect it to be corrected.
