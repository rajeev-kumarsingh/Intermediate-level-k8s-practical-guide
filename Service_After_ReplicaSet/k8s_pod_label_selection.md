
# Kubernetes Pod Selection Using Labels (Namespace: service-lab)

## 🧠 Objective
Extract a pod name using labels inside the `service-lab` namespace and assign it to a shell variable.

---

## ✅ Working Example

### Get Pod Name Using Label Selectors
```bash
POD_NAME=$(kubectl get pods -n service-lab --no-headers -o=custom-columns=NAME:.metadata.name -l app=db,type=backend-db | tail -1)
```

### Explanation
- `-n service-lab`: Target the correct namespace.
- `--no-headers`: Remove table headers for easy parsing.
- `-o=custom-columns=NAME:.metadata.name`: Print only the pod names.
- `-l app=db,type=backend-db`: Select pods with matching labels.
- `tail -1`: In case of multiple matches, just take the last one.

### Get Pod Logs
```bash
kubectl logs $POD_NAME -n service-lab
```

### Open Shell in Pod
```bash
kubectl exec -it $POD_NAME -n service-lab -- sh
```

---

## 🔍 Background Labels Used
```bash
kubectl get pods -n service-lab --show-labels
```

### Output Sample
```
NAME              READY   STATUS    RESTARTS   AGE   LABELS
api-rs-fh85l      1/1     Running   0          11m   app=api,env=test,type=backend-api
api-rs-rks6x      1/1     Running   0          11m   app=api,env=test,type=backend-api
api-rs-sb6gk      1/1     Running   0          11m   app=api,env=test,type=backend-api
db-pod-rs-qdlnl   1/1     Running   0          11m   app=db,env=test,type=backend-db
```

---

## 📌 Tip
Always double-check labels using:
```bash
kubectl get pods -n <namespace> --show-labels
```

This helps you adjust `-l` selectors to match the actual labels.
