# Discovering Services

Learn the process of discovering Services.

## Discovering Services using DNS and environment variables#

Services can be discovered through two principal modes:

- Environment Variable
- DNS

---

Every Pod gets environment variables for each of the active Services. They are provided in the same format as what Docker links expect, as well as with the simpler Kubernetes-specific syntax.

- Let's get environment variables for replicaset-svc:

```bash
POD_NAME=$(kubectl get pods -n service-lab --no-headers -o=custom-columns=NAME:.metadata.name -l app=db,type=backend-db | tail -1)

```

```bash
kubectl exec -n service-lab $POD_NAME -- env
```

```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=db-pod-rs-qdlnl
GOSU_VERSION=1.7
MONGO_MAJOR=3.3
MONGO_VERSION=3.3.15
MONGO_PACKAGE=mongodb-org-unstable
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
API_SERVICE_PORT_8080_TCP_PROTO=tcp
REPLICASET_SVC_SERVICE_HOST=10.96.208.253
REPLICASET_SVC_SERVICE_PORT=27017
REPLICASET_SVC_PORT_27017_TCP_PROTO=tcp
REPLICASET_SVC_PORT_27017_TCP_PORT=27017
API_SERVICE_SERVICE_PORT=8080
KUBERNETES_PORT_443_TCP_PROTO=tcp
REPLICASET_SVC_PORT=tcp://10.96.208.253:27017
API_SERVICE_PORT_8080_TCP_PORT=8080
API_SERVICE_PORT_8080_TCP_ADDR=10.96.219.140
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
REPLICASET_SVC_PORT_27017_TCP_ADDR=10.96.208.253
API_SERVICE_SERVICE_HOST=10.96.219.140
API_SERVICE_PORT_8080_TCP=tcp://10.96.219.140:8080
REPLICASET_SVC_PORT_27017_TCP=tcp://10.96.208.253:27017
API_SERVICE_PORT=tcp://10.96.219.140:8080
KUBERNETES_SERVICE_HOST=10.96.0.1
HOME=/root
```

- The first five variables use the Docker format. If you have already worked with Docker networking, you must be familiar with the way Swarm (standalone) and Docker Compose operate. The later versions of Swarm (Mode) still generate the environment variables but they are mostly abandoned by the users in favor of DNS.

- The last two environment variables are Kubernetes-specific and follow the `[SERVICE_NAME]\_SERVICE_HOST` and `[SERVICE_NAME]\_SERIVCE_PORT` format (the Service name is upper-cased).

- No matter which set of environment variables you choose to use (if any), they all serve the same purpose. They provide a reference that we can use to connect to a Service and, therefore, to the related Pods.
- Things will become more evident when we describe the replicaset-svc Service:

```bash
kubectl describe -n service-lab svc/replicaset-svc
```

The output is as follows:

```bash
Name:                     replicaset-svc
Namespace:                service-lab
Labels:                   <none>
Annotations:              <none>
Selector:                 app=db,env=test,type=backend-db
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.208.253
IPs:                      10.96.208.253
Port:                     <unset>  27017/TCP
TargetPort:               27017/TCP
Endpoints:                10.244.2.9:27017
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

The key is in the IP field. That is the IP through which this service can be accessed and it matches the values of the environment variables `REPLICASET_SVC_SERVICE*\*` and `REPLICASET_SVC_SERVICE_HOST`.

- Let’s look at the snippet from the api-rs.yml ReplicaSet definition:

```yaml
---
env:
  - name: DB
    value: replicaset-svc
---
```

We declare an environment variable with the name of the Service (`replicaset-svc`). That variable is used by the code as a connection string to the database.

Kubernetes converts Service names into DNS and adds them to the DNS server.

## Sequential breakdown of the process#

Let’s go through the sequence of events related to Service discovery and components involved.

- When the api container `api` tries to connect with the `replicaset-svc` Service, it looks at the nameserver configured in `/etc/resolv.conf`. The kubelet configures the nameserver with the `kube-dns` Service IP (10.96.0.10) during the Pod scheduling process.
- The container queries the DNS server listening to port 53. The `replicaset-svc` DNS gets resolved to the service IP `10.0.0.19`. This DNS record was added by kube-dns during the Service and creation process.
- Since we only have one replica of the `db-pod-rs` Pod, iptables forwards requests to just one endpoint. If we had multiple replicas, iptables would act as a load balancer and forward requests randomly among endpoints of the Service.

## Try it yourself#

```bash
POD_NAME=$(kubectl get pods -n service-lab --no-headers -o=custom-columns=NAME:.metadata.name -l app=db,type=backend-db | tail -1)

kubectl exec $POD_NAME -- env

kubectl describe svc replicaset.svc

k3d cluster delete mucluster --all
```

-
-
-
-
-
-
