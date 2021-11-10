## 概述

Java 程序天生就是**多线程程序**，我们可以通过 [JMX](https://orzlinux.cn/blog/Java_rmi_jmx20210910.html) 来看一下一个普通的 Java 程序有哪些线程，代码如下。

```java
public class MultiThread {
    public static void main(String[] args) {
        // 获取Java线程管理MXBean
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 不需要获取同步的 monitor 和 synchronizer 信息，仅获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        for (ThreadInfo threadInfo:threadInfos) {
            System.out.println("[" + threadInfo.getThreadId() + "]" + threadInfo.getThreadName());
        }
    }
}

```

输出：

```java
[6]Monitor Ctrl-Break
[5] Attach Listener //添加事件
[4] Signal Dispatcher // 分发处理给 JVM 信号的线程
[3] Finalizer //调用对象 finalize 方法的线程
[2] Reference Handler //清除 reference 线程
[1] main //main 线程,程序入口
```

**一个 Java 程序的运行是 main 线程和多个其他线程同时运行**。

> 一个进程中可以有多个线程，多个线程共享进程的**堆**和**方法区 (JDK1.8 之后的元空间)**资源，但是每个线程有自己的**程序计数器**、**虚拟机栈** 和 **本地方法栈**。
>
> **总结：** 线程是进程划分成的更小的运行单位。线程和进程最大的不同在于基本上各进程是独立的，而各线程则不一定，因为同一进程中的线程极有可能会相互影响。线程执行开销小，但不利于资源的管理和保护；而进程正相反。

一个 Native Method 就是一个 Java 调用非 Java 代码的接口。一个 Native Method 是这样一个 Java 的方法：该方法的实现由非 Java 语言实现，比如 C。

 **虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

![image-20210911115106164](https://gitee.com/hqinglau/img/raw/master/img/20210911115126.png)

`java.lang.Thread.State`枚举类中定义了六种线程的状态，可以调用线程Thread中的`getState()`方法**获取当前线程的状态**。

![image-20210911144057552](https://gitee.com/hqinglau/img/raw/master/img/20210911144059.png)

> 原图中 wait 到 runnable 状态的转换中，`join`实际上是`Thread`类的方法，但这里写成了`Object`。

线程在执行过程中会有自己的运行条件和状态（也称**上下文**），比如程序计数器，栈信息等。

**死锁**例子：

```java
public class MultiThread {
    private static Object resource1 = new Object();
    private static Object resource2 = new Object();

    public static void main(String[] args) {
        new Thread(()-> {
            synchronized (resource1) {
                System.out.println(Thread.currentThread()+"get resource1");
                try {
                    Thread.sleep(1_000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread()+"waiting get resource2");
                synchronized (resource2) {
                    System.out.println(Thread.currentThread()+"get resource2");
                }
            }
        },"线程1").start();
        new Thread(()-> {
            synchronized (resource2) {
                System.out.println(Thread.currentThread()+"get resource2");
                try {
                    Thread.sleep(1_000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread()+"waiting get resource1");
                synchronized (resource1) {
                    System.out.println(Thread.currentThread()+"get resource1");
                }
            }
        },"线程2").start();
    }
}

/*
输出：
Thread[线程1,5,main]get resource1
Thread[线程2,5,main]get resource2
Thread[线程2,5,main]waiting get resource1
Thread[线程1,5,main]waiting get resource2
 */
```

**死锁的必备条件**：

资源锁、获取资源阻塞时不释放、不被抢、循环等待资源。

**`sleep()` 方法和 `wait()` 方法区别和共同点**：

- 两者最主要的区别在于：`sleep()` 方法没有释放锁，而 `wait()` 方法释放了锁。
- 两者都可以暂停线程的执行。
- `wait()` 通常被用于线程间交互/通信，`sleep() `通常被用于暂停执行。
- `wait()` 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 `notify() `或者 `notifyAll()` 方法。`sleep() `方法执行完成后，线程会自动苏醒。或者可以使用 `wait(long timeout)` 超时后线程会自动苏醒。

## `synchronized`

**`synchronized` 关键字解决的是多个线程之间访问资源的同步性，`synchronized`关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。**

另外，在 Java 早期版本中，`synchronized` 属于 **重量级锁**，效率低下。

因为监视器锁（monitor）是依赖于底层的操作系统的 `Mutex Lock` 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高。

庆幸的是在 Java 6 之后 Java 官方对从 JVM 层面对 `synchronized` 较大优化，所以现在的 `synchronized` 锁效率也优化得很不错了。JDK1.6 对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

所以，你会发现目前的话，不论是各种开源框架还是 JDK 源码都大量使用了 `synchronized` 关键字。

### **三种使用方式**

```java
// 1 修饰实例方法，获取当前对象实例的锁
synchronized void method() {
    // code
}

// 2 修饰静态方法, 获得当前class的锁，所以1和2两个方法不会冲突，不是一个锁
synchronized static void method() {
    // code
}

// 3 修饰代码块，获取指定对象的锁
synchronized(this) {
    // code
}
// 尽量不要使用 synchronized(String a) 因为 JVM 中，字符串常量池具有缓存功能！
```

### `volatile`

**可见性，是指线程之间的可见性，一个线程修改的状态对另一个线程是可见的。**也就是一个线程修改的结果。另一个线程马上就能看到。比如：用volatile修饰的变量，就会具有可见性。volatile修饰的变量不允许线程内部缓存和重排序，即直接修改内存。所以对其他线程是可见的。

### 双重校验锁实现单例模式

```java
public class Singleton {
    private volatile static Singleton uniqueInstance;
    private Singleton() {}

    public static Singleton getUniqueInstance() {
        // 基本判断不加锁，提升效率
        // 不会出现就算是有实例了也要加锁情况
        if (uniqueInstance == null) {
            // 加锁是因为此时可能有线程初始化好了实例
            synchronized (Singleton.class) {
                if(uniqueInstance == null) {
                    // volatile 防止指令重排序
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

**构造方法本身就属于线程安全的**，不存在同步的构造方法一说。

### `Synchronized`原理

```java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized 代码块");
        }
    }
}
```

通过 JDK 自带的 `javap` 命令查看 `SynchronizedDemo` 类的相关字节码信息：

```shell
$ javap -c -s -v -l SynchronizedDemo.class
Classfile .../SynchronizedDemo.class
  Last modified 2021年9月11日; size 546 bytes
  MD5 checksum a4e4607df26d89d05a469490ff762e94
  Compiled from "SynchronizedDemo.java"
    ...
  public void method();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  # 指向同步代码块的开始位置
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String synchronized code block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit   # 指向同步代码块的结束位置
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 5: 0
        line 6: 4
        line 7: 12
        line 8: 22
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class com/lhq/SynchronizedDemo, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "SynchronizedDemo.java"
```

当执行 `monitorenter` 指令时，线程试图获取锁也就是获取 **对象监视器 `monitor`** 的持有权。`wait/notify`等方法也依赖于`monitor`对象，这就是为什么只有在同步的块或者方法中才能调用`wait/notify`等方法，否则会抛出`java.lang.IllegalMonitorStateException`的异常的原因。

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

**两者的本质都是对对象监视器 monitor 的获取。**

### `ReentrantLock`

**可重入锁**

重入锁实现可重入性原理或机制是：每一个锁关联一个线程持有者和计数器，当计数器为 0 时表示该锁没有被任何线程持有，那么任何线程都可能获得该锁而调用相应的方法；当某一线程请求成功后，JVM 会记下锁的持有线程，并且将计数器置为 1；此时其它线程请求该锁，则必须等待；而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增；当线程退出同步代码块时，计数器会递减，如果计数器为 0，则释放该锁。

### `ReentrantLock`与`synchronized`的比较

#### 相似点

它们都是加锁方式同步，而且都是阻塞式的同步，也就是说当如果一个线程获得了对象锁，进入了同步块，其他访问该同步块的线程都必须阻塞在同步块外面等待，等到释放掉锁或者唤醒后才能继续获得锁。

**都是可重入锁**。

#### 区别

对于Synchronized来说，它是java语言的关键字，是原生语法层面的互斥，需要jvm实现。而ReentrantLock它是JDK 1.5之后提供的API层面的互斥锁，需要lock()和unlock()方法配合try/finally语句块来完成

便利性：Synchronized的使用比较方便简洁，并且由编译器去保证锁的加锁和释放，而ReenTrantLock需要手工声明来加锁和释放锁，为了避免忘记手工释放锁造成死锁，所以最好在finally中声明释放锁。

虚拟机团队在 JDK1.6 为 `synchronized` 关键字进行了很多优化，但是这些优化都是在虚拟机层面实现的，并没有直接暴露给我们。

相比`synchronized`，`ReentrantLock`增加了一些高级功能。主要来说主要有三点：

- **等待可中断** : `ReentrantLock`提供了一种能够中断等待锁的线程的机制，通过 `lock.lockInterruptibly()` 来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。

`lock` 优先考虑获取锁，待获取锁成功后，才响应中断。
`lockInterruptibly` 优先考虑响应中断，而不是响应锁的普通获取或重入获取。

> **`ReentrantLock.lockInterruptibly`允许在等待时，由其它线程调用等待线程的`Thread.interrupt`方法来中断等待线程的等待而直接返回，这时不用获取锁，而会抛出一个`InterruptedException`。** `ReentrantLock.lock`方法不允许`Thread.interrupt`中断，即使检测到`Thread.isInterrupted`，一样会继续尝试获取锁，失败则继续休眠。只是在最后获取锁成功后再把当前线程置为`interrupted`状态,然后再中断线程。

```java
public class SynchronizedDemo {
    public static void main(String[] args) throws InterruptedException {
        final Lock lock = new ReentrantLock();
        lock.lock();
        Thread.sleep(1000);
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //lock.lock();  // 即使执行了interrupt()方法也没有反应
                try {
                    lock.lockInterruptibly();  
                    // 输出 Thread-0 interrupted.抛出异常
                    // 见代码最后
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" interrupted.");
            }
        });
        t1.start();
        Thread.sleep(1000);
        t1.interrupt();
        Thread.sleep(1000);
    }
}
/*
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.lhq.SynchronizedDemo$1.run(SynchronizedDemo.java:18)
	at java.lang.Thread.run(Thread.java:748)
 */
```

- **可实现公平锁** : `ReentrantLock`可以指定是公平锁还是非公平锁。而`synchronized`只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。`ReentrantLock`默认情况是非公平的，可以通过 `ReentrantLock`类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
- **可实现选择性通知（锁可以绑定多个条件）**: `synchronized`关键字与`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。`ReentrantLock`类当然也可以实现，但是需要借助于`Condition`接口与`newCondition()`方法。

公平锁在公平性得以保障，但因为公平的获取锁没有考虑到操作系统对线程的调度因素以及其他因素，会影响性能。非公平锁：饥饿问题。



## 参考

[并发基础常见面试题总结](https://snailclimb.gitee.io/javaguide/#/docs/java/multi-thread/2020%E6%9C%80%E6%96%B0Java%E5%B9%B6%E5%8F%91%E5%9F%BA%E7%A1%80%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93)

[什么是Native方法](https://www.jianshu.com/p/22517a150fe5)

[Java：线程的六种状态及转化](https://www.cnblogs.com/summerday152/p/12288671.html)

[Java中Volatile关键字详解](https://www.cnblogs.com/zhengbin/p/5654805.html)

[RMI和JMX](https://orzlinux.cn/blog/Java_rmi_jmx20210910.html)

[ReentrantLock详解](https://blog.csdn.net/SunStaday/article/details/107451530)

[lock()与lockInterruptibly()的区别](https://blog.csdn.net/yyd19921214/article/details/49737061?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link)