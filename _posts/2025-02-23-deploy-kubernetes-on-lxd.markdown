# Setting Up LXD and Kubernetes Cluster

## Installing LXD
LXD is a next-generation system container and virtual machine manager. To install it on your system, use the following command:
```sh
sudo snap install lxd
```
This will install LXD from the Snap package manager, ensuring you have the latest stable version.

## Initializing LXD
Once LXD is installed, initialize it using:
```sh
sudo lxd init
```
You'll be prompted with several configuration options:
- **Clustering**: Choose `no` unless you plan to set up multiple LXD instances.
- **Storage Pool**: Skip this step (`no`) for now.
- **Network Bridge**: Choose `yes` to create a network bridge.
- **IPv4 Address**: Set it to `192.168.101.1/24`.
- **IPv6 Address**: Set to `none` to disable IPv6.

## Creating a Storage Pool
Now, create a storage pool named `pool1`:
```sh
lxc storage create pool1 dir
```
Verify it with:
```sh
lxc storage ls
```

## Creating a Profile for Kubernetes
A profile named `k8s` is required for LXC virtual machines:
```sh
lxc profile create k8s
```
Edit the profile using:
```sh
lxc profile edit k8s
```
Configure it with the following settings:
```yaml
name: k8s
description: K8s Testing LAB
config:
  limits.cpu: "4"
  limits.memory: 8GB
  linux.kernel_modules: ip_tables,ip6_tables,nf_nat,overlay,br_netfilter
  raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop= \nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw sys:rw"
  security.nesting: "true"
  security.privileged: "true"
devices:
  knet1:
    nictype: bridged
    parent: lxdbr0
    type: nic
  root:
    path: /
    pool: pool1
    size: 100GB
    type: disk
```

## Launching LXC Virtual Machines
Launch three Ubuntu 24.04 virtual machines:
```sh
lxc launch ubuntu:24.04 kmaster --vm --profile k8s
lxc launch ubuntu:24.04 kworker1 --vm --profile k8s
lxc launch ubuntu:24.04 kworker2 --vm --profile k8s
```
Verify the instances:
```sh
lxc list
```

## Setting Up Kubernetes Nodes

Repeat the steps on all nodes except Initialize Kubernetes Cluster which we will run only on master node.
Explaining the steps on how to do this on master node:

### Access the Master Node
```sh
lxc shell kmaster
```
Update the system and install dependencies:
```sh
sudo apt update && sudo apt upgrade -y
sudo apt install apt-transport-https curl -y
```

### Install and Configure Containerd
```sh
sudo apt install containerd -y
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Install Kubernetes Components
```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Disable Swap
```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Load Required Kernel Modules
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure System Parameters
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### Initialize Kubernetes Cluster (Master Node Only)
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Set up kubectl for the current user:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Deploy Flannel for networking:
```sh
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Join Worker Nodes to the Cluster
Copy the `kubeadm join` command output from the master node and run it on each worker node:
```sh
kubeadm join 192.168.101.168:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify Cluster Setup
On the master node, check the cluster status:
```sh
kubectl get nodes
kubectl cluster-info
```
Your Kubernetes cluster is now ready for deployments!


