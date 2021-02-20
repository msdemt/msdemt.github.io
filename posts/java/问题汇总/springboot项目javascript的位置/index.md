# SpringBoot项目JavaScript的位置


> 参考：   
> https://www.cnblogs.com/AinRain/p/3793602.html
> https://www.zhihu.com/question/23196829
> 

## 介绍

java script 代码该放在页面的什么地方呢？

js 代码需要放在 head 或 body 里。

区别
- 在 HTML head 部分的 JavaScript 会在被调用的时候才执行
- 在 HTML body 部分的 JavaScript 会在页面加载的时候被执行

如果 js 中需要操作 dom 元素，那么 js 需要放在 body 其他内容的最后面，因为 jsp 是从上向下加载的，若在 dom 元素加载之前操作该 dom ，会出现错误。

## js 放在 head 部分

js 放在 head ，当 js 被调用或事件触发才会执行，不能在 js 中直接操作 dom 元素，因为 head 里的 js 是在 body 里的 dom 元素之前加载，若在 head 的js 里操作 dom 元素，会出现错误。

示例如下

**示例1 js 代码直接放在 head 里**

jsp 页面内容如下

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Test</title>

    <script>
        function alertAjaxTest (method, url) {
            // 1. 创建 XMLHttpRequest 对象
            let xmlHttp;
            if (window.XMLHttpRequest) {
                xmlHttp = new XMLHttpRequest();
            } else {
                xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
            }
            console.log(xmlHttp);
            // 2. 发送 ajax 请求
            xmlHttp.open(method, url, true);
            xmlHttp.send();
            // 3. 处理服务器响应
            xmlHttp.onreadystatechange = function () {
                if (xmlHttp.readyState == 4 && xmlHttp.status == 200) {
                    let t = xmlHttp.responseText;
                    alert(t);
                }
            }
        }
    </script>
</head>
<body>

<input id="testButton" type="button" value="hello" onclick="alertAjaxTest('GET', '/ajax/content')"/>

</body>
</html>
```

**示例2 通过外部文件引用 js 代码**

jsp 文件内容如下

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Test</title>

    <script type="text/javascript" src="<%=request.getContextPath()%>/js/test.js"></script>
</head>
<body>

<input id="testButton" type="button" value="hello" onclick="alertAjaxTest('GET', '/ajax/content')"/>

</body>
</html>
```

js 文件放在 src/main/resources/static/js 文件中

因为 spring boot 会将 `/META-INF/resources` 、 `/public` 、 `/static` 、 `/public` 映射为静态资源目录，这些目录下的资源是可以直接访问的，所以在 jsp 文件中通过 `/js/test.js` 访问到 `test.js` 文件。

test.js 文件内容如下

```js
function alertAjaxTest (method, url) {
    // 1. 创建 XMLHttpRequest 对象
    let xmlHttp;
    if (window.XMLHttpRequest) {
        xmlHttp = new XMLHttpRequest();
    } else {
        xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
    }
    console.log(xmlHttp);
    // 2. 发送 ajax 请求
    xmlHttp.open(method, url, true);
    xmlHttp.send();
    // 3. 处理服务器响应
    xmlHttp.onreadystatechange = function () {
        if (xmlHttp.readyState == 4 && xmlHttp.status == 200) {
            let t = xmlHttp.responseText;
            alert(t);
        }
    }
}
```

在 chrome 浏览器中直接访问 test.js 会发现中文乱码了，在 chrome 中按 F12 打开浏览器开发工具，在控制台中输入 document.charset ，可以发现当前页面是 GBK 编码（当文件中未指定编码，浏览器会使用系统默认编码打开文件），但是我们的 test.jsp 使用的是 UTF-8 编码，所以出现了中文乱码。

解决这个乱码问题可以将 test.js 改为 GBK 编码或者将系统默认编码改为 UTF-8 ，但是感觉这种方法适用性不高，不能从根本解决问题。

没有在网上找到如何在 js 文件中指定编码的方法，好像 js 文件就不能在文件内部指定编码。最后乱码就乱码吧，以后尽量少在 js 文件里加中文描述，并且线上环境也不推荐使用带有注释的 js 。

## js 放在 body 部分

注意：有两种情况，一种是在 js 中直接操作 dom 元素，一种是不直接操作 dom 元素。

如果在 js 中直接操作 dom 元素，则 js 需要放在 body 其他内容的后面，示例如下

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Test</title>
</head>
<body>

<input id="testButton" type="button" value="hello"/>


<script>
    document.getElementById("testButton").onclick = function () {
        // 1. 创建 XMLHttpRequest 对象
        let xmlHttp;
        if (window.XMLHttpRequest) {
            xmlHttp = new XMLHttpRequest();
        } else {
            xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
        }
        console.log(xmlHttp);
        // 2. 发送 ajax 请求
        xmlHttp.open("GET", "/ajax/content", true);
        xmlHttp.send();
        // 3. 处理服务器响应
        xmlHttp.onreadystatechange = function () {
            if (xmlHttp.readyState == 4 && xmlHttp.status == 200) {
                let t = xmlHttp.responseText;
                alert(t);
            }
        }
    }
</script>
</body>
</html>
```

js 中不直接操作 dom 元素，此时放在 body 其他元素前面或后面都可以，示例如下

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Test</title>
</head>
<body>

<script>
    function alertAjaxTest (method, url) {
        // 1. 创建 XMLHttpRequest 对象
        let xmlHttp;
        if (window.XMLHttpRequest) {
            xmlHttp = new XMLHttpRequest();
        } else {
            xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
        }
        console.log(xmlHttp);
        // 2. 发送 ajax 请求
        xmlHttp.open(method, url, true);
        xmlHttp.send();
        // 3. 处理服务器响应
        xmlHttp.onreadystatechange = function () {
            if (xmlHttp.readyState == 4 && xmlHttp.status == 200) {
                let t = xmlHttp.responseText;
                alert(t);
            }
        }
    }
</script>

<input id="testButton" type="button" value="hello" onclick="alertAjaxTest('GET', '/ajax/content')"/>

</body>
</html>
```



