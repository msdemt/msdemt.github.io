# Java_子module依赖pom类型的dependency


java 多module项目，父module的pom.xml摘要如下

```xml
    <dependencyManagement>
        <dependencies>
           <!-- Zookeeper dependencies -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-dependencies-zookeeper</artifactId>
                <version>${dubbo.version}</version>
                <type>pom</type>
                <!--<scope>import</scope>-->
                <exclusions>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>slf4j-api</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>slf4j-log4j12</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>log4j</groupId>
                        <artifactId>log4j</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

子模块想要依赖 dubbo-dependencies-zookeeper ，若子模块的pom.xml配置如下

```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
        </dependency>
    </dependencies>
```

idea maven一栏点击 reload project，发现依赖没有引入，而且因为这个依赖没引入，导致其他修改的依赖不能更新（比如另一个依赖修改了名字，这个名字无法更新）

正确的引入方式为

```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <type>pom</type>
        </dependency>
    </dependencies>
```

注意，需要将父模块里的`<scope>import</scope>`注释掉，不然子模块加了`<type>pom</type>`也无法引入


问题：加与不加 `<scope>import</scope>` 有什么区别呢？

由 [maven_scope属性](./maven_scope属性.md) 文章可知，scope属性为import，表示将该dependency替换为其内部定义的dependencyManagement，所以加上scope import，对子模块来说就无法直接引用该artifactId了。

子模块里想引入的是dubbo-dependencies-zookeeper本身，而不是 dubbo-dependencies-zookeeper里包含的依赖，所以注释掉scope import，注释掉scope import后，子模块既可以引用dubbo-dependencies-zookeeper本身，也可以引用dubbo-dependencies-zookeeper的子依赖，比如zookeeper

dubbo-dependencies-zookeeper包的pom.xml内容摘要如下

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies-bom</artifactId>
        <version>${project.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-recipes</artifactId>
    </dependency>
    <dependency>
      <groupId>org.apache.zookeeper</groupId>
      <artifactId>zookeeper</artifactId>
    </dependency>
  </dependencies>
```

可以看到，里面既有dependencyManagement，也有不包含在dependencyManagement里的dependencies，若项目A引入该依赖时使用scope import，则实际上会将该pom.xml中的dependencyManagement替换到项目A的dependencyManagement中。



