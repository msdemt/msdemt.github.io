# Java中的变量



1. 实例变量：类中声明的，且在类的其他成员方法之外的非静态的变量；有默认值

    示例

    ```java
    public class Person {
        
        private String name; //实例变量
        private int age; //实例变量
        private String address; //实例变量

        @Override
        public String toString() {
            return "Person{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    ", address='" + address + '\'' +
                    '}';
        }
    }
    ```

2. 类变量：类中声明的，且在类的其他成员方法之外的静态的变量；有默认值

    示例：

    ```java
    public class Person {

        private String name; //实例变量
        private int age; //实例变量
        private String address; //实例变量
        private static String country; //类变量

        @Override
        public String toString() {
            return "Person{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    ", address='" + address + '\'' +
                    '}';
        }
    }
    ```

3. 常量：在程序运行中保持不变的量

    示例：

    ```java
    public class HelloWorld {
        // 静态成员常量
        public static final double PI = 3.14;
        // 声明成员常量
        final int y = 10;

        public static void main(String[] args) {
            // 声明局部常量
            final double x = 3.3;
        }
    }
    ```

4. 全局变量，又叫成员变量，包括实例变量、类变量、成员常量

5. 局部变量，是指在方法内定义的变量，没有初始值，需要手工赋值后使用。
