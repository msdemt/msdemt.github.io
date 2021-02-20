# 使用jackson实现json、对象、map之间的转换


测试类

```java
package org.msdemt.simple;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.Data;
import lombok.experimental.Accessors;

import java.util.*;

public class JacksonTest {

    @Accessors(chain = true)
    @Data
    static
    class Link {
        private String phone;
        private String address;
        private String email;
    }

    @Accessors(chain = true)
    @Data
    static class User {
        private String name;
        private Integer age;
        private Double pay;
        private boolean valid;
        private char one;
        private Date birtiday;
        private Link link;

        private Map map;
        private List list;
        private Set set;
    }

    /**
     * https://www.iteye.com/blog/rsy-2303323
     * https://www.cnblogs.com/pejsidney/p/9235343.html
     * @param args
     * @throws JsonProcessingException
     */
    public static void main(String[] args) throws JsonProcessingException {
        User user = new User()
                .setName("张三")
                .setAge(30)
                .setPay(8888.88)
                .setValid(true)
                .setOne('A')
                .setBirtiday(new Date())
                .setLink(new Link()
                        .setPhone("123456")
                        .setAddress("bj")
                        .setEmail("123@g.com"))
                .setMap(new HashMap<String, String>() {
                    {
                        put("map1", "111");
                        put("map2", "222");
                    }
                })
                .setList(new ArrayList() {
                    {
                        add("list1");
                        add("list2");
                    }
                })
                .setSet(new HashSet() {
                    {
                        add("set1");
                        add("set2");
                    }
                })
                ;

        ObjectMapper objectMapper = new ObjectMapper(); //转换器

        //对象转json
        String json = objectMapper.writeValueAsString(user);
        System.out.println(json);

        //json转map
        Map map = objectMapper.readValue(json, Map.class);
        map.forEach((k,v) -> System.out.printf("key: %s,\tvalue: %s\n", k, v));

        //map转json
        String str = objectMapper.writeValueAsString(map);
        System.out.println(str);

        //json转对象
        User user1 = objectMapper.readValue(json, User.class);
        System.out.println(user1);

        //对象转二进制
        byte[] bytes = objectMapper.writeValueAsBytes(user);

    }

}
```


1. `@Accessors` 注解需要指定 `chain = true` 。若指定 `fluent = true` ，则会出现如下错误

    ```log
    com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class org.msdemt.simple.Person and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)
    ```

    将某个类转为 json ，要求该类的属性必须为 public 或者有 public 的 get() 方法。指定 `fluent = true` 后，则不会生成 public 的 get() 方法，所以会出现异常。


2. 集合定义时进行初始化
    ```java
    new HashMap<String, String>() {
        {
            put("map1", "111");
            put("map2", "222");
        }
    }
    ```

    双括号“`{{}}`”，用来初始化，使代码简洁易读。

    第一层括弧实际是定义了一个匿名内部类 (Anonymous Inner Class)，第二层括弧实际上是一个实例初始化块 (instance initializer block)，这个块在内部匿名类构造时被执行。

    效率对比，使用 jdk8u202测试

    ```java
    package org.msdemt.simple;

    import java.util.HashMap;

    public class Test {
        public static void main(String[] args) {
            long st = System.currentTimeMillis();
            for (int i = 0; i < 10000000; i++) {
                HashMap<String, String> map = new HashMap<String, String>() {
                    {
                        put("name", "test");
                        put("age", "20");
                    }
                };
            }
            System.out.println(System.currentTimeMillis() - st); // 862


            //for (int i = 0; i < 10000000; i++) {
            //    HashMap<String, String> map = new HashMap<String, String>();
            //    map.put("name", "test");
            //    map.put("age", "20");
            //}
            //System.out.println(System.currentTimeMillis() - st); // 1030
        }
    }
    ```
