# Tidb V3



官方文档地址：<https://pingcap.com/docs-cn/v3.0/>

## TiDB in Kubernetes

### 安装本地持久卷提供者

参考：<https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner#version-compatibility>

在[下载页面](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/releases)下载 发布包

```console
wget https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/archive/v2.3.3.tar.gz
```

解压

```console
tar -zxvf v2.3.3.tar.gz
```

新建一个 StorageClass
To delay volume binding until pod scheduling and to handle multiple local PVs in a single pod, a StorageClass must to be created with `volumeBindingMode` set to `WaitForFirstConsumer`.

```console
cd sig-storage-local-static-provisioner-2.3.3
cp deployment/kubernetes/example/default_example_storageclass.yaml deployment/kubernetes/example/default_example_storageclass.yaml.orig
```

修改 StorageClass 的 yaml ，将 sc 的名字由 `fast-disks` 改为 `local-disk-storage` 。（此步可以跳过，我只是觉得 `local-disk-storage` 这个名字更好辨认）

```console
vi deployment/kubernetes/example/default_example_storageclass.yaml
```

部署 StorageClass

```console
kubectl create -f deployment/kubernetes/example/default_example_storageclass.yaml
```

查看新建的 StorageClass

```console
$ kubectl get sc
NAME                            PROVISIONER                    AGE
local-disk-storage              kubernetes.io/no-provisioner   6s
managed-nfs-storage (default)   nfs-storage                    12h
```

使用 helm 创建本地持久卷提供者

修改 helm 的 values.yaml

```console
cp helm/provisioner/values.yaml helm/provisioner/values.yaml.orig
```

修改后和原文对比如下

```console
# diff helm/provisioner/values.yaml helm/provisioner/values.yaml.orig
12c12
<   namespace: local-static-provisioner
---
>   namespace: default
16c16
<   createNamespace: true
---
>   createNamespace: false
53c53
< - name: local-disk-storage # Defines name of storage classe.
---
> - name: fast-disks # Defines name of storage classe.
56c56
<   hostDir: /home/data/local-disk-storage
---
>   hostDir: /mnt/fast-disks
```

修改了命名空间名，改为了将改 helm 的资源安装到 local-static-provisioner 命名空间，并且自动新建该命名空间，修改了hostDir，改为了 `/home/data/local-disk-storage`， 注意，因为在上面修改了 StorageClass 的名字，所以需要在 values.yaml 里也修改下。

使用 helm template 命令将该 chart 生成为一个清单文件。

```console
helm template ./helm/provisioner > deployment/kubernetes/provisioner_generated.yaml
```

部署提供者

```console
kubectl create -f deployment/kubernetes/provisioner_generated.yaml
```

测试

本地卷提供者启动成功后，它会检查并创建 本地卷 pv。比如，如果 /home/data/local-disk-storage 目录中包含一个子目录 /home/data/local-disk-storage/vol1 ，**且该子目录已经被挂载了**，那么静态本地卷提供者会自动创建对应的 pv 。

```console
mkdir /home/data/local-disk-storage/vol1
mount -t tmpfs /home/data/local-disk-storage/vol1/ /home/data/local-disk-storage/vol1/
```

上文中，使用的是该文件夹自己挂载自己，来模拟文件夹被其他磁盘卷挂载。

可以看到，本地卷提供者自动新建了 pv

```console
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                STORAGECLASS          REASON   AGE
local-pv-5b8d1757                          7997Mi     RWO            Delete           Available                                                        local-disk-storage             4m41s
$ kubectl describe pv local-pv-5b8d1757 
Name:              local-pv-5b8d1757
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: local-volume-provisioner-k8s-m1-7725028a-d16c-455f-93d5-400dccf19d8a
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-disk-storage
Status:            Available
Claim:
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          7997Mi
Node Affinity:
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [k8s-m1]
Message:
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /home/data/local-disk-storage/vol1
Events:    <none>
```

> <https://www.codercto.com/a/53701.html>
> <https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/faqs.md#why-i-need-to-bind-mount-normal-directories-to-create-pvs-for-them>
> Why I need to bind mount normal directories to create PVs for them
> This is because there is a race between mounting another filesystem volume on a normal directory and creating a PV for it. If you want to create PVs for normal directories which do not have a mount point, you need to bind mount them onto another directory under discovery directory, or themselves if they are already in. Mount point on directory explicitly express it is ready to have a PV created for it.
> 本地卷提供者自动创建 pv 需要两个条件：该目录是被绑定的，该目录是发现目录的第一级子目录。
> Provisioner only creates local PVs for mounted directories in the first level of discovery directory. This is because there is no way to detect whether the sub-directory is created by system admin for discovering as local PVs or by user applications.

创建持久卷声明（pvc），使用上面的持久卷（pv）

```console
$ cat test-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-disk-storage
```

注意，storageClassName 需要和持久卷（pv）的保持一致
可以看到 pvc 处于 pending 状态，

```console
$ kubectl get pvc
NAME                  STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS         AGE
example-local-claim   Pending                                      local-disk-storage   114s
$ kubectl describe pvc example-local-claim
...
  Type    Reason                Age                  From                         Message
  ----    ------                ----                 ----                         -------
  Normal  WaitForFirstConsumer  6s (x10 over 2m15s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

因为 StorageClass local-disk-storage 配置了 WaitForFirstConsumer ，需要有消费者消费后才会绑定。

删除 pvc

```console
kubectl delete pvc example-local-claim
```

pvc成功删除

删除 pv

```console
kubectl delete pv local-pv-5b8d1757
```

发现 本地持久卷又新建了

```console
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                STORAGECLASS          REASON   AGE
local-pv-5b8d1757                          7997Mi     RWO            Delete           Available                                                        local-disk-storage             13s
```

删除的话，需要先删除对应的vol1文件夹，因为vol1文件夹被绑定了，所以首先得解绑

```console
umount /home/data/local-disk-storage/vol1
rm -rf /home/data/local-disk-storage/vol1
kubectl delete pv local-pv-5b8d1757
```

再次查看，pv 没有了。

自动建立的 pv 的容量是怎么分配的呢？pv 的容量是挂载的磁盘的空间？

感觉这个 本地卷提供者 不是很好用。它让集群管理员不用手动建立 pv 了，免除了手动建立 local pv 的麻烦

手动新建本地持久卷（local persistent volume）

<https://kubernetes.io/docs/concepts/storage/volumes/#local>

A local volume represents a mounted local storage device such as a disk, partition or directory.  
本地卷 表示 一个 已挂载的本地存储设备 如 磁盘， 分区 或 目录。

Local volumes can only be used as a statically created PersistentVolume. Dynamic provisioning is not supported yet.  
本地卷只能作为静态创建的持久卷。尚不支持动态提供。

Compared to hostPath volumes, local volumes can be used in a durable and portable manner without manually scheduling Pods to nodes, as the system is aware of the volume’s node constraints by looking at the node affinity on the PersistentVolume.  
与 hostPath 卷 相比，本地卷可以以持久和可移植的方式使用，无需手动将 pod 调度到节点，因为系统通过查看 在 持久卷 上的节点亲和性来了解卷的节点约束。

However, local volumes are still subject to the availability of the underlying node and are not suitable for all applications. If a node becomes unhealthy, then the local volume will also become inaccessible, and a Pod using it will not be able to run. Applications using local volumes must be able to tolerate this reduced availability, as well as potential data loss, depending on the durability characteristics of the underlying disk.  
同时，本地卷仍取决于基础节点的可用性，并且不适合所有的应用。如果一个节点变得不健康，那么在其上的本地卷同时也会变得不可访问，并且 使用该本地卷的 pod 将无法运行。使用本地卷的应用程序必须能够容忍这种可用性降低，潜在数据丢失，取决于基础磁盘持久性的缺点。

The following is an example of PersistentVolume spec using a local volume and nodeAffinity:  
下面是一个使用本地卷和节点亲和性的例子

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  # volumeMode field requires BlockVolume Alpha feature gate to be enabled.
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

PersistentVolume `nodeAffinity` is required when using local volumes. It enables the Kubernetes scheduler to correctly schedule Pods using local volumes to the correct node.  
当使用本地卷的时候需要配置持久卷节点亲和性。它能够让 kubernetes scheduler 将使用本地卷的 pod 分配到正确的节点上。

PersistentVolume `volumeMode` can now be set to “Block” (instead of the default value “Filesystem”) to expose the local volume as a raw block device. The `volumeMode` field requires `BlockVolume` Alpha feature gate to be enabled.  
持久卷模式现在可以设置为 “Block”（默认的模式为“Filesystem”）来将本地卷作为 原生的块设备进行暴露。volumeMode 字段需要开启 BlockVolume Alpha 特征功能。

When using local volumes, it is recommended to create a StorageClass with `volumeBindingMode` set to `WaitForFirstConsumer`. See the [example](https://kubernetes.io/docs/concepts/storage/storage-classes/#local). Delaying volume binding ensures that the PersistentVolumeClaim binding decision will also be evaluated with any other node constraints the Pod may have, such as node resource requirements, node selectors, Pod affinity, and Pod anti-affinity.  
当使用本地卷时，建议创建一个带有 volumeBindingMode 设置为 WaitForFirstConsumer 的 StorageClass 。 见例子。 延迟卷绑定以确保持久卷声明（PersistentVolumeClaim）绑定决策同时也能够与其他节点上的 pod 可能有的约束一起评估，如节点资源要求，节点选择，pod 亲和性和 pod 反亲和性。

An external static provisioner can be run separately for improved management of the local volume lifecycle. Note that this provisioner does not support dynamic provisioning yet. For an example on how to run an external local provisioner, see the [local volume provisioner user guide](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner).  
一个额外的静态提供者可以单独运行来改善本地卷生命周期的管理。注意，此提供者目前不支持动态提供。有关如何运行一个额外的本地卷提供者的示例，见本地卷提供者用户指南。

Note: The local PersistentVolume requires manual cleanup and deletion by the user if the external static provisioner is not used to manage the volume lifecycle.  
注意：如果额外的静态提供者没有配置管理本地卷的声明周期，那么本地持久卷需要用户手动清理和删除

由于 测试环境没有磁盘可供挂载，而且如果使用本地卷提供者，不能自定义pv 的容量，所以，本文使用手动创建本地卷的方式为 tidb 提供持久卷。

删除本地卷提供者

```console
kubectl delete -f deployment/kubernetes/provisioner_generated.yaml
kubectl delete -f deployment/kubernetes/example/default_example_storageclass.yaml
```



安装 tidb operator

> 参考：https://github.com/pingcap/tidb-operator  
> https://pingcap.com/docs-cn/v3.0/tidb-in-kubernetes/deploy/tidb-operator

TiDB Operator 使用 CRD (Custom Resource Definition) 扩展 Kubernetes，所以要使用 TiDB Operator，必须先创建 TidbCluster 自定义资源类型。只需要在你的 Kubernetes 集群上创建一次即可：

```console
kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/crd.yaml && \
kubectl get crd tidbclusters.pingcap.com
```

创建 TidbCluster 自定义资源类型后，接下来在 Kubernetes 集群上安装 TiDB Operator。



查看当前的支持的版本

```console
[root@k8s-m1 tidb]# helm search -l tidb-operator
NAME                 	CHART VERSION 	APP VERSION	DESCRIPTION                            
pingcap/tidb-operator	v1.1.0-alpha.3	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.1.0-alpha.2	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.1.0-alpha.1	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.1.0-alpha.0	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.0.1        	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.0.0        	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.0.0-rc.1   	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.0.0-beta.3 	           	tidb-operator Helm chart for Kubernetes
pingcap/tidb-operator	v1.0.0-beta.2 	           	tidb-operator Helm chart for Kubernetes
```

获取你要安装的 tidb-operator chart 中的 values.yaml 文件：

```console
mkdir -p /home/k8s/tidb/tidb-operator && \
helm inspect values pingcap/tidb-operator --version=v1.0.1 > /home/k8s/tidb/tidb-operator/values-tidb-operator.yaml
```

配置 TiDB Operator

TiDB Operator 里面会用到 k8s.gcr.io/kube-scheduler 镜像，如果下载不了该镜像，可以修改 /home/k8s/tidb/tidb-operator/values-tidb-operator.yaml 文件中的 scheduler.kubeSchedulerImage 为 registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler。

修改 values.yaml 中的 `defaultStorageClassName` 为 `tidb-local-storage`

安装 TiDB Operator

```console
helm install pingcap/tidb-operator --name=tidb-operator --namespace=tidb-admin --version=v1.0.1 -f /home/k8s/tidb/tidb-operator/values-tidb-operator.yaml && \
kubectl get po -n tidb-admin -l app.kubernetes.io/name=tidb-operator
```

通过下面命令获取待安装的 tidb-cluster chart 的 values.yaml 配置文件：

```console
mkdir -p /home/k8s/tidb/tidb-cluster && \
helm inspect values pingcap/tidb-cluster --version=v1.0.1 > /home/k8s/tidb/tidb-cluster/values-tidb-cluster.yaml
```

release-name 将会作为 Kubernetes 相关资源（例如 Pod，Service 等）的前缀名，可以起一个方便记忆的名字，要求全局唯一，通过 helm ls -q 可以查看集群中已经有的 release-name。
chart-version 是 tidb-cluster chart 发布的版本，可以通过 helm search -l tidb-cluster 查看当前支持的版本。

通过查看 tidb-cluster 的 values.yaml 文件，发现需要很多 StorageClass，手动创建很麻烦。
对不同类型的存储使用了不同的支持本地卷的 StorageClass

具体如下：

- pd 使用的 StorageClass 为  tidb-pd-local-storage ，3个pod，对应 pv 的容量为 1Gi
- tikv 使用的 StorageClass 为  tidb-kv-local-storage ，3个pod，对应 pv 的容量为 10Gi
- monitor 使用的 StorageClass 为  tidb-monitor-local-storage ，对应 pv 的容量为 10Gi
- binlog.pump 使用的 StorageClass 为  tidb-pump-local-storage ，对应 pv 的容量为 20Gi
- binlog.drainer 使用的 StorageClass 为  tidb-drainer-local-storage ，对应 pv 的容量为 10Gi
- scheduledBackup 使用的 StorageClass 为  tidb-scheduledbackup-local-storage ，对应 pv 的容量为 100Gi

**修改了storageclass和镜像的tag，pd3.0.1的镜像有个bug,所以统一使用了当前最新的3.0.4版本。bug: https://github.com/pingcap/tidb-operator/issues/568**

新建对应的StorageClass，新建对应pv 时，容量建议比对应标准的容量大些

配置完后，部署 TiDB 集群：

```console
helm install pingcap/tidb-cluster --name=tidb-cluster --namespace=tidb-cluster --version=v1.0.1 -f /home/k8s/tidb/tidb-cluster/values-tidb-cluster.yaml
```

查看 tidb-cluster 暴露的服务

```console
[root@k8s-m1 tidb-cluster]# kubectl get svc -n tidb-cluster 
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
tidb-cluster-discovery          ClusterIP   10.98.10.70      <none>        10261/TCP                        83s
tidb-cluster-grafana            NodePort    10.96.109.41     <none>        3000:32416/TCP                   83s
tidb-cluster-monitor-reloader   NodePort    10.110.218.98    <none>        9089:32371/TCP                   83s
tidb-cluster-pd                 ClusterIP   10.103.39.52     <none>        2379/TCP                         82s
tidb-cluster-pd-peer            ClusterIP   None             <none>        2380/TCP                         82s
tidb-cluster-prometheus         NodePort    10.110.101.169   <none>        9090:31686/TCP                   83s
tidb-cluster-tidb               NodePort    10.108.230.95    <none>        4000:32409/TCP,10080:30948/TCP   83s
```

修改名为 tidb-cluster-tidb 的 Service 的 NodePort，改为 4000映射到节点的40000端口，方便记忆。

因为 kube-apiserver 中配置的默认的 NodePort 范围为 30000-32767，所以需要修改下，改为 30000-60000。

```
$ kubectl edit svc -n tidb-cluster tidb-cluster-tidb
service/tidb-cluster-tidb edited
$ kubectl get svc -n tidb-cluster 
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
tidb-cluster-discovery          ClusterIP   10.98.10.70      <none>        10261/TCP                        9m18s
tidb-cluster-grafana            NodePort    10.96.109.41     <none>        3000:32416/TCP                   9m18s
tidb-cluster-monitor-reloader   NodePort    10.110.218.98    <none>        9089:32371/TCP                   9m18s
tidb-cluster-pd                 ClusterIP   10.103.39.52     <none>        2379/TCP                         9m17s
tidb-cluster-pd-peer            ClusterIP   None             <none>        2380/TCP                         9m17s
tidb-cluster-prometheus         NodePort    10.110.101.169   <none>        9090:31686/TCP                   9m18s
tidb-cluster-tidb               NodePort    10.108.230.95    <none>        4000:40000/TCP,10080:30948/TCP   9m18s
```

集群外可以通过 40000 端口访问了。默认 root 的密码为空，使用如下命令修改密码

```console
SET PASSWORD FOR 'root'@'%' = 'oGJQmu3pwu'; FLUSH PRIVILEGES;
```

granfa监控页面可以通过端口 32416 访问。

将51saas的数据库从143:3306导入到tidb中，报错：
[ERR] 8004 - transaction too large, len:300001
参考：https://pingcap.com/docs-cn/v3.0/reference/error-codes/
8004 单个事务过大，原因及解决方法请参考这里
4.3.3 Transaction too large 是什么原因，怎么解决？
由于分布式事务要做两阶段提交，并且底层还需要做 Raft 复制，如果一个事务非常大，会使得提交过程非常慢，并且会卡住下面的 Raft 复制流程。为了避免系统出现被卡住的情况，我们对事务的大小做了限制：
单个事务包含的 SQL 语句不超过 5000 条（默认）
单条 KV entry 不超过 6MB
KV entry 的总条数不超过 30w
KV entry 的总大小不超过 100MB
在 Google 的 Cloud Spanner 上面，也有类似的限制。




再次安装 ， 使用 helm v3

配置 PingCAP 官方 chart 仓库

```console
helm repo add pingcap https://charts.pingcap.org/
```

检查是否配置成功

```console
helm search repo pingcap 
NAME                  	CHART VERSION	APP VERSION	DESCRIPTION                            
pingcap/tidb-backup   	v1.0.3       	           	A Helm chart for TiDB Backup or Restore
pingcap/tidb-cluster  	v1.0.3       	           	A Helm chart for TiDB Cluster          
pingcap/tidb-lightning	dev          	           	A Helm chart for TiDB Lightning        
pingcap/tidb-operator 	v1.0.3       	           	tidb-operator Helm chart for Kubernetes
```

安装 自定义资源

```console
wget https://github.com/pingcap/tidb-operator/archive/v1.0.3.tar.gz
tar zxf v1.0.3.tar.gz
kubectl apply -f tidb-operator-1.0.3/manifests/crd.yaml
```

检查 crd

```console
# kubectl get crd tidbclusters.pingcap.com
NAME                       CREATED AT
tidbclusters.pingcap.com   2019-10-24T07:31:31Z
```

```console
wget https://github.com/pingcap/tidb-operator/releases/download/v1.0.3/tidb-operator-chart-v1.0.3.tgz
tar zxf tidb-operator-chart-v1.0.3.tgz
```

修改 tidb-operator/values.yaml，修改内容如下

- timezone修改为 Asia/Shanghai
- defaultStorageClassName 修改为 tidb-local-storage
- scheduler.kubeSchedulerImageName 修改为 registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler

安装 tidb-operator chart

```console
helm install ./tidb-operator --name-template=tidb-operator --namespace=tidb-admin --version=v1.0.3 -f tidb-operator/values.yaml
```

检查 tide-operator 是否安装成功

```console
[root@k8s-m1 tidb]# helm ls -n tidb-admin
NAME          NAMESPACE   REVISION  UPDATED                                 STATUS    CHART                 APP VERSION
tidb-operator tidb-admin  1         2019-11-18 15:24:28.215966065 +0800 CST deployed  tidb-operator-v1.0.3  

[root@k8s-m1 tidb]# kubectl get pods --namespace tidb-admin -l app.kubernetes.io/instance=tidb-operator
NAME                                      READY   STATUS    RESTARTS   AGE
tidb-controller-manager-95cbd5d8f-5gf4s   1/1     Running   0          3m24s
tidb-scheduler-66c9ccb8b9-xwfmk           2/2     Running   0          3m24s
```

tidb-operator 运行成功后，开始部署 tidb-cluster

```console
wget https://github.com/pingcap/tidb-operator/releases/download/v1.0.3/tidb-cluster-chart-v1.0.3.tgz
tar zxf tidb-cluster-chart-v1.0.3.tgz
```

修改 tidb-cluster/values.yaml，修改内容如下：

- timezone 修改为 Asia/Shanghai
- pd 使用的 StorageClass 为  tidb-pd-local-storage ，3个pod，对应 pv 的容量为 1Gi
- tikv 使用的 StorageClass 为  tidb-kv-local-storage ，3个pod，对应 pv 的容量为 10Gi
- monitor 使用的 StorageClass 为  tidb-monitor-local-storage ，对应 pv 的容量为 10Gi
- binlog.pump 使用的 StorageClass 为  tidb-pump-local-storage ，对应 pv 的容量为 20Gi
- binlog.drainer 使用的 StorageClass 为  tidb-drainer-local-storage ，对应 pv 的容量为 10Gi
- scheduledBackup 使用的 StorageClass 为  tidb-scheduledbackup-local-storage ，对应 pv 的容量为 100Gi

安装 tidb-cluster chart

```console
helm install ./tidb-cluster --name-template=tidb-cluster --namespace=tidb-cluster --version=v1.0.3 -f tidb-cluster/values.yaml
```

检查 tidb-cluster 是否运行成功

```console
[root@k8s-m1 tidb]# kubectl get pods --namespace tidb-cluster -l app.kubernetes.io/instance=tidb-cluster -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
tidb-cluster-discovery-56b7b8479b-jgh6k   1/1     Running   0          12m     172.30.182.26   c48-1    <none>           <none>
tidb-cluster-monitor-7b7f584fc9-9z4k5     3/3     Running   0          12m     172.30.182.52   c48-1    <none>           <none>
tidb-cluster-pd-0                         1/1     Running   0          12m     172.30.81.171   k8s-m2   <none>           <none>
tidb-cluster-pd-1                         1/1     Running   0          12m     172.30.11.55    k8s-m3   <none>           <none>
tidb-cluster-pd-2                         1/1     Running   0          12m     172.30.42.142   k8s-m1   <none>           <none>
tidb-cluster-tidb-0                       2/2     Running   0          6m12s   172.30.182.44   c48-1    <none>           <none>
tidb-cluster-tidb-1                       2/2     Running   0          6m12s   172.30.42.171   k8s-m1   <none>           <none>
tidb-cluster-tikv-0                       1/1     Running   0          9m12s   172.30.42.145   k8s-m1   <none>           <none>
tidb-cluster-tikv-1                       1/1     Running   0          9m12s   172.30.11.3     k8s-m3   <none>           <none>
tidb-cluster-tikv-2                       1/1     Running   0          9m12s   172.30.81.168   k8s-m2   <none>           <none>
[root@k8s-m1 tidb]# kubectl get services --namespace tidb-cluster -l app.kubernetes.io/instance=tidb-cluster
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
tidb-cluster-discovery          ClusterIP   10.99.62.162     <none>        10261/TCP                        13m
tidb-cluster-grafana            NodePort    10.101.233.134   <none>        3000:39821/TCP                   13m
tidb-cluster-monitor-reloader   NodePort    10.104.235.74    <none>        9089:32135/TCP                   13m
tidb-cluster-pd                 ClusterIP   10.98.189.149    <none>        2379/TCP                         13m
tidb-cluster-pd-peer            ClusterIP   None             <none>        2380/TCP                         13m
tidb-cluster-prometheus         NodePort    10.106.206.24    <none>        9090:59683/TCP                   13m
tidb-cluster-tidb               NodePort    10.106.198.189   <none>        4000:58911/TCP,10080:52494/TCP   13m
tidb-cluster-tidb-peer          ClusterIP   None             <none>        10080/TCP                        6m29s
tidb-cluster-tikv-peer          ClusterIP   None             <none>        20160/TCP                        9m29s
```

修改名为 tidb-cluster-tidb 的 Service 的 NodePort，改为 4000映射到节点的40000端口，方便记忆。

```console
kubectl edit svc -n tidb-cluster tidb-cluster-tidb
```

默认 tidb 的用户名为 root ，密码为空，登录 tidb 后使用如下命令修改 root 用户密码。

```sql
SET PASSWORD FOR 'root'@'%' = 'DrDMnmsl8a'; FLUSH PRIVILEGES;
```

tidb cluster 监控页面，通过 svc/tidb-cluster-grafana 可知暴漏的端口为 39821 ，该 grafana 的用户名密码为 admin/admin 。

至此， tidb 就安装好了。

