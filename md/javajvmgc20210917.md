## JVM 内存分配和回收

概述见《[Java虚拟机概述](https://orzlinux.cn/blog/jvm20210916.html)》**分代**一节。

Hotspot 遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了 `survivor` 区的一半时，取这个年龄和 `MaxTenuringThreshold` 中更小的一个值，作为新的晋升年龄阈值。

![image-20210917111244752](https://gitee.com/hqinglau/img/raw/master/img/20210917111244.png)

**示例**

参数配置：

![image-20210917110924993](https://gitee.com/hqinglau/img/raw/master/img/20210917110925.png)

详见注释：

```java
public class GCTest {
    public static void main(String[] args) {
        // 空main eden space 40960K,  10% used
        // from space 5120K,   0% used
        //  to  space 5120K,   0% used
        byte[] byte1m_1 = new byte[1 * 1024 * 1024];
        byte[] byte1m_2 = new byte[1 * 1024 * 1024];
        byte[] byte1m_3 = new byte[1 * 1024 * 1024];

        // eden又多了3M

        makeMyGarbage(33);
        // eden又多了33M

        byte[] byteArr = new byte[10*1024*1024];

        // eden放不下了，触发GC
        // 一次GC之后，前面三个数组引用还活着，进入from survivor区
        // 但是from区只有5M，这里占了3M，到了60%，重新计算对象晋升的age，1
        // 新的数组进入eden, eden 10M

        // 2021-09-17T10:47:43.971+0800: [GC (Allocation Failure) 2021-09-17T10:47:43.971+0800: [ParNew
        //Desired survivor size 3145728 bytes, new threshold 1 (max 3)
        //- age   1:    3931960 bytes,    3931960 total
        //: 40168K->3852K(46080K), 0.0021462 secs] 40168K->3852K(199680K), 0.0022156 secs]
        //   [Times: user=0.00 sys=0.00, real=0.00 secs]

        makeMyGarbage(33);

        // 最前面三个数组age->2, 进入老年代
        // byteArr放survivor放不下，直接进入老年代
        // Heap
        // par new generation   total 46080K, used 34611K


        // 2021-09-17T10:48:32.848+0800: [GC (Allocation Failure) 2021-09-17T10:48:32.848+0800: [ParNew
        //Desired survivor size 3145728 bytes, new threshold 3 (max 3)
        //: 14912K->0K(46080K), 0.0070790 secs] 14912K->14055K(199680K), 0.0071117 secs] [Times:
        //   user=0.06 sys=0.00, real=0.01 secs]
    }

    private static void makeMyGarbage(int size) {
        byte[] byteArrTemp = new byte[size * 1024 * 1024];
    }
}
```

针对 HotSpot VM 的实现，它里面的 **GC** 其实准确分类只有两大种：

部分收集 (**Partial GC**)：

- 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
- 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。

整堆收集 (**Full GC**)：收集整个 Java 堆和方法区。

**空间分配担保**是为了确保在 Minor GC 之前老年代本身还有容纳新生代所有对象的剩余空间。JDK 6 Update 24之后的规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Minor GC，否则将进行 Full GC。

## 判断对象死亡

### 引用计数

简单高效，难解决循环引用问题。（两个对象相互引用，但是除此之外是废弃的）。

### 可达性分析

![image-20210917123213717](https://gitee.com/hqinglau/img/raw/master/img/20210917123213.png)

可作为 GC Roots 的对象包括下面几种:

- 虚拟机栈(栈帧中的本地变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象

### 引用分类

**强引用**

**软引用**：可用来实现内存敏感的高速缓存。内存够就不回收，不够就回收它。

**弱引用**：GC 发现它就回收

**虚引用**：如同没有引用，任何时候可回收。主要用来跟踪对象被垃圾回收的活动。必须和引用队列联合使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

### 不可达对象处死

第一次标记并筛选，看是否有必要执行 finalize 方法，需要执行的话放在队列进行第二次标记，除非和其它的对象建立关联，否则真回收。

### 废弃常量

运行时常量池主要回收的是废弃的常量。

**JDK1.8 hotspot 移除了永久代用元空间(Metaspace)取而代之, 这时候字符串常量池还在堆, 运行时常量池还在方法区, 只不过方法区的实现从永久代变成了元空间(Metaspace)**

### 无用类判别

类需要同时满足下面 3 个条件才能算是 **无用的类** ：

- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
- 加载该类的 `ClassLoader` 已经被回收。
- 该类对应的 `java.lang.Class` 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是可以，而并不是和对象一样不使用了就会必然被回收。

## 垃圾收集算法

### 标记-清除算法

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210917131358.png" alt="image-20210917131358891" style="zoom:80%;" />

标记所以不需要回收的，然后统一回收。效率问题、空间 碎片问题。

### 标记-复制算法

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210917131534.png" alt="image-20210917131534797" style="zoom:80%;" />

活着的移动到另一半。

### 标记-整理算法

![image-20210917131634179](https://gitee.com/hqinglau/img/raw/master/img/20210917131634.png)

### 分代收集算法

每个年代可以采用合适的垃圾收集算法。

比如在新生代中，每次收集都会有大量对象死去，所以可以选择**标记-复制算法**，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择**标记-清除**或**标记-整理**算法进行垃圾收集。

## 垃圾收集器

**Serial收集器**：最基本、历史最悠久，单线程收集器。**新生代采用标记-复制算法，老年代采用标记-整理算法。**

**ParNew 收集器**：其实就是 Serial 收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和 Serial 收集器完全一样。

**Parallel Scavenge 收集器**：关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。 Parallel Scavenge 收集器提供了很多参数供用户找到最合适的停顿时间或最大吞吐量，如果对于收集器运作不太了解，手工优化存在困难的时候，使用 Parallel Scavenge 收集器配合自适应调节策略，把内存管理优化交给虚拟机去完成也是一个不错的选择。**这是 JDK1.8 默认收集器**

**Serial Old 收集器**：老年代版本

Parallel Old：

**CMS（Concurrent Mark Sweep）收集器**：是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

![image-20210917132842873](https://gitee.com/hqinglau/img/raw/master/img/20210917132842.png)

整个过程分为四个步骤：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短
- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

主要优点：**并发收集、低停顿**。但是它有下面三个明显的缺点：

- 对 CPU 资源敏感；
- 无法处理浮动垃圾；
- 它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。

**G1 收集器**：Garbage first。G1 的主要关注点在于达到可控的停顿时间，在这个基础上尽可能提高吞吐量，这一点非常重要。

G1 被设计用来长期取代 CMS 收集器，和 CMS 相同的地方在于，它们都属于并发收集器，在大部分的收集阶段都不需要挂起应用程序。区别在于，G1 没有 CMS 的碎片化问题（或者说不那么严重），同时提供了更加可控的停顿时间。

如果你的应用使用了较大的堆（如 6GB 及以上）而且还要求有较低的垃圾收集停顿时间（如 0.5 秒），那么 G1 是不错的选择。

之前介绍的分代收集器将整个堆分为年轻代、老年代和永久代，每个代的空间是确定的。

**而 G1 将整个堆划分为一个个大小相等的小块（每一块称为一个 region），每一块的内存是连续的**。和分代算法一样，G1 中每个块也会充当 Eden、Survivor、Old 三种角色，但是它们不是固定的，这使得内存使用更加地灵活。

![image-20210917133543717](https://gitee.com/hqinglau/img/raw/master/img/20210917133543.png)

 G1 不是一个实时收集器，它会尽力满足我们的停顿时间要求，但也不是绝对的，它基于之前垃圾收集的数据统计，估计出在用户指定的停顿时间内能收集多少个区块。



## PS

IDEA设置运行时 Java 参数：

![image-20210917103503254](https://gitee.com/hqinglau/img/raw/master/img/20210917103510.png)

## 参考

[Intellij IDEA设置运行时Java参数](https://www.cnblogs.com/liugh/p/7645135.html)

[JVM 垃圾回收](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/JVM垃圾回收?id=jvm-垃圾回收)

[Java虚拟机概述](https://orzlinux.cn/blog/jvm20210916.html)

[Major GC和Full GC的区别是什么？触发条件呢？ - RednaxelaFX的回答 - 知乎]( https://www.zhihu.com/question/41922036/answer/93079526)

[可达性算法中不可达的对象是否一定会死亡（不一定）](https://blog.csdn.net/qq_41999455/article/details/102608447)

[大白话讲解Jvm的G1垃圾收集器](https://www.itqiankun.com/article/jvm-g1-memory-management-model)