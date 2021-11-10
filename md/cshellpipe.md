<img src="https://gitee.com/hqinglau/img/raw/master/img/20210903182148.png" alt="image-20210903182141083" style="zoom: 67%;" />

一个简单的测试如下：

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
int main()
{
    char * commands = (char *)malloc(1024);

    int pipefd[2]; // 0 读 1 写
    pipe(pipefd);

    if(fork()==0) // ls 子进程
    {
        close(STDOUT_FILENO);
        dup(pipefd[1]); //现在1由标准输出变成了输出到管道
        
        close(pipefd[0]);
        close(pipefd[1]);
        char *argv[] = {"ls",0};
        execvp("ls",argv);
    }

    if(fork()==0) // ls 子进程
    {
        close(STDIN_FILENO);
        dup(pipefd[0]); //现在1由标准输出变成了输出到管道
        
        close(pipefd[0]);
        close(pipefd[1]);
        char *argv[] = {"ls",0};
        execvp("ls",argv);
    }

    close(pipefd[0]);
    close(pipefd[1]);
    int r;
    wait(&r);
    wait(&r);

    return 0;
}
```

结果：

```shell
$ ./a.out 
a.out
counter.c
dimalloc
dimalloc.c
...
```

这里面遇到的一个坑就是我在fork出的子进程更新变量了，这样是不对的。如果要并行运行，可以创建多个管道阻塞等待，最后输出回`stdout`。