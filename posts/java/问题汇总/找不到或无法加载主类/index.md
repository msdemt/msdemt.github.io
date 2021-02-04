# 找不到或无法加载主类


原因是当前执行的 class 不在 classpath 路径下，原因可能是环境变量 CLASSPATH 中没有当前路径 `.`，建议不用配置 CLASSPATH

找不到或无法加载主类：
https://blog.csdn.net/ncc1995/article/details/84932759

测试
```shell
[root@c48-1 test]# cat Test.java 
import java.net.InetAddress;
import java.net.UnknownHostException;

public class Test {


    public static void main(String[] args)throws UnknownHostException {

        System.out.println(InetAddress.getLocalHost().getHostAddress());

        
    }
}
[root@c48-1 test]# echo $CLASSPATH
/usr/local/jdk1.7.0_80/lib/dt.jar:/usr/local/jdk1.7.0_80/lib/tools.jar
[root@c48-1 test]# javac Test.java 
[root@c48-1 test]# java Test
错误: 找不到或无法加载主类 Test
[root@c48-1 test]# vi /etc/profile
[root@c48-1 test]# source /etc/profile
[root@c48-1 test]# echo $CLASSPATH
.:/usr/local/jdk1.7.0_80/lib/dt.jar:/usr/local/jdk1.7.0_80/lib/tools.jar
[root@c48-1 test]# java Test
192.168.15.161
[root@c48-1 test]# vi /etc/profile
[root@c48-1 test]# source /etc/profile
[root@c48-1 test]# echo $CLASSPATH

[root@c48-1 test]# java Test
192.168.15.161
```
