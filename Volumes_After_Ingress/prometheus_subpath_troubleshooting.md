
# Troubleshooting Prometheus Deployment Error in kind Cluster

## Issue

While deploying Prometheus using a custom `prometheus.yml` configuration file in a Kubernetes `kind` cluster, the pod was stuck with the following error:

```shell
kubectl get pods
NAME                          READY   STATUS                       RESTARTS   AGE
prometheus-xxxxxxxxxx-xxxxx   0/1     CreateContainerConfigError   0          8m
```

### Events Log:

```shell
kubectl describe pod prometheus-xxxxxxxxxx-xxxxx

Warning  Failed     ...  kubelet  Error: failed to prepare subPath for volumeMount "prom-conf" of container "prometheus"
```

---

## Root Cause

The `Deployment` YAML used a `hostPath` of type `File` and tried to use a `subPath` in the `volumeMount`. This is invalid.

### Problematic Configuration:

```yaml
volumeMounts:
  - mountPath: /etc/prometheus/prometheus.yml
    name: prom-conf
    subPath: prometheus-conf.yml

volumes:
  - name: prom-conf
    hostPath:
      path: /files/prometheus-conf.yml
      type: File
```

Using `subPath` assumes that the volume is a directory and tries to mount only the specified file inside it. But `hostPath` is directly pointing to a file, not a directory — so the `subPath` cannot be used this way.

---

## Solution

### ✅ Option 1: Remove `subPath` and mount file directly

This is the most straightforward and recommended approach.

#### ✅ Fixed Deployment YAML Snippet:

```yaml
volumeMounts:
  - mountPath: /etc/prometheus/prometheus.yml
    name: prom-conf

volumes:
  - name: prom-conf
    hostPath:
      path: /files/prometheus-conf.yml
      type: File
```

---

## Steps to Apply the Fix

1. Edit your `Deployment` YAML (`prometheus.yml`) to remove `subPath` as shown above.
2. Apply the changes:
   ```bash
   kubectl apply -f prometheus.yml
   ```
3. Restart the deployment to apply volume mount changes:
   ```bash
   kubectl rollout restart deployment prometheus
   ```
4. Monitor the pod:
   ```bash
   kubectl get pods -w
   ```

Once the pod shows `STATUS: Running`, your Prometheus instance is working correctly.

---

## Notes

- If you want to use `subPath`, make sure the `hostPath` is a **directory**, not a single file.
- Always check for `CreateContainerConfigError` and look at `kubectl describe pod <pod-name>` for volume mount errors.

---

## Result

Prometheus pod now runs successfully and mounts the custom configuration as expected.

```shell
kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```
