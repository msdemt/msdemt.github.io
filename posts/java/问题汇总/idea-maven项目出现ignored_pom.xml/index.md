# IDEA Maven项目出现ignored pom


> 参考：  
> https://blog.csdn.net/CF779/article/details/112269347

在 idea java maven项目中新建名为 demo 的module，将该 module删除后，再次新建名为 demo 的 module，该 module的 pom.xml 会显示为 ingored pom.xml

解决：进入 File-->Settings-->Build,Execution,Deployment-->Build Tools-->Maven-->Ignored Files，将demo module前的勾去掉即可。

