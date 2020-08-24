# K8s证书过期问题


## 20190620

问题：

下午出现了线上环境k8s上的docker容器服务无法接收请求的问题。具体表现如下

```shell
[root@k8s-m1 ~]# kubectl get nodes
NAME     STATUS      ROLES           AGE    VERSION
k8s-m1   NotReady    master,worker   256d   v1.6.2
k8s-m2   Ready       master,worker   256d   v1.6.2
k8s-m3   Ready       master,worker   256d   v1.6.2
[root@k8s-m1 ~]# 
```

node的状态为NotReady，但是可以执行k8s的命令。
- 若执行命令kubectl delete -f 8081.yaml 删除Deployment的yaml文件，则该部署的pod一直处于Terminating状态。
- 若执行命令kubectl create -f 8081.yaml新增加部署，则该pod一直处于pending状态，

查看该部署的状态 kubectl describe pods ，日志显示无可用节点。

查看系统日志
```shell
tail -1000f /var/log/message
```
日志显示：
```
Jun 19 16:18:06 clfw1 kube-apiserver: E0619 16:18:06.190083   10596 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
```

kube-apiserver组件提示无法通过证书认证请求，因为证书已过期或目前不可用。

分析：证书过期，导致服务不可用。

模拟证书过期的情况，将测试环境的k8s集群的时间调整到证书到期的时间之后。

1. 查看测试环境证书的到期时间，使用如下命令查看admin证书的到期时间，pem证书同样可以使用此命令查看。
    ```shell
    [root@c48-1 pki]# openssl x509 -noout -in admin.crt -enddate
    notAfter=Sep 30 02:43:06 2046 GMT
    ```
    该证书到期时间是2046年9月30日。
2. 将系统时间调整到证书到期时间之后，使用如下命令调整系统时间
    ```shell
    [root@c48-1 pki]# date -s '2047-09-09 12:00'
    ```
    将系统时间调整到2047年，观察证书到期后k8s的具体情况，
    ```shell
    [root@c48-1 pki]# date -s '2047-09-09 12:00'
    2047年 09月 09日 星期一 12:00:00 CST
    [root@c48-1 pki]# kubectl get nodes
    Unable to connect to the server: x509: certificate has expired or is not yet valid
    [root@c48-1 pki]# kubectl get pods
    Unable to connect to the server: x509: certificate has expired or is not yet valid
    [root@c48-1 pki]# kubectl get cs
    Unable to connect to the server: x509: certificate has expired or is not yet valid
    [root@c48-1 pki]# 
    ```
    发现，若证书到期，k8s的命令无法执行，且k8s上的容器服务无法正常提供服务。
 
 
 
线上问题解决办法：

重新制作k8s各个组件证书

制作方法请参考https://github.com/opsnull/follow-me-install-kubernetes-cluster分支1.6.2

目前线上环境证书配置为10年。

证书截至日期2029年6月16日
```shell
[root@slfw2 ssl]# openssl x509 -noout -in admin.pem -enddate
notAfter=Jun 16 14:13:00 2029 GMT
```

线上问题原因分析：

k8s集群内的etcd组件证书过期，其他组件证书有效，导致了kube-apiserver访问etcd时提示证书失效。
上次更新k8s证书有效期为100年时，遗漏etcd证书导致的此次问题。

## 20200622

k8s线上环境出现问题，服务无法部署成功，且原有的pod无法删除（通过 kubectl delete -f abc.yaml 提示删除成功，但是实际docker容器还是存在）

查看系统日志，发现是证书问题

```shell
kube-apiserver: E0622 19:41:05.146795    1021 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
kube-apiserver: I0622 19:41:05.146860    1021 wrap.go:75] GET /api/v1/nodes/172.17.60.112?resourceVersion=0: (81.124µs) 401 [[kubelet/v1.6.2 (linux/amd64) kubernetes/477efc3] 172.17.60.112:32843]
kubelet: E0622 19:41:05.147265    2153 kubelet_node_status.go:326] Error updating node status, will retry: error getting node "172.17.60.112": the server has asked for the client to provide credentials (get nodes 172.17.60.112)
kube-apiserver: E0622 19:41:05.147554    1021 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
kube-apiserver: I0622 19:41:05.147586    1021 wrap.go:75] GET /api/v1/nodes/172.17.60.112: (44.506µs) 401 [[kubelet/v1.6.2 (linux/amd64) kubernetes/477efc3] 172.17.60.112:32843]
kube-apiserver: E0622 19:41:05.148081    1021 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
kubelet: E0622 19:41:05.147843    2153 kubelet_node_status.go:326] Error updating node status, will retry: error getting node "172.17.60.112": the server has asked for the client to provide credentials (get nodes 172.17.60.112)
kubelet: E0622 19:41:05.148392    2153 kubelet_node_status.go:326] Error updating node status, will retry: error getting node "172.17.60.112": the server has asked for the client to provide credentials (get nodes 172.17.60.112)
kubelet: E0622 19:41:05.148906    2153 kubelet_node_status.go:326] Error updating node status, will retry: error getting node "172.17.60.112": the server has asked for the client to provide credentials (get nodes 172.17.60.112)
kube-apiserver: I0622 19:41:05.148110    1021 wrap.go:75] GET /api/v1/nodes/172.17.60.112: (52.508µs) 401 [[kubelet/v1.6.2 (linux/amd64) kubernetes/477efc3] 172.17.60.112:32843]
kube-apiserver: E0622 19:41:05.148627    1021 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
kube-apiserver: I0622 19:41:05.148655    1021 wrap.go:75] GET /api/v1/nodes/172.17.60.112: (35.999µs) 401 [[kubelet/v1.6.2 (linux/amd64) kubernetes/477efc3] 172.17.60.112:32843]
kube-apiserver: E0622 19:41:05.149120    1021 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
kube-apiserver: I0622 19:41:05.149149    1021 wrap.go:75] GET /api/v1/nodes/172.17.60.112: (36.26µs) 401 [[kubelet/v1.6.2 (linux/amd64) kubernetes/477efc3] 172.17.60.112:32843]
kubelet: E0622 19:41:05.149475    2153 kubelet_node_status.go:326] Error updating node status, will retry: error getting node "172.17.60.112": the server has asked for the client to provide credentials (get nodes 172.17.60.112)
kubelet: E0622 19:41:05.149487    2153 kubelet_node_status.go:318] Unable to update node status: update node status exceeds retry count
```

kubelet证书过期，查看 k8s 的证书有效期

```shell
# openssl x509 -noout -in ca.pem -enddate
notAfter=Jun 17 15:11:00 2024 GMT
# openssl x509 -noout -in /etc/etcd/ssl/etcd.pem -enddate
notAfter=Jun 16 15:12:00 2029 GMT 
# openssl x509 -noout -in admin.pem -enddate
notAfter=Jun 16 15:14:00 2029 GMT
# openssl x509 -noout -in /etc/flanneld/ssl/flanneld.pem -enddate
notAfter=Jun 16 15:15:00 2029 GMT
# openssl x509 -noout -in kubernetes.pem -enddate
notAfter=Jun 16 15:16:00 2029 GMT
# openssl x509 -noout -in kube-proxy.pem -enddate
notAfter=Jun 16 15:19:00 2029 GMT
# openssl x509 -noout -in kubelet.crt -enddate
notAfter=Jun 19 09:22:59 2019 GMT
# openssl x509 -noout -in kubelet-client.crt -enddate
notAfter=Jun 18 14:05:00 2020 GMT
```

kubelet.crt 证书是创建k8s集群时，kubelet首次启动时生成的证书，kubelet-client.crt是kubelet与kube-apiserver交互使用的证书，kubelet-client.

kubelet证书续期:
1. 删除 kubelet.kubeconfig 文件（文件位置：/etc/kubernetes ，此处安全起见，使用备份的方式）
    ```shell
    mv kubelet.kubeconfig kubelet.kubeconfig.bak
    ```

2. 重启 kubelet 服务

    ```shell
    systemctl restart kubelet
    ```
3. 通过 csr 请求

    ```shell
    $ kubectl get csr
    NAME        AGE       REQUESTOR           CONDITION
    csr-2b308   4m        kubelet-bootstrap   Pending
    ```

    ```shell
    $ kubectl certificate approve csr-2b308
    certificatesigningrequest "csr-2b308" approved
    $ kubectl get nodes
    NAME        STATUS    AGE       VERSION
    10.64.3.7   Ready     49m       v1.6.2
    ```

    之后会重新生成新的 kubelet-client.crt 证书文件，续期一年。

```shell
[root@k8s-1 ~]# cd /etc/kubernetes/ssl/
[root@k8s-1 ssl]# ls
admin.csr       admin.pem       ca-csr.json  etcd.csr       flanneld-csr.json   kubelet.crt     kube-proxy-csr.json  kubernetes.csr       kubernetes.pem
admin-csr.json  ca-config.json  ca-key.pem   etcd-csr.json  kubelet-client.crt  kubelet.key     kube-proxy-key.pem   kubernetes-csr.json
admin-key.pem   ca.csr          ca.pem       flanneld.csr   kubelet-client.key  kube-proxy.csr  kube-proxy.pem       kubernetes-key.pem
[root@k8s-1 ssl]# openssl x509 -noout -in kubelet.crt -enddate
notAfter=Jun 19 09:22:59 2019 GMT
[root@k8s-1 ssl]# openssl x509 -noout -in kubelet-client.crt -enddate
notAfter=Jun 18 14:05:00 2020 GMT
[root@k8s-1 ssl]# cd ..
[root@k8s-1 kubernetes]# ls
bootstrap.kubeconfig  kubelet.kubeconfig  kube-proxy.kubeconfig  ssl  token.csv
[root@k8s-1 kubernetes]# mv kubelet.kubeconfig /home
[root@k8s-1 kubernetes]# systemctl restart kubelet
[root@k8s-1 kubernetes]# ls
bootstrap.kubeconfig  kube-proxy.kubeconfig  ssl  token.csv
[root@k8s-1 kubernetes]# kubectl get csr
NAME        AGE       REQUESTOR           CONDITION
csr-86w67   1y        kubelet-bootstrap   Approved,Issued
csr-lnfrv   2y        kubelet-bootstrap   Approved,Issued
csr-zq6cl   5s        kubelet-bootstrap   Pending
[root@k8s-1 kubernetes]# kubectl certificate approve csr-zq6cl
certificatesigningrequest "csr-zq6cl" approved
[root@k8s-1 kubernetes]# kubectl get csr
NAME        AGE       REQUESTOR           CONDITION
csr-86w67   1y        kubelet-bootstrap   Approved,Issued
csr-lnfrv   2y        kubelet-bootstrap   Approved,Issued
csr-zq6cl   27s       kubelet-bootstrap   Approved,Issued
[root@k8s-1 kubernetes]# ls
bootstrap.kubeconfig  kubelet.kubeconfig  kube-proxy.kubeconfig  ssl  token.csv
[root@k8s-1 kubernetes]# cd ssl/
[root@k8s-1 ssl]# ls
admin.csr       admin.pem       ca-csr.json  etcd.csr       flanneld-csr.json   kubelet.crt     kube-proxy-csr.json  kubernetes.csr       kubernetes.pem
admin-csr.json  ca-config.json  ca-key.pem   etcd-csr.json  kubelet-client.crt  kubelet.key     kube-proxy-key.pem   kubernetes-csr.json
admin-key.pem   ca.csr          ca.pem       flanneld.csr   kubelet-client.key  kube-proxy.csr  kube-proxy.pem       kubernetes-key.pem
[root@k8s-1 ssl]# openssl x509 -noout -in kubelet-client.crt -enddate
notAfter=Jun 23 00:39:00 2021 GMT
[root@k8s-1 ssl]# vi /usr/lib/systemd/system/kubelet^C
[root@k8s-1 ssl]# openssl x509 -noout -in kubelet.crt -enddate
notAfter=Jun 19 09:22:59 2019 GMT
[root@k8s-1 ssl]#
```


![图 1](../../../images/70a4f79fa4aa527cf9e7274a792414f0422cc7f97c4fd471d18bbf2f64a4bc42.png)  

https://www.cnblogs.com/centos-python/articles/13168559.html
