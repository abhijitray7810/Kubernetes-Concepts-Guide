# 🚀 Kubernetes (K8s) — Deep Dive Knowledge Base

> **One-line definition:** Kubernetes is an open-source **container orchestration platform** that automates deployment, scaling, networking, and lifecycle management of containerized applications across a cluster of machines.

---

## Table of Contents

1. [What is Kubernetes? Orchestration Tools Landscape](#1-what-is-kubernetes) 
2. [Communication Protocols — gRPC Deep Dive](#2-grpc--kubernetes-communication)
3. [K8s Architecture — Full Deep Dive](#3-kubernetes-architecture)
   - 3.1 [Cluster](#31-cluster)
   - 3.2 [Master Node (Control Plane)](#32-master-node--control-plane)
     - 3.2.1 [Cloud Controller Manager](#321-cloud-controller-manager)
     - 3.2.2 [kube-apiserver](#322-kube-apiserver)
     - 3.2.3 [etcd](#323-etcd)
     - 3.2.4 [kube-scheduler](#324-kube-scheduler)
     - 3.2.5 [kube-controller-manager](#325-kube-controller-manager)
   - 3.3 [Worker Node](#33-worker-node)
4. [Container Runtime Deep Dive — CRI → OCI → runc → cgroups](#4-container-runtime-deep-dive)
5. [Pod Lifecycle Flow — API Server → Container Running](#5-pod-lifecycle-flow)
6. [kubelet — The Node Agent](#6-kubelet--the-node-agent)
7. [Key Metrics Reference](#7-key-metrics-reference)
8. [Networking Communication — Q&A](#8-networking-communication--qa)
9. [kube-proxy & iptables](#9-kube-proxy--iptables)
10. [Component Failure Impact Analysis](#10-component-failure-impact-analysis)

---

## 1. What is Kubernetes?

### Definition
Kubernetes (K8s) is **not** a single tool — it is a **set of controllers** working together to manage the desired state of containerized workloads. Every piece of K8s is a control loop: *observe → compare → act*.

### K8s vs Similar Orchestration Tools

| Tool | Description | When to Use |
|------|-------------|-------------|
| **Kubernetes (K8s)** | Full-featured container orchestration, self-healing, scaling, networking | Production-grade, complex microservices |
| **Docker Swarm** | Simple built-in Docker clustering, smaller feature set | Small teams, simpler deployments |
| **Apache Mesos** | Datacenter OS, manages both containers and non-container workloads | Large-scale mixed workloads |
| **Nomad (HashiCorp)** | Lightweight orchestrator for containers, VMs, binaries | Multi-runtime workloads |
| **ECS (AWS)** | AWS-native managed container service | AWS-only, simpler ops burden |
| **OpenShift** | Enterprise K8s with added developer tools (Red Hat) | Enterprise teams needing built-in CI/CD |

### Why K8s is a "Set of Controllers"
Every major K8s component is a **reconciliation loop**:

```
Desired State (etcd)  →  Observe Current State  →  Act to Reconcile
        ↑                                                   ↓
        └───────────────────────────────────────────────────┘
```

- `ReplicaSet Controller` — ensures N pod replicas always run
- `Deployment Controller` — manages rolling updates
- `Node Controller` — detects node failures
- `Service Controller` — provisions cloud load balancers
- `Job Controller` — ensures batch jobs complete

> **Key insight:** K8s does not "run" applications. It **declares** desired state and continuously works to match reality to that declaration.

---

## 2. gRPC & Kubernetes Communication

### What is gRPC?

**Full Form:** `gRPC Remote Procedure Call` *(the "g" stands for Google, original creator)*

**One-line definition:** gRPC is a high-performance, open-source **Remote Procedure Call (RPC) framework** that uses HTTP/2 for transport and Protocol Buffers (protobuf) for serialization.

### Why Does K8s Need gRPC?

Traditional REST over HTTP/1.1 has limitations in a distributed system:

| Problem with REST/HTTP1.1 | How gRPC Solves It |
|---------------------------|-------------------|
| Text-based JSON (verbose, slow to parse) | Binary protobuf (10x smaller, faster) |
| One request per TCP connection | HTTP/2 multiplexing — many streams on one connection |
| No native streaming | Bidirectional streaming built-in |
| No strong typing | Strongly typed `.proto` schema contracts |
| High latency for frequent small calls | Persistent connections, lower overhead |

### Where K8s Uses gRPC

```
kubelet  ←──── gRPC ────→  CRI (containerd, CRI-O)
kubelet  ←──── gRPC ────→  CNI plugins (network setup)
kubelet  ←──── gRPC ────→  CSI plugins (storage)
kube-apiserver  ←── gRPC ──→  etcd
```

### How gRPC Works (Step by Step)

**Step 1 — Define a `.proto` contract:**
```protobuf
service RuntimeService {
  rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);
  rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
}
```

**Step 2 — Generate client/server code** (Go, Java, Python, etc.) from `.proto`

**Step 3 — Call remote function as if it were local:**
```go
// kubelet calls containerd as if it's a local function
response, err := runtimeClient.CreateContainer(ctx, &CreateContainerRequest{...})
```

**Step 4 — gRPC serializes via protobuf, sends over HTTP/2, deserializes on the other side**

### gRPC Streaming Types Used in K8s

| Type | Description | K8s Use Case |
|------|-------------|--------------|
| Unary | Single request → single response | Create Pod, Delete Pod |
| Server streaming | Single request → stream of responses | `kubectl logs -f` |
| Client streaming | Stream of requests → single response | Metrics aggregation |
| Bidirectional | Full duplex streams | `kubectl exec`, `kubectl attach` |

### Protocol Summary in K8s

| Communication Path | Protocol |
|-------------------|----------|
| `kubectl` → `kube-apiserver` | HTTPS / REST |
| `kube-apiserver` → `etcd` | gRPC |
| `kubelet` → `kube-apiserver` | HTTPS / REST + WebSocket |
| `kubelet` → `containerd` (CRI) | gRPC (Unix socket) |
| `kubelet` → CNI plugins | gRPC |
| `kube-controller-manager` → `kube-apiserver` | HTTPS / REST |
| Pod → Pod (same node) | Direct via virtual ethernet (veth) |
| Pod → Pod (different node) | Encapsulated via CNI overlay (VXLAN, etc.) |

---

## 3. Kubernetes Architecture

### Bird's Eye View

```
┌──────────────────────────────────────────────────────────────────┐
│                         K8s CLUSTER                              │
│                                                                  │
│  ┌─────────────────────────────┐   ┌────────────────────────┐   │
│  │      MASTER NODE            │   │     WORKER NODE(s)     │   │
│  │   (Control Plane)           │   │                        │   │
│  │                             │   │  ┌──────────────────┐  │   │
│  │  ┌──────────────────────┐   │   │  │      POD         │  │   │
│  │  │  kube-apiserver      │   │   │  │  ┌────────────┐  │  │   │
│  │  ├──────────────────────┤   │   │  │  │ Container1 │  │  │   │
│  │  │  etcd                │   │   │  │  ├────────────┤  │  │   │
│  │  ├──────────────────────┤   │   │  │  │ Container2 │  │  │   │
│  │  │  kube-scheduler      │   │   │  │  └────────────┘  │  │   │
│  │  ├──────────────────────┤   │   │  └──────────────────┘  │   │
│  │  │  controller-manager  │   │   │  kubelet               │   │
│  │  ├──────────────────────┤   │   │  kube-proxy            │   │
│  │  │  cloud-controller    │   │   └────────────────────────┘   │
│  │  └──────────────────────┘   │                                │
│  └─────────────────────────────┘                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3.1 Cluster

**One-line definition:** A cluster is the **entire Kubernetes environment** — a collection of one master node + multiple worker nodes managed as a single unit.

### Q: What is a K8s Cluster?
**A:** A cluster is the top-level logical boundary in Kubernetes. It contains:
- **Control Plane** (master) — the brain making decisions
- **Worker Nodes** — the muscle running workloads
- **Shared networking** — a flat network where every Pod can reach every other Pod
- **Shared etcd** — the single source of truth for all state

### Q: What happens if the cluster doesn't exist?
**A:** There is nothing to orchestrate. Every concept in K8s — Pods, Services, Deployments — exists only within a cluster boundary.

### Q: Can there be multiple clusters?
**A:** Yes. Organizations run multiple clusters for:
- Environment isolation (dev / staging / prod)
- Geographic distribution (us-east, eu-west)
- Regulatory compliance (PCI, HIPAA data isolation)
- Multi-cloud resilience

---

## 3.2 Master Node / Control Plane

**One-line definition:** The master node is the **brain of the cluster** — it makes all scheduling decisions, maintains desired state, and exposes the API.

### Q: What does the master node do?
**A:** It runs four core processes: `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager`. Together they:
1. Accept user intent (via `kubectl` or API)
2. Store that intent (etcd)
3. Decide where to run workloads (scheduler)
4. Continuously reconcile actual ↔ desired state (controllers)

### Q: What happens if the master node goes down?
**A:** Running applications on worker nodes **continue running** — pods already scheduled keep working. However:
- No new pods can be scheduled
- No scaling or self-healing occurs
- No deployments or changes can be made
- etcd data may be at risk (if single master)

> **Best practice:** Run 3 or 5 master nodes for high availability (odd number for etcd quorum).

---

## 3.2.1 Cloud Controller Manager

**One-line definition:** Integrates K8s with **cloud provider APIs** (AWS, GCP, Azure) to provision cloud-native resources like load balancers, storage volumes, and node lifecycle.

### What does it do?
It runs cloud-specific control loops:

| Controller | What it does |
|-----------|-------------|
| **Node controller** | Detects when a cloud VM is deleted, removes that node from K8s |
| **Route controller** | Configures routes in cloud VPC for pod CIDR blocks |
| **Service controller** | Creates/updates/deletes cloud load balancers for `type: LoadBalancer` Services |

### Q: Why is it separate from kube-controller-manager?
**A:** To keep core K8s cloud-agnostic. Cloud-specific logic is isolated so the same K8s core works on AWS, GCP, Azure, or bare metal. Cloud providers implement and ship their own `cloud-controller-manager`.

### Q: What if cloud-controller-manager is missing?
**A:** Services of type `LoadBalancer` stay in `<pending>` state forever — the external IP is never provisioned. Node objects may not be cleaned up when cloud VMs are terminated.

---

## 3.2.2 kube-apiserver

**One-line definition:** The **single entry point** for all K8s operations — it validates, authenticates, authorizes, and persists every API request to etcd.

### What does it actually do?

```
User / kubectl
     │
     ▼
kube-apiserver
     │
     ├── 1. Authentication (who are you? — certs, tokens, OIDC)
     ├── 2. Authorization (can you do this? — RBAC)
     ├── 3. Admission Control (should this be allowed? — webhooks, policies)
     ├── 4. Validation (is the object schema correct?)
     └── 5. Persist to etcd → Notify watchers (scheduler, controllers, kubelet)
```

### Admission Controller Deep Dive

The API server runs a pipeline of **admission controllers** before writing to etcd:

```
Request → AuthN → AuthZ → [MutatingAdmissionWebhooks] → [ValidatingAdmissionWebhooks] → etcd
```

| Admission Controller | Purpose |
|---------------------|---------|
| `NamespaceLifecycle` | Rejects resources in terminating namespaces |
| `LimitRanger` | Enforces resource limits on pods |
| `ResourceQuota` | Enforces namespace-level resource quotas |
| `PodSecurity` | Enforces pod security standards |
| `MutatingWebhookConfiguration` | Custom mutation (inject sidecars, set defaults) |
| `ValidatingWebhookConfiguration` | Custom validation (policy enforcement) |

### Q: What if kube-apiserver goes down?
**A:** The entire cluster management plane stops. No `kubectl` commands work. Existing workloads continue (kubelet runs independently), but no new scheduling, no self-healing, no config changes possible.

### Q: What protocol does kube-apiserver use?
**A:** It exposes a **RESTful HTTPS API** on port `6443`. Internally it talks to etcd via **gRPC**.

---

## 3.2.3 etcd

**One-line definition:** A distributed, strongly consistent **key-value store** that is the single source of truth for all cluster state in Kubernetes.

### What does etcd store?
Everything. Literally every K8s object:

```
/registry/pods/default/my-pod
/registry/services/default/my-service
/registry/deployments/default/my-deployment
/registry/nodes/node-1
/registry/secrets/default/my-secret
/registry/configmaps/default/my-config
```

### How etcd Achieves Consistency — Raft Consensus

etcd uses the **Raft algorithm** to elect a leader and replicate data:

```
Client Write Request
       │
       ▼
  etcd Leader ──── replicates log ────► etcd Follower 1
       │                                etcd Follower 2
       ▼
 Commit only after majority (quorum) ACK
```

- **3 nodes** → tolerates 1 failure
- **5 nodes** → tolerates 2 failures
- **Quorum = (n/2) + 1**

### Q: Why does K8s use etcd instead of a regular database?
**A:** etcd provides:
1. **Watch API** — components (scheduler, controllers, kubelet) *watch* keys and get notified instantly on change — no polling needed
2. **Consistency guarantee** — every read sees the latest committed write
3. **Leader election** — etcd itself is used by K8s controllers to elect a single active leader

### Q: What if etcd goes down?
**A:** The cluster state is frozen. The API server cannot read or write any object. New deployments, scaling, and healing all stop. Running pods continue (kubelet has local state), but no cluster-level changes are possible. **This is why etcd backup is critical.**

### Q: How do controllers get notified of changes?
**A:** Via the **Watch mechanism**. Components register a watch on specific key prefixes:

```go
// kube-scheduler watches for unscheduled pods
watcher := etcdClient.Watch(ctx, "/registry/pods/", clientv3.WithPrefix())
for event := range watcher {
    if event.Type == PUT {
        pod := deserialize(event.Value)
        if pod.Spec.NodeName == "" {
            scheduleQueue.Add(pod)
        }
    }
}
```

---

## 3.2.4 kube-scheduler

**One-line definition:** Watches for **unscheduled pods** and assigns them to the best-fit worker node based on resource availability, constraints, and policies.

### Scheduling Process (Step by Step)

```
New Pod (NodeName = "")
        │
        ▼
  1. FILTERING (Predicates)
     - Node has enough CPU/Memory?
     - NodeSelector / nodeAffinity match?
     - Taints and Tolerations match?
     - Port conflicts?
     - Volume zone constraints?
        │
        ▼
  2. SCORING (Priorities)
     - LeastRequestedPriority (prefer less loaded nodes)
     - BalancedResourceAllocation (balance CPU + memory)
     - NodeAffinityPriority (prefer matching affinity)
     - InterPodAffinityPriority (spread/co-locate pods)
        │
        ▼
  3. BINDING
     - Highest scoring node selected
     - Pod.Spec.NodeName = "selected-node"
     - Written to etcd via kube-apiserver
     - kubelet on that node picks it up and starts the pod
```

### Q: What if kube-scheduler goes down?
**A:** New pods stay in `Pending` state indefinitely with the message `no nodes available to schedule pods`. Existing running pods are **unaffected**. Multi-master setups run standby schedulers via leader election.

### Q: Can I write a custom scheduler?
**A:** Yes. K8s supports multiple schedulers. Pods can specify `spec.schedulerName: my-custom-scheduler` to use a different one.

---

## 3.2.5 kube-controller-manager

**One-line definition:** Runs all the **built-in control loops** (controllers) as a single binary — each controller watches cluster state and reconciles actual state toward desired state.

### Built-in Controllers

| Controller | What It Watches | What It Does |
|-----------|----------------|-------------|
| **ReplicaSet** | ReplicaSet objects | Ensures correct number of pod replicas |
| **Deployment** | Deployment objects | Manages rollouts, rollbacks |
| **StatefulSet** | StatefulSet objects | Manages ordered, stateful pods |
| **DaemonSet** | DaemonSet objects | Ensures one pod per node |
| **Job** | Job objects | Ensures batch pods complete |
| **CronJob** | CronJob objects | Schedules Jobs on a cron schedule |
| **Node** | Node objects | Detects node failures, evicts pods |
| **Endpoints** | Service + Pod objects | Populates Endpoints for Services |
| **Namespace** | Namespace objects | Manages namespace lifecycle |
| **ServiceAccount** | Namespace objects | Creates default ServiceAccounts |
| **PersistentVolume** | PV/PVC objects | Binds PVCs to PVs |

### The Control Loop Pattern (Universal Pattern)

```go
for {
    desiredState := getFromEtcd()        // What should be?
    actualState  := observeCluster()     // What is?
    
    if desiredState != actualState {
        reconcile(desiredState, actualState) // Make it so
    }
    
    sleep(resyncPeriod)
}
```

### Q: What if kube-controller-manager goes down?
**A:** Cluster loses **self-healing**. If a pod crashes, it is not restarted. If you scale a Deployment, nothing happens. Running pods are unaffected — the damage shows when something goes wrong and no one fixes it.

---

## 3.3 Worker Node

**One-line definition:** A worker node is a **machine (VM or physical)** that runs the actual application workloads (pods/containers), managed by the control plane.

### Worker Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Node agent — ensures containers run as specified |
| **kube-proxy** | Network proxy — manages iptables/IPVS rules for Services |
| **Container Runtime** | Runs containers (containerd, CRI-O) |
| **Pods** | The actual application workloads |

---

## 4. Container Runtime Deep Dive

### The Full Stack: K8s → CRI → OCI → runc → cgroups → Container

```
K8s kubelet
     │  gRPC (CRI interface)
     ▼
containerd  (CRI implementation)
     │  OCI spec (config.json)
     ▼
runc  (OCI runtime — written in Go, calls C)
     │  Linux syscalls
     ▼
Linux Kernel
  ├── cgroups (resource limits: CPU, memory, I/O)
  ├── namespaces (isolation: PID, NET, MNT, UTS, IPC, USER)
  └── overlay filesystem (container layers)
     │
     ▼
CONTAINER RUNNING ✓
```

### CRI — Container Runtime Interface

**One-line definition:** CRI is the **gRPC API specification** that defines how kubelet communicates with any container runtime.

**Why CRI exists:** Originally K8s only supported Docker (hardcoded). As more runtimes emerged, a standard interface was needed so any runtime could plug in without changing K8s core.

**CRI defines two services:**
```protobuf
service RuntimeService {
  // Pod-level
  rpc RunPodSandbox(...)          // Create pod network namespace + sandbox
  rpc StopPodSandbox(...)         // Stop sandbox
  rpc RemovePodSandbox(...)       // Delete sandbox
  
  // Container-level
  rpc CreateContainer(...)        // Create container inside sandbox
  rpc StartContainer(...)         // Start container
  rpc StopContainer(...)          // Stop container
  rpc RemoveContainer(...)        // Delete container
  rpc ExecSync(...)               // Execute command in container
}

service ImageService {
  rpc PullImage(...)              // Pull image from registry
  rpc ListImages(...)             // List local images
  rpc RemoveImage(...)            // Delete local image
}
```

### CRI Implementations

| Runtime | Description | Status |
|---------|-------------|--------|
| **containerd** | Industry standard, CNCF graduated, used by Docker internally | ✅ Default |
| **CRI-O** | Lightweight CRI for K8s only (RedHat), no extra features | ✅ Used in OpenShift |
| **rkt** (Rocket) | CoreOS runtime, now deprecated | ❌ Deprecated |
| **Docker** (via dockershim) | Original runtime, shim removed in K8s 1.24 | ❌ Removed |

### OCI — Open Container Initiative

**One-line definition:** OCI is a set of **open industry standards** defining what a container image looks like and how a container is run — ensuring portability across all runtimes.

**Why OCI exists:** Before OCI (2015), Docker was the only standard. OCI (created by Docker, CoreOS, Google, etc.) standardized:
1. **Image Spec** — how container images are stored and layered (tar layers + JSON manifests)
2. **Runtime Spec** — what parameters define a running container (`config.json`)
3. **Distribution Spec** — how images are pushed/pulled from registries

**OCI `config.json` example (what runc receives):**
```json
{
  "ociVersion": "1.0.0",
  "process": {
    "args": ["/bin/myapp"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin"],
    "user": { "uid": 1000, "gid": 1000 }
  },
  "root": { "path": "rootfs", "readonly": false },
  "linux": {
    "namespaces": [
      {"type": "pid"}, {"type": "network"}, {"type": "mount"},
      {"type": "ipc"}, {"type": "uts"}
    ],
    "cgroupsPath": "/kubepods/besteffort/pod-uid/container-id",
    "resources": {
      "cpu": { "quota": 100000, "period": 100000 },
      "memory": { "limit": 536870912 }
    }
  }
}
```

### runc — The OCI Runtime

**One-line definition:** `runc` is the **reference OCI runtime** — a Go CLI tool that reads `config.json` and uses Linux kernel features to create an isolated container process.

**runc internals:**
```
runc create <container-id>
     │
     ├── Written in Go (uses os/exec, syscall packages)
     ├── Calls clone() syscall with namespace flags
     ├── Sets up cgroups via cgroupfs or systemd
     ├── Mounts overlay filesystem (container root)
     ├── Applies seccomp, AppArmor, SELinux
     └── Executes container entrypoint via exec()
```

**runc → C relationship:** runc is written in Go, but critically it **calls C code via CGo** for certain low-level operations (libcontainer). Some parts use pure Linux kernel syscalls directly from Go via the `syscall` and `golang.org/x/sys/unix` packages.

### cgroups — Control Groups

**One-line definition:** cgroups is a **Linux kernel feature** that limits, accounts for, and isolates resource usage (CPU, memory, disk I/O, network) of process groups — enforcing container resource constraints.

### How cgroups Work for Containers

```
Pod spec:
  resources:
    requests:
      cpu: "250m"       # 0.25 CPU cores reserved
      memory: "128Mi"   # 128MB memory reserved
    limits:
      cpu: "500m"       # Max 0.5 CPU cores
      memory: "256Mi"   # Max 256MB memory — OOMKill if exceeded
```

kubelet translates this to cgroup configuration:

```
/sys/fs/cgroup/
└── kubepods/
    └── burstable/
        └── pod-<uid>/
            ├── cpu.cfs_quota_us    = 50000   (500m = 50% of 100ms period)
            ├── cpu.cfs_period_us   = 100000
            ├── memory.limit_in_bytes = 268435456  (256Mi)
            └── <container-id>/
                ├── cpu.cfs_quota_us
                └── memory.limit_in_bytes
```

**What cgroups isolate:**

| Resource | cgroup Subsystem | What It Controls |
|----------|-----------------|-----------------|
| CPU time | `cpu`, `cpuacct` | CPU scheduling share and limits |
| Memory | `memory` | RAM limits, OOM killer behavior |
| Disk I/O | `blkio` | Read/write bandwidth, IOPS |
| Network | `net_cls`, `net_prio` | Traffic classification |
| PIDs | `pids` | Maximum number of processes |

### Linux Namespaces — Container Isolation

| Namespace | Isolates | Container Effect |
|-----------|----------|-----------------|
| `PID` | Process IDs | Container sees only its own PIDs (PID 1 = entrypoint) |
| `NET` | Network stack | Container gets its own network interfaces, IP |
| `MNT` | Mount points | Container sees its own filesystem |
| `UTS` | Hostname | Container has its own hostname |
| `IPC` | SysV IPC, POSIX MQ | Container has isolated IPC resources |
| `USER` | UIDs/GIDs | Container user ≠ host user (rootless containers) |

---

## 5. Pod Lifecycle Flow

### Full Flow: API Server → Container Running

```
kubectl apply -f pod.yaml
        │
        ▼ HTTPS/REST
[kube-apiserver]
        │
        ├── 1. Authentication (client cert / token)
        ├── 2. Authorization (RBAC: can this user create pods?)
        ├── 3. Admission Controllers
        │       ├── MutatingWebhook (inject sidecar? set defaults?)
        │       └── ValidatingWebhook (policy checks)
        ├── 4. Schema validation
        └── 5. Write pod object to etcd (status: Pending)
        │
        ▼ Watch notification
[kube-scheduler]
        │
        ├── Detects pod with NodeName = ""
        ├── Filter nodes (resource fit, affinity, taints)
        ├── Score nodes (least loaded, best fit)
        └── Bind pod: NodeName = "worker-node-2" → write to etcd
        │
        ▼ Watch notification
[kubelet on worker-node-2]
        │
        ├── Detects pod assigned to this node
        ├── Calls CRI: RunPodSandbox (creates pause container + network namespace)
        │       │ gRPC
        │       ▼
        │   [containerd]
        │       ├── Creates Linux network namespace
        │       ├── Calls CNI plugin to assign Pod IP (e.g., 10.244.1.5)
        │       └── Starts pause container (holds namespace alive)
        │
        ├── Calls CRI: PullImage (if not cached locally)
        │
        ├── Calls CRI: CreateContainer + StartContainer
        │       │ gRPC
        │       ▼
        │   [containerd]
        │       ├── Calls runc with OCI config.json
        │       │       ├── Creates cgroups (CPU/memory limits)
        │       │       ├── Joins pod's network namespace
        │       │       ├── Mounts overlay filesystem
        │       │       └── exec() → application starts
        │       └── Returns container ID
        │
        ├── Configures Pod networking via CNI
        ├── Configures cgroups for resource limits
        ├── Starts liveness/readiness probes
        └── Reports pod status: Running → etcd via API server
        │
        ▼
APPLICATION IS RUNNING ✓
```

---

## 6. kubelet — The Node Agent

**One-line definition:** kubelet is the **primary node agent** running on every worker node — it ensures containers described in PodSpecs are running and healthy.

### What kubelet Does (Complete List)

| Responsibility | How |
|---------------|-----|
| **Pull container images** | Via CRI ImageService |
| **Create/start containers** | Via CRI RuntimeService |
| **Configure cgroups** | Direct cgroupfs/systemd writes |
| **Configure pod network** | Calls CNI plugin after sandbox creation |
| **Run health checks** | liveness, readiness, startup probes |
| **Report node status** | CPU/memory capacity, conditions → API server |
| **Report pod status** | Running, Succeeded, Failed → API server |
| **Mount volumes** | Via CSI plugins or direct mounts |
| **Manage secrets/configmaps** | Projects them into container filesystems |
| **Handle pod eviction** | Evicts pods when node is under pressure |
| **Garbage collect** | Removes unused images and dead containers |

### kubelet ↔ CRI Communication Flow

```
kubelet
  │
  │  gRPC over Unix socket
  │  /run/containerd/containerd.sock
  ▼
containerd (CRI server)
  │
  ├── containerd-shim-runc-v2 (one per container)
  │         │
  │         ▼
  │       runc (creates the container once, then exits)
  │         │
  │         ▼
  │   Container Process (PID 1 in container)
  │
  └── snapshotter (manages filesystem layers)
```

---

## 7. Key Metrics Reference

### etcd Metrics

| Metric | What It Measures | Alert When |
|--------|-----------------|------------|
| `etcd_server_leader_changes_seen_total` | Number of leader elections | > 3 in 1 hour (cluster instability) |
| `etcd_server_is_leader` | Is this node the leader? (1=yes, 0=no) | Used for leader identification |
| `etcd_network_peer_round_trip_time_seconds_bucket` | Latency between etcd peers | p99 > 150ms (network issue) |
| `etcd_server_proposals_failed_total` | Failed Raft proposals | > 0 (write failures) |
| `etcd_disk_wal_fsync_duration_seconds` | WAL write latency | p99 > 10ms (disk too slow) |
| `etcd_mvcc_db_total_size_in_bytes` | etcd database size | > 2GB (needs compaction) |

### WorkQueue Metrics (Controller Manager)

| Metric | What It Measures | Alert When |
|--------|-----------------|------------|
| `workqueue_adds_total` | Total items added to queue | Baseline deviation |
| `workqueue_depth` | Current items waiting in queue | Consistently > 0 (backlog) |
| `workqueue_queue_duration_seconds` | Time items wait before processing | p99 > 1s (controller slow) |
| `workqueue_work_duration_seconds` | Time to process each item | p99 > 10s (reconcile slow) |
| `workqueue_retries_total` | Items retried after failure | High = reconcile errors |

### kubelet Metrics

| Metric | What It Measures | Alert When |
|--------|-----------------|------------|
| `kubelet_running_pods` | Number of pods currently running on node | Compare to expected |
| `kubelet_pod_start_duration_seconds_count` | How long pods take to start | p99 > 30s (runtime slow) |
| `kubelet_pod_worker_duration_seconds` | Time to process pod update | p99 > 10s |
| `kubelet_node_name` | Node identity | Used for label filtering |
| `kubelet_volume_stats_used_bytes` | Volume usage | > 85% capacity |

### API Server Metrics

| Metric | What It Measures | Alert When |
|--------|-----------------|------------|
| `apiserver_admission_controller_admission_duration_seconds` | Admission webhook latency | p99 > 1s (webhook slow) |
| `apiserver_request_total` | Total API requests by verb/resource | Sudden spikes |
| `apiserver_request_duration_seconds` | API response latency | p99 > 1s |
| `apiserver_current_inflight_requests` | Concurrent requests in flight | > 400 (overload) |
| `rule_evaluations_total` | Prometheus rule evaluations | Monitoring health |

---

## 8. Networking Communication — Q&A

### Q1: Two containers in the SAME Pod — how do they communicate?

**A:** Containers in the same pod **share the same network namespace**. They communicate via `localhost`.

```
Pod: my-pod
├── Container: app       (listens on :8080)
└── Container: sidecar   (connects to localhost:8080)
```

- Both containers see the same network interfaces (`eth0`, `lo`)
- Same IP address
- Communication: `localhost:PORT` — direct, no network hop
- They also share the same PID namespace (optional) and can see each other's processes

```python
# In sidecar container — connects to app on same pod
import requests
response = requests.get("http://localhost:8080/api/data")
```

### Q2: Two Pods in the SAME Namespace, SAME Node — how do they communicate?

**A:** Via **Pod IP addresses** through the node's virtual ethernet bridge (veth pairs).

```
Node
├── Pod A  IP: 10.244.1.3   (veth0 → cbr0)
└── Pod B  IP: 10.244.1.7   (veth1 → cbr0)
                                   │
                              Linux Bridge
                               (cbr0/cni0)
```

- Each pod gets a unique IP from the CNI plugin
- Packets route through the node's bridge device
- No NAT — direct pod-to-pod IP routing
- Pod A → `10.244.1.7:8080` → bridge → Pod B

```yaml
# Pod A calls Pod B directly by IP (or Service DNS)
curl http://10.244.1.7:8080
# OR via Service (recommended)
curl http://pod-b-service.default.svc.cluster.local:8080
```

### Q3: Two Pods in DIFFERENT Namespaces, SAME Node — how do they communicate?

**A:** **Namespaces are NOT a network boundary in K8s.** Pods in different namespaces on the same node communicate identically to pods in the same namespace — directly via Pod IPs through the node bridge.

```
Node
├── Namespace: frontend   Pod A  IP: 10.244.1.3
└── Namespace: backend    Pod B  IP: 10.244.1.8
```

- Pod A can reach Pod B at `10.244.1.8:port` directly
- **Namespace isolation is achieved via NetworkPolicy, not the namespace boundary itself**
- To restrict cross-namespace traffic, apply a `NetworkPolicy` resource

```yaml
# NetworkPolicy to BLOCK cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: backend
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {}        # Only allow from same namespace
```

### Q4: Two Pods on DIFFERENT Nodes — how do they communicate?

**A:** Via **CNI overlay networking** (VXLAN, BGP, etc.). The CNI plugin handles routing between nodes.

```
Node 1 (10.0.0.1)          Node 2 (10.0.0.2)
Pod A: 10.244.1.3           Pod B: 10.244.2.5
     │                              │
  veth0                           veth0
     │                              │
  cbr0/cni0                     cbr0/cni0
     │                              │
  flannel0/calico               flannel0/calico
     │     (tunnel / BGP route)     │
     └──────── Node Network ────────┘
               10.0.0.0/24
```

**With Flannel (VXLAN):**
1. Pod A sends packet to `10.244.2.5`
2. Flannel on Node 1 sees this is on another node's subnet
3. Flannel encapsulates packet in VXLAN UDP packet to Node 2 (`10.0.0.2`)
4. Flannel on Node 2 decapsulates and delivers to Pod B

**With Calico (BGP):**
1. Pod A sends packet to `10.244.2.5`
2. Calico uses BGP-advertised routes — no encapsulation
3. Packet routes directly via node network to Node 2

### Q5: Two Containers in the SAME Pod — memory/IPC sharing?

**A:** Containers in the same pod CAN share:
- **Network namespace** — always shared (same IP, same ports)
- **IPC namespace** — shared if `spec.shareProcessNamespace: true`
- **Volumes** — shared if the same volume is mounted in both containers

```yaml
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: writer
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

### Q6: Two CLUSTERS — how do they communicate?

**A:** Clusters are isolated by default. Cross-cluster communication requires explicit setup:

| Method | How | Use Case |
|--------|-----|----------|
| **External Service / LoadBalancer** | Service exposed externally, other cluster connects via public/private IP | Simple, no special tooling |
| **Istio multicluster** | Service mesh federation, transparent cross-cluster service discovery | Microservices, transparent routing |
| **Submariner** | CNCF project for cross-cluster pod/service networking (connects pod CIDRs) | Direct pod-to-pod across clusters |
| **KubeFed v2 / Admiral** | Federation control plane, replicates resources across clusters | Multi-region active-active |
| **VPN / VPC Peering** | Network-level connectivity between cluster node networks | Cloud environments |

```
Cluster A (us-east)          Cluster B (eu-west)
  Service: api-svc              
  ExternalIP: 34.1.2.3  ←────── External call from Pod in Cluster B
```

---

## 9. kube-proxy & iptables

**One-line definition:** kube-proxy is a **network proxy running on every node** that maintains network rules (iptables/IPVS) to implement K8s Service virtual IPs and load balancing.

### What Problem Does kube-proxy Solve?

Without kube-proxy:
- A Service `ClusterIP: 10.96.1.5` is just a number in etcd
- No actual network routing exists for that IP
- Pod cannot reach the Service

With kube-proxy:
- Every node has iptables rules translating `10.96.1.5:80` → one of `[10.244.1.3:8080, 10.244.2.7:8080]`
- Traffic is automatically load-balanced (round-robin or random)

### iptables Mode (Default)

```
Pod sends packet to ClusterIP 10.96.1.5:80
        │
        ▼
iptables PREROUTING chain
        │
        ▼ DNAT rule (kube-proxy created this)
Random selection:
  50% → 10.244.1.3:8080  (Pod A on Node 1)
  50% → 10.244.2.7:8080  (Pod B on Node 2)
```

kube-proxy watches Services and Endpoints in etcd. When pods are added/removed, it updates iptables rules automatically.

### kube-proxy Modes

| Mode | How | Scale | Latency |
|------|-----|-------|---------|
| **iptables** (default) | netfilter rules in kernel | ~10k services | Low, but O(n) rule lookup |
| **IPVS** | Linux Virtual Server, hash table lookup | 100k+ services | Lower latency, O(1) lookup |
| **eBPF** (Cilium) | Kernel-level network programs, no kube-proxy | 100k+ services | Lowest latency |

### Q: What if kube-proxy goes down?
**A:** Existing iptables rules remain (kernel keeps them). But:
- New Services do not get routing rules
- Removed pods are not removed from routing — traffic goes to dead endpoints
- Eventually rules become stale and traffic fails

---

## 10. Component Failure Impact Analysis

### If Each Component Dies — What Breaks?

| Component | Immediate Impact | Running Workloads | Fix |
|-----------|-----------------|------------------|-----|
| **kube-apiserver** | No kubectl, no API | ✅ Continue running | Restart / HA failover |
| **etcd** | API server read/write blocked | ✅ Continue running | Restore from backup, restart |
| **kube-scheduler** | New pods stay `Pending` | ✅ Continue running | Restart |
| **kube-controller-manager** | No self-healing, no scaling | ✅ Continue running | Restart |
| **cloud-controller-manager** | No LB provisioning | ✅ Continue running | Restart |
| **kubelet** | Node marked `NotReady`, pods evicted elsewhere | ❌ Stop eventually | Fix node, restart kubelet |
| **kube-proxy** | New service routes stop working | ⚠️ Degrade over time | Restart |
| **containerd** | No new containers can start | ❌ Existing may crash | Restart containerd |
| **etcd leader** | New leader elected via Raft | ✅ Brief disruption | Automatic |

### Summary: What You Can't Live Without

```
For RUNNING existing workloads:
  Only kubelet + container runtime are strictly required

For MANAGING the cluster:
  kube-apiserver + etcd are the critical path

For SELF-HEALING & SCALING:
  + kube-scheduler + kube-controller-manager

For NETWORKING:
  + kube-proxy (or CNI with built-in proxy like Cilium)

For CLOUD INTEGRATION:
  + cloud-controller-manager
```

---

## Quick Reference — One-Line Definitions

| Component | One-Line Definition |
|-----------|-------------------|
| **Cluster** | The entire K8s environment — all master + worker nodes managed as one unit |
| **Master Node** | The control plane — runs API server, scheduler, controllers, etcd |
| **Worker Node** | Runs application pods via kubelet + container runtime |
| **kube-apiserver** | The single HTTPS gateway for all K8s operations — validates, persists, notifies |
| **etcd** | Distributed key-value store — the single source of truth for all cluster state |
| **kube-scheduler** | Assigns unscheduled pods to best-fit nodes based on resources and constraints |
| **kube-controller-manager** | Runs all built-in controllers that reconcile desired ↔ actual state |
| **cloud-controller-manager** | Cloud API integration — provisions LBs, routes, node lifecycle |
| **kubelet** | Node agent — ensures pod containers run and are healthy via CRI |
| **kube-proxy** | Maintains iptables/IPVS rules on each node for Service virtual IP routing |
| **CRI** | gRPC API standard defining how kubelet talks to any container runtime |
| **containerd** | The standard CRI-compliant container runtime (what actually runs containers) |
| **OCI** | Open standard defining container image format and runtime parameters |
| **runc** | The reference OCI runtime that creates container processes via Linux kernel features |
| **cgroups** | Linux kernel feature limiting CPU/memory/IO for container process groups |
| **Namespaces (Linux)** | Linux kernel isolation for PID, network, filesystem, IPC per container |
| **CNI** | Plugin interface for container network setup — assigns pod IPs, configures routes |
| **gRPC** | High-performance binary RPC framework over HTTP/2 used for K8s internal communication |
| **Pod** | Smallest deployable unit — one or more containers sharing network + storage |
| **Admission Controller** | API server plugin that validates/mutates requests before they reach etcd |
| **Pause container** | Holds the pod's network namespace alive — all containers in pod join it |

---

*Generated for deep-dive K8s study — covers architecture, protocols, runtime stack, networking, and operational knowledge.*
