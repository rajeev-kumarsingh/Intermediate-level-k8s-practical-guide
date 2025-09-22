# Kubernetes Ingress Troubleshooting Guide (Kind / Bare Metal / No External IP)

This guide helps troubleshoot Ingress not working when `kubectl get ingress` shows no `ADDRESS`, especially in local clusters like Kind or on bare metal.

---

## ✅ Problem: Ingress ADDRESS is blank

```bash
kubectl get ingress -n ingress-lab
NAME        CLASS    HOSTS   ADDRESS   PORTS   AGE
go-demo-2   <none>   *                 80      32m
```

## 🔍 Cause:
Ingress NGINX controller is of type `LoadBalancer`, but your environment (e.g., Kind, Docker, bare metal) does not support cloud-based external load balancers.

---

## ✅ Step-by-Step Solution

### 1. Check Ingress Controller Service Type

```bash
kubectl get svc -n ingress-nginx
```

Expected output:

```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.x.x.x       <pending>     80:30903/TCP,443:31133/TCP   ...
```

### 2. Convert LoadBalancer to NodePort

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "NodePort"}}'
```

Then verify:

```bash
kubectl get svc -n ingress-nginx
```

Look for `NodePort` type and note the port like `80:30903/TCP`.

---

### 3. Get Node IP

```bash
kubectl get nodes -o wide
```

Use the `INTERNAL-IP` of any Ready node, e.g., `172.19.0.5`.

---

### 4. Test Ingress from Inside the Cluster

Run a temporary pod:

```bash
kubectl run curl --image=radial/busyboxplus:curl -it --rm --restart=Never -- sh
```

From inside the pod:

```sh
curl http://go-demo-2-api.ingress-lab.svc.cluster.local:8080
curl http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/demo
```

---

### 5. Check Your Ingress Manifest

Make sure it has:

```yaml
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /demo
        pathType: Prefix
        backend:
          service:
            name: go-demo-2-api
            port:
              number: 8080
```

Apply again:

```bash
kubectl apply -f go-demo-2-ingress.yaml
```

---

### 6. Check Ingress Events and Logs

```bash
kubectl describe ingress go-demo-2 -n ingress-lab
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep demo
```

---

## 🛠 Optional: Recreate Cluster with Host Port Mapping

If you're using Kind, create `kind-ingress-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

Then create cluster:

```bash
kind create cluster --config kind-ingress-config.yaml
```

Apply Ingress NGINX for Kind:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/kind/deploy.yaml
```

Then access via:

```bash
curl http://localhost/demo
```

---

## ✅ Summary

| What to Check | Command |
|---------------|---------|
| Ingress Controller Type | `kubectl get svc -n ingress-nginx` |
| NodePort Assigned | `kubectl patch svc ...` |
| Ingress Config Valid | `kubectl describe ingress ...` |
| Backend Service Reachable | `curl <service>.<namespace>.svc.cluster.local` |
| Ingress Logs | `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` |