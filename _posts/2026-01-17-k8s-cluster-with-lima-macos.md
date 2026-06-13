# Kubernetes Lab with kubeadm and Lima VMs

I will walk you through my thought process and steps to create a simple kubernetes cluster in your mac. This can be used for learning and also to create a local test environment. I currently use a similar setup to test my production ansible scripts. 

## What You'll Build

A complete Kubernetes cluster with:
- 1 Control Plane node (k8s-control-plane)
- 2 Worker nodes (k8s-worker1, k8s-worker2)
- Cilium networking (replaces kube-proxy)
- Prometheus + Grafana monitoring

## Prerequisites

- macOS with Lima installed
- 8GB+ RAM available
- 20GB+ disk space
- Internet connection

## Step-by-Step Guide

### **Step 1: Create VMs**
Inorder to make the lcoal k8s cluster resemble a prodcution cluster as closely as possible, I needed to use VMs. When looking for VM creation in mac, UTM and Lima are my top two contenders.
I decided to use [Lima](https://github.com/lima-vm/lima), a lightweight virtual machine manager for macOS. I only needed to run linux on the VMs and lima was simple enough to use.

#### Why `user-v2` Networking?

We use the `user-v2` networking mode in Lima because it provides a simple and reliable way to enable network connectivity for the VMs. This mode allows the VMs to access the internet and communicate with each other without requiring complex network configurations or administrative privileges. It also avoids potential conflicts with macOS's native networking stack.

#### Networking Changes for Kubernetes Compatibility

To ensure the VMs are compatible with Kubernetes, we need to make the following networking changes:
1. Enable the `br_netfilter` kernel module to allow bridge traffic to be visible to iptables.
2. Configure sysctl settings to allow IP forwarding and proper handling of bridged traffic.

These changes are necessary because Kubernetes relies on container networking interfaces (CNI) to manage pod-to-pod communication and service discovery.

#### Steps to Create VMs Using Lima Templates

Instead of using a Makefile, we will directly use a Lima configuration file (`k8s.yaml`) to define and create our VMs. Below is an example of the `k8s.yaml` configuration file:

```yaml
# k8s.yaml
images:
  - location: "https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img"
    arch: "x86_64"

cpus: 2
memory: "4GiB"
disk: "20GiB"

mounts:
  - location: "~"
    writable: false
  - location: "/tmp/lima"
    writable: true

network:
  - lima: "shared"
    socket: "user-v2"

provision:
  - mode: system
    script: |
      #!/bin/bash
      sudo modprobe br_netfilter
      echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
      echo 1 | sudo tee /proc/sys/net/ipv4/conf/all/forwarding
      echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
      echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-ip6tables
```

To create the VMs, follow these steps:

1. Save the above configuration as `k8s.yaml` in your working directory.
2. Create the VMs using the following command:
   ```bash
   limactl start --name=k8s-control-plane k8s.yaml
   limactl start --name=k8s-worker1 k8s.yaml
   limactl start --name=k8s-worker2 k8s.yaml
   ```

This will create three VMs: `k8s-control-plane`, `k8s-worker1`, and `k8s-worker2`, each configured with the specified resources and networking settings.

### **Step 2: Initialize Control Plane**

Once the VMs are created, we need to initialize the control plane node. This involves setting up the Kubernetes control plane components and configuring the cluster.

#### Steps to Initialize the Control Plane

1. Access the control plane VM:
   ```bash
   limactl shell k8s-control-plane
   ```

2. Copy the required certificates to the appropriate locations:
   ```bash
   sudo cp /path/to/certs/* /etc/kubernetes/pki/
   ```

3. Install necessary dependencies, such as `kubeadm`, `kubelet`, and `kubectl`:
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubeadm kubelet kubectl
   ```

4. Initialize the Kubernetes control plane:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```

### **Step 3: Configure kubectl**
```bash
# Still on the control plane VM
mkdir $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config

# Copy kubeconfig to your Mac
# (Open new terminal on Mac)
limactl cp k8s-control-plane:.kube/config ~/.kube/config.k8s-on-macos
export KUBECONFIG=~/.kube/config.k8s-on-macos
```

### **Step 4: Install Networking and Monitoring**

Networking and monitoring are critical components of a Kubernetes cluster. We use [Cilium](https://cilium.io/) as the container networking interface (CNI) because it provides advanced networking features, such as network policies and load balancing, and replaces the default `kube-proxy`. For monitoring, we use Prometheus and Grafana, which are widely adopted tools for observability in Kubernetes.

#### Steps to Install Networking and Monitoring

1. Install Cilium for networking:
   ```bash
   # Install Cilium CLI
   curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v0.12.0/cilium-linux-amd64.tar.gz
   tar xzvf cilium-linux-amd64.tar.gz
   sudo mv cilium /usr/local/bin/

   # Deploy Cilium
   cilium install
   cilium status
   ```

2. Install Prometheus and Grafana for monitoring:
   ```bash
   # Deploy Prometheus
   kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

   # Deploy Grafana
   kubectl apply -f https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/templates/deployment.yaml
   ```

### **Step 5: Verify Cluster**
```bash
# Check nodes
kubectl get nodes

# Check pods
kubectl get pods -n kube-system
```

## Test Your Cluster

### **Deploy a Test Application**
```bash
# Deploy nginx
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --type=NodePort

# Check if it's running
kubectl get pods
kubectl get services
```

### **Access Monitoring**
```bash
# Access Prometheus
kubectl port-forward -n kube-system svc/prometheus 9090:9090
# Open http://localhost:9090

# Access Grafana
kubectl port-forward -n kube-system svc/grafana 3000:3000
# Open http://localhost:3000
```

### **Step 6: Clean Up**

After completing the lab, you can clean up by deleting the VMs and the `k8s-lab` folder:

```bash
# Stop and delete all VMs
limactl delete k8s-control-plane
limactl delete k8s-worker1
limactl delete k8s-worker2

# Remove the k8s-lab folder
rm -rf ~/k8s-lab
```

## Troubleshooting

### **Common Issues**

**VMs won't start:**
```bash
# Check Lima status
limactl list
# Restart if needed
limactl start k8s-control-plane
```

**kubectl not working:**
```bash
# Check kubeconfig
export KUBECONFIG=~/.kube/config.k8s-on-macos
kubectl get nodes
```

**Pods not starting:**
```bash
# Check CNI status
kubectl get pods -n kube-system
# Restart Cilium if needed
sudo ./shell/4-cilium.sh
```

## What we Learned

- How Kubernetes clusters are built from scratch
- The role of control plane and worker nodes
- Container networking with Cilium
- Monitoring with Prometheus and Grafana
- Basic kubectl commands

## Next Steps

- Deploy your own applications
- Learn about Kubernetes services and ingress
- Explore persistent storage
- Try advanced networking features

## Reference
[kubeadm config](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/)

## Repository
Find the complete lab setup and scripts in the [k8s-lab folder](https://github.com/ypeavler/lima-k8s) of this repository.

---

*Perfect for beginners who want to understand how Kubernetes clusters work under the hood!*
