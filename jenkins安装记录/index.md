# Jenkins安装记录


运行 jenkins master 的清单文件，使用上面的镜像部署一个 jenkins 的服务。

```console
kubectl apply -f jenkins.yaml -f jenkins-ing.yml
```

查看 jenkins 是否运行成功

```console
$ kubectl get pods -n jenkins-ns
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-deploy-7dbcdd9848-rkksg   1/1     Running   0          16m
```

jenkins 首次登录需要输入自动生成的密码，使用如下命令获取

```console
kubectl logs -n jenkins-ns -f jenkins-deploy-7dbcdd9848-rkksg
```

密码如下

```console
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

5ad0222348dc4a24b36a9238b82ae725

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
```

安装推荐的插件，由于网络的原因，可能会下载失败。

跳过，

jenkins 默认通过 `http://updates.jenkins-ci.org` 域名下载插件，如 `http://updates.jenkins-ci.org/download/plugins/trilead-api/1.0.5/trilead-api.hpi`，某些情况下该域名无法访问，所以跳过推荐插件安装，待进入 jenkins 页面后配置插件源再安装需要的插件。

配置jenkins 插件源

进入jenkins 页面后，依次选择 Manage Jenkins - Manage Plugins - Advanced - Update Site 
将 `https://updates.jenkins.io/update-center.json` 修改为清华大学镜像源 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`，点击Submit。

安装推荐的插件

jenkins 默认推荐的插件如下图，可按需安装
![IMAGES](../../images/jenkins-plugins-20191025.png)


这么多推荐的插件，一个个手动搜索安装特别麻烦

或者 jenkins pod 启动成功后，修改挂载的文件中指定的插件源，
本文是使用 nfs 挂载的，找到 jenkins 挂载出的目录
/home/data/nfs/data/jenkins-ns-jenkins-home-claim-pvc-6db72e8b-85a9-448c-be0f-4499d19983cd

修改 hudson.model.UpdateCenter.xml

将 https://updates.jenkins.io/update-center.json 修改为

https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

修改后重启 jenkin pod，从而让 jenkins 重新加载插件源的地址

```
kubectl delete po -n jenkins-ns jenkins-deploy-7dbcdd9848-l7wq2
```

然后登录 jenkins 进行推荐插件的安装

发现，还是有下载失败的插件，查看 jenkins 的日志，发现插件还是从updates.jenkins-ci.org 下载的

因为https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json 中指定的插件的下载地址是updates.jenkins-ci.org。

解决：修改 jenkins目录/updates/default.json，将updates.jenkins-ci.org/download 修改为https://mirrors.tuna.tsinghua.edu.cn/jenkins，从而真正使用清华大学的镜像地址下载插件。

方式一：使用 vim 进行批量修改
vim default.json
输入 `:` 进入命令模式，然后输入如下内容进行替换，然后按Enter
```
1,$s/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g
```

替换连接测试url，再次输入 `:` 进入命令模式，然后输入如下命令后按Enter
```
1,$s/http:\/\/www.google.com/https:\/\/www.baidu.com/g
```

替换玩后输入 `:wq` 保存退出

方式二： 使用 sed 修改

```
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```


修改后重启 jenkin pod，从而让 jenkins 重新加载插件源的地址

```
kubectl delete po -n jenkins-ns jenkins-deploy-7dbcdd9848-l7wq2
```


登录 jenkins

通过上面查看到的初始密码，若忘记了，可以通过 jenkins 挂载目录下的 secret 中的 initialAdminPassword文件查看，如本文的为 
```console
[root@c48-2 secrets]# pwd
/home/data/nfs/data/jenkins-ns-jenkins-home-claim-pvc-1391b4f1-d614-4321-85f9-6279e0dca936/secrets
[root@c48-2 secrets]# cat initialAdminPassword 
73c7a3a8a9ed4972a7cdb06d5de72076
```
安装推荐的插件，速度很快就安装好了。


创建第一个管理员用户

实例配置，默认即可

登录后，页面显示空白！

搜索了下，说是空白的原因是因为 镜像的升级站点为 https 导致的，需要改为 http。

尝试了下，访问 `https://jenkins.k8s.abc/pluginManager/advanced` ，当前的升级站点为 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`，改为 `http://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`，再次访问主页就正常了。

所以 修改的时候，只修改 updates/default.json 就可以了，不用修改 hudson.model.UpdateCenter.xml

不行，这样首次登录还是页面空白

需要将 hudson.model.UpdateCenter.xml 中的地址改为 http，且修改update/default.json。

验证后，确实改为 http 后 首次登录正常了。为什么不该为清华镜像的http呢？？

所以，总结下，将`hudson.model.UpdateCenter.xml`中的地址改为 `http://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json` ，然后修改updates/default.json。

```
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' updates/default.json
```

再次安装，首次登录后又出现了空白的情况，重启下 pod 就可以了。

有强迫症的话，也可以把 updates/default.json 里清华大学镜像源的地址配置为 http 的。


如果 jenkins 有启动参数可以修改插件的地址就好了。


总结： jenkins 部署后，修改 jenkins 挂载出的目录的 hudson.model.UpdateCenter.xml ，将 url 的值修改为 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`，jenkins 第一次启动会使用默认的值 `https://updates.jenkins.io/update-center.json` 进行查询，从而生成updates目录及其中的文件。如果这个地址访问不了，则 jenkins 就启动不成功，pod 显示 0/1 状态

```
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-deploy-7dbcdd9848-zxhbv   0/1     Running   0          18m
```

查看 pod 的 日志，如果出现这种情况的话，将 hudson.model.UpdateCenter.xml 中的值改为清华大学镜像，重启 pod 就可以了。

jenkins 的 pod 启动成功后，会在挂载的目录里生成 updates 目录，修改其中的 default.json 

然后重启 pod 安装推荐的插件，使用的就是真正的清华镜像源安装插件。

如果首次登录出现空白页，那么在重启下 jenkins。
