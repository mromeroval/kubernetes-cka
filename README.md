# Kubernetes CKA Lab Setup

This repository provides a comprehensive guide for setting up a Kubernetes cluster using kubeadm, specifically designed for practicing for the Certified Kubernetes Administrator (CKA) exam.

## Overview

- **Kubernetes Version**: 1.32
- **Container Runtime**: containerd 2.0.3
- **Network Plugin**: Calico
- **Operating System**: Ubuntu 22.04
- **Cluster Type**: Single control-plane, multi-node

## Prerequisites

### System Requirements

- **Operating System**: Ubuntu 22.04 or CentOS
- **Memory**: 2GiB RAM or more
- **CPU**: 2 CPUs or more on the control-plane node
- **Network**: Connectivity between all nodes

### Architecture Check

First, determine your system architecture:

```bash
uname -m
```

- If `x86_64`: Use AMD64 binaries
- If `aarch64` or `arm64`: Use ARM64 binaries

## Table of Contents

1. [System Preparation](#1-system-preparation)
2. [Container Runtime Installation](#2-container-runtime-installation)
3. [Kubernetes Tools Installation](#3-kubernetes-tools-installation)
4. [Control Plane Initialization](#4-control-plane-initialization)
5. [Network Plugin Installation](#5-network-plugin-installation)
6. [Worker Node Setup](#6-worker-node-setup)
7. [Cluster Verification](#7-cluster-verification)
8. [Additional Configuration](#8-additional-configuration)

## 1. System Preparation

### 1.1 Enable IPv4 Packet Forwarding

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### 1.2 Check Firewall Status

```bash
sudo ufw status verbose
```

If inactive, all ports are open (recommended for lab environment).

### 1.3 Disable Swap

Kubernetes requires swap to be disabled:

```bash
# Check current swap status
swapon --show
free -h

# Disable swap if enabled
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 2. Container Runtime Installation

### 2.1 Install containerd

**For AMD64 systems:**
```bash
wget https://github.com/containerd/containerd/releases/download/v2.0.3/containerd-2.0.3-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-2.0.3-linux-amd64.tar.gz
```

**For ARM64 systems:**
```bash
wget https://github.com/containerd/containerd/releases/download/v2.0.3/containerd-2.0.3-linux-arm64.tar.gz
sudo tar Cxzvf /usr/local containerd-2.0.3-linux-arm64.tar.gz
```

### 2.2 Configure containerd

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

### 2.3 Install systemd Service

```bash
sudo mkdir -p /usr/local/lib/systemd/system/
sudo curl https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### 2.4 Configure Systemd Cgroup Driver

Edit the containerd configuration.
The ` SystemdCgroup = true ` setting ensures that containerd uses the systemd cgroup driver, which is required for Kubernetes.

Edit `/etc/containerd/config.toml` and add the following plugin to config.toml under the  [plugins.'io.containerd.grpc.v1.cri']

```bash
sudo nano /etc/containerd/config.toml
```

Find the `[plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.runc.options]` section and ensure the setting is correct or add it if it doesn't exist.

```toml
    [plugins.'io.containerd.grpc.v1.cri'.containerd]
      discard_unpacked_layers = true
      [plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes]
        [plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.runc]
          runtime_type = 'io.containerd.runc.v2'
          [plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.runc.options]
            SystemdCgroup = true
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

### 2.5 Install runc

```bash
sudo apt update
sudo apt install -y runc
```

### 2.6 Install CNI Plugins

**For AMD64 systems:**
```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
```

**For ARM64 systems:**
```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-arm64-v1.6.2.tgz
```

Install the plugins:
```bash
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-*.tgz
```

## 3. Kubernetes Tools Installation

### 3.1 Add Kubernetes Repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 3.2 Install Kubernetes Components

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

## 4. Control Plane Initialization

### 4.1 Initialize the Cluster

**Run only on the control-plane node:**

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### 4.2 Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.3 Save Join Command

Save the `kubeadm join` command output for worker nodes. If lost, regenerate with:

```bash
sudo kubeadm token create --print-join-command
```

## 5. Network Plugin Installation

### 5.1 Download Calico Manifest

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O
```

### 5.2 Configure Calico

Edit the manifest to match your pod network CIDR:

```bash
nano calico.yaml
```

Find and update the CALICO_IPV4POOL_CIDR:

```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```

### 5.3 Apply Calico

```bash
kubectl apply -f calico.yaml
```

### 5.4 Verify Installation

```bash
kubectl get pods -n kube-system
kubectl get nodes
```

## 6. Worker Node Setup

### 6.1 Prepare Worker Nodes

On each worker node, complete steps 1-3:
- **System Preparation** (Section 1)
- **Container Runtime Installation** (Section 2) - Including CNI plugins
- **Kubernetes Tools Installation** (Section 3)

**Important**: CNI plugins (section 2.6) must be installed on all nodes, including worker nodes, as they are required for pod networking.

### 6.2 Join Worker Nodes

Run the join command from step 4.3 on each worker node:

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 6.3 Verify Cluster

From the control-plane node:

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

## 7. Cluster Verification

### 7.1 Test Cluster Functionality

```bash
# Create a test deployment
kubectl create deployment nginx-test --image=nginx --replicas=3

# Check deployment status
kubectl get deployments
kubectl get pods

# Test service connectivity
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get services

# Clean up test resources
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```

## 8. Additional Configuration

### 8.1 kubectl Autocompletion

```bash
# For current session
source <(kubectl completion bash)

# For permanent setup
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

### 8.2 Useful Aliases

```bash
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
```

## Troubleshooting

### Common Issues

1. **Architecture Mismatch**: Ensure all binaries match your system architecture
2. **Swap Enabled**: Kubernetes requires swap to be disabled
3. **Firewall Issues**: Ensure required ports are open between nodes
4. **Container Runtime**: Verify containerd is running and configured correctly

### Useful Commands

```bash
# Check kubelet status
sudo systemctl status kubelet

# View kubelet logs
sudo journalctl -xeu kubelet

# Check containerd status
sudo systemctl status containerd

# Reset cluster (if needed)
sudo kubeadm reset
```

## Next Steps

After successful cluster setup, you can proceed with:

- Practicing CKA exam scenarios

## Resources

- [Official Kubernetes Documentation](https://kubernetes.io/docs/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [containerd Documentation](https://containerd.io/docs/)
- [Calico Documentation](https://docs.projectcalico.org/)