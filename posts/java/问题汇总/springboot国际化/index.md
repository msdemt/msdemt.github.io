# Springboot国际化


> 参考：  
> https://www.toutiao.com/i6800527379874710029/

## 介绍

国际化，也叫 `i18n` ，因为国际化的英文是 `internationalization` ，在首尾字母之间有 `18` 个字母，所以叫 `i18n` 。我们的应用如果做了国际化，就可以在不同的语言环境下显示不同的语言。

在 `Spring` 中，通过 `AcceptHeaderLocaleResolver` 对国际化提供支持，开发者通过简单的配置，就可以在项目中使用国际化的功能了。

在 `Spring Boot` 中，我们可以通过几行代码就能实现国际化的功能。

`Spring` 和 `Spring Boot` 的国际化支持，都是通过 `AcceptHeaderLocaleResolver` 解析器完成的，该解析器默认通过判断请求头中的 Accept-Language 字段来判断当前请求所属的语言环境，进而响应所属的语言。

`spring boot` 项目启动时，会自动配置 `org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration` 类（可以通过查看 `spring-boot-autoconfigure-2.4.2.jar!\META-INF\spring.factories``，该文件中配置了该自动配置类），MessageSourceAutoConfiguration` 自动配置类进行配置时，会由 `spring` 容器新建 `org.springframework.context.MessageSource` 实例，并将该实例放在 `spring` 容器中，这样，我们就可以通过 `@Autowired` 在 `spring boot` 项目中引入该实例，从而进行国际化配置了。

## 实践

下面进行 `spring boot` 国际化项目实践

1. 使用 `idea` 新建一个 `spring boot` 项目，添加 `spring-boot-starter-web` 依赖

2. 在 `/src/main/resources` 目录下，新建文件 `messages.properties` 、`messages_zh_CN.properties` 、`messages_en_US.properties` 、 `messages_zh_TW.properties`

    `messages.properties` 是默认的语言配置，`zh_CN` 是简体中文，`en_US` 是英语（美国），`zh_TW` 是繁体中文

    向 `messages.properties` 添加如下内容

    ```properties
    hello=你好，世界
    ```

    向 `messages_zh_CN.properties` 添加如下内容

    ```properties
    hello=你好，世界
    test=测试
    ```

    向 `messages_en_US.properties` 添加如下内容

    ```properties
    hello=Hello, World
    test=test
    ```

    向 `messages_zh_TW.properties` 添加如下内容

    ```properties
    hello=妳好，世界
    test=測試
    ```

3. 创建 `HelloController` ，内容如下：

    ```java
    package com.example.i18ndemo.controller;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.context.MessageSource;
    import org.springframework.context.i18n.LocaleContextHolder;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class HelloController {

        MessageSource messageSource;

        @Autowired
        public void setMessageSource(MessageSource messageSource) {
            this.messageSource = messageSource;
        }

        @RequestMapping("/hello")
        public String hello(){
            return messageSource.getMessage("hello", null, LocaleContextHolder.getLocale());
        }

        @RequestMapping("/test")
        public String test(){
            return messageSource.getMessage("test", null, LocaleContextHolder.getLocale());
        }
    }
    ```

    在 `HelloController` 中可以直接注入 `MessageSource` 实例，然后调用该实例的 `getMessge` 方法获取变量的值，第一个参数是要获取变量的 `key` ，第二个参数是如果 值 中有占位符，可以在第二个参数传入占位符对应的值，第三个参数传递一个 `Locale` 实例，相当于当前的语言环境。

4. 启动该项目

5. 使用 `idea` 自带的 `http client` 进行测试

    在 `src/test` 目录新建 `test.http` ，添加如下内容

    ```
    ###
    GET http://localhost:8080/test
    Accept-Language: en-US
    ```

    运行该测试，可以得到 `test`

    将 `Accept-Language` 改为其他语言，可以得到对应语言的 `test` 的值

## 自定义解析方式

多语言环境默认是通过请求头的 `Accept-Language` 判断响应的语言，我们也可以自定义语言解析方式，例如通过请求 `url` 参数传入该请求希望响应的语言。

添加如下配置类

```java
package com.example.i18ndemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;

@Configuration
public class I18nConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        registry.addInterceptor(interceptor);
    }

    @Bean
    LocaleResolver localeResolver(){
        SessionLocaleResolver sessionLocaleResolver = new SessionLocaleResolver();
        return sessionLocaleResolver;
    }
}
```

在上述配置中，我们配置了一个 LocalResolver 实例，该实例会替换默认的 AcceptHeaderLocaleResolver ，不同于 AcceptHeaderLocaleResolver 通过请求头来判断当前环境， SessionLocaleResolver 会将客户端的 Locale 保存到 HttpSession 对象中。

另外，还配置了一个拦截器，这个拦截器会拦截请求中 key 为 lang 的参数（不配置的话是 locale），这个参数指定了当前的环境信息。

配置完成后，启动项目

在 test.http 中添加如下测试内容

```
###
GET http://localhost:8080/test?lang=en-US
```

通过在请求中添加 lang 来指定当前的环境信息，这个只需要指定一次即可，在 session 不变的情况下，下次请求可以不必带上 lang 参数，服务端已经知道当前的环境信息了（当前环境信息已存在 session 中）。


## 自定义资源位置

默认情况下，多语言的配置文件位于 /src/main/resources 目录下，如果希望自定义，也是可以的，比如将多语言配置文件置于 /src/main/resources/i18n 目录下

但是将多语言配置文件置于 /src/main/resources/i18n 后，spring boot 去 resources 目录下找不到，需要告知 spring boot 多语言配置文件的新位置，需要在 application.yml 文件中配置如下参数

```yml
spring:
  messages:
    # 配置i18n配置文件的路径
    basename: i18n/messages
```

