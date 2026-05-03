# ☸️ Kubernetes Deep Dive — Complete Study & Interview Guide

> **Covers:** Labels · Annotations · Cluster Architecture · Pods · ReplicaSets · Deployments · Volumes · Probes · OOM · Rollout/Rollback · SLO/SLA/SLI · and more

---

## Table of Contents

1. [Cluster Architecture Overview](#1-cluster-architecture-overview)
2. [Labels — Deep Dive](#2-labels--deep-dive)
3. [Annotations](#3-annotations)
4. [Pod Spec — Complete YAML Analysis](#4-pod-spec--complete-yaml-analysis)
5. [Volumes & Persistent Storage](#5-volumes--persistent-storage)
6. [Probe Types](#6-probe-types)
7. [Pod Status & Lifecycle](#7-pod-status--lifecycle)
8. [Replication Controller (RC)](#8-replication-controller-rc)
9. [ReplicaSet (RS)](#9-replicaset-rs)
10. [Selectors — matchLabels vs matchExpressions](#10-selectors--matchlabels-vs-matchexpressions)
11. [ownerReference & Scale-Down Algorithm](#11-ownerreference--scale-down-algorithm)
12. [Homogeneous vs Heterogeneous Pods](#12-homogeneous-vs-heterogeneous-pods)
13. [OOM Kill & Ephemeral Containers](#13-oom-kill--ephemeral-containers)
14. [Deployments — Full Deep Dive](#14-deployments--full-deep-dive)
15. [Rollout & Rollback](#15-rollout--rollback)
16. [Taints & Tolerations](#16-taints--tolerations)
17. [Pod Priority & Privilege](#17-pod-priority--privilege)
18. [SLO · SLA · SLI & Error Budget](#18-slo--sla--sli--error-budget)
19. [kubectl Cheat Sheet](#19-kubectl-cheat-sheet)
20. [Interview Q&A](#20-interview-qa)

---

## 1. Cluster Architecture Overview

```
CLUSTER
 ├── Control Plane (Master Node)
 │    ├── kube-apiserver        ← All kubectl commands hit here
 │    ├── etcd                  ← Key-value store (source of truth)
 │    ├── kube-scheduler        ← Assigns pods to nodes
 │    ├── kube-controller-mgr   ← Runs RC, RS, Deployment controllers
 │    └── cloud-controller-mgr  ← AWS/GCP/Azure integration
 │
 └── Worker Nodes
      ├── kubelet               ← Agent on each node; manages pod lifecycle
      ├── kube-proxy            ← Networking / iptables rules
      ├── container-runtime     ← containerd / CRI-O / Docker
      │
      └── Pod
           ├── Container(s)
           │    └── cgroup      ← CPU/Memory limits enforced here
           └── Image(s)         ← Pulled from registry
```

### Hierarchy (top → bottom)

```
Cluster → Node → Namespace → Deployment → ReplicaSet → Pod → Container → cgroup → Image
```

### Why Data Survives Cluster Deletion

Kubernetes itself is **stateless compute**. Data lives on **Persistent Volumes (PV)** — typically backed by cloud storage (EBS, GCS, Azure Disk) or NFS. When you delete a cluster:

- The PV object may be deleted, but the **underlying disk/storage is retained** if `reclaimPolicy: Retain`.
- You can recreate a new cluster, create a new PVC pointing to the same volume, and recover all data.

```
ReclaimPolicy options:
  Retain   → Disk kept; manual recovery needed
  Recycle  → Deprecated (basic scrub)
  Delete   → Disk deleted with PVC
```

---

## 2. Labels — Deep Dive

### What Is a Label?

A **label** is a `key=value` metadata pair attached to any Kubernetes object (Pod, Node, Service, etc.).

- Max key length: **63 characters** (prefix optional, up to 253 chars)
- Max value length: **63 characters**
- Characters allowed: alphanumeric, `-`, `_`, `.`

### Auto-Label Behavior

If you **do not** specify labels when creating a pod using `kubectl run`:

```bash
kubectl run mypod --image=nginx
# Kubernetes auto-applies:
#   run=mypod    ← pod name becomes the label value
```

### Why Use Labels?

| Use Case | Example |
|---|---|
| Grouping | `env=production`, `env=staging` |
| Selecting pods for a Service | `app=frontend` |
| ReplicaSet/Deployment targeting | `app=myapp, tier=backend` |
| Node affinity | `disktype=ssd` |
| Canary releases | `version=canary` |
| Cost attribution | `team=payments` |
| Blue-Green deployments | `slot=blue` / `slot=green` |

### Labels YAML Example

```yaml
# labels-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo                     # Pod name
  labels:                              # ← Key-value metadata
    app: webserver                     # app label
    env: production                    # environment label
    release: alpha1.1                  # release version label
    work: tut                          # custom label
    tier: frontend                     # tier label
spec:
  containers:
    - name: nginx
      image: nginx:1.21
```

> ⚠️ **Important:** You **cannot** add two labels with the **same key** in one pod.yml. Each key must be unique. You **can** have many different key-value pairs.

```yaml
# WRONG — duplicate key
labels:
  app: web
  app: backend     # ← Error! Duplicate key 'app'

# CORRECT — different keys
labels:
  app: web
  tier: backend    # ← OK, different key
```

### Label Commands

```bash
# Apply pod with labels
kubectl apply -f labels-pod.yml

# Show all labels
kubectl get pods --show-labels

# Filter by labels (equality)
kubectl get pods -l release=alpha1.1,work=tut

# Filter by single label
kubectl get pods -l run=mypodname

# NOT equal filter
kubectl get pods -l run!=mypodname

# Set-based selector
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l 'env notin (development)'

# Add a label to existing pod
kubectl label pod label-demo version=v2

# Overwrite existing label
kubectl label pod label-demo version=v3 --overwrite

# Remove a label (trailing -)
kubectl label pod label-demo version-

# Label a node
kubectl label node node01 disktype=ssd
```

### Label Selector Types

**Equality-based (RC, Service)**
```yaml
selector:
  app: myapp
  env: production
```

**Set-based (RS, Deployment — more powerful)**
```yaml
selector:
  matchLabels:
    app: myapp
  matchExpressions:
    - key: env
      operator: In
      values: [production, staging]
    - key: version
      operator: NotIn
      values: [deprecated]
    - key: critical
      operator: Exists
```

---
![image](https://github.com/abhijitray7810/Kubernetes-Concepts-Guide/blob/b41f1140852c06c9b2371b08fee379d4c8d17c9e/Kubernetes%20deep%20dive%203/pod-livecycle.png)
## 3. Annotations

### What Are Annotations?

Annotations are **non-identifying metadata** — they are NOT used for selection. They store arbitrary information for tools, humans, or automation.

```yaml
metadata:
  annotations:
    build-timestamp: "2024-01-15T10:30:00Z"
    git-commit: "abc123def456"
    maintainer: "devops@company.com"
    description: "Frontend web server pod"
    feature-flag: "new-checkout-enabled"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

### Labels vs Annotations

| Feature | Labels | Annotations |
|---|---|---|
| Used for selection? | ✅ Yes | ❌ No |
| Max value size | 63 chars | Unlimited (large JSON OK) |
| Queryable with -l? | ✅ Yes | ❌ No |
| Purpose | Identity/grouping | Metadata/tooling |
| Examples | `env=prod` | build timestamp, URLs, configs |

### Annotation Commands

```bash
# Add annotation
kubectl annotate pod label-demo maintainer="devops@gmail.com"

# Add build info
kubectl annotate pod label-demo build-date="2024-01-15" git-sha="abc123"

# Overwrite annotation
kubectl annotate pod label-demo maintainer="new@gmail.com" --overwrite

# Remove annotation
kubectl annotate pod label-demo maintainer-

# View annotations
kubectl describe pod label-demo | grep -A 10 Annotations
```

---

## 4. Pod Spec — Complete YAML Analysis

```yaml
# complete-pod.yml — every field explained
apiVersion: v1               # API version (v1 for core objects like Pod)
kind: Pod                    # Object type
metadata:
  name: full-demo-pod        # Unique name within namespace
  namespace: default         # Which namespace this pod belongs to
  labels:
    app: demo                # Used by selectors (Services, RS, etc.)
    env: production
    version: "1.0"
  annotations:
    description: "Demo pod showing all fields"
    maintainer: "team@company.com"

spec:                        # Desired state of the pod
  # ─── Scheduling ───
  nodeName: node01           # Pin to specific node (bypasses scheduler)
  nodeSelector:              # Schedule only on nodes with this label
    disktype: ssd

  # ─── Init Containers (run before main containers) ───
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

  # ─── Main Containers ───
  containers:
    - name: web                        # Container name (unique in pod)
      image: nginx:1.21                # Docker image
      imagePullPolicy: IfNotPresent    # Always | Never | IfNotPresent

      # Ports
      ports:
        - name: http
          containerPort: 80            # Port container listens on
          protocol: TCP

      # Resource Requests & Limits (fed to cgroup)
      resources:
        requests:                      # Minimum guaranteed (scheduler uses this)
          cpu: "250m"                  # 250 millicores = 0.25 CPU
          memory: "128Mi"
        limits:                        # Maximum allowed (cgroup enforces this)
          cpu: "500m"
          memory: "256Mi"              # If exceeded → OOM Kill!

      # Environment Variables
      env:
        - name: DB_HOST
          value: "postgres-service"
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: secret-key
        - name: CONFIG_VALUE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: config-key

      # Volume Mounts
      volumeMounts:
        - name: data-volume
          mountPath: /data             # Where volume appears inside container
        - name: config-volume
          mountPath: /etc/config
          readOnly: true

      # Probes
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 10
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5

      startupProbe:
        httpGet:
          path: /started
          port: 80
        failureThreshold: 30
        periodSeconds: 10

      # Security Context (container-level)
      securityContext:
        runAsUser: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          add: ["NET_ADMIN"]
          drop: ["ALL"]

      # Lifecycle hooks
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo started > /tmp/started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]

  # ─── Volumes ───
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: my-pvc              # Binds to existing PVC
    - name: config-volume
      configMap:
        name: app-config

  # ─── Restart Policy ───
  restartPolicy: Always                # Always | OnFailure | Never

  # ─── DNS ───
  dnsPolicy: ClusterFirst

  # ─── Service Account ───
  serviceAccountName: my-service-account

  # ─── Pod Security Context ───
  securityContext:
    fsGroup: 2000
    runAsUser: 1000

  # ─── Tolerations ───
  tolerations:
    - key: "node-role"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

  # ─── Affinity ───
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [us-east-1a, us-east-1b]
```

---

## 5. Volumes & Persistent Storage

### Volume Types

| Type | Use Case | Persists? |
|---|---|---|
| `emptyDir` | Scratch space, shared between containers in pod | ❌ Pod lifetime |
| `hostPath` | Access node filesystem | Node-scoped |
| `configMap` | Inject config files | ❌ (read config) |
| `secret` | Inject secrets | ❌ (read secret) |
| `persistentVolumeClaim` | Durable storage | ✅ Survives pod |
| `nfs` | Shared NFS mount | ✅ |
| `awsElasticBlockStore` | AWS EBS | ✅ |
| `gcePersistentDisk` | GCP Persistent Disk | ✅ |

### PV / PVC / StorageClass Chain

```
StorageClass (defines provisioner + params)
    ↓ dynamically provisions
PersistentVolume (actual storage resource)
    ↓ bound to
PersistentVolumeClaim (pod's request for storage)
    ↓ mounted in
Pod
```

```yaml
# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce          # RWO | ROX (ReadOnlyMany) | RWX (ReadWriteMany)
  reclaimPolicy: Retain      # Retain | Recycle | Delete
  storageClassName: standard
  hostPath:
    path: /mnt/data

---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

---

## 6. Probe Types

### Three Probe Types

| Probe | Purpose | Failure Action |
|---|---|---|
| **livenessProbe** | Is container alive? | Restart container |
| **readinessProbe** | Is container ready to serve? | Remove from Service endpoints |
| **startupProbe** | Did slow app finish starting? | Kill + restart if over threshold |

### Three Probe Mechanisms

```yaml
# 1. HTTP GET
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
    - name: X-Health-Check
      value: "true"

# 2. TCP Socket
tcpSocket:
  port: 3306

# 3. Exec Command (exit code 0 = success)
exec:
  command:
    - cat
    - /tmp/healthy
```

### Probe Tuning Parameters

```yaml
initialDelaySeconds: 15   # Wait before first probe
periodSeconds: 10         # How often to probe
timeoutSeconds: 5         # Probe timeout
successThreshold: 1       # Consecutive successes to mark healthy
failureThreshold: 3       # Consecutive failures before action
```

---

## 7. Pod Status & Lifecycle

### Pod Phase

| Phase | Meaning |
|---|---|
| `Pending` | Scheduled but containers not started yet |
| `Running` | At least one container running |
| `Succeeded` | All containers exited with 0 (Jobs) |
| `Failed` | At least one container exited non-zero |
| `Unknown` | Node communication lost |
| `CrashLoopBackOff` | Container keeps crashing; backoff delay |
| `ImagePullBackOff` | Cannot pull image |
| `OOMKilled` | Out of Memory — killed by cgroup |
| `Evicted` | Node pressure eviction |
| `Terminating` | Pod is being deleted |

### Container States

```
Waiting → Running → Terminated
```

---

## 8. Replication Controller (RC)

### What Is RC?

The **original** (legacy) workload controller. Ensures N copies of a pod run at all times.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: web-rc
spec:
  replicas: 3
  selector:              # ← Equality-based ONLY (limitation vs RS)
    app: web
  template:
    metadata:
      labels:
        app: web         # Must match selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
```

### RC vs RS

| Feature | ReplicationController | ReplicaSet |
|---|---|---|
| Selector type | Equality only | Set-based + equality |
| API version | v1 | apps/v1 |
| Used directly? | Rarely (legacy) | Via Deployment |
| `matchExpressions`? | ❌ | ✅ |

---

## 9. ReplicaSet (RS)

### What Is RS?

Modern replacement for RC. Supports **set-based selectors**. Usually managed by a Deployment.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3
  selector:                        # ← Set-based selector
    matchLabels:
      app: web
    matchExpressions:
      - key: env
        operator: In
        values: [production, staging]
  template:
    metadata:
      labels:
        app: web
        env: production
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```

### RS Selector Behavior

```
Selector Check on Creation:
  → If NO matching pods found  → RS creates N new pods
  → If matching pods found     → RS ADOPTS them (ownerReference set)
  → If MORE pods than replicas → RS deletes extras
  → If FEWER pods than replicas → RS creates more
```

> **Key insight:** The selector check happens FIRST. If pods with matching labels already exist (even without ownerReference), the RS will adopt them. This is the "acquire" behavior.

---

## 10. Selectors — matchLabels vs matchExpressions

```yaml
selector:
  matchLabels:                   # Simple equality match
    app: myapp
    
  matchExpressions:              # Advanced set-based
    - key: env
      operator: In               # In | NotIn | Exists | DoesNotExist
      values:
        - production
        - staging
    - key: deprecated
      operator: DoesNotExist     # Key must not exist
```

### Operators

| Operator | Meaning |
|---|---|
| `In` | Label value must be in list |
| `NotIn` | Label value must NOT be in list |
| `Exists` | Key must exist (any value) |
| `DoesNotExist` | Key must not exist |

---

## 11. ownerReference & Scale-Down Algorithm

### ownerReference

Every pod created by a RS/Deployment has an `ownerReference` field:

```yaml
metadata:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: web-rs-abc123
      uid: "12345-abcd-..."
      controller: true
      blockOwnerDeletion: true
```

This links the pod back to its parent. When the RS is deleted, garbage collection deletes all owned pods.

### Scale-Down Pod Deletion Algorithm

When a ReplicaSet scales DOWN, the controller picks which pods to delete using this priority order:

```
1. Pending pods (not yet running) — deleted FIRST
   ↓
2. Pods with lower pod-deletion-cost annotation
   (kubernetes.io/pod-deletion-cost: "10")
   Lower value = deleted first
   ↓
3. Pods on nodes with MORE replicas
   (spreads remaining pods more evenly)
   ↓
4. Pods that were created more recently
   (older pods are more "valuable")
   ↓
5. Random selection among remaining
```

```bash
# Set pod deletion cost (lower = deleted first when scaling down)
kubectl annotate pod mypod kubernetes.io/pod-deletion-cost="100"
```

---

## 12. Homogeneous vs Heterogeneous Pods

### Homogeneous Pods

All pods in a ReplicaSet/Deployment run the **same container image and configuration**.

```
ReplicaSet (app=web, image=nginx:1.21)
  ├── Pod-1 [nginx:1.21]
  ├── Pod-2 [nginx:1.21]
  └── Pod-3 [nginx:1.21]
```

- ✅ Simple scaling
- ✅ Easy load balancing
- ❌ Cannot run different versions simultaneously

**Use case:** Standard microservices, stateless web servers.

### Heterogeneous Pods

Different pods in a group run **different containers or configurations**. Usually managed by separate Deployments or with different labels.

```
Deployment-v1 (version=v1)        Deployment-v2 (version=v2)
  ├── Pod [app:v1]                   └── Pod [app:v2]

Both selected by Service:
  selector: app=web   (matches both)
```

**Use cases:**
- Canary deployments (90% v1, 10% v2)
- Blue-Green deployments
- A/B testing

```yaml
# Canary: 2 separate Deployments, 1 Service selects both
# Service selector: app=frontend
# Deployment-stable: app=frontend, track=stable  (replicas: 9)
# Deployment-canary: app=frontend, track=canary  (replicas: 1)
# Result: ~10% traffic to canary
```

---

## 13. OOM Kill & Ephemeral Containers

### OOM Kill (Out of Memory Kill)

When a container exceeds its memory **limit**, the Linux kernel's **cgroup** OOM killer terminates it.

```
Container memory usage > limits.memory
        ↓
cgroup OOM killer sends SIGKILL
        ↓
Container state: OOMKilled
        ↓
kubelet restarts (if restartPolicy=Always)
        ↓
CrashLoopBackOff (if keeps happening)
```

```bash
# Check OOM kills
kubectl describe pod mypod | grep -i oom
kubectl get pod mypod -o json | jq '.status.containerStatuses[].lastState'

# Pod status shows:
#   reason: OOMKilled
#   exitCode: 137   ← 128 + 9 (SIGKILL)
```

**Prevention:**
- Set realistic `limits.memory` based on profiling
- Add JVM heap flags: `-Xmx256m` for Java apps
- Use Vertical Pod Autoscaler (VPA) for recommendations

### Ephemeral Containers

Temporary containers injected into a **running pod** for debugging. Cannot be restarted.

```bash
# Debug a distroless container (no shell)
kubectl debug -it mypod --image=busybox --target=mycontainer

# Attach ephemeral container with full tools
kubectl debug -it mypod --image=nicolaka/netshoot --target=web
```

```yaml
# Ephemeral container appears in pod spec (read-only after creation)
ephemeralContainers:
  - name: debugger
    image: busybox
    command: ["sh"]
    stdin: true
    tty: true
```

---

## 14. Deployments — Full Deep Dive

### What Is a Deployment?

A Deployment manages a ReplicaSet which manages Pods. It adds **rollout**, **rollback**, and **update strategies** on top of RS.

```
Deployment
  └── ReplicaSet (current)
        ├── Pod-1
        ├── Pod-2
        └── Pod-3
  └── ReplicaSet (old — kept for rollback)
```

### Complete Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  namespace: production
  labels:
    app: web
    version: "1.0"
  annotations:
    kubernetes.io/change-cause: "Initial release v1.0"  # ← Shows in rollout history
spec:
  replicas: 3                        # Desired pod count

  selector:
    matchLabels:
      app: web                       # Must match pod template labels

  # ─── Update Strategy ───
  strategy:
    type: RollingUpdate              # RollingUpdate | Recreate

    rollingUpdate:
      maxUnavailable: 1              # Max pods that can be down during update
                                     # (number or %)
      maxSurge: 1                    # Max extra pods during update
                                     # (number or %)

  # ─── Revision History ───
  revisionHistoryLimit: 10           # How many old RS to keep for rollback

  # ─── Minimum Ready Time ───
  minReadySeconds: 10                # Pod must be Ready for 10s before counted

  # ─── Progress Deadline ───
  progressDeadlineSeconds: 600       # Fail if not progressed in 10 min

  template:                          # Pod template
    metadata:
      labels:
        app: web                     # Must match selector
        version: "1.0"
    spec:
      containers:
        - name: nginx
          image: nginx:1.21

          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
```

### Update Strategies

**RollingUpdate (default):**
```
Old:  [v1][v1][v1]
Step1: [v1][v1][v2]    ← maxSurge=1 adds new pod
Step2: [v1][v2][v2]    ← old pod removed (maxUnavailable=1)
Step3: [v2][v2][v2]    ← complete
```

**Recreate:**
```
Old:  [v1][v1][v1]
Step1: [  ][  ][  ]    ← ALL old pods deleted (DOWNTIME!)
Step2: [v2][v2][v2]    ← all new pods created
```

**Business Decision:**
- `RollingUpdate` → Zero downtime; both versions briefly serve traffic
- `Recreate` → Downtime OK; strict "no two versions at once" requirement

### Scaling

```bash
# Scale up/down
kubectl scale deployment web-deployment --replicas=5

# HPA (Horizontal Pod Autoscaler)
kubectl autoscale deployment web-deployment --min=3 --max=10 --cpu-percent=70

# Update image (triggers rollout)
kubectl set image deployment/web-deployment nginx=nginx:1.22

# Edit directly
kubectl edit deployment web-deployment
```

---

## 15. Rollout & Rollback

### Rollout Commands

```bash
# Check rollout status (live progress)
kubectl rollout status deployment/web-deployment

# View rollout history
kubectl rollout history deployment/web-deployment

# View specific revision details
kubectl rollout history deployment/web-deployment --revision=2

# Pause rollout (canary-style manual control)
kubectl rollout pause deployment/web-deployment

# Resume paused rollout
kubectl rollout resume deployment/web-deployment
```

### Rollback Commands

```bash
# Rollback to previous version
kubectl rollout undo deployment/web-deployment

# Rollback to specific revision
kubectl rollout undo deployment/web-deployment --to-revision=1

# Rollback and watch
kubectl rollout undo deployment/web-deployment --to-revision=2 && \
kubectl rollout status deployment/web-deployment
```

### Rollout History with change-cause

```yaml
# Add annotation to track reason in history
metadata:
  annotations:
    kubernetes.io/change-cause: "Upgrade nginx to 1.22 for security patch"
```

```bash
# History shows:
# REVISION  CHANGE-CAUSE
# 1         Initial release v1.0
# 2         Upgrade nginx to 1.22 for security patch
# 3         Rollback: revert to v1.0 due to latency regression
```

---

## 16. Taints & Tolerations

### Taints (Applied to Nodes)

Taints **repel** pods. Only pods with matching tolerations can schedule on tainted nodes.

```bash
# Add taint
kubectl taint nodes node01 key=value:effect

# Effects:
#   NoSchedule       → New pods without toleration won't schedule
#   PreferNoSchedule → Soft: try to avoid, not guaranteed
#   NoExecute        → Evict existing pods WITHOUT toleration

# Examples:
kubectl taint nodes node01 gpu=true:NoSchedule
kubectl taint nodes node02 maintenance=true:NoExecute

# Remove taint
kubectl taint nodes node01 gpu=true:NoSchedule-
```

### Tolerations (Applied to Pods)

```yaml
spec:
  tolerations:
    # Match the gpu taint
    - key: "gpu"
      operator: "Equal"           # Equal | Exists
      value: "true"
      effect: "NoSchedule"

    # Tolerate ANY taint with effect NoExecute
    - operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300      # Evict after 300s if node problem

    # Tolerate ALL taints (dangerous — use carefully)
    - operator: "Exists"
```

### Use Cases

| Taint | Purpose |
|---|---|
| `dedicated=gpu:NoSchedule` | Reserve GPU nodes for ML workloads |
| `maintenance=true:NoExecute` | Drain node for maintenance |
| `node.kubernetes.io/not-ready:NoExecute` | Auto-applied when node fails |
| `spot-instance=true:PreferNoSchedule` | Soft preference against spot nodes |

---

## 17. Pod Priority & Privilege

### privileged: true

Gives container root-level access to the host. **Dangerous — avoid in production.**

```yaml
securityContext:
  privileged: true    # Full host access — use only for system daemons like CNI, CSI
```

### Pod Security Context vs Container Security Context

```yaml
spec:
  securityContext:             # Pod-level (applies to all containers)
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true

  containers:
    - securityContext:         # Container-level (overrides pod-level)
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]
```

### Accessing a Private Registry Image

```bash
# Create docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=email@example.com
```

```yaml
spec:
  imagePullSecrets:
    - name: regcred          # Reference the secret
  containers:
    - image: registry.example.com/myapp:v1.0
```

---

## 18. SLO · SLA · SLI & Error Budget

### Definitions

| Term | Definition | Example |
|---|---|---|
| **SLI** (Service Level Indicator) | A quantitative measure of service performance | Request success rate: 99.5% |
| **SLO** (Service Level Objective) | The target for an SLI | Success rate ≥ 99.9% |
| **SLA** (Service Level Agreement) | Legal/contractual commitment to customers | 99.9% uptime or credit issued |
| **Error Budget** | How much failure is allowed | 100% - SLO = 0.1% = 43.8 min/month |

### Relationship

```
SLA  ← external promise to customers (legal)
 └── SLO  ← internal team target (slightly stricter)
      └── SLI  ← actual measurement from metrics
           └── Error Budget = 1 - SLO
```

### Error Budget in Practice

```
Monthly Error Budget (99.9% SLO):
  Total minutes/month: 43,200
  Allowed downtime: 43.2 minutes/month

If 30 min already used:
  Remaining budget: 13.2 minutes
  → Slow down risky deployments!
  → Focus on reliability improvements

If budget exhausted:
  → Feature freeze until next month
  → Only reliability work allowed
```

### Application Architecture Connection

```
Business SLA (99.9%)
  ↓
SLO per service (99.95%) ← stricter internal target
  ↓
SLIs measured:
  - Request latency p99 < 200ms
  - Error rate < 0.05%
  - Availability > 99.95%
  ↓
Error Budget monitored:
  - Budget used → slow rollouts
  - Budget healthy → feature releases OK
```

### K8s Alignment

```yaml
# Deployment supports SLO via:
spec:
  strategy:
    rollingUpdate:
      maxUnavailable: 0    # ← Zero downtime supports high availability SLO
  minReadySeconds: 30      # ← Ensures stable pods before traffic
  progressDeadlineSeconds: 300

# PodDisruptionBudget — protects SLO during node drain
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2          # Always keep 2 pods available
  selector:
    matchLabels:
      app: web
```

---

## 19. kubectl Cheat Sheet

```bash
# ═══ Labels ═══
kubectl apply -f labels-pod.yml
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get pods -l app=web,env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l run!=podname
kubectl label pod mypod version=v2
kubectl label pod mypod version=v3 --overwrite
kubectl label pod mypod version-

# ═══ Annotations ═══
kubectl annotate pod mypod maintainer="gmail@gmail.com"
kubectl annotate pod mypod note="test" --overwrite

# ═══ Pods ═══
kubectl get pods -o wide
kubectl describe pod mypod
kubectl logs mypod -c container-name --previous
kubectl exec -it mypod -- /bin/bash
kubectl delete pod mypod
kubectl debug -it mypod --image=busybox --target=main

# ═══ ReplicaSet ═══
kubectl get rs
kubectl describe rs web-rs
kubectl scale rs web-rs --replicas=5

# ═══ Deployment ═══
kubectl apply -f deployment.yml
kubectl get deploy
kubectl set image deployment/web-deployment nginx=nginx:1.22
kubectl scale deployment web-deployment --replicas=5
kubectl edit deployment web-deployment

# ═══ Rollout ═══
kubectl rollout status deployment/web-deployment
kubectl rollout history deployment/web-deployment
kubectl rollout history deployment/web-deployment --revision=2
kubectl rollout pause deployment/web-deployment
kubectl rollout resume deployment/web-deployment
kubectl rollout undo deployment/web-deployment
kubectl rollout undo deployment/web-deployment --to-revision=1

# ═══ Namespace ═══
kubectl get pods -n kube-system
kubectl create namespace staging
kubectl config set-context --current --namespace=staging

# ═══ Node ═══
kubectl get nodes
kubectl describe node node01
kubectl label node node01 disktype=ssd
kubectl taint nodes node01 key=val:NoSchedule
kubectl cordon node01      # Stop new pods from scheduling
kubectl drain node01       # Evict pods (for maintenance)
kubectl uncordon node01    # Allow scheduling again

# ═══ Misc ═══
kubectl top pods           # CPU/memory (needs metrics-server)
kubectl top nodes
kubectl get events --sort-by=.lastTimestamp
kubectl explain pod.spec.containers
```

---

## 20. Interview Q&A

### Labels

**Q: What is a label in Kubernetes?**
A: A label is a key-value pair attached to Kubernetes objects. They are used for identification, grouping, and selection. Keys must be unique per object. Labels enable selectors in Services, ReplicaSets, and Deployments to target specific pods.

**Q: What happens if you don't specify labels when running a pod?**
A: Kubernetes auto-applies `run=<pod-name>` as the label when using `kubectl run`.

**Q: Can you have two labels with the same key in one pod?**
A: No. Each key must be unique in the labels map. You can have multiple different key-value pairs but duplicate keys are not allowed.

**Q: What are label use cases?**
A: Environment separation (prod/staging/dev), version tracking, canary deployments, blue-green deployments, cost attribution by team/app, node affinity targeting, Service traffic routing, and ReplicaSet pod selection.

**Q: Difference between labels and annotations?**
A: Labels are used for selection and grouping; they're queryable via `-l`. Annotations store metadata for tooling/humans only; they're not queryable, support larger values, and are never used for selection.

### ReplicaSet & Selector

**Q: How does a ReplicaSet selector work?**
A: When a RS is created, it checks for existing pods matching its selector. If found, it adopts them (sets ownerReference). If fewer than desired, it creates new ones. If more exist, it deletes extras. The selector is immutable after creation.

**Q: What is ownerReference?**
A: A field in pod metadata that links a pod to its controlling resource (RS, Deployment). Enables garbage collection — when the owner is deleted, owned pods are deleted. Also used by the RS controller to track which pods it manages.

**Q: How does the RS scale-down algorithm work?**
A: It prioritizes deleting: (1) Pending pods, (2) pods with lower `pod-deletion-cost` annotation, (3) pods on nodes with more replicas of the same RS, (4) more recently created pods. This ensures even distribution and preserves stable, older pods.

### Deployments

**Q: What is the difference between RC, RS, and Deployment?**
A: RC (legacy) supports equality selectors only. RS supports set-based selectors and is the modern replacement. Deployment wraps RS and adds rolling updates, rollback, and history. You should always use Deployments in production.

**Q: What's the difference between RollingUpdate and Recreate strategies?**
A: RollingUpdate gradually replaces old pods with new ones using maxUnavailable/maxSurge controls — zero downtime but two versions run briefly. Recreate kills all old pods first (causing downtime) then starts new ones — ensures only one version runs at a time.

**Q: What are maxUnavailable and maxSurge?**
A: `maxUnavailable`: max pods that can be unavailable during update. `maxSurge`: max extra pods allowed above desired during update. Both can be numbers or percentages. Setting `maxUnavailable=0, maxSurge=1` ensures zero downtime.

**Q: How do you do a rollback?**
A: `kubectl rollout undo deployment/name` for previous version, or `kubectl rollout undo deployment/name --to-revision=N` for specific revision. Add `kubernetes.io/change-cause` annotation to track why changes were made.

### OOM & Resources

**Q: What is OOM Kill?**
A: When a container's memory usage exceeds its `limits.memory`, the Linux kernel cgroup OOM killer sends SIGKILL (exit code 137). The container restarts (if `restartPolicy=Always`). Repeated OOM kills lead to `CrashLoopBackOff`.

**Q: Difference between resource requests and limits?**
A: Requests are the guaranteed minimum — the scheduler uses them to find a suitable node. Limits are the maximum — cgroup enforces them. CPU throttles on limit; Memory OOM-kills on limit.

### Probes

**Q: What are the three probe types and when would each fire?**
A: Liveness: container is alive; failure → restart. Readiness: container can serve traffic; failure → remove from Service endpoints (no restart). Startup: for slow-starting apps; replaces liveness until startup succeeds; failure → restart.

### Taints & Tolerations

**Q: What's the difference between a taint and a toleration?**
A: A taint is applied to a **node** and repels pods. A toleration is applied to a **pod** and allows it to schedule on a tainted node. Together they control which pods can run where.

**Q: What are the taint effects?**
A: `NoSchedule` — new pods without toleration won't be placed. `PreferNoSchedule` — soft preference to avoid. `NoExecute` — evicts existing pods without toleration (can use `tolerationSeconds` for grace period).

### SLO/SLA/SLI

**Q: What is an error budget?**
A: Error budget = 1 - SLO. For a 99.9% SLO, you have 0.1% allowed errors (~43 min/month). It governs deployment velocity — when budget is consumed, risky changes stop and reliability work takes priority.

**Q: How does Kubernetes support SLOs?**
A: Via PodDisruptionBudgets (ensure minimum availability during disruptions), `maxUnavailable=0` deployments (zero downtime updates), readiness probes (keep unhealthy pods out of load balancer), and HPA (scale to meet demand before latency degrades).

---

## Quick Reference Card

```
Cluster → Node → Namespace → Deployment → ReplicaSet → Pod → Container → cgroup

Labels     → key=value, used for SELECTION, queryable with -l
Annotations→ key=value, NOT for selection, metadata for tools

RC  → Legacy, equality selector only
RS  → Set-based selector, modern
Deploy → Wraps RS + rolling updates + rollback

RollingUpdate: zero downtime, two versions briefly coexist
Recreate: downtime, one version at a time

OOM Kill → container exceeds limits.memory → SIGKILL → exit 137

Liveness  → restart if dead
Readiness → remove from endpoints if not ready
Startup   → wait for slow start before liveness kicks in

Taint  → repel pods from node
Toleration → pod can schedule on tainted node

SLI → measurement   (99.5% success rate)
SLO → target        (≥99.9%)
SLA → contract      (99.9% or credit)
Error Budget → 1 - SLO → gates release velocity
```

---

*Generated for deep K8s interview prep — covers Labels, Selectors, Pods, RS, Deployments, Rollout/Rollback, OOM, Taints, Probes, Volumes, SLO/SLA/SLI*
