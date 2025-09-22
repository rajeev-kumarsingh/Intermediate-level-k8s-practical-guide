
# 🚀 k3d Installation Guide for Ubuntu

This guide will walk you through the installation of **k3d** on an Ubuntu system.

---

## 📘 What is k3d?

**k3d** is a lightweight wrapper to run **k3s (Kubernetes)** in Docker containers. It's perfect for local development and CI pipelines.

---

## 🧰 Prerequisites

- **Docker** must be installed and running.

### ✅ Check Docker

```bash
docker --version
```

If not installed, run:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker  # or logout/login to refresh group permissions
```

---

## 🏗 Step-by-Step k3d Installation

### 📦 Step 1: Install k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

This installs `k3d` and places the binary in `/usr/local/bin`.

---

### 🔍 Step 2: Verify Installation

```bash
k3d version
```

Example output:

```
k3d version v5.6.0
k3s version v1.29.1-k3s1 (default)
```

---

### 🚀 Step 3: Create a Kubernetes Cluster

```bash
k3d cluster create mycluster
```

This creates a single-node Kubernetes cluster in Docker named `mycluster`.

---

### ✅ Step 4: Validate the Cluster

Check if your cluster is running:

```bash
kubectl get nodes
kubectl get pods -A
```

You should see the node and system pods up and running.

---

### 🧹 Step 5: Delete the Cluster (Optional)

```bash
k3d cluster delete mycluster
```

---

## 🏁 Done!

You now have a fully working **k3d Kubernetes cluster** on your Ubuntu system. You can now deploy services and test Kubernetes locally.

