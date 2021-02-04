# Idea配置自动注释



**idea注释开头怎么才能不显示在行首**

如下图，取消红框中的勾

![idea注释开头不显示在行首](/images/20201127-idea注释开头不显示在行首.png)  

**idea自动去除未引用的import**

![idea自动去除未引用的import](/images/20201127-idea自动去除未引用的import.png)  

**idea配置注释**

![idea配置方法注释1](/images/20201127-idea配置方法注释1.png)  

![idea配置方法注释2](/images/20201127-idea配置方法注释2.png)  

```text
**
 * $description$
 $param$
 $return$
 * @author $author$
 * @date $date$
 */
```

para groovy 脚本内容

```groovy
groovyScript("if(\"${_1}\".length() == 2) {return '';} else {def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList();for(i = 0; i < params.size(); i++) {if(i==0){result+='\\n * @param ' + params[i] + ': '}else{result+='\\n' + ' * @param ' + params[i] + ': '}}; return result;}", methodParameters());
```

return groovy脚本内容

```groovy
groovyScript("def returnType = \"${_1}\"; if(returnType == 'void') {return '';} else { def result = '\\n * @return: ' + returnType; return result;}", methodReturnType()); 
```







**idea自动添加class注释**

![idea自动添加class注释](/images/20201127-idea自动添加class注释.png)  

![idea自动添加interface注释](/images/20201127-idea自动添加interface注释.png)  

```text
/**
 * @description: ${DESCRIPTION}
 * @author: ${USER}
 * @date: ${DATE}
 */
```

**idea代码自动格式化**

https://www.cnblogs.com/xiaojf/p/13098663.html


**idea鼠标悬浮显示提示**

![idea鼠标悬浮显示提示](/images/20201127-idea鼠标悬浮显示提示.png)  

