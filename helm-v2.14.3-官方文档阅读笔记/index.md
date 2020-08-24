# Helm v2.14.3 官方文档阅读笔记


***helm 官方文档地址：<https://helm.sh/docs/>***

The package manager for Kubernetes  
helm 是 kubernetes 的包管理器。

**NOTE**: Kubernetes versions prior to 1.6 have limited or no support for role-based access controls (RBAC).

Helm will figure out where to install Tiller by reading your Kubernetes configuration file (usually `$HOME/.kube/config`). This is the same file that `kubectl` uses.

To find out which cluster Tiller would install to, you can run `kubectl config current-context` or `kubectl cluster-info`.

```shell
[root@k8s-m1 ~]# kubectl config current-context
kubernetes-admin@kubernetes
```

## Understand your Security Context

As with all powerful tools, ensure you are installing it correctly for your scenario.  
与所有强大的工具一样，请确保针对您的情况正确安装了它。

If you’re using Helm on a cluster that you completely control, like minikube or a cluster on a private network in which sharing is not a concern, the default installation – which applies no security configuration – is fine, and it’s definitely the easiest. To install Helm without additional security steps, [install Helm](https://helm.sh/docs/using_helm/#installing-helm) and then [initialize Helm](https://helm.sh/docs/using_helm/#initialize-helm-and-install-tiller).

However, if your cluster is exposed to a larger network or if you share your cluster with others – production clusters fall into this category(生产环境集群属于这一类) – you must take extra steps to secure your installation to prevent careless or malicious(恶意的) actors from damaging the cluster or its data. To apply configurations that secure Helm for use in production environments and other multi-tenant scenarios(多租户场景), see [Securing a Helm installation](https://helm.sh/docs/using_helm/#securing-your-helm-installation)

If your cluster has Role-Based Access Control (RBAC) enabled, you may want to [configure a service account](https://helm.sh/docs/using_helm/#role-based-access-control) and rules before proceeding.

### Role-based Access Control

In Kubernetes, granting a role to an application-specific service account is a best practice to ensure that your application is operating in the scope that you have specified. Read more about service account permissions [in the official Kubernetes docs](https://kubernetes.io/docs/admin/authorization/rbac/#service-account-permissions).

Bitnami also has a fantastic guide for [configuring RBAC in your cluster](https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/) that takes you through RBAC basics.

This guide is for users who want to restrict(限制) Tiller’s capabilities(能力) to install resources to certain namespaces, or to grant a Helm client running access to a Tiller instance.

#### TILLER AND ROLE-BASED ACCESS CONTROL

You can add a service account to Tiller using the `--service-account <NAME>` flag while you’re configuring Helm. As a prerequisite, you’ll have to create a role binding which specifies a [role](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) and a [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) name that have been set up in advance.

Once you have satisfied the pre-requisite and have a service account with the correct permissions, you’ll run a command like this: `helm init --service-account <NAME>`

##### Example: Service account with cluster-admin role

In `rbac-config.yaml`:

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Note: The cluster-admin role is created by default in a Kubernetes cluster, so you don’t have to define it explicitly.

```shell
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding "tiller" created
```

```shell
helm init --service-account tiller --history-max 200
```

##### Example: Deploy Tiller in a namespace, restricted to deploying resources only in that namespace

In the example above, we gave Tiller admin access to the entire cluster. You are not at all required to give Tiller cluster-admin access for it to work. Instead of specifying a ClusterRole or a ClusterRoleBinding, you can specify a Role and RoleBinding to limit Tiller’s scope to a particular namespace.

```shell
$ kubectl create namespace tiller-world
namespace "tiller-world" created
$ kubectl create serviceaccount tiller --namespace tiller-world
serviceaccount "tiller" created
```

Define a Role that allows Tiller to manage all resources in `tiller-world` like in `role-tiller.yaml`:

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-manager
  namespace: tiller-world
rules:
- apiGroups: ["", "batch", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

```shell
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

In `rolebinding-tiller.yaml`,

```shell
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-binding
  namespace: tiller-world
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: tiller-world
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```shell
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```

Afterwards you can run `helm init` to install Tiller in the `tiller-world` namespace.

```shell
$ helm init --service-account tiller --tiller-namespace tiller-world
$HELM_HOME has been configured at /Users/awesome-user/.helm.

Tiller (the Helm server side component) has been installed into your Kubernetes Cluster.

$ helm install stable/lamp --tiller-namespace tiller-world --namespace tiller-world
NAME:   wayfaring-yak
LAST DEPLOYED: Mon Aug  7 16:00:16 2017
NAMESPACE: tiller-world
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod
NAME                  READY  STATUS             RESTARTS  AGE
wayfaring-yak-alpine  0/1    ContainerCreating  0         0s
```

##### Example: Deploy Tiller in a namespace, restricted to deploying resources in another namespace

In the example above, we gave Tiller admin access to the namespace it was deployed inside. Now, let’s limit Tiller’s scope to deploy resources in a different namespace!

For example, let’s install Tiller in the namespace `myorg-system` and allow Tiller to deploy resources in the namespace `myorg-users`.

```shell
$ kubectl create namespace myorg-system
namespace "myorg-system" created
$ kubectl create serviceaccount tiller --namespace myorg-system
serviceaccount "tiller" created
```

Define a Role that allows Tiller to manage all resources in `myorg-users` like in `role-tiller.yaml`:

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-manager
  namespace: myorg-users
rules:
- apiGroups: ["", "batch", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

```shell
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

Bind the service account to that role. In `rolebinding-tiller.yaml`,

```shell
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-binding
  namespace: myorg-users
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: myorg-system
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```shell
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```

We’ll also need to grant Tiller access to read configmaps in myorg-system so it can store release information. In `role-tiller-myorg-system.yaml`:

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: myorg-system
  name: tiller-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["configmaps"]
  verbs: ["*"]
```

```shell
$ kubectl create -f role-tiller-myorg-system.yaml
role "tiller-manager" created
```

And the respective role binding. In `rolebinding-tiller-myorg-system.yaml`:

```shell
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-binding
  namespace: myorg-system
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: myorg-system
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```shell
$ kubectl create -f rolebinding-tiller-myorg-system.yaml
rolebinding "tiller-binding" created
```

#### HELM AND ROLE-BASED ACCESS CONTROL

When running a Helm client in a pod, in order for the Helm client to talk to a Tiller instance, it will need certain privileges to be granted. Specifically, the Helm client will need to be able to create pods, forward ports and be able to list pods in the namespace where Tiller is running (so it can find Tiller).

##### Example: Deploy Helm in a namespace, talking to Tiller in another namespace

In this example, we will assume Tiller is running in a namespace called `tiller-world` and that the Helm client is running in a namespace called `helm-world`. By default, Tiller is running in the `kube-system` namespace.

In `helm-user.yaml`:

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm
  namespace: helm-world
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tiller-user
  namespace: tiller-world
rules:
- apiGroups:
  - ""
  resources:
  - pods/portforward
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tiller-user-binding
  namespace: tiller-world
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tiller-user
subjects:
- kind: ServiceAccount
  name: helm
  namespace: helm-world
```

```shell
$ kubectl create -f helm-user.yaml
serviceaccount "helm" created
role "tiller-user" created
rolebinding "tiller-user-binding" created
```

## INSTALL HELM

Download a binary release of the Helm client. You can use tools like `homebrew`, or look at [the official releases page](https://github.com/helm/helm/releases).

For more details, or for other options, see [the installation guide](https://helm.sh/docs/using_helm/#installing-helm).

There are two parts to Helm: The Helm client (`helm`) and the Helm server (`Tiller`). This guide shows how to install the client, and then proceeds to show two ways to install the server.

IMPORTANT: If you are responsible for ensuring your cluster is a controlled environment, especially when resources are shared, it is strongly recommended installing Tiller using a secured configuration. For guidance, see [Securing your Helm Installation](https://helm.sh/docs/using_helm/#securing-your-helm-installation).

### INSTALLING THE HELM CLIENT

Every [release](https://github.com/helm/helm/releases) of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.

1. Download your [desired version](https://github.com/helm/helm/releases)
2. Unpack it (`tar -zxvf helm-v2.0.0-linux-amd64.tgz`)
3. Find the `helm` binary in the unpacked directory, and move it to its desired destination (`mv linux-amd64/helm /usr/local/bin/helm`)

From there, you should be able to run the client: `helm help`.

### INSTALLING TILLER

Tiller, the server portion of Helm, typically runs inside of your Kubernetes cluster. But for development, it can also be run locally, and configured to talk to a remote Kubernetes cluster

**Special Note for RBAC Users:**

Most cloud providers enable a feature called Role-Based Access Control - RBAC for short. If your cloud provider enables this feature, you will need to create a service account for Tiller with the right roles and permissions to access resources.

Check the [Kubernetes Distribution Guide](https://helm.sh/docs/using_helm/##kubernetes-distribution-guide) to see if there’s any further points of interest on using Helm with your cloud provider. Also check out the guide on [Tiller and Role-Based Access Control](https://helm.sh/docs/using_helm/#role-based-access-control) for more information on how to run Tiller in an RBAC-enabled Kubernetes cluster.

#### Easy In-Cluster Installation

The easiest way to install `tiller` into the cluster is simply to run `helm init`. This will validate that `helm`’s local environment is set up correctly (and set it up if necessary). Then it will connect to whatever cluster `kubectl` connects to by default (`kubectl config view`). Once it connects, it will install `tiller` into the `kube-system` namespace.

After `helm init`, you should be able to run `kubectl get pods --namespace kube-system` and see Tiller running.

You can explicitly tell `helm init` to…

- Install the canary(金丝雀) build with the `--canary-image` flag
- Install a particular image (version) with `--tiller-image`
- Install to a particular cluster with `--kube-context`
- Install into a particular namespace with `--tiller-namespace`
- Install Tiller with a Service Account with `--service-account` (for [RBAC enabled clusters](https://helm.sh/docs/using_helm/#rbac))
- Install Tiller without mounting a service account with `--automount-service-account false`

Once Tiller is installed, running `helm version` should show you both the client and server version. (If it shows only the client version, `helm` cannot yet connect to the server. Use `kubectl` to see if any `tiller` pods are running.)

Helm will look for Tiller in the `kube-system` namespace unless `--tiller-namespace` or `TILLER_NAMESPACE` is set.

#### Installing Tiller Canary Builds

Canary images are built from the `master` branch. They may not be stable, but they offer you the chance to test out the latest features.

The easiest way to install a canary image is to use `helm init` with the `--canary-image` flag:

```shell
helm init --canary-image
```

This will use the most recently built container image. You can always uninstall Tiller by deleting the Tiller deployment from the `kube-system` namespace using `kubectl`.

#### Running Tiller Locally

For development, it is sometimes easier to work on Tiller locally, and configure it to connect to a remote Kubernetes cluster.

The process of building Tiller is explained above.

Once `tiller` has been built, simply start it:

```shell
$ bin/tiller
Tiller running on :44134
```

When Tiller is running locally, it will attempt to connect to the Kubernetes cluster that is configured by `kubectl`. (Run `kubectl config view` to see which cluster that is.)

You must tell `helm` to connect to this new local Tiller host instead of connecting to the one in-cluster. There are two ways to do this. The first is to specify the `--host` option on the command line. The second is to set the `$HELM_HOST` environment variable.

```shell
export HELM_HOST=localhost:44134
```

```shell
$ helm version # Should connect to localhost.
Client: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"db...", GitTreeState:"dirty"}
Server: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"a5...", GitTreeState:"dirty"}
```

Importantly, even when running locally, Tiller will store release configuration in ConfigMaps inside of Kubernetes.

### UPGRADING TILLER

As of Helm 2.2.0, Tiller can be upgraded using `helm init --upgrade`.

For older versions of Helm, or for manual upgrades, you can use `kubectl` to modify the Tiller image:

```shell
export TILLER_TAG=v2.0.0-beta.1        # Or whatever version you want
```

```shell
$ kubectl --namespace=kube-system set image deployments/tiller-deploy tiller=gcr.io/kubernetes-helm/tiller:$TILLER_TAG
deployment "tiller-deploy" image updated
```

Setting TILLER_TAG=canary will get the latest snapshot of master.

### DELETING OR REINSTALLING TILLER

Because Tiller stores its data in Kubernetes ConfigMaps, you can safely delete and re-install Tiller without worrying about losing any data. The recommended way of deleting Tiller is with `kubectl delete deployment tiller-deploy --namespace kube-system`, or more concisely `helm reset`.

Tiller can then be re-installed from the client with:

```shell
helm init
```

### INITIALIZE HELM AND INSTALL TILLER

Once you have Helm ready, you can initialize the local CLI and also install Tiller into your Kubernetes cluster in one step:

```shell
helm init --history-max 200
```

TIP: Setting `--history-max` on helm init is recommended as configmaps and other objects in helm history can grow large in number if not purged by max limit. Without a max history set the history is kept indefinitely, leaving a large number of records for helm and tiller to maintain.

This will install Tiller into the Kubernetes cluster you saw with `kubectl config current-context`.

TIP: Want to install into a different cluster? Use the `--kube-context` flag.

TIP: When you want to upgrade Tiller, just run `helm init --upgrade`.

By default, when Tiller is installed, it does not have authentication enabled. To learn more about configuring strong TLS authentication for Tiller, consult [the Tiller TLS guide](https://helm.sh/docs/using_helm/#using-ssl-between-helm-and-tiller).

#### INSTALL AN EXAMPLE CHART

To install a chart, you can run the `helm install` command. Helm has several ways to find and install a chart, but the easiest is to use one of the official `stable` charts.

```shell
helm repo update              # Make sure we get the latest list of charts
```

```shell
$ helm install stable/mysql
NAME:   wintering-rodent
LAST DEPLOYED: Thu Oct 18 14:21:18 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                    AGE
wintering-rodent-mysql  0s

==> v1/ConfigMap
wintering-rodent-mysql-test  0s

==> v1/PersistentVolumeClaim
wintering-rodent-mysql  0s

==> v1/Service
wintering-rodent-mysql  0s

==> v1beta1/Deployment
wintering-rodent-mysql  0s

==> v1/Pod(related)

NAME                                    READY  STATUS   RESTARTS  AGE
wintering-rodent-mysql-6986fd6fb-988x7  0/1    Pending  0         0s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
wintering-rodent-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default wintering-rodent-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:

    mysql -h wintering-rodent-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/wintering-rodent-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

In the example above, the `stable/mysql` chart was released, and the name of our new release is `wintering-rodent`. You get a simple idea of the features of this MySQL chart by running `helm inspect stable/mysql`.

Whenever you install a chart, a new release is created. So one chart can be installed multiple times into the same cluster. And each can be independently managed and upgraded.

The `helm install` command is a very powerful command with many capabilities. To learn more about it, check out the [Using Helm Guide](https://helm.sh/docs/using_helm/#using-helm)

## Using Helm

This guide explains the basics of using Helm (and Tiller) to manage packages on your Kubernetes cluster. It assumes that you have already [installed](https://helm.sh/docs/using_helm/#installing-helm) the Helm client and the Tiller server (typically by `helm init`).

### THREE BIG CONCEPTS

A ***Chart*** is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. Think of it like the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file.

A ***Repository*** is the place where charts can be collected and shared. It’s like [Perl’s CPAN](https://www.cpan.org/) archive or the [Fedora Package Database](https://apps.fedoraproject.org/packages/s/pkgdb), but for Kubernetes packages.

A ***Release*** is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster. And each time it is installed, a new ***release*** is created. Consider a MySQL chart. If you want two databases running in your cluster, you can install that chart twice. Each one will have its own ***release***, which will in turn have its own ***release name***.

With these concepts in mind, we can now explain Helm like this:

Helm installs ***charts*** into Kubernetes, creating a new ***release*** for each installation. And to find new charts, you can search Helm chart ***repositories***.

### ‘HELM SEARCH’: FINDING CHARTS

When you first install Helm, it is preconfigured to talk to the official Kubernetes charts repository. This repository contains a number of carefully curated and maintained charts. This chart repository is named `stable` by default.

You can see which charts are available by running `helm search`:

```shell
$ helm search
NAME                VERSION     DESCRIPTION
stable/drupal       0.3.2       One of the most versatile open source content m...
stable/jenkins      0.1.0       A Jenkins Helm chart for Kubernetes.
stable/mariadb      0.5.1       Chart for MariaDB
stable/mysql        0.1.0       Chart for MySQL
...
```

With no filter, `helm search` shows you all of the available charts. You can narrow down your results by searching with a filter:

```shell
$ helm search mysql
NAME            VERSION DESCRIPTION
stable/mysql    0.1.0   Chart for MySQL
stable/mariadb  0.5.1   Chart for MariaDB
```

Now you will only see the results that match your filter.

Why is `mariadb` in the list? Because its package description relates it to MySQL. We can use `helm inspect chart` to see this:

```shell
$ helm inspect stable/mariadb
Fetched stable/mariadb to mariadb-0.5.1.tgz
description: Chart for MariaDB
engine: gotpl
home: https://mariadb.org
keywords:
  - mariadb
  - mysql
  - database
  - sql
...
```

Search is a good way to find available packages. Once you have found a package you want to install, you can use `helm install` to install it.

### ‘HELM INSTALL’: INSTALLING A PACKAGE

To install a new package, use the `helm install` command. At its simplest, it takes only one argument: The name of the chart.

```shell
$ helm install stable/mariadb
Fetched stable/mariadb-0.3.0 to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
NAME: happy-panda
LAST DEPLOYED: Wed Sep 28 12:32:28 2016
NAMESPACE: default
STATUS: DEPLOYED

Resources:
==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         0         0            0           1s

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         1s

==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   1s


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

Now the `mariadb` chart is installed. Note that installing a chart creates a new ***release*** object. The release above is named `happy-panda`. (If you want to use your own release name, simply use the `--name` flag on `helm install`.)

During installation, the `helm` client will print useful information about which resources were created, what the state of the release is, and also whether there are additional configuration steps you can or should take.

Helm does not wait until all of the resources are running before it exits. Many charts require Docker images that are over 600M in size, and may take a long time to install into the cluster.

To keep track of a release’s state, or to re-read configuration information, you can use `helm status`:

```shell
$ helm status happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   4m

==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         1         1            1           4m

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         4m


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

The above shows the current state of your release.

#### Customizing the Chart Before Installing

Installing the way we have here will only use the default configuration options for this chart. Many times, you will want to customize the chart to use your preferred configuration.

To see what options are configurable on a chart, use `helm inspect values`:

```shell
helm inspect values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: https://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
```

You can then override any of these settings in a YAML formatted file, and then pass that file during installation.

```shell
$ cat << EOF > config.yaml
mariadbUser: user0
mariadbDatabase: user0db
EOF
```

```shell
helm install -f config.yaml stable/mariadb
```

The above will create a default MariaDB user with the name `user0`, and grant this user access to a newly created `user0db` database, but will accept all the rest of the defaults for that chart.

There are two ways to pass configuration data during install:

- `--values` (or `-f`): Specify a YAML file with overrides. This can be specified multiple times and the rightmost file will take precedence
- `--set` (and its variants `--set-string` and `--set-file`): Specify overrides on the command line.

If both are used, `--set` values are merged into `--values` with higher precedence. Overrides specified with `--set` are persisted in a configmap. Values that have been `--set` can be viewed for a given release with `helm get values <release-name>`. Values that have been `--set` can be cleared by running `helm upgrade` with `--reset-values` specified.

**The Format and Limitations of `--set`**

The `--set` option takes zero or more name/value pairs. At its simplest, it is used like this: --set name=value. The YAML equivalent of that is:

```yaml
name: value
```

Multiple values are separated by `,` characters. So `--set a=b,c=d` becomes:

```yaml
a: b
c: d
```

More complex expressions are supported. For example, `--set outer.inner=value` is translated into this:

```shell
outer:
  inner: value
```

Lists can be expressed by enclosing values in { and }. For example, `--set name={a, b, c}` translates to:

```shell
name:
  - a
  - b
  - c
```

As of Helm 2.5.0, it is possible to access list items using an array index syntax. For example, --set servers[0].port=80 becomes:

```shell
servers:
  - port: 80
    host: example
```

Sometimes you need to use special characters in your `--set` lines. You can use a backslash to escape the characters; `--set name="value1\,value2"` will become:

```shell
name: "value1,value2"
```

Similarly, you can escape dot sequences as well(类似地，你同样可以使用点来转义序列), which may come in handy when charts use the toYaml function to parse annotations, labels and node selectors. The syntax for `--set nodeSelector."kubernetes\.io/role"=master` becomes:

```shell
nodeSelector:
  kubernetes.io/role: master
```

Deeply nested data structures(多层嵌套的数据结构) can be difficult to express using `--set`. Chart designers are encouraged to consider the `--set` usage when designing the format of a `values.yaml` file.

Helm will cast certain values specified with `--set` to integers. For example, `--set foo=true` results Helm to cast `true` into an int64 value. In case you want a string, use a `--set`’s variant named `--set-string`. `--set-string foo=true` results in a string value of `"true"`.

`--set-file key=filepath` is another variant of `--set`. It reads the file and use its content as a value. An example use case of it is to inject a multi-line text into values without dealing with indentation(缩进) in YAML. Say you want to create a [brigade](https://github.com/Azure/brigade) project with certain value containing 5 lines JavaScript code, you might write a `values.yaml` like:

```JavaScript
defaultScript: |
  const { events, Job } = require("brigadier")
  function run(e, project) {
    console.log("hello default script")
  }
  events.on("run", run)
```

Being embedded in a YAML, this makes it harder for you to use IDE features and testing framework and so on that supports writing code. Instead, you can use --set-file defaultScript=brigade.js with brigade.js containing:

```JavaScript
const { events, Job } = require("brigadier")
function run(e, project) {
  console.log("hello default script")
}
events.on("run", run)
```

#### More Installation Methods

The `helm install` command can install from several sources:

- A chart repository (as we’ve seen above)
- A local chart archive (`helm install foo-0.1.1.tgz`)
- An unpacked chart directory (`helm install path/to/foo`)
- A full URL (`helm install https://example.com/charts/foo-1.2.3.tgz`)

### ‘HELM UPGRADE’ AND ‘HELM ROLLBACK’: UPGRADING A RELEASE, AND RECOVERING ON FAILURE

When a new version of a chart is released, or when you want to change the configuration of your release, you can use the `helm upgrade` command.

An upgrade takes an existing release and upgrades it according to the information you provide. Because Kubernetes charts can be large and complex, Helm tries to perform the least invasive upgrade. It will only update things that have changed since the last release.

```shell
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded.
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

In the above case, the `happy-panda` release is upgraded with the same chart, but with a new YAML file:

```shell
mariadbUser: user1
```

We can use `helm get values` to see whether that new setting took effect.

```shell
$ helm get values happy-panda
mariadbUser: user1
```

The `helm get` command is a useful tool for looking at a release in the cluster. And as we can see above, it shows that our new values from `panda.yaml` were deployed to the cluster.

Now, if something does not go as planned during a release, it is easy to roll back to a previous release using `helm rollback [RELEASE] [REVISION]`.

```shell
helm rollback happy-panda 1
```

The above rolls back our happy-panda to its very first release version. A release version is an incremental revision. Every time an install, upgrade, or rollback happens, the revision number is incremented by 1. The first revision number is always 1. And we can use `helm history [RELEASE]` to see revision numbers for a certain release.

### HELPFUL OPTIONS FOR INSTALL/UPGRADE/ROLLBACK

There are several other helpful options you can specify for customizing the behavior of Helm during an install/upgrade/rollback. Please note that this is not a full list of cli flags. To see a description of all flags, just run `helm <command> --help`.

- `--timeout`: A value in seconds to wait for Kubernetes commands to complete This defaults to 300 (5 minutes)
- `--wait`: Waits until all Pods are in a ready state, PVCs are bound, Deployments have minimum (`Desired` minus `maxUnavailable`) Pods in ready state and Services have an IP address (and Ingress if a `LoadBalancer`) before marking the release as successful. It will wait for as long as the `--timeout` value. If timeout is reached, the release will be marked as `FAILED`. Note: In scenario where Deployment has `replicas` set to 1 and `maxUnavailable` is not set to 0 as part of rolling update strategy, `--wait` will return as ready as it has satisfied the minimum Pod in ready condition.
- `--no-hooks`: This skips running hooks for the command
- `--recreate-pods` (only available for `upgrade` and `rollback`): This flag will cause all pods to be recreated (with the exception of pods belonging to deployments)

### ‘HELM DELETE’: DELETING A RELEASE

When it is time to uninstall or delete a release from the cluster, use the `helm delete` command:

```shell
helm delete happy-panda
```

使用时加上 `--purge` 参数，会彻底删除这个 release

```shell
--purge   remove the release from the store and make its name free for later use
```

This will remove the release from the cluster. You can see all of your currently deployed releases with the helm list command:

```shell
$ helm list
NAME            VERSION   UPDATED                         STATUS          CHART
inky-cat        1         Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
```

From the output above, we can see that the happy-panda release was deleted.

However, Helm always keeps records of what releases happened. Need to see the deleted releases? `helm list --deleted` shows those, and `helm list --all` shows all of the releases (deleted and currently deployed, as well as releases that failed):

```shell
$  helm list --all
NAME            VERSION   UPDATED                         STATUS          CHART
happy-panda     2         Wed Sep 28 12:47:54 2016        DELETED         mariadb-0.3.0
inky-cat        1         Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
kindred-angelf  2         Tue Sep 27 16:16:10 2016        DELETED         alpine-0.1.0
```

Because Helm keeps records of deleted releases, a release name cannot be re-used. (If you *really* need to re-use a release name, you can use the `--replace` flag, but it will simply re-use the existing release and replace its resources.)

Note that because releases are preserved in this way, you can rollback a deleted resource, and have it re-activate.

### ‘HELM REPO’: WORKING WITH REPOSITORIES

So far, we’ve been installing charts only from the `stable` repository. But you can configure `helm` to use other repositories. Helm provides several repository tools under the `helm repo` command.

You can see which repositories are configured using `helm repo list`:

```shell
$ helm repo list
NAME            URL
stable          https://kubernetes-charts.storage.googleapis.com
local           http://localhost:8879/charts
mumoshu         https://mumoshu.github.io/charts
```

And new repositories can be added with `helm repo add`:

```shell
helm repo add dev https://example.com/dev-charts
```

Because chart repositories change frequently, at any point you can make sure your Helm client is up to date by running `helm repo update`.

### CREATING YOUR OWN CHARTS

The [Chart Development Guide](https://helm.sh/docs/developing_charts/) explains how to develop your own charts. But you can get started quickly by using the `helm create` command:

```shell
$ helm create deis-workflow
Creating deis-workflow
```

Now there is a chart in `./deis-workflow`. You can edit it and create your own templates.

As you edit your chart, you can validate that it is well-formatted by running `helm lint`.

When it’s time to package the chart up for distribution, you can run the `helm package` command:

```shell
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

And that chart can now easily be installed by `helm install`:

```shell
$ helm install ./deis-workflow-0.1.0.tgz
...
```

Charts that are archived can be loaded into chart repositories. See the documentation for your chart repository server to learn how to upload.

Note: The `stable` repository is managed on the [Helm Charts GitHub repository](https://github.com/helm/charts). That project accepts chart source code, and (after audit) packages those for you.

### TILLER, NAMESPACES AND RBAC

In some cases you may wish to scope Tiller or deploy multiple Tillers to a single cluster. Here are some best practices when operating in those circumstances.

1. Tiller can be [installed](https://helm.sh/docs/using_helm/#installing-helm) into any namespace. By default, it is installed into kube-system. You can run multiple Tillers provided they each run in their own namespace.
2. Limiting Tiller to only be able to install into specific namespaces and/or resource types is controlled by Kubernetes [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) roles and rolebindings. You can add a service account to Tiller when configuring Helm via `helm init --service-account <NAME>`. You can find more information about that [here](https://helm.sh/docs/using_helm/#role-based-access-control).
3. Release names are unique PER TILLER INSTANCE.
4. Charts should only contain resources that exist in a single namespace.
5. It is not recommended to have multiple Tillers configured to manage resources in the same namespace.

### CONCLUSION

This chapter has covered the `basic` usage patterns of the helm client, including searching, installation, upgrading, and deleting. It has also covered useful utility commands like `helm status`, `helm get`, and `helm repo`.

For more information on these commands, take a look at Helm’s built-in help: `helm help`.

In the next chapter, we look at the process of developing charts.

## Charts

Helm uses a packaging format called ***charts***. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

Charts are created as files laid out in a particular directory tree, then they can be packaged into versioned archives to be deployed.

This document explains the chart format, and provides basic guidance for building charts with Helm.

### THE CHART FILE STRUCTURE

A chart is organized as a collection of files inside of a directory. The directory name is the name of the chart (without versioning information). Thus, a chart describing WordPress would be stored in the `wordpress/` directory.

Inside of this directory, Helm will expect a structure that matches this:

```shell
wordpress/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: A YAML file listing dependencies for the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

Helm reserves use of the `charts/` and `templates/` directories, and of the listed file names. Other files will be left as they are.

#### THE CHART.YAML FILE

The `Chart.yaml` file is required for a chart. It contains the following fields:

```yaml
apiVersion: The chart API version, always "v1" (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
engine: gotpl # The name of the template engine (optional, defaults to gotpl)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)
tillerVersion: The version of Tiller that this chart requires. This should be expressed as a SemVer range: ">2.0.0" (optional)
```

If you are familiar with the `Chart.yaml` file format for Helm Classic, you will notice that fields specifying dependencies have been removed. That is because the new Chart format expresses dependencies using the `charts/` directory.

Other fields will be silently ignored.

##### Charts and Versioning

Every chart must have a version number. A version must follow the [SemVer 2](https://semver.org/) standard. Unlike Helm Classic, Kubernetes Helm uses version numbers as release markers. Packages in repositories are identified by name plus version.

For example, an `nginx` chart whose version field is set to `version: 1.2.3` will be named:

```shell
nginx-1.2.3.tgz
```

More complex SemVer 2 names are also supported, such as `version: 1.2.3-alpha.1+ef365`. But non-SemVer names are explicitly disallowed by the system.

**NOTE**: Whereas Helm Classic and Deployment Manager were both very GitHub oriented when it came to charts, Kubernetes Helm does not rely upon or require GitHub or even Git. Consequently, it does not use Git SHAs for versioning at all.

The `version` field inside of the `Chart.yaml` is used by many of the Helm tools, including the CLI and the Tiller server. When generating a package, the `helm package` command will use the version that it finds in the `Chart.yaml` as a token in the package name. The system assumes that the version number in the chart package name matches the version number in the `Chart.yaml`. Failure to meet this assumption will cause an error.

##### The appVersion field

Note that the `appVersion` field is not related to the `version` field. It is a way of specifying the version of the application. For example, the `drupal` chart may have an `appVersion: 8.2.1`, indicating that the version of Drupal included in the chart (by default) is `8.2.1`. This field is informational, and has no impact on chart version calculations.

##### Deprecating Charts

When managing charts in a Chart Repository, it is sometimes necessary to deprecate a chart. The optional #deprecated# field in #Chart.yaml# can be used to mark a chart as deprecated. If the ##latest## version of a chart in the repository is marked as deprecated, then the chart as a whole is considered to be deprecated. The chart name can later be reused by publishing a newer version that is not marked as deprecated. The workflow for deprecating charts, as followed by the [helm/charts](https://github.com/helm/charts) project is:

- Update chart’s Chart.yaml to mark the chart as deprecated, bumping the version
- Release the new chart version in the Chart Repository
- Remove the chart from the source repository (e.g. git)

#### CHART LICENSE, README AND NOTES

Charts can also contain files that describe the installation, configuration, usage and license of a chart.

A **LICENSE** is a plain text file containing the [license](https://en.wikipedia.org/wiki/Software_license) for the chart. The chart can contain a license as it may have programming logic in the templates and would therefore not be configuration only. There can also be separate license(s) for the application installed by the chart, if required.

A **README** for a chart should be formatted in Markdown (README.md), and should generally contain:

- A description of the application or service the chart provides
- Any prerequisites or requirements to run the chart
- Descriptions of options in `values.yaml` and default values
- Any other information that may be relevant to the installation or configuration of the chart

The chart can also contain a short plain text `templates/NOTES.txt` file that will be printed out after installation, and when viewing the status of a release. This file is evaluated as a [template](https://helm.sh/docs/developing_charts/#templates-and-values), and can be used to display usage notes, next steps, or any other information relevant to a release of the chart. For example, instructions could be provided for connecting to a database, or accessing a web UI. Since this file is printed to STDOUT when running `helm install` or `helm status`, it is recommended to keep the content brief and point to the **README** for greater detail.

#### CHART DEPENDENCIES

In Helm, one chart may depend on any number of other charts. These dependencies can be dynamically linked through the `requirements.yaml` file or brought in to the `charts/` directory and managed manually.

Although manually managing your dependencies has a few advantages some teams need, the preferred method of declaring dependencies is by using a `requirements.yaml` file inside of your chart.

**Note**: The `dependencies:` section of the `Chart.yaml` from Helm Classic has been completely removed.

##### Managing Dependencies with `requirements.yaml`

A `requirements.yaml` file is a simple file for listing your dependencies.

```shell
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

- The `name` field is the name of the chart you want.
- The `version` field is the version of the chart you want.
- The `repository` field is the full URL to the chart repository. Note that you must also use `helm repo add` to add that repo locally.

Once you have a dependencies file, you can run `helm dependency update` and it will use your dependency file to download all the specified charts into your `charts/` directory for you.

```shell
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete.
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

When `helm dependency update` retrieves charts, it will store them as chart archives in the `charts/` directory. So for the example above, one would expect to see the following files in the charts directory:

```shell
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

Managing charts with `requirements.yaml` is a good way to easily keep charts updated, and also share requirements information throughout a team.

##### Alias field in requirements.yaml

In addition to the other fields above, each requirements entry may contain the optional field `alias`.

Adding an alias for a dependency chart would put a chart in dependencies using alias as name of new dependency.

One can use `alias` in cases where they need to access a chart with other name(s).
如果他们需要访问具有其他名字的 chart ，可以使用 alias 。

```shell
# parentchart/requirements.yaml

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

In the above example we will get 3 dependencies in all for `parentchart`

```shell
subchart
new-subchart-1
new-subchart-2
```

The manual way of achieving this is by copy/pasting the same chart in the `charts/` directory multiple times with different names.

##### Tags and Condition fields in requirements.yaml

In addition to the other fields above, each requirements entry may contain the optional fields `tags` and `condition`.

All charts are loaded by default. If `tags` or `condition` fields are present, they will be evaluated(被评估) and used to control loading for the chart(s) they are applied to.

**Condition** - The condition field holds one or more YAML paths (delimited by commas (以逗号分隔)). If this path exists in the parent’s values and resolves to a boolean value, the chart will be enabled or disabled based on that boolean value. Only the first valid path found in the list is evaluated and if no paths exist then the condition has no effect. For multiple level dependencies the condition is prependend by the path to the parent chart.

**Tags** - The tags field is a YAML list of labels to associate with this chart. In the top parent’s values, all charts with tags can be enabled or disabled by specifying the tag and a boolean value.

```shell
# parentchart/requirements.yaml

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled
    tags:
      - front-end
      - subchart1

  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```shell
# subchart2/requirements.yaml

dependencies:
  - name: subsubchart
    repository: http://localhost:10191
    version: 0.1.0
    condition: subsubchart.enabled
```

```shell
# parentchart/values.yaml

subchart1:
  enabled: true
subchart2:
  subsubchart:
    enabled: false
tags:
  front-end: false
  back-end: true
```

In the above example all charts with the tag `front-end` would be disabled but since the `subchart1.enabled` path evaluates to ‘true’ in the parent’s values, the condition will override the `front-end` tag and `subchart1` will be enabled.

Since `subchart2` is tagged with `back-end` and that tag evaluates to `true`, `subchart2` will be enabled. Also note that although `subchart2` has a condition specified in `requirements.yaml`, there is no corresponding path and value in the parent’s values so that condition has no effect.

`subsubchart` is disabled by default but can be enabled by setting `subchart2.subsubchart.enabled=true`. Hint: disabling `subchart2` via tag will also disable all sub-charts (even if overriding the value `subchart2.subsubchart.enabled=true`).

**Using the CLI with Tags and Conditions:**

The `--set` parameter can be used as usual to alter tag and condition values.

```shell
helm install --set tags.front-end=true --set subchart2.enabled=false
```

**Tags and Condition Resolution:**

- Conditions (when set in values) always override tags.
- The first condition path that exists wins and subsequent ones for that chart are ignored.
- Tags are evaluated as ‘if any of the chart’s tags are true then enable the chart’.
- Tags and conditions values must be set in the top parent’s values.
- The `tags:` key in values must be a top level key. Globals and nested `tags:` tables are not currently supported.

##### Importing Child Values via requirements.yaml

In some cases it is desirable to allow a child chart’s values to propagate(传播) to the parent chart and be shared as common defaults. An additional benefit of using the `exports` format is that it will enable future tooling to introspect user-settable values.

The keys containing the values to be imported can be specified in the parent chart’s `requirements.yaml` file using a YAML list. Each item in the list is a key which is imported from the child chart’s `exports` field.

To import values not contained in the `exports` key, use the [child-parent](https://helm.sh/docs/developing_charts/#using-the-child-parent-format) format. Examples of both formats are described below.

###### Using the exports format

If a child chart’s `values.yaml` file contains an `exports` field at the root, its contents may be imported directly into the parent’s values by specifying the keys to import as in the example below:

```shell
# parent's requirements.yaml file

    ...
    import-values:
      - data
```

```shell
# child's values.yaml file

...
exports:
  data:
    myint: 99
```

Since we are specifying the key `data` in our import list, Helm looks in the `exports` field of the child chart for `data` key and imports its contents.

The final parent values would contain our exported field:

```shell
# parent's values file

...
myint: 99
```

Please note the parent key `data` is not contained in the parent’s final values. If you need to specify the parent key, use the `‘child-parent’ format`.

###### Using the child-parent format

To access values that are not contained in the `exports` key of the child chart’s values, you will need to specify the source key of the values to be imported (`child`) and the destination path in the parent chart’s values (`parent`).

The `import-values` in the example below instructs Helm to take any values found at `child:` path and copy them to the parent’s values at the path specified in `parent`:

```shell
# parent's requirements.yaml file

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

In the above example, values found at `default.data` in the subchart1’s values will be imported to the `myimports` key in the parent chart’s values as detailed below:

```shell
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```shell
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

The parent chart’s resulting values would be:

```shell
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

The parent’s final values now contains the `myint` and `mybool` fields imported from subchart1.

##### Managing Dependencies manually via the `charts/` directory

If more control over dependencies is desired, these dependencies can be expressed explicitly by copying the dependency charts into the `charts/` directory.

A dependency can be either a chart archive (`foo-1.2.3.tgz`) or an unpacked chart directory. But its name cannot start with `_` or `.`. Such files are ignored by the chart loader.

For example, if the WordPress chart depends on the Apache chart, the Apache chart (of the correct version) is supplied in the WordPress chart’s `charts/` directory:

```shell
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

The example above shows how the WordPress chart expresses its dependency on Apache and MySQL by including those charts inside of its `charts/` directory.

**TIP**: To drop a dependency into your `charts/` directory, use the `helm fetch` command

##### Operational aspects of using dependencies

The above sections explain how to specify chart dependencies, but how does this affect chart installation using `helm install` and `helm upgrade`?

Suppose that a chart named “A” creates the following Kubernetes objects

- namespace “A-Namespace”
- statefulset “A-StatefulSet”
- service “A-Service”

Furthermore, A is dependent on chart B that creates objects

- namespace “B-Namespace”
- replicaset “B-ReplicaSet”
- service “B-Service”

After installation/upgrade of chart A, a single Helm release is created/modified. The release will create/update all of the above Kubernetes objects in the following order:

- A-Namespace
- B-Namespace
- A-StatefulSet
- B-ReplicaSet
- A-Service
- B-Service

This is because when Helm installs/upgrades charts, the Kubernetes objects from the charts and all its dependencies are

- aggregated into a single set; then
- sorted by type followed by name; and then
- created/updated in that order.

Hence(因此) a single release is created with all the objects for the chart and its dependencies.

The install order of Kubernetes types is given by the enumeration(列举) InstallOrder in kind_sorter.go (see [the Helm source file](https://github.com/helm/helm/blob/master/pkg/tiller/kind_sorter.go#L26)).

#### TEMPLATES AND VALUES

Helm Chart templates are written in the [Go template language](https://golang.org/pkg/text/template/), with the addition of 50 or so add-on template functions [from the Sprig library](https://github.com/Masterminds/sprig) and a few other [specialized functions](https://helm.sh/docs/developing_charts/#chart-development-tips-and-tricks).

All template files are stored in a chart’s `templates/` folder. When Helm renders(渲染) the charts, it will pass every file in that directory through the template engine.

Values for the templates are supplied two ways:

- Chart developers may supply a file called `values.yaml` inside of a chart. This file can contain default values.
- Chart users may supply a YAML file that contains values. This can be provided on the command line with `helm install`.

When a user supplies custom values, these values will override the values in the chart’s `values.yaml` file.

##### Template Files

Template files follow the standard conventions for writing Go templates (see the [text/template Go package documentation](https://golang.org/pkg/text/template/) for details). An example template file might look something like this:

```shell
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

The above example, based loosely on <https://github.com/deis/charts>, is a template for a Kubernetes replication controller. It can use the following four template values (usually defined in a `values.yaml` file):

- `imageRegistry`: The source registry for the Docker image.
- `dockerTag`: The tag for the docker image.
- `pullPolicy`: The Kubernetes pull policy.
- `storage`: The storage backend, whose default is set to `"minio"`

All of these values are defined by the template author. Helm does not require or dictate parameters.

To see many working charts, check out the [Helm Charts project](https://github.com/helm/charts)

###### Predefined Values

Values that are supplied via a `values.yaml` file (or via the `--set` flag) are accessible from the `.Values` object in a template. But there are other pre-defined pieces of data you can access in your templates.

The following values are pre-defined, are available to every template, and cannot be overridden. As with all values, the names are ***case sensitive***.

- `Release.Name`: The name of the release (not the chart)
- `Release.Time`: The time the chart release was last updated. This will match the `Last Released` time on a Release object.
- `Release.Namespace`: The namespace the chart was released to.
- `Release.Service`: The service that conducted the release. Usually this is `Tiller`.
- `Release.IsUpgrade`: This is set to true if the current operation is an upgrade or rollback.
- `Release.IsInstall`: This is set to true if the current operation is an install.
- `Release.Revision`: The revision number. It begins at 1, and increments with each `helm upgrade`.
- `Chart`: The contents of the `Chart.yaml`. Thus, the chart version is obtainable as `Chart.Version` and the maintainers are in `Chart.Maintainers`.
- `Files`: A map-like object containing all non-special files in the chart. This will not give you access to templates, but will give you access to additional files that are present (unless they are excluded using `.helmignore`). Files can be accessed using `{{index .Files "file.name"}}` or using the `{{.Files.Get name}}` or `{{.Files.GetString name}}` functions. You can also access the contents of the file as `[]byte` using `{{.Files.GetBytes}}`
- `Capabilities`: A map-like object that contains information about the versions of Kubernetes (`{{.Capabilities.KubeVersion}}`, Tiller (`{{.Capabilities.TillerVersion}}`, and the supported Kubernetes API versions (`{{.Capabilities.APIVersions.Has "batch/v1"`)

**NOTE**: Any unknown Chart.yaml fields will be dropped. They will not be accessible inside of the Chart object. Thus, Chart.yaml cannot be used to pass arbitrarily structured data(自定义结构的数据) into the template. The values file can be used for that, though.

##### Values files

Considering the template in the previous section, a `values.yaml` file that supplies the necessary values would look like this:

```shell
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

A values file is formatted in YAML. A chart may include a default `values.yaml` file. The Helm install command allows a user to override values by supplying additional YAML values:

```shell
helm install --values=myvals.yaml wordpress
```

When values are passed in this way, they will be merged into the default values file. For example, consider a `myvals.yaml` file that looks like this:

```shell
storage: "gcs"
```

When this is merged with the `values.yaml` in the chart, the resulting generated content will be:

```shell
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

Note that only the last field was overridden.

**NOTE**: The default values file included inside of a chart must be named `values.yaml`. But files specified on the command line can be named anything.

**NOTE**: If the `--set` flag is used on `helm install` or `helm upgrade`, those values are simply converted to YAML on the client side.

**NOTE**: If any required entries in the values file exist, they can be declared as required in the chart template by using the [‘required’ function](https://helm.sh/docs/developing_charts/#chart-development-tips-and-tricks)

Any of these values are then accessible inside of templates using the `.Values` object:

```shell
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

###### Scope, Dependencies, and Values

Values files can declare values for the top-level chart, as well as for any of the charts that are included in that chart’s `charts/` directory. Or, to phrase it differently, a values file can supply values to the chart as well as to any of its dependencies. For example, the demonstration WordPress chart above has both `mysql` and `apache` as dependencies. The values file could supply values to all of these components:

```shell
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

Charts at a higher level have access to all of the variables defined beneath(下面). So the WordPress chart can access the MySQL password as `.Values.mysql.password`. But lower level charts cannot access things in parent charts, so MySQL will not be able to access the `title` property. Nor, for that matter, can it access `apache.port` (同样，它也不能访问`apache.port`。).

Values are namespaced, but namespaces are pruned(修剪的). So for the WordPress chart, it can access the MySQL password field as `.Values.mysql.password`. But for the MySQL chart, the scope of the values has been reduced and the namespace prefix removed, so it will see the password field simply as `.Values.password`.

**Global Values:**

As of 2.0.0-Alpha.2, Helm supports special “global” value. Consider this modified version of the previous example:

```shell
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

The above adds a `global` section with the value `app: MyWordPress`. This value is available to all charts as `.Values.global.app`.

For example, the `mysql` templates may access app as `{{.Values.global.app}}`, and so can the `apache` chart. Effectively, the values file above is regenerated like this:

```shell
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

This provides a way of sharing one top-level variable with all subcharts, which is useful for things like setting `metadata` properties like labels.

If a subchart declares a global variable, that global will be passed ***downward*** (to the subchart’s subcharts), but not ***upward*** to the parent chart. There is no way for a subchart to influence the values of the parent chart.

Also, global variables of parent charts take precedence over(优先于) the global variables from subcharts.

###### References

When it comes to writing templates and values files, there are several standard references that will help you out.

- [Go templates](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [The YAML format](https://yaml.org/spec/)

### USING HELM TO MANAGE CHARTS

The `helm` tool has several commands for working with charts.

It can create a new chart for you:

```shell
$ helm create mychart
Created mychart/
```

Once you have edited a chart, `helm` can package it into a chart archive for you:

```shell
$ helm package mychart
Archived mychart-0.1.-.tgz
```

You can also use helm to help you find issues with your chart’s formatting or information:

```shell
$ helm lint mychart
No issues found
```

### CHART REPOSITORIES

A ***chart repository*** is an HTTP server that houses one or more packaged charts. While `helm` can be used to manage local chart directories, when it comes to sharing charts, the preferred mechanism is a chart repository.

Any HTTP server that can serve YAML files and tar files and can answer GET requests can be used as a repository server.

Helm comes with built-in package server for developer testing (`helm serve`). The Helm team has tested other servers, including Google Cloud Storage with website mode enabled, and S3 with website mode enabled.

A repository is characterized primarily(主要特征) by the presence(存在) of a special file called `index.yaml` that has a list of all of the packages supplied by the repository, together with metadata that allows retrieving and verifying those packages.

On the client side, repositories are managed with the `helm repo` commands. However, Helm does not provide tools for uploading charts to remote repository servers. This is because doing so would add substantial requirements(额外的要求) to an implementing server, and thus raise the `barrier`(障碍) for setting up a repository.

### CHART STARTER PACKS

The `helm create` command takes an optional --starter option that lets you specify a “starter chart”.

Starters are just regular charts, but are located in `$HELM_HOME/starters`. As a chart developer, you may author charts that are specifically designed to be used as starters. Such charts should be designed with the following considerations in mind:

- The `Chart.yaml` will be overwritten by the generator.
- Users will expect to modify such a chart’s contents, so documentation should indicate how users can do so.
- All occurrences of `<CHARTNAME>` in files within the templates directory will be replaced with the specified chart name so that starter charts can be used as templates. Additionally, occurrences of `<CHARTNAME>` in values.yaml will also be replaced.

Currently the only way to add a chart to `$HELM_HOME/starters` is to manually copy it there. In your chart’s documentation, you may want to explain that process.

## Hooks

Helm provides a hook mechanism to allow chart developers to intervene(干预) at certain points in a release’s life cycle. For example, you can use hooks to:

- Load a ConfigMap or Secret during install before any other charts are loaded.
- Execute a Job to back up a database before installing a new chart, and then execute a second job after the upgrade in order to restore data.
- Run a Job before deleting a release to gracefully take a service out of rotation before removing it.

Hooks work like regular templates, but they have special annotations that cause Helm to utilize(利用) them differently. In this section, we cover the basic usage pattern for hooks.

Hooks are declared as an annotation in the metadata section of a manifest:

```shell
apiVersion: ...
kind: ....
metadata:
  annotations:
    "helm.sh/hook": "pre-install"
# ...
```

### THE AVAILABLE HOOKS

The following hooks are defined:

- pre-install: Executes after templates are rendered, but before any resources are created in Kubernetes.
- post-install: Executes after all resources are loaded into Kubernetes
- pre-delete: Executes on a deletion request before any resources are deleted from Kubernetes.
- post-delete: Executes on a deletion request after all of the release’s resources have been deleted.
- pre-upgrade: Executes on an upgrade request after templates are rendered, but before any resources are loaded into Kubernetes (e.g. before a Kubernetes apply operation).
- post-upgrade: Executes on an upgrade after all resources have been upgraded.
- pre-rollback: Executes on a rollback request after templates are rendered, but before any resources have been rolled back.
- post-rollback: Executes on a rollback request after all resources have been modified.
- crd-install: Adds CRD resources before any other checks are run. This is used only on CRD definitions that are used by other manifests in the chart.
- test-success: Executes when running `helm test` and expects the pod to return successfully (return code == 0).
- test-failure: Executes when running `helm test` and expects the pod to fail (return code != 0).

### HOOKS AND THE RELEASE LIFECYCLE

Hooks allow you, the chart developer, an opportunity to perform operations at strategic points in a release lifecycle. For example, consider the lifecycle for a `helm install`. By default, the lifecycle looks like this:

1. User runs `helm install foo`
2. Chart is loaded into Tiller
3. After some verification, Tiller renders(渲染) the `foo` templates
4. Tiller loads the resulting resources into Kubernetes
5. Tiller returns the release name (and other data) to the client
6. The client exits

Helm defines two hooks for the `install` lifecycle: `pre-install` and `post-install`. If the developer of the `foo` chart implements both hooks, the lifecycle is altered like this:

1. User runs `helm install foo`
2. Chart is loaded into Tiller
3. After some verification, Tiller renders the `foo` templates
4. Tiller prepares to execute the `pre-install` hooks (loading hook resources into Kubernetes)
5. Tiller sorts hooks by weight (assigning a weight of 0 by default) and by name for those hooks with the same weight in ascending order.
6. Tiller then loads the hook with the lowest weight first (negative to positive)
7. Tiller waits until the hook is “Ready” (except for CRDs)
8. Tiller loads the resulting resources into Kubernetes. Note that if the `--wait` flag is set, Tiller will wait until all resources are in a ready state and will not run the `post-install` hook until they are ready.
9. Tiller executes the post-install hook (loading hook resources)
10. Tiller waits until the hook is “Ready”
11. Tiller returns the release name (and other data) to the client
12. The client exits

What does it mean to wait until a hook is ready? This depends on the resource declared in the hook. If the resources is a `Job` kind, Tiller will wait until the job successfully runs to completion. And if the job fails, the release will fail. This is a blocking operation, so the Helm client will pause while the Job is run.

For all other kinds, as soon as Kubernetes marks the resource as loaded (added or updated), the resource is considered “Ready”. When many resources are declared in a hook, the resources are executed serially(连续地). If they have hook weights (see below), they are executed in weighted order. Otherwise, ordering is not guaranteed. (In Helm 2.3.0 and after, they are sorted alphabetically(按字母顺序). That behavior, though, is not considered binding and could change in the future.) It is considered good practice to add a hook weight, and set it to 0 if weight is not important.

#### Hook resources are not managed with corresponding releases

The resources that a hook creates are not tracked or managed as part of the release. Once Tiller verifies that the hook has reached its ready state, it will leave the hook resource alone.

Practically speaking, this means that if you create resources in a hook, you cannot rely upon `helm delete` to remove the resources. To destroy such resources, you need to either write code to perform this operation in a `pre-delete` or `post-delete` hook or add `"helm.sh/hook-delete-policy"` annotation to the hook template file.

### WRITING A HOOK

Hooks are just Kubernetes manifest files with special annotations in the `metadata` section. Because they are template files, you can use all of the normal template features, including reading `.Values`, `.Release`, and `.Template`.

For example, this template, stored in `templates/post-install-job.yaml`, declares a job to be run on `post-install`:

```shell
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    app.kubernetes.io/managed-by: {{.Release.Service | quote }}
    app.kubernetes.io/instance: {{.Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{.Release.Name}}"
      labels:
        app.kubernetes.io/managed-by: {{.Release.Service | quote }}
        app.kubernetes.io/instance: {{.Release.Name | quote }}
        helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default "10" .Values.sleepyTime}}"]
```

What makes this template a hook is the annotation:

```shell
  annotations:
    "helm.sh/hook": post-install
```

One resource can implement multiple hooks:

```shell
  annotations:
    "helm.sh/hook": post-install,post-upgrade
```

Similarly, there is no limit to the number of different resources that may implement a given hook. For example, one could declare both a secret and a config map as a pre-install hook.

When subcharts declare hooks, those are also evaluated. There is no way for a top-level chart to disable the hooks declared by subcharts.

It is possible to define a weight for a hook which will help build a deterministic(确定性的) executing order. Weights are defined using the following annotation:

```shell
  annotations:
    "helm.sh/hook-weight": "5"
```

Hook weights can be positive or negative numbers but must be represented as strings. When Tiller starts the execution cycle of hooks of a particular kind (ex. the `pre-install` hooks or `post-install` hooks, etc.) it will sort those hooks in ascending(上升的) order.

It is also possible to define policies that determine when to delete corresponding hook resources. Hook deletion policies are defined using the following annotation:

```shell
  annotations:
    "helm.sh/hook-delete-policy": hook-succeeded
```

You can choose one or more defined annotation values:

- `"hook-succeeded"` specifies Tiller should delete the hook after the hook is successfully executed.
- `"hook-failed"` specifies Tiller should delete the hook if the hook failed during execution.
- `"before-hook-creation"` specifies Tiller should delete the previous hook before the new hook is launched.

By default Tiller will wait for 60 seconds for a deleted hook to no longer exist in the API server before timing out. This behavior can be changed using the `helm.sh/hook-delete-timeout` annotation. The value is the number of seconds Tiller should wait for the hook to be fully deleted. A value of 0 means Tiller does not wait at all.

#### Defining a CRD with the `crd-install` Hook

Custom Resource Definitions (CRDs) are a special kind in Kubernetes. They provide a way to define other kinds.

On occasion, a chart needs to both define a kind and then use it. This is done with the `crd-install` hook.

The `crd-install` hook is executed very early during an installation, before the rest of the manifests are verified. CRDs can be annotated with this hook so that they are installed before any instances of that CRD are referenced. In this way, when verification happens later, the CRDs will be available.

Here is an example of defining a CRD with a hook, and an instance of the CRD:

```shell
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
  annotations:
    "helm.sh/hook": crd-install
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

And:

```shell
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: {{ .Release.Name }}-inst
```

Both of these can now be in the same chart, provided that the CRD is correctly annotated.

#### Automatically delete hook from previous release

When a helm release, that uses a hook, is being updated, it is possible that the hook resource might already exist in the cluster. In such circumstances, by default, helm will fail trying to install the hook resource with an `"... already exists"` error.

A common reason why the hook resource might already exist is that it was not deleted following use on a previous install/upgrade. There are, in fact, good reasons why one might want to keep the hook: for example, to aid manual debugging in case something went wrong. In this case, the recommended way of ensuring subsequent attempts to create the hook do not fail is to define a "`hook-delete-policy`" that can handle this: `"helm.sh/hook-delete-policy": "before-hook-creation"`. This hook annotation causes any existing hook to be removed, before the new hook is installed.

If it is preferred to actually delete the hook after each use (rather than have to handle it on a subsequent use, as shown above), then this can be achieved using a delete policy of `"helm.sh/hook-delete-policy": "hook-succeeded,hook-failed"`.

### Chart Development Tips and Tricks

This guide covers some of the tips and tricks Helm chart developers have learned while building production-quality charts.

#### KNOW YOUR TEMPLATE FUNCTIONS

Helm uses [Go templates](https://godoc.org/text/template) for templating your resource files. While Go ships several built-in functions, we have added many others.

First, we added almost all of the functions in the [Sprig library](https://godoc.org/github.com/Masterminds/sprig). We removed two for security reasons: `env` and `expandenv` (which would have given chart authors access to Tiller’s environment).

We also added two special template functions: `include` and `required`. The `include` function allows you to bring in another template, and then pass the results to other template functions.

For example, this template snippet(片段) includes a template called `mytpl`, then lowercases the result, then wraps that in double quotes(双引号).

```shell
value: {{ include "mytpl" . | lower | quote }}
```

The `required` function allows you to declare a particular values entry as required for template rendering. If the value is empty, the template rendering will fail with a user submitted error message.

The following example of the `required` function declares an entry for .Values.who is required, and will print an error message when that entry is missing:

```shell
value: {{ required "A valid .Values.who entry required!" .Values.who }}
```

When using the `include` function, you can pass it a custom object tree built from the current context by using the `dict` function:

```shell
{{- include "mytpl" (dict "key1" .Values.originalKey1 "key2" .Values.originalKey2) }}
```

#### QUOTE STRINGS, DON’T QUOTE INTEGERS

When you are working with string data, you are always safer quoting the strings than leaving them as bare words:

```shell
name: {{ .Values.MyName | quote }}
```

But when working with integers do not quote the values. That can, in many cases, cause parsing errors inside of Kubernetes.

```shell
port: {{ .Values.Port }}
```

This remark does not apply to env variables values which are expected to be string, even if they represent integers:

```shell
env:
  -name: HOST
    value: "http://host"
  -name: PORT
    value: "1234"
```

#### USING THE ‘INCLUDE’ FUNCTION

Go provides a way of including one template in another using a built-in `template` directive. However, the built-in function cannot be used in Go template pipelines.

To make it possible to include a template, and then perform an operation on that template’s output, Helm has a special `include` function:

```shell
{{- include "toYaml" $value | nindent 2 }}
```

The above includes a template called `toYaml`, passes it `$value`, and then passes the output of that template to the `nindent` function. Using the `{{- ... | nindent _n_ }}` pattern makes it easier to read the `include` in context, because it chomps the whitespace to the left (including the previous newline), then the `nindent` re-adds the newline and indents the included content by the requested amount.

Because YAML ascribes significance to indentation levels and whitespace(因为YAML将重要性归因于缩进级别和空白), this is one great way to include snippets of code, but handle indentation in a relevant context.

#### USING THE ‘REQUIRED’ FUNCTION

Go provides a way for setting template options to control behavior when a map is indexed with a key that’s not present in the map. This is typically set with `template.Options(“missingkey=option”)`, where option can be `default`, `zero`, or `error`. While setting this option to `error` will stop execution with an error, this would apply to every missing key in the map. There may be situations where a chart developer wants to enforce this behavior for select values in the values.yml file.

The `required` function gives developers the ability to declare a value entry as required for template rendering. If the entry is empty in values.yml, the template will not render and will return an error message supplied by the developer.

For example:

```shell
{{ required "A valid foo is required!" .Values.foo }}
```

The above will render the template when .Values.foo is defined, but will fail to render and exit when .Values.foo is undefined.

#### USING THE ‘TPL’ FUNCTION

The `tpl` function allows developers to evaluate strings as templates inside a template(tpm 功能允许开发者来测试将字符串作为模板传递给template). This is useful to pass a template string as a value to a chart or render external configuration files. Syntax: `{{ tpl TEMPLATE_STRING VALUES }}`

Examples:

```shell
# values
template: "{{ .Values.name }}"
name: "Tom"

# template
{{ tpl .Values.template . }}

# output
Tom
```

Rendering a external configuration file:

```shell
# external configuration file conf/app.conf
firstName={{ .Values.firstName }}
lastName={{ .Values.lastName }}

# values
firstName: Peter
lastName: Parker

# template
{{ tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```

#### CREATING IMAGE PULL SECRETS

Image pull secrets are essentially(实质上) a combination of ***registry***, ***username***, and ***password***. You may need them in an application you are deploying, but to create them requires running ***base64*** a couple of times. We can write a helper template to compose the Docker configuration file for use as the Secret’s payload(有效载荷). Here is an example:

First, assume that the credentials are defined in the `values.yaml` file like so:

```shell
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

We then define our helper template as follows:

```shell
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```

Finally, we use the helper template in a larger template to create the Secret manifest:

```shell
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

#### AUTOMATICALLY ROLL DEPLOYMENTS WHEN CONFIGMAPS OR SECRETS CHANGE

Often times configmaps or secrets are injected as configuration files in containers. Depending on the application a restart may be required should those be updated with a subsequent `helm upgrade`, but if the deployment spec itself didn’t change the application keeps running with the old configuration resulting in an inconsistent deployment.

The `sha256sum` function can be used to ensure a deployment’s annotation section is updated if another file changes:

```shell
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

See also the `helm upgrade --recreate-pods` flag for a slightly different way of addressing this issue.

#### TELL TILLER NOT TO DELETE A RESOURCE

Sometimes there are resources that should not be deleted when Helm runs a `helm delete`. Chart developers can add an annotation to a resource to prevent it from being deleted.

```shell
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

(Quotation marks are required)

The annotation `"helm.sh/resource-policy": keep` instructs Tiller to skip this resource during a `helm delete` operation. However, this resource becomes orphaned(孤儿). Helm will no longer manage it in any way. This can lead to problems if using `helm install --replace` on a release that has already been deleted, but has kept resources.

To explicitly(明确的) opt in to resource deletion, for example when overriding a chart’s default annotations, set the resource policy annotation value to `delete`.

#### USING “PARTIALS” AND TEMPLATE INCLUDES

Sometimes you want to create some reusable parts in your chart, whether they’re blocks or template partials(部分). And often, it’s cleaner to keep these in their own files.

In the `templates/` directory, any file that begins with an underscore(`_`) is not expected to output a Kubernetes manifest file. So by convention, helper templates and partials are placed in a `_helpers.tpl` file.

#### COMPLEX CHARTS WITH MANY DEPENDENCIES

Many of the charts in the [official charts repository](https://github.com/helm/charts) are “building blocks” for creating more advanced applications. But charts may be used to create instances of large-scale applications. In such cases, a single umbrella chart may have multiple subcharts, each of which functions as a piece of the whole.

The current best practice for composing a complex application from discrete parts is to create a top-level umbrella chart that exposes the global configurations, and then use the `charts/` subdirectory to embed each of the components.

Two strong design patterns are illustrated by these projects:

SAP’s [Converged charts](https://github.com/sapcc/helm-charts): These charts install SAP Converged Cloud a full OpenStack IaaS on Kubernetes. All of the charts are collected together in one GitHub repository, except for a few submodules.

Deis’s [Workflow](https://github.com/deis/workflow/tree/master/charts/workflow): This chart exposes the entire Deis PaaS system with one chart. But it’s different from the SAP chart in that this umbrella chart is built from each component, and each component is tracked in a different Git repository. Check out the `requirements.yaml` file to see how this chart is composed by their CI/CD pipeline.

Both of these charts illustrate proven techniques for standing up complex environments using Helm.

#### YAML IS A SUPERSET OF JSON

According to the YAML specification, YAML is a superset of JSON. That means that any valid JSON structure ought to be valid in YAML.

This has an advantage: Sometimes template developers may find it easier to express a data structure with a JSON-like syntax rather than deal with YAML’s whitespace sensitivity.

As a best practice, templates should follow a YAML-like syntax unless the JSON syntax substantially reduces the risk of a formatting issue.

#### BE CAREFUL WITH GENERATING RANDOM VALUES

There are functions in Helm that allow you to generate random data, cryptographic keys, and so on. These are fine to use. But be aware that during upgrades, templates are re-executed. When a template run generates data that differs from the last run, that will trigger an update of that resource.

#### UPGRADE A RELEASE IDEMPOTENTLY

In order to use the same command when installing and upgrading a release, use the following command:

```shell
helm upgrade --install <release name> --values <values file> <chart directory>
```

类似于 `kubectl apply`，一个命令既可以新建又可以更新。

## The Chart Repository Guide

This section explains how to create and work with Helm chart repositories. At a high level, a chart repository is a location where packaged charts can be stored and shared.

The official chart repository is maintained by the [Helm Charts](https://github.com/helm/charts), and we welcome participation. But Helm also makes it easy to create and run your own chart repository. This guide explains how to do so.

### PREREQUISITES

- Go through the [Quickstart](https://helm.sh/docs/developing_charts/#quickstart) Guide
- Read through the [Charts](https://helm.sh/docs/developing_charts/#charts) document

### CREATE A CHART REPOSITORY

A **chart repository** is an HTTP server that houses an index.yaml file and optionally some packaged charts. When you’re ready to share your charts, the preferred way to do so is by uploading them to a chart repository.

**Note**: For Helm 2.0.0, chart repositories do not have any intrinsic(固有) authentication(认证方式). There is an [issue tracking progress](https://github.com/helm/helm/issues/1038) in GitHub.

Because a chart repository can be any HTTP server that can serve YAML and tar files and can answer GET requests, you have a plethora(过多) of options when it comes down to hosting your own chart repository. For example, you can use a Google Cloud Storage (GCS) bucket(桶), Amazon S3 bucket, Github Pages, or even create your own web server.

#### The chart repository structure

A chart repository consists of packaged charts and a special file called `index.yaml` which contains an index of all of the charts in the repository. Frequently(通常地), the charts that `index.yaml` describes are also hosted on the same server, as are the [provenance files(源文件)](https://helm.sh/docs/developing_charts/#helm-provenance-and-integrity).

For example, the layout of the repository `https://example.com/charts` might look like this:

```shell
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```

In this case, the index file would contain information about one chart, the Alpine chart, and provide the download URL `https://example.com/charts/alpine-0.1.2.tgz` for that chart.

It is not required that a chart package be located on the same server as the `index.yaml` file. However, doing so is often the easiest.

#### The index file

The index file is a yaml file called `index.yaml`. It contains some metadata about the package, including the contents of a chart’s `Chart.yaml` file. A valid chart repository must have an index file. The index file contains information about each chart in the chart repository. The `helm repo index` command will generate an index file based on a given local directory that contains packaged charts.

This is an example of an index file:

```shell
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://k8s.io/helm
      name: nginx
      sources:
      - https://github.com/helm/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```

A generated index and packages can be served from a basic webserver. You can test things out locally with the `helm serve` command, which starts a local server.

```shell
$ helm serve --repo-path ./charts
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

The above starts a local webserver, serving the charts it finds in `./charts`. The serve command will automatically generate an `index.yaml` file for you during startup.

### HOSTING CHART REPOSITORIES

This part shows several ways to serve a chart repository.

#### ChartMuseum

The Helm project provides an open-source Helm repository server called [ChartMuseum](https://chartmuseum.com/) that you can host yourself.

ChartMuseum supports multiple cloud storage backends. Configure it to point to the directory or bucket containing your chart packages, and the index.yaml file will be generated dynamically.

It can be deployed easily as a [Helm chart](https://github.com/helm/charts/tree/master/stable/chartmuseum):

```shell
helm install stable/chartmuseum
```

and also as a [Docker image](https://hub.docker.com/r/chartmuseum/chartmuseum/tags):

```shell
docker run --rm -it \
  -p 8080:8080 \
  -v $(pwd)/charts:/charts \
  -e DEBUG=true \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  chartmuseum/chartmuseum
```

You can then add the repo to your local repository list:

```shell
helm repo add chartmuseum http://localhost:8080
```

ChartMuseum provides other features, such as an API for chart uploads. Please see the [README](https://github.com/helm/chartmuseum) for more info.

#### Github Pages example

In a similar way you can create charts repository using GitHub Pages.

GitHub allows you to serve static web pages in two different ways:

- By configuring a project to serve the contents of its `docs/` directory
- By configuring a project to serve a particular branch

We’ll take the second approach, though the first is just as easy.

The first step will be to **create your gh-pages branch**. You can do that locally as.

```shell
git checkout -b gh-pages
```

Or via web browser using **Branch** button on your Github repository:

![image](../../images/create-a-gh-page-button-20191014.png)

Next, you’ll want to make sure your **gh-pages branch** is set as Github Pages, click on your repo **Settings** and scroll down to **Github pages** section and set as per below:

![image](../../images/set-a-gh-page-20191014.png)

By default **Source** usually gets set to **gh-pages branch**. If this is not set by default, then select it.

You can use a **custom domain** there if you wish so.

And check that **Enforce HTTPS** is ticked, so the **HTTPS** will be used when charts are served.

In such setup you can use **master branch** to store your charts code, and **gh-pages branch** as charts repository, e.g.: `https://USERNAME.github.io/REPONAME`. The demonstration(示范) [TS Charts](https://github.com/technosophos/tscharts) repository is accessible at `https://technosophos.github.io/tscharts/`.

#### Ordinary(普通的) web servers

To configure an ordinary web server to serve Helm charts, you merely need to do the following:

- Put your index and charts in a directory that the server can serve
- Make sure the `index.yaml` file can be accessed with no authentication requirement
- Make sure `yaml` files are served with the correct content type (`text/yaml` or `text/x-yaml`)

For example, if you want to serve your charts out of `$WEBROOT/charts`, make sure there is a `charts/` directory in your web root, and put the index file and charts inside of that folder.

### MANAGING CHART REPOSITORIES

Now that you have a chart repository, the last part of this guide explains how to maintain charts in that repository.

#### Store charts in your chart repository

Now that you have a chart repository, let’s upload a chart and an index file to the repository. Charts in a chart repository must be packaged (`helm package chart-name/`) and versioned correctly (following [SemVer 2](https://semver.org/) guidelines).

These next steps compose an example workflow, but you are welcome to use whatever workflow you fancy for storing and updating charts in your chart repository.

Once you have a packaged chart ready, create a new directory, and move your packaged chart to that directory.

```shell
helm package docs/examples/alpine/
mkdir fantastic-charts
mv alpine-0.1.0.tgz fantastic-charts/
helm repo index fantastic-charts --url https://fantastic-charts.storage.googleapis.com
```

The last command takes the path of the local directory that you just created and the URL of your remote chart repository and composes an `index.yaml` file inside the given directory path.

Now you can upload the chart and the index file to your chart repository using a sync tool or manually. If you’re using Google Cloud Storage, check out this [example workflow](https://helm.sh/docs/developing_charts/#developing_charts_sync_example) using the gsutil client. For GitHub, you can simply put the charts in the appropriate destination branch.

#### Add new charts to an existing repository

Each time you want to add a new chart to your repository, you must regenerate the index. The `helm repo index` command will completely rebuild the `index.yaml` file from scratch, including only the charts that it finds locally.

However, you can use the `--merge` flag to incrementally add new charts to an existing `index.yaml` file (a great option when working with a remote repository like GCS). Run `helm repo index --help` to learn more,

Make sure that you upload both the revised `index.yaml` file and the chart. And if you generated a provenance file, upload that too.

#### Share your charts with others

When you’re ready to share your charts, simply let someone know what the URL of your repository is.

From there, they will add the repository to their helm client via the `helm repo add [NAME] [URL]` command with any name they would like to use to reference the repository.

```shell
helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```shell
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

If the charts are backed by HTTP basic authentication, you can also supply the username and password here:

```shell
helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com --username my-username --password my-password
```

```shell
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

Note: A repository will not be added if it does not contain a valid `index.yaml`.

After that, your users will be able to search through your charts. After you’ve updated the repository, they can use the `helm repo update` command to get the latest chart information.

Under the hood, the `helm repo add` and `helm repo update` commands are fetching the index.yaml file and storing them in the `$HELM_HOME/repository/cache/` directory. This is where the `helm search` function finds information about charts.

### Chart Tests

A chart contains a number of Kubernetes resources and components that work together. As a chart author, you may want to write some tests that validate that your chart works as expected when it is installed. These tests also help the chart consumer understand what your chart is supposed to do.

A **test** in a helm chart lives under the `templates/` directory and is a pod definition that specifies a container with a given command to run. The container should exit successfully (exit 0) for a test to be considered a success. The pod definition must contain one of the helm test hook annotations: `helm.sh/hook: test-success` or `helm.sh/hook: test-failure`.

Example tests: - Validate that your configuration from the values.yaml file was properly injected. - Make sure your username and password work correctly - Make sure an incorrect username and password does not work - Assert that your services are up and correctly loadbalanced. - etc.

You can run the pre-defined tests in Helm on a release using the command `helm test <RELEASE_NAME>`. For a chart consumer, this is a great way to sanity check that their release of a chart (or application) works as expected.

#### A BREAKDOWN OF THE HELM TEST HOOKS

In Helm, there are two test hooks: `test-success` and `test-failure`

`test-success` indicates that test pod should complete successfully. In other words, the containers in the pod should exit 0. `test-failure` is a way to assert that a test pod should not complete successfully. If the containers in the pod do not exit 0, that indicates success.

#### EXAMPLE TEST

Here is an example of a helm test pod definition in an example wordpress chart. The test verifies the access and login to the mariadb database:

```shell
wordpress/
  Chart.yaml
  README.md
  values.yaml
  charts/
  templates/
  templates/tests/test-mariadb-connection.yaml
```

In `wordpress/templates/tests/test-mariadb-connection.yaml`:

```shell
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-credentials-test"
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
  - name: {{ .Release.Name }}-credentials-test
    image: {{ .Values.image }}
    env:
      - name: MARIADB_HOST
        value: {{ template "mariadb.fullname" . }}
      - name: MARIADB_PORT
        value: "3306"
      - name: WORDPRESS_DATABASE_NAME
        value: {{ default "" .Values.mariadb.mariadbDatabase | quote }}
      - name: WORDPRESS_DATABASE_USER
        value: {{ default "" .Values.mariadb.mariadbUser | quote }}
      - name: WORDPRESS_DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ template "mariadb.fullname" . }}
            key: mariadb-password
    command: ["sh", "-c", "mysql --host=$MARIADB_HOST --port=$MARIADB_PORT --user=$WORDPRESS_DATABASE_USER --password=$WORDPRESS_DATABASE_PASSWORD"]
  restartPolicy: Never
```

#### STEPS TO RUN A TEST SUITE ON A RELEASE

1. `$ helm install stable/wordpress`

    ```shell
    NAME:   quirky-walrus
    LAST DEPLOYED: Mon Feb 13 13:50:43 2017
    NAMESPACE: default
    STATUS: DEPLOYED
    ```

2. `$ helm test quirky-walrus`

    ```shell
    RUNNING: quirky-walrus-credentials-test
    SUCCESS: quirky-walrus-credentials-test
    ```

**NOTES:**

- You can define as many tests as you would like in a single yaml file or spread across several yaml files in the `templates/` directory
- You are welcome to nest your test suite under a `tests/` directory like `<chart-name>/templates/tests/` for more isolation

## The Chart Template Developer’s Guide

This guide provides an introduction to Helm’s chart templates, with emphasis(重点) on the template language.

Templates generate manifest files, which are YAML-formatted resource descriptions that Kubernetes can understand. We’ll look at how templates are structured, how they can be used, how to write Go templates, and how to debug your work.

This guide focuses on the following concepts:

- The Helm template language
- Using values
- Techniques for working with templates

This guide is oriented toward learning the ins and outs of the Helm template language. Other guides provide introductory material(介绍性质的材料)), examples(示例), and best practices(最佳实践).

### Getting Started with a Chart Template

In this section of the guide, we’ll create a chart and then add a first template. The chart we created here will be used throughout the rest of the guide.

To get going, let’s take a brief look at a Helm chart.

#### CHARTS

As described in the [Charts Guide](https://helm.sh/docs/chart_template_guide/#../charts), Helm charts are structured like this:

```shell
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

The `templates/` directory is for template files. When Tiller evaluates a chart, it will send all of the files in the `templates/` directory through the template rendering engine. Tiller then collects the results of those templates and sends them on to Kubernetes.

The `values.yaml` file is also important to templates. This file contains the **default values** for a chart. These values may be overridden by users during `helm install` or `helm upgrade`.

The `Chart.yaml` file contains a description of the chart. You can access it from within a template. The `charts/` directory **may** contain other charts (which we call **subcharts**). Later in this guide we will see how those work when it comes to template rendering.

#### A STARTER CHART

For this guide, we’ll create a simple chart called `mychart`, and then we’ll create some templates inside of the chart.

```shell
$ helm create mychart
Creating mychart
```

From here on, we’ll be working in the `mychart` directory.

##### A Quick Glimpse of `mychart/templates/`

If you take a look at the `mychart/templates/` directory, you’ll notice a few files already there.

- **NOTES.txt**: The “help text” for your chart. This will be displayed to your users when they run `helm install`.
- **deployment.yaml**: A basic manifest for creating a Kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- **service.yaml**: A basic manifest for creating a [service endpoint](https://kubernetes.io/docs/concepts/services-networking/service/) for your deployment
- **_helpers.tpl**: A place to put template helpers that you can re-use throughout the chart

And what we’re going to do is… ***remove them all!*** That way we can work through our tutorial from scratch. We’ll actually create our own `NOTES.txt` and `_helpers.tpl` as we go.

```shell
rm -rf mychart/templates/*.*
```

When you’re writing production grade charts, having basic versions of these charts can be really useful. So in your day-to-day chart authoring, you probably won’t want to remove them.

#### A FIRST TEMPLATE

The first template we are going to create will be a `ConfigMap`. In Kubernetes, a ConfigMap is simply a container for storing configuration data. Other things, like pods, can access the data in a ConfigMap.

Because ConfigMaps are basic resources, they make a great starting point for us.

Let’s begin by creating a file called `mychart/templates/configmap.yaml`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

**TIP**: Template names do not follow a rigid(刚性，生硬，严格) naming pattern. However, we recommend using the suffix .yaml for YAML files and .tpl for helpers.

The YAML file above is a bare-bones ConfigMap, having the minimal necessary fields. In virtue of the fact that this file is in the `templates/` directory, it will be sent through the template engine.

It is just fine to put a plain YAML file like this in the `templates/` directory. When Tiller reads this template, it will simply send it to Kubernetes as-is.

With this simple template, we now have an installable chart. And we can install it like this:

```shell
$ helm install ./mychart
NAME: full-coral
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA      AGE
mychart-configmap   1         1m
```

In the output above, we can see that our ConfigMap was created. Using Helm, we can retrieve the release and see the actual template that was loaded.

```shell
$ helm get manifest full-coral
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

The `helm get manifest` command takes a release name (`full-coral`) and prints out all of the Kubernetes resources that were uploaded to the server. Each file begins with `---` to indicate the start of a YAML document, and then is followed by an automatically generated comment line that tells us what template file generated this YAML document.

From there on, we can see that the YAML data is exactly what we put in our `configmap.yaml` file.

Now we can delete our release: `helm delete full-coral`.

##### Adding a Simple Template Call

Hard-coding the `name`: into a resource is usually considered to be bad practice. Names should be unique to a release. So we might want to generate a name field by inserting the release name.

**TIP**: The `name`: field is limited to 63 characters because of limitations to the DNS system. For that reason, release names are limited to 53 characters. Kubernetes 1.3 and earlier limited to only 24 characters (thus 14 character names).

Let’s alter `configmap.yaml` accordingly.

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

The big change comes in the value of the `name:` field, which is now `{{ .Release.Name }}-configmap`.

A template directive(指示，指令) is enclosed in {{ and }} blocks.

The template directive `{{ .Release.Name }}` injects the release name into the template. The values that are passed into a template can be thought of as `namespaced objects`, where a dot (`.`) separates each namespaced element.

The leading dot before `Release` indicates that we start with the top-most namespace for this scope (we’ll talk about scope in a bit). So we could read `.Release.Name` as “start at the top namespace, find the `Release` object, then look inside of it for an object called `Name`”.

The `Release` object is one of the built-in objects for Helm, and we’ll cover it in more depth later. But for now, it is sufficient to say that this will display the release name that Tiller assigns to our release.

Now when we install our resource, we’ll immediately see the result of using this template directive:

```shell
$ helm install ./mychart
NAME: clunky-serval
LAST DEPLOYED: Tue Nov  1 17:45:37 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                      DATA      AGE
clunky-serval-configmap   1         1m
```

Note that in the `RESOURCES` section, the name we see there is `clunky-serval-configmap` instead of `mychart-configmap`.

You can run `helm get manifest clunky-serval` to see the entire generated YAML.

At this point, we’ve seen templates at their most basic: YAML files that have template directives embedded in `{{` and `}}`. In the next part, we’ll take a deeper look into templates. But before moving on, there’s one quick trick that can make building templates faster: When you want to test the template rendering, but not actually install anything, you can use `helm install ./mychart --debug --dry-run`. This will send the chart to the Tiller server, which will render the templates. But instead of installing the chart, it will return the rendered template to you so you can see the output:

```shell
$ helm install ./mychart --debug --dry-run
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   goodly-guppy
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"
```

Using `--dry-run` will make it easier to test your code, but it won’t ensure that Kubernetes itself will accept the templates you generate. It’s best not to assume that your chart will install just because `--dry-run` works.

In the next few sections, we’ll take the basic chart we defined here and explore the Helm template language in detail. And we’ll get started with built-in objects.

### Built-in Objects

Objects are passed into a template from the template engine. And your code can pass objects around (we’ll see examples when we look at the `with` and `range` statements). There are even a few ways to create new objects within your templates, like with the `list` function we’ll see later.

Objects can be simple, and have just one value. Or they can contain other objects or functions. For example. the `Release` object contains several objects (like `Release.Name`) and the `Files` object has a few functions.

In the previous section, we use `{{.Release.Name}}` to insert the name of a release into a template. `Release` is one of the top-level objects that you can access in your templates.

- `Release`: This object describes the release itself. It has several objects inside of it:
  - `Release.Name`: The release name
  - `Release.Time`: The time of the release
  - `Release.Namespace`: The namespace to be released into (if the manifest doesn’t override)
  - `Release.Service`: The name of the releasing service (always `Tiller`).
  - `Release.Revision`: The revision number of this release. It begins at 1 and is incremented for each helm upgrade.
  - `Release.IsUpgrade`: This is set to `true` if the current operation is an upgrade or rollback.
  - `Release.IsInstall`: This is set to `true` if the current operation is an install.

- `Values`: Values passed into the template from the `values.yaml` file and from user-supplied files. By default, `Values` is empty.
- `Chart`: The contents of the `Chart.yaml` file. Any data in `Chart.yaml` will be accessible here. For example `{{.Chart.Name}}-{{.Chart.Version}}` will print out the `mychart-0.1.0`.
  - The available fields are listed in the [Charts Guide](https://github.com/helm/helm/blob/master/docs/charts.md#the-chartyaml-file)

- `Files`: This provides access to all non-special files in a chart. While you cannot use it to access templates, you can use it to access other files in the chart. See the section *Accessing Files* for more.
  - `Files.Get` is a function for getting a file by name (`.Files.Get config.ini`)
  - `Files.GetBytes` is a function for getting the contents of a file as an array of bytes instead of as a string. This is useful for things like images.

- `Capabilities`(能力): This provides information about what capabilities the Kubernetes cluster supports.
  - `Capabilities.APIVersions` is a set of versions.
  - `Capabilities.APIVersions.Has $version` indicates whether a version (e.g., `batch/v1`) or resource (e.g., `apps/v1/Deployment`) is available on the cluster. Note, resources were not available before Helm v2.15.
  - `Capabilities.KubeVersion` provides a way to look up the Kubernetes version. It has the following values: `Major`, `Minor`, `GitVersion`, `GitCommit`, `GitTreeState`, `BuildDate`, `GoVersion`, `Compiler`, and `Platform`.
  - `Capabilities.TillerVersion` provides a way to look up the Tiller version. It has the following values: `SemVer`, `GitCommit`, and `GitTreeState`.

- `Template`: Contains information about the current template that is being executed
  - `Name`: A namespaced filepath to the current template (e.g. `mychart/templates/mytemplate.yaml`)
  - `BasePath`: The namespaced path to the templates directory of the current chart (e.g. `mychart/templates`).

The values are available to any top-level template. As we will see later, this does not necessarily mean that they will be available everywhere.

The built-in values always begin with a capital letter. This is in keeping with Go’s naming convention. When you create your own names, you are free to use a convention that suits your team. Some teams, like the [Helm Charts](https://github.com/helm/charts) team, choose to use only initial lower case letters in order to distinguish local names from those built-in. In this guide, we follow that convention.

### Values Files

In the previous section we looked at the built-in objects that Helm templates offer. One of these built-in objects is `Values`. This object provides access to values passed into the chart. Its contents come from four sources:

- The `values.yaml` file in the chart
- If this is a subchart, the `values.yaml` file of a parent chart
- A values file is passed into `helm install` or `helm upgrade` with the `-f` flag (`helm install -f myvals.yaml ./mychart`)
- Individual parameters passed with `--set` (such as `helm install --set foo=bar ./mychart`)

The list above is in order of specificity: `values.yaml` is the default, which can be overridden by a parent chart’s `values.yaml`, which can in turn be overridden by a user-supplied values file, which can in turn be overridden by `--set` parameters.

Values files are plain YAML files. Let’s edit `mychart/values.yaml` and then edit our ConfigMap template.

Removing the defaults in `values.yaml`, we’ll set just one parameter:

```shell
favoriteDrink: coffee
```

Now we can use this inside of a template:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

Notice on the last line we access `favoriteDrink` as an attribute of `Values`: `{{ .Values.favoriteDrink }}`.

Let’s see how this renders.

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   geared-marsupi
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: geared-marsupi-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

Because `favoriteDrink` is set in the default `values.yaml` file to `coffee`, that’s the value displayed in the template. We can easily override that by adding a `--set` flag in our call to `helm install`:

```shell
helm install --dry-run --debug --set favoriteDrink=slurm ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   solid-vulture
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

Since `--set` has a higher precedence than the default `values.yaml` file, our template generates `drink: slurm`.

Values files can contain more structured content, too. For example, we could create a `favorite` section in our `values.yaml` file, and then add several keys there:

```shell
favorite:
  drink: coffee
  food: pizza
```

Now we would have to modify the template slightly:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

While structuring data this way is possible, the recommendation is that you keep your values trees shallow, favoring flatness. When we look at assigning values to subcharts, we’ll see how values are named using a tree structure.

#### DELETING A DEFAULT KEY

If you need to delete a key from the default values, you may override the value of the key to be `null`, in which case Helm will remove the key from the overridden values merge.

For example, the stable Drupal chart allows configuring the liveness probe, in case you configure a custom image. Here are the default values:

```shell
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

If you try to override the livenessProbe handler to `exec` instead of `httpGet` using `--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`, Helm will coalesce the default and overridden keys together, resulting in the following YAML:

```shell
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

However, Kubernetes would then fail because you can not declare more than one livenessProbe handler. To overcome this, you may instruct Helm to delete the `livenessProbe.httpGet` by setting it to null:

```shell
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

At this point, we’ve seen several built-in objects, and used them to inject information into a template. Now we will take a look at another aspect of the template engine: functions and pipelines.

### Template Functions and Pipelines

So far, we’ve seen how to place information into a template. But that information is placed into the template unmodified. Sometimes we want to transform the supplied data in a way that makes it more usable to us.

Let’s start with a best practice: When injecting strings from the `.Values` object into the template, we ought to quote these strings. We can do that by calling the `quote` function in the template directive:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

Template functions follow the syntax `functionName arg1 arg2....` In the snippet above, `quote .Values.favorite.drink` calls the quote function and passes it a single argument.

Helm has over 60 available functions. Some of them are defined by the [Go template language](https://godoc.org/text/template) itself. Most of the others are part of the [Sprig template library](https://godoc.org/github.com/Masterminds/sprig). We’ll see many of them as we progress through the examples.

While we talk about the “Helm template language” as if it is Helm-specific, it is actually a combination of the Go template language, some extra functions, and a variety of wrappers to expose certain objects to the templates. Many resources on Go templates may be helpful as you learn about templating.

#### PIPELINES

One of the powerful features of the template language is its concept of **pipelines**(管道). Drawing on a concept from UNIX, pipelines are a tool for chaining together a series of template commands to compactly express a series of transformations. In other words, pipelines are an efficient way of getting several things done in sequence. Let’s rewrite the above example using a pipeline.

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

In this example, instead of calling `quote ARGUMENT`, we inverted the order. We “sent” the argument to the function using a pipeline (`|`): `.Values.favorite.drink | quote`. Using pipelines, we can chain several functions together:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

Inverting the order is a common practice in templates. You will see `.val | quote` more often than `quote .val`. Either practice is fine.

When evaluated, that template will produce this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

Note that our original `pizza` has now been transformed to `"PIZZA"`.

When pipelining arguments like this, the result of the first evaluation (`.Values.favorite.drink`) is sent as the ***last argument to the function***. We can modify the drink example above to illustrate with a function that takes two arguments: `repeat COUNT STRING`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

The `repeat` function will echo the given string the given number of times, so we will get this for output:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

#### USING THE `DEFAULT` FUNCTION

One function frequently used in templates is the `default` function: `default DEFAULT_VALUE GIVEN_VALUE`. This function allows you to specify a default value inside of the template, in case the value is omitted. Let’s use it to modify the drink example above:

```shell
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

If we run this as normal, we’ll get our `coffee`:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

Now, we will remove the favorite drink setting from `values.yaml`:

```shell
favorite:
  #drink: coffee
  food: pizza
```

Now re-running `helm install --dry-run --debug ./mychart` will produce this YAML:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

In an actual chart, all static default values should live in the values.yaml, and should not be repeated using the `default` command (otherwise they would be redundant). However, the `default` command is perfect for computed values, which can not be declared inside values.yaml. For example:

```shell
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

In some places, an `if` conditional guard may be better suited than `default`. We’ll see those in the next section.

Template functions and pipelines are a powerful way to transform information and then insert it into your YAML. But sometimes it’s necessary to add some template logic that is a little more sophisticated than just inserting a string. In the next section we will look at the control structures provided by the template language.

#### OPERATORS ARE FUNCTIONS

Operators are implemented as functions that return a boolean value. To use `eq`, `ne`, `lt`, `gt`, and, `or`, `not` etcetera(等等) place the operator at the front of the statement followed by its parameters just as you would a function. To chain multiple operations together, separate individual functions by surrounding them with parentheses.

```shell
{{/* include the body of this if statement when the variable .Values.fooString exists and is set to "foo" */}}
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}


{{/* include the body of this if statement when the variable .Values.anUnsetVariable is set or .values.aSetVariable is not set */}}
{{ if or .Values.anUnsetVariable (not .Values.aSetVariable) }}
   {{ ... }}
{{ end }}
```

Now we can turn from functions and pipelines to flow control with conditions, loops, and scope modifiers.

### Flow Control

Control structures (called “actions” in template parlance(用语)) provide you, the template author, with the ability to control the flow of a template’s generation. Helm’s template language provides the following control structures:

- `if/else` for creating conditional blocks
- `with` to specify a scope
- `range`, which provides a “for each”-style loop

In addition to these, it provides a few actions for declaring and using named template segments:

- `define` declares a new named template inside of your template
- `template` imports a named template
- `block` declares a special kind of fillable template area

In this section, we’ll talk about `if`, `with`, and `range`. The others are covered in the “Named Templates” section later in this guide.

#### IF/ELSE

The first control structure we’ll look at is for conditionally including blocks of text in a template. This is the `if`/`else` block.

The basic structure for a conditional looks like this:

```shell
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

Notice that we’re now talking about `pipelines` instead of values. The reason for this is to make it clear that control structures can execute an entire pipeline, not just evaluate a value.

A pipeline is evaluated as **false** if the value is:

- a boolean false
- a numeric zero
- an empty string
- a `nil` (empty or null)
- an empty collection (`map`, `slice`, `tuple`, `dict`, `array`)

In any other case, the condition is evaluated to true and the pipeline is executed.

Let’s add a simple conditional to our ConfigMap. We’ll add another setting if the drink is set to coffee:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if and .Values.favorite.drink (eq .Values.favorite.drink "coffee") }}mug: true{{ end }}
```

Note that `.Values.favorite.drink` must be defined or else it will throw an error when comparing it to “coffee”. Since we commented out `drink: coffee` in our last example, the output should not include a `mug: true` flag. But if we add that line back into our `values.yaml` file, the output should look like this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

#### CONTROLLING WHITESPACE

While we’re looking at conditionals, we should take a quick look at the way whitespace is controlled in templates. Let’s take the previous example and format it to be a little easier to read:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
```

Initially, this looks good. But if we run it through the template engine, we’ll get an unfortunate result:

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

What happened? We generated incorrect YAML because of the whitespacing above.

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

mug is incorrectly indented. Let’s simply out-dent that one line, and re-run:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{end}}
```

When we sent that, we’ll get YAML that is valid, but still looks a little funny:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true

```

Notice that we received a few empty lines in our YAML. Why? When the template engine runs, it removes the contents inside of `{{` and `}}`, but it leaves the remaining whitespace exactly as is.

YAML ascribes meaning to whitespace, so managing the whitespace becomes pretty important. Fortunately, Helm templates have a few tools to help.

First, the curly brace syntax(花括号语法) of template declarations can be modified with special characters to tell the template engine to chomp(吃掉，去除) whitespace. `{{-` (with the dash(破折号) and space(空格) added) indicates that whitespace should be chomped left, while `-}}` means whitespace to the right should be consumed. ***Be careful! Newlines are whitespace!***

Make sure there is a **space** between the `-` and the rest of your directive. `{{- 3 }}` means “trim left whitespace and print 3” while `{{-3}}` means “print -3”.

Using this syntax, we can modify our template to get rid of those new lines:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

Just for the sake of making this point clear, let’s adjust the above, and substitute an `*` for each whitespace that will be deleted following this rule. an `*` at the end of the line indicates a newline character that would be removed

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}*
**{{- if eq .Values.favorite.drink "coffee"}}
  mug: true*
**{{- end}}
```

Keeping that in mind, we can run our template through Helm and see the result:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clunky-cat-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

Be careful with the chomping modifiers. It is easy to accidentally do things like this:

```shell
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: true
  {{- end -}}

```

That will produce `food: "PIZZA"mug:true` because it consumed newlines on both sides.

For the details on whitespace control in templates, see the [Official Go template documentation](https://godoc.org/text/template)

Finally, sometimes it’s easier to tell the template system how to indent(缩进) for you instead of trying to master the spacing of template directives. For that reason, you may sometimes find it useful to use the `indent` function (`{{indent 2 "mug:true"}}`).

#### MODIFYING SCOPE USING `WITH`

The next control structure to look at is the `with` action. This controls variable scoping. Recall that `.` is a reference to the current scope. So `.Values` tells the template to find the `Values` object in the current scope.

The syntax for `with` is similar to a simple `if` statement:

```shell
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

Scopes can be changed. `with` can allow you to set the current scope (`.`) to a particular object. For example, we’ve been working with `.Values.favorites`. Let’s rewrite our ConfigMap to alter the `.` scope to point to `.Values.favorites`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

(Note that we removed the `if` conditional from the previous exercise)

Notice that now we can reference `.drink` and `.food` without qualifying them. That is because the `with` statement sets `.` to point to `.Values.favorite`. The `.` is reset to its previous scope after `{{ end }}`.

But here’s a note of caution! Inside of the restricted scope, you will not be able to access the other objects from the parent scope. This, for example, will fail:

```shell
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```

It will produce an error because `Release.Name` is not inside of the restricted scope for `.`. However, if we swap the last two lines, all will work as expected because the scope is reset after `{{end}}`.

```shell
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  release: {{ .Release.Name }}
```

After looking at `range`, we will take a look at template variables, which offers one solution to the scoping issue above.

#### LOOPING WITH THE `RANGE` ACTION

Many programming languages have support `for` looping using for loops, `foreach` loops, or similar functional mechanisms. In Helm’s template language, the way to iterate(重复) through a collection is to use the `range` operator.

To start, let’s add a list of pizza toppings to our `values.yaml` file:

```shell
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

Now we have a list (called a `slice` in templates) of `pizzaToppings`. We can modify our template to print this list into our ConfigMap:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

Let’s take a closer look at the `toppings`: list. The `range` function will “range over” (iterate through) the `pizzaToppings` list. But now something interesting happens. Just like `with` sets the scope of ., so does a `range` operator. Each time through the loop, . is set to the current pizza topping. That is, the first time, `.` is set to `mushrooms`. The second iteration it is set to `cheese`, and so on.

We can send the value of `.` directly down a pipeline, so when we do `{{ . | title | quote }}`, it sends `.` to `title` (title case function) and then to `quote`. If we run this template, the output will be:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

Now, in this example we’ve done something tricky(棘手的). The `toppings: |-` line is declaring a multi-line string. So our list of toppings is actually not a YAML list. It’s a big string. Why would we do this? Because the data in ConfigMaps `data` is composed of key/value pairs, where both the key and the value are simple strings. To understand why this is the case, take a look at the [Kubernetes ConfigMap docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). For us, though, this detail doesn’t matter much.

> The `|-` marker in YAML takes a multi-line string. This can be a useful technique for embedding big blocks of data inside of your manifests, as exemplified here.

Sometimes it’s useful to be able to quickly make a list inside of your template, and then iterate over that list. Helm templates have a function that’s called just that: `list`.

```shell
  sizes: |-
    {{- range list "small" "medium" "large" }}
    - {{ . }}
    {{- end }}
```

The above will produce this:

```shell
  sizes: |-
    - small
    - medium
    - large
```

In addition to lists, `range` can be used to iterate over collections that have a key and a value (like a `map` or `dict`). We’ll see how to do that in the next section when we introduce template variables.

### Variables

With functions, pipelines, objects, and control structures under our belts, we can turn to one of the more basic ideas in many programming languages: variables. In templates, they are less frequently used. But we will see how to use them to simplify code, and to make better use of `with` and `range`.

In an earlier example, we saw that this code will fail:

```shell
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```

`Release.Name` is not inside of the scope that’s restricted in the `with` block. One way to work around scoping issues is to assign objects to variables that can be accessed without respect to the present scope.

In Helm templates, a variable is a named reference to another object. It follows the form `$name`. Variables are assigned with a special assignment operator: `:=`. We can rewrite the above to use a variable for `Release.Name`.

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

Notice that before we start the `with` block, we assign `$relname := .Release.Name`. Now inside of the `with` block, the `$relname` variable still points to the release name.

Running that will produce this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: viable-badger-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: viable-badger
```

Variables are particularly useful in `range` loops. They can be used on list-like objects to capture both the index and the value:

```shell
  toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}
```

Note that `range` comes first, then the variables, then the assignment operator, then the list. This will assign the integer index (starting from zero) to `$index` and the value to `$topping`. Running it will produce:

```shell
  toppings: |-
      0: mushrooms
      1: cheese
      2: peppers
      3: onions
```

For data structures that have both a key and a value, we can use `range` to get both. For example, we can loop through `.Values.favorite` like this:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end}}
```

Now on the first iteration, `$key` will be `drink` and `$val` will be `coffee`, and on the second, `$key` will be `food` and `$val` will be `pizza`. Running the above will generate this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eager-rabbit-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

Variables are normally not “global”. They are scoped to the block in which they are declared. Earlier, we assigned `$relname` in the top level of the template. That variable will be in scope for the entire template. But in our last example, `$key` and `$val` will only be in scope inside of the `{{range...}}{{end}}` block.

However, there is one variable that is always global - `$` - this variable will always point to the root context. This can be very useful when you are looping in a range and need to know the chart’s release name.

An example illustrating this:

```shell
{{- range .Values.tlsSecrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .name }}
  labels:
    # Many helm templates would use `.` below, but that will not work,
    # however `$` will work here
    app.kubernetes.io/name: {{ template "fullname" $ }}
    # I cannot reference .Chart.Name, but I can do $.Chart.Name
    helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
    app.kubernetes.io/instance: "{{ $.Release.Name }}"
    # Value from appVersion in Chart.yaml
    app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
    app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
type: kubernetes.io/tls
data:
  tls.crt: {{ .certificate }}
  tls.key: {{ .key }}
---
{{- end }}
```

So far we have looked at just one template declared in just one file. But one of the powerful features of the Helm template language is its ability to declare multiple templates and use them together. We’ll turn to that in the next section.

### Named Templates

It is time to move beyond one template, and begin to create others. In this section, we will see how to define ***named templates*** in one file, and then use them elsewhere. A ***named template*** (sometimes called a ***partial*** or a ***subtemplate***) is simply a template defined inside of a file, and given a name. We’ll see two ways to create them, and a few different ways to use them.

In the “Flow Control” section we introduced three actions for declaring and managing templates: `define`, `template`, and `block`. In this section, we’ll cover those three actions, and also introduce a special-purpose `include` function that works similarly to the `template` action.

An important detail to keep in mind when naming templates: **template names are global**. If you declare two templates with the same name, whichever one is loaded last will be the one used. Because templates in subcharts are compiled together with top-level templates, you should be careful to name your templates with **chart-specific names**.

One popular naming convention is to prefix each defined template with the name of the chart: `{{ define "mychart.labels" }}`. By using the specific chart name as a prefix we can avoid any conflicts that may arise due to two different charts that implement templates of the same name.

#### PARTIALS AND `_` FILES

So far, we’ve used one file, and that one file has contained a single template. But Helm’s template language allows you to create named embedded templates, that can be accessed by name elsewhere.

Before we get to the nuts-and-bolts(螺母和螺栓) of writing those templates, there is file naming convention that deserves mention:

- Most files in `templates/` are treated as if they contain Kubernetes manifests
- The `NOTES.txt` is one exception
- But files whose name begins with an underscore (`_`) are assumed to ***not*** have a manifest inside. These files are not rendered to Kubernetes object definitions, but are available everywhere within other chart templates for use.

These files are used to store partials and helpers. In fact, when we first created `mychart`, we saw a file called `_helpers.tpl`. That file is the default location for template partials.

#### DECLARING AND USING TEMPLATES WITH `DEFINE` AND `TEMPLATE`

The `define` action allows us to create a named template inside of a template file. Its syntax goes like this:

```shell
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

For example, we can define a template to encapsulate(封装) a Kubernetes block of labels:

```shell
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

Now we can embed this template inside of our existing ConfigMap, and then include it with the `template` action:

```shell
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

When the template engine reads this file, it will store away the reference to `mychart.labels` until `template "mychart.labels"` is called. Then it will render that template inline. So the result will look like this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

Conventionally(按照惯例), Helm charts put these templates inside of a partials file, usually `_helpers.tpl`. Let’s move this function there:

```shell
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

By convention, `define` functions should have a simple documentation block (`{{/* ... */}}`) describing what they do.

Even though this definition is in `_helpers.tpl`, it can still be accessed in `configmap.yaml`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

As mentioned above, **template names are global**. As a result of this, if two templates are declared with the same name the last occurrence will be the one that is used. Since templates in subcharts are compiled together with top-level templates, it is best to name your templates with chart specific names. A popular naming convention is to prefix each defined template with the name of the chart: `{{ define "mychart.labels" }}`.

#### SETTING THE SCOPE OF A TEMPLATE

In the template we defined above, we did not use any objects. We just used functions. Let’s modify our defined template to include the chart name and chart version:

```shell
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

If we render this, the result will not be what we expect:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart:
    version:
```

What happened to the name and version? They weren’t in the scope for our defined template. When a named template (created with `define`) is rendered, it will receive the scope passed in by the `template` call. In our example, we included the template like this:

```shell
{{- template "mychart.labels" }}
```

No scope was passed in, so within the template we cannot access anything in `.`. This is easy enough to fix, though. We simply pass a scope to the template:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```

Note that we pass `.` at the end of the `template` call. We could just as easily pass `.Values` or `.Values.favorite` or whatever scope we want. But what we want is the top-level scope.

Now when we execute this template with `helm install --dry-run --debug ./mychart`, we get this:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plinking-anaco-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart: mychart
    version: 0.1.0
```

Now `{{ .Chart.Name }}` resolves to `mychart`, and `{{ .Chart.Version }}` resolves to `0.1.0`.

#### THE `INCLUDE` FUNCTION

Say we’ve defined a simple template that looks like this:

```shell
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}
```

Now say I want to insert this both into the `labels:` section of my template, and also the `data:` section:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" .}}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

The output will not be what we expect:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
app_name: mychart
app_version: "0.1.0+1478129847"
```

Note that the indentation on `app_version` is wrong in both places. Why? Because the template that is substituted(替代的) in has the text aligned to the right. Because `template` is an action, and not a function, there is no way to pass the output of a `template` call to other functions; the data is simply inserted inline.

To work around this case, Helm provides an alternative to `template` that will import the contents of a template into the present pipeline where it can be passed along to other functions in the pipeline.

Here’s the example above, corrected to use `nindent` to indent the `mychart_app` template correctly:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{- include "mychart.app" . | nindent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
  {{- include "mychart.app" . | nindent 2 }}
```

Now the produced YAML is correctly indented for each section:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```

> It is considered preferable to use `include` over `template` in Helm templates simply so that the output formatting can be handled better for YAML documents.

Sometimes we want to import content, but not as templates. That is, we want to import files verbatim(逐字). We can achieve this by accessing files through the `.Files` object described in the next section.

### Accessing Files Inside Templates

In the previous section we looked at several ways to create and access named templates. This makes it easy to import one template from within another template. But sometimes it is desirable to import a ***file that is not a template*** and inject its contents without sending the contents through the template renderer.

Helm provides access to files through the `.Files` object. Before we get going with the template examples, though, there are a few things to note about how this works:

- It is okay to add extra files to your Helm chart. These files will be bundled and sent to Tiller. Be careful, though. Charts must be smaller than 1M because of the storage limitations of Kubernetes objects.
- Some files cannot be accessed through the `.Files` object, usually for security reasons.
  - Files in `templates/` cannot be accessed.
  - Files excluded using `.helmignore` cannot be accessed.
- Charts do not preserve(保留) UNIX mode information, so file-level permissions will have no impact on the availability of a file when it comes to the `.Files` object.

- [Basic example](https://helm.sh/docs/chart_template_guide/#basic-example)
- [Path helpers](https://helm.sh/docs/chart_template_guide/#path-helpers)
- [Glob patterns](https://helm.sh/docs/chart_template_guide/#glob-patterns)
- [ConfigMap and Secrets utility functions](https://helm.sh/docs/chart_template_guide/#configmap-and-secrets-utility-functions)
- [Encoding](https://helm.sh/docs/chart_template_guide/#encoding)
- [Lines](https://helm.sh/docs/chart_template_guide/#lines)

#### BASIC EXAMPLE

With those caveats(注意事项) behind, let’s write a template that reads three files into our ConfigMap. To get started, we will add three files to the chart, putting all three directly inside of the `mychart/` directory.

`config1.toml`:

```shell
message = "Hello from config 1"
```

`config2.toml`:

```shell
message = "This is config 2"
```

`config3.toml`:

```shell
message = "Goodbye from config 3"
```

Each of these is a simple TOML file (think old-school Windows INI files). We know the names of these files, so we can use a `range` function to loop through them and inject their contents into our ConfigMap.

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range list "config1.toml" "config2.toml" "config3.toml" }}
  {{ . }}: |-
    {{ $files.Get . }}
  {{- end }}
```

This config map uses several of the techniques discussed in previous sections. For example, we create a `$files` variable to hold a reference to the `.Files` object. We also use the `list` function to create a list of files that we loop through. Then we print each file name (`{{.}}: |-`) followed by the contents of the file `{{ $files.Get . }}`.

Running this template will produce a single ConfigMap with the contents of all three files:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quieting-giraf-configmap
data:
  config1.toml: |-
    message = "Hello from config 1"

  config2.toml: |-
    message = "This is config 2"

  config3.toml: |-
    message = "Goodbye from config 3"
```

#### PATH HELPERS

When working with files, it can be very useful to perform some standard operations on the file paths themselves. To help with this, Helm imports many of the functions from Go’s [path](https://golang.org/pkg/path/) package for your use. They are all accessible with the same names as in the Go package, but with a lowercase first letter. For example, `Base` becomes `base`, etc.

The imported functions are:

- Base
- Dir
- Ext
- IsAbs
- Clean

#### GLOB PATTERNS

As your chart grows, you may find you have a greater need to organize your files more, and so we provide a `Files.Glob(pattern string)` method to assist in extracting certain files with all the flexibility of [glob patterns](https://godoc.org/github.com/gobwas/glob).

`.Glob` returns a `Files` type, so you may call any of the `Files` methods on the returned object.

For example, imagine the directory structure:

```shell
foo/:
  foo.txt foo.yaml

bar/:
  bar.go bar.conf baz.yaml
```

You have multiple options with Globs:

```shell
{{ $root := . }}
{{ range $path, $bytes := .Files.Glob "**.yaml" }}
{{ $path }}: |-
{{ $root.Files.Get $path }}
{{ end }}
```

Or

```shell
{{ $root := . }}
{{ range $path, $bytes := .Files.Glob "foo/*" }}
{{ base $path }}: '{{ $root.Files.Get $path | b64enc }}'
{{ end }}
```

#### CONFIGMAP AND SECRETS UTILITY FUNCTIONS

(Not present in version 2.0.2 or prior(之前的))

It is very common to want to place file content into both configmaps and secrets, for mounting into your pods at run time. To help with this, we provide a couple utility methods on the `Files` type.

For further organization, it is especially useful to use these methods in conjunction with the `Glob` method.

Given the directory structure from the [Glob](https://helm.sh/docs/chart_template_guide/#glob-patterns) example above:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
  {{- (.Files.Glob "foo/*").AsConfig | nindent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
  {{- (.Files.Glob "bar/*").AsSecrets | nindent 2 }}
```

#### ENCODING

You can import a file and have the template base-64 encode it to ensure successful transmission:

```shell
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  token: |-
    {{ .Files.Get "config1.toml" | b64enc }}
```

The above will take the same `config1.toml` file we used before and encode it:

```shell
# Source: mychart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucky-turkey-secret
type: Opaque
data:
  token: |-
    bWVzc2FnZSA9IEhlbGxvIGZyb20gY29uZmlnIDEK
```

#### LINES

Sometimes it is desirable to access each line of a file in your template. We provide a convenient `Lines` method for this.

```shell
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```

Currently, there is no way to pass files external to the chart during `helm install`. So if you are asking users to supply data, it must be loaded using `helm install -f` or `helm install --set`.

This discussion wraps up our dive into the tools and techniques for writing Helm templates. In the next section we will see how you can use one special file, `templates/NOTES.txt`, to send post-installation instructions to the users of your chart.

### Creating a NOTES.txt File

In this section we are going to look at Helm’s tool for providing instructions to your chart users. At the end of a `helm install` or `helm upgrade`, Helm can print out a block of helpful information for users. This information is highly customizable using templates.

To add installation notes to your chart, simply create a `templates/NOTES.txt` file. This file is plain text, but it is processed like as a template, and has all the normal template functions and objects available.

Let’s create a simple `NOTES.txt` file:

```txt
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get {{ .Release.Name }}

```

Now if we run `helm install ./mychart` we will see this message at the bottom:

```shell
RESOURCES:
==> v1/Secret
NAME                   TYPE      DATA      AGE
rude-cardinal-secret   Opaque    1         0s

==> v1/ConfigMap
NAME                      DATA      AGE
rude-cardinal-configmap   3         0s


NOTES:
Thank you for installing mychart.

Your release is named rude-cardinal.

To learn more about the release, try:

  $ helm status rude-cardinal
  $ helm get rude-cardinal
```

Using `NOTES.txt` this way is a great way to give your users detailed information about how to use their newly installed chart. Creating a `NOTES.txt` file is strongly recommended, though it is not required.

### Subcharts and Global Values

To this point we have been working only with one chart. But charts can have dependencies, called ***subcharts***, that also have their own values and templates. In this section we will create a subchart and see the different ways we can access values from within templates.

Before we dive into the code, there are a few important details to learn about subcharts.

1. A subchart is considered “stand-alone”, which means a subchart can never explicitly(明确的) depend on its parent chart.
2. For that reason, a subchart cannot access the values of its parent.
3. A parent chart can override values for subcharts.
4. Helm has a concept of global values that can be accessed by all charts.

As we walk through the examples in this section, many of these concepts will become clearer.

#### CREATING A SUBCHART

For these exercises, we’ll start with the `mychart/` chart we created at the beginning of this guide, and we’ll add a new chart inside of it.

```shell
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*.*
```

Notice that just as before, we deleted all of the base templates so that we can start from scratch. In this guide, we are focused on how templates work, not on managing dependencies. But the [Charts Guide](https://helm.sh/docs/chart_template_guide/#../charts) has more information on how subcharts work.

#### ADDING VALUES AND A TEMPLATE TO THE SUBCHART

Next, let’s create a simple template and values file for our `mysubchart` chart. There should already be a `values.yaml` in `mychart/charts/mysubchart`. We’ll set it up like this:

```shell
dessert: cake
```

Next, we’ll create a new ConfigMap template in `mychart/charts/mysubchart/templates/configmap.yaml`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

Because every subchart is a stand-alone chart, we can test mysubchart on its own:

```shell
$ helm install --dry-run --debug mychart/charts/mysubchart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart/charts/mysubchart
NAME:   newbie-elk
TARGET NAMESPACE:   default
CHART:  mysubchart 0.1.0
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: newbie-elk-cfgmap2
data:
  dessert: cake
```

#### OVERRIDING VALUES OF A CHILD CHART

Our original chart, `mychart` is now the ***parent*** chart of `mysubchart`. This relationship is based entirely on the fact that `mysubchart` is within `mychart/charts`.

Because `mychart` is a parent, we can specify configuration in `mychart` and have that configuration pushed into `mysubchart`. For example, we can modify `mychart/values.yaml` like this:

```shell
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

Note the last two lines. Any directives inside of the `mysubchart` section will be sent to the `mysubchart` chart. So if we run `helm install --dry-run --debug mychart`, one of the things we will see is the `mysubchart` ConfigMap:

```shell
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-bee-cfgmap2
data:
  dessert: ice cream
```

The value at the top level has now overridden the value of the subchart.

There’s an important detail to notice here. We didn’t change the template of `mychart/charts/mysubchart/templates/configmap.yaml` to point to `.Values.mysubchart.dessert`. From that template’s perspective, the value is still located at `.Values.dessert`. As the template engine passes values along, it sets the scope. So for the `mysubchart` templates, only values specifically for `mysubchart` will be available in `.Values`.

Sometimes, though, you do want certain values to be available to all of the templates. This is accomplished using global chart values.

#### GLOBAL CHART VALUES

Global values are values that can be accessed from any chart or subchart by exactly the same name. Globals require explicit declaration. You can’t use an existing non-global as if it were a global.

The Values data type has a reserved section called `Values.global` where global values can be set. Let’s set one in our `mychart/values.yaml` file.

```shell
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

Because of the way globals work, both `mychart/templates/configmap.yaml` and `mychart/charts/mysubchart/templates/configmap.yaml` should be able to access that value as `{{ .Values.global.salad}}`.

`mychart/templates/configmap.yaml`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  salad: {{ .Values.global.salad }}
```

`mychart/charts/mysubchart/templates/configmap.yaml`:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

Now if we run a dry run install, we’ll see the same value in both outputs:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-configmap
data:
  salad: caesar

---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```

Globals are useful for passing information like this, though it does take some planning to make sure the right templates are configured to use globals.

#### SHARING TEMPLATES WITH SUBCHARTS

Parent charts and subcharts can share templates. Any defined block in any chart is available to other charts.

For example, we can define a simple template like this:

```shell
{{- define "labels" }}from: mychart{{ end }}
```

Recall how the labels on templates are ***globally shared***. Thus, the `labels` chart can be included from any other chart.

While chart developers have a choice between `include` and `template`, one advantage of using `include` is that `include` can dynamically reference templates:

```shell
{{ include $mytemplate }}
```

The above will dereference `$mytemplate`. The `template` function, in contrast(相反地), will only accept a string literal.

#### AVOID USING BLOCKS

The Go template language provides a · keyword that allows developers to provide a default implementation which is overridden later. In Helm charts, blocks are not the best tool for overriding because if multiple implementations of the same block are provided, the one selected is unpredictable.

The suggestion is to instead use `include`.

### Debugging Templates

Debugging templates can be tricky(棘手的) simply because the templates are rendered on the Tiller server, not the Helm client. And then the rendered templates are sent to the Kubernetes API server, which may reject the YAML files for reasons other than formatting.

There are a few commands that can help you debug.

- `helm lint` is your go-to tool for verifying that your chart follows best practices
- `helm install --dry-run --debug`: We’ve seen this trick already. It’s a great way to have the server render your templates, then return the resulting manifest file.
- `helm get manifest`: This is a good way to see what templates are installed on the server.

When your YAML is failing to parse, but you want to see what is generated, one easy way to retrieve the YAML is to comment out the problem section in the template, and then re-run `helm install --dry-run --debug`:

```shell
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

The above will be rendered and returned with the comments intact:

```shell
apiVersion: v1
# some: problem section
#  "bar"
```

This provides a quick way of viewing the generated content without YAML parse errors blocking.

### Wrapping Up

This guide is intended to give you, the chart developer, a strong understanding of how to use Helm’s template language. The guide focuses on the technical aspects of template development.

But there are many things this guide has not covered when it comes to the practical day-to-day development of charts. Here are some useful pointers to other documentation that will help you as you create new charts:

- The [Helm Charts project](https://github.com/helm/charts) is an indispensable(必不可少的) source of charts. That project is also sets the standard for best practices in chart development.
- The Kubernetes [Documentation](https://kubernetes.io/docs/home/) provides detailed examples of the various resource kinds that you can use, from ConfigMaps and Secrets to DaemonSets and Deployments.
- The Helm [Charts Guide](https://helm.sh/docs/chart_template_guide/#../charts) explains the workflow of using charts.
- The Helm [Chart Hooks Guide](https://helm.sh/docs/chart_template_guide/#../charts_hooks) explains how to create lifecycle hooks.
- The Helm [Charts Tips and Tricks](https://helm.sh/docs/chart_template_guide/#../chart-development-tips-and-tricks) article provides some useful tips for writing charts.
- The [Sprig documentation](https://github.com/Masterminds/sprig) documents more than sixty of the template functions.
- The [Go template docs](https://godoc.org/text/template) explain the template syntax in detail.
- The [Schelm tool](https://github.com/databus23/schelm) is a nice helper utility for debugging charts.

Sometimes it’s easier to ask a few questions and get answers from experienced developers. The best place to do this is in the [Kubernetes Slack](https://kubernetes.slack.com/) Helm channels:

- [#helm-users](https://kubernetes.slack.com/messages/helm-users)
- [#helm-dev](https://kubernetes.slack.com/messages/helm-dev)
- [#charts](https://kubernetes.slack.com/messages/charts)

Finally, if you find errors or omissions in this document, want to suggest some new content, or would like to contribute, visit [The Helm Project](https://github.com/helm/helm).

## YAML Techniques

Most of this guide has been focused on writing the template language. Here, we’ll look at the YAML format. YAML has some useful features that we, as template authors, can use to make our templates less error prone and easier to read.

### SCALARS AND COLLECTIONS

According to the [YAML spec](https://yaml.org/spec/1.2/spec.html), there are two types of collections, and many scalar types.

The two types of collections are maps and sequences:

```shell
map:
  one: 1
  two: 2
  three: 3

sequence:
  - one
  - two
  - three
```

Scalar values(标量值) are individual values (as opposed to collections)

### Scalar Types in YAML

In Helm’s dialect(方言) of YAML, the scalar data type of a value is determined by a complex set of rules, including the Kubernetes schema for resource definitions. But when inferring types, the following rules tend to hold true.

If an integer or float is an unquoted bare word, it is typically treated as a numeric type:

```shell
count: 1
size: 2.34
```

But if they are quoted, they are treated as strings:

```shell
count: "1" # <-- string, not int
size: '2.34' # <-- string, not float
```

The same is true of booleans:

```shell
isGood: true   # bool
answer: "true" # string
```

The word for an empty value is `null` (not `nil`).

Note that `port: "80"` is valid YAML, and will pass through both the template engine and the YAML parser, but will fail if Kubernetes expects `port` to be an integer.

In some cases, you can force a particular type inference(推理) using YAML node tags:

```shell
coffee: "yes, please"
age: !!str 21
port: !!int "80"
```

In the above, `!!str` tells the parser that `age` is a string, even if it looks like an int. And `port` is treated as an int, even though it is quoted.

### STRINGS IN YAML

Much of the data that we place in YAML documents are strings. YAML has more than one way to represent a string. This section explains the ways and demonstrates how to use some of them.

There are three “inline” ways of declaring a string:

```shell
way1: bare words
way2: "double-quoted strings"
way3: 'single-quoted strings'
```

All inline styles must be on one line.

- Bare words are unquoted, and are not escaped. For this reason, you have to be careful what characters you use.
- Double-quoted strings can have specific characters escaped with `\`. For example `"\"Hello\", she said"`. You can escape line breaks with `\n`.
- Single-quoted strings are “literal” strings, and do not use the `\` to escape characters. The only escape sequence is `''`, which is decoded as a single `'`.

In addition to the one-line strings, you can declare multi-line strings:

```shell
coffee: |
  Latte
  Cappuccino
  Espresso
```

The above will treat the value of `coffee` as a single string equivalent to `Latte\nCappuccino\nEspresso\n`.

Note that the first line after the `|` must be correctly indented. So we could break the example above by doing this:

```shell
coffee: |
         Latte
  Cappuccino
  Espresso
```

Because `Latte` is incorrectly indented, we’d get an error like this:

```shell
Error parsing file: error converting YAML to JSON: yaml: line 7: did not find expected key
```

In templates, it is sometimes safer to put a fake “first line” of content in a multi-line document just for protection from the above error:

```shell
coffee: |
  # Commented first line
         Latte
  Cappuccino
  Espresso
```

Note that whatever that first line is, it will be preserved in the output of the string. So if you are, for example, using this technique to inject a file’s contents into a ConfigMap, the comment should be of the type expected by whatever is reading that entry.

### Controlling Spaces in Multi-line Strings

In the example above, we used `|` to indicate a multi-line string. But notice that the content of our string was followed with a trailing `\n`. If we want the YAML processor to strip off the trailing newline, we can add a `-` after the `|`:

```shell
coffee: |-
  Latte
  Cappuccino
  Espresso
```

Now the `coffee` value will be: `Latte\nCappuccino\nEspresso` (with no trailing `\n`).

Other times, we might want all trailing whitespace to be preserved(保留所有尾随的空白). We can do this with the `|+` notation:

```shell
coffee: |+
  Latte
  Cappuccino
  Espresso


another: value
```

Now the value of `coffee` will be `Latte\nCappuccino\nEspresso\n\n\n`.

Indentation inside of a text block is preserved, and results in the preservation of line breaks, too(保留文本块内部的缩进，并且也导致换行符的保留):

```shell
coffee: |-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

In the above case, `coffee` will be `Latte\n 12 oz\n 16 oz\nCappuccino\nEspresso`.

### Indenting and Templates

When writing templates, you may find yourself wanting to inject the contents of a file into the template. As we saw in previous chapters, there are two ways of doing this:

- Use `{{ .Files.Get "FILENAME" }}` to get the contents of a file in the chart.
- Use `{{ include "TEMPLATE" . }}` to render a template and then place its contents into the chart.
When inserting files into YAML, it’s good to understand the multi-line rules above. Often times, the easiest way to insert a static file is to do something like this:

```shell
myfile: |
{{ .Files.Get "myfile.txt" | indent 2 }}
```

Note how we do the indentation above: `indent 2` tells the template engine to indent every line in “myfile.txt” with two spaces. Note that we do not indent that template line. That’s because if we did, the file content of the first line would be indented twice.

### Folded Multi-line Strings

Sometimes you want to represent a string in your YAML with multiple lines, but want it to be treated as one long line when it is interpreted(被解释). This is called “folding”. To declare a folded block, use `>` instead of `|`:

```shell
coffee: >
  Latte
  Cappuccino
  Espresso

```

The value of `coffee` above will be `Latte Cappuccino Espresso\n`. Note that all but the last line feed will be converted to spaces. You can combine the whitespace controls with the folded text marker, so `>-` will replace or trim all newlines.

Note that in the folded syntax, indenting text will cause lines to be preserved.

```shell
coffee: >-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

The above will produce `Latte\n 12 oz\n 16 oz\nCappuccino Espresso`. Note that both the spacing and the newlines are still there.

### EMBEDDING MULTIPLE DOCUMENTS IN ONE FILE

It is possible to place more than one YAML documents into a single file. This is done by prefixing a new document with `---` and ending the document with `...`

```shell
---
document:1
...
---
document: 2
...
```

In many cases, either the `---` or the `...` may be omitted.

Some files in Helm cannot contain more than one doc. If, for example, more than one document is provided inside of a `values.yaml` file, only the first will be used.

Template files, however, may have more than one document. When this happens, the file (and all of its documents) is treated as one object during template rendering. But then the resulting YAML is split into multiple documents before it is fed to Kubernetes.

We recommend only using multiple documents per file when it is absolutely necessary. Having multiple documents in a file can be difficult to debug.

### YAML IS A SUPERSET OF JSON

Because YAML is a superset of JSON, any valid JSON document should be valid YAML.

```json
{
  "coffee": "yes, please",
  "coffees": [
    "Latte", "Cappuccino", "Espresso"
  ]
}
```

The above is another way of representing this:

```yaml
coffee: yes, please
coffees:
- Latte
- Cappuccino
- Espresso
```

And the two can be mixed (with care):

```yaml
coffee: "yes, please"
coffees: [ "Latte", "Cappuccino", "Espresso"]
```

All three of these should parse into the same internal representation.

While this means that files such as `values.yaml` may contain JSON data, Helm does not treat the file extension `.json` as a valid suffix.

### YAML ANCHORS(锚))

The YAML spec provides a way to store a reference to a value, and later refer to that value by reference. YAML refers to this as “anchoring”:

```yaml
coffee: "yes, please"
favorite: &favoriteCoffee "Cappucino"
coffees:
  - Latte
  - *favoriteCoffee
  - Espresso
```

In the above, `&favoriteCoffee` sets a reference to `Cappuccino`. Later, that reference is used as `*favoriteCoffee`. So `coffees` becomes `Latte, Cappuccino, Espresso`.

While there are a few cases where anchors are useful, there is one aspect of them that can cause subtle bugs: The first time the YAML is consumed, the reference is expanded and then discarded.

So if we were to decode and then re-encode the example above, the resulting YAML would be:

```yaml
coffee: yes, please
favorite: Cappucino
coffees:
- Latte
- Cappucino
- Espresso
```

Because Helm and Kubernetes often read, modify, and then rewrite YAML files, the anchors will be lost.

## Appendix: Go Data Types and Templates

The Helm template language is implemented in the strongly typed Go programming language. For that reason, variables in templates are ***typed***. For the most part, variables will be exposed as one of the following types:

- string: A string of text
- bool: a true or false
- int: An integer value (there are also 8, 16, 32, and 64 bit signed and unsigned variants of this)
- float64: a 64-bit floating point value (there are also 8, 16, and 32 bit varieties of this)
- a byte slice ([]byte), often used to hold (potentially) binary data
- struct: an object with properties and methods
- a slice (indexed list) of one of the previous types
- a string-keyed map (map[string]interface{}) where the value is one of the previous types

There are many other types in Go, and sometimes you will have to convert between them in your templates. The easiest way to debug an object’s type is to pass it through `printf "%t"` in a template, which will print the type. Also see the `typeOf` and `kindOf` functions.

