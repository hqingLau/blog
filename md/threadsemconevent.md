来自Dijkstra的凝视：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210825160112.png" alt="image-20210825160112821" style="width:200px; display:block;margin:0 auto;" />

## 条件变量

condition variable. 是一个显式队列，执行状态不满足条件时，线程可以把自己加入队列，等待条件。另外一个线程改变了这个状态时，就可以唤醒一个或者多个等待线程。

两个操作`wait()`和`signal()`。

```c
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_signal(pthread_cond_t *cond);
```

`wait()`的一个参数是已经上锁的互斥量，`wait()`会释放锁，并让线程休眠，等到被唤醒时，重新获取锁，返回调用者。

```c
// 一般用法
// thread A
pthread_mutex_lock(&mutex);
while (false == ready) {             //1         
    pthread_cond_wait(&cond, &mutex); //2
    // 2相当于:
    // 释放锁并休眠等待signal
    // 等待中...
    // 接收到signal，获取锁，继续循环
}
pthread_mutex_unlock(&mutex);

// thread B 写法一
pthread_mutex_lock(&mutex);
ready = true;
pthread_mutex_unlock(&mutex);  // 3
pthread_cond_signal(&cond);    // 4
```

> 互斥锁是为了保护`ready`变量，因为1语句和2语句可能被其他线程中断，如果不加锁，刚判断`ready`是false，`ready`就被线程B设置为了true，然后调用了`signal`，问题在于这个时候thread A还没准备好，还没`wait`。参数mtx的解锁和等待参数cond的动作是原子性的，所以thread B修改`ready`的时候，`wait`已经准备好了。

为什么需要循环？因为3和4之间有间隙，可能被中断，假设此时2正在等待`signal`，`ready`没被thread A锁住，也没被thread B锁住，这时过来一个其它的线程，一顿操作`ready`变成false了，然后thread B给`wait`发了个信号说准备好了，其实已经被偷家了，要用while。而且有的线程库的实现，有可能出现一个信号唤醒两个线程的情况（spurious wakeup）。

```c
// thread B 写法二
pthread_mutex_lock(&mutex);
ready = true;
pthread_cond_signal(&cond);    // 4
pthread_mutex_unlock(&mutex);  // 3
```

交换一下3和4，不就没上面的问题了吗？**理论版本（看看就行）**：这个是性能的考虑。4之后，`wait`被唤醒，切换到thread A，A想要获取锁，锁还在thread B这呢，thread A等待，切换到B，B解锁，此时A在等锁，切换上下文到A，A继续。切换上下文比较耗时。

> 理论情况是上面这样，但是实现，如pthread很多都对这部分有优化，直接把条件变量的队列给锁的队列，只有解锁时才切换到A。所以很多实现上写法二并不会性能差，而且锁和信号之间的间隙没有了，所以一般要持有锁，也就是采用写法二。

还有一个问题，为什么要判断`ready`，我就单纯想等待一个信号呢？原因是线程运行的不确定性，可能信号来了另一个线程还没开始`wait`，所以有个标志`ready`判断成立之后在这种情况就能直接返回。

生产者/消费者（有界缓冲区）：生产者满了之后`wait`，发信号给消费者满了，消费者收到信号，发送`empty`信号。比单纯的锁减少了上下文切换，单纯的锁如果消费者比较慢，生产者满了，可能需要释放锁wait一段时间重试，这里可以直接条件变量。

最后一个问题还需要知道唤醒哪个线程。例如分配内存但是内存没了，thread A申请100KB，thread B申请10KB，等待一阵之后，thread C释放了50KB，怎么唤醒合适的线程？

一个方法是`broadcast`，唤醒所有线程。这种条件变量称为**覆盖条件**：能唤醒所有该被唤醒的，代价是不该唤醒的也唤醒了。

## 信号量

一个有整数值得对象，可以用`sem_wait()`和`sem_post()`操作它。

`sem_wait()`：信号值大于等于1，立刻返回，否则挂起等待post。

信号内都是原子的。

### 二值信号量（锁）

```c
// 设置为1，表示wait（等>0）可以直接返回
sem_init(&m,0,1);

sem_wait(&m); // 返回，进入临界区 ->0
// critical section
sem_post(&m); // post,信号量->1 ,其它线程的wait可以返回了
```

### 信号量作为条件变量

信号量也可以用于等待某些条件发生变化，即作为条件变量。

### 读写锁

两点，读不需要加锁，写需要加锁。基本原理如下：

```c
#include <semaphore.h>
typedef struct
{
    sem_t lock;      // 保护这个结构的锁
    sem_t writelock; // 写锁
    int readers;     // 记录正在读取的数目，无读锁
} rwlock_t;

void rwlock_init(rwlock_t *rw)
{
    rw->readers = 0;
    sem_init(&rw->lock, 0, 1);
    sem_init(&rw->writelock, 0, 1);
}

void rwlock_acquire_readlock(rwlock_t *rw)
{
    sem_wait(&rw->lock); // 保护rw结构
    rw->readers++;
    if (rw->readers == 1)
        sem_wait(&rw->writelock); // 因为有读的了肯定不能现在改了
    sem_post(&rw->lock);
}

void rwlock_release_readlock(rwlock_t *rw)
{
    sem_wait(&rw->lock); // 保护rw结构
    rw->readers--;
    if (rw->readers == 0)
        sem_post(&rw->writelock); // 没人读了，表示，想写的可以写了
    sem_post(&rw->lock);
}

void rwlock_acquire_writelock(rwlock_t *rw)
{
    sem_wait(&rw->writelock); // 等写锁，也就是没人读了
}
void rwlock_release_writelock(rwlock_t *rw)
{
    sem_post(&rw->writelock);
}
```

这个读写锁比较简略，要是一直有人想读，写会被饿死，好一些的策略应该是有人想写了，就不再接受新的读请求。

### 哲学家就餐问题

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210826163212.png" alt="image-20210826142303592" style="zoom:80%; display:block;margin:0 auto;" />

P是哲学家，f是叉子，每个人左右各有一把叉子，怎么吃？

一个思路是每把叉子一个锁，哲学家分别先拿起左边叉子，锁住，再拿起右边叉子，锁住，被其他哲学家把叉子拿走了就等待信号。这种思路的问题是死锁，每个人都拿起了左边的叉子，就没人能够得到右边的叉子，等待信号就等不到，造成死锁。

一个最简单的方法就是破除依赖，让其中一个哲学家先拿起右叉子，想一下就想通了。

## 常见并发问题

### 非死锁缺陷

- 违反原子性缺陷

  例如`if`判断条件以后刚要执行操作，另一个线程把条件改了，现在`if`里面的操作执行就可能导致程序崩溃（如访问一个被置为`NULL`的结构体）。

  正式表达就是：我想要它是原子的（多次访问内存），但是实现并没有实现原子性。

- 违反顺序缺陷

  线程的执行顺序是不确定的，默认一个线程先执行就可能造成程序崩溃。

### 死锁

如上面的哲学家问题。常见原因是复杂的依赖，还有就是封装，如`a.add(b)`先加锁A，加锁B之前被另一线程中断，另一线程调用了`b.add(a)`，先加锁B，想要加锁A，但是被占用了，就等待，两个线程就死锁了。而且由于封装问题，你还不知道这个地方锁的顺序有问题了。

死锁条件：互斥、持有锁并等待、非抢占、循环等待（哲学家问题）。

针对具体死锁条件的预防措施：

- 循环等待

  加锁的顺序全部一致或者约定部分锁的顺序固定。例如上面的`a.add(b)`问题如何避免？可以根据锁的地址作为获取锁的顺序。如a的地址较大，`a.add(b)`先加锁A，`b.add(a)`还是先加锁A，就避免了死锁的问题。

- 持有并等待

  加个锁，把加锁的过程保护起来。

- 非抢占

  `trylock()`，尝试获取锁，获取不到就不干了，把之前获取的锁也释放了。

  活锁：两个线程都重复这一个过程，但是没有死锁，又没有进展，称为活锁。解决方法：不干了之后随即等待一段时间。还有的问题就是封装问题，函数里面你不知道干了什么，你释放了自己的锁，但是封装函数里面干了什么你并不清楚。

- 互斥

  不用锁了，用之前的硬件指令。如用比较并交换指令插入链表头：

  ```c
  void insert(int value)
  {
      node_t *n = malloc(sizeof(node_t));
      assert(n!=NULL);
      n->value = value;
      do {
          n->next = head;
      } while(CompareAndSwap(&head,n->next,n));
  }
  ```

通过调度避免死锁：就是用到很多相同锁的线程，OS调度把他们的时间片分开，不同时运行就没死锁了。

还有的死锁避免花费的代价很大，影响又很小，那能fix最好，不能fix也过得去。

### 事件并发

`select()`、`poll()`、`epoll()`那些。

阻塞系统调用如IO请求，会阻塞线程。解决方案异步IO，程序发出IO请求之后，交给系统处理，处理好IO之后再返回给调用者控制权。

Linux上比较好的异步IO是近两年出的io_uring，内核版本需要5.x以上，看了看我的版本，留下了不学无术的泪水。

```c
Linux centos 3.10.0-1160.11.1.el7.x86_64 #1 SMP Fri Dec 18 16:34:56 UTC 2020 x86_64
```

异步IO还是个大块，之前也没怎么了解。有空用虚拟机学习一下。

## 参考

1. OSTEP

2. [条件变量 之 稀里糊涂的锁](https://zhuanlan.zhihu.com/p/55123862)