
# Installing Ingress-NGINX Controller in Kind Cluster

This guide shows how to install the **Ingress-NGINX Controller** in a **Kind** (Kubernetes in Docker) cluster.

---

## 📘 Step 1: Create Kind Cluster with Ingress Support

Create a file named `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
      - containerPort: 443
        hostPort: 443
```

Then create the cluster:

```bash
kind create cluster --name kind-ingress --config kind-config.yaml
```

---

## 📘 Step 2: Install Ingress-NGINX Controller

Use the official manifest for Kind:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/kind/deploy.yaml
```

> 🔄 Replace `v1.11.1` with the latest version if needed: https://github.com/kubernetes/ingress-nginx/releases

---

## 📘 Step 3: Wait for Pods to be Ready

Check the status of the pods:

```bash
kubectl get pods -n ingress-nginx
```

Or use `kubectl wait`:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## 📘 Step 4: Deploy Sample App and Ingress

### Create a sample app:

```bash
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo --port=80 --target-port=80 --type=ClusterIP
```

### Create an ingress resource:

```yaml
# demo-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f demo-ingress.yaml
```

### Test Access:

```bash
curl http://localhost
```

---

## ✅ Done!

You now have the Ingress-NGINX controller running in your Kind cluster!
