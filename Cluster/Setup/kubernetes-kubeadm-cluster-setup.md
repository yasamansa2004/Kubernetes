# Kubernetes Cluster Setup: 1 Master + 3 Workers

A step-by-step guide to building a Kubernetes cluster with one control-plane node and three workers, using **Ubuntu Server + containerd + kubeadm + Calico**, followed by a learning roadmap that takes you from installation to production-oriented operations.

The guide gives you:

1. A practical installation guide for a cluster with one master (control plane) and three worker nodes.
2. A learning roadmap so you understand what you are doing instead of simply copying commands.

> For the installation, treat the official Kubernetes kubeadm documentation as the primary reference.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [Learning roadmap](#2-learning-roadmap)
3. [Prepare your four servers](#3-prepare-your-four-servers)
4. [Prepare Linux on all four nodes](#4-prepare-linux-on-all-four-nodes)
5. [Install containerd on every node](#5-install-containerd-on-every-node)
6. [Install Kubernetes components on all four nodes](#6-install-kubernetes-components-on-all-four-nodes)
7. [Initialize the master node](#7-initialize-the-master-node)
8. [Install Calico networking](#8-install-calico-networking)
9. [Join the three workers](#9-join-the-three-workers)
10. [Your first Kubernetes application](#10-your-first-kubernetes-application)
11. [Learn Kubernetes by experimenting](#11-learn-kubernetes-by-experimenting)
12. [Troubleshooting commands to memorize](#12-troubleshooting-commands-to-memorize)
13. [Seven-day learning plan](#13-seven-day-learning-plan)

---

## 1. Architecture

| Node | Role | Example IP | Main responsibility |
|------|------|------------|---------------------|
| `k8s-master` | Control plane | 192.168.1.10 | Manages the cluster |
| `k8s-worker1` | Worker | 192.168.1.11 | Runs application workloads |
| `k8s-worker2` | Worker | 192.168.1.12 | Runs application workloads |
| `k8s-worker3` | Worker | 192.168.1.13 | Runs application workloads |

> **Important:** These IP addresses are examples. Replace them with your actual server IPs.

### What does each component do?

- **Control plane:** Makes scheduling and cluster-management decisions.
- **kube-apiserver:** Provides the Kubernetes API.
- **etcd:** Stores Kubernetes cluster state.
- **kube-scheduler:** Selects which worker should run a Pod.
- **kube-controller-manager:** Maintains the desired state of cluster resources.
- **kubelet:** Runs on each node and manages its Pods.
- **containerd:** Runs containers.
- **CNI (Calico):** Provides Pod networking between nodes.
- **kube-proxy:** Implements Kubernetes Service networking in the default kube-proxy-based setup.

For example, when you deploy an Nginx application, you tell Kubernetes how many replicas you want. The control plane schedules the Pods, and the workers run them.

> **Limitation:** A cluster with only one control-plane node is not highly available. If the master goes down, existing workloads may continue running, but cluster management and normal scheduling can be disrupted.

---

## 2. Learning roadmap

Don't try to learn every Kubernetes feature during installation. Follow this order.

| Phase | Topic | Practical result |
|-------|-------|------------------|
| 1 | Linux and networking prerequisites | Prepare all four servers |
| 2 | containerd, kubelet, kubeadm, kubectl | Install Kubernetes components |
| 3 | Control-plane initialization | Initialize the master |
| 4 | CNI networking | Enable communication between Pods |
| 5 | Worker joining | Add all three workers |
| 6 | Pods and Deployments | Deploy an Nginx application |
| 7 | Services and DNS | Expose applications inside the cluster |
| 8 | Ingress and TLS | Access applications through a domain |
| 9 | ConfigMaps, Secrets and storage | Deploy a realistic application |
| 10 | Helm and GitOps | Manage deployments efficiently |
| 11 | Monitoring and logging | Monitor cluster health |
| 12 | Security, backup and troubleshooting | Prepare for production operations |

**Recommended approach:** Spend the first day building the cluster, the second day learning Pods, Deployments and Services, and the following week practicing real deployment scenarios.

---

## 3. Prepare your four servers

Before installing anything, confirm that you can access all four servers over SSH.

### Step 1: Check server specifications

Run on all four servers:

```bash
hostname
hostname -I
cat /etc/os-release
nproc
free -h
df -h /
```

Suggested resources for a learning cluster:

| Resource | Master | Each worker |
|----------|--------|-------------|
| CPU | 2–4 vCPU | 2–4 vCPU |
| RAM | 4 GB minimum for a small lab | 4 GB minimum for a small lab |
| Disk | 30 GB or more | 30 GB or more |
| Operating system | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |

These are suggested lab specifications, not universal Kubernetes requirements. Your application workloads may need considerably more RAM and storage.

### Step 2: Set the hostnames

On the master:

```bash
sudo hostnamectl set-hostname k8s-master
```

On worker 1:

```bash
sudo hostnamectl set-hostname k8s-worker1
```

On worker 2:

```bash
sudo hostnamectl set-hostname k8s-worker2
```

On worker 3:

```bash
sudo hostnamectl set-hostname k8s-worker3
```

Reconnect your SSH sessions if necessary, then verify:

```bash
hostname
```

### Step 3: Configure name resolution

On all four servers, edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add these example entries:

```text
192.168.1.10 k8s-master
192.168.1.11 k8s-worker1
192.168.1.12 k8s-worker2
192.168.1.13 k8s-worker3
```

Replace the example addresses with the actual, stable private IP addresses of your servers.

Test connectivity from the master:

```bash
ping -c 3 k8s-worker1
ping -c 3 k8s-worker2
ping -c 3 k8s-worker3
```

Repeat the connectivity tests from each worker if you have firewall rules that might block communication.

> **Why does this matter?** Kubernetes depends on reliable network connectivity between nodes. Hostnames are useful for administration, while Kubernetes components also need to communicate using the correct node addresses.

### Step 4: Configure the required network access

At minimum, plan for these ports:

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API server |
| 2379–2380 | TCP | etcd client and peer communication |
| 10250 | TCP | Kubelet API |
| 10257 | TCP | Controller manager secure endpoint |
| 10259 | TCP | Scheduler secure endpoint |
| 30000–32767 | TCP/UDP as needed | Default NodePort range |
| 4789 | UDP | Calico VXLAN, if using VXLAN encapsulation |

The control-plane component ports normally only need to be accessible where required, not exposed publicly. Port 4789 is only necessary if your selected Calico configuration uses VXLAN. Additional ports may be required by your CNI, DNS, application, or monitoring configuration.

> **Security rule:** Keep Kubernetes control-plane and etcd ports restricted to trusted networks. Do not expose etcd or the kubelet API directly to the public internet.

---

## 4. Prepare Linux on all four nodes

Run the following commands on the master and all three workers.

### Step 1: Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget gpg ca-certificates apt-transport-https
```

If these servers already run important workloads, avoid a blind full upgrade and schedule maintenance appropriately.

### Step 2: Disable swap

The kubelet normally expects swap to be disabled unless you deliberately configure supported swap behavior.

Temporarily disable swap:

```bash
sudo swapoff -a
```

Check whether swap is disabled:

```bash
free -h
swapon --show
```

For the conventional kubeadm setup in this guide, `swapon --show` should return no active swap devices.

To disable swap persistently, inspect `/etc/fstab`:

```bash
cat /etc/fstab
```

If there is a swap entry, edit the file:

```bash
sudo nano /etc/fstab
```

Comment out the applicable swap entry by adding `#` at the beginning. Do not comment out unrelated filesystem entries.

### Step 3: Enable IP forwarding

```bash
sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
```

Load the modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure networking:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the settings:

```bash
sudo sysctl --system
```

Verify:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
```

Expected IP forwarding result:

```text
net.ipv4.ip_forward = 1
```

> **What you learned:** Kubernetes networking requires Linux networking capabilities that allow traffic to be forwarded between interfaces and Pods.

---

## 5. Install containerd on every node

Kubernetes needs a container runtime. This guide uses containerd rather than Docker Engine directly.

Docker and containerd are related, but Kubernetes requires a runtime that implements the Container Runtime Interface (CRI). Docker Engine requires an additional adapter called `cri-dockerd` when used with Kubernetes.

### Step 1: Install containerd

Run on all four servers:

```bash
sudo apt update
sudo apt install -y containerd
```

If you already use Docker Engine, check which containerd package is installed before changing anything, so you don't disrupt existing Docker workloads.

### Step 2: Generate the containerd configuration

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Enable the systemd cgroup driver. First, inspect the configuration:

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

For containerd 1.x, the setting normally appears under the runc options. For containerd 2.x, the configuration structure differs, so inspect the applicable runc runtime section.

For the standard Ubuntu containerd 1.x configuration, use:

```bash
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

Verify:

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

Make sure the active runc runtime configuration has `SystemdCgroup = true`.

> **Important:** Don't blindly use the 1.x configuration instructions on containerd 2.x. Check your installed version and configuration structure first.

### Step 3: Start containerd

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify:

```bash
sudo systemctl status containerd --no-pager
```

You should see:

```text
Active: active (running)
```

Also check the version:

```bash
containerd --version
```

> **Learning checkpoint:** You should now understand the difference between the container runtime and the Kubernetes components that manage workloads.

---

## 6. Install Kubernetes components on all four nodes

Three tools are installed:

| Tool | Purpose |
|------|---------|
| `kubeadm` | Initializes the cluster and joins nodes |
| `kubelet` | Runs on each node and manages Pods |
| `kubectl` | Lets you manage the cluster through the Kubernetes API |

### Step 1: Add the Kubernetes repository

This example uses Kubernetes **1.36** as the installation target. This is an example version selection, not a recommendation to always use that minor release. Before installing, check the supported releases and select the version you intend to maintain.

Run on every node:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL \
  https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor --yes \
      -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Step 2: Install the tools

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
```

Prevent automatic package upgrades from unexpectedly changing your Kubernetes components:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet:

```bash
sudo systemctl enable --now kubelet
```

Check versions:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

The three components should use compatible versions. For this example, all nodes should use the same Kubernetes minor release.

> **Why hold the packages?** Kubernetes upgrades need planning. You don't want an unattended package upgrade to change your node components independently of the cluster upgrade process.

Official reference: [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

---

## 7. Initialize the master node

Run this section **only on `k8s-master`**.

### Step 1: Identify the master IP

```bash
ip -br addr
```

Find the private IP address that the workers can reach. This guide uses `192.168.1.10`.

### Step 2: Choose the Pod network CIDR

Kubernetes needs an IP address range for Pods. This guide uses:

```text
192.168.0.0/16
```

This is the example Pod network for the Calico configuration.

Before continuing, confirm that `192.168.0.0/16` does not overlap with your server network, VPNs, or any other networks that need to communicate with the cluster. If it overlaps, select another compatible CIDR and configure Calico to use the same range.

### Step 3: Initialize Kubernetes

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --pod-network-cidr=192.168.0.0/16
```

Replace `192.168.1.10` with your actual master IP.

Wait for initialization to finish. If successful, you'll see a message similar to:

```text
Your Kubernetes control-plane has initialized successfully!
```

The output also includes a `kubeadm join` command. Save it securely; you'll need it when adding the workers.

### Step 4: Configure kubectl

Run as your normal Linux user on the master:

```bash
mkdir -p "$HOME/.kube"

sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"

sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

Test:

```bash
kubectl cluster-info
kubectl get nodes
```

At this stage, the master may show `NotReady`. That is expected before the Pod networking plugin is installed.

> **Security warning:** `/etc/kubernetes/admin.conf` provides powerful cluster administrator access. Never share it publicly or commit it to Git.

---

## 8. Install Calico networking

The cluster now has its control plane, but Pods need a networking implementation. This guide uses Calico.

### Step 1: Install Calico

Run on the master:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml
```

This command uses a version-pinned manifest as an example. Before deployment, check the current Calico documentation and its compatibility with your Kubernetes version.

The Pod network range configured for Calico must match the range used during `kubeadm init`.

Official reference: [Calico documentation](https://docs.tigera.io/calico/latest/about/)

### Step 2: Check Calico Pods

```bash
kubectl get pods -n kube-system
```

You should eventually see the networking Pods running.

Check node status:

```bash
kubectl get nodes -o wide
```

You may see:

```text
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   5m    v1.36.x
```

Wait for the networking components to start.

> **Important:** Don't assume networking works just because the manifest was accepted. Confirm that Calico Pods are healthy and that the master becomes `Ready`.

---

## 9. Join the three workers

### Step 1: Generate a join command

On the master:

```bash
kubeadm token create --print-join-command
```

It returns a command similar to:

```bash
sudo kubeadm join 192.168.1.10:6443 \
  --token <generated-token> \
  --discovery-token-ca-cert-hash sha256:<generated-hash>
```

The token and hash above are placeholders. Use the actual values printed by your master.

### Step 2: Join each worker

SSH into `k8s-worker1`, `k8s-worker2`, and `k8s-worker3` in turn, and run the generated join command with `sudo` on each.

Make sure all three workers have completed the previous installation steps first.

### Step 3: Verify the cluster

Back on the master:

```bash
kubectl get nodes -o wide
```

Your goal is four nodes with `Ready` status:

```text
NAME          STATUS   ROLES           VERSION
k8s-master    Ready    control-plane   v1.36.x
k8s-worker1   Ready    <none>          v1.36.x
k8s-worker2   Ready    <none>          v1.36.x
k8s-worker3   Ready    <none>          v1.36.x
```

The actual version numbers and ages depend on your installation.

Check the system Pods:

```bash
kubectl get pods -A
```

CoreDNS should become healthy after networking is configured.

Once all four nodes are `Ready` and the system Pods are healthy, your Kubernetes cluster is operational.

---

## 10. Your first Kubernetes application

Deploy Nginx with three replicas.

### Step 1: Create a Deployment

```bash
kubectl create deployment nginx --image=nginx:stable --replicas=3
```

A Deployment tells Kubernetes that you want three replicas of the Nginx application.

### Step 2: Check the Deployment

```bash
kubectl get deployments
```

Expected result:

```text
NAME    READY   UP-TO-DATE   AVAILABLE
nginx   3/3     3            3
```

Check the Pods:

```bash
kubectl get pods -o wide
```

You should see three Pods. Kubernetes may distribute them across different workers, depending on scheduling constraints and available resources.

### Step 3: Expose Nginx inside the cluster

```bash
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Check the Service:

```bash
kubectl get services
```

### Step 4: Test the application

```bash
kubectl run nginx-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- wget -qO- http://nginx
```

If networking and DNS are working, you should receive the Nginx HTML response.

This test teaches three important concepts:

- **Deployment:** Manages application replicas.
- **Pod:** Runs the application container.
- **Service:** Provides a stable network endpoint for accessing the Pods.

---

## 11. Learn Kubernetes by experimenting

### Exercise 1: Scale your application

Increase the replicas to five:

```bash
kubectl scale deployment nginx --replicas=5
kubectl get pods -o wide
```

Reduce the replicas to two:

```bash
kubectl scale deployment nginx --replicas=2
```

**Question:** What happens to the extra Pods when you reduce the replica count?

### Exercise 2: Investigate a Pod

Choose a Pod name from `kubectl get pods` and run:

```bash
kubectl describe pod <pod-name>
```

View its logs:

```bash
kubectl logs <pod-name>
```

Learn how to identify events, image-pull errors, scheduling issues, and container restarts.

### Exercise 3: Delete a Pod

```bash
kubectl delete pod <pod-name>
kubectl get pods -o wide
```

Notice that the Deployment creates a replacement Pod to maintain the desired replica count. This is one of the most important Kubernetes concepts: Kubernetes continuously works to maintain the desired state.

### Exercise 4: Learn YAML

Create a file called `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-yaml
  template:
    metadata:
      labels:
        app: nginx-yaml
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Inspect it:

```bash
kubectl get deployment nginx-yaml
kubectl get pods -l app=nginx-yaml
```

This is how you'll manage real Kubernetes applications: define the desired state in YAML and apply it.

---

## 12. Troubleshooting commands to memorize

| Problem | Command |
|---------|---------|
| Node is not ready | `kubectl describe node <node-name>` |
| Pod is not starting | `kubectl describe pod <pod-name>` |
| Application logs | `kubectl logs <pod-name>` |
| System component status | `kubectl get pods -n kube-system` |
| Cluster events | `kubectl get events -A --sort-by=.metadata.creationTimestamp` |
| Node resource usage | `kubectl top nodes` |
| Pod resource usage | `kubectl top pods -A` |
| Runtime status | `sudo systemctl status containerd` |
| Kubelet status | `sudo systemctl status kubelet` |
| Kubelet logs | `sudo journalctl -u kubelet -n 100 --no-pager` |

> `kubectl top` requires a working Metrics Server installation.

For production readiness, you'll eventually need monitoring, persistent storage, backup and restore procedures, access control, network policies, and a plan for control-plane failure. A single-master cluster is a good learning environment, but it is not highly available.

---

## 13. Seven-day learning plan

| Day | Topics | Deliverable |
|-----|--------|-------------|
| 1 | Install Kubernetes, understand the control plane, join workers | Four healthy nodes |
| 2 | Pods, Deployments, ReplicaSets | Deploy and scale Nginx |
| 3 | Services, DNS, ClusterIP, NodePort | Access an application through a Service |
| 4 | ConfigMaps, Secrets, environment variables | Deploy an app with externalized configuration |
| 5 | PersistentVolumes, PersistentVolumeClaims, StorageClasses | Run an application that needs persistent data |
| 6 | Ingress, TLS, Helm | Publish an application through a domain |
| 7 | Metrics Server, Prometheus, Grafana, troubleshooting | Monitor cluster and application health |

After this, progress to:

- **Week 2:** RBAC, namespaces, resource requests and limits, probes, and NetworkPolicies.
- **Week 3:** Helm, GitLab CI/CD, container registry integration, and automated deployments.
- **Week 4:** Cluster upgrades, etcd backup and restore, disaster recovery, and high availability.

For your DevOps career, eventually deploy an application through a complete CI/CD pipeline: GitLab builds the image, pushes it to your registry, and updates the Kubernetes Deployment.
