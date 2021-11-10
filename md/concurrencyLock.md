锁：放在临界区周围，保证临界区像单条原子指令一样执行。锁让原本由OS调度的混乱状态变得可控。

## 锁的实现

讨论实现之前应设定目标，那么如何评价一个锁实现的效果？第一是基本任务：锁住，提供互斥，阻止多个线程进入临界区。第二是公平性：尽量保证每一个竞争锁的线程有同样的机会。第三是性能，分为没有竞争和有竞争的情况。

### 控制中断

在临界区关闭中断（为单处理器开发的）。

简单，缺点也很多：恶意程序从头lock到尾不unlock，中断关了，操作系统没法得到控制器，只能重启；不适用于多处理器，一个CPU中断关了其他CPU还是可能访问临界区；中断丢失，自损八百；开关中断效率低。

内核自己某些情况可以用这个方法。

### 硬件支持

以下硬件支持都是原子的。

原子交换：硬件支持，最简单的是测试并设置指令，即原子交换。测试并设置指令做的是：对于一个变量，获取old value，设置new value，然后返回old value。

比较并交换：如名。

链接的加载（load linked）和条件式存储（store-conditional）指令：只有上一次加载的地址在期间没有更新时才会成功。

获取并增加（fetch and add）：获取值，并在原地址+1。

### 锁的实现

自旋锁大致原理如下：

```c
typedef struct lock_t
{
    int flag;
} lock_t;

void init(lock_t *mutex)
{
    // 0 -> lock is available, 1 -> held
    mutex->flag = 0;
}

void lock(lock_t *mutex)
{
    while (mutex->flag == 1) // TEST the flag
        ;                    // spin-wait (do nothing)
    mutex->flag = 1;         // now SET it!
}

void unlock(lock_t *mutex)
{
    mutex->flag = 0;
}
```

> 问题也很明显，软件不够用，判别语句和接下来要执行的语句可能被另一个线程打断，需要硬件支持。

基于原子交换指令就可以修改之前的lock：

```c
void lock(lock_t *lock)
{
    // 1: 被其他线程锁住了，循环
    // 0: 设置为1，返回0，lock返回
    while(TestAndSet(&lock->flag,1)==1)
        ;
}
```

单处理器需要时钟中断自旋的线程运行其他线程。自旋线程永远不会放弃CPU。

评价自旋锁：正确性有了，公平性完全没谈，性能单CPU开销非常大，需要自旋时间片等操作系统时钟中断运行另一个线程释放锁再回到自旋的线程，多CPU性能不错，因为临界区一般很短，很快就能使用。

链接的加载和条件式存储实现自旋锁如下：

```c
void lock(lock_t *lock)
{
	while (1) 
    {
        while (LoadLinked(&lock->flag) == 1)
            ;
        if(StoreConditional(&lock->flag,1)==1)
            return;  // 设置锁成功return，失败loop，条件存储只有一个能成功
    }
}
```

同样，通过获取并加一也可以实现锁。

```c
// 仅为实现逻辑，应为硬件提供支持
int FetchAndAdd(int *ptr)
{
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
typedef struct lock_t
{
    int ticket;
    int turn;
} lock_t;

void lock_init(lock_t *lock)
{
    lock->ticket = 0;
    lock->turn = 0;
}

void lock(lock_t *lock)
{
    // 想加锁，取个票排个号
    int myturn = FetchAndAdd(&lock->ticket);
    while (lock->turn != myturn)
        ; // spin
}

void unlock(lock_t *lock)
{
    // 解锁，叫下一个号
    FetchAndAdd(&lock->turn);
}

```

fetch and add 能保证所有线程都能抢到锁。总会排到号的。

自旋的问题：自旋是消耗CPU的，单处理器下还是整个时间片，因为它不会让出CPU。

第一种解决办法是主动放弃CPU。原代码为：

```c
while (mutex->flag == 1) // TEST the flag
    ;                    // spin-wait (do nothing)
```

修改为：

```c
while (mutex->flag == 1) // TEST the flag
    yield();             // 放弃CPU
```

`yield()`是操作系统原语，主动放弃CPU，让running变为ready态（线程三种状态：运行，就绪和阻塞，有的分的更细致）。

两个线程`yield()`不错，很多线程抢一个锁的问题在于ready态可能下一个还是运行它，就会跳来跳去，而上下文切换成本很高。

对应的一种解决办法是不跳到ready态，而是blocked。

上面的锁就可以改进为一个队列，一个`park()`让线程休眠，一个`unpark()`唤醒线程。

两阶段锁：先自旋一段时间，没有获取到再睡眠。

## 基于锁的并发数据结构

### 并发计数器

通常做法：

```c
typedef struct {
    int value;
    pthread_mutex_t lock;
}counter;

void init(counter *c)
{
    pthread_mutex_init(&c->lock,NULL);
    c->value = 0;
}

void inc(counter *c,int n)
{
    /pthread_mutex_lock(&c->lock);
    c->value += n;
    pthread_mutex_unlock(&c->lock);   
}

int get(counter *c,int n)
{
    pthread_mutex_lock(&c->lock);
    int ret = c->value;
    pthread_mutex_unlock(&c->lock);  
    return ret; 
}
```

测试代码如下：

```c
void *inc10000000(void *p)
{
    counter *c = (counter *)p;
    for(int i=0;i<10000000;i++)
        inc(c,1);
}

int main()
{
    counter c;
    init(&c);
    pthread_t pid1;
    pthread_t pid2;
    pthread_create(&pid1,NULL,inc10000000,(void *)&c);
    pthread_create(&pid2,NULL,inc10000000,(void *)&c);
    pthread_join(pid1,NULL);
    pthread_join(pid2,NULL);
    printf("done, c->value: %d\n",c.value);
    return 0;
}
```

结果对比：

```shell
# 不加锁
l@vm:~/ostep$ time ./a.out 
done, c->value: 10804806  # 不加锁结果混乱

real	0m0.105s    
user	0m0.199s
sys	0m0.004s

# 加锁
l@vm:~/ostep$ time ./a.out 
done, c->value: 20000000

real	0m0.301s  # 加锁速度变慢
user	0m0.295s
sys	0m0.008s
```

一种改良机制：局部锁，懒惰方法，局部加到一定值才请求全局锁。

```c
#define NCPU 4

typedef struct {
    int value;
    pthread_mutex_t lock;

    int localv[NCPU];
    pthread_mutex_t locallock[NCPU];
}counter;

typedef struct {
    counter *c;
    int who;
}counterinfo;

void init(counter *c)
{
    pthread_mutex_init(&c->lock,NULL);
    c->value = 0;
    for(int i=0;i<NCPU;i++)
    {
        c->localv[i] = 0;
        pthread_mutex_init(&c->locallock[i],NULL);
    }
}

void inc(counter *c,int n)
{
    pthread_mutex_lock(&c->lock);
    c->value += n;
    pthread_mutex_unlock(&c->lock); 
}

void inc2(counter *c,int n,int who)
{
    pthread_mutex_lock(&c->locallock[who]);
    c->localv[who] += n;
    if(c->localv[who]>50)
    {
        inc(c,c->localv[who]);
        c->localv[who] = 0;
    }
    pthread_mutex_unlock(&c->locallock[who]);  
}
```

测试代码：

```c
void *inc10000000(void *p)
{
    counterinfo *c1 = (counterinfo *)p;
    counter *c = c1->c;
    int who = c1->who;
    for(int i=0;i<10000000;i++)
        inc2(c,1,who);
    inc(c,c->localv[who]);
}

int main()
{
    counter c;
    init(&c);
    pthread_t pids[NCPU];
    counterinfo cinfo[NCPU];
    for(int i=0;i<NCPU;i++)
    {
        cinfo[i].c = &c;
        cinfo[i].who = i;
        pthread_create(&pids[i],NULL,inc10000000,(void *)&cinfo[i]);
    }
    for(int i=0;i<NCPU;i++)
        pthread_join(pids[i],NULL);
    printf("done, c->value: %d\n",c.value);
    return 0;
}
```

在树莓派上测试的，arm 4核。

```shell
# 局部变量加锁
pi@raspberrypi:~/ostep $ time ./a.out 
done, c->value: 40000000

real	0m5.126s
user	0m16.283s
sys	0m0.001s

# 局部变量不加锁
pi@raspberrypi:~/ostep $ time ./a.out 
done, c->value: 40000000

real	0m0.264s
user	0m0.837s
sys	0m0.031s
```

> 在限定条件下，如此例，局部不加锁应该也是可以的。这里的懒惰方法按我的测试其实是效果变差了的，这个例子应该是临界区太短，加锁解锁的开销甚至比等待全局锁的时间还大。然后我将inc任务加了个延迟，懒惰方法的优势就比较明显了。

### 并发链表

全局锁没啥说的。

过手锁：hand over hand locking，也叫锁耦合，lock coupling。每个节点都有一个锁，但是缺陷也和之前的一样，加锁和解锁开销巨大，反而比单锁的还慢。

### 并发队列

同样可以用全局锁。

这个根据队列的性质，只能处理开头和结尾的节点，所以入队列和出队列可以两把锁并发执行。

## 参考

Operating Systems: Three Easy Pieces
