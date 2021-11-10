尝试着写下内存分配函数，dimalloc和difree。

> 目标：如何使用 mmap()获取一大块匿名内存，然后使用指针，以构建一个简单的空闲列表来管理空间。采用最优匹配，也就是每次全部遍历找最合适的大小。

测试效果如下：

```c
// main.c
#include <stdio.h>
#include "dimalloc.h"

int main()
{
    char *str = (char *)dimalloc(sizeof(char)*64);
    sprintf(str,"%s","hi");
    printf("%s\n",str);

    char *str2 = (char *)dimalloc(sizeof(char)*128);
    sprintf(str2,"%s","hi2");
    printf("%s\n",str2);
    difree(str2);

    char *str3 = (char *)dimalloc(sizeof(char)*256);
    sprintf(str3,"%s","hi3");
    printf("%s\n",str3);
    difree(str);
    difree(str3);
    return 0;
}
```

我用freeNode来存放空闲区间。

```c
typedef struct freeNode
{
    int start;
    int end;
    struct freeNode *before;
    struct freeNode *next;
}freeNode;
```

但是问题又出现了：鸡生蛋还是蛋生鸡的问题，freeNode的空间放哪，我这里就是在空闲区间里找个空闲地方放入了，但是这个策略貌似不太好，如下图最后，free node反而割裂了空闲区间，不过也是一次有趣的尝试了。

如上代码main.c里面的描述，图像化分配区间如下，灰色部分为分配区间长度，这样`free(void *p)`就不用考虑长度了。

![image-20210823220645258](https://gitee.com/hqinglau/img/raw/master/img/20210823220645.png)

最后的空闲区间列表如下：

![image-20210823214722389](https://gitee.com/hqinglau/img/raw/master/img/20210823214722.png)

> 没仔细测试，可能仍有bug

```c
#include "dimalloc.h"

void *baseaddr = NULL;
freeNode *head = NULL;
#define BLOCKSIZE 1024 * 1024 * 32

void *dimalloc(int sz)
{
    void *ret;
    // 分配32MB内存作为大内存块
    if (!baseaddr)
    {
        baseaddr = mmap(NULL, BLOCKSIZE, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);

        // 内存分配之前int类型表明大小
        head = baseaddr + 4;
        head->start = sizeof(freeNode) + 4;
        head->end = BLOCKSIZE; // 不包含
        head->before = NULL;
        head->next = NULL;
        *(int *)(baseaddr) = sizeof(freeNode);
    }
    // 最优算法：便利找最合适的大小
    freeNode *p = head;
    freeNode *bestp = NULL;
    while (p)
    {
        if (!bestp)
        {
            if ((p->end - p->start + 4) >= sz)
                bestp = p;
        }
        else
        {
            if (((bestp->end - bestp->start) > (p->end - p->start)) && (p->end - p->start + 4 >= sz))
            {
                bestp = p;
            }
        }
        p = p->next;
    }
    if (!bestp)
    {
        //抛出异常
    }
    ret = baseaddr + bestp->start + 4;
    *(int *)(baseaddr + bestp->start) = sz;
    if (sz == bestp->end - bestp->start + 4)
    {
        if (bestp->before)
        {
            bestp->before = bestp->next;
            bestp->next->before = bestp->before->next;
        }
        else
        {
            head = bestp->next;
            bestp->next->before = NULL;
        }
        difree(bestp);
    }
    else
    {
        bestp->start = bestp->start + 4 + sz;
    }
    return ret;
}

freeNode *newFreeNode(int start, int end)
{
    freeNode *p = (freeNode *)dimalloc(sizeof(freeNode));
    p->start = start;
    p->end = end;
    p->next = NULL;
    p->before = NULL;
}

void difree(void *p)
{
    int sz = *(int *)(p - 4);
    int start = p - baseaddr - 4;
    int end = start + sz + 4;
    if (!head)
    {
        head = (freeNode *)dimalloc(sizeof(freeNode));
        head->start = start;
        head->end = end;
        head->next = NULL;
        head->before = NULL;
        return;
    }

    freeNode *tmp = head;
    freeNode *left = NULL;
    freeNode *right = NULL;
    while (tmp)
    {
        if (tmp->end <= start)
        {
            left = tmp;
            tmp = tmp->next;
        }
        else
            break;
    }
    if (left)
        right = left->next;
    else
        right = head;

    if (left && right && start == left->end && end == right->start)
    {
        left->end = right->end;
        left->next = right->next;
        difree(right);
    }
    else if (right && end == right->start)
        right->start = start;
    else if (left && start == left->end)
        left->end = end;
    else
    {
        tmp = newFreeNode(start, end);
        if (!left)
        {
            head = tmp;
            tmp->next = right;
            right->before = tmp;
        }
        if (!right)
        {
            tmp->before = left;
            left->next = tmp;
        }
    }
}
```