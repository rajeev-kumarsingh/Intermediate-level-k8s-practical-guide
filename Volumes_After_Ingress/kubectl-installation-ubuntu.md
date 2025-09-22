# Installing kubectl on Ubuntu

Follow these steps to install the latest version of `kubectl` on Ubuntu:

## Step 1: Update the apt package index and install packages needed to use the Kubernetes apt repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

## Step 2: Download the Google Cloud public signing key

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
  https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

## Step 3: Add the Kubernetes apt repository

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
  https://apt.kubernetes.io/ kubernetes-xenial main" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## Step 4: Update apt package index and install kubectl

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

## Step 5: Verify installation

```bash
kubectl version --client
```

You should see output confirming the `kubectl` client version.

---

For more details, visit the official Kubernetes documentation:  
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
