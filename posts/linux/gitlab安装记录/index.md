# Gitlab安装记录


使用 k8s 安装 gitlab https://docs.gitlab.com/charts/

gitlab 官方的 helm chart 地址：https://gitlab.com/gitlab-org/charts/gitlab

下载 master 分支的 chart

```console
mkdir /home/k8s/gitlab && cd /home/k8s/gitlab
wget https://gitlab.com/gitlab-org/charts/gitlab/-/archive/master/gitlab-master.tar.gz
tar -zxf gitlab-master.tar.gz
```

查看 chart 里的一些说明，如：**`gitlab-master/doc/advanced/external-nginx/index.md`** 中指出了自定义 gitlab 的 ingress 选项， 如果 k8s 集群中有多个 nginx ingress 控制器，则 Ingress 使用 annotation 来标志哪一个控制器实现它，如果你不想使用 git lab chart 自带的 nginx ingress 控制器， 而是想使用其他的 nginx ingress 控制器，可以通过 `global.ingress.class` 进行指定。

多个 nginx ingress controller 参考：<https://github.com/kubernetes/ingress-nginx/blob/c3ce6b892e5f1cc1066f81c19482dded2901ad45/docs/user-guide/multiple-ingress.md>

如果你的 k8s 集群中已有一个 nginx ingress 控制器了，你不想 gitlab chart 再安装一个 nginx ingress 控制器，可以通过 `nginx-ingress.enabled=false` 来让 gitlab chart 不安装 nginx ingress 控制器。

该文件里还指出了自定义证书管理器的说明：如果你的 k8s 集群里已经有一个额外的 ingress 控制器了，那么你可能也已有一个额外的 cert-manager 实例或其他管理证书的工具。如果你的 k8s 集群中已经有 cert-manager 了，不想 gitlab chart 安装，可以通过 `certmanager.install=false` 来让 gitlab chart 不安装 cert-manager。并且通过设置 `global.ingress.configureCertmanager=false` 来让 gitlab chart 不去创建证书。详细的文档可以参考：`https://gitlab.com/gitlab-org/charts/gitlab/blob/master/doc/installation/tls.md`。

查看 **`gitlab-master/doc/installation/tls.md`**，里面说如果你已经安装了 cert-manager ，可以通过使用 `global.ingress.annotations` 来为你的cert-manager 部署配置适合的 annotations。本文已经安装 nginx ingress 控制器、 cert-manager 了，并在 cert-manager 配置了 Issuer ，所以需要看 `External cert-manager and Issuer (external)`，内容如下

```markdown
To make use of an external `cert-manager` and `Issuer` resource you must provide several items, so that self-signed certificates
are not activated.

1. Annotations to activate the external `cert-manager` (see [documentation][cm-annotations] for further details)
1. Names of TLS secrets for each service (this deactivates [self-signed behaviors](#option-4-use-auto-generated-self-signed-wildcard-certificate))


helm install ...
  --set cert-manager.install=false \
  --set global.ingress.configureCertmanager=false \
  --set global.ingress.annotations."kubernetes\.io/tls-acme"=true \
  --set gitlab.unicorn.ingress.tls.secretName=RELEASE-gitlab-tls \
  --set registry.ingress.tls.secretName=RELEASE-registry-tls \
  --set minio.ingress.tls.secretName=RELEASE-minio-tls

```

查看 **`gitlab-master/doc/charts/globals.md`**，里面说明了怎么指定 annotations 和 configureCertmanager的作用。

本文使用的 cert-manager 自动生成证书的 annotation 为 `cert-manager.io/cluster-issuer: cluster-issuer`，需要替换下，然后下面的三个 secretName 是 cluster-issuer 生成的 Ingress 使用的 secret （用于 https）。

综上，我们使用如下命令生成 gitlab chart 的清单文件

```console
[root@k8s-m1 gitlab]# helm template ./gitlab-master --name mygitlab --set global.edition=ce --set global.time_zone=CST+8 --set global.hosts.domain=k8s.abc --set nginx-ingress.enabled=false --set global.ingress.class=nginx --set prometheus.install=false --set certmanager.install=false --set global.ingress.configureCertmanager=false --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer --set gitlab.unicorn.ingress.tls.secretName=RELEASE-gitlab-tls --set registry.ingress.tls.secretName=RELEASE-registry-tls --set minio.ingress.tls.secretName=RELEASE-minio-tls --namespace gitlab > gitlab.yaml
```

报错了

```console
Error: found in requirements.yaml, but missing in charts/ directory: cert-manager, prometheus, postgresql, gitlab-runner, grafana
```

因为 gitlab chart 依赖这些chart，使用如下命令下载依赖

```console
helm dep up ./gitlab-master
```

因为这些子 chart 的地址是 `kubernetes-charts.storage.googleapis.com`，国内无法访问，所以可能下载失败。

可以通过配置代理的方式，实现下载这些子 chart，在 /etc/profile 里添加 `export http_proxy="http://192.168.14.58:8580"`，然后再执行 source /etc/profile，这台 linux 通过代理下载子chart 了。

如果想要去除这个代理，在 /etc/profile 里将 http_proxy 修改为 `export http_proxy=`，然后执行 source /etc/profile 就可以了。

注意：不要配置 https_proxy，如果配置了可能会报错 `proxyconnect tcp: tls: first record does not look like a TLS handshake`。

```console
[root@k8s-m1 gitlab]# helm dep up ./gitlab-master
Hang tight while we grab the latest from your chart repositories...
...Unable to get an update from the "local" chart repository (http://127.0.0.1:8879/charts):
  Get http://127.0.0.1:8879/charts/index.yaml: dial tcp 127.0.0.1:8879: connect: connection refused
...Unable to get an update from the "harbor" chart repository (https://helm.goharbor.io):
  Get https://helm.goharbor.io/index.yaml: EOF
...Unable to get an update from the "stable" chart repository (https://kubernetes-charts.storage.googleapis.com):
  Get https://kubernetes-charts.storage.googleapis.com/index.yaml: EOF
...Unable to get an update from the "pingcap" chart repository (https://charts.pingcap.org/):
  Get https://charts.pingcap.org/index.yaml: unexpected EOF
...Unable to get an update from the "istio.io" chart repository (https://storage.googleapis.com/istio-release/releases/1.3.1/charts/):
  Get https://storage.googleapis.com/istio-release/releases/1.3.1/charts/index.yaml: EOF
...Unable to get an update from the "rancher-stable" chart repository (https://releases.rancher.com/server-charts/stable):
  Get https://releases.rancher.com/server-charts/stable/index.yaml: EOF
...Unable to get an update from the "elastic" chart repository (https://helm.elastic.co):
  Get https://helm.elastic.co/index.yaml: EOF
...Unable to get an update from the "gitlab" chart repository (https://charts.gitlab.io/):
  Get https://charts.gitlab.io/index.yaml: EOF
Update Complete.
Saving 5 charts
Downloading cert-manager from repo https://kubernetes-charts.storage.googleapis.com/
Downloading prometheus from repo https://kubernetes-charts.storage.googleapis.com/
Downloading postgresql from repo https://kubernetes-charts.storage.googleapis.com/
Downloading gitlab-runner from repo https://charts.gitlab.io/
Downloading grafana from repo https://kubernetes-charts.storage.googleapis.com/
Deleting outdated charts
[root@k8s-m1 gitlab]# cd gitlab-master/charts/
[root@k8s-m1 charts]# ls
certmanager-issuer       gitlab-runner-0.10.0.tgz  nginx                  redis     shared-secrets
cert-manager-v0.4.0.tgz  grafana-3.8.15.tgz        postgresql-0.12.0.tgz  redis-ha
gitlab                   minio                     prometheus-5.5.3.tgz   registry
```

已经成功下载 `cert-manager-v0.4.0.tgz`、`gitlab-runner-0.10.0.tgz`、`grafana-3.8.15.tgz`、`postgresql-0.12.0.tgz`、`prometheus-5.5.3.tgz` 到 `gitlab-master/charts/` 目录下。

再次生成清单文件

```console
[root@k8s-m1 gitlab]# helm template ./gitlab-master --name mygitlab --set global.edition=ce --set global.time_zone=CST+8 --set global.hosts.domain=k8s.abc --set nginx-ingress.enabled=false --set global.ingress.class=nginx --set prometheus.install=false --set certmanager.install=false --set global.ingress.configureCertmanager=false  --set gitlab.unicorn.ingress.tls.secretName=RELEASE-gitlab-tls --set registry.ingress.tls.secretName=RELEASE-registry-tls --set minio.ingress.tls.secretName=RELEASE-minio-tls --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer --namespace gitlab > gitlab.yaml
```

解释下: 安装社区版gitlab；将时区设置为 CST+8，表示中国标准时间；域名设置为 k8s.abc，生成的 ingress 里的域名会自动加上前缀，如 gitlab.k8s.abc；不安装 nginx ingress controller ; 在 Ingress 上配置使用名为 nginx 的 nginx ingress controller，这个是默认的 nginx ingress controller 的名字，也是本文的 nginx ingress controller 的名字；不安装 prometheus；不安装cert-manager；configureCertmanager置为false，就不需要输入cert-manager 验证用的邮箱了，而且也不会自动生成对应的 secret ，如果向要生成 secret 的话需要指定名字，其后的三条就是指定了 secret 的名字；annotations 这条是通知 cert-manager 的 shim 组件去根据 secret 的名字生成对应的 ingress 使用的 https 的 secret ；最后一条是执行安装在 gitlab 命名空间内。

新建 gitlab 命名空间

```console
kubectl create ns gitlab
```

执行清单文件，安装 gitlab

```console
kubectl apply -f gitlab.yaml
```

部署后发现，postgresql 和 gitlab-runner 仍然运行在 default 命名空间内，删除部署

```console
kubectl delete -f gitlab.yaml
```

修改下 gitlab.yaml，为没有配置 namespace 的 配置上 namespace。
在配置上 imagePullPolicy: IfNotPresent，chart里推荐 `'Always' if imageTag is 'latest', else set to 'IfNotPresent'`

时区修改为Beijing，如果是CST+8，会报错`ArgumentError: Invalid Timezone: CST+8`
<https://stackoverflow.com/questions/31565999/invalid-timezone>

再次部署，发现 cert-manager 没有生成 ingress 里的 secret ，查看 cert-manager 的日志，发现

```console
I1031 12:59:32.896141       1 controller.go:129] cert-manager/controller/ingress-shim "level"=0 "msg"="syncing item" "key"="gitlab/mygitlab-minio" 
E1031 12:59:32.903598       1 controller.go:131] cert-manager/controller/ingress-shim "msg"="re-queuing item  due to error processing" "error"="Certificate.cert-manager.io \"RELEASE-minio-tls\" is invalid: metadata.name: Invalid value: \"RELEASE-minio-tls\": a DNS-1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')" "key"="gitlab/mygitlab-minio" 
I1031 12:59:33.217104       1 controller.go:129] cert-manager/controller/ingress-shim "level"=0 "msg"="syncing item" "key"="gitlab/mygitlab-registry"
```

原来 secret 名不符合要求，再修改下，将 secret 中的大写字母改为小写

再次部署，发现 pod mygitlab-migrations.0-55445 报错

```console
2019-10-31T13:00:21.000Z 8 TID-gmpl6bw2c WARN: PG::ConnectionBad: could not connect to server: No route to host
  Is the server running on host "mygitlab-postgresql" (10.105.112.88) and accepting
  TCP/IP connections on port 5432?
```

因为这个 pod 在 c48-1 主机上，这台主机开启了 firewalld 防火墙，将 tcp 5432 端口开放就不报错了。
不理解为什么要开启这个端口， pod 访问 postgresql 应该是走的 k8s 的内网，而且 c48-1 上的 tcp 5432 端口未被监听。

再次查看 gitlab 的 pod 的状态

```console
[root@k8s-m1 gitlab]# kubectl get pods -n gitlab -o wide
NAME                                           READY   STATUS             RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
mygitlab-gitaly-0                              1/1     Running            0          31m   172.30.81.181   k8s-m2   <none>           <none>
mygitlab-gitlab-exporter-787d755bd7-zg6wx      1/1     Running            0          31m   172.30.77.33    c48-2    <none>           <none>
mygitlab-gitlab-runner-79df86778c-f4shq        0/1     CrashLoopBackOff   7          31m   172.30.182.35   c48-1    <none>           <none>
mygitlab-gitlab-shell-845977ff9c-9g7jh         1/1     Running            0          31m   172.30.42.135   k8s-m1   <none>           <none>
mygitlab-gitlab-shell-845977ff9c-z6zzn         1/1     Running            0          31m   172.30.77.37    c48-2    <none>           <none>
mygitlab-gitlab-upgrade-check-g2hps            0/1     Completed          0          31m   172.30.182.31   c48-1    <none>           <none>
mygitlab-migrations.0-55445                    0/1     Completed          0          31m   172.30.77.35    c48-2    <none>           <none>
mygitlab-minio-cdf46ff8-7ss5d                  1/1     Running            0          31m   172.30.182.29   c48-1    <none>           <none>
mygitlab-postgresql-5d49d8c57d-8jbqs           2/2     Running            0          31m   172.30.77.36    c48-2    <none>           <none>
mygitlab-redis-6b9477b6d8-4cqtc                2/2     Running            0          31m   172.30.42.136   k8s-m1   <none>           <none>
mygitlab-registry-85c79f7866-6v4ww             1/1     Running            0          31m   172.30.77.34    c48-2    <none>           <none>
mygitlab-registry-85c79f7866-9zn2n             1/1     Running            0          31m   172.30.81.180   k8s-m2   <none>           <none>
mygitlab-shared-secrets.0-8qk-selfsign-9xxj9   0/1     Completed          0          31m   172.30.42.137   k8s-m1   <none>           <none>
mygitlab-shared-secrets.0-ii7-6dphn            0/1     Completed          0          31m   172.30.42.132   k8s-m1   <none>           <none>
mygitlab-sidekiq-all-in-1-786cd5454f-7n7zm     1/1     Running            0          31m   172.30.182.36   c48-1    <none>           <none>
mygitlab-task-runner-66fd7cdfc9-2gddp          1/1     Running            0          31m   172.30.182.38   c48-1    <none>           <none>
mygitlab-unicorn-76c764b677-8sdwz              2/2     Running            0          31m   172.30.182.44   c48-1    <none>           <none>
mygitlab-unicorn-76c764b677-xqfk7              2/2     Running            0          31m   172.30.77.38    c48-2    <none>           <none>
mygitlab-unicorn-test-runner-nzz88             0/1     Error              0          31m   172.30.182.30   c48-1    <none>           <none>
```

pod gitlab-runner 在不停重启，查看下日志

```console
ERROR: Registering runner... failed                 runner=egNEppfW status=couldn't execute POST against https://gitlab.k8s.abc/api/v4/runners: Post https://gitlab.k8s.abc/api/v4/runners: dial tcp: lookup gitlab.k8s.abc on 10.96.0.10:53: no such host
PANIC: Failed to register this runner. Perhaps you are having network problems
Registration attempt 13 of 30
Runtime platform                                    arch=amd64 os=linux pid=246 revision=1564076b version=12.4.0
WARNING: Running in user-mode.
WARNING: The user-mode requires you to manually start builds processing:
WARNING: $ gitlab-runner run
WARNING: Use sudo for system-mode:
WARNING: $ sudo gitlab-runner...
```

在 gitlab-runner 容器中访问 gitlab.k8s.abc 不通，通过 coredns （10.96.0.10） 查不到 gitlab.k8s.abc 对应的 ip。

解决办法为 配置 coredns 的查询规则，先查 /etc/hosts 中的域名，如果查不到，再使用其他方式查询。
需要使用 coredns 的 hosts 插件，见: <https://coredns.io/plugins/hosts/>

首先将 宿主机的 /etc/hosts 文件 挂载到 coredns 容器 的 /etc/hosts 上，然后配置 coredns 的查询规则，再 Corefile 中 kubernetes 的前面添加 `hosts { fallthrouth }` ，表示解析域名时，先查 /etc/hosts 中的信息，如果查不到，就跳到下一个查询，使用 kubernetes 查询。

```yaml
... ...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        hosts {
            fallthrough
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
... ...
      containers:
      - name: coredns
        image: coredns/coredns:1.4.0
... ...
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        - name: host-time
          mountPath: /etc/localtime
          readOnly: true
        - name: hosts
          mountPath: /etc/hosts
          readOnly: true
... ...
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
        - name: host-time
          hostPath:
            path: /etc/localtime
        - name: hosts
          hostPath:
            path: /etc/hosts
---
... ...
```

更新好后，重新部署下 coredns

```console
kubectl apply -f coredns.yaml
```

然后运行一个测试的yaml文件，测试下能否解析域名，如果测试的容器的基础镜像为 centos ，就安装下 dns-utils，如果是 debian ，就安装 dnsutils 。安装后，就可以使用 nslookup 查询域名了。

如 nslookup gitlab.k8s.abc 10.96.0.10，这条查询会查 /etc/hosts
如果查service，比如查 kube-system 命名空间下的 metrics-server ，则使用命令 nslookup metrics-server.kube-system 10.96.0.10

继续，gitlab-runner 报了另一个错误

```console
ERROR: Registering runner... failed                 runner=egNEppfW status=couldn't execute POST against https://gitlab.k8s.abc/api/v4/runners: Post https://gitlab.k8s.abc/api/v4/runners: x509: certificate signed by unknown authority
PANIC: Failed to register this runner. Perhaps you are having network problems
Registration attempt 7 of 30
Runtime platform                                    arch=amd64 os=linux pid=176 revision=1564076b version=12.4.0
WARNING: Running in user-mode.
WARNING: The user-mode requires you to manually start builds processing:
WARNING: $ gitlab-runner run
WARNING: Use sudo for system-mode:
WARNING: $ sudo gitlab-runner...
```

这个错误表示 runner 向 域名 <https://gitlab.k8s.abc> 发送 post 请求，证书错误。<https://gitlab.k8s.abc> 这个域名使用的证书是上文使用 cert-manager 自动生成的，使用的是我们自己的 CA ，runner 这个pod 使用的是什么证书呢？

查看 gitlab chart 的 values.yaml中关于 runner 的部分
`doc/installation/secrets.md` 介绍了 gitlab chart 的 secret ，里面好像没有指定 runner 使用的 证书。
`doc/charts/gitlab/gitlab-runner/index.md` 介绍了 gitlab-runner ，但是里面也没有提到 runner 使用的证书，只有提到 runner 向 gitlab 注册token 和 gitlab 的 url 。

看看 gitlab-runner chart 里的信息，里面的 values.yaml 提到了一个地址， <https://docs.gitlab.com/runner/configuration/tls-self-signed.html#supported-options-for-self-signed-certificates> 支持自签名证书，这个应该就是我们要找的。里面的内容是：
从 gitlab runner 0.7.0 版本开始，允许用户配置用来验证和 gitlab server tls 连接的证书，这可以解决当注册 runner 时的 `x509: certificate signed by unknown authority` 问题。
gitlab runner 提供如下选项：

  1. 默认： gitlab runner 读取系统证书库并使用系统中的 CA 验证 gitlab server

  2. gitlab runner 从以下文件中读取 pem 证书（der 格式的证书不支持）
    - 当 gitlab runner 在 *nix 系统以 root 用户执行时对应的文件为 `/etc/gitlab-runner/certs/hostname.crt`
    - 当 gitlab runner 在 *nix 系统以非 root 用户执行时对应的文件为 `~/.gitlab-runner/certs/hostname.crt`
    - 其他系统上对应的文件为 `./certs/hostname.crt`
  
      如果 gitlab server 的地址是 <https://my.gitlab.server.com:8443/> ，则需要创建如下形式的证书：`/etc/gitlab-runner/certs/my.gitlab.server.com.crt` 。

  3. gitlab runner 在向 gitlab server [注册](https://docs.gitlab.com/runner/commands/README.html#gitlab-runner-register)时，提供了 tls-ca-file 参数（gitlab-runner register --tls-ca-file=/path），并且在 [config.toml](https://docs.gitlab.com/runner/configuration/advanced-configuration.html) 中的`[[runners]]` 部分，这两种方式允许你指定一个自定义的证书文件，这个文件需要在访问 gitlab server 的时候时刻都准备好。

我们使用第二种方式，通过进入 gitlab runner 的 pod 查看，可以查到 gitlab runner 使用的是 debian 系统，是类unix 系统，并且用户为 gitlab-runner。所以，需要在`/home/gitlab-runner/.gitlab-runner/certs/`目录下新建一个 gitlab.k8s.abc.crt，这个 crt 需要对应 上文 ingress-nginx 一节的 ca.crt。我们使用 secret 的方式进行挂载。

在 gitlab 命名空间新建 gitlab.k8s.abc.crt 对应的 secret

```console
kubectl create secret generic gitlab.k8s.abc.crt --from-file=gitlab.k8s.abc.crt=ca.crt -n gitlab
```

修改 gitlab runner 的 Deployment 文件，将该 secret 挂载到 `/home/gitlab-runner/.gitlab-runner/certs/`

```yaml
# Source: gitlab/charts/gitlab-runner/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mygitlab-gitlab-runner
  namespace: gitlab
... ...
spec:
... ...
      initContainers:
      - name: configure
        command: ['sh', '/config/configure']
        image: gitlab/gitlab-runner:alpine-v12.4.0
... ...
        volumeMounts:
        - name: runner-secrets
          mountPath: /secrets
          readOnly: false
        - name: scripts
          mountPath: /config
          readOnly: true
        - name: init-runner-secrets
          mountPath: /init-secrets
          readOnly: true
... ...
      containers:
      - name: mygitlab-gitlab-runner
        image: gitlab/gitlab-runner:alpine-v12.4.0
        imagePullPolicy: "IfNotPresent"
        command: ["/bin/bash", "/scripts/entrypoint"]
... ...
        volumeMounts:
        - name: runner-secrets
          mountPath: /secrets
        - name: etc-gitlab-runner
          mountPath: /home/gitlab-runner/.gitlab-runner
        - name: gitlab-runner-crt
          mountPath: /home/gitlab-runner/.gitlab-runner/certs/
        - name: scripts
          mountPath: /scripts
        resources:
          {}
      volumes:
      - name: runner-secrets
        emptyDir:
          medium: "Memory"
      - name: etc-gitlab-runner
        emptyDir:
          medium: "Memory"
      - name: gitlab-runner-crt
        secret:
          secretName: gitlab.k8s.abc.crt
      - name: init-runner-secrets
        projected:
          sources:
            - secret:
                name: "mygitlab-minio-secret"
            - secret:
                name: "mygitlab-gitlab-runner-secret"
                items:
                  - key: runner-registration-token
                    path: runner-registration-token
                  - key: runner-token
                    path: runner-token
      - name: scripts
        configMap:
          name: mygitlab-gitlab-runner
```

再次执行，就解决 gitlab runner 证书授权的问题了。

通过 `doc/installation/secrets.md` 中的 `Registry authentication certificates` 可以知道 gitlab registry 使用的是自签名证书，这个有影响吗？

gitlab 和 registry 之间的交流是在 ingress 后面进行的，所以使用自签名证书能满足大部分的场景。如果流量通过一个网络，那你需要生成公共校验证书。

生成一个证书密钥对

```console
mkdir -p certs
openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -x509 -keyout certs/registry-example-com.key -out certs/registry-example-com.crt
```

生成一个 secret 包含这些证书，我们会新建一个名为 `<name>-registry-secret` 的secret，包含 `registry-auth.key` 和 `registry-auth.crt`，将 `<name>` 替换为对应的 release 名。

```console
kubectl create secret generic <name>-registry-secret --from-file=registry-auth.key=certs/registry-example-com.key --from-file=registry-auth.crt=certs/registry-example-com.crt
```

这个 secret 可以通过设置 `global.registry.certificate.secret` 引用。

还有一个 `wildcard-tls-ca`，见 `doc/installation/tls.md`

```console
The `shared-secrets` chart will then produce a CA certificate and wildcard certificate for use by all externally
accessible services. The secrets containing these will be `RELEASE-wildcard-tls` and `RELEASE-wildcard-tls-ca`.
The `RELEASE-wildcard-tls-ca` contains the public CA certificate that can be distributed to users and systems that
will access the deployed GitLab instance.
```

`shared-secrets` 子 chart 生成的，它会生成一个 ca 证书 和 通配符证书 供外部的服务使用。如果这个 CA 使用 自己的 ca 会不会好一点。

查看 `shared-secrets` chart 的说明 `doc/charts/shared-secrets/index.md`，`shared-secrets` 子 chart 负责提供安装过程中的各种 secret ，除了其他手动指定的。包括

- 初始 root 密码

- 所有公共服务的自签名 tls 证书，包括 gitlab, minio 和 registry

- registry 认证证书

- minio, registry, gitlab shell 和 gitaly 的 secret

- ssh host keys

其中 gitlab, minio 和 registry公共服务的自签名证书应该在上文生成 gitlab.yaml 时指定了，由 cert-manager 生成。registry 认证证书可以通过 `global.registry.certificate.secret` 指定。

它提供的参数里，没有指定 ca 的参数，如果可以指定 ca 的话，就统一使用我们自己的 ca 签发证书，不用自签名了，就可以彻底避免 证书认证的问题。

在 doc/charts/shared-secret 和 charts/shared-secret 查看 `shared-secrets` chart 的信息，没有找到关于指定 ca 的参数

观察下 gitlab.yaml , 其中通过名为 shared-secrets 的 Job 可以了解下`shared-secrets` chart的作用，它使用 `cfssl-self-sign:1.2` 镜像中的 cfssl 工具，利用 环境变量里的信息生成自签名证书，然后把生成的证书放在 emptyDir 中，然后 kubectl 镜像将 emptyDir 中的证书生成 secret 。 可知，使用 `shared-secrets` chart 无法自定义我们自己的 CA，如果想使用 自己的 CA，应该是只能不安装这个 chart ，然后手动生成各种 secret ，手动生成 secret 参考 `doc/installation/secret.md`

`shared-secrets` chart 在 Job 中生成了 通用证书 secret （mygitlab-wildcard-tls）和 ca 证书 secret （mygitlab-wildcard-tls-ca），其中，mygitlab-wildcard-tls 可以在 gitlab.yaml 中看到它的使用，它对应的名字为 custom-ca-certificates， 都是挂载到了 alpine-certificates 容器中，这个是一个初始化的容器，没找到这个镜像的 dockerfile，不知道做了什么。

上面是手动修改的 gitlan runner 和 postgresql 的命名空间，能否生成 yaml 文件的时候指定呢？
查看 postgresql 里的文件和它的 chart ，没有发现指定命名空间的地方。

查看 doc/charts/gitlab/gitlab-runner/index.md ，其中 `gitlab-runner.runners.namespace` 可以指定 runner 的命名空间，经过测试，这个参数无效

使用 helm v3 安装 gitlab

```console
helm template . \
  --name-template ${release} \
  --set global.edition=ce \
  --set global.time_zone=Beijing \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=hekai@abc.com \
  --set global.email.reply_to=hekai@abc.com \
  --set global.email.subject_suffix=@abc.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.abc.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=hekai@abc.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

根据 doc/installation/storage.md 中描述，建议 storageclass 的 reclaimPolicy 使用 retain 模式

```console
The default storage class should:

- Use fast SSD storage when available
- Set `reclaimPolicy` to `Retain`

> Uninstalling GitLab without the `reclaimPolicy` set to `Retain` allows automated jobs to completely delete the volume, disk and data.
> Some platforms set the default `reclaimPolicy` to `Delete`. The `gitaly` persistent volume claims do not follow this rule because
> they belong to a [StatefulSet][].
```

由于 上文创建的 managed-nfs-storage 的默认 reclaimPolicy 使用的是 Delete 模式，并且已经有 一些 容器使用 这个 storageclass 了，所以可以新建一个 nfs storage class 来支持 retain 模式，并把该 storage class 置为默认。

```console
[root@k8s-m1 deploy]# cat class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
[root@k8s-m1 deploy]# cat class-retain.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage-retain
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" # 将该sc配置为默认的sc
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
# Supported policies: Delete, Retain
reclaimPolicy: Retain
[root@k8s-m1 deploy]# kubectl apply -f class.yaml -f class-retain.yaml
```

-----------------------------

1. 配置 coredns 使用 hosts 插件 挂载 宿主机的 /etc/hosts

   目前的问题是 coredns 使用 hosts插件，如果宿主机的 /etc/hosts 中内容变化，coredns里的解析不会重新加载

2. 配置 ingress-controller 暴漏 22 端口

3. 新建命名空间

```console
kubectl create ns gitlab
```

4. 使用 CA 证书生成 gitlab-runner 连接 gitlab server 认证用的 secret
kubectl create secret generic gitlab.k8s.abc.crt --from-file=gitlab.k8s.abc.crt=ca.crt -n gitlab

5. 生成 gitlab server 与 registry 交互使用的证书
生成私钥和证书签名请求
openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -keyout registry-k8s-abc.key -out registry-k8s-abc.csr
使用CA证书根据证书签名请求签发证书
openssl x509 -req -sha256 -days 365 -in registry-k8s-abc.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out registry-k8s-abc.crt

> 原文是自签名生成一个证书和私钥
> openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -x509 -keyout certs/registry-example-com.key -out certs/registry-example-com.crt

根据生成的证书和私钥建一个secret
kubectl create secret generic ${release}-registry-secret --from-file=registry-auth.key=registry-k8s-abc.key --from-file=registry-auth.crt=registry-k8s-abc.crt -n gitlab

然后生成yaml时可以通过 global.registry.certificate.secret 引用该 secret

1. 生成 imap和smtp 所需的密码

```console
kubectl create secret generic gitlab-incoming-imap-pwd-secret --from-file=./gitlab-incoming-imap-pwd-key -n gitlab

kubectl create secret generic gitlab-outgoing-smtp-pwd-secret --from-file=gitlab-outgoing-smtp-pwd-key=gitlab-incoming-imap-pwd-key -n gitlab
```

6. 生成gitlab.yaml

```console
helm template . \
  --name-template ${release} \
  --set global.edition=ce \
  --set global.time_zone=Beijing \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=276516848@qq.com \
  --set global.email.reply_to=276516848@qq.com \
  --set global.email.subject_suffix=@qq.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.qq.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=276516848@qq.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

7. 挂载 gitlab.k8s.abc.crt

8. 修改 gitlab-runner 和 postgresql 相关资源的命名空间


注释掉 configmap mygitlab-nginx-ingress-tcp，因为这个是配置ingress nginx的，我们自己的已经配置好了

注释掉 PersistentVolumeClaim ，将对应的 Deployment 改为 StatefulSet ，并添加 pvc

为所有容器挂载宿主机 /etc/localtime

然后执行 kubectl apply -f gitlab.yaml


升级 到 v2.5.1，同时将smtp修改为使用qq提供
export release=mygitlab
1. 修改 smtp 密码
  
```shell
[root@k8s-m1 gitlab-v2.5.1]# kubectl delete secret -n gitlab gitlab-outgoing-smtp-pwd-secret 
secret "gitlab-outgoing-smtp-pwd-secret" deleted
[root@k8s-m1 gitlab-v2.5.1]# kubectl delete secret -n gitlab gitlab-incoming-imap-pwd-secret 
secret "gitlab-incoming-imap-pwd-secret" deleted
[root@k8s-m1 gitlab-v2.5.1]# cat gitlab-outgoing-smtp-pwd-key
xxxxxxxxxxxx
[root@k8s-m1 gitlab-v2.5.1]# kubectl create secret generic gitlab-outgoing-smtp-pwd-secret --from-file=gitlab-outgoing-smtp-pwd-key=gitlab-outgoing-smtp-pwd-key -n gitlab
secret/gitlab-outgoing-smtp-pwd-secret created

```

[root@k8s-m1 gitlab-v2.5.1]# helm dependency update


生成gitlab.yaml
```console
helm template . \
  --name-template ${release} \
  --set global.edition=ce \
  --set global.time_zone=Aisa/Shanghai \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=276516848@qq.com \
  --set global.email.reply_to=276516848@qq.com \
  --set global.email.subject_suffix=@qq.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.qq.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=276516848@qq.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

7. 挂载 gitlab.k8s.abc.crt

8. 修改 gitlab-runner 和 postgresql 相关资源的命名空间

这个也可以通过 kubectl apply -n gitlab 来指定


注释掉 configmap mygitlab-nginx-ingress-tcp，因为这个是配置ingress nginx的，我们自己的已经配置好了

注释掉 PersistentVolumeClaim ，将对应的 Deployment 改为 StatefulSet ，并添加 pvc

为所有容器挂载宿主机 /etc/localtime

不能直接执行 kubectl apply -f 进行更新，会报错，因为我们已经使用了 Stateful 进行了持久化，所以，可以先删除原来的部署，再重新部署

kubectl delete -f ../gitlab-v2.4.6/gitlab.yaml
kubectl create -f gitlab.yaml


发现 postgresql 没有启动，查看日志

```shell
[root@k8s-m1 gitlab-v2.5.1]# kubectl get statefulset -n gitlab
NAME                  READY   AGE
mygitlab-gitaly       1/1     3m28s
mygitlab-minio        0/1     3m29s
mygitlab-postgresql   0/1     3m29s
mygitlab-redis        0/1     3m28s
[root@k8s-m1 gitlab-v2.5.1]# kubectl describe statefulset -n gitlab mygitlab-postgresql
... ...
Events:
  Type     Reason        Age                   From                    Message
  ----     ------        ----                  ----                    -------
  Warning  FailedCreate  62s (x16 over 3m46s)  statefulset-controller  create Pod mygitlab-postgresql-0 in StatefulSet mygitlab-postgresql failed error: Pod "mygitlab-postgresql-0" is invalid: spec.containers[0].volumeMounts[0].name: Not found: "data"
  ```

原因是 statefulset 中的 pvc 和 pod中挂载的名字不一致（原因是v2.4.6版本我改成统一格式的名字了）。

修改后执行安装

```shell
kubectl apply -f gitlab.yaml -n gitlab
```

20191204 重新安装 gitlab-v2.5.4，已删除 gitlab 命名空间

```shell
[root@k8s-m1 gitlab]# kubectl create ns gitlab
namespace/gitlab created
```

