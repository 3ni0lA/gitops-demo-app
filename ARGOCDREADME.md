# GitOps Demo — ArgoCD on Kind

A working GitOps pipeline where a Git commit is the **only** deployment mechanism.  
No `kubectl apply`. No CI pipeline pushing to the cluster. The cluster watches Git and pulls.

---

## What This Proves

Most engineers understand CI/CD. Fewer understand GitOps — and hiring managers know the difference.

**CI/CD (push model):** A pipeline builds your code and *pushes* changes to the cluster. The pipeline holds the keys.

**GitOps (pull model):** A controller *inside* the cluster watches a Git repo and pulls changes in. Git is the single source of truth. If the cluster drifts from what Git says, the controller corrects it automatically — no human intervention.

This project demonstrates that model end-to-end.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Your Machine                        │
│                                                      │
│   Git commit ──► GitHub Repo                        │
│                      │                              │
│                      │ ArgoCD polls every 3 min     │
│                      ▼                              │
│         ┌────────────────────────┐                  │
│         │   Kind Cluster          │                  │
│         │                        │                  │
│         │  ┌──────────────────┐  │                  │
│         │  │  ArgoCD           │  │                  │
│         │  │  (argocd ns)      │  │                  │
│         │  └────────┬─────────┘  │                  │
│         │           │ detects drift, syncs           │
│         │           ▼            │                  │
│         │  ┌──────────────────┐  │                  │
│         │  │  demo-app         │  │                  │
│         │  │  (default ns)     │  │                  │
│         │  │  nginx:1.25.x     │  │                  │
│         │  └──────────────────┘  │                  │
│         └────────────────────────┘                  │
└─────────────────────────────────────────────────────┘
```

---

## Stack

| Tool | Version | Role |
|------|---------|------|
| [Kind](https://kind.sigs.k8s.io/) | v0.23.0 | Local Kubernetes cluster (runs in Docker) |
| [ArgoCD](https://argo-cd.readthedocs.io/) | v3.3.8 | GitOps controller |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | latest stable | Cluster CLI |
| [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) | v3.3.8 | ArgoCD management CLI |
| nginx | 1.25.x | Demo application |

---

## Prerequisites

- Docker installed and running
- `kubectl` installed ([install guide](https://kubernetes.io/docs/tasks/tools/))
- `kind` installed ([install guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation))
- A GitHub account

---

## Setup

### 1. Create the cluster

```bash
kind create cluster --name gitops-demo
kubectl cluster-info --context kind-gitops-demo
kubectl get nodes
```

One node should appear in `Ready` status. That node is a Docker container running the full Kubernetes control plane and worker combined.

---

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Watch pods come up — wait until all show `Running`:

```bash
kubectl get pods -n argocd -w
```

**Key pods and what they do:**

| Pod | Role |
|-----|------|
| `argocd-application-controller` | The GitOps brain — compares Git state to cluster state |
| `argocd-repo-server` | Clones and reads your Git repo |
| `argocd-server` | Serves the UI and API |
| `argocd-redis` | Caches state between components |
| `argocd-applicationset-controller` | Manages ApplicationSets (not used in this demo) |

---

### 3. Expose the ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal open — the tunnel stays alive as long as the command runs. Open `https://localhost:8080` in your browser and proceed past the TLS warning (self-signed cert).

**Tip:** Run the port-forward in a persistent tmux session so it doesn't drop when you switch terminals:

```bash
tmux new -s argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Ctrl+B then D to detach — port-forward keeps running
# tmux attach -t argocd to return
```

**Get the initial admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

> The `&& echo` adds a newline so your terminal prompt doesn't attach to the password string. Login with username `admin`.

**Important:** Kubernetes Secrets store data base64-encoded, not encrypted. This is fine for local clusters but worth knowing for production setups.

---

### 4. Install the ArgoCD CLI

Match the CLI version to your server version:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v3.3.8/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/argocd
argocd version --client
```

Login via CLI:

```bash
argocd login localhost:8080 --username admin --insecure
```

---

### 5. Fork or clone this repo

This repo contains the Kubernetes manifests ArgoCD watches. The `k8s/` folder is the desired state — what ArgoCD will make your cluster match.

```
gitops-demo-app/
└── k8s/
    ├── deployment.yaml   ← Deployment with pinned nginx image tag
    └── service.yaml      ← ClusterIP service
```

> **Why a pinned image tag matters:** Never use `latest` in GitOps. If you do, the cluster and repo always "match" — ArgoCD can never detect drift because the tag never changes. Always pin versions.

---

### 6. Create the ArgoCD Application

```bash
kubectl apply -f argocd-app.yaml
```

The `argocd-app.yaml` connects ArgoCD to this repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/3ni0lA/gitops-demo-app.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true      # deletes resources removed from Git
      selfHeal: true   # reverts manual cluster changes back to Git state
```

**Key fields:**

- `prune: true` — if you delete a manifest from Git, ArgoCD removes the resource from the cluster. Without this, deleted files leave orphaned resources behind.
- `selfHeal: true` — if anyone runs `kubectl` manually and changes something, ArgoCD detects the drift and reverts it. Git is the only authority.

After applying, open the ArgoCD UI. You should see `demo-app` appear, turn green, and show `Synced`.

---

## Demonstrating GitOps

### Demo 1 — Image tag update (the main GitOps demo)

Edit `k8s/deployment.yaml` in GitHub. Change:

```yaml
image: nginx:1.25.0
```

to:

```yaml
image: nginx:1.25.3
```

Commit to main. ArgoCD detects the drift within 3 minutes and syncs automatically. Force an immediate sync via CLI:

```bash
argocd app sync demo-app
```

Watch pods cycle in real time:

```bash
kubectl get pods -w
```

Old pods terminate, new ones start — triggered entirely by a Git commit.

---

### Demo 2 — Self-healing (proves Git is the only authority)

Manually scale the deployment outside of Git:

```bash
kubectl scale deployment demo-app --replicas=5
```

Watch ArgoCD detect the drift and correct it:

```bash
kubectl get application demo-app -n argocd -w
```

You'll see: `Synced` → `OutOfSync` → `Synced` within ~15 seconds. The replica count returns to 2 — what Git says — automatically.

This is `selfHeal: true` making the cluster self-correcting.

---

## Errors Encountered (and fixes)

Real-world debugging log from the actual setup of this project.

---

### Error 1 — `argocd-applicationset-controller` CrashLoopBackOff

**Symptom:**
```
argocd-applicationset-controller   0/1   CrashLoopBackOff   5 (17s ago)   19m
```

**Root cause:**  
The ApplicationSet CRD was not registered in the cluster when the controller started. The controller looked for its CRD, couldn't find it, retried every 10 seconds for 2 minutes, then crashed — repeating on a loop.

**Fix:**  
Apply the missing CRD using server-side apply (standard apply exceeds the 262KB annotation limit on this manifest):

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.8/manifests/crds/applicationset-crd.yaml
```

> **Why `--server-side`:** Normal `kubectl apply` stores the full manifest in a `last-applied-configuration` annotation for change tracking. This CRD is large enough to exceed Kubernetes's 262KB annotation limit. Server-side apply moves the diff logic to the API server, which doesn't use that annotation.

---

### Error 2 — ArgoCD can't reach GitHub (DNS resolution failure)

**Symptom:**
```
ComparisonError: dial tcp: lookup github.com on 10.96.0.10:53: no such host
```

**Root cause:**  
ArgoCD runs inside the Kind cluster. The cluster's internal DNS (CoreDNS) was inheriting the WSL2 host's `/etc/resolv.conf` nameserver, which doesn't forward external queries correctly from inside nested containers.

**Fix:**  
Patch CoreDNS to forward external lookups directly to Google's DNS:

```bash
kubectl edit configmap coredns -n kube-system
```

Change:
```
forward . /etc/resolv.conf
```

To:
```
forward . 8.8.8.8 8.8.4.4
```

Restart CoreDNS:
```bash
kubectl rollout restart deployment coredns -n kube-system
```

---

### Error 3 — Port-forward drops silently

**Symptom:**  
`curl: (7) Failed to connect to localhost port 8080` — the ArgoCD UI becomes unreachable mid-session.

**Root cause:**  
`kubectl port-forward` is a foreground process. It dies when the terminal closes, goes idle, or loses focus.

**Fix:**  
Run port-forward inside a tmux session so it persists independently of terminal state:

```bash
tmux new -s argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Ctrl+B then D to detach
```

---

## Key Concepts

**GitOps vs CI/CD**  
CI/CD pipelines push changes to the cluster — the pipeline holds credentials and reaches out to deploy. GitOps flips this: a controller inside the cluster pulls changes from Git. No external system ever touches the cluster directly.

**Drift detection**  
ArgoCD polls Git every 3 minutes. If the cluster state doesn't match Git, it marks the app `OutOfSync` and syncs automatically (when `automated` sync policy is set).

**Why pinned image tags are required**  
With `latest`, the cluster and Git always report the same tag even when running different images. ArgoCD cannot detect or correct that drift. Pinned tags make drift visible and correctable.

**Server-side apply**  
For large manifests (CRDs especially), use `kubectl apply --server-side` to bypass the 262KB client-side annotation limit.

---

## Cleanup

```bash
kind delete cluster --name gitops-demo
```

Deletes the cluster and all resources. Nothing persists except the Docker image cache.

---

## What's Next

- [ ] Add Prometheus + Grafana monitoring stack
- [ ] Add GitHub Actions CI pipeline to build and push image on commit
- [ ] Add multi-environment setup (`k8s/staging/`, `k8s/production/`)
- [ ] Explore ApplicationSets for multi-app management

---

## Author

**Eniola Adewale** — Automation Workflow Specialist & DevOps Engineer  
[linkedin.com/in/eniola-adewale-d3v0ps](https://www.linkedin.com/in/eniola-adewale-d3v0ps)  
[github.com/3ni0lA](https://github.com/3ni0lA)
