# Kubernetes-Dashboard-V1.10.1-官方文档阅读笔记


> 参考：  
> <https://github.com/kubernetes/dashboard>

Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.

## Total View

### Getting Started

**IMPORTANT:** Read the [Access Control](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md) guide before performing any further steps. The default Dashboard deployment contains a minimal set of RBAC privileges needed to run.

To deploy Dashboard, execute following command:

```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

To access Dashboard from your local workstation you must create a secure channel to your Kubernetes cluster. Run the following command:

```sh
$ kubectl proxy
```

Now access Dashboard at:

[`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`](
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/).

### Create An Authentication Token (RBAC)

To find out how to create sample user and log in follow [Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md) guide.

**NOTE:**

* Kubeconfig Authentication method does not support external identity providers or certificate-based authentication.
* Dashboard can only be accessed over HTTPS
* [Heapster](https://github.com/kubernetes/heapster/) has to be running in the cluster for the metrics and graphs to be available. Read more about it in [Integrations](https://github.com/kubernetes/dashboard/blob/master/docs/user/integrations.md) guide.

### Documentation

Dashboard documentation can be found on [docs](https://github.com/kubernetes/dashboard/blob/master/docs/README.md) directory which contains:

* [Common](https://github.com/kubernetes/dashboard/blob/master/docs/common/README.md): Entry-level overview
* [User Guide](https://github.com/kubernetes/dashboard/blob/master/docs/user/README.md): [Installation](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md), [Accessing Dashboard](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md) and more for users
* [Developer Guide](https://github.com/kubernetes/dashboard/blob/master/docs/developer/README.md): [Getting Started](https://github.com/kubernetes/dashboard/blob/master/docs/developer/getting-started.md), [Dependency Management](https://github.com/kubernetes/dashboard/blob/master/docs/developer/dependency-management.md) and more for anyone interested in contributing

## Kubernetes Dashboard Documentation

### [Common](https://github.com/kubernetes/dashboard/blob/master/docs/common/README.md)

* [FAQ](https://github.com/kubernetes/dashboard/blob/master/docs/common/faq.md)
* [Roadmap](https://github.com/kubernetes/dashboard/blob/master/docs/common/roadmap.md)
* [Dashboard arguments](https://github.com/kubernetes/dashboard/blob/master/docs/common/dashboard-arguments.md)

### [User Guide](https://github.com/kubernetes/dashboard/blob/master/docs/user/README.md)

* [Installation](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md)
* [Certificate management](https://github.com/kubernetes/dashboard/blob/master/docs/user/certificate-management.md)
* [Accessing Dashboard](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md)
  * [1.7.x and above](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.7.x-and-above.md)
  * [1.6.x and below](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.6.x-and-below.md)
* [Access control](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md)
  * [Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)
* [Integrations](https://github.com/kubernetes/dashboard/blob/master/docs/user/integrations.md)
* [Labels](https://github.com/kubernetes/dashboard/blob/master/docs/user/labels.md)

### [Developer Guide](https://github.com/kubernetes/dashboard/blob/master/docs/developer/README.md)

* [Getting started](https://github.com/kubernetes/dashboard/blob/master/docs/developer/getting-started.md)
* [Release procedures](https://github.com/kubernetes/dashboard/blob/master/docs/developer/release-procedures.md)
* [Dependency management](https://github.com/kubernetes/dashboard/blob/master/docs/developer/dependency-management.md)
* [Architecture](https://github.com/kubernetes/dashboard/blob/master/docs/developer/architecture.md)
* [Code conventions](https://github.com/kubernetes/dashboard/blob/master/docs/developer/code-conventions.md)
* [Text conventions](https://github.com/kubernetes/dashboard/blob/master/docs/developer/text-conventions.md)
* [Internationalization](https://github.com/kubernetes/dashboard/blob/master/docs/developer/internationalization.md)
* [Plugins](https://github.com/kubernetes/dashboard/blob/master/docs/plugins/README.md)

#### Common

##### Dashboard arguments

Dashboard container accepts multiple arguments that can be used to customize it a bit. In example we are using `--auto-generate-certificates` flag in our recommended setup [YAML files](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml) to pass certificates to Dashboard.

###### Arguments

| Argument name | Default value | Description |
|---------------|---------------|-------------|
| insecure-port | 9090          | The port to listen to for incoming HTTP requests. |
| port          | 8443          | The secure port to listen to for incoming HTTPS requests. |
| insecure-bind-address | 127.0.0.1 | The IP address on which to serve the `--insecure-port` (set to 127.0.0.1 for all interfaces). |
| bind-address  | 0.0.0.0       | The IP address on which to serve the `--port` (set to 0.0.0.0 for all interfaces). |
| default-cert-dir | /certs     | Directory path containing `--tls-cert-file` and `--tls-key-file` files. Used also when auto-generating certificates flag is set. Relative to the container, not the host. |
| tls-cert-file | -             | File containing the default x509 Certificate for HTTPS. |
| tls-key-file  | -             | File containing the default x509 private key matching --tls-cert-file. |
| auto-generate-certificates | false | When set to true, Dashboard will automatically generate certificates used to serve HTTPS. |
| apiserver-host | -            | The address of the Kubernetes Apiserver to connect to in the format of protocol://address:port, e.g., http://localhost:8080. If not specified, the assumption is that the binary runs inside a Kubernetes cluster and local discovery is attempted. |
| api-log-level | INFO          | Level of API request logging. Should be one of 'INFO|NONE|DEBUG'. |
| heapster-host | -             | The address of the Heapster Apiserver to connect to in the format of protocol://address:port, e.g., http://localhost:8082. If not specified, the assumption is that the binary runs inside a Kubernetes cluster and service proxy will be used. |
| sidecar-host  | -             | The address of the Sidecar Apiserver to connect to in the format of protocol://address:port, e.g., http://localhost:8000. If not specified, the assumption is that the binary runs inside a Kubernetes cluster and service proxy will be used.
| metrics-provider | sidecar    | Select provider type for metrics. 'none' will not check metrics. |
| metric-client-check-period | 30 | Time in seconds that defines how often configured metric client health check should be run. |
| kubeconfig    | -             | Path to kubeconfig file with authorization and master location information. |
| namespace     | kube-system   | When non-default namespace is used, create encryption key in the specified namespace. |
| token-ttl     | 900           | Expiration time (in seconds) of JWE tokens generated by dashboard. '0' never expires.
| authentication-mode | token   | Enables authentication options that will be reflected on login screen. Supported values: token, basic. Note that basic option should only be used if apiserver has '--authorization-mode=ABAC' and '--basic-auth-file' flags set. |
| enable-insecure-login | false | When enabled, Dashboard login view will also be shown when Dashboard is not served over HTTPS. |
| enable-skip-login | false | When enabled, the skip button on the login page will be shown. |
| disable-settings-authorizer | false | When enabled, Dashboard settings page will not require user to be logged in and authorized to access settings page. |
| locale-config | ./locale_conf.json |File containing the configuration of locales.
| system-banner | -             | When non-empty displays message to Dashboard users. Accepts simple HTML tags. |
| system-banner-severity | INFO | Severity of system banner. Should be one of 'INFO|WARNING|ERROR'. |

#### User Guide

##### Installation

###### Official release

**IMPORTANT:** Before upgrading from older version of Dashboard to 1.7+ make sure to delete Cluster Role Binding for `kubernetes-dashboard` Service Account, otherwise Dashboard will have full admin access to the cluster.

***Quick setup***

The fastest way of deploying Dashboard has been described in our [README](https://github.com/kubernetes/dashboard/blob/master/README.md). It is destined for people that are new to Kubernetes and want to quickly start using Dashboard. Other possible setups for more experienced users, that want to know more about our deployment procedure can be found below.

***Recommended setup***

To access Dashboard directly (without `kubectl proxy`) valid certificates should be used to establish a secure HTTPS connection. They can be generated using public trusted Certificate Authorities like [Let's Encrypt](https://letsencrypt.org/). Use them to replace the auto-generated certificates from Dashboard.

By default self-signed certificates are generated and stored in-memory. In case you would like to use your custom certificates follow the below steps, otherwise skip directly to the Dashboard deploy part.

Custom certificates have to be stored in a secret named `kubernetes-dashboard-certs` in the same namespace as Kubernetes Dashboard. Assuming that you have `dashboard.crt` and `dashboard.key` files stored under `$HOME/certs` directory, you should create secret with contents of these files:

```console
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
```

Afterwards, you are ready to deploy Dashboard using the following command:

```console
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml
```

****Alternative setup****

This setup is not fully secure. Certificates are not used and Dashboard is exposed only over HTTP. In this setup access control can be ensured only by using [Authorization Header](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#authorization-header) feature.

To deploy Dashboard execute following command:

```console
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/alternative.yaml
```

###### Development release

Besides official releases, there are also development releases, that are pushed after every successful master build. It is not advised to use them on production environment as they are less stable than the official ones. Following sections describe installation and discovery of development releases.

***Installation***

In most of the use cases you need to execute the following command to deploy latest development release:

```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/head.yaml
```

***Update***

Once installed, the deployment is not automatically updated. In order to update it you need to delete the deployment's pods and wait for it to be recreated. After recreation, it should use the latest image.

Delete all Dashboard pods (assuming that Dashboard is deployed in kubernetes-dashboard namespace):

```console
kubectl -n kubernetes-dashboard delete $(kubectl -n kubernetes-dashboard get pod -o name | grep dashboard)
pod "dashboard-metrics-scraper-fb986f88d-gnfnk" deleted
pod "kubernetes-dashboard-7d8b9cc8d-npljm" deleted
```

##### Certificate management

This document describes shortly how to get certificates, that can be used to enable HTTPS in Dashboard. There are two steps required to do it:

1. Generate certificates.
    1. [Public trusted CA](#public-trusted-certificate-authority).
    2. [Self-signed certificate](#self-signed-certificate).
2. Pass them to Dashboard.
    1. In case you are following [Recommended Setup](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#recommended-setup) to deploy Dashboard just generate certificates and follow it.
    2. In any other case you need to alter Dashboard's YAML deploy file and pass --tls-key-file and --tls-cert-file flags to Dashboard. More information about how to mount them into the pods can be found [here](https://kubernetes.io/docs/concepts/storage/volumes/).

###### Public trusted Certificate Authority

There are many public and free certificate providers to choose from. One of the best trusted certificate providers is [Let's encrypt](https://letsencrypt.org/). Everything you need to know about how to generate certificates signed by their trusted CA can be found [here](https://letsencrypt.org/getting-started/).

###### Self-signed certificate

In case you want to generate certificates on your own you need library like [OpenSSL](https://www.openssl.org/) that will help you do that.

***Generate private key and certificate signing request***

A private key and certificate signing request are required to create an SSL certificate. These can be generated with a few simple commands. When the openssl req command asks for a “challenge password”, just press return, leaving the password empty. This password is used by Certificate Authorities to authenticate the certificate owner when they want to revoke their certificate. Since this is a self-signed certificate, there’s no way to revoke it via CRL (Certificate Revocation List).

```console
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
...
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
# Writing RSA key
rm dashboard.pass.key
openssl req -new -key dashboard.key -out dashboard.csr
...
Country Name (2 letter code) [AU]: US
...
A challenge password []:
...
```

***Generate SSL certificate***

The self-signed SSL certificate is generated from the `dashboard.key` private key and `dashboard.csr` files.

```console
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
```

The `dashboard.crt` file is your certificate suitable for use with Dashboard along with the `dashboard.key` private key.

##### Accessing Dashboard

Once Dashboard is installed on your cluster it can be accessed in a few different ways. Note that this document does not describe all possible ways of accessing cluster applications. In case of any error while trying to access Dashboard, please first read our [FAQ](https://github.com/kubernetes/dashboard/blob/master/docs/common/faq.md) and check [closed issues](https://github.com/kubernetes/dashboard/issues?q=is%3Aissue+is%3Aclosed). In most cases errors are caused by cluster configuration issues.

Choose version of Dashboard you are using to get information about how to access it:

* [1.7.x and above](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.7.x-and-above.md)
* [1.6.x and below](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.6.x-and-below.md)

###### Accessing Dashboard 1.7.x and above

**IMPORTANT:** HTTPS endpoints are only available if you used [Recommended Setup](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#recommended-setup), followed [Getting Started](https://github.com/kubernetes/dashboard/blob/master/README.md#getting-started) guide to deploy Dashboard or manually provided `--tls-key-file` and `--tls-cert-file` flags. In case you did not and you access Dashboard over HTTP, then Dashboard can be accessed the same way as [older versions](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.6.x-and-below.md).

**NOTE:** Dashboard should not be exposed publicly over HTTP. For domains accessed over HTTP it will not be possible to sign in. Nothing will happen after clicking Sign in button on login page.

***`kubectl proxy`***

`kubectl proxy` creates proxy server between your machine and Kubernetes API server. By default it is only accessible locally (from the machine that started it).

First let's check if `kubectl` is properly configured and has access to the cluster. In case of error follow [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install and set up `kubectl`.

```console
$ kubectl cluster-info
# Example output
Kubernetes master is running at https://192.168.30.148:6443
KubeDNS is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Start local proxy server.

```console
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Once the proxy server is started you should be able to access Dashboard from your browser.

To access the HTTPS endpoint of dashboard go to: `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

NOTE: Dashboard should not be exposed publicly using `kubectl proxy` command as it only allows HTTP connection. For domains other than `localhost` and `127.0.0.1` it will not be possible to sign in. Nothing will happen after clicking `Sign in` button on login page.

***`kubectl port-forward`***

Instead of `kubectl proxy`, you can use `kubectl port-forward` and access dashboard with simpler URL than using `kubectl proxy`.

```console
kube@minikube:~$ kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 10443:443
Forwarding from 127.0.0.1:10443 -> 8443
```

NOTE: If you need to expose dashboard to other hosts besides the host running `kubectl`, you should set the `--address [ IP address that your browser's host has ]` option. You can set this option to `--address 0.0.0.0`, but it means publicly exposing the dashboard.

***NodePort***

This way of accessing Dashboard is only recommended for development environments in a single node setup.

Edit `kubernetes-dashboard` service.

```console
$ kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```

You should see `yaml` representation of the service. Change `type: ClusterIP` to `type: NodePort` and save file. If it's already changed go to next step.

```console
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Next we need to check port on which Dashboard was exposed.

```console
$ kubectl -n kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   NodePort   10.100.124.90   <nodes>       443:31707/TCP   21h
```

Dashboard has been exposed on port `31707 (HTTPS)`. Now you can access it from your browser at: `https://<master-ip>:31707`. `master-ip` can be found by executing `kubectl cluster-info`. Usually it is either `127.0.0.1` or IP of your machine, assuming that your cluster is running directly on the machine, on which these commands are executed.

In case you are trying to expose Dashboard using `NodePort` on a multi-node cluster, then you have to find out IP of the node on which Dashboard is running to access it. Instead of accessing `https://<master-ip>:<nodePort>` you should access `https://<node-ip>:<nodePort>`.

***API Server***

In case Kubernetes API server is exposed and accessible from outside you can directly access dashboard at: `https://<master-ip>:<apiserver-port>/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

**Note:** This way of accessing Dashboard is only possible if you choose to install your user certificates in the browser. In example certificates used by kubeconfig file to contact API Server can be used.

***Ingress***

Dashboard can be also exposed using Ingress resource. For more information check: https://kubernetes.io/docs/concepts/services-networking/ingress.

###### Accessing Dashboard 1.6.x and below

***`kubectl proxy`***

`kubectl proxy` creates proxy server between your machine and Kubernetes API server. By default it is only accessible locally (from the machine that started it).

First let's check if `kubectl` is properly configured and has access to the cluster. In case of error follow [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install and set up `kubectl`.

```console
$ kubectl cluster-info
# Example output
Kubernetes master is running at https://192.168.30.148:6443
Heapster is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Start local proxy server.

```console
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Once proxy server is started you should be able to access Dashboard from your browser at: `http://localhost:8001/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/`.

***NodePort***

This way of accessing Dashboard is only recommended for development environments in a single node setup.

Edit `kubernetes-dashboard` service.

```console
$ kubectl -n kube-system edit service kubernetes-dashboard
```

You should see `yaml` representation of the service. Change `type: ClusterIP` to `type: NodePort` and save file. If it's already changed go to next step.

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2017-09-11T08:00:46Z
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "1300"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
  uid: 51392867-96c7-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.103.169.125
  externalTrafficPolicy: Cluster
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Next we need to check port on which Dashboard was exposed.

```console
$ kubectl -n kube-system get service kubernetes-dashboard
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   10.103.169.125   <nodes>       80:32703/TCP   1d
```

Dashboard has been exposed on a port `32703`. Now you can access it from your browser at: `http://<master-ip>:32703`. `master-ip` can be found by executing `kubectl cluster-info`. Usually it is either `127.0.0.1` or IP of your machine, assuming that you cluster is running directly on the machine, on which these commands are executed.

***API Server***

In case Kubernetes API server is exposed and accessible from outside you can directly access dashboard at: `http(s)://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/`.

***Ingress***

Dashboard can be also exposed using Ingress resource. For more information check: https://kubernetes.io/docs/concepts/services-networking/ingress.

##### Access control

Once Dashboard is installed and accessible we can focus on configuring access control to the cluster resources for users. As of release 1.7 Dashboard no longer has full admin privileges granted by default. All the privileges are revoked and only [minimal privileges granted](#default-dashboard-privileges), that are required to make Dashboard work.

**IMPORTANT:** This note is only directed to people using Dashboard 1.7 and above. In case Dashboard is accessible only by trusted set of people, all with full admin privileges you may want to grant it [admin privileges](#admin-privileges). Note that other applications should not access Dashboard directly as it may cause privileges escalation. Make sure that in-cluster traffic is restricted to namespaces or just revoke access to Dashboard for other applications inside the cluster.

###### Introduction

Kubernetes supports few ways of authenticating and authorizing users. You can read about them [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) and [here](https://kubernetes.io/docs/reference/access-authn-authz/authorization/). Authorization is handled by Kubernetes API server. Dashboard only acts as a proxy and passes all auth information to it. In case of forbidden access corresponding warnings will be displayed in Dashboard.

###### Default Dashboard privileges

***v1.7***

* `create` and `watch` permissions for secrets in `kube-system` namespace required to create and watch for changes of `kubernetes-dashboard-key-holder` secret.
* `get`, `update` and `delete` permissions for secrets named `kubernetes-dashboard-key-holder` and `kubernetes-dashboard-certs` in `kube-system` namespace.
* `proxy` permission to `heapster` service in `kube-system` namespace required to allow getting metrics from heapster.

***v1.8***

* `create` permission for secrets in `kube-system` namespace required to create `kubernetes-dashboard-key-holder` secret.
* `get`, `update` and `delete` permissions for secrets named `kubernetes-dashboard-key-holder` and `kubernetes-dashboard-certs` in `kube-system` namespace.
* `get` and `update` permissions for config map named `kubernetes-dashboard-settings` in `kube-system` namespace.
* `proxy` permission to `heapster` service in `kube-system` namespace required to allow getting metrics from heapster.

***v1.10***

_T.B.D._

***v2.0***

_T.B.D._

***Authentication***

As of release 1.7 Dashboard supports user authentication based on:

* [`Authorization: Bearer <token>`](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#authorization-header) header passed in every request to Dashboard. Supported from release 1.6. Has the highest priority. If present, login view will not be shown.
* [Bearer Token](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#bearer-token) that can be used on Dashboard [login view](#login-view).
* [Username/password](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#basic) that can be used on Dashboard [login view](#login-view).
* [Kubeconfig](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#kubeconfig) file that can be used on Dashboard [login view](#login-view).

***Login view***

Login view has been introduced in release 1.7. In case you are using the latest recommended installation then login functionality will be enabled by default. In any other case and if you prefer to configure certificates manually you need to pass `--tls-cert-file` and `--tls-cert-key` flags to Dashboard. HTTPS endpoint will be exposed on port `8443` of Dashboard container. You can change it by providing `--port` flag.

Using `Skip` option will make Dashboard use privileges of Service Account used by Dashboard. `Skip` button is disabled by default since 1.10.1. Use `--enable-skip-login` dashboard flag to display it.

***Authorization header***

Using authorization header is the only way to make Dashboard act as an user, when accessing it over HTTP. Note that there are some risks since plain HTTP traffic is vulnerable to [MITM attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack).

To make Dashboard use authorization header you simply need to pass `Authorization: Bearer <token>` in every request to Dashboard. This can be achieved i.e. by configuring reverse proxy in front of Dashboard. Proxy will be responsible for authentication with identity provider and will pass generated token in request header to Dashboard. Note that Kubernetes API server needs to be configured properly to accept these tokens.

To quickly test it check out [Requestly](https://chrome.google.com/webstore/detail/requestly-redirect-url-mo/mdnleldcmiljblolnjhpnblkcekpdkpa) Chrome browser plugin that allows to manually modify request headers.

**IMPORTANT:** Authorization header will not work if Dashboard is accessed through API server proxy. Both `kubectl proxy` and `API Server` way of accessing Dashboard described in [Accessing Dashboard](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md) guide will not work. It is due to the fact that once request reaches API server all additional headers are dropped.

***Bearer Token***

It is recommended to get familiar with [Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) documentation first to find out how to get token, that can be used to login. In example every Service Account has a Secret with valid Bearer Token that can be used to login to Dashboard.

Recommended lecture to find out how to create Service Account and grant it privileges:

* [Service Account Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)
* [Role and ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)
* [Service Account Permissions](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions)

***Sample Bearer Token***

To create sample user and to get its token, see [Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md) guide.

***Getting token with `kubectl`***

There are many Service Accounts created in Kubernetes by default. All with different access permissions. In order to find any token, that can be used to login we'll use `kubectl`:

```console
# Check existing secrets in kubernetes-dashboard namespace
$ kubectl -n kubernetes-dashboard get secret
NAME                               TYPE                                  DATA   AGE
default-token-2pjhm                kubernetes.io/service-account-token   3      81m
kubernetes-dashboard-certs         Opaque                                0      81m
kubernetes-dashboard-csrf          Opaque                                1      81m
kubernetes-dashboard-key-holder    Opaque                                2      81m
kubernetes-dashboard-token-x9nd8   kubernetes.io/service-account-token   3      81m

$ kubectl -n kubernetes-dashboard describe secrets kubernetes-dashboard-token-x9nd8
Name:         kubernetes-dashboard-token-x9nd8
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: 2140a425-447f-437f-9966-24ab4e57217a

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi14OW5kOCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjIxNDBhNDI1LTQ0N2YtNDM3Zi05OTY2LTI0YWI0ZTU3MjE3YSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.oSOjJZpQq-yiAIQWM12gFpVA6jiJz8-zApC0Wbet9iwzflmCVFlT1lWjEEduKMnJOF-viJ4fwLixA3INfCxDgWIBmxEvoA-R6ExQNkmFi4ljGdBX98fI2B4WFuqWIPoEjqf1l3eXHKmXgqbiMYA-UH_Ih4m2-aKKO3dfkmc5HmPP1ZjotCQKGpcq60c1y-SASqbC_FC3LHvp0l5N9bfhAOraNC_34ZlL3zkQ6cAL6mZG8Ci1MuXMHTH9g04QaVZb14f6BAY-K2X-Z5yDpMr4Zs5h6DOc_18sysf4uOVyo0wMXfI9gLsda-e3zX_5W39piBj-PwfBwBGslC_JztTCSQ
ca.crt:     1066 bytes
namespace:  20 bytes
```

We can now use printed `token` to login to Dashboard. To find out more about how to configure and use Bearer Tokens, please read [Introduction](#introduction) section.

***Basic***
Basic authentication is disabled by default. The reason is that Kubernetes API server needs to be configured with authorization mode ABAC and `--basic-auth-file` flag provided. Without that API server automatically falls back to [anonymous user](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests) and there is no way to check if provided credentials are valid.

In order to enable basic auth in Dashboard `--authentication-mode=basic` flag has to be provided. By default it is set to `--authentication-mode=token`.

***Kubeconfig***

This method of logging in is provided for convenience. Only authentication options specified by `--authentication-mode` flag are supported in kubeconfig file. In case it is configured to use any other way, error will be shown in Dashboard. External identity providers or certificate-based authentication are not supported at this time.

###### Admin privileges

**IMPORTANT:** Make sure that you know what you are doing before proceeding. Granting admin privileges to Dashboard's Service Account might be a security risk.

You can grant full admin privileges to Dashboard's Service Account by creating below `ClusterRoleBinding`. Copy the YAML file based on chosen installation method and save as, i.e. `dashboard-admin.yaml`. Use `kubectl create -f dashboard-admin.yaml` to deploy it. Afterwards you can use `Skip` option on login page to access Dashboard.

***Official release***

```console
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
```

***Development release***

```console
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-head
  namespace: kubernetes-dashboard-head
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard-head
    namespace: kubernetes-dashboard-head
```

##### Creating sample user

In this guide, we will find out how to create a new user using Service Account mechanism of Kubernetes, grant this user admin permissions and login to Dashboard using bearer token tied to this user.

**IMPORTANT:** Make sure that you know what you are doing before proceeding. Granting admin privileges to Dashboard's Service Account might be a security risk.

Copy following snippets for `ServiceAccount` and `ClusterRoleBinding` to new manifest file like `dashboard-adminuser.yaml` and use `kubectl apply -f dashboard-adminuser.yaml` to create them.

## Create Service Account

We are creating Service Account with name `admin-user` in namespace `kubernetes-dashboard` first.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

## Create ClusterRoleBinding

In most cases after provisioning our cluster using `kops` or `kubeadm` or any other popular tool, the `ClusterRole` `admin-Role` already exists in the cluster. We can use it and create only `ClusterRoleBinding` for our `ServiceAccount`.

**NOTE:** `apiVersion` of `ClusterRoleBinding` resource may differ between Kubernetes versions. Prior to Kubernetes `v1.8` the `apiVersion` was `rbac.authorization.k8s.io/v1beta1`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

## Bearer Token

Now we need to find token we can use to log in. Execute following command:

```console
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

It should print something like:

```console
Name:         admin-user-token-v57nw
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 0303243c-4040-4a58-8a47-849ee9ba79c1

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXY1N253Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwMzAzMjQzYy00MDQwLTRhNTgtOGE0Ny04NDllZTliYTc5YzEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.Z2JrQlitASVwWbc-s6deLRFVk5DWD3P_vjUFXsqVSY10pbjFLG4njoZwh8p3tLxnX_VBsr7_6bwxhWSYChp9hwxznemD5x5HLtjb16kI9Z7yFWLtohzkTwuFbqmQaMoget_nYcQBUC5fDmBHRfFvNKePh_vSSb2h_aYXa8GV5AcfPQpY7r461itme1EXHQJqv-SN-zUnguDguCTjD80pFZ_CmnSE1z9QdMHPB8hoB4V68gtswR1VLa6mSYdgPwCHauuOobojALSaMc3RH7MmFUumAgguhqAkX3Omqd3rJbYOMRuMjhANqd08piDC3aIabINX6gP5-Tuuw2svnV6NYQ
```

Now copy the token and paste it into `Enter token` field on login screen.

Click `Sign in` button and that's it. You are now logged in as an admin.

In order to find out more about how to grant/deny permissions in Kubernetes read official [authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) & [authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) documentation.

