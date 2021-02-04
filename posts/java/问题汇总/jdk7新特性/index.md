# Jdk7新特性



1. 新增 java.lang.AutoCloseable 接口

2. 二进制字面量

    jdk7开始，可以使用二进制来表示整数（byte, short, int 和 long）

    只要在二进制数值前面加上 `0b` 或 `0B` 即可。

    ```java
    /**
     * jdk7新特性：二进制字面量表示整数
     */
    @Test
    public void testBinaryLiteral(){
        int bin = 0b010101;  //二进制，jdk7开始，以0b或0B开头表示
        System.out.println(bin); //输出该二进制值对应的十进制值：21
        int oct = 0177; //八进制，以0开头表示
        System.out.println(oct); //输出该八进制值对应的十进制值：127
        int dec = 123; //十进制
        System.out.println(dec); //输出十进制值：123
        int hex = 0XFD10; //十六进制，以0x或0X开头表示
        System.out.println(hex); //输出该16进制值对应的十进制值：64784
    }
    ```

3. 数字字面量可以出现下划线

    为了增强对数值的阅读性，JDK7和以后的版本可以使用 `_` 对数值分隔。

    注意：
    - 不能出现在进制标识和数值之间
    - 不能出现在数值开头和结尾
    - 不能出现在小数点旁边

    ```java
    @Test
    public void testNumericLiteral(){
        byte a = 127;
        short b = 10_000;
        int c = 100_000;
        float d = 12.34_56F;
        double e = 12.34_56D;

        //float f = 12.3456_F; //error
        //float f = _12.3456F; //error
        //float f = 12.3456F_; //error
        //float f = 12._3456F; //error
        //int j = 0_x52; //error
        //int j = 0x_52; //error
    }
    ```

4. switch 语句可以使用字符串


    switch 语句能否作用在byte、long、String上？

    JDK7之前，在switch(expr1)中，expr1只能是一个整数表达式或者枚举常量，整数表达式可以是int基本类型或Integer包装类型，由于byte、short、char都可以隐式转换为int，所以这些类型及对应的包装类可以作为switch的参数。显然，由于long和String类型都不符合switch的语法规定，并且不能隐式转为int类型，所以，他们不能用于switch语句中。

    JDK7开始，已经支持String类型作为switch的参数

    ```java
    @Test
    public void testSwitch(){
        String season = "春天";
        switch (season) {
            case "春天":
                System.out.println("春天来了");
                break;
            case "夏天":
                System.out.println("夏天来了");
                break;
            case "秋天":
                System.out.println("秋天来了");
                break;
            case "冬天":
                System.out.println("冬天来了");
                break;
        }
    }
    ```

5. 泛型简化（泛型实例化类型推断）

    泛型实例的创建可以通过类型推断来简化，可以去掉后面new部分的泛型类型，只用<>就可以了。

    ```java
    @Test
    public void testGeberic(){
        //jdk7之前
        List<String> listPreJDK7 = new ArrayList<String>();
        //jdk7及以后
        List<String> listForJDK7 = new ArrayList<>();
    }
    ```

6. catch语句可以捕获多个异常

    JDK7之前，一个catch语句只能捕获一个异常，JDK7之后，一个catch语句可以捕获多个异常

    ```java
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

7. try-with-resource语句

    jdk7之前，使用try-catch-finally语句进行异常捕获和资源释放。

    jdk7开始，支持try-with-resource语句，try-with-resources语句是一种声明了一种或多种资源的try语句。资源是指在程序用完了之后必须要关闭的对象。try-with-resources语句保证了每个声明了的资源在语句结束的时候都会被关闭。任何实现了java.lang.AutoCloseable接口的对象实现了java .io .Closeable接口的对象，都可以当做资源使用。

    jdk7之前使用try-catch-finally语句代码示例
    
    ```java
    /**
     * java对象序列化测试
     */
    @Test
    public void testSerialization() {
        User user = new User(null, "jack", 23, "Shanghai");

        ObjectOutputStream oos = null;

        try {
            oos = new ObjectOutputStream(new FileOutputStream("tempFile"));
            oos.writeObject(user);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (oos != null) {
                    oos.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```

    jdk7使用try-with-resource语句代码示例

    ```java
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
    ```

> 参考：  
> https://blog.csdn.net/csdnlijingran/article/details/88855000  
> https://www.cnblogs.com/excellencesy/p/8613743.html
