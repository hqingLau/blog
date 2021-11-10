> OSTEP项目二：[github测试文件代码](https://github.com/remzi-arpacidusseau/ostep-projects)

这个就一个功能：reverse。

```shell
prompt> ./reverse
prompt> ./reverse input.txt
prompt> ./reverse input.txt output.txt
```

输入：（可以是文件，也可以是`stdin`）：

```shell
hello
this
is 
a file
```

输出：（可以是文件，也可以是`stdout`）：

```shell
a file
is
this
hello
```

总体感觉这个项目还挺有练习效果，建议做做，里面主要有几个点：

- 每行可能任意多个字符，那最好做一下动态扩容。
- 反转行，用栈的思路写比较方便，没必要写个通用栈（`push`，`pop`之类的），结合具体应用就行。
- 文件为同一个文件要报错退出，问题？怎么判断同一文件？文件名？那link呢？

```c
// stack
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

typedef struct strInStack
{
    char *str;
    struct strInStack *next;
    struct strInStack *before;
} strInStack;

typedef struct
{
    strInStack *tail;
    int size;
} distack;

void assertMalloc(char *buf)
{
    if (buf == NULL)
    {
        fprintf(stderr, "malloc failed\n");
        exit(1);
    }
}

int readOneline(distack *stack, FILE *fpin)
{
    // 动态扩容，开辟了最后再释放，作为缓冲
    char *buf = (char *)malloc(sizeof(char) * 129);
    assertMalloc(buf);
    int bufSize = 128;
    int i = 0;
    int nread;
    for (;;)
    {
        nread = fread(buf + i, 1, 1, fpin);

        if (nread == 0)
        {
            buf[i] = 0;
            break;
        }
        if (buf[i] == '\n')
        {
            buf[i + 1] = 0;
            break;
        }
        i++;
        // 容量不够了，2倍扩容
        if (i == bufSize)
        {
            bufSize *= 2;
            char *newbuf = (char *)malloc(sizeof(char) * (bufSize + 1));
            assertMalloc(newbuf);
            strncpy(newbuf, buf, i);
            free(buf);
            buf = newbuf;
            newbuf = NULL;
        }
    }
    strInStack *sis = (strInStack *)malloc(sizeof(strInStack));
    if (sis == NULL)
    {
        fprintf(stderr, "bad malloc\n");
        exit(1);
    }
    sis->str = buf;
    sis->before = NULL;
    sis->next = NULL;
    if (stack->size == 0)
    {
        stack->tail = sis;
        stack->size = 1;
        return 0;
    }
    sis->before = stack->tail;
    stack->tail->next = sis;
    stack->tail = sis;
    stack->size += 1;
    if (nread == 0)
        return 1; //文件末尾
    return 0;
}

void popAllStack(distack *stack, FILE *fpout)
{
    strInStack *sis = stack->tail;
    while (stack->size > 0)
    {
        fprintf(fpout, "%s", sis->str);
        free(sis->str);
        sis = sis->before;
        free(stack->tail);
        stack->tail = sis;
        stack->size--;
    }
}

int main(int argc, char **argv)
{
    if (argc > 3)
    {
        fprintf(stderr, "usage: reverse <input> <output>\n");
        exit(1);
    }
    FILE *fpin = stdin;
    FILE *fpout = stdout;

    if (argc >= 2)
    {
        fpin = fopen(argv[1], "r");
        if (fpin == NULL)
        {
            fprintf(stderr, "reverse: cannot open file '%s'\n", argv[1]);
            exit(1);
        }
        if (argc == 3)
        {
            // 不能仅通过文件名判断为统一文件，也可能是符号链接或者硬链接
            // if(strcmp(argv[1],argv[2])==0)
            // {
            //     fprintf(stderr,"reverse: input and output file must differ\n");
            //     exit(1);
            // }
            //应通过文件节点是否相同来判断
            struct stat file1;
            struct stat file2;
            stat(argv[1], &file1);
            stat(argv[2], &file2);
            if ((file1.st_dev == file2.st_dev) && (file1.st_ino == file2.st_ino))
            {
                fprintf(stderr, "reverse: input and output file must differ\n");
                exit(1);
            }

            fpout = fopen(argv[2], "w");
            if (fpout == NULL)
            {
                fprintf(stderr, "reverse: cannot open file '%s'\n", argv[2]);
                exit(1);
            }
        }
    }
    int state;
    distack *stack = (distack *)malloc(sizeof(distack));
    if (stack == NULL)
    {
        fprintf(stderr, "bad malloc\n");
        exit(1);
    }
    for (;;)
    {
        state = readOneline(stack, fpin);
        if (state == 1)
            break;
    }
    popAllStack(stack, fpout);
    free(stack);
    return 0;
}
```

![image-20210817161512107](https://gitee.com/hqinglau/img/raw/master/img/20210817161514.png)