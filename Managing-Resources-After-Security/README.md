# Getting Started with Managing Resources

Learn why resource management is important.

## Why is resource management important?

- Without an indication how much CPU and memory a container needs, Kubernetes has no other option to treat all containers are equally. That often produced a very uneven distribution of resource usage. Asking Kubernetes to schedule containers without resource specifications is like getting in a boat
  without a captain.
- We have come a long way towards understanding many of the essential Kubernetes objects types and principles. One of the most important thing we're still missing is resource management. Kubernetes was blindly scheduling the applications we had deployed so far. We never give it any indication of how many resources we expect thoes application to use, and did not establish any limits.
- Without those indications, Kubernetes carried out it's tasks in a very myopic fashion. Kubernetes could see a lot, but not enough. We'll change that soon. We'll give Kubernetes a pair of glasses that will provide it much better vision.
- Once we learn how to define resources, we'll go further and make sure that certain limitations are set, some default are determined, and there are quotas that will prevent application from overloading the cluster.
- After this chapter, we’ll have enough knowledge tot run Kubernetes in a production environment.

## Enabling Metrics Server#

- The only new thing we'll do this time is to enable one or more add-on. We'll add the Metrics Server to the cluster. It's too soon to explain what it does and why we will need it. For now, just remember that there will soon be something in your cluster called Metrics Server.

---

# Defining Container Memory and CPU Resources

Learn about CPU, Memory and other resource management terminology.

## Getting familiar with the terminologies

- So far we have not specified how much memory and CPU container should be used, nor what their limit should be. If we do that, Kubernetes's scheduler will have a much better idea about the needs of those containers, and it will make much better decisions on which node to place the Pods and what to do if they start misbehaving.

## Looking into the definition:

Let's take a look at modifies `go-demo-2` definition in `go-demo-2-random.yml`. The specification is almost same as those we used before. The only new entries are in the `resources` section.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  selector:
    matchLabels:
      type: db
      service: go-demo-2
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        type: db
        service: go-demo-2
        vendor: MongoLabs
    spec:
      containers:
        - name: db
          image: mongo:3.3
          resources:
            limits:
              memory: 200Mi
              cpu: 0.5
            requests:
              memory: 100Mi
              cpu: 0.3

---
apiVersion: v1
kind: Service
metadata:
  name: go-demo-2-db
spec:
  ports:
    - port: 27017
  selector:
    type: db
    service: go-demo-2

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  replicas: 3
  selector:
    matchLabels:
      type: api
      service: go-demo-2
  template:
    metadata:
      labels:
        type: api
        service: go-demo-2
        language: go
    spec:
      containers:
        - name: api
          image: vfarcic/go-demo-2
          env:
            - name: DB
              value: go-demo-2-db
          readinessProbe:
            httpGet:
              path: /demo/hello
              port: 8080
            periodSeconds: 1
          livenessProbe:
            httpGet:
              path: /demo/hello
              port: 8080
          resources:
            requests:
              memory: 50Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 200m

---
apiVersion: v1
kind: Service
metadata:
  name: go-demo-2-api
spec:
  ports:
    - port: 8080
  selector:
    type: api
    service: go-demo-2
```

- We specified limits and requests entries in the resources section.

## Understanding CPU resources#

- CPU resources are measured in `cpu` units. The exact meaning of a `cpu` unit depends on where we host our cluster. If servers are virtualized, one `cpu` unit is equivalent to one virtualized processor (vCPU). When running on bare-metal with Hyperthreading, one `cpu` equals one Hyperthread. For the sake of simplification, we’ll assume that one `cpu` resource is one CPU processor (even though that is not entirely true).
- If one container is set to use `2` CPU, and the other is set to `1` CPU, the latter is guaranteed half as much processing power.
- CPU values can be fractioned. In our example, the `db` container has the CPU requests set to `0.5` which is equivalent to half CPU. The same value could be expressed as `500m`, which translates to five hundred `millicpu`. If you take another look at the CPU specs of the `api` container, you’ll see that its CPU limit is set to `200m` and the requests to 100m. They are equivalent to `0.2` and `0.1` CPUs.

## Understanding memory resources#

- ## Memory resources follow a similar pattern as CPU. The significant difference is in the units. Memory can be expressed as follows:
  - `K` (Kilobyte)
  - `M` (Megabyte)
  - `G` (Gigabyte)
  - `T` (Terabyte)
  - `P` (Petabyte)
  - `E` (Exabyte)
- We can also use the power-of-two equivalents `Ki`, `Mi`, `Gi`, `Ti`, `Pi`, and `Ei`.
- If we go back to the `go-demo-2-random.yml` definition, we’ll see that the `db` container has the limit set to `200Mi` (two hundred megabytes) and the requests to `100Mi` (one hundred megabytes).

## Limits and requests#

We have already mentioned limits and requests quite a few times, yet we have not explained what each of them means.

### Limits#

- A limit represents the number of resources that a container should not pass. The assumption is that we define limits as upper boundaries which, when reached, indicate that something went wrong, as well as a way to guard our resources from being overtaken by a single rogue container due to memory leaks or similar problems.
- If a container is restartable, Kubernetes will restart a container that exceeds its memory limit. Otherwise, it might terminate it. Keep in mind that a terminated container will be recreated if it belongs to a Pod (as all Kubernetes-controlled containers do).
- Unlike memory, CPU limits never result in termination or restarts. Instead, a container will not be allowed to consume more than the CPU limit for an extended period.

### Requests#

- Requests represent the expected resource utilization. They are used by Kubernetes to decide where to place Pods depending on the actual resource utilization of the nodes that form the cluster.
- If a container exceeds its memory requests, the Pod it resides in might be evicted if a node runs out of memory. Such eviction usually results in the Pod being scheduled on a different node, as long as there is one with enough available memory.
- If a Pod cannot be scheduled to any of the nodes due to a lack of available resources, it enters the pending state, waiting until resources on one of the nodes are freed or a new node is added to the cluster.

---

# Getting Practical with Container Memory and CPU Resources

- Explore the usage of resources with the help of some practical examples.
- In the previous lesson, we discussed the theory of `resources`. Let’s get practical by looking into some practical examples related to `resources`.

## Creating the resources#

We’ll move on and create the resources defined in the `go-demo-2-random.yml` file.

```bash
kubectl create \
    -f go-demo-2-random.yml \
    --record --save-config

kubectl rollout status \
    deployment go-demo-2-api
```

We created the resources and waited until the `go-demo-2-api` Deployment was rolled out. The output of the later command should be as follows.

```bash
deployment "go-demo-2-api" successfully rolled out
```

Let’s describe the `go-demo-2-api` Deployment and see its limits and requests.

```bash
kubectl describe deploy go-demo-2-api
```

The output (limited to the limits and the requests) is as follows:

```yml
...
Pod Template:
  ...
  Containers:
    ...
   Limits:
      cpu:    200m
      memory: 100Mi
    Requests:
      cpu:    100m
      memory: 50Mi
...
```

- We can see that the limits and the requests correspond to those we defined in the go-demo-2-random.yml file. That should come as no surprise.

## Looking into the node description#

Let’s describe the nodes that form the cluster (even though there’s only one).

```bash
kubectl describe node/mycluster-worker
```

The output is as follows:

```bash
...
Capacity:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
System Info:
  Machine ID:                 9d13f7966c8f433da6dffdab04c92ecc
  System UUID:                9d13f7966c8f433da6dffdab04c92ecc
  Boot ID:                    a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Kernel Version:             6.10.14-linuxkit
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://2.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
ProviderID:                   kind://docker/mycluster/mycluster-worker
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                       CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                       ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-ccnbg                   100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     12d
  kube-system                 kindnet-ktw85                              100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      19d
  kube-system                 kube-proxy-7hwc9                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
  local-path-storage          local-path-provisioner-7dc846544d-qt2nh    0 (0%)        0 (0%)      0 (0%)           0 (0%)         12d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                200m (2%)   100m (1%)
  memory             120Mi (3%)  220Mi (5%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
...
```

- Lines 2–7: The Capacity represents the overall capacity of a node. In our case, the mycluster node has 8 CPUs and 4GB of RAM and can run up to one hundred and ten Pods. Those are the upper limits imposed by the hardware.

```bash
kubectl describe node/mycluster-worker2
```

The output is as follows:

```bash
...
Capacity:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
System Info:
  Machine ID:                 364ba2457f86477bba0dadedf53a6bed
  System UUID:                364ba2457f86477bba0dadedf53a6bed
  Boot ID:                    a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Kernel Version:             6.10.14-linuxkit
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://2.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.2.0/24
PodCIDRs:                     10.244.2.0/24
ProviderID:                   kind://docker/mycluster/mycluster-worker2
Non-terminated Pods:          (3 in total)
  Namespace                   Name                        CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                        ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-wz7jv    100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     12d
  kube-system                 kindnet-h4vhj               100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      19d
  kube-system                 kube-proxy-wns7v            0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                200m (2%)   100m (1%)
  memory             120Mi (3%)  220Mi (5%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
...
```

```bash
kubectl describe nodes
```

The output is as follows:

```bash
Name:               mycluster-control-plane
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=arm64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=arm64
                    kubernetes.io/hostname=mycluster-control-plane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 26 Aug 2025 09:54:56 +0530
Taints:             node.kubernetes.io/unreachable:NoExecute
                    node-role.kubernetes.io/control-plane:NoSchedule
                    node.kubernetes.io/unreachable:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  mycluster-control-plane
  AcquireTime:     <unset>
  RenewTime:       Mon, 01 Sep 2025 20:45:02 +0530
Conditions:
  Type             Status    LastHeartbeatTime                 LastTransitionTime                Reason              Message
  ----             ------    -----------------                 ------------------                ------              -------
  MemoryPressure   Unknown   Mon, 01 Sep 2025 19:12:48 +0530   Mon, 01 Sep 2025 20:53:07 +0530   NodeStatusUnknown   Kubelet stopped posting node status.
  DiskPressure     Unknown   Mon, 01 Sep 2025 19:12:48 +0530   Mon, 01 Sep 2025 20:53:07 +0530   NodeStatusUnknown   Kubelet stopped posting node status.
  PIDPressure      Unknown   Mon, 01 Sep 2025 19:12:48 +0530   Mon, 01 Sep 2025 20:53:07 +0530   NodeStatusUnknown   Kubelet stopped posting node status.
  Ready            Unknown   Mon, 01 Sep 2025 19:12:48 +0530   Mon, 01 Sep 2025 20:53:07 +0530   NodeStatusUnknown   Kubelet stopped posting node status.
Addresses:
  InternalIP:  172.19.0.6
  Hostname:    mycluster-control-plane
Capacity:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
System Info:
  Machine ID:                 92e555f12d064564ada72946f185b967
  System UUID:                92e555f12d064564ada72946f185b967
  Boot ID:                    d2034409-42a9-419d-bc89-9718a2c4a6dc
  Kernel Version:             6.10.14-linuxkit
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://2.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
ProviderID:                   kind://docker/mycluster/mycluster-control-plane
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                               ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-lxf72                           100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     19d
  kube-system                 coredns-674b8bbfcf-pp4mc                           100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     19d
  kube-system                 etcd-mycluster-control-plane                       100m (1%)     0 (0%)      100Mi (2%)       0 (0%)         19d
  kube-system                 kindnet-7fsrn                                      100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      19d
  kube-system                 kube-apiserver-mycluster-control-plane             250m (3%)     0 (0%)      0 (0%)           0 (0%)         19d
  kube-system                 kube-controller-manager-mycluster-control-plane    200m (2%)     0 (0%)      0 (0%)           0 (0%)         19d
  kube-system                 kube-proxy-2rv8j                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
  kube-system                 kube-scheduler-mycluster-control-plane             100m (1%)     0 (0%)      0 (0%)           0 (0%)         19d
  local-path-storage          local-path-provisioner-7dc846544d-htrxx            0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                950m (11%)  100m (1%)
  memory             290Mi (7%)  390Mi (9%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
Events:
  Type    Reason          Age   From             Message
  ----    ------          ----  ----             -------
  Normal  RegisteredNode  34m   node-controller  Node mycluster-control-plane event: Registered Node mycluster-control-plane in Controller


Name:               mycluster-worker
Roles:              <none>
Labels:             beta.kubernetes.io/arch=arm64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=arm64
                    kubernetes.io/hostname=mycluster-worker
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 26 Aug 2025 09:55:10 +0530
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  mycluster-worker
  AcquireTime:     <unset>
  RenewTime:       Sun, 14 Sep 2025 11:08:17 +0530
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 14 Sep 2025 11:08:17 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 14 Sep 2025 11:08:17 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 14 Sep 2025 11:08:17 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 14 Sep 2025 11:08:17 +0530   Tue, 26 Aug 2025 09:55:25 +0530   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.19.0.5
  Hostname:    mycluster-worker
Capacity:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
System Info:
  Machine ID:                 9d13f7966c8f433da6dffdab04c92ecc
  System UUID:                9d13f7966c8f433da6dffdab04c92ecc
  Boot ID:                    a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Kernel Version:             6.10.14-linuxkit
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://2.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
ProviderID:                   kind://docker/mycluster/mycluster-worker
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                       CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                       ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-ccnbg                   100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     12d
  kube-system                 kindnet-ktw85                              100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      19d
  kube-system                 kube-proxy-7hwc9                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
  local-path-storage          local-path-provisioner-7dc846544d-qt2nh    0 (0%)        0 (0%)      0 (0%)           0 (0%)         12d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                200m (2%)   100m (1%)
  memory             120Mi (3%)  220Mi (5%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
Events:
  Type     Reason                   Age                From             Message
  ----     ------                   ----               ----             -------
  Normal   Starting                 35m                kube-proxy
  Normal   Starting                 35m                kubelet          Starting kubelet.
  Normal   NodeAllocatableEnforced  35m                kubelet          Updated Node Allocatable limit across pods
  Normal   NodeHasSufficientMemory  35m (x6 over 35m)  kubelet          Node mycluster-worker status is now: NodeHasSufficientMemory
  Normal   NodeHasNoDiskPressure    35m (x6 over 35m)  kubelet          Node mycluster-worker status is now: NodeHasNoDiskPressure
  Normal   NodeHasSufficientPID     35m (x6 over 35m)  kubelet          Node mycluster-worker status is now: NodeHasSufficientPID
  Warning  Rebooted                 35m                kubelet          Node mycluster-worker has been rebooted, boot id: a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Normal   RegisteredNode           34m                node-controller  Node mycluster-worker event: Registered Node mycluster-worker in Controller


Name:               mycluster-worker2
Roles:              <none>
Labels:             beta.kubernetes.io/arch=arm64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=arm64
                    kubernetes.io/hostname=mycluster-worker2
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 26 Aug 2025 09:55:10 +0530
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  mycluster-worker2
  AcquireTime:     <unset>
  RenewTime:       Sun, 14 Sep 2025 11:08:17 +0530
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 14 Sep 2025 11:07:18 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 14 Sep 2025 11:07:18 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 14 Sep 2025 11:07:18 +0530   Tue, 26 Aug 2025 09:55:10 +0530   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 14 Sep 2025 11:07:18 +0530   Tue, 26 Aug 2025 09:55:24 +0530   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.19.0.6
  Hostname:    mycluster-worker2
Capacity:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  61202244Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             4013480Ki
  pods:               110
System Info:
  Machine ID:                 364ba2457f86477bba0dadedf53a6bed
  System UUID:                364ba2457f86477bba0dadedf53a6bed
  Boot ID:                    a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Kernel Version:             6.10.14-linuxkit
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://2.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.2.0/24
PodCIDRs:                     10.244.2.0/24
ProviderID:                   kind://docker/mycluster/mycluster-worker2
Non-terminated Pods:          (3 in total)
  Namespace                   Name                        CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                        ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-wz7jv    100m (1%)     0 (0%)      70Mi (1%)        170Mi (4%)     12d
  kube-system                 kindnet-h4vhj               100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      19d
  kube-system                 kube-proxy-wns7v            0 (0%)        0 (0%)      0 (0%)           0 (0%)         19d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                200m (2%)   100m (1%)
  memory             120Mi (3%)  220Mi (5%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
Events:
  Type     Reason                   Age                From             Message
  ----     ------                   ----               ----             -------
  Normal   Starting                 35m                kube-proxy
  Normal   Starting                 35m                kubelet          Starting kubelet.
  Normal   NodeAllocatableEnforced  35m                kubelet          Updated Node Allocatable limit across pods
  Normal   NodeHasSufficientMemory  35m (x6 over 35m)  kubelet          Node mycluster-worker2 status is now: NodeHasSufficientMemory
  Normal   NodeHasNoDiskPressure    35m (x6 over 35m)  kubelet          Node mycluster-worker2 status is now: NodeHasNoDiskPressure
  Normal   NodeHasSufficientPID     35m (x6 over 35m)  kubelet          Node mycluster-worker2 status is now: NodeHasSufficientPID
  Warning  Rebooted                 35m                kubelet          Node mycluster-worker2 has been rebooted, boot id: a8a85d7a-0ba6-473b-bbe7-0a9af5d66546
  Normal   RegisteredNode           34m                node-controller  Node mycluster-worker2 event: Registered Node mycluster-worker2 in Controller
```

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl create \
    -f go-demo-2-random.yml \
    --record --save-config

kubectl rollout status \
    deployment go-demo-2-api

kubectl describe deploy go-demo-2-api

kubectl describe nodes
```

---

# Measuring the Actual Memory and CPU Consumption

Learn how to measure the actual memory and CPU consumption.

## Exploring the options#

- How did we come up with the current memory and CPU values? Why did we set the memory of the Mongo image to `100Mi`? Why not `50Mi` or `1Gi`? It is embarrassing to admit that the values we have right now are random. We guessed that the containers based on the vfarcic/go-demo-2 image require fewer resources than the Mongo image, so their values are comparatively smaller. That was the only criteria we used to define the resources.
- Before we put random values for resources, we should know that we do not have any metrics to back us up. Anybody’s guess is as good as ours.
- The only way to truly know how much memory and CPU an application use is by retrieving metrics. We’ll use [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) for that purpose.
- Metrics Server collects and interprets various signals like compute resource usage, lifecycle events, etc. In our case, we’re interested only in the CPU and memory consumption of the containers we’re running in our cluster.
- k3d clusters come with metrics-server already deployed as a system application.
- In Kind clusters we have to deploy the metrics-server to measeure the required details of resources.
- The idea of developing a Metrics Server as a tool for monitoring needs is mostly abandoned. Its primary focus is to serve as an internal tool required for some of the Kubernetes features.
- Instead, I’d suggest a combination of [Prometheus](https://prometheus.io/) combined with the Kubernetes API as the source of metrics and [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) for our alerting needs. However, those tools are not in the scope of this chapter, so you might need to educate yourself from their documentation.
  > `Note`: Use Metrics Server only as a quick-and-dirty way to retrieve metrics. Explore the combination of Prometheus and Alertmanager for your monitoring and alerting needs.
- Now that we have clarified what Metrics Server is good for, as well as what it isn’t, we can proceed and confirm that it is indeed running inside our cluster.

```bash
kubectl --namespace kube-system \
    get pods
```

The output is as follows:

```bash
NAME                                              READY   STATUS        RESTARTS      AGE
coredns-674b8bbfcf-ccnbg                          1/1     Running       4 (23h ago)   13d
coredns-674b8bbfcf-lxf72                          1/1     Terminating   0             19d
coredns-674b8bbfcf-pp4mc                          1/1     Terminating   0             19d
coredns-674b8bbfcf-wz7jv                          1/1     Running       4 (23h ago)   13d
etcd-mycluster-control-plane                      1/1     Running       0             19d
kindnet-7fsrn                                     1/1     Running       0             19d
kindnet-h4vhj                                     1/1     Running       5 (23h ago)   19d
kindnet-ktw85                                     1/1     Running       5 (23h ago)   19d
kube-apiserver-mycluster-control-plane            1/1     Running       0             19d
kube-controller-manager-mycluster-control-plane   1/1     Running       0             19d
kube-proxy-2rv8j                                  1/1     Running       0             19d
kube-proxy-7hwc9                                  1/1     Running       5 (23h ago)   19d
kube-proxy-wns7v                                  1/1     Running       5 (23h ago)   19d
kube-scheduler-mycluster-control-plane            1/1     Running       0             19d
```

we don't have any metrics-server running inside kube-system namespace. Let's deploy in our kind cluster.

<!-- ```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

The output is as follows:

```bash
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

Let's verify the metrics-server is deployed or not

```bash
kubectl --namespace kube-system get pods
```

The output is as follows:

```bash
NAME                                              READY   STATUS              RESTARTS      AGE
coredns-674b8bbfcf-ccnbg                          1/1     Running             4 (26h ago)   13d
coredns-674b8bbfcf-lxf72                          1/1     Terminating         0             20d
coredns-674b8bbfcf-pp4mc                          1/1     Terminating         0             20d
coredns-674b8bbfcf-wz7jv                          1/1     Running             4 (26h ago)   13d
etcd-mycluster-control-plane                      1/1     Running             0             20d
kindnet-7fsrn                                     1/1     Running             0             20d
kindnet-h4vhj                                     1/1     Running             5 (26h ago)   20d
kindnet-ktw85                                     1/1     Running             5 (26h ago)   20d
kube-apiserver-mycluster-control-plane            1/1     Running             0             20d
kube-controller-manager-mycluster-control-plane   1/1     Running             0             20d
kube-proxy-2rv8j                                  1/1     Running             0             20d
kube-proxy-7hwc9                                  1/1     Running             5 (26h ago)   20d
kube-proxy-wns7v                                  1/1     Running             5 (26h ago)   20d
kube-scheduler-mycluster-control-plane            1/1     Running             0             20d
metrics-server-867d48dc9c-94q2h                   0/1     ContainerCreating   0             25s
```

- From the logs, your metrics-server pod is created but stuck in 0/1 Running, which usually means the container is up but not healthy. This is a very common issue in KIND / local clusters.
- Here’s how to troubleshoot and fix it step by step:

> ✅ Step 1: Check logs of metrics-server
> Run:

```bash
kubectl -n kube-system logs deploy/metrics-server
```

The output is as follows:

```bash
I0915 07:50:26.737704       1 serving.go:380] Generated self-signed cert (/tmp/apiserver.crt, /tmp/apiserver.key)
I0915 07:50:27.007397       1 handler.go:288] Adding GroupVersion metrics.k8s.io v1beta1 to ResourceManager
I0915 07:50:27.113872       1 secure_serving.go:211] Serving securely on [::]:10250
I0915 07:50:27.113911       1 dynamic_serving_content.go:135] "Starting controller" name="serving-cert::/tmp/apiserver.crt::/tmp/apiserver.key"
I0915 07:50:27.113912       1 configmap_cafile_content.go:205] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
I0915 07:50:27.113915       1 requestheader_controller.go:180] Starting RequestHeaderAuthRequestController
I0915 07:50:27.113926       1 shared_informer.go:350] "Waiting for caches to sync" controller="RequestHeaderAuthRequestController"
I0915 07:50:27.113926       1 shared_informer.go:350] "Waiting for caches to sync" controller="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
I0915 07:50:27.113941       1 configmap_cafile_content.go:205] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
I0915 07:50:27.113946       1 shared_informer.go:350] "Waiting for caches to sync" controller="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
I0915 07:50:27.114096       1 tlsconfig.go:243] "Starting DynamicServingCertificateController"
E0915 07:50:27.116702       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:50:27.122408       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:50:27.128079       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
I0915 07:50:27.214258       1 shared_informer.go:357] "Caches are synced" controller="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
I0915 07:50:27.214312       1 shared_informer.go:357] "Caches are synced" controller="RequestHeaderAuthRequestController"
I0915 07:50:27.214410       1 shared_informer.go:357] "Caches are synced" controller="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
E0915 07:50:41.985807       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:50:41.985808       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:50:41.989783       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:50:48.318935       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:50:56.983572       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:50:56.983653       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:50:56.984379       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:50:58.315123       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:51:08.312797       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:51:11.993738       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:51:11.994265       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:51:11.997370       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:51:18.318110       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:51:26.972040       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:51:26.982443       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:51:27.003715       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:51:28.367422       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:51:38.314243       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:51:41.972111       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:51:41.980047       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:51:41.983885       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:51:44.643329       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:51:54.637617       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:51:56.979777       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:51:56.983342       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:51:56.988908       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:52:04.637495       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:52:11.973626       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:52:11.984422       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:52:11.988751       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:52:14.632023       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:52:24.632402       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:52:26.982025       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:52:26.988641       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:52:26.989580       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:52:34.634249       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:52:41.973336       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:52:41.973980       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:52:41.987575       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
I0915 07:52:44.636793       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:52:54.660193       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:52:56.986065       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:52:56.987299       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:52:56.991300       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:53:04.627129       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:53:11.974263       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:53:11.974873       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
E0915 07:53:11.983814       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
I0915 07:53:14.627966       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0915 07:53:24.646513       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:53:26.972856       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:53:26.976778       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:53:27.033475       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:53:34.637332       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0915 07:53:41.974780       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.5:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.5 because it doesn't contain any IP SANs" node="mycluster-worker"
E0915 07:53:41.974952       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-control-plane"
E0915 07:53:41.982272       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.19.0.6:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.19.0.6 because it doesn't contain any IP SANs" node="mycluster-worker2"
I0915 07:53:44.626126       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
```

The logs contain issues:

```vbnet
tls: failed to verify certificate ... doesn't contain any IP SANs
```

This is exactly the self-signed kubelet certificate problem I suspected. The fix is to tell metrics-server to skip TLS verification when scraping kubelets.

- 🔧 Quick Fix with kubectl patch
  Instead of editing YAML manually, run this one-liner:

```bash
kubectl -n kube-system patch deployment metrics-server \
  --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP"}]'
```

The output is as follows:

```bash
deployment.apps/metrics-server patched
```

### 🚀 Verify

After patching, restart and check:

```bash
kubectl -n kube-system rollout restart deploy metrics-server
# Output: deployment.apps/metrics-server restarted
kubectl -n kube-system get pods | grep metrics-server
```

The output of the later command is as follows:

```bash
metrics-server-f5d799455-t4kqv                    1/1     Running       0             48s
```

- As you can see, “metrics-server” is running.
- Let’s try a very simple query of Metrics Server.

```bash
kubectl top pods
```

The output is as follows:

```bash
error: Metrics API not available
```

That means the APIService (metrics.k8s.io) is not yet ready. This usually takes a minute or two after the pod is up. Let’s troubleshoot:

- ✅ Step 1: Check APIService status

```bash
kubectl get apiservices | grep metrics
```

Expected:

```bash
v1beta1.metrics.k8s.io   kube-system/metrics-server   True    5m
```

But the putput is as follows:

```bash
v1beta1.metrics.k8s.io            kube-system/metrics-server   False (FailedDiscoveryCheck)   30m
```

If AVAILABLE is False or missing, the API aggregation layer hasn’t registered metrics-server yet.

- ✅ Step 2: Describe the APIService

```bash
kubectl describe apiservice v1beta1.metrics.k8s.io
```

- Look for Conditions → Available: False and check the error message (usually a TLS handshake or connection error).
- The output is as follows:

```bash
Name:         v1beta1.metrics.k8s.io
Namespace:
Labels:       k8s-app=metrics-server
Annotations:  <none>
API Version:  apiregistration.k8s.io/v1
Kind:         APIService
Metadata:
  Creation Timestamp:  2025-09-15T07:48:26Z
  Resource Version:    177930
  UID:                 2552c092-757a-435d-90c5-a9101d40f922
Spec:
  Group:                     metrics.k8s.io
  Group Priority Minimum:    100
  Insecure Skip TLS Verify:  true
  Service:
    Name:            metrics-server
    Namespace:       kube-system
    Port:            443
  Version:           v1beta1
  Version Priority:  100
Status:
  Conditions:
    Last Transition Time:  2025-09-15T07:48:26Z
    Message:               failing or missing response from https://10.96.156.233:443/apis/metrics.k8s.io/v1beta1: Get "https://10.96.156.233:443/apis/metrics.k8s.io/v1beta1": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
    Reason:                FailedDiscoveryCheck
    Status:                False
    Type:                  Available
Events:                    <none>
```

The key line is here:

```yml
Message: failing or missing response from https://10.96.156.233:443/apis/metrics.k8s.io/v1beta1
Reason: FailedDiscoveryCheck
Status: False
```

- That means the APIService (metrics.k8s.io) can’t talk to the metrics-server service on port 443.
- This is a very common issue in KIND because the default components.yaml for metrics-server uses a secured port (443), but the container itself is only serving metrics on 4443 (as per args).

#### 🔧 Fix: Point APIService to the correct port

Patch the service so it exposes port 443 → 4443 correctly.

```bash
kubectl -n kube-system edit service metrics-server\
```

Change:

```yml
ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 4443
```

Save & exit.

#### 🔄 Restart metrics-server

```bash
kubectl -n kube-system rollout restart deploy metrics-server
```

#### ✅ Verify

Check again after ~30s:

```bash
kubectl get apiservices v1beta1.metrics.k8s.io -o yaml | grep -A5 "conditions:"
```

The output is as follows:

```yml
conditions:
  - lastTransitionTime: "2025-09-15T07:48:26Z"
    message:
      'failing or missing response from https://10.96.156.233:443/apis/metrics.k8s.io/v1beta1:
      Get "https://10.96.156.233:443/apis/metrics.k8s.io/v1beta1": context deadline
      exceeded (Client.Timeout exceeded while awaiting headers)'
    reason: FailedDiscoveryCheck
```

the condition still shows FailedDiscoveryCheck → meaning the APIService still can’t reach metrics-server.

- Let’s narrow this down step by step:

### 🔎 Step 1: Check Service mapping

Run:

```bash
kubectl -n kube-system get svc metrics-server -o yaml | grep -A5 ports:
```

The putput is as follows:

```bash
  ports:
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: 4443
```

- So the Service mapping is correct (443 → 4443).
- Yet your APIService still says **FailedDiscoveryCheck (context deadline exceeded)** → `meaning the aggregator can’t actually connect to metrics-server`.
- 🔍 Likely Causes in KIND
  1. Pod not reachable via ClusterIP
  - Metrics-server service (10.96.156.233:443) may not be routing correctly.
  - We can test this from inside the cluster.
  2. Liveness probe blocking readiness
  - If probes keep failing, the service might never report healthy to APIService.
- 🛠 Next Debug Steps

  - 1. Exec into another pod and curl the service

    ```bash
     kubectl -n kube-system run curl --rm -i -t --image=curlimages/curl --restart=Never -- \
     curl -vk https://metrics-server.kube-system.svc:443/metrics

    ```

### ✅ Fix: Change the deployment args so the metrics-server listens on 4443:

```bash
kubectl -n kube-system patch deploy metrics-server \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
    "--cert-dir=/tmp",
    "--secure-port=4443",
    "--kubelet-insecure-tls",
    "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
    "--kubelet-use-node-status-port",
    "--metric-resolution=15s"
  ]}]'

```

The output is as follows:

```bash
deployment.apps/metrics-server patched
```

```bash
kubectl -n kube-system delete deploy metrics-server
``` -->

## Step 2: Apply KIND-friendly manifest (everything pre-configured):

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      containers:
        - name: metrics-server
          image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
          args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --kubelet-insecure-tls
            - --kubelet-preferred-address-types=InternalIP
            - --metric-resolution=15s
          ports:
            - name: https
              containerPort: 4443
              protocol: TCP
```

## Step 3: Service

```yml
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    k8s-app: metrics-server
  ports:
    - name: https
      port: 443
      targetPort: 4443
```

```bash
kubectl apply -f metrics-server.yml
```

The output is as follows:

```bash
deployment.apps/metrics-server created
service/metrics-server configured
```

### kubectl -n kube-system get pods -w

```bash
kubectl -n kube-system get pods -w
```

---

# Let’s try a very simple query of Metrics Server.

```bash
kubectl top pods
```

The output is as follows:

```bash
NAME                             CPU(cores)   MEMORY(bytes)
go-demo-2-api-678f4bdd44-ft6t2   4m           17Mi
go-demo-2-api-678f4bdd44-qcsn7   3m           10Mi
go-demo-2-db-85bddb479b-gqt9k    12m          78Mi


# Get details after few minutes

NAME                             CPU(cores)   MEMORY(bytes)
go-demo-2-api-678f4bdd44-ft6t2   4m           25Mi
go-demo-2-api-678f4bdd44-qcsn7   3m           18Mi
go-demo-2-db-85bddb479b-gqt9k    13m          96Mi
```

---

# Allocating Insufficient Resource than the Actual Usage

Explore what happens when we allocate insufficient resources than required by an application.

## Allocating Insufficient Resources#

- Let’s look at a slightly modified version of the `go-demo-2` definition as `go-demo-2-insuf-mem.yml`. When compared with the previous definition, the difference is only in `resources` of the `db` container in the `go-demo-2-db` Deployment.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: db
        image: mongo:3.3
        resources:
          limits:
            memory: 20Mi
            cpu: 0.5
          requests:
            memory: 10Mi
            cpu: 0.3
```

- The memory limit is set to `20Mi` and the request to `10Mi`. Since we already know from Metrics Server’s data that MongoDB requires around `35Mi`, memory resources are much lower than the actual usage this time.

## Applying the definition#

Let’s see what will happen when we apply the new configuration.

```bash
kubectl apply \
    -f go-demo-2-insuf-mem.yml \
    --record

kubectl get pods
```

We apply the new configuration and retrieve the Pods. The output is as follows:

```bash
kubectl get pods
NAME                             READY   STATUS      RESTARTS     AGE
go-demo-2-api-678f4bdd44-kbtkp   0/1     Running     0            9s
go-demo-2-api-678f4bdd44-mp8vx   0/1     Running     0            9s
go-demo-2-api-678f4bdd44-wvwqj   0/1     Running     0            9s
go-demo-2-db-57d9544f4b-vj7s8    0/1     OOMKilled   1 (8s ago)   9s
```

- In your case, the status might not be “OOMKilled”. If so, wait for a while longer and retrieve the Pods again. The status should eventually change to “CrashLoopBackOff”.

```bash
kubectl get pods
NAME                             READY   STATUS             RESTARTS      AGE
go-demo-2-api-678f4bdd44-kbtkp   0/1     Error              3 (20s ago)   80s
go-demo-2-api-678f4bdd44-mp8vx   0/1     Error              3 (20s ago)   80s
go-demo-2-api-678f4bdd44-wvwqj   0/1     Running            3 (20s ago)   80s
go-demo-2-db-57d9544f4b-vj7s8    0/1     CrashLoopBackOff   3 (34s ago)   80s
```

- As you can see, the status of the `go-demo-2-db` Pod is “OOMKilled” (out of memory killed). Kubernetes detected that the actual usage is way above the limit and it declared the Pod as a candidate for termination.
- The container was terminated shortly afterward. Kubernetes will recreate the terminated container a while later only to discover that the memory usage is still above the limit. And so on, and so forth. The loop will continue.
  > Note: A container can exceed its memory request if the node has enough available memory. On the other hand, a container is not allowed to use more memory than the limit. When that happens, it becomes a candidate for termination.

# Looking into the Deployment’s description

Let’s describe the Deployment and see the status of the `db` container.

```bash
kubectl describe pod/go-demo-2-db-57d9544f4b-vj7s8
```

The output is as follows:

```bash
Name:             go-demo-2-db-57d9544f4b-vj7s8
Namespace:        default
Priority:         0
Service Account:  default
Node:             mycluster-worker2/172.19.0.3
Start Time:       Fri, 19 Sep 2025 07:19:58 +0530
Labels:           pod-template-hash=57d9544f4b
                  service=go-demo-2
                  type=db
                  vendor=MongoLabs
Annotations:      <none>
Status:           Running
IP:               10.244.1.9
IPs:
  IP:           10.244.1.9
Controlled By:  ReplicaSet/go-demo-2-db-57d9544f4b
Containers:
  db:
    Container ID:   containerd://8725eb248ce4a1e66a3be3a16c7779eb875f5ecb46766440d86dcb581f01074e
    Image:          mongo:3.3
    Image ID:       docker.io/library/mongo@sha256:08a90c3d7c40aca81f234f0b2aaeed0254054b1c6705087b10da1c1901d07b5d
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Fri, 19 Sep 2025 07:20:12 +0530
      Finished:     Fri, 19 Sep 2025 07:20:13 +0530
    Ready:          False
    Restart Count:  2
    Limits:
      cpu:     500m
      memory:  20Mi
    Requests:
      cpu:        300m
      memory:     10Mi
    Environment:  <none>
    ...
    Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  32s                default-scheduler  Successfully assigned default/go-demo-2-db-57d9544f4b-vj7s8 to mycluster-worker2
  Normal   Pulled     18s (x3 over 32s)  kubelet            Container image "mongo:3.3" already present on machine
  Normal   Created    18s (x3 over 32s)  kubelet            Created container: db
  Normal   Started    18s (x3 over 32s)  kubelet            Started container db
  Warning  BackOff    2s (x3 over 30s)   kubelet            Back-off restarting failed container db in pod go-demo-2-db-57d9544f4b-vj7s8_default(ba6b81a4-4bc2-45ed-b0f7-edef5998f109)
```

- We can see that the last state of the `db` container is `OOMKilled`. When we explore the events, we can see that, so far, the container is restarted eight times with the reason `BackOff`.

## Try it yourself#

For your convenience, a list of all the commands used in this lesson is given below:

```bash
kubectl apply \
    -f go-demo-2-insuf-mem.yml \
    --record

kubectl get pods

kubectl describe pod go-demo-2-db
```

---

# Allocating Excessive Resources

Explore what happens when we allocate excessive resources than required by an application.

## Allocating excessive memory#

Let’s explore another possible situation through yet another updated definition `go-demo-2-insuf-node`. Just as before, the change is only in the resources of the `go-demo-2-db` Deployment.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: db
        image: mongo:3.3
        resources:
          limits:
            memory: 8Gi
            cpu: 0.5
          requests:
            memory: 12Gi
            cpu: 0.3
```

- This time we specify that the requested memory is twice as much as the total memory of the node (6GB). The memory limit is even higher.

## Applying the definition#

Let’s apply the change and observe what happens.

```bash
kubectl apply \
    -f go-demo-2-insuf-node.yml \
    --record

kubectl get pods
```

- The output of the latter command is as follows:

```bash
NAME                             READY   STATUS              RESTARTS   AGE
go-demo-2-api-678f4bdd44-hs4ns   0/1     ContainerCreating   0          9s
go-demo-2-api-678f4bdd44-kfnmx   0/1     ContainerCreating   0          9s
go-demo-2-api-678f4bdd44-xkg94   0/1     ContainerCreating   0          9s
go-demo-2-db-56fb7667cb-949bv    0/1     Pending             0          9s
```

- This time, the status of the Pod is “Pending”. Kubernetes could not place it anywhere in the cluster and is waiting until the situation changes.
- Even though memory requests are associated with containers, it often makes sense to translate them into Pod requirements. We can say that the requested memory of a Pod is the sum of the requests of all the containers that form it. In our case, the Pod has only one container, so the requested memory for the Pod and the container are equal. The same can be said for limits.
- During the scheduling process, Kubernetes sums the requests of a Pod and looks for a node that has enough available memory and CPU. If Pod’s request cannot be satisfied, it is placed in the pending state in the hope that resources will be freed on one of the nodes or that a new server will be added to the cluster. Since such a thing will not happen in our case, the Pod created through the `go-demo-2-db` Deployment will be pending forever unless we change the memory request again.
  > > `Note`: When Kubernetes cannot find enough free resources to satisfy the resource requests of all the containers that form a Pod, it changes its state to Pending. Such Pods will remain in this state until the requested resources become available.

## Looking into the Deployment’s description#

Let’s describe the `go-demo-2-db` Deployment and see whether there is some additional useful information in it.

```bash
kubectl describe pod/go-demo-2-db-56fb7667cb-949bv
```

The output(limited to the events section) is as follows:

```bash
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  3m13s  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient memory. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```

We can see that it has already “FailedScheduling” seven times and that the message clearly indicates that there is “Insufficient memory”.

## Re-Applying the initial definition#

We’ll revert to the initial definition. Even though we know that its resources are incorrect, we know that it satisfies all the requirements and that all the Pods will be scheduled successfully.

```bash
kubectl apply \
    -f go-demo-2-random.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-db

kubectl rollout status \
    deployment go-demo-2-api
```

The output is as follows:

```bash
Waiting for deployment "go-demo-2-db" rollout to finish: 0 of 1 updated replicas are available...
deployment "go-demo-2-db" successfully rolled out

deployment "go-demo-2-api" successfully rolled out
```

## Re-Applying the initial definition#

- We’ll revert to the initial definition. Even though we know that its resources are incorrect, we know that it satisfies all the requirements and that all the Pods will be scheduled successfully.

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
go-demo-2-api-b8c88b7b4-776qb   1/1     Running   0          3m39s
go-demo-2-api-b8c88b7b4-82ttq   1/1     Running   0          3m56s
go-demo-2-api-b8c88b7b4-pfhf2   1/1     Running   0          10m
go-demo-2-db-5558849d58-bbptc   1/1     Running   0          10m
```

Now that all the Pods are running, we should try to write a better definition. For that, we need to observe memory and CPU usage and use that information to decide the requests and the limits.

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl apply \
    -f go-demo-2-insuf-node.yml \
    --record

kubectl get pods

kubectl describe pod go-demo-2-db

kubectl apply \
    -f go-demo-2-random.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-db

kubectl rollout status \
    deployment go-demo-2-api
```

---

# Adjusting Resources Based on Actual Usage

Learn about actual resource usage of running containers and adjusting allocation based on need.

## Getting the actual resource usage#

- We saw some of the effects that can be caused by a discrepancy between resource usage and resource specification. It’s only natural that we should adjust our specification to reflect the actual memory and CPU usage better.

## The DB container#

Let’s start with the database.

```bash
kubectl top pods
```

The output is as follows:

```bash
NAME              CPU(cores) MEMORY(bytes)
go-demo-2-api-... 0m         4Mi
go-demo-2-api-... 0m         4Mi
go-demo-2-api-... 0m         4Mi
go-demo-2-db-...  4m         34Mi
```

As expected, an `api` container uses even fewer resources than MongoDB. Its memory is somewhere between `2Mi` and `6Mi`. Its CPU usage is so low that Metrics Server rounded it to 0m.

> The memory usage resources may differ from the above-mentioned limits.

# Things to keep in mind#

- Equipped with this knowledge, we can proceed to update our YAML definition. Still, before we do that, we need to clarify a few things.
- The metrics we collected are based on applications that do nothing. Once they start getting a real load and start hosting production size data, the metrics will change drastically.
- We need a way to predict how many resources an application will use in production, not in a simple test environment. You might be inclined to run stress tests that simulate a production setup. It’s significant, but it does not necessarily result in real production-like behavior.
- Replicating the production and behavior of real users is tough. Stress tests will only get you so farTo get an accurate estimate, we’ll have to monitor our applications in production and, among other things, adjust resources accordingly.
- There are many additional things we should take into account but for now, we want to stress that applications that do nothing are not a good measure of resource usage. Still, we’re going to imagine that the applications we’re currently running are under production-like load and the metrics we retrieved represent how the applications would behave in production.
  > > Note: Simple test environments do not reflect the production usage of resources. Stress tests are a good start, but not a complete solution. Only production provides real metrics.
- Let’s look at a new definition that better represents resource usage of the applications `go-demo-2.yml`.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: db
        image: mongo:3.3
        resources:
          limits:
            memory: "100Mi"
            cpu: 0.1
          requests:
            memory: "50Mi"
            cpu: 0.01
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: api
        image: vfarcic/go-demo-2
        ...
        resources:
          limits:
            memory: "10Mi"
            cpu: 0.1
          requests:
            memory: "5Mi"
            cpu: 0.01
```

- This is much better. The resource requests are only slightly higher than the current usage. We set the memory limits value to double that of the requests so that the applications have ample resources for occasional (and short-lived) bursts of additional memory consumption. CPU limits are much higher than the requests, mostly because it is not good to put anything less than a tenth of a CPU as the limit.
- The point is that requests are close to the observed usage, and limits are higher so that applications have some space to breathe in case of a temporary spike in resource usage.

## Applying the new definition#

```bash
kubectl apply \
    -f go-demo-2.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-api
```

The deployment "`go-demo-2-api`" is `successfully rolled out`, and we can move to the next subject.

# Try it yourself

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl top pods

kubectl apply \
    -f go-demo-2.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-api
```

---

# Exploring Quality of Service (QoS) Contracts

Explore the Quality of Service contracts and types of QoS classes.

## Understanding the scheduling process#

When we send a request to the Kubernetes API to create a Pod (directly or through one of the controllers), it initiates the scheduling process. To be more precise, what happens next (where it will decide to run a Pod) depends hugely on the resources we defined for the containers that form the Pod. In a nutshell, Kubernetes will decide to deploy a Pod, whenever it is possible, inside one of the nodes that have enough available memory.

## Memory requests and limits#

- When memory requests are defined, Pods will get the memory they requested. If the memory usage of one of the containers exceeds the requested amount, or if some other Pod needs that memory, the Pod hosting it might be terminated. Please note that we wrote that a Pod might be terminated. Whether that will happen depends on the requests from other Pods and the available memory in the cluster. On the other hand, containers that exceed their memory limits are always terminated (unless it is a temporary situation).

## CPU requests and limits#

CPU requests and limits work a bit differently. Containers that exceed specified CPU resources are not killed. Instead, they are throttled.

## Quality of Service (QoS)#

- Now that we have understood how Kubernetes terminates certain activities, we should note that (almost) nothing happens randomly. When there aren’t enough resources to serve the needs of all the Pods, Kubernetes will destroy one or more containers. Its decision to destroy a certain container will not be random, but based on the assigned Quality of Service (QoS). Those with lowest priority will be terminated first.
- Since this might be the first time you have heard about QoS, we’ll spend some time explaining what it is and how it works.
- Pods are the smallest units in Kubernetes. Since almost everything ends up as a Pod (one way or another), it is no wonder that Kubernetes promises specific guarantees to all the Pods running inside the cluster. Whenever we send a request to the API to create or update a Pod, it gets assigned one of the Quality of Service (QoS) classes. They are used to make decisions such as where to schedule a Pod or whether to evict it.
- We do not specify QoS directly. Instead, they are assigned based on the decisions we make with resource requests and limits.

## Exploring the types of QoS classes#

- At the moment, three QoS classes are available. Each Pod can have the `Guaranteed`, the `Burstable`, or the `BestEffort` QoS.

## Guaranteed QoS#

- Guaranteed QoS is assigned only to Pods that have set CPU requests and limits and memory requests and limits for all of their containers. The Pods we created with the last definition match that criteria.
- However, there’s one more necessary condition that must be met. The requests and limits values must be the same per container. There is one more catch. When a container specifies only limits, requests are automatically set to the same values. In other words, containers without requests will have Guaranteed QoS if their limits are defined.
- We can summarize the criteria for Guaranteed QoS as follows.

  - Both memory and CPU limits must be set.
  - Memory and CPU requests must be set to the same values as the limits, or they can be left empty, in which case they default to the limits (we’ll explore them soon).

- Pods with Guaranteed QoS assigned are the top priority and will never be terminated unless they exceed their limits or are unhealthy. They are the last to go when things go wrong. As long as their resource usage is within limits, Kubernetes will always choose to terminate Pods with other QoS assignments when resource usage is over the capacity.

Let’s move to the next QoS.

## Burstable QoS#

- Burstable QoS is assigned to Pods that do not meet the criteria for Guaranteed QoS but have at least one container with memory or CPU requests defined.
- Pods with the Burstable QoS are guaranteed minimal (requested) memory usage. They might be able to use more resources if they are available. If the system is under pressure and needs more available memory, containers belonging to the Pods with the Burstable QoS are more likely to be terminated than those with Guaranteed QoS when there are no Pods with the BestEffort QoS. You can consider the Pods with this QoS as a medium priority.

Finally, we reach the last QoS class.

## BestEffort QoS#

- `BestEffort` QoS is given to the Pods that do not qualify as Guaranteed or Burstable. They are Pods that consist of containers that have none of their resources defined. Containers in Pods qualified as BestEffort can use any available memory they need.
- When in need of more resources, Kubernetes will start terminating containers residing in the Pods with BestEffort QoS. They are the lowest priority, so they’re the first to disappear when more memory is needed.

---

# Examining QoS in Action

Learn to use Qos and see QoS in action.

## Checking QoS with initial definition#

Let's look at the QoS to which our `go-demo-2-db` Pod is assigned:

```bash
kubectl describe pod/go-demo-2-db-7f7754454d-j5cf9
```

The output, limited to relevant parts, is as follows:

```bash
...
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  39s   default-scheduler  Successfully assigned default/go-demo-2-db-7f7754454d-j5cf9 to mycluster-worker2
  Normal  Pulled     38s   kubelet            Container image "mongo:3.3" already present on machine
  Normal  Created    38s   kubelet            Created container: db
  Normal  Started    37s   kubelet            Started container db
```

The Pod is assigned `Burstable` QoS.

`go-demo-2-db` manifest

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  selector:
    matchLabels:
      type: db
      service: go-demo-2
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        type: db
        service: go-demo-2
        vendor: MongoLabs
    spec:
      containers:
        - name: db
          image: mongo:3.3
          resources:
            limits:
              memory: 100Mi
              cpu: 0.1
            requests:
              memory: 50Mi
              cpu: 0.01
```

> > `Note`: Its limits are different from requests, so it did not qualify for Guaranteed QoS.

- Since its resources are set, and it is not eligible for Guaranteed QoS, Kubernetes assigned it the second best QoS.

## Checking QoS with a modified definition#

Now, let’s look at a slightly modified definition `go-demo-2-qos.yml`.

```yml
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
    - host: go-demo-2.com
      http:
        paths:
          - path: /demo
            pathType: ImplementationSpecific
            backend:
              service:
                name: go-demo-2-api
                port:
                  number: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  selector:
    matchLabels:
      type: db
      service: go-demo-2
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        type: db
        service: go-demo-2
        vendor: MongoLabs
    spec:
      containers:
        - name: db
          image: mongo:3.3
          resources:
            limits:
              memory: "50Mi"
              cpu: 0.1
            requests:
              memory: "50Mi"
              cpu: 0.1

---
apiVersion: v1
kind: Service
metadata:
  name: go-demo-2-db
spec:
  ports:
    - port: 27017
  selector:
    type: db
    service: go-demo-2

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  replicas: 3
  selector:
    matchLabels:
      type: api
      service: go-demo-2
  template:
    metadata:
      labels:
        type: api
        service: go-demo-2
        language: go
    spec:
      containers:
        - name: api
          image: vfarcic/go-demo-2
          env:
            - name: DB
              value: go-demo-2-db
          readinessProbe:
            httpGet:
              path: /demo/hello
              port: 8080
            periodSeconds: 1
          livenessProbe:
            httpGet:
              path: /demo/hello
              port: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: go-demo-2-api
spec:
  ports:
    - port: 8080
  selector:
    type: api
    service: go-demo-2
```

This time, we specified that both `cpu` and `memory` should have the same values for both the requests and the limits for the containers that will be created with the `go-demo-2-db` Deployment.

> > `Note`: As a result, db Deployment should be assigned Guaranteed QoS.

- The containers of the `go-demo-2-api` Deployment are void of any resources definitions.
  > > `Note`: The api Deployment should be be assigned BestEffort QoS.

## Applying the definition#

Let’s confirm that both assumptions are indeed correct.

```bash
kubectl apply \
    -f go-demo-2-qos.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-db
```

We apply the new definition and output the rollout status of the go-demo-2-db Deployment.

## Verification of db Deployment#

Now we can describe the Pod created through the `go-demo-2-db` Deployment and check its QoS.

```bash
kubectl describe pod/go-demo-2-db-86b48c995d-rdbdt
```

The output (limited to relevent parts) is as follows:

```bash
...
Containers:
db:
...
Limits:
      cpu:     100m
      memory:  50Mi
    Requests:
      cpu:        100m
      memory:     50Mi
...
QoS Class:                   Guaranteed
...

```

Memory and CPU limits and requests are the same, and, as a result, the QoS is Guaranteed.

## Verification of api Deployment

Let’s check the QoS of the Pods created through the `go-demo-2-api` Deployment.

```bash
kubectl describe pod/go-demo-2-api-6695f75946-5tsbx
```

The output(limited to relevant parts) is as follows:

```bash
...
Containers:
  api:
  ...
QoS Class:                   BestEffort
...
QoS Class:                   BestEffort
...
QoS Class:                   BestEffort
...
```

The three Pods created through the `go-demo-2-api` Deployment are without any resource definitions and, therefore, their QoS is set to BestEffort.

## Destroying the objects#

We don’t need the objects we’ve created so far, so let’s remove them before moving on to the next subject.

```bash
kubectl delete \
    -f go-demo-2-qos.yml
```

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl describe pod go-demo-2-db

kubectl apply \
    -f go-demo-2-qos.yml \
    --record

kubectl rollout status \
    deployment go-demo-2-db

kubectl describe pod go-demo-2-db

kubectl describe pod go-demo-2-api

kubectl delete \
    -f go-demo-2-qos.yml
```

---

# Defining Resource Defaults and Limitations within a Namespace

Learn why resource defaults and limitations are needed and how to define these.

## Why we need resource defaults and limitations?#

- We have already learned how to leverage Kubernetes namespaces to create clusters within a cluster. When combined with RBAC, we can create namespaces and give users permissions to use them without exposing the whole cluster. Still, one thing is missing.
- Let’s say we create a test namespace and allow users to create objects without permitting them to access other namespaces. Even though that is better than allowing everyone full access to the cluster, such a strategy would not prevent people from bringing the whole cluster down or affecting the performance of applications running in other namespaces. The missing piece of the puzzle is resource control on the namespace level.
- We already discussed that every container should have resource `limits` and `requests` defined. That information helps Kubernetes schedule Pods more efficiently. It also provides it with the information it can use to decide whether a Pod should be evicted or restarted.
- Still, the fact that we can specify `resources` does not mean we are forced to define them. We should have the ability to set default `resources` that will be applied when we forget to specify them explicitly.
- Even if we define default `resources`, we also need a way to set limits. Otherwise, everyone with permissions to deploy a Pod can potentially run an application that requests more resources than we’re willing to give.

## Defining default requests and limits#

Our next task is to define default requests and limits and to specify minimum and maximum values someone can define for a Pod.

## Creating a namespace#

We’ll start by creating a `test` Namespace.

```bash
kubectl create ns test
```

## Looking into the definition#

With a playground namespace created, we can look at a new definition `limit-range.yml`.

```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
    - default:
        memory: 50Mi
        cpu: 0.2
      defaultRequest:
        memory: 30Mi
        cpu: 0.05
      max:
        memory: 80Mi
        cpu: 0.5
      min:
        memory: 10Mi
        cpu: 0.01
      type: Container
```

We specify that the resource should be of the `LimitRange` kind. Its spec has four limits.

## The `default` limit and `defaultRequest` entries#

- The `default` limit and `defaultRequest` entries will be applied to the containers that do not specify resources. If a container does not have memory or CPU limits, it’ll be assigned the values set in the `LimitRange`. The default entries are used as limits, and the `defaultRequest` entries are used as requests.

## The min and max values

- When a container does have the resources defined, they will be evaluated against `LimitRange` thresholds specified as `max` and `min`. If a container does not meet the criteria, the Pod that hosts the containers will not be created.

## Creating the resource#

We’ll see a practical implementation of the four `limits` soon. For now, the next step is to create the limit-range` resource.

```bash
kubectl --namespace test create \
    -f limit-range.yml \
    --save-config --record
```

The output is as follows:

```bash
limitrange/limit-range created
```

We create the `LimitRange` resource.

## Looking into the description#

Let’s describe the `test` namespace where the resource was created.

```bash
kubectl describe ns/test
```

The output is as follows:

```bash
Name:         test
Labels:       kubernetes.io/metadata.name=test
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---   ---   ---------------  -------------  -----------------------
 Container  cpu       10m   500m  50m              200m           -
 Container  memory    10Mi  80Mi  30Mi             50Mi           -
```

- We can see that the `test` namespace has the resource limits we specified. We set four out of five possible values.
- The `maxLimitRequestRatio` is missing, and we’ll describe it only briefly. When `MaxLimitRequestRatio` is set, the container request and limit resources must both be non-zero, and the limit divided by the request must be less than or equal to the enumerated value.

## Looking into the updated definition

Let’s look at yet another variation of the `go-demo` definition `go-demo-2-no-res.yml`.

```bash
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      memory: 50Mi
      cpu: 0.2
    defaultRequest:
      memory: 30Mi
      cpu: 0.05
    max:
      memory: 80Mi
      cpu: 0.5
    min:
      memory: 10Mi
      cpu: 0.01
    type: Container
```

The only thing to note is that none of the containers have any resources defined.

## Creating resources#

Next, we’ll create the objects defined in the `go-demo-2-no-res.yml` file.

```bash
kubectl --namespace test create \
    -f go-demo-2-no-res.yml \
    --save-config --record

kubectl --namespace test \
    rollout status \
    deployment go-demo-2-api
```

- We created the objects inside the `test` namespace and wait until the deployment `go-demo-2-api` is successfully rolled out.

## Looking into the description#

Let’s describe one of the Pods we created.

```bash
kubectl --namespace test \
> describe pod/go-demo-2-db-767dbdd68b-4bfbm
```

The output (limited to the relevant parts) is as follows:

```bash
...
Containers:
  db:
...
Limits:
      cpu:     200m
      memory:  50Mi
    Requests:
      cpu:        50m
      memory:     30Mi
```

- Even though we did not specify the resources of the `db` container inside the `go-demo-2-db` Pod, the resources are set. The db container is assigned the `default` limits of the `test` namespace as the container limit. Similarly, the `defaultRequest` limits are used as container requests.
- As we can see, any attempt to create Pods hosting containers without resources will result in the namespace limits applied.

> > Note: We should still define container resources instead of relying on namespace default limits. After all, they are only a fallback in case someone forgot to define resources.

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl create ns test

kubectl --namespace test create \
    -f limit-range.yml \
    --save-config --record

kubectl describe namespace test

kubectl --namespace test create \
    -f go-demo-2-no-res.yml \
    --save-config --record

kubectl --namespace test \
    rollout status \
    deployment go-demo-2-api

kubectl --namespace test describe \
    pod go-demo-2-db
```

---

# The Mismatch Scenario

- Explore what happens when a namespace's defined limits are violated.
- Let’s see what happens when resources are defined but they do not match the namespace `min` and `max` limits.

# Looking into the definition#

We’ll use the same `go-demo-2.yml` that we used before. The output, (limited to the relevant parts) is as follows.

```yml
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: db
        image: mongo:3.3
        resources:
          limits:
            memory: 100Mi
            cpu: 0.1
          requests:
            memory: 50Mi
            cpu: 0.01
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: api
        ...
        resources:
          limits:
            memory: 10Mi
            cpu: 0.1
          requests:
            memory: 5Mi
            cpu: 0.01
...
```

In this output, the resources for both Deployments are defined.

## Creating resources#

Let’s create the objects and retrieve the events to help us understand what’s happening.

```bash
kubectl --namespace test apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace test \
    get events \
    --watch
```

The output of the latter command (limited to the relevant parts) is as follows:

```bash
... Error creating: pods "go-demo-2-db-868dbbc488-s92nm" is forbidden: maximum memory usage per Container is 80Mi, but limit is 100Mi.
...
... Error creating: pods "go-demo-2-api-6bd767ffb6-96mbl" is forbidden: minimum memory usage per Container is 10Mi, but request is 5Mi.
...
```

- We can see that we are not allowed to create either of the two Pods. The difference between those events is in what caused Kubernetes to reject our request.
- The `go-demo-2-db-*` Pod could not be created because its maximum memory usage per Container is `80Mi`, but `limit is 100Mi`. On the other hand, we are not allowed to create the `go-demo-2-api-\*` Pods because the `minimum memory usage per Container is 10Mi`, `but the request is 5Mi`.
  > > `Note`: All the containers within the test namespace will have to comply with the min and max limits. Otherwise, we cannot create them.
- Container limits cannot be higher than the namespace `max` limits. On the other hand, container resource requests cannot be smaller than namespace `min` limits.
- If we think about namespace limits as lower and upper thresholds, we can say that container requests cannot be below them and that container limits can’t be above them.
  > > `Note`: Please press the `CTRL` and `c` keys to stop watching the events.

## Destroying the namespace#

We’ll delete the `test` namespace before we move to the next subject.

```bash
kubectl delete namespace test
```

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl create ns test

kubectl --namespace test apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace test \
    get events \
    --watch

kubectl delete namespace test
```

## Defining Resource Quotas for a Namespace

Learn why resource quotas are used and how to define them.

## The problem#

- Resource defaults and limitations are a good first step towards preventing malicious or accidental deployment of Pods that can potentially produce adverse effects on the cluster. Still, any user with the permissions to create Pods in a namespace can overload the system. Even if `max` values are set to some reasonably small amount of memory and CPU, a user could deploy thousands or even millions of Pods and consume all the available cluster resources. Such an effect might not be even produced out of malice but accidentally.
- A Pod might be attached to a system that scales it automatically without defining upper bounds, and before we know it, it might scale to too many replicas. There are also many other ways things might get out of control.

## The solution#

We need to define namespace boundaries through quotas.

- With quotas, we can guarantee that each namespace gets its fair share of resources. Unlike `LimitRange` rules that are applied to each container, `ResourceQuota` defines namespace limits based on aggregate resource consumption.
- We can use `ResourceQuota` objects to define the total amount of compute resources (memory and CPU) that can be spent in a namespace. We can also use it to limit storage utilization or the number of objects of a certain type that can be created in a namespace.

## Devising a plan#

- Let’s look at the cluster resources we have in our cluster. It is small, and it’s not even a real cluster. However, it’s the only one we have (for now). For now, we’ll consider it as a real cluster…
- Our cluster has 2 CPUs and 2 GB of memory. Now, let’s say that this cluster serves only development and production purposes. We can use the default namespace for production and create a dev namespace for development. We can assume that the production should consume all the resources of the cluster except those given to the dev namespace, which, on the other hand, should not exceed a specific limit.
- The truth is that with 2 CPUs and 2 GB of memory, there isn’t much we can give to developers. Still, we’ll try to be generous. We’ll give them 500 MB and 0.8 CPUs for requests. We’ll allow occasional bursts in resource usage by defining limits of 1 CPU and 1 GB of memory. Furthermore, we might want to limit the number of Pods to ten. Finally, as a way to reduce risks, we will deny developers the right to expose node ports.
- This seems like a decent plan, no? Let’s move on and create the quotas we discussed.

## Defining the quotas#

Let’s look at the dev.yaml definition:

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev
  namespace: dev
spec:
  hard:
    requests.cpu: 0.8
    requests.memory: 500Mi
    limits.cpu: 1
    limits.memory: 1Gi
    pods: 10
    services.nodeports: "0"
```

- Besides creating the `dev` namespace, we also created a `ResourceQuota`. It specifies a set of `hard` limits. Remember, they are based on aggregated data, not on a per-container basis like `LimitRanges`.
- We set requests quotas to `0.8` CPUs and `500Mi` of RAM. Similarly, limit quotas are set to `1` CPU and `1Gi` of memory. Finally, we specify that the `dev` namespace can have only `10` Pods and that there can be no NodePorts. That’s the plan we formulated and defined.

---

# Exploring the Effects by Violating Quotas

Explore how to violate some quotas and analyze the consequences.

## Exploring the effects#

Now let’s create the objects and explore the effects as we defined the resource quotas in the previous lesson.

## Creating the dev namespace#

Let’s get started by creating the dev namespace as per our plan:

```bash
kubectl create ns dev

kubectl create \
    -f dev.yml \
    --record --save-config
```

We can see from the output that the `namespace` "`dev`" is created as well as the `resourcequota` "`dev`". To be on the safe side, we’ll describe the newly created `dev`quota.

```bash
kubectl --namespace dev describe \
    quota dev
```

The output is as follows:

```bash
Name:               dev
Namespace:          dev
Resource            Used  Hard
--------            ----  ----
limits.cpu          0     1
limits.memory       0     1Gi
pods                0     10
requests.cpu        0     800m
requests.memory     0     500Mi
services.nodeports  0     0
```

- We can see that the hard limits are set and that there’s currently no usage. This is expected since we’re not running any objects in the dev namespace.

## Creating resources#

Let’s create `go-demo-2` objects:

```bash
kubectl --namespace dev create \
    -f go-demo-2.yml \
    --save-config --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api
```

We created the objects from the `go-demo-2.yml` file and waited until the `go-demo-2-api` Deployment rolled out.

## Looking into the description#

Now we can revisit the values of the `dev` quota.

```bash
kubectl --namespace dev describe \
    quota dev
```

The output is as follows:

```bash
Name:               dev
Namespace:          dev
Resource            Used   Hard
--------            ----   ----
limits.cpu          400m   1
limits.memory       160Mi  1Gi
pods                4      10
requests.cpu        40m    800m
requests.memory     80Mi   500Mi
services.nodeports  0      0
```

Judging from the `Used` column, we can see that we are currently running `4` Pods and still below the limit of ` 10``. One of those Pods are created through the go-demo-2-dbDeployment, and the other three with thego-demo-2-api. If we summarize the resources we specify for the containers that form those Pods, we’ll see that the values match the used limitsandrequests  `.`

## Violating the number of pods#

So far, we did not reach any of the quotas. Let’s try to break at least one of them `go-demo-2-scaled.yml`.

```bash
...
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  replicas: 15
  ...
```

The definition of the `go-demo-2-scaled.yml` is almost the same as the one in `go-demo-2.yml`. The only difference is that the number of replicas of the g`o-demo-2-api` Deployment is increased to `fifteen`. As you already know, that should result in `fifteen` Pods created through that Deployment.

## Applying the definition#

Now, let’s apply the new definition and see what happens:

```bash
kubectl --namespace dev apply \
    -f go-demo-2-scaled.yml \
    --record
```

We applied the new definition. We’ll give Kubernetes a few moments to do the work before we take a look at the events it’ll generate. So, take a deep breath and count from one to the number of processors in your machine.

```bash
kubectl --namespace dev get events
```

The output is as follows:

```bash
...
3m33s       Warning   FailedCreate        replicaset/go-demo-2-api-6b97b79f84   Error creating: pods "go-demo-2-api-6b97b79f84-rmctf" is forbidden: exceeded quota: dev, requested: limits.cpu=100m,pods=1, used: limits.cpu=1,pods=10, limited: limits.cpu=1,pods=10
3m33s       Warning   FailedCreate        replicaset/go-demo-2-api-6b97b79f84   Error creating: pods "go-demo-2-api-6b97b79f84-6d6dv" is forbidden: exceeded quota: dev, requested: limits.cpu=100m,pods=1, used: limits.cpu=1,pods=10, limited: limits.cpu=1,pods=10
2m56s       Warning   FailedCreate        replicaset/go-demo-2-api-6b97b79f84   (combined from similar events): Error creating: pods "go-demo-2-api-6b97b79f84-f4kd2" is forbidden: exceeded quota: dev, requested: limits.cpu=100m,pods=1, used: limits.cpu=1,pods=10, limited: limits.cpu=1,pods=10
...
```

```bash
kubectl --namespace dev describe quota dev
```

The output is as follows:

```bash
Name:               dev
Namespace:          dev
Resource            Used   Hard
--------            ----   ----
limits.cpu          1      1
limits.memory       260Mi  1Gi
pods                10     10
requests.cpu        100m   800m
requests.memory     130Mi  500Mi
services.nodeports  0      0
```

- We can see that we have reached two of the limits imposed by the namespace quota. We have reached the maximum amount of CPUs (`1`) and Pods (`10`). As a result, the ReplicaSet controller cannot create new Pods.

## Analyzing the error#

We should be able to confirm which hard limits were reached by describing the dev namespace.

```bash
kubectl describe namespace dev
```

The output is as follows:

```bash
Name:         dev
Labels:       kubernetes.io/metadata.name=dev
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:               dev
  Resource            Used   Hard
  --------            ---    ---
  limits.cpu          1      1
  limits.memory       260Mi  1Gi
  pods                10     10
  requests.cpu        100m   800m
  requests.memory     130Mi  500Mi
  services.nodeports  0      0

No LimitRange resource.
```

- As the events showed us, the values of “limits.cpu” and `pods` resources are the same in both the “User” and “Hard” columns. As a result, we won’t be able to create any more Pods, nor will we be allowed to increase CPU limits for those that are already running.
- Finally, let’s look at the Pods inside the `dev` namespace:

```bash
kubectl get pods --namespace dev
```

The output is as follows:

```bash
NAME              READY STATUS  RESTARTS AGE
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        5m
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        5m
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        3m
go-demo-2-api-... 1/1   Running 0        5m
go-demo-2-db-...  1/1   Running 0        5m
```

The `go-demo-2-api` Deployment manages to create nine Pods. Together with the Pod created through the `go-demo-2-db`, we reach the limit (`10`).

## Reverting back to the previous definition#

We have confirmed that the limit and the Pod quotas work. We’ll revert to the previous definition (the one that does not reach any of the quotas) before we move on to the next verification.

```bash
kubectl --namespace dev apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api
```

The output of the latter command should indicate that the deployment "`go-demo-2-api was successfully rolled out`.

## Violating the memory quota#

Let’s look at yet another slightly modified definition of the `go-demo-2` objects `go-demo-2-mem.yml`.

```yml
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-db
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: db
        image: mongo:3.3
        resources:
          limits:
            memory: "100Mi"
            cpu: 0.1
          requests:
            memory: "50Mi"
            cpu: 0.01
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-demo-2-api
spec:
  replicas: 3
  ...
  template:
    ...
    spec:
      containers:
      - name: api
        ...
        resources:
          limits:
            memory: "200Mi"
            cpu: 0.1
          requests:
            memory: "200Mi"
            cpu: 0.01
...
```

- Both memory request and limit of the `api` container of the `go-demo-2-api` Deployment is set to `200Mi` while the database remains with the memory request of `50Mi`. Knowing that the `requests.memory` quota of the dev namespace is `500Mi`, it’s enough to do simple math conclude that we won’t be able to run all three replicas of the `go-demo-2-api` Deployment.

## Applying the definition#

```bash
kubectl --namespace dev apply \
    -f go-demo-2-mem.yml \
    --record
```

Just as before, we should wait for a while before taking a look at the events of the `dev` namespace.

```bash
kubectl --namespace dev get events \
    | grep mem
```

The output is as follows:

```bash
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-ftmbj" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=470Mi, limited: requests.memory=500Mi
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-h5kq8" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=470Mi, limited: requests.memory=500Mi
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-k7c5v" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=470Mi, limited: requests.memory=500Mi
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-4rh5j" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=470Mi, limited: requests.memory=500Mi
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-7x964" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=470Mi, limited: requests.memory=500Mi
24s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-fbb6v" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=460Mi, limited: requests.memory=500Mi
23s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-4g6wl" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=460Mi, limited: requests.memory=500Mi
23s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-n556k" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=460Mi, limited: requests.memory=500Mi
22s         Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   Error creating: pods "go-demo-2-api-7bc4875dbd-fr48q" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=460Mi, limited: requests.memory=500Mi
3s          Warning   FailedCreate        replicaset/go-demo-2-api-7bc4875dbd   (combined from similar events): Error creating: pods "go-demo-2-api-7bc4875dbd-zsd2s" is forbidden: exceeded quota: dev, requested: requests.memory=200Mi, used: requests.memory=460Mi, limited: requests.memory=500Mi
```

- We have reached the quota of the `requests.memory`. As a result, the creation of at least one of the Pods is forbidden. We can see that we requested the creation of a Pod that requests `200Mi` of memory. Since the current summary of the memory requests is `455Mi`, creating that Pod would exceed the allocated `500Mi`.

## Analyzing the error#

Let’s take a closer look at the namespace.

```bash
kubectl describe namespace dev
```

The output is as follows:

```bash
Name:         dev
Labels:       kubernetes.io/metadata.name=dev
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:               dev
  Resource            Used   Hard
  --------            ---    ---
  limits.cpu          400m   1
  limits.memory       520Mi  1Gi
  pods                4      10
  requests.cpu        40m    800m
  requests.memory     460Mi  500Mi
  services.nodeports  0      0

No LimitRange resource.
```

The amount of used memory requests is “460Mi”, meaning that we could create additional Pods with up to “40Mi”, not “200Mi”.

## Reverting back to the previous definition#

We’ll revert to the `go-demo-2.yml` one more time before we explore the last quota we defined.

```bash
kubectl --namespace dev apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api
```

## Violating the services quota#

The only quota we haven’t yet verified is `services.nodeports`. We set it to 0 and, as a result, we should not be allowed to expose any node ports. Let’s confirm if that’s true.

```bash
kubectl expose deployment go-demo-2-api \
    --namespace dev \
    --name go-demo-2-api \
    --port 8080 \
    --type NodePort
```

The output is as follows:

```bash
Error from server (Forbidden): services "go-demo-2-api" is forbidden: exceeded quota: dev, requested: services.nodeports=1, used: services.nodeports=0, limited: services.nodeports=0
```

- All our quotas work as expected. However, we won’t have time to explore examples of all the quotas we can use. Instead, we’ll list them all for future reference.

## Destroying everything#

We are about to delete the cluster for the last time.

```bash
k3d cluster delete mycluster --all
```

`for kind cluster`

```bash
kind delete cluster --name mycluster
```

## Try it yourself#

For your convenience, a list of all the commands used in the lesson is given below:

```bash
kubectl create \
    -f dev.yml \
    --record --save-config

kubectl --namespace dev describe \
    quota dev

kubectl --namespace dev create \
    -f go-demo-2.yml \
    --save-config --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api

kubectl --namespace dev describe \
    quota dev

kubectl --namespace dev apply \
    -f go-demo-2-scaled.yml \
    --record

kubectl --namespace dev get events

kubectl describe namespace dev

kubectl get pods --namespace dev

kubectl --namespace dev apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api

kubectl --namespace dev apply \
    -f go-demo-2-mem.yml \
    --record

kubectl --namespace dev get events \
    | grep mem

kubectl describe namespace dev

kubectl --namespace dev apply \
    -f go-demo-2.yml \
    --record

kubectl --namespace dev \
    rollout status \
    deployment go-demo-2-api

kubectl expose deployment go-demo-2-api \
    --namespace dev \
    --name go-demo-2-api \
    --port 8080 \
    --type NodePort

k3d cluster delete mycluster --all
kind delete cluster --name mycluster
```

---

# Exploring the types of Quotas

Explore the several types/groups of quotas.

- We can divide quotas into several groups.
  > 1. Compute resource quotas#
- **Compute resource quotas** limit the total sum of the compute resources. They are as follows:
  | Resource Name | Description |
  |-----------------|-----------------------------------------------------------------------------|
  | `cpu` | Across all Pods in a non-terminal state, the sum of CPU requests cannot exceed this value. |
  | `limits.cpu` | Across all Pods in a non-terminal state, the sum of CPU limits cannot exceed this value. |
  | `limits.memory` | Across all Pods in a non-terminal state, the sum of memory limits cannot exceed this value. |
  | `memory` | Across all Pods in a non-terminal state, the sum of memory requests cannot exceed this value. |
  | `requests.cpu` | Across all Pods in a non-terminal state, the sum of CPU requests cannot exceed this value. |
  | `requests.memory` | Across all Pods in a non-terminal state, the sum of memory requests cannot exceed this value. |

## 2. Storage resource quotas#

`Storage resource quotas` limit the total sum of the storage resources. We did not yet explore storage (beyond a few local examples), so you might want to keep the list that follows for future reference:
