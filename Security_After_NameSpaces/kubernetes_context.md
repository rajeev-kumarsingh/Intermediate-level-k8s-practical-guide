# Kubernetes Context

## 🔎 What is a Context in Kubernetes?
A **context** is a named access configuration in your `kubeconfig` file that tells `kubectl` **which cluster, which user, and which namespace** to use by default.  

It’s basically a **shortcut** for your Kubernetes API connection.  

Think of it like:  
> *"Which cluster am I talking to? Who am I, and in which namespace?"*

---

## 📂 Context Structure
A context is made up of 3 things:

1. **Cluster** → The Kubernetes cluster (API server endpoint, certificate authority).  
2. **User** → The identity used to access the cluster (certs, tokens, etc.).  
3. **Namespace (optional)** → Default namespace for commands.  

---

## 📄 Example: kubeconfig with Contexts

`~/.kube/config` might look like this:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: dev-cluster
  cluster:
    server: https://dev.example.com:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt

- name: prod-cluster
  cluster:
    server: https://prod.example.com:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt

users:
- name: bkumar
  user:
    client-certificate: /etc/kubernetes/pki/users/bkumar.crt
    client-key: /etc/kubernetes/pki/users/bkumar.key

contexts:
- name: dev-context
  context:
    cluster: dev-cluster
    user: bkumar
    namespace: dev

- name: prod-context
  context:
    cluster: prod-cluster
    user: bkumar
    namespace: prod

current-context: dev-context
```

---

## ⚡ Using Contexts

### 1. View contexts
```bash
kubectl config get-contexts
```

Example output:
```
CURRENT   NAME          CLUSTER        AUTHINFO   NAMESPACE
*         dev-context   dev-cluster    bkumar     dev
          prod-context  prod-cluster   bkumar     prod
```

### 2. Switch between contexts
```bash
kubectl config use-context prod-context
```

Now all `kubectl` commands run against the **prod cluster** in the **prod namespace** by default.

### 3. Override namespace without switching context
```bash
kubectl get pods -n dev
```

---

## ✅ Why Contexts Are Useful
- Manage **multiple clusters** (dev, staging, prod).  
- Avoid accidentally deploying to the **wrong cluster**.  
- Save time by not typing `--namespace` every time.  

---

📌 **In short:**  
A **context in Kubernetes is your working environment (Cluster + User + Namespace).**  
You can switch contexts to move between dev, staging, and prod clusters easily.
