# Java项目配置阿里云仓库


> 参考：  
> https://maven.aliyun.com/mvn/guide  
> https://www.cnblogs.com/default/p/11856188.html


在java maven项目中，配置阿里云依赖jar包仓库和maven插件仓库，在 pom.xml中添加如下配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <repositories>
        <!--maven仓库-->
        <repository>
            <id>aliyun</id>
            <name>aliyun repo</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
    <pluginRepositories>
        <!--maven插件仓库-->
        <pluginRepository>
            <id>aliyun-plugin</id>
            <name>aliyun plugin</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>
</project>
```




