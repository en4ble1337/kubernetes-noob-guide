# Kubernetes Components Explained 🎯

A beginner-friendly guide to understanding what runs in a Kubernetes cluster and why.

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Control Plane Components](#control-plane-components)
  - [kube-apiserver](#kube-apiserver)
  - [etcd](#etcd)
  - [kube-scheduler](#kube-scheduler)
  - [kube-controller-manager](#kube-controller-manager)
  - [cloud-controller-manager](#cloud-controller-manager)
- [Node Components](#node-components)
  - [kubelet](#kubelet)
  - [kube-proxy](#kube-proxy)
  - [Container Runtime](#container-runtime)
- [Add-on Components](#add-on-components)
  - [CoreDNS](#coredns)
  - [Calico (Network Plugin)](#calico-network-plugin)
- [Application Components](#application-components)
  - [Pods](#pods)
  - [Deployments](#deployments)
  - [ReplicaSets](#replicasets)
  - [Services](#services)
- [How They All Work Together](#how-they-all-work-together)
- [Quick Reference](#quick-reference)

---

## The Big Picture

Think of a Kubernetes cluster like a **factory**:

- **Control Plane** = Management office (makes decisions, keeps records)
- **Worker Nodes** = Factory floor (does the actual work)
- **Pods** = Workers (run your applications)
- **Services** = Reception desk (directs traffic to the right worker)

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE NODE                      │
│  ┌──────────┐  ┌──────┐  ┌───────────┐  ┌───────────────┐ │
│  │   API    │  │ etcd │  │ Scheduler │  │  Controller   │ │
│  │  Server  │  │      │  │           │  │   Manager     │ │
│  └──────────┘  └──────┘  └───────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
┌───────▼───────────┐                    ┌──────────▼──────────┐
│   WORKER NODE 1   │                    │   WORKER NODE 2     │
│  ┌────────────┐   │                    │  ┌────────────┐     │
│  │  kubelet   │   │                    │  │  kubelet   │     │
│  │ kube-proxy │   │                    │  │ kube-proxy │     │
│  │ containerd │   │                    │  │ containerd │     │
│  └────────────┘   │                    │  └────────────┘     │
│  ┌─────┐  ┌─────┐ │                    │  ┌─────┐  ┌─────┐  │
│  │ Pod │  │ Pod │ │                    │  │ Pod │  │ Pod │  │
│  └─────┘  └─────┘ │                    │  └─────┘  └─────┘  │
└───────────────────┘                    └────────────────────┘
```

---

## Control Plane Components

The "brain" of Kubernetes - runs on the control plane node (c1-cp1 in our setup).

### kube-apiserver

**What it is:** The front door to Kubernetes.

**What it does:**
- Receives all your `kubectl` commands
- Acts as the central communication hub
- Validates and processes API requests
- Everyone (other components, users) talks through this

**Real-world analogy:** Like a receptionist who handles all calls and directs them to the right department.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep apiserver
kubectl logs -n kube-system kube-apiserver-c1-cp1
```

**Why it matters:** Without the API server, nothing in Kubernetes works. It's the most critical component.

---

### etcd

**What it is:** The cluster's database/memory.

**What it does:**
- Stores ALL cluster data (configuration, state, secrets)
- Keeps track of what's running, where, and how
- Backs up the entire cluster state

**Real-world analogy:** Like the company's filing cabinet that stores all important documents.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep etcd
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 member list
```

**Why it matters:** If etcd is lost, you lose all cluster configuration. Always backup etcd!

---

### kube-scheduler

**What it is:** The cluster's resource planner.

**What it does:**
- Decides which node should run each new pod
- Considers: available resources, node health, affinity rules
- Assigns pods to nodes (but doesn't start them)

**Real-world analogy:** Like a project manager assigning tasks to team members based on their workload and skills.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep scheduler
kubectl logs -n kube-system kube-scheduler-c1-cp1
```

**Why it matters:** Ensures efficient resource usage and workload distribution across nodes.

---

### kube-controller-manager

**What it is:** The cluster's autopilot system.

**What it does:**
- Runs multiple "controllers" (background processes)
- Watches the cluster state and makes corrections
- Ensures desired state matches actual state

**Built-in controllers:**
- **Node Controller:** Monitors node health
- **Replication Controller:** Maintains correct pod count
- **Endpoints Controller:** Connects services to pods
- **Service Account Controller:** Creates default accounts

**Real-world analogy:** Like a quality control inspector who constantly checks if everything is working as it should and fixes problems.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep controller-manager
kubectl logs -n kube-system kube-controller-manager-c1-cp1
```

**Why it matters:** Keeps your applications running even when things fail.

---

### cloud-controller-manager

**What it is:** Bridge to cloud providers (AWS, Azure, GCP).

**What it does:**
- Manages cloud-specific resources
- Handles load balancers, storage, networking in cloud
- Only present in cloud-managed Kubernetes (AKS, EKS, GKE)

**Real-world analogy:** Like a liaison between your company and external vendors.

**Why it matters:** Makes Kubernetes work seamlessly with cloud services.

---

## Node Components

These run on **every node** (including the control plane) - the "workers" of Kubernetes.

### kubelet

**What it is:** The node's agent/manager.

**What it does:**
- Ensures containers are running in pods
- Talks to the API server
- Reports node and pod status
- Starts and stops containers based on instructions

**Real-world analogy:** Like a floor supervisor who ensures workers (containers) are doing their jobs.

**How to check it:**
```bash
# Check kubelet status
sudo systemctl status kubelet

# View kubelet logs
sudo journalctl -u kubelet -f

# Check kubelet version
kubelet --version
```

**Why it matters:** Without kubelet, pods can't run on a node. It's the executor of the API server's commands.

---

### kube-proxy

**What it is:** The node's network traffic controller.

**What it does:**
- Maintains network rules on nodes
- Routes traffic to correct pods
- Implements Kubernetes Services
- Handles load balancing

**Real-world analogy:** Like a mail sorter who ensures packages reach the right destination.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx
```

**Why it matters:** Enables pod-to-pod and external-to-pod communication.

---

### Container Runtime

**What it is:** The software that actually runs containers.

**Common runtimes:**
- **containerd** (used in our setup)
- Docker Engine
- CRI-O

**What it does:**
- Pulls container images
- Starts and stops containers
- Manages container lifecycle

**Real-world analogy:** Like the tools and equipment that workers use to do their job.

**How to check it:**
```bash
# Check containerd status
sudo systemctl status containerd

# List running containers
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

# Check containerd logs
sudo journalctl -u containerd -f
```

**Why it matters:** Without a container runtime, Kubernetes can't run containers.

---

## Add-on Components

Optional but commonly used components that enhance cluster functionality.

### CoreDNS

**What it is:** The cluster's internal DNS server.

**What it does:**
- Resolves service names to IP addresses
- Allows pods to find services by name
- Example: `my-service` → `10.96.0.100`

**Real-world analogy:** Like a phone directory that translates names to phone numbers.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system coredns-xxxxx

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

**Why it matters:** Makes service discovery easy - use names instead of IP addresses.

---

### Calico (Network Plugin)

**What it is:** Pod networking solution (CNI plugin).

**What it does:**
- Assigns IP addresses to pods
- Enables pod-to-pod communication
- Implements network policies (firewall rules)

**Real-world analogy:** Like the internal phone system that connects all departments.

**How to check it:**
```bash
kubectl get pods -n kube-system | grep calico
kubectl logs -n kube-system calico-node-xxxxx

# Check network configuration
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDR:.spec.podCIDR
```

**Why it matters:** Without a network plugin, pods can't communicate.

---

## Application Components

These are what you create and deploy - your actual applications.

### Pods

**What it is:** The smallest deployable unit in Kubernetes.

**What it does:**
- Wraps one or more containers
- Shares network and storage
- Has its own IP address
- Ephemeral (can be deleted and recreated)

**Real-world analogy:** Like a single worker or a small team working together on one task.

**Key characteristics:**
- **One IP per pod** (shared by all containers)
- **Shared storage** (volumes)
- **Scheduled together** (always on same node)
- **Lifecycle bound** (start and stop together)

**How to check it:**
```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```

**Example structure:**
```
┌─────────────────────────────┐
│         POD (IP: 10.244.1.5)│
│  ┌────────────────────────┐ │
│  │   Main Container       │ │
│  │   (Your app)           │ │
│  └────────────────────────┘ │
│  ┌────────────────────────┐ │
│  │   Sidecar Container    │ │
│  │   (Optional helper)    │ │
│  └────────────────────────┘ │
│  ┌────────────────────────┐ │
│  │   Shared Volume        │ │
│  └────────────────────────┘ │
└─────────────────────────────┘
```

**Why it matters:** Pods are where your applications actually run.

---

### Deployments

**What it is:** A controller that manages pods.

**What it does:**
- Declares desired state (how many pods, which image)
- Creates and manages ReplicaSets
- Handles rolling updates
- Enables rollbacks

**Real-world analogy:** Like a hiring manager who ensures you always have the right number of employees.

**How to check it:**
```bash
kubectl get deployments
kubectl describe deployment <deployment-name>
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
```

**Example:**
```yaml
Deployment (desired: 3 replicas)
  └─> ReplicaSet (manages pods)
        ├─> Pod 1
        ├─> Pod 2
        └─> Pod 3
```

**Why it matters:** Provides self-healing, scaling, and updates for your applications.

---

### ReplicaSets

**What it is:** Ensures a specified number of pod replicas are running.

**What it does:**
- Maintains pod count
- Creates new pods if some fail
- Deletes excess pods
- Usually managed by Deployments (don't create manually)

**Real-world analogy:** Like a supervisor ensuring there are always 5 workers on the production line.

**How to check it:**
```bash
kubectl get replicasets
kubectl describe replicaset <replicaset-name>
```

**Why it matters:** Provides high availability by maintaining multiple pod copies.

---

### Services

**What it is:** A stable network endpoint for accessing pods.

**What it does:**
- Provides a consistent IP and DNS name
- Load balances traffic across pods
- Survives pod restarts (pods get new IPs, service IP stays same)

**Service Types:**

1. **ClusterIP** (default)
   - Internal only
   - Accessible within cluster
   - Example: `my-database:3306`

2. **NodePort**
   - Exposes on each node's IP
   - Accessible externally
   - Example: `<NodeIP>:30080`

3. **LoadBalancer**
   - Cloud load balancer
   - Public IP address
   - Example: Cloud provider assigns `52.123.45.67`

**Real-world analogy:** Like a company phone number that routes to available employees (pods).

**How to check it:**
```bash
kubectl get services
kubectl describe service <service-name>
kubectl get endpoints <service-name>
```

**Example flow:**
```
External User → Service (10.96.0.100:80) → Load Balances to:
                                              ├─> Pod 1 (10.244.1.5:8080)
                                              ├─> Pod 2 (10.244.1.6:8080)
                                              └─> Pod 3 (10.244.2.4:8080)
```

**Why it matters:** Services provide stable networking for dynamic pod environments.

---

## How They All Work Together

Let's trace what happens when you run: `kubectl create deployment nginx --image=nginx`

### Step-by-Step Flow:

```
1. YOU
   └─> Run: kubectl create deployment nginx --image=nginx
        │
        ▼
2. API SERVER (kube-apiserver)
   └─> Receives request
   └─> Validates request
   └─> Stores in etcd
        │
        ▼
3. ETCD
   └─> Saves: "Deployment named 'nginx' should exist"
        │
        ▼
4. CONTROLLER MANAGER
   └─> Notices: New deployment created
   └─> Creates: ReplicaSet
        │
        ▼
5. REPLICASET CONTROLLER
   └─> Notices: Need to create pods
   └─> Creates: Pod definition
        │
        ▼
6. SCHEDULER
   └─> Notices: Unassigned pod exists
   └─> Decides: Best node for the pod (e.g., c1-node1)
   └─> Assigns: Pod to c1-node1
        │
        ▼
7. KUBELET (on c1-node1)
   └─> Notices: New pod assigned to me
   └─> Instructs: Container runtime (containerd)
        │
        ▼
8. CONTAINERD
   └─> Pulls: nginx image from registry
   └─> Creates: Container
   └─> Starts: Container
        │
        ▼
9. KUBELET
   └─> Monitors: Container health
   └─> Reports: Status back to API server
        │
        ▼
10. KUBE-PROXY
    └─> Updates: Network rules
    └─> Enables: Pod networking
         │
         ▼
11. YOUR POD IS RUNNING! 🎉
```

### Real-World Analogy:

Imagine ordering food delivery:

1. **You** (kubectl) → Order food via app
2. **Restaurant Manager** (API Server) → Receives order
3. **Order Book** (etcd) → Records order
4. **Kitchen Manager** (Controller Manager) → Sees new order
5. **Chef Scheduler** (Scheduler) → Assigns available chef
6. **Chef** (kubelet) → Starts cooking
7. **Kitchen Tools** (containerd) → Used to prepare food
8. **Delivery Person** (kube-proxy) → Delivers to you

---

## Quick Reference

### Control Plane (Brain) - Runs on Control Plane Node

| Component | Purpose | Analogy |
|-----------|---------|---------|
| **kube-apiserver** | Front door, handles all requests | Receptionist |
| **etcd** | Database, stores cluster state | Filing cabinet |
| **kube-scheduler** | Decides where to run pods | Project manager |
| **kube-controller-manager** | Ensures desired state | Quality control |
| **cloud-controller-manager** | Cloud integration | Cloud liaison |

### Node Components (Workers) - Runs on All Nodes

| Component | Purpose | Analogy |
|-----------|---------|---------|
| **kubelet** | Manages pods on node | Floor supervisor |
| **kube-proxy** | Network routing | Mail sorter |
| **Container Runtime** | Runs containers | Worker tools |

### Add-ons - Optional but Important

| Component | Purpose | Analogy |
|-----------|---------|---------|
| **CoreDNS** | Service name resolution | Phone directory |
| **Calico** | Pod networking | Internal phone system |

### Your Applications - What You Deploy

| Component | Purpose | Analogy |
|-----------|---------|---------|
| **Pod** | Smallest unit, runs containers | Worker/team |
| **Deployment** | Manages pods, updates | Hiring manager |
| **ReplicaSet** | Maintains pod count | Supervisor |
| **Service** | Stable network endpoint | Company phone number |

---

## Visual Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                           │
│                                                                 │
│  ┌─────────────────── CONTROL PLANE ───────────────────┐       │
│  │                                                      │       │
│  │  📞 API Server (Front Door)                         │       │
│  │  📚 etcd (Database)                                  │       │
│  │  📋 Scheduler (Task Assigner)                       │       │
│  │  ⚙️  Controller Manager (Autopilot)                 │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                           │                                     │
│                           │ Commands                            │
│                           ▼                                     │
│  ┌────────────────── WORKER NODES ─────────────────────┐       │
│  │                                                      │       │
│  │  Node 1                      Node 2                 │       │
│  │  ├─ 👨‍💼 kubelet               ├─ 👨‍💼 kubelet           │       │
│  │  ├─ 📮 kube-proxy            ├─ 📮 kube-proxy       │       │
│  │  ├─ 🔧 containerd            ├─ 🔧 containerd       │       │
│  │  │                            │                     │       │
│  │  └─ Pods:                     └─ Pods:              │       │
│  │     ┌────────┐                  ┌────────┐          │       │
│  │     │  🚀 App│                  │  🚀 App│          │       │
│  │     │Container│                 │Container│         │       │
│  │     └────────┘                  └────────┘          │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  🌐 Services (Load Balancers)                                  │
│  📛 CoreDNS (Name Resolution)                                  │
│  🔌 Calico (Networking)                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Check Your Understanding

Run these commands to see components in action:

```bash
# See all system components
kubectl get pods -n kube-system

# See your applications
kubectl get pods

# See everything
kubectl get all --all-namespaces

# Describe a specific component
kubectl describe pod <pod-name> -n kube-system

# View logs
kubectl logs <pod-name> -n kube-system
```

---

## Remember

- **Control Plane** = Makes decisions (brain)
- **Nodes** = Do the work (muscles)
- **Pods** = Run your apps (workers)
- **Services** = Provide stable access (reception)
- **Everything talks through the API Server**
- **etcd stores everything**
- **Controllers keep things running**

---

**Next Steps:** Now that you understand what runs where, try the main deployment guide to build your own cluster! https://github.com/en4ble1337/kubernetes-noob-guide

**Version**: Kubernetes v1.29.1
**Last Updated**: October 2025
