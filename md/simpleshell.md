> OSTEP项目三：[github测试文件代码](https://github.com/remzi-arpacidusseau/ostep-projects)

写了个半成品`shell`，没写完，但是基本框架已经差不多了，我的休假已经快没了，要和女朋友看电视去了。

大致效果就是下面的`wish`，支持`stdin`，或者批量命令文件，`&`连接命令并行执行：

```shell
wish> sleep 3 & echo hi & sleep 5 &echo ha
hi
ha
wish>
```

其他测试如下：

```shell
pi@raspberrypi:~/ostep-projects/ostep-projects/processes-shell $ ./wish 
wish> ls
a.txt  README.md  tests  tests-out  test-wish.sh  wish  wish.c
wish> pwd
/home/pi/ostep-projects/ostep-projects/processes-shell
wish> cd tests
wish> pwd
/home/pi/ostep-projects/ostep-projects/processes-shell/tests
wish> ps aux | grep wish   # 管道未实现
error: garbage option

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.

For more details see ps(1).
wish> ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  33804  8060 ?        Ss   15:22   0:04 /sbin/init splash
root         2  0.0  0.0      0     0 ?        S    15:22   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        I<   15:22   0:00 [rcu_gp]
...
pi        6316  0.1  0.0   8516  3608 pts/2    Ss+  19:29   0:00 bash
root      6317  0.0  0.0      0     0 ?        I<   19:29   0:00 [kworker/2:1H]
pi        6324  0.0  0.0   9788  2464 pts/1    R+   19:30   0:00 ps aux
wish> exit
pi@raspberrypi:~/ostep-projects/ostep-projects/processes-shell $ 
```

大致思路比较简单：

- 首先是`built-in`函数，如`cd`，`exit`，需要自己处理来`chdir`，或者退出程序。
- 主程序判断参数，是文件还是`stdin`，然后裹进一个大循环。
- 获取每一行参数，并行处理符号`&`，可以用`strsep`函数，命令空格规范。
- `fork`之后，可以用`access`系统调用判断是否可读或者可执行来遍历`path`组装绝对路径。

代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>
#include <sys/prctl.h>
#include <ctype.h>

char error_message[30] = "An error has occurred\n";
char *path;

// 返回参数个数
int dealWithBlank(char *str,int n)
{
    int resultBlankCount = 0;
    int i = 0;
    int dstid = 0;
    while(i<n&&isspace(str[i])) i++;
    while(i<n)
    {
        while(i<n&&!isspace(str[i]))
        {
            str[dstid++] = str[i];
            i++;
        } 
        while(i<n&&isspace(str[i])) i++;
        if(dstid<n) str[dstid++] = ' ';
        resultBlankCount++;
    }
    if(dstid>0&&isspace(str[dstid-1])) 
    {
        dstid--;
        resultBlankCount--;
    }
    str[dstid] = 0;
    return resultBlankCount+1;
}

int processCommand(char *command,int isBatch)
{
    int args = dealWithBlank(command,strlen(command));
    // printf("***%s***\n",command);
    if(strcmp(command,"exit")==0)
    {
        exit(0);
    }
    else if(strncmp(command,"path",4)==0)
    {
        strncpy(path,command+5,255);
        dealWithBlank(path,strlen(path));
        // printf("--%s--\n",path);
        return 0;
    }
    else if(strncmp(command,"cd",2)==0)
    {
        int cdState = 0;
        if(args!=2) cdState = 1;
        else
        {
            if(chdir(command+3)!=0) cdState = 1;
        }
        return cdState;
    }
    pid_t pid = fork();
    if(pid == 0) //child
    {
        // 设置父进程退出子进程也退出
        prctl(PR_SET_PDEATHSIG,SIGKILL);
        char *arg[256];
        int n = 0;
        while((arg[n] = strsep(&command," ")))
        {
            n++;
        }
        arg[n] = NULL;
        char *originPath = malloc(1024);
        strcpy(originPath,path);
        char absolutePath[1024];
        char *path1;
        while(path1=strsep(&originPath," "))
        {

            sprintf(absolutePath,"%s/%s",path1,arg[0]);
            // printf("%s\n",absolutePath);
            if(access(absolutePath,X_OK)==0)
            {
                // printf("%s\n",absolutePath);
                if(execv(absolutePath,arg)==-1)
                {
                    write(STDERR_FILENO, error_message, strlen(error_message)); 
                    exit(0);
                } 
            }
            
        }
    }
    return 0;
}
// batch = 1出错直接exit
int processOneline(char *commands,int isBatch)
{
    char *command[256];
    int n = 0;
    int stat = 0;
    while((command[n] = strsep(&commands,"&")))
    {
        stat = processCommand(command[n],isBatch);
        if(stat!=0) 
        {
            write(STDERR_FILENO, error_message, strlen(error_message)); 
            if(isBatch)
                exit(0);
            return 1;
        }
        n++;
    }
    for(int i=0;i<n;i++)
        wait(NULL);
}

int main(int argc,char **argv)
{
    path = (char *)malloc(sizeof(char)*1024);
    strcpy(path,"/bin");
    char *buf = (char *)malloc(sizeof(char)*256);
    int nsize;
    if(argc==1)
    {
        for(;;)
        {
            printf("wish> ");
            getline(&buf,&nsize,stdin);
            processOneline(buf,0);
        }
    }
    else if(argc==2)
    {
        FILE *fp = fopen(argv[1],"r");
        if(fp == NULL)
        {
            perror("file: ");
            exit(0);
        }
        while(getline(&buf,&nsize,fp)>0)
        {
            processOneline(buf,1);
        }
    }
    else{
        printf("usage\n");
    }
}
```

简单的多进程调试：

```shell
(gdb) set follow-fork-mode child
(gdb) b 69
Breakpoint 1 at 0x10c20: file ./wish.c, line 70.
(gdb) r
Starting program: /home/pi/ostep-projects/ostep-projects/processes-shell/wish 
wish> ls
***ls***
[Attaching after process 5402 fork to child process 5405]
[New inferior 2 (process 5405)]
[Detaching after fork from parent process 5402]
[Inferior 1 (process 5402) detached]
[Switching to process 5405]

Thread 2.1 "wish" hit Breakpoint 1, processCommand (command=0x23a68 "ls", isBatch=0) at ./wish.c:70
70              int n = 0;
```

