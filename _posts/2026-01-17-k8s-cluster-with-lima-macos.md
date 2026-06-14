---
layout: post
title: "Kubernetes Lab with kubeadm and Lima on macOS"
categories: Local kubernetes
author:
- Yuva Peavler
---

This post walks through a local Kubernetes lab that runs on Lima VMs on
macOS. The goal is to get a
small environment that feels closer to a real kubeadm-based cluster.

This setup is useful for learning, testing cluster bootstrap steps, and
validating infrastructure automation before it is pointed at anything more
important.

## What We'll Build

A small Kubernetes lab with:

- 1 control plane node: `k8s-control-plane`
- 2 worker nodes: `k8s-worker1` and `k8s-worker2`
- kubeadm for cluster bootstrap
- Cilium as the CNI, with `kube-proxy` disabled
- Optional Prometheus and Grafana monitoring

## Why Lima

For this kind of lab, full Linux VMs are more useful than containers
pretending to be nodes. Lima is a good fit on macOS because it is
lightweight, can be automated and easy to setup and tear down.

That matters when we want to:

- Practice kubeadm workflows
- Test kernel and networking settings
- Validate CNI behavior
- Repeat cluster bootstrap from scratch

## Example Repo

The repo for this lab is [the lima-k8s repository](https://github.com/ypeavler/lima-k8s):

- `1-k8s.yaml` - the Lima VM template
- `2-install-deps.sh` - installs Kubernetes packages and CRI tooling
- `3-cluster-init.sh` - runs `kubeadm init` with a generated config
- `3b-join-worker-nodes.sh` - installs worker dependencies and joins workers
- `4-cilium.sh` - installs Cilium with Helm
- `5-monitoring.sh` - enables Prometheus and Grafana-related monitoring bits
- `Makefile` - convenience targets for creating and cleaning up VMs

## Prerequisites

- macOS
- Lima installed and working
- Apple Silicon if you plan to use the current `aarch64` template as-is
- at least 8 GiB of free RAM for the lab
- enough free disk for 3 VMs and container images
- internet access from the VMs

## Step 0: Review the Lima Template

We use a single `k8s.yaml` file at the repo root. This file is
used for each VM.

At a high level, the template does four things:

1. Picks an Ubuntu image and VM type
2. Configures CPU, memory, disk, and mounts
3. Enables Lima networking and NodePort forwarding
4. Prepares the guest kernel and sysctls for Kubernetes networking

### VM type and architecture

```yaml
vmType: vz
arch: "aarch64"
```

That makes sense for Apple Silicon Macs. `vz` uses Apple Virtualization
Framework support, and `aarch64` matches the current Ubuntu ARM image in the
repo.

### Resource settings

```yaml
cpus: 2
memory: "2GiB"
disk: "8GiB"
```

That is enough for a lab, but it is still tight. If you add more workloads,
monitoring, or heavier test apps, expect to increase memory and disk.

### Networking

```yaml
networks:
- lima: user-v2
```

For this lab, `user-v2` is a good default because it gives the VMs outbound
network access and lets the nodes talk to each other without requiring more
complicated host network setup.

The template also forwards the Kubernetes NodePort range:

```yaml
portForwards:
- guestPortRange: [30000, 32767]
  hostPortRange: [30000, 32767]
  hostIP: "0.0.0.0"
```

That makes it easier to test `NodePort` services from the Mac host.

### Why the provisioning section matters

The `provision` block in `k8s.yaml` intentionally stays small. It handles
local guest preparation only:

- loads `overlay` and `br_netfilter`
- loads IPVS-related modules
- writes persistent module configuration under `/etc/modules-load.d`
- enables required sysctls for bridged traffic and IP forwarding

## Step 1: Create the VMs

From the repo root on the Mac, create the VMs with the Makefile:

```bash
make create-single-cp-cluster
```

That target creates and starts the following lima vms:

- `k8s-control-plane`
- `k8s-worker1`
- `k8s-worker2`

Check if the VMs exist:

```bash
limactl list
```

The Makefile also prints the IPs for the three lab VMs after startup.

## Step 2: Install Kubernetes Dependencies on the Control Plane

The preferred interface is the host-side Make target:

```bash
make install-dependencies
```

That target shells into the control-plane VM and runs the dependency install
flow there. The lower-level guest-side commands are still described below for
readers who want to understand what the automation is doing.

This script installs:

- `cri-tools`
- `kubernetes-cni`
- `kubelet`
- `kubeadm`
- `kubectl`

It also:

- configures `crictl`
- adds the Kubernetes apt repository
- enables `kubelet`
- restarts containerd

### Note for corporate proxy or TLS inspection networks

If the guest VM sits behind a corporate proxy or a TLS inspection layer, this
step may fail before package installation starts because the guest does not yet
trust the corporate root certificates.

In that case, install the required corporate root certificates into the vm
trust store first, then re-run `make install-dependencies` from the host.

```bash
limatctl shell <vm1>
sudo mkdir -p /usr/local/share/ca-certificates
sudo cp /path/to/corporate-root-certs/*.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates --fresh
```

If the environment also requires explicit proxy variables, set them in the
guest shell before running the install scripts using the values provided by the
local network or corporate environment.

After that, re-run:

```bash
make install-dependencies
```

The script performs a simple HTTPS preflight and fails early with a more useful
message instead of failing deep inside `apt` or `curl`.

## Step 3: Initialize the Control Plane

The preferred host-side target for this step is:

```bash
make init-control-plane
```

Internally, that host-side target invokes the guest-side `3-cluster-init.sh`
helper inside the control-plane VM.

That helper does more than a plain `kubeadm init` command. It builds a
`kubeadm` config file and then initializes the cluster with choices that match
this lab.

### What `3-cluster-init.sh` actually configures

The script creates a kubeadm config with:

- `containerd` as the CRI socket
- `kube-proxy` disabled
- pod CIDR `10.254.0.0/16`
- service CIDR `10.255.0.0/16`
- cluster name `lima-vm-cluster`

It also:

- pre-pulls Kubernetes images
- uploads control plane certs for future joins
- removes the control plane taint so the control plane can run workloads
- rewrites the kubeconfig server address to `127.0.0.1`
- copies kubeconfig into both root's home and the invoking guest user's home

### Why disable `kube-proxy`

This lab installs Cilium with `kubeProxyReplacement=true`, so the script skips
`kube-proxy` up front. That keeps the cluster configuration aligned with the
networking stack we install next.

### Why remove the control plane taint

In production we usually keep workloads off the control plane. In a small lab,
removing the taint is useful because we only have three nodes and may want to
run extra test workloads without reserving the control plane strictly for
control plane components.

## Step 4: Join the Worker Nodes

Once the control plane is initialized, exit back to your Mac host and run:

```bash
make join-worker-nodes
```

That target calls `3b-join-worker-nodes.sh`, which:

- generates a fresh `kubeadm join` command from the control plane
- installs Kubernetes dependencies on each worker if needed
- joins `k8s-worker1` and `k8s-worker2` to the cluster
- prints the resulting node list at the end

This is the piece that turns the lab from "one initialized control plane plus
two extra VMs" into an actual 3-node cluster.

If the workers also need extra trust roots or explicit proxy variables for
outbound HTTPS, fix that in the VMs first and then re-run
`make join-worker-nodes`.

## Step 5: Configure kubectl on the Mac Host

Once the control plane is initialized, copy the kubeconfig back to macOS:

```bash
make copy-kubeconfig
export KUBECONFIG=~/.kube/config.k8s-on-macos
```

Verify access from the host:

```bash
kubectl get nodes -o wide
```

The control plane init script rewrites the kubeconfig server address to
`127.0.0.1`, which is why this works cleanly from the Mac host.

## Step 6: Install Cilium

Next, install Cilium using the host-side Make target:

```bash
make install-cilium
```

This script:

- installs the Cilium CLI if needed
- installs Helm if needed
- adds the Cilium Helm repo
- installs Cilium with `kubeProxyReplacement=true`
- enables Hubble and Prometheus-related metrics

The script also detects the Kubernetes API address automatically:

- in the single control plane flow it uses the control-plane node IP
- in the HA flow it uses `controlPlaneEndpoint` from the kubeadm config

That matters because the old hardcoded HA VIP did not work for the single
control plane lab.

The script also sets the pod CIDR to match the kubeadm config:

```text
10.254.0.0/16
```

That alignment matters. If the CNI and kubeadm disagree about pod networking,
the cluster will not behave correctly.

## Step 7: Enable Monitoring

If you want the extra monitoring pieces, run:

```bash
make install-monitoring
```

This script reuses the Cilium Helm values and enables the monitoring example
manifests that ship with Cilium 1.17.3.

At that point you should have the pieces needed for Prometheus and Grafana in a
small lab environment.

## Step 8: Verify the Cluster

From your Mac host:

```bash
export KUBECONFIG=~/.kube/config.k8s-on-macos
kubectl get nodes -o wide
kubectl get pods -A
```

From the control plane VM, you can also verify Cilium:

```bash
cilium status --wait
```

Things to check:

- all three nodes show up
- the nodes eventually become `Ready`
- `kube-system` pods are healthy
- Cilium reports ready status

## Step 9: Deploy a Test Workload

A quick smoke test is to deploy nginx:

```bash
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --type=NodePort --port=80
kubectl get pods
kubectl get svc nginx
```

Because the template forwards the NodePort range, you can also inspect the
assigned service port and test it from the host.

## Troubleshooting

### VMs do not start

Check Lima state first:

```bash
limactl list
```

If an instance is broken, stop and recreate it.

### `2-install-deps.sh` fails with HTTPS or certificate errors

That means the guest trust store is not sufficient for the network you are on.
Install the required trust roots in the guest and re-run `make install-dependencies`.

### Workers do not join

Run the helper again:

```bash
make join-worker-nodes
```

The worker join script is idempotent enough for normal retry use. If a worker
already has `/etc/kubernetes/kubelet.conf`, it is treated as already joined.

### `kubectl` cannot reach the cluster

Make sure you exported the right kubeconfig:

```bash
export KUBECONFIG=~/.kube/config.k8s-on-macos
kubectl get nodes
```

If the kubeconfig file was copied before the control plane init completed,
re-copy it.

### Pods stay pending or nodes stay `NotReady`

Check system pods and Cilium status:

```bash
kubectl get pods -A
limactl shell k8s-control-plane cilium status --wait
```

Also confirm that the pod CIDR in the kubeadm config matches the Cilium Helm
settings.

### The lab feels under-provisioned

The current template uses `2GiB` memory and `8GiB` disk per VM. That is enough
for a basic lab, but it is not generous. If you add more workloads or heavy
observability components, increase the VM resources.

## Clean Up

To remove all Lima VMs created for the lab:

```bash
make clean-lima-vms
```

Or remove them manually:

```bash
limactl delete k8s-control-plane
limactl delete k8s-worker1
limactl delete k8s-worker2
```
