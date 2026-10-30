# Kubernetes Cluster Setup Using kubeadm

<p align="center">
  <img src="https://img.shields.io/badge/Kubernetes-v1.36.6-326CE5?logo=kubernetes&logoColor=white" />
  <img src="https://img.shields.io/badge/Container_Runtime-containerd-575757?logo=containerd&logoColor=white" />
  <img src="https://img.shields.io/badge/CNI-Calico-0052CC" />
  <img src="https://img.shields.io/badge/OS-Ubuntu%2026.10-E95420?logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Architecture-ARM64-blue" />
</p>

## Overview

This repository documents the step-by-step installation of a Kubernetes cluster using **kubeadm**.

The objective is to understand how Kubernetes components work together, configure the container runtime, prepare Linux nodes, bootstrap the control plane, and establish cluster networking using Calico.

This is a hands-on implementation rather than a deployment using managed Kubernetes services such as Amazon EKS, Azure AKS, or Google GKE.

## Architecture

The cluster consists of one control-plane node and one worker node.

```text
                  Kubernetes Cluster
                 --------------------

                  Control Plane
                 192.168.64.4
                       |
              +------------------+
              |   kube-apiserver |
              |   etcd           |
              |   Scheduler      |
              |   Controller     |
              +------------------+
                       |
                 Kubernetes API
                    Port 6443
                       |
              +------------------+
              |    Worker Node   |
              |    kubelet       |
              |    containerd    |
              |    kube-proxy    |
              +------------------+
                       |
                  Calico CNI
                       |
                  Pod Network
                192.168.0.0/16
```

### Cluster Configuration

| Component | Configuration |
|---|---|
| Kubernetes | v1.36.6 |
| Installation Tool | kubeadm |
| Container Runtime | containerd |
| Container Runtime Interface | CRI |
| Network Plugin | Calico v3.33.0 |
| Operating System | Ubuntu 26.10 |
| Architecture | ARM64 |
| Pod CIDR | 192.168.0.0/16 |
| API Server Port | 6443 |
| Control Plane | 192.168.64.4 |

## 1. Prerequisites

Before installing Kubernetes, prepare the Linux machines.

Required components:

- Ubuntu Linux on both nodes.
- Root or sudo privileges.
- Network connectivity between nodes.
- Container runtime (containerd).
- kubeadm, kubelet and kubectl (kubectl is needed on the control plane).
- Swap disabled.
- Required Linux kernel modules and sysctl parameters configured.

## 2. Install Container Runtime

Perform these steps on **both control-plane and worker nodes**.

### Update packages

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gpg containerd
```

### Configure kernel modules

Kubernetes networking and container filesystems require specific kernel modules.

Load the modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

**overlay**

Provides OverlayFS, which allows container images to use layered filesystems. Read-only image layers are combined with a writable container layer.

**br_netfilter**

Allows bridged network traffic to be processed by Linux Netfilter rules. This is important for Kubernetes networking and network policy implementations.

Persist the modules across reboots:

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

### Configure network parameters

Create a sysctl configuration:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the configuration:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

Expected:

```text
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
```

**Why are these parameters required?**

- `net.bridge.bridge-nf-call-iptables`: Allows bridged IPv4 packets to pass through iptables-compatible Netfilter processing.
- `net.ipv4.ip_forward`: Enables IPv4 packet forwarding between network interfaces.

### Configure containerd

Generate the default containerd configuration:

```bash
sudo mkdir -p /etc/containerd

sudo containerd config default | \
sudo tee /etc/containerd/config.toml
```

Configure the systemd cgroup driver:

```bash
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```

**Why SystemdCgroup?**

Kubernetes kubelet and containerd should use compatible cgroup drivers.

When systemd manages the node's resources, using the systemd cgroup driver provides consistent resource management.

Enable and restart containerd:

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify:

```bash
sudo systemctl is-active containerd
```

Expected:

```text
active
```

## 3. Disable Swap

Kubernetes kubelet traditionally requires swap to be disabled unless explicitly configured to support swap.

Disable swap immediately:

```bash
sudo swapoff -a
```

Verify:

```bash
free -mh
```

Expected:

```text
Swap: 0B 0B 0B
```

Disable swap permanently.

Edit `/etc/fstab`:

```bash
sudo vi /etc/fstab
```

Comment out the swap entry:

```text
# /swap.img none swap sw 0 0
```

Verify that swap is disabled:

```bash
swapon --show
```

No output should be returned.

Repeat on both nodes.

## 4. Install Kubernetes Components

Perform these steps on both nodes.

Create the keyring directory:

```bash
sudo mkdir -p /etc/apt/keyrings
```

Download and install the Kubernetes repository signing key:

```bash
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
sudo gpg --dearmor --yes \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update package information:

```bash
sudo apt-get update
```

### Install control-plane and worker packages

On the control-plane node:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

On worker nodes:

```bash
sudo apt-get install -y kubelet kubeadm
```

Prevent automatic Kubernetes package upgrades:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

On worker nodes, hold the installed packages:

```bash
sudo apt-mark hold kubelet kubeadm
```

Enable kubelet:

```bash
sudo systemctl enable --now kubelet
```

**Note:** kubelet may repeatedly restart before kubeadm initializes or joins the node. This is expected during initial setup.

## 5. Initialize the Kubernetes Control Plane

Run the following command only on the control-plane node.

```bash
sudo kubeadm init \
--apiserver-advertise-address=192.168.64.4 \
--pod-network-cidr=192.168.0.0/16
```

### Understanding the command

| Parameter | Description |
|---|---|
| `kubeadm init` | Initializes the Kubernetes control plane |
| `--apiserver-advertise-address` | IP address exposed by the API server |
| `--pod-network-cidr` | IP address range allocated for Kubernetes Pods |

### What happens during kubeadm init?

kubeadm performs several operations:

1. Runs preflight checks.
2. Pulls the required Kubernetes images.
3. Generates Kubernetes certificates.
4. Generates kubeconfig files.
5. Creates static Pod manifests.
6. Starts the control-plane components.
7. Configures bootstrap tokens and RBAC.
8. Deploys CoreDNS and kube-proxy.

The main control-plane components are:

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd

### Configure kubectl

After successful initialization:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
$HOME/.kube/config

sudo chown $(id -u):$(id -g) \
$HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

At this stage, the node may show `NotReady` because the Pod networking plugin has not been installed yet.

### Verify API Server

```bash
nc -zv 192.168.64.4 6443
```

Port 6443 is the Kubernetes API server endpoint.

Before `kubeadm init`, a connection refusal is expected because the API server is not running yet.

## 6. Install Calico CNI

Kubernetes requires a Container Network Interface (CNI) plugin to provide Pod networking.

For this cluster, we use Calico.

Calico provides:

- Pod-to-Pod networking.
- Network routing.
- Network policies.
- Network isolation.
- Optional BGP-based routing.

### Step 1: Install Calico CRDs

```bash
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml
```

Verify:

```bash
kubectl get crd | grep projectcalico
```

**Important:** Installing CRDs only registers Calico's custom resource types with the Kubernetes API. It does not install the networking components.

### Step 2: Install Tigera Operator

The Tigera Operator manages the Calico installation and lifecycle.

```bash
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

Verify:

```bash
kubectl get pods -n tigera-operator
```

Wait until the operator Pod is `Running` and ready.

### Step 3: Configure Calico networking

Apply the Calico custom resources:

```bash
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
```

The default Pod CIDR in this manifest is:

```text
192.168.0.0/16
```

This matches the CIDR supplied during `kubeadm init`.

**Important:** If you change the Pod CIDR during initialization, make sure the Calico IP pool is configured to match it.

### Verify Calico

```bash
kubectl get pods -n tigera-operator

kubectl get pods -n calico-system

kubectl get nodes -o wide
```

Expected node status:

```text
NAME   STATUS   ROLES           AGE   VERSION
k8s    Ready    control-plane   ...   v1.36.6
```

## 7. Join Worker Nodes

The worker node needs the container runtime, kubelet and kubeadm installed and configured before joining.

Repeat the following prerequisites on every worker:

- Install containerd.
- Configure systemd cgroups.
- Load overlay and br_netfilter.
- Enable IPv4 forwarding.
- Disable swap.
- Install kubelet and kubeadm.

### Generate the join command

On the control-plane node:

```bash
sudo kubeadm token create --print-join-command
```

This generates a command similar to:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Run the generated command on the worker node using sudo.

**Security:** Never commit bootstrap tokens or private keys to a public GitHub repository.

### Verify worker registration

On the control-plane node:

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME         STATUS   ROLES           VERSION
k8s          Ready    control-plane   v1.36.6
workernode2  Ready    <none>          v1.36.6
```

## 8. Cluster Verification

After joining the worker node, verify the cluster.

### Check nodes

```bash
kubectl get nodes -o wide
```

### Check system Pods

```bash
kubectl get pods -n kube-system
```

### Check Calico Pods

```bash
kubectl get pods -n calico-system
```

### Check all namespaces

```bash
kubectl get pods -A
```

### Check cluster information

```bash
kubectl cluster-info
```

### Check cluster events

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

## 9. Troubleshooting

### Issue 1: API Server connection refused

```bash
nc -zv 127.0.0.1 6443
```

**Reason:** The API server may not have been initialized or started.

Check:

```bash
sudo crictl ps -a
sudo journalctl -u kubelet -f
```

### Issue 2: Node remains NotReady

**Possible reason:** CNI plugin is missing or not functioning.

Check:

```bash
kubectl get pods -A
kubectl describe node <NODE_NAME>
```

### Issue 3: kubeadm command not found

**Reason:** kubeadm is not installed on the node.

Install:

```bash
sudo apt-get install -y kubeadm
```

### Issue 4: IPv4 forwarding preflight failure

Error:

```text
[ERROR FileContent--proc-sys-net-ipv4-ip_forward]:
/proc/sys/net/ipv4/ip_forward contents are not set to 1
```

Fix:

```bash
echo 'net.ipv4.ip_forward = 1' | \
sudo tee /etc/sysctl.d/99-kubernetes.conf

sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

### Issue 5: Swap preflight warning

Disable swap:

```bash
sudo swapoff -a
```

Check:

```bash
swapon --show
```

Also verify that the swap entry is commented out in `/etc/fstab`.

## 10. Useful Commands

| Command | Purpose |
|---|---|
| `kubectl get nodes -o wide` | List cluster nodes |
| `kubectl get pods -A` | List all Pods |
| `kubectl get pods -n kube-system` | Check Kubernetes system Pods |
| `kubectl describe node <name>` | Inspect node details |
| `kubectl get events -A` | View cluster events |
| `sudo crictl ps -a` | List runtime containers |
| `sudo systemctl status kubelet` | Check kubelet |
| `sudo systemctl status containerd` | Check containerd |
| `sudo journalctl -u kubelet -f` | Follow kubelet logs |
| `sudo kubeadm token create --print-join-command` | Generate worker join command |

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a Cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Containerd Documentation](https://containerd.io/docs/)
- [Calico Documentation](https://docs.tigera.io/calico/latest/about/)
- [Tigera Operator](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

---

<p align="center">
  <b>Built as a hands-on Kubernetes learning project.</b>
  <br/>
  Understanding Kubernetes from the Linux layer to the control plane and networking layer.
</p>