# 配置docker远程访问


> 参考：  
> https://www.cnblogs.com/niceyoo/p/13270224.html  
> https://docs.docker.com/engine/reference/commandline/dockerd/  
> https://docs.docker.com/engine/security/https/

**docker 开启2375端口远程访问**

编辑 `/usr/lib/systemd/system/docker.service`

找到 `[Service]` 节点，修改 `ExecStart` 属性，增加 `-H tcp://0.0.0.0:2375`

修改后 Service 节点如下

```shell
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -H tcp://0.0.0.0:2375
```

重新加载docker配置

```shell
systemctl daemon-reload
systemctl restart docker
```

通过浏览器访问 2375 测试，格式为：http://ip:2375/version

如果无法访问，检查是否防火墙2375端口被禁止，如果是，使用如下命令开启防火墙2375端口

```shell
firewall-cmd --zone=public --add-port=2375/tcp --permanent
firewall-cmd --reload
```

如果还是不能访问，如果使用的是云主机，需要到服务器安全组规则查看是否开放了2375端口，如未配置，增加端口配置即可

**IDEA连接docker**

idea 安装 docker 插件

配置docker tcp socket 为 tcp://ip:2375

**vscode配置docker**

vscode 安装 docker 插件

在 settings.json 中添加 `"docker.host": "http://192.168.15.7:2375",`


**docker 远程端口2375和2376端口的区别：**
```text
It is conventional to use port 2375 for un-encrypted, and port 2376 for encrypted communication with the daemon.
```
参考：https://docs.docker.com/engine/reference/commandline/dockerd/
