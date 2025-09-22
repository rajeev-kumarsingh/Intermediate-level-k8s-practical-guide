
# Kubernetes Ingress Explained – `go-demo-2`

This document explains each part of the provided Kubernetes Ingress resource YAML.

## 📄 YAML Overview

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-demo-2
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
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

## 🔍 Step-by-Step Explanation

### 1. `apiVersion: networking.k8s.io/v1`
Specifies the API version used for the Ingress resource.

### 2. `kind: Ingress`
Declares that this is an **Ingress** resource, which is used to expose services over HTTP/HTTPS.

### 3. `metadata:`
- `name: go-demo-2`: This gives the Ingress a name for identification.

### 4. `annotations:`
Used to configure behavior specific to the Ingress Controller (here NGINX).
- `kubernetes.io/ingress.class: "nginx"`: This tells Kubernetes to use the **NGINX Ingress Controller**.
- `ingress.kubernetes.io/ssl-redirect: "false"`: Disables automatic redirect from HTTP to HTTPS (legacy annotation).
- `nginx.ingress.kubernetes.io/ssl-redirect: "false"`: Same as above but in NGINX-specific namespace.

### 5. `spec.rules:`
Defines the routing rules for HTTP.

### 6. `- http:`
Starts an HTTP rule block.

### 7. `paths:`
Defines a list of paths to match for routing traffic.

### 8. `- path: /demo`
Any HTTP request with path `/demo` will match this rule.

### 9. `pathType: ImplementationSpecific`
- Leaves path interpretation up to the ingress controller.
- Often defaults to prefix-based match.

### 10. `backend:`
Defines the target **service** and port to route matching requests.

- `service.name: go-demo-2-api`: This is the **Service** that traffic will be sent to.
- `port.number: 8080`: The port on that service which will receive traffic.

---

## ✅ What This Ingress Does

When a user accesses:

```
http://<your-ingress-controller-ip>/demo
```

The request is forwarded to the `go-demo-2-api` service on port `8080`.

Ensure:
- The NGINX Ingress controller is installed and running.
- The service `go-demo-2-api` exists and is running.
- The Ingress resource is applied in the correct namespace.

---

## 🔧 Apply This Ingress

```bash
kubectl apply -f go-demo-2-ingress.yaml
```

---

## 📌 Notes

- If you're using port-forwarding, you can test with:

```
curl http://localhost:<forwarded-port>/demo
```

- If using domain names and DNS, you'd need to add a host entry and specify it in the `rules.host` field.

---

## 📂 File: `go-demo-2-ingress.yaml`

Save the YAML and apply it as needed in your cluster.

---
