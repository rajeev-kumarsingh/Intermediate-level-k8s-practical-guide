
# 🚀 Deploying Prometheus with Ingress-NGINX in Kind Cluster

This guide includes:
- Deploying Prometheus
- Creating Ingress to expose Prometheus
- Troubleshooting common `Ingress-NGINX` errors like webhook failures and Pending pods

---

## 📦 Step 1: Create Prometheus Deployment and Service

```yaml
# prometheus.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  selector:
    matchLabels:
      type: monitor
      service: prometheus
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        type: monitor
        service: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.51.2
        command:
        - /bin/prometheus
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--web.console.libraries=/usr/share"
        - "--web.external-url=https://your-host/prometheus"

---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  ports:
  - port: 9090
  selector:
    type: monitor
    service: prometheus
```

Apply it:

```bash
kubectl apply -f prometheus.yml
```

---

## 🌐 Step 2: Create Ingress for Prometheus

Replace deprecated annotation with modern `ingressClassName`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /prometheus
        pathType: ImplementationSpecific
        backend:
          service:
            name: prometheus
            port:
              number: 9090
```

Apply it:

```bash
kubectl apply -f prometheus-ingress.yaml
```

---

## ❗ Common Error: Admission Webhook Fails

```text
Error: failed calling webhook "validate.nginx.ingress.kubernetes.io":
Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/...":
connect: connection refused
```

### ✅ Fix:

1. Make sure `ingress-nginx-controller` is running
2. Make sure `admission` jobs completed:
   ```bash
   kubectl get jobs -n ingress-nginx
   ```
3. If stuck in `Pending`, run:
   ```bash
   kubectl describe pod -n ingress-nginx ingress-nginx-controller-xxxx
   ```

---

## 🛠 Ingress-NGINX Controller Pod Pending Fix

### Reason:

```text
FailedScheduling: node(s) didn't match Pod's node affinity/selector
```

### Fix:

Check if it uses this nodeSelector:

```yaml
nodeSelector:
  ingress-ready: "true"
```

### Add Label to a Node:

```bash
kubectl label node mycluster-worker ingress-ready=true
```

Then verify:

```bash
kubectl get nodes --show-labels | grep ingress-ready
```

### Restart Controller:

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

---

## ✅ Final Step: Reapply Prometheus Ingress

```bash
kubectl apply -f prometheus.yml
```

Check if everything is running:

```bash
kubectl get pods -A
kubectl get ingress
```

---

## 🎉 Done!

Prometheus is now deployed and exposed via Ingress in your Kind cluster using Ingress-NGINX.

