# Idea配置自动生成serialVersionUID


> 参考：  
> https://jingyan.baidu.com/article/656db918c36534e381249c83.html

Java的序列化机制是通过判断serialVersionUID来验证版本的一致性，在反序列化的时候与本地类的serialVersionUID进行比较，若一致，则可以进行反序列化，若不一致则会抛出InvalidCastException。

使用idea时，可以配置在类实现java.io.Serializable接口时，自动生成serialVersionUID，步骤如下

File-->Settings-->Editor-->Inspections-->在搜索框输入serialVersionUID，然后勾选 `Serialzable class without serialVersionUID`  -->Apply

然后，新建类，实现 Serialiable接口，然后光标放在该类名上，点击Alt+Insert，就可以实现自动生成SerialVersionUID了。


```java
public class User implements Serializable {
    private static final long serialVersionUID = 6151537955659372302L;

    private Integer id;
    private String name;
    private Integer age;
    private String address;
}
```

查看Serializable接口文档

```
If a serializable class does not explicitly declare a serialVersionUID, then the serialization runtime will calculate a default serialVersionUID value for that class based on various aspects of the class, as described in the Java(TM) Object Serialization Specification. However, it is strongly recommended that all serializable classes explicitly declare serialVersionUID values, since the default serialVersionUID computation is highly sensitive to class details that may vary depending on compiler implementations, and can thus result in unexpected InvalidClassExceptions during deserialization. Therefore, to guarantee a consistent serialVersionUID value across different java compiler implementations, a serializable class must declare an explicit serialVersionUID value. It is also strongly advised that explicit serialVersionUID declarations use the private modifier where possible, since such declarations apply only to the immediately declaring class--serialVersionUID fields are not useful as inherited members. Array classes cannot declare an explicit serialVersionUID, so they always have the default computed value, but the requirement for matching serialVersionUID values is waived for array classes.
```

上面说如果一个类实现了Serialiable接口，没有配置实现serialVersionUID，则序列化运行时会自动生成一个serialVersionUID，但是，不推荐使用这种方式，因为序列化运行时自动生成的serialVersionUID每次都是不同的，不能保证反序列化的时候和本地类的serialVersionUID一致。



如果一个类不实现Serializable接口，但是在该类中，定义serialVersionUID，此时有作用吗？

不行，这种情况如果进行序列化时，会抛出 `java.io.NotSerializableException` 异常，对序列化文件进行反序列化时会抛出 `java.io.WriteAbortedException: writing aborted; java.io.NotSerializableException` 异常。


测试类：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private static final long serialVersionUID = 6151537955659372302L;

    private Integer id;
    private String name;
    private Integer age;
    private String address;
}
```

```java

@SpringBootTest
public class SerializationTest {

    /**
     * 使用jdk7 try-with-resources 新特性
     * java对象序列化测试
     */
    @Test
    public void testSerializationForJDK7() {
        User user = new User(null, "jack", 23, "Shanghai");

        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"))) {
            oos.writeObject(user);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

        /**
     * 使用jdk7 try-with-resources 新特性
     * java对象反序列化测试
     */
    @Test
    public void testDeserializationForJDK7() {
        File file = new File("tempFile");

        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file))) {
            User user = (User) ois.readObject();
            System.out.println(user);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
    }
}
```

> 序列化相关文章  
> https://www.cnblogs.com/junzi2099/p/9014814.html  
> https://blog.csdn.net/qq_37552636/article/details/110651198

