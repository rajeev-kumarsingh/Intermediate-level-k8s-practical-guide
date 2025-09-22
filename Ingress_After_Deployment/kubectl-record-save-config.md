
# Kubernetes `--record`, `--save-config`, and Rollout History Examples

## 📌 Overview

This guide covers practical usage of:

- `--record` (deprecated)
- `--save-config`
- `kubectl rollout history`
- Best practices with change tracking

---

## 1. ✅ Creating a Deployment with `--save-config`

```bash
kubectl create deployment my-app --image=nginx --save-config -o yaml --dry-run=client > my-app.yaml
kubectl apply -f my-app.yaml
```

> `--save-config` stores the YAML in the object's annotations for future comparison during `kubectl apply`.

---

## 2. 🚫 Using `--record` (Deprecated)

```bash
kubectl apply -f my-app.yaml --record
```

> ❌ As of Kubernetes v1.18+, `--record` is deprecated. The preferred method is to manually annotate the change.

---

## 3. ✅ Manual Change Cause Annotation

```bash
kubectl annotate deployment my-app   kubernetes.io/change-cause="Initial deployment with nginx"
```

---

## 4. 📜 Viewing Rollout History

```bash
kubectl rollout history deployment my-app
```

> Shows the change cause if it was recorded using `--record` (legacy) or manual annotations.

---

## 5. 🚀 Updating the Deployment and Recording Change

```bash
kubectl set image deployment my-app nginx=nginx:1.21
kubectl annotate deployment my-app   kubernetes.io/change-cause="Upgraded nginx to 1.21"
```

Check history again:

```bash
kubectl rollout history deployment my-app
```

---

## ✅ Best Practices

- Always use `kubectl apply` with version-controlled YAML.
- Use `kubectl annotate` to record meaningful change causes.
- Use `kubectl rollout history` to audit deployment changes.

---

## 📚 References

- [Kubernetes kubectl official docs](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [Kubernetes deployment strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
