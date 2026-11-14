# Kubernetes Interview Guide for Java / Spring Boot Developers (9+ Years Experience)

> **How to use this guide:** Each answer has a **30-second version** (what you actually say first) and a **Deep dive** (for follow-ups). Memory hooks are marked with 🧠. Scenario answers follow a fixed structure so you can reproduce them under pressure.

---

## Table of Contents

1. [The Interview Strategy for a Senior Java Dev](#1-the-interview-strategy-for-a-senior-java-dev)
2. [Kubernetes Fundamentals](#2-kubernetes-fundamentals)
3. [Architecture (Control Plane & Node)](#3-architecture-control-plane--node)
4. [Pods & Pod Lifecycle](#4-pods--pod-lifecycle)
5. [Workload Controllers](#5-workload-controllers)
6. [Services, Networking & Ingress](#6-services-networking--ingress)
7. [ConfigMaps, Secrets & Spring Boot Configuration](#7-configmaps-secrets--spring-boot-configuration)
8. [Health Probes & Spring Boot Actuator](#8-health-probes--spring-boot-actuator)
9. [Resource Management & the JVM in Containers](#9-resource-management--the-jvm-in-containers)
10. [Storage: PV, PVC, StorageClass](#10-storage-pv-pvc-storageclass)
11. [Scaling: HPA, VPA, KEDA, Cluster Autoscaler](#11-scaling-hpa-vpa-keda-cluster-autoscaler)
12. [Deployment Strategies & Zero-Downtime Releases](#12-deployment-strategies--zero-downtime-releases)
13. [Graceful Shutdown (the #1 Spring Boot + K8s Question)](#13-graceful-shutdown-the-1-spring-boot--k8s-question)
14. [Security: RBAC, ServiceAccounts, Pod Security](#14-security-rbac-serviceaccounts-pod-security)
15. [Observability: Logs, Metrics, Tracing](#15-observability-logs-metrics-tracing)
16. [Helm, Kustomize & GitOps](#16-helm-kustomize--gitops)
17. [Scheduling: Affinity, Taints, Topology Spread](#17-scheduling-affinity-taints-topology-spread)
18. [Dockerfile & Image Best Practices for Spring Boot](#18-dockerfile--image-best-practices-for-spring-boot)
19. [Scenario-Based Questions (30 Scenarios)](#19-scenario-based-questions)
20. [Troubleshooting Playbook & Command Cheat Sheet](#20-troubleshooting-playbook--command-cheat-sheet)
21. [Rapid-Fire Revision (Last 30 Minutes Before Interview)](#21-rapid-fire-revision)

---

## 1. The Interview Strategy for a Senior Java Dev

At 9 years, interviewers do **not** expect you to be a CKA-level cluster administrator. They expect:

| They test | They do NOT usually test |
|---|---|
| Can you deploy and debug a Spring Boot app on K8s? | Writing etcd backup scripts from memory |
| Do you understand probes, resources, and graceful shutdown? | CNI plugin internals |
| Can you explain why your pod was OOMKilled? | Building a custom scheduler |
| Do you know config/secret management patterns? | kubeadm cluster bootstrap flags |
| Can you reason about a production incident? | CRD controller code in Go (unless platform role) |

**Golden rule for every answer:** connect Kubernetes concepts back to Java/Spring Boot. Saying *"readiness probe"* is average. Saying *"readiness probe pointed at `/actuator/health/readiness` so traffic only arrives after the connection pool and Kafka consumer are up"* is senior.

🧠 **Memory hook — the "SPRING" checklist** for any "how do you run Spring Boot on K8s" question:
- **S**hutdown gracefully (preStop + graceful shutdown)
- **P**robes (liveness/readiness/startup)
- **R**esources (requests/limits + `MaxRAMPercentage`)
- **I**mage (layered, non-root, distroless/JRE slim)
- **N**etworking (Service, Ingress, DNS)
- **G**uarded config (ConfigMap/Secret, no baked-in props)

---

## 2. Kubernetes Fundamentals

### Q1. What is Kubernetes and what problem does it solve?

**30-second version:**
Kubernetes is a container orchestration platform. It takes a *declarative* description of desired state ("I want 5 replicas of this image, exposed on port 8080, with these env vars") and continuously reconciles reality to match it. It solves scheduling, self-healing, service discovery, scaling, rolling deployments, and config/secret distribution — the things you'd otherwise hand-script around Docker.

**Deep dive:**
Before K8s, we deployed Spring Boot WARs/JARs on VMs with Ansible + a load balancer, and scaling meant raising a ticket. K8s replaces that with:
- **Scheduling** — decides which node runs your pod based on resources/constraints.
- **Self-healing** — restarts crashed containers, replaces dead nodes' pods, kills pods failing health checks.
- **Service discovery & LB** — stable DNS name + virtual IP in front of ephemeral pods.
- **Declarative rollouts** — versioned, rollback-able deployments.
- **Horizontal scaling** — HPA based on CPU/memory/custom metrics.

🧠 **One-liner to remember:** *"Kubernetes is a control loop over declarative state."* Everything else in K8s is a variation of `while (desired != actual) { fix(); }`.

### Q2. Declarative vs imperative — why does it matter?

- **Imperative:** `kubectl run`, `kubectl scale`, `kubectl create` — you tell K8s *what to do*.
- **Declarative:** `kubectl apply -f deployment.yaml` — you tell K8s *what you want*, it figures out the diff.

**Why senior engineers care:** declarative YAML in Git = auditable, reviewable, reproducible, and GitOps-compatible. Imperative commands are fine for debugging but create drift. In interviews always say: *"imperative for investigation, declarative for anything that reaches an environment."*

**Plain-language analogy:** imperative is giving a taxi driver turn-by-turn directions ("left, then right…"); declarative is giving the destination and letting the driver figure out the route. If the road is blocked (a pod dies), the declarative system re-plans on its own, whereas the imperative command already ran once and won't run again.

**Example of the drift problem:**
```bash
kubectl scale deployment/order-service --replicas=10   # imperative: live cluster now has 10
# Git still says replicas: 3. The next `kubectl apply` / GitOps sync silently resets it to 3,
# and nobody knows why the extra capacity vanished.
```
**How `apply` works:** it compares the YAML you submit with the last-applied configuration (stored in an annotation) and the live object, then sends only the *difference* — that is why re-running `apply` on the same file is **idempotent** (safe to repeat). Interview probe: *"What is the difference between `create`, `apply` and `replace`?"* → `create` fails if the object exists, `apply` merges changes, `replace` overwrites the whole object.

### Q3. What is a Namespace and how do you use it in practice?

A namespace is a **logical partition** of cluster resources — a scope for names, RBAC, quotas, and network policies. It is **not** a security boundary by itself (pods in different namespaces can still talk unless a NetworkPolicy stops them).

Practical usage patterns:
- Per environment on a shared non-prod cluster: `dev`, `qa`, `uat`.
- Per team/domain in prod: `payments`, `orders`, `identity`.
- `ResourceQuota` + `LimitRange` per namespace to stop one team eating the cluster.

Cross-namespace DNS: `orders-service.orders.svc.cluster.local`.

🧠 Namespaces scope: **names, RBAC, quota, network policy**. They do **not** scope: nodes, PersistentVolumes, StorageClasses, ClusterRoles.

### Q4. Explain labels, selectors, and annotations. Why three concepts?

| Concept | Purpose | Queryable? | Example |
|---|---|---|---|
| **Label** | Identify & group objects | Yes | `app=order-service, version=v2` |
| **Selector** | Query labels | — | Service selects `app=order-service` |
| **Annotation** | Attach non-identifying metadata | No | `prometheus.io/scrape: "true"` |

Rule of thumb: **if a controller needs to find it, it's a label. If a tool needs to read it, it's an annotation.** Ingress controller config, Prometheus scrape hints, `kubectl.kubernetes.io/last-applied-configuration`, and change-cause all live in annotations.

Recommended standard labels:
```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/instance: order-service-prod
  app.kubernetes.io/version: "2.14.1"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: order-platform
```

### Q5. What are the core object types you use daily?

```
Workloads:    Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob
Networking:   Service, Ingress, NetworkPolicy, EndpointSlice
Config:       ConfigMap, Secret
Storage:      PersistentVolume, PersistentVolumeClaim, StorageClass
Scaling:      HorizontalPodAutoscaler, PodDisruptionBudget
Security:     ServiceAccount, Role, RoleBinding, ClusterRole, ClusterRoleBinding
Governance:   ResourceQuota, LimitRange
Extension:    CustomResourceDefinition (CRD), Operator
```

What each group means in one line (so you can explain it, not just list it):
- **Workloads** — *how your code runs*: a Pod is one running instance; the other kinds (Deployment, StatefulSet, DaemonSet, Job, CronJob) manage pods for you with different guarantees (see Q14).
- **Networking** — *how pods are reached*: a Service gives a stable address to a changing set of pods, an Ingress routes external HTTP traffic in, a NetworkPolicy is a pod-level firewall, and an EndpointSlice is the list of pod IPs behind a Service.
- **Config** — *settings kept outside the image*: ConfigMap for plain config, Secret for credentials (Q24).
- **Storage** — *disks that outlive pods*: a PVC requests storage, a PV is the disk, a StorageClass says how to create one on demand (Q41).
- **Scaling / availability** — the HPA changes replica count by load; a PodDisruptionBudget limits how many pods may be taken down at once during voluntary maintenance (node drains).
- **Security** — who may do what: a ServiceAccount is a pod's identity; Role/ClusterRole list permissions; the *Binding* objects attach those permissions to an identity (Q56).
- **Governance** — guardrails: ResourceQuota caps a namespace's total usage, LimitRange sets per-container defaults/bounds (Q40).
- **Extension** — a CRD teaches the API server a new object type (e.g. `Certificate`, `KafkaTopic`); an Operator is a controller that watches those objects and automates operations (e.g. the Strimzi Kafka operator).

Quick tip: `kubectl api-resources` lists every kind the cluster knows, and `kubectl explain deployment.spec.strategy` documents any field offline.

---

## 3. Architecture (Control Plane & Node)

### Q6. Walk me through Kubernetes architecture.

**30-second version:**
The **control plane** decides what should happen; the **worker nodes** make it happen. The API server is the only component that talks to etcd; everything else talks to the API server and watches for changes.

**Control plane components:**

| Component | Job | Interview soundbite |
|---|---|---|
| **kube-apiserver** | REST front door; auth, validation, admission, persistence | "The only door into the cluster" |
| **etcd** | Distributed key-value store; the single source of truth | "The cluster's database — back this up or lose everything" |
| **kube-scheduler** | Assigns pending pods to nodes (filter → score → bind) | "It only *decides*; kubelet actually runs the pod" |
| **kube-controller-manager** | Runs control loops (Deployment, ReplicaSet, Node, Endpoint, etc.) | "The reconciliation engine" |
| **cloud-controller-manager** | Cloud-specific bits: LoadBalancers, routes, node lifecycle | "Where `type: LoadBalancer` becomes an actual ELB/ALB" |

**Node components:**

| Component | Job |
|---|---|
| **kubelet** | Node agent. Watches API server for pods assigned to its node, tells the container runtime to start them, runs probes, reports status |
| **Container runtime** | containerd / CRI-O. Actually pulls images and runs containers (Docker shim was removed in 1.24) |
| **kube-proxy** | Programs iptables/IPVS rules so Service ClusterIPs route to pod IPs. Being replaced by eBPF in Cilium-based clusters |
| **CoreDNS** | Cluster DNS (runs as a Deployment, but node-level relevance) |

### Q7. What happens end-to-end when you run `kubectl apply -f deployment.yaml`?

This is a very common senior question. Say it as a chain:

1. **kubectl** reads YAML → converts to JSON → sends `POST/PATCH` to kube-apiserver over TLS.
2. **API server**: authenticates (cert/token/OIDC) → authorizes (RBAC) → runs **admission controllers** (mutating first — e.g., sidecar injection, defaulting; then validating — e.g., OPA/Gatekeeper, Pod Security) → validates schema.
3. **etcd**: object persisted. API server returns 201 to kubectl. *At this point the request is "done" — but nothing is running yet.*
4. **Deployment controller** (in controller-manager) watches Deployments, sees a new one, creates a **ReplicaSet**.
5. **ReplicaSet controller** sees replicas=3, actual=0 → creates 3 **Pod** objects with `nodeName` empty (state: Pending).
6. **Scheduler** watches for unbound pods → **filters** nodes (resources, taints, affinity, node selectors) → **scores** the survivors → binds the pod to the best node (writes `nodeName`).
7. **kubelet** on that node sees a pod bound to it → calls **CRI** to pull the image and start the container(s), calls **CNI** to attach network/assign pod IP, calls **CSI** to mount volumes.
8. kubelet runs **startup/readiness/liveness probes**, updates pod status back to the API server.
9. **EndpointSlice controller** sees a Ready pod matching a Service selector → adds pod IP to the EndpointSlice.
10. **kube-proxy** on every node updates iptables/IPVS → traffic to the Service ClusterIP now reaches the new pod.

🧠 **Memory hook — "A-E-C-S-K-E":** **A**PI server → **E**tcd → **C**ontroller → **S**cheduler → **K**ubelet → **E**ndpoints.

### Q8. Why is etcd important and how is it protected?

etcd holds all cluster state (objects, secrets — base64, not encrypted by default). It uses **Raft** consensus, needs an **odd number of members** (3 or 5) for quorum, and tolerates `(n-1)/2` failures.

Production concerns you should mention:
- Regular `etcdctl snapshot save` backups + tested restore procedure.
- **Encryption at rest** for secrets (`EncryptionConfiguration` on the API server) — otherwise secrets are only base64.
- Low-latency disk (etcd is fsync-sensitive; slow disks = slow cluster).
- Never expose etcd; only the API server should reach it.

**Why it matters / how it works (plain language):**
- Every object you `kubectl apply` ends up as a key/value entry in etcd (e.g. `/registry/deployments/orders/order-service`). Controllers don't poll etcd — they **watch** the API server, which streams changes from etcd. So etcd is the "memory" of the cluster; lose it and the cluster forgets every Deployment, Service and Secret (running containers keep running, but nothing manages them).
- **Raft** is a leader-election-plus-replicated-log algorithm: one member is the leader, writes are accepted only when a **majority (quorum)** has stored them. That's why you want an odd number — 3 members tolerate 1 failure (quorum 2), 5 tolerate 2 (quorum 3); 4 members still only tolerate 1, so the extra node buys nothing.
- On managed services (EKS/AKS/GKE) the provider runs and backs up etcd for you; the interview answer is then "I don't operate it, but I know what it stores and why API-server latency or object-count explosions (e.g. millions of Events/Jobs) can slow the whole cluster".

Quick check of Secrets being only base64: `kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d` prints the plain password for anyone with read access.

### Q9. If the control plane goes down, do my running pods die?

**No.** Already-running pods keep serving traffic — kubelet and kube-proxy work from local state.
What you *lose*: scheduling new pods, self-healing, scaling, deployments, `kubectl`, and Service endpoint updates. So a control plane outage isn't an immediate customer outage, but the cluster stops reacting to failures — the next pod crash won't be replaced.

This is a great answer to demonstrate operational maturity.

**Why this works:** the control plane is only the *brain*; the kubelet on each node is the *hands*. The kubelet already has the pod specs cached and keeps containers running and restarting them locally (per the pod's `restartPolicy`) even when it can't reach the API server. kube-proxy rules and CoreDNS are also already in place, so existing Service traffic keeps flowing.

**What breaks, concretely:**
- A pod crashes on a node → the kubelet may restart the *container*, but if the whole node dies, nobody reschedules its pods elsewhere (the controller-manager is down).
- A new pod becomes Ready or an old one dies → EndpointSlices aren't updated, so a Service may keep sending traffic to a dead pod IP.
- HPA can't scale, `kubectl` calls fail, rolling deployments stall, and CronJobs don't fire.

**Follow-up probe:** *"How do you make the control plane highly available?"* → run 3 API servers behind a load balancer, a 3- or 5-member etcd quorum, and leader-elected scheduler/controller-manager instances (only one is active at a time, the others stand by). Managed clusters do this for you.

---

## 4. Pods & Pod Lifecycle

### Q10. What is a Pod and why not just a container?

A Pod is the smallest deployable unit: one or more containers that share a **network namespace (same IP, same port space, can talk over `localhost`)**, **IPC**, and optionally **volumes**. Containers in a pod are always co-scheduled on the same node and share a lifecycle.

🧠 Think of the pod as **"a logical host"**. Your Spring Boot app is the process; the sidecar (Envoy, fluent-bit, Vault agent) is another process on the same host.

**Rule:** one main application container per pod. Sidecars only for cross-cutting concerns.

**Why not just a container?** Kubernetes needs a unit that can hold *tightly coupled* processes which must share a network identity and disk. A bare container can't express "these two processes must always run on the same machine, talk over `localhost`, and share a folder". The pod is that wrapper — and it is also the unit that gets an IP, is scheduled, scaled and killed. Scaling is done by adding **pods** (copies of the whole group), never by adding containers to a pod.

**Example — Spring Boot app + log-shipper sidecar sharing a volume:**
```yaml
spec:
  containers:
    - name: app
      image: registry/order-service:2.14.1
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
    - name: log-shipper              # reads what the app writes; same pod IP, same lifecycle
      image: fluent/fluent-bit:2.2
      volumeMounts: [{ name: logs, mountPath: /var/log/app, readOnly: true }]
  volumes:
    - name: logs
      emptyDir: {}                   # scratch volume that lives and dies with the pod
```
**Pitfalls / probes:** (1) Two containers in one pod can't both bind port 8080 — they share the port space. (2) Pods are **ephemeral**: you almost never create bare pods; a controller (Deployment etc., Q14) recreates them, and each replacement gets a *new IP*, which is why you talk to a Service instead of a pod IP.

### Q11. Init containers vs sidecar containers.

| | Init container | Sidecar |
|---|---|---|
| When | Runs **before** app containers, sequentially, to completion | Runs **alongside** app container |
| Failure | Pod restarts init (blocks app start) | App continues (unless it depends on it) |
| Spring Boot use case | Wait for DB, run Flyway/Liquibase migration, fetch secrets/certs, `chown` a volume | Log shipper, Envoy proxy, Vault agent, metrics adapter |

Real Spring Boot pattern — DB migration as init container so N replicas don't race on Flyway:
```yaml
initContainers:
  - name: flyway-migrate
    image: flyway/flyway:10
    args: ["-url=jdbc:postgresql://db:5432/orders", "migrate"]
    envFrom:
      - secretRef: { name: db-credentials }
```

**Note (K8s 1.29+):** native sidecars exist as init containers with `restartPolicy: Always` — they start before app containers and keep running. Mentioning this signals you're current.

### Q12. Explain pod phases and container states.

**Pod phases:** `Pending` → `Running` → `Succeeded` / `Failed` (and `Unknown`).

**Container states:** `Waiting` (with a reason like `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`), `Running`, `Terminated` (with exit code).

**What these mean in practice:**
- `Pending` — the pod object exists but isn't running yet: either the scheduler hasn't found a node (not enough CPU/memory, unmatched taints/affinity — see `kubectl describe pod` → Events → `FailedScheduling`) or images are still being pulled.
- `Running` — at least one container is running (or starting/restarting). **Running does not mean Ready** — readiness is a separate condition driven by the readiness probe.
- `Succeeded` / `Failed` — all containers ended (exit 0 / non-zero); these are the normal end states for Job pods.
- `Unknown` — the node stopped reporting; usually a node/network problem.
- `ImagePullBackOff` = can't pull the image (wrong tag, private registry without `imagePullSecrets`); `ErrImagePull` is the first failure, `BackOff` is the retry delay.

Quick triage: `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'` shows *why the previous container died* (reason + exit code).

**Exit codes worth memorizing:**

| Code | Meaning | Typical Spring Boot cause |
|---|---|---|
| 0 | Clean exit | Job finished; or app exited because no web server on classpath |
| 1 | Application error | Uncaught exception at startup, bad config, `ApplicationContext` failure |
| 137 | SIGKILL (128+9) | **OOMKilled** or shutdown grace period exceeded |
| 143 | SIGTERM (128+15) | Normal pod termination (this is *fine*) |
| 139 | SIGSEGV | Native/JNI crash |

🧠 **137 = memory (Killed), 143 = terminated politely.** If you see 137 with `Reason: OOMKilled`, it's the memory limit, not your code.

### Q13. What is CrashLoopBackOff and how do you debug it?

It means the container starts, exits, and kubelet restarts it with **exponential backoff** (10s, 20s, 40s … capped at 5 min).

Debug sequence:
```bash
kubectl describe pod <pod>              # events + last state + exit code + reason
kubectl logs <pod> --previous           # logs from the CRASHED instance (key flag!)
kubectl logs <pod> -c <container>       # multi-container pods
kubectl get events --sort-by=.lastTimestamp
```

Top 6 causes for Spring Boot:
1. **Bad config** — missing env var/ConfigMap key → `ApplicationContext` fails to start.
2. **DB/dependency unreachable** at startup → HikariCP init fails.
3. **OOMKilled** — heap + metaspace + native > memory limit.
4. **Liveness probe too aggressive** — app needs 60s to boot, probe kills it at 30s (looks like a crash loop but the app is healthy).
5. **Wrong port/command** — probe port doesn't match `server.port`.
6. **Missing secret/volume mount** or file permission issue with non-root user.

---

## 5. Workload Controllers

### Q14. Deployment vs ReplicaSet vs StatefulSet vs DaemonSet vs Job/CronJob.

| Controller | Use for | Key property |
|---|---|---|
| **Deployment** | Stateless apps (99% of Spring Boot services) | Manages ReplicaSets → rolling updates + rollback |
| **ReplicaSet** | Rarely direct | Just keeps N pods alive; no rollout history |
| **StatefulSet** | Kafka, Zookeeper, Cassandra, Postgres, Elasticsearch | Stable **ordinal names** (`kafka-0`), stable **DNS**, stable **per-pod PVC**, **ordered** create/update/delete |
| **DaemonSet** | One pod per node: log agents (fluent-bit), node-exporter, CNI, CSI | Auto-scales with the node pool |
| **Job** | Run-to-completion batch: data migration, one-off report | `completions`, `parallelism`, `backoffLimit` |
| **CronJob** | Scheduled batch: nightly reconciliation, cleanup | Cron schedule + `concurrencyPolicy` |

🧠 **Deployment = cattle (interchangeable). StatefulSet = pets (identity matters).**

**What each really does (plain language):**
- A **ReplicaSet** is the loop "make sure exactly N pods matching this template exist" — if one dies it creates another. A **Deployment** sits on top: when you change the pod template (new image), it creates a *new* ReplicaSet and gradually moves pods across, keeping the old one around so you can roll back. That's why you almost never create a ReplicaSet yourself.
- A **StatefulSet** exists for software that must know *who it is*: `kafka-0` is always `kafka-0`, reachable at a stable DNS name and always re-attached to its own disk, even after being rescheduled. Pods start in order (0, 1, 2) and stop in reverse.
- A **DaemonSet** guarantees one copy per node (or per matching node): when a node joins, a pod appears automatically; ideal for per-machine agents (log shipper, node metrics, CNI/CSI plugins).
- A **Job** runs a pod until it *completes successfully* (retrying on failure up to `backoffLimit`), then stops — unlike a Deployment, which restarts forever. A **CronJob** creates a Job on a schedule.

**Quick chooser:** stateless HTTP service → Deployment; needs stable identity/disk → StatefulSet; one per node → DaemonSet; run once and finish → Job; run nightly → CronJob.
```bash
kubectl get deploy,rs,sts,ds,job,cronjob -n orders      # see all controllers and what they own
kubectl get pods -n orders --show-labels                # pod names reveal the owner, e.g. order-service-7c9d8f-x2k4p
```
Pod names tell you the owner: `order-service-<replicaset-hash>-<random>` = Deployment; `kafka-0` = StatefulSet.

### Q15. When would a Spring Boot app need a StatefulSet?

Usually it shouldn't — a well-designed Spring Boot service is stateless and belongs in a Deployment. Legitimate cases:
- The app itself is the stateful system (an embedded-store service, Hazelcast/Infinispan cluster needing stable peer discovery).
- Sharded consumers where each instance owns a fixed shard and needs a stable identity.
- Leader election implemented via ordinal (`pod-0` is leader) — though `ShedLock`/`Quartz` in a DB is usually better.

Say clearly: *"If I need a StatefulSet for my Spring Boot app, I first check whether I've accidentally kept state in the app that belongs in Postgres, Redis, or S3."*

**Why this is the right instinct:** a Deployment's pods are interchangeable — any replica can be killed and replaced on any node, and the load balancer can send any request to any replica. That only works if no replica holds unique, irreplaceable data. Local files, in-memory HTTP sessions, or "instance 1 owns customers A–M" break that assumption. Externalising state (sessions → Redis via Spring Session, files → S3, data → DB) makes the app horizontally scalable and disposable.

**What a StatefulSet actually gives the app in the legitimate cases:**
- A stable hostname (`app-0.app-headless`) → cluster members (Hazelcast/Infinispan, embedded Kafka-like brokers) can find each other by DNS even after restarts.
- A stable ordinal → the instance can derive "I'm shard 2 of 6" from its pod name (`HOSTNAME` env var, `app-2`).
- A per-pod PVC → each instance keeps its own on-disk state (e.g. a local RocksDB/Kafka Streams state store) across restarts, avoiding long rebuilds.

**Pitfalls:** StatefulSet rolling updates are slower (one pod at a time, in reverse order); pod-0 "leader" logic breaks when pod-0 is down; and for Kafka Streams with local state stores you also configure `num.standby.replicas` so a restart doesn't mean a full state rebuild.

### Q16. How do rolling updates work? What are maxSurge and maxUnavailable?

```yaml
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # extra pods above replicas (count or %)
      maxUnavailable: 0    # pods that may be unavailable during the rollout
  minReadySeconds: 15      # pod must stay Ready this long before counted as available
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 5
```

- `maxUnavailable: 0` + `maxSurge: 1` → **safest**, zero capacity loss, slowest rollout.
- `maxSurge: 25%, maxUnavailable: 25%` (default) → faster, but you temporarily run at 75% capacity.
- `maxUnavailable: 0` requires headroom on the cluster for the surge pods.

Under the hood: the Deployment creates a **new ReplicaSet** and shifts replicas from old RS to new RS while respecting these two numbers. Old ReplicaSets are kept (scaled to 0) for rollback — that's what `revisionHistoryLimit` controls.

**Commands:**
```bash
kubectl rollout status deployment/order-service
kubectl rollout history deployment/order-service
kubectl rollout undo deployment/order-service                # previous revision
kubectl rollout undo deployment/order-service --to-revision=3
kubectl rollout restart deployment/order-service             # recreate pods (re-read ConfigMaps)
kubectl rollout pause/resume deployment/order-service        # manual canary
```

### Q17. What is `Recreate` strategy and when is it appropriate?

`type: Recreate` kills **all** old pods before creating new ones → guaranteed downtime, but guarantees **no two versions run simultaneously**. Use when:
- A DB schema change is backward-incompatible (old and new can't coexist).
- The app uses a non-shareable exclusive resource (single-writer file lock, exclusive Kafka consumer with a fixed partition assignment).
- Local dev environments where speed > uptime.

**How it works:** the Deployment scales the old ReplicaSet to 0 and *waits until those pods are gone*, then scales the new ReplicaSet up. Compare that to RollingUpdate, which overlaps old and new.
```yaml
spec:
  strategy:
    type: Recreate        # no maxSurge/maxUnavailable allowed with this type
```
**Concrete scenario:** release v2 renames a DB column in a destructive migration. If v1 pods are still running while v2 pods start, v1 starts throwing `column not found`. With `Recreate` you accept a short outage but never have the two versions touching the schema at once. The better long-term fix for zero downtime is the expand/contract migration pattern (Q51), which lets you stay on RollingUpdate.

**Pitfall:** downtime lasts as long as the old pods' graceful shutdown plus new pods' startup and readiness — for a slow-booting Spring Boot app that can be minutes, so tune `startupProbe` and `terminationGracePeriodSeconds` accordingly.

### Q18. Job vs CronJob details worth knowing.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-reconciliation }
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Kolkata"          # 1.27+ — otherwise UTC, a classic prod bug
  concurrencyPolicy: Forbid          # Allow | Forbid | Replace
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600    # hard timeout
      ttlSecondsAfterFinished: 86400 # auto-cleanup finished jobs
      template:
        spec:
          restartPolicy: OnFailure   # Never | OnFailure (Always is invalid for Jobs)
          containers:
            - name: batch
              image: registry/order-batch:1.4.0
              args: ["--job=reconciliation"]
```

Senior points:
- **`concurrencyPolicy: Forbid`** is the answer to "what if last night's job is still running?"
- **`timeZone`** — before 1.27 CronJobs were UTC-only; countless "why did it run at 7:30 AM IST" incidents.
- Spring Batch on K8s: run as a Job, not a `@Scheduled` method in a 3-replica Deployment (which would run three times). If you must use `@Scheduled`, guard with **ShedLock**.

---

## 6. Services, Networking & Ingress

### Q19. Explain the Service types.

| Type | What it does | When |
|---|---|---|
| **ClusterIP** (default) | Virtual IP reachable only inside the cluster | Internal microservice-to-microservice calls |
| **NodePort** | Opens a port (30000–32767) on **every** node | Dev/debug, or behind an external LB |
| **LoadBalancer** | Provisions a cloud LB (ELB/ALB/NLB) pointing at NodePorts | Cloud-exposed single service |
| **ExternalName** | CNAME to an external DNS name; no proxying | Point `payments-db` at an RDS endpoint without code change |
| **Headless** (`clusterIP: None`) | No VIP; DNS returns **all pod IPs** | StatefulSets, client-side LB, gRPC, Kafka bootstrap |

**Why headless matters for Java:** gRPC and HTTP/2 clients hold long-lived connections. A ClusterIP Service load-balances at *connection* time, so one connection = one pod forever → skewed load. With a headless Service the client resolves all pod IPs and does client-side round-robin per request.

**Port terminology (a frequent source of 'connection refused'):**
```yaml
apiVersion: v1
kind: Service
metadata: { name: order-service }
spec:
  type: ClusterIP
  selector: { app: order-service }   # pods with this label become the endpoints
  ports:
    - name: http
      port: 80            # port clients call on the Service (http://order-service:80)
      targetPort: 8080    # port the Spring Boot container actually listens on (server.port)
      # nodePort: 30080   # only for NodePort/LoadBalancer types
```
`port` is the Service's own port, `targetPort` is the pod's port (can be a *named* container port, so the number can change without editing the Service). **Types are layered:** a LoadBalancer Service also creates a NodePort, which also creates a ClusterIP — each type builds on the previous one. **Pitfall:** the Service's `selector` must match the *pod* labels (not the Deployment's labels); a mismatch gives an empty endpoint list and no traffic (Scenario 7).

### Q20. How does a Service actually route traffic? Is it a proxy?

There's **no process** sitting on the ClusterIP. Flow:
1. Service gets a virtual ClusterIP from the service CIDR.
2. **EndpointSlice controller** watches pods matching the selector **and Ready** → maintains a list of pod IP:port.
3. **kube-proxy** on each node writes **iptables** (or **IPVS**) rules: "traffic to `10.96.0.42:80` → DNAT to one of these pod IPs".
4. Packets are rewritten in the kernel. Load balancing is random/round-robin at the connection level.

🧠 **"A Service is a DNS name + a set of iptables rules, not a proxy."** Saying this correctly is a strong signal.

Important consequence: **only Ready pods receive traffic**, which is exactly why the readiness probe is your traffic gate.

### Q21. How does DNS work in the cluster?

CoreDNS gives every Service an A record:
```
<service>.<namespace>.svc.cluster.local
```
Examples from a Spring Boot app:
```properties
# same namespace
order.service.url=http://order-service:8080
# cross namespace
payment.service.url=http://payment-service.payments.svc.cluster.local:8080
```

Pods get `/etc/resolv.conf` with `search <ns>.svc.cluster.local svc.cluster.local cluster.local` and `ndots:5` — which means short names get tried with the search suffixes first. That `ndots:5` is a known source of DNS latency for external hostnames; fix with `dnsConfig: options: [{name: ndots, value: "2"}]` or FQDN with a trailing dot.

**JVM gotcha:** the JVM caches DNS. `networkaddress.cache.ttl` defaults can pin a resolved IP, so a pod that restarted with a new IP is never re-resolved. For Service DNS (stable VIP) it's fine, but for **headless** Services set `-Dsun.net.inetaddr.ttl=30` or `networkaddress.cache.ttl=30`. This is a great senior detail to drop.

### Q22. Service vs Ingress vs Gateway API.

- **Service** = L4 (TCP/UDP), one service per exposure, no host/path routing.
- **Ingress** = L7 (HTTP/HTTPS). One entry point, host and path routing, TLS termination, one cloud LB for many services. Needs an **Ingress Controller** (NGINX, Traefik, HAProxy, AWS ALB Controller) — the Ingress object alone does nothing.
- **Gateway API** = the successor: role-oriented (`GatewayClass`/`Gateway`/`HTTPRoute`), richer routing (header/weight-based), portable across implementations. Mention it as "where the ecosystem is heading; solves Ingress's annotation sprawl".

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend: { service: { name: order-service, port: { number: 8080 } } }
          - path: /payments
            pathType: Prefix
            backend: { service: { name: payment-service, port: { number: 8080 } } }
```

**Why three options exist:** exposing every service with its own `type: LoadBalancer` means one cloud load balancer (and bill, and public IP) per service, with no host/path routing, no centralised TLS. Ingress fixes that by putting **one HTTP entry point in front of many Services**.

**How the request flows:** client → cloud LB → **Ingress controller pods** (e.g. NGINX, which watches Ingress objects and regenerates its own config) → Service → Spring Boot pod. The Ingress resource is only *configuration*; the controller is the actual reverse proxy. `pathType: Prefix` means `/orders` also matches `/orders/42`; `Exact` matches only that exact path.

**Spring Boot points:** the app sits behind a proxy, so set `server.forward-headers-strategy=framework` (or `native`) so Spring honours `X-Forwarded-For/Proto/Host` — otherwise generated redirects/links use `http://pod-ip:8080`. Keep the Actuator on a separate management port that is *not* routed by the Ingress. Match timeouts: the proxy read timeout should exceed your slowest legitimate request, and the proxy's keep-alive/idle timeout should be longer than the app's (see Scenario 5).

**Gateway API vs Ingress in one line:** Ingress = one object mixing infra and app concerns plus vendor annotations; Gateway API splits it — platform team owns `Gateway`, app teams own `HTTPRoute` — and supports traffic weights/header matches natively (handy for canaries, Q50).
### Q23. What is a NetworkPolicy?

By default **all pods can talk to all pods** — a flat network. A NetworkPolicy is a namespaced firewall selected by pod labels. Key facts:
- Enforced by the **CNI** (Calico, Cilium, Weave). On a CNI without support (plain flannel), policies are silently ignored — a dangerous false sense of security.
- Policies are **additive allow-lists**. As soon as one policy selects a pod, that pod is default-deny for the specified direction.

Default-deny + explicit allow (the pattern to show):
```yaml
# 1. Deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress, namespace: orders }
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
# 2. Allow only api-gateway -> order-service:8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-gateway-to-order, namespace: orders }
spec:
  podSelector: { matchLabels: { app: order-service } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { name: edge } }
          podSelector: { matchLabels: { app: api-gateway } }
      ports:
        - port: 8080
          protocol: TCP
```

🧠 Remember to **always allow DNS egress (UDP/TCP 53 to kube-system)** when you write egress policies — forgetting this breaks every service call and is the classic NetworkPolicy incident.

---

## 7. ConfigMaps, Secrets & Spring Boot Configuration

### Q24. ConfigMap vs Secret — real differences.

| | ConfigMap | Secret |
|---|---|---|
| Data | Plain text/UTF-8 | base64-encoded (`data`) or plain (`stringData`) |
| Encryption | None | **None by default** — only base64. Needs etcd encryption-at-rest |
| Size cap | ~1 MiB (etcd limit) | ~1 MiB |
| RBAC | Usually readable | Should be tightly restricted |
| Mount | env, volume, args | env, volume (volume preferred) |

**Honest senior answer to "are Secrets secure?"** — *"A Secret is not encryption, it's separation of concerns plus RBAC plus optional etcd encryption. For real secret management I use an external store — HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault — surfaced via the External Secrets Operator or the Secrets Store CSI driver, so the secret never lives in Git and rotation happens outside the deploy cycle."*

### Q25. Four ways to get config into a Spring Boot pod (and which to prefer).

**1. Individual env vars** — good for a couple of values:
```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: prod
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef: { name: db-credentials, key: password }
```
Spring Boot's **relaxed binding** maps `SPRING_DATASOURCE_PASSWORD` → `spring.datasource.password`. For nested/indexed properties: `MYAPP_CLIENTS_0_URL` → `myapp.clients[0].url`.

**2. `envFrom`** — inject an entire ConfigMap/Secret as env vars:
```yaml
envFrom:
  - configMapRef: { name: order-service-config }
  - secretRef:    { name: order-service-secrets }
```

**3. Mount as a volume containing `application.yaml`** — the cleanest for Spring:
```yaml
volumes:
  - name: config
    configMap: { name: order-service-config }
containers:
  - name: app
    volumeMounts:
      - name: config
        mountPath: /config
        readOnly: true
    env:
      - name: SPRING_CONFIG_ADDITIONAL_LOCATION
        value: file:/config/
```
Spring Boot picks up `/config/application.yaml` automatically (it's a default search location) and it **overrides** the packaged one. Using `SPRING_CONFIG_ADDITIONAL_LOCATION` (not `SPRING_CONFIG_LOCATION`) preserves the classpath defaults.

**4. Spring Cloud Kubernetes** — the app reads ConfigMaps/Secrets from the API directly and can **hot-reload** them:
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```
Requires a ServiceAccount with `get/list/watch` on configmaps and secrets. Downside: couples your app to the K8s API and needs RBAC. Prefer plain volume/env unless you truly need dynamic refresh.

**Preference order to state in the interview:** volume-mounted `application.yaml` for app config → env vars for a few overrides → Secret volume (not env) for credentials → external secret store for anything sensitive/rotating.

### Q26. Why prefer volume-mounted Secrets over env vars?

1. Env vars are visible in `kubectl describe pod`, in the process environment (`/proc/<pid>/environ`), and often in crash dumps, heap dumps, and log frameworks that print the environment.
2. Env vars are **fixed at container start** — a rotated Secret needs a pod restart. Volume-mounted secrets are **updated in place** (~kubelet sync period, ~60s) so an app that re-reads the file can pick up rotation without a restart.
3. Child processes inherit env vars.

**How to consume a volume-mounted Secret in Spring Boot** — mount it, then use Spring's `configtree` import, which turns each *file* into a property (file name = key, file content = value):
```yaml
volumes:
  - name: db-secret
    secret: { secretName: db-credentials, defaultMode: 0400 }   # owner read-only
containers:
  - name: app
    volumeMounts: [{ name: db-secret, mountPath: /etc/secrets/db, readOnly: true }]
    env:
      - name: SPRING_CONFIG_IMPORT
        value: configtree:/etc/secrets/db/
# file /etc/secrets/db/spring.datasource.password  ->  property spring.datasource.password
```
**Caveat:** the file on disk changes on rotation, but Spring has already read the value at startup — the app still needs a refresh/restart (or a library that re-reads the file, e.g. a Hikari credentials provider) to actually use the new password. Many teams simply roll the pods after rotation.

### Q27. If I update a ConfigMap, does my Spring Boot app see the change?

| Injection method | Live update? |
|---|---|
| `env` / `envFrom` | ❌ No — needs a pod restart |
| Volume mount | ✅ File content updates (~1 min), but **Spring Boot won't re-read it** unless you refresh |
| `subPath` volume mount | ❌ No — subPath mounts are never updated (classic gotcha!) |
| Spring Cloud Kubernetes + `@RefreshScope` | ✅ Yes, with reload enabled |
| Immutable ConfigMap (`immutable: true`) | ❌ By design (better performance, safer) |

**Standard, boring, production-correct answer:** *"I treat config as immutable and roll the pods. I add a checksum annotation to the pod template so a ConfigMap change automatically triggers a rolling restart."*

Helm pattern:
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Manual equivalent: `kubectl rollout restart deployment/order-service`.

---

## 8. Health Probes & Spring Boot Actuator

### Q28. Explain the three probes and what each one does when it fails.

| Probe | Question it answers | On failure |
|---|---|---|
| **Startup** | "Has the app finished booting?" | Keeps restarting? No — it **disables** liveness/readiness until it passes; if it exceeds `failureThreshold × periodSeconds`, container is **killed** |
| **Readiness** | "Can it serve traffic *right now*?" | Pod removed from Service endpoints. **Container is NOT restarted** |
| **Liveness** | "Is it alive or wedged?" | Container is **restarted** by kubelet |

🧠 **Liveness = restart. Readiness = remove from load balancer. Startup = give me time to boot.**

**Plain-language analogy:** think of a restaurant. *Startup* = "the kitchen is still being set up — don't judge us yet". *Readiness* = the "Open / Closed" sign: when we're swamped or the fridge is down we flip to Closed, customers go elsewhere, but nobody is fired. *Liveness* = checking whether the chef has collapsed — only then do we replace them.

**How each probe works:** the kubelet (on the pod's node) runs the check every `periodSeconds`; a check fails if it returns an error/non-2xx-3xx status or exceeds `timeoutSeconds`. After `failureThreshold` *consecutive* failures the probe is considered failed and the action in the table is taken. `successThreshold` (consecutive passes needed to flip back) must be 1 for liveness and startup.

**Why the startup probe was invented:** before it, people raised `initialDelaySeconds` on liveness to cover the slowest boot, which also meant a deadlocked pod wasn't detected for minutes. The startup probe holds liveness/readiness off *only during boot*; once it succeeds once, the tight liveness settings take over.

**Failure behaviours to state precisely:** a failing readiness probe only changes the pod's `Ready` condition → it is dropped from EndpointSlices (and `READY` shows `0/1`); a failing liveness or startup probe causes the kubelet to kill and restart the container per `restartPolicy`, which you see as `RESTARTS` increasing and an `Unhealthy` event. Readiness runs for the **whole life** of the pod, so a pod can flip in and out of rotation repeatedly — good for temporary overload, e.g. a saturated thread pool.

### Q29. Production-grade probe configuration for Spring Boot.

```yaml
containers:
  - name: order-service
    ports:
      - name: http
        containerPort: 8080
      - name: management
        containerPort: 8081
    startupProbe:
      httpGet: { path: /actuator/health/liveness, port: management }
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 30        # allows up to 150s to boot
    readinessProbe:
      httpGet: { path: /actuator/health/readiness, port: management }
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
    livenessProbe:
      httpGet: { path: /actuator/health/liveness, port: management }
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
```

Spring Boot side:
```yaml
management:
  server:
    port: 8081                      # separate management port: not exposed via Ingress
  endpoint:
    health:
      probes:
        enabled: true               # enables /health/liveness and /health/readiness
      show-details: when-authorized
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**How to read this config (why each part is there):**
- **Separate management port (8081)** — probes and metrics hit a different connector than user traffic, so the Ingress can't expose `/actuator`, and probes still answer when all 200 Tomcat request threads are busy (Scenario 3).
- **`probes.enabled: true`** — Spring Boot auto-enables the liveness/readiness groups on Kubernetes (detected via env vars), but being explicit avoids surprises when running elsewhere.
- **`startupProbe` math:** `failureThreshold × periodSeconds` = 30 × 5 = **150 s** to boot; once it passes, liveness/readiness take over. This replaces a large `initialDelaySeconds`.
- **Readiness `periodSeconds: 5`, `failureThreshold: 3`** — a pod is pulled from rotation after ~15 s of failures; liveness is deliberately slower (30 s) because a restart is a heavier action than removing from rotation.
- **`timeoutSeconds: 3`** — gives headroom for short GC pauses; 1 s is a common cause of spurious restarts.
- **Why `startupProbe` uses the liveness endpoint:** during boot it only asks "has the JVM/context started?" — checking dependencies there would just delay startup for no reason.
- **`server.shutdown: graceful`** isn't a probe setting but belongs in the same config: when the pod is terminating Spring flips readiness to `REFUSING_TRAFFIC` and drains in-flight requests (Q53).

**Verify locally:** `curl localhost:8081/actuator/health/readiness` → `{"status":"UP"}`; during shutdown it returns `OUT_OF_SERVICE`/503.

### Q30. What is the single most important rule about liveness probes? (Very common senior question)

**Never put dependency checks in the liveness probe.**

If your liveness endpoint checks the database and the DB has a 2-minute blip, Kubernetes restarts **every replica of every service** that touches that DB. You turn a dependency hiccup into a cluster-wide crash loop — a self-inflicted outage that also hammers the DB with reconnect storms during recovery.

- **Liveness** → is *this JVM* wedged? (deadlock, unrecoverable state). Should be nearly always-true and cheap.
- **Readiness** → can I serve *now*? Here dependency checks are debatable but *safer* — worst case, you stop taking traffic.

Spring Boot's `/actuator/health/liveness` is correct by default: it reflects only `LivenessState` (the internal `ApplicationAvailability`), **not** the DB/Redis/Kafka indicators. `/actuator/health` (the aggregate) *does* include them — so **never point liveness at `/actuator/health`.** That mistake is extremely common and calling it out scores points.

To include a specific indicator in readiness only:
```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState,db
          additional-path: "server:/readyz"   # expose on the main port too
```

### Q31. How do you control readiness from application code?

```java
@Component
@RequiredArgsConstructor
public class WarmUpListener {

    private final ApplicationEventPublisher publisher;
    private final CacheWarmer cacheWarmer;

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        cacheWarmer.loadReferenceData();     // prime caches, JIT-warm hot paths
        AvailabilityChangeEvent.publish(publisher, this, ReadinessState.ACCEPTING_TRAFFIC);
    }

    // e.g. when a critical downstream is circuit-broken open for too long
    public void degrade() {
        AvailabilityChangeEvent.publish(publisher, this, ReadinessState.REFUSING_TRAFFIC);
    }
}
```
This lets you keep the pod out of the load balancer until caches are warm — which avoids the classic "new pod gets traffic immediately, times out for 20 seconds while JIT compiles and caches fill".

### Q32. Probe types other than HTTP?

- `exec` — runs a command inside the container; exit 0 = pass. Costs a process spawn each period; avoid on tight periods.
- `tcpSocket` — just opens a TCP connection. Useless for Spring Boot: Tomcat binds the port *before* the context is fully ready, so a TCP probe passes while the app still returns 500s.
- `grpc` — native gRPC health checking (1.24+), good for gRPC-only Java services.

**Why probe type matters:** a probe is just "kubelet runs a check on a schedule and looks at the result". The cheaper and more truthful the check, the better. HTTP against Actuator is best for Spring Boot because the response body reflects *application* state, not merely "a socket is open".
```yaml
# exec – e.g. a batch worker with no HTTP port; the command's exit code decides
livenessProbe:
  exec: { command: ["sh", "-c", "test $(( $(date +%s) - $(stat -c %Y /tmp/heartbeat) )) -lt 60"] }
  periodSeconds: 30
# grpc – requires the service to implement the standard grpc.health.v1.Health service
readinessProbe:
  grpc: { port: 9090 }
```
Pitfall: `exec` probes run inside the container, so the binary (`sh`, `curl`) must exist — distroless images have neither, another reason to prefer `httpGet`. Also keep `timeoutSeconds` ≥ the realistic response time; a GC pause longer than the timeout will register as a failed probe.

---

## 9. Resource Management & the JVM in Containers

This section is **the** differentiator for a Java candidate. Interviewers love it.

### Q33. Requests vs limits — what does each actually do?

- **Request** = what the **scheduler** reserves. It's the only number used for placement, and it's the number that matters for HPA percentages.
- **Limit** = the **hard ceiling** enforced by the kernel cgroup at runtime.

| Resource | Over the limit → |
|---|---|
| **CPU** (compressible) | **Throttled** (CFS quota). Latency spikes, no kill |
| **Memory** (incompressible) | **OOMKilled** (SIGKILL, exit 137) |

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "768Mi"
  limits:
    cpu: "2"
    memory: "768Mi"
```

**Units and mechanics:** CPU is measured in cores; `500m` = 500 millicores = half a core. Memory uses binary units: `768Mi` = 768 mebibytes (`768M` would be decimal megabytes — a common typo with small but real differences).

**How it works:**
- **Requests drive scheduling:** the scheduler adds up the requests of pods already on a node and only places your pod where `sum(requests) ≤ node allocatable`. A node can be at 10% *actual* usage and yet "full" because requests reserve the capacity. This is why inflated requests make a cluster look busy while idle (Scenario 17).
- **Limits are enforced by Linux cgroups on the node:** the CPU limit becomes a **CFS quota** (e.g. limit `2` = 200 ms of CPU time per 100 ms period across all threads); the memory limit becomes `memory.max`, and the kernel OOM killer kills the process when RSS exceeds it.
- **CPU requests also set relative weight** (`cpu.shares`/`cpu.weight`): when the node is contended, a pod requesting 2 cores gets twice the CPU share of one requesting 1.

**Example reading:** with the config above the pod is *guaranteed* 0.5 CPU, may burst up to 2 cores if the node has spare, and is killed if it ever uses more than 768 MiB.

**Pitfalls:** a request larger than any node's allocatable capacity keeps the pod `Pending` forever; no memory limit means one leaking pod can exhaust the node and trigger evictions of neighbours; and **limits without requests** makes Kubernetes default the request to the limit.

### Q34. What are QoS classes and why do they matter?

| QoS | Condition | Eviction order under node memory pressure |
|---|---|---|
| **Guaranteed** | requests == limits for **all** containers, both cpu & memory | Evicted **last** |
| **Burstable** | Has requests, but != limits | Middle |
| **BestEffort** | No requests or limits at all | Evicted **first** |

For latency-sensitive Spring Boot services in prod, set **memory request == memory limit** (predictable, no eviction surprises) and give CPU a higher limit than request so you can burst through startup.

🧠 **"Set memory requests = limits. Set CPU request low-ish, limit generous."** Memory is a cliff; CPU is a slope.

**What QoS (Quality of Service) is:** you never set it directly — Kubernetes **derives** the class from the requests/limits you wrote, and uses it to decide *who gets sacrificed first* when a node runs out of memory. Under node memory pressure the kubelet evicts BestEffort pods first, then Burstable pods that exceed their requests, and Guaranteed pods last. (The kernel OOM killer likewise prefers to kill low-QoS processes via `oom_score_adj`.)

**Concrete examples:**
```yaml
# Guaranteed: every container has cpu+memory requests == limits
resources: { requests: { cpu: "1", memory: 1Gi }, limits: { cpu: "1", memory: 1Gi } }
# Burstable: requests set but lower than limits (or only some resources set)
resources: { requests: { cpu: 250m, memory: 512Mi }, limits: { cpu: "2", memory: 1Gi } }
# BestEffort: nothing specified  -> first to be evicted
```
See the assigned class with `kubectl get pod <pod> -o jsonpath='{.status.qosClass}'`.

**Pitfalls:** with a sidecar (Envoy, fluent-bit) *every* container must satisfy the rule or the pod falls to Burstable; a pod using more memory than its *request* is the first Burstable candidate for eviction, so a request far below real usage is risky; and note a pod hitting its **own memory limit** is OOMKilled regardless of QoS — QoS only matters when the *node* is short of memory.

### Q35. Should I set a CPU limit on a JVM app? (Nuanced — say both sides)

**Argument against limits:** CFS throttling on a bursty JVM is brutal. GC threads, JIT compiler threads, and the Netty/Tomcat pool all spike; hitting the quota mid-request adds tens/hundreds of ms of latency. Many teams set CPU **requests only** and rely on requests for fair sharing.

**Argument for limits:** without them a noisy neighbour or a runaway thread can starve co-located pods, and capacity planning gets fuzzy.

**Senior answer:** *"I always set CPU requests. For latency-critical services I set a generous CPU limit (2–4× request) or omit it, and I monitor `container_cpu_cfs_throttled_seconds_total`. If throttling is >5% of periods, the limit is too low. I never set a CPU limit below ~1 core for a JVM app, because the JVM sizes GC and JIT threads off available CPUs and a sub-core quota makes startup pathologically slow."*

### Q36. Why did my Spring Boot pod get OOMKilled when heap usage was only 60%?

**This is the classic Java-on-K8s question.** Because the container limit governs **total RSS**, not heap:

```
Container memory = Heap
                 + Metaspace / CodeCache (JIT)
                 + Thread stacks (~1 MB × threads)
                 + GC structures & card tables
                 + Direct/NIO ByteBuffers (Netty, gRPC, Kafka clients!)
                 + Compressed class space
                 + JVM native / malloc arenas
                 + JVM itself
```
Heap is typically only **60–75%** of total footprint.

**Fixes:**
1. Use `-XX:MaxRAMPercentage` instead of `-Xmx` so it scales with the limit:
   ```yaml
   env:
     - name: JAVA_TOOL_OPTIONS
       value: >-
         -XX:MaxRAMPercentage=75.0
         -XX:InitialRAMPercentage=50.0
         -XX:+UseG1GC
         -XX:MaxMetaspaceSize=256m
         -XX:MaxDirectMemorySize=128m
         -XX:+ExitOnOutOfMemoryError
         -XX:+HeapDumpOnOutOfMemoryError
         -XX:HeapDumpPath=/dumps
   ```
2. Cap the unbounded regions: `MaxMetaspaceSize`, `MaxDirectMemorySize`, `-XX:ActiveProcessorCount` if needed.
3. Turn on **Native Memory Tracking** to find the culprit: `-XX:NativeMemoryTracking=summary` then `jcmd <pid> VM.native_memory summary`.
4. Add `-XX:+ExitOnOutOfMemoryError` so a heap OOM **kills the pod** and K8s replaces it, rather than leaving a zombie JVM thrashing GC and still passing a naive liveness probe.

🧠 **Memory hook: "Heap ≠ RSS."** Budget roughly `limit = heap / 0.75`.

### Q37. Does the JVM understand container limits?

Yes, since **Java 10** (backported to 8u191): `-XX:+UseContainerSupport` is **on by default**. The JVM reads cgroup limits to derive:
- `availableProcessors()` → from CPU quota/shares
- default max heap (`MaxRAMPercentage`, default 25% of the limit — often too low!)

**Danger with Java 8 older than 8u191:** the JVM sees the **host's** memory and CPU. A 512 MiB-limited container on a 64 GB node sizes its heap to ~16 GB and gets OOMKilled instantly, and `availableProcessors()` returns 64 → 64 GC threads, huge thread pools. For a 9-year Java dev, mentioning this is a strong signal. If stuck on old Java 8: `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`.

**Also note:** cgroup **v2** support requires JDK 15+ (backports in 11.0.16+, 17+). Old JDKs on a cgroup-v2 host can misdetect limits again.

Verify inside the pod:
```bash
kubectl exec -it <pod> -- java -XX:+PrintFlagsFinal -version | grep -Ei "maxheapsize|useg1gc|activeprocessor"
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.max
```

### Q38. Which GC for containerized Spring Boot?

| GC | Best for | Notes |
|---|---|---|
| **SerialGC** | < 1 CPU, small heap (<512 MB), sidecars/batch | Lowest overhead & footprint |
| **G1GC** (default ≥ JDK 9, if ≥2 CPU & ≥2 GB) | Typical REST microservice | Balanced pause/throughput; good default answer |
| **ParallelGC** | Batch/throughput jobs, small containers | Higher throughput, longer pauses |
| **ZGC / Shenandoah** | Large heaps, strict p99 latency | Sub-ms pauses, more native memory overhead |

**Watch out:** with < 2 CPUs or < 1792 MB, the JVM silently picks **SerialGC**, which can make a "small" pod perform far worse than expected. Another reason not to set tiny CPU limits.

### Q39. How do you make Spring Boot start fast in Kubernetes?

Because slow startup hurts scaling and rollouts:
1. **Layered JARs + Docker layer caching** (see §18) — faster builds/pulls.
2. **CDS / AppCDS** (JDK 12+, much better in JDK 19+; Spring Boot 3.3 has first-class support) — cuts class-loading time 20–40%.
3. **`-XX:TieredStopAtLevel=1`** for short-lived batch pods (skip C2 compilation).
4. **Lazy initialization** — `spring.main.lazy-initialization=true` (careful: moves failures to first request).
5. **Trim autoconfiguration**, avoid classpath scanning of huge packages.
6. **GraalVM Native Image** (`spring-boot-starter-parent` + `native` profile) — startup in tens of milliseconds, tiny memory. Trade-off: long build times, reflection config, no dynamic proxies without hints. Great for serverless/Knative, scale-to-zero.
7. **Generous `startupProbe.failureThreshold`** instead of a long `initialDelaySeconds`.

**Why startup time matters (the "why"):** when the HPA scales out during a traffic spike, new pods only help once they're Ready — a 90-second boot means 90 seconds of overload. During a rolling update, slow boots stretch the whole rollout, and pods that restart often (spot-node churn) are unavailable longer. Fast startup = faster scaling, faster recovery, cheaper idle capacity.

**How to approach it — measure first:** Spring logs `Started OrderApplication in 38.2 seconds`. Enable `BufferingApplicationStartup` (`app.setApplicationStartup(new BufferingApplicationStartup(2048))`) and expose the `startup` actuator endpoint to see *which beans* take the time before applying tricks blindly. Typical big wins: avoid eager DB schema validation/Flyway on every replica (run migration as an init container/Job, Q11), and reduce classpath scanning.

**Trade-offs to quote:** lazy init hides misconfiguration until the first request (a failed readiness is better than a 500 in production); native image changes how you debug and build; AppCDS adds a build step but is low risk. Mention you would **give the JVM enough CPU at startup** (CPU request, or a higher limit) because JIT/class loading is CPU-bound — a 250m limit can double startup.

### Q40. ResourceQuota and LimitRange.

- **ResourceQuota** (namespace-level cap): total cpu/memory requests+limits, object counts (pods, services, PVCs).
- **LimitRange** (per-pod/container defaults & bounds): default requests/limits when a dev forgets, plus min/max.

**Why both exist:** a Kubernetes cluster is shared. Without guardrails one team's runaway deployment (or a forgotten `replicas: 500`) can consume all CPU/memory and starve everyone else. ResourceQuota limits the *namespace total*; LimitRange governs *each individual container/pod* inside it.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: orders-quota, namespace: orders }
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    persistentvolumeclaims: "20"
```
With this, the 101st pod — or any pod pushing total requested CPU past 20 cores — is **rejected at admission** with a `forbidden: exceeded quota` error. Check usage with `kubectl describe quota -n orders`.

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: orders }
spec:
  limits:
    - type: Container
      default:        { cpu: "500m", memory: "512Mi" }   # applied if not specified
      defaultRequest: { cpu: "200m", memory: "256Mi" }
      max:            { cpu: "4",    memory: "8Gi" }
      min:            { cpu: "50m",  memory: "64Mi" }
```
Gotcha to mention: once a ResourceQuota for cpu/memory exists in a namespace, **every** pod must declare requests/limits or creation is rejected — hence pairing it with a LimitRange.

---

## 10. Storage: PV, PVC, StorageClass

### Q41. Explain PV, PVC, StorageClass.

- **PersistentVolume (PV)** — the actual piece of storage (cluster-scoped resource).
- **PersistentVolumeClaim (PVC)** — a namespaced *request* for storage ("5Gi, RWO, fast-ssd"). Your pod references the PVC.
- **StorageClass** — the template for **dynamic provisioning**; defines the provisioner (CSI driver), parameters (disk type, IOPS), `reclaimPolicy`, and `volumeBindingMode`.

🧠 Analogy: **StorageClass = the catalogue, PVC = the purchase order, PV = the delivered disk.**

**Access modes:**

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | One **node** can mount read-write (EBS, GCE PD) |
| `ReadOnlyMany` (ROX) | Many nodes read-only |
| `ReadWriteMany` (RWX) | Many nodes read-write (NFS, EFS, Azure Files, CephFS) |
| `ReadWriteOncePod` (1.22+) | Exactly **one pod** — true single-writer lock |

**Key interview trap:** RWO is per-**node**, not per-**pod**. Two pods on the *same* node can share an RWO volume. And RWO + `RollingUpdate` on a Deployment can deadlock: the new pod can't mount the volume (it's on another node) while the old pod holds it → use `Recreate` or a StatefulSet.

`volumeBindingMode: WaitForFirstConsumer` is important in multi-AZ clusters: it delays PV creation until the pod is scheduled, so the disk is created in the right AZ. Otherwise you get "volume node affinity conflict".

`reclaimPolicy`: `Delete` (disk destroyed with PVC — fine for caches) vs `Retain` (disk survives — use for anything you'd cry about losing).

### Q42. StatefulSet volumeClaimTemplates.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka-headless
  replicas: 3
  podManagementPolicy: OrderedReady   # or Parallel
  selector: { matchLabels: { app: kafka } }
  template:
    metadata: { labels: { app: kafka } }
    spec:
      terminationGracePeriodSeconds: 120
      containers:
        - name: kafka
          image: bitnami/kafka:3.7
          volumeMounts:
            - { name: data, mountPath: /bitnami/kafka }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources: { requests: { storage: 100Gi } }
```
Each replica gets its own PVC: `data-kafka-0`, `data-kafka-1`… and if `kafka-1` is rescheduled it **reattaches the same PVC**. Deleting the StatefulSet does **not** delete the PVCs (by design; `persistentVolumeClaimRetentionPolicy` in 1.27+ lets you change that).

**Why it exists:** in a Deployment, all replicas would reference one shared PVC — fine for read-only data, wrong for a database/broker where each node owns *different* data. `volumeClaimTemplates` is a stamp: for every replica the StatefulSet controller creates a matching PVC (name = `<template>-<statefulset>-<ordinal>`), so pod identity and disk identity stay bound together across restarts and node moves.

**What the other fields do:**
- `serviceName: kafka-headless` — points to a headless Service (Q19) that gives each pod a stable DNS name such as `kafka-1.kafka-headless.<ns>.svc.cluster.local`, which peers use to find each other.
- `podManagementPolicy: OrderedReady` — start `-0`, wait until Ready, then `-1`, then `-2` (needed when a node must join an existing cluster in order); `Parallel` starts all at once for faster startup when pods are independent.
- `terminationGracePeriodSeconds: 120` — gives a broker time to flush data and hand over leadership before SIGKILL.

**Pitfall:** `volumeClaimTemplates` can't be changed on an existing StatefulSet — resizing means editing each PVC (if the StorageClass has `allowVolumeExpansion: true`) or recreating the set while keeping the PVCs (`kubectl delete sts kafka --cascade=orphan`). Interview probe: *"What happens to the data if a node dies?"* → with network disks (EBS/PD) the PVC reattaches on another node in the **same AZ**; with local disks the data is gone with the node, so replication at the application level (Kafka replication factor 3) is your real protection.

### Q43. Volume types a Spring Boot dev actually uses.

| Type | Use |
|---|---|
| `emptyDir` | Scratch space, temp files, heap dump target, shared dir with a sidecar. Dies with the pod. `medium: Memory` = tmpfs (counts against memory limit!) |
| `configMap` / `secret` | Config & credential files |
| `persistentVolumeClaim` | Real durable storage |
| `projected` | Combine SA token + ConfigMap + Secret into one dir |
| `downwardAPI` | Expose pod name/namespace/labels as files or env |
| `hostPath` | Node filesystem — avoid in app pods (security/portability); fine for DaemonSet agents |

Practical Spring Boot pattern — write heap dumps to an emptyDir so you can grab them before the pod dies:
```yaml
volumes:
  - name: dumps
    emptyDir: { sizeLimit: 2Gi }
volumeMounts:
  - { name: dumps, mountPath: /dumps }
```
Then `kubectl cp <pod>:/dumps/java_pid1.hprof ./local.hprof`.

**Why a volume at all?** A container's own filesystem is a throwaway layer — it vanishes when the container restarts, and an OOMKilled container restarts in seconds, taking any file it wrote with it. A volume mounted into the pod outlives *container* restarts (an `emptyDir` still dies with the *pod*; a PVC outlives the pod). Choosing the type = choosing how long the data must survive and who may see it.

**Related pitfalls:**
- `emptyDir` with `medium: Memory` is RAM — the data counts toward the container memory limit, so a big `/tmp` can OOMKill your app.
- With `readOnlyRootFilesystem: true` (Q58) Spring Boot/Tomcat still need a writable `/tmp`; mount an `emptyDir` there or the app fails at startup with "Read-only file system".
- Heap dumps can be GBs and contain secrets in memory; set `sizeLimit` and restrict who can `kubectl cp`/`exec`.

### Q44. Downward API — how do you get the pod name into logs?

```yaml
env:
  - name: POD_NAME
    valueFrom: { fieldRef: { fieldPath: metadata.name } }
  - name: POD_NAMESPACE
    valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
  - name: NODE_NAME
    valueFrom: { fieldRef: { fieldPath: spec.nodeName } }
  - name: POD_IP
    valueFrom: { fieldRef: { fieldPath: status.podIP } }
  - name: CPU_LIMIT
    valueFrom: { resourceFieldRef: { containerName: app, resource: limits.cpu } }
```
Then in `logback-spring.xml` include `${POD_NAME}` in the pattern or as an MDC value, so a log line tells you exactly which replica produced it. Also good for `spring.application.instance-id` and Micrometer common tags.

**Why it exists:** a container normally doesn't know anything about the pod it runs in (its name, IP, node, resource limits) and shouldn't call the Kubernetes API just to find out. The **Downward API** lets the kubelet inject that metadata, either as env vars (`fieldRef`) or as files in a volume. No RBAC or client library required.

**Example of using it in Spring Boot:**
```yaml
# application.yaml
spring:
  application:
    instance-id: ${POD_NAME:local}
management:
  metrics:
    tags:
      pod: ${POD_NAME:local}
      namespace: ${POD_NAMESPACE:local}
```
```xml
<!-- logback-spring.xml -->
<pattern>%d %-5level [${POD_NAME:-local}] %logger{36} - %msg%n</pattern>
```
**Pitfalls:** env vars from the Downward API are fixed at container start (labels/annotations only update live when exposed as a **volume**); and `resourceFieldRef` for `limits.cpu` is rounded up to a whole number by default, which is useful for sizing thread pools but not exact.

---

## 11. Scaling: HPA, VPA, KEDA, Cluster Autoscaler

### Q45. How does HPA work?

The HPA controller polls metrics every ~15s and computes:

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / targetMetricValue ) )
```

Crucially, the **CPU percentage is relative to the request, not the limit.** If `request: 500m` and pods use `750m`, utilization is **150%**. Getting this right in an interview is a strong signal.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: order-service }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Pods                          # custom metric via Prometheus Adapter
      pods:
        metric: { name: http_server_requests_seconds_count }
        target: { type: AverageValue, averageValue: "50" }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - { type: Percent, value: 100, periodSeconds: 30 }   # can double every 30s
    scaleDown:
      stabilizationWindowSeconds: 300                        # wait 5 min before shrinking
      policies:
        - { type: Pods, value: 1, periodSeconds: 60 }        # shrink slowly
```

Prerequisites: **metrics-server** installed, and **resource requests defined** (no request = no CPU utilization = HPA can't work — a very common bug).

**Java-specific caveat:** CPU-based HPA is noisy for JVMs because JIT compilation spikes CPU at startup. A new pod shows 100% CPU while warming up, which can trigger *more* scale-up → a feedback loop. Mitigations: `behavior.scaleUp` policies, HPA on request-rate or queue depth instead of CPU, and `stabilizationWindowSeconds`.

### Q46. HPA vs VPA vs Cluster Autoscaler vs KEDA.

| Tool | Scales | Notes |
|---|---|---|
| **HPA** | Number of pods (horizontal) | The default for stateless Spring Boot |
| **VPA** | CPU/memory requests of pods (vertical) | Recreates pods to apply; **conflicts with HPA on the same metric**. Use in `recommendation` mode for right-sizing |
| **Cluster Autoscaler / Karpenter** | Number of **nodes** | Reacts to `Pending` pods that don't fit |
| **KEDA** | Pods, incl. **scale-to-zero** | Event-driven: Kafka lag, RabbitMQ depth, SQS length, cron. Ideal for Spring Boot consumers |

**KEDA for a Kafka consumer** — the answer that impresses:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: order-consumer }
spec:
  scaleTargetRef: { name: order-consumer }
  minReplicaCount: 1
  maxReplicaCount: 10          # never exceed partition count — extra pods idle
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-group
        topic: orders
        lagThreshold: "1000"
```
Add: *"I cap max replicas at the partition count, because a Kafka consumer group can't have more active consumers than partitions."*

**Why four tools — they scale different things:** the HPA adds *pods* when load rises; but pods need room, so when no node has capacity the pods go `Pending` and the Cluster Autoscaler (or Karpenter) adds a *node*. The VPA changes how *big* a pod is (right-sizing requests). KEDA is an HPA on steroids that reacts to *events* instead of CPU.

**Chain for a traffic spike:** load ↑ → HPA creates pods → no room → Pending → Cluster Autoscaler provisions a node (1–3 minutes) → pods schedule → image pull → Spring Boot starts → Ready. That latency chain is why scaling is never instantaneous (Scenario 9).

**Why KEDA suits consumers:** CPU is a poor signal for a queue worker — a consumer can be idle-CPU while a million messages pile up. KEDA reads the actual backlog (Kafka consumer-group lag) and creates the HPA for you, including scaling to **zero** when the queue is empty and back up when messages arrive. `lagThreshold: 1000` means "aim for roughly one replica per 1000 unprocessed messages".

**Rules of thumb:** don't run HPA and VPA on the *same* metric (CPU) — they fight; use VPA in `Off`/recommendation mode to learn right-sized requests and let HPA do the scaling. Karpenter vs Cluster Autoscaler: both add nodes, but Karpenter picks instance types per pending pod and is generally faster.

### Q47. What is a PodDisruptionBudget and why do you need it?

A PDB constrains **voluntary** disruptions (node drain, cluster upgrade, autoscaler consolidation) so an operator can't take down all your replicas at once.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: order-service-pdb }
spec:
  minAvailable: 2            # or maxUnavailable: 1
  selector:
    matchLabels: { app: order-service }
```
Points to make:
- PDBs **do not** protect against involuntary disruption (node hardware failure, OOMKill).
- `minAvailable: 100%` or a PDB that can never be satisfied will **block node drains forever** — a real operational trap.
- Pair PDB (protects during drains) with `topologySpreadConstraints` (spreads across zones) with `maxUnavailable` (protects during rollouts). Together = a genuinely resilient deployment.

**Why it's needed (scenario):** the platform team drains a node for an upgrade. `kubectl drain` *evicts* pods using the Eviction API, which checks PDBs. Without a PDB, if 3 of your 3 `order-service` replicas happen to sit on that node (or several nodes are drained in quick succession), all of them are evicted together → outage. With `minAvailable: 2`, the eviction of the second pod is **refused (HTTP 429) until a replacement elsewhere is Ready**, so the drain proceeds one pod at a time.

**How to choose:** `minAvailable` says "at least N must stay up"; `maxUnavailable` says "at most N may be down" (scales better with HPA because it can be a percentage). Check status with `kubectl get pdb` — the `ALLOWED DISRUPTIONS` column is the live number of pods that may currently be evicted; `0` means drains will wait.

**Common pitfalls:** a PDB on a single-replica Deployment with `minAvailable: 1` blocks node drains forever; and a PDB doesn't help during a Deployment rolling update (that's governed by `maxUnavailable` in the strategy, Q16).

---

## 12. Deployment Strategies & Zero-Downtime Releases

### Q48. Compare deployment strategies.

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Rolling update** | Replace pods gradually | Built-in, no extra infra, no extra cost | Both versions live at once → needs backward compatibility |
| **Recreate** | Kill all, then start new | No version overlap | Downtime |
| **Blue/Green** | Two full stacks, flip the Service selector | Instant switch, instant rollback | 2× resources |
| **Canary** | Send x% of traffic to new version | Real-user validation, small blast radius | Needs traffic splitting + good metrics |
| **A/B** | Route by header/cookie/user segment | Product experimentation | Most complex |
| **Shadow/mirror** | Duplicate prod traffic to new version, discard responses | Zero user risk | Side effects must be handled |

**Plain-language notes on each:**
- **Rolling update** is the Kubernetes default — swap pods a few at a time, so users never see a full outage but for a while v1 and v2 serve requests *simultaneously* (so API/DB changes must be backward compatible, Q51).
- **Blue/Green** keeps the current version ("blue") untouched while the new version ("green") is fully deployed and tested beside it; releasing = flipping traffic in one step, rolling back = flipping back. You pay by running double the pods.
- **Canary** (named after the coal-mine canary) sends a small slice, e.g. 5%, to the new version, watches error rate/latency, and widens it step by step; Argo Rollouts/Flagger can automate promotion or abort on metrics.
- **A/B** is a business experiment rather than a safety mechanism — which *users* see which version is decided by header, cookie or segment.
- **Shadow** copies live requests to the new version and throws its responses away; great for validating a rewrite under real load, dangerous if the new version writes to a DB or calls payment APIs (it would execute those side effects twice).

**How to pick (a 20-second answer):** *"Rolling by default; blue/green when I need instant rollback for a risky release and can afford double capacity; canary when I have good metrics and a mesh/ingress that can split traffic."*

### Q49. How do you do blue/green with plain Kubernetes?

Two Deployments with a `version` label, one Service whose selector you flip:
```yaml
# Service — the switch
spec:
  selector:
    app: order-service
    version: blue          # patch to "green" to cut over
```
```bash
kubectl patch service order-service -p '{"spec":{"selector":{"version":"green"}}}'
# rollback is the same command with "blue"
```
Caveats to mention: in-flight requests on blue, DB schema must serve both, and long-lived connections don't move until reconnected.

**Why this works:** a Service sends traffic to pods whose labels match its `selector`. Both Deployments (`order-service-blue`, `order-service-green`) keep running; only the Service's selector decides which set is "live". Changing it re-computes the EndpointSlice and kube-proxy updates the rules within seconds, so the cut-over is effectively instant and needs no pod restarts.

**Step-by-step:**
1. Deploy `order-service-green` (new image, label `version: green`). The live Service still points to blue, so nobody is affected.
2. Smoke-test green privately — e.g. via a second Service/Ingress (`order-service-green-preview`) or `kubectl port-forward`.
3. Flip the selector (`kubectl patch` above). Watch error rate and latency.
4. Keep blue running for a while for instant rollback; scale it down or delete it once confident.

**Pitfalls:** make sure the readiness probe is passing on green *before* the flip (otherwise the Service has no endpoints and you cause an outage); watch out for connection draining on blue (Q52); and two Deployments consume double the resources while both exist. Tools like Argo Rollouts automate exactly this pattern.

### Q50. How do you do a canary?

**Poor-man's canary (no extra tooling):** two Deployments behind one Service; 9 stable pods + 1 canary pod ≈ 10% traffic. Crude (granularity tied to replica count) but works.

**NGINX Ingress canary:**
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
    # or: canary-by-header: "X-Canary"
```

**Proper way — Argo Rollouts / Flagger + service mesh:** declarative progressive delivery with automated analysis and rollback:
```yaml
kind: Rollout
spec:
  strategy:
    canary:
      canaryService: order-canary
      stableService: order-stable
      analysis:
        templates: [{ templateName: success-rate }]
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```
The **analysis template** queries Prometheus (e.g. Micrometer's `http_server_requests_seconds_count{status=~"5.."}` ratio, p99 latency) and auto-aborts if the canary degrades. Mentioning automated metric-driven rollback is a senior-level answer.

**Why the plain approaches are limited:** a Service load-balances roughly evenly across all matching pods, so with replica-based canaries "10%" is only an approximation, and you can't say "send only internal testers" or "exactly 1%". The NGINX annotations and Argo Rollouts/service meshes (Istio, Linkerd) split at the **L7 proxy**, so the percentage is exact and independent of pod count.

**How the Argo Rollouts flow reads (step by step):** (1) a new image creates the canary ReplicaSet; (2) `setWeight: 5` routes 5% of traffic to it; (3) `pause` lets metrics accumulate; (4) the analysis runs queries — success moves to the next weight, failure triggers an **automatic rollback** to the stable version; (5) 100% = promotion.

**What to watch in a canary:** error rate (5xx), p95/p99 latency, JVM health (GC pauses, heap), and business metrics (orders placed per minute) — comparing canary versus stable, not versus yesterday. **Pitfall:** a canary that receives too little traffic (5% at night) gives statistically meaningless data; also sticky sessions or long-lived connections can pin users to one version.

### Q51. Backward compatibility rules for zero-downtime with a database.

Because rolling updates run **two versions simultaneously**, you must use the **expand/contract (parallel change)** pattern:

1. **Expand** — add the new nullable column / new topic / new endpoint. Deploy. Old code ignores it.
2. **Migrate** — new code writes **both** old and new; backfill existing rows.
3. **Switch** — new code reads from the new field.
4. **Contract** — after all instances are on the new version, drop the old column.

That's **3–4 separate releases** for one rename. Never in one deploy:
- ❌ `ALTER TABLE ... DROP COLUMN` while old pods still select it
- ❌ `NOT NULL` column without a default
- ❌ Renaming a column
- ❌ Removing a JSON field consumers still read
- ❌ Changing a Kafka message schema incompatibly (use Schema Registry with `BACKWARD` compatibility)

🧠 **"Two versions, one database — always."**

**Why this matters:** during a rolling update (or canary), pods running version N and N+1 hit the *same* schema simultaneously for minutes. A schema change that is fine for the new code can crash the old code (or vice versa), and a rollback re-introduces the old code against the *new* schema. So every change must work for both neighbouring versions.

**Worked example — rename `customer_name` to `full_name`:**
```sql
-- Release 1 (expand): add new column, nullable; v1 code ignores it
ALTER TABLE orders ADD COLUMN full_name VARCHAR(200);
-- Release 2 (migrate): app writes BOTH columns; backfill old rows in batches
UPDATE orders SET full_name = customer_name WHERE full_name IS NULL;   -- run in chunks, not one huge lock
-- Release 3 (switch): app reads full_name only (still dual-writes in case of rollback)
-- Release 4 (contract): once no v1 pod exists and rollback window has passed
ALTER TABLE orders DROP COLUMN customer_name;
```
With Flyway/Liquibase each step is a separate migration shipped with a separate release. **Where migrations run** matters too: use a Job/init container (Q11) so replicas don't race. Same idea for APIs (only *add* fields, deprecate before removing) and Kafka events (Avro/Protobuf with backward-compatible schema evolution).

---

## 13. Graceful Shutdown (the #1 Spring Boot + K8s Question)

### Q52. Describe exactly what happens when a pod is deleted, and why requests still fail.

**The sequence:**
1. Pod marked `Terminating`; `deletionTimestamp` set.
2. **In parallel** (this is the crux):
   - **(a)** kubelet runs the `preStop` hook, then sends **SIGTERM** to PID 1.
   - **(b)** The endpoint controller removes the pod from EndpointSlices → kube-proxy on **every node** must then rewrite iptables, and the Ingress controller must refresh its upstreams.
3. After `terminationGracePeriodSeconds` (default **30s**), kubelet sends **SIGKILL** (exit 137).

**Why you get 502/504s:** step 2 is **not ordered**. Endpoint propagation is *eventually* consistent and takes hundreds of milliseconds to several seconds. If your app shuts down instantly on SIGTERM, there's a window where kube-proxy/Ingress still routes traffic to a dead pod.

**The fix — a `preStop` sleep:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 60
```
The pod keeps serving during the sleep while endpoints propagate, *then* receives SIGTERM. This is the single most valuable practical answer in this whole guide.

*(K8s 1.29+ has a native `sleep` preStop handler: `preStop: { sleep: { seconds: 15 } }` — no shell needed, works with distroless images.)*

### Q53. Full graceful-shutdown configuration for Spring Boot.

```yaml
# application.yaml
server:
  shutdown: graceful                 # Boot 2.3+: stop accepting new, finish in-flight
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # max wait for in-flight requests
```

```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60   # MUST be > preStop sleep + shutdown timeout
      containers:
        - name: order-service
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 15"] }
```

**The budget must add up:** `preStop (15s) + graceful shutdown (30s) = 45s < terminationGracePeriodSeconds (60s)`. If the grace period is smaller, you get SIGKILL mid-request and lose in-flight work.

🧠 **Memory hook: "SLEEP then STOP, and the grace period must cover both."**

### Q54. What else must shut down cleanly in a Spring Boot app?

```java
@Component
public class ShutdownHooks {

    // 1. Kafka: Spring's container stops polling and commits offsets on context close.
    //    Ensure max.poll.interval / processing time fits inside the grace period.

    // 2. Scheduled executors: let tasks finish
    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setWaitForTasksToCompleteOnShutdown(true);
        ex.setAwaitTerminationSeconds(20);
        return ex;
    }

    // 3. Register the JVM hook (Spring does this by default; explicit here for clarity)
    @PreDestroy
    public void onShutdown() {
        log.info("SIGTERM received: draining");
        // deregister from external discovery, flush metrics/buffers, close clients
    }
}
```

Checklist: in-flight HTTP requests ✔, Kafka/RabbitMQ consumers stop polling and commit ✔, scheduled/async tasks drained ✔, DB connection pool closed ✔, distributed locks released ✔, final metrics scraped ✔.

**Why it's needed:** `server.shutdown=graceful` only covers the *web server's* in-flight HTTP requests. Everything else with its own threads — Kafka listeners, `@Async`/`@Scheduled` work, a hand-made `ExecutorService` — has to be told to stop, otherwise SIGKILL cuts it mid-task: a half-processed Kafka record is redelivered (duplicate processing), a half-finished batch leaves partial state, a lock held in Redis/DB stays until its TTL.

**How Spring does it:** on SIGTERM the JVM runs its shutdown hook, which closes the `ApplicationContext`. Beans are destroyed in **reverse dependency order** and Lifecycle/`SmartLifecycle` beans are stopped by *phase* — web server and message listener containers first, then your `@PreDestroy` methods, then the `DataSource` last (so tasks that still need the DB can finish). `spring.lifecycle.timeout-per-shutdown-phase` bounds each phase.

**Pitfalls:** consumers must be **idempotent** anyway (a pod can be SIGKILLed); a `ThreadPoolTaskExecutor` without `setWaitForTasksToCompleteOnShutdown(true)` drops queued tasks; and the total (preStop + shutdown phases + `awaitTermination`) must fit in `terminationGracePeriodSeconds` (Q53). Interview probe: *"How do you avoid duplicate Kafka processing on rebalance during shutdown?"* → commit offsets after processing, use idempotent handlers (dedupe key) and, for stricter guarantees, transactions/exactly-once semantics.

### Q55. Why must SIGTERM reach the JVM, and how can it fail to?

If your entrypoint is a **shell wrapper**, the shell becomes PID 1 and may not forward signals to the java child. Then SIGTERM is swallowed, no graceful shutdown happens, and you always get SIGKILL at 30s.

❌ Wrong:
```dockerfile
ENTRYPOINT ["sh", "-c", "java -jar /app.jar"]
```
✅ Right (exec form, JVM is PID 1):
```dockerfile
ENTRYPOINT ["java", "-jar", "/app.jar"]
# or, when you need a wrapper script, use `exec java ...` inside it
```
Verify: `kubectl exec <pod> -- ps -ef | head` — java should be PID 1. Alternatively `shareProcessNamespace` / `tini` as an init to reap zombies.

**Background (why PID 1 is special):** the container's first process is PID 1 inside its namespace, and Kubernetes/the runtime delivers SIGTERM **to that process only**. On Linux, PID 1 gets *no default signal handlers* — a normal program that doesn't explicitly handle SIGTERM would be ignored — and a plain shell (`sh -c "java ..."`) doesn't forward signals to its child. The JVM does install a SIGTERM handler (it runs shutdown hooks → Spring closes the context), but only if it actually receives the signal.

**Symptom to recognise:** a pod takes exactly 30s (the grace period) to terminate and ends with exit code 137, and the logs never show Spring's "Closing ApplicationContext" / graceful-shutdown messages.

**Fixes:**
```dockerfile
# 1. exec form — java is PID 1
ENTRYPOINT ["java", "-jar", "/app.jar"]
# 2. wrapper script that needs env expansion: use `exec` so the shell is *replaced* by java
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app.jar"]
# 3. Buildpacks / Jib images already launch java directly — no change needed
```
`tini`/`dumb-init` (or `docker run --init`) are tiny init programs used as PID 1 that forward signals and reap zombie child processes — useful when you spawn subprocesses. Note that with `sh -c "exec java ..."` the JVM still becomes PID 1, so the signal reaches it.

---

## 14. Security: RBAC, ServiceAccounts, Pod Security

### Q56. Explain RBAC in Kubernetes.

Four objects, two axes:

| | Namespaced | Cluster-wide |
|---|---|---|
| **Permissions** | `Role` | `ClusterRole` |
| **Grant** | `RoleBinding` | `ClusterRoleBinding` |

Subjects: `User`, `Group`, `ServiceAccount`.

🧠 **Role = what you can do. Binding = who can do it.** A `RoleBinding` can reference a `ClusterRole` — that's the common trick to reuse one permission set across many namespaces without granting cluster-wide scope.

Least-privilege example for a Spring Cloud Kubernetes app:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: order-service, namespace: orders }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: config-reader, namespace: orders }
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: order-service-config-reader, namespace: orders }
subjects:
  - kind: ServiceAccount
    name: order-service
    namespace: orders
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```
Then `spec.serviceAccountName: order-service` in the pod.

Facts worth knowing: RBAC is **allow-only** (no deny rules); permissions are purely **additive**; requests are denied unless some rule allows them.

**Why RBAC (Role-Based Access Control) exists:** every call to the API server — from `kubectl`, CI, or a pod — must be allowed. Without RBAC a compromised pod with a mounted token could read every Secret or delete Deployments. RBAC says *which identity may perform which verb on which resource*.

**How to read a rule:** `apiGroups: [""]` is the core API group (pods, configmaps, secrets, services); `apps` is for deployments/statefulsets, `batch` for jobs. `resources` are the object types, `verbs` the actions: `get` (one object), `list` (all), `watch` (stream changes — needed by Spring Cloud Kubernetes reload), `create`, `update`, `patch`, `delete`. You can restrict to specific objects with `resourceNames`.

**Role vs ClusterRole, binding scopes:**
- `Role` + `RoleBinding` → permissions in **one namespace**.
- `ClusterRole` + `ClusterRoleBinding` → cluster-wide (also needed for non-namespaced things like nodes, PVs, namespaces).
- `ClusterRole` + `RoleBinding` → reuse the same permission set but only inside the binding's namespace (the built-in `view`/`edit`/`admin` ClusterRoles are used this way).

**Pitfalls:** granting `*` verbs or `cluster-admin` "to make it work" is the classic anti-pattern; `secrets` with `list`/`watch` effectively exposes every secret's contents in the namespace; and `pods/exec` is a separate resource — whoever can exec into prod pods can read their environment and files, so restrict it.

Debug with:
```bash
kubectl auth can-i list secrets --as=system:serviceaccount:orders:order-service -n orders
kubectl auth can-i --list --as=system:serviceaccount:orders:order-service -n orders
```

### Q57. ServiceAccount best practices.

- Never use the `default` ServiceAccount for real workloads; create one per app.
- Disable token mounting if the app doesn't call the API:
  ```yaml
  spec:
    serviceAccountName: order-service
    automountServiceAccountToken: false
  ```
- Modern tokens are **short-lived, audience-bound projected tokens** (1.24+ stopped auto-creating non-expiring Secret tokens).
- **Cloud IAM federation** — the pattern to name-drop: **IRSA** on EKS, **Workload Identity** on GKE, **Azure AD Workload Identity** on AKS. The SA token is exchanged for cloud credentials, so your Spring Boot app hits S3/SQS/Secrets Manager with **no static keys anywhere**.

**What a ServiceAccount is, in plain terms:** a *pod's identity* when it talks to the Kubernetes API (users are for humans, ServiceAccounts for workloads). Pods that don't specify one run as the namespace's `default` SA — shared by everything, so any permission granted to it leaks to every app. Hence one SA per app, with only the RBAC it needs (Q56).

**Why disable token mounting:** by default the kubelet mounts a token at `/var/run/secrets/kubernetes.io/serviceaccount/token`. If an attacker gets remote code execution in a vulnerable Spring app, that token lets them call the API with the SA's rights. An ordinary REST service doesn't need it, so don't give it one.

**How workload identity (IRSA etc.) works, step by step:**
1. The SA is annotated with a cloud role, e.g. on EKS `eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/order-service-s3`.
2. The pod gets a short-lived, audience-bound OIDC token (projected volume) signed by the cluster.
3. The AWS SDK in the app (Java SDK v2's default credential chain does this automatically) sends that token to AWS STS; the cloud trusts the cluster's OIDC issuer and returns temporary credentials.
4. Credentials expire and are refreshed automatically — nothing to rotate, nothing in Git.

**Pitfall:** old SDK versions that don't support web-identity credentials fall back to the node's instance role (too broad) or fail — check the SDK version.

### Q58. Pod securityContext — what should a hardened Spring Boot pod look like?

```yaml
spec:
  securityContext:                    # pod-level
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001                    # makes mounted volumes group-writable
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: order-service
      securityContext:                # container-level (wins on conflict)
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
        capabilities:
          drop: ["ALL"]
      volumeMounts:                   # readOnlyRootFilesystem needs writable temp dirs
        - { name: tmp,   mountPath: /tmp }
        - { name: dumps, mountPath: /dumps }
  volumes:
    - { name: tmp,   emptyDir: {} }
    - { name: dumps, emptyDir: { sizeLimit: 2Gi } }
```
**Spring Boot gotcha with `readOnlyRootFilesystem: true`:** Tomcat needs a writable temp dir for multipart uploads and its work directory. Mount an emptyDir at `/tmp` and set `server.tomcat.basedir=/tmp` — otherwise the app fails at startup or on first upload. Same for heap dumps and any file-based cache.

**What each setting defends against (the "why"):**
- **`runAsNonRoot` / `runAsUser`** — by default many images run as root (UID 0). If an attacker exploits a vulnerability in your app they'd be root inside the container, which makes breakouts and writing to mounted volumes much easier. Running as UID 10001 limits the damage; the image must also have a matching `USER` (Q71).
- **`fsGroup`** — Kubernetes sets this group on mounted volumes, so a non-root user can write to a PVC/emptyDir; without it you hit `Permission denied` on volume paths.
- **`seccompProfile: RuntimeDefault`** — seccomp filters which Linux system calls the container may make; the default profile blocks dozens of dangerous ones.
- **`allowPrivilegeEscalation: false`** — stops processes from gaining more privileges than their parent (e.g. via setuid binaries).
- **`readOnlyRootFilesystem: true`** — malware can't drop or modify binaries; only explicit writable mounts are writable.
- **`capabilities: drop: ["ALL"]`** — Linux splits root powers into "capabilities" (bind low ports, change file owners, raw sockets…). A normal REST service needs none; because the app listens on **8080** (>1024) it needs no `NET_BIND_SERVICE`.
- **`privileged: false`** — a privileged container has near-host-level access; never needed for an app pod.

**Pod-level vs container-level:** pod-level settings apply to all containers (incl. init containers); container-level ones override on conflict. Some fields exist only at one level (`fsGroup` = pod; `readOnlyRootFilesystem`/`capabilities` = container).

**Quick verification:** `kubectl exec <pod> -- id` should show `uid=10001`, and `kubectl exec <pod> -- touch /test` should fail with "Read-only file system".

### Q59. What replaced PodSecurityPolicy?

PSP was deprecated in 1.21 and **removed in 1.25**. Replacements:
1. **Pod Security Admission (PSA)** — built-in, namespace labels selecting a profile (`privileged` / `baseline` / `restricted`) and a mode (`enforce` / `audit` / `warn`):
   ```bash
   kubectl label ns orders \
     pod-security.kubernetes.io/enforce=restricted \
     pod-security.kubernetes.io/warn=restricted
   ```
2. **Policy engines** — OPA/Gatekeeper or **Kyverno** for anything custom ("all images must come from our registry", "every pod must have resource limits", "no `:latest` tags", "PDB required").

**Why PSP was replaced:** PodSecurityPolicy was confusing (it only took effect when you bound users to a policy through RBAC; mistakes silently allowed or blocked pods) so it was dropped in favour of something simple.

**What the three PSA profiles mean:**
- `privileged` — no restrictions (system components such as CNI/monitoring agents).
- `baseline` — blocks known privilege escalations: no `privileged` containers, no `hostNetwork`/`hostPID`, restricted host-path volumes.
- `restricted` — current hardening best practice: must run as non-root, `allowPrivilegeEscalation: false`, drop all capabilities, a `RuntimeDefault` seccomp profile. This matches the hardened pod in Q58.

**Modes:** `enforce` rejects violating pods, `warn` lets them in but shows a warning to the user, `audit` writes an audit-log entry. Rollout tip: label namespaces with `warn`/`audit` first, fix the workloads, *then* switch on `enforce`. Common Spring Boot failure when moving to `restricted`: *"container has runAsNonRoot and image will run as root"* — fix the Dockerfile with a numeric `USER` (Q58/§18).

PSA only checks pod fields; it can't express org-specific rules like "images only from our registry", which is why policy engines (Kyverno/Gatekeeper) are used alongside it.

### Q60. How do you keep secrets out of Git?

State the options and pick one:
- **External Secrets Operator** — a `SecretStore` + `ExternalSecret` CR pulls from Vault/AWS Secrets Manager/GCP SM and *materializes* a K8s Secret. Supports periodic refresh. **My usual recommendation.**
- **Secrets Store CSI Driver** — mounts secrets as files directly from the provider; the secret needn't exist as a K8s object at all.
- **Sealed Secrets (Bitnami)** — encrypt with a cluster public key; the encrypted `SealedSecret` is safe in Git. Good, simple GitOps fit.
- **SOPS + age/KMS** — encrypt YAML values, decrypt in CI or by Flux.
- **Vault Agent Injector** — sidecar renders secrets into a shared volume, supports dynamic DB credentials with short TTLs.

**Why this question exists:** a Secret manifest is only base64, so committing it to Git exposes the password to everyone with repo access *and* to the full commit history forever. But GitOps wants everything declared in Git — the options above resolve that tension in two ways: **(a)** keep only an *encrypted* form or a *reference* in Git (Sealed Secrets, SOPS, ExternalSecret), or **(b)** keep the real value in an external store and fetch at runtime (CSI driver, Vault agent).

**How External Secrets Operator looks in practice** — Git holds only a pointer; the operator creates the real Secret in-cluster and refreshes it:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-credentials, namespace: orders }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secrets-manager, kind: ClusterSecretStore }
  target: { name: db-credentials }          # the K8s Secret that will be created
  data:
    - secretKey: password
      remoteRef: { key: prod/orders/db, property: password }
```
**Trade-offs:** Sealed Secrets = simplest, but secrets are tied to one cluster key (disaster recovery needs the key backed up) and rotation means re-sealing. ESO/CSI = central rotation and audit in the vault, but the cluster now depends on the vault being available. Vault dynamic credentials = best security (unique short-lived DB user per pod), most operational effort. Also remember: once a secret lands in a K8s Secret, protect it with RBAC and etcd encryption (Q8, Q24), and if a secret *was* ever committed, **rotate it** — deleting the commit isn't enough.

Add the rotation angle: *"With dynamic Vault DB credentials, HikariCP needs to handle credential rotation — either a short `maxLifetime` on connections or a `DataSource` that re-reads credentials, otherwise the pool holds connections authenticated with an expired lease."* That's a genuinely senior detail.

### Q61. Other security practices you'd list quickly.

- Scan images in CI (Trivy/Grype/Snyk) and **fail the build** on criticals.
- Sign images (Cosign) + admission policy requiring signatures.
- Pin by **digest**, not tag, in prod.
- Non-root, distroless/JRE-slim, minimal base = smaller CVE surface.
- NetworkPolicies default-deny.
- Audit logging on the API server; alert on `exec` into prod pods.
- Never mount the Docker socket; never `hostNetwork`/`hostPID` for app pods.
- mTLS between services via a service mesh (Istio/Linkerd) if compliance requires it.

**Why each one matters (one-liners you can expand on):**
- **Image scanning** compares the packages/JARs in your image against CVE databases; run it in CI (`trivy image --exit-code 1 --severity CRITICAL myimage:tag`) so a vulnerable Log4j-style dependency never reaches the cluster. Re-scan regularly — new CVEs appear after you've shipped.
- **Signing + verification** proves an image came from your pipeline and wasn't tampered with; an admission policy (Kyverno/Sigstore policy-controller) refuses unsigned images.
- **Digest pinning** — a tag like `1.4.0` can be re-pushed to point at different content; `image: registry/order@sha256:ab12…` is immutable, so what you tested is exactly what runs.
- **Non-root / minimal base** — if an attacker gets code execution, they hold a non-root user in a container with no shell or package manager, which limits what they can do (Q58).
- **Default-deny NetworkPolicy** limits the blast radius of a compromised pod (Q23).
- **Audit logging** records who called the API (e.g. `kubectl exec` into prod); alerting on it detects misuse.
- **Docker socket / `hostNetwork` / `hostPID`** give a container control over the node or visibility of other workloads — effectively a container escape.
- **mTLS** encrypts and mutually authenticates pod-to-pod traffic; a mesh sidecar does it without code changes (cost: extra latency and operational complexity).

---

## 15. Observability: Logs, Metrics, Tracing

### Q62. How should a Spring Boot app log in Kubernetes?

**Log to stdout/stderr, in JSON, and never to a file.** The container runtime captures stdout; a DaemonSet agent (fluent-bit/fluentd/Vector/promtail) ships it to Elasticsearch/Loki/Splunk. Writing to files means logs die with the pod and fill the node disk.

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="!local">
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <customFields>{"app":"order-service","pod":"${POD_NAME:-unknown}"}</customFields>
      </encoder>
    </appender>
    <root level="INFO"><appender-ref ref="JSON"/></root>
  </springProfile>
</configuration>
```
(Spring Boot 3.4+ can do this natively with `logging.structured.format.console=ecs|logstash|gelf`.)

Why JSON: the aggregator can index fields, and — critically — **multi-line Java stack traces** break line-based parsers; a JSON encoder keeps the whole exception in one event.

**Why stdout and not files — the pipeline:** the container runtime captures anything written to stdout/stderr and the kubelet stores it on the node (`/var/log/containers/…`, rotated). A log agent running as a **DaemonSet** (one per node) tails those files, adds Kubernetes metadata (pod, namespace, labels) and ships them to a central store (Loki/Elasticsearch/Splunk). `kubectl logs` reads the same data. Files inside the container, in contrast, vanish when the pod is rescheduled and can fill the pod's ephemeral storage, causing eviction.

**What a good log line contains:** timestamp, level, logger, message, `traceId`/`spanId` (to jump from a log to a trace, Q64), and pod/app identity (Q44). Example of a JSON event:
```json
{"@timestamp":"2025-01-10T09:15:02.123Z","level":"ERROR","logger":"c.e.orders.PaymentClient",
 "message":"payment timeout","traceId":"4bf92f3577b34da6","app":"order-service","pod":"order-service-7c9d8f-x2k4p",
 "stack_trace":"java.net.SocketTimeoutException: ...\n\tat ..."}
```
**Pitfalls:** logging at DEBUG in prod can saturate the node's disk/log agent (change levels at runtime via the Actuator `loggers` endpoint instead, Scenario 13); never log secrets/PII; async appenders can lose the last log lines on SIGKILL, so make sure the logging system is flushed in the graceful-shutdown path; and logs are lost if the node dies before the agent ships them, so ship promptly.

### Q63. Metrics: how do you expose and scrape them?

Micrometer + Actuator → Prometheus:
```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
```yaml
management:
  endpoints: { web: { exposure: { include: health,info,prometheus } } }
  metrics:
    tags:
      application: ${spring.application.name}
      pod: ${POD_NAME:unknown}
  # Boot 3.x: percentile histograms for proper latency SLOs
  observations: { key-values: { region: ap-south-1 } }
```
Scraping options: annotations (`prometheus.io/scrape: "true"`, `prometheus.io/port: "8081"`, `prometheus.io/path: "/actuator/prometheus"`) or, better, a **ServiceMonitor/PodMonitor** CR with the Prometheus Operator.

**Metrics that matter for a JVM on K8s (be ready to name these):**
- `jvm_memory_used_bytes{area="heap"}` vs the container limit
- `jvm_gc_pause_seconds` (count and sum)
- `jvm_threads_live_threads`
- `hikaricp_connections_pending` / `_active` / `_timeout_total`
- `http_server_requests_seconds` (rate, error ratio, p95/p99)
- `container_memory_working_set_bytes` (what the OOM killer actually watches)
- `container_cpu_cfs_throttled_seconds_total` (throttling!)
- `kube_pod_container_status_restarts_total`

🧠 **The 4 golden signals**: latency, traffic, errors, saturation. Plus **USE** for infra: utilization, saturation, errors.

**Why Prometheus pulls from pods (how the flow works):** each pod exposes a plain-text page at `/actuator/prometheus` listing current metric values (counters, gauges, histograms). Prometheus *scrapes* (HTTP GETs) every ~15–30 s and stores the time series; Grafana draws dashboards from it and Alertmanager sends alerts. Because pods are ephemeral and get new IPs, Prometheus uses **Kubernetes service discovery** to find targets: via annotations on pods, or via a `ServiceMonitor`/`PodMonitor` CR (Prometheus Operator) that says "scrape pods matching these labels on port `management`".
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: order-service }
spec:
  selector: { matchLabels: { app: order-service } }
  endpoints:
    - port: management          # named port on the Service
      path: /actuator/prometheus
      interval: 15s
```
**Metric types in one line each:** a **counter** only goes up (requests served) — graph it with `rate()`; a **gauge** goes up and down (heap used, pool active connections); a **histogram/timer** buckets durations so you can compute p95/p99 (`histogram_quantile`). Enable `management.metrics.distribution.percentiles-histogram.http.server.requests=true` for latency histograms you can aggregate across pods.

**Pitfalls:** don't put high-cardinality values (user id, full URL with IDs) in metric tags — it explodes the number of time series and can topple Prometheus; add the `pod`/`application` tags so you can compare replicas (Scenario 6); and keep `/actuator/prometheus` on the internal management port so it's not publicly exposed.

### Q64. Distributed tracing across pods?

Micrometer Tracing (Boot 3.x; Spring Cloud Sleuth in 2.x) + OpenTelemetry exporter → Tempo/Jaeger/Zipkin.
```xml
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-otlp</artifactId></dependency>
```
```yaml
management:
  tracing: { sampling: { probability: 0.1 } }   # 10% in prod
  otlp: { tracing: { endpoint: http://otel-collector.observability:4317 } }
```
Key points: propagate **W3C traceparent** through HTTP and Kafka headers, put `traceId` in MDC so logs and traces correlate, use the **OTel Java agent** (`-javaagent:opentelemetry-javaagent.jar`) for zero-code instrumentation when you can't modify the app, and sample tail-based at the collector to always keep errors and slow traces.

**What tracing is and why you need it:** in a microservice system one user request hops through many pods (gateway → order → payment → inventory). Metrics tell you *something* is slow and logs tell you what one pod did, but only a trace shows *which hop* was slow. Terms: a **trace** = the full journey of one request, identified by a `traceId`; a **span** = one timed step inside it (an HTTP call, a DB query), each with a `spanId` and parent; **context propagation** = passing the ids to the next service in a header so spans link up.

**How it works step by step:**
1. The first service creates a `traceId` and a root span.
2. When it calls the next service (RestClient/WebClient/Kafka producer are auto-instrumented), the `traceparent` header (`00-<traceId>-<spanId>-01`) is added.
3. Each service continues the same trace, records its own spans, and sends them to the OTel Collector (OTLP).
4. The collector forwards to Tempo/Jaeger/Zipkin where you view the waterfall.

**Pitfalls:** sampling `1.0` in prod is costly in CPU, network and storage (hence `0.1` or tail-based sampling); a hand-written `new Thread`/executor loses the context unless you wrap it with Micrometer's context-propagating executor; and a non-instrumented hop (e.g. a custom message format) breaks the chain.

### Q65. Where do heap dumps and thread dumps come from in K8s?

```bash
# thread dump via Actuator
kubectl exec <pod> -- curl -s localhost:8081/actuator/threaddump

# or with jcmd (needs a JDK image or jattach in a debug container)
kubectl exec <pod> -- jcmd 1 Thread.print
kubectl exec <pod> -- jcmd 1 GC.heap_info
kubectl exec <pod> -- jcmd 1 VM.native_memory summary

# heap dump to a mounted emptyDir, then copy out
kubectl exec <pod> -- jcmd 1 GC.heap_dump /dumps/heap.hprof
kubectl cp <pod>:/dumps/heap.hprof ./heap.hprof

# ephemeral debug container for distroless images (1.23+)
kubectl debug -it <pod> --image=eclipse-temurin:21-jdk --target=order-service
```
Say this too: *"For a distroless or JRE-only image I can't exec a JDK tool, so I use `kubectl debug` to attach an ephemeral container sharing the process namespace."* That's a modern, senior answer.

**What each dump tells you and when to take it:**
- **Thread dump** — a snapshot of every thread's stack. Use it when the app is *hung or slow* (CPU high, requests timing out): look for threads `BLOCKED` on the same lock (deadlock), many threads waiting on the DB pool (`HikariPool.getConnection`), or a hot loop. Take **3 dumps ~10 s apart** — one dump is a photo, three show what is *stuck* versus merely busy.
- **Heap dump** — a snapshot of all objects in memory (`.hprof`). Use it for memory leaks or `OutOfMemoryError`; open it in Eclipse MAT or VisualVM and look at the **dominant objects / leak suspects**. It pauses the JVM (stop-the-world) and can be as large as the heap, and it contains in-memory secrets/PII — handle it as sensitive data.
- **Native memory summary** — shows non-heap usage (metaspace, threads, code cache); the tool for "heap is fine but the pod is OOMKilled" (Q36). Needs `-XX:NativeMemoryTracking=summary` on startup.

**Why `jcmd 1`:** PID 1 is the JVM in a correctly built image (Q55). **Why the dump may be missing:** if the pod is OOMKilled (SIGKILL) the JVM has no chance to write a dump, which is why `-XX:+HeapDumpOnOutOfMemoryError` (a *Java* OOM) and keeping `-Xmx` safely below the container limit matter. Also remember that `emptyDir` is wiped when the pod is deleted — copy the file out first, or write to a PVC.

**`kubectl debug` explained:** adds a temporary container to the *running* pod (image of your choice) with `--target` to share the app container's process namespace, so tools like `jcmd`, `jstack`, `top`, `tcpdump` can see the Spring Boot process without changing your production image or restarting the pod. Note that `jcmd` must run as the **same user** as the JVM and share `/tmp` to attach — a common reason it fails with "Could not attach".

---

## 16. Helm, Kustomize & GitOps

### Q66. What is Helm and what problem does it solve?

Helm is the package manager: a **chart** = templated K8s manifests + a `values.yaml`. It solves duplication across environments and gives you **release versioning, upgrade, and rollback** as a unit.

```
order-service/
├── Chart.yaml
├── values.yaml            # defaults
├── values-prod.yaml       # per-env overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── configmap.yaml
    └── _helpers.tpl
```
```bash
helm upgrade --install order-service ./order-service \
  -f values-prod.yaml --set image.tag=2.14.1 \
  -n orders --atomic --timeout 5m
helm rollback order-service 4
helm template ./order-service -f values-prod.yaml   # render locally, no cluster
helm diff upgrade order-service ./order-service      # plugin, essential in CI
```
`--atomic` is worth calling out: on failure it automatically rolls back, so you never leave a half-applied release.

**Key concepts defined:**
- **Chart** — a folder/package of templates plus metadata (`Chart.yaml`). **Release** — one *installed instance* of a chart in a cluster (e.g. `order-service` in namespace `orders`); you can install the same chart multiple times under different release names.
- **`values.yaml`** — defaults for the template variables; `-f values-prod.yaml` and `--set` override them, which is how one chart serves dev/QA/prod.
- **Templates** use Go templating, e.g. `replicas: {{ .Values.replicaCount }}`; `_helpers.tpl` holds reusable snippets such as common labels.
- Helm stores each release revision as a Secret in the namespace, which is what makes `helm history` and `helm rollback` possible.

**Why it exists:** without Helm, you'd copy-paste near-identical YAML for each environment and hand-edit it (image tag, replicas, hostnames) — error-prone and unreviewable. A chart parameterises the differences and treats Deployment + Service + Ingress + HPA + ConfigMap as one versioned unit.

**Tiny template example:**
```yaml
# templates/deployment.yaml (excerpt)
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
# values.yaml
replicaCount: 2
image: { repository: registry/order-service, tag: "2.14.0" }
```
**Pitfalls:** templates get hard to read with heavy logic; `helm rollback` reverts manifests but **not database migrations**; secrets in `values-prod.yaml` must not be committed in plain text (use SOPS/helm-secrets/ESO, Q60). `helm upgrade --install` is the idempotent command used in CI.

### Q67. Helm vs Kustomize.

| | Helm | Kustomize |
|---|---|---|
| Approach | Templating (Go templates) | **Overlays/patching** of plain YAML |
| Learning curve | Steeper (templates + values + functions) | Gentler; YAML stays valid YAML |
| Packaging/distribution | Excellent (repos, versions, dependencies) | None built-in |
| Release lifecycle | Install/upgrade/rollback/history | Just produces manifests (`kubectl apply -k`) |
| Best for | Distributing third-party apps; many knobs | Your own apps with small per-env deltas |

**Pragmatic answer:** *"Helm for third-party components (Prometheus, Kafka, ingress) because I want the upstream chart; Kustomize (or Helm with per-env values) for our own services. Kustomize's base + overlay model is easier to review in a PR because you see actual YAML, not template soup."*

```yaml
# overlays/prod/kustomization.yaml
resources: [../../base]
namespace: orders
replicas:
  - { name: order-service, count: 6 }
images:
  - name: registry/order-service
    newTag: 2.14.1
patches:
  - path: resources-patch.yaml
configMapGenerator:
  - name: order-service-config
    files: [application.yaml]     # generates a hashed name → auto rolling restart!
```
🧠 Kustomize's `configMapGenerator` appends a **content hash** to the name, so changing config changes the ConfigMap name → the Deployment's pod template changes → automatic rolling restart. That's the elegant answer to Q27.

### Q68. What is GitOps?

Git is the single source of truth for desired cluster state; an in-cluster agent (**ArgoCD** or **Flux**) continuously reconciles the cluster to the repo.

Benefits to list: auditability (every change is a PR), easy rollback (`git revert`), **drift detection and self-healing**, no CI system holding cluster credentials (pull-based, not push-based).

Typical pipeline:
```
git push (app code)
  → CI: build, test, build image, scan, push image:2.14.1
  → CI: bump image tag in the config repo (or Argo Image Updater does it)
  → ArgoCD detects the commit
  → syncs to cluster (with optional manual approval gate for prod)
  → health assessment; auto-rollback on failure
```
Best practice: **separate app repo and config repo** so an image bump doesn't retrigger CI, and prod syncs can require approval.

**Plain-language explanation:** in classic CI/CD, the pipeline *pushes* changes into the cluster using `kubectl apply`/`helm upgrade`, so the CI server needs powerful cluster credentials and nobody can tell if someone later changed the cluster by hand. In GitOps the cluster *pulls*: an agent running inside it compares "what Git says" with "what is live" every few minutes (or on webhook) and corrects differences.

**Key terms:**
- **Desired state** = the YAML/Helm/Kustomize in Git; **live state** = what's actually in the cluster.
- **Sync** = apply the desired state; **drift** = live state that differs from Git (e.g. someone ran `kubectl edit`); **self-heal** = the agent reverts drift automatically; **prune** = deleting cluster objects that were removed from Git.
- **ArgoCD** is an `Application` CR pointing at a repo path + target namespace and has a UI showing health/sync status; **Flux** is a set of controllers (Source, Kustomize, Helm) with no built-in UI.

**Tiny Argo example:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: order-service-prod, namespace: argocd }
spec:
  project: default
  source: { repoURL: https://git.example.com/platform/config.git, path: apps/order-service/overlays/prod, targetRevision: main }
  destination: { server: https://kubernetes.default.svc, namespace: orders }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```
**Pitfalls / probes:** secrets can't go in Git as plain text (Q60); with `selfHeal` a manual hot-fix in prod will be reverted, which is intended — fix it in Git; and **HPA vs Git replicas** conflict (omit `replicas` from the Deployment when an HPA owns it, or Argo will keep resetting it). *"How do you roll back?"* → `git revert` the commit and let the agent sync.

---

## 17. Scheduling: Affinity, Taints, Topology Spread

### Q69. nodeSelector vs affinity vs taints/tolerations.

- **nodeSelector** — simple hard equality match on node labels.
- **nodeAffinity** — expressive: `requiredDuringSchedulingIgnoredDuringExecution` (hard) and `preferredDuringScheduling...` (soft, weighted). Supports `In`, `NotIn`, `Exists`, `Gt`, `Lt`.
- **podAffinity/podAntiAffinity** — schedule relative to **other pods** (co-locate with cache; spread replicas apart).
- **Taints & tolerations** — the **inverse**: the *node* repels pods; only pods with a matching toleration land there.

🧠 **Affinity = the pod chooses the node. Taints = the node chooses the pod.**

```yaml
spec:
  # Keep replicas off the same node (soft, so scheduling never blocks)
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels: { app: order-service }
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - { key: node-type, operator: In, values: ["memory-optimized"] }
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "payments"
      effect: "NoSchedule"
```

Taint effects: `NoSchedule` (don't place new pods), `PreferNoSchedule` (soft), `NoExecute` (also **evict** existing pods that don't tolerate it — this is how node-not-ready eviction works, with `tolerationSeconds`).

**Why they exist (the problem each solves):** by default the scheduler may put your pod on *any* node with free resources. Real clusters need control: GPU/memory-optimised nodes for specific workloads, keeping latency-sensitive replicas apart so one node failure doesn't take them all, and reserving nodes for a team.

**How each piece works:**
- **Node labels** are key/values on nodes (`node-type=memory-optimized`, `topology.kubernetes.io/zone=ap-south-1a`). `nodeSelector`/`nodeAffinity` match on them. Hard (`required…`) = pod stays `Pending` if no match; soft (`preferred…`) = scheduler tries, but can ignore. The `IgnoredDuringExecution` suffix means: if labels change *after* scheduling, the running pod isn't moved.
- **Taints** are applied to nodes: `kubectl taint nodes node1 dedicated=payments:NoSchedule`. Without a matching **toleration** a pod is repelled. Note: a toleration only *permits* landing there — it doesn't *attract* the pod. To pin payments pods to those nodes you need **taint + toleration + nodeAffinity/nodeSelector** together.
- **topologyKey** in pod (anti-)affinity defines "same place": `kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same AZ.

**Typical uses:** control-plane nodes are tainted so app pods don't run there; GPU pool tainted so only ML workloads use it; DaemonSets add tolerations to run on every node; spot-instance pools tainted so only interruption-tolerant pods are placed there.

**Pitfall:** hard anti-affinity with 6 replicas on a 3-node cluster leaves 3 pods `Pending` forever — the reason the example above uses `preferred`.

### Q70. topologySpreadConstraints — and why prefer it over anti-affinity?

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway      # DoNotSchedule = hard
    labelSelector:
      matchLabels: { app: order-service }
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: { app: order-service }
```
Pod anti-affinity is effectively "all or nothing" per topology and is **computationally expensive** at scale (O(pods²) evaluations slow the scheduler). Topology spread lets you say *"balanced across zones, tolerate a skew of 1"* — which is what you actually want for HA — and performs much better.

With `ScheduleAnyway`, a zone outage still allows scheduling (degraded but running). With `DoNotSchedule`, pods stay `Pending` rather than clump — safer for strict HA, riskier for availability. Know the trade-off.

**What the fields mean:** `topologyKey` = *what to spread across* (zones, nodes); `maxSkew` = the maximum allowed difference in matching-pod count between the most and least populated domain; `labelSelector` = which pods are counted.

**Worked example:** 6 replicas across 3 zones with `maxSkew: 1` → the allowed layout is 2/2/2. A layout like 3/2/1 has a skew of 2 (3 − 1), so it violates the rule and the scheduler would put the next pod in the least-populated zone. With 5 pods the best possible is 2/2/1 (skew 1). If a zone goes down, replacement pods must land in the surviving zones; with `DoNotSchedule` a strict skew rule can leave them `Pending` (the skew is computed over eligible domains, so check your cluster's behaviour) — hence `ScheduleAnyway` when availability beats perfect balance.

**Why it matters for HA:** you want an AZ failure to cost you ~1/3 of capacity, not all of it. Spread across **zones** (survive AZ loss) *and* **nodes** (survive a node loss), as in the example. Pair with a PodDisruptionBudget (Q47). Pitfall: spreading only counts pods when they are *scheduled*; after scale-down or node churn, balance can drift (the descheduler can rebalance).

---

## 18. Dockerfile & Image Best Practices for Spring Boot

### Q71. Write a production Dockerfile for a Spring Boot app.

```dockerfile
# ---------- Stage 1: build ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
RUN mvn -B dependency:go-offline          # cached layer: deps change rarely
COPY src ./src
RUN mvn -B clean package -DskipTests

# ---------- Stage 2: extract layers ----------
FROM eclipse-temurin:21-jre-alpine AS extract
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract --destination extracted

# ---------- Stage 3: runtime ----------
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S -g 10001 spring && adduser -S -u 10001 -G spring spring
WORKDIR /app

# Order matters: least-changing layer first = maximum cache reuse
COPY --from=extract --chown=spring:spring /app/extracted/dependencies/         ./
COPY --from=extract --chown=spring:spring /app/extracted/spring-boot-loader/   ./
COPY --from=extract --chown=spring:spring /app/extracted/snapshot-dependencies/ ./
COPY --from=extract --chown=spring:spring /app/extracted/application/          ./

USER 10001:10001
EXPOSE 8080 8081
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Why each choice — say these out loud:**
1. **Multi-stage** → JDK, Maven and source aren't in the final image (smaller + fewer CVEs).
2. **Layered JAR (`layertools`)** → dependencies (~50 MB, rarely change) sit in a separate layer from your code (~1 MB). A code-only change re-pushes/re-pulls ~1 MB instead of 50 MB → dramatically faster deploys and rollouts.
3. **JRE not JDK** → ~half the size.
4. **Non-root numeric USER** → satisfies `runAsNonRoot` (K8s can't verify a *named* user is non-root, so use the UID).
5. **Exec-form ENTRYPOINT** → JVM is PID 1 → receives SIGTERM → graceful shutdown works.
6. **`JAVA_TOOL_OPTIONS`** → overridable per environment without rebuilding.
7. **`.dockerignore`** (target/, .git/, .idea/) → smaller build context.

Alternatives to mention: `spring-boot-maven-plugin`'s `build-image` goal (**Cloud Native Buildpacks/Paketo**) — no Dockerfile at all, sensible JVM memory calculator baked in; and **Jib** (no Docker daemon needed, reproducible builds, great in CI).

**Concepts behind the list (so you can explain, not just recite):**
- **Image layers:** a Docker image is a stack of read-only layers, one per `COPY`/`RUN`. Registries and nodes cache layers by content hash, so an unchanged layer isn't re-uploaded/re-downloaded. A fat JAR in one `COPY` produces one 50 MB layer that changes on *every* build; splitting it with Spring's `layertools` into `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application` means only the tiny `application` layer changes on a code edit. Order matters because once one layer changes, **all layers after it are rebuilt** — put what changes least first.
- **Multi-stage build:** several `FROM` stages in one Dockerfile; only the *last* stage ends up in the image. Build tools (Maven, JDK, source code, `~/.m2`) stay in earlier stages, so the runtime image has no compiler to abuse and is far smaller.
- **Why a numeric UID:** `runAsNonRoot: true` makes the kubelet refuse to start a container whose user is root. With `USER spring` the kubelet can't prove that name isn't UID 0, so it errors (`cannot verify user is non-root`); `USER 10001` is verifiable.
- **Why alpine/slim/distroless:** fewer OS packages = fewer CVEs and less to scan. Caveats: Alpine uses **musl libc** (rare native-library/DNS quirks, some profilers misbehave) and distroless has no shell, so `kubectl exec ... sh` fails (use `kubectl debug`, Q65).
- **`.dockerignore`** works like `.gitignore` for the build context: without it `docker build` uploads `.git` and `target/` to the daemon and invalidates cache needlessly.

**Handy commands:** `docker history <image>` to see layer sizes; `dive <image>` to explore wasted space.

### Q72. Image tagging: what's wrong with `:latest`?

- Non-deterministic: two pods created minutes apart can run **different code**.
- Breaks rollback (there's no previous tag to go back to).
- With `imagePullPolicy: Always` you add a registry dependency to every pod start; with `IfNotPresent` you may silently run a stale cached image.
- `:latest` implicitly sets `imagePullPolicy: Always`, which surprises people.

**Do instead:** immutable tags from CI — semantic version + git SHA (`2.14.1-a1b2c3d`) — and pin by **digest** in prod (`image: repo/app@sha256:...`). Enable registry tag immutability so a tag can never be overwritten.

**Why these matter, with a scenario:** Monday you deploy `app:latest` (v1). Tuesday a CI build pushes a new `latest` (v2, with a bug). Overnight a node is replaced; the new node pulls `latest` and now runs v2 while the other nodes still run cached v1 — you have a mixed fleet and nothing in the manifest explains why. `kubectl rollout undo` doesn't help either, because the old and new ReplicaSet both say `image: app:latest`.

**`imagePullPolicy` rules:** if you don't set it, K8s uses `Always` for `:latest` (or no tag) and `IfNotPresent` for any other tag. `IfNotPresent` is correct for immutable tags — it avoids hitting the registry on every pod start (faster scale-out, resilience to registry outages).

**Tag vs digest:** a tag is a movable label; a digest (`sha256:…`) is the content hash of the image, so it can never point to different bits. A typical setup: CI tags `2.14.1-a1b2c3d` for readability, and the GitOps config records the digest for prod (tools like Renovate / Argo Image Updater can bump it). Interview probe: *"How do you trace a running pod back to a commit?"* → the tag includes the git SHA, plus OCI labels (`org.opencontainers.image.revision`) or the Spring Boot `build-info`/`git.properties` exposed at `/actuator/info`.

---

## 19. Scenario-Based Questions

> **Answer template — use this every single time.** Interviewers score *structure* as much as content:
> **1. Clarify** (one question: "is it all pods or one? Did anything deploy recently?") → **2. Observe** (exact commands) → **3. Hypothesize** (2–3 ranked causes) → **4. Mitigate** (stop the bleeding) → **5. Fix root cause** → **6. Prevent** (so it can't recur).
>
> 🧠 Mnemonic: **C-O-H-M-F-P** — *"Clarify, Observe, Hypothesize, Mitigate, Fix, Prevent."*

---

### Scenario 1 — "Your pod is stuck in `Pending`. Walk me through it."

**Observe:**
```bash
kubectl describe pod <pod> | tail -30      # the Events section IS the answer
kubectl get events --sort-by=.lastTimestamp | grep <pod>
kubectl get nodes -o wide
kubectl describe node <node> | grep -A6 "Allocated resources"
```

**Ranked causes and what the event says:**

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient cpu/memory` | No node has room for the **requests** | Lower requests, scale the node pool, check Cluster Autoscaler logs |
| `node(s) had untolerated taint` | Missing toleration | Add toleration or remove taint |
| `node(s) didn't match Pod's node affinity/selector` | Label typo or no matching nodes | Fix selector / label the nodes |
| `pod has unbound immediate PersistentVolumeClaims` | PVC not bound | Check PVC/StorageClass; often needs `WaitForFirstConsumer` |
| `volume node affinity conflict` | Disk is in AZ-a, only AZ-b has capacity | `WaitForFirstConsumer`, or multi-AZ storage |
| `too many pods` | Node's max-pods limit reached (e.g., ENI limits on EKS) | Bigger instance type or more nodes |
| `exceeded quota` | Namespace ResourceQuota | Raise quota or reduce requests |
| No events, just Pending | Scheduler down, or pod is `Pending` because it's `Terminating` elsewhere | Check control plane |

**Senior addition:** *"If it's insufficient resources, I first check whether our requests are honest. I've seen a team request 4 CPUs for a service that averaged 200m — that's not a capacity problem, it's a right-sizing problem. I'd pull the actual usage from Prometheus or VPA recommendations before adding nodes."*

---

### Scenario 2 — "The app was working. You deploy. Now `ImagePullBackOff`."

**Observe:** `kubectl describe pod <pod>` → read the exact pull error.

| Error | Cause | Fix |
|---|---|---|
| `manifest unknown` / `not found` | Wrong tag or image name; CI didn't push | Verify tag exists in registry; check CI push step |
| `unauthorized` / `denied` | Missing/expired `imagePullSecrets` | Create docker-registry secret; attach to SA or pod |
| `dial tcp ... i/o timeout` | Node can't reach the registry (no NAT/egress, firewall, private registry DNS) | Network/VPC endpoint fix |
| `toomanyrequests` | Docker Hub rate limit | Authenticate, or mirror into ECR/Artifactory |
| Works on some nodes only | New nodes lack cached image + credentials issue | Check node IAM role / pull secret propagation |

```yaml
imagePullSecrets:
  - name: regcred
```
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com --docker-username=ci --docker-password=$TOKEN
```
**Prevent:** validate the image exists in CI before applying manifests; use IAM-based registry auth (ECR + node role / IRSA) so there are no static credentials to expire.

---

### Scenario 3 — "Pods restart every ~2 minutes. Logs look normal, no exceptions. What's happening?"

**This is the liveness-probe trap.** Normal logs + regular restarts + no application error ⇒ Kubernetes is killing a *healthy* app.

**Observe:**
```bash
kubectl describe pod <pod>
# look for: Liveness probe failed: Get "http://10.1.2.3:8080/actuator/health": context deadline exceeded
# and: Last State: Terminated, Exit Code: 137, Reason: Error
kubectl get events | grep Unhealthy
```

**Root causes, ranked:**
1. **`initialDelaySeconds` too short** — Spring Boot takes 45s to start, probe fires at 15s, `failureThreshold: 3` × 5s kills it at 30s → infinite restart. **Fix: use a `startupProbe`.**
2. **Liveness points at `/actuator/health` (the aggregate)** which includes DB/Redis/Kafka. A downstream blip fails the probe → restart. **Fix: point at `/actuator/health/liveness`.**
3. **`timeoutSeconds: 1` under GC pressure** — a 1.5s G1 pause makes the probe time out. **Fix: `timeoutSeconds: 3–5`, `failureThreshold: 3`.**
4. **Thread pool exhaustion** — Tomcat's 200 threads are all blocked on a slow downstream, so the probe request can't even be served. **Fix: separate `management.server.port` so the actuator has its own connector/thread pool — probes then survive app-thread starvation.** (Excellent senior answer.)
5. **OOMKilled** — check `Reason: OOMKilled` (see Scenario 4).

**Mitigate now:** raise `failureThreshold` / temporarily remove the liveness probe to stop the restart loop, so you can actually investigate a live pod.

---

### Scenario 4 — "Pod restarts with exit code 137 and `Reason: OOMKilled`. Heap looks fine at 60%. Explain."

**Observe:**
```bash
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl top pod <pod> --containers
# Prometheus: container_memory_working_set_bytes vs kube_pod_container_resource_limits
kubectl exec <pod> -- jcmd 1 VM.native_memory summary    # if NMT enabled
```

**Explanation to give:** the cgroup limit caps **total RSS**, not heap. Heap at 60% of `-Xmx` says nothing about metaspace, code cache, thread stacks, and — most often — **direct/native memory** from Netty, gRPC, or the Kafka client. Off-heap growth is invisible in heap metrics.

**Ranked root causes:**
1. **`-Xmx` set too close to the limit** (e.g., `-Xmx1g` with `limit: 1Gi` → guaranteed OOMKill).
2. **Unbounded direct memory** — Netty `ByteBuf` leak or missing `-XX:MaxDirectMemorySize`.
3. **Metaspace growth** — heavy CGLIB/ByteBuddy proxying, dynamic class generation, repeated context refresh.
4. **Thread explosion** — an unbounded `Executors.newCachedThreadPool()` → 1 MB stack each.
5. **Genuinely under-provisioned** — usage grew with traffic.

**Fix:**
```yaml
resources:
  requests: { memory: "1Gi" }
  limits:   { memory: "1Gi" }
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxRAMPercentage=70 -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=128m -XX:NativeMemoryTracking=summary -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps -XX:+ExitOnOutOfMemoryError"
```
**Prevent:** alert on `container_memory_working_set_bytes / limit > 0.85`; heap dump on OOM to an emptyDir + copy out; run a load test at the limit before prod; track `jvm_buffer_memory_used_bytes` for direct memory.

---

### Scenario 5 — "Users get intermittent 502s during every deployment, even though replicas never drop to zero."

**Diagnosis:** classic **termination race**. When a pod is deleted, SIGTERM and endpoint removal happen **in parallel**. Spring Boot shuts down in ~1s, but kube-proxy on other nodes and the Ingress controller take a few seconds to stop routing → requests hit a dead pod.

**Fix (all four together):**
```yaml
spec:
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
  minReadySeconds: 15
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 15"] }   # keep serving while endpoints propagate
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8081 }
            periodSeconds: 5
```
```yaml
# application.yaml
server: { shutdown: graceful }
spring: { lifecycle: { timeout-per-shutdown-phase: 30s } }
```

**Also check:**
- **Keep-alive mismatch:** the Ingress/client holds a keep-alive connection to a terminating pod. Set the app's `server.tomcat.keep-alive-timeout` **lower** than the LB's idle timeout so the *server* closes first, and enable retries on idempotent requests at the ingress.
- **Cloud LB targeting NodePorts:** with `externalTrafficPolicy: Cluster` the LB may keep sending to a node whose pod is gone; deregistration delay on the target group matters too.
- **Client-side:** Feign/RestTemplate/WebClient should retry idempotent GETs on connection-reset (Resilience4j retry + a *short* connect timeout).

**One-line summary to say:** *"Zero-downtime needs `maxUnavailable: 0`, a readiness probe, a preStop sleep, graceful shutdown, and a grace period longer than both."*

---

### Scenario 6 — "One replica out of six is slow. Same image, same config. Why?"

**Ranked hypotheses:**
1. **Noisy neighbour / CPU throttling** — that pod's node is saturated, or the pod is hitting its CFS quota. Check `container_cpu_cfs_throttled_seconds_total` for that pod and node-level load; `kubectl get pod -o wide` to see the node and what else runs there.
2. **JVM state** — that instance is doing more GC (check `jvm_gc_pause_seconds` per pod), or has a leak building up, or a thread pool is saturated (`hikaricp_connections_pending` high on that pod only).
3. **Cold start / no warm-up** — it was just replaced, JIT is still interpreting and caches are cold.
4. **Uneven load balancing** — with HTTP/1.1 keep-alive or gRPC, a client pins connections; one pod can hold most of the long-lived connections. Check request rate per pod: if the *rate* differs, it's LB skew, not the pod.
5. **Node-level issue** — degraded disk/network, different instance type in a mixed node pool, different AZ adding cross-AZ latency to the DB.

**Actions:** `kubectl delete pod` to confirm it follows the pod (app/JVM) or stays on the node (infra). Then fix properly: `topologySpreadConstraints` for placement, per-pod dashboards, client-side load balancing or a mesh for connection-level skew.

**Sound bite:** *"First question I answer is whether the anomaly follows the pod or the node — that single test halves the search space."*

---

### Scenario 7 — "Service A can't reach Service B. Debug it."

Work up the stack, **one layer at a time**:

```bash
# 0. Are B's pods even Ready?
kubectl get pods -l app=service-b
kubectl get endpointslices -l kubernetes.io/service-name=service-b   # EMPTY = the answer

# 1. DNS resolution from inside A
kubectl exec -it <pod-a> -- nslookup service-b
kubectl exec -it <pod-a> -- nslookup service-b.other-ns.svc.cluster.local

# 2. TCP reachability
kubectl exec -it <pod-a> -- nc -zv service-b 8080
kubectl exec -it <pod-a> -- curl -sv http://service-b:8080/actuator/health

# 3. Bypass the Service — hit the pod IP directly
kubectl get pod -l app=service-b -o wide      # get pod IP
kubectl exec -it <pod-a> -- curl http://<pod-ip>:8080/actuator/health

# 4. Debug pod if A has no tools (distroless)
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
kubectl debug -it <pod-a> --image=nicolaka/netshoot --target=app
```

**Interpretation table:**

| Symptom | Cause |
|---|---|
| Empty EndpointSlice | **Selector doesn't match pod labels**, or no pod is **Ready** (readiness failing) — the #1 cause |
| DNS fails | Wrong namespace/FQDN, CoreDNS down, or NetworkPolicy blocking UDP 53 egress |
| DNS ok, connection refused | Wrong `port`/`targetPort`, app bound to `127.0.0.1` instead of `0.0.0.0`, or wrong `containerPort` |
| Pod IP works, Service IP doesn't | Service port mapping wrong, or kube-proxy/iptables issue |
| Timeout (not refused) | **NetworkPolicy** dropping packets, or firewall/security group |
| Works intermittently | Only *some* replicas are broken — one bad pod is in the endpoint list |

**Most common real answer for Spring Boot:** `targetPort` ≠ `server.port`, or `spring.profiles` mismatch causing the app to bind a different port. Second most common: readiness failing so endpoints are empty, and the developer only looks at the Service.

---

### Scenario 8 — "You need to roll out v2 with a breaking DB change to a 10-replica service, zero downtime."

**Answer with the expand/contract pattern** (see Q51), spread over releases:

**Release 1 (expand):** add the new column as nullable with a default; deploy code that **ignores** it. Backward compatible → safe rolling update.
**Release 2 (dual-write):** code writes both old and new; still reads old. Run a backfill Job for existing rows (batched, throttled).
**Release 3 (switch reads):** code reads new, still writes both. Verify with metrics/queries.
**Release 4 (contract):** remove old writes; only then drop the old column.

**Migration mechanics:** Flyway/Liquibase in an **init container or a pre-upgrade Helm hook Job**, never concurrently in all 10 replicas (Flyway takes a lock, but 10 pods contending on startup causes timeouts and confusing failures).

```yaml
# Helm pre-upgrade hook
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
```

**Also mention:** feature flags to decouple deploy from release, `CREATE INDEX CONCURRENTLY` on Postgres to avoid table locks, and a rollback plan — *"if the migration is not backward compatible, my rollback is a forward fix, not `helm rollback`. That has to be a conscious decision before I press deploy."*

---

### Scenario 9 — "Production traffic spikes 5× at 9 AM. Pods scale up but users still see timeouts for 3 minutes. Fix it."

**Why HPA alone doesn't save you — the latency chain:**
```
metrics-server scrape (up to 15s)
+ HPA evaluation (15s)
+ pod scheduling (1–5s)
+ node provisioning if no room (60–120s!)
+ image pull (5–60s)
+ Spring Boot startup (30–60s)
+ JIT warm-up + cold caches (10–30s of slow responses)
= 2–4 minutes before real capacity arrives
```

**Fixes, in order of impact:**
1. **Pre-scale on schedule** — a CronJob or KEDA cron trigger that raises `minReplicas` at 8:45. Predictable traffic doesn't need reactive autoscaling.
2. **Raise `minReplicas`** so baseline capacity absorbs the spike; autoscale for the *unpredictable* part only.
3. **Aggressive `behavior.scaleUp`** (`Percent: 100, periodSeconds: 15`) + `stabilizationWindowSeconds: 0`.
4. **Overprovisioning / pause pods** — low-priority placeholder pods holding node capacity. Real pods with higher `PriorityClass` preempt them instantly, so you skip the node-provisioning wait.
5. **Faster startup** — AppCDS, lazy init, smaller layered image, `TieredStopAtLevel=1` for short-lived pods; GraalVM native if it fits.
6. **Warm-up before serving traffic** — readiness stays false until caches are primed (see Q31), so cold pods never take traffic.
7. **Load shedding + backpressure** — Resilience4j bulkhead/rate limiter so overload degrades gracefully instead of timing out everything.
8. **Scale on a leading indicator** — request rate or queue depth rather than CPU, so you scale *before* saturation.

**Sound bite:** *"Autoscaling is a cost optimization, not a latency solution. For a known 9 AM spike I pre-scale; I use HPA for the traffic I can't predict."*

---

### Scenario 10 — "A ConfigMap change was applied but nothing changed in the app. Why?"

Because env-injected values and even volume-mounted files are not automatically re-read (see Q27). Diagnose:
```bash
kubectl get cm order-config -o yaml                  # did it actually update?
kubectl exec <pod> -- env | grep MY_KEY              # env = frozen at start
kubectl exec <pod> -- cat /config/application.yaml    # volume = updated eventually
kubectl exec <pod> -- curl localhost:8081/actuator/env | jq   # what Spring actually sees
```

**Three specific gotchas:**
1. **Env vars never update** → restart required.
2. **`subPath` mounts never update** → this catches everyone. Mount the whole ConfigMap dir instead.
3. **Immutable ConfigMap** → updates rejected outright.

**Fix now:** `kubectl rollout restart deployment/order-service`.
**Prevent:** Helm `checksum/config` annotation, or Kustomize `configMapGenerator` (hash-suffixed names) so config changes automatically trigger a rollout. If you truly need runtime refresh, Spring Cloud Kubernetes with `@RefreshScope` + reload, but call out the caveat: **`@RefreshScope` doesn't re-create everything** — `@Value` in singletons, datasource pools, and Kafka listener containers often need a restart anyway, so a rolling restart is usually the more honest design.

---

### Scenario 11 — "`kubectl get pods` shows Running 1/1 but the app returns 503. What now?"

`1/1 Running` means the **container process is alive**, not that the app works. `READY` reflects the readiness probe — if there's no readiness probe defined, K8s calls it Ready immediately.

```bash
kubectl logs <pod> --tail=100
kubectl exec <pod> -- curl -s localhost:8081/actuator/health | jq
kubectl port-forward <pod> 8080:8080     # bypass Service/Ingress entirely
```
Ranked causes:
1. **No readiness probe** → traffic arrives before the Spring context is up.
2. Spring Boot returns 503 from `/actuator/health` because a **health indicator is DOWN** (DB, Redis, disk space, Kafka). `/actuator/health` with `show-details` tells you which.
3. **Circuit breaker open** → app is up but shedding.
4. **App started with the wrong profile** → wrong DB, wrong feature flags.
5. **503 comes from the Ingress, not the app** (no healthy upstreams) — `port-forward` distinguishes these instantly.

🧠 **"`Running` is about the process. `Ready` is about the contract."**

---

### Scenario 12 — "Node goes down. How long until pods are rescheduled, and how do you make it faster?"

**Default timeline (~5+ minutes):**
- kubelet stops posting status → after `node-monitor-grace-period` (**40s**) node is marked `NotReady`.
- Node gets a `NoExecute` taint; pods have a default `tolerationSeconds` of **300s**.
- Only then are pods evicted and rescheduled, plus scheduling + image pull + Spring Boot startup.

**Make it faster:**
```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
```
**But say the trade-off:** shorter timeouts cause spurious evictions during transient network blips → churn. The real answer is *"don't depend on rescheduling speed"*: spread replicas across nodes/zones (`topologySpreadConstraints`), keep enough replicas that losing one node is a non-event, and use a PDB. Rescheduling speed is a recovery detail; **spread** is the actual availability control.

Also: **StatefulSet pods on a lost node won't reschedule until the node object is deleted** (K8s can't be sure the old pod isn't still writing to the RWO volume). This surprises people during a node failure with Kafka/Postgres — worth flagging.

---

### Scenario 13 — "You have to debug a running production pod. What are your options, in order of risk?"

1. **`kubectl logs`** (+ `--previous`, `-f`, `--since=10m`) — zero risk.
2. **Actuator endpoints** — `/health`, `/env`, `/metrics`, `/threaddump`, `/heapdump`, `/loggers`, `/configprops`, `/beans`. Change a log level at runtime with no restart:
   ```bash
   kubectl exec <pod> -- curl -X POST localhost:8081/actuator/loggers/com.example.orders \
     -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'
   ```
3. **`kubectl port-forward`** — reach the pod directly from your laptop, bypassing the Service and Ingress. Also how you attach a **local JMX/JFR/VisualVM** session.
4. **`kubectl exec`** — shell in. Risk: it's a shared prod process; audit-logged; not possible on distroless.
5. **`kubectl debug`** — ephemeral container (or `--copy-to` a debug clone of the pod). Best option for distroless/slim images:
   ```bash
   kubectl debug -it <pod> --image=nicolaka/netshoot --target=app       # network tools
   kubectl debug <pod> --copy-to=debug-pod --set-image=app=temurin:21-jdk
   ```
6. **JFR for a slow-but-not-crashing service** — start a recording, copy it out, analyze in JMC:
   ```bash
   kubectl exec <pod> -- jcmd 1 JFR.start duration=120s filename=/dumps/rec.jfr settings=profile
   kubectl cp <pod>:/dumps/rec.jfr ./rec.jfr
   ```
7. **Remote debug** — only in non-prod. `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005` + `port-forward 5005`. Never in prod (a breakpoint pauses a live JVM serving customers).

**Add:** *"Anything I do interactively, I follow up with the permanent fix — a metric, a log line, or a dashboard — so the next person doesn't need to exec into prod at all."*

---

### Scenario 14 — "Your Spring Boot app connects to Postgres. After scaling to 20 replicas, you get connection errors. Why?"

**The arithmetic nobody does:** `20 replicas × HikariCP maximumPoolSize 10 = 200 connections`. Postgres default `max_connections` is **100**. You exhausted the database, not the cluster.

**Symptoms:** `FATAL: sorry, too many clients already`, `HikariPool-1 - Connection is not available, request timed out after 30000ms`, rising `hikaricp_connections_pending`.

**Fixes:**
1. **Right-size the pool per replica.** Total connection budget ÷ max replicas. Formula worth quoting: `connections ≈ (2 × cores) + effective_spindles` — for a container with 2 CPUs, a pool of **5** is often plenty. Big pools *reduce* throughput via contention.
   ```yaml
   spring:
     datasource:
       hikari:
         maximum-pool-size: 5
         minimum-idle: 2
         connection-timeout: 3000
         max-lifetime: 900000      # < DB/proxy idle timeout & shorter than credential TTL
         leak-detection-threshold: 20000
   ```
2. **Introduce a connection pooler** — PgBouncer (transaction pooling) between app and DB. Caveat: transaction pooling breaks prepared statements/session state → set `prepareThreshold=0` on the PG driver.
3. **Cap HPA `maxReplicas`** based on the connection budget, and document it in the HPA.
4. **Raise `max_connections`** only as a last resort (each PG connection costs memory/backend process).
5. **Reduce connection need** — caching, read replicas for reporting, shorter transactions, async where appropriate.

**Sound bite:** *"Horizontal scaling multiplies your dependency footprint. Every stateless-service scaling decision is implicitly a decision about the database."* This answer strongly signals real production experience.

---

### Scenario 15 — "A Kafka consumer Spring Boot app is deployed with 3 replicas but a topic has 6 partitions. Then you scale to 10 replicas. What happens?"

- **3 replicas / 6 partitions:** each consumer gets 2 partitions. Fine.
- **10 replicas / 6 partitions:** 6 consumers get one partition each; **4 sit completely idle** — you pay for them and gain nothing. Consumer parallelism is capped by partition count.

**Also happening on every scale event:** a **consumer group rebalance**. With the default eager rebalancing, *all* consumers stop consuming during the rebalance (stop-the-world) → a latency spike and possible duplicate processing after reassignment.

**Fixes:**
1. Cap replicas at partition count (`maxReplicaCount: 6` in KEDA/HPA).
2. Scale on **consumer lag**, not CPU — KEDA Kafka scaler.
3. Use **`CooperativeStickyAssignor`** for incremental rebalancing so unaffected partitions keep flowing:
   ```yaml
   spring.kafka.consumer.properties.partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
   ```
4. Plan partitions for peak parallelism up front (you can add partitions but not remove them, and adding breaks key→partition affinity for ordering).
5. Ensure **idempotent consumers** — rebalances and at-least-once delivery mean reprocessing is normal, not exceptional.
6. Graceful shutdown must let the container commit offsets: `terminationGracePeriodSeconds` > `max.poll.interval.ms` handling time, or you'll reprocess a batch every deploy.

---

### Scenario 16 — "`@Scheduled` job runs 3 times a night. Why, and how do you fix it?"

Because you have 3 replicas, and each JVM has its own scheduler. K8s doesn't know a method is supposed to be a singleton.

**Fixes, best to worst:**
1. **Move it to a `CronJob`** — one pod, one run, retries and history built in, and it doesn't consume the API service's resources. **Preferred answer.**
2. **Distributed lock — ShedLock** (a DB/Redis-backed lock; the winner runs, the rest skip):
   ```java
   @Scheduled(cron = "0 0 2 * * *")
   @SchedulerLock(name = "nightlyReconciliation", lockAtMostFor = "PT30M", lockAtLeastFor = "PT5M")
   public void reconcile() { ... }
   ```
3. **Quartz with a JDBC JobStore** in clustered mode — heavier, but gives you misfire handling and persistence.
4. **Leader election** — K8s `Lease` object (Spring Cloud Kubernetes leader election), or Redis/ZooKeeper.
5. ❌ **Anti-patterns:** a separate profile with `replicas: 1` (no HA, and someone will scale it), or `if (POD_NAME.endsWith("-0"))` (fragile, and Deployments have random names).

**`lockAtMostFor` matters:** if the pod holding the lock is SIGKILLed, the lock must expire or the job never runs again. Mention that.

---

### Scenario 17 — "The cluster is at capacity. `kubectl top` shows nodes at 30% CPU. Explain the contradiction."

**Because the scheduler places pods by `requests`, not by actual usage.** If everyone requests 2 CPUs and uses 200m, nodes are "full" at 10% real utilization. This is the single most common K8s cost problem.

**Diagnose:**
```bash
kubectl describe node <node> | grep -A8 "Allocated resources"   # requests vs capacity
kubectl top nodes                                               # actual usage
# The gap between these two numbers IS the waste.
```
```bash
# find the worst offenders
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory'
```

**Fix:**
1. **VPA in recommendation mode** (or Goldilocks) to get data-driven requests; set requests near p95 of real usage, not a guess.
2. **Fix the guessing culture** — a LimitRange with sane defaults, and a Kyverno/Gatekeeper policy rejecting absurd requests.
3. **Bin-packing** — switch the scheduler to `MostAllocated` scoring, or use Karpenter for consolidation.
4. **Descheduler** to rebalance pods after nodes fill unevenly.
5. **Right node shapes** — a memory-heavy JVM fleet on CPU-optimized nodes wastes CPU by definition.

**Sound bite:** *"Requests are a reservation, not a measurement. Cluster utilization is a request-accuracy problem before it's a capacity problem."*

---

### Scenario 18 — "You deployed a new version and error rates jumped. You have 60 seconds. What do you do?"

**Mitigate first, diagnose second.** Say that explicitly — it's what separates senior from mid.

```bash
kubectl rollout undo deployment/order-service -n prod
kubectl rollout status deployment/order-service -n prod
```
Then, once the bleeding stops:
```bash
kubectl rollout history deployment/order-service
kubectl logs -l app=order-service --tail=200 --since=10m | grep -i error
kubectl get events --sort-by=.lastTimestamp | head -30
```
**Caveats to raise (this is the senior part):**
- `rollout undo` **does not roll back a database migration**. If v2 ran a destructive migration, rolling back the code may break against the new schema — the reason for expand/contract (Scenario 8).
- It also doesn't revert ConfigMap/Secret changes unless they're hash-suffixed or in Git — another argument for GitOps, where rollback is `git revert` of *everything*.
- Check whether the errors are actually caused by the deploy or coincide with a downstream incident; compare error rate by pod version label before assuming.

**Prevent:** canary with automated analysis (Argo Rollouts) so the *system* rolls back in 30s without a human; `--atomic` on Helm; SLO-based alerting on the canary.

---

### Scenario 19 — "Two teams share a cluster. Team A's batch job starves Team B's API. Design the fix."

**Layered answer:**
1. **Namespace isolation + ResourceQuota** per team (cap total requests/limits and object counts).
2. **LimitRange** for defaults so an unset request can't become BestEffort and eat everything.
3. **PriorityClass + preemption** — the API is high priority, batch is low. Under contention, batch pods get preempted:
   ```yaml
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata: { name: high-priority-api }
   value: 1000000
   preemptionPolicy: PreemptLowerPriority
   globalDefault: false
   ```
4. **Guaranteed QoS for the API** (requests == limits) so it's evicted last under node pressure.
5. **Node pool separation + taints/tolerations** — batch runs on its own (possibly spot) node pool; the API can't be affected by a runaway batch job at all. **This is the strongest isolation and usually the right answer.**
6. **CPU limits on batch** to cap throttling damage, and NetworkPolicies so teams can't reach each other's services.
7. **Governance** — chargeback/showback per namespace, so cost pressure fixes behaviour without you policing YAML.

---

### Scenario 20 — "A service needs to talk to a legacy system outside the cluster with a fixed IP allowlist. How?"

Three approaches:
1. **Egress gateway with a static source IP** — route egress through dedicated nodes with an Elastic/static NAT IP (or an Istio egress gateway), then allowlist that IP. Cleanest and most common.
2. **`ExternalName` Service or a Service with manual Endpoints** to give the legacy system a stable in-cluster DNS name, so app config doesn't hardcode IPs:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata: { name: legacy-mainframe }
   spec:
     ports: [{ port: 1414, targetPort: 1414 }]
   ---
   apiVersion: v1
   kind: Endpoints
   metadata: { name: legacy-mainframe }     # name MUST match the Service
   subsets:
     - addresses: [{ ip: 10.50.4.21 }]
       ports: [{ port: 1414 }]
   ```
3. **Dedicated node pool + NAT gateway** with a fixed IP, plus `nodeSelector` for the pods that need it.

Add: NetworkPolicy egress rules so *only* that service can reach the legacy CIDR; and if the legacy system has low connection limits, put a small proxy/pool in front rather than letting 20 replicas hammer it.

---

### Scenario 21 — "Latency p99 spikes every few minutes. CPU and memory look fine. Investigate."

**Ranked hypotheses (JVM-specific — this is where a Java dev shines):**
1. **GC pauses** — check `jvm_gc_pause_seconds_max`. A 2s full GC every 5 minutes matches this pattern exactly. Fix: heap sizing, G1 region tuning, `MaxGCPauseMillis`, or ZGC/Shenandoah for strict p99.
2. **CPU throttling despite "fine" average CPU** — average hides bursts. `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` is the metric that reveals it. Average CPU 40% with 30% of periods throttled is a real scenario.
3. **Connection pool exhaustion** — `hikaricp_connections_pending` spikes while CPU idles; threads are *waiting*, not working.
4. **DNS latency** — `ndots:5` causing extra lookups, or CoreDNS overloaded/conntrack-limited. Fix: `dnsConfig` ndots, NodeLocal DNSCache.
5. **Periodic work colliding** — a `@Scheduled` cache refresh, metrics flush, log rotation, or a nightly cron on the same node.
6. **JIT deoptimization / re-compilation** after a code path shifts.
7. **Neighbour pod bursting** on the same node (`kubectl get pods -o wide --field-selector spec.nodeName=<node>`).

**Method:** correlate the latency spike timestamps against GC pauses, throttling, and pool metrics on one Grafana dashboard. If the spikes align with GC → JVM. If they align with throttling → limits. If they align across *all* pods simultaneously → a shared dependency, not the pods.

**Definitive tool:** JFR recording during a spike window shows GC, thread stalls, lock contention, and socket read times in one file.

---

### Scenario 22 — "How would you migrate a monolithic Spring Boot app on VMs to Kubernetes?"

Give a phased plan, not a big bang:

**Phase 0 — Assess.** Inventory state: local file writes, in-memory HTTP sessions, cron jobs, hardcoded hostnames/IPs, singleton assumptions, log files, licensing tied to MAC/IP.
**Phase 1 — Twelve-factor the app.** Externalize config (env/ConfigMap), log to stdout, move sessions to Redis (`spring-session-data-redis`), move file storage to S3, add Actuator health/metrics, verify graceful shutdown, upgrade Java to a container-aware version (≥ 11, ideally 17/21).
**Phase 2 — Containerize.** Layered multi-stage Dockerfile, non-root, image scanning in CI, and run it in a local kind/minikube cluster.
**Phase 3 — Lift and shift.** Deploy the monolith *as-is* to K8s (Deployment + Service + Ingress + probes + resources). Don't refactor and re-platform simultaneously — you won't know which change broke what.
**Phase 4 — Operationalize.** HPA, PDB, topology spread, dashboards, alerts, log aggregation, tracing, runbooks. Run in parallel with the VM deployment; shift traffic gradually at the DNS/LB layer with a fast rollback path.
**Phase 5 — Decompose (only if needed).** Strangler-fig pattern: extract high-change or independently-scaling modules into separate services one at a time, keeping the monolith as the fallback. **Say explicitly that "monolith on K8s" is a perfectly valid end state** if the business doesn't need microservices — that maturity impresses interviewers.

**Risks to name:** sticky sessions, batch jobs double-running (Scenario 16), stateful file dependencies, the DB becoming the new bottleneck once you can scale the app freely, and the team's operational readiness (on-call needs to understand K8s before you cut over).

---

### Scenario 23 — "A pod is stuck in `Terminating` for 20 minutes. What's going on?"

**Causes, ranked:**
1. **App ignores SIGTERM** and `terminationGracePeriodSeconds` is huge (or 0 for the wrong reasons) — kubelet is waiting.
2. **Volume won't unmount** — a CSI driver issue, or the process still has the mount busy. Very common with NFS.
3. **A finalizer is blocking deletion** — `kubectl get pod <pod> -o yaml | grep -A3 finalizers`.
4. **kubelet on the node is dead/unreachable** — nobody is there to confirm the pod is gone.
5. **A preStop hook that hangs** (e.g., `curl` to an unreachable endpoint with no timeout).

**Actions:**
```bash
kubectl describe pod <pod>                     # events tell you which of the above
kubectl get pod <pod> -o yaml | yq '.metadata.finalizers, .spec.terminationGracePeriodSeconds'
kubectl delete pod <pod> --grace-period=0 --force    # LAST resort
```
**Warn about `--force`:** it removes the object from etcd without confirming the container stopped. For a StatefulSet with an RWO volume, you can end up with **two pods writing the same disk** (split brain) — data corruption. Say this; it shows you understand *why* the safeguards exist.

---

### Scenario 24 — "How do you handle secrets rotation without downtime?"

**Design goal: rotation must not require a deploy.**

1. **Overlapping validity** — the credential provider issues the new secret while the old is still valid. Every rotation strategy depends on this window.
2. **External Secrets Operator with `refreshInterval`** → updates the K8s Secret automatically from Vault/AWS SM.
3. **Volume-mounted secrets** (not env) so the file content updates in place (~60s) without a restart.
4. **App re-reads on change** — for a datasource, that means a `DataSource` wrapper that fetches credentials per connection, plus a short HikariCP `maxLifetime` so connections cycle onto the new credential naturally. Or use Vault dynamic DB credentials with `spring-cloud-vault`.
5. **If the app can't re-read:** trigger a rolling restart on secret change (Reloader/Stakater watches Secrets and annotates the Deployment).
6. **For TLS:** cert-manager renews certs into a Secret; a sidecar or the app watches the file and reloads the SSL context.
7. **Audit + alerting** on secret age; treat "secret older than 90 days" as a violation.

**Sound bite:** *"Any rotation design that requires a deploy will get skipped under deadline pressure. Automate it or it won't happen."*

---

### Scenario 25 — "Your team's deploys take 25 minutes for a 10-replica service. Speed it up."

**Break the time down first** (always quantify before optimizing):
```
CI build+test    ?  → parallelize tests, cache ~/.m2, incremental builds
image build      ?  → layered JAR + BuildKit cache, or Jib
image push/pull  ?  → smaller image; regional registry; pre-pull via DaemonSet
rollout          ?  → the K8s part
```

**Rollout math:** with `maxSurge: 1, maxUnavailable: 0`, pods go out **one at a time**, and each takes `startupProbe delay + boot time + minReadySeconds`. 10 pods × (60s boot + 15s minReady + 15s preStop) ≈ 15 minutes. That's the bottleneck.

**Fixes:**
1. `maxSurge: 3` (or `25%`) so 3 pods roll in parallel — cuts it ~3×. Keep `maxUnavailable: 0` for safety.
2. **Cut Spring Boot startup time** — the biggest lever (AppCDS, lazy init, fewer autoconfigs, trimmed classpath).
3. **Smaller images + pre-pulled base layers** on nodes.
4. **Tune `minReadySeconds`** to something evidence-based, not 60.
5. Don't set `initialDelaySeconds` high on readiness — use a `startupProbe` with a short period so readiness passes the instant the app is up.

**Trade-off to voice:** *"Faster rollout means more pods in flight, so a bad build reaches more users before I notice. I'd rather get to a fast rollout plus an automated canary than just crank maxSurge."*

---

### Scenario 26 — "How do you decide requests and limits for a new Spring Boot service with no production data?"

**Method, not a magic number:**
1. **Start from the JVM.** Decide max heap from load testing (say 768 MB), then `memory limit ≈ heap / 0.7` ≈ **1.1 Gi → round to 1.25Gi**, with `MaxRAMPercentage=70`.
2. **CPU:** measure locally under load. A typical REST service needs `request: 500m`, `limit: 2` — enough to burst through JIT warm-up.
3. **Load test in a K8s environment with the real limits set.** Testing on a laptop with 8 cores tells you nothing about a 500m pod.
4. **Set memory request == limit** (Guaranteed QoS) and make CPU burstable.
5. **Deploy conservatively high, then shrink with data** — over-provisioning costs money; under-provisioning costs an incident. Start safe.
6. **After 2 weeks:** set requests at ~p95 of observed usage (VPA recommendations or Prometheus `quantile_over_time`). Keep limits with headroom for spikes.
7. **Re-evaluate after major changes** — a new library that adds direct memory (Netty, a caching layer) changes the arithmetic.

**Numbers to have ready as a sanity anchor:** a plain Spring Boot REST service with G1 typically needs **512 Mi–1 Gi memory** and **250–500m CPU request**, needs ~**1 CPU** to start in a reasonable time, and boots in 20–60s. Quoting realistic ranges signals experience.

---

### Scenario 27 — "Health checks pass, but the app is silently not processing Kafka messages. How would you have caught this?"

**Because liveness only proves the HTTP server responds.** A consumer whose container has stopped polling (paused, deserialization error loop, rebalance stuck, thread died) is "healthy" by HTTP standards. This is a **liveness blind spot** — a great topic to discuss.

**Detection:**
1. **Alert on consumer lag** (`kafka_consumergroup_lag`) — the real signal, and it works even if the app is completely wedged. Primary answer.
2. **A custom health indicator** that inspects the listener container state:
   ```java
   @Component
   @RequiredArgsConstructor
   public class KafkaConsumerHealthIndicator implements HealthIndicator {
       private final KafkaListenerEndpointRegistry registry;

       @Override
       public Health health() {
           boolean allRunning = registry.getListenerContainers().stream()
                   .allMatch(MessageListenerContainer::isRunning);
           return allRunning ? Health.up().build()
                             : Health.down().withDetail("reason", "listener container stopped").build();
       }
   }
   ```
   Include it in the **readiness** group (safe) — and only in liveness if you're confident a restart is the right remedy.
3. **Heartbeat metric** — `lastMessageProcessedAt` gauge; alert if stale beyond an expected interval. Works for low-traffic topics where lag is a poor signal.
4. **Dead-letter topic monitoring** — a spike in DLT means messages are failing, not missing.
5. **Set `ContainerStoppingErrorHandler` or `DefaultErrorHandler` deliberately** so a poison pill doesn't silently kill the container.

**Sound bite:** *"Liveness answers 'is the process alive', not 'is the work getting done'. For async workloads I always add a work-progress signal."*

---

### Scenario 28 — "Half your pods are in one AZ. That AZ goes down. Design so this doesn't hurt."

```yaml
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: order-service } }
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: order-service } }
```
Plus:
- **Node pool per AZ**, with the Cluster Autoscaler configured to balance across them.
- **PDB** (`minAvailable: 4` of 6) so a drain can't compound the outage.
- **Capacity headroom:** if you run 6 pods across 3 AZs at 90% utilization, losing one AZ leaves 4 pods carrying 100% of traffic → they fall over. Provision for **N-1 zone capacity** — this is the point most candidates miss.
- **Multi-AZ dependencies:** an RDS instance in the failed AZ, or an RWO volume there, defeats app-level spread. Multi-AZ RDS, cross-zone LB, and no zone-pinned storage for stateless services.
- **Cross-AZ latency and cost:** with `topologyAwareHints`/`trafficDistribution: PreferClose`, traffic prefers same-zone endpoints — good for latency and data transfer cost, but verify it doesn't create hot spots during a partial outage.

---

### Scenario 29 — "You inherit a cluster with no monitoring, no limits, and `:latest` tags everywhere. What do you fix first?"

Prioritize by **blast radius**, and say why:

**Week 1 — Visibility (you can't fix what you can't see).**
- Prometheus + Grafana + Alertmanager; log aggregation; basic alerts on pod restarts, OOMKills, error rate, p99, node pressure.

**Week 2 — Stop the bleeding.**
- Pin image tags (immutable version+SHA) so behaviour becomes reproducible. This also makes rollback possible, which is the real win.
- Add resource requests to the top 10 workloads (a BestEffort pod is first to be evicted and can OOM a node).
- Add readiness probes — instantly fixes deploy-time 502s.

**Week 3 — Reliability.**
- Liveness/startup probes (correctly configured), graceful shutdown + preStop, PDBs, topology spread, `maxUnavailable: 0`.

**Week 4+ — Governance.**
- Move manifests into Git; GitOps with ArgoCD; Kyverno policies (no `:latest`, requests required, non-root); RBAC least privilege; NetworkPolicy default-deny; image scanning in CI; etcd/backup and a **tested** restore.

**Framing that lands well:** *"I'd write the one-page current-state risk list, get agreement on the top three, and fix those with the team rather than unilaterally rewriting everyone's YAML. Adoption matters more than correctness here — a policy nobody understands gets bypassed."*

---

### Scenario 30 — "Your service must not lose in-flight work during a deploy. It processes long-running (5-minute) jobs. Design it."

**Key insight: a 5-minute task cannot be drained inside a normal grace period, so redesign rather than tune timeouts.**

**Option A — Make the work resumable (best).** Persist job state in a DB with a status column. On SIGTERM, stop accepting new work, checkpoint progress, exit. A new pod picks up `IN_PROGRESS` jobs whose lease expired. This survives OOMKills and node failures too, which timeout tuning never will.

**Option B — Long grace period.** `terminationGracePeriodSeconds: 400` + graceful shutdown, with the executor set to `waitForTasksToCompleteOnShutdown`. Works, but slows every deploy, doesn't survive node loss or SIGKILL, and PDB/drain operations get painful.

**Option C — Move long work out of the request path.** The API accepts and enqueues (Kafka/SQS/DB); a separate worker Deployment (or a K8s **Job per task**) does the processing. The API deploys in seconds; workers use at-least-once delivery + **idempotent processing**, so a killed worker just means the message is redelivered. **This is the answer I'd advocate.**

**Whichever you pick, say the invariant:** *"I design for the pod dying at any moment without warning — SIGKILL, node loss, OOM. If correctness depends on a clean shutdown, the design is wrong. Graceful shutdown should be an optimization, not a requirement."* That's a genuinely senior closing statement.

---

## 20. Troubleshooting Playbook & Command Cheat Sheet

### The universal debug flow

```
Symptom
  ├── Pod not starting?
  │     ├── Pending          → describe pod → Events → scheduling/quota/PVC
  │     ├── ImagePullBackOff → describe pod → registry/tag/credentials
  │     ├── CrashLoopBackOff → logs --previous → exit code
  │     │     ├── 137 + OOMKilled → memory limit vs total RSS
  │     │     ├── 1              → app/config error
  │     │     └── 143            → probe or grace-period kill
  │     ├── Init:Error         → logs -c <init-container>
  │     └── CreateContainerConfigError → missing ConfigMap/Secret key
  ├── Pod running but not serving?
  │     ├── 0/1 Ready  → readiness probe → /actuator/health
  │     └── 1/1 + 503  → no probe / health indicator DOWN / wrong profile
  ├── Can't reach the service?
  │     └── endpoints → DNS → port → NetworkPolicy → Ingress  (see Scenario 7)
  └── Slow / flaky?
        └── GC pauses → CPU throttling → pool exhaustion → DNS → neighbours
```

### Commands worth memorizing

```bash
# --- Inspect ---
kubectl get pods -o wide --sort-by=.status.startTime
kubectl get pods --field-selector status.phase!=Running -A
kubectl describe pod <pod>                        # ALWAYS start here; read Events
kubectl get events --sort-by=.lastTimestamp -A | tail -40
kubectl get pod <pod> -o yaml | less              # ground truth

# --- Logs ---
kubectl logs <pod> -f --tail=100
kubectl logs <pod> --previous                     # crashed container's logs
kubectl logs -l app=order-service --all-containers --since=15m --prefix
kubectl logs <pod> -c <container>

# --- Interact ---
kubectl exec -it <pod> -- sh
kubectl port-forward <pod> 8080:8080
kubectl port-forward svc/order-service 8080:8080
kubectl cp <pod>:/dumps/heap.hprof ./heap.hprof
kubectl debug -it <pod> --image=nicolaka/netshoot --target=app

# --- Rollouts ---
kubectl rollout status/history/undo/restart deployment/<name>
kubectl set image deployment/<name> app=repo/app:2.14.1 --record
kubectl scale deployment/<name> --replicas=6

# --- Resources ---
kubectl top nodes; kubectl top pods --containers --sort-by=memory
kubectl describe node <node> | grep -A8 "Allocated resources"
kubectl get pdb,hpa,resourcequota -A

# --- Networking ---
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl get svc,ingress -A
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash

# --- RBAC ---
kubectl auth can-i list secrets --as=system:serviceaccount:orders:order-service -n orders
kubectl auth can-i --list -n orders

# --- Config debugging ---
kubectl get cm <name> -o jsonpath='{.data}' | jq
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d
kubectl exec <pod> -- env | sort

# --- Safety net ---
kubectl apply --dry-run=server -f deployment.yaml
kubectl diff -f deployment.yaml                   # what would change
kubectl explain deployment.spec.strategy --recursive
```

### JVM-in-pod commands

```bash
kubectl exec <pod> -- java -XX:+PrintFlagsFinal -version | grep -Ei 'MaxHeapSize|UseG1GC|ActiveProcessorCount'
kubectl exec <pod> -- cat /sys/fs/cgroup/memory.max          # cgroup v2
kubectl exec <pod> -- jcmd 1 VM.flags
kubectl exec <pod> -- jcmd 1 Thread.print
kubectl exec <pod> -- jcmd 1 GC.heap_info
kubectl exec <pod> -- jcmd 1 VM.native_memory summary
kubectl exec <pod> -- curl -s localhost:8081/actuator/health | jq
kubectl exec <pod> -- curl -s localhost:8081/actuator/metrics/jvm.memory.used | jq
kubectl exec <pod> -- curl -s localhost:8081/actuator/env | jq '.propertySources[].name'
```

---

## 21. Rapid-Fire Revision

### The 20 facts most likely to be tested

1. **Liveness fails → restart. Readiness fails → removed from endpoints. Startup fails → killed after threshold.**
2. **Never point liveness at `/actuator/health`** (aggregate) — use `/actuator/health/liveness`. Never check dependencies in liveness.
3. **Exit 137 = OOMKilled/SIGKILL. Exit 143 = SIGTERM.**
4. **Heap ≠ container memory.** Budget `limit ≈ heap / 0.7–0.75`; use `-XX:MaxRAMPercentage`, not `-Xmx`.
5. **`UseContainerSupport` is default since Java 10** (8u191+). Old Java 8 sees host memory/CPU → instant OOMKill.
6. **CPU over limit = throttled; memory over limit = killed.**
7. **HPA CPU % is relative to the request, not the limit.** No requests = HPA can't work.
8. **QoS: Guaranteed (req==lim) > Burstable > BestEffort** — eviction order is the reverse.
9. **A Service is DNS + iptables rules, not a proxy.** Only **Ready** pods get traffic.
10. **DNS format:** `service.namespace.svc.cluster.local`.
11. **Zero-downtime deploy = `maxUnavailable: 0` + readiness probe + `preStop sleep 15` + `server.shutdown: graceful` + a grace period longer than both.**
12. **ENTRYPOINT must be exec form** so the JVM is PID 1 and receives SIGTERM.
13. **Env vars and `subPath` mounts never live-update.** Volume mounts do (~60s), but Spring won't re-read without refresh. Use a config checksum annotation.
14. **Secrets are base64, not encrypted.** Enable etcd encryption-at-rest; prefer an external secret store.
15. **RWO is per-node, not per-pod.** RWO + RollingUpdate can deadlock → use `Recreate` or a StatefulSet.
16. **Rolling updates run two versions at once** → every DB/API change must be backward compatible (expand/contract).
17. **Deployment = stateless/cattle. StatefulSet = stable identity, DNS, and per-pod PVC.**
18. **PDB protects against voluntary disruptions only** (drains/upgrades), not node crashes.
19. **N replicas × HikariCP pool size ≤ DB max_connections.** Scaling out is a database decision.
20. **`@Scheduled` runs once per replica** → CronJob, ShedLock, or leader election.

### Fill-in-the-blanks (test yourself)

| Question | Answer |
|---|---|
| Default `terminationGracePeriodSeconds`? | 30 |
| Default HPA sync period? | ~15s |
| NodePort range? | 30000–32767 |
| CrashLoopBackOff max backoff? | 5 minutes |
| Max ConfigMap/Secret size? | ~1 MiB |
| Node marked NotReady after? | ~40s |
| Default pod eviction toleration on NotReady? | 300s |
| Default rolling update maxSurge/maxUnavailable? | 25% / 25% |
| Which component talks to etcd? | Only kube-apiserver |
| Which component actually starts containers? | kubelet (via CRI) |
| Which probe type is useless for Spring Boot? | tcpSocket (port binds before app is ready) |
| Where do heap dumps go? | An emptyDir volume, then `kubectl cp` |

### Questions to ask *them* (shows seniority)

- How do you handle database migrations relative to rolling deployments?
- Do you use canary/progressive delivery, and is rollback automated on metrics?
- How are resource requests decided — measured or estimated?
- How do you manage secrets, and what does rotation look like?
- What's your on-call story for a pod-level incident: runbooks, dashboards, ownership?
- Are you on Ingress or migrating to Gateway API? Any service mesh?
- Helm, Kustomize, or GitOps — and who owns the manifests, devs or platform?

### 60-second closing summary you can recite

> *"For a Spring Boot service on Kubernetes, the things I always get right are: a layered multi-stage image running non-root with the JVM as PID 1; `MaxRAMPercentage` sized against the container limit with memory requests equal to limits; a startup probe so slow boots aren't mistaken for crashes, readiness pointed at `/actuator/health/readiness` as the traffic gate, and liveness pointed at `/actuator/health/liveness` with no dependency checks; graceful shutdown with a preStop sleep and a grace period that covers it; `maxUnavailable: 0` with a PDB and topology spread; config from ConfigMaps with a checksum annotation so changes roll the pods, and secrets from an external store; and Micrometer metrics plus structured JSON logs to stdout with trace correlation. Beyond that, the discipline that matters most is that every change — schema, API, or config — has to be backward compatible, because a rolling update always means two versions are running at the same time."*

---

**Good luck. Practice the C-O-H-M-F-P structure out loud on three scenarios — structure under pressure is what gets remembered.**
