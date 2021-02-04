# Springboot集成redis


由[springboot官方参考文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-redis)可知

Redis is a cache, message broker, and richly-featured key-value store. Spring Boot offers basic auto-configuration for the Lettuce and Jedis client libraries and the abstractions on top of them provided by Spring Data Redis.  
redis是一个缓存、消息代理、也是功能丰富的键值存储仓库。spring boot 为 Lettuce 和 Jedis 客户端库提供了基本的自动配置，并在 Lettuce 和 Jedis 客户端库之上由 spring data redis 提供了抽象。

There is a spring-boot-starter-data-redis “Starter” for collecting the dependencies in a convenient way. By default, it uses Lettuce. That starter handles both traditional and reactive applications.  
spring-boot-starter-data-redis 这个starter用于方便地收集redis的相关依赖项，默认情况使用 Lettuce 作为 redis客户端，这个starter可以处理传统应用和响应式应用。

You can inject an auto-configured **RedisConnectionFactory**, **StringRedisTemplate**, or vanilla **RedisTemplate** instance as you would any other Spring Bean. By default, the instance tries to connect to a Redis server at localhost:6379. The following listing shows an example of such a bean:  
你可以注入一个已自动配置好的RedisConnectionFactory、StringRedisTemplate或 普通的 RedisTemplate 实例像注入其他spring bean 一样。默认情况下，该实例尝试连接到地址为 localhost:6379 的redis服务器，如下列表展示了注入这样的bean的例子：  

```java
@Component
public class MyBean {

    private StringRedisTemplate template;

    @Autowired
    public MyBean(StringRedisTemplate template) {
        this.template = template;
    }

    // ...

}
```

You can also register an arbitrary number of beans that implement **LettuceClientConfigurationBuilderCustomizer** for more advanced customizations. If you use Jedis, **JedisClientConfigurationBuilderCustomizer** is also available.  
你也可以注册任意数量的实现了LettuceClientConfigurationBuilderCustomizer的bean，来获得高级的自定义配置。如果你使用的是Jedis，JedisClientConfigurationBuilderCustomizer也是可以使用的

If you add your own @Bean of any of the auto-configured types, it replaces the default (except in the case of RedisTemplate, when the exclusion is based on the bean name, redisTemplate, not its type). By default, if commons-pool2 is on the classpath, you get a pooled connection factory.  
如果你添加自己的任意类型的自动配置的@Bean，它将替换默认的配置（当排除的是基于bean名称，而不是基于类型的redisTemplate的时候，RedisTemplate不会被替换）。默认情况下，如果类路径中存在 commons-pools2 依赖包，你可以得到一个池化的连接工厂。

springboot官方文档中对springboot如何集成redis的介绍不太详细。

github上由spring-projects官方的examples，地址：https://github.com/spring-projects/spring-data-examples.git

> 参考：  
> https://blog.csdn.net/qq_41921994/article/details/109627736

Lettuce 和 Jedis 的都是连接Redis Server的客户端程序。Jedis在实现上是直连redis server，多线程环境下非线程安全（即多个线程对一个连接实例操作，是线程不安全的），除非使用连接池，为每个Jedis实例增加物理连接。Lettuce基于Netty的连接实例（StatefulRedisConnection），可以在多个线程间并发访问，且线程安全，满足多线程环境下的并发访问（即多个线程公用一个连接实例，线程安全），同时它是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。

lettuce依赖commons-pool2

## 实践

### win10系统启动 redis-server

下载 redis server win10版本安装包，下载地址：https://github.com/MicrosoftArchive/redis/releases

比如下载 Redis-x64-3.0.504.zip，解压到某个目录下，双击 redis-server.exe 即可将 redis 启动。

若启动失败，显示如下错误

```powershell
PS D:\Develop\Redis-x64-3.0.504> .\redis-server.exe
[21368] 29 Jan 09:16:00.558 # Warning: no config file specified, using the default config. In order to specify a config file use D:\Develop\Redis-x64-3.0.504\redis-server.exe /path/to/redis.conf
[21368] 29 Jan 09:16:00.565 # Creating Server TCP listening socket *:6379: bind: No such file or directory
```

网上搜索的方法不好使，重启下电脑就可以了

### 写代码

新建 simple-011-springboot-redis-demo 项目

src/main/java目录添加springboot启动类

```java
package org.msdemt.simple.redis_demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RedisApplication {

    public static void main(String[] args) {
        SpringApplication.run(RedisApplication.class, args);
    }
}
```

添加User类

```java
package org.msdemt.simple.redis_demo.pojo;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private static final long serialVersionUID = 2489911387827654482L;

    private String name;
    private Integer age;
    private String address;
}
```

pom文件中添加如下依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!--spring-boot-starter-data-redis 默认使用 Lettuce 作为 redis 客户端，Lettuce 需要依赖 commons-pool2-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```

然后在 idea 右边栏，maven页面，reload一下，这样项目才能下载依赖包。

因为 spring-boot-starter-data-redis 默认连接 127.0.0.1:6379 的 redis server，为了方便，此时可以不修改 application.yml 文件

当然，如果想要显式地配置redis的参数，使得项目更清晰，可以新建 src/main/resources/application.yml 文件，并在application.yml文件中添加如下内容

```yml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    lettuce: #springboot默认使用lettuce作为redis客户端
      #在关闭客户端连接之前等待任务处理完成的最长时间，在这之后，无论任务是否执行完成，都会被执行器关闭，默认100ms
      shutdown-timeout: 100
      pool:
        # 连接池最大连接数（负值表示没有限制）
        max-active: 8
        # 连接池中的最大空闲连接
        max-idle: 8
        # 连接池中的最小空闲连接
        min-idle: 0
        # 连接池最大阻塞等待时间（使用负值代表没有限制），单位ms
        max-wait: -1
  cache:
    redis:
      #是否缓存空值，默认为true
      cache-null-values: false
```

在src/test目录下新建测试源码目录（src/test/java）和测试资源目录（src/test/resources）

src/test/java 目录下添加测试文件

```java
package org.msdemt.simple.redis_demo.test;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.msdemt.simple.redis_demo.pojo.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;

@SpringBootTest
public class RedisTest {

    @Autowired
    private RedisTemplate redisTemplate;

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Test
    public void testRedisTemplateForSetStringValue(){

        redisTemplate.opsForValue().set("hello", "world");
        stringRedisTemplate.opsForValue().set("key", "value");
        Assertions.assertEquals("world", redisTemplate.opsForValue().get("hello"));
        Assertions.assertEquals("value", stringRedisTemplate.opsForValue().get("key"));
    }

    @Test
    public void testRedisTemplateForSetObjectValue() throws JsonProcessingException {
        User user = new User("nike", 30, "beijing");
        redisTemplate.opsForValue().set("admin", user);
        System.out.println(user.toString());
        System.out.println(new ObjectMapper().writeValueAsString(user));
        System.out.println((User)redisTemplate.opsForValue().get("admin"));
        Assertions.assertEquals(user, redisTemplate.opsForValue().get("admin"));
    }
}
```

执行该测试包中的测试类时，是不会按照src/main/resources/application.yml中的配置进行的，如果想定义测试类的redis配置，需要在src/test/resources/application.yml文件中添加redis的配置

执行 testRedisTemplateForSetObjectValue 后，使用 redis desktop manager 查看 redis server 中的数据，发现 redisTemplate 存储的键值是乱码的，stringRedisTemplate 存储的键值是正常的，为什么呢？

`StringRedisTemplate` 是 `RedisTemplate<String, String>` 的子类
```java
public class StringRedisTemplate extends RedisTemplate<String, String> {
```

通过debug，可以发现 RedisTemplate 的序列化使用的是 JdkSerializationRedisSerializer，StringRedisTemplate的序列化使用的是StringRedisSerializer，应该是序列化的原因，导致的乱码。

具体见下

springboot启动时会查找META-INF/spring.factories文件，所以先看下spring-boot-autoconfigure包下META-INF/spring.factories文件，发现有 `org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration`，进入这个类里看下，

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)  //当类路径中有RedisOperations类时，该类会进行自动配置
@EnableConfigurationProperties(RedisProperties.class)  //RedisProperties里配置了redis连接参数
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class }) //RedisConnectionFactory 是在这里面实例化的
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")  //当容器中没有名为redisTemplate的实例时，该实例进行初始化
	@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```

可以看到，在上面这个类里对RedisTemplate和StringRedisTemplate进行了实例化并且放在了spring容器里，所以我们才可以使用@Autowired进行自动注入。

org.springframework.data.redis.core.RedisTemplate 类

```java
	public RedisTemplate() {}

	@Override
	public void afterPropertiesSet() {

		super.afterPropertiesSet();

		boolean defaultUsed = false;

		if (defaultSerializer == null) {

			defaultSerializer = new JdkSerializationRedisSerializer(
					classLoader != null ? classLoader : this.getClass().getClassLoader());
		}

		if (enableDefaultSerializer) {

			if (keySerializer == null) {
				keySerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (valueSerializer == null) {
				valueSerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (hashKeySerializer == null) {
				hashKeySerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (hashValueSerializer == null) {
				hashValueSerializer = defaultSerializer;
				defaultUsed = true;
			}
		}

		if (enableDefaultSerializer && defaultUsed) {
			Assert.notNull(defaultSerializer, "default serializer null and not all serializers initialized");
		}

		if (scriptExecutor == null) {
			this.scriptExecutor = new DefaultScriptExecutor<>(this);
		}

		initialized = true;
	}
```

初始化 RedisTemplate 实例，执行其构造方法后，defaultSerializer 为 null，所以 defaultSerializer 置为了JdkSerializationRedisSerializer，RedisTemplate 构造方法中没有对keySerializer、valueSerializer、hashKeySerializer、hashValueSerializer赋值，所以它们也是null，所以被置为了JdkSerializationRedisSerializer

org.springframework.data.redis.core.StringRedisTemplate 类中

```java
public class StringRedisTemplate extends RedisTemplate<String, String> {

	public static final StringRedisSerializer UTF_8 = new StringRedisSerializer(StandardCharsets.UTF_8);

    	public StringRedisTemplate() {
		setKeySerializer(RedisSerializer.string());
		setValueSerializer(RedisSerializer.string());
		setHashKeySerializer(RedisSerializer.string());
		setHashValueSerializer(RedisSerializer.string());
	}

}
```

因为 StringRedisTemplate 继承 RedisTemplate，所以先执行 RedisTemplate 的构造方法，是空的，再执行 StringRedisTemplate 的构造方法，设置了keySerializer、valueSerializer、hashKeySerializer、hashValueSerializer的值为StringRedisSerializer，构造方法执行完后，还要执行下父类中定义的afterPropertiesSet方法，由于keySerializer、valueSerializer、hashKeySerializer、hashValueSerializer都有值了，所以不会重新赋值。所以 StringRedisTemplate 对象使用的序列化是 StringRedisSerializer


springboot里为redis提供了哪些序列化工具呢？

查看 org.springframework.data.redis.serializer.RedisSerializer 接口的实现类
- org.springframework.data.redis.serializer.ByteArrayRedisSerializer
- org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer
- org.springframework.data.redis.serializer.GenericToStringSerializer
- org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer
- org.springframework.data.redis.serializer.JdkSerializationRedisSerializer
- org.springframework.data.redis.serializer.OxmSerializer
- org.springframework.data.redis.serializer.StringRedisSerializer

> 参考：  
> https://gitee.com/fengzxia/spring-boot-redis-cache/blob/master/Jackson%20Serializer%E7%BC%93%E5%AD%98%E6%95%B0%E6%8D%AE%E5%BA%8F%E5%88%97%E5%8C%96%E9%97%AE%E9%A2%98.md  


JdkSerializationRedisSerializer 使用JDK提供的序列化功能。 优点是反序列化时不需要提供类型信息(class)，但缺点是序列化后的结果非常庞大，是JSON格式的5倍左右，这样就会消耗redis服务器的大量内存。

Jackson2JsonRedisSerializer 使用Jackson库将对象序列化为JSON字符串。优点是速度快，序列化后的字符串短小精悍。但缺点也非常致命，那就是此类的构造函数中有一个类型参数，必须提供要序列化对象的类型信息(.class对象)。 通过查看源代码，发现其只在反序列化过程中用到了类型信息。如果使用该序列化器，第一种选择是在该序列化器的构造函数中传入 Object 作为序列化的类型，这样就可以序列化所有对象了，但是缺点是无法进行反序列化（ClassCastException）（当然，可以通过将对象转为json存储到redis中，然后再从redis中取出json，再转为对象，解决无法反序列化的问题），第二种选择是为每个需要存入redis的对象都配置一个Jackson2JsonRedisSerializer，这种做法无疑是不现实的，工作量太大。

GenericJackson2JsonRedisSerializer 和 Jackson2JsonRedisSerializer 类似。但是它不需要提供序列化对象的类型信息，解决了 Jackson2JsonRedisSerializer 的反序列化问题，所以推荐使用这个。

RedisTemplate默认使用的JdkSerializationRedisSerializer效率不高，查看时还出现乱码，而且序列化后占用的内存空间也很大。如果希望替换替换它的序列化器，可以仿照RedisAutoConfiguration类，自定义一个返回值为`RedisTemplate<Object, Object>`的使用@Bean修饰的方法。

配置redisTemplate的序列化器

测试用例：

```java
    @Test
    public void testRedisTemplateForSetObjectValue() throws JsonProcessingException {
        User user = new User("nike", 30, "beijing");
        redisTemplate.opsForValue().set("admin", user);
        System.out.println(user.toString());
        System.out.println(new ObjectMapper().writeValueAsString(user));
        System.out.println((User)redisTemplate.opsForValue().get("admin"));
        Assertions.assertEquals(user, redisTemplate.opsForValue().get("admin"));
    }
```

1. 使用默认的序列化器 JdkSerializationRedisSerializer，
  
    使用 JdkSerializationRedisSerializer 的时候，被序列化的类必须实现 Serializable 接口，否则序列化时会出现错误：
    ```log
    Caused by: org.springframework.core.serializer.support.SerializationFailedException: Failed to serialize object using DefaultSerializer; nested exception is java.lang.IllegalArgumentException: DefaultSerializer requires a Serializable payload but received an object of type [org.msdemt.simple.redis_demo.pojo.User]
    ```

    测试用例成功
    
    使用 redis desktop manager 查看

    key: "\xAC\xED\x00\x05t\x00\x05admin"

    value（使用plain text视图查看，存储的是二进制）: ��

    value使用16进制视图查看：

    ```
    \xAC\xED\x00\x05sr\x00&org.msdemt.simple.redis_demo.pojo.User"\x8D\xF17\x11\xAA\xCFR\x02\x00\x03L\x00\x07addresst\x00\x12Ljava/lang/String;L\x00\x03aget\x00\x13Ljava/lang/Integer;L\x00\x04nameq\x00~\x00\x01xpt\x00\x07beijingsr\x00\x11java.lang.Integer\x12\xE2\xA0\xA4\xF7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xAC\x95\x1D\x0B\x94\xE0\x8B\x02\x00\x00xp\x00\x00\x00\x1Et\x00\x04nike
    ```

    测试用例成功，输出结果:
    ```
    User(name=nike, age=30, address=beijing)
    {"name":"nike","age":30,"address":"beijing"}
    User(name=nike, age=30, address=beijing)
    User(name=nike, age=30, address=beijing)
    ```


2. 使用 Jackson2JsonRedisSerializer 解析器，解析器参数置为Object，

    添加如下配置类

    ```java
    package org.msdemt.simple.redis_demo.config;

    import org.msdemt.simple.redis_demo.pojo.User;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.data.redis.core.RedisTemplate;
    import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

    @Configuration
    public class RedisConfig {

        @Bean
        public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
            RedisTemplate<Object, Object> template = new RedisTemplate<>();
            template.setConnectionFactory(redisConnectionFactory);

            template.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(Object.class));

            return template;
        }
    }
    ```

    使用 redis desktop manager 查看

    key: "admin"

    value: {"name":"nike","age":30,"address":"beijing"}

    输出结果：

    ```
    User(name=nike, age=30, address=beijing)
    {"name":"nike","age":30,"address":"beijing"}
    {name=nike, age=30, address=beijing}

    java.lang.ClassCastException: java.util.LinkedHashMap cannot be cast to org.msdemt.simple.redis_demo.pojo.User
    ```

    此时(User)redisTemplate.opsForValue().get("admin")发生类型强转错误，因为此时redis中没有存储类型信息

    当然，这是可以将对象转为json存储到redis中，然后再从redis中取出json，再转为对象。

    redisTemplate.opsForValue().get("admin") 打印出的结果是 {name=nike, age=30, address=beijing}

    并且assert判断会失败（assert判断还没执行到，因为上一条抛出异常了），信息如下（期望是一个user类，但实际上redis中存储的是字符串）

    ```
    org.opentest4j.AssertionFailedError:
    Expected :User(name=nike, age=30, address=beijing)
    Actual   :{name=nike, age=30, address=beijing}
    ```

    使用 Jackson2JsonRedisSerializer 解析器，解析器参数置为User，该序列化器可以对User类进行序列化和反序列化

    ```java
    package org.msdemt.simple.redis_demo.config;

    import org.msdemt.simple.redis_demo.pojo.User;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.data.redis.core.RedisTemplate;
    import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

    @Configuration
    public class RedisConfig {

        @Bean
        public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
            RedisTemplate<Object, Object> template = new RedisTemplate<>();
            template.setConnectionFactory(redisConnectionFactory);

            template.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(User.class));

            return template;
        }
    }
    ```

    使用 redis desktop manager 查看

    key: "admin"

    value: {"name":"nike","age":30,"address":"beijing"}

    测试用例执行成功，输出结果：

    ```
    User(name=nike, age=30, address=beijing)
    {"name":"nike","age":30,"address":"beijing"}
    User(name=nike, age=30, address=beijing)
    User(name=nike, age=30, address=beijing)
    ```

3. 使用 GenericJackson2JsonRedisSerializer 解析器，该解析器会自动保存对象类型，不用传入参数，

    ```java
    package org.msdemt.simple.redis_demo.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.data.redis.core.RedisTemplate;
    import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;

    @Configuration
    public class RedisConfig {

        @Bean
        public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
            RedisTemplate<Object, Object> template = new RedisTemplate<>();
            template.setConnectionFactory(redisConnectionFactory);

            template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());

            return template;
        }
    }
    ```

    使用 redis desktop manager 查看

    key: "admin"

    value: {"@class":"org.msdemt.simple.redis_demo.pojo.User","name":"nike","age":30,"address":"beijing"}

    测试用例执行成功，输出结果：

    ```
    User(name=nike, age=30, address=beijing)
    {"name":"nike","age":30,"address":"beijing"}
    User(name=nike, age=30, address=beijing)
    User(name=nike, age=30, address=beijing)
    ```

总结，一般情况下，使用 GenericJackson2JsonRedisSerializer 即可。


## 高性能序列化器 kyro

> 参考:  
> https://www.cnblogs.com/hntyzgn/p/7122709.html
> https://blog.csdn.net/boling_cavalry/article/details/80710774
> https://github.com/EsotericSoftware/kryo  


Kryo is a fast and efficient binary object graph serialization framework for Java.  
Kryo是用于Java的快速高效的二进制对象图序列化框架。

Kryo publishes two kinds of artifacts/jars:
- the default jar (with the usual library dependencies) which is meant for direct usage in applications (not libraries)
- a dependency-free, "versioned" jar which should be used by other libraries. Different libraries shall be able to use different major versions of Kryo.  

To use the latest Kryo release in your application, use this dependency entry in your pom.xml:
```xml
<dependency>
   <groupId>com.esotericsoftware</groupId>
   <artifactId>kryo</artifactId>
   <version>5.0.3</version>
</dependency>
```
To use the latest Kryo release in a library you want to publish, use this dependency entry in your pom.xml:
```xml
<dependency>
   <groupId>com.esotericsoftware.kryo</groupId>
   <artifactId>kryo5</artifactId>
   <version>5.0.3</version>
</dependency>
```




```log
2021-01-29 14:18:45.088 ERROR 10824 --- [nio-8080-exec-1] o.m.s.r.serializer.KryoRedisSerializer   : Class is not registered: org.msdemt.simple.redis_kyro.pojo.User
Note: To register this class use: kryo.register(org.msdemt.simple.redis_kyro.pojo.User.class);

java.lang.IllegalArgumentException: Class is not registered: org.msdemt.simple.redis_kyro.pojo.User
Note: To register this class use: kryo.register(org.msdemt.simple.redis_kyro.pojo.User.class);
```
