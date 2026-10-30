## InstallingK8s
    Installing k8s using kubeadm
## Install container first 
    mhasifkamal@k8s:~$ date
    Thu Oct 29 11:11:31 PM IST 2026

## Check port is open in order for Kubernetes components to communicate with each other.
    mhasifkamal@k8s:~$ nc 127.0.0.1 6443 -zv -w 2
    nc: connect to 127.0.0.1 port 6443 (tcp) failed: Connection refused

## Disable swap
    mhasifkamal@k8s:~$ sudo swapoff -a
    mhasifkamal@k8s:~$ free -mh
                total        used        free      shared  buff/cache   available
    Mem:           1.6Gi       319Mi       1.1Gi       5.2Mi       274Mi       1.3Gi
    Swap:             0B          0B          0B
    mhasifkamal@k8s:~$ cat /etc/fstab
    # /etc/fstab: static file system information.
    #
    # Use 'blkid' to print the universally unique identifier for a
    # device; this may be used with UUID= as a more robust way to name devices
    # that works even if disks are added and removed. See fstab(5).
    #
    # <file system> <mount point>   <type>  <options>       <dump>  <pass>
    # / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
    /dev/disk/by-id/dm-uuid-LVM-9XVoZtgTFodSmlVurEsEafIXLDkHU7quIXzQ1VzuOKFbhZGmhYfUmgMlP4Kmhu3e / ext4 defaults 0 1
    # /boot was on /dev/vda2 during curtin installation
    /dev/disk/by-uuid/11e3d738-0663-42e8-85c3-e3552d7fdd20 /boot ext4 defaults 0 1
    # /boot/efi was on /dev/vda1 during curtin installation
    /dev/disk/by-uuid/9CD9-DEB8 /boot/efi vfat defaults 0 1
    /swap.img	none	swap	sw	0	0
    mhasifkamal@k8s:~$ sudo vi /etc/fstab


<img width="993" height="349" alt="Screenshot 2026-10-29 at 11 22 18 PM" src="https://github.com/user-attachments/assets/9ae1e203-b536-455c-aa8e-ade082bfe840" />

## Install the required packages
    sudo apt-get update
    sudo apt-get install -y curl ca-certificates gpg containerd
    """
    curl -> Makes HTTP/HTTPS requests(Download files and check endpoints)
    ca-certificates -> Provides trusted certificate authorities(Validate HTTPS certificates)
    gpg -> Verifies cryptographic signatures(Verify repository signing keys)
    containerd -> Container runtime(Downloads images and runs containers)
    """
    # Load the Linux kernel modules
    sudo modprobe overlay
    sudo modprobe br_netfilter
    """
    The overlay module provides OverlayFS, a union filesystem.
    Container images consist of layers. OverlayFS can combine read-only image layers with a writable container layer.

    br_netfilter -> This module allows bridged network traffic to be processed by Linux's Netfilter framework.
    """

    sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
    modprobe loads modules immediately, but by itself it doesn't guarantee they're loaded after reboot.
    /etc/modules-load.d/: load automatically at startup.

    sudo tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
    """
    net.bridge.bridge-nf-call-iptables = 1

    Allows bridged IPv4 packets to be processed by iptables-compatible Netfilter rules when bridge filtering is available.
    Kubernetes networking and some CNI plugins rely on packet filtering to implement network behavior.

    net.ipv4.ip_forward = 1

    Allows Linux to route IPv4 packets between network interfaces.

    This matters when traffic must travel through a node rather than terminate on that node.
    """

    #Generate the containerd configuration
    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

    """
    containerd config default -> (Prints containerd's default configuration.)

    cat /etc/containerd/config.toml | grep "SystemdCgroup"

    sudo sed -i \
    's/SystemdCgroup = false/SystemdCgroup = true/' \
    /etc/containerd/config.toml

    #Enable and Verify containerd
    mhasifkamal@k8s:~$ sudo systemctl enable --now containerd
    mhasifkamal@k8s:~$ sudo systemctl restart containerd
    mhasifkamal@k8s:~$ sudo systemctl is-active containerd
    active
    mhasifkamal@k8s:~$ 



## Installing a container runtime
    Enable IPv4 packet forwarding
    # sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.ipv4.ip_forward = 1
    EOF

    # Apply sysctl params without reboot
    sudo sysctl --system

    image.png


# Install kubeadm, kubelete, kubectl
Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    # Add the appropriate Kubernetes apt repository.
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    sudo systemctl enable --now kubelet



    sudo kubeadm init \
    --apiserver-advertise-address=192.168.64.4 \
    --pod-network-cidr=192.168.0.0/16


    mhasifkamal@k8s:~$ sudo kubeadm init \
  --apiserver-advertise-address=192.168.64.4 \
  --pod-network-cidr=192.168.0.0/16
I1030 00:25:34.800603   11753 version.go:260] remote version is much newer: v1.37.2; falling back to: stable-1.36
[init] Using Kubernetes version: v1.36.6
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.64.4]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s localhost] and IPs [192.168.64.4 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s localhost] and IPs [192.168.64.4 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 230.875µs
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.64.4:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 2.006704ms
[control-plane-check] kube-scheduler is healthy after 2.140829ms
[control-plane-check] kube-apiserver is healthy after 1.506555948s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: b4zx5n.tc94fncpopxbols8
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Configured RBAC rules to allow the API server kubelet client certificate to access the kubelet API
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.64.4:6443 --token b4zx5n.tc94fncpopxbols8 \
	--discovery-token-ca-cert-hash ************************ 


# calico installation

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml

mhasifkamal@k8s:~$ kubectl get crd | grep projectcalico
bgpconfigurations.projectcalico.org                 2026-10-29T18:58:16Z
bgpfilters.projectcalico.org                        2026-10-29T18:58:16Z
bgppeers.projectcalico.org                          2026-10-29T18:58:16Z
blockaffinities.projectcalico.org                   2026-10-29T18:58:16Z
caliconodestatuses.projectcalico.org                2026-10-29T18:58:16Z
clusterinformations.projectcalico.org               2026-10-29T18:58:16Z
felixconfigurations.projectcalico.org               2026-10-29T18:58:16Z
globalnetworkpolicies.projectcalico.org             2026-10-29T18:58:16Z
globalnetworksets.projectcalico.org                 2026-10-29T18:58:16Z
hostendpoints.projectcalico.org                     2026-10-29T18:58:16Z
ipamblocks.projectcalico.org                        2026-10-29T18:58:16Z
ipamconfigurations.projectcalico.org                2026-10-29T18:58:16Z
ipamhandles.projectcalico.org                       2026-10-29T18:58:16Z
ippools.projectcalico.org                           2026-10-29T18:58:16Z
ipreservations.projectcalico.org                    2026-10-29T18:58:16Z
kubecontrollersconfigurations.projectcalico.org     2026-10-29T18:58:16Z
networkpolicies.projectcalico.org                   2026-10-29T18:58:16Z
networksets.projectcalico.org                       2026-10-29T18:58:16Z
stagedglobalnetworkpolicies.projectcalico.org       2026-10-29T18:58:16Z
stagedkubernetesnetworkpolicies.projectcalico.org   2026-10-29T18:58:16Z
stagednetworkpolicies.projectcalico.org             2026-10-29T18:58:16Z
tiers.projectcalico.org                             2026-10-29T18:58:16Z
mhasifkamal@k8s:~$ 
Your node is still NotReady because creating the CRDs only defines Calico's resource types; it does not install the networking components yet. Let's install those next.

The operator is a Kubernetes controller that manages Calico's installation and lifecycle
mhasifkamal@k8s:~$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
namespace/tigera-operator created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator-secrets created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
rolebinding.rbac.authorization.k8s.io/tigera-operator-secrets created
deployment.apps/tigera-operator created
mhasifkamal@k8s:~$ 



Check that it starts:
kubectl get pods -n tigera-operator


Wait until the operator Pod shows Running and 1/1 ready. If it stays pending or shows an image-pull error, share the output rather than proceeding blindly.
Step 3 — Configure Calico's Pod network
Once the operator is running, create Calico's installation resources:
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml


The official manifest uses 192.168.0.0/16, matching the Pod CIDR supplied to kubeadm init.

mhasifkamal@k8s:~$ kubectl get pods -n tigera-operator
NAME                               READY   STATUS    RESTARTS   AGE
tigera-operator-686489b657-7bt5j   1/1     Running   0          113s
mhasifkamal@k8s:~$ kubectl get nodes -o wide
NAME   STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION             CONTAINER-RUNTIME
k8s    Ready    control-plane   5m46s   v1.36.6   192.168.64.4   <none>        Ubuntu 26.10   7.3.0-10-generic (arm64)   containerd://2.3.4+dfsg1
mhasifkamal@k8s:~$ 




worker node 

mhasifkamal@workernode2:~$ kubeadm join 192.168.64.4:6443 --token 0l9gw0.qccimiku4t73h97j --discovery-token-ca-cert-hash *************************
kubeadm: command not found
mhasifkamal@workernode2:~$ sudo apt-get update
sudo apt-get install -y curl ca-certificates gpg containerd
[sudo: authenticate] Password: 
Hit:1 http://in.archive.ubuntu.com/ubuntu stonking InRelease
Get:2 http://security.ubuntu.com/ubuntu stonking-security InRelease [60.6 kB]
Hit:3 http://in.archive.ubuntu.com/ubuntu stonking-updates InRelease
Hit:4 http://in.archive.ubuntu.com/ubuntu stonking-backports InRelease
Get:5 http://security.ubuntu.com/ubuntu stonking-security/main arm64 Packages [19.9 kB]
Get:6 http://security.ubuntu.com/ubuntu stonking-security/universe arm64 Packages [12.7 kB]
Fetched 93.2 kB in 2s (54.6 kB/s)   
Reading package lists... Done
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
curl is already the newest version (8.20.0-2ubuntu11.1).
curl set to manually installed.
ca-certificates is already the newest version (20260816ubuntu0.26.10.1).
ca-certificates set to manually installed.
gpg is already the newest version (2.4.9-4ubuntu3).
gpg set to manually installed.
Solving dependencies... Done
The following packages were automatically installed and are no longer required:
  linux-headers-7.3.0-9 linux-headers-7.3.0-9-generic linux-image-unsigned-7.3.0-9-generic linux-main-modules-zfs-7.3.0-9-generic
  linux-modules-7.3.0-9-generic linux-tools-7.3.0-9 linux-tools-7.3.0-9-generic
Use 'sudo apt autoremove' to remove them.
The following additional packages will be installed:
  runc
The following NEW packages will be installed:
  containerd runc
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 31.7 MB of archives.
After this operation, 125 MB of additional disk space will be used.
Get:1 http://in.archive.ubuntu.com/ubuntu stonking/main arm64 runc arm64 1.5.1-0ubuntu2 [7,049 kB]
Get:2 http://in.archive.ubuntu.com/ubuntu stonking/main arm64 containerd arm64 2.3.4+dfsg1-0ubuntu2 [24.7 MB]
Fetched 31.7 MB in 4s (7,634 kB/s)     
Selecting previously unselected package runc.
(Reading database… 155373 files and directories currently installed.)
Preparing to unpack …/runc_1.5.1-0ubuntu2_arm64.deb…
Unpacking runc (1.5.1-0ubuntu2)…
Selecting previously unselected package containerd.
Preparing to unpack …/containerd_2.3.4+dfsg1-0ubuntu2_arm64.deb…
Unpacking containerd (2.3.4+dfsg1-0ubuntu2)…
Setting up runc (1.5.1-0ubuntu2)…
Setting up containerd (2.3.4+dfsg1-0ubuntu2)…
Created symlink '/etc/systemd/system/multi-user.target.wants/containerd.service' → '/usr/lib/systemd/system/containerd.service'.
Processing triggers for man-db (2.13.1-1build1)…
Scanning processes...                                                                                                                          
Scanning linux images...                                                                                                                       

Running kernel seems to be up-to-date (ABI upgrades are not detected).

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
mhasifkamal@workernode2:~$ sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active containerd
active
mhasifkamal@workernode2:~$ sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm
sudo apt-mark hold kubelet kubeadm
sudo systemctl enable --now kubelet
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /
Hit:1 http://in.archive.ubuntu.com/ubuntu stonking InRelease
Hit:3 http://security.ubuntu.com/ubuntu stonking-security InRelease 
Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb  InRelease [1,230 B]
Get:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb  Packages [10.7 kB]
Hit:5 http://in.archive.ubuntu.com/ubuntu stonking-updates InRelease
Hit:6 http://in.archive.ubuntu.com/ubuntu stonking-backports InRelease
Fetched 11.9 kB in 1s (17.7 kB/s)
Reading package lists... Done
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Solving dependencies... Done
The following packages were automatically installed and are no longer required:
  linux-headers-7.3.0-9 linux-headers-7.3.0-9-generic linux-image-unsigned-7.3.0-9-generic linux-main-modules-zfs-7.3.0-9-generic
  linux-modules-7.3.0-9-generic linux-tools-7.3.0-9 linux-tools-7.3.0-9-generic
Use 'sudo apt autoremove' to remove them.
The following additional packages will be installed:
  kubernetes-cni
The following NEW packages will be installed:
  kubeadm kubelet kubernetes-cni
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 58.0 MB of archives.
After this operation, 220 MB of additional disk space will be used.
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb  kubeadm 1.36.6-1.1 [10.7 MB]
Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb  kubernetes-cni 1.9.1-1.1 [36.1 MB]
Get:3 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb  kubelet 1.36.6-1.1 [11.2 MB]
Fetched 58.0 MB in 5s (12.8 MB/s)   
Selecting previously unselected package kubeadm.
(Reading database… 155438 files and directories currently installed.)
Preparing to unpack …/kubeadm_1.36.6-1.1_arm64.deb…
Unpacking kubeadm (1.36.6-1.1)…
Selecting previously unselected package kubernetes-cni.
Preparing to unpack …/kubernetes-cni_1.9.1-1.1_arm64.deb…
Unpacking kubernetes-cni (1.9.1-1.1)…
Selecting previously unselected package kubelet.
Preparing to unpack …/kubelet_1.36.6-1.1_arm64.deb…
Unpacking kubelet (1.36.6-1.1)…
Setting up kubeadm (1.36.6-1.1)…
Setting up kubernetes-cni (1.9.1-1.1)…
Setting up kubelet (1.36.6-1.1)…
Scanning processes...                                                                                                                          
Scanning linux images...                                                                                                                       

Running kernel seems to be up-to-date (ABI upgrades are not detected).

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
kubelet set on hold.
kubeadm set on hold.
mhasifkamal@workernode2:~$ kubeadm join 192.168.64.4:6443 --token 0l9gw0.qccimiku4t73h97j --discovery-token-ca-cert-hash sha256:09e1969b0a994ad27af8f36bf0042bd0f91ce1ea8e87a08db8ce2ce6923403bf
[preflight] Running pre-flight checks
[preflight] Some fatal errors occurred:
	[ERROR IsPrivilegedUser]: user is not running as root
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
error: error execution phase preflight: preflight checks failed
To see the stack trace of this error execute with --v=5 or higher
mhasifkamal@workernode2:~$ sudo kubeadm join 192.168.64.4:6443 --token 0l9gw0.qccimiku4t73h97j --discovery-token-ca-cert-hash sha256:09e1969b0a994ad27af8f36bf0042bd0f91ce1ea8e87a08db8ce2ce6923403bf
[preflight] Running pre-flight checks
	[WARNING Swap]: swap is supported for cgroup v2 only. The kubelet must be properly configured to use swap. Please refer to https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory, or disable swap on the node
[preflight] Some fatal errors occurred:
	[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
error: error execution phase preflight: preflight checks failed
To see the stack trace of this error execute with --v=5 or higher
mhasifkamal@workernode2:~$ sudo swapoff -a
swapon --show
mhasifkamal@workernode2:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-Ly8zyvR4tcYVfy6AZRDXFk41he6iLI3XIx2U7pQi6LEyPqbMhhscqEcchBZU07uv / ext4 defaults 0 1
# /boot was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/907acbb2-0bd5-4e3a-9c88-34af557a42ff /boot ext4 defaults 0 1
# /boot/efi was on /dev/vda1 during curtin installation
/dev/disk/by-uuid/284D-5CF8 /boot/efi vfat defaults 0 1
/swap.img	none	swap	sw	0	0
mhasifkamal@workernode2:~$ sed -i s'/\/swap.img'/#\/swap.img/g /etc/fstab
sed: -e expression #1, char 16: unknown option to `s'
mhasifkamal@workernode2:~$ sudo sed -i '\|^/swap.img[[:space:]]|s|^|#|' /etc/fstab
mhasifkamal@workernode2:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-Ly8zyvR4tcYVfy6AZRDXFk41he6iLI3XIx2U7pQi6LEyPqbMhhscqEcchBZU07uv / ext4 defaults 0 1
# /boot was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/907acbb2-0bd5-4e3a-9c88-34af557a42ff /boot ext4 defaults 0 1
# /boot/efi was on /dev/vda1 during curtin installation
/dev/disk/by-uuid/284D-5CF8 /boot/efi vfat defaults 0 1
#/swap.img	none	swap	sw	0	0
mhasifkamal@workernode2:~$ echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kubernetes.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
* Applying /usr/lib/sysctl.d/10-apparmor.conf ...
* Applying /usr/lib/sysctl.d/10-coredump-debian.conf ...
* Applying /usr/lib/sysctl.d/50-default.conf ...
* Applying /usr/lib/sysctl.d/50-pid-max.conf ...
* Applying /usr/lib/sysctl.d/55-bufferbloat.conf ...
* Applying /usr/lib/sysctl.d/55-console-messages.conf ...
* Applying /usr/lib/sysctl.d/55-ipv6-privacy.conf ...
* Applying /usr/lib/sysctl.d/55-kernel-hardening.conf ...
* Applying /usr/lib/sysctl.d/55-magic-sysrq.conf ...
* Applying /usr/lib/sysctl.d/55-map-count.conf ...
* Applying /usr/lib/sysctl.d/55-network-security.conf ...
* Applying /usr/lib/sysctl.d/55-ptrace.conf ...
* Applying /usr/lib/sysctl.d/55-zeropage.conf ...
* Applying /etc/sysctl.d/99-kubernetes.conf ...
kernel.apparmor_restrict_unprivileged_userns = 1
kernel.apparmor_restrict_unprivileged_unconfined = 1
kernel.core_pattern = core
kernel.sysrq = 0x01b6
kernel.core_uses_pid = 1
net.ipv4.conf.default.rp_filter = 2
net.ipv4.conf.enp0s1.rp_filter = 2
net.ipv4.conf.lo.rp_filter = 2
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.enp0s1.accept_source_route = 0
net.ipv4.conf.lo.accept_source_route = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.enp0s1.promote_secondaries = 1
net.ipv4.conf.lo.promote_secondaries = 1
net.ipv4.ping_group_range = 0 2147483647
net.core.default_qdisc = fq_codel
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_regular = 2
fs.protected_fifos = 1
vm.max_map_count = 1048576
kernel.pid_max = 4194304
net.core.default_qdisc = fq_codel
kernel.printk = 4 4 1 7
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
kernel.kptr_restrict = 1
kernel.sysrq = 176
vm.max_map_count = 1048576
net.ipv4.conf.default.rp_filter = 2
net.ipv4.conf.all.rp_filter = 2
kernel.yama.ptrace_scope = 1
vm.mmap_min_addr = 32768
net.ipv4.ip_forward = 1
net.ipv4.ip_forward = 1
mhasifkamal@workernode2:~$ 