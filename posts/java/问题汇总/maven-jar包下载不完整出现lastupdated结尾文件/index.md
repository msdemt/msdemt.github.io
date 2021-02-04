# Maven-jar包下载不完整，出现lastUpdated结尾文件


## lastUpdated文件

![maven-springbootparent出现警告](/images/20210127-maven-springbootparent出现警告.png)  

 
原因：依赖没有下载完整，查看 maven 本地仓库，发现依赖为：spring-boot-starter-parent-2.4.0.pom.lastUpdated

解决：删除该 lastUpdated 文件，然后重新下载依赖

![vscode-maven重新下载依赖](/images/20210127-vscode-maven重新下载依赖.png)  


## win10批量删除lastUpdated文件

在 CMD 窗口中打开 maven 本地仓库目录，如 D:\Develop\mvnResp

执行如下命令

```shell
D:\Develop\mvnRespo> for /r %i in (*.lastUpdated) do del %i
```



