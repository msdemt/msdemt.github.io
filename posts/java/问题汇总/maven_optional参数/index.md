# Maven_optional参数


https://www.cnblogs.com/FraserYu/p/11796301.html


比如：https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using-boot-devtools

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

```gradle
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}
```


Flagging the dependency as optional in Maven or using the developmentOnly configuration in Gradle (as shown above) prevents devtools from being **transitively** applied to other modules that use your project.


