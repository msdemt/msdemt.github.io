# Vscode配置kubernetes



**下载kubectl 和 helm 二进制文件**

kubectl 二进制文件下载地址：

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#client-binaries

得到适用于win10的kubectl下载链接：
https://dl.k8s.io/v1.18.12/kubernetes-client-windows-amd64.tar.gz

helm 二进制文件下载地址
https://github.com/helm/helm/tags

得到适用于win10的helm下载链接：
https://get.helm.sh/helm-v3.4.1-windows-amd64.zip

将下载后的 kubectl 和 helm 二进制文件解压到某个目录，如D:Develop/k8s/bin

将k8s集群中的/root/.kube/config文件拷贝到D:Develop/k8s/conf

**vscode配置kubernetes**

vscode安装kubernetes插件

在`首选项 - 设置`中，选择`vs-kubernetes`，点击`在settings.json中编辑`

编辑后的内容如下

```json
 "vs-kubernetes": {
    "vs-kubernetes.namespace": "",
    "vs-kubernetes.kubectl-path": "d:\\Develop\\k8s\\bin\\kubectl.exe",
    "vs-kubernetes.helm-path": "d:\\Develop\\k8s\\bin\\helm.exe",
    "vs-kubernetes.draft-path": "",
    "vs-kubernetes.minikube-path": "",
    "vs-kubernetes.kubectlVersioning": "user-provided",
    "vs-kubernetes.outputFormat": "yaml",
    "vs-kubernetes.kubeconfig": "d:\\Develop\\k8s\\conf\\config",
    "vs-kubernetes.knownKubeconfigs": [
        "d:\\Develop\\k8s\\conf\\config"
    ],
    "vs-kubernetes.autoCleanupOnDebugTerminate": false,
    "vs-kubernetes.nodejs-autodetect-remote-root": true,
    "vs-kubernetes.nodejs-remote-root": "",
    "vs-kubernetes.nodejs-debug-port": 9229,
    "checkForMinikubeUpgrade": true,
    "logsDisplay": "webview",
    "imageBuildTool": "Docker",
},
```
