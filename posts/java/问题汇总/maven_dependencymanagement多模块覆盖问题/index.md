# Maven_dependencyManagement多模块覆盖问题


maven多模块项目，如 项目A是根项目，项目B是项目A的子模块，项目C是项目B的子模块，简单示例如下

project A
- module B
  - module C

项目A中pom.xml配置了dependencyManagement，如下

```xml
    <dependencyManagement>
        <dependencies>
            <!-- SpringBoot的依赖配置-->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

项目B中pom.xml也配置了dependencyManagement，如下

```xml
    <dependencyManagement>
        <dependencies>
            <!-- Apache Dubbo  -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-dependencies-bom</artifactId>
                <version>${dubbo.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Dubbo Spring Boot Starter -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-spring-boot-starter</artifactId>
                <version>${dubbo.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring-context</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>

            <!-- Zookeeper dependencies -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-dependencies-zookeeper</artifactId>
                <version>${dubbo.version}</version>
                <type>pom</type>
                <exclusions>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>slf4j-log4j12</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

项目C的pom.xml没有配置dependencyManagement，直接配置了dependencies，如下

```xml
    <dependencies>
        <!-- Spring Boot dependencies-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

此时启动项目C，会失败，错误信息如下：

```log
Exception in thread "main" java.lang.NoClassDefFoundError: org/springframework/core/metrics/ApplicationStartup
	at org.springframework.boot.SpringApplication.<init>(SpringApplication.java:251)
	at org.springframework.boot.SpringApplication.<init>(SpringApplication.java:264)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1311)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1300)
	at com.example.simple.dubbo_demo_provider.ProviderApplication.main(ProviderApplication.java:10)
Caused by: java.lang.ClassNotFoundException: org.springframework.core.metrics.ApplicationStartup
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 5 more
```

由此现象推断：在pom.xml中，dependencyManagement是会覆盖的，假设有多层父子module，则某项目父级的dependencyManagement会覆盖祖父级的dependencyManagement，是全部覆盖，祖父pom中dependencyManagement定义的元素对其孙module是不可见的。

解决项目C启动失败问题：

方案1：将项目B的denpendencyManagement内容移动到项目A中

方案2：在项目B的dependencyManagement中添加`spring-boot-dependencies`

所以，创建多层级module项目时，需要注意dependencyManagement的覆盖问题。查看网上的项目，多层级module项目，一般都是在最高级声明dependencyManagement，子项目里不写dependencyManagement的。

如果在祖父项目的子项目里写上了空的dependencyManagement，和不写的效果时一样的，不会导致覆盖祖父项目的dependencyManagement。

```xml
    <dependencyManagement>
        <dependencies>
        </dependencies>
    </dependencyManagement>
```

当然，最好还是不要在祖父项目的子项目写dependencyManagement

如果想在同个项目里使用多个dubbo版本，可以在祖父项目的子项目里定义dependencyManagement，不同的子项目使用不同的dubbo版本，注意，由于子项目的dependencyManagement覆盖了祖父项目的dependencyMangement，需要把祖父项目里dependencyMangement中的关键依赖同时添加到子项目的dependencyManagement中。

如果是project-dependencies-dependency元素呢？

祖父项目A添加依赖

```xml
    <dependencies>
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>4.5.7</version>
        </dependency>
    </dependencies>
```

祖父项目的子项目B，添加依赖如下

```xml
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
        </dependency>
    </dependencies>
```

查看祖父项目的孙项目C，继承了祖父项目的依赖和父项目的依赖，孙项目里同时依赖了hutool和fastjson
