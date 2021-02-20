# Springboot项目运行需要使用临时目录


## 临时文件

`spring boot` 在 `centos7` 上运行后，在 `/tmp` 目录自动生成了如下文件夹

```shell
[root@centos7 tmp]# ls
hsperfdata_root  tomcat.8080.9098756374404445017  tomcat-docbase.8080.5749437145987244272
[root@centos7 tmp]# 
```

1. `hsperfdata_root` 目录存储的是 `spring boot` 项目运行的 `pid` （进程号）

    ```shell
    [root@centos7 tmp]# ps -ef|grep java
    root      19819   4174  3 17:02 pts/0    00:00:17 java -jar i18n-demo-0.0.1-SNAPSHOT.jar
    root      31247  20944  0 17:10 pts/1    00:00:00 grep --color=auto java
    [root@centos7 tmp]# ls hsperfdata_root/
    19819
    [root@centos7 tmp]# 
    ```

2. `tomcat.8080.9098756374404445017` （tomcat.端口号.其他） 里存储的是 `tomcat` 的 `work` 目录

    `work` 目录是 `tomcat` 的工作目录，也就是 tomcat 把 `jsp` 转换为 `class` 文件的工作目录。

    当浏览器访问某个 `jsp` 页面时， `tomcat` 会在 `work` 目录里把这个 jsp 页面转换为 java 文件，比如将 `index.jsp` 转换为 `index_jsp.java` ，然后编译为 `index_jsp.class` ，最后 `tomcat` 容器通过 `ClassLoader` 把 `index_jsp.class` 类装载入内存，从而响应客户端浏览器。

    `tomcat` 会定时扫描容器内的 `jsp` 文件，读取每个文件的属性，当发现某个 `jsp` 文件发生改变时（ `jsp` 文件的最后修改时间与上次扫描时的不相同）， `tomcat` 会重新转换、编译这个 `jsp` 文件，但是 to`mcat 的扫描不是实时的，需要几分钟后才能生效。为了立刻生效，可以在修改 `jsp` 页面后清除 `work` 目录里的文件。

    `tomcat` 容器中，最转换后的 `java` 文件（如 `index_jsp.java`）的编译最大支持 `64k` 。


    ```shell
    [root@centos7 tmp]# ls 
    hsperfdata_root  tomcat.8080.9098756374404445017  tomcat-docbase.8080.5749437145987244272
    [root@centos7 tmp]# ls tomcat.8080.9098756374404445017/work/Tomcat/localhost/ROOT/
    ```

3. `tomcat-docbase.8080.5749437145987244272` 目录是存放文件上传（`Multipart(form-data)`）的临时目录



## 自定义位置

centos7 会自动清理 10 天前未使用过的 /tmp 目录下的文件，所以若 10 天没有文件上传的操作， spring boot 的临时目录会被 centos7 清理，导致下次进行文件上传时出现错误。

所以，建议修改 spring boot 临时文件目录位置

1. 方式1

    spring boot  application.yml 配置文件中，配置临时目录的位置

    修改tomcat 工作目录临时文件位置，在 application.yml 中添加如下内容，`@artifactId@` 表示获取 pom.xml 中定义的 `artifactId` 作为目录名，运行后，会在当前 springboot 项目所在的目录下生成 artifactId 目录作为 tomcat 的工作目录。

    ```yml
    server:
      tomcat:
        basedir: @artifactId@
    ```

2. jar 包启动时配置参数，指定临时目录位置

    ```java
    java -jar -Djava.io.tmpdir=/home/app/tmp
    ```

以上两种方式都是修改的工作目录的位置，上传文件的临时目录tomcat-docbase和pid目录都还在　/tmp 目录下

网上没有找到配置上传文件临时目录的方法，所以只能在代码里配置了。

```java
@Bean
public MultipartConfigElement multipartConfigElement(){
 MultipartConfigFactory multipartConfigFactory = new MultipartConfigFactory();
 String location = System.getProperty("user.dir") + "/data/tmp";
 File file = new File(location);
 if(!file.exists()){
  file.mkdirs();
 }
 multipartConfigFactory.setLocation(location);
 return multipartConfigFactory.createMultipartConfig();
}
```

https://www.cnblogs.com/jpfss/p/12193245.html
