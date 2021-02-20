# Request和Response中文乱码


> 参考：  
> https://blog.csdn.net/lgl782519197/article/details/107271674/  
> https://www.cnblogs.com/jinzhiming/p/5765672.html

## 使用 idea 新建纯 servlet 项目

New Project - Java Enterprise - Specifications 里选择 Servlet - Next - Finish

会自动生成纯 servlet 项目，项目结构如下

pom.xml 依赖主要有 javax.servlet-api ，并且scope 是 provided ，表示打包时不将该依赖放进包里，这个依赖由 servket 容器提供。

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
</dependency>
```

src/main/webapp 目录下有 index.jsp ，该 jsp 可以浏览器直接访问，如果希望 jsp 不能被浏览器直接访问，可以将 jsp 放在 src/main/webapp/WEB-INF/jsp 目录中。 WEB-INF 目录是 web 安全目录，只能有服务端访问。

在 WEB-INF/jsp 目录下新建 GarbledTest.jsp ，用来进行 request 请求包含中文乱码测试，内容如下

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>乱码测试</title>
</head>
<body>

<form action="<%=request.getContextPath()%>/test" method="post">
    <input type="text" name="name" value="你好">
    <input type="submit" value="request post 乱码测试"/>
</form>

</body>
</html>
```

新建 GarbledTestServlet ，处理 jsp 的请求

```java
package org.msdemt.simple_014_pure_servlet_demo;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

@WebServlet("/test")
public class GarbledTestServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        String encoding = req.getCharacterEncoding();
        System.out.println("encoding: " + encoding);
        String name = req.getParameter("name");
        System.out.println("get name: " + name);
        this.doPost(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("default characterEncoding: " + req.getCharacterEncoding());
        req.setCharacterEncoding("UTF-8");  //注释该行，输出的name是乱码
        String name = req.getParameter("name");
        System.out.println("post name: " + name);

        //response 中文乱码
        resp.setContentType("text/html;charset=utf-8"); //注释该行，响应到浏览器的中文是乱码
        PrintWriter writer = resp.getWriter();
        writer.println("你好，世界");
    }
}
```

因为 GarbledTest.jsp 在 WEB-INF 目录下，无法通过浏览器直接访问该页面，所以，修改下 HelloServlet ，通过访问 index.jsp ，跳转到 GarbledTest.jsp

HelloServlet 内容如下

```java
package com.example.demo;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet(name = "helloServlet", value = "/hello-servlet")
public class HelloServlet extends HttpServlet {

    public void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.getRequestDispatcher("/WEB-INF/jsp/GarbledTest.jsp").forward(request, response);
    }
}
```

index.jsp 不用修改，默认内容如下

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<!DOCTYPE html>
<html>
<head>
    <title>JSP - Hello World</title>
</head>
<body>
<h1><%= "Hello World!" %>
</h1>
<br/>
<a href="hello-servlet">Hello Servlet</a>
</body>
</html>
```


注意： 引入 外部 js 脚本的时候，记得不要使用如下方式

```js
<script type="text/javascript" src="<%=request.getContextPath()%>/js/test.js"/>
```

在 chrome 浏览器会导致页面空白，而且 f12 开发者模式也看不到什么错误

使用如下方式引入

```js
<script type="text/javascript" src="<%=request.getContextPath()%>/js/test.js"></script>
```


## request 中文乱码

从浏览器发起请求（ request ）的方式有三种：
- 在地址栏直接输入 url 访问，这种是 get 方式，浏览器会默认将参数按照 UTF-8 进行编码
- 点击页面中的超链接访问，这种也是 get 方式， 浏览器将参数按照当前页面显示的编码进行编码（页面显示的编码可以使用 document.charset 进行查看）
- 提交表单访问，这种是 get 或 post 方式，浏览器将参数按照当前页面显示的编码进行编码

因为我们在编写页面的时候，比如 jsp 页面，经常会添加如下配置

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
```

该配置就指定了当前页面显示的编码为 UTF-8 编码。

如果不在页面中设置显示编码，则该页面会使用系统的默认编码显示，比如 win10 中文系统的默认编码是 GBK

所以，对于 request 请求中文乱码的问题，只需要在服务端设置相应的解码格式即可。

tomcat7 及更早的版本，URIEncoding 的默认值为 ISO-8859-1 ，servlet 容器会使用该编码对 url 进行解码，但是浏览器发送 url 请求时使用的 UTF-8 进行的编码，编码、解码使用的码表不一致，所以就会出现乱码的问题。tomcat7 官方文档说明如下

地址： https://tomcat.apache.org/tomcat-7.0-doc/config/http.html

```
URIEncoding	
This specifies the character encoding used to decode the URI bytes, after %xx decoding the URL. If not specified, ISO-8859-1 will be used.
```

从 tomcat8 版本开始，URIEncoding 的默认值为 UTF-8 ，这样使用 url 进行请求的编码、解码使用的码表一致，就避免了乱码的问题。
get 方式的请求，是将参数放在 url 中传输的，所以 tomcat8 开始，get 请求不会出现中文乱码问题。

tomcat8 官方文档说明如下

地址：https://tomcat.apache.org/tomcat-8.5-doc/config/http.html

```
URIEncoding	
This specifies the character encoding used to decode the URI bytes, after %xx decoding the URL. If not specified, UTF-8 will be used unless the org.apache.catalina.STRICT_SERVLET_COMPLIANCE system property is set to true in which case ISO-8859-1 will be used.
```

post 方式使用的编码是当前页面显示的编码进行编码，由上文可知，通常我们会将当前页面显示的编码置为 UTF-8 ，若没有设置页面显示的编码，则会使用系统默认的编码

但是 servlet 容器收到 post 请求时，默认使用 `ISO-8859-1` 进行解码，这种情况下，编码、解码使用的码表不一致，就会出现中文乱码的问题

解决方案是 servlet doPost() 方法内首先设置字符码表为 UTF-8 ，再进行获取参数的操作

示例：

```java
@Override
protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    System.out.println("default characterEncoding: " + req.getCharacterEncoding());
    req.setCharacterEncoding("UTF-8");
    String name = req.getParameter("name");
    System.out.println("post name: " + name);
}
```

springboot 项目不会出现 post 中文乱码的问题，因为 spring boot 项目将 characterEncoding 置为了 UTF-8

`org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration`

## Response 中文乱码

服务器输出字符数据到浏览器时，如果是中文，会出现乱码问题

示例

```java
package com.example.jspexecutablewardemo.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

@Controller
public class GarbledTestController {

    @RequestMapping("/test1")
    public void test1(HttpServletResponse response) throws IOException {

        PrintWriter printWriter = response.getWriter();
        printWriter.write("你好世界");

    }
}
```

启动项目后，浏览 `http://localhost:8080/test1` 无法正确显示中文，内容如下

```
????
```

原因：

`javax.servlet.ServletResponse#getWriter`

```java
    /**
     * Returns a <code>PrintWriter</code> object that can send character text to
     * the client. The <code>PrintWriter</code> uses the character encoding
     * returned by {@link #getCharacterEncoding}. If the response's character
     * encoding has not been specified as described in
     * <code>getCharacterEncoding</code> (i.e., the method just returns the
     * default value <code>ISO-8859-1</code>), <code>getWriter</code> updates it
     * to <code>ISO-8859-1</code>.
     * <p>
     * Calling flush() on the <code>PrintWriter</code> commits the response.
     * <p>
     * Either this method or {@link #getOutputStream} may be called to write the
     * body, not both.
     *
     * @return a <code>PrintWriter</code> object that can return character data
     *         to the client
     * @exception java.io.UnsupportedEncodingException
     *                if the character encoding returned by
     *                <code>getCharacterEncoding</code> cannot be used
     * @exception IllegalStateException
     *                if the <code>getOutputStream</code> method has already
     *                been called for this response object
     * @exception IOException
     *                if an input or output exception occurred
     * @see #getOutputStream
     * @see #setCharacterEncoding
     */
    public PrintWriter getWriter() throws IOException;
```

由上面代码可知， Response.getWriter() 方法默认输出的字符编码为 ISO-8859-1 ，可以通过 setCharacterEncoding 设置输出的字符集编码

修改代码，使用 setCharacterEncoding 设置输出编码为 UTF-8

```java
    @RequestMapping("/test1")
    public void test1(HttpServletResponse response) throws IOException {

        response.setCharacterEncoding("UTF-8");
        PrintWriter printWriter = response.getWriter();
        printWriter.write("你好世界");

    }
```

修改之后，再次查看浏览器，还是乱码

原因是浏览器使用了系统默认的编码（ GBK ）解析的页面，可以通过 document.charset 查看，如下 

![浏览器默认使用系统编码](/images/20210207-浏览器默认使用系统编码.png)  

所以可以将 setCharacterEncoding 参数配置为 GBK ，解决乱码问题

```java
    @RequestMapping("/test1")
    public void test1(HttpServletResponse response) throws IOException {

        response.setCharacterEncoding("GBK");
        PrintWriter printWriter = response.getWriter();
        printWriter.write("你好世界");

    }
```

上述方式将输出字符置为了 GBK 编码，但是如果浏览器的默认编码为 utf-8 ，还是会出现乱码

能不能告诉浏览器某个编码，让浏览器按照该编码去解析呢？

浏览器可以根据 header 中的 content-type 判断网页的解析编码，所以可以通过 response.setHeader("Content-Type", "text/html;charset=utf-8"); 告知浏览器使用 utf-8 编码解析该字符

并且 `response.setHeader("Content-Type", "text/html;charset=utf-8");` 后，也不用添加 `response.setCharacterEncoding("UTF-8");` 了，因为它已经同时把 response 的字符集改为 UTF-8 了。

代码如下

```java
    @RequestMapping("/test1")
    public void test1(HttpServletResponse response) throws IOException {
        response.setHeader("Content-Type", "text/html;charset=utf-8");
        
        PrintWriter printWriter = response.getWriter();
        printWriter.write("你好世界");

    }
```

response 提供了一个更简单的方法替代 `response.setHeader("Content-Type", "text/html;charset=utf-8");` ，即 `response.setContentType("text/html;charset=utf-8");`

```java
    @RequestMapping("/test1")
    public void test1(HttpServletResponse response) throws IOException {
        response.setContentType("text/html;charset=utf-8");

        PrintWriter printWriter = response.getWriter();
        printWriter.write("你好世界");

    }
```

所以，可以通过配置 `response.setContentType("text/html;charset=utf-8");` 解决 response 写中文时出现的乱码问题。
