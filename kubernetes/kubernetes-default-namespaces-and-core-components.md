# Kubernetes Default Namespaces & Core Components Guide

A reference for the namespaces and core components that every Kubernetes cluster
ships with out of the box — what they are, where they live, and what they do.

---

## Table of Contents

- [Kubernetes Default Namespaces \& Core Components Guide](#kubernetes-default-namespaces--core-components-guide)
  - [Table of Contents](#table-of-contents)
  - [Default Namespaces](#default-namespaces)
    - [`default`](#default)
    - [`kube-system`](#kube-system)
    - [`kube-public`](#kube-public)
    - [`kube-node-lease`](#kube-node-lease)
  - [Core Components by Namespace](#core-components-by-namespace)
  - [Node-Level Components](#node-level-components)
  - [Static Pods](#static-pods)
  - [CoreDNS](#coredns)
    - [DNS Naming Pattern](#dns-naming-pattern)
    - [Quick Checks](#quick-checks)
  - [kube-proxy](#kube-proxy)
    - [Modes](#modes)
    - [Quick Checks](#quick-checks-1)
  - [CNI Plugin](#cni-plugin)
  - [metrics-server](#metrics-server)
  - [kubeadm Bootstrap](#kubeadm-bootstrap)
    - [`kubeadm init` (first control-plane node)](#kubeadm-init-first-control-plane-node)
    - [`kubeadm join` (worker / additional nodes)](#kubeadm-join-worker--additional-nodes)
    - [The chicken-and-egg problem it solves](#the-chicken-and-egg-problem-it-solves)
  - [Namespaced vs. Cluster-Scoped Resources](#namespaced-vs-cluster-scoped-resources)
  - [Per-Namespace Governance](#per-namespace-governance)
  - [Deleting Namespaces (and the `Terminating` Gotcha)](#deleting-namespaces-and-the-terminating-gotcha)
  - [Quick Reference](#quick-reference)
  - [Useful Commands](#useful-commands)

---

## Default Namespaces

Every Kubernetes cluster is created with four namespaces:

| Namespace | Purpose |
|-----------|---------|
| `default` | The namespace where your resources go if you don't specify one. Meant for user workloads in small/simple clusters. |
| `kube-system` | Houses all control plane and cluster infrastructure components. You generally don't deploy your own workloads here. |
| `kube-public` | Readable by **all** users, including unauthenticated ones. Used for cluster-wide public information. |
| `kube-node-lease` | Holds `Lease` objects (one per node) used for node heartbeats. |

### `default`

The catch-all namespace for objects with no other namespace assigned. Using it is
fine for experimentation, but production setups usually create dedicated
namespaces per team/app for isolation and RBAC.

### `kube-system`

Contains the components that make the cluster work:

- `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`
  (as [static Pods](#static-pods) on self-managed control planes)
- `coredns` — cluster DNS / service discovery
- `kube-proxy` — Service networking
- CNI plugins (Calico, Cilium, Flannel, etc.)
- Cloud / metrics / logging add-ons

> ⚠️ Avoid deploying application workloads here. Mistakes in `kube-system` can
> destabilize the whole cluster.

### `kube-public`

A namespace readable by everyone — even unauthenticated clients. In practice it
contains a single ConfigMap, `cluster-info`, which holds the API server endpoint
and CA certificate. This is used during `kubeadm` bootstrap so new nodes can
discover and trust the cluster before they have credentials.

```bash
kubectl get configmap cluster-info -n kube-public -o yaml
```

### `kube-node-lease`

Holds one `Lease` object per node. Each node's kubelet periodically renews its
Lease as a lightweight **heartbeat**. The control plane uses these Leases to
determine node liveness instead of repeatedly updating the (heavier) Node object,
which dramatically improves scalability in large clusters.

```bash
kubectl get leases -n kube-node-lease
```

---

## Core Components by Namespace

| Component | Namespace | Runs As | Function |
|-----------|-----------|---------|----------|
| `kube-apiserver` | `kube-system` | Static Pod | Front door to the cluster; serves the Kubernetes API. |
| `etcd` | `kube-system` | Static Pod | Distributed key-value store; the cluster's source of truth. |
| `kube-scheduler` | `kube-system` | Static Pod | Assigns Pods to nodes. |
| `kube-controller-manager` | `kube-system` | Static Pod | Runs controllers (Deployment, Node, etc.). |
| `cloud-controller-manager` | `kube-system` | Static Pod | Integrates with the cloud provider (load balancers, nodes, routes). Only on cloud clusters. |
| `coredns` | `kube-system` | Deployment | Cluster DNS / service discovery. |
| `kube-proxy` | `kube-system` | DaemonSet | Implements Service networking on each node. |
| CNI plugin | `kube-system` | DaemonSet | Pod networking — allocates Pod IPs and wires up inter-Pod connectivity. |
| `metrics-server` | `kube-system` | Deployment | (Add-on) Powers `kubectl top` and the Horizontal Pod Autoscaler. |

> On managed clusters (EKS / AKS / GKE) the control-plane components
> (`kube-apiserver`, `etcd`, schedulers, controllers) are hidden and managed by
> the provider — you typically only see `coredns`, `kube-proxy`, and the CNI in
> `kube-system`.

---

## Node-Level Components

Not everything that makes a cluster work runs as a Pod. Each node also runs two
components as **host processes** (typically managed by `systemd`), which is why
they don't show up in `kubectl get pods`:

| Component | Runs As | Function |
|-----------|---------|----------|
| `kubelet` | systemd service | The node agent. Registers the node, watches the API server for Pods assigned to it, and tells the container runtime to start/stop containers. It also renews the node's `Lease` heartbeat (see `kube-node-lease`). |
| Container runtime | systemd service | Actually runs containers — `containerd` or `CRI-O`. The kubelet talks to it through the **CRI** (Container Runtime Interface). Docker support (`dockershim`) was removed in Kubernetes 1.24. |

```bash
# On a node (SSH in), check the host components
systemctl status kubelet
systemctl status containerd        # or crio

# Which runtime is each node using?
kubectl get nodes -o wide          # see CONTAINER-RUNTIME column
```

> **Mental model:** the kubelet is the node's "hands" — the scheduler decides
> *where* a Pod goes, the kubelet makes it actually happen via the runtime.

---

## Static Pods

A **static Pod** is a Pod managed directly by the **kubelet** on a node, not by
the API server. The kubelet watches a manifest directory (default
`/etc/kubernetes/manifests`) and runs whatever Pod specs it finds there.

- This is how self-managed control planes run `kube-apiserver`, `etcd`,
  `kube-scheduler`, and `kube-controller-manager` — there's no API server yet to
  schedule them, so the kubelet bootstraps them from files.
- The API server shows a read-only **mirror Pod** for each static Pod so you can
  see it with `kubectl`, but you can't edit or delete it via the API — you edit
  the manifest file instead.

```bash
# On a control-plane node
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

---

## CoreDNS

**Namespace:** `kube-system` · **Runs as:** Deployment

A DNS server that provides service discovery within the cluster.

- Every Service gets a DNS record, e.g.
  `my-service.my-namespace.svc.cluster.local`
- Pods resolve other Services **by name** instead of hardcoding IPs
- Forwards queries it can't answer (external names) to upstream resolvers
- Default cluster DNS since Kubernetes 1.11 (replaced the older `kube-dns`)

### DNS Naming Pattern

```
<service>.<namespace>.svc.cluster.local      # Service
<pod-ip-dashes>.<namespace>.pod.cluster.local # Pod (e.g. 10-1-2-3)
```

### Quick Checks

```bash
# Is CoreDNS running?  (note: the label is the legacy name kube-dns)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View its config
kubectl get configmap coredns -n kube-system -o yaml

# Test resolution from a temporary Pod
kubectl run dnsutils --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup kubernetes.default
```

---

## kube-proxy

**Namespace:** `kube-system` · **Runs as:** DaemonSet (one Pod per node)

Implements the Service networking abstraction.

- Watches the API server for Service and Endpoint changes
- Programs node network rules (`iptables` or `ipvs`) so traffic to a Service's
  ClusterIP is load-balanced to the correct backend Pod IPs
- Without it, Service IPs (virtual IPs with no real network interface) would be
  unreachable

### Modes

| Mode | Notes |
|------|-------|
| `iptables` | Default on most clusters. Simple, reliable, scales to thousands of Services. |
| `ipvs` | Better performance at very large scale; supports more load-balancing algorithms. |
| `nftables` | Newer backend, replacing `iptables` in recent Kubernetes versions. |

### Quick Checks

```bash
# Is kube-proxy running on every node?
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# View its config (mode, cluster CIDR, etc.)
kubectl get configmap kube-proxy -n kube-system -o yaml
```

> **In short:** CoreDNS = name → IP (DNS resolution); kube-proxy = IP → Pod
> (traffic routing / load balancing).

---

## CNI Plugin

**Namespace:** `kube-system` · **Runs as:** DaemonSet (one Pod per node)

The **CNI** (Container Network Interface) plugin provides **Pod networking** —
this is a distinct concern from kube-proxy's Service networking:

- Allocates an IP address to every Pod
- Wires up the network so any Pod can reach any other Pod across nodes (the flat
  "Pod network")
- May also enforce **NetworkPolicies** (firewall rules between Pods)

Kubernetes ships **no** built-in CNI — you must install one. Common choices:

| Plugin | Notes |
|--------|-------|
| Calico | Popular; supports NetworkPolicy, BGP routing. |
| Cilium | eBPF-based; high performance, rich observability and policy. |
| Flannel | Simple overlay network; easy to start with. |
| Cloud CNI | AWS VPC CNI, Azure CNI — assign real VPC/VNet IPs to Pods. |

> **CNI vs. kube-proxy:** CNI gets a packet from Pod A to Pod B (Pod-to-Pod).
> kube-proxy translates a Service's virtual IP into a real Pod IP
> (Service-to-Pod). Both are needed for normal cluster networking.

```bash
# Which CNI Pods are running?
kubectl get pods -n kube-system -o wide | grep -Ei 'calico|cilium|flannel|cni'

# A node stuck NotReady with "network plugin not ready" usually means no CNI
kubectl get nodes
```

---

## metrics-server

**Namespace:** `kube-system` · **Runs as:** Deployment · **Optional add-on**

Aggregates CPU/memory usage from each node's kubelet and exposes it through the
Metrics API. It's not installed by default but is near-ubiquitous because it
powers:

- `kubectl top nodes` / `kubectl top pods`
- The **Horizontal Pod Autoscaler** (HPA) and Vertical Pod Autoscaler (VPA)

```bash
# Is it installed?
kubectl get deployment metrics-server -n kube-system

# Use it
kubectl top nodes
kubectl top pods -A
```

> It provides **live** metrics for autoscaling only — it is *not* a monitoring
> or historical-metrics store (use Prometheus for that).

---

## kubeadm Bootstrap

`kubeadm` is the official tool for initializing and managing self-managed
clusters. "Bootstrap" is the process of bringing a cluster up from scratch.

### `kubeadm init` (first control-plane node)

- Generates TLS certificates and keys for all control plane components
- Writes kubeconfig files (`admin.conf`, `kubelet.conf`, etc.)
- Starts `kube-apiserver`, `etcd`, `kube-scheduler`,
  `kube-controller-manager` as **static Pods**
- Installs `coredns` and `kube-proxy`
- Creates a bootstrap token and writes the cluster CA + API server address into
  the `cluster-info` ConfigMap in `kube-public`

### `kubeadm join` (worker / additional nodes)

- Fetches the `cluster-info` ConfigMap from `kube-public` (publicly readable, no
  auth needed) to get the API server address and CA cert
- Uses the bootstrap token to authenticate and obtain a signed kubelet client
  certificate (TLS Bootstrap flow)
- Once the kubelet has its cert, it registers the node with the API server

### The chicken-and-egg problem it solves

A new node needs to trust the API server to join, but it has no credentials yet.
The `kube-public/cluster-info` ConfigMap + bootstrap token provide a minimal,
unauthenticated starting point: the node can verify the CA and use the
short-lived token to get a real certificate.

```bash
# List active bootstrap tokens
kubeadm token list

# Create a new join command (token + CA hash)
kubeadm token create --print-join-command
```

---

## Namespaced vs. Cluster-Scoped Resources

Namespaces only partition **some** resources. Many objects are **cluster-scoped**
(global) and don't belong to any namespace — a frequent source of confusion.

| Scope | Examples |
|-------|----------|
| **Namespaced** | Pods, Deployments, Services, ConfigMaps, Secrets, PersistentVolumeClaims, ServiceAccounts, Roles, RoleBindings, Ingress |
| **Cluster-scoped** | Nodes, Namespaces, PersistentVolumes, StorageClasses, ClusterRoles, ClusterRoleBindings, CustomResourceDefinitions, IngressClass |

```bash
# See which resources are namespaced
kubectl api-resources --namespaced=true

# See which are cluster-scoped
kubectl api-resources --namespaced=false
```

> Rule of thumb: if it represents physical/cluster-wide infrastructure (nodes,
> volumes, RBAC roles that span the cluster), it's cluster-scoped. If it's an
> app workload or its config, it's namespaced.

---

## Per-Namespace Governance

Namespaces are also the unit for several governance features:

| Object | Purpose |
|--------|---------|
| **Default ServiceAccount** | Every namespace automatically gets a `default` ServiceAccount; Pods that don't specify one use it. |
| **ResourceQuota** | Caps total resource usage (CPU, memory, object counts) within a namespace. |
| **LimitRange** | Sets default / min / max CPU & memory for individual Pods and containers in a namespace. |
| **NetworkPolicy** | Namespaced firewall rules controlling Pod-to-Pod traffic (requires a CNI that supports it). |
| **RBAC (Role / RoleBinding)** | Grant permissions scoped to a single namespace. |

```bash
kubectl get serviceaccount default -n <namespace>
kubectl get resourcequota,limitrange -n <namespace>
```

---

## Deleting Namespaces (and the `Terminating` Gotcha)

Deleting a namespace deletes **everything inside it** (cascading delete):

```bash
kubectl delete namespace <namespace>
```

Sometimes a namespace gets stuck in `Terminating` forever. This is almost always
caused by a **finalizer** on a resource whose controller is gone (commonly a
Custom Resource whose operator/CRD was removed).

```bash
# Find what's blocking deletion
kubectl get namespace <ns> -o yaml          # look at spec.finalizers / status
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n1 kubectl get --show-kind --ignore-not-found -n <ns>
```

> ⚠️ Force-removing the namespace finalizer (via the `finalize` API) will delete
> the namespace record even if child resources still exist, potentially orphaning
> them. Fix the underlying stuck resource first; only force as a last resort.

---

## Quick Reference

| Thing | Namespace | One-liner |
|-------|-----------|-----------|
| `default` | — | Where your resources go if unspecified. |
| `kube-system` | — | Control plane + infrastructure components. |
| `kube-public` | — | Public, unauthenticated-readable; holds `cluster-info`. |
| `kube-node-lease` | — | Per-node `Lease` heartbeats for node liveness. |
| CoreDNS | `kube-system` | Service discovery (name → IP). |
| kube-proxy | `kube-system` | Service networking (IP → Pod). |
| CNI plugin | `kube-system` | Pod networking (Pod → Pod) + IP allocation. |
| metrics-server | `kube-system` | Live CPU/mem metrics for `kubectl top` + HPA. |
| kubelet | node (systemd) | Node agent; runs Pods via the container runtime. |
| Container runtime | node (systemd) | `containerd` / `CRI-O`; runs the containers. |
| Static Pod | node | Pod run by the kubelet from `/etc/kubernetes/manifests`. |
| kubeadm bootstrap | — | Initializes the cluster and lets nodes join securely. |

---

## Useful Commands

```bash
# List all namespaces
kubectl get namespaces

# What's running in kube-system
kubectl get pods -n kube-system

# Inspect the public cluster-info ConfigMap
kubectl get configmap cluster-info -n kube-public -o yaml

# Node heartbeat Leases
kubectl get leases -n kube-node-lease

# Namespaced vs cluster-scoped resource types
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false

# Live resource usage (needs metrics-server)
kubectl top nodes
kubectl top pods -A

# Node host components (run on the node)
systemctl status kubelet

# Set a default namespace for the current context
kubectl config set-context --current --namespace=<namespace>
```
