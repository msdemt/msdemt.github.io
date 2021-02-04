# Edge浏览器的JSONView插件无法显示json



edge浏览器安装 JSON View插件后，有些json格式的字符串无法显示为json的易读形式

![图 2](/images/20210129-json无法显示为易读格式.png)  


原因是 controller 返回类型是 String，所以http响应里的格式是text/html格式

```java
    @RequestMapping(value = "/get/{id}", method = RequestMethod.GET)
    @ResponseBody
    public String get(@PathVariable("id") final int id) throws JsonProcessingException {
        User User = null;
        try{
            User = redisUtil.getObject(key(id));
        }catch(Exception e){
            log.error("get from redis error, ", e);
        }

        return null==User ? ("can not get User by id [" + id + "]") : new ObjectMapper().writeValueAsString(User);
    }
```

解决： controller返回类型改为对象（User），spring会自动将对象转为json，或者指定响应类型

解决方案一
```java
    @RequestMapping(value = "/get/{id}", method = RequestMethod.GET)
    @ResponseBody()
    public User get(@PathVariable("id") final int id) throws JsonProcessingException {
        User user = null;
        try{
            user = redisUtil.getObject(key(id));
        }catch(Exception e){
            log.error("get from redis error, ", e);
        }

        //return null== user ? ("can not get User by id [" + id + "]") : new ObjectMapper().writeValueAsString(user);
        return user;
    }
```

解决方案二
```java
    @RequestMapping(value = "/get/{id}", method = RequestMethod.GET, produces = {"application/json"})
    @ResponseBody()
    public String get(@PathVariable("id") final int id) throws JsonProcessingException {
        User user = null;
        try{
            user = redisUtil.getObject(key(id));
        }catch(Exception e){
            log.error("get from redis error, ", e);
        }

        return null== user ? ("can not get User by id [" + id + "]") : new ObjectMapper().writeValueAsString(user);
    }
```
