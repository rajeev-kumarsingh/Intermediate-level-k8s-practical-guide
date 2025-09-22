
# Kubernetes ReplicaSet API/DB Troubleshooting Guide

This guide covers the step-by-step troubleshooting process encountered during deployment and testing of ReplicaSet-based API and DB pods in a Kubernetes namespace `service-lab`.

---

## 🔧 Issue 1: Invalid `kubectl get pods` Command

**Command:**
```bash
POD_NAME=$(kubectl get pods --no-header -o=custom-columns=NAME:.metadat.name -l type=dd,service=replicaset-svc | tail -1)
```

**Error:**
```
error: unknown flag: --no-header
```

**Fix:**
Use the correct flag `--no-headers` (note the plural form):
```bash
POD_NAME=$(kubectl get pods --no-headers -o=custom-columns=NAME:.metadata.name -l type=dd,service=replicaset-svc | tail -1)
```

---

## 🔧 Issue 2: Container Not Found During `kubectl exec`

**Command:**
```bash
kubectl exec -it api-rs-ckdjj -n service-lab -- curl http://localhost:8080/demo/hello
```

**Error:**
```
error: Internal error occurred: unable to upgrade connection: container not found ("api")
```

**Fix:**
Ensure the pod name is valid and check if it has the correct container name. Use:
```bash
kubectl get pods -n service-lab
kubectl describe pod <pod-name> -n service-lab
```

---

## 🔧 Issue 3: CrashLoopBackOff with Panic Error

**Command:**
```bash
kubectl logs api-rs-9zp5c -n service-lab
```

**Error:**
```
panic: no reachable servers
```

**Fix:**
- This indicates the API container cannot connect to the MongoDB database.
- Check the DB service name in the API pod environment:
```yaml
env:
  - name: DB
    value: replicaset-svc
```
- Ensure the MongoDB service name and port match:
```yaml
kind: Service
metadata:
  name: replicaset-svc
```

---

## 🔧 Issue 4: Port-Forwarding Incorrect Service Name

**Command:**
```bash
kubectl port-forward service/replicaset-api-svc -n service-lab 8080:8080
```

**Error:**
```
Error from server (NotFound): services "replicaset-api-svc" not found
```

**Fix:**
Use the actual service name defined in your manifest:
```bash
kubectl port-forward service/api-service -n service-lab 8080:8080
```
Then test:
```bash
curl http://localhost:8080/demo/hello
```

---

## ✅ Final Working State

```bash
kubectl get pods -n service-lab -o wide
```

Should show all API and DB pods as `Running` with `1/1` ready status.

---

## 📘 Notes

- Always match service names used in commands with those defined in the YAML.
- Use `kubectl describe` and `kubectl logs` for effective debugging.
- CrashLoopBackOff often indicates missing dependencies or failed health checks.
