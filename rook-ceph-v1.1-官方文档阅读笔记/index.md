# Rook-Ceph-v1.1-官方文档阅读笔记


> 参考：<https://rook.io/docs/rook/v1.1/>

## Rook

Rook is an open source `cloud-native storage orchestrator`, providing the platform, framework, and support for a diverse set of storage solutions to natively integrate with cloud-native environments.

Rook turns storage software into **self-managing**, **self-scaling**, and **self-healing** storage services. It does this by automating deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migration, disaster recovery, monitoring, and resource management. Rook uses the facilities provided by the underlying cloud-native container management, scheduling and orchestration platform to perform its duties.

Rook integrates deeply into cloud native environments leveraging extension points and providing a seamless experience for scheduling, lifecycle management, resource management, security, monitoring, and user experience.

For more details about the status of storage solutions currently supported by Rook, please refer to [the project status](https://github.com/rook/rook/blob/master/README.md#project-status) section of the Rook repository. We plan to continue adding support for other storage systems and environments based on community demand and engagement in future releases.

## Quick Start Guides

Starting Rook in your cluster is as simple as two `kubectl` commands. See our [Quickstart](https://rook.io/docs/rook/v1.1/quickstart-toc.html) guide for the details on what you need to get going.

Storage Provider | Status | Description
---|---|---
[Ceph](https://rook.io/docs/rook/v1.1/ceph-quickstart.html) | V1 | Ceph is a highly scalable distributed storage solution for block storage, object storage, and shared file systems with years of production deployments.
[EdgeFS](https://rook.io/docs/rook/v1.1/edgefs-quickstart.html) | V1 | EdgeFS is high-performance and low-latency object storage system with Geo-Transparent data access via standard protocols (S3, NFS, iSCSI) from on-prem, private/public clouds or small footprint edge (IoT) devices.
[Cassandra](https://rook.io/docs/rook/v1.1/cassandra.html) | Alpha | Cassandra is a highly available NoSQL database featuring lightning fast performance, tunable consistency and massive scalability.
[CockroachDB](https://rook.io/docs/rook/v1.1/cockroachdb.html) | Alpha | CockroachDB is a cloud-native SQL database for building global, scalable cloud services that survive disasters.
[Minio](https://rook.io/docs/rook/v1.1/minio-object-store.html) | Alpha | Minio is a high performance distributed object storage server, designed for large-scale private cloud infrastructure.
[NFS](https://rook.io/docs/rook/v1.1/nfs.html) | Alpha | NFS allows remote hosts to mount file systems over a network and interact with those file systems as though they are mounted locally.
[YugabyteDB](https://rook.io/docs/rook/v1.1/yugabytedb.html) | Alpha | YugaByteDB is a high-performance, cloud-native distributed SQL database which can tolerate disk, node, zone and region failures automatically.

### Ceph

This guide will walk you through the basic setup of a Ceph cluster and enable you to consume `block`, `object`, and `file storage` from other pods running in your cluster.

#### Prerequisites

Rook can be installed on any existing Kubernetes clusters as long as it meets the minimum version and have the required privilege to run in the cluster (see below for more information). If you dont have a Kubernetes cluster, you can quickly set one up using [Minikube](#minikube), [Kubeadm](#kubeadm) or [CoreOS/Vagrant](#new-local-kubernetes-cluster-with-vagrant).

If you are using `dataDirHostPath` to persist rook data on kubernetes hosts, make sure your host has at least 5GB of space available on the specified path.

##### Minimum Version

Kubernetes **v1.10** or higher is supported by Rook.

##### Privileges and RBAC

Rook requires privileges to manage the storage in your cluster. See the details [here](https://rook.io/docs/rook/v1.1/psp.html) or below for setting up Rook in a Kubernetes cluster with Pod Security Policies enabled.

###### Using Rook with Pod Security Policies

***Cluster Role***

**NOTE** Cluster role configuration is only needed when you are not already `cluster-admin` in your Kubernetes cluster!

Creating the Rook operator requires privileges for setting up RBAC. To launch the operator you need to have created your user certificate that is bound to ClusterRole `cluster-admin`.

One simple way to achieve it is to assign your certificate with the `system:masters` group:

```shell
-subj "/CN=admin/O=system:masters"
```

`system:masters` is a special group that is bound to `cluster-admin` ClusterRole, but it can't be easily revoked so be careful with taking that route in a production setting.

Binding individual certificate to ClusterRole `cluster-admin` is revocable by deleting the ClusterRoleBinding.

***RBAC for PodSecurityPolicies***

If you have activated the [PodSecurityPolicy Admission Controller](https://kubernetes.io/docs/admin/admission-controllers/#podsecuritypolicy) and thus are using [PodSecurityPolicies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/), you will require additional `(Cluster)RoleBindings` for the different `ServiceAccounts` Rook uses to start the Rook Storage Pods.

Security policies will differ for different backends. See Ceph's Pod Security Policies set up in its
[common.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/common.yaml) for an example of how this is done in practice.

**Note**: You do not have to perform these steps if you do not have the `PodSecurityPolicy` Admission Controller activated!

**PodSecurityPolicy**

You need at least one `PodSecurityPolicy` that allows privileged `Pod` execution. Here is an example
which should be more permissive than is needed for any backend:

这个 PodSecurityPolicy 在 [common.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/common.yaml) 也有。

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
spec:
  fsGroup:
    rule: RunAsAny
  privileged: true
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
    - '*'
  allowedCapabilities:
    - '*'
  hostPID: true
  # hostNetwork is required for using host networking
  hostNetwork: false
```

**Hint**: Allowing `hostNetwork` usage is required when using `hostNetwork: true` in a Cluster `CustomResourceDefinition`!

You are then also required to allow the usage of `hostPorts` in the `PodSecurityPolicy`. The given
port range will allow all ports:

 ```yaml
   hostPorts:
     # Ceph msgr2 port
     - min: 1
       max: 65535
```

##### Flexvolume Configuration

Rook uses [FlexVolume](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md) to integrate with Kubernetes for performing storage operations. In some operating systems where Kubernetes is deployed, the [default Flexvolume plugin directory](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md#prerequisites) (the directory where FlexVolume drivers are installed) is **read-only**.

This is the case for Kubernetes deployments on:

* [Atomic](https://www.projectatomic.io/)
* [ContainerLinux](https://coreos.com/os/docs/latest/) (previously named CoreOS)
* [OpenShift](https://www.openshift.com/)
* [Rancher](http://rancher.com/)
* [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/)
* [Azure AKS](https://docs.microsoft.com/en-us/azure/aks/)

Especially in these environments, the kubelet needs to be told to use a different FlexVolume plugin directory that is accessible and read/write (`rw`). These steps need to be carried out on **all nodes** in your cluster.

Please refer to the section that is applicable to your environment/platform, it contains more information on FlexVolume on your platform.

###### Not a listed platform

If you are not using a platform that is listed above and the path `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` is read/write, you don't need to configure anything.

That is because `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` is the kubelet default FlexVolume path and Rook assumes the default FlexVolume path if not set differently.

If running `mkdir -p /usr/libexec/kubernetes/kubelet-plugins/volume/exec/` should give you an error about read-only filesystem, you need to use the [most common read/write FlexVolume path](#most-common-readwrite-flexvolume-path) and configure it on the Rook operator and kubelet.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Atomic

See the [OpenShift](#openshift) section, unless running with OpenStack Magnum, then see [OpenStack Magnum](#openstack-magnum) section.

###### ContainerLinux

Use the [Most common read/write FlexVolume path](#most-common-readwrite-flexvolume-path) for the next steps.

The kubelet's systemD unit file can be located at: `/etc/systemd/system/kubelet.service`.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Kubespray

Kubespray uses a non-standard [FlexVolume plugin directory](https://github.com/kubernetes-sigs/kubespray/blob/master/roles/kubernetes/node/defaults/main.yml#L55): `/var/lib/kubelet/volume-plugins`.

The Kubespray configured kubelet is already configured to use that directory.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### OpenShift

To find out which FlexVolume directory path you need to set on the Rook operator, please look at the OpenShift docs of the version you are using, [latest OpenShift Flexvolume docs](https://docs.openshift.org/latest/install_config/persistent_storage/persistent_storage_flex_volume.html#flexvolume-installation) (they also contain the FlexVolume path for Atomic).

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Rancher

Rancher provides an easy way to configure kubelet. The FlexVolume flag will be shown later on in the [configuring the FlexVolume path](#configuring-the-flexvolume-path).

It can be provided to the kubelet configuration template at deployment time or by using the `up to date` feature if Kubernetes is already deployed.

Rancher deploys kubelet as a docker container, you need to mount the host's flexvolume path into the kubelet image as a volume, this can be done in the `extra_binds` section of the kubelet cluster config.

Configure the Rancher deployed kubelet by updating the `cluster.yml` file kubelet section:

```yaml
services:
  kubelet:
    extra_args:
      volume-plugin-dir: /usr/libexec/kubernetes/kubelet-plugins/volume/exec
    extra_binds:
      - /usr/libexec/kubernetes/kubelet-plugins/volume/exec:/usr/libexec/kubernetes/kubelet-plugins/volume/exec
```

If you're using [rke](https://github.com/rancher/rke), run `rke up`, this will update and restart your kubernetes cluster system components, in this case the kubelet docker instance(s) will get restarted with the new volume bind and volume plugin dir flag.

The default FlexVolume path for Rancher is `/usr/libexec/kubernetes/kubelet-plugins/volume/exec` which is also the default FlexVolume path for the Rook operator.

If the default path as above is used no further configuration is required, otherwise if a different path is used the Rook operator will need to be reconfigured, to do this continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Google Kubernetes Engine (GKE)

Google's Kubernetes Engine uses a non-standard FlexVolume plugin directory: `/home/kubernetes/flexvolume`

The kubelet on GKE is already configured to use that directory.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Azure AKS

AKS uses a non-standard FlexVolume plugin directory: `/etc/kubernetes/volumeplugins`

The kubelet on AKS is already configured to use that directory.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Tectonic

Follow [these instructions](https://rook.io/docs/rook/v1.1/tectonic.html) to configure the Flexvolume plugin for Rook on Tectonic during ContainerLinux node ignition file provisioning.
If you want to use Rook with an already provisioned Tectonic cluster, please refer to the [ContainerLinux](#containerlinux) section.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### OpenStack Magnum

OpenStack Magnum is using Atomic, which uses a non-standard FlexVolume plugin directory at:  `/var/lib/kubelet/volumeplugins`

The kubelet in OpenStack Magnum is already configured to use that directory.

You will need to use this value when [configuring the Rook operator](#configuring-the-rook-operator)

###### Custom containerized kubelet

Use the [most common read/write FlexVolume path](#most-common-readwrite-flexvolume-path) for the next steps.

If your kubelet is running as a (Docker, rkt, etc) container you need to make sure that this directory from the host is reachable by the kubelet inside the container.

Continue with [configuring the FlexVolume path](#configuring-the-flexvolume-path) to configure Rook to use the FlexVolume path.

###### Configuring the FlexVolume path

If the environment specific section doesn't mention a FlexVolume path in this doc or external docs, please refer to the [most common read/write FlexVolume path](#most-common-readwrite-flexvolume-path) section, before continuing to [configuring the FlexVolume path](#configuring-the-flexvolume-path).

***Most common read/write FlexVolume path***

The most commonly used read/write FlexVolume path on most systems is `/var/lib/kubelet/volumeplugins`.

This path is commonly used for FlexVolume because `/var/lib/kubelet` is read write on most systems.

***Configuring the Rook operator***

You must provide the above found FlexVolume path when deploying the [rook-operator](https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/ceph/operator.yaml) by setting the environment variable `FLEXVOLUME_DIR_PATH`.
For example:

```yaml
env:
[...]
- name: FLEXVOLUME_DIR_PATH
  value: "/var/lib/kubelet/volumeplugins"
```

(In the `operator.yaml` manifest replace `<PathToFlexVolumes>` with the path or if you use helm set the `agent.flexVolumeDirPath` to the FlexVolume path)

***Configuring the Kubernetes kubelet***

You need to add the flexvolume flag with the path to all nodes's kubelet in the Kubernetes cluster:

```shell
--volume-plugin-dir=PATH_TO_FLEXVOLUME
```

(Where the `PATH_TO_FLEXVOLUME` is the above found FlexVolume path)

The location where you can set the kubelet FlexVolume path (flag) depends on your platform.
Please refer to your platform documentation for that and/or the [platform specific FlexVolume path](#platform-specific-flexvolume-path) for information about that.

After adding the flag to kubelet, kubelet must be restarted for it to pick up the new flag.

##### Kernel with RBD module

Rook Ceph requires a Linux kernel built with the RBD module. Many distributions of Linux have this module but some don’t, e.g. the GKE Container-Optimised OS (COS) does not have RBD. You can test your Kubernetes nodes by running modprobe rbd. If it says ‘not found’, you may have to [rebuild your kernel](https://rook.io/docs/rook/master/common-issues.html#rook-agent-rbd-module-missing-error) or choose a different Linux distribution.

##### Kernel modules directory configuration

Normally, on Linux, kernel modules can be found in `/lib/modules`. However, there are some distributions that put them elsewhere. In that case the environment variable `LIB_MODULES_DIR_PATH` can be used to override the default. Also see the documentation in [helm-operator](https://rook.io/docs/rook/v1.1/helm-operator.html) on the parameter `agent.libModulesDirPath`. One notable distribution where this setting is useful would be [NixOS](https://nixos.org/).

##### Extra agent mounts

On certain distributions it may be necessary to mount additional directories into the agent container. That is what the environment variable `AGENT_MOUNTS` is for. Also see the documentation in [helm-operator](https://rook.io/docs/rook/v1.1/helm-operator.html) on the parameter `agent.mounts`. The format of the variable content should be `mountname1=/host/path1:/container/path1,mountname2=/host/path2:/container/path2`.

##### LVM package

Some Linux distributions do not ship with the `lvm2` package. This package is required on all storage nodes in your k8s cluster. Please install it using your Linux distribution’s package manager; for example:

```shell
# Centos
sudo yum install -y lvm2

# Ubuntu
sudo apt-get install -y lvm2
```

##### Bootstrapping Kubernetes

Rook will run wherever Kubernetes is running. Here are some simple environments to help you get started with Rook.

***Minikube***  
To install minikube, refer to this [page](https://github.com/kubernetes/minikube/releases). Once you have `minikube` installed, start a cluster by doing the following:

```shell
$ minikube start
Starting local Kubernetes cluster...
Starting VM...
SSH-ing files into VM...
Setting up certs...
Starting cluster components...
Connecting to cluster...
Setting up kubeconfig...
Kubectl is now configured to use the cluster.
```

After these steps, your minikube cluster is ready to install Rook on.

***Kubeadm***  
You can easily spin up Rook on top of a `kubeadm` cluster. You can find the instructions on how to install `kubeadm` in the [Install](https://kubernetes.io/docs/setup/independent/install-kubeadm/) `kubeadm` page.

By using `kubeadm`, you can use Rook in just a few minutes!

***New local Kubernetes cluster with Vagrant***  
For a quick start with a new local cluster, use the Rook fork of [coreos-kubernetes](https://github.com/rook/coreos-kubernetes). This will bring up a multi-node Kubernetes cluster with `vagrant` and CoreOS virtual machines ready to use Rook immediately.

```shell
git clone https://github.com/rook/coreos-kubernetes.git
cd coreos-kubernetes/multi-node/vagrant
vagrant up
export KUBECONFIG="$(pwd)/kubeconfig"
kubectl config use-context vagrant-multi
```

Then wait for the cluster to come up and verify that kubernetes is done initializing (be patient, it takes a bit):

```shell
kubectl cluster-info
```

Once you see a url response, your cluster is [ready for use by Rook](https://rook.io/docs/rook/v1.1/ceph-quickstart.html#deploy-rook).

##### Support for authenticated docker registries

If you want to use an image from authenticated docker registry (e.g. for image cache/mirror), you'll need to add an `imagePullSecret` to all relevant service accounts. This way all pods created by the operator (for service account: `rook-ceph-system`) or all new pods in the namespace (for service account: `default`) will have the `imagePullSecret` added to their spec.

The whole process is described in the [official kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account).

###### Example setup for a ceph cluster

To get you started, here's a quick rundown for the ceph example from the [quickstart guide](https://rook.io/Documentation/ceph-quickstart.md).

First, we'll create the secret for our registry as described [here](https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod):

```bash
# for namespace rook-ceph
kubectl -n rook-ceph create secret docker-registry my-registry-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL

# and for namespace rook-ceph (cluster)
kubectl -n rook-ceph create secret docker-registry my-registry-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

Next we'll add the following snippet to all relevant service accounts as described [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account):

```yaml
imagePullSecrets:
- name: my-registry-secret
```

The service accounts are:

* `rook-ceph-system` (namespace: `rook-ceph`): Will affect all pods created by the rook operator in the `rook-ceph` namespace.
* `default` (namespace: `rook-ceph`): Will affect most pods in the `rook-ceph` namespace.
* `rook-ceph-mgr` (namespace: `rook-ceph`): Will affect the MGR pods in the `rook-ceph` namespace.
* `rook-ceph-osd` (namespace: `rook-ceph`): Will affect the OSD pods in the `rook-ceph` namespace.

You can do it either via e.g. `kubectl -n <namespace> edit serviceaccount default` or by modifying the [`operator.yaml`](https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/ceph/operator.yaml)
and [`cluster.yaml`](https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/ceph/cluster.yaml) before deploying them.

Since it's the same procedure for all service accounts, here is just one example:

```bash
kubectl -n rook-ceph edit serviceaccount default
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: rook-ceph
secrets:
- name: default-token-12345
imagePullSecrets:                # here are the new
- name: my-registry-secret       # parts
```

After doing this for all service accounts all pods should be able to pull the image from your registry.

#### TL;DR

If you’re feeling lucky, a simple Rook cluster can be created with the following kubectl commands and [example yaml files](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph). For the more detailed install, skip to the next section to [deploy the Rook operator](https://rook.io/docs/rook/v1.1/ceph-quickstart.html#deploy-the-rook-operator).

```shell
cd cluster/examples/kubernetes/ceph
kubectl create -f common.yaml
kubectl create -f operator.yaml
kubectl create -f cluster-test.yaml
```

After the cluster is running, you can create [block, object, or file storage](https://rook.io/docs/rook/v1.1/ceph-quickstart.html#storage) to be consumed by other applications in your cluster.

***Production Environments***  
For production environments it is required to have **local storage devices** attached to your nodes. In this walkthrough, the requirement of local storage devices is relaxed so you can get a cluster up and running as a “test” environment to experiment with Rook. A Ceph filestore OSD will be created in a `directory` instead of requiring a device. For production environments, you will want to follow the example in `cluster.yaml` instead of `cluster-test.yaml` in order to configure the devices instead of test directories. See the [Ceph examples](https://rook.io/docs/rook/v1.1/ceph-examples.html) for more details.  
生产环境需要使用本地存储设备连接到 k8s 的节点上，在本流程中，本地存储设备的要求被放松了，因此你可以以测试环境启动的方式启动和运行 rook-ceph 集群体验下 rook。 ceph filestore osd 将会在文件夹中创建，而不是需要一个存储设备。 对于生产环境，你需要按照 example 中的 cluster.yaml 而不是 cluster-test.yaml 配置存储设备而不是配置文件夹。

#### Deploy the Rook Operator

The first step is to deploy the Rook operator. Check that you are using the [example yaml files](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph) that correspond to your release of Rook. For more options, see the [examples documentation](https://rook.io/docs/rook/v1.1/ceph-examples.html).

```shell
cd cluster/examples/kubernetes/ceph
kubectl create -f common.yaml
kubectl create -f operator.yaml

# verify the rook-ceph-operator is in the `Running` state before proceeding
kubectl -n rook-ceph get pod
```

You can also deploy the operator with the [Rook Helm Chart](https://rook.io/docs/rook/v1.1/helm-operator.html).

##### Ceph Examples

Configuration for Rook and Ceph can be configured in multiple ways to provide block devices, shared file system volumes or object storage in a kubernetes namespace. We have provided several examples to simplify storage setup, but remember there are many tunables and you will need to decide what settings work for your use case and environment.

See the **[example yaml files](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph)** folder for all the rook/ceph setup example spec files.

###### Common Resources

The first step to deploy Rook is to create the common resources. The configuration for these resources will be the same for most deployments. The [common.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/common.yaml) sets these resources up.

```shell
kubectl create -f common.yaml
```

The examples all assume the operator and all Ceph daemons will be started in the same namespace. If you want to deploy the operator in a separate namespace, see the comments throughout `common.yaml`.

###### Operator

After the common resources are created, the next step is to create the Operator deployment. Several spec file examples are provided in [this directory](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/):

* `operator.yaml`: The most common settings for production deployments
  * `kubectl create -f operator.yaml`
* `operator-openshift.yaml`: Includes all of the operator settings for running a basic Rook cluster in an OpenShift environment. You will also want to review the [OpenShift Prerequisites](https://rook.io/docs/rook/v1.1/openshift.html) to confirm the settings.
  * `oc create -f operator-openshift.yaml`

Settings for the operator are configured through environment variables on the operator deployment. The individual settings are documented in [operator.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/operator.yaml).

###### Cluster CRD

Now that your operator is running, let's create your Ceph storage cluster:

* `cluster.yaml`: This file contains common settings for a production storage cluster. Requires at least three nodes.
* `cluster-test.yaml`: Settings for a test cluster where redundancy is not configured. Requires only a single node.
* `cluster-minimal.yaml`: Brings up a cluster with only one [ceph-mon](http://docs.ceph.com/docs/nautilus/man/8/ceph-mon/) and a [ceph-mgr](http://docs.ceph.com/docs/nautilus/mgr/) so the Ceph dashboard can be used for the remaining cluster configuration.

See the [Cluster CRD](https://rook.io/docs/rook/v1.1/ceph-cluster-crd.html) topic for more details on the settings.

Monitors may be configured to run on PVC storage. Details on [how to set this up
and some minor restrctions are described here](https://rook.io/docs/rook/v1.1/ceph-cluster-crd.md#mon-settings).

###### Setting up consumable storage

Now we are ready to setup [block](https://ceph.com/ceph-storage/block-storage/), [shared filesystem](https://ceph.com/ceph-storage/file-system/) or [object storage](https://ceph.com/ceph-storage/object-storage/) in the Rook Ceph cluster. These kinds of storage are respectively referred to as CephBlockPool, CephFilesystem and CephObjectStore in the spec files.

***Block Devices***

Ceph can provide raw block device volumes to pods. Each example below sets up a storage class which can then be used to provision a block device in kubernetes pods. The storage class is defined with [a pool](http://docs.ceph.com/docs/master/rados/operations/pools/) which defines the level of data redundancy in Ceph:

* `storageclass.yaml`: This example illustrates replication of 3 for production scenarios and requires at least three nodes. Your data is replicated on three different kubernetes worker nodes and intermittent or long-lasting single node failures will not result in data unavailability or loss.
* `storageclass-ec.yaml`: Configures erasure coding for data durability rather than replication. [Ceph's erasure coding](http://docs.ceph.com/docs/master/rados/operations/erasure-code/) is more efficient than replication so you can get high reliability without the 3x replication cost of the preceding example (but at the cost of higher computational encoding and decoding costs on the worker nodes). Erasure coding requires at least three nodes. See the [Erasure coding](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html#erasure-coded) documentation for more details. **Note: Erasure coding is only available with the flex driver. Support from the CSI driver is coming soon.**
* `storageclass-test.yaml`: Replication of 1 for test scenarios and it requires only a single node. Do not use this for applications that store valuable data or have high-availability storage requirements, since a single node failure can result in data loss.

The storage classes are found in different sub-directories depending on the driver:

* `csi/rbd`: The CSI driver for block devices. This is the preferred driver going forward.
* `flex`: The flex driver will be deprecated in a future release to be determined.

See the [Ceph Pool CRD](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html) topic for more details on the settings.

***Shared File System***

Ceph file system (CephFS) allows the user to 'mount' a shared posix-compliant folder into one or more hosts (pods in the container world). This storage is similar to NFS shared storage or CIFS shared folders, as explained [here](https://ceph.com/ceph-storage/file-system/).

File storage contains multiple pools that can be configured for different scenarios:

* `filesystem.yaml`: Replication of 3 for production scenarios. Requires at least three nodes.
* `filesystem-ec.yaml`: Erasure coding for production scenarios. Requires at least three nodes.
* `filesystem-test.yaml`: Replication of 1 for test scenarios. Requires only a single node.

Dynamic provisioning is possible with the CSI driver. The storage class for shared file systems is found in the `csi/cephfs` directory.

See the [Shared File System CRD](https://rook.io/docs/rook/v1.1/ceph-filesystem-crd.html) topic for more details on the settings.

***Object Storage***

Ceph supports storing blobs of data called objects that support HTTP(s)-type get/put/post and delete semantics. This storage is similar to AWS S3 storage, for example.

Object storage contains multiple pools that can be configured for different scenarios:

* `object.yaml`: Replication of 3 for production scenarios.  Requires at least three nodes.
* `object-openshift.yaml`: Replication of 3 with rgw in a port range valid for OpenShift.  Requires at least three nodes.
* `object-ec.yaml`: Erasure coding rather than replication for production scenarios.  Requires at least three nodes.
* `object-test.yaml`: Replication of 1 for test scenarios. Requires only a single node.

See the [Object Store CRD](https://rook.io/docs/rook/v1.1/ceph-object-store-crd.html) topic for more details on the settings.

***Object Storage User***

* `object-user.yaml`: Creates a simple object storage user and generates credentials for the S3 API

***Object Storage Buckets***

The Ceph operator also runs an object store bucket provisioner which can grant access to existing buckets or dynamically provision new buckets.

* [object-bucket-claim-retain.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/object-bucket-claim-retain.yaml) Creates a request for a new bucket by referencing a StorageClass which saves the bucket when the initiating OBC is deleted.
* [object-bucket-claim-delete.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/object-bucket-claim-delete.yaml) Creates a request for a new bucket by referencing a StorageClass which deletes the bucket when the initiating OBC is deleted.
* [storageclass-bucket-retain.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/storageclass-bucket-retain.yaml) Creates a new StorageClass which defines the Ceph Object Store, a region, and retains the bucket after the initiating OBC is deleted.
* [storageclass-bucket-delete.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/storageclass-bucket-delete.yaml) Creates a new StorageClass which defines the Ceph Object Store, a region, and deletes the bucket after the initiating OBC is deleted.

##### Ceph Operator Helm Chart

Installs [rook](https://github.com/rook/rook) to create, configure, and manage Ceph clusters on Kubernetes.

###### Introduction

This chart bootstraps a [rook-ceph-operator](https://github.com/rook/rook) deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

###### Prerequisites

* Kubernetes 1.10+

***RBAC***  
If role-based access control (RBAC) is enabled in your cluster, you may need to give Tiller (the server-side component of Helm) additional permissions. **If RBAC is not enabled, be sure to set `rbacEnable` to `false` when installing the chart.**

```console
# Create a ServiceAccount for Tiller in the `kube-system` namespace
kubectl --namespace kube-system create sa tiller

# Create a ClusterRoleBinding for Tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

# Patch Tiller's Deployment to use the new ServiceAccount
kubectl --namespace kube-system patch deploy/tiller-deploy -p '{"spec": {"template": {"spec": {"serviceAccountName": "tiller"}}}}'
```

###### Installing

The Ceph Operator helm chart will install the basic components necessary to create a storage platform for your Kubernetes cluster. After the helm chart is installed, you will need to [create a Rook cluster](https://rook.io/docs/rook/v1.1/ceph-quickstart.html#create-a-rook-cluster).

The `helm install` command deploys rook on the Kubernetes cluster in the default configuration. The [configuration](#configuration) section lists the parameters that can be configured during installation. It is recommended that the rook operator be installed into the `rook-ceph` namespace (you will install your clusters into separate namespaces).

Rook currently publishes builds of the Ceph operator to the `release` and `master` channels.

***Release***  
The release channel is the most recent release of Rook that is considered stable for the community.

```console
helm repo add rook-release https://charts.rook.io/release
helm install --namespace rook-ceph rook-release/rook-ceph
```

***Master***  
The master channel includes the latest commits, with all automated tests green. Historically it has been very stable, though it is only recommended for testing.

The critical point to consider is that upgrades are not supported to or from master builds.

To install the helm chart from master, you will need to pass the specific version returned by the `search` command.

```console
helm repo add rook-master https://charts.rook.io/master
helm search rook-ceph
helm install --namespace rook-ceph rook-master/rook-ceph --version <version>
```

For example:

```
helm install --namespace rook-ceph rook-master/rook-ceph --version v0.7.0-278.gcbd9726
```

****Development Build****  
To deploy from a local build from your development environment:

1. Build the Rook docker image: `make`
2. Copy the image to your K8s cluster, such as with the `docker save` then the `docker load` commands
3. Install the helm chart

    ```console
    cd cluster/charts/rook-ceph
    helm install --namespace rook-ceph --name rook-ceph .
    ```

###### Uninstalling the Chart

To uninstall/delete the `rook-ceph` deployment:

```console
$ helm delete --purge rook-ceph
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

###### Configuration

The following tables lists the configurable parameters of the rook-operator chart and their default values.

| Parameter                    | Description                                                                                             | Default                                                |
| ---------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `image.repository`           | Image                                                                                                   | `rook/ceph`                                            |
| `image.tag`                  | Image tag                                                                                               | `master`                                               |
| `image.pullPolicy`           | Image pull policy                                                                                       | `IfNotPresent`                                         |
| `rbacEnable`                 | If true, create & use RBAC resources                                                                    | `true`                                                 |
| `pspEnable`                  | If true, create & use PSP resources                                                                     | `true`                                                 |
| `resources`                  | Pod resource requests & limits                                                                          | `{}`                                                   |
| `annotations`                | Pod annotations                                                                                         | `{}`                                                   |
| `logLevel`                   | Global log level                                                                                        | `INFO`                                                 |
| `nodeSelector`               | Kubernetes `nodeSelector` to add to the Deployment.                                                     | <none>                                                 |
| `tolerations`                | List of Kubernetes `tolerations` to add to the Deployment.                                              | `[]`                                                   |
| `currentNamespaceOnly`   | Whether the operator should watch cluster CRD in its own namespace or not                               | `false`                                                 |
| `hostpathRequiresPrivileged` | Runs Ceph Pods as privileged to be able to write to `hostPath`s in OpenShift with SELinux restrictions. | `false`                                                |
| `agent.flexVolumeDirPath`    | Path where the Rook agent discovers the flex volume plugins (*)                                         | `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` |
| `agent.libModulesDirPath`    | Path where the Rook agent should look for kernel modules (*)                                            | `/lib/modules`                                         |
| `agent.mounts`               | Additional paths to be mounted in the agent container (**)                                              | <none>                                                 |
| `agent.mountSecurityMode`    | Mount Security Mode for the agent.                                                                      | `Any`                                                  |
| `agent.toleration`           | Toleration for the agent pods                                                                           | <none>                                                 |
| `agent.tolerationKey`        | The specific key of the taint to tolerate                                                               | <none>                                                 |
| `agent.tolerations`          | Array of tolerations in YAML format which will be added to agent deployment                             | <none>                                                 |
| `agent.nodeAffinity`         | The node labels for affinity of `rook-agent` (***)                                                      | <none>                                                 |
| `discover.toleration`        | Toleration for the discover pods                                                                        | <none>                                                 |
| `discover.tolerationKey`     | The specific key of the taint to tolerate                                                               | <none>                                                 |
| `discover.tolerations`       | Array of tolerations in YAML format which will be added to discover deployment                          | <none>                                                 |
| `discover.nodeAffinity`      | The node labels for affinity of `discover-agent` (***)                                                  | <none>                                                 |
| `mon.healthCheckInterval`    | The frequency for the operator to check the mon health                                                  | `45s`                                                  |
| `mon.monOutTimeout`          | The time to wait before failing over an unhealthy mon                                                   | `600s`                                                 |

For information on what to set `agent.flexVolumeDirPath` to, please refer to the [Rook flexvolume documentation](https://rook.io/docs/rook/v1.1/flexvolume.html)

`agent.mounts` should have this format `mountname1=/host/path:/container/path,mountname2=/host/path2:/container/path2`

`agent.nodeAffinity` and `discover.nodeAffinity` should have the format `"role=storage,rook; storage=ceph"` or `storage=;role=rook-example` or `storage=;` (_checks only for presence of key_)

***Command Line***  
You can pass the settings with helm command line parameters. Specify each parameter using the
`--set key=value[,key=value]` argument to `helm install`. For example, the following command will install rook where RBAC is not enabled.

```console
$ helm install --namespace rook-ceph --name rook-ceph rook-release/rook-ceph --set rbacEnable=false
```

***Settings File***  
Alternatively, a yaml file that specifies the values for the above parameters (`values.yaml`) can be provided while installing the chart.

```console
$ helm install --namespace rook-ceph --name rook-ceph rook-release/rook-ceph -f values.yaml
```

Here are the sample settings to get you started.

```yaml
image:
  prefix: rook
  repository: rook/ceph
  tag: master
  pullPolicy: IfNotPresent

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

rbacEnable: true
pspEnable: true
```

#### Create a Rook Ceph Cluster

Now that the Rook operator is running we can create the Ceph cluster. For the cluster to survive reboots, make sure you set the `dataDirHostPath` property that is valid for your hosts. For more settings, see the documentation on [configuring the cluster](https://rook.io/docs/rook/v1.1/ceph-cluster-crd.html).

> CephCluster 这个 CRD 的介绍见 [configuring the cluster](https://rook.io/docs/rook/v1.1/ceph-cluster-crd.html)

Save the cluster spec as `cluster-test.yaml`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    # For the latest ceph images, see https://hub.docker.com/r/ceph/ceph/tags
    image: ceph/ceph:v14.2.4-20190917
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
  dashboard:
    enabled: true
  storage:
    useAllNodes: true
    useAllDevices: false
    # Important: Directories should only be used in pre-production environments
    directories:
    - path: /var/lib/rook

```

Create the cluster:

```bash
kubectl create -f cluster-test.yaml
```

Use `kubectl` to list pods in the `rook-ceph` namespace. You should be able to see the following pods once they are all running.

The number of osd pods will depend on the number of nodes in the cluster and the number of devices and directories configured.

If you did not modify the `cluster-test.yaml` above, it is expected that one OSD will be created per node.

The `rook-ceph-agent` and `rook-discover` pods are also optional depending on your settings.

```bash
$ kubectl -n rook-ceph get pod
NAME                                   READY   STATUS      RESTARTS   AGE
rook-ceph-agent-4zkg8                  1/1     Running     0          140s
rook-ceph-mgr-a-d9dcf5748-5s9ft        1/1     Running     0          77s
rook-ceph-mon-a-7d8f675889-nw5pl       1/1     Running     0          105s
rook-ceph-mon-b-856fdd5cb9-5h2qk       1/1     Running     0          94s
rook-ceph-mon-c-57545897fc-j576h       1/1     Running     0          85s
rook-ceph-operator-6c49994c4f-9csfz    1/1     Running     0          141s
rook-ceph-osd-0-7cbbbf749f-j8fsd       1/1     Running     0          23s
rook-ceph-osd-1-7f67f9646d-44p7v       1/1     Running     0          24s
rook-ceph-osd-2-6cd4b776ff-v4d68       1/1     Running     0          25s
rook-ceph-osd-prepare-node1-vx2rz      0/2     Completed   0          60s
rook-ceph-osd-prepare-node2-ab3fd      0/2     Completed   0          60s
rook-ceph-osd-prepare-node3-w4xyz      0/2     Completed   0          60s
rook-discover-dhkb8                    1/1     Running     0          140s
```

To verify that the cluster is in a healthy state, connect to the [Rook toolbox](https://rook.io/docs/rook/v1.1/ceph-toolbox.html) and run the `ceph status` command.

* All mons should be in quorum
* A mgr should be active
* At least one OSD should be active
* If the health is not `HEALTH_OK`, the warnings or errors should be investigated

```console
$ ceph status
  cluster:
    id:     a0452c76-30d9-4c1a-a948-5d8405f19a7c
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 3m)
    mgr: a(active, since 2m)
    osd: 3 osds: 3 up (since 1m), 3 in (since 1m)
...
```

If the cluster is not healthy, please refer to the [Ceph common issues](https://rook.io/docs/rook/v1.1/ceph-common-issues.html) for more details and potential solutions.

#### Storage

For a walkthrough of the three types of storage exposed by Rook, see the guides for:

* **[Block](https://rook.io/docs/rook/v1.1/ceph-block.html)**: Create block storage to be consumed by a pod
* **[Object](https://rook.io/docs/rook/v1.1/ceph-object.html)**: Create an object store that is accessible inside or outside the Kubernetes cluster
* **[Shared File System](https://rook.io/docs/rook/v1.1/ceph-filesystem.html)**: Create a file system to be shared across multiple pods

##### Block Storage

Block storage allows a single pod to mount storage. This guide shows how to create a simple, multi-tier web application on Kubernetes using persistent volumes enabled by Rook.

###### Prerequisites

This guide assumes a Rook cluster as explained in the [Quickstart](https://rook.io/docs/rook/v1.1/ceph-quickstart.html).

###### Provision Storage

Before Rook can provision storage, a [`StorageClass`](https://kubernetes.io/docs/concepts/storage/storage-classes) and [`CephBlockPool`](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html) need to be created. This will allow Kubernetes to interoperate(互操作) with Rook when provisioning persistent volumes.

> storageclass.yaml 包含了 StorageClass 和 CephBlockPool，关于 CephBlockPool 这个 CRD （CustomResourceDefinition）的介绍见 [`CephBlockPool`](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html)

**NOTE:** This sample requires *at least 1 OSD per node*, with each OSD located on *3 different nodes*.

Each OSD must be located on a different node, because the [`failureDomain`](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html#spec) is set to `host` and the `replicated.size` is set to `3`.

**NOTE** This example uses the CSI driver, which is the preferred driver going forward for K8s 1.13 and newer. Examples are found in the [CSI RBD](https://github.com/rook/rook/tree/release-1.1/cluster/examples/kubernetes/ceph/csi/rbd) directory. For an example of a storage class using the flex driver (required for K8s 1.12 or earlier), see the [Flex Driver](#flex-driver) section below, which has examples in the [flex](https://github.com/rook/rook/tree/release-1.1/cluster/examples/kubernetes/ceph/flex) directory.

Save this `StorageClass` definition as `storageclass.yaml`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    clusterID: rook-ceph
    # Ceph pool into which the RBD image shall be created
    pool: replicapool

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
    imageFeatures: layering

    # The secrets contain Ceph admin credentials.
    csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

    # Specify the filesystem type of the volume. If not specified, csi-provisioner
    # will set default as `ext4`.
    csi.storage.k8s.io/fstype: xfs

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete
```

If you've deployed the Rook operator in a namespace other than "rook-ceph" as is common change the prefix in the provisioner to match the namespace you used. For example, if the Rook operator is running in "rook-op" the provisioner value should be "rook-op.rbd.csi.ceph.com".

Create the storage class.

```bash
kubectl create -f cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml
```

**NOTE** As [specified by Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#retain), when using the `Retain` reclaim policy, any Ceph RBD image that is backed by a `PersistentVolume` will continue to exist even after the `PersistentVolume` has been deleted. These Ceph RBD images will need to be cleaned up manually using `rbd rm`.

###### Consume the storage: Wordpress sample

We create a sample app to consume the block storage provisioned by Rook with the classic wordpress and mysql apps. Both of these apps will make use of block volumes provisioned by Rook.

Start mysql and wordpress from the `cluster/examples/kubernetes` folder:

```bash
kubectl create -f mysql.yaml
kubectl create -f wordpress.yaml
```

Both of these apps create a block volume and mount it to their respective pod. You can see the Kubernetes volume claims by running the following:

```bash
$ kubectl get pvc
NAME             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
mysql-pv-claim   Bound     pvc-95402dbc-efc0-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
wp-pv-claim      Bound     pvc-39e43169-efc1-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
```

Once the wordpress and mysql pods are in the `Running` state, get the cluster IP of the wordpress app and enter it in your browser:

```bash
$ kubectl get svc wordpress
NAME        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
wordpress   10.3.0.155   <pending>     80:30841/TCP   2m
```

You should see the wordpress app running.

If you are using Minikube, the Wordpress URL can be retrieved with this one-line command:

```console
echo http://$(minikube ip):$(kubectl get service wordpress -o jsonpath='{.spec.ports[0].nodePort}')
```

**NOTE:** When running in a vagrant environment, there will be no external IP address to reach wordpress with.  You will only be able to reach wordpress via the `CLUSTER-IP` from inside the Kubernetes cluster.

###### Consume the storage: Toolbox

With the pool that was created above, we can also create a block image and mount it directly in a pod. See the [Direct Block Tools](https://rook.io/docs/rook/v1.1/direct-tools.html#block-storage-tools) topic for more details.

###### Teardown

To clean up all the artifacts created by the block demo:

```console
kubectl delete -f wordpress.yaml
kubectl delete -f mysql.yaml
kubectl delete -n rook-ceph cephblockpools.ceph.rook.io replicapool
kubectl delete storageclass rook-ceph-block
```

###### Flex Driver

To create a volume based on the flex driver instead of the CSI driver, see the following example of a storage class. Make sure the flex driver is enabled over Ceph CSI.

For this, you need to set `ROOK_ENABLE_FLEX_DRIVER` to `true` in your operator deployment in the `operator.yaml` file.

The pool definition is the same as for the CSI driver.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  # The value of "clusterNamespace" MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
# Optional, default reclaimPolicy is "Delete". Other options are: "Retain", "Recycle" as documented in https://kubernetes.io/docs/concepts/storage/storage-classes/
reclaimPolicy: Retain
# Optional, if you want to add dynamic resize for PVC. Works for Kubernetes 1.14+
# For now only ext3, ext4, xfs resize support provided, like in Kubernetes itself.
allowVolumeExpansion: true
```

Create the pool and storage class.

```bash
kubectl create -f cluster/examples/kubernetes/ceph/flex/storageclass.yaml
```

Continue with the example above for the [wordpress application](#consume-the-storage-wordpress-sample).

###### Advanced Example: Erasure Coded Block Storage

**IMPORTANT:** This is only possible when using the Flex driver. Ceph CSI 1.2 (with Rook 1.1) does not support this type of configuration yet.

If you want to use erasure coded pool with RBD, your OSDs must use `bluestore` as their `storeType`.
Additionally the nodes that are going to mount the erasure coded RBD block storage must have Linux kernel >= `4.11`.

To be able to use an erasure coded pool you need to create two pools (as seen below in the definitions): one erasure coded and one replicated. The replicated pool must be specified as the `blockPool` parameter. It is used for the metadata of the RBD images.

The erasure coded pool must be set as the `dataBlockPool` parameter below. It is used for the data of the RBD images.

**NOTE:** This example requires *at least 3 bluestore OSDs*, with each OSD located on a *different node*.

The OSDs must be located on different nodes, because the [`failureDomain`](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html#spec) is set to `host` and the `erasureCoded` chunk settings require at least 3 different OSDs (2 `dataChunks` + 1 `codingChunks`).

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicated-metadata-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-data-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  # Make sure you have enough nodes and OSDs running bluestore to support the replica size or erasure code chunks.
  # For the below settings, you need at least 3 OSDs on different nodes (because the `failureDomain` is `host` by default).
  erasureCoded:
    dataChunks: 2
    codingChunks: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicated-metadata-pool
  dataBlockPool: ec-data-pool
  # Specify the namespace of the rook cluster from which to create volumes.
  # If not specified, it will use `rook` as the default namespace of the cluster.
  # This is also the namespace where the cluster will be
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
# Works for Kubernetes 1.14+
allowVolumeExpansion: true
```

(These definitions can also be found in the [`storageclass-ec.yaml`](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/flex/storage-class-ec.yaml) file)

##### Object Storage

Object storage exposes an S3 API to the storage cluster for applications to put and get data.

###### Prerequisites

This guide assumes a Rook cluster as explained in the [Quickstart](https://rook.io/docs/rook/v1.1/ceph-quickstart.html).

###### Create an Object Store

The below sample will create a `CephObjectStore` that starts the RGW service in the cluster with an S3 API.

**NOTE:** This sample requires *at least 3 bluestore OSDs*, with each OSD located on a *different node*.

The OSDs must be located on different nodes, because the [`failureDomain`](https://rook.io/docs/rook/v1.1/ceph-pool-crd.html#spec) is set to `host` and the `erasureCoded` chunk settings require at least 3 different OSDs (2 `dataChunks` + 1 `codingChunks`).

See the [Object Store CRD](https://rook.io/docs/rook/v1.1/ceph-object-store-crd.html#object-store-settings), for more detail on the settings available for a `CephObjectStore`.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  preservePoolsOnDelete: true
  gateway:
    type: s3
    sslCertificateRef:
    port: 80
    securePort:
    instances: 1
```

After the `CephObjectStore` is created, the Rook operator will then create all the pools and other resources necessary to start the service. This may take a minute to complete.

```bash
# Create the object store
kubectl create -f object.yaml

# To confirm the object store is configured, wait for the rgw pod to start
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
```

###### Create a User

Next, create a `CephObjectStoreUser`, which will be used to connect to the RGW service in the cluster using the S3 API.

See the [Object Store User CRD](https://rook.io/docs/rook/v1.1/ceph-object-store-user-crd.html) for more detail on the settings available for a `CephObjectStoreUser`.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: my-user
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "my display name"
```

When the `CephObjectStoreUser` is created, the Rook operator will then create the RGW user on the specified `CephObjectStore` and store the Access Key and Secret Key in a kubernetes secret in the same namespace as the `CephObjectStoreUser`.

```bash
# Create the object store user
kubectl create -f object-user.yaml

# To confirm the object store user is configured, describe the secret
kubectl -n rook-ceph describe secret rook-ceph-object-user-my-store-my-user

Name:       rook-ceph-object-user-my-store-my-user
Namespace:  rook-ceph
Labels:       app=rook-ceph-rgw
                  rook_cluster=rook-ceph
                  rook_object_store=my-store
Annotations:    <none>

Type:   kubernetes.io/rook

Data
====
AccessKey:  20 bytes
SecretKey:  40 bytes
```

The AccessKey and SecretKey data fields can be mounted in a pod as an environment variable. More information on consuming kubernetes secrets can be found in the [K8s secret documentation](https://kubernetes.io/docs/concepts/configuration/secret/)

To directly retrieve the secrets:

```bash
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep AccessKey | awk '{print $2}' | base64 --decode
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep SecretKey | awk '{print $2}' | base64 --decode
```

###### Consume the Object Storage

Use an S3 compatible client to create a bucket in the `CephObjectStore`.

This section will allow you to test connecting to the `CephObjectStore` and uploading and downloading from it. Run the following commands after you have connected to the [Rook toolbox](https://rook.io/docs/rook/v1.1/ceph-toolbox.html).

***Install s3cmd***

To test the `CephObjectStore` we will install the `s3cmd` tool into the toobox pod.

```bash
yum --assumeyes install s3cmd
```

***Connection Environment Variables***

To simplify the s3 client commands, you will want to set the four environment variables for use by your client (ie. inside the toolbox):

```bash
export AWS_HOST=<host>
export AWS_ENDPOINT=<endpoint>
export AWS_ACCESS_KEY_ID=<accessKey>
export AWS_SECRET_ACCESS_KEY=<secretKey>
```

* `Host`: The DNS host name where the rgw service is found in the cluster. Assuming you are using the default `rook-ceph` cluster, it will be `rook-ceph-rgw-my-store.rook-ceph`.
* `Endpoint`: The endpoint where the rgw service is listening. Run `kubectl -n rook-ceph get svc rook-ceph-rgw-my-store`, then combine the clusterIP and the port.
* `Access key`: The user's `access_key` as printed above
* `Secret key`: The user's `secret_key` as printed above

The variables for the user generated in this example would be:

```bash
export AWS_HOST=rook-ceph-rgw-my-store.rook-ceph
export AWS_ENDPOINT=10.104.35.31:80
export AWS_ACCESS_KEY_ID=XEZDB3UJ6X7HVBE7X7MA
export AWS_SECRET_ACCESS_KEY=7yGIZON7EhFORz0I40BFniML36D2rl8CQQ5kXU6l
```

The access key and secret key can be retrieved as described in the section above on [creating a user](#create-a-user).

***Create a bucket***

Now that the user connection variables were set above, we can proceed to perform operations such as creating buckets.

Create a bucket in the `CephObjectStore`

```bash
s3cmd mb --no-ssl --host=${AWS_HOST} --region=":default-placement" --host-bucket="" s3://rookbucket
```

List buckets in the `CephObjectStore`

```bash
s3cmd ls --no-ssl --host=${AWS_HOST}
```

***PUT or GET an object***

Upload a file to the newly created bucket

```bash
echo "Hello Rook" > /tmp/rookObj
s3cmd put /tmp/rookObj --no-ssl --host=${AWS_HOST} --host-bucket=  s3://rookbucket
```

Download and verify the file from the bucket

```bash
s3cmd get s3://rookbucket/rookObj /tmp/rookObj-download --no-ssl --host=${AWS_HOST} --host-bucket=
cat /tmp/rookObj-download
```

###### Access External to the Cluster

Rook sets up the object storage so pods will have access internal to the cluster. If your applications are running outside the cluster, you will need to setup an external service through a `NodePort`.

First, note the service that exposes RGW internal to the cluster. We will leave this service intact and create a new service for external access.

```bash
$ kubectl -n rook-ceph get service rook-ceph-rgw-my-store
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
rook-ceph-rgw-my-store   10.3.0.177   <none>        80/TCP      2m
```

Save the external service as `rgw-external.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-rgw-my-store-external
  namespace: rook-ceph
  labels:
    app: rook-ceph-rgw
    rook_cluster: rook-ceph
    rook_object_store: my-store
spec:
  ports:
  - name: rgw
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: rook-ceph-rgw
    rook_cluster: rook-ceph
    rook_object_store: my-store
  sessionAffinity: None
  type: NodePort
```

Now create the external service.

```bash
kubectl create -f rgw-external.yaml
```

See both rgw services running and notice what port the external service is running on:

```bash
$ kubectl -n rook-ceph get service rook-ceph-rgw-my-store rook-ceph-rgw-my-store-external
NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
rook-ceph-rgw-my-store            ClusterIP   10.104.82.228    <none>        80/TCP         4m
rook-ceph-rgw-my-store-external   NodePort    10.111.113.237   <none>        80:31536/TCP   39s
```

Internally the rgw service is running on port `80`. The external port in this case is `31536`. Now you can access the `CephObjectStore` from anywhere! All you need is the hostname for any machine in the cluster, the external port, and the user credentials.

##### Shared File System

A shared file system can be mounted with read/write permission from multiple pods. This may be useful for applications which can be clustered using a shared filesystem.

This example runs a shared file system for the [kube-registry](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry).

###### Prerequisites

This guide assumes you have created a Rook cluster as explained in the main [Kubernetes guide](https://rook.io/docs/rook/v1.1/ceph-quickstart.html)

###### Multiple File Systems Not Supported

By default only one shared file system can be created with Rook. Multiple file system support in Ceph is still considered experimental and can be enabled with the environment variable `ROOK_ALLOW_MULTIPLE_FILESYSTEMS` defined in `operator.yaml`.

Please refer to [cephfs experimental features](http://docs.ceph.com/docs/master/cephfs/experimental-features/#multiple-filesystems-within-a-ceph-cluster) page for more information.

###### Create the File System

Create the file system by specifying the desired settings for the metadata pool, data pools, and metadata server in the `CephFilesystem` CRD. In this example we create the metadata pool with replication of three and a single data pool with replication of three. For more options, see the documentation on [creating shared file systems](https://rook.io/docs/rook/v1.1/ceph-filesystem-crd.html).

> CephFilesystem 这个 CRD 的介绍见：<https://rook.io/docs/rook/v1.1/ceph-filesystem-crd.html>

Save this shared file system definition as `filesystem.yaml`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  preservePoolsOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

The Rook operator will create all the pools and other resources necessary to start the service. This may take a minute to complete.

```bash
# Create the file system
$ kubectl create -f filesystem.yaml

# To confirm the file system is configured, wait for the mds pods to start
$ kubectl -n rook-ceph get pod -l app=rook-ceph-mds
NAME                                      READY     STATUS    RESTARTS   AGE
rook-ceph-mds-myfs-7d59fdfcf4-h8kw9       1/1       Running   0          12s
rook-ceph-mds-myfs-7d59fdfcf4-kgkjp       1/1       Running   0          12s
```

To see detailed status of the file system, start and connect to the [Rook toolbox](https://rook.io/docs/rook/v1.1/ceph-toolbox.html). A new line will be shown with `ceph status` for the `mds` service. In this example, there is one active instance of MDS which is up, with one MDS instance in `standby-replay` mode in case of failover.

```bash
$ ceph status
  ...
  services:
    mds: myfs-1/1/1 up {[myfs:0]=mzw58b=up:active}, 1 up:standby-replay
```

###### Provision Storage

Before Rook can start provisioning storage, a StorageClass needs to be created based on the filesystem. This is needed for Kubernetes to interoperate with the CSI driver to create persistent volumes.

**NOTE** This example uses the CSI driver, which is the preferred driver going forward for K8s 1.13 and newer. Examples are found in the [CSI CephFS](https://github.com/rook/rook/tree/release-1.1/cluster/examples/kubernetes/ceph/csi/cephfs) directory. For an example of a volume using the flex driver (required for K8s 1.12 and earlier), see the [Flex Driver](#flex-driver) section below.

Save this storage class definition as `storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where operator is deployed.
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-data0

  # Root path of an existing CephFS volume
  # Required for provisionVolume: "false"
  # rootPath: /absolute/path

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete
```

If you've deployed the Rook operator in a namespace other than "rook-ceph" as is common change the prefix in the provisioner to match the namespace you used. For example, if the Rook operator is running in "rook-op" the provisioner value should be "rook-op.rbd.csi.ceph.com".

Create the storage class.

```bash
kubectl create -f cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml
```

###### Consume the Shared File System: K8s Registry Sample

As an example, we will start the kube-registry pod with the shared file system as the backing store.
Save the following spec as `kube-registry.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: kube-registry
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: registry
        image: registry:2
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 100m
            memory: 100Mi
        env:
        # Configuration reference: https://docs.docker.com/registry/configuration/
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_HTTP_SECRET
          value: "Ple4seCh4ngeThisN0tAVerySecretV4lue"
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: registry
        readinessProbe:
          httpGet:
            path: /
            port: registry
      volumes:
      - name: image-store
        persistentVolumeClaim:
          claimName: cephfs-pvc
          readOnly: false
```

Create the Kube registry deployment:

```bash
kubectl create -f cluster/examples/kubernetes/ceph/csi/cephfs/kube-registry.yaml
```

You now have a docker registry which is HA with persistent storage.

###### Kernel Version Requirement

If the Rook cluster has more than one filesystem and the application pod is scheduled to a node with kernel version older than 4.7, inconsistent results may arise since kernels older than 4.7 do not support specifying filesystem namespaces.

###### Consume the Shared File System: Toolbox

Once you have pushed an image to the registry (see the [instructions](https://github.com/kubernetes/kubernetes/tree/release-1.9/cluster/addons/registry) to expose and use the kube-registry), verify that kube-registry is using the filesystem that was configured above by mounting the shared file system in the toolbox pod. See the [Direct Filesystem](https://rook.io/docs/rook/v1.1/direct-tools.html#shared-filesystem-tools) topic for more details.

###### Teardown
To clean up all the artifacts created by the file system demo:

```bash
kubectl delete -f kube-registry.yaml
```

To delete the filesystem components and backing data, delete the Filesystem CRD. **Warning: Data will be deleted if preservePoolsOnDelete=false**

```console
kubectl -n rook-ceph delete cephfilesystem myfs
```

Note: If the "preservePoolsOnDelete" filesystem attribute is set to true, the above command won't delete the pools. Creating again the filesystem with the same CRD will reuse again the previous pools.

###### Flex Driver

To create a volume based on the flex driver instead of the CSI driver, see the [kube-registry.yaml](https://github.com/rook/rook/blob/release-1.1/cluster/examples/kubernetes/ceph/flex/kube-registry.yaml) example manifest or refer to the complete flow in the Rook v1.0 [Shared File System](https://rook.io/docs/rook/v1.0/ceph-filesystem.html) documentation.

***Advanced Example: Erasure Coded Filesystem***  
The Ceph filesystem example can be found here: [Ceph Shared File System - Samples - Erasure Coded](https://rook.io/docs/rook/v1.1/ceph-filesystem-crd.html#erasure-coded).

#### Ceph Dashboard

The dashboard is a very helpful tool to give you an overview of the status of your cluster, including overall health, status of the mon quorum, status of the mgr, osd, and other Ceph daemons, view pools and PG status, show logs for the daemons, and more. Rook makes it simple to enable the dashboard.

![The Ceph dashboard](../../images/rook-ceph-dashboard-20191020.png)

##### Enable the Dashboard

The [dashboard](http://docs.ceph.com/docs/mimic/mgr/dashboard/) can be enabled with settings in the cluster CRD. The cluster CRD must have the dashboard `enabled` setting set to `true`.

This is the default setting in the example manifests.

```yaml
  spec:
    dashboard:
      enabled: true
```

The Rook operator will enable the ceph-mgr dashboard module. A K8s service will be created to expose that port inside the cluster. Rook will enable port 8443 for https access.

This example shows that port 8443 was configured.

```bash
kubectl -n rook-ceph get service
NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr                ClusterIP   10.108.111.192   <none>        9283/TCP         3h
rook-ceph-mgr-dashboard      ClusterIP   10.110.113.240   <none>        8443/TCP         3h
```

The first service is for reporting the [Prometheus metrics](ceph-monitoring.md), while the latter service is for the dashboard.

If you are on a node in the cluster, you will be able to connect to the dashboard by using either the DNS name of the service at `https://rook-ceph-mgr-dashboard-https:8443` or by connecting to the cluster IP, in this example at `https://10.110.113.240:8443`.

###### Credentials

After you connect to the dashboard you will need to login for secure access. Rook creates a default user named `admin` and generates a secret called `rook-ceph-dashboard-admin-password` in the namespace where rook is running.

To retrieve the generated password, you can run the following:

```console
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

##### Configure the Dashboard

The following dashboard configuration settings are supported:

```yaml
  spec:
    dashboard:
      urlPrefix: /ceph-dashboard
      port: 8443
      ssl: true
```

* `urlPrefix` If you are accessing the dashboard via a reverse proxy, you may wish to serve it under a URL prefix.  To get the dashboard to use hyperlinks that include your prefix, you can set the `urlPrefix` setting.

* `port` The port that the dashboard is served on may be changed from the default using the `port` setting. The corresponding K8s service exposing the port will automatically be updated.

* `ssl` The dashboard may be served without SSL (useful for when you deploy the dashboard behind a proxy already served using SSL) by setting the `ssl` option to be false.

##### Viewing the Dashboard External to the Cluster

Commonly you will want to view the dashboard from outside the cluster. For example, on a development machine with the cluster running inside minikube you will want to access the dashboard from the host.

There are several ways to expose a service that will depend on the environment you are running in.
You can use an [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/) or [other methods](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) for exposing services such as NodePort, LoadBalancer, or ExternalIPs.

###### Node Port

The simplest way to expose the service in minikube or similar environment is using the NodePort to open a port on the VM that can be accessed by the host. To create a service with the NodePort, save this yaml as `dashboard-external-https.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-external-https
  namespace: rook-ceph
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
spec:
  ports:
  - name: dashboard
    port: 8443
    protocol: TCP
    targetPort: 8443
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  sessionAffinity: None
  type: NodePort
```

Now create the service:

```bash
$ kubectl create -f dashboard-external-https.yaml
```

You will see the new service `rook-ceph-mgr-dashboard-external-https` created:

```bash
$ kubectl -n rook-ceph get service
NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr                           ClusterIP   10.108.111.192   <none>        9283/TCP         4h
rook-ceph-mgr-dashboard                 ClusterIP   10.110.113.240   <none>        8443/TCP         4h
rook-ceph-mgr-dashboard-external-https  NodePort    10.101.209.6     <none>        8443:31176/TCP   4h
```

In this example, port `31176` will be opened to expose port `8443` from the ceph-mgr pod. Find the ip address of the VM. If using minikube, you can run `minikube ip` to find the ip address.

Now you can enter the URL in your browser such as `https://192.168.99.110:31176` and the dashboard will appear.

###### Load Balancer

If you have a cluster on a cloud provider that supports load balancers, you can create a service that is provisioned with a public hostname.

The yaml is the same as `dashboard-external-https.yaml` except for the following line:

```yaml
  type: LoadBalancer
```

Now create the service:

```bash
$ kubectl create -f dashboard-loadbalancer.yaml
```

You will see the new service `rook-ceph-mgr-dashboard-loadbalancer` created:

```bash
$ kubectl -n rook-ceph get service
NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)             AGE
rook-ceph-mgr                            ClusterIP      172.30.11.40     <none>                                                                    9283/TCP            4h
rook-ceph-mgr-dashboard                  ClusterIP      172.30.203.185   <none>                                                                    8443/TCP            4h
rook-ceph-mgr-dashboard-loadbalancer     LoadBalancer   172.30.27.242    a7f23e8e2839511e9b7a5122b08f2038-1251669398.us-east-1.elb.amazonaws.com   8443:32747/TCP      4h
```

Now you can enter the URL in your browser such as

```console
https://a7f23e8e2839511e9b7a5122b08f2038-1251669398.us-east-1.elb.amazonaws.com:8443
```

and the dashboard will appear.

###### Ingress Controller

If you have a cluster with an [nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/) and a Certificate Manager (e.g. [cert-manager](https://cert-manager.readthedocs.io/)) then you can create an Ingress like the one below. This example achieves four things:

1. Exposes the dashboard on the Internet (using an reverse proxy)
2. Issues an valid TLS Certificate for the specified domain name (using [ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment))
3. Tells the reverse proxy that the dashboard itself uses HTTPS
4. Tells the reverse proxy that the dashboard itself does not have a valid certificate (it is self-signed)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rook-ceph-mgr-dashboard
  namespace: rook-ceph
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/server-snippet: |
      proxy_ssl_verify off;
spec:
  tls:
   - hosts:
     - rook-ceph.example.com
     secretName: rook-ceph.example.com
  rules:
  - host: rook-ceph.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: rook-ceph-mgr-dashboard
          servicePort: https-dashboard
```

Customise the Ingress resource to match your cluster. Replace the example domain name `rook-ceph.example.com` with a domain name that will resolve to your Ingress Controller (creating the DNS entry if required).

Now create the Ingress:

```bash
$ kubectl create -f dashboard-ingress-https.yaml
```

You will see the new Ingress `rook-ceph-mgr-dashboard` created:

```bash
$ kubectl -n rook-ceph get ingress
NAME                      HOSTS                      ADDRESS   PORTS     AGE
rook-ceph-mgr-dashboard   rook-ceph.example.com      80, 443   5m
```

And the new Secret for the TLS certificate:

```bash
$ kubectl -n rook-ceph get secret rook-ceph.example.com
NAME                       TYPE                DATA      AGE
rook-ceph.example.com      kubernetes.io/tls   2         4m
```

You can now browse to `https://rook-ceph.example.com/` to log into the dashboard.

##### Enabling Dashboard Object Gateway management

Provided you have deployed the [Ceph Toolbox](https://rook.io/docs/rook/v1.1/ceph-toolbox.html), created an [Object Store](https://rook.io/docs/rook/v1.1/ceph-object.html) and a user, you can enable [Object Gateway management](http://docs.ceph.com/docs/master/mgr/dashboard/#enabling-the-object-gateway-management-frontend) by providing the user credentials to the dashboard:

```console
# Access toolbox CLI:
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

# Enable system flag on the user:
radosgw-admin user modify --uid=my-user --system

# Provide the user credentials:
ceph dashboard set-rgw-api-user-id my-user
ceph dashboard set-rgw-api-access-key <access-key>
ceph dashboard set-rgw-api-secret-key <secret-key>
```

Now you can access the *Object Gateway* menu items.

#### Tools

We have created a toolbox container that contains the full suite of Ceph clients for debugging and troubleshooting your Rook cluster.  Please see the [toolbox readme](https://rook.io/docs/rook/v1.1/ceph-toolbox.html) for setup and usage information. Also see our [advanced configuration](https://rook.io/docs/rook/v1.1/advanced-configuration.md) document for helpful maintenance and tuning examples.

#####  Rook Toolbox

The Rook toolbox is a container with common tools used for rook debugging and testing.

The toolbox is based on CentOS, so more tools of your choosing can be easily installed with `yum`.

###### Running the Toolbox in Kubernetes

The rook toolbox can run as a deployment in a Kubernetes cluster.  After you ensure you have a running Kubernetes cluster with rook deployed (see the [Kubernetes](https://rook.io/docs/rook/v1.1/ceph-quickstart.html) instructions), launch the rook-ceph-tools pod.

Save the tools spec as `toolbox.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-tools
  namespace: rook-ceph
  labels:
    app: rook-ceph-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rook-ceph-tools
  template:
    metadata:
      labels:
        app: rook-ceph-tools
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: rook-ceph-tools
        image: rook/ceph:master
        command: ["/tini"]
        args: ["-g", "--", "/usr/local/bin/toolbox.sh"]
        imagePullPolicy: IfNotPresent
        env:
          - name: ROOK_ADMIN_SECRET
            valueFrom:
              secretKeyRef:
                name: rook-ceph-mon
                key: admin-secret
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /dev
            name: dev
          - mountPath: /sys/bus
            name: sysbus
          - mountPath: /lib/modules
            name: libmodules
          - name: mon-endpoint-volume
            mountPath: /etc/rook
      # if hostNetwork: false, the "rbd map" command hangs, see https://github.com/rook/rook/issues/2021
      hostNetwork: true
      volumes:
        - name: dev
          hostPath:
            path: /dev
        - name: sysbus
          hostPath:
            path: /sys/bus
        - name: libmodules
          hostPath:
            path: /lib/modules
        - name: mon-endpoint-volume
          configMap:
            name: rook-ceph-mon-endpoints
            items:
            - key: data
              path: mon-endpoints
```

Launch the rook-ceph-tools pod:

```bash
kubectl create -f toolbox.yaml
```

Wait for the toolbox pod to download its container and get to the `running` state:

```bash
kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
```

Once the rook-ceph-tools pod is running, you can connect to it with:

```bash
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```

All available tools in the toolbox are ready for your troubleshooting needs.  Example:

```bash
ceph status
ceph osd status
ceph df
rados df
```

When you are done with the toolbox, you can remove the deployment:

```bash
kubectl -n rook-ceph delete deployment rook-ceph-tools
```

###### Troubleshooting without the Toolbox

The Ceph tools will commonly be the only tools needed to troubleshoot a cluster. In that case, you can connect to any of the rook pods and execute the ceph commands in the same way that you would in the toolbox pod such as the mon pods or the operator pod.

If connecting to the mon pods, make sure you connect to the mon most recently started. The mons keep the config updated in memory after starting and may not have the latest config on disk.
For example, after starting the cluster connect to the `mon2` pod instead of `mon0`.

#### Monitoring

Each Rook Ceph cluster has some built in metrics collectors/exporters for monitoring with [Prometheus](https://prometheus.io/).

If you do not have Prometheus running, follow the steps below to enable monitoring of Rook. If your cluster already contains a Prometheus instance, it will automatically discover Rooks scrape endpoint using the standard `prometheus.io/scrape` and `prometheus.io/port` annotations.

##### Prometheus Operator

First the Prometheus operator needs to be started in the cluster so it can watch for our requests to start monitoring Rook and respond by deploying the correct Prometheus pods and configuration.
A full explanation can be found in the [Prometheus operator repository on GitHub](https://github.com/coreos/prometheus-operator), but the quick instructions can be found here:

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/v0.26.0/bundle.yaml
```

This will start the Prometheus operator, but before moving on, wait until the operator is in the `Running` state:

```bash
kubectl get pod
```

Once the Prometheus operator is in the `Running` state, proceed to the next section.

##### Prometheus Instances

With the Prometheus operator running, we can create a service monitor that will watch the Rook cluster and collect metrics regularly.

From the root of your locally cloned Rook repo, go the monitoring directory:

```bash
cd cluster/examples/kubernetes/ceph/monitoring
```

Create the service monitor as well as the Prometheus server pod and service:

```bash
kubectl create -f service-monitor.yaml
kubectl create -f prometheus.yaml
kubectl create -f prometheus-service.yaml
```

Ensure that the Prometheus server pod gets created and advances to the `Running` state before moving on:

```bash
kubectl -n rook-ceph get pod prometheus-rook-prometheus-0
```

##### Prometheus Web Console

Once the Prometheus server is running, you can open a web browser and go to the URL that is output from this command:

```bash
echo "http://$(kubectl -n rook-ceph -o jsonpath={.status.hostIP} get pod prometheus-rook-prometheus-0):30900"
```

You should now see the Prometheus monitoring website.

![Prometheus Monitoring Website](../../images/rook-ceph-prometheus-monitor-20191020.png)


Click on `Graph` in the top navigation bar.

![Prometheus Add graph](../../images/rook-ceph-prometheus-graph-20191020.png)


In the dropdown that says `insert metric at cursor`, select any metric you would like to see, for example `ceph_cluster_total_used_bytes`

![Prometheus Select Metric](../../images/rook-ceph-prometheus-metric-cursor-20191020.png)


Click on the `Execute` button.

![Prometheus Execute Metric](../../images/rook-ceph-prometheus-execute-metric-cursor-20191020.png)

Below the `Execute` button, ensure the `Graph` tab is selected and you should now see a graph of your chosen metric over time.

![Prometheus Execute Metric](../../images/rook-ceph-prometheus-metric-cursor-graph-20191020.png)

##### Prometheus Consoles

You can find Prometheus Consoles here: <https://github.com/ceph/cephmetrics/tree/master/dashboards/current>.

A guide to how you can write your own Prometheus consoles can be found on the official Prometheus site here: <https://prometheus.io/docs/visualization/consoles/>.

##### Prometheus Alerts

To enable prometheus alerts,

first, create the RBAC rules to enable monitoring

```BASH
kubectl create -f cluster/examples/kubernetes/ceph/monitoring/rbac.yaml
```

then, make following changes to `cluster.yaml` and deploy.

```YAML
monitoring:
  enabled: true
  rulesNamespace: "rook-ceph"
```

```BASH
kubectl apply -f cluster.yaml
```

_Note: This expects prometheus to be pre-installed by the admin._

##### Grafana Dashboards

The dashboards have been created by [@galexrt](https://github.com/galexrt). For feedback on the dashboards please reach out to him on the [Rook.io Slack](https://slack.rook.io).

> **NOTE** The dashboards are only compatible with Grafana 5.0.3 or higher.

The following Grafana dashboards are available:

* [Ceph - Cluster](https://grafana.com/dashboards/2842)
* [Ceph - OSD](https://grafana.com/dashboards/5336)
* [Ceph - Pools](https://grafana.com/dashboards/5342)

##### Teardown

To clean up all the artifacts created by the monitoring walkthrough, copy/paste the entire block below (note that errors about resources "not found" can be ignored):

```bash
kubectl delete -f service-monitor.yaml
kubectl delete -f prometheus.yaml
kubectl delete -f prometheus-service.yaml
kubectl delete -f https://raw.githubusercontent.com/coreos/prometheus-operator/v0.26.0/bundle.yaml
```

Then the rest of the instructions in the [Prometheus Operator docs](https://github.com/coreos/prometheus-operator#removal) can be followed to finish cleaning up.

##### Special Cases

###### Tectonic Bare Metal

Tectonic strongly discourages the `tectonic-system` Prometheus instance to be used outside their intentions, so you need to create a new [Prometheus Operator](https://coreos.com/operators/prometheus/docs/latest/) yourself.

After this you only need to create the service monitor as stated above.

###### CSI Liveness

To integrate CSI liveness and grpc into ceph monitoring we will need to deploy a service and service monitor.

```bash
kubectl create -f csi-metrics-service-monitor.yaml
```

This will create the service monitor to have promethues monitor CSI

#### Teardown

If you want to tear down the cluster and bring up a new one, be aware of the following resources that will need to be cleaned up:

* `rook-ceph` namespace: The Rook operator and cluster created by `operator.yaml` and `cluster.yaml` (the cluster CRD)
* `/var/lib/rook`: Path on each host in the cluster where configuration is cached by the ceph mons and osds

Note that if you changed the default namespaces or paths such as `dataDirHostPath` in the sample yaml files, you will need to adjust these namespaces and paths throughout these instructions.

If you see issues tearing down the cluster, see the [Troubleshooting](#troubleshooting) section below.

If you are tearing down a cluster frequently for development purposes, it is instead recommended to use an environment such as Minikube that can easily be reset without worrying about any of these steps.

##### Delete the Block and File artifacts

First you will need to clean up the resources created on top of the Rook cluster.

These commands will clean up the resources from the [block](https://rook.io/docs/rook/v1.1/ceph-block.html#teardown) and [file](https://rook.io/docs/rook/v1.1/ceph-filesystem.html#teardown) walkthroughs (unmount volumes, delete volume claims, etc). If you did not complete those parts of the walkthrough, you can skip these instructions:

```console
kubectl delete -f ../wordpress.yaml
kubectl delete -f ../mysql.yaml
kubectl delete -n rook-ceph cephblockpool replicapool
kubectl delete storageclass rook-ceph-block
kubectl delete -f csi/cephfs/kube-registry.yaml
kubectl delete storageclass rook-cephfs
```

##### Delete the CephCluster CRD

After those block and file resources have been cleaned up, you can then delete your Rook cluster. This is important to delete **before removing the Rook operator and agent or else resources may not be cleaned up properly**.

```console
kubectl -n rook-ceph delete cephcluster rook-ceph
```

Verify that the cluster CRD has been deleted before continuing to the next step.

```
kubectl -n rook-ceph get cephcluster
```

##### Delete the Operator and related Resources

This will begin the process of the Rook Ceph operator and all other resources being cleaned up.
This includes related resources such as the agent and discover daemonsets with the following commands:

```console
kubectl delete -f operator.yaml
kubectl delete -f common.yaml
```

##### Delete the data on hosts

IMPORTANT: The final cleanup step requires deleting files on each host in the cluster. All files under the `dataDirHostPath` property specified in the cluster CRD will need to be deleted. Otherwise, inconsistent state will remain when a new cluster is started.

Connect to each machine and delete `/var/lib/rook`, or the path specified by the `dataDirHostPath`.

In the future this step will not be necessary when we build on the K8s local storage feature.

If you modified the demo settings, additional cleanup is up to you for devices, host paths, etc.

Disks on nodes used by Rook for osds can be reset to a usable state with the following methods:

```sh
#!/usr/bin/env bash
DISK="/dev/sdb"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
sgdisk --zap-all $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
rm -rf /dev/ceph-*
```

##### Troubleshooting

If the cleanup instructions are not executed in the order above, or you otherwise have difficulty cleaning up the cluster, here are a few things to try.

The most common issue cleaning up the cluster is that the `rook-ceph` namespace or the cluster CRD remain indefinitely in the `terminating` state. A namespace cannot be removed until all of its resources are removed, so look at which resources are pending termination.

Look at the pods:

```console
kubectl -n rook-ceph get pod
```

If a pod is still terminating, you will need to wait or else attempt to forcefully terminate it (`kubectl delete pod <name>`).

Now look at the cluster CRD:

```console
kubectl -n rook-ceph get cephcluster
```

If the cluster CRD still exists even though you have executed the delete command earlier, see the next section on removing the finalizer.

###### Removing the Cluster CRD Finalizer

When a Cluster CRD is created, a [finalizer](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#finalizers) is added automatically by the Rook operator. 

The finalizer will allow the operator to ensure that before the cluster CRD is deleted, all block and file mounts will be cleaned up. Without proper cleanup, pods consuming the storage will be hung indefinitely until a system reboot.

The operator is responsible for removing the finalizer after the mounts have been cleaned up.
If for some reason the operator is not able to remove the finalizer (ie. the operator is not running anymore), you can delete the finalizer manually with the following command:

```console
kubectl -n rook-ceph patch crd cephclusters.ceph.rook.io --type merge -p '{"metadata":{"finalizers": [null]}}'
```

Within a few seconds you should see that the cluster CRD has been deleted and will no longer block other cleanup such as deleting the `rook-ceph` namespace.

