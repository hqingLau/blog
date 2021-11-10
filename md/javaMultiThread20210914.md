## `This` 逃逸

只有当构造函数返回时，`this`引用才应该从线程中逸出。构造函数可以将`this`引用保存到某个地方，只要其他线程不会在构造函数完成之前使用它。

## `Future`

Future代表的是异步执行的结果，意思是当异步执行结束之后，返回的结果将会保存在Future中。

一般来说，当我们执行一个长时间运行的任务时，使用Future就可以让我们暂时去处理其他的任务，等长任务执行完毕再返回其结果。

经常会使用到Future的场景有：1. 计算密集场景。2. 处理大数据量。3. 远程方法调用等。

```java
public class Demo {
    private ExecutorService executorService = Executors.newSingleThreadExecutor();
    public Future<Integer> calculate(Integer integer) {
        return executorService.submit(()->{
            System.out.println("Calculating..."+integer);
            Thread.sleep(1000);
            return integer*integer;
        });
    }
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        Demo demo = new Demo();
        Future<Integer> result = demo.calculate(20);
        //do something
        Thread.sleep(300);
        while(!result.isDone()) {
            System.out.println("Calculating...");
            //do something
            Thread.sleep(300);
        }
        // 阻塞操作，会一直等待异步执行完毕才返回结果
        Integer ret = result.get();
        System.out.println(ret);
    }
}
/**
 * Calculating...20
 * Calculating...
 * Calculating...
 * Calculating...
 * 400
 */
```

## 线程池

**使用线程池的好处**：

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

### `Executor` 框架

易于管理，效率高，避免 this 逃逸问题。

![image-20210914101730813](https://gitee.com/hqinglau/img/raw/master/img/20210914101730.png)

`Runnable` 接口或 `Callable` 接口 实现类都可以被 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor` 执行。

**需要更多关注的是 `ThreadPoolExecutor` 这个类，这个类在实际使用线程池的过程中，使用频率非常高。**

 `ScheduledThreadPoolExecutor` 实际上是继承了 `ThreadPoolExecutor` 并实现了 ScheduledExecutorService ，而 `ScheduledExecutorService` 又继承了 `ExecutorService`。

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService
```

调用`submit()`返回一个`Future`对象：

```java
public abstract java.util.concurrent.Future<?> submit(@NotNull Runnable task)
```

**Executor 框架使用示意图**

![image-20210914103608678](https://gitee.com/hqinglau/img/raw/master/img/20210914103608.png)

### `ThreadPoolExecutor`

这个在之前的《[Java多线程volatile、ThreadLocal、线程池、atomic](https://orzlinux.cn/blog/javaMultiThread20210913.html)》写过了，这里再归纳一遍。

```java
/**
* 用给定的初始参数创建一个新的ThreadPoolExecutor。
*/
public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                          int maximumPoolSize,//线程池的最大线程数
                          long keepAliveTime,//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                          TimeUnit unit,//时间单位
                          BlockingQueue<Runnable> workQueue,//任务队列，用来储存等待执行任务的队列
                          ThreadFactory threadFactory,//线程工厂，用来创建线程，一般默认即可
                          RejectedExecutionHandler handler//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                         ) 
```

**`ThreadPoolExecutor` 3 个最重要的参数：**

- `corePoolSize` : 核心线程数线程数定义了最小可以同时运行的线程数量。
- `maximumPoolSize` : 当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
- `workQueue`: 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

`ThreadPoolExecutor`**其他常见参数**:

1. `keepAliveTime`:当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁；
2. `unit` : `keepAliveTime` 参数的时间单位。
3. `threadFactory` :executor 创建新线程的时候会用到。
4. `handler` :饱和策略。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210914104332.png" alt="image-20210914104332278" style="zoom:80%;" />

### 几个对比

`Runnable` 接口不会返回结果或抛出**检查异常**，但是 `Callable` 接口可以。所以，如果任务不需要返回结果或抛出异常推荐使用 `Runnable` 接口，这样代码看起来会更加简洁。

`execute()`方法用于**提交不需要返回值的任务**，所以无法判断任务是否被线程池执行成功与否。

`submit()`方法用于提交需要返回值的任务。线程池会返回一个 `Future` 类型的对象，通过这个 `Future` 对象可以判断任务是否执行成功 ，并且可以通过 `Future` 的 `get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

```java
public abstract void execute(@NotNull Runnable command);
public abstract java.util.concurrent.Future<?> submit(@NotNull Runnable task);
```

`shutdown（）` : **关闭线程池**，线程池的状态变为 `SHUTDOWN`。线程池不再接受新任务了，但是队列里的任务得执行完毕。

`shutdownNow（）` : 关闭线程池，线程的状态变为 `STOP`。线程池会终止当前正在运行的任务，并停止处理排队的任务并返回正在等待执行的 List。

`isShutDown` 当调用 `shutdown()` 方法后返回为 true。

`isTerminated` 当调用 `shutdown()` 方法后，并且**所有提交的任务完成后返回为 true**。

### `ScheduledThreadPoolExecutor`

> Quartz 是一个由 java 编写的任务调度库。在实际项目开发中使用 Quartz 的还是居多，比较推荐使用 Quartz。因为 Quartz 理论上能够同时对上万个任务进行调度，拥有丰富的功能特性，包括任务调度、任务持久化、可集群化、插件等等。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210914110727.png" alt="image-20210914110727933" style="zoom:80%;" />

周期任务：

![image-20210914110800860](https://gitee.com/hqinglau/img/raw/master/img/20210914110800.png)



## `Semaphore`

```java
public class Yadi {
    private static final int threadCount = 550;

    public static void main(String[] args) {
        ExecutorService threadPool =  Executors.newFixedThreadPool(300);
        final Semaphore semaphore = new Semaphore(5);
        for (int i = 0; i < threadCount; i++) {
            final int threadnum = i;
            threadPool.execute(()->{
                try {
                    // 获取不到就阻塞
                    // Semaphore 只是维持了一个可获得许可证的数量。 
                    // Semaphore 经常用于限制获取某种资源的线程数量。
                    semaphore.acquire();
                    test(threadnum);
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        threadPool.shutdown();
        System.out.println("finish");
    }

    public static void test(int threadnum) throws InterruptedException {
        Thread.sleep(1000);// 模拟请求的耗时操作
        System.out.println("threadnum:" + threadnum);
        Thread.sleep(1000);// 模拟请求的耗时操作
    }
}
```

默认抢占式：

```java
finish
threadnum:0
threadnum:1
threadnum:2
threadnum:4
threadnum:3
threadnum:300
threadnum:301
threadnum:234
threadnum:304
```

可以改成公平式，FIFO

```java
// public Semaphore(int permits, boolean fair)

threadnum:0
threadnum:3
threadnum:2
threadnum:1
threadnum:4
threadnum:9
threadnum:5
threadnum:7
threadnum:8
```

> `Semaphore` 与 `CountDownLatch` 一样，也是共享锁的一种实现。它默认构造 AQS 的 state 为 `permits`。当执行任务的线程数量超出 `permits`，那么多余的线程将会被放入阻塞队列 Park,并自旋判断 state 是否大于 0。只有当 state 大于 0 的时候，阻塞的线程才能继续执行,此时先前执行任务的线程继续执行 `release()` 方法，`release()` 方法使得 state 的变量会加 1，那么自旋的线程便会判断成功。 如此，每次只有最多不超过 `permits` 数量的线程能自旋成功，便限制了执行任务线程的数量。



## `CountDownLatch`

`CountDownLatch` 允许 `count` 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

`CountDownLatch` 是共享锁的一种实现,它默认构造 AQS 的 `state` 值为 `count`。当线程使用 `countDown()` 方法时,其实使用了`tryReleaseShared`方法以 CAS 的操作来减少 `state`,直至 `state` 为 0 。当调用 `await()` 方法的时候，如果 `state` 不为 0，那就证明任务还没有执行完毕，`await()` 方法就会一直阻塞，也就是说 `await()` 方法之后的语句不会被执行。然后，`CountDownLatch` 会自旋 CAS 判断 `state == 0`，如果 `state == 0` 的话，就会释放所有等待的线程，`await()` 方法之后的语句得到执行。

**两种用法**：

- 等待n个线程执行完毕，如加载组件。
- 作为信号枪，同时开始线程。先创建一个`CountDownLatch`对象，初始化为1，多个线程执行前都`await()`，主线程调用`countDown`时，多个线程同时被唤醒。

```java
public class CountDownLathDemo {
    private static final int threadCount = 550;

    public static void main(String[] args) throws InterruptedException {
        ExecutorService threadPool =  Executors.newFixedThreadPool(300);
        final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            final int threadnum = i;
            threadPool.execute(()->{
                try {
                    test(threadnum);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        threadPool.shutdown();
        System.out.println("finish");
    }

    public static void test(int threadnum) throws InterruptedException {
        Thread.sleep(1000);// 模拟请求的耗时操作
        System.out.println("threadnum:" + threadnum);
        Thread.sleep(1000);// 模拟请求的耗时操作
    }
}

/**
 * ...
 * threadnum:477
 * threadnum:478
 * threadnum:481
 * threadnum:458
 * finish
 */
```

> `CountDownLatch` 是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当 `CountDownLatch` 使用完毕后，它不能再次被使用。

## `CyclicBarrier`

`CyclicBarrier` 可以用于多线程计算数据。让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

```java
public class CyclicBarrierDemo {
    private static final int threadCount = 550;
    private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);
    public static void main(String[] args) throws InterruptedException {
        ExecutorService threadPool =  Executors.newFixedThreadPool(10);

        for (int i = 0; i < threadCount; i++) {
            final int threadnum = i;
            threadPool.execute(()->{
                try {
                    test(threadnum);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                } catch (TimeoutException e) {
                    e.printStackTrace();
                }
            });
        }
        threadPool.shutdown();
        System.out.println("finish");
    }

    public static void test(int threadnum) throws InterruptedException, BrokenBarrierException, TimeoutException {
        System.out.println("threadnum:" + threadnum+" is ready");
        //等到来了5个才放行
        cyclicBarrier.await(60, TimeUnit.SECONDS);
        System.out.println("threadnum:" + threadnum+" is finish");
    }
}

// threadnum:5 is ready
// threadnum:3 is ready
// threadnum:1 is ready
// threadnum:2 is ready
// threadnum:4 is ready
// threadnum:0 is ready
// threadnum:4 is finish
// threadnum:2 is finish
// ...
```

可以指定屏障可以打开时要首先执行的函数：

```java
// public CyclicBarrier(int parties, Runnable barrierAction)
private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
    System.out.println("屏障可以打开了");
});
    
// threadnum:0 is ready
// threadnum:1 is ready
// threadnum:2 is ready
// threadnum:3 is ready
// threadnum:4 is ready
// 屏障可以打开了
```

对于 `CountDownLatch` 来说，重点是“一个线程（多个线程）等待”，而其他的 N 个线程在完成“某件事情”之后，可以终止，也可以等待。而对于 `CyclicBarrier`，重点是多个线程，在任意一个线程没有完成，所有的线程都必须等待。

`CountDownLatch` 是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而 `CyclicBarrier` 更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。



## 参考

[java线程池学习总结](https://snailclimb.gitee.io/javaguide/#/./docs/java/multi-thread/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93)

[this引用逸出](https://www.cnblogs.com/whatisjava/archive/2013/05/29/3106336.html)

[java中Future的使用](https://www.cnblogs.com/flydean/p/12680281.html)

[AQS 原理以及 AQS 同步组件总结](https://snailclimb.gitee.io/javaguide/#/docs/java/multi-thread/AQS原理以及AQS同步组件总结)

