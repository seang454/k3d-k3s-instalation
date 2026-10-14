# ⚖️ Deep-Dive: How Load Balancing Works in k3d & k3s

When you run a multi-node Kubernetes cluster locally using **k3d**, incoming network traffic from your laptop into your pods is handled by a cooperative **Two-Layer Load Balancing Architecture**:

1. **Outer Layer (Docker Host Level)**: The `k3d-proxy` container (`k3d-<cluster>-serverlb`).
2. **Inner Layer (Kubernetes Cluster Level)**: The `svclb-traefik` DaemonSet pods (Klipper ServiceLB).

---

## 🧭 Table of Contents
1. [The Big Picture: Two Cooperative Layers](#1-the-big-picture-two-cooperative-layers)
2. [Layer 1: The Outer Load Balancer (`k3d-proxy` Container)](#2-layer-1-the-outer-load-balancer-k3d-proxy-container)
3. [Layer 2: The Inner Load Balancer (`svclb-traefik` / Klipper-LB)](#3-layer-2-the-inner-load-balancer-svclb-traefik--klipper-lb)
4. [Why Are There Exactly 3 `svclb-traefik` Pods?](#4-why-are-there-exactly-3-svclb-traefik-pods)
5. [Complete End-to-End Packet Journey (Diagram)](#5-complete-end-to-end-packet-journey-diagram)
6. [Side-by-Side: Local k3d Load Balancing vs. Cloud Providers](#6-side-by-side-local-k3d-load-balancing-vs-cloud-providers)
7. [Summary & Key Takeaways](#7-summary--key-takeaways)

---

## 1. The Big Picture: Two Cooperative Layers

```
[ Your Windows Browser ]
        │
        │ Requests: http://localhost:8080
        ▼
================================================================================
 LAYER 1: The Docker Outer Load Balancer (k3d Proxy)
 Container: k3d-mycluster-serverlb (nginx reverse proxy)
 Role: Bridges Windows host traffic into the private Docker network
================================================================================
        │
        ▼
================================================================================
 LAYER 2: The Kubernetes Inner Load Balancer (Klipper ServiceLB)
 Pods: svclb-traefik-xxxxx (Runs as a DaemonSet across all nodes)
 Role: Catches packets arriving at any node and forwards to Traefik Ingress
================================================================================
        │
        ▼
 [ Traefik Ingress Pod ] ──► [ ClusterIP Service: my-nginx ] ──► [ NGINX Pods ]
```

---

## 2. Layer 1: The Outer Load Balancer (`k3d-proxy` Container)

### What is it?
When you inspect your Docker containers with `docker ps`, you see:
```text
CONTAINER ID   IMAGE                            PORTS                                           NAMES
1a47c24694d2   ghcr.io/k3d-io/k3d-proxy:5.9.0   0.0.0.0:8080->80/tcp, 0.0.0.0:6443->6443/tcp   k3d-mycluster-serverlb
```
This is a dedicated, ultra-lightweight **NGINX reverse proxy container** created by `k3d`.

### Why is it needed?
* In a multi-node cluster, your worker nodes (`agent-0`, `agent-1`) and master node (`server-0`) live inside an isolated Docker bridge network (e.g. `172.20.0.0/16`).
* Windows cannot route packets directly to those private container IPs.
* The `k3d-proxy` container sits at the boundary between your Windows laptop (`localhost`) and the Docker network:
  1. It binds to Windows port **`8080`**.
  2. It balances traffic across the real IP addresses of `server-0`, `agent-0`, and `agent-1`.
  3. If you add or remove worker nodes (`k3d node create/delete`), `k3d-proxy` updates its internal upstream routing table automatically!

---

## 3. Layer 2: The Inner Load Balancer (`svclb-traefik` / Klipper-LB)

### What is it?
When you inspect pods inside the `kube-system` namespace, you see:
```text
NAMESPACE     NAME                           READY   STATUS    RESTARTS   NODE
kube-system   svclb-traefik-30976a88-5kr87   2/2     Running   0          k3d-mycluster-server-0
kube-system   svclb-traefik-30976a88-jhn4r   2/2     Running   0          k3d-mycluster-agent-0
kube-system   svclb-traefik-30976a88-wmp8d   2/2     Running   0          k3d-mycluster-agent-1
```
The **`svclb`** prefix stands for **Service Load Balancer**. This is Rancher's built-in **Klipper ServiceLB**.

### How does Klipper ServiceLB work?
* In Kubernetes, when a Service is created with `type: LoadBalancer`, Kubernetes expects an external load balancer to assign an IP address.
* K3s does not need AWS or GCP; it implements Klipper:
  1. Klipper watches the API server for any Service with `type: LoadBalancer` (such as the default `traefik` service).
  2. For every such service, Klipper deploys a tiny container running on host networking on **every single node**.
  3. This container uses Linux network rules (`iptables` / IPVS) to direct any packet landing on port 80 of that node directly into the actual Traefik pod endpoints!

---

## 4. Why Are There Exactly 3 `svclb-traefik` Pods?

Look at the node column:
```powershell
kubectl get pods -n kube-system -l app=svclb-traefik -o wide
```

```text
NAME                           READY   STATUS    NODE
svclb-traefik-30976a88-5kr87   2/2     Running   k3d-mycluster-server-0  (Master)
svclb-traefik-30976a88-jhn4r   2/2     Running   k3d-mycluster-agent-0   (Worker 1)
svclb-traefik-30976a88-wmp8d   2/2     Running   k3d-mycluster-agent-1   (Worker 2)
```

### The Reason: DaemonSet Architecture
* Klipper deploys as a **DaemonSet** — Kubernetes terminology meaning *"Run exactly one replica of this pod on every node in the cluster"*.
* Because you created a **3-node cluster** (1 master + 2 workers), Kubernetes scheduled **3 pods**.
* **Why this is powerful**:
  * You do not have to care which node receives the request from Layer 1.
  * If traffic from `k3d-proxy` hits `agent-0`, `svclb-traefik` is there.
  * If traffic hits `agent-1`, `svclb-traefik` is there.
  * If a worker node crashes, the other nodes continue taking traffic with zero interruption.

---

## 5. Complete End-to-End Packet Journey (Diagram)

Here is the exact journey of a network packet from your browser to your application:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: CLIENT (WINDOWS LAPTOP)                                                        │
│ User opens browser: http://localhost:8080                                              │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: OUTER LOAD BALANCER (k3d-mycluster-serverlb)                                   │
│ Container: k3d-proxy                                                                   │
│ • Listens on 0.0.0.0:8080 on Windows host                                              │
│ • Forwards packet into Docker internal network (e.g. 172.20.0.4:80 on agent-0)        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: INNER LOAD BALANCER (svclb-traefik on agent-0)                                 │
│ Pod: svclb-traefik-xxxxx (DaemonSet)                                                   │
│ • Catches packet arriving on node port 80                                              │
│ • Uses iptables DNAT rules to route packet directly to the Traefik Ingress Pod         │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: INGRESS CONTROLLER (traefik pod)                                               │
│ Pod: traefik-xxxxx                                                                     │
│ • Inspects HTTP request path ("/") and hostname ("*")                                  │
│ • Looks up matching Ingress rule: path "/" ──► Service "my-nginx:80"                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: KUBERNETES SERVICE (my-nginx)                                                  │
│ Service: ClusterIP (Virtual IP: 10.43.21.219:80)                                       │
│ • Discovers healthy endpoint: 10.42.2.8:80                                             │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ STEP 6: APPLICATION WORKLOAD (my-nginx pod)                                            │
│ Pod: nginx-deployment-68fd69ffdc-h24ht                                                 │
│ • Web server processes request and returns HTTP 200 OK: "Welcome to nginx!"            │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Side-by-Side: Local k3d Load Balancing vs. Cloud Providers

| Feature | Local k3d Cluster | Cloud Kubernetes (AWS EKS / GCP GKE) |
| :--- | :--- | :--- |
| **Outer LB** | `k3d-proxy` Docker container (free, local) | Cloud Load Balancer (AWS ALB/NLB, Google Cloud LB) |
| **Inner LB** | Klipper ServiceLB (`svclb-*` DaemonSet) | Cloud Controller Manager (CCM) cloud API integration |
| **Cost** | **\$0.00** (runs entirely in local RAM) | \$15 – \$25+ per month per load balancer |
| **Provisioning Time**| **Instant** (< 2 seconds) | 2 to 5 minutes waiting for cloud DNS / LB provisioning |
| **Multi-Node Support**| DaemonSet on every local container node | Distributes to physical VM instances across availability zones |

---

## 7. Summary & Key Takeaways

1. **Two Layers, One Team**: 
   * `k3d-proxy` gets traffic **into Docker**.
   * `svclb-traefik` gets traffic **into the Kubernetes Ingress**.
2. **Why 3 Pods?**: 
   * Exactly 1 pod runs on each of your 3 nodes (`server-0`, `agent-0`, `agent-1`) as a **DaemonSet**.
3. **Zero Configuration**: 
   * You don't have to configure IP tables, bridge adapters, or cloud accounts. K3s and k3d wire the entire pipeline automatically when you pass `-p "8080:80@loadbalancer"`.
