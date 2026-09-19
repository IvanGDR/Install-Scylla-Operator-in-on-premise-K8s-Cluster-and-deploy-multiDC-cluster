# Install-Scylla-Operator-in-on-premise-K8s-Cluster-and-deploy-multiDC-cluster

Following documentation for creating a K8s cluster
[Create K8s Cluster with Containerd and Calico](https://github.com/IvanGDR/Deploy-K8S-cluster-using-containerd-and-Calico-network-add-on)

Our next step is to be able to install ScyllaDB on this cluster using Scylla Operator, for this we need to prepare the cluster so ScyllaDB cluster/nodes can be deployed accordingly.

Currently it is only supported to deploy ScyllaDB via Operator on especific Cloud Enviroenments. In general we need to follow operator installation and pre-requisites accordingly

[Scylla Operator - Deploy MultiDC](https://operator.docs.scylladb.com/stable/deploy-scylladb/deploy-multi-datacenter-cluster.html)


In the following exercise, the K8s cluster consists of, one master node and two worker nodes:


```
$ kubectl get nodes
NAME                                 STATUS   ROLES           AGE   VERSION
scylla-ivan-k8s-2-europe-west2-a-1   Ready    control-plane   30d   v1.36.3
scylla-ivan-k8s-2-europe-west2-b-1   Ready    <none>          30d   v1.36.3
scylla-ivan-k8s-2-europe-west2-c-1   Ready    <none>          30d   v1.36.3
```



**a) Taint worker nodes**

```
$ kubectl taint nodes scylla-ivan-k8s-2-europe-west2-b-1 scylla-operator.scylladb.com/dedicated=scyllaclusters:NoSchedule --overwrite

$ kubectl taint nodes scylla-ivan-k8s-2-europe-west2-c-1 scylla-operator.scylladb.com/dedicated=scyllaclusters:NoSchedule --overwrite

```
This adds a taint to a Kubernetes node, which repels pods that don't explicitly tolerate it. These nodes are being reserved for ScyllaDB workloads. Only pods that tolerate the (scylla-operator.scylladb.com/dedicated=scyllaclusters) taint (typically the ScyllaDB operator's own pods, via its scheduling config) will be scheduled onto it — everything else is blocked from landing there going forward.

To confirm taints:
```
$ kubectl get nodes -l scylla.scylladb.com/node-type=scylla   -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

scylla-ivan-k8s-2-europe-west2-b-1	[{"effect":"NoSchedule","key":"scylla-operator.scylladb.com/dedicated","value":"scyllaclusters"}]
scylla-ivan-k8s-2-europe-west2-c-1	[{"effect":"NoSchedule","key":"scylla-operator.scylladb.com/dedicated","value":"scyllaclusters"}]

```
note: Only nodes with taint in place are shown.

To see all taints, accross the k8s cluster, including master node:

```
$ kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

NAME                                 TAINTS
scylla-ivan-k8s-2-europe-west2-a-1   [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
scylla-ivan-k8s-2-europe-west2-b-1   [map[effect:NoSchedule key:scylla-operator.scylladb.com/dedicated value:scyllaclusters]]
scylla-ivan-k8s-2-europe-west2-c-1   [map[effect:NoSchedule key:scylla-operator.scylladb.com/dedicated value:scyllaclusters]]

```


**b) Label worker nodes**

```
$ kubectl label nodes scylla-ivan-k8s-2-europe-west2-b-1 scylla.scylladb.com/node-type=scylla

$ kubectl label nodes scylla-ivan-k8s-2-europe-west2-c-1 scylla.scylladb.com/node-type=scylla
```

and

```
$ kubectl label nodes scylla-ivan-k8s-2-europe-west2-b-1 topology.kubernetes.io/zone=europe-west2-b

$ kubectl label nodes scylla-ivan-k8s-2-europe-west2-c-1 topology.kubernetes.io/zone=europe-west2-c
```


The nodes get tagged with metadata (scylla.scylladb.com/node-type=scylla) on both nodes and then each worker node respectively with (topology.kubernetes.io/zone=europe-west2-b or topology.kubernetes.io/zone=europe-west2-c) so other Kubernetes objects can select on. Unlike a taint, a label doesn't block or restrict anything by itself — it's purely informational/selectable.

To confim labels:

```
$ kubectl get nodes -L topology.kubernetes.io/zone,scylla.scylladb.com/node-type

NAME                                 STATUS   ROLES           AGE   VERSION   ZONE             NODE-TYPE
scylla-ivan-k8s-2-europe-west2-a-1   Ready    control-plane   36d   v1.36.3                    
scylla-ivan-k8s-2-europe-west2-b-1   Ready    <none>          36d   v1.36.3   europe-west2-b   scylla
scylla-ivan-k8s-2-europe-west2-c-1   Ready    <none>          36d   v1.36.3   europe-west2-c   scylla

```
note: Just 2 nodes (worker nodes) shown labels .


In summary, for the two previous steps, label(s) is the companion piece to the taint. The taint repels pods that don't tolerate it, while this label lets you attract the right pods there deliberately, typically via a nodeSelector or nodeAffinity rule in the ScyllaCluster's pod spec, e.g. matching (scylla.scylladb.com/node-type: scylla)


So together:

Taint → keeps everything else off the node.
Label → lets you explicitly target the node for ScyllaDB pods.


**c) Dealing with CPU pinning in on-premise k8s enviroenment:**
https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#changing-the-cpu-manager-policy

before deploying the cluster:
go to worker nodes and stop kubelet
```
$ sudo systemctl stop kubelet
```
In the directory /var/lib/kubelet, open: config.yaml file and add the following:

```
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
cpuManagerPolicy: static
kubeReserved:
  cpu: "500m"
systemReserved:
  cpu: "500m"
cpuManagerReconcilePeriod: 0s
```

in the same directory remove file cpu_manager_state (this wil be regenarate once kubelet restart)
Then re-start 
```
$kubelet sudo systemctl start kubelet
```


**d) Install Operator**
[Install Operator with GitOps](https://operator.docs.scylladb.com/v1.21/install-operator/install-with-gitops.html)

fisrt install cert manager

```
$ kubectl apply --server-side -f=https://raw.githubusercontent.com/scylladb/scylla-operator/v1.21/examples/third-party/cert-manager.yaml
```

then the Operator Itself:

```
$ kubectl -n=scylla-operator apply --server-side -f=https://raw.githubusercontent.com/scylladb/scylla-operator/v1.21/deploy/operator.yaml
```


**e) Deploy nodeconfig for Google Cloud Instances, as using Google Cloud VMs**

```
$ kubectl apply --server-side -f=https://raw.githubusercontent.com/scylladb/scylla-operator/v1.21/examples/gke/nodeconfig-alpha.yaml
```
then

```
$ kubectl get nodeconfigs.scylla.scylladb.com
NAME                  AVAILABLE   PROGRESSING   DEGRADED   AGE
scylladb-nodepool-1   True        False         False      36d

```

**f) Deploy ScyllaDB's Local CSI Driver**

```
kubectl -n=local-csi-driver apply --server-side -f=https://raw.githubusercontent.com/scylladb/scylla-operator/v1.21/examples/common/local-volume-provisioner/local-csi-driver/{00_clusterrole_def,00_clusterrole_def_openshift,00_clusterrole,00_namespace,00_scylladb-local-xfs.storageclass,10_csidriver,10_serviceaccount,20_clusterrolebinding,50_daemonset}.yaml
```


**g) Create ScyllaDB configuration file**

```
kubectl apply --server-side -f=- <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: scylladb-config
data:
  scylla.yaml: |
    authenticator: PasswordAuthenticator
    authorizer: CassandraAuthorizer
EOF
```

Note: other parameters can be passed to modify default values.

**h) kubectl set context**

Check first what is in `cat $HOME/.kube/config` if nothing there, then:

```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
To review the k8s config file:

```
$ cat $HOME/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJRGp5d0xaRkpiWGd3RFFZSktvWklodmNOQVF...
    server: https://10.154.0.53:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJSUd2L0NQOUo1ZTR3RFFZSktvWklodmNOQVF...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFtLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJSUdtLS1CRUd...
```

To create contexts:
```

kubectl config set-context dc1 --cluster=kubernetes --user=kubernetes-admin
kubectl config set-context dc2 --cluster=kubernetes --user=kubernetes-admin
```

To check modified k8s config file


```
$ cat $HOME/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJRGp5d0xaRkpiWGd3RFFZSktvWklodmNOQVF...
    server: https://10.154.0.53:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: dc1
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: dc2
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJSUd2L0NQOUo1ZTR3RFFZSktvWklodmNOQVF...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFtLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJSUdtLS1CRUd...
```

Also create enviroenment variables to reference contexts
```
$ export CONTEXT_DC1=dc1
$ export CONTEXT_DC2=dc2
```


**i) Install MultiDC ScyllaDB cluster**

There are two ways:

newer: apiVersion: scylla.scylladb.com/v1

Note: Dedicated namespaces are required for each manifest file related to the DC.

Manifests are as follows:

for DC1 (dc1.yaml)

```
$ kubectl --context="${CONTEXT_DC1}" apply --server-side -f=dc1.yaml

```

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla                  # Must be different than dc2.yaml
spec:
  agentVersion: 3.10.1
  version: 2026.1.3
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  datacenter:
    name: us-east-1                  #May be different name than dc2.yaml
    racks:
    - name: a                        #May be different than dc2.yaml
```

Full dc1.yaml file

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  datacenter:
    name: europe-west2-b-dc1
    racks:
    - name: b
      members: 1
      storage:
        storageClassName: scylladb-local-xfs
        capacity: 100G
      agentResources:
        requests:
          cpu: 100m
          memory: 250M
        limits:
          cpu: 100m
          memory: 250M
      resources:
        requests:
          cpu: 2
          memory: 5G
        limits:
          cpu: 2
          memory: 5G
      placement:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: scylla
                scylla/cluster: scylla-cluster
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - europe-west2-b
              - key: scylla.scylladb.com/node-type
                operator: In
                values:
                - scylla
        tolerations:
        - effect: NoSchedule
          key: scylla-operator.scylladb.com/dedicated
          operator: Equal
          value: scyllaclusters
```

then:
```
$ kubectl get pods -n scylla
NAME                                                  READY   STATUS    RESTARTS      AGE
scylla-cluster-europe-west2-b-dc1-b-0                 4/4     Running   1 (39d ago)   39d
```

Nodetool Status (at this moment just one DC)
```
$ kubectl --context="${CONTEXT_DC1}" -n=scylla exec -it pod/scylla-cluster-europe-west2-b-dc1-b-0 -c=scylla -- nodetool status

Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load    Tokens Owns Host ID                              Rack
UN 192.168.49.198 1.04 MB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
```


for DC2 (dc2.yaml)

```
$ kubectl --context="${CONTEXT_DC2}" apply --server-side -f=dc2.yaml
```

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla2                 #Must be different than dc1.yaml
spec:
  agentVersion: 3.10.1
  version: 2026.1.3
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  datacenter:
    name: us-east-1                   #May be different than dc1.yaml
    racks:
    - name: a                         #May be different than dc1.yaml

```
Full dc2.yaml file

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla2
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  externalSeeds:
  - 192.168.49.198 
  datacenter:
    name: europe-west2-c-dc2
    racks:
    - name: c
      members: 1
      storage:
        storageClassName: scylladb-local-xfs
        capacity: 100G
      agentResources:
        requests:
          cpu: 100m
          memory: 250M
        limits:
          cpu: 100m
          memory: 250M
      resources:
        requests:
          cpu: 2
          memory: 5G
        limits:
          cpu: 2
          memory: 5G
      placement:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: scylla
                scylla/cluster: scylla-cluster
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - europe-west2-c
              - key: scylla.scylladb.com/node-type
                operator: In
                values:
                - scylla
        tolerations:
        - effect: NoSchedule
          key: scylla-operator.scylladb.com/dedicated
          operator: Equal
          value: scyllaclusters
```

then:

```
$ kubectl get pods -n scylla2
NAME                                    READY   STATUS    RESTARTS      AGE
scylla-cluster-europe-west2-c-dc2-c-0   4/4     Running   1 (12m ago)   15m

```


Nodetool Status (At this moment 2 DCs)

```
$ kubectl --context="${CONTEXT_DC2}" -n=scylla2 exec -it pod/scylla-cluster-europe-west2-c-dc2-c-0 -c=scylla -- nodetool status

Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load    Tokens Owns Host ID                              Rack
UN 192.168.49.198 1.04 MB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
Datacenter: europe-west2-c-dc2
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address         Load      Tokens Owns Host ID                              Rack
UN 192.168.147.139 544.50 KB 256    ?    f381543d-c737-4ce6-8aa4-94d6eae6ee3c c   

```

It is interesting to note that, when executing:

```
$ kubectl -n scylla2 edit scyllaclusters.scylla.scylladb.com scylla-cluster
```

This will show seed from the first DC created `europe-west2-b-dc1`:
```
externalSeeds:
  - 192.168.49.198

```

Also, `scylla` cluster deployment won't potentially show external seed if this Dc is a single node as in this case.

```
$ kubectl -n scylla edit scyllaclusters.scylla.scylladb.com scylla-cluster
```
In other words, node in `europe-west2-b-dc1` DC, won't show any external seed(s).


For completenes, we should keep nodes from both DC in the seed list, therefore, proceed to edit both ScyllaDB object deployments and include the seeds as required. Furthermore, instead of using plain IPs (IPs are ephemeral) and since it's one k8s cluster, cross-namespace DNS resolution (scylla.svc.cluster.local reachable from scylla2, and vice versa, scylla) works out of the box — no special networking needed there.

To get cross DNS resolution:

```
$ kubectl -n scylla get svc | grep dc1-b-0

scylla-cluster-europe-west2-b-dc1-b-0   ClusterIP   None           <none>        7000/TCP,7001/TCP,9042/TCP,9142/TCP,19042/TCP,19142/TCP,7199/TCP,10001/TCP,9180/TCP,5090/TCP,9100/TCP,9160/TCP   39d
````
```
$ kubectl -n scylla2 get svc | grep dc2-c-0

scylla-cluster-europe-west2-c-dc2-c-0   ClusterIP   None             <none>        7000/TCP,7001/TCP,9042/TCP,9142/TCP,19042/TCP,19142/TCP,7199/TCP,10001/TCP,9180/TCP,5090/TCP,9100/TCP,9160/TCP   16h

```

So when editing scylladb deployment, suing the following commands:
```
kubectl -n scylla edit scyllaclusters.scylla.scylladb.com scylla-cluster
and
kubectl -n scylla2 edit scyllaclusters.scylla.scylladb.com scylla-cluster
```
at the end both files with:
```
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  externalSeeds:
  - scylla-cluster-europe-west2-b-dc1-b-0.scylla-cluster-client.scylla.svc.cluster.local
  - scylla-cluster-europe-west2-c-dc2-c-0.scylla-cluster-
  repository: docker.io/scylladb/scylla
  version: 2026.2.2
```
It will take some time for things to reconcile. If stale entries persist after the address config is fixed and both nodes are confirmed reachable, you may need nodetool removenode (graceful, needs the down node's Host ID) or nodetool assassinate <stale-ip> (forceful, use only if removenode won't work) on the side holding the stale entry, then let it rejoin gossip naturally.

  
Additionally, the operator's exposeOptions configuration (nodeService:ClusterIP / clients:ServiceClusterIP / nodes:ServiceClusterIP) works fine when everything is inside one Kubernetes cluster, because ClusterIPs are only routable within that cluster.


Finally, execute `nodetool status` from any of the live nodes within a namespace. Note the IPs have changed.

```
$ kubectl --context="${CONTEXT_DC2}" -n=scylla2 exec -it pod/scylla-cluster-europe-west2-c-dc2-c-0 -c=scylla -- nodetool status

Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load    Tokens Owns Host ID                              Rack
UN 192.168.49.209 1.25 MB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
Datacenter: europe-west2-c-dc2
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address         Load      Tokens Owns Host ID                              Rack
UN 192.168.147.149 940.95 KB 256    ?    f381543d-c737-4ce6-8aa4-94d6eae6ee3c c   
```
