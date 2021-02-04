# Springboot Pom文件中的resources标签


spring-boot-starter-parent 的 pom.xml 文件中有如下内容

```xml
    <properties>
        <java.version>1.8</java.version>
        <spring-boot.version>2.4.2</spring-boot.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
        <project.encoding>UTF-8</project.encoding>
        <resource.delimiter>@</resource.delimiter>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>${project.encoding}</project.build.sourceEncoding>
        <project.reporting.outputEncoding>${project.encoding}</project.reporting.outputEncoding>

        <revision>1.0.0</revision>
    </properties>

    <build>
        <resources>
            <resource>
                <directory>${basedir}/src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>**/application*.yml</include>
                    <include>**/application*.yaml</include>
                    <include>**/application*.properties</include>
                </includes>
            </resource>
            <resource>
                <directory>${basedir}/src/main/resources</directory>
                <excludes>
                    <exclude>**/application*.yml</exclude>
                    <exclude>**/application*.yaml</exclude>
                    <exclude>**/application*.properties</exclude>
                </excludes>
            </resource>
        </resources>
    </build>
```


1. 为什么把`${basedir}/src/main/resources`目录下的`**/application*.yml`文件include到classpath后又exclude了呢？

    `resource` 标签是按照顺序进行加载的

    第一个`resource`标签，表示加载`项目路径/src/main/resources`下的application*.yml文件

    然后再加载第二个 `resource` 标签，表示加载除了第一个`resource`标签加载内容（`项目路径/src/main/resources`下的application*.yml文件）之外的其他文件

2. `<filtering>true</filtering>`的作用是什么呢？

    参考：https://www.cnblogs.com/wangxuejian/p/13551292.html

    SpringEL表达式的取值一般使用 `${var}` 方式，如 `application.properties` 和 `@Value("${var}")` 中的取值

    maven的pom.xml文件也有类似的取值方式，也是使用 `${var}` 方式取值。

    **然而，它们并不是一个东西**

    SpringEL表达式取值适用于配置文件和代码中的注解

    maven的占位符取值表达式默认仅仅适用于pom.xml文件中

    所以，默认情况下是不能在`application.properties`里获取pom.xml中定义的值的。

    如果我们想打通二者的交流，该怎么实现呢？这时filtering就派上用场了

    maven的占位符解析表达式默认只在pom.xml文件内使用，如果想扩大它的活动范围，就必须指定需要扩大到哪些文件，然后指定 `filtering = true`，这样，maven的占位符解析表达式就可以用于那些文件里进行表达式解析了。

    如上，springboot parent中的pom.xml，配置 `<resource.delimiter>@</resource.delimiter>` 将maven的占位符置为`@`，同时，在第一个resource标签中指定maven占位符的使用范围扩大到`项目路径/src/main/resources`下的application*.yml文件，这样，application.yml文件就能使用`@var@`的形式获取maven的参数了。
