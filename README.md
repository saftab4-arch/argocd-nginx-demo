# ArgoCD GitOps Demo — nginx on kind

A hands-on GitOps project that deploys nginx to a local Kubernetes cluster **entirely through Git** — no `kubectl apply -f` for the app. You commit YAML to a GitHub repo, and **ArgoCD** watches that repo and makes the cluster match it. Change the cluster by hand, and ArgoCD reverts it. Git is the single source of truth.

This README is written so anyone can replicate the project from scratch. Every command, every file, and every design decision is explained.

---

## What is GitOps (in one paragraph)

Traditional deployment: *you* run `kubectl apply` (or your CI pipeline does) to **push** changes into the cluster. GitOps flips this: a controller running **inside** the cluster continuously **pulls** the desired state from a Git repo and reconciles the cluster to match it. Git becomes the source of truth for *what should be running*. The tool doing the pulling and reconciling here is **ArgoCD**.

**The payoff:** the only durable way to change the system is to change Git. Manual `kubectl` edits get detected as drift and reverted. That makes the repo a complete, auditable, enforced record of what's running.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub repo (source of truth)             │
│                                                              │
│   argocd-nginx-demo/                                         │
│      └── manifests/                                          │
│            ├── deployment.yaml   (2 nginx pods)              │
│            └── service.yaml      (NodePort :30950)           │
└──────────────────────────┬───────────────────────────────────┘
                           │  ArgoCD watches this repo/path
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  kind cluster (local, Docker)                │
│                                                              │
│   ┌──────────────────────────────────────────────────┐      │
│   │  namespace: argocd                                 │      │
│   │    ArgoCD (7 components: server, repo-server,      │      │
│   │    application-controller, redis, dex, etc.)       │      │
│   │              │ syncs desired → actual              │      │
│   └──────────────┼─────────────────────────────────────┘      │
│                  ▼                                            │
│   ┌──────────────────────────────────────────────────┐      │
│   │  namespace: nginx-demo (auto-created by ArgoCD)    │      │
│   │    Deployment → ReplicaSet → 2 nginx pods          │      │
│   │    Service (NodePort 30950)                        │      │
│   └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### The port chain (how your browser reaches nginx)

This is the part worth understanding carefully. Four ports line up to route a browser request all the way to a pod:

```
browser → localhost:80      (kind hostPort mapping, set in kind-config.yaml)
        → node:30950        (NodePort — MUST match service.yaml nodePort)
        → service :80       (Service port)
        → nginx pod :80     (targetPort / containerPort)
```

The linchpin is **30950**: it appears in *both* the kind config (`containerPort: 30950`) and the service (`nodePort: 30950`). If those two numbers don't match, `localhost` hits a dead port and nothing loads.

---

## Prerequisites

You need these installed and working:

| Tool | Check it works |
|------|----------------|
| Docker | `docker ps` |
| kind | `kind version` |
| kubectl | `kubectl version --client` |
| git | `git --version` |
| A GitHub account | — |

> **Note on the node image version:** the kind node image (`kindest/node:v1.31.2` in this project) must match your installed kind version. If `kind create cluster` fails on the image, either omit the `image:` line (kind uses its own matching default) or look up the correct node image tag in your kind release's notes.

---

## Step-by-step

### Step 1 — Create the kind cluster

We define the cluster **declaratively** in a YAML file (reproducible, and it documents exactly how the cluster was built).

**`kind-config.yaml`:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.2
    extraPortMappings:
      - containerPort: 30950
        hostPort: 80
        protocol: TCP
  - role: worker
    image: kindest/node:v1.31.2
```

**Field explanations:**
- `role: control-plane` + `role: worker` — a two-node cluster so workloads schedule onto the worker (the control-plane is tainted against normal pods by default).
- `extraPortMappings` — maps your machine's **localhost:80** into the node at **port 30950**. This is what lets a browser reach the nginx NodePort later.
- `image` — pins the Kubernetes version for reproducibility (must match your kind version).

Create the cluster:

```bash
kind create cluster --name argocd-demo --config kind-config.yaml
```

Verify both nodes are `Ready` (give it ~30s — `NotReady` right after creation is just the CNI starting up):

```bash
kubectl get nodes
```

Expected:

```
NAME                        STATUS   ROLES           AGE   VERSION
argocd-demo-control-plane   Ready    control-plane   1m    v1.31.2
argocd-demo-worker          Ready    <none>          1m    v1.31.2
```

---

### Step 2 — Install ArgoCD

Create the namespace ArgoCD lives in (the official install manifest expects it to be named `argocd`):

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> **⚠️ Gotcha — the 256KB CRD error.** With plain `kubectl apply`, one CRD (`applicationsets.argoproj.io`) fails with:
> ```
> The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
> metadata.annotations: Too long: must have at most 262144 bytes
> ```
> **Why:** client-side apply stores a full copy of each resource in a `last-applied-configuration` annotation. Kubernetes caps any single annotation at 256KB. That CRD's schema is too big.
>
> **Fix — use server-side apply**, which tracks field ownership on the API server instead of writing that annotation:
> ```bash
> kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
> ```
> This is the officially recommended install method for exactly this reason. It's idempotent — safe to re-run.

Wait for all 7 pods to be `Running` (`1/1`):

```bash
kubectl get pods -n argocd -w
```

Confirm the 3 CRDs registered (this is ArgoCD teaching the cluster new object types):

```bash
kubectl get crd | grep argoproj
```

Expected:

```
applications.argoproj.io
applicationsets.argoproj.io
appprojects.argoproj.io
```

---

### Step 3 — Access the ArgoCD UI

Get the auto-generated admin password (ArgoCD stores it in a Secret; all Secret values are base64-encoded, so we decode):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Port-forward the UI to your laptop (the argocd-server Service is `ClusterIP` — only reachable inside the cluster — so we tunnel out):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

> This terminal **blocks** and holds the tunnel open — leave it running, use another terminal for other commands. `Ctrl+C` closes it.

Open **https://localhost:8080** (accept the self-signed cert warning), log in with:
- **Username:** `admin`
- **Password:** (the decoded value from above)

---

### Step 4 — Create the app manifests and push to GitHub

Create a **public**, **empty** GitHub repo named `argocd-nginx-demo` (no README/.gitignore, so nothing conflicts on first push). Public means ArgoCD reads it without auth setup.

Clone it and create the manifests folder:

```bash
git clone https://github.com/<your-username>/argocd-nginx-demo.git
cd argocd-nginx-demo
mkdir manifests
```

**`manifests/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

**Field explanations:**
- `replicas: 2` — run two pods (so we can watch one get replaced later).
- `selector.matchLabels` **must equal** `template.metadata.labels` (`app: nginx`) — this is how the Deployment finds and owns its pods. The single most important pattern in Kubernetes.
- `image: nginx:1.27` — pinned tag (not `latest`) for reproducibility.
- **No `namespace:` field** — intentional. The namespace is decided by the ArgoCD Application (see Step 5), which keeps the manifest reusable across environments.

**`manifests/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30950
```

**Field explanations:**
- `type: NodePort` — opens a port on the node reachable from outside the cluster.
- `selector: app: nginx` — sends traffic to pods with that label (load-balances across both).
- `nodePort: 30950` — **must match** the `containerPort: 30950` in `kind-config.yaml`. This is the linchpin of the port chain.
- `port: 80` → the Service's internal port; `targetPort: 80` → the pod/container port.

Commit and push (**stage → commit → push** are three distinct steps):

```bash
git add manifests/
git commit -m "Add nginx deployment and service manifests"
git push origin main
```

> If GitHub prompts for a password on push, it wants a **Personal Access Token (PAT)**, not your account password.

---

### Step 5 — Create the ArgoCD Application

The **Application** is the object that connects your repo to your cluster: *"watch THIS repo path, and make THIS cluster/namespace match it."*

In the ArgoCD UI, click **+ NEW APP** and fill in:

| Field | Value | Why |
|-------|-------|-----|
| Application Name | `nginx-demo` | — |
| Project | `default` | built-in project; grouping/permissions come later |
| Sync Policy | `Manual` | so we consciously click Sync and watch it happen |
| Repository URL | `https://github.com/<your-username>/argocd-nginx-demo.git` | the source of truth |
| Revision | `HEAD` | latest commit on the default branch |
| Path | `manifests` | the folder inside the repo (NOT the root) |
| Cluster URL | `https://kubernetes.default.svc` | "the same cluster ArgoCD runs in" (in-cluster API server DNS name) |
| Namespace | `nginx-demo` | **where** to deploy — decided here, not in the manifests |
| Sync Options | ☑ **AUTO-CREATE NAMESPACE** | creates `nginx-demo` since it doesn't exist yet |

Click **CREATE**. The app appears as **Missing / OutOfSync** — correct, because sync is manual and nothing is deployed yet.

- **Missing** = health: the resources don't exist in the cluster.
- **OutOfSync** = sync: desired (Git) ≠ actual (cluster).

Click **SYNC → SYNCHRONIZE**. ArgoCD creates the namespace, applies both manifests, and starts 2 nginx pods. Status flips to:

- **💚 Healthy** = resources up and running.
- **✅ Synced** = desired matches actual.

---

### Step 6 — Enable auto-sync + self-heal

Open the `nginx-demo` app → **APP DETAILS** → **SYNC POLICY** → **ENABLE AUTO-SYNC**, then enable both:

- ☑ **PRUNE RESOURCES** — deleting a manifest from Git deletes it from the cluster (no orphans).
- ☑ **SELF HEAL** — manual cluster changes get reverted to match Git.

Now a `git push` deploys automatically, and the cluster defends its own state.

---

## Tests / Demos

### Test 1 — App is reachable in the browser

Open **http://localhost** (plain http, port 80). Expected: the **"Welcome to nginx!"** page.

This proves the full port chain: `browser → localhost:80 → node:30950 → service:80 → pod:80`.

### Test 2 — Pods run in the correct namespace

```bash
kubectl get pods                    # → "No resources found in default namespace."
kubectl get pods -n nginx-demo      # → 2 pods, Running, 1/1
```

Proves the namespace decision lives in the Application, not the manifests. The pods land in `nginx-demo` (auto-created by ArgoCD), not `default`.

### Test 3 — Kubernetes native self-healing (pod level)

Delete a pod by hand:

```bash
kubectl get pods -n nginx-demo
kubectl delete pod <one-pod-name> -n nginx-demo
kubectl get pods -n nginx-demo      # → a fresh pod (low AGE) has replaced it
```

The **Deployment/ReplicaSet controller** (plain Kubernetes) notices it's down to 1 of 2 and creates a replacement. Nothing to do with ArgoCD.

### Test 4 — ArgoCD self-healing (config drift)

Scale the deployment by hand, contradicting Git (Git says 2, we say 3):

```bash
kubectl scale deployment nginx -n nginx-demo --replicas=3
kubectl get pods -n nginx-demo      # → still only 2 pods
```

Kubernetes is happy to run 3 — but **ArgoCD** detects the cluster no longer matches Git and reverts it to 2, in under a second. Git wins.

To watch the revert happen live, stream pod changes in one terminal:

```bash
kubectl get pods -n nginx-demo -w
```

...then scale to 3 in another. You'll see a 3rd pod appear and then get **Terminated** as ArgoCD reverts.

### The two layers of self-healing

```
K8s self-heal:    keeps ACTUAL matching the DEPLOYMENT spec  (pod dies → replace)
ArgoCD self-heal: keeps the DEPLOYMENT spec matching GIT     (drift → revert)
```

Together: the only durable way to change the system is to change Git.

---

## Cleanup

```bash
# delete just this cluster (leaves any others untouched)
kind delete cluster --name argocd-demo
```

---

## Lessons learned

- **GitOps means no `kubectl apply` for the app.** You commit YAML; ArgoCD deploys it. Git is the interface.
- **Server-side apply** solves the 256KB annotation limit on large CRDs — and is the recommended way to install ArgoCD.
- **A Service gives pods a stable address.** Pods are ephemeral with changing IPs; you target the Service (`svc/...`), which routes to whatever pod is live.
- **ClusterIP vs NodePort vs LoadBalancer** — ArgoCD's server is `ClusterIP` (internal only, hence port-forward); nginx uses `NodePort` (externally reachable); cloud setups use `LoadBalancer`.
- **Namespace belongs in the Application, not the manifest** — keeps manifests reusable across dev/staging/prod.
- **Two layers of self-healing** — Kubernetes heals pod failures; ArgoCD heals config drift against Git.

---

## Tech stack

kind · Kubernetes · ArgoCD · GitHub · nginx · Docker

---

## Next steps

- Split into the full two-repo GitOps pattern (app-code repo + config repo) with GitHub Actions building images and bumping tags.
- Define the ArgoCD Application itself as YAML in Git (App-of-Apps pattern) so even ArgoCD's config is version-controlled.
