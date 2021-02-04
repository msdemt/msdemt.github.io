# K8s删除处于terminating状态的namespace


> 参考：
> <https://blog.csdn.net/fengwuxichen/article/details/87877214>

k8s集群删除安装的一些插件后，发现，如果删除 namespace ，有些 namespace 会一直处于 terminating 状态，如下所示：

  ```shell
  [root@k8s-m1 ~]# kubectl get ns
  NAME                 STATUS        AGE
  c-gbkmh              Terminating   84d
  cattle-system        Terminating   84d
  default              Active        84d
  knative-monitoring   Terminating   10d
  knative-serving      Terminating   10d
  kube-node-lease      Active        84d
  kube-public          Active        84d
  kube-system          Active        84d
  local                Active        84d
  logging              Active        13d
  monitoring           Terminating   14d
  nfs-storageclass     Terminating   84d
  p-9628v              Active        84d
  p-bpmct              Active        84d
  p-hrt5k              Active        84d
  p-jb9sx              Active        84d
  rook-ceph            Terminating   14d
  tidb-admin           Terminating   12d
  tidb-cluster         Terminating   12d
  user-b7qf5           Active        84d
  wayne                Active        84d
  ```

使用`kubectl delete ns 命名空间名 --force` 也无法删除。

最终是使用两个方法删除了处于terminating状态的namespace。

1. 使用 `kubectl edit ns 命名空间名称`，将finalizers对应的内容删除，然后保存退出（:wq）。

    具体步骤如下：

    ```shell
    [root@k8s-m1 ~]# kubectl edit ns harbor

    # Please edit the object below. Lines beginning with a '#' will be ignored,
    # and an empty file will abort the edit. If an error occurs while saving this file will be
    # reopened with the relevant failures.
    #
    apiVersion: v1
    kind: Namespace
    metadata:
      annotations:
        cattle.io/status: '{"Conditions":[{"Type":"ResourceQuotaInit","Status":"True","Message":"","LastUpdateTime":"2019-07-17T15:50:49Z"},{"Type":"InitialRolesPopulated","Status":"True","Message":"","LastUpdateTime":"2019-07-17T15:50:51Z"}]}'
        lifecycle.cattle.io/create.namespace-auth: "true"
      creationTimestamp: "2019-07-17T15:23:03Z"
      deletionGracePeriodSeconds: 0
      deletionTimestamp: "2019-10-10T01:15:49Z"
      finalizers:
      - controller.cattle.io/namespace-auth
      name: harbor
      resourceVersion: "25421542"
      selfLink: /api/v1/namespaces/harbor
      uid: 93e447e2-1bac-4e1c-825c-8578af09fa0b
    spec: {}
    status:
      phase: Terminating
    ```

    将如下内容删掉

    ```shell
      finalizers:
      - controller.cattle.io/namespace-auth
    ```

    保存退出（:wq）

    使用 `kubectl get ns` 查看当前的命名空间，发现 harbor 命名空间已删除。

    使用该方法删除了绝大多数的命名空间，但是存在两个命名空间无法删除，分别为 cattle-system 和 c-gbkmh 。

2. 使用 kube-proxy 删除处于 terminating 状态的命名空间

    cattle-system命名空间的内容如下：

    ```shell
    [root@k8s-m1 ~]# kubectl edit ns cattle-system

    # Please edit the object below. Lines beginning with a '#' will be ignored,
    # and an empty file will abort the edit. If an error occurs while saving this file will be
    # reopened with the relevant failures.
    #
    apiVersion: v1
    kind: Namespace
    metadata:
      annotations:
        cattle.io/status: '{"Conditions":[{"Type":"ResourceQuotaInit","Status":"True","Message":"","LastUpdateTime":"2019-07-17T15:50:47Z"},{"Type":"InitialRolesPopulated","Status":"True","Message":"","LastUpdateTime":"2019-07-17T15:50:53Z"}]}'
        field.cattle.io/projectId: local:p-bpmct
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{},"name":"cattle-system"}}
        lifecycle.cattle.io/create.namespace-auth: "true"
      creationTimestamp: "2019-07-17T15:46:04Z"
      deletionTimestamp: "2019-10-10T01:15:18Z"
      name: cattle-system
      resourceVersion: "25439200"
      selfLink: /api/v1/namespaces/cattle-system
      uid: 840d8744-81f2-4b0f-be0e-5bfb3d91f471
    spec:
      finalizers:
      - kubernetes
    status:
      phase: Terminating
    ```

    它不像上面 harbor 命名空间，它的 finalizers 位于 spec ，且 finalizers 为 kubernetes 。

    尝试手动删除 finalizers 字段，保存退出后，namespace 没有被删除。

    在终端中依次输入以下语句

    ```shell
    NAMESPACE=cattle-system
    kubectl proxy &
    kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
    curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
    ```

    在一个 k8s 节点（k8s-m1）上返回错误（可能是 k8s-m1 节点的 kube-proxy 存在问题）

    ```shell
    [root@k8s-m1 ~]# NAMESPACE=cattle-system
    [root@k8s-m1 ~]# kubectl proxy &
    [1] 33592
    [root@k8s-m1 ~]# kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
    Starting to serve on 127.0.0.1:8001
    [root@k8s-m1 ~]# curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
    curl: (52) Empty reply from server
    ```

    在另一个节点（k8s-m2）上删除成功，成功如下所示：

    ```shell
    [root@k8s-m2 ~]# NAMESPACE=cattle-system
    [root@k8s-m2 ~]# kubectl proxy &
    [1] 25118
    [root@k8s-m2 ~]# kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
    Starting to serve on 127.0.0.1:8001

    [root@k8s-m2 ~]# curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "cattle-system",
        "selfLink": "/api/v1/namespaces/cattle-system/finalize",
        "uid": "840d8744-81f2-4b0f-be0e-5bfb3d91f471",
        "resourceVersion": "25440818",
        "creationTimestamp": "2019-07-17T15:46:04Z",
        "deletionTimestamp": "2019-10-10T01:15:18Z",
        "annotations": {
          "cattle.io/status": "{\"Conditions\":[{\"Type\":\"ResourceQuotaInit\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-17T15:50:47Z\"},{\"Type\":\"InitialRolesPopulated\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-17T15:50:53Z\"}]}",
          "field.cattle.io/projectId": "local:p-bpmct",
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{\"cattle.io/status\":\"{\\\"Conditions\\\":[{\\\"Type\\\":\\\"ResourceQuotaInit\\\",\\\"Status\\\":\\\"True\\\",\\\"Message\\\":\\\"\\\",\\\"LastUpdateTime\\\":\\\"2019-07-17T15:50:47Z\\\"},{\\\"Type\\\":\\\"InitialRolesPopulated\\\",\\\"Status\\\":\\\"True\\\",\\\"Message\\\":\\\"\\\",\\\"LastUpdateTime\\\":\\\"2019-07-17T15:50:53Z\\\"}]}\",\"field.cattle.io/projectId\":\"local:p-bpmct\",\"lifecycle.cattle.io/create.namespace-auth\":\"true\"},\"creationTimestamp\":\"2019-07-17T15:46:04Z\",\"deletionTimestamp\":\"2019-10-10T01:15:18Z\",\"name\":\"cattle-system\",\"resourceVersion\":\"25439200\",\"selfLink\":\"/api/v1/namespaces/cattle-system\",\"uid\":\"840d8744-81f2-4b0f-be0e-5bfb3d91f471\"},\"spec\":{\"finalizers\":[]},\"status\":{\"phase\":\"Terminating\"}}\n",
          "lifecycle.cattle.io/create.namespace-auth": "true"
        }
      },
      "spec": {

      },
      "status": {
        "phase": "Terminating"
      }
    }
    [root@k8s-m2 ~]# NAMESPACE=c-gbkmh
    [root@k8s-m2 ~]# kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
    [root@k8s-m2 ~]# curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "c-gbkmh",
        "selfLink": "/api/v1/namespaces/c-gbkmh/finalize",
        "uid": "d5b5121a-8927-41e4-9252-a60794ec1ba5",
        "resourceVersion": "25422683",
        "creationTimestamp": "2019-07-18T00:34:05Z",
        "deletionTimestamp": "2019-10-10T01:15:18Z",
        "annotations": {
          "cattle.io/status": "{\"Conditions\":[{\"Type\":\"ResourceQuotaInit\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-18T00:34:06Z\"},{\"Type\":\"InitialRolesPopulated\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-18T00:34:06Z\"}]}",
          "lifecycle.cattle.io/create.namespace-auth": "true",
          "management.cattle.io/system-namespace": "true"
        }
      },
      "spec": {

      },
      "status": {
        "phase": "Terminating"
      }
    }[root@k8s-m2 ~]# kubectl get ns
    NAME              STATUS   AGE
    default           Active   84d
    kube-node-lease   Active   84d
    kube-public       Active   84d
    kube-system       Active   84d
    local             Active   84d
    logging           Active   13d
    p-9628v           Active   84d
    p-bpmct           Active   84d
    p-hrt5k           Active   84d
    p-jb9sx           Active   84d
    user-b7qf5        Active   84d
    wayne             Active   84d
    ```

    应该是将 finalizers 置为空，然后发送到 kube-proxy（感觉和手动修改 svc 差不多，下次可以再次试下手动修改 svc 能否有效），temp.json内容如下：

    ```shell
    [root@k8s-m2 ~]# cat temp.json
    {
      "apiVersion": "v1",
      "kind": "Namespace",
      "metadata": {
        "annotations": {
          "cattle.io/status": "{\"Conditions\":[{\"Type\":\"ResourceQuotaInit\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-18T00:34:06Z\"},{\"Type\":\"InitialRolesPopulated\",\"Status\":\"True\",\"Message\":\"\",\"LastUpdateTime\":\"2019-07-18T00:34:06Z\"}]}",
          "lifecycle.cattle.io/create.namespace-auth": "true",
          "management.cattle.io/system-namespace": "true"
        },
        "creationTimestamp": "2019-07-18T00:34:05Z",
        "deletionTimestamp": "2019-10-10T01:15:18Z",
        "name": "c-gbkmh",
        "resourceVersion": "25422683",
        "selfLink": "/api/v1/namespaces/c-gbkmh",
        "uid": "d5b5121a-8927-41e4-9252-a60794ec1ba5"
      },
      "spec": {
        "finalizers": []
      },
      "status": {
        "phase": "Terminating"
      }
    }
    ```

