# Swagger的使用


想将swagger集成到我的项目里了，项目比较老了，是使用的servlet和webservice暴漏的服务

pom.xml中加入swagger依赖后，启动报错，

```log
Caused by: java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ObjectMapper
```

swagger依赖jackson，所以需要添加jackson依赖

加上jackson依赖后，报错如下

```log
org.springframework.context.ApplicationContextException: Failed to start bean 'documentationPluginsBootstrapper'; nested exception is java.lang.NoClassDefFoundError: org/springframework/http/codec/multipart/FilePart
```

因为当前项目使用的spring版本是4.3.30.RELEASE，但是`org/springframework/http/codec/multipart/FilePart`是在spring5.0开始加的，所以需要使用低版本的swagger。

swagger集成好了之后，发现webservice无法使用swagger，虽然servlet可以使用，但是配置比较复杂，不友好，所以放弃了

servlet配置swagger可以参考官方demo：https://github.com/swagger-api/swagger-samples/tree/master/java/java-servlet


Swagger 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

适用于restful风格的web服务，不适用于servlet和webservice服务。

