# Springboot项目集成jsp


> 参考：https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-spring-mvc-template-engines  
> https://blog.csdn.net/z595746437/article/details/89366178  
> https://stackoverflow.com/questions/29782915/spring-boot-jsp-404  
> https://www.cnblogs.com/luzhanshi/p/10923867.html  

## 介绍

spring boot 官方文档关于 jsp 的介绍如下

If possible, JSPs should be avoided. There are several known [limitations](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-jsp-limitations) when using them with embedded servlet containers.

**JSP Limitations**  
When running a Spring Boot application that uses an embedded servlet container (and is packaged as an executable archive), there are some limitations in the JSP support.

With Jetty and Tomcat, it should work if you use war packaging. An executable war will work when launched with java -jar, and will also be deployable to any standard container. JSPs are not supported when using an executable jar.

Undertow does not support JSPs.

Creating a custom error.jsp page does not override the default view for error handling. Custom error pages should be used instead.

由上可知， spring boot 官方不推荐使用 jsp ，并且jsp 不支持在可执行 jar 包中使用

网上搜索可知， spring-boot-maven-plugin 插件版本 1.4.2.RELEASE 是支持 jar 包中包含 jsp 的，高于此版本就不支持了。若使用高于 1.4.2.RELEASE 的 spring-boot-maven-plugin 插件构建可执行 jar 包，访问 jsp 页面会出现 404 。原因未知。

spring boot 项目中，若使用 jsp ，有两种方案
- 第一种方案是将 jsp 文件放在 src/main/webapp/jsp 目录下
- 第二种方案是将 jsp 文件放在 src/main/resources/META-INF/resources/WEB-INF/jsp 目录下

## 实践

### spring boot 集成 jsp 生成可执行 jar 包

#### 可执行 jar 包

若希望生成可执行 `jar` 包，则需要使用 `1.4.2.RELEASE` 版本的 `spring-boot-maven-plugin`

1. 新建 `spring boot` 项目

2. `pom.xml` 文件中，修改 `spring-boot-maven-plugin` 的版本为 `1.4.2.RELEASE` ，添加 `spring-boot-starter-web` 依赖，此外，还需添加 `jstl` 依赖和 `tomcat-embed-jasper` 依赖， 打包方式（ `packaging` ）默认为 `jar` ，也可以不显式指定。 
   
   
   如果使用 `src/main/webapp/WEB-INF/jsp` 存放 `jsp` 文件，需要添加资源打包方式（ `build/resources/resource` ）那么对应的 `pom.xml` 文件内容如下

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.4.2</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <groupId>com.example</groupId>
        <artifactId>jsp-executable-jar-demo</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>jsp-executable-jar-demo</name>
        <description>Demo project for Spring Boot</description>
        
        <packaging>jar</packaging>
        
        <properties>
            <java.version>1.8</java.version>
        </properties>
        
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <optional>true</optional>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>

            <!--JavaServer Pages Standard Tag Library，JSP标准标签库-->
            <dependency>
                <groupId>javax.servlet</groupId>
                <artifactId>jstl</artifactId>
            </dependency>
            <!--内置tomcat对Jsp支持的依赖，用于编译Jsp-->
            <dependency>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-jasper</artifactId>
                <scope>provided</scope>
            </dependency>
        </dependencies>

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
                <resource>
                    <directory>${basedir}/src/main/webapp</directory>
                    <targetPath>META-INF/resources</targetPath>
                    <includes>
                        <include>**/**</include>
                    </includes>
                    <filtering>false</filtering>
                </resource>
            </resources>

            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>1.4.2.RELEASE</version>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </build>

    </project>
    ```

    如果使用 `src/main/resources/META-INF/resources/WEB-INF/jsp` 存放 `jsp` 文件，使用 `spring boot` 默认的资源打包方式即可，对应的 pom.xml 文件内容如下

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.4.2</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <groupId>com.example</groupId>
        <artifactId>jsp-executable-jar-demo</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>jsp-executable-jar-demo</name>
        <description>Demo project for Spring Boot</description>

        <packaging>jar</packaging>

        <properties>
            <java.version>1.8</java.version>
        </properties>

        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <optional>true</optional>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>

            <!--JavaServer Pages Standard Tag Library，JSP标准标签库-->
            <dependency>
                <groupId>javax.servlet</groupId>
                <artifactId>jstl</artifactId>
            </dependency>
            <!--内置tomcat对Jsp支持的依赖，用于编译Jsp-->
            <dependency>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-jasper</artifactId>
                <scope>provided</scope>
            </dependency>
        </dependencies>

        <build>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>1.4.2.RELEASE</version>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </build>

    </project>
    ```

    添加后，点击下 `idea` 右侧 `maven` 栏的 `reload` 按钮，才能将依赖加载到项目中。

3. 如果使用 `src/main/webapp/WEB-INF/jsp` 存放 `jsp` 文件，需要在 `src/main` 目录下新建目录 `webapp/WEB-INF/jsp` ，此时选中 `jsp` 目录后右键无法新建 `jsp` 文件，需要按照第四步配置下
    由第四步可知，其实最终是将 `jsp` 文件放在 `META-INF/resources/WEB-INF/jsp` 目录的。

    如果使用 `src/main/resources/META-INF/resources/WEB-INF/jsp` 存放 `jsp` 文件，则跳过第四步。
    
    
4. 点击 `Project Structure` ，配置 `web 资源目录`

    ![project structure](/images/20210205-project%20structure.png)  

    注意： 该步骤配置了之后， idea 就会将 webapp 目录作为资源目录，在 idea 上直接运行时，idea 会把资源目录下的文件放在运行路径中，是能正常显式 jsp 页面的。但是构建为 jar 包运行后，无法显式 jsp 页面，解决方法为在 pom.xml 添加如下内容，表示将 ${basedir}/src/main/webapp 的所有文件编译时移动到 META-INF/resources 目录下，这样可执行 jar 包就能正常显示 jsp 了。

    ```xml
    <resources>
        <resource>
            <directory>${basedir}/src/main/webapp</directory>
            <targetPath>META-INF/resources</targetPath>
            <includes>
                <include>**/**</include>
            </includes>
            <filtering>false</filtering>
        </resource>
    </resources>
    ```

    注意：如果没有把 webapp 目录配置为 web 资源目录的时候，加上上面的 resource 配置后，构建后的 target/class 和 jar 包里没有包含 src/main/resources 目录的 application 配置文件，也即构建后的 jar 包里丢失了 spring.mvc.view.prefix 等配置。如下图

    ![图 1](images/20210206-未配置web资源目录.png)  


    我猜测应该是该 build/resources/resource 配置 将 spring boot parent 的 build/resources/resource 配置覆盖了。执行构建 jar 包时的日志如下

    ```log
    /usr/local/jdk/oraclejdk/jdk1.8.0_202.jdk/Contents/Home/bin/java -Dmaven.multiModuleProjectDirectory=/Users/hekai/Documents/workspace/IdeaProjects/jsp-executable-jar-demo -Dmaven.home=/Applications/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3 -Dclassworlds.conf=/Applications/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3/bin/m2.conf -Dmaven.ext.class.path=/Applications/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven-event-listener.jar -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=58454:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Applications/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3/boot/plexus-classworlds.license:/Applications/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3/boot/plexus-classworlds-2.6.0.jar org.codehaus.classworlds.Launcher -Didea.version=2020.3.2 package
    [INFO] Scanning for projects...
    [INFO] 
    [INFO] ----------------< com.example:jsp-executable-jar-demo >-----------------
    [INFO] Building jsp-executable-jar-demo 0.0.1-SNAPSHOT
    [INFO] --------------------------------[ jar ]---------------------------------
    [INFO] 
    [INFO] --- maven-resources-plugin:3.2.0:resources (default-resources) @ jsp-executable-jar-demo ---
    [INFO] Using 'UTF-8' encoding to copy filtered resources.
    [INFO] Using 'UTF-8' encoding to copy filtered properties files.
    [INFO] Copying 1 resource to META-INF/resources
    [INFO] 
    [INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ jsp-executable-jar-demo ---
    [INFO] Changes detected - recompiling the module!
    [INFO] Compiling 2 source files to /Users/hekai/Documents/workspace/IdeaProjects/jsp-executable-jar-demo/target/classes
    [INFO] 
    [INFO] --- maven-resources-plugin:3.2.0:testResources (default-testResources) @ jsp-executable-jar-demo ---
    [INFO] Using 'UTF-8' encoding to copy filtered resources.
    [INFO] Using 'UTF-8' encoding to copy filtered properties files.
    [INFO] skip non existing resourceDirectory /Users/hekai/Documents/workspace/IdeaProjects/jsp-executable-jar-demo/src/test/resources
    [INFO] 
    [INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ jsp-executable-jar-demo ---
    [INFO] Changes detected - recompiling the module!
    [INFO] Compiling 1 source file to /Users/hekai/Documents/workspace/IdeaProjects/jsp-executable-jar-demo/target/test-classes
    [INFO] 
    [INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ jsp-executable-jar-demo ---
    [INFO] 
    [INFO] -------------------------------------------------------
    [INFO]  T E S T S
    [INFO] -------------------------------------------------------
    [INFO] Running com.example.jspexecutablejardemo.JspExecutableJarDemoApplicationTests
    ```

    可以看到 maven-resources-plugin 插件 只是执行了 pom.xml 中定义的资源拷贝（将 ${basedir}/src/main/webapp 下的资源拷贝到 META-INF/resources 目录）

    没有执行 spring boot 自带的资源拷贝，spring boot parent 中定义的资源拷贝如下

    ```xml
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
    ```

    这样就会导致构建后的 `jar` 包缺少 `src/main/resource` 下的文件，导致 `jar` 包运行异常。

    
    所以，如果使用 `src/main/webapp/WEB-INF/jsp` 方式存放 `jsp` 文件，建议完整的 资源打包方式 配置如下

    ```xml
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
      <resource>
        <directory>${basedir}/src/main/webapp</directory>
        <targetPath>META-INF/resources</targetPath>
        <includes>
          <include>**/**</include>
        </includes>
        <filtering>false</filtering>
      </resource>
    </resources>
    ```

    避免由于忘记配置 `web 资源目录`导致构建的 `jar` 包缺少文件。

    如果使用 `src/main/resources/META-INF/resources/WEB-INF/jsp` 存放 `jsp` 文件，那就不用配置 `build/resources` 了，使用 `spring boot` 默认的资源打包方式即可


5. 选中 jsp 目录，右键 -> New -> JSP/JSPX ，新建 hello.jsp

    ```jsp
    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <html>
    <head>
        <title>Test</title>
    </head>
    <body>
    ${world}
    </body>
    </html>
    ```

6. application.properties 配置 jsp 资源查找路径

    ```properties
    spring.mvc.view.prefix=/WEB-INF/jsp/
    spring.mvc.view.suffix=.jsp
    ```

7. 添加 Controller ，代码如下

    ```java
    package com.example.jspexecutablejardemo.controller;

    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.servlet.ModelAndView;
    @RequestMapping("/")
    @Controller
    public class HelloController {

        @RequestMapping("/hello")
        public ModelAndView hello(){
            ModelAndView mav = new ModelAndView("/hello");
            mav.addObject("name", "world");
            return mav;
        }
    }
    ```

    注意：如果 `spring.mvc.view.prefix` 配置的是 `/WEB-INF/jsp` ，那么 `Controller` 里的 `new ModelAndView("/hello")` 的参数必须以 `/` 开头，如果参数使用 `hello` ，会找不到 `jsp` 页面。

    所以个人建议配置 `spring.mvc.view.prefix` 时以 `/` 结尾，本文为 `/WEB-INF/jsp/` ，这样的话 `new ModelAndView("/hello")` 的参数既可以是 `/hello` 也可以是 `hello`

    这里需要返回视图，类上不能使用 `@RestController`

至此，就配置好了，可以使用 maven 构建为 jar 包后运行 jsp 项目了。

注意： 如果构建 springboot 集成 jsp 可执行 jar 包，目前只能使用 spring-boot-maven-plugin 插件的 1.4.2.RELEASE 及以下的版本，高于这个版本构建的 jar 包，运行后无法访问到 jsp 页面。


### spring boot 集成 jsp 生成可执行 war 包

spring boot 官方文档 jsp 介绍中有如下内容：

With Jetty and Tomcat, it should work if you use war packaging. An executable war will work when launched with java -jar, and will also be deployable to any standard container. JSPs are not supported when using an executable jar.

使用 jetty 或 tomcat 容器时，jsp 使用 war 包构建能够正常工作。可以使用 jar -jar 命令启动可执行 war 包，该 war 包也可以部署到任何标准的容器中。 jsp 不支持以可执行 jar 包的方式运行。


> 参考：  
> https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-create-a-deployable-war-file  
> https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-embedded-container-context-initializer

上面的官方链接讲的比较详细了，偷个懒就不介绍了。

1. pom.xml 中引入依赖

    pom.xml 文件内容示例

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <parent>
            <artifactId>simple-013-jsp-demo</artifactId>
            <groupId>org.msdemt</groupId>
            <version>${revision}</version>
        </parent>
        <modelVersion>4.0.0</modelVersion>
        <artifactId>simple-013-jsp-war-demo</artifactId>
        <packaging>war</packaging>

        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <optional>true</optional>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
                <scope>provided</scope>
            </dependency>

            <!--JavaServer Pages Standard Tag Library，JSP标准标签库-->
            <dependency>
                <groupId>javax.servlet</groupId>
                <artifactId>jstl</artifactId>
            </dependency>
            <!--内置tomcat对Jsp支持的依赖，用于编译Jsp-->
            <dependency>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-jasper</artifactId>
                <scope>provided</scope>
            </dependency>
        </dependencies>

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
                <resource>
                    <directory>${basedir}/src/main/webapp</directory>
                    <targetPath>META-INF/resources</targetPath>
                    <includes>
                        <include>**/**</include>
                    </includes>
                    <filtering>false</filtering>
                </resource>
            </resources>

            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </build>

    </project>
    ```


    > 说明：
    > - spring-boot-maven-plugin 版本没有限制，此处使用的是默认版本
    > - 需要指定构建 `war` 包 （ `<packaging>war</packaging>` ）  
    > - 需要明确引入 spring-boot-starter-tomcat 依赖，并且将 scope 置为 provided ，这个依赖默认是会自动引入的，此处显式引入，是为了将它置为 provided ，表示构建包时不会将该依赖放入包中，避免和 servlet 容器冲突。
    > - 最好也加上自定义的资源打包方式，将 src/main/resource 和 src/main/webapp 的资源放入构建的包中

2. 修改 spring boot 启动类

    spring boot 启动类示例如下

    ```java
    package com.example.jspexecutablewardemo;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.boot.builder.SpringApplicationBuilder;
    import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

    @SpringBootApplication
    public class JspExecutableWarDemoApplication extends SpringBootServletInitializer {

        @Override
        protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
            return builder.sources(JspExecutableWarDemoApplication.class);
        }

        public static void main(String[] args) {
            SpringApplication.run(JspExecutableWarDemoApplication.class, args);
        }

    }
    ```

    构建 war 包时，spring boot 启动类需要继承 SpringBootServletInitializer ，并且实现 configure() 方法。这样 war 包构建的 spring boot 应用才能使用 spring 框架的 servlet 3.0 的支持，并且当应用由 servlet 容器启动时，你也能够对应用进行配置。


war 包扩展：

嵌入式的 servlet 容器不会直接执行 servlet 3.0 及以上版本的 javax.servlet.ServletContainerInitializer 接口或者 Spring 的 org.springframework.web.WebApplicationInitializer 接口。这个是故意这么设计的，为了降低在 war 包中运行第三方库可能中断 spring boot 应用的风险。

如果需要在 spring boot 应用中执行 servlet 上下文的初始化，那么你应该注册一个实现了 org.springframework.boot.web.servlet.ServletContextInitializer 接口的 bean 。单独的 onStart 方法提供对 ServletContext 的访问，并且可以在必要的时候轻松地用作现有的 WebApplicationInitializer 的适配器。

怎么理解上面这段话呢？

https://blog.csdn.net/z69183787/article/details/104557820

