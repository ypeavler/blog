
---
layout: post
title: "Podman and Minikube"
categories: Local kubernetes
author:
- Yuva Peavler
---

Follow the steps to setup podman as Docker replacement with Minikube on macOS.

## Prerequisites

- macOS with Homebrew
- 8GB+ RAM, 20GB+ disk space

## Setup

### Install Podman
```bash
brew install podman podman-desktop
podman machine init
podman machine start
```

### Add Corporate Certificates
```bash
podman machine ssh
# Add any corp certificates inside the machine if needed
```

### Create Docker Alias
```bash
sudo tee -a /usr/local/bin/docker <<EOF
#!/bin/bash
exec podman "\$@"
EOF
sudo chmod +x /usr/local/bin/docker
```

### VS Code Configuration
If you use devcontainers a lot and want to use podman instead of docker.

```json
{
  "dev.containers.dockerPath": "podman"
}
```

### Install Minikube
```bash
brew install minikube
```

### Start Cluster
```bash
# Multi-node cluster
minikube start --driver=podman --embed-certs --nodes=3 --cpus=2 --memory=2048

# With CRI-O runtime
minikube start --driver=podman --embed-certs --nodes=3 --cpus=2 --memory=2048 --container-runtime=cri-o -p test-cluster
```

### Enable Registry
```bash
minikube addons enable registry
```

## Test Setup

### Deploy Application
```bash
# Deploy nginx (ARM images for Apple Silicon)
kubectl create deployment hello-minikube --image=docker.io/arm64v8/nginx:latest
kubectl expose deployment hello-minikube --type=NodePort --port=80
```

### Access Application
```bash
# Access via Minikube service
minikube service hello-minikube
# Or with specific profile
minikube service hello-minikube -p test-cluster
```


## Commands

### Podman
```bash
podman machine list
podman machine stop/start
podman ps
podman images
```

### Minikube
```bash
minikube status
minikube stop/start/delete
minikube node list
minikube service list
minikube tunnel
minikube dashboard
```

### Kubernetes
```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

## Tips

- Use `minikube profile` for multiple clusters
- Enable `minikube tunnel` for LoadBalancer services
- Use ARM images for Apple Silicon
- Check `minikube addons list` for features
- Use `podman machine ssh` to debug
