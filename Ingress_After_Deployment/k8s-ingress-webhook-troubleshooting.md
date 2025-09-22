
# 🚑 Kubernetes Ingress Webhook Troubleshooting Guide

This guide helps resolve the error when applying an Ingress resource:

```
Error from server (InternalError): error when creating "go-demo-2-ingress.yml":
Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io":
failed to call webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/...":
context deadline exceeded
```

---

## ✅ What This Means

The Ingress NGINX controller uses a **validating webhook** to inspect Ingress definitions.

This error means the webhook endpoint exists (`ingress-nginx-controller-admission`), but the Kubernetes API server cannot **reach it or complete the TLS handshake**.

---

## 🔍 Root Causes

| Cause | Description |
|-------|-------------|
| 🔐 TLS Misconfigured | Webhook is unreachable due to invalid certificate setup. |
| 🔒 NetworkPolicy Blocking | Prevents API server from calling the webhook service. |
| 🐞 Buggy or Manual Install | Admission controller was misconfigured during manual install. |
| 🛠️ Local Networking Issues | Happens often with Minikube, kind, k3d, etc. |

---

## 🛠️ Step-by-Step Troubleshooting

### 1. ✅ Check Admission Service Exists

```bash
kubectl get svc -n ingress-nginx
```

You should see:

```
ingress-nginx-controller-admission   ClusterIP   ...   443/TCP
```

### 2. ✅ Check Webhook Endpoints Are Populated

```bash
kubectl get endpoints ingress-nginx-controller-admission -n ingress-nginx
```

Expected output:

```
NAME                                 ENDPOINTS         AGE
ingress-nginx-controller-admission   10.244.2.3:8443   7d
```

If `<none>`, the webhook backend pod is missing.

---

### 3. 🧪 Option 1: Dev Fix — Delete Webhook (Not for Production)

```bash
kubectl delete validatingwebhookconfiguration ingress-nginx-admission
kubectl apply -f go-demo-2-ingress.yml
```

This bypasses the webhook for now.

---

### 4. 🧼 Option 2: Reinstall Ingress NGINX (Helm Recommended)

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

Then reinstall cleanly:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

---

### 5. 🔍 Option 3: Check Webhook Logs

Check logs for deeper diagnosis:

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=admission-webhook
```

---

## ✅ Update Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-demo-2
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /demo
        pathType: ImplementationSpecific
        backend:
          service:
            name: go-demo-2-api
            port:
              number: 8080
```

---

## 📌 Recommendation

| Environment | Suggested Action |
|-------------|------------------|
| Local / Dev | Delete webhook temporarily |
| Production  | Reinstall Ingress with Helm |

---

## 📬 Want Automation?

Ask ChatGPT for a script that:
- Installs Ingress via Helm
- Deploys a test app with Ingress
- Verifies it's working on `/demo`
