# 🚀 Step-by-Step k3d Cluster Setup & Deep-Dive Explanation Guide

This guide walks you through setting up a multi-node Kubernetes cluster using **k3d**, along with a detailed explanation of **why** each tool is needed, **why** each command flag is used, and **how** traffic flows from your browser into the containers.

---

## 🧭 Table of Contents
1. [Why Are We Doing It This Way? (The Big Picture)](#1-why-are-we-doing-it-this-way-the-big-picture)
2. [Step 1: Install k3d CLI](#step-1-install-k3d-cli)
3. [Step 2: Verify Prerequisites](#step-2-verify-prerequisites)
4. [Step 3: Create Your Multi-Node Cluster](#step-3-create-your-multi-node-cluster)
5. [Step 4: Verify the Cluster and Inspect Containers](#step-4-verify-the-cluster-and-inspect-containers)
6. [Step 5: Deploy a Test Application (Full Flow)](#step-5-deploy-a-test-application-full-flow)
7. [Step 6: Cluster Lifecycle & Resource Management](#step-6-cluster-lifecycle--resource-management)
8. [Summary: Troubleshooting & Key Takeaways](#summary-troubleshooting--key-takeaways)

---

## 1. Why Are We Doing It This Way? (The Big Picture)

### The Problem With Traditional Kubernetes Learning
* Standard Kubernetes (`kubeadm` or bare-metal VMs via VirtualBox/Vagrant) requires **3 separate virtual machines**, consuming 6–8 GB of RAM and 15–30 minutes of complex networking configuration just to test a single pod.
* Minikube creates a single VM, which does not simulate real multi-node worker scheduling.

### Why k3d is the Superior Local Solution
1. **Container-in-Container Efficiency**: Instead of heavy VMs, each Kubernetes node is simply a **Docker container** running `k3s`.
2. **Speed**: Spawns a fully functioning 3-node Kubernetes cluster in **under 15 seconds**.
3. **Low Resource Footprint**: Consumes only ~1.5 GB RAM for 3 nodes combined.
4. **Instant Ingress**: Includes a built-in proxy (`k3d-proxy`) that automatically handles port routing from Windows `localhost` directly into cluster pods.

---

## Step 1: Install k3d CLI

### Option A: Via Official Bash Script (Linux, WSL, or Git Bash) ⭐
If you are using **WSL2 (Ubuntu/Debian)** or **Git Bash**, run the official install script:
```bash
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
*(Or using `curl`)*:
```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### Option B: Via Native Windows PowerShell
If you are running directly in native **Windows PowerShell**:
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/k3d-io/k3d/main/install.ps1 -UseBasicParsing | Invoke-Expression
```
*(Or via Windows Package Manager)*:
```powershell
winget install k3d.k3d
```

### 💡 Why do we need this and what does `wget ... | bash` do?
* `wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash`:
  * `wget -q`: Runs silently without filling the terminal with download clutter.
  * `-O -`: Outputs the downloaded shell script directly to standard output (STDOUT) rather than saving a temporary file to disk.
  * `| bash`: Pipes the script directly into the `bash` interpreter, which detects your OS (Linux/macOS/Windows) and CPU architecture (x86_64, arm64), fetches the latest release binary from GitHub, and installs it into `/usr/local/bin/k3d`.
* `k3d` is not Kubernetes itself; it is a **CLI management tool** (written in Go) that speaks to your Docker daemon.
* When you tell `k3d` to "create a cluster", it pulls the official Rancher `k3s` Docker image and orchestrates container creation, shared Docker networks, volume mounts, and kubeconfig generation.

> [!NOTE]
> If you run `wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash` inside **WSL (Ubuntu)**, make sure you run your `k3d` and `kubectl` commands from inside that WSL terminal so they have access to the binary. If you want to run `k3d` from native **Windows PowerShell**, use Option B.

---

## Step 2: Verify Prerequisites

### The Commands:
```powershell
# 1. Verify Docker Engine is active
docker ps

# 2. Verify k3d CLI is installed
k3d version

# 3. Verify kubectl CLI is installed
kubectl version --client
```

### 💡 Why do we need this?
* **Why Docker?** k3d cannot run without Docker. Docker is the runtime engine that hosts the node containers. If Docker Desktop is stopped, `k3d` commands will fail.
* **Why `kubectl`?** While `k3d` manages the *cluster infrastructure* (creating, starting, deleting nodes), `kubectl` is the tool used to manage *Kubernetes workloads* (pods, deployments, services, ingress) inside the cluster.

---

## Step 3: Create Your Multi-Node Cluster

### The Command:
```powershell
k3d cluster create mycluster `
  --image rancher/k3s:v1.31.4-k3s1 `
  --servers 1 `
  --agents 2 `
  -p "8080:80@loadbalancer"
```

> [!TIP]
> **Why `--image rancher/k3s:v1.31.4-k3s1`?**
> The unpinned `latest` image (`k3s v1.35+`) strictly requires Linux `cgroups v2`. Most Windows Docker Desktop engines run on `cgroups v1`, which causes `kubelet` to crash on start. Pinning to stable `v1.31.4-k3s1` ensures rock-solid compatibility with Windows Docker Desktop and your installed `kubectl v1.31`.

### 💡 Deep-Dive: Why These Specific Flags?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       k3d cluster create mycluster                          │
└───────┬───────────────────────────────┬─────────────────────────────┬───────┘
        │                               │                             │
        ▼                               ▼                             ▼
  --servers 1                      --agents 2              -p "8080:80@loadbalancer"
  (1 Master Node)                  (2 Worker Nodes)        (Host Port Forwarding)
```

1. **`mycluster`**:
   * Gives your cluster a distinct name. Prefixes all Docker containers and network bridges with `k3d-mycluster-*`.
2. **`--servers 1` (Master / Control Plane)**:
   * Spins up a container named `k3d-mycluster-server-0`.
   * Runs the `k3s server` process (API Server, Controller Manager, Scheduler, SQLite datastore).
3. **`--agents 2` (Worker Nodes)**:
   * Spins up two containers: `k3d-mycluster-agent-0` and `k3d-mycluster-agent-1`.
   * Runs `k3s agent` (`kubelet`, `containerd`, `kube-proxy`, Flannel CNI).
   * **Why 2 agents?** Having 2 workers allows you to observe Kubernetes load balancing, pod distribution across nodes, node cordoning, and failure rescheduling.
4. **`-p "8080:80@loadbalancer"` (Host Port Forwarding)**:
   * **Why `@loadbalancer`?** k3d spins up a lightweight Nginx-based reverse proxy container (`k3d-mycluster-serverlb`).
   * This flag maps port `8080` on your Windows host (`localhost:8080`) to port `80` of this load balancer container.
   * Traffic sent to `localhost:8080` is routed into the cluster's Ingress controller (Traefik) without needing to configure complex host routes.

---

## Step 4: Verify the Cluster and Inspect Containers

### The Commands:
```powershell
# 1. Inspect Kubernetes nodes
kubectl get nodes -o wide

# 2. Inspect the Docker containers powering the nodes
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Expected Docker Output:
```text
NAMES                     STATUS         PORTS
k3d-mycluster-serverlb    Up 1 minute    0.0.0.0:8080->80/tcp, 0.0.0.0:6443->6443/tcp
k3d-mycluster-server-0    Up 1 minute    
k3d-mycluster-agent-0     Up 1 minute    
k3d-mycluster-agent-1     Up 1 minute    
```

### 💡 Why does `kubectl` work immediately?
* When `k3d cluster create` finishes, it automatically pulls the cluster credentials and merges them into your `~/.kube/config` file.
* It sets your current `kubectl` context to `k3d-mycluster`. You don't need any manual IP copying or token exports.

---

## Step 5: Deploy a Test Application (Full Flow)

Now we prove the cluster works by deploying a web server and accessing it from Windows.

### 1. Create a Deployment (3 Pod Replicas)
```powershell
kubectl create deployment nginx-demo --image=nginx:alpine --replicas=3
```
* **Why 3 replicas?** Because we have 2 worker nodes. Kubernetes will distribute the 3 pods across `agent-0` and `agent-1`.

### 2. Verify Pod Placement
```powershell
kubectl get pods -o wide
```
* Observe the `NODE` column: you will see pods scheduled on both `k3d-mycluster-agent-0` and `k3d-mycluster-agent-1`.

### 3. Expose Pods with a Service
```powershell
kubectl expose deployment nginx-demo --port=80 --type=ClusterIP
```
* **Why a Service?** Pods are ephemeral; their internal IPs change if they restart. A `ClusterIP` Service provides a stable internal virtual IP and DNS name (`nginx-demo`) to load-balance traffic across all 3 pods.

### 4. Route Traffic via Ingress (Traefik)
```powershell
kubectl create ingress nginx-ingress --rule="/=nginx-demo:80"
```
* **Why Ingress?** Ingress is a set of routing rules that tells k3s's built-in **Traefik Ingress Controller**: *"Whenever an HTTP request hits `/`, forward it to the `nginx-demo` Service on port 80"*.

### 5. Access the Web Server
Open your browser or run in PowerShell:
```powershell
curl.exe http://localhost:8080
```
You will receive the HTML response: **"Welcome to nginx!"**.

---

### 🌐 Complete Network Traffic Flow Diagram

Here is what happens under the hood when you open `http://localhost:8080`:

```
1. Browser requests: http://localhost:8080
         │
         ▼
2. Windows Host forwards port 8080 to Docker container
         │
         ▼
3. [Container: k3d-mycluster-serverlb] (Port 80)
   (Reverse proxy receives request and forwards into cluster network)
         │
         ▼
4. [Traefik Ingress Controller] (inside k3s)
   (Reads the ingress rule: path "/" -> forward to Service "nginx-demo")
         │
         ▼
5. [Kubernetes Service: nginx-demo]
   (Round-robin load balances between healthy pod endpoints)
         │
         ├───────────────────────────────┐
         ▼                               ▼
6. [Pod: nginx-demo-xxx]        [Pod: nginx-demo-yyy]
   (Running on agent-0)           (Running on agent-1)
```

---

## Step 6: Cluster Lifecycle & Resource Management

Since you are running this on a laptop, you don't want Kubernetes consuming battery and CPU when you are not actively using it.

| Task | Command | Why / What it Does |
| :--- | :--- | :--- |
| **List clusters** | `k3d cluster list` | Shows active clusters and node counts. |
| **Pause / Stop cluster** | `k3d cluster stop mycluster` | Pauses all 4 Docker containers. **Frees 100% of RAM and CPU**, while preserving all your pods, state, and files! |
| **Resume / Start cluster** | `k3d cluster start mycluster` | Restarts the containers. Your cluster is back in ~5 seconds with everything intact. |
| **Delete cluster** | `k3d cluster delete mycluster` | Destroys all containers, Docker networks, and deletes the kubeconfig context cleanly. |

---

## Step 7: 🚨 Real-World Troubleshooting & Gotchas (Windows Specific)

During setup on Windows with Docker Desktop, you may hit two common pitfalls:

### 1. Kubelet Crash / Cluster Creation Freezes on Node Registration
* **Error**: `Node k3d-mycluster-agent-1 failed to get ready: stopped returning log lines` or `failed to validate kubelet configuration, error: kubelet is configured to not run on a host using cgroup v1`.
* **Root Cause**: Running `k3d cluster create` without an `--image` flag pulls the latest bleeding-edge image (e.g. `k3s v1.35+`), which strictly enforces Linux **cgroups v2**. If Docker Desktop WSL2 runs on **cgroups v1**, `kubelet` crashes repeatedly.
* **Solution**: Always pin the image to a proven stable version that matches your installed `kubectl`:
  ```powershell
  k3d cluster create mycluster --image rancher/k3s:v1.31.4-k3s1 --servers 1 --agents 2 -p "8080:80@loadbalancer"
  ```

### 2. `kubectl` Dial Timeout / Connection Refused (`10.0.0.1:6443` or `dial tcp: connectex`)
* **Error**: `Unable to connect to the server: dial tcp 10.0.0.1:6443: connectex: A connection attempt failed because the connected party did not properly respond`.
* **Root Cause 1**: Wrong context selected. Your kubeconfig may still point to an old cluster (e.g. `k8scluster` on `10.0.0.1:6443`).
  * **Fix**: Switch context to k3d:
    ```powershell
    kubectl config use-context k3d-mycluster
    ```
* **Root Cause 2**: On Windows, `host.docker.internal` may resolve to your external Wi-Fi/LAN IP instead of localhost `127.0.0.1`, causing timeouts.
  * **Fix**: Point the cluster endpoint directly to `127.0.0.1`:
    ```powershell
    # Get the assigned random port:
    docker port k3d-mycluster-serverlb 6443
    # Update kubeconfig to point to 127.0.0.1:<port>:
    kubectl config set-cluster k3d-mycluster --server=https://127.0.0.1:65503 --insecure-skip-tls-verify=true
    ```

---

## Summary: Key Takeaways

1. **`k3d` = Dockerized `k3s`**: You get a real, multi-node Kubernetes control plane and worker nodes running entirely inside isolated Docker containers.
2. **Zero Overhead**: No VirtualBox or Hyper-V virtual disk allocations required.
3. **Pin Your k3s Image**: Always pass `--image rancher/k3s:v1.31.4-k3s1` to avoid cgroup compatibility crashes on Windows.
4. **Built-in Port Forwarding**: The `-p "8080:80@loadbalancer"` flag allows direct access to apps via `localhost:8080` through Traefik Ingress.
5. **Clean & Disposable**: You can spin up, stop, restart, or destroy the entire cluster with a single command without leaving leftover files on your operating system.

