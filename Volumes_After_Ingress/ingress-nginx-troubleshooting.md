
# 🔧 Ingress-NGINX Troubleshooting in Kind Cluster

This guide helps you troubleshoot Ingress-NGINX pods stuck in `Pending` status in a Kind (Kubernetes in Docker) cluster.

---

## ✅ Common Symptoms

```bash
kubectl get pods -n ingress-nginx
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxx              0/1     Pending   0          10m
```

---

## 🔍 Step 1: Check Node Status

```bash
kubectl get nodes
```

Ensure node is in `Ready` state.

If node is `NotReady`, it's a sign of deeper issues.

---

## 🛠️ Step 2: Describe the Node

```bash
kubectl describe node <node-name>
```

Look at:
- `Conditions` (e.g., MemoryPressure, DiskPressure)
- `Events` at the bottom for scheduling issues

---

## 📋 Step 3: Check Events in Namespace

```bash
kubectl get events -n ingress-nginx --sort-by=.metadata.creationTimestamp
```

Look for:
- "node(s) had taint [...] that the pod didn't tolerate"
- CNI or networking issues

---

## ✅ Step 4: Patch the Controller to Tolerate Control-Plane Node

```bash
kubectl -n ingress-nginx patch deployment ingress-nginx-controller \
  --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/tolerations",
    "value": [
      {
        "key": "node-role.kubernetes.io/control-plane",
        "operator": "Exists",
        "effect": "NoSchedule"
      },
      {
        "key": "node-role.kubernetes.io/master",
        "operator": "Exists",
        "effect": "NoSchedule"
      }
    ]
  }
]'
```

Then restart:

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

---

## 🔄 Step 5: Reapply Ingress Controller (Optional Clean Slate)

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/kind/deploy.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/kind/deploy.yaml
```

Watch status:

```bash
kubectl get pods -n ingress-nginx -w
```

---

## 🧪 Step 6: Validate

```bash
kubectl get pods -n ingress-nginx
kubectl get events -n ingress-nginx
```

---

## ✅ Success

Once `ingress-nginx-controller` shows `Running`, Ingress is operational.

You can now deploy an Ingress resource and access your services!

---

