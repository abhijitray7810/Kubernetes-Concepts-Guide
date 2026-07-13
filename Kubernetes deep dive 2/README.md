# 🚀 Kubernetes (K8s) — Ultimate Deep Dive Study Guide
 
> **For interviews, production knowledge, and real-world understanding**
 
--- 

## Table of Contents

1. [kind — Kubernetes IN Docker](#1-kind--kubernetes-in-docker)
2. [Backward Compatibility — Master vs Worker Version](#2-backward-compatibility--version-skew-policy)
3. [Stateless vs Stateful Applications](#3-stateless-vs-stateful-applications)
4. [Essential kubectl Commands Deep Dive](#4-essential-kubectl-commands-deep-dive)
5. [k3s vs k8s vs k9s — What, Why, Where](#5-k3s-vs-k8s-vs-k9s)
6. [kubectl api-resources & get pods -owide](#6-kubectl-api-resources--get-pods--owide)
7. [Trivy — Image Scanner & CVE](#7-trivy--image-scanner--cve)
8. [Kyverno — Policy Engine Deep Dive](#8-kyverno--policy-engine-deep-dive)
9. [kube-linter & kube-bench](#9-kube-linter--kube-bench)
10. [Rate Limiting — 429 Error & Backoff Algorithm](#10-rate-limiting--429-error--backoff-algorithm)
11. [Init Container vs Sidecar Container](#11-init-container-vs-sidecar-container)
12. [Pod Lifecycle & Termination — Graceful Delete](#12-pod-lifecycle--termination)
13. [RuntimeClass — Kata Containers & Security](#13-runtimeclass--kata-containers)
14. [Security Context Deep Dive](#14-security-context-deep-dive)
15. [Interview Q&A Master List](#15-interview-qa-master-list)

---

## 1. kind — Kubernetes IN Docker

### What is kind?

**One-line definition:** `kind` (Kubernetes IN Docker) is a tool that runs K8s cluster nodes as **Docker containers** on your local machine — used for local development and CI/CD testing.

### Command Deep Dive

```bash
kind create cluster --name Anythink --config download.yml
```

| Part | Meaning |
|------|---------|
| `kind create cluster` | Create a new K8s cluster using Docker containers as nodes |
| `--name Anythink` | Name this cluster "Anythink" (kubeconfig context = `kind-Anythink`) |
| `--config download.yml` | Use this YAML file to define HOW the cluster is built |

### What is inside `download.yml` (kind config)?

```yaml
# download.yml — kind cluster configuration
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane          # This node = master/control-plane
    image: kindest/node:v1.29.0  # K8s version for master
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80        # Map port 80 from container to host
        hostPort: 80
      - containerPort: 443
        hostPort: 443

  - role: worker                 # Worker node 1
    image: kindest/node:v1.28.0  # Can be different version (skew policy)

  - role: worker                 # Worker node 2
    image: kindest/node:v1.28.0
```

**Every step in this file tells kind:**
- How many nodes (control-plane + workers)
- Which K8s image version per node
- Port mappings (so you can access apps from your laptop)
- Extra kubelet arguments
- Extra mounts (host path → container)

### kind Under the Hood

```
Your Laptop
└── Docker Engine
    ├── Container: Anythink-control-plane  (runs: etcd, api-server, scheduler, controller-manager, kubelet)
    ├── Container: Anythink-worker         (runs: kubelet, kube-proxy, containerd)
    └── Container: Anythink-worker2        (runs: kubelet, kube-proxy, containerd)
```

Each "node" is actually a Docker container running `systemd` + `kubelet` + `containerd` inside it.

### Q: Why use kind instead of minikube?
**A:**
| Feature | kind | minikube |
|---------|------|---------|
| Multi-node clusters | ✅ Yes | ⚠️ Limited |
| CI/CD friendly | ✅ Docker-native | ⚠️ Needs VM driver |
| Speed | ✅ Fast (Docker) | Slower (VM) |
| Production-like | ✅ More realistic | Less realistic |
| Custom node images | ✅ Yes | ✅ Yes |

---

## 2. Backward Compatibility — Version Skew Policy

### The Rule: Why Master Must Be Newer Than Worker

```
Master (control-plane): v1.29  ✅
Worker nodes:           v1.27  ✅  (2 versions behind = OK)
Worker nodes:           v1.26  ❌  (3 versions behind = NOT supported)
```

**K8s Version Skew Policy:**
- `kube-apiserver` must be **≥ kubelet version** on workers
- kubelet can be at most **2 minor versions behind** the API server
- `kubectl` can be **1 version ahead or behind** the API server

### Why This Rule Exists — Interview Answer

```
v1.29 API server introduces new fields in Pod spec
↓
v1.27 kubelet may not understand those new fields
↓
kubelet ignores unknown fields (backward compatible)
↓
Pod still runs — kubelet just skips what it doesn't know
```

**The reverse (worker newer than master) is DANGEROUS:**
```
v1.30 kubelet uses new API calls
↓
v1.29 API server doesn't understand them
↓
kubelet FAILS to register with API server
↓
Node stays NotReady — pods cannot be scheduled
```

### Real Interview Question: "Why is 1.29 master with 1.27 worker OK?"

**Answer:**
> Kubernetes guarantees backward compatibility for 2 minor versions. The API server (v1.29) understands requests from kubelet v1.27 because the API server is always designed to handle older clients. The kubelet (v1.27) may not use new v1.29 features, but it can still manage containers using the APIs it knows. This allows rolling upgrades — you upgrade the control plane first, then worker nodes one by one, maintaining cluster availability throughout.

### Upgrade Order (Always)

```
1. Upgrade etcd (if external)
2. Upgrade kube-apiserver (master)
3. Upgrade kube-controller-manager
4. Upgrade kube-scheduler
5. Upgrade cloud-controller-manager
6. Upgrade kubelet + kube-proxy on workers (one by one)
7. Upgrade kubectl (client)
```

---

## 3. Stateless vs Stateful Applications

### Stateless — "I remember nothing"

**One-line definition:** A stateless application does **not store any client session data** on the server — every request is independent and can be handled by any instance.

**Examples:** Frontend web servers (Nginx, React app), REST APIs, microservices

```
User Request 1 → Pod A (no memory of user)
User Request 2 → Pod B (same result — doesn't matter which pod)
User Request 3 → Pod C (completely fine)
```

**K8s resource:** `Deployment` — any pod replica is identical and replaceable

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3           # 3 identical, interchangeable pods
  selector:
    matchLabels:
      app: frontend
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        # No persistent storage needed
```

**Why stateless is easy to scale:**
- Kill any pod → no data loss
- Scale from 3→10 instantly
- Rolling updates with zero downtime
- Any pod can handle any request

### Stateful — "I remember everything"

**One-line definition:** A stateful application **stores data that must persist** across restarts and requires stable identity (hostname, storage) — each instance is unique.

**Examples:** PostgreSQL, MySQL, MongoDB, Kafka, Redis (persistent), Elasticsearch

```
User writes data → Pod mysql-0 (stores in /data/mysql)
Pod mysql-0 crashes ↓
Pod mysql-0 restarts → MUST get SAME /data/mysql back
```

**K8s resource:** `StatefulSet` — pods have stable identity, ordered deployment, persistent storage

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless     # Stable DNS: mysql-0.mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:           # Each pod gets its OWN PVC
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 10Gi
```

**StatefulSet guarantees:**
- Pod names are stable: `mysql-0`, `mysql-1`, `mysql-2`
- DNS names are stable: `mysql-0.mysql-headless.default.svc.cluster.local`
- Each pod gets its own PVC that survives pod deletion
- Ordered startup: `mysql-0` starts first, then `mysql-1`, then `mysql-2`
- Ordered deletion: reverse order

### Comparison Table

| Feature | Stateless (Deployment) | Stateful (StatefulSet) |
|---------|----------------------|----------------------|
| Pod identity | Random (pod-abc123) | Stable (mysql-0) |
| Storage | Shared or none | Per-pod PVC |
| Scaling | Instant, any order | Ordered (0, 1, 2...) |
| Pod replacement | Any replica | Must get same identity |
| DNS | Service IP only | Per-pod stable DNS |
| Examples | Frontend, API | DB, Kafka, Zookeeper |
| Failure impact | Minimal | Must recover data |

### Interview Q: "Explain stateless vs stateful with a real example"

**Answer:**
> Think of a coffee shop. A **stateless** barista can be replaced — you walk up, order, get coffee, done. Any barista works. A **stateful** barista is your personal banker who knows your entire financial history. You can't just swap bankers — your specific banker holds your data. In K8s: Nginx serving a React app is stateless (deploy 10 replicas, kill any, no problem). PostgreSQL is stateful — it holds your rows, transactions, and write-ahead log. If you kill the wrong pod without proper PVC management, you lose data forever.

---

## 4. Essential kubectl Commands Deep Dive

### `alias k=kubectl`

**Why:** Saves typing. Instead of `kubectl get pods`, type `k get pods`.

```bash
# Add to ~/.bashrc or ~/.zshrc
alias k=kubectl
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kns='kubectl config set-context --current --namespace'

# Also enable autocomplete with alias
complete -F __start_kubectl k
```

### `kubectl run nginx --image nginx --sleep 1d`

**Full breakdown:**

```bash
kubectl run nginx \           # Create a pod named "nginx"
  --image nginx \             # Using the nginx container image
  --sleep 1d                  # Override command: sleep for 1 day (keeps pod alive)
```

**What this actually does:**
```yaml
# Equivalent YAML generated:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    command: ["sleep"]
    args: ["1d"]        # sleep 1 day = 86400 seconds
```

**Why `--sleep 1d`?**
- Nginx normally runs as a foreground web server
- `--sleep 1d` overrides the default command
- Keeps the pod alive for 1 day so you can `exec` into it for debugging
- Used for **debugging pods** — run a shell image and sleep to explore

**Variation: Debug pod pattern**
```bash
kubectl run debug-pod \
  --image=busybox \
  --restart=Never \
  -it \
  -- sh

# OR use ephemeral debug container
kubectl debug -it nginx --image=busybox --target=nginx
```

### `kubectl exec -it nginx -- bash`

| Flag/Part | Meaning |
|-----------|---------|
| `exec` | Execute a command in a running container |
| `-i` | Interactive — keep STDIN open (send input to container) |
| `-t` | TTY — allocate a pseudo-terminal (enables colors, proper shell) |
| `-it` | Both together — gives you an interactive shell session |
| `nginx` | The pod name |
| `--` | Separator: everything after is the command to run |
| `bash` | Run bash shell inside the container |

```bash
# -i only (no TTY): pipe commands in
echo "ls -la" | kubectl exec -i nginx -- sh

# -t only (TTY, no stdin): rarely used
kubectl exec -t nginx -- bash

# -it together: full interactive terminal
kubectl exec -it nginx -- bash

# Specify container in multi-container pod
kubectl exec -it nginx -c sidecar -- bash

# Run a one-off command (no -it needed)
kubectl exec nginx -- cat /etc/nginx/nginx.conf
```

**What happens when you run `kubectl exec -it nginx -- bash`:**
```
kubectl → kube-apiserver (HTTPS)
              ↓ WebSocket upgrade
         kubelet on node (port 10250)
              ↓ CRI exec
         containerd
              ↓
         container process (bash)
              ↑↓ stdin/stdout/stderr streamed back to your terminal
```

### `kubectl get pods` vs `kubectl get pods -owide`

```bash
# Basic output
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          5m

# Wide output (-owide)
kubectl get pods -owide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          5m    10.244.1.5   kind-worker    <none>           <none>
```

**Extra columns in `-owide`:**
| Column | Meaning |
|--------|---------|
| `IP` | Pod's IP address (assigned by CNI) |
| `NODE` | Which worker node the pod runs on |
| `NOMINATED NODE` | For preemption scheduling — node nominated before binding |
| `READINESS GATES` | Custom readiness conditions |

### `kubectl get nodes`

```bash
kubectl get nodes
NAME                     STATUS   ROLES           AGE   VERSION
anythink-control-plane   Ready    control-plane   1h    v1.29.0
anythink-worker          Ready    <none>          1h    v1.27.0   ← version skew visible here
anythink-worker2         Ready    <none>          1h    v1.27.0
```

---

## 5. k3s vs k8s vs k9s

### k8s — Kubernetes (Full)

**One-line:** Full production-grade container orchestration — all features, all components.

- Used in production (EKS, GKE, AKS, on-prem)
- Heavy (~600MB binary for components)
- All components: etcd, API server, scheduler, controller-manager, etc.

### k3s — Lightweight Kubernetes

**One-line:** A certified, lightweight K8s distribution with half the memory footprint — designed for edge, IoT, and resource-constrained environments.

**Full name:** No official expansion — "k3s" because it's half the size of k8s (5 letters → 3)

```
k8s:  ~300MB RAM minimum per node
k3s:  ~512MB RAM for full server (master + worker)
```

**What k3s removes / replaces:**
| k8s Component | k3s Replacement |
|--------------|----------------|
| etcd | SQLite (default) or embedded etcd |
| Docker | containerd (built-in) |
| cloud-controller-manager | Removed (not needed on edge) |
| Multiple binaries | Single binary (~70MB) |

**Where k3s is used:**
- Raspberry Pi clusters
- Edge computing / IoT devices
- CI/CD pipelines (faster than kind)
- Development laptops with low RAM

```bash
# Install k3s (single command!)
curl -sfL https://get.k3s.io | sh -

# Check nodes
k3s kubectl get nodes
```

### k9s — Kubernetes Terminal UI

**One-line:** A terminal-based UI dashboard for K8s that lets you navigate and manage cluster resources interactively without typing full kubectl commands.

**What k9s is NOT:** It's not a K8s distribution. It's a **kubectl wrapper with a TUI (Terminal User Interface)**.

```
k9s features:
├── Real-time pod/node/service view (auto-refreshes)
├── Press 'l' → view logs
├── Press 'e' → edit resource YAML
├── Press 'd' → describe resource
├── Press 's' → shell into container (exec)
├── Press 'ctrl+d' → delete resource
├── Filter by namespace, label
└── Port-forward from UI
```

```bash
# Install k9s
brew install k9s        # macOS
# or download binary from github.com/derailed/k9s

# Launch
k9s
k9s --namespace production
k9s --context kind-Anythink
```

### Comparison

| | k8s | k3s | k9s |
|-|-----|-----|-----|
| **What** | Full orchestration platform | Lightweight K8s distro | Terminal UI for K8s |
| **Where used** | Production clusters | Edge, IoT, dev | Anywhere with kubectl access |
| **RAM** | 300MB+ per node | ~512MB total | Minimal (it's a UI) |
| **Purpose** | Run workloads | Run workloads (small) | Manage/inspect cluster |
| **Replaces** | Nothing (it IS K8s) | Full K8s in light env | Complex kubectl commands |

---

## 6. kubectl api-resources & get pods -owide

### `kubectl api-resources`

**One-line:** Lists ALL resource types (API objects) available in your cluster — built-in and custom (CRDs).

```bash
kubectl api-resources

NAME                              SHORTNAMES   APIVERSION                        NAMESPACED   KIND
bindings                                       v1                                true         Binding
componentstatuses                 cs           v1                                false        ComponentStatus
configmaps                        cm           v1                                true         ConfigMap
endpoints                         ep           v1                                true         Endpoints
events                            ev           v1                                true         Event
namespaces                        ns           v1                                false        Namespace
nodes                             no           v1                                false        Node
pods                              po           v1                                true         Pod
services                          svc          v1                                true         Service
deployments                       deploy       apps/v1                           true         Deployment
replicasets                       rs           apps/v1                           true         ReplicaSet
statefulsets                      sts          apps/v1                           true         StatefulSet
daemonsets                        ds           apps/v1                           true         DaemonSet
ingresses                         ing          networking.k8s.io/v1              true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1              true         NetworkPolicy
clusterroles                                   rbac.authorization.k8s.io/v1     false        ClusterRole
roles                                          rbac.authorization.k8s.io/v1     true         Role
```

**Column meanings:**
| Column | Meaning |
|--------|---------|
| `NAME` | Resource type name (used in kubectl) |
| `SHORTNAMES` | Abbreviation (e.g., `po` for pods) |
| `APIVERSION` | API group + version |
| `NAMESPACED` | true = lives in a namespace, false = cluster-wide |
| `KIND` | The object Kind used in YAML |

**Useful filters:**
```bash
# Only namespaced resources
kubectl api-resources --namespaced=true

# Only cluster-scoped resources
kubectl api-resources --namespaced=false

# Filter by API group
kubectl api-resources --api-group=apps

# Filter by verb
kubectl api-resources --verbs=list,get
```

---

## 7. Trivy — Image Scanner & CVE

### What is Trivy?

**One-line definition:** Trivy is an open-source, comprehensive **vulnerability scanner** for container images, filesystems, Git repos, Kubernetes clusters, and IaC files.

**Created by:** Aqua Security. CNCF project.

### What is CVE?

**Full form:** `Common Vulnerabilities and Exposures`

**One-line:** CVE is a **standardized identifier** for a publicly known cybersecurity vulnerability — a unique ID like `CVE-2023-44487` assigned to each known security flaw.

```
CVE-2023-44487  →  HTTP/2 Rapid Reset Attack (affected Nginx, Apache, etc.)
CVE-2021-44228  →  Log4Shell (critical Java logging vulnerability)
CVE-2022-0847   →  Dirty Pipe (Linux kernel privilege escalation)
```

**CVE ID Format:** `CVE-[YEAR]-[NUMBER]`

### CVE Severity Levels

| Severity | CVSS Score | Meaning | Action |
|----------|-----------|---------|--------|
| **CRITICAL** | 9.0 – 10.0 | Remote code execution, data breach possible NOW | Fix immediately, do not deploy |
| **HIGH** | 7.0 – 8.9 | Significant risk, easily exploitable | Fix within 24-72 hours |
| **MEDIUM** | 4.0 – 6.9 | Moderate risk, requires specific conditions | Fix in next sprint |
| **LOW** | 0.1 – 3.9 | Minimal risk, hard to exploit | Fix in planned maintenance |
| **UNKNOWN** | N/A | No CVSS score assigned yet | Investigate |

### Trivy Commands Deep Dive

```bash
# Scan an image (pulls from Docker Hub)
trivy image nginx

# Scan with severity filter
trivy image nginx --severity CRITICAL
trivy image nginx --severity CRITICAL,HIGH

# Scan a specific image version
trivy image nginx:1.25.3-alpine

# Scan local image (already pulled)
trivy image --input nginx.tar

# Output formats
trivy image nginx --format json --output report.json
trivy image nginx --format table   # default
trivy image nginx --format sarif   # for GitHub Security tab

# Scan Kubernetes cluster
trivy k8s --report summary cluster

# Scan a running pod's image
trivy k8s pod/nginx

# Ignore unfixed vulnerabilities
trivy image nginx --ignore-unfixed

# Fail CI if CRITICAL found (exit code 1)
trivy image nginx --exit-code 1 --severity CRITICAL
```

### Sample Trivy Output

```
nginx:latest (debian 12.4)
================================
Total: 87 (CRITICAL: 3, HIGH: 14, MEDIUM: 28, LOW: 42)

┌───────────────────┬────────────────┬──────────┬────────┬───────────────────┬─────────────────────────────────┐
│      Library      │  Vulnerability │ Severity │ Status │  Installed Ver    │ Fixed Version                   │
├───────────────────┼────────────────┼──────────┼────────┼───────────────────┼─────────────────────────────────┤
│ openssl           │ CVE-2023-5678  │ CRITICAL │ fixed  │ 3.0.10            │ 3.0.12                          │
│ libssl3           │ CVE-2023-4807  │ HIGH     │ fixed  │ 3.0.10            │ 3.0.11                          │
│ zlib1g            │ CVE-2023-45853 │ CRITICAL │ fixed  │ 1.2.13            │ 1.3.0                           │
└───────────────────┴────────────────┴──────────┴────────┴───────────────────┴─────────────────────────────────┘
```

### Why Trivy? — Use Cases

| Use Case | Why |
|----------|-----|
| **Pre-deploy scan** | Catch vulnerabilities before they reach production |
| **CI/CD gate** | Block deployment if CRITICAL CVEs found |
| **Registry scan** | Scan images in ECR, GCR, ACR continuously |
| **Base image selection** | Compare `nginx:latest` vs `nginx:alpine` — alpine has fewer CVEs |
| **Compliance** | PCI-DSS, SOC2, HIPAA require vulnerability management |
| **K8s cluster scan** | Find misconfigured RBAC, exposed secrets, privileged pods |

### Image Scanner Tools Comparison

| Tool | By | What It Scans | K8s Native |
|------|----|--------------| ----------|
| **Trivy** | Aqua Security / CNCF | Images, FS, K8s, IaC, Git | ✅ Yes |
| **Grype** | Anchore | Images, FS | ⚠️ Partial |
| **Snyk** | Snyk | Images, code, IaC | ✅ Yes |
| **Clair** | Quay/Red Hat | Images (registry-focused) | ❌ No |
| **Falco** | Sysdig / CNCF | Runtime behavior | ✅ Yes |
| **Twistlock/Prisma** | Palo Alto | Full stack | ✅ Yes |

### Trivy in Your Aegis-Stack Project

```bash
# Your production workflow
trivy image <your-app-image> --severity CRITICAL,HIGH --exit-code 1

# Scan entire cluster
trivy k8s --report all --format json cluster > cluster-report.json

# With Kyverno integration: Kyverno can block admission if image not scanned
# (via image verification policies — Cosign signature check)
```

---

## 8. Kyverno — Policy Engine Deep Dive

### What is Kyverno?

**One-line definition:** Kyverno is a **Kubernetes-native policy engine** that validates, mutates, and generates K8s resources using policies written as Kubernetes CRDs — no Rego/OPA needed.

**Name origin:** Greek for "govern"

### Why Kyverno? The Problem It Solves

Without Kyverno, in production:
```
Developer deploys pod with:
  - image: nginx:latest        ← no pinned version
  - no resource limits         ← can consume all node memory
  - privileged: true           ← full root access to host
  - no securityContext         ← runs as root in container

Result: Security breach, noisy neighbor, cluster instability
```

With Kyverno:
```
Same deployment attempt:
  → Kyverno intercepts via Admission Controller
  → Policy: "no latest tag" → REJECT
  → Policy: "require resource limits" → REJECT
  → Policy: "no privileged containers" → REJECT
  → Deployment BLOCKED with clear error message
```

### Kyverno Prerequisites & Installation (Helm)

```bash
# Step 1: Add Helm repo
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Step 2: Install Kyverno (with HA for production)
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set replicaCount=3 \               # HA: 3 replicas
  --set admissionController.replicas=3

# Step 3: Verify
kubectl get pods -n kyverno
NAME                                        READY   STATUS    RESTARTS
kyverno-admission-controller-xxx            1/1     Running   0
kyverno-background-controller-xxx           1/1     Running   0
kyverno-cleanup-controller-xxx              1/1     Running   0
kyverno-reports-controller-xxx              1/1     Running   0
```

### Kyverno Architecture — 4 Controllers

```
K8s API Server
     │
     │  Webhook (ValidatingWebhookConfiguration / MutatingWebhookConfiguration)
     ▼
┌──────────────────────────────────────────────────────────────┐
│                        KYVERNO                               │
│                                                              │
│  ┌─────────────────────┐   ┌──────────────────────────────┐ │
│  │ Admission Controller │   │   Background Controller      │ │
│  │                     │   │                              │ │
│  │ • Validate          │   │ • Scans existing resources   │ │
│  │ • Mutate            │   │ • Applies policies to old    │ │
│  │ • Generate          │   │   resources (not just new)   │ │
│  └─────────────────────┘   └──────────────────────────────┘ │
│                                                              │
│  ┌─────────────────────┐   ┌──────────────────────────────┐ │
│  │  Reports Controller  │   │   Cleanup Controller         │ │
│  │                     │   │                              │ │
│  │ • Creates Policy-   │   │ • Deletes resources by       │ │
│  │   Reports CRDs      │   │   policy (TTL-based)         │ │
│  │ • Compliance view   │   │ • Scheduled cleanup          │ │
│  └─────────────────────┘   └──────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### Kyverno CRDs

| CRD | Purpose |
|-----|---------|
| `ClusterPolicy` | Cluster-wide policy |
| `Policy` | Namespace-scoped policy |
| `PolicyReport` | Results of policy evaluation per namespace |
| `ClusterPolicyReport` | Cluster-wide policy results |
| `CleanupPolicy` | TTL-based resource cleanup |
| `GlobalContextEntry` | Shared data across policies |

### Kyverno Workflow — API Request Flow

```
kubectl apply -f pod.yaml
        │
        ▼
kube-apiserver
  │
  ├── Authentication (who are you?)
  ├── Authorization (RBAC — can you do this?)
  │
  ├── MutatingAdmissionWebhook ──────► Kyverno Admission Controller
  │                                         │
  │                                    MUTATE rules:
  │                                    • Add labels
  │                                    • Set default resource limits
  │                                    • Add imagePullPolicy: Always
  │                                    • Inject sidecar annotations
  │                                         │
  │   ◄──────── mutated object ─────────────┘
  │
  ├── Schema validation
  │
  ├── ValidatingAdmissionWebhook ─────► Kyverno Admission Controller
  │                                         │
  │                                    VALIDATE rules:
  │                                    • No latest tag?
  │                                    • Resource limits set?
  │                                    • No privileged?
  │                                    • SecurityContext required?
  │                                         │
  │                                    ALLOW or DENY
  │
  └── Write to etcd (if allowed)
```

### Your Aegis-Stack Kyverno Policies

#### `kustomization.yml`

```yaml
# kustomization.yml — orchestrates all policy files
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - require-security-context.yml
  - disallow-latest-tag.yml
  - allow-tags.yml
  - require-resource-limits.yml
```

#### `require-security-context.yml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-security-context
  annotations:
    policies.kyverno.io/description: |
      Require all pods to have a securityContext defined.
      Prevents running as root, requires read-only root filesystem.
spec:
  validationFailureAction: Enforce   # Enforce=block, Audit=log only
  background: true
  rules:
  - name: require-pod-security-context
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Pod must have a securityContext defined."
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true           # Must not run as root
            runAsUser: ">0"              # UID must be > 0
            seccompProfile:
              type: RuntimeDefault       # Must have seccomp profile

  - name: require-container-security-context
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "All containers must have securityContext with readOnlyRootFilesystem."
      pattern:
        spec:
          containers:
          - securityContext:
              readOnlyRootFilesystem: true
              allowPrivilegeEscalation: false
              capabilities:
                drop: ["ALL"]
```

#### `disallow-latest-tag.yml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Image tag 'latest' is not allowed. Use a specific version tag."
      pattern:
        spec:
          containers:
          - image: "!*:latest"          # Image must NOT end in :latest
          initContainers:
          - image: "!*:latest"
```

#### `allow-tags.yml` — Image Authentication / Allowlist

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-registry
spec:
  validationFailureAction: Enforce
  background: false                    # Must run at admission time
  rules:
  - name: verify-image-signature       # Cosign signature verification
    match:
      any:
      - resources:
          kinds: [Pod]
    verifyImages:
    - imageReferences:
      - "ghcr.io/your-org/*"           # Only allow your org's images
      attestors:
      - entries:
        - keyless:
            subject: "https://github.com/your-org/*"
            issuer: "https://token.actions.githubusercontent.com"
    - imageReferences:
      - "docker.io/library/*"          # Allow official Docker Hub images
      mutateDigest: true               # Replace tag with digest (immutable)
      verifyDigest: true
```

### Kyverno Use Cases in Production

| Policy | Problem Solved | Without It |
|--------|---------------|-----------|
| Disallow `latest` tag | Image drift — `latest` changes silently | Unpredictable deployments |
| Require resource limits | Noisy neighbor — one pod consumes all node CPU/memory | Node OOM, evictions |
| Require securityContext | Container escapes, privilege escalation | Security breach |
| Require labels | No visibility, can't filter resources | Chaos in production |
| Verify image signatures | Supply chain attack — someone pushes malicious image | Code injection |
| Auto-generate NetworkPolicy | New namespace has no network isolation | Lateral movement attack |
| Require probes | Unhealthy pods receive traffic | Bad user experience |

### kube-linter with Kyverno

```bash
# Before applying YAML, lint it
kube-linter lint require-security-context.yml

# Example output:
KubeLinter v0.6.4
require-security-context.yml: (object: kyverno/ClusterPolicy) container "nginx" 
  does not have a read-only root file system (check: no-read-only-root-fs)
  
# Then run Kyverno policy test
kyverno test .
```

---

## 9. kube-linter & kube-bench

### kube-linter

**One-line definition:** kube-linter is a **static analysis tool** that checks Kubernetes YAML manifests for security issues and best-practice violations before they are applied to a cluster.

**When it runs:** BEFORE deployment — in your local machine or CI/CD pipeline.

```bash
# Install
brew install kube-linter

# Lint a file
kube-linter lint deployment.yaml

# Lint entire directory
kube-linter lint ./k8s/

# Lint with specific checks
kube-linter lint deployment.yaml --include no-read-only-root-fs,no-extensions-v1beta

# Custom config
cat .kube-linter.yaml
---
checks:
  addAllBuiltIn: true
  exclude:
    - "unset-cpu-requirements"    # Allow no CPU limits (if you use VPA)
```

**What kube-linter checks:**
- No resource limits set
- No liveness/readiness probes
- Container running as root
- Privilege escalation allowed
- No read-only root filesystem
- Deprecated API versions used
- Missing labels
- Privileged containers

### kube-bench

**One-line definition:** kube-bench is a tool that checks whether your Kubernetes cluster is configured securely according to **CIS (Center for Internet Security) Kubernetes Benchmark** standards.

**What it checks:** Not your app manifests — the **cluster infrastructure itself**.

```bash
# Run kube-bench as a K8s Job
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View results
kubectl logs job/kube-bench

# Run on master node
kube-bench run --targets master

# Run on worker node  
kube-bench run --targets node
```

**Sample kube-bench output:**
```
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
[PASS] 1.1.1 Ensure that the --anonymous-auth argument is set to false
[FAIL] 1.1.2 Ensure that the --token-auth-file parameter is not set
[WARN] 1.1.3 Ensure that the --DenyServiceExternalIPs is not set
[PASS] 1.1.6 Ensure that the --kubelet-certificate-authority argument is set

== Summary master ==
42 checks PASS
8 checks FAIL
11 checks WARN
```

### Comparison: kube-linter vs kube-bench vs Trivy

| Tool | What It Checks | When | Output |
|------|---------------|------|--------|
| **kube-linter** | YAML manifests — best practices | Before apply (CI) | Policy violations |
| **kube-bench** | Cluster config — CIS benchmark | After cluster setup | Pass/Fail security checks |
| **Trivy** | Container images + cluster + IaC | Before deploy + ongoing | CVE vulnerabilities |
| **Kyverno** | Live admission requests | At deploy time (runtime) | Allow/Deny decisions |

---

## 10. Rate Limiting — 429 Error & Backoff Algorithm

### What is 429 Error?

**One-line:** HTTP 429 `Too Many Requests` — the server is rejecting your request because you've **exceeded the allowed rate limit** (too many requests in a given time window).

### Where 429 Appears in K8s

```
kubectl get pods   →  429 Too Many Requests from kube-apiserver
Prometheus scrape  →  429 from metrics endpoint
Trivy image scan   →  429 from Docker Hub (pull rate limit)
Helm install       →  429 from chart repository
```

**kube-apiserver rate limiting:**
```yaml
# kube-apiserver flags
--max-requests-inflight=400         # Max concurrent non-mutating requests
--max-mutating-requests-inflight=200 # Max concurrent mutating requests

# When exceeded → 429 with Retry-After header
HTTP/1.1 429 Too Many Requests
Retry-After: 1
Content-Type: application/json

{"kind":"Status","apiVersion":"v1","status":"Failure",
 "message":"Too many requests, please try again later.",
 "reason":"TooManyRequests","code":429}
```

### Backoff Algorithm — Exponential Backoff

**One-line:** Exponential backoff is a **retry strategy** where each retry waits exponentially longer than the previous one, plus random jitter, to avoid thundering herd problems.

**Why "exponential":**
```
Attempt 1: wait 1s   → retry
Attempt 2: wait 2s   → retry
Attempt 3: wait 4s   → retry
Attempt 4: wait 8s   → retry
Attempt 5: wait 16s  → retry
Attempt 6: wait 32s  → give up (max retries)
```

**Formula:** `wait = min(cap, base * 2^attempt) + random_jitter`

**With jitter (prevents thundering herd):**
```
1000 clients all get 429 at the same time
Without jitter: all retry at t=2s → server gets 1000 requests simultaneously again
With jitter:    each waits 2s ± random(0-1s) → spread out → server recovers
```

**kubectl backoff implementation:**
```go
// K8s client-go uses exponential backoff
backoff := wait.Backoff{
    Steps:    5,               // Max 5 retries
    Duration: 10 * time.Millisecond,
    Factor:   1.0,
    Jitter:   0.1,
}
err := wait.ExponentialBackoff(backoff, func() (bool, error) {
    result, err := kubeClient.CoreV1().Pods("default").List(ctx, opts)
    if errors.IsTooManyRequests(err) {
        return false, nil   // Retry
    }
    return true, err        // Success or non-retryable error
})
```

**Docker Hub rate limiting (Trivy use case):**
```bash
# Docker Hub: 100 pulls/6hr (unauthenticated), 200/6hr (free account)
# Trivy hits this when scanning many images

# Solution: Authenticate Trivy
trivy image --username $DOCKER_USER --password $DOCKER_PASS nginx

# Or use a registry mirror
trivy image --registry-mirror https://mirror.gcr.io nginx
```

---

## 11. Init Container vs Sidecar Container

### Init Container

**One-line definition:** An init container is a **special container that runs to completion BEFORE** the main application containers start — used for setup, configuration, and prerequisite checks.

### What Makes Init Containers Special

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  initContainers:
  - name: wait-for-db          # Init container 1 — RUNS FIRST
    image: busybox
    command: ['sh', '-c', 'until nc -z mysql 3306; do sleep 2; done']
    # Waits until MySQL is reachable on port 3306
    # EXIT CODE 0 = success → next init container starts

  - name: run-migrations        # Init container 2 — RUNS SECOND
    image: myapp:1.0
    command: ['python', 'manage.py', 'migrate']
    # Runs DB migrations before app starts
    # EXIT CODE 0 = success → main containers start

  containers:
  - name: app                   # Main container — RUNS LAST
    image: myapp:1.0
    command: ['python', 'manage.py', 'runserver']
```

### Init Container Rules

| Rule | Detail |
|------|--------|
| **Sequential** | Init containers run one at a time, in order |
| **Must exit 0** | If any init container fails (non-zero exit), the pod restarts |
| **Blocking** | Main containers do NOT start until ALL init containers complete |
| **Separate image** | Can use different, minimal images (busybox, alpine) |
| **No probes** | No liveness/readiness probes on init containers |
| **Restartable** | If pod restarts, init containers run again (unless `restartPolicy: OnFailure`) |

### Pod Lifecycle with Init Containers

```
kubectl apply pod.yaml
       │
       ▼
Pod Status: Pending
       │
       ▼
Init Container 1 starts → runs → EXIT 0
       │
       ▼
Init Container 2 starts → runs → EXIT 0
       │
       ▼
Main Container starts
       │
       ▼
Pod Status: Running
       │
       ▼
Probes (liveness, readiness) activate
       │
       ▼
Traffic flows to pod ✓
```

**If init container fails:**
```
Init Container 1 → EXIT 1 (failure)
       │
       ▼
Pod Status: Init:Error
       │
       ▼ (based on restartPolicy)
Pod restarts → Init Container 1 runs again
       │
       ▼ (after maxRetries)
Pod Status: CrashLoopBackOff
```

### Sidecar Container

**One-line definition:** A sidecar container **runs alongside the main container** within the same pod — sharing its network and storage — to provide supporting functionality without modifying the main application.

**New in K8s 1.29:** Native sidecar support (`initContainers` with `restartPolicy: Always`) — sidecars now start before main containers but don't block them, and survive main container restarts.

```yaml
spec:
  initContainers:
  - name: log-collector          # Native sidecar (K8s 1.29+)
    image: fluentd:v1.16
    restartPolicy: Always        # THIS makes it a sidecar — runs forever
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app    # Shared volume with sidecar
```

### Classic Sidecar Pattern (Pre-1.29)

```yaml
spec:
  containers:
  - name: app                    # Main container
    image: myapp:1.0

  - name: log-shipper            # Sidecar container
    image: fluentd:v1.16         # Runs ALONGSIDE main container
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  - name: envoy-proxy            # Another sidecar (service mesh)
    image: envoyproxy/envoy:v1.28
    # Intercepts all network traffic to/from main container
```

### Sidecar Use Cases

| Pattern | Sidecar | What It Does |
|---------|---------|-------------|
| **Logging** | Fluentd, Filebeat | Collects logs from shared volume, ships to Elasticsearch |
| **Metrics** | Prometheus exporter | Exposes `/metrics` endpoint for main app |
| **Service Mesh** | Envoy (Istio), Linkerd | Intercepts all network I/O, adds mTLS, circuit breaking |
| **Secret sync** | Vault agent | Fetches secrets from Vault, writes to shared volume |
| **Config refresh** | Config watcher | Watches ConfigMap changes, signals main app to reload |
| **Auth proxy** | OAuth2 proxy | Handles authentication before traffic reaches main app |

### Distroless Images

**One-line:** Distroless images contain **only the application and its runtime dependencies** — no shell, no package manager, no OS utilities — minimizing attack surface.

```dockerfile
# Standard image — has bash, apt, curl, etc.
FROM debian:12
COPY myapp /myapp
CMD ["/myapp"]

# Distroless — has nothing except what myapp needs
FROM gcr.io/distroless/base-debian12
COPY myapp /myapp
CMD ["/myapp"]
```

**Benefits:**
- Far fewer CVEs (no system tools to exploit)
- Smaller image size
- No shell access (attacker can't exec in)

**Challenge:** No shell = can't `kubectl exec -- bash`
**Solution:** Use ephemeral debug containers:
```bash
kubectl debug -it mypod --image=busybox --target=myapp
```

### Init Container vs Sidecar — Comparison

| | Init Container | Sidecar Container |
|-|---------------|------------------|
| **When runs** | BEFORE main container | ALONGSIDE main container |
| **Lifecycle** | Runs to completion (exits) | Runs for pod lifetime |
| **Exit behavior** | Must exit 0 to proceed | Restart if it crashes |
| **Purpose** | Setup, prerequisites | Support, observation |
| **Blocks main?** | Yes — main waits for all | No — runs in parallel |
| **Probes** | No probes supported | Can have probes |
| **Examples** | DB migration, wait-for-service | Log shipping, proxy, metrics |
| **Shares network?** | No (has own namespace) | Yes (same pod network) |
| **Shares volumes?** | Yes (passed to main) | Yes (shared with main) |

---

## 12. Pod Lifecycle & Termination

### Pod Phases

| Phase | Meaning |
|-------|---------|
| `Pending` | Pod accepted by API server; waiting for scheduling or image pull |
| `Running` | Pod bound to node; at least one container running |
| `Succeeded` | All containers exited with code 0 (Jobs) |
| `Failed` | All containers terminated; at least one exited non-zero |
| `Unknown` | Node communication lost; pod state unknown |

### Container States

| State | Meaning |
|-------|---------|
| `Waiting` | Container waiting to start (pulling image, init containers running) |
| `Running` | Container executing |
| `Terminated` | Container finished (exit code 0 or non-zero) |

### Full Pod Lifecycle — Step by Step

```
kubectl apply -f pod.yaml
        │
        ▼
Phase: Pending
  │  ├── API server validates and stores in etcd
  │  ├── Scheduler assigns node
  │  ├── kubelet sees pod assignment
  │  ├── kubelet pulls images
  │  └── Init containers run (sequentially)
        │
        ▼
Phase: Running
  │  ├── Main containers start
  │  ├── postStart lifecycle hook (if defined)
  │  ├── Startup probe (if defined) — must pass before others activate
  │  ├── Liveness probe — kill & restart if fails
  │  ├── Readiness probe — remove from Service endpoints if fails
  │  └── Application serving traffic
        │
        ▼
Phase: Terminating (on delete)
  │  ├── Pod gets deletionTimestamp set in etcd
  │  ├── preStop hook executes
  │  ├── SIGTERM sent to container PID 1
  │  ├── Grace period countdown (default 30s)
  │  ├── Traffic stops (removed from Service endpoints)
  │  └── If not exited after grace period → SIGKILL
        │
        ▼
Phase: Terminated / Deleted
```

### Probes Deep Dive

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0

    startupProbe:                    # Runs FIRST — must pass before others activate
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30           # Allow 30 * 10s = 5min for slow startup
      periodSeconds: 10

    livenessProbe:                   # Is the app alive? If fails → restart container
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10        # Wait 10s before first check
      periodSeconds: 15              # Check every 15s
      failureThreshold: 3            # Fail 3 times → restart

    readinessProbe:                  # Is the app ready for traffic? If fails → remove from Service
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
      successThreshold: 1
```

**Probe types:**

| Type | Method | Use Case |
|------|--------|----------|
| `httpGet` | HTTP GET to path/port | Web servers, REST APIs |
| `tcpSocket` | TCP connection to port | Databases, non-HTTP services |
| `exec` | Run command inside container | Custom health check scripts |
| `grpc` | gRPC health check protocol | gRPC services |

### Graceful Termination — Deep Dive

```
kubectl delete pod nginx
        │
        ▼ Step 1: API server sets deletionTimestamp on pod object
        │         (pod is NOT deleted yet — just marked for deletion)
        │
        ▼ Step 2: Endpoint controller removes pod IP from Service endpoints
        │         (traffic stops flowing to this pod)
        │         ← This happens in PARALLEL with preStop, not before!
        │
        ▼ Step 3: kubelet detects deletionTimestamp
        │
        ▼ Step 4: preStop hook executes (if defined)
        │         spec.containers[*].lifecycle.preStop
        │         Example: drain connections, deregister from discovery
        │
        ▼ Step 5: SIGTERM sent to container PID 1
        │         App should catch SIGTERM and start graceful shutdown:
        │         - Stop accepting new requests
        │         - Finish in-flight requests
        │         - Close DB connections
        │         - Flush logs
        │
        ▼ Step 6: Grace period countdown (terminationGracePeriodSeconds: 30)
        │         Default = 30 seconds
        │
        ├── If container exits during grace period → Pod deleted ✓
        │
        └── If container still running after grace period:
                │
                ▼ Step 7: SIGKILL — immediate forceful termination
                          No cleanup possible after this point
```

### Graceful Termination Configuration

```yaml
apiVersion: v1
kind: Pod
spec:
  terminationGracePeriodSeconds: 60   # Give app 60s to shutdown gracefully

  containers:
  - name: app
    image: myapp:1.0
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]   # Wait 5s for LB to drain

      postStart:                        # Runs AFTER container starts
        exec:
          command: ["/bin/sh", "-c", "echo 'container started' >> /var/log/app.log"]
```

**The "sleep 5" preStop pattern:** Because endpoint removal and preStop run in parallel, a small sleep in preStop ensures the load balancer has time to stop sending traffic before SIGTERM is sent.

### Force Delete (Dangerous!)

```bash
# Normal graceful delete (waits for grace period)
kubectl delete pod nginx

# Force delete — skips grace period, SIGKILL immediately
kubectl delete pod nginx --force --grace-period=0

# When to use force delete:
# - Pod stuck in Terminating state (node is dead)
# - etcd shows pod but node is gone
# NEVER use in production for database pods — data corruption risk!
```

---

## 13. RuntimeClass — Kata Containers

### What is RuntimeClass?

**One-line definition:** RuntimeClass is a K8s API object that lets you specify **which container runtime** (and how isolated) a pod should use — enabling per-pod runtime selection.

**Why it exists:** Not all workloads need the same security/performance tradeoff:
- A trusted internal service → use `containerd` (fast, lightweight)
- An untrusted user-provided code → use `kata-containers` (VM-level isolation)
- A GPU workload → use specialized runtime

### RuntimeClass — Object Definition

```yaml
# Define a RuntimeClass
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-isolated            # Name used in pod spec
handler: kata                    # Maps to runtime on node (configured in containerd)
overhead:
  podFixed:
    memory: "160Mi"              # Extra overhead for VM
    cpu: "250m"
scheduling:
  nodeSelector:
    kata-runtime: "true"         # Only schedule on nodes with kata installed
```

```yaml
# Use RuntimeClass in a Pod
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-workload
spec:
  runtimeClassName: kata-isolated   # Use kata containers runtime
  containers:
  - name: app
    image: user-provided-app:1.0
```

### Kata Containers — What Are They?

**One-line:** Kata Containers runs each pod inside a **lightweight virtual machine** (using QEMU or Firecracker) while maintaining the K8s pod interface — providing VM-level isolation with container-level speed.

```
Standard container (containerd + runc):
  Host Kernel
  └── Container (process with namespaces + cgroups)
       └── App

Kata Container:
  Host Kernel (hypervisor)
  └── Lightweight VM (separate kernel!)
       └── Container (runc inside VM)
            └── App

Security boundary: Entire VM kernel, not just namespaces
```

**When to use Kata:**
- Multi-tenant clusters (running untrusted code)
- Regulatory compliance (strong isolation required)
- Serverless platforms (FaaS — unknown user code)

### Container Runtimes Comparison

| Runtime | Isolation | Speed | Security | Use Case |
|---------|-----------|-------|----------|---------|
| **runc** (default) | Namespaces + cgroups | Fastest | Standard | Trusted workloads |
| **kata-containers** | Full VM (QEMU/Firecracker) | ~10% overhead | Strongest | Untrusted code, multi-tenant |
| **gVisor (runsc)** | Syscall interception in userspace | Medium | High | Moderate isolation need |
| **Nabla** | Unikernel approach | Fast | High | Specialized use cases |

### RuntimeClass with containerd Configuration

```toml
# /etc/containerd/config.toml on worker nodes
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
  
  # Default runtime
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"

  # Kata containers runtime  
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
    runtime_type = "io.containerd.kata.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata.options]
      ConfigPath = "/opt/kata/share/defaults/kata-containers/configuration.toml"

  # gVisor runtime
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
```

---

## 14. Security Context Deep Dive

### What is SecurityContext?

**One-line definition:** SecurityContext defines **privilege and access control settings** for a Pod or container — controlling what Linux capabilities, user IDs, filesystem permissions, and kernel features it can use.

### Pod-Level vs Container-Level SecurityContext

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:                   # Pod-level — applies to ALL containers
    runAsNonRoot: true               # Container must not run as root (UID 0)
    runAsUser: 1000                  # Run all containers as UID 1000
    runAsGroup: 3000                 # Primary GID = 3000
    fsGroup: 2000                    # Volumes owned by GID 2000
    seccompProfile:
      type: RuntimeDefault           # Apply default seccomp profile
    sysctls:                         # Kernel parameter tuning
    - name: net.core.somaxconn
      value: "1024"

  containers:
  - name: app
    image: myapp:1.0
    securityContext:                 # Container-level — overrides pod-level
      readOnlyRootFilesystem: true   # Container cannot write to its own filesystem
      allowPrivilegeEscalation: false # Cannot gain more privileges than parent
      privileged: false              # Not a privileged container (no host access)
      capabilities:
        drop: ["ALL"]                # Drop ALL Linux capabilities
        add: ["NET_BIND_SERVICE"]    # Add back only what's needed (bind port <1024)
```

### Linux Capabilities Explained

Traditional Unix model: root can do everything, non-root cannot.
Linux capabilities: break root's power into individual granular privileges.

| Capability | What It Allows | Drop If Not Needed |
|-----------|---------------|-------------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | Keep only for web servers |
| `NET_ADMIN` | Network interface configuration | Drop for most apps |
| `SYS_ADMIN` | Many sys operations (mount, hostname) | Almost always drop |
| `CHOWN` | Change file ownership | Drop for most apps |
| `SETUID` | Change process UID | Drop if not needed |
| `KILL` | Send signals to any process | Drop for most apps |
| `SYS_PTRACE` | Trace/debug processes | Drop (security risk) |
| `ALL` | Everything — super dangerous | Always drop, then add back minimum |

**Best practice:**
```yaml
capabilities:
  drop: ["ALL"]              # Drop everything
  add: ["NET_BIND_SERVICE"]  # Add back ONLY what's needed
```

### Seccomp — System Call Filtering

**One-line:** Seccomp (Secure Computing Mode) filters which **Linux system calls** a container process can make — reducing kernel attack surface.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault       # Use containerd's default seccomp profile
    # OR
    type: Localhost            # Use custom profile from node
    localhostProfile: profiles/my-profile.json
    # OR
    type: Unconfined           # No seccomp (dangerous, avoid in production)
```

**RuntimeDefault profile** blocks ~300+ dangerous syscalls while allowing common ones. Containers that try to call blocked syscalls get `EPERM` (Operation not permitted).

### AppArmor — MAC for Containers

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: runtime/default
    # OR load a custom profile:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/my-nginx-profile
```

### Pod Security Standards (PSS) — Replacing PSP

K8s 1.25+ removed PodSecurityPolicy. Replaced with Pod Security Standards:

| Level | Restrictions | Use For |
|-------|-------------|---------|
| **Privileged** | No restrictions | System components (kube-system) |
| **Baseline** | Minimal restrictions, prevents known escalations | General workloads |
| **Restricted** | Heavily restricted, follows best practices | High-security workloads |

```yaml
# Apply to namespace via label
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted    # Block non-compliant pods
    pod-security.kubernetes.io/audit: restricted      # Log non-compliant pods
    pod-security.kubernetes.io/warn: restricted       # Warn on non-compliant pods
```

---

## 15. Interview Q&A Master List

### kind & Cluster Setup

**Q: What does `kind create cluster --name X --config file.yml` do?**
> Creates a local K8s cluster using Docker containers as nodes. The config file defines the number of nodes, their roles (control-plane/worker), K8s versions, port mappings, and extra configuration. Each node becomes a Docker container running kubelet + containerd internally.

**Q: Why might you use kind over minikube?**
> kind supports true multi-node clusters within Docker, making it more realistic for testing scheduling, node affinity, and failure scenarios. It's also more CI/CD friendly since it only needs Docker, not a VM driver.

---

### Version Skew

**Q: Why must the master node always run a newer or equal version than worker nodes?**
> The kube-apiserver is designed to be backward compatible with older kubelet clients. K8s guarantees support for kubelet versions up to 2 minor versions behind the API server. Workers running a newer version than the master would use API features the master doesn't understand, causing kubelet registration failures and nodes staying NotReady.

**Q: What is the supported version skew between kubectl and kube-apiserver?**
> kubectl can be 1 minor version ahead or behind the API server. E.g., kubectl v1.30 with API server v1.29 is supported.

---

### Stateless vs Stateful

**Q: You have a Redis cache and a React frontend — which K8s resource for each?**
> React frontend → Deployment (stateless, any replica handles any request). Redis used as a persistent cache (with AOF/RDB) → StatefulSet (needs stable storage, stable identity). Redis used as ephemeral cache only → Deployment is acceptable.

**Q: What happens when a StatefulSet pod is deleted?**
> K8s creates a new pod with the SAME name (e.g., `mysql-1`) and binds it to the SAME PVC. Data is preserved because the PVC is not deleted. The pod gets the same stable DNS name and picks up from where it left off.

---

### kubectl Commands

**Q: Explain `kubectl exec -it pod -- bash`. What does -i and -t do separately?**
> `-i` (interactive) keeps STDIN open so you can type commands. `-t` (tty) allocates a pseudo-terminal giving you a proper shell experience with colors, backspace support, and shell prompt. Without `-t`, commands work but you get no prompt and no terminal features. Without `-i`, you can't type input. Together `-it` gives a full interactive terminal session.

**Q: What does `kubectl get pods -owide` show that `kubectl get pods` doesn't?**
> `-owide` adds: Pod IP address, the node the pod is running on, nominated node (preemption), and readiness gates. This is useful for debugging networking issues and understanding pod placement.

---

### k3s vs k8s vs k9s

**Q: When would you recommend k3s over k8s?**
> k3s for: edge computing (Raspberry Pi, IoT devices), single-node homelab setups, CI/CD environments needing fast K8s spin-up, teams with limited ops resources. k8s for: production multi-node, cloud-native full features (HPA, VPA, multiple storage classes), enterprise compliance requirements.

**Q: k9s is not a K8s distribution. What is it?**
> k9s is a terminal-based UI (TUI) for Kubernetes. It wraps kubectl commands in an interactive terminal interface, allowing you to navigate pods, view logs, exec into containers, edit resources, and delete objects — all without typing full kubectl commands. Productivity tool, not a K8s variant.

---

### Trivy & CVE

**Q: What is CVE and what are the severity levels?**
> CVE (Common Vulnerabilities and Exposures) is a standardized identifier for known security vulnerabilities (e.g., CVE-2021-44228). Severity levels based on CVSS score: CRITICAL (9.0-10.0, immediate risk), HIGH (7.0-8.9, significant risk), MEDIUM (4.0-6.9, moderate), LOW (0.1-3.9, minimal risk).

**Q: How would you integrate Trivy in CI/CD to block deployments with CRITICAL CVEs?**
> ```bash
> trivy image $IMAGE_NAME --severity CRITICAL --exit-code 1
> # Exit code 1 fails the CI pipeline if CRITICAL CVEs found
> # Combine with --ignore-unfixed to only fail on fixable vulnerabilities
> ```

**Q: Difference between Trivy and Falco?**
> Trivy is a static scanner — scans images before/at deploy time for known CVEs in packages. Falco is a runtime security tool — monitors actual system calls at runtime and alerts on suspicious behavior (container opening shell, reading /etc/passwd, etc.). Both are complementary.

---

### Kyverno

**Q: What is Kyverno and why is it needed?**
> Kyverno is a K8s-native policy engine that enforces security and operational policies via admission control. Without it, developers can deploy containers running as root, with no resource limits, using unverified images with critical CVEs — all of which cause security breaches and cluster instability.

**Q: What are Kyverno's 4 controllers and what do they do?**
> 1. **Admission Controller**: Intercepts API requests, validates and mutates resources at admission time. 2. **Background Controller**: Applies policies to existing resources (not just new ones). 3. **Reports Controller**: Creates PolicyReport CRDs showing compliance status. 4. **Cleanup Controller**: Deletes resources based on TTL/schedule policies.

**Q: Difference between Kyverno validationFailureAction: Enforce vs Audit?**
> `Enforce`: Blocks the API request — the resource is NOT created/updated if it violates policy. `Audit`: Allows the request but logs a violation in PolicyReport — useful for rolling out policies without breaking existing workloads.

**Q: How does Kyverno verify image signatures?**
> Via Cosign integration in `verifyImages` policy rules. Kyverno calls Cosign to verify that the image in an admission request has a valid cryptographic signature from a trusted source (e.g., GitHub Actions OIDC). Unsigned images are rejected.

---

### kube-linter & kube-bench

**Q: Difference between kube-linter and kube-bench?**
> kube-linter: Static analysis of YAML manifests — runs BEFORE deployment in CI/CD, checks for missing resource limits, no probes, running as root, deprecated APIs. kube-bench: Runs AFTER cluster setup, checks the cluster infrastructure itself against CIS K8s Benchmark (API server flags, etcd permissions, kubelet config, RBAC settings).

---

### 429 & Backoff

**Q: What causes HTTP 429 in Kubernetes and how does the client handle it?**
> 429 occurs when kube-apiserver's rate limits are exceeded (--max-requests-inflight exceeded) or when hitting external rate limits (Docker Hub image pulls). The K8s client-go library handles 429 automatically using exponential backoff with jitter — it reads the `Retry-After` header and waits that duration before retrying, doubling the wait on each failure.

**Q: What is a thundering herd problem and how does jitter solve it?**
> When 1000 clients all fail simultaneously and all retry at the same fixed interval, they all hit the server at the same time again, causing another failure cascade. Jitter adds random delay to each client's backoff, spreading retries over a time window so the server can recover incrementally.

---

### Init Container vs Sidecar

**Q: What is the difference between init container and sidecar container?**
> Init containers run sequentially BEFORE main containers start and must exit with code 0 — used for prerequisites (wait for DB, run migrations, copy configs). Sidecar containers run ALONGSIDE the main container for the pod's lifetime — used for supporting functions (log shipping, service mesh proxy, secret injection). Init containers block the pod from starting; sidecars don't.

**Q: Your init container is in CrashLoopBackOff. What is happening?**
> The init container is repeatedly failing (non-zero exit code). K8s restarts the pod each time an init container fails. CrashLoopBackOff means K8s is applying exponential backoff to the restart cycle. The main containers have never started because init containers must complete successfully first.

**Q: What is a distroless image and what's the tradeoff?**
> Distroless images contain only the app and its runtime dependencies — no shell, no package manager. Benefit: dramatically fewer CVEs, smaller attack surface. Tradeoff: cannot `kubectl exec -- bash` for debugging. Solution: use `kubectl debug` with ephemeral debug containers.

---

### Pod Termination

**Q: Walk me through what happens when you run `kubectl delete pod nginx`?**
> 1. API server sets `deletionTimestamp` on the pod object. 2. In parallel: endpoint controller removes pod from Service endpoints (traffic stops), AND kubelet detects deletionTimestamp. 3. kubelet runs `preStop` hook. 4. kubelet sends SIGTERM to container PID 1. 5. Grace period countdown starts (default 30s). 6. If container exits gracefully → pod deleted. If not → SIGKILL sent after grace period.

**Q: What does `--grace-period=0 --force` do? When would you use it?**
> Skips the grace period and sends SIGKILL immediately. Use only when a pod is stuck in `Terminating` state because its node is dead and kubelet can't report back. NEVER use on database pods — bypassing graceful shutdown can cause data corruption.

---

### RuntimeClass & Kata

**Q: What is RuntimeClass and why would you use it?**
> RuntimeClass is a K8s API object that lets you select which container runtime a pod uses. You'd use it to run untrusted workloads in Kata Containers (VM-level isolation) while running trusted internal services with standard runc (lower overhead). Enables per-pod security/performance tradeoffs.

**Q: How is Kata Containers different from standard runc containers?**
> runc uses Linux namespaces and cgroups — containers share the host kernel. Kata runs each pod inside a lightweight VM with its own separate kernel (using QEMU or Firecracker). A kernel exploit in a Kata container cannot escape to the host because it's contained within the VM's kernel. Much stronger security isolation at the cost of slightly higher overhead.

---

### Security Context

**Q: What does `allowPrivilegeEscalation: false` do?**
> It prevents a process inside the container from gaining more privileges than its parent process — specifically it sets the `no_new_privs` bit on the container process. This prevents exploits that use setuid binaries or capabilities to escalate from a low-privilege process to root.

**Q: What is the difference between `runAsNonRoot: true` and `runAsUser: 1000`?**
> `runAsNonRoot: true` is a guard — K8s rejects the pod if the image's configured user is root (UID 0), but doesn't specify WHICH non-root user. `runAsUser: 1000` explicitly sets UID 1000 for all processes. Best practice: use both together to be explicit and have the guard.

**Q: What are Pod Security Standards and what replaced PSP?**
> Pod Security Standards (PSS) replaced PodSecurityPolicy (removed in K8s 1.25). PSS defines three levels: Privileged (no restrictions), Baseline (prevents known escalations), Restricted (heavily restricted, best practices). Applied via namespace labels, not per-pod. Can also use Kyverno or OPA/Gatekeeper for more granular policy control.

---

## Appendix: Key One-Line Definitions

| Term | One-Line Definition |
|------|-------------------|
| **kind** | Tool to run K8s clusters as Docker containers locally |
| **k3s** | Lightweight, single-binary K8s for edge/IoT (uses SQLite instead of etcd) |
| **k9s** | Terminal UI for navigating and managing K8s clusters interactively |
| **CVE** | Common Vulnerabilities and Exposures — unique ID for known security flaws |
| **Trivy** | Open-source vulnerability scanner for images, filesystems, and K8s clusters |
| **Kyverno** | K8s-native policy engine that validates/mutates/generates resources via CRDs |
| **kube-linter** | Static YAML analysis tool checking manifests for security and best practice violations |
| **kube-bench** | Tool checking cluster infrastructure against CIS K8s security benchmark |
| **429** | HTTP Too Many Requests — server rejecting due to rate limit exceeded |
| **Exponential backoff** | Retry strategy where wait time doubles each retry + jitter to prevent thundering herd |
| **Init container** | Runs before main containers, must exit 0, used for prerequisites and setup |
| **Sidecar container** | Runs alongside main container for pod lifetime, provides supporting functions |
| **Distroless** | Container image with only app + runtime — no shell, no OS tools, minimal CVEs |
| **RuntimeClass** | K8s object selecting which container runtime a pod uses (runc, kata, gvisor) |
| **Kata containers** | Runs pods in lightweight VMs for VM-level isolation via K8s |
| **SecurityContext** | Defines privilege and access control (UID, capabilities, seccomp) for pod/container |
| **Seccomp** | Linux feature filtering which syscalls a container process can make |
| **cgroups** | Linux kernel feature limiting CPU/memory/IO for container process groups |
| **Grace period** | Time given to a pod to shut down cleanly after SIGTERM before SIGKILL |
| **preStop hook** | Lifecycle hook running before SIGTERM — used to drain connections gracefully |
| **Readiness probe** | Checks if container is ready to receive traffic; removes from Service if failing |
| **Liveness probe** | Checks if container is alive; restarts container if failing |
| **Startup probe** | Checks if container has started; delays liveness/readiness until it passes |
| **PSS** | Pod Security Standards — three levels (Privileged/Baseline/Restricted) replacing PSP |
| **OCI** | Open Container Initiative — standard specs for image format and container runtime |
| **CRI** | Container Runtime Interface — gRPC API between kubelet and container runtimes |
| **PVC** | PersistentVolumeClaim — request for storage, bound to a PersistentVolume |
| **StatefulSet** | K8s controller for stateful apps — stable identity, ordered ops, per-pod PVC |
| **Deployment** | K8s controller for stateless apps — manages ReplicaSets, rolling updates |

---

*Kubernetes Deep Dive — v2.0 | Covers: Architecture, Security, Networking, Policy, Lifecycle, Runtimes*
*Study guide for interviews and production operations*
