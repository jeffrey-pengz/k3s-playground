# Install K3S Cluster

This guide will help you quickly launch a k3s cluster (3 control nodes and 6 worker nodes for our dev cluster) in HA mode with embedded etcd, Calico, MetalLB, and Longhorn.

## 1. Node Information
This is our dev stack node setup, just for reference

```shell
root@control01:~# cat /etc/hosts
127.0.0.1       localhost

192.168.99.5 control01 control01.local
192.168.99.6 control02 control02.local
192.168.99.7 control03 control03.local

192.168.99.8 worker01 worker01.local
192.168.99.9 worker02 worker02.local
192.168.99.10 worker03 worker03.local
192.168.99.11 worker04 worker04.local
192.168.99.12 worker05 worker05.local
192.168.99.13 worker06 worker06.local
```

## 2. Control Plane

### Control01 (Primary Control Node)

This is the primary node, one of 3 control nodes.

To get started, first launch a server node with the *cluster-init* flag to enable clustering and a token that will be used as a shared secret to join additional servers to the cluster. Some additional flags are also used:
+ *flannel-backend=none* disables default k3s flannel backend as calico cni will be used instead later. 
+ *disable-network-policy* disables k3s default network policy controller as calico network policy wil be used. 
+ *cluster-cidr* specifies network CIDR to use for pod IPs (default: "10.42.0.0/16", change it if default is already used in your network)
+ *service-cidr* specifies network CIDR to use for services IPs (default: "10.43.0.0/16"), change it if default is already used in your network)
+ *disable=traefik,servicelb* opts out k3s packaged traefik-v1 and servicelb components as traefik-v2 and metalLB will be deployed instead.

```shell
curl -sfL https://get.k3s.io | K3S_TOKEN="your_token" INSTALL_K3S_VERSION="v1.21.1+k3s1" sh -s - server --cluster-init --flannel-backend=none --disable-network-policy --cluster-cidr=10.42.0.0/16 --disable=traefik,servicelb,local-storage
```

By default, in k3s control nodes will be schedulable and thus your workloads can get launched on them. If you wish to have a dedicated control plane where no user workloads will run, you can use taints. The **node-taint** parameter will allow you to configure nodes with taints, for example **--node-taint CriticalAddonsOnly=true:NoExecute**.

More server configuration info can be found from https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/

### Control02 and Control03

After launching the first server, join the second and third servers to the cluster using the shared secret:

```shell
curl -sfL https://get.k3s.io | K3S_TOKEN="your_token" INSTALL_K3S_VERSION="v1.21.1+k3s1" sh -s - server --server https://192.168.99.5:6443 --flannel-backend=none --disable-network-policy --cluster-cidr=10.42.0.0/16 --disable=traefik,servicelb,local-storage
```

For the --server flag we are using the IP of our primary control01. This will create HA control plane for our cluster.

Control plane should be done:

```shell
root@control01:~# kubectl get node
NAME        STATUS   ROLES                       AGE   VERSION
control01   NotReady    control-plane,etcd,master   15d   v1.20.7+k3s1
control02   NotReady    control-plane,etcd,master   15d   v1.20.7+k3s1
control03   NotReady    control-plane,etcd,master   15d   v1.20.7+k3s1
```
Since the network (CNI) is not configured yet, none of these nodes are ready. As soon as we apply Calico specs to the cluster, the nodes will become ready.

## 3. Install Calico on Cluster

Official calico installation guid can be found from https://docs.projectcalico.org/getting-started/kubernetes/k3s/multi-node-install

We will start by downloading the Calico manifest file from https://docs.projectcalico.org/manifests/calico.yaml and modifying file. I have done it and stored the modified manifest file in *./infrastructure/calico/calico.yaml*. Changes i made to the files:
```yaml
...
# The default IPv4 pool to create on startup if none exists. Pod IPs will be
# chosen from this range. Changing this value after installation will have
# no effect. This should fall within `--cluster-cidr` in k3s installation.
    - name: CALICO_IPV4POOL_CIDR
      value: "10.42.0.0/16"
...
```

```yaml
...
# Enable IPIP
# "Always" by default, but as our nodes live under the same subnet, so we use 
# "CrossSubnet" to give high-performance direct routing inside the subnet, and 
# only enable encapsulated traffic to route across subnets.
    - name: CALICO_IPV4POOL_IPIP
      value: "CrossSubnet"

# Disable VXLAN on the default IP pool.
    - name: CALICO_IPV4POOL_VXLAN
      value: "Never"
...
```

Also add following value to **cni_network_config** section
```json
"container_settings": {
    "allow_ip_forwarding": true
}
```

Now we are ready to install Calico by using the modified manifest file:
```shell
kubectl apply -f ./infrastructure/calico/calico.yaml
```
In a few minutes, the cluster becomes ready.
```shell
root@control01:~# kubectl get node
NAME        STATUS   ROLES                       AGE   VERSION
control01   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control02   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control03   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
```

## 4. Workers or Agents

We need to join worker nodes now, in our case worker01 to 06.

On every worker node do:

```shell
curl -sfL https://get.k3s.io | K3S_URL="https://192.168.99.5:644" K3S_TOKEN="your_token" INSTALL_K3S_VERSION="v1.21.1+k3s1" sh -
```

At the end of this step, you should have the whole cluster ready to use:
```shell
root@control01:~# kubectl get node
NAME        STATUS   ROLES                       AGE   VERSION
control01   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control02   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control03   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
worker01    Ready    <none>                      15d   v1.20.7+k3s1
worker02    Ready    <none>                      15d   v1.20.7+k3s1
worker03    Ready    <none>                      15d   v1.20.7+k3s1
worker04    Ready    <none>                      15d   v1.20.7+k3s1
worker05    Ready    <none>                      15d   v1.20.7+k3s1
worker06    Ready    <none>                      15d   v1.20.7+k3s1
```

## 5. Setting role/labels (Optional)

We can tag our cluster nodes, to give them labels. For example, we can label worker nodes with role=worker:

```shell
kubectl label nodes worker01 worker02 worker03 worker04 worker05 worker06 kubernetes.io/role=worker
```

```shell
root@control01:~# kubectl get node
NAME        STATUS   ROLES                       AGE   VERSION
control01   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control02   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
control03   Ready    control-plane,etcd,master   15d   v1.20.7+k3s1
worker01    Ready    worker                      15d   v1.20.7+k3s1
worker02    Ready    worker                      15d   v1.20.7+k3s1
worker03    Ready    worker                      15d   v1.20.7+k3s1
worker04    Ready    worker                      15d   v1.20.7+k3s1
worker05    Ready    worker                      15d   v1.20.7+k3s1
worker06    Ready    worker                      15d   v1.20.7+k3s1
```

You can use also `kubectl get nodes --show-labels` to show all labels for nodes.

## 6. Setup KUBECONFIG
Lastly add following into /etc/environment ( this is so the HELM and other programs know where the kubernetes config is. )

On every control node:

```shell
echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/environment
```

If you want to access the cluster from outside with kubectl, you can copy `/etc/rancher/k3s/k3s.yaml` onto your machine located outside the cluster as `~/.kube/config`. Then replace "localhost" with the IP of the K3s server (192.168.99.5 in our case). `kubectl` can now manage your K3s cluster from your machine.

## 7. Install MetalLB

Please refer to https://metallb.universe.tf/installation/ for more details about MetalLB installation.

We use installation by manifest files for example.

To install MetalLB, apply the manifest:

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/metallb.yaml
```
This will deploy MetalLB to your cluster, under the `metallb-system`
namespace. The components in the manifest are:

- The `metallb-system/controller` deployment. This is the cluster-wide
  controller that handles IP address assignments.
- The `metallb-system/speaker` daemonset. This is the component that
  speaks the protocol(s) of your choice to make the services
  reachable.
- Service accounts for the controller and speaker, along with the
  RBAC permissions that the components need to function.

The installation manifest does not include a configuration
file. MetalLB's components will still start, but will remain idle
until we define and deploy a configmap. The following is the config we used for our dev cluster:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.99.210-192.168.99.250
```
This setting gives MetaLB control over IPs from 192.168.99.210 to 192.168.99.250 as external IP for services.

So apply the config:

```shell
kubectl apply -f ./infrastructure/metallb/layer2config.yaml
````
Check if everything deployed ok:

```shell
root@control01:~# kubectl get all -n metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-64f86798cc-lxw46   1/1     Running   0          4d
pod/speaker-22ltc                 1/1     Running   1          16d
pod/speaker-64h8m                 1/1     Running   0          16d
pod/speaker-9pwl4                 1/1     Running   0          16d
pod/speaker-c9kz4                 1/1     Running   0          16d
pod/speaker-c9z8q                 1/1     Running   0          16d
pod/speaker-fmzjb                 1/1     Running   0          16d
pod/speaker-h95cl                 1/1     Running   0          16d
pod/speaker-prmtg                 1/1     Running   1          16d
pod/speaker-wnvd4                 1/1     Running   0          16d

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   9         9         9       9            9           kubernetes.io/os=linux   16d

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           16d

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-64f86798cc   1         1         1       16d
```
You should have as many speaker-xxxx as you have nodes in cluster, since they run one per node.

Now services that use LoadBalancer should have external IP assigned to it.

For example:

```shell
root@control01:~# kubectl get svc -n argocd
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
argocd-dex-server       ClusterIP      10.43.135.6     <none>           5556/TCP,5557/TCP,5558/TCP   15d
argocd-metrics          ClusterIP      10.43.157.76    <none>           8082/TCP                     15d
argocd-redis            ClusterIP      10.43.247.59    <none>           6379/TCP                     15d
argocd-repo-server      ClusterIP      10.43.145.118   <none>           8081/TCP,8084/TCP            15d
argocd-server           LoadBalancer   10.43.112.167   192.168.99.210   80:31375/TCP,443:30359/TCP   15d
argocd-server-metrics   ClusterIP      10.43.97.121    <none>           8083/TCP                     15d
```

## 8. Install Longhorn
Before start installing Longhorn, some requirements have to be checked and met.
Please refer to https://longhorn.io/docs/1.1.1/deploy/install/#installation-requirements for details. But the most important one is to make sure `open-iscsi` is installed on each node.

For the installation, it is fairly easy - just simply follow the official guide at https://longhorn.io/docs/1.1.1/deploy/install/#installation-requirements

Give it some time, it should deploy, maybe some pods do restarts but in the end it should look something like this:

```shell
root@control01:~# kubectl get all -n longhorn-system
NAME                                            READY   STATUS    RESTARTS   AGE
pod/csi-attacher-5dcdcd5984-dbddz               1/1     Running   2          16d
pod/csi-attacher-5dcdcd5984-hx252               1/1     Running   2          16d
pod/csi-attacher-5dcdcd5984-qkjkd               1/1     Running   1          16d
pod/csi-provisioner-5c9dfb6446-8dclp            1/1     Running   0          4d
pod/csi-provisioner-5c9dfb6446-fcjct            1/1     Running   0          4d
pod/csi-provisioner-5c9dfb6446-nrdzs            1/1     Running   3          16d
pod/csi-resizer-6696d857b6-bc6lb                1/1     Running   0          4d
pod/csi-resizer-6696d857b6-xmg5c                1/1     Running   0          4d
pod/csi-resizer-6696d857b6-zflz9                1/1     Running   3          16d
pod/csi-snapshotter-96bfff7c9-dn5bc             1/1     Running   0          4d
pod/csi-snapshotter-96bfff7c9-rhxsj             1/1     Running   2          16d
pod/csi-snapshotter-96bfff7c9-vmsbw             1/1     Running   0          16d
pod/engine-image-ei-611d1496-4k7km              1/1     Running   1          16d
pod/engine-image-ei-611d1496-6zphj              1/1     Running   1          16d
pod/engine-image-ei-611d1496-c2t4b              1/1     Running   0          16d
pod/engine-image-ei-611d1496-cq87t              1/1     Running   0          16d
pod/engine-image-ei-611d1496-dcp6j              1/1     Running   0          16d
pod/engine-image-ei-611d1496-dz7nq              1/1     Running   0          16d
pod/engine-image-ei-611d1496-gjlsp              1/1     Running   0          16d
pod/engine-image-ei-611d1496-rcn6t              1/1     Running   0          16d
pod/engine-image-ei-611d1496-zg85d              1/1     Running   0          16d
pod/instance-manager-e-00e7c99b                 1/1     Running   0          16d
pod/instance-manager-e-1faf75ec                 1/1     Running   0          16d
pod/instance-manager-e-3cd301f0                 1/1     Running   0          16d
pod/instance-manager-e-3ed353d9                 1/1     Running   0          3d20h
pod/instance-manager-e-485e4cc3                 1/1     Running   0          30h
pod/instance-manager-e-6b7ecb15                 1/1     Running   0          16d
pod/instance-manager-e-70c8c512                 1/1     Running   0          9d
pod/instance-manager-e-79c5461d                 1/1     Running   0          3d20h
pod/instance-manager-e-bf525425                 1/1     Running   0          3d20h
pod/instance-manager-r-262518a8                 1/1     Running   0          3d20h
pod/instance-manager-r-38f5c7e2                 1/1     Running   0          16d
pod/instance-manager-r-84b8d0aa                 1/1     Running   0          16d
pod/instance-manager-r-916d96a4                 1/1     Running   0          3d20h
pod/instance-manager-r-a23fca7b                 1/1     Running   0          30h
pod/instance-manager-r-abe3575e                 1/1     Running   0          9d
pod/instance-manager-r-c3a2e019                 1/1     Running   0          16d
pod/instance-manager-r-df79c0da                 1/1     Running   0          3d20h
pod/instance-manager-r-fc17d2b1                 1/1     Running   0          16d
pod/longhorn-csi-plugin-22rqh                   2/2     Running   0          16d
pod/longhorn-csi-plugin-2xv9w                   2/2     Running   0          16d
pod/longhorn-csi-plugin-4mhxw                   2/2     Running   0          16d
pod/longhorn-csi-plugin-5xfd4                   2/2     Running   3          16d
pod/longhorn-csi-plugin-98zkl                   2/2     Running   0          16d
pod/longhorn-csi-plugin-b2fxc                   2/2     Running   0          16d
pod/longhorn-csi-plugin-kdbkn                   2/2     Running   0          16d
pod/longhorn-csi-plugin-s4tbp                   2/2     Running   3          16d
pod/longhorn-csi-plugin-wxghd                   2/2     Running   0          16d
pod/longhorn-driver-deployer-5d45dcdc5d-svbfq   1/1     Running   0          16d
pod/longhorn-manager-9n7vl                      1/1     Running   0          16d
pod/longhorn-manager-9nsrw                      1/1     Running   0          16d
pod/longhorn-manager-dsdct                      1/1     Running   0          16d
pod/longhorn-manager-hk47b                      1/1     Running   0          16d
pod/longhorn-manager-mqvh4                      1/1     Running   0          16d
pod/longhorn-manager-vcn8c                      1/1     Running   0          16d
pod/longhorn-manager-vnf6b                      1/1     Running   1          16d
pod/longhorn-manager-wjczm                      1/1     Running   1          16d
pod/longhorn-manager-zstmf                      1/1     Running   1          16d
pod/longhorn-ui-5879656c55-khjcd                1/1     Running   0          16d

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/csi-attacher        ClusterIP   10.43.164.3     <none>        12345/TCP   16d
service/csi-provisioner     ClusterIP   10.43.114.29    <none>        12345/TCP   16d
service/csi-resizer         ClusterIP   10.43.136.207   <none>        12345/TCP   16d
service/csi-snapshotter     ClusterIP   10.43.78.25     <none>        12345/TCP   16d
service/longhorn-backend    ClusterIP   10.43.230.94    <none>        9500/TCP    16d
service/longhorn-frontend   ClusterIP   10.43.26.180    <none>        80/TCP      16d

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/engine-image-ei-611d1496   9         9         9       9            9           <none>          16d
daemonset.apps/longhorn-csi-plugin        9         9         9       9            9           <none>          16d
daemonset.apps/longhorn-manager           9         9         9       9            9           <none>          16d

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-attacher               3/3     3            3           16d
deployment.apps/csi-provisioner            3/3     3            3           16d
deployment.apps/csi-resizer                3/3     3            3           16d
deployment.apps/csi-snapshotter            3/3     3            3           16d
deployment.apps/longhorn-driver-deployer   1/1     1            1           16d
deployment.apps/longhorn-ui                1/1     1            1           16d

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-attacher-5dcdcd5984               3         3         3       16d
replicaset.apps/csi-provisioner-5c9dfb6446            3         3         3       16d
replicaset.apps/csi-resizer-6696d857b6                3         3         3       16d
replicaset.apps/csi-snapshotter-96bfff7c9             3         3         3       16d
replicaset.apps/longhorn-driver-deployer-5d45dcdc5d   1         1         1       16d
replicaset.apps/longhorn-ui-5879656c55                1         1         1       16d
```
### Make Longhorn the default storageclass

As you can see both `local-path` (k3s default storage provider) and `longhorn` are marked as default:

```shell
root@control01:~# kubectl get storageclass
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default) rancher.io/local-path   Delete          WaitForFirstConsumer   false                  16d
longhorn (default)   driver.longhorn.io      Delete          Immediate              true                   16d
```

But we would like to have longhorn as THE default storageclass for any workloads. so we have to execute following command:

```shell
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Then longhorn is the only default storageclass

```shell
root@control01:~# kubectl get storageclass
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path           rancher.io/local-path   Delete          WaitForFirstConsumer   false                  16d
longhorn (default)   driver.longhorn.io      Delete          Immediate              true                   16d
```


## 8. Uninstall
If the cluster is broken and beyond repair, we have to clean up everything and start over again.

### 1. Uninstall Workers

Run the following on each worker node
```shell
/usr/local/bin/k3s-agent-uninstall.sh
```
### 2. Uninstall Controls
Run the following on each control node
```shell
/usr/local/bin/k3s-uninstall.sh
```
### 3. Storage Cleanup
Run the following on each longhorn storage node. The default storage location is `/var/lib/longhorn` on host.
```shell
rm -rf /var/lib/longhorn/*
```
