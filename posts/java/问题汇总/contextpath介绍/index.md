# ContextPath介绍



我们在写 jsp 页面的时候，经常会看到 `<%=request.getContextPath()%>` ，这个 ContextPath 指的是项目部署到 servlet 容器中的上下文名称。

spring boot 项目，使用内嵌的tomcat 运行时，contextpath 默认是 / ，因此我们才能通过 localhost:8080 访问 springboot 应用。

可以通过在 application.properties 配置 `server.servlet.context-path` 修改 contextpath 的值，比如将 contextpath 修改为项目名

注意：contextPath 必须以 / 开头

```
server.servlet.context-path=/${spring.application.name}
```

当然，也可以引用 maven 中的 artifactId 来配置 contextPath

```
server.servlet.context-path=/@artifactId@
```

springboot 项目配置 ContextPath 后，就不能直接通过 localhost:8080 访问项目了，需要使用 localhost:8080/ContextPathName 来访问

比如将 ContextPath 配置为 abc

```
server.servlet.context-path=/abc
```

那么访问的地址就为： `http://localhost:8080/abc`


对于部署在外置的 tomcat 中的应用， ContextPath 指的是该应用在 tomcat/webapps 中的包名，即在 tomcat 中的上下文名称。

我们在写 jsp 的时候，如果引用某个静态资源或者 form 表单提交时，都需要使用 `<%=request.getContextPath()%>` 指定该应用的上下文名称（因为 springboot 默认上下文名称为 `/` ，所以不指定也可以。），否则就会出现找不到对应资源的情况。







