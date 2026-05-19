# Kubernetes Multi-Node Cluster Setup on AWS EC2 (Ubuntu)

> Complete step-by-step guide to create a Kubernetes multi-node cluster using Ubuntu EC2 instances, CRI-O, kubeadm, and Flannel.

## Tech Stack

* Kubernetes v1.29
* Ubuntu 22.04/24.04
* CRI-O
* kubeadm
* Flannel CNI
* AWS EC2

---

## Overview

This document explains step-by-step how to create a Kubernetes multi-node cluster using:

* Ubuntu EC2 instances
* Kubernetes v1.29
* CRI-O container runtime
* kubeadm
* Flannel CNI
* AWS EC2

Cluster Architecture:

| Node        | Role          |
| ----------- | ------------- |
| k8s-master  | Control Plane |
| k8s-workers | Worker Node   |

---

# Step 1 — Create EC2 Instances

Create 2 Ubuntu EC2 instances.

Recommended:

| Setting       | Value                 |
| ------------- | --------------------- |
| OS            | Ubuntu 22.04/24.04    |
| Instance Type | t3.small or t3.medium |
| Storage       | 20 GB                 |

Instances:

* Master Node
* Worker Node

---

# Step 2 — Configure Security Group

Go to:

AWS Console → EC2 → Security Groups

Add inbound rule:

| Type        | Protocol | Port | Source              |
| ----------- | -------- | ---- | ------------------- |
| All Traffic | All      | All  | Same Security Group |

This allows:

* kubelet communication
* Kubernetes API traffic
* Flannel VXLAN networking
* Inter-node communication

---

# Step 3 — SSH Into Instances

## Master

```bash
ssh -i mykey.pem ubuntu@MASTER_PUBLIC_IP
```

## Worker

```bash
ssh -i mykey.pem ubuntu@WORKER_PUBLIC_IP
```

---

# Step 4 — Set Hostnames

## On Master

```bash
sudo hostnamectl set-hostname k8s-master
```

## On Worker

```bash
sudo hostnamectl set-hostname k8s-workers
```

---

# Step 5 — Get Private IPs

Run on both nodes:

```bash
hostname -I
```

Example:

| Node   | Private IP   |
| ------ | ------------ |
| Master | 172.31.21.31 |
| Worker | 172.31.22.10 |

---

# Step 6 — Configure /etc/hosts

Run on BOTH nodes:

```bash
sudo nano /etc/hosts
```

Add:

```text
172.31.21.31 k8s-master
172.31.22.10 k8s-workers
```

Save file.

---

# Step 7 — Create common.sh

Run on BOTH nodes.

```bash
nano common.sh
```

Paste:

```bash
#!/bin/bash

set -e

echo "========================================"
echo " Kubernetes Common Setup "
echo "========================================"

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system

# Update system
sudo apt update

# Install dependencies
sudo apt install -y \
apt-transport-https \
ca-certificates \
curl \
gpg

# Create keyrings dir
sudo mkdir -p /etc/apt/keyrings

# Kubernetes Repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

# CRI-O Repo
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.29/deb/ /' \
| sudo tee /etc/apt/sources.list.d/cri-o.list

# Install Kubernetes + CRI-O
sudo apt update

sudo apt install -y \
cri-o \
kubelet \
kubeadm \
kubectl

# Hold versions
sudo apt-mark hold kubelet kubeadm kubectl

# Enable services
sudo systemctl enable crio --now
sudo systemctl enable kubelet --now

echo ""
echo "========================================"
echo " Common Setup Complete "
echo "========================================"
```

---

# Step 8 — Run common.sh

Run on BOTH nodes:

```bash
chmod +x common.sh
./common.sh
```

---

# Step 9 — Create master.sh

Run ONLY on MASTER node.

```bash
nano master.sh
```

Paste:

```bash
#!/bin/bash

set -e

MASTER_IP=$(hostname -I | awk '{print $1}')

echo "========================================"
echo " Initializing Kubernetes Master "
echo "========================================"

sudo kubeadm init \
--apiserver-advertise-address=$MASTER_IP \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=Mem
```

Save.

---

# Step 10 — Run master.sh

```bash
chmod +x master.sh
./master.sh
```

After successful initialization you will see:

```bash
kubeadm join ...
```

Copy this command.

---

# Step 11 — Configure kubectl

Run on MASTER:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then:

```bash
export KUBECONFIG=$HOME/.kube/config
```

---

# Step 12 — Install Flannel Network Plugin

Run on MASTER:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Verify:

```bash
kubectl get pods -A
```

Wait until:

* kube-flannel = Running
* coredns = Running

---

# Step 13 — Join Worker Node

On WORKER node run the command copied earlier:

```bash
sudo kubeadm join 172.x.x.x:6443 \
--token xxxxx \
--discovery-token-ca-cert-hash sha256:xxxxx
```

---

# Step 14 — Verify Cluster

Run on MASTER:

```bash
kubectl get nodes
```

Expected:

```text
NAME          STATUS   ROLES           VERSION
k8s-master    Ready    control-plane   v1.29.15
k8s-workers   Ready    <none>          v1.29.15
```

---

# Step 15 — Deploy Test Application

Create nginx deployment:

```bash
kubectl create deployment nginx --image=nginx --replicas=4
```

Verify:

```bash
kubectl get pods -o wide
```

Expected:

```text
nginx-xxxxx   Running   k8s-workers
```

This confirms:

* scheduler working
* pod networking working
* multi-node cluster healthy

---

# Step 16 — Final Verification Screenshot

Best command for screenshot:

```bash
kubectl get nodes && echo "" && kubectl get pods -o wide
```

---

# Important Kubernetes Concepts Learned

## kubeadm

Used to bootstrap Kubernetes cluster.

## kubelet

Agent running on every node.

## CRI-O

Container runtime.

## Flannel

CNI networking plugin.

## Control Plane

Runs:

* API Server
* etcd
* scheduler
* controller-manager

## Worker Node

Runs application pods.

---

# Useful Troubleshooting Commands

## Check Nodes

```bash
kubectl get nodes
```

## Check Pods

```bash
kubectl get pods -A
```

## Describe Pod

```bash
kubectl describe pod POD_NAME
```

## View Logs

```bash
kubectl logs POD_NAME
```

## kubelet Logs

```bash
journalctl -u kubelet -f
```

## CRI-O Status

```bash
systemctl status crio
```

---

# Common Problems

## Worker Node NotReady

Possible causes:

* Security group issue
* Flannel not running
* kubelet stopped

## Pods Stuck ContainerCreating

Usually:

* CNI issue
* Flannel problem

## kubeadm join Failed

Generate new token:

```bash
kubeadm token create --print-join-command
```

---

# Real Production Notes

## Never expose Kubernetes Dashboard publicly.

## Use:

* Lens
* Rancher
* Prometheus
* Grafana

for production management.

---

# What This Setup Proves

This project demonstrates:

* Kubernetes cluster bootstrap
* Multi-node Kubernetes setup
* Worker node joining
* Pod scheduling
* Container runtime integration
* Kubernetes networking
* Deployment management
* Basic DevOps/SRE skills

---

# Final Successful Output

```bash
kubectl get nodes
```

```text
NAME          STATUS   ROLES           AGE     VERSION
k8s-master    Ready    control-plane   v1.29.15
k8s-workers   Ready    <none>          v1.29.15
```

```bash
kubectl get pods -o wide
```

```text
NAME                     READY   STATUS    NODE
nginx-xxxxx              1/1     Running   k8s-workers
```

Cluster successfully created.
