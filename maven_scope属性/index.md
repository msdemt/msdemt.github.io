# Maven_scope属性


> 参考：  
> https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#dependency-scope  

Dependency scope is used to limit the transitivity of a dependency and to determine when a dependency is included in a classpath.  
dependency scope 参数用于限制依赖关系的传递性，并用于确定是否将一个依赖包含在classpath中。

There are 6 scopes:  
scope参数共有6个可选值：

- compile
    This is the default scope, used if none is specified. Compile dependencies are available in all classpaths of a project. Furthermore, those dependencies are propagated to dependent projects.  
    compile是scope属性的默认值。scope类型的dependency在项目的所有类路径中都可以使用。此外，compile类型的依赖是可以传递的（比如项目A有scope为compile类型的依赖fastjson，若项目B依赖项目A，那么项目B也依赖了fastjson，项目B的classpath也会有fastjson）  
    举例：  

    ```xml
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.70</version>
    </dependency>
    ```

- provided
    This is much like compile, but indicates you expect the JDK or a container to provide the dependency at runtime. For example, when building a web application for the Java Enterprise Edition, you would set the dependency on the Servlet API and related Java EE APIs to scope provided because the web container provides those classes. A dependency with this scope is added to the classpath used for compilation and test, but not the runtime classpath. It is not transitive.  
    scope属性值为provided时很像compile，但是，若依赖scope 为 provided，说明你期望该依赖在项目可以在项目运行时由JDK或者容器运行时（tomcat等）提供。简言之，scope属性值为provided的依赖会用于编译和测试，不会包含在运行时的类路径中，即打包时不会包含该依赖，该依赖有servlet容器或jdk提供。并且，该依赖是不可传递的。  
    举例：
    ```xml
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
        <scope>provided</scope>
    </dependency>
    ```

- runtime
    This scope indicates that the dependency is not required for compilation, but is for execution. Maven includes a dependency with this scope in the runtime and test classpaths, but not the compile classpath.  
    scope属性值为runtime表示该依赖不是编译必须的，但是运行时必须的。maven在运行时和测试时的类路径中包含该依赖，在编译类路径中不包含该依赖。  
    举例：
    ```xml
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.23</version>
        <scope>runtime</scope>
    </dependency>
    ```

- test
    This scope indicates that the dependency is not required for normal use of the application, and is only available for the test compilation and execution phases. This scope is not transitive. Typically this scope is used for test libraries such as JUnit and Mockito. It is also used for non-test libraries such as Apache Commons IO if those libraries are used in unit tests (src/test/java) but not in the model code (src/main/java).  
    scope属性值为test表示该依赖对于应用的正常运行不是必须的，并且仅在测试的编译和执行阶段使用。该属性的依赖是不可传递的。scope值为test的依赖仅可用于`src/test/java`包中。  
    举例：
    ```xml
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>
    ```


- system
    This scope is similar to provided except that you have to provide the JAR which contains it explicitly. The artifact is always available and is not looked up in a repository.  
    scope属性值为system的依赖与值为provided的规则相似，除了必须显示提供对应的jar包。该依赖是引用自本地的依赖jar包，并且不会去依赖仓库中查找。  
    举例：  
    ```xml
    <dependency>
        <groupId>com.jacob</groupId>
        <artifactId>jacob</artifactId>
        <version>1.0</version>
        <scope>system</scope>
        <systemPath>${project.basedir}/src/main/webapp/WEB-INF/lib/jacob.jar</systemPath>
    </dependency>
    ```

- import
    This scope is only supported on a dependency of type pom in the <dependencyManagement> section. It indicates the dependency is to be replaced with the effective list of dependencies in the specified POM's <dependencyManagement> section. Since they are replaced, dependencies with a scope of import do not actually participate in limiting the transitivity of a dependency.  
    scope属性值为import仅支持在`<dependencyManagement>`属性中定义的type为 pom 的 dependency，它表示该dependency会被该dependency内部pom.xml中的`<dependencyManagement>`替代。由于被替代了，dependency为import的依赖不会参与限制依赖的可传递性。简言之，若denpendency的scope配置为scope，那么该dependency会被其内部pom.xml中声明的`<dependencyManagement>`替代。  
    举例：  
    父项目pom.xml中配置摘要如下
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
    实际上，该依赖，会被spring-boot-dependencies项目的pom.xml（spring-boot-dependencies-2.4.2.pom）内部的dependencyManagement替代，所以，实际上声明的dependencyManagement如下（内容太多了，所以省略了）
    ```xml
    <dependencyManagement>
      <dependencies>
        <dependency>
          <groupId>org.apache.activemq</groupId>
          <artifactId>activemq-amqp</artifactId>
          <version>${activemq.version}</version>
        </dependency>
        ... ...
      <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-bom</artifactId>
        <version>${spring-security.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-bom</artifactId>
        <version>${spring-session-bom.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      </dependencies>
    </dependencyManagement>
    ```

    由此可知，可以在dependencyManagement定义多个scope为import的dependency。

