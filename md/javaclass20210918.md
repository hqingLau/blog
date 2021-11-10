## JDK 监控和故障处理工具

### 命令行工具

>  混个眼熟就行了，有可视化工具这些一般不用，用的时候一堆指令也记不住，还得再查。

#### `jps`：查看所有 Java 进程

JVM process status，类似于 UNIX 的 `ps`。

```shell
$ jps
10492 RemoteMavenServer36
11340 Launcher
7340
8252 Jps
```

参数

```shell
# 列出详细的主类全名，如果是 Jar 包，输出 Jar 路径
$ jps -l
12152 jdk.jcmd/sun.tools.jps.Jps
10492 org.jetbrains.idea.maven.server.RemoteMavenServer36
11340 org.jetbrains.jps.cmdline.Launcher
7340

# 虚拟机启动时 JVM 参数
$ jps -v
10492 RemoteMavenServer36 -Djava.awt.headless=true -Dmaven.defaultProjectBuilder.disableGlobalModelCache=true -Xmx768m -Didea.maven.embedder.version=3.6.1 -Dmaven.ext.class.path=C:\Program Files\JetBrains\IntelliJ IDEA 2019.3.1\plugins\maven\lib\maven-event-listener.jar -Dfile.encoding=GBK
...

# 传递给 Java 进程 main() 函数的参数
$ jps -m
11340 Launcher C:/Program Files/JetBrains/IntelliJ IDEA 2019.3.1/plugins/java/lib/javac2.jar;C:/Program Files/JetBrains/IntelliJ IDEA 2019.3.1/plugins/java/lib/aether-api-1.1.0.jar;
...
```

#### `jstat`：监视虚拟机运行状态信息

JVM statistics monitoring tool

```shell
# 分析进程 id 为 31736 的 gc 情况，每隔 1000ms 打印一次记录，打印 10 次停止，每 3 行后打印指标头部。
$ jstat -gc -h3 9904 1000 3
```

#### `jinfo`: 实时地查看和调整虚拟机各项参数

```shell
$ jinfo 10492
Java System Properties:
#Sat Sep 18 17:51:59 CST 2021
sun.desktop=windows
awt.toolkit=sun.awt.windows.WToolkit
java.specification.version=11
sun.cpu.isalist=amd64
sun.jnu.encoding=GBK
java.class.path=C\:\\Progr...
```

使用 `jinfo` 可以在不重启虚拟机的情况下，可以动态的修改 JVM  的参数。尤其在线上的环境特别有用。

#### `jmap`：生成堆转储快照

并不仅仅是为了获取 dump 文件，它还可以查询 finalizer 执行队列、Java 堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。

#### `jhat`: 分析 heapdump 文件

**`jhat`** 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果。

#### `jstack` :生成虚拟机当前时刻的线程快照

`jstack`（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。

### JDK 可视化工具分析

#### JConsole 

JConsole 是基于 JMX 的可视化监视、管理工具。可以很方便的监视本地及远程服务器的 Java 进程的内存使用情况。可以在控制台输出`console`命令启动或者在 JDK 目录下的 bin 目录找到`jconsole.exe`然后双击启动。

![image-20210918175757128](https://gitee.com/hqinglau/img/raw/master/img/20210918175759.png)

继续看一下`RemoteMavenServer36`：

![image-20210918180308161](https://gitee.com/hqinglau/img/raw/master/img/20210918180311.png)

#### Visual VM

![image-20210918180546206](https://gitee.com/hqinglau/img/raw/master/img/20210918180547.png)

## class文件结构

```java
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
// （常量池计数器是从 1 开始计数的，将第 0 项常量空出来是有特殊考虑
// 的，索引值为 0 代表“不引用任何一个常量池项”）。
    
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
// 这个标志用于识别一些类或者接口层次的访问信息，包括：这个 Class 是类
// 还是接口，是否为 public 或者 abstract 类型，如果是类的话是否声明为 final 等等。
    
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    
    field_info     fields[fields_count];//一个类会可以有多个字段
// 字段表（field info）用于描述接口或类中声明的变量。字段包括类级变量
// 以及实例变量，但不包括在方法内部声明的局部变量。
    
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
// 如：max_stack代表了操作数栈深度的最大值，虚拟机会根据这个值来分配栈帧中的操栈深度，
// max_locals代表了局部变量所需的存储空间，
}
```

**常量池**主要存放两大常量：字面量和符号引用。字面量比较接近于 Java 语言层面的的常量概念，如文本字符串、声明为 final 的常量值等。而符号引用则属于编译原理方面的概念。包括下面三类常量：

- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符

使用 `jclasslib` 不光可以直观地查看某个类对应的字节码文件，还可以查看类的基本信息、常量池、接口、属性、函数等信息。

![image-20210918181813648](https://gitee.com/hqinglau/img/raw/master/img/20210918181815.png)

## 类的加载过程

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210918182659.png" alt="image-20210918182657583" style="zoom:80%;" />

在加载阶段，虚拟机需要完成以下三件事情：

- 通过一个类的全限定名来获取其定义的二进制字节流。

- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。

- 在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

**方法区**

保存在着被加载过的每一个类的信息；这些信息由类加载器在加载类的时候，从类的源文件中抽取出来；static变量信息也保存在方法区中；

**可以看做是将类（Class）的元数据，保存在方法区里；**

方法区是线程共享的；当有多个线程都用到一个类的时候，而这个类还未被加载，则应该只有一个线程去加载类，让其他线程等待；

方法区的大小不必是固定的，jvm可以根据应用的需要动态调整。jvm也可以允许用户和程序指定方法区的初始大小，最小和最大限制；

方法区同样存在垃圾收集，因为通过用户定义的类加载器可以动态扩展Java程序，这样可能会导致一些类，不再被使用，变为垃圾。这时候需要进行垃圾清理。

**类型的常量池 （即运行时常量池）**

每一个Class文件中，都维护着一个常量池（这个保存在类文件里面，不要与方法区的运行时常量池搞混），里面存放着编译时期生成的各种字面值和符号引用；这个常量池的内容，在类加载的时候，被复制到方法区的运行时常量池 ；

字面值：就是像string, 基本数据类型，以及它们的包装类的值，以及final修饰的变量，简单说就是在编译期间，就可以确定下来的值；

符号引用：不同于我们常说的引用，它们是对类型，域和方法的引用，类似于面向过程语言使用的前期绑定，对方法调用产生的引用；

存在这里面的数据，类似于保存在数组中，外部根据索引来获得它们 。

## 参考

[JDK 监控和故障处理工具总结](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/JDK监控和故障处理工具总结?id=jdk-监控和故障处理工具总结)

[类文件结构](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/类文件结构?id=类文件结构)

[【Java基础】类加载过程](https://www.jianshu.com/p/dd39654231e0)

[回顾一下类加载过程](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/类加载器?id=回顾一下类加载过程)

[IDEA字节码学习查看神器jclasslib bytecode viewer介绍](https://www.cnblogs.com/tangliMeiMei/p/13033572.html)

[方法区（关于java虚拟机内存的那些事）](https://blog.csdn.net/youngyouth/article/details/79933612)