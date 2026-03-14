# ☸ Kubernetes — Complete Beginner's Notes
### Based on: Jerney Blog Platform | From Zero to Interview-Ready

---

## Table of Contents

1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [The Jerney Project — What Are We Deploying?](#2-the-jerney-project)
3. [Namespace](#3-namespace)
4. [Secrets](#4-secrets)
5. [Storage — StorageClass & PVC](#5-storage)
6. [Deployments](#6-deployments)
7. [Services](#7-services)
8. [NetworkPolicy](#8-networkpolicy)
9. [Tolerations, Taints & Karpenter](#9-tolerations-taints--karpenter)
10. [Essential kubectl Commands](#10-essential-kubectl-commands)
11. [Interview Questions & Answers](#11-interview-questions--answers)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)
13. [What to Improve for Production](#13-what-to-improve-for-production)

---

## 1. What is Kubernetes?

### 1.1 The Problem It Solves

Imagine you wrote a web application. It runs fine on your laptop. You deploy it to one server. Traffic grows — one server isn't enough. You need 10 servers. Now you have questions:

- Which server does the app run on?
- What if one server crashes at 3am?
- How do you update the app without downtime?
- How do you scale up during peak hours and scale down to save money?
- How do services like backend and database talk to each other securely?

Doing this manually is painful and doesn't scale. **This is the problem Kubernetes solves.**

> **One-Line Definition:** Kubernetes (K8s — 8 letters between K and s) is an open-source system that automates deploying, scaling, and managing containerised applications across a cluster of machines.

---

### 1.2 Containers — The Foundation

Before Kubernetes, you need to understand containers. A container is a lightweight, self-contained package that includes your app code plus everything it needs — runtime, libraries, config files.

| Concept | What it means |
|---|---|
| Virtual Machine (VM) | An entire OS boots up. Heavy (GBs). Slow to start. Isolated at hardware level. |
| Container | Shares the host OS kernel. Lightweight (MBs). Starts in seconds. Isolated at process level. |
| Docker | The most popular tool for building and running containers. |
| Container Image | A snapshot/template of a container. Like a class in programming — running it creates a container (instance). |

---

### 1.3 Kubernetes Architecture — The Big Picture

A Kubernetes setup is called a **cluster**. It has two types of machines:

#### 🏛️ Control Plane (The Brain)
Lives on master nodes. Makes decisions about the cluster — scheduling, scaling, responding to failures.

| Component | Role |
|---|---|
| API Server | The front door. Every `kubectl` command hits this. |
| etcd | A key-value store. The single source of truth for all cluster state. |
| Scheduler | Decides which node to place new pods on. |
| Controller Manager | Runs control loops that watch state and fix things (e.g. restarts crashed pods). |

#### ⚙️ Worker Nodes (The Muscle)
These machines actually run your applications.

| Component | Role |
|---|---|
| kubelet | Agent that talks to the API server and manages pods on its node. |
| kube-proxy | Handles networking rules so pods can communicate. |
| Container Runtime | Usually containerd or Docker. Actually runs containers. |

---

### 1.4 Key Kubernetes Concepts at a Glance

| Concept | What it does |
|---|---|
| Pod | Smallest deployable unit in K8s. Wraps one or more containers. |
| Deployment | Manages Pods. Ensures the right number are running. Handles rolling updates. |
| Service | Gives Pods a stable network endpoint. Pods can die and restart with new IPs — Services don't change. |
| Namespace | Virtual cluster inside a cluster. Used for isolation and organisation. |
| ConfigMap | Stores non-secret configuration data as key-value pairs. |
| Secret | Stores sensitive data (passwords, keys) in base64 encoded form. |
| PersistentVolume | Storage in the cluster that outlives individual pods. |
| PersistentVolumeClaim | A request for storage by a Pod. |
| StorageClass | Defines how storage is provisioned (e.g. AWS EBS gp3 SSD). |
| NetworkPolicy | Firewall rules for pods. Controls which pods can talk to which. |
| Ingress | Routes external HTTP/HTTPS traffic to internal Services. |
| Node | A physical or virtual machine in the cluster. |

---

## 2. The Jerney Project

### 2.1 Application Architecture

Jerney is a **three-tier blog platform** — three separate layers, each doing a specific job:

| Layer | Role |
|---|---|
| Frontend | The web interface users see. Runs Nginx on port 8080. Talks to the backend to get/send data. |
| Backend | The API layer. Handles business logic. Exposes REST endpoints (like `/api/health`). Runs on port 5000. |
| Database | PostgreSQL. Stores all blog data permanently. Runs on port 5432. |

**Data Flow:**
```
User's browser → Frontend (8080) → Backend (5000) → PostgreSQL (5432)
```
Each arrow goes only one way — Frontend never talks directly to the database. This is enforced by NetworkPolicy.

---

### 2.2 Folder Structure Explained

```
k8s/
├── namespace/       → Isolation boundary for all resources
├── secrets/         → Sensitive credentials (DB password etc.)
├── storage/         → Disk storage definition (StorageClass + PVC)
├── database/        → PostgreSQL Deployment + Service
├── backend/         → Backend API Deployment + Service
├── frontend/        → Frontend Deployment + Service
├── network/         → NetworkPolicy rules (who talks to whom)
└── jerney.yaml      → Optional: single file with everything combined
```

This is a best-practice pattern. In real teams, different engineers own different folders, and GitOps tools like ArgoCD can sync them independently.

---

## 3. Namespace

### 3.1 What is a Namespace?

A Namespace divides a single Kubernetes cluster into multiple **virtual clusters**. Think of it like folders on your computer — files with the same name in different folders won't conflict.

> **Analogy:** A large office building where multiple companies rent space. Each company has its own floor — isolated, with their own rules, but sharing the same building infrastructure (electricity, internet).

### 3.2 Why Use Namespaces?

- **Isolation** — Resources in namespace A can't accidentally interfere with namespace B
- **Organisation** — Group related resources together (all Jerney resources live in `jerney`)
- **Access Control** — RBAC can be scoped to a namespace
- **Resource Quotas** — Limit CPU/memory per namespace
- **Multiple Environments** — Run dev, staging, production in the same cluster using separate namespaces

### 3.3 The Namespace YAML — Annotated

```yaml
apiVersion: v1           # The API version for this resource type
kind: Namespace          # What type of Kubernetes object this is
metadata:
  name: jerney           # The namespace name — all resources reference this
  labels:
    app.kubernetes.io/part-of: jerney   # A label for grouping/filtering
```

**Every YAML file has these top fields:**

| Field | Purpose |
|---|---|
| `apiVersion` | Which version of the K8s API to use. Different types use different versions (`v1`, `apps/v1`, `networking.k8s.io/v1`, etc.) |
| `kind` | What type of object you're creating (Namespace, Deployment, Service, etc.) |
| `metadata` | Info about the object: name, namespace it belongs to, labels, annotations |
| `spec` | The desired state — what you want Kubernetes to create/maintain |

### 3.4 Labels — The Kubernetes Tagging System

Labels are key-value pairs attached to any Kubernetes object. Used for:

- **Selecting** — Services use label selectors to find which Pods to route traffic to
- **Filtering** — `kubectl get pods -l app=backend`
- **Grouping** — `app.kubernetes.io/*` is a standard set of recommended label keys

**Standard labels used in Jerney:**
```
app.kubernetes.io/name       → identifies the specific component (jerney-db, jerney-backend)
app.kubernetes.io/part-of    → identifies the larger application (jerney)
app.kubernetes.io/component  → describes the role (database, backend, frontend)
```

---

## 4. Secrets

### 4.1 What is a Secret?

A Secret is a Kubernetes object that stores sensitive information such as passwords, API keys, or TLS certificates. Kubernetes keeps them separate from your application code so you don't hardcode passwords in your container images.

> ⚠️ **Important:** Secrets are **NOT encrypted** by default. They are base64 encoded — which is just a text encoding, not encryption. Anyone with `kubectl` access to the namespace can read them.
>
> In production, use:
> - **External Secrets Operator (ESO)** — syncs secrets from AWS Secrets Manager / HashiCorp Vault
> - **Sealed Secrets** — encrypts secrets so they can be safely stored in Git

### 4.2 The Secret YAML — Annotated

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jerney-db-secret
  namespace: jerney        # Must be in same namespace as Pods that use it
type: Opaque               # Generic secret (most common type)
data:
  POSTGRES_USER: amVybmV5X3VzZXI=            # base64 of 'jerney_user'
  POSTGRES_PASSWORD: amVybmV5X3Bhc3NfMjAyNg==  # base64 of 'jerney_pass_2026'
  POSTGRES_DB: amVybmV5X2Ri                  # base64 of 'jerney_db'
```

### 4.3 How to Encode / Decode base64

```bash
# Encode a value
echo -n 'jerney_user' | base64
# Output: amVybmV5X3VzZXI=

# Decode a value
echo 'amVybmV5X3VzZXI=' | base64 --decode
# Output: jerney_user

# The -n flag is critical: without it, echo adds a newline which changes the output
```

### 4.4 How Pods Consume Secrets

**Method 1 — `secretRef` (load all keys as env vars):**
```yaml
envFrom:
  - secretRef:
      name: jerney-db-secret
# Result: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB are all
# available as environment variables inside the container.
```

**Method 2 — `secretKeyRef` (load individual keys, rename them):**
```yaml
env:
  - name: DB_USER          # The env var name inside the container
    valueFrom:
      secretKeyRef:
        name: jerney-db-secret   # Secret object name
        key: POSTGRES_USER       # Key within the secret
```

The backend uses Method 2 — picking individual keys and renaming them (e.g. `POSTGRES_USER` becomes `DB_USER`). More explicit and recommended for clarity.

---

## 5. Storage

### 5.1 Why Databases Need Special Storage

Containers are ephemeral — when a Pod dies, everything written to its local filesystem disappears. For a database this is catastrophic. You need storage that:

- **Persists** — survives Pod restarts and node failures
- **Is Durable** — data is not lost when containers are replaced
- **Is Attachable** — can be detached from one Pod and re-attached to a replacement

### 5.2 The Storage Hierarchy

```
StorageClass
    ↓ defines HOW to provision
PersistentVolume (PV)
    ↓ an actual piece of storage (auto-created by StorageClass)
PersistentVolumeClaim (PVC)
    ↓ a Pod's REQUEST for storage (K8s binds a matching PV)
Pod
    ↓ mounts the PVC at a path in the container filesystem
```

### 5.3 StorageClass YAML — Annotated

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jerney-ebs-sc
provisioner: ebs.csi.eks.amazonaws.com   # AWS EBS CSI driver — tells K8s HOW to create the disk
parameters:
  type: gp3          # General Purpose SSD v3 — better performance than gp2
  encrypted: "true"  # Encrypt the disk at rest using AWS KMS
reclaimPolicy: Retain           # When PVC is deleted, KEEP the underlying disk (don't delete data!)
volumeBindingMode: WaitForFirstConsumer  # Don't create disk until a Pod actually needs it
allowVolumeExpansion: true       # Allow resizing the disk later without recreating it
```

| Setting | Explanation |
|---|---|
| `reclaimPolicy: Retain` | When PVC is deleted, the EBS volume is kept. You must manually delete it. Safest for databases. |
| `reclaimPolicy: Delete` | When PVC is deleted, the disk is also deleted. Risky — data is gone. |
| `WaitForFirstConsumer` | Provision disk only when a Pod is scheduled. Ensures disk is in the same AZ as the Pod — critical for EBS. |
| `Immediate` | Provision disk as soon as PVC is created. May end up in wrong AZ. |

### 5.4 PersistentVolumeClaim YAML — Annotated

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jerney-db-pvc
  namespace: jerney
spec:
  accessModes:
    - ReadWriteOnce    # Only ONE Node can mount this volume at a time (correct for EBS)
  storageClassName: jerney-ebs-sc   # Which StorageClass to use
  resources:
    requests:
      storage: 10Gi   # Request 10 Gigabytes of disk space
```

**Access Modes:**

| Mode | Meaning |
|---|---|
| `ReadWriteOnce (RWO)` | One node can read AND write. Required for block storage like EBS. |
| `ReadOnlyMany (ROX)` | Multiple nodes can read. No writes. |
| `ReadWriteMany (RWX)` | Multiple nodes can read AND write simultaneously. Requires shared storage like EFS/NFS. |

**How PVC connects to a Pod:**
```yaml
# In the Deployment spec:
volumes:
  - name: db-storage
    persistentVolumeClaim:
      claimName: jerney-db-pvc   # Reference the PVC by name

# In the container spec:
volumeMounts:
  - name: db-storage
    mountPath: /var/lib/postgresql/data
    subPath: pgdata    # Mount only a subdirectory (avoids 'lost+found' issue)
```

---

## 6. Deployments

### 6.1 What is a Deployment?

A Deployment is the most common way to run stateless applications in Kubernetes. It manages a set of identical Pods and ensures:

- The desired number of Pods are always running (self-healing)
- Rolling updates can be performed with zero downtime
- You can roll back to a previous version if something goes wrong

> **Relationship:** `Deployment → ReplicaSet → Pod → Container`
>
> You almost always create a Deployment — not Pods directly. The Deployment handles creating and replacing Pods automatically.

### 6.2 PostgreSQL Deployment — Deep Dive

```yaml
apiVersion: apps/v1          # Deployments live in the apps/v1 API group
kind: Deployment
metadata:
  name: jerney-db
  namespace: jerney
spec:
  replicas: 1                # Run exactly 1 Pod (DB must be single instance with EBS)
  strategy:
    type: Recreate           # Kill existing Pod BEFORE starting new one
                             # Required when using ReadWriteOnce volumes
  selector:
    matchLabels:
      app.kubernetes.io/name: jerney-db   # Manages Pods with this label
  template:                  # Template for creating Pods
    metadata:
      labels:
        app.kubernetes.io/name: jerney-db  # Label on Pod (must match selector above)
    spec:
      automountServiceAccountToken: false  # Security: DB doesn't need K8s API access
      containers:
        - name: postgres
          image: postgres:16-alpine        # Official Postgres, Alpine = smaller image
          ports:
            - containerPort: 5432
```

**Update Strategies:**

| Strategy | Behaviour |
|---|---|
| `RollingUpdate` (default) | Gradually replaces old Pods with new ones. Zero downtime. Cannot be used with RWO volumes — old and new Pods would fight over the same disk. |
| `Recreate` | Kills ALL old Pods first, then creates new ones. Brief downtime. Required here because EBS is ReadWriteOnce. |

### 6.3 Init Containers — Wait for DB

The backend uses an `initContainer` to wait for the database before starting. This prevents connection errors on startup.

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36       # Tiny utility image
    command:
      - sh
      - -c
      - |
        echo "Waiting for jerney-db:5432..."
        until nc -z jerney-db 5432; do   # nc = netcat, tests TCP connection
          echo "DB not ready, retrying in 3s..."
          sleep 3
        done
        echo "DB is ready!"
# The main container ONLY starts after ALL initContainers complete successfully.
```

> **Init Container vs Regular Container:**
> - Init Containers run BEFORE main containers and must complete successfully
> - Regular Containers run in parallel, indefinitely, until the Pod is terminated
> - Use cases: waiting for dependencies, running DB migrations, fetching config

### 6.4 Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: 250m       # Guaranteed minimum CPU: 250 millicores = 0.25 of a CPU core
    memory: 256Mi   # Guaranteed minimum RAM: 256 Mebibytes
  limits:
    cpu: 500m       # Maximum CPU: 500 millicores = 0.5 of a CPU core
    memory: 512Mi   # Maximum RAM. If exceeded, container is OOMKilled (restarted)
```

| Concept | Meaning |
|---|---|
| `requests` | The Scheduler guarantees this amount will be available. Used for placement decisions. |
| `limits` | The maximum the container can use. CPU is throttled. Memory causes OOMKill (restart). |
| `250m CPU` | 250 millicores. 1000m = 1 full CPU core. So 250m = a quarter of a core. |
| `256Mi` | 256 Mebibytes (binary). Use `Mi` not `M` in K8s to be precise. |

### 6.5 Health Probes — Liveness and Readiness

Kubernetes uses probes to check whether a container is alive and ready to serve traffic. Without probes, K8s only knows your process is running — not that your app actually works.

| Probe Type | What happens on failure |
|---|---|
| `livenessProbe` | Container is **restarted**. Detects deadlocks and hung processes. |
| `readinessProbe` | Pod is **removed from Service endpoints** — no traffic until it recovers. |
| `startupProbe` | Overrides liveness during slow startup. Protects slow-starting apps. |

```yaml
# Backend — HTTP probe (checks the /api/health endpoint)
livenessProbe:
  httpGet:
    path: /api/health
    port: 5000
  initialDelaySeconds: 10  # Wait 10s before first check (give app time to start)
  periodSeconds: 15        # Check every 15 seconds
  timeoutSeconds: 5        # Fail if no response within 5 seconds

# PostgreSQL — exec probe (runs pg_isready inside the container)
livenessProbe:
  exec:
    command: ["pg_isready", "-U", "jerney_user", "-d", "jerney_db"]
  initialDelaySeconds: 15
  periodSeconds: 10
```

### 6.6 Security Context

A `securityContext` defines security settings for a Pod or container. It follows the **principle of least privilege** — give the app only the permissions it absolutely needs.

```yaml
securityContext:
  allowPrivilegeEscalation: false  # Cannot gain more privileges than parent process
  readOnlyRootFilesystem: true     # Container filesystem is read-only (prevents tampering)
  runAsNonRoot: true               # Must NOT run as root user
  runAsUser: 1000                  # Run as UID 1000 (not root/0)
  capabilities:
    drop:
      - ALL                        # Drop ALL Linux capabilities (least privilege)
```

> **Why Each Setting Matters:**
> - `allowPrivilegeEscalation: false` — prevents sudo-like escalation inside the container
> - `readOnlyRootFilesystem: true` — if an attacker gets in, they can't install tools or modify files
> - `runAsNonRoot: true` — process can't access root-only files
> - `drop: ALL` — removes capabilities like `NET_ADMIN`, `SYS_PTRACE` that attackers exploit
>
> **Note:** PostgreSQL cannot use these strict settings because it needs root to initialise the database. Nginx needs write access for PID files and cache.

---

## 7. Services

### 7.1 Why Services Exist

Pods are ephemeral. They die, restart, get replaced. Every time a new Pod starts, it gets a **new IP address**. If your frontend hardcoded the backend Pod's IP, it would break every time the backend restarted.

A Service provides a **stable virtual IP (ClusterIP) and DNS name** that never changes. It load-balances traffic across all healthy Pods matching its selector.

> **Service Discovery:** Every Service gets a DNS name automatically:
> ```
> <service-name>.<namespace>.svc.cluster.local
> ```
> Within the same namespace, just use the service name:
> - `jerney-db` → resolves to the database ClusterIP
> - `jerney-backend` → resolves to the backend ClusterIP
>
> This is why the backend uses `DB_HOST=jerney-db` — it's a DNS name, not an IP.

### 7.2 Service Types

| Type | When to use it |
|---|---|
| `ClusterIP` (default) | Only accessible INSIDE the cluster. Used for DB and backend — no business being on the internet. |
| `NodePort` | Exposes on a static port on every Node's IP. Accessible externally. Used for Jerney's frontend. |
| `LoadBalancer` | Provisions a cloud load balancer (AWS ELB). Production way to expose services externally. |
| `ExternalName` | Maps a service to an external DNS name. Used to refer to external services by an internal name. |

### 7.3 Service YAML — Annotated

```yaml
# ClusterIP Service (Database — internal only)
apiVersion: v1
kind: Service
metadata:
  name: jerney-db
  namespace: jerney
spec:
  type: ClusterIP         # Default. Only reachable within cluster.
  selector:
    app.kubernetes.io/name: jerney-db   # Route traffic to Pods with this label
  ports:
    - port: 5432          # Port the Service listens on
      targetPort: 5432    # Port on the Pod to forward to
      protocol: TCP
```

```yaml
# NodePort Service (Frontend — external access)
spec:
  type: NodePort
  ports:
    - port: 80            # Service port (inside cluster)
      targetPort: 8080    # Container port (where Nginx actually listens)
      protocol: TCP
      # nodePort: auto-assigned in range 30000-32767

# Access via port-forward:
# kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
```

### 7.4 How Selectors Link Services to Pods

The Service finds its target Pods using a **label selector**. It routes traffic to all Pods whose labels match.

```yaml
# Service selector:
selector:
  app.kubernetes.io/name: jerney-backend

# Pod labels (in Deployment template):
labels:
  app.kubernetes.io/name: jerney-backend    ← This Pod matches!
  app.kubernetes.io/part-of: jerney
```

---

## 8. NetworkPolicy

### 8.1 What Problem Does NetworkPolicy Solve?

By default, every Pod in a Kubernetes cluster can communicate with every other Pod — across namespaces. This is a security problem. If a frontend container gets compromised, the attacker could directly query the database.

NetworkPolicy lets you define **explicit firewall rules at the Pod level**.

> **Jerney's Network Security Model:**
> ```
> ✅ Internet/User  →  Frontend Pod   (allowed)
> ✅ Frontend Pod   →  Backend Pod    (allowed via NetworkPolicy)
> ✅ Backend Pod    →  Database Pod   (allowed via NetworkPolicy)
>
> ❌ Frontend Pod   →  Database Pod   (BLOCKED — no direct DB access)
> ❌ External       →  Backend Pod    (BLOCKED — internal only)
> ❌ External       →  Database Pod   (BLOCKED — internal only)
> ```

### 8.2 Database NetworkPolicy — Annotated

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jerney-db-allow-backend-only
  namespace: jerney
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jerney-db  # This policy APPLIES TO the database pods
  policyTypes:
    - Ingress          # We're controlling INCOMING traffic to the DB
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: jerney-backend  # Only BACKEND pods allowed in
      ports:
        - port: 5432   # Only on this port
          protocol: TCP
```

### 8.3 NetworkPolicy Concepts

| Field | Meaning |
|---|---|
| `podSelector` | Which pods this policy applies to. Empty selector = ALL pods in namespace. |
| `policyTypes` | `Ingress` = control incoming traffic. `Egress` = control outgoing traffic. |
| `ingress.from` | Allowed traffic sources. Can be `podSelector`, `namespaceSelector`, or `ipBlock`. |
| `egress.to` | Allowed traffic destinations. |
| `ports` | Which ports/protocols to allow. If omitted, all ports are allowed. |

> ⚠️ **NetworkPolicy Requirement:** Requires a CNI plugin that supports it — Calico, Cilium, or Weave Net. The default AWS EKS VPC CNI does **NOT** support NetworkPolicy. Always verify your CNI before relying on this for security.

---

## 9. Tolerations, Taints & Karpenter

### 9.1 Taints and Tolerations

They work together to control which Pods can be scheduled on which Nodes.

- **Taint** — applied to a Node. Repels Pods from being scheduled there unless they tolerate it.
- **Toleration** — applied to a Pod. Says "I'm okay with being placed on a node with this taint."

> **Analogy:** A node has a sign: *"Warning: This node may be disrupted."* A Pod with a toleration says: *"I've read the warning and I'm okay with it."* Pods without the toleration won't be scheduled there.

### 9.2 Karpenter Toleration in Jerney

```yaml
tolerations:
  - key: "karpenter.sh/disrupted"
    operator: "Exists"     # Tolerate any value — just the key existing is enough
    effect: "NoSchedule"   # The taint effect to tolerate
```

**Karpenter** is a node auto-provisioner for AWS EKS. When it wants to remove a node (to save costs), it first taints it with `karpenter.sh/disrupted:NoSchedule`. By tolerating this taint, these Pods agree to be rescheduled if their node gets disrupted.

| Taint Effect | Behaviour |
|---|---|
| `NoSchedule` | Don't schedule NEW pods here, but don't evict existing ones. |
| `PreferNoSchedule` | Try to avoid scheduling here, but allow it if necessary. |
| `NoExecute` | Evict existing pods immediately if they don't tolerate this. Don't schedule new ones. |

---

## 10. Essential kubectl Commands

### Applying the Manifest

```bash
# Apply a single file
kubectl apply -f namespace/namespace.yaml

# Apply all files in a directory (recursively)
kubectl apply -f k8s/ -R

# Apply in correct order (dependencies first)
kubectl apply -f namespace/
kubectl apply -f secrets/
kubectl apply -f storage/
kubectl apply -f database/
kubectl apply -f backend/
kubectl apply -f frontend/
kubectl apply -f network/
```

### Checking Status

```bash
# Get all resources in a namespace
kubectl get all -n jerney

# Get Pods
kubectl get pods -n jerney
kubectl get pods -n jerney -w    # Watch for changes in real time

# Describe a resource (shows events, conditions, full details)
kubectl describe pod <pod-name> -n jerney
kubectl describe deployment jerney-backend -n jerney

# Get logs
kubectl logs <pod-name> -n jerney
kubectl logs <pod-name> -n jerney -f         # Follow (like tail -f)
kubectl logs <pod-name> -n jerney -c postgres  # Specific container in a pod
```

### Accessing the Frontend

```bash
# Port-forward the frontend service to your local machine
kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
# Then open: http://localhost:8080
```

### Debugging

```bash
# Execute a command inside a running container (shell access)
kubectl exec -it <pod-name> -n jerney -- /bin/sh

# Check cluster events (very useful for debugging stuck Pods)
kubectl get events -n jerney --sort-by='.lastTimestamp'

# Check PVC status (Pending = no matching StorageClass or binding issue)
kubectl get pvc -n jerney

# Decode a secret value
kubectl get secret jerney-db-secret -n jerney \
  -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
```

### Useful Shortcuts

| Command / Flag | What it does |
|---|---|
| `kubectl get po` | Short for: `kubectl get pods` |
| `kubectl get svc` | Short for: `kubectl get services` |
| `kubectl get deploy` | Short for: `kubectl get deployments` |
| `kubectl get ns` | Short for: `kubectl get namespaces` |
| `-n <namespace>` | Specify namespace |
| `--all-namespaces` or `-A` | Show resources from all namespaces |
| `-o yaml` | Output full YAML of a resource |
| `-o wide` | Extra columns like Node name and Pod IP |
| `kubectl diff -f file.yaml` | Show what would change before applying |
| `kubectl rollout status deploy/jerney-backend -n jerney` | Watch a rollout's progress |
| `kubectl rollout undo deploy/jerney-backend -n jerney` | Roll back to previous version |

---

## 11. Core Concepts

**Q: What is Kubernetes and why is it used?**
> Kubernetes is an open-source container orchestration platform. It automates deploying, scaling, and managing containerised applications. We use it because manually managing containers across multiple servers is error-prone and doesn't scale. K8s handles self-healing (auto-restart on failure), horizontal scaling, rolling updates, service discovery, and storage management.

**Q: What's the difference between a Pod and a Deployment?**
> A Pod is the smallest deployable unit — it wraps one or more containers. But if you create a Pod directly and it dies, it stays dead. A Deployment manages Pods — it ensures the desired number are always running, handles rolling updates, and can roll back. In Jerney, we never create Pods directly; we always use Deployments.

**Q: What is a Service and why do we need it?**
> Pods get new IP addresses every time they restart. A Service provides a stable ClusterIP and DNS name that stays constant. It load-balances traffic across all healthy Pods matching its selector. Without Services, applications would need to track Pod IPs manually — which change constantly.

**Q: What is a Namespace? Why use one?**
> A Namespace is a virtual cluster inside a K8s cluster. It provides isolation, organisation, and access control boundaries. The Jerney app runs in the `jerney` namespace — all its resources are grouped there. This prevents naming conflicts with other apps and allows RBAC and resource quotas scoped to just Jerney.

---

### Storage

**Q: Why does a database need a PersistentVolumeClaim?**
> Containers are ephemeral — their local storage is deleted when the Pod dies. For a database, this would mean losing all data on every restart. A PVC requests persistent storage that outlives Pods. In Jerney, we use an AWS EBS (gp3) volume, so PostgreSQL's data directory persists even when the Pod is recreated.

**Q: What is a StorageClass? What is `reclaimPolicy: Retain`?**
> A StorageClass defines HOW storage gets provisioned — which provisioner (e.g. AWS EBS CSI), which disk type (gp3), whether it's encrypted. `reclaimPolicy: Retain` means when the PVC is deleted, the underlying EBS volume is NOT automatically deleted. Critical for production — you don't want to accidentally delete data if someone runs `kubectl delete pvc`.

---

### Security

**Q: How are secrets managed in Kubernetes?**
> Kubernetes Secrets store sensitive data as base64-encoded values. In Jerney, the DB credentials are stored in `jerney-db-secret`. The database Pod uses `envFrom` to load all keys as env vars. The backend uses `secretKeyRef` to load individual keys. Important caveat: base64 is NOT encryption. In production, you'd use External Secrets Operator with AWS Secrets Manager, or Sealed Secrets.

**Q: What is a NetworkPolicy? How is it used in Jerney?**
> NetworkPolicy is a Kubernetes firewall for Pods. By default, all Pods can talk to all other Pods. Jerney has two policies: (1) the database only accepts traffic from the backend pods on port 5432; (2) the backend only accepts traffic from frontend pods on port 5000. This means even if the frontend is compromised, it cannot directly query the database.

**Q: What is `securityContext` and why use it?**
> `securityContext` defines security settings for a Pod or container. The Jerney backend uses: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, and drops ALL Linux capabilities. This follows the principle of least privilege — if the container is compromised, the attacker has minimal power.

---

### Deployments & Operations

**Q: What is the difference between `RollingUpdate` and `Recreate`?**
> `RollingUpdate` (default) gradually replaces old Pods with new ones — zero downtime. `Recreate` terminates ALL old Pods first, then creates new ones — brief downtime. Jerney's database uses `Recreate` because the EBS volume is `ReadWriteOnce`. If we used `RollingUpdate`, the old and new DB Pods would fight over the same volume.

**Q: What are liveness and readiness probes?**
> **Liveness probe:** Checks if the app is still alive. If it fails, Kubernetes restarts the container. Detects deadlocks. **Readiness probe:** Checks if the app is ready to serve traffic. If it fails, the Pod is removed from the Service's endpoints — no traffic goes to it until it recovers. Jerney's backend uses HTTP probes on `/api/health`. PostgreSQL uses exec probes running `pg_isready`.

**Q: What are resource requests and limits?**
> **Requests:** the guaranteed minimum CPU/memory the Scheduler sets aside. **Limits:** the maximum a container can use. CPU is throttled if exceeded. Memory: if a container exceeds its limit, it's OOMKilled (restarted). Requests affect scheduling — a Pod is only placed on a Node with enough free resources.

**Q: What are Init Containers?**
> Init Containers run before the main containers in a Pod and must complete successfully. The Jerney backend uses one that loops on `nc -z jerney-db 5432` until the database is reachable — preventing the backend from starting and failing with connection errors if the DB isn't ready yet.

**Q: What are taints and tolerations?**
> Taints are applied to Nodes to repel Pods. Tolerations are applied to Pods to allow them on tainted Nodes. Jerney's Pods tolerate `karpenter.sh/disrupted:NoSchedule` — because Karpenter (EKS auto-scaler) taints nodes before removing them. The toleration means these Pods accept being on nodes that might be disrupted and will be gracefully rescheduled.

---

## 12. Quick Reference Cheat Sheet

### Objects in Jerney

| Kind | File | Purpose |
|---|---|---|
| Namespace | `namespace/namespace.yaml` | Isolation boundary |
| Secret | `secrets/db-secret.yaml` | Sensitive credentials |
| StorageClass | `storage/storage-class.yaml` | How to provision disks |
| PersistentVolumeClaim | `storage/postgres-pvc.yaml` | Request for disk space |
| Deployment | `*-deployment.yaml` × 3 | Manage running Pods |
| Service | `*-service.yaml` × 3 | Stable network endpoint for Pods |
| NetworkPolicy | `network/*.yaml` × 2 | Pod-level firewall rules |

### Port Map

| Component | Port |
|---|---|
| Frontend (Nginx) | 8080 (container) / 80 (service) |
| Backend (API) | 5000 (container + service) |
| Database (PostgreSQL) | 5432 (container + service) |

### apiVersion Reference

| apiVersion | Resource Types |
|---|---|
| `v1` | Namespace, Pod, Service, Secret, PVC, ConfigMap |
| `apps/v1` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `storage.k8s.io/v1` | StorageClass |
| `networking.k8s.io/v1` | NetworkPolicy, Ingress |
| `rbac.authorization.k8s.io/v1` | Role, RoleBinding, ClusterRole |

### Probe Types

| Type | Success condition |
|---|---|
| `httpGet` | HTTP response code 200–399 |
| `exec` | Command exits with code 0 |
| `tcpSocket` | TCP connection established |
| `grpc` | gRPC health check passes |

### Kubernetes Glossary

| Term | Meaning |
|---|---|
| etcd | Distributed key-value store — stores all cluster state |
| kubelet | Agent on each node. Manages pods. Talks to API server. |
| kube-proxy | Handles iptables/IPVS rules for Service networking on each node |
| CNI | Container Network Interface — plugin for pod networking |
| CSI | Container Storage Interface — plugin for storage (e.g. EBS CSI driver) |
| CRD | Custom Resource Definition — extend K8s with your own object types |
| RBAC | Role-Based Access Control — who can do what in the cluster |
| Helm | Package manager for K8s — like npm but for K8s manifests |
| ArgoCD | GitOps tool — applies K8s manifests from a Git repo automatically |
| Karpenter | Node auto-provisioner for EKS — scales nodes up/down automatically |
| HPA | Horizontal Pod Autoscaler — scales Pod count based on CPU/memory/custom metrics |
| VPA | Vertical Pod Autoscaler — adjusts resource requests/limits automatically |
| StatefulSet | Like Deployment but for stateful apps — stable Pod names and storage |
| DaemonSet | Runs one Pod on every Node — used for monitoring agents, log collectors |
| Job / CronJob | Run a task once (Job) or on a schedule (CronJob) |

---

## 13. What to Improve for Production

The Jerney manifest is excellent for learning but has several things to upgrade for real production use:

### Secrets Management
- **Problem:** Secrets are base64 encoded in a YAML file. Not encrypted. Not safe to commit to Git.
- **Solution:** Use [External Secrets Operator](https://external-secrets.io/) + AWS Secrets Manager, or [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets).

### Database — Use StatefulSet
- **Problem:** Deployment + RWO PVC works but StatefulSet is the right K8s primitive for stateful apps.
- **Solution:** StatefulSet provides stable Pod names (`jerney-db-0`), stable network identity, and ordered Pod creation/deletion.

### Frontend — Use Ingress Instead of NodePort
- **Problem:** NodePort exposes raw ports on every Node. No TLS. No path-based routing. Not production-grade.
- **Solution:** Use an Ingress controller (AWS ALB Ingress Controller, Nginx Ingress) with a proper domain and TLS certificate.

### Add Pod Disruption Budget (PDB)
- **Problem:** During node maintenance, K8s could evict all backend Pods simultaneously causing downtime.
- **Solution:** PodDisruptionBudget ensures a minimum number of Pods stay running during voluntary disruptions.

### Add Horizontal Pod Autoscaler (HPA)
- **Problem:** `replicas: 2` is fixed. Traffic spikes need more Pods. Off-hours waste money.
- **Solution:** HPA automatically scales Pod count between min/max based on CPU or custom metrics.

### Add Egress NetworkPolicy
- **Problem:** Current policies only control Ingress. A compromised Pod can still make outbound connections to the internet.
- **Solution:** Add Egress rules — backend can only call the DB, frontend can only call the backend.

---

*End of Notes — Thank you for reading! ☸*