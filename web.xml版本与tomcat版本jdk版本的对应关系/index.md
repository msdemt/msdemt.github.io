# Web.xml版本与tomcat版本、jdk版本的对应关系


web.xml 示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
            http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
           version="3.0">

</web-app>
```

web.xml文件中的版本号（version="3.0"）指的是servlet的版本

servlet版本与tomcat版本、jdk版本的对应关系可以查看tomcat官网的 whichversion 页面

http://tomcat.apache.org/whichversion.html

![tomcat-whichversion](../../../images/20210125-tomcat-whichversion.png)  

