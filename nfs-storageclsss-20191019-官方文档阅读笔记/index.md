# Nfs-Storageclsss-20191019-官方文档阅读笔记


## StorageClass

> 参考：  
<https://kubernetes.io/docs/concepts/storage/storage-classes/>  
<https://github.com/kubernetes-incubator/external-storage>

### Introduction

A `StorageClass` provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service(服务质量) levels, or to backup policies(备份策略), or to arbitrary(任意的) policies determined by the cluster administrators. Kubernetes itself is unopinionated(不关注，没有意见) about what classes represent. This concept is sometimes called “profiles” in other storage systems.

### The StorageClass Resource

Each `StorageClass` contains the fields `provisioner(供应者)`, `parameters`, and `reclaimPolicy`, which are used when a `PersistentVolume` belonging to the class needs to be dynamically(动态地) provisioned.

The name of a `StorageClass` object is significant(重要), and is how users can request a particular class. Administrators set the name and other parameters of a class when first creating `StorageClass` objects, and the objects cannot be updated once they are created.

Administrators can specify a default `StorageClass` just for PVCs that don’t request any particular class to bind to: see the [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1) section for details.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

#### Provisioner

Storage classes have a provisioner that determines what volume plugin is used for provisioning PVs. This field must be specified.

Volume Plugin | Internal Provisioner | Config Example
---|---|---
AWSElasticBlockStore | ✓ | [AWS EBS](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs)
AzureFile | ✓ | [Azure File](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-file)
AzureDisk | ✓ | [Azure Disk](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-disk)
CephFS | - | -
Cinder | ✓ | [OpenStack Cinder](https://kubernetes.io/docs/concepts/storage/storage-classes/#openstack-cinder)
FC | - | -
FlexVolume | - | -
Flocker | ✓ | -
GCEPersistentDisk | ✓ | [GCE PD](https://kubernetes.io/docs/concepts/storage/storage-classes/#gce-pd)
Glusterfs | ✓ | [Glusterfs](https://kubernetes.io/docs/concepts/storage/storage-classes/#glusterfs)
iSCSI | - | -
Quobyte | ✓ | [Quobyte](https://kubernetes.io/docs/concepts/storage/storage-classes/#quobyte)
NFS | - | -
RBD | ✓ | [Ceph RBD](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd)
VsphereVolume | ✓ | [vSphere](https://kubernetes.io/docs/concepts/storage/storage-classes/#vsphere)
PortworxVolume | ✓ | [Portworx Volume](https://kubernetes.io/docs/concepts/storage/storage-classes/#portworx-volume)
ScaleIO | ✓ | [ScaleIO](https://kubernetes.io/docs/concepts/storage/storage-classes/#scaleio)
StorageOS | ✓ | [StorageOS](https://kubernetes.io/docs/concepts/storage/storage-classes/#storageos)
Local | - | [Local](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)

You are not restricted to specifying the “internal” provisioners listed here (whose names are prefixed with “kubernetes.io” and shipped alongside Kubernetes). You can also run and specify external provisioners, which are independent programs that follow a [specification](https://git.k8s.io/community/contributors/design-proposals/storage/volume-provisioning.md) defined by Kubernetes. Authors of external provisioners have full discretion over where their code lives, how the provisioner is shipped, how it needs to be run, what volume plugin it uses (including Flex), etc. The repository [kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner) houses a library for writing external provisioners that implements the bulk of the specification. Some external provisioners are listed under the repository [kubernetes-incubator/external-storage](https://github.com/kubernetes-incubator/external-storage).

For example, NFS doesn’t provide an internal provisioner, but an external provisioner can be used. There are also cases when 3rd party storage vendors provide their own external provisioner.

#### Reclaim Policy(回收策略)

Persistent Volumes that are dynamically created by a storage class will have the reclaim policy specified in the `reclaimPolicy` field of the class, which can be either `Delete` or `Retain`. If no `reclaimPolicy` is specified when a StorageClass object is created, it will default to `Delete`.

Persistent Volumes that are created manually and managed via a storage class will have whatever reclaim policy they were assigned at creation.

#### Allow Volume Expansion(允许卷扩容)

**FEATURE STATE**: Kubernetes v1.11 beta

Persistent Volumes can be configured to be expandable. This feature when set to `true`, allows the users to resize the volume by editing the corresponding PVC object.

The following types of volumes support volume expansion, when the underlying Storage Class has the field `allowVolumeExpansion` set to true.

Volume type | Required Kubernetes version
---|---
gcePersistentDisk | 1.11
awsElasticBlockStore | 1.11
Cinder | 1.11
glusterfs | 1.11
rbd | 1.11
Azure File | 1.11
Azure Disk | 1.11
Portworx | 1.11
FlexVolume | 1.13
CSI | 1.14 (alpha), 1.16 (beta)

> **Note**: You can only use the volume expansion feature to grow a Volume, not to shrink(收缩) it.

#### Mount Options

Persistent Volumes that are dynamically created by a storage class will have the mount options specified in the `mountOptions` field of the class.

If the volume plugin does not support mount options but mount options are specified, provisioning will fail. Mount options are not validated on either the class or PV, so mount of the PV will simply fail if one is invalid.

#### Volume Binding Mode

The `volumeBindingMode` field controls when [volume binding and dynamic provisioning](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#provisioning) should occur.

By default, the `Immediate` mode indicates that volume binding and dynamic provisioning occurs once the PersistentVolumeClaim is created. For storage backends that are topology-constrained(拓扑约束) and not globally accessible from all Nodes in the cluster, PersistentVolumes will be bound or provisioned without knowledge of the Pod’s scheduling requirements. This may result in unschedulable Pods.

A cluster administrator can address this issue by specifying the `WaitForFirstConsumer` mode which will delay the binding and provisioning of a PersistentVolume until a Pod using the PersistentVolumeClaim is created. PersistentVolumes will be selected or provisioned conforming to the topology that is specified by the Pod’s scheduling constraints. These include, but are not limited to, [resource requirements](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container), [node selectors](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector), [pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity), and [taints and tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration).

The following plugins support `WaitForFirstConsumer` with dynamic provisioning:

- [AWSElasticBlockStore](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs)
- [GCEPersistentDisk](https://kubernetes.io/docs/concepts/storage/storage-classes/#gce-pd)
- [AzureDisk](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-disk)

The following plugins support `WaitForFirstConsumer` with pre-created PersistentVolume binding:

- All of the above
- [Local](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)

**FEATURE STATE**: Kubernetes 1.14 beta

CSI volumes are also supported with dynamic provisioning and pre-created PVs, but you’ll need to look at the documentation for a specific CSI driver to see its supported topology keys and examples. The `CSINodeInfo` feature gate must be enabled.

#### Allowed Topologies

When a cluster operator specifies the `WaitForFirstConsumer` volume binding mode, it is no longer necessary to restrict provisioning to specific topologies in most situations. However, if still required, `allowedTopologies` can be specified.

This example demonstrates how to restrict the topology of provisioned volumes to specific zones and should be used as a replacement for the zone and zones parameters for the supported plugins.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

### Parameters

Storage classes have parameters that describe volumes belonging to the storage class. Different parameters may be accepted depending on the `provisioner`. For example, the value `io1`, for the parameter `type`, and the parameter `iopsPerGB` are specific to EBS. When a parameter is omitted, some default is used.

#### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

- **type**: `io1`, `gp2`, `sc1`, `st1`. See [AWS docs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html) for details. Default: `gp2`.
- **zone** (Deprecated): AWS zone. If neither `zone` nor `zones` is specified, volumes are generally round-robin-ed across all active zones where Kubernetes cluster has a node. `zone` and `zones` parameters must not be used at the same time.
- **zones** (Deprecated): A comma separated list of AWS zone(s). If neither `zone` nor `zones` is specified, volumes are generally round-robin-ed across all active zones where Kubernetes cluster has a node. `zone` and `zones` parameters must not be used at the same time.
- **iopsPerGB**: only for `io1` volumes. I/O operations per second per GiB. AWS volume plugin multiplies this with size of requested volume to compute IOPS of the volume and caps it at 20 000 IOPS (maximum supported by AWS, see [AWS docs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html). A string is expected here, i.e. `"10"`, not `10`.
- **fsType**: fsType that is supported by kubernetes. Default: "`ext4`".
- **encrypted**: denotes whether the EBS volume should be encrypted or not. Valid values are `"true"` or `"false"`. A string is expected here, i.e. `"true"`, not `true`.
- **kmsKeyId**: optional. The full Amazon Resource Name of the key to use when encrypting the volume. If none is supplied but `encrypted` is true, a key is generated by AWS. See AWS docs for valid ARN value.

> **Note**: `zone` and `zones` parameters are deprecated and replaced with [allowedTopologies](https://kubernetes.io/docs/concepts/storage/storage-classes/#allowed-topologies)

#### GCE PD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
```

- **type**: `pd-standard` or `pd-ssd`. Default: `pd-standard`
- **zone** (Deprecated): GCE zone. If neither `zone` nor `zones` is specified, volumes are generally round-robin-ed across all active zones where Kubernetes cluster has a node. `zone` and `zones` parameters must not be used at the same time.
- **zones** (Deprecated): A comma separated list of GCE zone(s). If neither `zone` nor `zones` is specified, volumes are generally round-robin-ed across all active zones where Kubernetes cluster has a node. `zone` and `zones` parameters must not be used at the same time.
- **replication-type**: `none` or `regional-pd`. Default: `none`.
  
  If `replication-type` is set to `none`, a regular (zonal) PD will be provisioned.

  If `replication-type` is set to `regional-pd`, a [Regional Persistent Disk](https://cloud.google.com/compute/docs/disks/#repds) will be provisioned. In this case, users must use `zones` instead of `zone` to specify the desired replication zones. If exactly two zones are specified, the Regional PD will be provisioned in those zones. If more than two zones are specified, Kubernetes will arbitrarily choose among the specified zones. If the `zones` parameter is omitted, Kubernetes will arbitrarily choose among zones managed by the cluster.

> **Note**: `zone` and `zones` parameters are deprecated and replaced with [allowedTopologies](https://kubernetes.io/docs/concepts/storage/storage-classes/#allowed-topologies)

#### Glusterfs

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://127.0.0.1:8081"
  clusterid: "630372ccdc720a92c681fb928f27b53f"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
```

- **resturl**: Gluster REST service/Heketi service url which provision gluster volumes on demand. The general format should be `IPaddress:Port` and this is a mandatory(强制性的) parameter for GlusterFS dynamic provisioner. If Heketi service is exposed as a routable service in openshift/kubernetes setup, this can have a format similar to `http://heketi-storage-project.cloudapps.mystorage.com` where the fqdn is a resolvable Heketi service url.
- **restauthenabled** : Gluster REST service authentication boolean that enables authentication to the REST server. If this value is `"true"`, `restuser` and `restuserkey` or `secretNamespace` + `secretName` have to be filled. This option is deprecated, authentication is enabled when any of `restuser`, `restuserkey`, `secretName` or `secretNamespace` is specified.
- **restuser** : Gluster REST service/Heketi user who has access to create volumes in the Gluster Trusted Pool.
- **restuserkey** : Gluster REST service/Heketi user’s password which will be used for authentication to the REST server. This parameter is deprecated in favor of `secretNamespace` + `secretName`.
- **secretNamespace**, **secretName** : Identification of Secret instance that contains user password to use when talking to Gluster REST service. These parameters are optional, empty password will be used when both `secretNamespace` and `secretName` are omitted. The provided secret must have type `"kubernetes.io/glusterfs"`, e.g. created in this way:

```shell
kubectl create secret generic heketi-secret \
  --type="kubernetes.io/glusterfs" --from-literal=key='opensesame' \
  --namespace=default
```

Example of a secret can be found in [glusterfs-provisioning-secret.yaml](https://github.com/kubernetes/examples/tree/master/staging/persistent-volume-provisioning/glusterfs/glusterfs-secret.yaml).

- **clusterid**: `630372ccdc720a92c681fb928f27b53f` is the ID of the cluster which will be used by Heketi when provisioning the volume. It can also be a list of clusterids, for example: `"8452344e2becec931ece4e33c4674e4e,42982310de6c63381718ccfa6d8cf397"`. This is an optional parameter.

- **gidMin**, **gidMax** : The minimum and maximum value of GID range for the storage class. A unique value (GID) in this range ( gidMin-gidMax ) will be used for dynamically provisioned volumes. These are optional values. If not specified, the volume will be provisioned with a value between 2000-2147483647 which are defaults for gidMin and gidMax respectively.

- **volumetype** : The volume type and its parameters can be configured with this optional value. If the volume type is not mentioned, it’s up to the provisioner to decide the volume type.
  For example:
  - **Replica volume**: `volumetype: replicate:3` where ‘3’ is replica count.
  - **Disperse/EC volume**: `volumetype: disperse:4:2` where ‘4’ is data and ‘2’ is the redundancy count.
  - **Distribute volume**: volumetype: none
  
  For available volume types and administration options, refer to the [Administration](https://access.redhat.com/documentation/en-US/Red_Hat_Storage/3.1/html/Administration_Guide/part-Overview.html) Guide.

For further reference information, see [How to configure Heketi](https://github.com/heketi/heketi/wiki/Setting-up-the-topology).

When persistent volumes are dynamically provisioned, the Gluster plugin automatically creates an endpoint and a headless service in the name `gluster-dynamic-<claimname>`. The dynamic endpoint and service are automatically deleted when the persistent volume claim is deleted.

#### OpenStack Cinder

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  availability: nova
```

- **availability**: Availability Zone. If not specified, volumes are generally round-robin-ed across all active zones where Kubernetes cluster has a node.

> **Note**:  
**FEATURE STATE**: Kubernetes 1.11 deprecated  
This internal provisioner of OpenStack is deprecated. Please use [the external cloud provider for OpenStack](https://github.com/kubernetes/cloud-provider-openstack).

#### vSphere

1. Create a StorageClass with a user specified disk format.

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: fast
    provisioner: kubernetes.io/vsphere-volume
    parameters:
      diskformat: zeroedthick
    ```

    **diskformat**: `thin`, `zeroedthick` and `eagerzeroedthick`. Default: "`thin`".

2. Create a StorageClass with a disk format on a user specified datastore.

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
    name: fast
    provisioner: kubernetes.io/vsphere-volume
    parameters:
        diskformat: zeroedthick
        datastore: VSANDatastore
    ```

    **datastore**: The user can also specify the datastore in the StorageClass. The volume will be created on the datastore specified in the storage class, which in this case is `VSANDatastore`. This field is optional. If the datastore is not specified, then the volume will be created on the datastore specified in the vSphere config file used to initialize the vSphere Cloud Provider.

3. Storage Policy Management inside kubernetes

   - Using existing vCenter SPBM policy

     One of the most important features of vSphere for Storage Management is policy based Management. Storage Policy Based Management (SPBM) is a storage policy framework that provides a single unified control plane across a broad range of data services and storage solutions. SPBM enables vSphere administrators to overcome upfront storage provisioning challenges, such as capacity planning, differentiated service levels and managing capacity headroom.

     The SPBM policies can be specified in the StorageClass using the `storagePolicyName` parameter.

   - Virtual SAN policy support inside Kubernetes

     Vsphere Infrastructure (VI) Admins will have the ability to specify custom Virtual SAN Storage Capabilities during dynamic volume provisioning. You can now define storage requirements, such as performance and availability, in the form of storage capabilities during dynamic volume provisioning. The storage capability requirements are converted into a Virtual SAN policy which are then pushed down to the Virtual SAN layer when a persistent volume (virtual disk) is being created. The virtual disk is distributed across the Virtual SAN datastore to meet the requirements.

     You can see [Storage Policy Based Management for dynamic provisioning of volumes](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/policy-based-mgmt.html) for more details on how to use storage policies for persistent volumes management.

There are few [vSphere examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere) which you try out for persistent volume management inside Kubernetes for vSphere.

#### Ceph RBD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.16.153.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

- **monitors**: Ceph monitors, comma delimited. This parameter is required.
- **adminId**: Ceph client ID that is capable of creating images in the pool. Default is “admin”.
- **adminSecretName**: Secret Name for `adminId`. This parameter is required. The provided secret must have type “kubernetes.io/rbd”.
- **adminSecretNamespace**: The namespace for `adminSecretName`. Default is “default”.
- **pool**: Ceph RBD pool. Default is “rbd”.
- **userId**: Ceph client ID that is used to map the RBD image. Default is the same as `adminId`.
- **userSecretName**: The name of Ceph Secret for `userId` to map RBD image. It must exist in the same namespace as PVCs. This parameter is required. The provided secret must have type “kubernetes.io/rbd”, e.g. created in this way:

    ```shell
    kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
    --from-literal=key='QVFEQ1pMdFhPUnQrSmhBQUFYaERWNHJsZ3BsMmNjcDR6RFZST0E9PQ==' \
    --namespace=kube-system
    ```

- **userSecretNamespace**: The namespace for `userSecretName`.
- **fsType**: fsType that is supported by kubernetes. Default: `"ext4"`.
- **imageFormat**: Ceph RBD image format, “1” or “2”. Default is “2”.
- **imageFeatures**: This parameter is optional and should only be used if you set `imageFormat` to “2”. Currently supported features are `layering` only. Default is “”, and no features are turned on.

#### Quobyte

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: slow
provisioner: kubernetes.io/quobyte
parameters:
    quobyteAPIServer: "http://138.68.74.142:7860"
    registry: "138.68.74.142:7861"
    adminSecretName: "quobyte-admin-secret"
    adminSecretNamespace: "kube-system"
    user: "root"
    group: "root"
    quobyteConfig: "BASE"
    quobyteTenant: "DEFAULT"
```

- **quobyteAPIServer**: API Server of Quobyte in the format `"http(s)://api-server:7860"`
- **registry**: Quobyte registry to use to mount the volume. You can specify the registry as `<host>:<port>` pair or if you want to specify multiple registries you just have to put a comma between them e.q. `<host1>:<port>,<host2>:<port>,<host3>:<port>`. The host can be an IP address or if you have a working DNS you can also provide the DNS names.
- **adminSecretNamespace**: The namespace for **adminSecretName**. Default is “default”.
- **adminSecretName**: secret that holds information about the Quobyte user and the password to authenticate against the API server. The provided secret must have type “kubernetes.io/quobyte” and the keys `user` and `password`, e.g. created in this way:

    ```shell
    kubectl create secret generic quobyte-admin-secret \
    --type="kubernetes.io/quobyte" --from-literal=user='admin' --from-literal=password='opensesame' \
    --namespace=kube-system
    ```

- **user**: maps all access to this user. Default is “root”.

- **group**: maps all access to this group. Default is “nfsnobody”.

- **quobyteConfig**: use the specified configuration to create the volume. You can create a new configuration or modify an existing one with the Web console or the quobyte CLI. Default is “BASE”.

- **quobyteTenant**: use the specified tenant ID to create/delete the volume. This Quobyte tenant has to be already present in Quobyte. Default is “DEFAULT”.

#### Azure Disk

***Azure Unmanaged Disk Storage Class***

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- **skuName**: Azure storage account Sku tier. Default is empty.
- **location**: Azure storage account location. Default is empty.
- **storageAccount**: Azure storage account name. If a storage account is provided, it must reside in the same resource group as the cluster, and location is ignored. If a storage account is not provided, a new storage account will be created in the same resource group as the cluster.

***New Azure Disk Storage Class (starting from v1.7.2)***

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: Shared
```

- **storageaccounttype**: Azure storage account Sku tier. Default is empty.
- **kind**: Possible values are `shared` (default), `dedicated`, and `managed`. When `kind` is `shared`, all unmanaged disks are created in a few shared storage accounts in the same resource group as the cluster. When `kind` is `dedicated`, a new dedicated storage account will be created for the new unmanaged disk in the same resource group as the cluster. When `kind` is `managed`, all managed disks are created in the same resource group as the cluster.
- **resourceGroup**: Specify the resource group in which the Azure disk will be created. It must be an existing resource group name. If it is unspecified, the disk will be placed in the same resource group as the current Kubernetes cluster.

- Premium VM can attach both Standard_LRS and Premium_LRS disks, while Standard VM can only attach Standard_LRS disks.

- Managed VM can only attach managed disks and unmanaged VM can only attach unmanaged disks.

#### Azure File

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- **skuName**: Azure storage account Sku tier. Default is empty.
- **location**: Azure storage account location. Default is empty.
- **storageAccount**: Azure storage account name. Default is empty. If a storage account is not provided, all storage accounts associated with the resource group are searched to find one that matches skuName and location. If a storage account is provided, it must reside in the same resource group as the cluster, and skuName and location are ignored.
- **secretNamespace**: the namespace of the secret that contains the Azure Storage Account Name and Key. Default is the same as the Pod.
- **secretName**: the name of the secret that contains the Azure Storage Account Name and Key. Default is `azure-storage-account-<accountName>-secret`
- **readOnly**: a flag indicating whether the storage will be mounted as read only. Defaults to false which means a read/write mount. This setting will impact the ReadOnly setting in VolumeMounts as well.

During storage provisioning, a secret named by `secretName` is created for the mounting credentials. If the cluster has enabled both [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [Controller Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles), add the create permission of resource `secret` for clusterrole `system:controller:persistent-volume-binder`.

In a multi-tenancy context, it is strongly recommended to set the value for `secretNamespace` explicitly, otherwise the storage account credentials may be read by other users.

#### Portworx Volume

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-io-priority-high
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  snap_interval:   "70"
  io_priority:  "high"
```

- **fs**: filesystem to be laid out: none/xfs/ext4 (default: ext4).
- **block_size**: block size in Kbytes (default: 32).
- **repl**: number of synchronous replicas to be provided in the form of replication factor `1..3` (default: 1) A string is expected here i.e. `"1"` and not `1`.
- **io_priority**: determines whether the volume will be created from higher performance or a lower priority storage `high/medium/low` (default: `low`).
- **snap_interval**: clock/time interval in minutes for when to trigger snapshots. Snapshots are incremental based on difference with the prior snapshot, 0 disables snaps (default: 0). A string is expected here i.e. "70" and not 70.
- **aggregation_level**: specifies the number of chunks the volume would be distributed into, 0 indicates a non-aggregated volume (default: 0). A string is expected here i.e. "0" and not 0
- **ephemeral**: specifies whether the volume should be cleaned-up after unmount or should be persistent. emptyDir use case can set this value to true and persistent volumes use case such as for databases like Cassandra should set to false, true/false (default false). A string is expected here i.e. "true" and not true.

#### ScaleIO

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/scaleio
parameters:
  gateway: https://192.168.99.200:443/api
  system: scaleio
  protectionDomain: pd0
  storagePool: sp1
  storageMode: ThinProvisioned
  secretRef: sio-secret
  readOnly: false
  fsType: xfs
```

- **provisioner**: attribute is set to `kubernetes.io/scaleio`
- **gateway**: address to a ScaleIO API gateway (required)
- **system**: the name of the ScaleIO system (required)
- **protectionDomain**: the name of the ScaleIO protection domain (required)
- **storagePool**: the name of the volume storage pool (required)
- **storageMode**: the storage provision mode: ThinProvisioned (default) or ThickProvisioned
- **secretRef**: reference to a configured Secret object (required)
- **readOnly**: specifies the access mode to the mounted volume (default false)
- **fsType**: the file system to use for the volume (default ext4)

The ScaleIO Kubernetes volume plugin requires a configured Secret object. The secret must be created with type `kubernetes.io/scaleio` and use the same namespace value as that of the PVC where it is referenced as shown in the following command:

```shell
kubectl create secret generic sio-secret --type="kubernetes.io/scaleio" \
--from-literal=username=sioadmin --from-literal=password=d2NABDNjMA== \
--namespace=default
```

#### StorageOS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/storageos
parameters:
  pool: default
  description: Kubernetes volume
  fsType: ext4
  adminSecretNamespace: default
  adminSecretName: storageos-secret
```

- **pool**: The name of the StorageOS distributed capacity pool to provision the volume from. Uses the `default` pool which is normally present if not specified.
- **description**: The description to assign to volumes that were created dynamically. All volume descriptions will be the same for the storage class, but different storage classes can be used to allow descriptions for different use cases. Defaults to `Kubernetes volume`.
- **fsType**: The default filesystem type to request. Note that user-defined rules within StorageOS may override this value. Defaults to ext4.
- **adminSecretNamespace**: The namespace where the API configuration secret is located. Required if adminSecretName set.
- **adminSecretName**: The name of the secret to use for obtaining the StorageOS API credentials. If not specified, default values will be attempted.

The StorageOS Kubernetes volume plugin can use a Secret object to specify an endpoint and credentials to access the StorageOS API. This is only required when the defaults have been changed. The secret must be created with type `kubernetes.io/storageos` as shown in the following command:

```shell
kubectl create secret generic storageos-secret \
--type="kubernetes.io/storageos" \
--from-literal=apiAddress=tcp://localhost:5705 \
--from-literal=apiUsername=storageos \
--from-literal=apiPassword=storageos \
--namespace=default
```

Secrets used for dynamically provisioned volumes may be created in any namespace and referenced with the `adminSecretNamespace` parameter. Secrets used by pre-provisioned volumes must be created in the same namespace as the PVC that references it.

#### Local

**FEATURE STATE**: Kubernetes v1.14 stable

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Local volumes do not currently support dynamic provisioning, however a StorageClass should still be created to delay volume binding until pod scheduling. This is specified by the `WaitForFirstConsumer` volume binding mode.

Delaying volume binding allows the scheduler to consider all of a pod’s scheduling constraints when choosing an appropriate PersistentVolume for a PersistentVolumeClaim.

## nfs-storageclass

### 部署 NFS server

略

### 部署 NFS client

> 参考：<https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client>

**nfs-client** is an automatic provisioner that use your *existing and already configured* NFS server to support dynamic provisioning of Kubernetes Persistent Volumes via Persistent Volume Claims. Persistent volumes are provisioned as ``${namespace}-${pvcName}-${pvName}``.

To note again, you must *already* have an NFS Server.

#### With Helm

Follow the instructions for the stable helm chart maintained at <https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner>

The tl;dr is

```bash
$ helm install stable/nfs-client-provisioner --set nfs.server=x.x.x.x --set nfs.path=/exported/path
```

#### Without Helm

**Step 1: Get connection information for your NFS server**. Make sure your NFS server is accessible from your Kubernetes cluster and get the information you need to connect to it. At a minimum you will need its hostname.

**Step 2: Get the NFS-Client Provisioner files**. To setup the provisioner you will download a set of YAML files, edit them to add your NFS server's connection information and then apply each with the ``kubectl`` / ``oc`` command.

Get all of the files in the [deploy](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client/deploy) directory of this repository. These instructions assume that you have cloned the [external-storage](https://github.com/kubernetes-incubator/external-storage) repository and have a bash-shell open in the ``nfs-client`` directory.

**Step 3: Setup authorization**. If your cluster has RBAC enabled or you are running OpenShift you must authorize the provisioner. If you are in a namespace/project other than "default" edit `deploy/rbac.yaml`.

**Kubernetes:**

```sh
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
$ kubectl create -f deploy/rbac.yaml
```

**OpenShift:**

On some installations of OpenShift the default admin user does not have cluster-admin permissions. If these commands fail refer to the OpenShift documentation for **User and Role Management** or contact your OpenShift provider to help you grant the right permissions to your admin user.

```sh
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NAMESPACE=`oc project -q`
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
$ oc create -f deploy/rbac.yaml
$ oadm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner
```

**Step 4: Configure the NFS-Client provisioner**

Note: To deploy to an ARM-based environment, use: `deploy/deployment-arm.yaml` instead, otherwise use `deploy/deployment.yaml`.

Next you must edit the provisioner's deployment file to add connection information for your NFS server. Edit `deploy/deployment.yaml` and replace the two occurences of `<YOUR NFS SERVER HOSTNAME>` with your server's hostname.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: <YOUR NFS SERVER HOSTNAME>
            - name: NFS_PATH
              value: /var/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: <YOUR NFS SERVER HOSTNAME>
            path: /var/nfs
```

You may also want to change the PROVISIONER_NAME above from ``fuseim.pri/ifs`` to something more descriptive like ``nfs-storage``, but if you do remember to also change the PROVISIONER_NAME in the storage class definition below:

This is `deploy/class.yaml` which defines the NFS-Client's Kubernetes Storage Class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
```

```console
kubectl create -f deploy/deployment.yaml -f deploy/class.yaml
```

**Step 5: Finally, test your environment!**

Now we'll test your NFS provisioner.

Deploy:

```sh
$ kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
```

Now check your NFS Server for the file `SUCCESS`.

```sh
kubectl delete -f deploy/test-pod.yaml -f deploy/test-claim.yaml
```

Now check the folder has been deleted.

**Step 6: Deploying your own PersistentVolumeClaims**. To deploy your own PVC, make sure that you have the correct `storage-class` as indicated by your `deploy/class.yaml` file.

For example:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

**Step 7: Change the default StorageClass**

> 参考：<https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/>

1. List the StorageClasses in your cluster:

    ```shell
    kubectl get storageclass
    ```

    The output is similar to this:

    ```shell
    NAME                 PROVISIONER               AGE
    standard (default)   kubernetes.io/gce-pd      1d
    gold                 kubernetes.io/gce-pd      1d
    ```

    The default StorageClass is marked by (default).

2. Mark the default StorageClass as non-default:

    The default StorageClass has an annotation `storageclass.kubernetes.io/is-default-class` set to `true`. Any other value or absence of the annotation is interpreted as `false`.

    To mark a StorageClass as non-default, you need to change its value to `false`:

    ```shell
    kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    ```

    where `<your-class-name>` is the name of your chosen StorageClass.

3. Mark a StorageClass as default:

    Similarly to the previous step, you need to add/set the annotation `storageclass.kubernetes.io/is-default-class=true`.

    ```shell
    kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

    Please note that at most one StorageClass can be marked as default. If two or more of them are marked as default, a `PersistentVolumeClaim` without `storageClassName` explicitly specified cannot be created.

4. Verify that your chosen StorageClass is default:

    ```shell
    kubectl get storageclass
    ```

    The output is similar to this:

    ```shell
    NAME             PROVISIONER               AGE
    standard         kubernetes.io/gce-pd      1d
    gold (default)   kubernetes.io/gce-pd      1d
    ```

