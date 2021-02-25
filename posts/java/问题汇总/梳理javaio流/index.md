# 梳理Java IO流


> 参考：  
> https://blog.csdn.net/qq_35598736/article/details/108750468


## File

File 类主要是对文件和目录的抽象表示，提供了对文件的创建、删除、查找等操作。主要有以下特点
- Java 的世界万物皆对象，文件和目录就可抽象为File对象
- 对于File而言，封装的并不是真正的文件，封装的仅仅是一个路径名，磁盘文件本身可以存在，也可以不存在
- 文件的内容不能用File读取，而是通过流来读取，File对象可以作为流的来源地和目的地

### File 类的常用构造方法

构造方法 | 方法说明
-- | --
File(String pathname) | 将路径字符串抽象为File实例，路径字符串可以是相对路径，也可以为绝对路径
File(String parent, String child) | 从父路径名和子路径名来构建File实例
File(File parent, String child) | 根据父File实例和子路径名来构建File实例

如下示例，表示这几种文件和目录的代码如下

![File-1](/images/20210224-File-1.png)  

```java
// pathname
File liuBei = new File("D:/三国/刘备.jpg");
// String parent, String child
File guanYu = new File("D:/三国", "关羽.jpg");
// 目录
File sanGuo = new File("D:/三国");
// File parent, String child
File zhangFei = new File(sanGuo, "张飞.txt");
// 可以声明不存在的文件
File zhuGeLiang = new File(sanGuo, "诸葛亮.txt");
```

### 绝对路径和相对路径

绝对路径：从盘符开始的路径，表示一个完整的路径。(经常使用)  
相对路径：相对于当前项目目录的路径

```java
File f = new File("D:/bbb.java");
// D:\bbb.java
System.out.println(f.getAbsolutePath());
File f2 = new File("bbb.java");
// F:\code\ad\bbb.java
System.out.println(f2.getAbsolutePath());
```

### 路径分隔符和换行符

路径分隔符
- windows 的路径分隔符： `\`
- linux 的路径分隔符： `/`

java 有常量 `java.io.File#separator` 可以表示不同系统的路径分隔符

```java
/**
 * The system-dependent default name-separator character, represented as a
 * string for convenience.  This string contains a single character, namely
 * <code>{@link #separatorChar}</code>.
 */
public static final String separator = "" + separatorChar;
```

换行符
- windows 的换行符： `\r\n`
- linux 的换行符： `\n`

java 提供了方法 `java.lang.System#lineSeparator` 可以获取不同平台的换行符

```java
/**
 * Returns the system-dependent line separator string.  It always
 * returns the same value - the initial value of the {@linkplain
 * #getProperty(String) system property} {@code line.separator}.
 *
 * <p>On UNIX systems, it returns {@code "\n"}; on Microsoft
 * Windows systems it returns {@code "\r\n"}.
 *
 * @return the system-dependent line separator string
 * @since 1.7
 */
public static String lineSeparator() {
    return lineSeparator;
}
```

### File 的常用方法

#### 创建、删除

方法名 | 方法说明
-- | --
boolean createNewFile() throws IOException | 当该名称的文件不存在时，创建一个由该抽象路径名的空文件并返回true，当文件存在时，返回false
boolean mkdir() | 创建由此抽象路径名命名的目录，不能级联创建目录
boolean mkdirs() | 创建由此抽象路径名命名的目录，包括任何必需但不存在的父目录。级联创建目录
boolean delete() | 删除由此抽象路径名表示的文件或目录

- 创建多级目录时，mkdir() 创建失败，返回false，mkdirs() 创建成功，返回true（推荐使用mkdirs）
- delete() 删除目录时，目录内不为空时，删除失败，返回false， 即只能删除文件或者空目录

```java
File shuiHu = new File("D:/四大名著/水浒传");
// 返回false 创建失败
boolean mkdir = shuiHu.mkdir();
// 返回true 创建失败
boolean mkdirs = shuiHu.mkdirs();

File four = new File("D:/四大名著");
// 返回false 删除目录时必须目录为空才能删除成功，此时 D:/四大名著 目录下还有 水浒传 ，所以删除失败
boolean delete = four.delete();

File shuiHu = new File("D:/四大名著/水浒传");
// true 正确删除了水浒传目录
boolean delete1 = shuiHu.delete();

File liuBei = new File("D:/三国/刘备.jpg");
// 返回true 正确删除了刘备.jpg文件
boolean delete2 = liuBei.delete();
```

使用递归的方式删除文件或目录

```java
import java.io.File;

public class FileDeleteTest {

    public static boolean deleteDirOrFile(String fileName) {
        File file = new File(fileName); //根据指定文件名创建 File 对象

        if (file.exists()) {
            //待删除的存在
            if (file.isFile()) {
                //如果待删除的是文件
                return deleteFile(file);
            } else {
                //如果待删除的是目录
                return deleteDir(file);
            }
        } else {
            //待删除的不存在
            System.out.println("待删除的文件不存在");
            return true;
        }
    }

    public static boolean deleteFile(File file) {
        if (file.delete()) {
            System.out.println("文件" + file.getName() + "删除成功");
            return true;
        } else {
            System.out.println("文件" + file.getName() + "删除失败");
            return false;
        }
    }

    private static boolean deleteDir(File file) {
        File[] files = file.listFiles(); //列出该文件夹下的所有文件及文件夹
        for (int i = 0; i < files.length; i++) {
            //使用递归的方式删除该文件夹下的所有文件和文件夹
            deleteDirOrFile(files[i].getAbsolutePath());
        }
        if (file.delete()) { //删除该文件夹
            System.out.println("目录" + file.getName() + "删除成功");
            return true;
        }
        System.out.println("目录" + file.getName() + "删除失败");
        return false;
    }

    public static void main(String[] args) {
        deleteDirOrFile("D:\\1\\142");
    }
}
```

#### 文件检测

方法名 | 方法说明
-- | --
boolean isDirectory() | 判断是否是目录
boolean isFile() | 判断是否是文件
boolean exists() | 判断文件或目录是否存在 
boolean canWrite() | 文件是否可写
boolean canRead() | 文件是否可读 
boolean canExecute() | 文件是否可执行
long lastModified() | 返回文件的上次修改时间

注意
- 文件或目录不存在时， isDirectory() 或 isFile() 返回 false
- 可读、可写、可执行是对操作系统给文件赋予的权限

```java
File xiYou = new File("D:/西游记");
// 文件或目录不存在时 返回false
System.out.println(xiYou.isDirectory());
```

#### 文件获取

方法名 | 方法说明
-- | --
String getAbsolutePath() | 返回File对象的绝对路径字符串
String getPath() | 将此抽象路径名转换为路径名字符串
String getName() | 返回文件或目录的名称
long length() | 返回由此File表示的文件的字节数
String[] list() | 返回目录中的文件和目录的名称字符串数组
File[] listFiles() | 返回目录中的文件和目录的File对象数组
String[] list(FilenameFilter filter) | 返回目录中指定的文件和目录的名称字符串数组
File[] listFiles(FileFilter filter) | 返回目录中指定的文件和目录的File对象数组

注意
- length() 返回的是文件的字节数，目录的长度是0
- getPath()在用绝对路径表示的文件时相同，用相对路径表示的文件时不同
- listFiles和list方法的调用，必须是实际存在的目录，否则返回null
- listFiles和list 可以传入FilenameFilter的实现类，用于按照文件名称过滤文件

```java
File shuiHu = new File("D:/水浒传");
// 0
System.out.println(shuiHu.length());
File liuBei = new File("D:/三国/刘备.jpg");
// 24591
System.out.println(liuBei.length());

File f = new File("D:/bbb.java");
 // D:\bbb.java
System.out.println(f.getPath());

File f2 = new File("bbb.java");
// bbb.java
System.out.println(f2.getPath());

File sanGuo2 = new File("D:/三国2");
// 该目录不存在，返回null
String[] list = sanGuo2.list();
```

过滤文件的接口

```java
@FunctionalInterface
public interface FilenameFilter {
  	// 参数为目录和指定过滤名称
    boolean accept(File dir, String name);
}
```

**java 查询目录下的所有文件及目录**

> 参考： https://www.cnblogs.com/henuyuxiang/p/11608997.html

```java
public static void main(String[] args) {
    List<String> filesList = new ArrayList<>();
    findFileList("D:\\1\\jsp-demo-0.0.1-SNAPSHOT1", filesList);
    for (String str : filesList) {
        System.out.println(str);
    }
    
}


public static void findFileList(String dir, List<String> fileList) {
    File file = new File(dir);
    if (!file.exists() || !file.isDirectory()) {
        //若文件不存在或者文件不为目录，则返回
        return;
    }

    File[] files = file.listFiles(); //获取目录下所有目录和文件
    for (int i = 0; i < files.length; i++) {
        if (files[i].isFile()) {
            //将文件添加进list
            fileList.add(files[i].getAbsolutePath());
        } else {
            //将文件夹添加进list
            fileList.add(files[i].getAbsolutePath());
            //递归文件夹
            findFileList(files[i].getAbsolutePath(), fileList);
        }
    }
}
```

## IO 流

IO 流主要是读取、传输、写入数据内容的。

这里的主体说的都是程序(即内存)，从外部设备中读取数据到程序中 即为输入流，从程序中写出到外部程序中即为输出流

![输入流与输出流](/images/20210224-输入流与输出流.png)  

### IO 的分类

- 本地 IO 和网络 IO
  - 本地IO主要是操作本地文件，例如在windows上复制粘贴操作文件，都可以使用java的io来操作
  - 网络IO主要是通过网络发送数据，或者通过网络上传、下载文件
- 按流向分：输入流和输出流
- 按数据类型分：字节流和字符流
- 按功能分：节点流和处理流
  - 程序直接操作目标设备的类称为节点流
  - 对节点流进行装饰，功能、性能进行增强，称为处理流

IO 流的主要入口是数据源，下面列举常见的源设备和目的设备

源设备
- 硬盘(文件)
- 内存(字节数组、字符串等)
- 网络(Socket)
- 键盘(System.in)

目的设备
- 硬盘(文件)
- 内存(字节数组、字符串等)
- 网络(Socket)
- 控制台(System.out)

### 字节流和字符流

字节流和字符流都实现了 Closeable 接口，实现了 close() 方法

字节流定义如下

```java
public abstract class InputStream implements Closeable
```
```java
public abstract class OutputStream implements Closeable, Flushable
```

字符流定义如下
```java
public abstract class Reader implements Readable, Closeable
```
```java
public abstract class Writer implements Appendable, Closeable, Flushable
```

Closeable 接口定义如下

```java
/**
 * A {@code Closeable} is a source or destination of data that can be closed.
 * The close method is invoked to release resources that the object is
 * holding (such as open files).
 *
 * @since 1.5
 */
public interface Closeable extends AutoCloseable {

    /**
     * Closes this stream and releases any system resources associated
     * with it. If the stream is already closed then invoking this
     * method has no effect.
     *
     * <p> As noted in {@link AutoCloseable#close()}, cases where the
     * close may fail require careful attention. It is strongly advised
     * to relinquish the underlying resources and to internally
     * <em>mark</em> the {@code Closeable} as closed, prior to throwing
     * the {@code IOException}.
     *
     * @throws IOException if an I/O error occurs
     */
    public void close() throws IOException;
}
```


Flushable 接口定义如下

```java
/**
 * A <tt>Flushable</tt> is a destination of data that can be flushed.  The
 * flush method is invoked to write any buffered output to the underlying
 * stream.
 *
 * @since 1.5
 */
public interface Flushable {

    /**
     * Flushes this stream by writing any buffered output to the underlying
     * stream.
     *
     * @throws IOException If an I/O error occurs
     */
    void flush() throws IOException;
}
```

方法名 | 方法说明
-- | --
public void close() throws IOException | 流操作完毕后，必须释放系统资源，调用close方法，一般放在finally块中保证一定被执行!

注意
- 程序中打开的IO资源不属于内存资源，垃圾回收机制无法回收该资源，需要显式的关闭文件资源
- IO流可以在finally语句块中关闭，也可以使用jdk7的try-with-resource特性简化IO流的关闭
  - try-with-resources语句保证了每个声明了的资源在语句结束的时候都会被关闭。任何实现了java.lang.AutoCloseable接口的对象和实现了java .io .Closeable接口的对象，都可以当做资源使用。
  - 示例
     ```java
    try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"))) {
                oos.writeObject(user);
            } catch (IOException e) {
                e.printStackTrace();
            }
     ```

#### 字节流

一切皆为字节

一切文件数据(文本、图片、视频等)在存储时，都是以二进制的形式保存，都可以通过使用字节流传输。

InputStream 是字节输入流的顶层抽象

```java
public abstract class InputStream implements Closeable
```

核心方法如下
- `int read() throws IOException;`: 每次读取一个字节的数据，提升为int类型，读取到文件末尾时返回 -1
- `int read(byte b[])throws IOException`: 每次读取到字节数组中，返回读取到的有效字节个数，读取到末尾时返回 -1(常用)
- `int read(byte b[], int off, int len)`: 每次读取到字节数组中，从偏移量off开始，长度为len，返回读取到的有效字节个数，读取到末尾时返回 -1

OutputStream 是字节输出流的顶层抽象

```java
public abstract class OutputStream implements Closeable, Flushable
```

核心方法如下
- `void write(int b) throws IOException;`: 将int值写入到输出流中
- `void write(byte[] b) throws IOException;`: 将字节数组写入到输出流中
- `void write(byte b[], int off, int len) throws IOException`: 将字节数组从偏移量off开始，写入len个长度到输出流中
- `void flush() throws IOException`: 刷新输出流并强制缓冲的字节被写出


##### 文件节点流

字节流 `java.io.InputStream` 和 `java.io.OutputStream` 有很多的实现类，先介绍下文件节点流，即目标设备是文件，输入流和输出流对应的是 `java.io.FileInputStream` 和 `java.io.FileOutputStream`

FileInputStream 主要从磁盘文件中读取数据，常用构造方法如下
- public FileInputStream(File file) throws FileNotFoundException
- public FileInputStream(String name) throws FileNotFoundException

当传入的文件不存在时，运行时会抛出 FileNotFoundException 异常

主要方法介绍

1. read() 从输入流中读取一个字节并返回，当输入流没准备好时该方法会阻塞。如果读到文件末尾返回 -1 ，只能读取英文字符，若读取中文会出现乱码，因为一个英文字符占一个字节，中文字符占多个字节
   
   ```java
    /**
     * Reads a byte of data from this input stream. This method blocks
     * if no input is yet available.
     *
     * @return     the next byte of data, or <code>-1</code> if the end of the
     *             file is reached.
     * @exception  IOException  if an I/O error occurs.
     */
    public int read() throws IOException {
        return read0();
    }
   ```

   示例

   ```java
    public static void main(String[] args) {
        File file = new File("D:/三国/诸葛亮.txt");
        try(FileInputStream fis = new FileInputStream(file)){

            int b;
            while ((b = fis.read()) != -1){
                System.out.println((char)b);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
   ```

2. read(byte[]) 

    ```java
    /**
     * Reads up to <code>byte.length</code> bytes of data from this
     * input stream into an array of bytes. This method blocks until some
     * input is available.
     * <p>
     * This method simply performs the call
     * <code>read(b, 0, b.length)</code> and returns
     * the  result. It is important that it does
     * <i>not</i> do <code>in.read(b)</code> instead;
     * certain subclasses of  <code>FilterInputStream</code>
     * depend on the implementation strategy actually
     * used.
     *
     * @param      b   the buffer into which the data is read.
     * @return     the total number of bytes read into the buffer, or
     *             <code>-1</code> if there is no more data because the end of
     *             the stream has been reached.
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.FilterInputStream#read(byte[], int, int)
     */
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }
    ```

    示例

    ```java
    public static void main(String[] args) {
        File file = new File("D:/三国/诸葛亮.txt");
        try(FileInputStream fis = new FileInputStream(file)){

            byte[] bytes = new byte[2];
            while (fis.read(bytes) != -1){
                System.out.println(new String(bytes));
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    ```

    若 `D:/三国/诸葛亮.txt` 中存储的是 abcde ，那么上述代码的结果如下

    ```
    ab
    cd
    ed
    ```

    由于最后一次读取时，只读到一个字节 e ，此时数组中还有上次残留的数据 cd ，只替换了 e ，所以最后输出了 ed

    修改代码如下

    ```java
    public static void main(String[] args) {
        File file = new File("D:/三国/诸葛亮.txt");
        try(FileInputStream fis = new FileInputStream(file)){

            byte[] bytes = new byte[2];
            int len; //len为每次读取的有效字节个数
            while ((len = fis.read(bytes)) != -1){
                System.out.println(new String(bytes, 0, len));
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    ```

3. read(byte b[], int off, int len)

    ```java
    /**
     * Reads up to <code>len</code> bytes of data from this input stream
     * into an array of bytes. If <code>len</code> is not zero, the method
     * blocks until some input is available; otherwise, no
     * bytes are read and <code>0</code> is returned.
     * <p>
     * This method simply performs <code>in.read(b, off, len)</code>
     * and returns the result.
     *
     * @param      b     the buffer into which the data is read.
     * @param      off   the start offset in the destination array <code>b</code>
     * @param      len   the maximum number of bytes read.
     * @return     the total number of bytes read into the buffer, or
     *             <code>-1</code> if there is no more data because the end of
     *             the stream has been reached.
     * @exception  NullPointerException If <code>b</code> is <code>null</code>.
     * @exception  IndexOutOfBoundsException If <code>off</code> is negative,
     * <code>len</code> is negative, or <code>len</code> is greater than
     * <code>b.length - off</code>
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.FilterInputStream#in
     */
    public int read(byte b[], int off, int len) throws IOException {
        return in.read(b, off, len);
    }
    ```

    示例

    ```java
    public static void main(String[] args) {
        File file = new File("D:/三国/诸葛亮.txt");
        try(FileInputStream fis = new FileInputStream(file)){

            byte[] bytes = new byte[2]; //存储已读取数据的缓存
            int off = 0; //从缓存中读取字节放入缓存 bytes 的起始偏移量
            int len = 2; //每次读取的字节数
            int readLen; //读取到的字节数，如果读取到文件末尾返回 -1
            while ((readLen = fis.read(bytes, off, len)) != -1){
                System.out.println(new String(bytes,off,readLen));
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    ```

使用数组读取，每次读取多个字节，减少了系统间的IO操作次数，从而提高了效率
- 如果用read()读取文件，每读取一个字节就要访问一次硬盘，这种效率较低。
- 如果用read(byte[])读取文件，一次读取多个字节，当文件很大时，也会频繁访问硬盘。如果一次读取超多字节，效率也不会高。

> java中文和英文占用的字节数：https://www.zhihu.com/question/30945431



FileOutputStream 主要是向磁盘文件中写出数据，常用构造方法如下
- `FileOutputStream(File file) throws FileNotFoundException`: 使用一个File对象来构建一个FileOutputStream
- `FileInputStream(String name) throws FileNotFoundException`: 使用一个文件名来构建一个FileOutputStream
- `FileOutputStream(File file, boolean append) throws FileNotFoundException`: append传true时，会对文件进行追加
- `FileOutputStream(String name, boolean append) throws FileNotFoundException`: append传true时，会对文件进行追加

注意：
- 上述构造方法执行后，如果file不存在，会自动创建该文件
- 如果file存在，append没有传或者传了false，会清空文件的数据
- 如果file存在，append传了true，不会清空文件的数据

```java
public static void main(String[] args) {
    File file = new File("D:/三国/赵云.txt");
    try(FileOutputStream fos = new FileOutputStream(file);
    FileOutputStream fos1 = new FileOutputStream("D:/三国/司马懿.txt")){

    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

上述代码中执行完后，`赵云.txt`和`司马懿.txt`都会自动创建出来

主要方法
- public void write(byte b[]) throws IOException
- public void write(byte b[], int off, int len) throws IOException
- public void write(int b) throws IOException

向文件写数据

```java
public static void main(String[] args) {
    File file = new File("D:/三国/赵云.txt");
    try(FileOutputStream fos = new FileOutputStream(file);
    FileOutputStream fos1 = new FileOutputStream("D:/三国/司马懿.txt")){
        fos.write(97);
        fos.write(98);
        fos.write(99);

        fos1.write("三国司马懿".getBytes());
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

上述代码每执行一次，文件里的内容就会被覆盖。如果想追加文件内容，则使用 `FileOutputStream fos = new FileOutputStream(file, true)` 方式定义

利用文件节点流实现文件的复制

```java
public static void copyFile(String fromFile, String toFile){
    try(InputStream is = new FileInputStream(new File(fromFile));
    OutputStream os = new FileOutputStream(new File(toFile))) {
        byte[] bytes = new byte[1024];
        int len = 0;
        while ((len = is.read(bytes)) != -1){
            os.write(bytes, 0, len);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

##### 内存节点流

`java.io.ByteArrayInputStream` 是从内存的字节数组中读取数据，定义如下

```java
/**
 * A <code>ByteArrayInputStream</code> contains
 * an internal buffer that contains bytes that
 * may be read from the stream. An internal
 * counter keeps track of the next byte to
 * be supplied by the <code>read</code> method.
 * <p>
 * Closing a <tt>ByteArrayInputStream</tt> has no effect. The methods in
 * this class can be called after the stream has been closed without
 * generating an <tt>IOException</tt>.
 *
 * @author  Arthur van Hoff
 * @see     java.io.StringBufferInputStream
 * @since   JDK1.0
 */
public
class ByteArrayInputStream extends InputStream
```

`java.io.ByteArrayInputStream` 构造方法如下

```java
/**
 * Creates a <code>ByteArrayInputStream</code>
 * so that it  uses <code>buf</code> as its
 * buffer array.
 * The buffer array is not copied.
 * The initial value of <code>pos</code>
 * is <code>0</code> and the initial value
 * of  <code>count</code> is the length of
 * <code>buf</code>.
 *
 * @param   buf   the input buffer.
 */
public ByteArrayInputStream(byte buf[]) {
    this.buf = buf;
    this.pos = 0;
    this.count = buf.length;
}

/**
 * Creates <code>ByteArrayInputStream</code>
 * that uses <code>buf</code> as its
 * buffer array. The initial value of <code>pos</code>
 * is <code>offset</code> and the initial value
 * of <code>count</code> is the minimum of <code>offset+length</code>
 * and <code>buf.length</code>.
 * The buffer array is not copied. The buffer's mark is
 * set to the specified offset.
 *
 * @param   buf      the input buffer.
 * @param   offset   the offset in the buffer of the first byte to read.
 * @param   length   the maximum number of bytes to read from the buffer.
 */
public ByteArrayInputStream(byte buf[], int offset, int length) {
    this.buf = buf;
    this.pos = offset;
    this.count = Math.min(offset + length, buf.length);
    this.mark = offset;
}
```



`java.io.ByteArrayOutputStream` 是向内存字节数组中写数据，内部维护了一个数组

定义如下

```java

/**
 * This class implements an output stream in which the data is
 * written into a byte array. The buffer automatically grows as data
 * is written to it.
 * The data can be retrieved using <code>toByteArray()</code> and
 * <code>toString()</code>.
 * <p>
 * Closing a <tt>ByteArrayOutputStream</tt> has no effect. The methods in
 * this class can be called after the stream has been closed without
 * generating an <tt>IOException</tt>.
 *
 * @author  Arthur van Hoff
 * @since   JDK1.0
 */

public class ByteArrayOutputStream extends OutputStream
```

`java.io.ByteArrayOutputStream` 构造方法如下

```java
/**
 * Creates a new byte array output stream. The buffer capacity is
 * initially 32 bytes, though its size increases if necessary.
 */
public ByteArrayOutputStream() {
    this(32);
}

/**
 * Creates a new byte array output stream, with a buffer capacity of
 * the specified size, in bytes.
 *
 * @param   size   the initial size.
 * @exception  IllegalArgumentException if size is negative.
 */
public ByteArrayOutputStream(int size) {
    if (size < 0) {
        throw new IllegalArgumentException("Negative initial size: "
                                           + size);
    }
    buf = new byte[size];
}
```


注意：`java.io.ByteArrayInputStream` 和 `java.io.ByteArrayOutputStream` 不需要 close 数据源和抛出 IOException ，因为不涉及底层的系统调用

内存节点流代码示例

```java
public static void copyUseByteArrayStream(String str) {
    ByteArrayInputStream bais = new ByteArrayInputStream(str.getBytes());
    ByteArrayOutputStream baos = new ByteArrayOutputStream();

    int len = 0;
    while ((len = bais.read()) != -1){
        baos.write(len);
    }

    System.out.println(new String(baos.toByteArray()));
}
```
应用场景
- 内存操作流一般在一些生成临时信息时会被使用（如果将临时信息保存在文件中，代码执行完还要删除文件比较麻烦，所以使用内存操作流将信息保存在内存中）
- 结合对象流，可以实现对象和字节数组的互转

#### 字符流

字符流封装了更加适合操作文本字符的方法

`java.io.Reader` 用于读取文本字符

定义如下

```java
public abstract class Reader implements Readable, Closeable 
```

核心方法
- `int read() throws IOException`: 从输入流中读取一个字符,读到文件末尾时返回 -1
- `int read(char cbuf[]) throws IOException`: 从输入流中读取字符到char数组中

`java.io.Writer` 用于写出文本字符

定义如下

```java
public abstract class Writer implements Appendable, Closeable, Flushable
```

核心方法
- `void write(int c) throws IOException`: 写入单个字符到输出流中
- `void write(char[] cbuf) throws IOException`: 写入字符数组到输出流中
- `void write(char[] cbuf, int off, int len) throws IOException`: 写入字符数组的一部分，偏移量off开始，长度为len到输出流中
- `void write(String str) throws IOException`: 直接写入字符串到输出流中(常用)
- `void write(String str, int off, int len) throws IOException`: 写入字符串的一部分，偏移量off开始，长度为len
- `Writer append(char c) throws IOException`: 追加字符到输出流中

##### 文件节点流

字符流操作纯文本字符的文件是最合适的，主要有 `java.io.FileReader` 和 `java.io.FileWriter`

FileReader 定义如下

```java
/**
 * Convenience class for reading character files.  The constructors of this
 * class assume that the default character encoding and the default byte-buffer
 * size are appropriate.  To specify these values yourself, construct an
 * InputStreamReader on a FileInputStream.
 *
 * <p><code>FileReader</code> is meant for reading streams of characters.
 * For reading streams of raw bytes, consider using a
 * <code>FileInputStream</code>.
 *
 * @see InputStreamReader
 * @see FileInputStream
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */
public class FileReader extends InputStreamReader
```

FileReader 主要是向磁盘文件中写出数据，常用构造方法如下
- `public FileReader(String fileName) throws FileNotFoundException`
- `public FileReader(File file) throws FileNotFoundException`

注意： 当读取的文件不存在时，会抛出 `FileNotFoundException` ，这点和FileInputStream 一致

主要方法介绍

1. read() 读取一个字符

    示例

    ```java
    try(FileReader fr = new FileReader("D:/三国/赵云.txt")){
        int b;
        while ((b = fr.read()) != -1){
            System.out.println((char)b);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    ```

2. read(char[]) 读取多个字符

    示例

    ```java
    try(FileReader fr = new FileReader("D:/三国/赵云.txt")){
        int len;
        char[] data = new char[2];
        while ((len = fr.read(data)) != -1){
            System.out.println(new String(data, 0, len));
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    ```

FileWriter 定义如下

```java
/**
 * Convenience class for writing character files.  The constructors of this
 * class assume that the default character encoding and the default byte-buffer
 * size are acceptable.  To specify these values yourself, construct an
 * OutputStreamWriter on a FileOutputStream.
 *
 * <p>Whether or not a file is available or may be created depends upon the
 * underlying platform.  Some platforms, in particular, allow a file to be
 * opened for writing by only one <tt>FileWriter</tt> (or other file-writing
 * object) at a time.  In such situations the constructors in this class
 * will fail if the file involved is already open.
 *
 * <p><code>FileWriter</code> is meant for writing streams of characters.
 * For writing streams of raw bytes, consider using a
 * <code>FileOutputStream</code>.
 *
 * @see OutputStreamWriter
 * @see FileOutputStream
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */

public class FileWriter extends OutputStreamWriter
```

FileWriter构造方法如下
- public FileWriter(String fileName) throws IOException
- public FileWriter(String fileName, boolean append) throws IOException
- public FileWriter(File file) throws IOException
- public FileWriter(File file, boolean append) throws IOException

示例

```java
FileWriter fw = null;
try {
    fw = new FileWriter("D:/三国/孙权.txt");
    fw.write(97);
    fw.write('b');
    fw.write("权力");
    fw.append("游戏");
} catch (IOException e) {
    e.printStackTrace();
}finally {
    //try {
    //    fw.close();
    //} catch (IOException e) {
    //    e.printStackTrace();
    //}
}
```

执行上述代码后，生成了 `D:/三国/孙权.txt` 文件，但文件中没有内容。原因是没有执行 close() 或 flush() 方法，数据还在缓冲区中，没有输出到文件中。

解决方法：finally 语句块中添加关闭流的操作

也可以使用 jdk7 的 try-with-resource 进行优化

```java
try(FileWriter fw = new FileWriter("D:/三国/孙权.txt")){
    fw.write(97);
    fw.write('b');
    fw.write("权力");
    fw.append("游戏");
} catch (IOException e) {
    e.printStackTrace();
}
```

##### 内存字节流

字符流也有对应的内存节点流，常用的有 `java.io.StringWriter` 和 `java.io.CharArrayWriter`

StringWriter 是向内部的 StringBuffer 对象写数据。

```java
// 定义
public class StringWriter extends Writer {

    private StringBuffer buf;

    public StringWriter() {
        buf = new StringBuffer();
        lock = buf;
    }
}

// 应用
StringWriter sw = new StringWriter();
sw.write("hello");

StringBuffer buffer = sw.getBuffer();
// 输出hello
System.out.println(buffer.toString());
```

CharArrayWriter 是向内部的 char 数组写数据

```java
// 定义
public class CharArrayWriter extends Writer {
    protected char buf[];
}

// 应用 
CharArrayWriter caw = new CharArrayWriter();
caw.write("hello");
char[] chars = caw.toCharArray();
for (char c : chars) {
	// 输出了h e l l o
	System.out.println(c);
}
```

四种常用节点流的使用总结

![节点流总结](/images/20210224-节点流总结.png)  

#### 字节流和字符流的共同点

OutputStream、Reader、Writer都实现了Flushable接口，Flushable接口有flush()方法

flush()：强制刷新缓冲区的数据到目的地,刷新后流对象还可以继续使用

close(): 强制刷新缓冲区后关闭资源，关闭后流对象不可以继续使用

缓冲区：可以理解为内存区域，程序频繁操作资源（如文件）时，性能较低，因为读写内存较快，利用内存缓冲一部分数据，不要频繁的访问系统资源，是提高效率的一种方式

具体的流只要内部有维护了缓冲区，必须要close()或者flush()，不然不会真正的输出到文件中

### 处理流

上面的章节介绍了字节流和字符流的常用节点流，但是真正开发中都是使用更为强大的处理流

处理流是对节点流在功能上、性能上的增强

字节流的处理流的基类是 `java.io.FilterInputStream` 和 `java.io.FilterOutputStream`

FilterInputStream 定义如下

```java
/**
 * A <code>FilterInputStream</code> contains
 * some other input stream, which it uses as
 * its  basic source of data, possibly transforming
 * the data along the way or providing  additional
 * functionality. The class <code>FilterInputStream</code>
 * itself simply overrides all  methods of
 * <code>InputStream</code> with versions that
 * pass all requests to the contained  input
 * stream. Subclasses of <code>FilterInputStream</code>
 * may further override some of  these methods
 * and may also provide additional methods
 * and fields.
 *
 * @author  Jonathan Payne
 * @since   JDK1.0
 */
public
class FilterInputStream extends InputStream
```

FilterOutputStream 的定义如下

```java
/**
 * This class is the superclass of all classes that filter output
 * streams. These streams sit on top of an already existing output
 * stream (the <i>underlying</i> output stream) which it uses as its
 * basic sink of data, but possibly transforming the data along the
 * way or providing additional functionality.
 * <p>
 * The class <code>FilterOutputStream</code> itself simply overrides
 * all methods of <code>OutputStream</code> with versions that pass
 * all requests to the underlying output stream. Subclasses of
 * <code>FilterOutputStream</code> may further override some of these
 * methods as well as provide additional methods and fields.
 *
 * @author  Jonathan Payne
 * @since   JDK1.0
 */
public
class FilterOutputStream extends OutputStream
```

#### 缓冲流（重点）

前面说了节点流，都是直接使用操作系统底层方法读取硬盘中的数据，缓冲流是处理流的一种实现，增强了节点流的性能，为了提高效率，缓冲流类在初始化对象的时候，内部有一个缓冲数组，一次性从底层流中读取数据到数组中，程序中执行read()或者read(byte[])的时候，就直接从内存数组中读取数据。

分类
- 字节缓冲流：BufferedInputStream ， BufferedOutputStream
- 字符缓冲流：BufferedReader ， BufferedWriter

##### 字节缓冲流

`java.io.BufferedInputStream` 定义如下

```java

/**
 * A <code>BufferedInputStream</code> adds
 * functionality to another input stream-namely,
 * the ability to buffer the input and to
 * support the <code>mark</code> and <code>reset</code>
 * methods. When  the <code>BufferedInputStream</code>
 * is created, an internal buffer array is
 * created. As bytes  from the stream are read
 * or skipped, the internal buffer is refilled
 * as necessary  from the contained input stream,
 * many bytes at a time. The <code>mark</code>
 * operation  remembers a point in the input
 * stream and the <code>reset</code> operation
 * causes all the  bytes read since the most
 * recent <code>mark</code> operation to be
 * reread before new bytes are  taken from
 * the contained input stream.
 *
 * @author  Arthur van Hoff
 * @since   JDK1.0
 */
public
class BufferedInputStream extends FilterInputStream
```

构造方法如下

```java
private static int DEFAULT_BUFFER_SIZE = 8192;

/**
 * Creates a <code>BufferedInputStream</code>
 * and saves its  argument, the input stream
 * <code>in</code>, for later use. An internal
 * buffer array is created and  stored in <code>buf</code>.
 *
 * @param   in   the underlying input stream.
 */
public BufferedInputStream(InputStream in) {
    this(in, DEFAULT_BUFFER_SIZE);
}

/**
 * Creates a <code>BufferedInputStream</code>
 * with the specified buffer size,
 * and saves its  argument, the input stream
 * <code>in</code>, for later use.  An internal
 * buffer array of length  <code>size</code>
 * is created and stored in <code>buf</code>.
 *
 * @param   in     the underlying input stream.
 * @param   size   the buffer size.
 * @exception IllegalArgumentException if {@code size <= 0}.
 */
public BufferedInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

BufferedInputStream 内部默认 8192 = 8 * 1024 即 8M 的缓冲区

`java.io.BufferedOutputStream` 定义如下

```java
/**
 * The class implements a buffered output stream. By setting up such
 * an output stream, an application can write bytes to the underlying
 * output stream without necessarily causing a call to the underlying
 * system for each byte written.
 *
 * @author  Arthur van Hoff
 * @since   JDK1.0
 */
public
class BufferedOutputStream extends FilterOutputStream
```

构造方法如下

```java
/**
 * Creates a new buffered output stream to write data to the
 * specified underlying output stream.
 *
 * @param   out   the underlying output stream.
 */
public BufferedOutputStream(OutputStream out) {
    this(out, 8192);
}

/**
 * Creates a new buffered output stream to write data to the
 * specified underlying output stream with the specified buffer
 * size.
 *
 * @param   out    the underlying output stream.
 * @param   size   the buffer size.
 * @exception IllegalArgumentException if size &lt;= 0.
 */
public BufferedOutputStream(OutputStream out, int size) {
    super(out);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

示例：使用基本流、缓冲流拷贝数据

示例中待拷贝的文件大小为：15.7 MB

代码如下

```java
public static void FileStreamCopyTest(){
    long start = System.currentTimeMillis();
    try(FileInputStream fis = new FileInputStream("D:\\LoveIt-0.2.10.zip");
    FileOutputStream fos = new FileOutputStream("D:\\test.zip")){
        //使用基本流一次读取、写入一个字节
        int data;
        while ((data = fis.read()) != -1){
            fos.write(data);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(Thread.currentThread() + "拷贝耗时：" + (System.currentTimeMillis() - start));
}

public static void FileStreamArrayCopyTest(){
    long start = System.currentTimeMillis();
    try(FileInputStream fis = new FileInputStream("D:\\LoveIt-0.2.10.zip");
        FileOutputStream fos = new FileOutputStream("D:\\test1.zip")){
        //使用基本流一次读取、写入8M字节数组
        int len;
        byte[] data = new byte[1024 * 8];
        while ((len = fis.read(data)) != -1){
            fos.write(data, 0, len);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(Thread.currentThread() + "拷贝耗时：" + (System.currentTimeMillis() - start));
}

public static void BufferedStreamCopyTest(){
    long start = System.currentTimeMillis();
    try(FileInputStream fis = new FileInputStream("D:\\LoveIt-0.2.10.zip");
        FileOutputStream fos = new FileOutputStream("D:\\test2.zip")){
        //使用缓冲流一次读取、写入1字节
        int data;
        while ((data = fis.read()) != -1){
            fos.write(data);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(Thread.currentThread() + "拷贝耗时：" + (System.currentTimeMillis() - start));
}

public static void BufferedStreamArrayCopyTest(){
    long start = System.currentTimeMillis();
    try(FileInputStream fis = new FileInputStream("D:\\LoveIt-0.2.10.zip");
        FileOutputStream fos = new FileOutputStream("D:\\test3.zip")){
        //使用缓冲流一次读取、写入8M字节数组
        int len;
        byte[] data = new byte[1024 * 8];
        while ((len = fis.read(data)) != -1){
            fos.write(data, 0, len);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(Thread.currentThread() + "拷贝耗时：" + (System.currentTimeMillis() - start));
}

public static void main(String[] args) {
    new Thread(() -> FileStreamCopyTest(), "基本流传输1字节").start();

    new Thread(() -> FileStreamArrayCopyTest(),"基本流传输8M字节").start();

    new Thread(() -> BufferedStreamCopyTest(),"缓冲流传输1字节").start();

    new Thread(() -> BufferedStreamArrayCopyTest(),"缓冲流传输8M字节").start();
}
```

运行结果

```
Thread[缓冲流传输8M字节,5,main]拷贝耗时：211
Thread[基本流传输8M字节,5,main]拷贝耗时：212
Thread[基本流传输1字节,5,main]拷贝耗时：228992
Thread[缓冲流传输1字节,5,main]拷贝耗时：229842
```

由上述示例可以得出结论：缓冲流比普通流要快

注意： 采用边读边写的方式，一次传输几兆的数据效率比较高，如果采用先把文件的数据都读入内存，在进行写出，这样读写的次数是较小，但是占用太大的内存空间，一次读太大的数据也严重影响效率!


##### 字符缓冲流

字符缓冲流 `java.io.BufferedReader` 和 `java.io.BufferedWriter` 是对字符节点流的装饰，

`java.io.BufferedReader` 的定义如下

```java

/**
 * Reads text from a character-input stream, buffering characters so as to
 * provide for the efficient reading of characters, arrays, and lines.
 *
 * <p> The buffer size may be specified, or the default size may be used.  The
 * default is large enough for most purposes.
 *
 * <p> In general, each read request made of a Reader causes a corresponding
 * read request to be made of the underlying character or byte stream.  It is
 * therefore advisable to wrap a BufferedReader around any Reader whose read()
 * operations may be costly, such as FileReaders and InputStreamReaders.  For
 * example,
 *
 * <pre>
 * BufferedReader in
 *   = new BufferedReader(new FileReader("foo.in"));
 * </pre>
 *
 * will buffer the input from the specified file.  Without buffering, each
 * invocation of read() or readLine() could cause bytes to be read from the
 * file, converted into characters, and then returned, which can be very
 * inefficient.
 *
 * <p> Programs that use DataInputStreams for textual input can be localized by
 * replacing each DataInputStream with an appropriate BufferedReader.
 *
 * @see FileReader
 * @see InputStreamReader
 * @see java.nio.file.Files#newBufferedReader
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */

public class BufferedReader extends Reader
```

构造方法ruxia

```java
private static int defaultCharBufferSize = 8192;


/**
 * Creates a buffering character-input stream that uses an input buffer of
 * the specified size.
 *
 * @param  in   A Reader
 * @param  sz   Input-buffer size
 *
 * @exception  IllegalArgumentException  If {@code sz <= 0}
 */
public BufferedReader(Reader in, int sz) {
    super(in);
    if (sz <= 0)
        throw new IllegalArgumentException("Buffer size <= 0");
    this.in = in;
    cb = new char[sz];
    nextChar = nChars = 0;
}

/**
 * Creates a buffering character-input stream that uses a default-sized
 * input buffer.
 *
 * @param  in   A Reader
 */
public BufferedReader(Reader in) {
    this(in, defaultCharBufferSize);
}

```

`java.io.BufferedWriter` 的定义如下

```java
/**
 * Writes text to a character-output stream, buffering characters so as to
 * provide for the efficient writing of single characters, arrays, and strings.
 *
 * <p> The buffer size may be specified, or the default size may be accepted.
 * The default is large enough for most purposes.
 *
 * <p> A newLine() method is provided, which uses the platform's own notion of
 * line separator as defined by the system property <tt>line.separator</tt>.
 * Not all platforms use the newline character ('\n') to terminate lines.
 * Calling this method to terminate each output line is therefore preferred to
 * writing a newline character directly.
 *
 * <p> In general, a Writer sends its output immediately to the underlying
 * character or byte stream.  Unless prompt output is required, it is advisable
 * to wrap a BufferedWriter around any Writer whose write() operations may be
 * costly, such as FileWriters and OutputStreamWriters.  For example,
 *
 * <pre>
 * PrintWriter out
 *   = new PrintWriter(new BufferedWriter(new FileWriter("foo.out")));
 * </pre>
 *
 * will buffer the PrintWriter's output to the file.  Without buffering, each
 * invocation of a print() method would cause characters to be converted into
 * bytes that would then be written immediately to the file, which can be very
 * inefficient.
 *
 * @see PrintWriter
 * @see FileWriter
 * @see OutputStreamWriter
 * @see java.nio.file.Files#newBufferedWriter
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */

public class BufferedWriter extends Writer
```

构造方法如下

```java
private static int defaultCharBufferSize = 8192;
/**
 * Creates a buffered character-output stream that uses a default-sized
 * output buffer.
 *
 * @param  out  A Writer
 */
public BufferedWriter(Writer out) {
    this(out, defaultCharBufferSize);
}

/**
 * Creates a new buffered character-output stream that uses an output
 * buffer of the given size.
 *
 * @param  out  A Writer
 * @param  sz   Output-buffer size, a positive integer
 *
 * @exception  IllegalArgumentException  If {@code sz <= 0}
 */
public BufferedWriter(Writer out, int sz) {
    super(out);
    if (sz <= 0)
        throw new IllegalArgumentException("Buffer size <= 0");
    this.out = out;
    cb = new char[sz];
    nChars = sz;
    nextChar = 0;

    lineSeparator = java.security.AccessController.doPrivileged(
        new sun.security.action.GetPropertyAction("line.separator"));
}
```

字符缓冲流的特有方法

类 | 方法名 | 方法说明
-- | -- | --
BufferedReader | String readLine() throws IOException | 一行行读取，读取到最后一行返回null
BufferedWriter | void newLine() throws IOException | 写一个换行符到文件中，实现换行

`java.io.BufferedReader#readLine()` 方法定义如下

```java

/**
 * Reads a line of text.  A line is considered to be terminated by any one
 * of a line feed ('\n'), a carriage return ('\r'), or a carriage return
 * followed immediately by a linefeed.
 *
 * @return     A String containing the contents of the line, not including
 *             any line-termination characters, or null if the end of the
 *             stream has been reached
 *
 * @exception  IOException  If an I/O error occurs
 *
 * @see java.nio.file.Files#readAllLines
 */
public String readLine() throws IOException {
    return readLine(false);
}
```

`java.io.BufferedWriter#newLine` 方法定义如下

```java
/**
 * Writes a line separator.  The line separator string is defined by the
 * system property <tt>line.separator</tt>, and is not necessarily a single
 * newline ('\n') character.
 *
 * @exception  IOException  If an I/O error occurs
 */
public void newLine() throws IOException {
    write(lineSeparator);
}
```

示例：

```java
try(BufferedReader br = new BufferedReader(new FileReader("D://三国//赵云.txt"));
    BufferedWriter bw = new BufferedWriter(new FileWriter("D://三国//赵子龙.txt"))){
    String line = null;
    while ((line = br.readLine()) != null){
        bw.write(line);
        bw.newLine();
    }
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```

#### 转换流

##### 字符编码和字符集

字符编码

计算机存储的数据都是二进制的，而我们在电脑上看到的数字、英文、汉字等都是二进制转换的结果
- 将字符转换成二进制，为编码
- 将二进制转换为字符，为解码

​字符编码 就是 自然语言和二进制的对应规则

字符集
- 就是一个编码表，常见的字符集有ASCII字符集、GBK字符集、Unicode字符集等，具体各个编码的介绍在这里就不介绍了。

![字符集](/images/20210225-字符集.png)  

编码和解码时使用的字符集不一致，就可能出现乱码。

##### InputStreamReader

定义如下

```java

/**
 * An InputStreamReader is a bridge from byte streams to character streams: It
 * reads bytes and decodes them into characters using a specified {@link
 * java.nio.charset.Charset charset}.  The charset that it uses
 * may be specified by name or may be given explicitly, or the platform's
 * default charset may be accepted.
 *
 * <p> Each invocation of one of an InputStreamReader's read() methods may
 * cause one or more bytes to be read from the underlying byte-input stream.
 * To enable the efficient conversion of bytes to characters, more bytes may
 * be read ahead from the underlying stream than are necessary to satisfy the
 * current read operation.
 *
 * <p> For top efficiency, consider wrapping an InputStreamReader within a
 * BufferedReader.  For example:
 *
 * <pre>
 * BufferedReader in
 *   = new BufferedReader(new InputStreamReader(System.in));
 * </pre>
 *
 * @see BufferedReader
 * @see InputStream
 * @see java.nio.charset.Charset
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */

public class InputStreamReader extends Reader
```

InputStreamReader 是 Reader 的子类，读取字节，并使用指定的字符集将其解码为字符。字符集可以自己指定，也可以使用平台的默认字符集。

构造方法如下

```java
/**
 * Creates an InputStreamReader that uses the default charset.
 *
 * @param  in   An InputStream
 */
public InputStreamReader(InputStream in) {
    super(in);
    try {
        sd = StreamDecoder.forInputStreamReader(in, this, (String)null); // ## check lock object
    } catch (UnsupportedEncodingException e) {
        // The default encoding should always be available
        throw new Error(e);
    }
}

/**
 * Creates an InputStreamReader that uses the named charset.
 *
 * @param  in
 *         An InputStream
 *
 * @param  charsetName
 *         The name of a supported
 *         {@link java.nio.charset.Charset charset}
 *
 * @exception  UnsupportedEncodingException
 *             If the named charset is not supported
 */
public InputStreamReader(InputStream in, String charsetName)
    throws UnsupportedEncodingException
{
    super(in);
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
}

/**
 * Creates an InputStreamReader that uses the given charset.
 *
 * @param  in       An InputStream
 * @param  cs       A charset
 *
 * @since 1.4
 * @spec JSR-51
 */
public InputStreamReader(InputStream in, Charset cs) {
    super(in);
    if (cs == null)
        throw new NullPointerException("charset");
    sd = StreamDecoder.forInputStreamReader(in, this, cs);
}

/**
 * Creates an InputStreamReader that uses the given charset decoder.
 *
 * @param  in       An InputStream
 * @param  dec      A charset decoder
 *
 * @since 1.4
 * @spec JSR-51
 */
public InputStreamReader(InputStream in, CharsetDecoder dec) {
    super(in);
    if (dec == null)
        throw new NullPointerException("charset decoder");
    sd = StreamDecoder.forInputStreamReader(in, this, dec);
}
```

示例：

读取 UTF-8 字符集文件中的字符 “你好” ，可以看到如果使用 GBK 编码读取，会出现乱码

```java
try (InputStreamReader isr = new InputStreamReader(new FileInputStream("D:/三国/utf8.txt")); //默认 utf8编码
     //指定 gbk编码
     InputStreamReader isr2 = new InputStreamReader(new FileInputStream("D:/三国/utf8.txt"), "gbk")) {

    int read;
    while ((read = isr.read()) != -1) {
        System.out.println((char) read);
    }

    while ((read = isr2.read()) != -1) {
        System.out.println((char) read); //乱码
    }
} catch (UnsupportedEncodingException e) {
    e.printStackTrace();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```

##### OutputStreamWriter

`java.io.OutputStreamWriter` 是 Writer 的子类，使用指定的字符集将字符编码为字节。字符集可以自己指定，也可以使用平台的默认字符集。

定义如下

```java
/**
 * An OutputStreamWriter is a bridge from character streams to byte streams:
 * Characters written to it are encoded into bytes using a specified {@link
 * java.nio.charset.Charset charset}.  The charset that it uses
 * may be specified by name or may be given explicitly, or the platform's
 * default charset may be accepted.
 *
 * <p> Each invocation of a write() method causes the encoding converter to be
 * invoked on the given character(s).  The resulting bytes are accumulated in a
 * buffer before being written to the underlying output stream.  The size of
 * this buffer may be specified, but by default it is large enough for most
 * purposes.  Note that the characters passed to the write() methods are not
 * buffered.
 *
 * <p> For top efficiency, consider wrapping an OutputStreamWriter within a
 * BufferedWriter so as to avoid frequent converter invocations.  For example:
 *
 * <pre>
 * Writer out
 *   = new BufferedWriter(new OutputStreamWriter(System.out));
 * </pre>
 *
 * <p> A <i>surrogate pair</i> is a character represented by a sequence of two
 * <tt>char</tt> values: A <i>high</i> surrogate in the range '&#92;uD800' to
 * '&#92;uDBFF' followed by a <i>low</i> surrogate in the range '&#92;uDC00' to
 * '&#92;uDFFF'.
 *
 * <p> A <i>malformed surrogate element</i> is a high surrogate that is not
 * followed by a low surrogate or a low surrogate that is not preceded by a
 * high surrogate.
 *
 * <p> This class always replaces malformed surrogate elements and unmappable
 * character sequences with the charset's default <i>substitution sequence</i>.
 * The {@linkplain java.nio.charset.CharsetEncoder} class should be used when more
 * control over the encoding process is required.
 *
 * @see BufferedWriter
 * @see OutputStream
 * @see java.nio.charset.Charset
 *
 * @author      Mark Reinhold
 * @since       JDK1.1
 */

public class OutputStreamWriter extends Writer
```

构造方法如下

```java
/**
 * Creates an OutputStreamWriter that uses the named charset.
 *
 * @param  out
 *         An OutputStream
 *
 * @param  charsetName
 *         The name of a supported
 *         {@link java.nio.charset.Charset charset}
 *
 * @exception  UnsupportedEncodingException
 *             If the named encoding is not supported
 */
public OutputStreamWriter(OutputStream out, String charsetName)
    throws UnsupportedEncodingException
{
    super(out);
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    se = StreamEncoder.forOutputStreamWriter(out, this, charsetName);
}

/**
 * Creates an OutputStreamWriter that uses the default character encoding.
 *
 * @param  out  An OutputStream
 */
public OutputStreamWriter(OutputStream out) {
    super(out);
    try {
        se = StreamEncoder.forOutputStreamWriter(out, this, (String)null);
    } catch (UnsupportedEncodingException e) {
        throw new Error(e);
    }
}

/**
 * Creates an OutputStreamWriter that uses the given charset.
 *
 * @param  out
 *         An OutputStream
 *
 * @param  cs
 *         A charset
 *
 * @since 1.4
 * @spec JSR-51
 */
public OutputStreamWriter(OutputStream out, Charset cs) {
    super(out);
    if (cs == null)
        throw new NullPointerException("charset");
    se = StreamEncoder.forOutputStreamWriter(out, this, cs);
}

/**
 * Creates an OutputStreamWriter that uses the given charset encoder.
 *
 * @param  out
 *         An OutputStream
 *
 * @param  enc
 *         A charset encoder
 *
 * @since 1.4
 * @spec JSR-51
 */
public OutputStreamWriter(OutputStream out, CharsetEncoder enc) {
    super(out);
    if (enc == null)
        throw new NullPointerException("charset encoder");
    se = StreamEncoder.forOutputStreamWriter(out, this, enc);
}
```

示例：将 “你好” 写入文件。写入后两个文件的字符集不一样，文件大小也不一样

```java
try(OutputStreamWriter osw1 = new OutputStreamWriter(new FileOutputStream("D://nihao1.txt"));
    OutputStreamWriter osw2 = new OutputStreamWriter(new FileOutputStream("D://nihao2.txt"), "gbk")
){
    osw1.write("你好"); //utf8编码，占6个字节
    osw2.write("你好"); //gbk编码，占4个字节
} catch (UnsupportedEncodingException e) {
    e.printStackTrace();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```

![转换流](/images/20210225-转换流.png)  

#### 对象流

##### 序列化

jdk提供了对象序列化的方式，该序列化机制将对象转为二进制流，二进制流主要包括对象的数据、对象的类型、对象的属性。可以将java对象转为二进制流写入文件中。文件会持久保存了对象的信息。

同理，从文件中读出对象的信息为反序列化的过程

对象想序列化，满足的条件:
- 该类必须实现 java.io.Serializable 接口， Serializable 是一个标记接口(没有任何抽象方法)，不实现此接口的类将不会使任何状态序列化或反序列化，会抛出 NotSerializableException 。
- 该类的所有属性必须是可序列化的，如果有一个属性不需要可序列化的，则该属性使用 transient 关键字修饰

##### ObjectOutputStream

该类实现将对象序列化后写出到外部设备，如硬盘文件

```java
public ObjectOutputStream(OutputStream out) throws IOException{}
```

常用方法
- `void writeObject(Object obj) throws IOException`: 将指定的对象写出

示例

```java
static class User implements Serializable{
    private static final long serialVersionUID = 1L;
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
public static void main(String[] args) {
    User user = new User("小明", 20);
    try(ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("D://obj.txt"));){
        oos.writeObject(user);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

注意：
- 实现了Serializable的实体一定要加一个serialVersionUID变量。
- serialVersionUID生成后不要改变，避免反序列化失败，改变后会抛出InvalidClassException异常

##### ObjectInputStream

该类将ObjectOutputStream写出的对象反序列化成java对象

```java
public ObjectInputStream(InputStream in) throws IOException 
```

常用方法
- `Object readObject() throws IOException, ClassNotFoundException`: 读取对象

示例

```java
try(ObjectInputStream ois = new ObjectInputStream(new FileInputStream("D://obj.txt"))){
    User user = (User) ois.readObject();
    System.out.println(user);
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
} catch (ClassNotFoundException e) {
    e.printStackTrace();
}
```

##### 对象与字节数组的转换

利用对象流和字节数组流结合 ，可以实现java对象和byte[]之间的互转

```java
// 将对象转为byte[]
public static <T> byte[] t1(T t) {
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(bos);
    oos.writeObject(t);
    return bos.toByteArray();
}
// 将byte[]转为对象
public static <T> T t2(byte[] data) throws IOException, ClassNotFoundException {
    ByteArrayInputStream bos = new ByteArrayInputStream(data);
    ObjectInputStream oos = new ObjectInputStream(bos);
    return (T) oos.readObject();
}
```

#### 管道流

管道流主要用于两个线程间的通信，即一个线程通过管道流给另一个线程发数据

注意：线程的通信一般使用wait()/notify(),使用流也可以达到通信的效果，并且可以传递数据

- java.io.PipedInputStream 和 java.io.PipedOutputStream
- java.io.PipedReader 和 java.io.PipedWriter

使用字节流示例

```java
class Sender implements Runnable {
    private PipedOutputStream pos;
    private String msg;

    public Sender(String msg) {
        this.pos = new PipedOutputStream();
        this.msg = msg;
    }

    @Override
    public void run() {
       pos.write(msg.getBytes())；
    }

    public PipedOutputStream getPos() {
        return pos;
    }
}

class Receiver implements Runnable {
    private PipedInputStream pis;

    public Receiver() {
        this.pis = new PipedInputStream();
    }


    @Override
    public void run() {
         byte[] data = new byte[1024];
            int len;
            while ((len = pis.read(data)) != -1) {
                System.out.println(new String(data, 0, len));
            }
    }
}
    
Sender sender = new Sender("hello");
Receiver receiver = new Receiver();
receiver.getPis().connect(sender.getPos());
new Thread(sender).start();
new Thread(receiver).start();
// 控制台输出  hello
```

#### 输入流与输出流

System.in 和 System.out 代表了系统标准的输入、输出设备

默认输入设备是键盘，默认输出设备是控制台

可以使用System类的setIn，setOut方法对默认设备进行改变

我们开发中经常使用的输出到控制台上的内容的方法。

```java
System.out.println("a");
System.out.print("b");
class System{
    public final static InputStream in = null;
    public final static PrintStream out = null;
}
public PrintStream(String fileName) throws FileNotFoundException{}
```
#### 数据流

主要方便读取Java基本类型以及String的数据,有DataInputStream 和 DataOutputStream两个实现类

```java
DataOutputStream dos = new DataOutputStream(new FileOutputStream("D:/三国/周瑜.txt"));
dos.writeUTF("周瑜");
dos.writeBoolean(false);
dos.writeLong(1234567890L);

DataInputStream dis = new DataInputStream(new FileInputStream("D:/三国/周瑜.txt"));
String s = dis.readUTF();
System.out.println(s);
boolean b = dis.readBoolean();
System.out.println(b);
// 输出
周瑜
false
```

## IO 流总结

字节流

![字节流](/images/20210225-字节流.png)  


字节流

![字符流](/images/20210225-字符流.png)  

按功能划分

![io流](/images/20210225-io流.png)  

![输入输出](/images/20210225-输入输出.png)  




