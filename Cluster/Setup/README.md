# Kubernetes 3-Node Cluster Setup (kubeadm)

This guide explains how to create a **3-node Kubernetes cluster from scratch** using **kubeadm** on **Ubuntu 22.04** for learning and practice.

---

## 🧱 Cluster Architecture

| Node     | CPU | RAM  | Disk |
|----------|-----|------|------|
| master   | 2   | 4GB  | 20GB |
| worker1  | 2   | 4GB  | 20GB |
| worker2  | 2   | 4GB  | 20GB |

---

##  Prepare Virtual Machines

Create **3 Ubuntu 22.04 VMs** and configure the following:

### Hostnames

Static IP Addresses (example)

```
     192.168.56.10 master
     192.168.56.11 worker1
     192.168.56.12 worker2
```

Update /etc/hosts on all nodes:

    sudo nano /etc/hosts

Add:

    192.168.56.10 master
    192.168.56.11 worker1
    192.168.56.12 worker2

## Disable Swap (ALL NODES)

Kubernetes requires swap to be disabled.

    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

Verify:

    swapon --show

## Install Container Runtime (containerd)
Install containerd

    sudo apt update
    sudo apt install -y containerd

Configure containerd

    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

Restart and enable service:

    sudo systemctl restart containerd
    sudo systemctl enable containerd

## Install Kubernetes Components (ALL NODES)
Install dependencies

    sudo apt update
    sudo apt install -y apt-transport-https ca-certificates curl

Add Kubernetes repository

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
    https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list

Install Kubernetes tools
    
    sudo apt update
    sudo apt install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

## Initialize Cluster (MASTER ONLY)

Run on the master node:

    sudo kubeadm init --pod-network-cidr=192.168.0.0/16

Configure kubectl access

    mkdir -p $HOME/.kube
    sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Install CNI (Network Plugin)
Recommended: Calico

    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

Wait until all pods are running:

    kubectl get pods -A

## Join Worker Nodes

On the master node, generate join command:

    kubeadm token create --print-join-command

Run the generated command on worker1 and worker2:

    sudo kubeadm join ...

## Verify Cluster 🎉

On the master node:

    kubectl get nodes

Expected Output
    
    master     Ready
    worker1    Ready
    worker2    Ready



## if you have any problem to create master cluster try to fix this part first:

# Kubernetes Prerequisites Setup (containerd + br_netfilter)

This README explains how to configure **containerd** and required **kernel networking modules** so that Kubernetes (`kubeadm`) can initialize successfully.

---

## 1. Enable SystemdCgroup for containerd

Kubernetes requires containerd to use **systemd cgroups**.

### Edit containerd configuration

```bash
sudo nano /etc/containerd/config.toml
```

Find the following line:

```toml
SystemdCgroup = false
```

Change it to:

```toml
SystemdCgroup = true
```

---

## 2. Restart and Enable containerd

Apply the configuration changes:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify containerd is running:

```bash
sudo systemctl status containerd
```

Expected status:

```
active (running)
```

---

## 3. Load the br_netfilter Kernel Module

Kubernetes networking requires the `br_netfilter` kernel module.

### Load the module manually

```bash
sudo modprobe br_netfilter
```

* If there is **no output**, the command was successful ✅
* If an error appears, it must be fixed before continuing

---

## 4. Verify Kernel Bridge Parameters

Check that the required kernel parameters now exist:

```bash
ls /proc/sys/net/bridge/
```

You should see:

```
bridge-nf-call-iptables
bridge-nf-call-ip6tables
```

---

## 5. Make br_netfilter Persistent Across Reboots

Ensure the module loads automatically on every boot:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

---

## 6. Configure Required sysctl Parameters

Kubernetes requires specific network-related sysctl settings.

Create the sysctl configuration file:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the settings immediately:

```bash
sudo sysctl --system
```

---

## ✅ Result

After completing these steps:

* containerd uses systemd cgroups
* Kernel bridge networking is enabled
* Required sysctl parameters are active

Your system is now ready to run:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

---

📌 **Note:** These steps are required on all Kubernetes nodes (control plane and workers).
