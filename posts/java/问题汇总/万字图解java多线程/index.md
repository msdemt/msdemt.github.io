# 万字图解Java多线程


> 转载自：  
> https://blog.csdn.net/qq_35598736/article/details/108431422  


## 什么是 Java 多线程？

### 进程与线程

进程
- 当一个程序被运行，就开启了一个进程，比如 qq
- 程序由指令和数据组成，指令要运行，数据要加载，指令被 cpu 加载运行，数据被加载到内存，指令运行时可由 cpu 调度硬盘、网络等设备

线程
- 一个进程可分为多个线程
- 一个线程就是一个指令流， cpu 调度的最小单位，由 cpu 一条一条执行指令

### 并发与并行

并发
- 单核 cpu 运行多线程时，时间片进行很快的切换。线程轮流执行 cpu ，同一时刻只有一个线程被执行

并行
- 多核 cpu 运行多线程时，真正的在同一时刻，多个线程同时在运行。

![并发与并行](/images/20210220-并发与并行.png)  

Java 提供了丰富的 api 来支持多线程。

## 为什么用多线程？

多线程能实现的都可以用单线程来完成，那么 java 为什么要引入多线程的概念呢？

多线程的好处
- 程序运行的更快
- 充分利用 cpu 资源，目前几乎没有单核的 cpu ，多线程能发挥多核 cpu 的优势

## 多线程难在那里？

单线程只有一条执行线，过程容易理解，可以清晰勾勒出代码的执行流程

多线程确实多条执行线同时运行，而且多条执行线之间可能还会有交互，多条执行线之间有时还需要通信，一般难点有以下几点
- 多线程的执行结果不确定，受 cpu 调度的影响
- 多线程的安全问题
- 线程资源宝贵，依赖线程池操作线程，线程池的参数设置问题
- 多线程的执行是动态的，同时难以追踪执行过程
- 多线程的底层是操作系统层面的，源码难度大

## 多线程的基本使用

### 定义任务、创建和运行线程

任务：线程的执行体，也就是期望线程运行的代码逻辑

**定义任务**
- 继承 Thread 类（将任务与线程耦合在一起了）
- 实现 Runnable 接口（任务与线程解耦合）
- 实现 Callable 接口（任务与线程解耦合，可以获取到任务的执行结果）

继承 Thread 类实现任务的局限性
- 任务逻辑写在 Thread 类的 run 方法中，存在单继承的局限性
- 创建多线程时，每个任务有成员变量时不能共享，必须加上 static 才能做到共享

实现 Runnable 接口和实现 Callable 接口实现任务解决了 继承 Thread 类的局限性

但是实现 Runnable 接口与 实现Callable 接口相比，有如下局限性
- 任务没有返回值
- 任务无法抛出异常给调用方

**定义任务示例如下**

```java
class T extends Thread {
    @Override
    public void run() {
        System.out.println("继承Thread的任务");
    }
}

class R implements Runnable {
    @Override
    public void run() {
        System.out.println("实现Runnable的任务");
    }
}

class C implements Callable {
    @Override
    public Object call() throws Exception {
        System.out.println("实现Callable的任务");
        return "success";
    }
}
```

**创建线程的方式**
- 通过 Thread 类直接创建线程
- 利用线程池内部创建线程

**启动线程的方式**
- 调用线程的 start() 方法

**通过 Thread 类创建并启动线程示例**

```java
//启动继承Thread类的任务
new T().start();

//使用Thread匿名内部类启动任务
new Thread(){
    @Override
    public void run() {
        System.out.println("Thread匿名内部类的任务");
    }
}.start();

//启动实现Runnable接口的任务
new Thread(new R()).start();

//启动实现Runnable匿名实现类的任务
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Runnable匿名内部类的任务");
    }
}).start();

//启动实现Runnable的lambda简化后的任务
new Thread(() -> System.out.println("Runnable的lambda简化后的任务")).start();

//启动实现了Callable接口的任务 结合FutureTask 可以获取线程执行的结果
FutureTask futureTask = new FutureTask<>(new C());
new Thread(futureTask).start();
try {
    System.out.println(futureTask.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

**通过线程池创建启动线程示例**

```java
ExecutorService executorService = Executors.newFixedThreadPool(10);
//使用线程池执行实现Runnable接口的任务，任务没有返回值
executorService.execute(new R());

//使用线程池执行实现Runnable接口的任务，任务执行成功后，返回值为null
Future<?> future = executorService.submit(new R());
try {
    System.out.println("future 执行结果：" + future.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}

//使用线程池执行实现Callable接口的任务，可以接收到任务返回值
Future future1 = executorService.submit(new C());
try {
    System.out.println("future1 执行结果：" + future1.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}

//使用线程池执行实现Runnable接口的任务，可以接收到任务返回值
Data data = new Data();
Future<Data> future2 = executorService.submit(new R(data), data);
try {
    System.out.println("future2 执行结果：" + future2.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}

//线程池用完后关闭，否则程序无法退出
executorService.shutdown();
```

上述测试完整代码如下

```java
package org.msdemt.simple;

import java.util.concurrent.*;

public class MultiThreadTest {
    static
    class T extends Thread {
        @Override
        public void run() {
            System.out.println("继承Thread的任务");
        }
    }

    static
    class R implements Runnable {
        private Data data;

        public R() {
        }

        public R(Data data) {
            this.data = data;
        }

        @Override
        public void run() {
            System.out.println("实现Runnable的任务");
            if (data != null) {
                data.setName("abc");
            }
        }
    }

    static
    class C implements Callable {
        @Override
        public Object call() throws Exception {
            System.out.println("实现Callable的任务");
            return "success";
        }
    }

    static
    class Data {
        String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        @Override
        public String toString() {
            return "Data{" +
                    "name='" + name + '\'' +
                    '}';
        }
    }

    public static void main(String[] args) {
        /*new T().start();

        new Thread(){
            @Override
            public void run() {
                System.out.println("Thread匿名内部类的任务");
            }
        }.start();

        new Thread(new R()).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Runnable匿名内部类的任务");
            }
        }).start();

        new Thread(() -> System.out.println("Runnable的lambda简化后的任务")).start();

        FutureTask futureTask = new FutureTask<>(new C());
        new Thread(futureTask).start();
        try {
            System.out.println(futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }*/

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        //使用线程池执行实现Runnable接口的任务，任务没有返回值
        executorService.execute(new R());

        //使用线程池执行实现Runnable接口的任务，任务执行成功后，返回值为null
        Future<?> future = executorService.submit(new R());
        try {
            System.out.println("future 执行结果：" + future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        //使用线程池执行实现Callable接口的任务，可以接收到任务返回值
        Future future1 = executorService.submit(new C());
        try {
            System.out.println("future1 执行结果：" + future1.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        Data data = new Data();
        Future<Data> future2 = executorService.submit(new R(data), data);
        try {
            System.out.println("future2 执行结果：" + future2.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        executorService.shutdown();
    }
}
```

### 上下文切换

多核 cpu 下，多线程是并行工作的，如果线程数多，单个核又会并发的调度线程，运行时就会有上下文切换的概念。

cpu 执行线程的任务时，会为线程分配时间片，以下几种情况会发生上下文切换
- 线程的 cpu 时间片用完
- 垃圾回收
- 线程自己调用了 sleep 、 yield 、 wait 、 join 、 park 、 synchronized 、 lock 等方法

当发生上下文切换时，操作系统会保存当前线程的状态，并恢复另一个线程的状态， jvm 中有块内存地址叫程序计数器，用于记录线程执行到哪一行代码，是线程私有的。

idea 打断点的时候在断点上右键可以设置为 Thread 模式，idea 的 debug 模式可以看出栈帧的变化。

### 线程的礼让 - yield() & 线程优先级

yield() 方法会让运行中的线程切换到就绪状态，重新争抢 cpu 的时间片，争抢时是否获取到时间片看 cpu 的分配。

yield() 定义如下

```java
/**
 * A hint to the scheduler that the current thread is willing to yield
 * its current use of a processor. The scheduler is free to ignore this
 * hint.
 *
 * <p> Yield is a heuristic attempt to improve relative progression
 * between threads that would otherwise over-utilise a CPU. Its use
 * should be combined with detailed profiling and benchmarking to
 * ensure that it actually has the desired effect.
 *
 * <p> It is rarely appropriate to use this method. It may be useful
 * for debugging or testing purposes, where it may help to reproduce
 * bugs due to race conditions. It may also be useful when designing
 * concurrency control constructs such as the ones in the
 * {@link java.util.concurrent.locks} package.
 */
public static native void yield();
```

示例

```java
public static void main(String[] args) {
    Runnable r1 = () -> {
        int count = 0;
        for(;;){
            System.out.println("r1 ---> " + count++);
        }
    };
    Runnable r2 = () -> {
        int count = 0;
        for(;;){
            Thread.yield();
            System.out.println("r2 ---> " + count++);
        }
    };

    new Thread(r1).start();
    new Thread(r2).start();
}
```

运行结果

```
r1 ---> 755182
r1 ---> 755183
r1 ---> 755184
r1 ---> 755185
r1 ---> 755186
r2 ---> 35811
r1 ---> 755187
r1 ---> 755188
r1 ---> 755189
r1 ---> 755190
r1 ---> 755191
```

可以看到， t2 线程每次执行时进行了 yield() ，线程1执行的机会明显比线程2要多。

**线程优先级**

线程内部用 1~10 来调整线程的优先级，默认的线程优先级为 `NORM_PRIORITY = 5`

```java
/**
 * The minimum priority that a thread can have.
 */
public final static int MIN_PRIORITY = 1;

/**
 * The default priority that is assigned to a thread.
 */
public final static int NORM_PRIORITY = 5;

/**
 * The maximum priority that a thread can have.
 */
public final static int MAX_PRIORITY = 10;


/**
 * Changes the priority of this thread.
 * <p>
 * First the <code>checkAccess</code> method of this thread is called
 * with no arguments. This may result in throwing a
 * <code>SecurityException</code>.
 * <p>
 * Otherwise, the priority of this thread is set to the smaller of
 * the specified <code>newPriority</code> and the maximum permitted
 * priority of the thread's thread group.
 *
 * @param newPriority priority to set this thread to
 * @exception  IllegalArgumentException  If the priority is not in the
 *               range <code>MIN_PRIORITY</code> to
 *               <code>MAX_PRIORITY</code>.
 * @exception  SecurityException  if the current thread cannot modify
 *               this thread.
 * @see        #getPriority
 * @see        #checkAccess()
 * @see        #getThreadGroup()
 * @see        #MAX_PRIORITY
 * @see        #MIN_PRIORITY
 * @see        ThreadGroup#getMaxPriority()
 */
public final void setPriority(int newPriority) {
    ThreadGroup g;
    checkAccess();
    if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
        throw new IllegalArgumentException();
    }
    if((g = getThreadGroup()) != null) {
        if (newPriority > g.getMaxPriority()) {
            newPriority = g.getMaxPriority();
        }
        setPriority0(priority = newPriority);
    }
}
```

cpu 比较忙时，优先级高的线程获取更多的时间片

cpu 比较闲时，优先级设置基本没用。


cpu 比较忙时，示例

```java
public static void main(String[] args) {
    Runnable r1 = () -> {
        int count = 0;
        for(;;){
            System.out.println("r1 ---> " + count++);
        }
    };
    Runnable r2 = () -> {
        int count = 0;
        for(;;){
            //Thread.yield();
            System.out.println("r2 ---> " + count++);
        }
    };

    Thread t1 = new Thread(r1);
    t1.setPriority(Thread.NORM_PRIORITY);
    t1.start();

    Thread t2 = new Thread(r2);
    t2.setPriority(Thread.MAX_PRIORITY);
    t2.start();
}
```

运行结果

```
r2 ---> 225257
r2 ---> 225258
r2 ---> 225259
r1 ---> 35545
r1 ---> 35546
r1 ---> 35547
r2 ---> 225260
r2 ---> 225261
r2 ---> 225262
r2 ---> 225263
```

cpu 比较闲时，示例

```java
public static void main(String[] args) {
    Runnable r1 = () -> {
        int count = 0;
        for (int i = 0; i < 10; i++) {
            System.out.println("r1 ---> " + count++);
        }
    };
    Runnable r2 = () -> {
        int count = 0;
        for (int i = 0; i < 10; i++) {
            //Thread.yield();
            System.out.println("r2 ---> " + count++);
        }
    };

    Thread t1 = new Thread(r1);
    t1.setPriority(Thread.NORM_PRIORITY);
    t1.start();

    Thread t2 = new Thread(r2);
    t2.setPriority(Thread.MAX_PRIORITY);
    t2.start();
}
```


运行结果（可能的结果：t1优先级低，却先运行完）

```
r1 ---> 0
r1 ---> 1
r1 ---> 2
r1 ---> 3
r1 ---> 4
r1 ---> 5
r2 ---> 0
r2 ---> 1
r2 ---> 2
r2 ---> 3
r2 ---> 4
r2 ---> 5
r2 ---> 6
r2 ---> 7
r2 ---> 8
r2 ---> 9
r1 ---> 6
r1 ---> 7
r1 ---> 8
r1 ---> 9
```

### 守护线程

默认情况下， java 进程需要等待所有线程都运行结束，才会结束，但有一种特殊线程叫守护线程，当所有非守护线程都结束后，即使它没有执行完，也会强制结束。

默认的线程都是非守护线程。

垃圾回收线程就是典型的守护线程

```java
public static void main(String[] args) {
    Thread thread = new Thread(() -> {
        while (true){
            System.out.println("thread running ...");
        }
    });
    //true 将该线程设置为守护线程，主线程结束后，守护线程也会终止运行
    //false 默认值，线程为非守护线程，主线程结束后，该线程还会继续运行
    thread.setDaemon(true);
    thread.start();
    System.out.println("主线程结束运行");
}
```

### 线程的阻塞

线程的阻塞可以分为好多种，从操作系统层面和 java 层面阻塞的定义可能不同，但是广义上使得线程阻塞的方式有下面几种
- BIO 阻塞，即使用了阻塞式的 io 流
- sleep(long time) 让线程休眠进入阻塞状态
- a.join() 调用该方法的线程进入阻塞，等待 a 线程执行完恢复运行
- sychronized 或 ReentrantLock 造成线程未获得锁而进入阻塞状态
- 线程获得锁后调用 wait() 方法，也会进入阻塞状态
- LockSupport.park() 让线程进入阻塞状态

#### sleep()

sleep() 会让线程休眠，将运行中的线程进入阻塞状态。当休眠时间结束后，重新争抢 cpu 的时间片继续运行

sleep() 方法定义如下  

java.lang.Thread#sleep(long)

```java
/**
 * Causes the currently executing thread to sleep (temporarily cease
 * execution) for the specified number of milliseconds, subject to
 * the precision and accuracy of system timers and schedulers. The thread
 * does not lose ownership of any monitors.
 *
 * @param  millis
 *         the length of time to sleep in milliseconds
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public static native void sleep(long millis) throws InterruptedException;
```

示例

```java
public static void main(String[] args) {
    try {
        //休眠2秒，休眠过程可被中断，中断后抛出 InterruptedException
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    try {
        //使用TimeUnit的api可替代Thread.sleep
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```


#### join()

join() 的作用是 join() 方法所在的线程进入阻塞状态，等待调用 join() 方法的线程执行完后join() 方法所在的线程恢复运行

join() 方法可以指定时间，指定时间结束后，若调用join()的线程还未执行完，join() 方法所在线程不继续等待就恢复运行

join() 的定义如下

```java
/**
 * Waits at most {@code millis} milliseconds for this thread to
 * die. A timeout of {@code 0} means to wait forever.
 *
 * <p> This implementation uses a loop of {@code this.wait} calls
 * conditioned on {@code this.isAlive}. As a thread terminates the
 * {@code this.notifyAll} method is invoked. It is recommended that
 * applications not use {@code wait}, {@code notify}, or
 * {@code notifyAll} on {@code Thread} instances.
 *
 * @param  millis
 *         the time to wait in milliseconds
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}

/**
 * Waits at most {@code millis} milliseconds plus
 * {@code nanos} nanoseconds for this thread to die.
 *
 * <p> This implementation uses a loop of {@code this.wait} calls
 * conditioned on {@code this.isAlive}. As a thread terminates the
 * {@code this.notifyAll} method is invoked. It is recommended that
 * applications not use {@code wait}, {@code notify}, or
 * {@code notifyAll} on {@code Thread} instances.
 *
 * @param  millis
 *         the time to wait in milliseconds
 *
 * @param  nanos
 *         {@code 0-999999} additional nanoseconds to wait
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative, or the value
 *          of {@code nanos} is not in the range {@code 0-999999}
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public final synchronized void join(long millis, int nanos)
throws InterruptedException {

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }

    join(millis);
}

/**
 * Waits for this thread to die.
 *
 * <p> An invocation of this method behaves in exactly the same
 * way as the invocation
 *
 * <blockquote>
 * {@linkplain #join(long) join}{@code (0)}
 * </blockquote>
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public final void join() throws InterruptedException {
    join(0);
}
```

示例

```java
public static void main(String[] args) {
    Integer r = 0;
    Thread t = new Thread(() -> {
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e){
            e.printStackTrace();
        }
        System.out.println("子线程运行结束");
    });

    t.start();

    try {
        t.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("主线程运行结束");
}
```

输出结果：

```
子线程运行结束
主线程运行结束
```

若注释掉 t.join() 所在的代码块，或者配置 t.join(1000) ，则运行结果如下

```
主线程运行结束
子线程运行结束
```

#### interrupt()

线程的打断标记
- true 表示线程被打断
- false 表示线程未被打断

interrupt() 打断当前线程
- Unless the current thread is interrupting itself, which is always permitted, the checkAccess method of this thread is invoked, which may cause a SecurityException to be thrown.
- 可以打断 sleep 、 wait 、 join 等显式的抛出 InterruptedException 方法的线程，但是打断后线程的标记还是 false
  - If this thread is blocked in an invocation of the wait(), wait(long), or wait(long, int) methods of the Object class, or of the join(), join(long), join(long, int), sleep(long), or sleep(long, int), methods of this class, then its interrupt status will be cleared and it will receive an InterruptedException.
- If this thread is blocked in an I/O operation upon an InterruptibleChannel then the channel will be closed, the thread's interrupt status will be set, and the thread will receive a java.nio.channels.ClosedByInterruptException.
- If this thread is blocked in a java.nio.channels.Selector then the thread's interrupt status will be set and it will return immediately from the selection operation, possibly with a non-zero value, just as if the selector's wakeup method were invoked.
- 打断正常线程，线程不会真正被中断，但是线程的打断标记为 true
  - If none of the previous conditions hold then this thread's interrupt status will be set.
- Interrupting a thread that is not alive need not have any effect.

isInterrupted() 获取线程的打断标记，调用后不会修改线程的打断标记
- Tests whether this thread has been interrupted. The interrupted status of the thread is unaffected by this method.

interrupted() 获取线程的打断标记，调用后清空打断标记，即如果获取的打断标记为 true ，调用该方法后会将打断标记置为 false
- Tests whether the current thread has been interrupted. The interrupted status of the thread is cleared by this method. In other words, if this method were to be called twice in succession, the second call would return false (unless the current thread were interrupted again, after the first call had cleared its interrupted status and before the second call had examined it).

定义如下

```java
/**
 * Interrupts this thread.
 *
 * <p> Unless the current thread is interrupting itself, which is
 * always permitted, the {@link #checkAccess() checkAccess} method
 * of this thread is invoked, which may cause a {@link
 * SecurityException} to be thrown.
 *
 * <p> If this thread is blocked in an invocation of the {@link
 * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
 * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
 * class, or of the {@link #join()}, {@link #join(long)}, {@link
 * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
 * methods of this class, then its interrupt status will be cleared and it
 * will receive an {@link InterruptedException}.
 *
 * <p> If this thread is blocked in an I/O operation upon an {@link
 * java.nio.channels.InterruptibleChannel InterruptibleChannel}
 * then the channel will be closed, the thread's interrupt
 * status will be set, and the thread will receive a {@link
 * java.nio.channels.ClosedByInterruptException}.
 *
 * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
 * then the thread's interrupt status will be set and it will return
 * immediately from the selection operation, possibly with a non-zero
 * value, just as if the selector's {@link
 * java.nio.channels.Selector#wakeup wakeup} method were invoked.
 *
 * <p> If none of the previous conditions hold then this thread's interrupt
 * status will be set. </p>
 *
 * <p> Interrupting a thread that is not alive need not have any effect.
 *
 * @throws  SecurityException
 *          if the current thread cannot modify this thread
 *
 * @revised 6.0
 * @spec JSR-51
 */
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}

private native void interrupt0();


/**
 * Tests whether the current thread has been interrupted.  The
 * <i>interrupted status</i> of the thread is cleared by this method.  In
 * other words, if this method were to be called twice in succession, the
 * second call would return false (unless the current thread were
 * interrupted again, after the first call had cleared its interrupted
 * status and before the second call had examined it).
 *
 * <p>A thread interruption ignored because a thread was not alive
 * at the time of the interrupt will be reflected by this method
 * returning false.
 *
 * @return  <code>true</code> if the current thread has been interrupted;
 *          <code>false</code> otherwise.
 * @see #isInterrupted()
 * @revised 6.0
 */
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}


/**
 * Tests whether this thread has been interrupted.  The <i>interrupted
 * status</i> of the thread is unaffected by this method.
 *
 * <p>A thread interruption ignored because a thread was not alive
 * at the time of the interrupt will be reflected by this method
 * returning false.
 *
 * @return  <code>true</code> if this thread has been interrupted;
 *          <code>false</code> otherwise.
 * @see     #interrupted()
 * @revised 6.0
 */
public boolean isInterrupted() {
    return isInterrupted(false);
}

/**
 * Tests if some Thread has been interrupted.  The interrupted state
 * is reset or not based on the value of ClearInterrupted that is
 * passed.
 */
private native boolean isInterrupted(boolean ClearInterrupted);

```


示例代码如下

```java
/**
 * ThreadTest
 * 
 * @author: hekai
 */
public class ThreadTest {

    // 监控线程
    private static Thread monitor;

    public static void start() {
        monitor = new Thread(() -> {
            while (true) {
                Thread thread = Thread.currentThread();
                // 判断当前线程释放被打断
                if (thread.isInterrupted()) {
                    System.out.println("当前线程被打断，结束运行");
                    break;
                }

                try {
                    // 打断 sleep 、 wait 、 join 等显式的抛出 InterruptedException 方法的线程时，
                    // 打断后线程的标记还是 false ，需要在异常捕获处将打断标记置为 true
                    Thread.sleep(1000);
                    System.out.println("线程监控中...");
                } catch (InterruptedException e) {
                    // 若睡眠被打断时跑出异常，在该处捕获
                    // 再调用一次打断，使打断标记置为true
                    System.out.println("睡眠被打断时抛出了异常...");
                    thread.interrupt();
                }
            }
        });

        monitor.start();
    }

    public static void stop() {
        monitor.interrupt();
    }

    public static void main(String[] args) {
        start();

        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        stop();
    }
}
```

### 线程的状态

线程的状态从操作系统层面可分为五种，从 java api 层面可分为六种。

五种状态

![线程的五种状态](/images/20210220-线程的五种状态.png)  

- 初始状态：创建线程对象时的状态
- 可运行状态（就绪状态）：调用 start() 方法后进入就绪状态，也就是准备好被 cpu 调度执行
- 运行状态：线程获取到 cpu 时间片，执行 run() 方法
- 阻塞状态：线程被阻塞，放弃 cpu 的控制权，等待解除阻塞后重新回到就绪状态争抢 cpu 时间片
- 终止状态：线程执行完成或者跑出异常后的状态

六种状态

![线程的六种状态](/images/20210220-线程的六种状态.png)  

java.lang.Thread 类中的内部内举状态

```java
/**
 * A thread state.  A thread can be in one of the following states:
 * <ul>
 * <li>{@link #NEW}<br>
 *     A thread that has not yet started is in this state.
 *     </li>
 * <li>{@link #RUNNABLE}<br>
 *     A thread executing in the Java virtual machine is in this state.
 *     </li>
 * <li>{@link #BLOCKED}<br>
 *     A thread that is blocked waiting for a monitor lock
 *     is in this state.
 *     </li>
 * <li>{@link #WAITING}<br>
 *     A thread that is waiting indefinitely for another thread to
 *     perform a particular action is in this state.
 *     </li>
 * <li>{@link #TIMED_WAITING}<br>
 *     A thread that is waiting for another thread to perform an action
 *     for up to a specified waiting time is in this state.
 *     </li>
 * <li>{@link #TERMINATED}<br>
 *     A thread that has exited is in this state.
 *     </li>
 * </ul>
 *
 * <p>
 * A thread can be in only one state at a given point in time.
 * These states are virtual machine states which do not reflect
 * any operating system thread states.
 *
 * @since   1.5
 * @see #getState
 */
public enum State {
    /**
     * Thread state for a thread which has not yet started.
     */
    NEW,

    /**
     * Thread state for a runnable thread.  A thread in the runnable
     * state is executing in the Java virtual machine but it may
     * be waiting for other resources from the operating system
     * such as processor.
     */
    RUNNABLE,

    /**
     * Thread state for a thread blocked waiting for a monitor lock.
     * A thread in the blocked state is waiting for a monitor lock
     * to enter a synchronized block/method or
     * reenter a synchronized block/method after calling
     * {@link Object#wait() Object.wait}.
     */
    BLOCKED,

    /**
     * Thread state for a waiting thread.
     * A thread is in the waiting state due to calling one of the
     * following methods:
     * <ul>
     *   <li>{@link Object#wait() Object.wait} with no timeout</li>
     *   <li>{@link #join() Thread.join} with no timeout</li>
     *   <li>{@link LockSupport#park() LockSupport.park}</li>
     * </ul>
     *
     * <p>A thread in the waiting state is waiting for another thread to
     * perform a particular action.
     *
     * For example, a thread that has called <tt>Object.wait()</tt>
     * on an object is waiting for another thread to call
     * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
     * that object. A thread that has called <tt>Thread.join()</tt>
     * is waiting for a specified thread to terminate.
     */
    WAITING,

    /**
     * Thread state for a waiting thread with a specified waiting time.
     * A thread is in the timed waiting state due to calling one of
     * the following methods with a specified positive waiting time:
     * <ul>
     *   <li>{@link #sleep Thread.sleep}</li>
     *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
     *   <li>{@link #join(long) Thread.join} with timeout</li>
     *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
     *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
     * </ul>
     */
    TIMED_WAITING,

    /**
     * Thread state for a terminated thread.
     * The thread has completed execution.
     */
    TERMINATED;
}
```

- New: 线程对象被创建
- Runnable: 线程调用了 start() 方法后进入该状态，该状态包含三种情况
  - 就绪状态：等待 cpu 分配时间片
  - 运行状态：进入 Runnable 方法执行任务
  - 阻塞状态：BIO 执行阻塞式 io 流时的状态
- Blocked: 没有获取到锁时的阻塞状态
- WAITING: 调用 wait()、join() 等方法后的状态
- TIMED_WAITING: 调用 sleep(time)、wait(time)、join(time)等方法后的状态
- TERMINATED: 线程执行完成或抛出异常后的状态

六种线程状态和java api方法的对应关系如下：

![六种线程状态和api方法对应关系](/images/20210220-六种线程状态和api方法对应关系.png)  

### 线程的相关方法总结

Thread 类的核心方法

方法名称 | 是否 static | 方法说明
-- | -- | --
start() | 否 | 让线程启动，进入就绪状态，等待cpu分配时间片
run() | 否 | 重写Runnable接口的方法，线程获取到cpu时间片时执行的具体逻辑
yield() | 是 | 线程的礼让，使得获取到cpu时间片的线程进入就绪状态，重新争抢时间片
sleep(time) | 是 | 线程休眠固定时间，进入阻塞状态，休眠时间完成后重新争抢时间片，休眠可被打断
join()/join(time) | 否 | 调用线程对象的join()方法，调用者线程进入阻塞，等待线程对象执行完或者到达指定时间才恢复，重新争抢时间片
isInterrupted() | 否 | 获取线程的打断标记，true: 被打断，false: 没有被打断。调用后不会修改打断标记
interrupt() | 否 | 打断线程，抛出 InterruptedException 异常的方法均可被打断，但是打断后不会修改打断标记，正常执行的线程被打断后会修改打断标记
interrupted() | 否 | 获取线程的打断标记，调用后会清空打断标记
stop() | 否 | 停止线程运行 不推荐
suspend() | 否 | 挂起线程 不推荐
resume() | 否 | 恢复线程运行 不推荐
currentThread() | 是 | 获取当前线程

Object 中与线程相关的方法

方法名称 | 方法说明
-- | --
wait()/wait(long timeout) | 获取到锁的线程进入阻塞状态
notify() | 随机唤醒被 wait() 的一个线程
notifyAll() | 唤醒被 wait() 的所有线程，重新争抢时间片

## 同步锁

### 线程安全

一个程序运行多个线程本身是没有问题的，但是如果多个线程访问共享资源时，若多个线程都是**读**共享资源，没有问题；若多个线程**读写**共享资源时，如果发生**指令交错**，就会出现问题

**临界区**：一段代码如果是对共享资源的多线程读写操作，这段代码就被称为临界区

注意：**指令交错**是指java代码在解析成字节码文件时，java代码的一行代码在字节码中可能有多行，在线程上下文切换时就有可能交错。

**线程安全**是指多线程调用同一个对象的临界区的方法时，对象的属性值一定不会发生错误，这就是保证了线程安全。


线程不安全示例

```java
//对象的成员变量
private static int count = 0;
public static void main(String[] args) throws InterruptedException {
    //t1线程对count加1执行5000次
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            count++;
        }
    });
    //t2线程对count减1执行5000次
    Thread t2 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            count--;
        }
    });

    t1.start();
    t2.start();

    //主线程等待t1和t2执行完
    t1.join();
    t2.join();
    //输出最后的结果
    System.out.printf("count = %d", count);
}
```

上述代码中，t1 线程对变量 count 执行 5000 次加 1 操作，t2 线程对变量 count 执行 5000 次减 1 操作，如果是线程安全，那么 count 的输出结果应该为 0 。但是实际运行后，count 的结果每次都不同，且都不是 0 ，所以上述示例不是线程安全的。

**线程安全的类一定所有的操作都是线程安全吗？**

开发中经常会说到一些线程安全的类，比如 ConcurrentHashMap ，线程安全指的是类里每一个独立的方法是线程安全的，但是**方法的组合就不一定是线程安全的**。

成员变量和静态变量是否线程安全？
- 如果没有多线程共享，则是线程安全的
- 如果存在多线程共享
  - 多线程只有读操作，则线程安全
  - 多线程存在写操作，写操作的代码又是临界区，则线程不安全

局部变量是否线程安全？
- 局部变量是线程安全的
- 局部变量引用的对象未必是线程安全的
  - 如果该对象没有逃离该方法的作用范围，则线程安全
  - 如果该对象逃离了该方法的作用范围，比如方法的返回值，则需要考虑线程安全

### synchronized

同步锁，也叫对象锁，是锁在对象上的，不同的对象就是不同的锁

该关键字用于保证线程安全，是**阻塞式**的解决方案

让同一时刻最多只有一个线程能持有对象锁，其他线程再想获取这个对象锁就会被阻塞，不用担心上下文切换的问题。

注意：不要理解为若代码块加了 synchronized 锁，获取锁的线程进入 synchronized 代码块后会一直执行下去。如果该线程在 synchronized 代码块中发生了时间片切换，该线程会进入就绪状态，等待再次获得时间片后继续执行 synchronized 代码块，在这个过程中，当前线程还持有着锁。

当一个线程执行完 synchronized 代码块后，会唤醒正在等待的其他线程。

synchronized 实际上适用对象锁保证临界区的**原子性**，保证临界区的代码是不可分割的，不会因为线程切换而打断。

synchronized 的使用示例

```java
//加在方法上，实际是对this对象加锁
private synchronized void a(){

}

//同步代码块，锁对象可以是任意的，加在this上和 a() 方法的作用相同
private void b(){
    synchronized(this){

    }
}

//加在静态方法上，实际是对类对象加锁
private synchronized static void c(){

}

//同步代码块，实际是对类对象加锁，和 c() 方法的作用相同
private void d(){
    synchronized (TestSynchronized.class){

    }
}
```

线程安全代码示例

```java
//对象的成员变量
private static int count = 0;
private static Object lock = new Object();
public static void main(String[] args) throws InterruptedException {
    //t1线程对count加1执行5000次
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            synchronized (lock){
                count++;
            }
        }
    });
    //t2线程对count减1执行5000次
    Thread t2 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            synchronized (lock){
                count--;
            }
        }
    });

    t1.start();
    t2.start();

    //主线程等待t1和t2执行完
    t1.join();
    t2.join();
    //输出最后的结果
    System.out.printf("count = %d", count);
}
```

重点：加锁是加在对象上，一定要保证加在同一个对象上，加锁才能生效。

### Lock


https://www.cnblogs.com/fsmly/p/10703804.html

https://blog.csdn.net/u013568373/article/details/98480603

synchronized锁： java 提供的内置锁机制，java中的每个对象都可以作为一个实现同步的锁（内置锁或监视器Monitor），线程在进入同步代码块之前需要获取这把锁，在退出同步代码块后会释放锁。 synchronized 这种内置锁是**互斥**的，即每把锁最多只能由一个线程持有。

lock锁：Lock 接口提供了与 synchronized 相似的同步功能，和 synchronized（隐式的获取和释放锁，主要体现在线程进入同步代码块之前需要获取锁，退出同步代码块时需要释放锁）不同的是，Lock 在使用的时候是显式的获取和释放锁。虽然 Lock 接口缺少了 synchronized 隐式获取和释放锁的便捷性，但是对于锁的操作具有更强的可操作性、可控制性以及提供可中断操作和超时获取锁等机制。

Lock 接口定义如下

```java
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;

/**
 * {@code Lock} implementations provide more extensive locking
 * operations than can be obtained using {@code synchronized} methods
 * and statements.  They allow more flexible structuring, may have
 * quite different properties, and may support multiple associated
 * {@link Condition} objects.
 *
 * <p>A lock is a tool for controlling access to a shared resource by
 * multiple threads. Commonly, a lock provides exclusive access to a
 * shared resource: only one thread at a time can acquire the lock and
 * all access to the shared resource requires that the lock be
 * acquired first. However, some locks may allow concurrent access to
 * a shared resource, such as the read lock of a {@link ReadWriteLock}.
 *
 * <p>The use of {@code synchronized} methods or statements provides
 * access to the implicit monitor lock associated with every object, but
 * forces all lock acquisition and release to occur in a block-structured way:
 * when multiple locks are acquired they must be released in the opposite
 * order, and all locks must be released in the same lexical scope in which
 * they were acquired.
 *
 * <p>While the scoping mechanism for {@code synchronized} methods
 * and statements makes it much easier to program with monitor locks,
 * and helps avoid many common programming errors involving locks,
 * there are occasions where you need to work with locks in a more
 * flexible way. For example, some algorithms for traversing
 * concurrently accessed data structures require the use of
 * &quot;hand-over-hand&quot; or &quot;chain locking&quot;: you
 * acquire the lock of node A, then node B, then release A and acquire
 * C, then release B and acquire D and so on.  Implementations of the
 * {@code Lock} interface enable the use of such techniques by
 * allowing a lock to be acquired and released in different scopes,
 * and allowing multiple locks to be acquired and released in any
 * order.
 *
 * <p>With this increased flexibility comes additional
 * responsibility. The absence of block-structured locking removes the
 * automatic release of locks that occurs with {@code synchronized}
 * methods and statements. In most cases, the following idiom
 * should be used:
 *
 *  <pre> {@code
 * Lock l = ...;
 * l.lock();
 * try {
 *   // access the resource protected by this lock
 * } finally {
 *   l.unlock();
 * }}</pre>
 *
 * When locking and unlocking occur in different scopes, care must be
 * taken to ensure that all code that is executed while the lock is
 * held is protected by try-finally or try-catch to ensure that the
 * lock is released when necessary.
 *
 * <p>{@code Lock} implementations provide additional functionality
 * over the use of {@code synchronized} methods and statements by
 * providing a non-blocking attempt to acquire a lock ({@link
 * #tryLock()}), an attempt to acquire the lock that can be
 * interrupted ({@link #lockInterruptibly}, and an attempt to acquire
 * the lock that can timeout ({@link #tryLock(long, TimeUnit)}).
 *
 * <p>A {@code Lock} class can also provide behavior and semantics
 * that is quite different from that of the implicit monitor lock,
 * such as guaranteed ordering, non-reentrant usage, or deadlock
 * detection. If an implementation provides such specialized semantics
 * then the implementation must document those semantics.
 *
 * <p>Note that {@code Lock} instances are just normal objects and can
 * themselves be used as the target in a {@code synchronized} statement.
 * Acquiring the
 * monitor lock of a {@code Lock} instance has no specified relationship
 * with invoking any of the {@link #lock} methods of that instance.
 * It is recommended that to avoid confusion you never use {@code Lock}
 * instances in this way, except within their own implementation.
 *
 * <p>Except where noted, passing a {@code null} value for any
 * parameter will result in a {@link NullPointerException} being
 * thrown.
 *
 * <h3>Memory Synchronization</h3>
 *
 * <p>All {@code Lock} implementations <em>must</em> enforce the same
 * memory synchronization semantics as provided by the built-in monitor
 * lock, as described in
 * <a href="https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4">
 * The Java Language Specification (17.4 Memory Model)</a>:
 * <ul>
 * <li>A successful {@code lock} operation has the same memory
 * synchronization effects as a successful <em>Lock</em> action.
 * <li>A successful {@code unlock} operation has the same
 * memory synchronization effects as a successful <em>Unlock</em> action.
 * </ul>
 *
 * Unsuccessful locking and unlocking operations, and reentrant
 * locking/unlocking operations, do not require any memory
 * synchronization effects.
 *
 * <h3>Implementation Considerations</h3>
 *
 * <p>The three forms of lock acquisition (interruptible,
 * non-interruptible, and timed) may differ in their performance
 * characteristics, ordering guarantees, or other implementation
 * qualities.  Further, the ability to interrupt the <em>ongoing</em>
 * acquisition of a lock may not be available in a given {@code Lock}
 * class.  Consequently, an implementation is not required to define
 * exactly the same guarantees or semantics for all three forms of
 * lock acquisition, nor is it required to support interruption of an
 * ongoing lock acquisition.  An implementation is required to clearly
 * document the semantics and guarantees provided by each of the
 * locking methods. It must also obey the interruption semantics as
 * defined in this interface, to the extent that interruption of lock
 * acquisition is supported: which is either totally, or only on
 * method entry.
 *
 * <p>As interruption generally implies cancellation, and checks for
 * interruption are often infrequent, an implementation can favor responding
 * to an interrupt over normal method return. This is true even if it can be
 * shown that the interrupt occurred after another action may have unblocked
 * the thread. An implementation should document this behavior.
 *
 * @see ReentrantLock
 * @see Condition
 * @see ReadWriteLock
 *
 * @since 1.5
 * @author Doug Lea
 */
public interface Lock {

    /**
     * Acquires the lock.
     *
     * <p>If the lock is not available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until the
     * lock has been acquired.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>A {@code Lock} implementation may be able to detect erroneous use
     * of the lock, such as an invocation that would cause deadlock, and
     * may throw an (unchecked) exception in such circumstances.  The
     * circumstances and the exception type must be documented by that
     * {@code Lock} implementation.
     */
    void lock();

    /**
     * Acquires the lock unless the current thread is
     * {@linkplain Thread#interrupt interrupted}.
     *
     * <p>Acquires the lock if it is available and returns immediately.
     *
     * <p>If the lock is not available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * one of two things happens:
     *
     * <ul>
     * <li>The lock is acquired by the current thread; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts} the
     * current thread, and interruption of lock acquisition is supported.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while acquiring the
     * lock, and interruption of lock acquisition is supported,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>The ability to interrupt a lock acquisition in some
     * implementations may not be possible, and if possible may be an
     * expensive operation.  The programmer should be aware that this
     * may be the case. An implementation should document when this is
     * the case.
     *
     * <p>An implementation can favor responding to an interrupt over
     * normal method return.
     *
     * <p>A {@code Lock} implementation may be able to detect
     * erroneous use of the lock, such as an invocation that would
     * cause deadlock, and may throw an (unchecked) exception in such
     * circumstances.  The circumstances and the exception type must
     * be documented by that {@code Lock} implementation.
     *
     * @throws InterruptedException if the current thread is
     *         interrupted while acquiring the lock (and interruption
     *         of lock acquisition is supported)
     */
    void lockInterruptibly() throws InterruptedException;

    /**
     * Acquires the lock only if it is free at the time of invocation.
     *
     * <p>Acquires the lock if it is available and returns immediately
     * with the value {@code true}.
     * If the lock is not available then this method will return
     * immediately with the value {@code false}.
     *
     * <p>A typical usage idiom for this method would be:
     *  <pre> {@code
     * Lock lock = ...;
     * if (lock.tryLock()) {
     *   try {
     *     // manipulate protected state
     *   } finally {
     *     lock.unlock();
     *   }
     * } else {
     *   // perform alternative actions
     * }}</pre>
     *
     * This usage ensures that the lock is unlocked if it was acquired, and
     * doesn't try to unlock if the lock was not acquired.
     *
     * @return {@code true} if the lock was acquired and
     *         {@code false} otherwise
     */
    boolean tryLock();

    /**
     * Acquires the lock if it is free within the given waiting time and the
     * current thread has not been {@linkplain Thread#interrupt interrupted}.
     *
     * <p>If the lock is available this method returns immediately
     * with the value {@code true}.
     * If the lock is not available then
     * the current thread becomes disabled for thread scheduling
     * purposes and lies dormant until one of three things happens:
     * <ul>
     * <li>The lock is acquired by the current thread; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts} the
     * current thread, and interruption of lock acquisition is supported; or
     * <li>The specified waiting time elapses
     * </ul>
     *
     * <p>If the lock is acquired then the value {@code true} is returned.
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while acquiring
     * the lock, and interruption of lock acquisition is supported,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * <p>If the specified waiting time elapses then the value {@code false}
     * is returned.
     * If the time is
     * less than or equal to zero, the method will not wait at all.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>The ability to interrupt a lock acquisition in some implementations
     * may not be possible, and if possible may
     * be an expensive operation.
     * The programmer should be aware that this may be the case. An
     * implementation should document when this is the case.
     *
     * <p>An implementation can favor responding to an interrupt over normal
     * method return, or reporting a timeout.
     *
     * <p>A {@code Lock} implementation may be able to detect
     * erroneous use of the lock, such as an invocation that would cause
     * deadlock, and may throw an (unchecked) exception in such circumstances.
     * The circumstances and the exception type must be documented by that
     * {@code Lock} implementation.
     *
     * @param time the maximum time to wait for the lock
     * @param unit the time unit of the {@code time} argument
     * @return {@code true} if the lock was acquired and {@code false}
     *         if the waiting time elapsed before the lock was acquired
     *
     * @throws InterruptedException if the current thread is interrupted
     *         while acquiring the lock (and interruption of lock
     *         acquisition is supported)
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    /**
     * Releases the lock.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>A {@code Lock} implementation will usually impose
     * restrictions on which thread can release a lock (typically only the
     * holder of the lock can release it) and may throw
     * an (unchecked) exception if the restriction is violated.
     * Any restrictions and the exception
     * type must be documented by that {@code Lock} implementation.
     */
    void unlock();

    /**
     * Returns a new {@link Condition} instance that is bound to this
     * {@code Lock} instance.
     *
     * <p>Before waiting on the condition the lock must be held by the
     * current thread.
     * A call to {@link Condition#await()} will atomically release the lock
     * before waiting and re-acquire the lock before the wait returns.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>The exact operation of the {@link Condition} instance depends on
     * the {@code Lock} implementation and must be documented by that
     * implementation.
     *
     * @return A new {@link Condition} instance for this {@code Lock} instance
     * @throws UnsupportedOperationException if this {@code Lock}
     *         implementation does not support conditions
     */
    Condition newCondition();
}
```

Lock 接口使用示例

```java
Lock lock = new ReentrantLock(); //可以使用自己实现的Lock接口实现类，也可以使用 jdk 提供的同步组件
lock.lock(); //不要将锁的获取放在 try 语句块中，因为如果放在try中，若获取锁时发生异常，会执行finally中的释放锁操作，此时没有锁，就会抛出异常

try{

}finally {
    lock.unlock();
}
```

Lock 接口方法
- `lock()` : 获取锁
- `lockInterruptibly() throws InterruptedException`: 若该线程未被中断，则获取锁。Acquires the lock unless the current thread is interrupted.
- `tryLock()`：Acquires the lock only if it is free at the time of invocation.
- `tryLock(long time, TimeUnit unit) throws InterruptedException` : 超时地获取锁
- `unlock()`：释放锁
- `newCondition()`: Returns a new Condition instance that is bound to this Lock instance.

Lock接口与synchronized对比，优点如下
- 可以尝试非阻塞地获取锁 tryLock(): 当前线程尝试获取锁，如果该时刻锁没有被其他线程获取到，就能成功获取并持有锁
- 能被中断地获取锁 lockInterruptibly()：获取到锁的线程能够响应中断，当获取到锁的线程被中断的时候，会抛出中断异常同时释放持有的锁
- 超时地获取锁 tryLock(long time, TimeUnit unit): 在指定时间获取锁，如果没有获取到则返回false

#### Condition

> 参考：https://www.cnblogs.com/gemine/p/9039012.html

在使用 Lock 之前，我们使用的最多的同步方式应该是 synchronized 关键字来实现同步，配合 Object 的 wait() 、 notify() 系列方法可以实现等待/通知模式。

Condition 接口也提供了类似 Object 的监视器方法，与 Lock 配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

Object 和 Condition 接口的一些对比（摘自 Java并发编程的艺术）

![Object与Condition对比](/images/20210223-Object与Condition对比.png)  

##### Condition接口介绍

Condition 对象是依赖于 Lock 对象的，即 Condition 对象需要通过 Lock 对象创建出来（调用 Lock 对象的 newCondition() 方法）。

Condition 对象的使用非常简单，但是需要注意在调用方法前获取锁

```java
public Lock lock2 = new ReentrantLock();
public Condition condition = lock2.newCondition();

public static void main(String[] args) {
    MultiThreadTest multiThreadTest = new MultiThreadTest();
    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(() -> {
        multiThreadTest.conditionWait();
    });
    executorService.execute(() -> {
        multiThreadTest.conditionSignal();
    });
}

private void conditionWait() {
    lock2.lock();
    try {
        System.out.println(Thread.currentThread().getName() + "拿到了锁");
        System.out.println(Thread.currentThread().getName() + "等待信号");
        condition.await();
        System.out.println(Thread.currentThread().getName() + "拿到了信号");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock2.unlock();
    }
}

private void conditionSignal() {
    lock2.lock();
    try {
        Thread.sleep(5000);
        System.out.println(Thread.currentThread().getName() + "拿到了锁");
        condition.signal();
        System.out.println(Thread.currentThread().getName() + "发出了信号");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock2.unlock();
    }
}
```

运行结果

```
pool-1-thread-1拿到了锁
pool-1-thread-1等待信号
pool-1-thread-2拿到了锁
pool-1-thread-2发出了信号
pool-1-thread-1拿到了信号
```

一般会将 Condition 对象作为成员变量，当调用 await() 方法后，当前线程会释放锁并在此等待，而其他线程调用 Condition 对象的 signal() 方法，通知当前线程后，当前线程才从 await() 方法返回，并且在返回前已经获取了锁。

##### Condition接口常用方法

Condition 可以通俗地理解为条件队列，当一个线程在调用了 await() 方法后，直到线程等待的某个条件为真的时候才会被唤醒。这种方式为线程提供了更加简单的等待/通知模式。

Condition 必须配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下，一个 Condition 的实例必须与一个 Lock 实例绑定，因此 Condition 一般都是作为 Lock 的内部实现。

- `void await() throws InterruptedException`: 造成当前线程在接收到信号或者被中断之前一直处于等待状态
- `boolean await(long time, TimeUnit unit) throws InterruptedException`：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态
- `long awaitNanos(long nanosTimeout) throws InterruptedException`：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。返回值表示剩余时间，如果在nanosTimesout之前唤醒，那么返回值 = nanosTimeout - 消耗时间，如果返回值 <= 0 ,则可以认定它已经超时了。
- `void awaitUninterruptibly()`: 造成当前线程在接到信号之前一直处于等待状态。【注意：该方法对中断不敏感】。
-  `boolean awaitUntil(Date deadline) throws InterruptedException`: 造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。如果没有到指定时间就被通知，则返回true，否则表示到了指定时间，返回返回false。
- `void signal()`: 唤醒一个等待线程。该线程从等待方法返回前必须获得与Condition相关的锁。
- `void signalAll()`: 唤醒所有等待线程。能够从等待方法返回的线程必须获得与Condition相关的锁。

##### Condition接口原理解析

ConditionObject 实现了 Condition 接口， 是 AQS (java.util.concurrent.locks.AbstractQueuedSynchronizer) 的内部类。每个 Condition 对象都包含了一个队列（等待队列）。等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。

等待队列的基本结构如下所示。

![Condition等待队列](/images/20210223-Condition等待队列.png)  

等待分为首节点和尾节点。当一个线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列。新增节点就是将尾部节点指向新增的节点。节点引用更新本来就是在获取锁以后的操作，所以不需要CAS保证。同时也是线程安全的操作。

**等待**

当线程调用了await方法以后。线程就作为队列中的一个节点被加入到等待队列中去了。同时会释放锁的拥有。当从await方法返回的时候。一定会获取condition相关联的锁。当等待队列中的节点被唤醒的时候，则唤醒节点的线程开始尝试获取同步状态。如果不是通过 其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException异常信息。

**通知**

调用Condition的signal()方法，将会唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到同步队列中。当前线程加入到等待队列中如图所示：

![当前线程加入等待队列](/images/20210223-当前线程加入等待队列.png)  

在调用signal()方法之前必须先判断是否获取到了锁。接着获取等待队列的首节点，将其移动到同步队列并且利用LockSupport唤醒节点中的线程。节点从等待队列移动到同步队列如下图所示

![节点从等待队列移到同步队列](/images/20210223-节点从等待队列移到同步队列.png)  

被唤醒的线程将从await方法中的while循环中退出。随后加入到同步状态的竞争当中去。成功获取到竞争的线程则会返回到await方法之前的状态。

**总结**

调用await方法后，将当前线程加入Condition等待队列中。当前线程释放锁。否则别的线程就无法拿到锁而发生死锁。自旋(while)挂起，不断检测节点是否在同步队列中了，如果是则尝试获取锁，否则挂起。当线程被signal方法唤醒，被唤醒的线程将从await()方法中的while循环中退出来，然后调用acquireQueued()方法竞争同步状态。

**利用Condition实现生产者消费者模式**

```java
import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedQueue {

    private LinkedList<Object> buffer;    //生产者容器
    private int maxSize ;           //容器最大值是多少
    private Lock lock;
    private Condition fullCondition;
    private Condition notFullCondition;
    BoundedQueue(int maxSize){
        this.maxSize = maxSize;
        buffer = new LinkedList<Object>();
        lock = new ReentrantLock();
        fullCondition = lock.newCondition();
        notFullCondition = lock.newCondition();
    }

    /**
     * 生产者
     * @param obj
     * @throws InterruptedException
     */
    public void put(Object obj) throws InterruptedException {
        lock.lock();    //获取锁
        try {
            while (maxSize == buffer.size()){
                notFullCondition.await();       //满了，添加的线程进入等待状态
            }
            buffer.add(obj);
            fullCondition.signal(); //通知
        } finally {
            lock.unlock();
        }
    }

    /**
     * 消费者
     * @return
     * @throws InterruptedException
     */
    public Object get() throws InterruptedException {
        Object obj;
        lock.lock();
        try {
            while (buffer.size() == 0){ //队列中没有数据了 线程进入等待状态
                fullCondition.await();
            }
            obj = buffer.poll();
            notFullCondition.signal(); //通知
        } finally {
            lock.unlock();
        }
        return obj;
    }

}
```

#### ReentrantLock

可重入锁：当某个线程请求一个被其他线程所持有的锁的时候，该线程会被阻塞（读写锁先不考虑），但是像 synchronized 这样的内置锁是可重入的，即一个线程试图获取一个已被该线程持有的锁，这个请求会成功。

synchronized 锁 和 ReentrantLock 两者都是可重入锁。

重入指的是锁的操作粒度是线程级别而不是调用级别。

ReentrantLock 也是可重入锁，除了支持可重入外，也支持公平和非公平的选择。

ReentrantLock 是 Lock 接口的实现类

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

##### ReentrantLock 实现的可重入性

对于锁的可重入性，需要解决如下两个问题
- 线程再次获取锁的识别问题（锁需要识别出当前要获取锁的线程是否为当前持有锁的线程）
- 锁的释放（同一个线程多次获取同一把锁，那么锁的记录也会不同。一般来说，当同一个线程重复 n 次获取锁之后，只有在之后的释放 n 次锁之后，其他线程才能去竞争这把锁）

ReentrantLock 锁的可重入测试示例

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test {

    Lock lock = new ReentrantLock();

    void m1(){
        try{
            lock.lock(); //加锁
            for (int i = 0; i < 5; i++) {
                TimeUnit.SECONDS.sleep(1);
                System.out.println("m1 method " + i);
            }
            m2(); //在释放锁之前，调用m2方法
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock(); //释放锁
        }
    }

    void m2() {
        lock.lock();
        System.out.println("m2 method");
        lock.unlock();
    }

    public static void main(String[] args) {
        final Test test = new Test();
        new Thread(() -> test.m1()).start();
        new Thread(() -> test.m2()).start();
    }
}
```

ReentrantLock 具有公平锁（FairSync）和非公平锁（NonfairSync）的实现，默认是非公平锁

公平锁：每一次 tryAcquire 都会检查 CLH 队列中是否仍有前驱元素，如果有就继续等待，通过这种方式保证先来先服务（FIFO）的原则

非公平锁：先检查并设置锁的状态，这种方式会出现即使队列中有等待的线程，新的线程仍然会与排队的线程中的对头线程竞争（排队的线程是先来先服务的），所以新的线程可能会在已经在排队的线程前抢到锁，也可能抢不到锁

- 公平锁能保证：老线程和新线程都排队使用锁
- 非公平锁能保证：老的线程排队使用锁，但是无法保证新的线程一定能在排队的线程前抢到锁（新的线程与队列队首的线程公平地抢占锁）。


源码如下

```java

/**
 * Base of synchronization control for this lock. Subclassed
 * into fair and nonfair versions below. Uses AQS state to
 * represent the number of holds on the lock.
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    /**
     * Performs {@link Lock#lock}. The main reason for subclassing
     * is to allow fast path for nonfair version.
     */
    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    /**
     * Reconstitutes the instance from a stream (that is, deserializes it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}

/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync();
}

/**
 * Creates an instance of {@code ReentrantLock} with the
 * given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## 线程通信


### wait & notify

线程间通信可以通过`共享变量 + wait() & notify()` 来实现

wait() 将线程进入阻塞状态

notify() 将线程唤醒

当多线程竞争访问对象的同步方法时，锁对象会关联一个底层的 Monitor 对象（重量级锁的实现）

如下图

![Monitor对象](/images/20210222-Monitor对象.png)  

- Thread-0 先获取到对象的锁，关联到 monitor 的 owner ，同步代码块内调用了锁对象的 wait() 方法，调用后会进入 waitSet 等待。Thread-1 同样如此，此时 Thread-0 的状态为 Waiting
- Thread-2、Thread-3、Thread-4、Thread-5 同时竞争，Thread-2 获取到锁后，关联了 monitor 的 owner，Thread-3、Thread-4、Thread-5 只能进入 EntryList 中等待，此时 Thread-2 线程状态为 Runnable ， Thread-3、Thread-4、Thread-5 的状态为 Blocked
- Thread-2 执行完后，唤醒 EntryList 中的线程，Thread-3、Thread-4、Thread-5 进行竞争锁，获取到锁的线程会关联 monitor 的 owner
- Thread-3、Thread-4、Thread-5 线程在执行过程中，调用了锁对象的 notify() 或 notifyAll() 时，会唤醒 waitSet 的线程，唤醒的线程进入 EntryList 等待重新竞争锁

注意：
- Blocked 状态和 Waiting 状态都是阻塞状态
- Blocked 状态的线程会在 owner 线程释放锁时被唤醒
- wait 和 notify 使用场景必须要有同步，且必须获得对象的锁后才能调用，否则会抛出异常
- wait() 会释放锁，进入 waitSet，wait()方法可指定时间，如果指定时间内未被唤醒，则自动唤醒
- notify() 会随机唤醒一个 waitSet 中的线程
- notifyAll() 会唤醒 waitSet 中所有的线程

示例

```java
static final Object lock1 = new Object();
public static void main(String[] args) {

    new Thread(() -> {
        synchronized (lock1){
            System.out.println("t1开始执行");
            try{
                lock1.wait(); //同步代码块内部才可以调用
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("t1继续执行核心逻辑");
        }
    },"t1").start();

    new Thread(()->{
        synchronized (lock1){
            System.out.println("t2开始执行");
            try{
                lock1.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("t2继续执行核心逻辑");
        }
    }, "t2").start();

    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("开始唤醒");

    synchronized (lock1){
        lock1.notifyAll(); //同步代码块内才能调用
    }
}
```

wait 和 sleep 的区别？
- 相同点：两者都会让线程进入阻塞状态
- 不同点：
  - wait 是 Object 的方法， sleep 是 Thread 的方法
  - wait 会立即释放锁， sleep 不会释放锁
  - wait 后线程的状态是 Waiting， sleep 后线程的状态是 Time_Waiting




实例：一个线程输出 a ，一个线程输出 b ，一个线程输出 c ， abc按照顺序输出，连续输出5次

这里考的是线程的通信，利用 wait()/notify() 和控制变量可以实现，此处使用 ReentrantLock 实现该功能。

```java
public static void main(String[] args) {
    AwaitSignal awaitSignal = new AwaitSignal(5);
    //构造三个条件变量
    Condition a = awaitSignal.newCondition();
    Condition b = awaitSignal.newCondition();
    Condition c = awaitSignal.newCondition();
    //开启三个线程
    new Thread(() -> {
        awaitSignal.print("a", a, b);
    }).start();
    new Thread(() -> {
        awaitSignal.print("b", b, c);
    }).start();
    new Thread(() -> {
        awaitSignal.print("c", c, a);
    }).start();

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    awaitSignal.lock();

    try {
        a.signal();
    }finally {
        awaitSignal.unlock();
    }

}

static class AwaitSignal extends ReentrantLock {
    //循环次数
    private int loopNumber;

    public AwaitSignal(int loopNumber){
        this.loopNumber = loopNumber;
    }

    /**
     *
     * @param print 输出的字符
     * @param current 当前条件变量
     * @param next 下一个条件变量
     */
    public void print(String print, Condition current, Condition next){
        for (int i = 0; i < loopNumber; i++) {
            lock();
            try{
                current.await();
                System.out.println(print);

                next.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                unlock();
            }
        }
    }
}
```


### park & unpark

LockSupport 是 java.util.concurrent 下的工具类，提供了 park 和 unpark 方法，可以实现线程通信

与wait&notify相比，不同点
- wait 和 notify 需要获取对象锁（配合 Object Monitor 使用），而 park 和 unpark 不需要
- unpark 可以指定唤醒线程， notify 是随机唤醒某个线程，notifyAll 是唤醒所有等待的线程
- park 和 unpark 的顺序可以先 unpark ，wait 和 notify 的顺序不能颠倒，必须先 wait

源码定义

```java
/**
 * Disables the current thread for thread scheduling purposes unless the
 * permit is available.
 *
 * <p>If the permit is available then it is consumed and the call
 * returns immediately; otherwise the current thread becomes disabled
 * for thread scheduling purposes and lies dormant until one of three
 * things happens:
 *
 * <ul>
 *
 * <li>Some other thread invokes {@link #unpark unpark} with the
 * current thread as the target; or
 *
 * <li>Some other thread {@linkplain Thread#interrupt interrupts}
 * the current thread; or
 *
 * <li>The call spuriously (that is, for no reason) returns.
 * </ul>
 *
 * <p>This method does <em>not</em> report which of these caused the
 * method to return. Callers should re-check the conditions which caused
 * the thread to park in the first place. Callers may also determine,
 * for example, the interrupt status of the thread upon return.
 */
public static void park() {
    UNSAFE.park(false, 0L);
}


/**
 * Makes available the permit for the given thread, if it
 * was not already available.  If the thread was blocked on
 * {@code park} then it will unblock.  Otherwise, its next call
 * to {@code park} is guaranteed not to block. This operation
 * is not guaranteed to have any effect at all if the given
 * thread has not been started.
 *
 * @param thread the thread to unpark, or {@code null}, in which case
 *        this operation has no effect
 */
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

> 参考：https://blog.csdn.net/e891377/article/details/104551335

park() 将线程挂起

unpark(Thread thread) 恢复某个线程运行

先 park 再 unpark ，代码示例

```java
public static void main(String[] args) {
    Thread t1 = new Thread(()->{
        System.out.println("t1 start");

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("t1 park");
        LockSupport.park(); //t1线程1s后挂起
        System.out.println("t1 resume");
    }, "t1");
    t1.start();

    try {
        Thread.sleep(2000); //主线程睡眠2s
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("unpark");
    LockSupport.unpark(t1);
}
```

先 unpark 再 park 代码示例

```java
public static void main(String[] args) {
    Thread t1 = new Thread(()->{
        System.out.println("t1 start");

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("t1 park");
        LockSupport.park(); //t1线程1s后挂起
        System.out.println("t1 resume");
    }, "t1");
    t1.start();

    try {
        Thread.sleep(1000); //主线程睡眠2s
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("unpark");
    LockSupport.unpark(t1);
}
```


### 生产者消费者模型

**生产者消费者模型**指的是有生产者来生产数据，消费者来消费数据，生产者生产满了就不生产了，通知消费者取，等消费了再进行生产。消费者消费不到了就不消费了，通知生产者生产，生产到了再继续消费。
 
![生产者消费者模型](/images/20210223-生产者消费者模型.png)  

生产者消费者代码示例

```java
public static void main(String[] args) {
    MessageQueue queue = new MessageQueue(2);

    //三个生产者想队列里存值
    for (int i = 0; i < 3; i++) {
        int id = i;
        new Thread(() -> {
            queue.put(new Message(id, "值：" + id));
        }, "生产者" + i).start();
    }

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    //一个消费者不停从队列中取值
    new Thread(()->{
        while (true){
            queue.take();
        }
    }, "消费者").start();
}

//消息队列被生产者和消费者持有
static class MessageQueue {

    private LinkedList<Message> list = new LinkedList<>();

    //容量
    private int capacity;

    public MessageQueue(int capacity) {
        this.capacity = capacity;
    }

    /**
     * 生产者生产
     * @param message
     */
    public void put(Message message){
        synchronized (list){
            while (list.size() == capacity){
                System.out.println("队列已满，生产者等待");
                try{
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            list.addLast(message);
            System.out.printf("生产者消息：{%s}\n", message);
            //生产后通知消费者
            list.notifyAll();
        }
    }

    /**
     * 消费者消费
     * @return
     */
    public Message take(){
        synchronized (list){
            while (list.isEmpty()){
                System.out.println("队列已空，消费者等待");
                try{
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            Message message = list.removeFirst();
            System.out.printf("消费消息：{%s}\n",message);
            list.notifyAll(); //消费后通知生产者
            return message;
        }
    }
}

static class Message{
    private int id;
    private Object value;

    public Message(int id, Object value) {
        this.id = id;
        this.value = value;
    }

    @Override
    public String toString() {
        return "Message{" +
                "id=" + id +
                ", value=" + value +
                '}';
    }
}
```

## 死锁

![死锁图解](/images/20210223-死锁图解.png)  

死锁的代码示例

```java
static class Beer{}
static class Story{}
static Beer beer = new Beer();
static Story story = new Story();
public static void main(String[] args) {
    new Thread(() -> {
        synchronized (beer){
            System.out.println("我有酒，给我故事");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (story){
                System.out.println("小王开始喝酒讲故事");
            }
        }
    },"小王").start();

    new Thread(() -> {
        synchronized (story){
            System.out.println("我有故事，给我酒");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (beer){
                System.out.println("老王开始喝酒讲故事");
            }
        }
    },"老王").start();
}
```

死锁导致程序无法正常运行

```
我有酒，给我故事
我有故事，给我酒
```

可以使用 jdk/bin/jvisualvm.exe 检测死锁原因

![死锁dump信息](/images/20210223-死锁dump信息.png)  


## Java内存模型（JMM）

> 参考:  
> https://www.cnblogs.com/xidongyu/articles/12240022.html  
> https://zhuanlan.zhihu.com/p/29881777  


### 什么是 Java 内存模型（JMM）

`JVM 内存模式`指的是 `JVM 的内存分区`，而 `Java内存模型`是一种虚拟机规范

Java虚拟机规范中定义了 Java内存模型（Java Memory Model, JMM），用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的并发效果，JMM 规范了 Java 虚拟机与计算机内存是如何协同工作的：规定了一个线程如何及何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步地访问共享变量。

原始的Java内存模型存在一些不足，因此Java内存模型在Java1.5时被重新修订。这个版本的Java内存模型在Java8中仍然在使用。

jmm 体现在以下三个方面
- 原子性 保证指令不会受到上下文切换的影响
- 可见性 保证指令不会受到cpu缓存的影响
- 有序性 保证指令不会受并行优化的影响

### Java内存模型和深度剖析volatile

> 参考：https://www.cnblogs.com/dolphin0520/p/3920373.html  

#### 内存模型的相关概念

大家都知道，计算机在执行程序时，每条指令都是在CPU中执行的，而执行指令过程中，势必涉及到数据的读取和写入。由于程序运行过程中的临时数据是存放在主存（物理内存）当中的，这时就存在一个问题，由于CPU执行速度很快，而从内存读取数据和向内存写入数据的过程跟CPU执行指令的速度比起来要慢的多，因此如果任何时候对数据的操作都要通过和内存的交互来进行，会大大降低指令执行的速度。因此在CPU里面就有了高速缓存。

也就是，当程序在运行过程中，会将运算需要的数据从主存复制一份到CPU的高速缓存当中，那么CPU进行计算时就可以直接从它的高速缓存读取数据和向其中写入数据，当运算结束之后，再将高速缓存中的数据刷新到主存当中。举个简单的例子，比如下面的这段代码：

```java
i = i + 1;
```

当线程执行这个语句时，会先从主存当中读取i的值，然后复制一份到高速缓存当中，然后CPU执行指令对i进行加1操作，然后将数据写入高速缓存，最后将高速缓存中i最新的值刷新到主存当中。

这个代码在单线程中运行是没有任何问题的，但是在多线程中运行就会有问题了。在多核CPU中，每条线程可能运行于不同的CPU中，因此每个线程运行时有自己的高速缓存（对单核CPU来说，其实也会出现这种问题，只不过是以线程调度的形式来分别执行的）。本文我们以多核CPU为例。

比如同时有2个线程执行这段代码，假如初始时i的值为0，那么我们希望两个线程执行完之后i的值变为2。但是事实会是这样吗？

可能存在下面一种情况：初始时，两个线程分别读取i的值存入各自所在的CPU的高速缓存当中，然后线程1进行加1操作，然后把i的最新值1写入到内存。此时线程2的高速缓存当中i的值还是0，进行加1操作之后，i的值为1，然后线程2把i的值写入内存。

最终结果i的值是1，而不是2。这就是著名的**缓存一致性问题**。

通常称这种被多个线程访问的变量为**共享变量**。

也就是说，如果一个变量在多个CPU中都存在缓存（一般在多线程编程时才会出现），那么就可能存在缓存不一致的问题。

为了解决缓存不一致性问题，通常来说有以下2种解决方法：
- 通过在总线加LOCK#锁的方式
- 通过缓存一致性协议

这2种方式都是硬件层面上提供的方式。

在早期的CPU当中，是通过在总线上加LOCK#锁的形式来解决缓存不一致的问题。因为CPU和其他部件进行通信都是通过总线来进行的，如果对总线加LOCK#锁的话，也就是说阻塞了其他CPU对其他部件访问（如内存），从而使得只能有一个CPU能使用这个变量的内存。比如上面例子中 如果一个线程在执行 `i = i + 1`，如果在执行这段代码的过程中，在总线上发出了LCOK#锁的信号，那么只有等待这段代码完全执行完毕之后，其他CPU才能从变量i所在的内存读取变量，然后进行相应的操作。这样就解决了缓存不一致的问题。

但是上面的方式会有一个问题，由于在锁住总线期间，其他CPU无法访问内存，导致效率低下。

所以就出现了**缓存一致性协议**。最出名的就是Intel 的MESI协议，MESI协议保证了每个缓存中使用的共享变量的副本是一致的。它核心的思想是：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

![缓存一致性图解](/images/20210223-缓存一致性图解.png)  

#### 并发编程中的三个概念

在并发编程中，我们通常会遇到以下三个问题：原子性问题，可见性问题，有序性问题。我们先看具体看一下这三个概念：

##### 原子性

**原子性**：即一个操作或者多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

一个很经典的例子就是银行账户转账问题：

比如从账户A向账户B转1000元，那么必然包括2个操作：从账户A减去1000元，往账户B加上1000元。

试想一下，如果这2个操作不具备原子性，会造成什么样的后果。假如从账户A减去1000元之后，操作突然中止。然后又从B取出了500元，取出500元之后，再执行 往账户B加上1000元 的操作。这样就会导致账户A虽然减去了1000元，但是账户B没有收到这个转过来的1000元。

所以这2个操作必须要具备原子性才能保证不出现一些意外的问题。

同样地反映到并发编程中会出现什么结果呢？

举个最简单的例子，大家想一下假如为一个32位的变量赋值过程不具备原子性的话，会发生什么后果？

```java
i = 9;
```

假若一个线程执行到这个语句时，我暂且假设为一个32位的变量赋值包括两个过程：为低16位赋值，为高16位赋值。

那么就可能发生一种情况：当将低16位数值写入之后，突然被中断，而此时又有一个线程去读取i的值，那么读取到的就是错误的数据。

##### 可见性

**可见性**：指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

举个简单的例子，看下面这段代码：

```java
//线程1执行的代码
int i = 0;
i = 10;
 
//线程2执行的代码
j = i;
```

假若执行线程1的是CPU1，执行线程2的是CPU2。由上面的分析可知，当线程1执行 i =10这句时，会先把i的初始值加载到CPU1的高速缓存中，然后赋值为10，那么在CPU1的高速缓存当中i的值变为10了，却没有立即写入到主存当中。

此时线程2执行 j = i，它会先去主存读取i的值并加载到CPU2的缓存当中，注意此时内存当中i的值还是0，那么就会使得j的值为0，而不是10.

这就是可见性问题，线程1对变量i修改了之后，线程2没有立即看到线程1修改的值。

##### 有序性

**有序性**：即程序执行的顺序按照代码的先后顺序执行。

```java
int i = 0;              
boolean flag = false;
i = 1;                //语句1  
flag = true;          //语句2
```

上面代码定义了一个int型变量，定义了一个boolean类型变量，然后分别对两个变量进行赋值操作。从代码顺序上看，语句1是在语句2前面的，那么JVM在真正执行这段代码的时候会保证语句1一定会在语句2前面执行吗？不一定，为什么呢？这里可能会发生**指令重排序（Instruction Reorder）**。

下面解释一下什么是指令重排序，一般来说，处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。

比如上面的代码中，语句1和语句2谁先执行对最终的程序结果并没有影响，那么就有可能在执行过程中，语句2先执行而语句1后执行。

但是要注意，虽然处理器会对指令进行重排序，但是它会保证程序最终结果会和代码顺序执行结果相同，那么它靠什么保证的呢？再看下面一个例子：

```java
int a = 10;    //语句1
int r = 2;    //语句2
a = a + 3;    //语句3
r = a*a;     //语句4
```

这段代码有4个语句，那么可能的一个执行顺序是：

![语句执行顺序](/images/20210223-语句执行顺序.png)  

那么可不可能是这个执行顺序呢： 语句2   语句1    语句4   语句3

不可能，因为处理器在进行重排序时是会考虑指令之间的数据依赖性，如果一个指令Instruction 2必须用到Instruction 1的结果，那么处理器会保证Instruction 1会在Instruction 2之前执行。

虽然重排序不会影响单个线程内程序执行的结果，但是多线程呢？下面看一个例子：

```java
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

上面代码中，由于语句1和语句2没有数据依赖性，因此可能会被重排序。假如发生了重排序，在线程1执行过程中先执行语句2，而此是线程2会以为初始化工作已经完成，那么就会跳出while循环，去执行doSomethingwithconfig(context)方法，而此时context并没有被初始化，就会导致程序出错。

从上面可以看出，指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。

也就是说，要想并发程序正确地执行，必须要保证原子性、可见性以及有序性。只要有一个没有被保证，就有可能会导致程序运行不正确。

#### Java 内存模型

在前面谈到了一些关于内存模型以及并发编程中可能会出现的一些问题。下面我们来看一下Java内存模型，研究一下Java内存模型为我们提供了哪些保证以及在java中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

在**Java虚拟机规范**中试图定义一种**Java内存模型（Java Memory Model，JMM）**来屏蔽各个硬件平台和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。那么Java内存模型规定了哪些东西呢，它定义了程序中变量的访问规则，往大一点说是定义了程序执行的次序。注意，为了获得较好的执行性能，Java内存模型并没有限制执行引擎使用处理器的寄存器或者高速缓存来提升指令执行速度，也没有限制编译器对指令进行重排序。也就是说，**在java内存模型中，也会存在缓存一致性问题和指令重排序的问题。**

Java内存模型规定所有的变量都是存在主存当中（类似于前面说的物理内存），每个线程都有自己的工作内存（类似于前面的高速缓存）。线程对变量的所有操作都必须在工作内存中进行，而不能直接对主存进行操作。并且每个线程不能访问其他线程的工作内存。

举个简单的例子：在java中，执行下面这个语句：

```java
i = 10;
```

执行线程必须先在自己的工作线程中对变量i所在的缓存行进行赋值操作，然后再写入主存当中。而不是直接将数值10写入主存当中。

那么Java语言 本身对 原子性、可见性以及有序性提供了哪些保证呢？

##### 原子性

在Java中，对基本数据类型的变量的读取和赋值操作是原子性操作，即这些操作是不可被中断的，要么执行，要么不执行。

上面一句话虽然看起来简单，但是理解起来并不是那么容易。看下面一个例子i：

请分析以下哪些操作是原子性操作：

```java
x = 10;         //语句1
y = x;         //语句2
x++;           //语句3
x = x + 1;     //语句4
```

咋一看，有些朋友可能会说上面的4个语句中的操作都是原子性操作。其实只有语句1是原子性操作，其他三个语句都不是原子性操作。

语句1是直接将数值10赋值给x，也就是说线程执行这个语句的会直接将数值10写入到工作内存中。

语句2实际上包含2个操作，它先要去读取x的值，再将x的值写入工作内存，虽然读取x的值以及 将x的值写入工作内存 这2个操作都是原子性操作，但是合起来就不是原子性操作了。

同样的，x++和 x = x+1包括3个操作：读取x的值，进行加1操作，写入新的值。

 所以上面4个语句只有语句1的操作具备原子性。

也就是说，只有简单的读取、赋值（而且必须是将数字赋值给某个变量，变量之间的相互赋值不是原子操作）才是原子操作。

不过这里有一点需要注意：在32位平台下，对64位数据的读取和赋值是需要通过两个操作来完成的，不能保证其原子性。但是好像在最新的JDK中，JVM已经保证对64位数据的读取和赋值也是原子性操作了。

从上面可以看出，Java内存模型只保证了基本读取和赋值是原子性操作，如果要实现更大范围操作的原子性，可以通过synchronized和Lock来实现。由于synchronized和Lock能够保证任一时刻只有一个线程执行该代码块，那么自然就不存在原子性问题了，从而保证了原子性。

##### 可见性

对于可见性，Java提供了volatile关键字来保证可见性。

当一个共享变量被volatile修饰时，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值。

而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。

另外，通过synchronized和Lock也能够保证可见性，synchronized和Lock能保证同一时刻只有一个线程获取锁然后执行同步代码，并且在释放锁之前会将对变量的修改刷新到主存当中。因此可以保证可见性。

##### 有序性

在Java内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

在Java里面，可以通过volatile关键字来保证一定的“有序性”（具体原理在下一节讲述）。另外可以通过synchronized和Lock来保证有序性，很显然，synchronized和Lock保证每个时刻是有一个线程执行同步代码，相当于是让线程顺序执行同步代码，自然就保证了有序性。

另外，**Java内存模型具备一些先天的“有序性”，即不需要通过任何手段就能够得到保证的有序性，这个通常也称为 happens-before 原则。如果两个操作的执行次序无法从happens-before原则推导出来，那么它们就不能保证它们的有序性，虚拟机可以随意地对它们进行重排序**。

下面就来具体介绍下happens-before原则（先行发生原则）：

- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
- 锁定规则：一个unLock操作先行发生于后面对同一个锁的lock操作
- volatile变量规则：对一个volatile变量的写操作先行发生于后面对这个变量的读操作
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始
　　
这8条原则摘自《深入理解Java虚拟机》。

这8条规则中，前4条规则是比较重要的，后4条规则都是显而易见的。

下面我们来解释一下前4条规则：

对于程序次序规则来说，我的理解就是一段程序代码的执行在单个线程中看起来是有序的。注意，虽然这条规则中提到“书写在前面的操作先行发生于书写在后面的操作”，这个应该是程序看起来执行的顺序是按照代码顺序执行的，因为虚拟机可能会对程序代码进行指令重排序。虽然进行重排序，但是最终执行的结果是与程序顺序执行的结果一致的，它只会对不存在数据依赖性的指令进行重排序。因此，在单个线程中，程序执行看起来是有序执行的，这一点要注意理解。事实上，这个规则是用来保证程序在单线程中执行结果的正确性，但无法保证程序在多线程中执行的正确性。

第二条规则也比较容易理解，也就是说无论在单线程中还是多线程中，同一个锁如果出于被锁定的状态，那么必须先对锁进行了释放操作，后面才能继续进行lock操作。

第三条规则是一条比较重要的规则，也是后文将要重点讲述的内容。直观地解释就是，如果一个线程先去写一个volatile变量，然后一个线程去进行读取，那么写入操作肯定会先行发生于读操作。

第四条规则实际上就是体现happens-before原则具备传递性。

#### 深入剖析volatile关键字

在前面讲述了很多东西，其实都是为讲述volatile关键字作铺垫，那么接下来我们就进入主题。

##### volatile关键字的两层语义

一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：
- 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
- 禁止进行指令重排序。

先看一段代码，假如线程1先执行，线程2后执行：

```java
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}
 
//线程2
stop = true;
```

这段代码是很典型的一段代码，很多人在中断线程时可能都会采用这种标记办法。但是事实上，这段代码会完全运行正确么？即一定会将线程中断么？不一定，也许在大多数时候，这个代码能够把线程中断，但是也有可能会导致无法中断线程（虽然这个可能性很小，但是只要一旦发生这种情况就会造成死循环了）。

下面解释一下这段代码为何有可能导致无法中断线程。在前面已经解释过，每个线程在运行过程中都有自己的工作内存，那么线程1在运行的时候，会将stop变量的值拷贝一份放在自己的工作内存当中。

那么当线程2更改了stop变量的值之后，但是还没来得及写入主存当中，线程2转去做其他事情了，那么线程1由于不知道线程2对stop变量的更改，因此还会一直循环下去。

但是用volatile修饰之后就变得不一样了：
- 使用volatile关键字会强制将修改的值立即写入主存；
- 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）；
- 由于线程1的工作内存中缓存变量stop的缓存行无效，所以线程1再次读取变量stop的值时会去主存读取。

那么在线程2修改stop值时（当然这里包括2个操作，修改线程2工作内存中的值，然后将修改后的值写入内存），会使得线程1的工作内存中缓存变量stop的缓存行无效，然后线程1读取时，发现自己的缓存行无效，它会等待缓存行对应的主存地址被更新之后，然后去对应的主存读取最新的值。

那么线程1读取到的就是最新的正确的值。

##### volatile保证原子性吗？

从上面知道volatile关键字保证了操作的可见性，但是volatile能保证对变量的操作是原子性吗？

下面看一个例子：

```java
public class Test {
    public volatile int inc = 0;
     
    public void increase() {
        inc++;
    }
     
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
         
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

大家想一下这段程序的输出结果是多少？也许有些朋友认为是10000。但是事实上运行它会发现每次运行结果都不一致，都是一个小于10000的数字。

可能有的朋友就会有疑问，不对啊，上面是对变量inc进行自增操作，由于volatile保证了可见性，那么在每个线程中对inc自增完之后，在其他线程中都能看到修改后的值啊，所以有10个线程分别进行了1000次操作，那么最终inc的值应该是1000*10=10000。

这里面就有一个误区了，volatile关键字能保证可见性没有错，但是上面的程序错在没能保证原子性。可见性只能保证每次读取的是最新的值，但是volatile没办法保证对变量的操作的原子性。

在前面已经提到过，自增操作是不具备原子性的，它包括读取变量的原始值、进行加1操作、写入工作内存。那么就是说自增操作的三个子操作可能会分割开执行，就有可能导致下面这种情况出现：

假如某个时刻变量inc的值为10，

线程1对变量进行自增操作，线程1先读取了变量inc的原始值，然后线程1被阻塞了；

然后线程2对变量进行自增操作，线程2也去读取变量inc的原始值，由于线程1只是对变量inc进行读取操作，而没有对变量进行修改操作，所以不会导致线程2的工作内存中缓存变量inc的缓存行无效，所以线程2会直接去主存读取inc的值，发现inc的值时10，然后进行加1操作，并把11写入工作内存，最后写入主存。

然后线程1接着进行加1操作，由于已经读取了inc的值，注意此时在线程1的工作内存中inc的值仍然为10，所以线程1对inc进行加1操作后inc的值为11，然后将11写入工作内存，最后写入主存。

那么两个线程分别进行了一次自增操作后，inc只增加了1。

解释到这里，可能有朋友会有疑问，不对啊，前面不是保证一个变量在修改volatile变量时，会让缓存行无效吗？然后其他线程去读就会读到新的值，对，这个没错。这个就是上面的happens-before规则中的volatile变量规则，但是要注意，线程1对变量进行读取操作之后，被阻塞了的话，并没有对inc值进行修改。然后虽然volatile能保证线程2对变量inc的值读取是从内存中读取的，但是线程1没有进行修改，所以线程2根本就不会看到修改的值。

根源就在这里，自增操作不是原子性操作，而且volatile也无法保证对变量的任何操作都是原子性的。

把上面的代码改成以下任何一种都可以达到效果：

**采用synchronized：**

```java
public class Test {
    public  int inc = 0;
    
    public synchronized void increase() {
        inc++;
    }
    
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

**采用Lock：**

```java
public class Test {
    public  int inc = 0;
    Lock lock = new ReentrantLock();
    
    public  void increase() {
        lock.lock();
        try {
            inc++;
        } finally{
            lock.unlock();
        }
    }
    
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

**采用AtomicInteger：**

```java
public class Test {
    public  AtomicInteger inc = new AtomicInteger();
     
    public  void increase() {
        inc.getAndIncrement();
    }
    
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

在java 1.5的java.util.concurrent.atomic包下提供了一些原子操作类，即对基本数据类型的 自增（加1操作），自减（减1操作）、以及加法操作（加一个数），减法操作（减一个数）进行了封装，保证这些操作是原子性操作。atomic是利用CAS来实现原子性操作的（Compare And Swap），CAS实际上是利用处理器提供的CMPXCHG指令实现的，而处理器执行CMPXCHG指令是一个原子性操作。

##### volatile 能保证有序性吗？

在前面提到volatile关键字能禁止指令重排序，所以volatile能在一定程度上保证有序性。

volatile关键字禁止指令重排序有两层意思：
- 当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；
- 在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

可能上面说的比较绕，举个简单的例子：

```java
//x、y为非volatile变量
//flag为volatile变量
 
x = 2;        //语句1
y = 0;        //语句2
flag = true;  //语句3
x = 4;         //语句4
y = -1;       //语句5
```

由于flag变量为volatile变量，那么在进行指令重排序的过程的时候，不会将语句3放到语句1、语句2前面，也不会讲语句3放到语句4、语句5后面。但是要注意语句1和语句2的顺序、语句4和语句5的顺序是不作任何保证的。

并且volatile关键字能保证，执行到语句3时，语句1和语句2必定是执行完毕了的，且语句1和语句2的执行结果对语句3、语句4、语句5是可见的。

那么我们回到前面举的一个例子：

```java
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

前面举这个例子的时候，提到有可能语句2会在语句1之前执行，那么久可能导致context还没被初始化，而线程2中就使用未初始化的context去进行操作，导致程序出错。

这里如果用volatile关键字对inited变量进行修饰，就不会出现这种问题了，因为当执行到语句2时，必定能保证context已经初始化完毕。

##### volatile的原理和实现机制

前面讲述了volatile关键字的一些使用，下面我们来探讨一下volatile到底如何保证可见性和禁止指令重排序的。

下面这段话摘自《深入理解Java虚拟机》：

“观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个**lock前缀指令**”

lock前缀指令实际上相当于一个**内存屏障**（也成**内存栅栏**），内存屏障会提供3个功能：

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

2. 它会强制将对缓存的修改操作立即写入主存；

3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

##### 使用volatile关键字的场景

synchronized关键字是防止多个线程同时执行一段代码，那么就会很影响程序执行效率，而volatile关键字在某些情况下性能要优于synchronized，但是要注意volatile关键字是无法替代synchronized关键字的，因为volatile关键字无法保证操作的原子性。通常来说，使用volatile必须具备以下2个条件：

1. 对变量的写操作不依赖于当前值

2. 该变量没有包含在具有其他变量的不变式中

实际上，这些条件表明，可以被写入 volatile 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。

事实上，我的理解就是上面的2个条件需要保证操作是原子性操作，才能保证使用volatile关键字的程序在并发时能够正确执行。

下面列举几个Java中使用volatile的几个场景。

**状态标记量**

```java
volatile boolean flag = false;
 
while(!flag){
    doSomething();
}
 
public void setFlag() {
    flag = true;
}
```

```java
volatile boolean inited = false;
//线程1:
context = loadContext();  
inited = true;
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

**double check**

```java
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {
         
    }
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

至于为何需要这么写请参考：
- Java 中的双重检查（Double-Check）: http://blog.csdn.net/dl88250/article/details/5439024
- http://www.iteye.com/topic/652440

### Java中的可见性问题

> 参考：  
> https://www.cnblogs.com/xidongyu/articles/12240022.html

#### 什么是Java内存模型

你已经知道，**导致可见性的原因是缓存，导致有序性的原因是编译优化**，那解决可见性、有序性最直接的办法就是**禁用缓存和编译优化**，但是这样问题虽然解决了，我们程序的性能可就堪忧了。

合理的方案应该是**按需禁用缓存以及编译优化**。那么，如何做到“按需禁用”呢？对于并发程序，何时禁用缓存以及编译优化只有程序员知道，那所谓“按需禁用”其实就是指按照程序员的要求来禁用。所以，为了解决可见性和有序性问题，只需要提供给程序员按需禁用缓存和编译优化的方法即可。

**Java 内存模型**是个很复杂的规范，可以从不同的视角来解读，站在我们这些程序员的视角，本质上可以理解为，**Java 内存模型规范了 JVM 如何提供按需禁用缓存和编译优化的方法。**具体来说，这些方法包括 `volatile`、`synchronized` 和 `final` 三个关键字，以及六项`Happens-Before 规则`，这也正是本期的重点内容。掌握这些方法，我们就可以按需地禁用缓存和编译优化了，也就掌握了Java内存模型最核心的东西。

#### 程序的顺序性

在分析Happens-Before规则之前，一定要搞懂程序的顺序性。程序的顺序性指在编译器以及CPU优化情况下(**编译器&CPU优化会破坏程序执行顺序**)，保证程序以单线程方式执行时，其结果的不变性。

举个例子，在单线中，下面的程序不论重复多少次(先执行writer后执行reader)，最后始终会输出42。这只对单线程程序而言，多线程程序(一个线程执行writer，另一个线程执行reader)中可能会输出0。

```java
class Example {

  int x = 0;
  boolean v = false;
  
  public void writer() {
    this.x = 42;
    this.v = true;
  }

  public void reader() {
    while (true) {
      if (v) {
        print(x);
        return;
      }
    }
  }
}
```

程序的顺序性在任何CPU，任何编程语言中都需遵守的基本规则，因此Java语言或者说Java内存模型天然支持的。因此，Happens-Before规则其实应用于多线程场景，我们在编写多线程程序时一定要考虑到Happens-Before规则的使用。

#### Happens-Before 规则

如何理解 Happens-Before 呢？如果望文生义（很多网文也都爱按字面意思翻译成“先行发生”），那就南辕北辙了，Happens-Before 并不是说前面一个操作发生在后续操作的前面，它真正要表达的是：**前面一个操作的结果对后续操作是可见的**。

就像有心灵感应的两个人，虽然远隔千里，一个人心之所想，另一个人都看得到。Happens-Before 规则就是要保证线程之间的这种“心灵感应”。所以比较正式的说法是：Happens-Before 约束了编译器的优化行为，虽允许编译器优化，但是要求编译器优化后一定遵守 Happens-Before 规则。

Happens-Before 规则应该是 Java 内存模型里面最晦涩的内容了，和程序员相关的规则一共有如下七项，都是关于**可见性**的。

##### volatile 变量规则

这条规则是指对一个 `volatile` 变量的写操作， Happens-Before 于后续对这个 volatile 变量的读操作，即**被volatile修饰的变量写对读是可见的**，确保线程在执行读操作时始终拿到的是新值。同时**volatile禁止重排序功能**，被volatile修饰的变量在进行读写时，这些变量是不能重排序；**volatile变量前后的读写操作不能重排序**

```java
volatile int a;
volatile int b;
volatile int c;
int d;
int e;
int f;

//(1)(2)(3)不能重排序
a=10;     (1)
b=20;     (2)
c=30;     (3)

//将(5)看成一堵障碍，其前后的操作不能跨越障碍
//因此(4)不能滞后于(5)执行，(6)(7)不能先于(5执行)，但(6)(7)可以重排序执行
d=40;     (4)
b=50;     (5)
e=60;     (6)
f=70;     (7)
```

##### 传递性规则

这条规则是指如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。

在解释传递性规则之前先修改开始出现的那段代码，用volatile修饰v变量

```java
class Example {

  int x = 0;
  volatile boolean v = false;
  
  public void writer() {
    this.x = 42;
    this.v = true;
  }

  public void reader() {
    while (true) {
      if (v) {
        print(x);
        return;
      }
    }
  }
}
```

示例代码中的传递性规则

从图中，我们可以看到：

“x=42” Happens-Before 写变量 “v=true” ，这是volatile规则的内容；

写变量“v=true” Happens-Before 读变量 “v=true”，这是volatile规则的内容 。

再根据这个传递性规则，我们得到结果：“x=42” Happens-Before 读变量“v=true”。这意味着什么呢？

如果线程 B 读到了“v=true”，那么线程 A 设置的“x=42”对线程 B 是可见的。也就是说，线程 B 能看到 “x == 42”

##### 管程中锁的规则

这条规则是指对一个锁的解锁 Happens-Before 于后续对这个锁的加锁，且被锁保护的共享资源在进行写操作时对其他线程可见。

要理解这个规则，就首先要了解“管程指的是什么”。**管程**是一种通用的同步原语，在 Java 中指的就是 synchronized，**synchronized 是 Java 里对管程的实现**。

管程中的锁在 Java 里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁，而在代码块执行完会自动释放锁，加锁以及释放锁都是编译器帮我们实现的。

```java
synchronized (this) { //此处自动加锁
  // x是共享变量,初始值=10
  if (this.x < 12) {
  	this.x = 12; 
  }  
} //此处自动解锁
```

所以结合规则 4 --- 管程中锁的规则，可以这样理解：假设 x 的初始值是 10，线程 A 执行完代码块后 x 的值会变成 12（执行完自动释放锁），线程 B 进入代码块时，能够看到线程 A 对 x 的写操作，也就是线程 B 能够看到 x==12。

##### 线程 start() 规则

这条是关于线程启动的。它是指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程在启动子线程 B 前的操作。

换句话说就是，如果线程 A 调用线程 B 的 start() 方法（即在线程 A 中启动线程 B），那么该 start() 操作 Happens-Before 于线程 B 中的任意操作。具体可参考下面示例代码。

```java
Thread B = new Thread(()->{
  // 主线程调用B.start()之前
  // 所有对共享变量的修改，此处皆可见
  // 此例中，var==77

});

// 此处对共享变量var修改
var = 77;

// 主线程启动子线程
B.start();
```

##### 线程 join() 规则

这条是关于线程等待的。它是指主线程 A 等待子线程 B 完成（主线程 A 通过调用子线程 B 的 join() 方法实现），当子线程 B 完成后（主线程 A 中 join() 方法返回），主线程能够看到子线程的操作。当然所谓的“看到”，指的是对共享变量的操作。

换句话说就是，如果在线程 A 中，调用线程 B 的 join() 并成功返回，那么线程 B 中的任意操作 Happens-Before 于该 join() 操作的返回。具体可参考下面示例代码。

```java
Thread B = new Thread(()->{

  // 此处对共享变量var修改
  var = 66;

});

// 例如此处对共享变量修改，
// 则这个修改结果对线程B可见
// 主线程启动子线程

B.start();

B.join()

// 子线程所有对共享变量的修改
// 在主线程调用B.join()之后皆可见
// 此例中，var==66
```
##### 线程中断规则

对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生

```java
//线程A此处对共享变量var修改
var = 66

//中断线程B
B.interrupt()
```
线程B被中断后，线程B看到共享变量var的值应为66

##### final 规则

前面我们讲 volatile 为的是禁用缓存以及编译优化，我们再从另外一个方面来看，有没有办法告诉编译器优化得更好一点呢？这个可以有，就是 final 关键字。

final 修饰变量时，初衷是告诉编译器：这个变量生而不变，可以可劲儿优化。Java 编译器在 1.5 以前的版本的确优化得很努力，以至于都优化错了。但是final使用不当会导致对象“逸出”。

“逸出”有点抽象，我们还是举个例子吧，在下面例子中，在构造函数里面将 this 赋值给了全局变量 global.obj，这就是“逸出”，线程通过 global.obj 读取 x 是有可能读到 0 的。因此我们一定要避免“逸出”。

因此在编程时最好不要在构造函数中把this赋值给一个全局变量。

```java
final int x;

// 错误的构造函数
public FinalFieldExample() { 
  x = 3;
  y = 4;
  // 此处就是讲this逸出，
  global.obj = this;
}
```

##### 总结

Java 的内存模型是并发编程领域的一次重要创新，之后 C++、C#、Golang 等高级语言都开始支持内存模型。Java 内存模型里面，最晦涩的部分就是 Happens-Before 规则了，Happens-Before 规则最初是在一篇叫做 Time, Clocks, and the Ordering of Events in a Distributed System 的论文中提出来的，在这篇论文中，Happens-Before 的语义是一种因果关系。在现实世界里，如果 A 事件是导致 B 事件的起因，那么 A 事件一定是先于（Happens-Before）B 事件发生的，这个就是 Happens-Before 语义的现实理解。

在 Java 语言里面，Happens-Before 的语义本质上是一种可见性，A Happens-Before B 意味着 A 事件对 B 事件来说是可见的，无论 A 事件和 B 事件是否发生在同一个线程里。例如 A 事件发生在线程 1 上，B 事件发生在线程 2 上，Happens-Before 规则保证线程 2 上也能看到 A 事件的发生。

Java 内存模型主要分为两部分，一部分面向你我这种编写并发程序的应用开发人员，另一部分是面向 JVM 的实现人员的，我们可以重点关注前者，也就是和编写并发程序相关的部分，这部分内容的核心就是 Happens-Before 规则。相信经过本章的介绍，你应该对这部分内容已经有了深入的认识。

> 参考：  
> https://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf  
> http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#finalWrong



## CAS

> 参考：  
> https://cloud.tencent.com/developer/article/1462258

在JDK 5之前Java语言是靠synchronized关键字保证同步的，这会导致有锁

锁机制存在以下问题：
- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。
- 一个线程持有锁会导致其它所有需要此锁的线程挂起。
- 如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险。

volatile是不错的机制，但是volatile不能保证原子性。因此对于同步最终还是要回到锁机制上来。

**独占锁**是一种**悲观锁**，`synchronized` 就是一种独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是**乐观锁**。所谓乐观锁就是，**每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止**。乐观锁用到的机制就是**CAS**，Compare and Swap。

CAS, compare and swap的缩写，中文翻译成**比较并交换**。

在计算机科学中，比较和交换（Conmpare And Swap）是用于实现多线程同步的**原子指令**。它将内存位置的内容与给定值进行比较，只有在相同的情况下，将该内存位置的内容修改为新的给定值。这是作为单个**原子操作**完成的。 

原子性保证新值基于最新信息计算; 如果该值在同一时间被另一个线程更新，则写入将失败。操作结果必须说明是否进行替换; 这可以通过一个简单的布尔响应（这个变体通常称为比较和设置），或通过返回从内存位置读取的值来完成。

CAS 操作包含三个操作数： **内存位置（V）**、**预期原值（A）**和**新值(B)**。 如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。

操作步骤如下：
- 比较 A 与 V 是否相等。（比较）
- 如果比较相等，将 B 写入 V。（交换）
- 返回操作是否成功。
- 当多个线程同时对某个资源进行CAS操作，只能有一个线程操作成功，但是并不会阻塞其他线程,其他线程只会收到操作失败的信号。可见 **CAS 其实是一个乐观锁**。

![CAS图解](/images/20210223-CAS图解.png)  

如上图中，主存中保存V值，线程中要使用V值要先从主存中读取V值到线程的工作内存A中，然后计算后变成B值，最后再把B值写回到内存V值中。多个线程共用V值都是如此操作。CAS的核心是在将B值写入到V之前要比较A值和V值是否相同，如果不相同证明此时V值已经被其他线程改变，重新将V值赋给A，并重新计算得到B，如果相同，则将B值赋给V。

**如果不使用CAS机制，看看存在什么问题:**

假如V=1，现在Thread1要对V进行加1，Thread2也要对V进行加1，首先Thread1读取V=1到自己工作内存A中此时A=1，假设Thread2此时也读取V=1到自己的工作内存A中，分别进行加1操作后，两个线程中B的值都为2，此时写回到V中时发现V的值为2，但是两个线程分别对V进行加处理结果却只加了1有问题。

### CAS 实现

本文将对 java.util.concurrent.atomic 包下的原子类 AtomicInteger 中的 compareAndSet 方法进行分析，就能发现最终调用的是 sum.misc.Unsafe 这个类。

看名称 Unsafe 就是一个不安全的类，这个类是利用了 Java 的类和包在可见性的的规则中的一个恰到好处的漏洞。Unsafe 这个类为了速度，在Java的安全标准上做出了一定的妥协。

再往下寻找我们发现 Unsafe的compareAndSwapInt 是 Native 的方法：

```java
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

也就是说，这几个 CAS 的方法应该是使用了本地的方法。所以这几个方法的具体实现需要我们自己去 jdk 的源码中搜索。

最终到搜索 `cmpxchg` 函数

```c
inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {  // 判断是否是多核 CPU
  int mp = os::is_MP();
  __asm {    // 将参数值放入寄存器中
    mov edx, dest    // 注意: dest 是指针类型，这里是把内存地址存入 edx 寄存器中
    mov ecx, exchange_value
    mov eax, compare_value    
    // LOCK_IF_MP
    cmp mp, 0
    /*
     * 如果 mp = 0，表明是线程运行在单核 CPU 环境下。此时 je 会跳转到 L0 标记处，
     * 也就是越过 _emit 0xF0 指令，直接执行 cmpxchg 指令。也就是不在下面的 cmpxchg 指令
     * 前加 lock 前缀。
     */
    je L0    /*
     * 0xF0 是 lock 前缀的机器码，这里没有使用 lock，而是直接使用了机器码的形式。至于这样做的
     * 原因可以参考知乎的一个回答：
     *     https://www.zhihu.com/question/50878124/answer/123099923
     */ 
    _emit 0xF0L0:    /*
     * 比较并交换。简单解释一下下面这条指令，熟悉汇编的朋友可以略过下面的解释:
     *   cmpxchg: 即“比较并交换”指令
     *   dword: 全称是 double word，在 x86/x64 体系中，一个 
     *          word = 2 byte，dword = 4 byte = 32 bit
     *   ptr: 全称是 pointer，与前面的 dword 连起来使用，表明访问的内存单元是一个双字单元
     *   [edx]: [...] 表示一个内存单元，edx 是寄存器，dest 指针值存放在 edx 中。
     *          那么 [edx] 表示内存地址为 dest 的内存单元
     *          
     * 这一条指令的意思就是，将 eax 寄存器中的值（compare_value）与 [edx] 双字内存单元中的值
     * 进行对比，如果相同，则将 ecx 寄存器中的值（exchange_value）存入 [edx] 内存单元中。
     */
    cmpxchg dword ptr [edx], ecx
  }
}
```

总结一下 JAVA 的 cas 是怎么实现的：
- java 的 cas 利用的的是 unsafe 这个类提供的 cas 操作。
- unsafe 的cas 依赖了的是 jvm 针对不同的操作系统实现的 Atomic::cmpxchg
- Atomic::cmpxchg 的实现使用了汇编的 cas 操作，并使用 cpu 硬件提供的 lock信号保证其原子性

### ABA问题

CAS 由三个步骤组成，分别是“读取->比较->写回”。

考虑这样一种情况，线程1和线程2同时执行 CAS 逻辑，两个线程的执行顺序如下：

- 时刻1：线程1执行读取操作，获取原值 A，然后线程被切换走
- 时刻2：线程2执行完成 CAS 操作将原值由 A 修改为 B
- 时刻3：线程2再次执行 CAS 操作，并将原值由 B 修改为 A
- 时刻4：线程1恢复运行，将比较值（compareValue）与原值（oldValue）进行比较，发现两个值相等。
- 然后用新值（newValue）写入内存中，完成 CAS 操作

如上流程，线程1并不知道原值已经被修改过了，在它看来并没什么变化，所以它会继续往下执行流程。

对于 ABA 问题，通常的处理措施是对每一次 CAS 操作设置版本号。java.util.concurrent.atomic 包下提供了一个可处理 ABA 问题的原子类 AtomicStampedReference，具体的实现这里就不分析了，有兴趣的朋友可以自己去看看。

ABA问题的解决办法
- 在变量前面追加版本号：每次变量更新就把版本号加1，则A-B-A就变成1A-2B-3A。
- atomic包下的AtomicStampedReference类：其compareAndSet方法首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用的该标志的值设置为给定的更新值。


### 其他问题

CAS除了ABA问题，仍然存在循环时间长开销大和只能保证一个共享变量的原子操作

1. 循环时间长开销大

    自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。

    如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

2. 只能保证一个共享变量的原子操作

    当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。

    比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

### CAS 的应用

- Java的concurrent包下就有很多类似的实现类，如Atomic开头那些。
- 自旋锁
- 令牌桶限流器

**令牌桶限流器**：就是系统以恒定的速度向桶内增加令牌。每次请求前从令牌桶里面获取令牌。如果获取到令牌就才可以进行访问。当令牌桶内没有令牌的时候，拒绝提供服务。我们来看看 eureka 的限流器是如何使用 CAS 来维护多线程环境下对 token 的增加和分发的。

## 线程池

线程池是java并发最重要的一个知识点，也是难点，是实际应用最广泛的。

线程的资源很宝贵，不可能无限的创建，必须要有管理线程的工具，线程池就是一种管理线程的工具，java开发中经常有池化的思想，如 数据库连接池、Redis连接池等。

预先创建好一些线程，任务提交时直接执行，既可以节约创建线程的时间，又可以控制线程的数量。

**线程池的好处**
- 降低资源消耗，通过池化思想，减少创建线程和销毁线程的消耗，控制资源
- 提高响应速度，任务到达时，无需创建线程即可运行
- 提供更多更强大的功能，可扩展性高

![线程池类图](/images/20210223-线程池类图.png)  

### 线程池的构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
 
}
```

- corePoolSize: 核心线程数
- maximumPoolSize: 最大线程数
- keepAliveTime: 救急线程的空闲时间
- unit: 救急线程的空闲时间单位
- workQueue: 阻塞队列
- threadFactory: 创建线程的工厂，主要定义线程名
- handler: 拒绝策略


### 线程池案例

![线程池案例](/images/20210223-线程池案例.png)  

如上图 银行办理业务。
- 客户到银行时，开启柜台进行办理，柜台相当于线程，客户相当于任务，有两个是常开的柜台，三个是临时柜台。2就是核心线程数，5是最大线程数。即有两个核心线程
- 当柜台开到第二个后，都还在处理业务。客户再来就到排队大厅排队。排队大厅只有三个座位。
- 排队大厅坐满时，再来客户就继续开柜台处理，目前最大有三个临时柜台，也就是三个救急线程
- 此时再来客户，就无法正常为其 提供业务，采用拒绝策略来处理它们
- 当柜台处理完业务，就会从排队大厅取任务，当柜台隔一段空闲时间都取不到任务时，如果当前线程数大于核心线程数时，就会回收线程。即撤销该柜台。

### 线程池的状态

线程池通过一个int变量的高3位来表示线程池的状态，低29位来存储线程池的数量

状态名称 | 高三位 | 接收新任务 | 处理阻塞队列任务 | 说明
Running | 111 | Y | Y | 正常接收任务，正常处理任务
Shutdown | 000 | N | Y | 不会接收任务,会执行完正在执行的任务,也会处理阻塞队列里的任务
Stop | 001 | N | N | 不会接收任务，会中断正在执行的任务,会放弃处理阻塞队列里的任务
Tidying | 010 | N | N | 任务全部执行完毕，当前活动线程是0，即将进入终结
Termitted | 011 | N | N | 终结状态

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

### 线程池的主要流程

**线程池创建、接收任务、执行任务、回收线程的步骤**
- 创建线程池后，线程池的状态是Running，该状态下才能有下面的步骤
- 提交任务时，线程池会创建线程去处理任务
- 当线程池的工作线程数达到corePoolSize时，继续提交任务会进入阻塞队列
- 当阻塞队列装满时，继续提交任务，会创建救急线程来处理
- 当线程池中的工作线程数达到maximumPoolSize时，会执行拒绝策略
- 当线程取任务的时间达到keepAliveTime还没有取到任务，工作线程数大于corePoolSize时，会回收该线程

注意： 不是刚创建的线程是核心线程，后面创建的线程是非核心线程，线程是没有核心非核心的概念的，这是我长期以来的误解。

**拒绝策略**
- 调用者抛出RejectedExecutionException (默认策略)
- 让调用者运行任务
- 丢弃此次任务
- 丢弃阻塞队列中最早的任务，加入该任务

**提交任务的方法**

```java
// 执行Runnable
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
// 提交Callable
public <T> Future<T> submit(Callable<T> task) {
  if (task == null) throw new NullPointerException();
   // 内部构建FutureTask
  RunnableFuture<T> ftask = newTaskFor(task);
  execute(ftask);
  return ftask;
}
// 提交Runnable,指定返回值
public Future<?> submit(Runnable task) {
  if (task == null) throw new NullPointerException();
  // 内部构建FutureTask
  RunnableFuture<Void> ftask = newTaskFor(task, null);
  execute(ftask);
  return ftask;
} 
//  提交Runnable,指定返回值
public <T> Future<T> submit(Runnable task, T result) {
  if (task == null) throw new NullPointerException();
   // 内部构建FutureTask
  RunnableFuture<T> ftask = newTaskFor(task, result);
  execute(ftask);
  return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
}
```

### Execetors创建线程池

注意： 下面几种方式都不推荐使用

1. newFixedThreadPool

    ```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    ```

    - 核心线程数 = 最大线程数 （没有救急线程）
    - 阻塞队列无界 可能导致oom

2. newCachedThreadPool

    ```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    ```

    - 核心线程数是0，最大线程数无限制 ，救急线程60秒回收
    - 队列采用 SynchronousQueue 实现 没有容量，即放入队列后没有线程来取就放不进去
    - 可能导致线程数过多，cpu负担太大

3. newSingleThreadExecutor

    ```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    ```

    - 核心线程数和最大线程数都是1，没有救急线程，无界队列 可以不停的接收任务
    - 将任务串行化 一个个执行， 使用包装类是为了屏蔽修改线程池的一些参数 比如 corePoolSize
    - 如果某线程抛出异常了，会重新创建一个线程继续执行
    - 可能造成oom

4. newScheduledThreadPool

    ```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    ```

    - 任务调度的线程池 可以指定延迟时间调用，可以指定隔一段时间调用

### 线程池的关闭


**shutdown()**：会让线程池状态为shutdown，不能接收任务，但是会将工作线程和阻塞队列里的任务执行完 相当于优雅关闭

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

**shutdownNow()**：会让线程池状态为stop， 不能接收任务，会立即中断执行中的工作线程，并且不会执行阻塞队列里的任务， 会返回阻塞队列的任务列表

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

### 线程池使用的正确姿势

线程池难就难在参数的配置，有一套理论配置参数

cpu密集型 : 指的是程序主要发生cpu的运算
- 核心线程数： CPU核心数+1

IO密集型: 远程调用RPC，操作数据库等，不需要使用cpu进行大量的运算。 大多数应用的场景
- 核心线程数=核数*cpu期望利用率 *总时间/cpu运算时间

但是基于以上理论还是很难去配置，因为cpu运算时间不好估算

实际配置大小可参考下表

~ | cpu密集型 | io密集型
-- | -- | --
线程数数量 | 核数<=x<=核数\*2 | 核心数*50<=x<=核心数 *100
队列长度 | y>=100 | 1<=y<=10

1. 线程池参数通过分布式配置（比如apollo, nacos），修改配置无需重启应用

    线程池参数是根据线上的请求数变化而变化的，最好的方式是 核心线程数、最大线程数 队列大小都是可配置的

    主要配置 corePoolSize maxPoolSize queueSize

    java提供了可方法覆盖参数，线程池内部会处理好参数 进行平滑的修改

    ```java
    public void setCorePoolSize(int corePoolSize) {
    }
    ```

    ![线程池配置流程图](/images/20210223-线程池配置流程图.png)  

2. 增加线程池的监控

3. io密集型可调整为先新增任务到最大线程数后再将任务放到阻塞队列

    代码 主要可重写阻塞队列 加入任务的方法

    ```java
    public boolean offer(Runnable runnable) {
        if (executor == null) {
            throw new RejectedExecutionException("The task queue does not have executor!");
        }

        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int currentPoolThreadSize = executor.getPoolSize();
           
            // 如果提交任务数小于当前创建的线程数, 说明还有空闲线程,
            if (executor.getTaskCount() < currentPoolThreadSize) {
                // 将任务放入队列中，让线程去处理任务
                return super.offer(runnable);
            }
    		// 核心改动
            // 如果当前线程数小于最大线程数，则返回 false ，让线程池去创建新的线程
            if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
                return false;
            }

            // 否则，就将任务放入队列中
            return super.offer(runnable);
        } finally {
            lock.unlock();
        }
    }
    ```

4. 拒绝策略 建议使用tomcat的拒绝策略(给一次机会)

    ```java
    // tomcat的源码
    @Override
    public void execute(Runnable command) {
        if ( executor != null ) {
            try {
                executor.execute(command);
            } catch (RejectedExecutionException rx) {
                // 捕获到异常后 在从队列获取，相当于重试1取不到任务 在执行拒绝任务
                if ( !( (TaskQueue) executor.getQueue()).force(command) ) throw new RejectedExecutionException("Work queue full.");
            }
        } else throw new IllegalStateException("StandardThreadPool not started.");
    }
    ```

    建议修改从队列取任务的方式： 增加超时时间，超时1分钟取不到在进行返回

    ```java
    public boolean offer(E e, long timeout, TimeUnit unit){}
    ```
