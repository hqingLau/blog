# Linux Program

## shell

### here文档

![](https://gitee.com/hqinglau/img/raw/master/img/20210407162907.png)

### vimdiff查看不同文件

```shell
[hqinglau@centos markdown]$ vimdiff linux程序设计笔记.md linux程序设计笔记.md.0
```

![image-20210418021500926](https://gitee.com/hqinglau/img/raw/master/img/20210418021503.png)

## 第三章 文件操作

### 3.2 系统调用函数

![image-20210407170136462](https://gitee.com/hqinglau/img/raw/master/img/20210407170136.png)

![image-20210407190100884](https://gitee.com/hqinglau/img/raw/master/img/20210407190100.png)

- 系统调用

```cpp
int fd = open("./makefile",O_RDONLY);
char firstTwo[3] = {0};
read(fd,firstTwo,2);
printf("%s\n",firstTwo);
lseek(fd,5,SEEK_CUR);
read(fd,firstTwo,2);
printf("%s\n",firstTwo);

struct stat st;
fstat(fd,&st); //获取文件属性
time_t t = st.st_ctim.tv_sec;
printf("%s\n",asctime(localtime(&t))); //转化为字符串时间
printf("link: %d\n",st.st_nlink);  // 硬链接个数
```



- IO库函数

```cpp
FILE *fd = fopen("./makefile","rb");
char chrs[11];
int readn;
//readn = fread(chrs,1,10,fd);
//output: 10: a.out: tes
readn = fread(chrs,5,2,fd);
//output: 2: a.out: tes
if(readn>0)
{
chrs[readn*5] = '\0';
printf("%d: %s\n",readn,chrs);

}
fclose(fd);
```

![image-20210408195119811](https://gitee.com/hqinglau/img/raw/master/img/20210408195119.png)

### 3.6 输入和输出

![image-20210408200945286](https://gitee.com/hqinglau/img/raw/master/img/20210408200945.png)

%*s表示忽略

```cpp
scanf("hi %s %*s %d haha",chrs,&num);
//星号（*）开头的控制符表示对应位置上的输入数据将被忽略。
//这意味着，这个数据不会被保存，因此不需要使用一个变量来接收它。
printf("output: %s %d\n",chrs,num);
```

### 3.7 文件和目录的维护

```cpp
printf("%d\n",chmod("./1m.file",S_IWUSR|S_IROTH|S_IRGRP));
// 没有权限输出-1，正常输出0
```

一些系统调用

```cpp
char cwd[256];
mkdir("newdir",S_IRUSR|S_IWUSR|S_IXUSR);
chdir("newdir");
getcwd(cwd,255);
printf("%s\n",cwd);
chdir("../");
rmdir("newdir");
```

`[hqinglau@VM-centos vscodeFile]$ ./a.out `
`/home/hqinglau/vscodeFile/newdir`

### 3.8 扫描目录

```cpp
void printdir(char *dir,int depth)
{
    DIR *dp = opendir(dir);
    struct dirent *entry;
    struct  stat statbuf;
    
    chdir(dir);

    while((entry=readdir(dp))!=NULL)
    {
        lstat(entry->d_name,&statbuf);
        if(S_ISDIR(statbuf.st_mode))
        {
            if(strcmp(".",entry->d_name)==0||
                strcmp("..",entry->d_name)==0)
                continue;

            //*代表缩进
            printf("%*s%s/\n",depth,"",entry->d_name);
            printdir(entry->d_name,depth+4);
        }
        else printf("%*s%s\n",depth,"",entry->d_name);
    }
    chdir("..");
    closedir(dp);
}

int main()
{
    printdir("./",0);
    printf("done.\n");
    return 0;
}
```

![image-20210409125028279](https://gitee.com/hqinglau/img/raw/master/img/20210409125028.png)

### 3.9 错误处理

![image-20210408211924548](https://gitee.com/hqinglau/img/raw/master/img/20210408211924.png)



## 第四章 Linux环境

### 4.2 环境变量

![image-20210409151427093](https://gitee.com/hqinglau/img/raw/master/img/20210409151427.png)

```cpp
#include <stdio.h>
#include <stdlib.h>

extern char ** environ;
int main()
{
    char **env = environ;
    while(*env) {
        printf("%s\n",*env);
        env++;
    }
    return 0;
}
```

![image-20210409151504005](https://gitee.com/hqinglau/img/raw/master/img/20210409151504.png)

### 4.3 时间日期

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>
int main()
{
    time_t cur_time,next_time;
    // 如果tloc不是一个空指针，time函数还会把返回值写入tloc指针指向的位置。
    cur_time = time((time_t *)0);
    printf("cur time is %ld\n",cur_time);
    sleep(2);
    next_time = time((time_t *)0);
    printf("nxt time is %ld\n",next_time);
    // 该函数用来计算两个time_t值之间的秒数并以double类型返回它。
    double diff = difftime(next_time,cur_time);
    printf("diff time is %f\n",diff);
    return 0;
}
```

![image-20210409153944099](https://gitee.com/hqinglau/img/raw/master/img/20210409153944.png)

tm结构

```cpp
 struct tm{
     int tm_sec;            //取值[0,59]，非正常情况下可到达61
     int tm_min;            //取值同上
     int tm_hour;            //取值[0,23]
     int tm_mday;            //取值[1,31]
     int tm_mon;            //取值[0,11]
     int tm_year;            //1900年起距今的年数
     int tm_wday;            //取值[0,6]
     int tm_yday;            //取值[0，366]
     int tm_isdst;            //夏令时标志
 };
```

要把已分解出来的tm结构再转换为原始的time_t时间值，你可以使用mktime函数。

![image-20210409154618054](https://gitee.com/hqinglau/img/raw/master/img/20210409154618.png)

strftime格式化

![image-20210409155121139](https://gitee.com/hqinglau/img/raw/master/img/20210409155121.png)

```cpp
time_t cur_time,next_tim;
cur_time = time((time_t *)0);
printf("cur time is %ld\n",cur_time);
char *stime = asctime(localtime(&cur_time));
printf("local time: %s",stime);
printf("ctime local time: %s",ctime(&cur_time));
```

![image-20210409161229939](https://gitee.com/hqinglau/img/raw/master/img/20210409161229.png)

### 4.4 用户信息

#### SUID

> Linux运行的每个程序实际上都是以某个用户的名义在运行，因此都有一个关联的UID。
>
> 你可以对程序进行设置，让它们的运行看上去好像是由另一个用户启动的。当一个程序的SUID位被置位时，它的运行就好像是由该可执行文件的属主启动的。当su命令被执行时，程序的运行就好像它是由超级用户启动的，它随后验证用户的访问权限，将UID改为目标账户的UID值并执行该账户的登录shell。采用这种方式还可以允许一个程序的运行就好像是由另一个用户启动的，它经常被系统管理员用来执行一些维护任务。

```cpp
uid_t id = getuid();
char *name = getlogin();
printf("%d: %s\n",id,name); // 1000: hqinglau
```

![image-20210409161957874](https://gitee.com/hqinglau/img/raw/master/img/20210409161957.png)

uid变了，name没变。

获取详细信息

![image-20210409161737201](https://gitee.com/hqinglau/img/raw/master/img/20210409161737.png)

```cpp
getuid(); // 当前uid
geteuid(); // 刚进入程序的uid
```

## 第七章 数据管理

> const会放到text段
>
> 全局-> data 段

### 7.2 锁文件

那这个就是锁的原理了吧（好像并不是，只是个锁文件）。

```cpp
int fd = open("/tmp/LCK",O_RDWR|O_CREAT|O_EXCL,0444);
if(fd==-1)
{
    printf("%d: \n",errno);
    perror("error: ");
}
else
{
    printf("success\n");
}
```

![image-20210412200557566](https://gitee.com/hqinglau/img/raw/master/img/20210412200604.png)

锁例子

```cpp
const char *lock_file = "/tmp/LCK.test";
int main()
{
    int fd;
    int tries = 10;
    while(tries--)
    {
        fd = open(lock_file,O_CREAT|O_RDWR|O_EXCL,0444);
        if(fd==-1)
        {
            printf("%d - lock already present\n",getpid());
            sleep(3);
        }
        else
        {
            printf("%d - access\n",getpid());
            sleep(1);
            close(fd);
            unlink(lock_file);
            sleep(2);
        }
    }
    return 0;
}
```

区域锁定

![image-20210414203321970](https://gitee.com/hqinglau/img/raw/master/img/20210414203329.png)

```cpp
/**
 * @file test.cc
 * @author hqinglau
 * @brief 区域锁测试1
 * @version 0.1
 * @date 2021-04-14
 * 
 * @copyright Copyright (c) 2021
 * 
 */

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <fcntl.h>
#include <unistd.h>

const char *test_file = "/tmp/LCK.test";
int main()
{
    int file_desc;
    int byte_count;
    char *byte_to_write = "A";
    struct flock region_1;
    struct flock region_2;
    int res;

    file_desc = open(test_file,O_RDWR|O_CREAT,0666);
    if(!file_desc) {
        fprintf(stderr,"unable to open %s for r/w\n",test_file);
        exit(EXIT_FAILURE);
    }
    for(byte_count=0;byte_count<100;byte_count++) {
        write(file_desc,byte_to_write,1);
    }
    region_1.l_type = F_RDLCK;
    region_1.l_whence = SEEK_SET;
    region_1.l_start = 10;
    region_1.l_len=20;

    region_2.l_type = F_WRLCK;
    region_2.l_whence = SEEK_SET;
    region_2.l_start = 40;
    region_2.l_len=10;

    printf("Process %d locking file\n",getpid());
    res = fcntl(file_desc,F_SETLK,&region_1);
    if(res==-1) fprintf(stderr,"Fail lk region1\n");
    res = fcntl(file_desc,F_SETLK,&region_2);
    if(res==-1) fprintf(stderr,"Fail lk region2\n");

    sleep(60);
    printf("Process %d closing file\n",getpid());
    close(file_desc);
    exit(EXIT_SUCCESS);
}
```

```cpp
/**
 * @file lock.cc
 * @author hqinglau
 * @brief 区域锁测试2
 * @version 0.1
 * @date 2021-04-14
 * 
 * @copyright Copyright (c) 2021
 * 
 */

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <fcntl.h>
#include <unistd.h>

const char *test_file = "/tmp/LCK.test";
const int SIZE_TO_TRY = 5;

void show_lock_info(struct flock*to_show) {
    printf("\tl_type %d, ",to_show->l_type);
    printf("l_whence %d, ",to_show->l_whence);
    printf("l_start %d, ",to_show->l_start);
    printf("l_len %d, ",to_show->l_len);
    printf("l_pid %d\n",to_show->l_pid);
}

int main()
{
    int file_desc;
    int start_byte;
    struct flock region_1;
    struct flock region_2;
    int res;

    file_desc = open(test_file,O_RDWR|O_CREAT,0666);
    if(!file_desc) {
        fprintf(stderr,"unable to open %s for r/w\n",test_file);
        exit(EXIT_FAILURE);
    }
    for(start_byte=0;start_byte<100;start_byte+=SIZE_TO_TRY) {
        region_1.l_type = F_WRLCK;
        region_1.l_whence = SEEK_SET;
        region_1.l_start = start_byte;
        region_1.l_len=SIZE_TO_TRY;
        region_1.l_pid=-1;
        printf("testing f_wrlck on region from %d to %d",start_byte,start_byte+SIZE_TO_TRY);
        res = fcntl(file_desc,F_GETLK,&region_1);
        if(res==-1) {
            fprintf(stderr,"Fail getlk region1\n");
            exit(EXIT_SUCCESS);
        }
        // 只有当没办法获取锁时才会！=-1,但是不锁，只获取状态
        if(region_1.l_pid!=-1) {
            printf("lock would fail\n");
            show_lock_info(&region_1);
        }
        else {
            printf("lock would succeed\n");
        }
    }
    

    close(file_desc);
    exit(EXIT_SUCCESS);
}
```



# 游双

## 第六章 高级io

### 6.1 pipe

![image-20210422215250796](https://gitee.com/hqinglau/img/raw/master/img/20210422215250.png)

![image-20210422215333422](https://gitee.com/hqinglau/img/raw/master/img/20210422215333.png)

### 6.2 dup

![image-20210415223932696](https://gitee.com/hqinglau/img/raw/master/img/20210415223932.png)

### telnet不上是防火墙问题

```shell
[hqinglau@centos ~]$ sudo firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=202.107.195.199 accept"
[sudo] password for hqinglau: 
success
[hqinglau@centos ~]$ sudo firewall-cmd --reload
success
```

得到回复：

![image-20210416145420404](https://gitee.com/hqinglau/img/raw/master/img/20210416145427.png)

![image-20210422195036010](https://gitee.com/hqinglau/img/raw/master/img/20210422195036.png)

安装完成后，将xinetd服务加入开机自启动:

> systemctl enable xinetd.service
>
> 将telnet服务加入开机自启动：
>
> systemctl enable telnet.socket
>
> 最后，启动以上两个服务即可：
>
> systemctl start telnet.socket
>
>  systemctl start xinetd（或service xinetd start）

### 6.4 sendfile

完全在内核中操作，避免内核缓冲区和用户缓冲区数据拷贝，效率很高，称为零拷贝。

![image-20210416162621634](https://gitee.com/hqinglau/img/raw/master/img/20210416162621.png)

in必须指向真实文件。out必须是一个socket。**几乎是专门为网络传输文件设计的。**

### 数据拷贝

```c
while((n = read(diskfd, buf, BUF_SIZE)) > 0)
    write(sockfd, buf , n);
```

![image-20210416150933892](https://gitee.com/hqinglau/img/raw/master/img/20210416150933.png)



mmap 内核态和用户态缓冲区共享

![image-20210416160352798](https://gitee.com/hqinglau/img/raw/master/img/20210416160352.png)

sendfile 无用户态操作

![image-20210416160432310](https://gitee.com/hqinglau/img/raw/master/img/20210416160432.png)

### 6.6 splice函数

零拷贝。至少有一个是管道文件描述符。

```cpp
int pipefd[2];
ret = pipe(pipefd);
ret = splice(connfd,NULL,pipefd[1],NULL,32768,SPLICE_F_MORE|SPLICE_F_MOVE);
ret = splice(pipefd[0],NULL,connfd,NULL,32768,SPLICE_F_MORE|SPLICE_F_MOVE);
```

![image-20210416164129658](https://gitee.com/hqinglau/img/raw/master/img/20210416164129.png)

### 6.7 tee函数

零拷贝。都是管道文件。不消耗数据，源文件描述符可用于后序操作。

```cpp
// 管道 文件
splice(STDIN_FILENO,NULL,pipefd_stdout[1],NULL,32768,SPLICE_F_MOVE|SPLICE_F_MORE);
tee(pipefd_stdout[0],pipefd_file[1],32768,SPLICE_F_MORE|SPLICE_F_MOVE);
splice(pipefd_file[0],NULL,filefd,NULL,32768,SPLICE_F_MOVE|SPLICE_F_MORE);
splice(pipefd_stdout[0],NULL,STDOUT_FILENO,NULL,32768,SPLICE_F_MORE|SPLICE_F_MOVE);
```

我的测试是：O_TRUNC和O_APPEND同时出现会清空文件，啥也写不进去。

### 防火墙策略修改

```shell
sudo firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=10.66.101.224 accept"
sudo firewall-cmd --reload
```

## 第七章 服务器程序规范

### 7.1 日志

syslog服务器的配置文件为/etc/rsyslog.conf，syslog( )函数把想记录的日志信息都发送给日志服务器，但此日志最终是否记录到文件或发送给远程服务器，则是由此配置文件来决定的，该配置文件就是告诉日志服务器要记录那些类型和级别的日志，如何记录等信息。

```cpp
#include <syslog.h>
#include <stdio.h>

int main(int argc,char *argv[])
{
    openlog(argv[0],LOG_CONS|LOG_PID,LOG_USER);
    syslog(LOG_DEBUG,"test: %s","string");
    setlogmask(LOG_UPTO(LOG_ERR));
    //syslog(LOG_INFO,"test: %s","string2");
    //info不会显示
    syslog(LOG_CRIT,"crit: %s","string2");
    closelog();
    return 0;
}
```

![image-20210416224140983](https://gitee.com/hqinglau/img/raw/master/img/20210416224144.png)



在/etc/rsyslog.conf作配置

![image-20210416224243326](https://gitee.com/hqinglau/img/raw/master/img/20210416224246.png)

### 7.3 进程间关系

![image-20210417235406688](https://gitee.com/hqinglau/img/raw/master/img/20210417235409.png)

作业

作业与进程组的区别：如果作业中的某个进程又创建了子进程，则子进程不属于作业。一旦作业运行结束，Shell就把自己提到前台，如果原来的前台进程还存在（如果这个子进程还没终止），它自动变为后台进程组。

```cpp
#include <unistd.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>

int main(int argc,char *argv[])
{
    pid_t id = fork();
    assert(id>=0);
    if(id==0) // new process
    {
        while(1)
        {
            printf("child\n");
            sleep(1);
        }
    }
    else
    {
        int count = 0;
        while(count<5)
        {
            printf("father\n");
            ++count;
            sleep(1);
        }
        exit(EXIT_SUCCESS);
    }
    return 0;
}
```

![image-20210418001121237](https://gitee.com/hqinglau/img/raw/master/img/20210418001124.png)

```shell
[hqinglau@centos server]$ for name in $(ls); do if [[ $name =~ ^[0-9][^0-9]+.*\.cc$ ]];then mv $name 0$name; fi; done
```

给单号文件前面加个零。

![image-20210418010030418](https://gitee.com/hqinglau/img/raw/master/img/20210418010031.png)

### getpgid()

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
    pid_t id = fork();
    pid_t pid = getpid();
    pid_t gid = getpgid(pid);
    assert(id>=0);
    if(id==0) //child
    {
        // 默认继承父进程的gid（应该）
        //setpgid(0,gid);
        while(1)
        {
            printf("child\n");
            sleep(1);
        }
    }
    else
    {
        int count = 0;
        while(count<5)
        {
            printf("father\n");
            ++count;
            sleep(1);
        }
        sleep(10);
        exit(EXIT_SUCCESS);
    }
    return 0;
}
```

![image-20210418011056844](https://gitee.com/hqinglau/img/raw/master/img/20210418011058.png)

### 系统资源限制

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <sys/resource.h>

int main()
{
    rlimit rli;
    getrlimit(RLIMIT_NPROC,&rli);
    printf("proc -- cur: %d, max: %d\n",rli.rlim_cur,rli.rlim_max);
    getrlimit(RLIMIT_NOFILE,&rli);
    printf("fd -- cur: %d, max: %d\n",rli.rlim_cur,rli.rlim_max);
    getrlimit(RLIMIT_STACK,&rli);
    printf("stack -- cur: %d, max: %d\n",rli.rlim_cur,rli.rlim_max);
    getrlimit(RLIMIT_DATA,&rli);
    printf("data -- cur: %d, max: %d\n",rli.rlim_cur,rli.rlim_max);
    // [hqinglau@centos server]$ ./a.out 
    // proc -- cur: 7093, max: 7093
    // fd -- cur: 100002, max: 100002
    // stack -- cur: 8388608, max: -1
    // data -- cur: -1, max: -1
    return 0;
}
```

### dir

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <sys/resource.h>

int main()
{
    char curdir[64];
    getcwd(curdir,64);
    printf("curdir: %s\n",curdir);
    
    int ret = chdir("/home/");
    if(ret==-1)
        perror("chdir false: ");
    getcwd(curdir,64);
    printf("curdir: %s\n",curdir);
    // [hqinglau@centos server]$ ./a.out 
    // curdir: /home/hqinglau/vscodeFile/server
    // curdir: /home
    return 0;
}
```

### deamonsize

![image-20210418020951653](https://gitee.com/hqinglau/img/raw/master/img/20210418020954.png)

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <sys/resource.h>
#include <sys/fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>

int main()
{
    pid_t pid = fork();
    assert(pid>=0);
    if(pid>0) exit(0);
    umask(0);
    pid_t sid = setsid();
    assert(sid>=0);
    chdir("/");
    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    open("/dev/null",O_RDONLY);
    open("/dev/null",O_RDWR);
    open("/dev/null",O_RDWR);
    return 0;
}
```

![image-20210418020836890](https://gitee.com/hqinglau/img/raw/master/img/20210418020840.png)

## 第八章 服务器框架

![image-20210418214658243](https://gitee.com/hqinglau/img/raw/master/img/20210418215258.png)

![image-20210418215630580](https://gitee.com/hqinglau/img/raw/master/img/20210418215632.png)



![image-20210418215729032](https://gitee.com/hqinglau/img/raw/master/img/20210418215731.png)

阻塞和非阻塞的概念能应用于所有文件描述符。

IO复用本身阻塞，提高效率在于同事监听多个IO能力。

### 8.4 事件处理模式

同步-Reactor：只负责监听是否有事发生

异步-Proactor：所有IO操作都交给主线程和内核

同步I/O模拟Proactor

### 8.5 并发模式

半同步/半异步：(这里的同步异步指的是按代码序列执行还是由系统事件驱动)

同步线程处理客户逻辑，异步线程处理IO事件。

## 第九章 IO复用

### 文件描述符就绪条件

socket中，socket可读情况：

![image-20210420162231698](https://gitee.com/hqinglau/img/raw/master/img/20210420162231.png)

可写情况：

![image-20210420162218785](https://gitee.com/hqinglau/img/raw/master/img/20210420162218.png)

### 带外数据

许多的传输层都具有带外数据（也称为 经加速数据 ）的概念，想法就是连接的某段发生了重要的事情，希望迅速的通知给对端。这里的迅速是指这种通知应该在已经排队了的带内数据之前发送。也就是说，带外数据拥有更高的优先级。带外数据不要求再启动一个连接进行传输，而是使用已有的连接进行传输。

其中，UDP没有实现带外数据（是个极端哦～）

TCP中telnet,rlogin,ftp等应用（除了这样的远程非活跃应用之外，几乎很少有使用到带外数据的地方）

**但是TCP是和正常数据一起发的**

TCP的带外数据：

头部标志： URG位，紧急指针。

数据包中：一个紧急指针只指向**一个字节**的带外数据的后已字节位置。紧急数据时插在正常数据流中进行传输。紧急指针用于指出带外数据字节在正常字节流中的位置。

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

// vscode代码对齐快捷键：
// 选中文本；
// Shift  +  Alt  + F     实现代码的对齐；

int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage: %s ip_address port_number\n", basename(argv[0]));
        return EXIT_FAILURE;
    }

    const char *ip = argv[1];
    int port = atoi(argv[2]);

    struct sockaddr_in address;
    bzero(&address,sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET,ip,&address.sin_addr);
    address.sin_port = htons(port);
    int sock = socket(AF_INET,SOCK_STREAM,0);
    assert(sock>=0);
    int ret = bind(sock,(struct sockaddr *)&address,sizeof(address));
    if(ret==-1)
    {
        perror("bind error: ");
        return EXIT_FAILURE;
    }
    ret = listen(sock,5);
    assert(ret!=-1);

    struct sockaddr_in client;
    socklen_t client_addrlen = sizeof(client);
    int connfd = accept(sock,(struct sockaddr*)&client,&client_addrlen);
    if(connfd<0)
    {
        perror("accept error: ");
        close(sock);
        return EXIT_FAILURE;
    }

    char buf[1024];
    fd_set read_fds;
    fd_set exception_fds;
    FD_ZERO(&read_fds);
    FD_ZERO(&exception_fds);
    while(1)
    {
        memset(buf,'\0',sizeof(buf));
        // 每次select之后要重新设置文件描述符（文件描述符集合会被内核修改）
        FD_SET(connfd,&read_fds);
        FD_SET(connfd,&exception_fds);
        ret = select(connfd+1,&read_fds,NULL,&exception_fds,NULL);
        if(ret<0)
        {
            printf("Select failure\n");
            break;
        }
        if(FD_ISSET(connfd,&read_fds))
        {
            ret = recv(connfd,buf,sizeof(buf)-1,0);
            if(ret==0)
            {
                printf("he left.\n");
                break;
            }
            if(ret<0)
            {
                break;
            }
            printf("get %d bytes of normal data:%s\n",ret,buf);
        }
        else if (FD_ISSET(connfd,&exception_fds))
        {
            ret = recv(connfd,buf,sizeof(buf)-1,MSG_OOB);
            if(ret==0)
            {
                printf("except he left.\n");
                break;
            }
            if(ret<0)
            {
                break;
            }
            printf("get %d bytes of oob data:%s\n",ret,buf);
        }
    }

    close(connfd);
    close(sock);

    return EXIT_SUCCESS;
}
```

### POLL

![image-20210420194650053](https://gitee.com/hqinglau/img/raw/master/img/20210420194650.png)

### epoll

比较重要，整张复制

![image-20210420204554200](https://gitee.com/hqinglau/img/raw/master/img/20210420204554.png)

![image-20210420204606922](https://gitee.com/hqinglau/img/raw/master/img/20210420204607.png)

![image-20210420204622205](https://gitee.com/hqinglau/img/raw/master/img/20210420204622.png)

![image-20210420204650517](https://gitee.com/hqinglau/img/raw/master/img/20210420204650.png)

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/epoll.h>
// vscode代码对齐快捷键：
// 选中文本；
// Shift  +  Alt  + F     实现代码的对齐；

const int MAX_EVENT_NUMBER = 1024;
const int BUFFFER_SIZE = 10;

void addfd2epoll(int epollfd,int fd,bool enable_et)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN;
    if(enable_et)
    {
        event.events|=EPOLLET;
    }
    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&event);
}

void lt(epoll_event *events,int number,int epollfd,int listenfd)
{
    char buf[BUFFFER_SIZE];
    for(int i=0;i<number;i++)
    {
        int sockfd = events[i].data.fd;
        if(sockfd==listenfd)
        {
            struct sockaddr_in client;
            socklen_t client_addrlen = sizeof(client);
            int connfd = accept(listenfd, (struct sockaddr *)&client, &client_addrlen);
            addfd2epoll(epollfd,connfd,false);
        }
        else if(sockfd&EPOLLIN)
        {
            printf("event trigger once\n");
            memset(buf,'\0',BUFFFER_SIZE);
            int ret = recv(sockfd,buf,BUFFFER_SIZE,0);
            if(ret<=0)
            {
                close(sockfd);
                continue;
            }
            printf("get %d bytes of normal data:%s\n",ret,buf);
        }
        else{
            printf("sth else happend.\n");
        }
    }
}

int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage: %s ip_address port_number\n", basename(argv[0]));
        return EXIT_FAILURE;
    }

    const char *ip = argv[1];
    int port = atoi(argv[2]);

    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    int ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    if (ret == -1)
    {
        perror("bind error: ");
        return EXIT_FAILURE;
    }
    ret = listen(listenfd, 5);
    assert(ret != -1);
    /**
     * ==============to do==================== 1
     */

    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd!=-1);
    addfd2epoll(epollfd,listenfd,true);
    for(;;)
    {
        ret = epoll_wait(epollfd,events,MAX_EVENT_NUMBER,-1); // -1阻塞
        if(ret<0)
        {
            printf("epoll failure.\n");
            break;
        }

        lt(events,ret,epollfd,listenfd);
    }


    /**
     * ============== end ==================== 1
     */

    close(listenfd);

    return EXIT_SUCCESS;
}
```

输入大于`BUFFER_SIZE`时，如：123456789012

![image-20210420211634969](https://gitee.com/hqinglau/img/raw/master/img/20210420211635.png)

如果是ET的话，就要循环队读，读完为止才可以。但是可以减少触发次数。

### EPOLLONESHOT

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/epoll.h>
#include <pthread.h>
#include <sys/fcntl.h>

const int MAX_EVENT_NUMBER = 1024;
const int BUFFFER_SIZE = 10;

struct fds
{
    int epollfd;
    int sockfd;
};


void addfd2epoll(int epollfd,int fd,bool oneshot)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN;
    if(oneshot)
    {
        event.events|=EPOLLONESHOT;
    }
    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&event);
    int old_option = fcntl(fd,F_GETFL);
    int new_option = old_option|O_NONBLOCK;
    fcntl(fd,F_SETFL,new_option);
}


void reset_oneshot(int epollfd,int fd)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN|EPOLLET|EPOLLONESHOT;
    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&event);
}

void *worker(void *arg)
{
    int sockfd = ((fds*)arg)->sockfd;
    int epollfd = ((fds*)arg)->epollfd;
    char buf[BUFFFER_SIZE];
    memset(buf,'\0',BUFFFER_SIZE);
    while(1)
    {
        int ret = recv(sockfd,buf,BUFFFER_SIZE-1,0);
        if(ret==0)
        {
            close(sockfd);
            printf("foreiner closed the connection\n");
            break;
        }
        else if(ret<0)
        {
            if(errno==EAGAIN)
            {
                // 非阻塞，到这说明已经5s没有响应了
                reset_oneshot(epollfd,sockfd);
                printf("read later\n");
                break;
            }
        }
        else{
            printf("get content: %s\n",buf);
            sleep(5);
        }
    }
    printf("end thread receiving data on fd: %d\n",sockfd);
}

void lt(epoll_event *events,int number,int epollfd,int listenfd)
{
    for(int i=0;i<number;i++)
    {
        int sockfd = events[i].data.fd;
        if(sockfd==listenfd)
        {
            struct sockaddr_in client;
            socklen_t client_addrlen = sizeof(client);
            int connfd = accept(listenfd, (struct sockaddr *)&client, &client_addrlen);
            printf("new connection\n");
            addfd2epoll(epollfd,connfd,true);
        }
        else if(events[i].events&EPOLLIN)
        {
            pthread_t thread;
            fds fds_for_new_worker;
            fds_for_new_worker.epollfd = epollfd;
            fds_for_new_worker.sockfd = sockfd;
            pthread_create(&thread,NULL,worker,(void *)&fds_for_new_worker);
        }
        else{
            printf("sth else happend.\n");
        }
    }
}

int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage: %s ip_address port_number\n", basename(argv[0]));
        return EXIT_FAILURE;
    }

    const char *ip = argv[1];
    int port = atoi(argv[2]);

    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    int ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    if (ret == -1)
    {
        perror("bind error: ");
        return EXIT_FAILURE;
    }
    ret = listen(listenfd, 5);
    assert(ret != -1);

    // struct sockaddr_in client;
    // socklen_t client_addrlen = sizeof(client);
    // int connfd = accept(sock, (struct sockaddr *)&client, &client_addrlen);
    // if (connfd < 0)
    // {
    //     perror("accept error: ");
    //     close(sock);
    //     return EXIT_FAILURE;
    // }
    /**
     * ==============to do==================== 1
     */

    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd!=-1);
    addfd2epoll(epollfd,listenfd,false);
    for(;;)
    {
        ret = epoll_wait(epollfd,events,MAX_EVENT_NUMBER,-1); // -1阻塞
        if(ret<0)
        {
            printf("epoll failure.\n");
            break;
        }

        lt(events,ret,epollfd,listenfd);
    }


    /**
     * ============== end ==================== 1
     */

    close(listenfd);

    return EXIT_SUCCESS;
}
```

### 非阻塞connect

非阻塞的socket调用connect，而连接没有建立时，返回EINPROGRESS，可以调用select，poll等函数来监听连接失败的socket的可写事件，返回后用getsockopt读取错误码并清除socket的错误，如果错误码是0，表示连接建立成功，否则失败。

## 第十章 信号

![image-20210422211119702](https://gitee.com/hqinglau/img/raw/master/img/20210422211119.png)

### 10.4 统一事件源

**信号处理函数中不写逻辑，只是把“信号”通过管道传给主函数，让主函数处理。**

因为：为了避免静态条件，信号处理期间，系统不会再次出发，所以信号处理函数要尽可能快执行完毕。

![image-20210422215359845](https://gitee.com/hqinglau/img/raw/master/img/20210422215359.png)

![image-20210422215418314](https://gitee.com/hqinglau/img/raw/master/img/20210422215418.png)

用法：

```cpp
int number = epoll_wait(epollfd,events,MAX_EVENT_NUMBER,-1);
if((number<0)&&(errno!=EINTR))
{
    printf("epoll failure\n");
    break;
}
for(int i=0;i<number;i++)
{
    int sockfd = events[i].data.fd;
    if(sockfd==listenfd)
    {
        struct sockaddr_in client_address;
        socklen_t client_addrlength = sizeof(client_address);
        int connfd = accept(listenfd,(struct sockaddr*) &client_address, &client_addrlength);
        if(connfd<0)
        {
            perror("error: ");
            continue;
        }
        else{
            addfd(epollfd,connfd);
        }
    }
    else if((sockfd==pipefd[0])&&(events[i].events&EPOLLIN))
    {
        int sig;
        char signals[1024];
        ret = recv(pipefd[0],signals,sizeof(signals),0);
        if(ret==-1) continue;
        else if(ret==0) continue;
        else
        {
            for(int i=0;i<ret;++i)
            {
                switch (signals[i])
                {
                    case SIGCHLD:
                    case SIGHUP:
                        {
                            continue;
                        }
                    case SIGTERM:
                    case SIGINT:
                        {
                            printf("hqing got signal.\n");
                            stop_server=true;
                        }
                }
            }
        }
    }
}
```

### 10.5 网络编程相关信号

进程挂起原因：

**（1） 终端用户的请求。当终端用户在自己的程序运行期间发现有可疑问题时，希望暂停使自己的程序静止下来。亦即，使正在执行的进程暂停执行；若此时用户进程正处于就绪状态而未执行，则该进程暂不接受调度，以便用户研究其执行情况或对程序进行修改。我们把这种静止状态成为“挂起状态”。**

（2）父进程的请求。有时父进程希望挂起自己的某个子进程，以便考察和修改子进程，或者协调各子进程间的活动。

（3）负荷调节的需要。当实时系统中的工作负荷较重，已可能影响到对实时任务的控制时，可由系统把一些不重要的进程挂起，以保证系统能正常运行。

（4）操作系统的需要。操作系统有时希望挂起某些进程，以便检查运行中的资源使用情况或进行记账。

（5）对换的需要。为了缓和内存紧张的情况，将内存中处于阻塞状态的进程换至外存上。

![image-20210424204153837](https://gitee.com/hqinglau/img/raw/master/img/20210424204156.png)

ECHO stream 端口

![image-20210424204227962](https://gitee.com/hqinglau/img/raw/master/img/20210424204230.png)



![image-20210424204331413](https://gitee.com/hqinglau/img/raw/master/img/20210424204333.png)



## 第十一章 定时器



### 11.1 socket选项



![image-20210424223623059](https://gitee.com/hqinglau/img/raw/master/img/20210424223625.png)



![image-20210424222429604](https://gitee.com/hqinglau/img/raw/master/img/20210424222432.png)



```cpp
int timeout_connect(const char *ip, int port, int time)
{
  int ret = 0;
  struct sockaddr_in address;
  bzero(&address, sizeof(address));
  address.sin_family = AF_INET;
  address.sin_port = htons(port);
  inet_pton(AF_INET, ip, &address.sin_addr);

  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  assert(sockfd >= 0);
  struct timeval timeout;
  timeout.tv_sec = time;
  timeout.tv_usec = 0;
  socklen_t len = sizeof(timeout);
  ret = setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);
  assert(ret != -1);
  ret = connect(sockfd, (struct sockaddr *)&address, sizeof(address));
  if (ret == -1)
  {
    /* 超时对应的错误号是EINPROGRESS */
    if(errno==EINPROGRESS)
    {
      printf("connecting timeout, process timeout logic \n");
      return -1;
    }
    printf("error occur when connectiong to server\n");
    return -1;
  }
  return sockfd;
}
```



![image-20210424223348698](https://gitee.com/hqinglau/img/raw/master/img/20210424223350.png)

## 第十二章 libevent

```cpp
#include <sys/signal.h>
#include <event.h>

void signal_cb(int fd, short event, void *argc)
{
    struct event_base *base = (event_base *)argc;
    struct timeval delay = {2, 0};
    printf("Caught an interrupt signal; exiting cleanly in two seconds... \n");
    event_base_loopexit(base, &delay);
}

void timeout_cb(int fd, short event, void *argc)
{
    printf("timeout\n");
}

int main()
{
    // 一个event_base相当于一个Reactor实例
    struct event_base *base = event_init();
    // 信号事件处理器
    struct event *signal_event = evsignal_new(base, SIGINT, signal_cb, base);
    event_add(signal_event, NULL);
    // 定时事件处理器
    timeval tv = {1,0};
    struct event *timeout_event = evtimer_new(base,timeout_cb,NULL);
    event_add(timeout_event,&tv);

    // 执行事件循环
    event_base_dispatch(base);

    // 释放系统资源
    event_free(timeout_event);
    event_free(signal_event);
    event_base_free(base);
}
```

## 第十三章 多进程多线程

僵尸进程：待父进程读取子进程的退出信息；父进程挂了。

如果父进程没有正确处理子进程的返回信息，子进程都将停留在僵尸态，占据内核资源。所以有`waitpid`等待子进程。

![image-20210503224323643](https://gitee.com/hqinglau/img/raw/master/img/20210503224323.png)

### -lpthread和-pthread的区别

编译程序包括 预编译， 编译，汇编，链接，包含头文件了，仅能说明有了线程函数的声明， 但是还没有实现， 加上-lpthread是在链接阶段，链接这个库。pthread是动态库，需要用-lpthread，所有的动态库都需要用-lxxx来引用用gcc编译使用了POSIX thread的程序时通常需要加额外的选项，以便使用thread-safe的库及头文件，一些老的书里说直接增加链接选项 -lpthread 就可以了而gcc手册里则指出应该在编译和链接时都增加 -pthread 选项。编译选项中指定 **-pthread 会附加一个宏定义 -D_REENTRANT，该宏会导致 libc 头文件选择那些thread-safe的实现**；链接选项中指定 -pthread 则同 -lpthread 一样，只表示链接 POSIX thread 库。由于 libc 用于适应 thread-safe 的宏定义可能变化，因此在编译和链接时都使用 -pthread 选项而不是传统的 -lpthread 能够保持向后兼容，并提高命令行的一致性。目前gcc 4.5.2中已经没有了关于 -lpthread的介绍了。所以以后的多线程编译应该用-pthread，而不是-lpthread。

### do while(0)

在linux内核中常常会看到do{} while(0)这样的语句，有人疑惑，认为无意义，因为他只执行一次，加不加do{} while(0)小过失完全一样的，那你就错了，没有完全了解do{} while(0)。下面看一个例子：

定义一个宏：

```cpp
#define SAFE_FREE(p)  do{free(p); p=NULL}  while(0)
```

假设这里去掉`do{....} while(0)`，及定义为：

```cpp
#define SAFE_FREE(p)  free(p); p=NULL；
```

那么以下代码

```cpp
If(NULL!=p)
   SAFE_FREE(p)
else
   .......
会被展开成：
If(NULL!=p)
   free(p); p=NULL；
else
   .......
```


展开存在两个问题：

因为if分支后面有两个语句，导致else分支没有对应的if，编译失败。
假设没有else分支，则SAFE_FREE中的第二个语句无论if测试是否通过，都会执行。如何解决以上问题呢？

有人说给SAFE_FREE的定义加上{}就可以解决上述问题了，即：

```cpp
#define SAFE_FREE(p)  { free(p); p=NULL; }
```

代码展开如下：

```cpp
If(NULL!=p)
   { free(p); p=NULL; }
else
   .......

// 但是，在C程序中，每个语句后面加分号是一种约定俗成的习惯，那么代码如下：
If(NULL!=p)
   { free(p); p=NULL; }；
else
   .......
```

问题又来了，这样else又没有对应的if了，编译还是失败。假设用了do{} while(0)就可以解决上面的一系列问题了，代码如下：

```cpp
If(NULL!=p)
  do { free(p); p=NULL; } while(0)；
else
   .......
```

### sigwait

在POSIX标准中，当进程收到信号时，如果是多线程的情况，我们是无法确定是哪一个线程处理这个信号。而sigwait是从进程中pending的信号中，取走指定的信号。这样的 话，如果要确保sigwait这个线程收到该信号，那么所有线程含主线程以及这个sigwait线程则必须block住这个信号，因为如果自己不阻塞就没有未决状态(阻塞状态)信号，别的所有线程不阻塞就有可能当信号过来时，被其他的线程处理掉。

在多线程代码中，总是使用sigwait或者sigwaitinfo或者sigtimedwait等函数来处理信号。

而不是signal或者sigaction等函数。因为在一个线程中调用signal或者sigaction等函数会改变所以线程中的信号处理函数，而不是仅仅改变调用signal/sigaction的那个线程的信号处理函数。

### epoll oneshot

因为et模式需要循环读取，但是在读取过程中，如果有新的事件到达，很可能触发了其他线程来处理这个socket，那就乱了。

EPOLL_ONESHOT就是用来避免这种情况。注意在一个线程处理完一个socket的数据，也就是触发EAGAIN errno时候，就应该重置EPOLL_ONESHOT的flag，这时候，新到的事件，就可以重新进入触发流程了。

注：EPOLL_ONESHOT的原理其实是，每次触发事件之后，就将事件注册从fd上清除了，也就不会再被追踪到；下次需要用epoll_ctl的EPOLL_CTL_MOD来手动加上才行。




## STL

当(通过如insert或push_back之类的操作)向容器中加入对象时，存入容器的是你所指定的对象的拷贝。当(通过如front或back之类的操作)从容器中取出一个对象时，你所得到的是容器中所保存的对象的拷贝。**进去的是拷贝，出来的也是拷贝(copy in, copy out)。这就是STL的工作方式**。

**剥离**问题意味着**向基类对象的容器中插入派生类对象几乎总是错误的****。**使拷贝动作高效、正确，并防止剥离问题发生的一个简单办法是使容器包含指针而不是对象**。

STL指针应用智能指针

# 附：

## vscode ssh

vscode ssh连linux当编辑器还不错，安装个remote ssh，c/c++插件就好了。

![image-20210408213040869](https://gitee.com/hqinglau/img/raw/master/img/20210408213040.png)

## 爬虫百度图片

```python
"""根据搜索词下载百度图片"""
import os
import re
from typing import List, Tuple
from urllib.parse import quote

import requests
"""爬虫相关配置"""

# 关键词, 改为你想输入的词即可, 相当于在百度图片里搜索一样
keyword = '水底'

# 最大下载数量
max_download_images = 1000

# 精简一下网址，去掉网址中无意义的参数
url_init_first = 'https://image.baidu.com/search/flip?tn=baiduimage&word='

# 表头
headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 11_2_3) AppleWebKit/537.36 (KHTML, like Gecko) '
                  'Chrome/88.0.4324.192 Safari/537.36'
}


def get_page_urls(page_url: str, headers: dict) -> Tuple[List[str], str]:
    """获取当前翻页的所有图片的链接
    Args:
        page_url: 当前翻页的链接
        headers: 请求表头
    Returns:
        当前翻页下的所有图片的链接, 当前翻页的下一翻页的链接
    """
    if not page_url:
        return [], ''
    try:
        html = requests.get(page_url, headers=headers)
        html.encoding = 'utf-8'
        html = html.text
    except IOError as e:
        print(e)
        return [], ''
    pic_urls = re.findall('"objURL":"(.*?)",', html, re.S)
    next_page_url = re.findall(re.compile(r'<a href="(.*)" class="n">下一页</a>'), html, flags=0)
    next_page_url = 'http://image.baidu.com' + next_page_url[0] if next_page_url else ''
    return pic_urls, next_page_url


def down_pic(pic_urls: List[str], max_download_images: int) -> None:
    """给出图片链接列表，下载所有图片
    Args:
        pic_urls: 图片链接列表
        max_download_images: 最大下载数量
    """
    pic_urls = pic_urls[:max_download_images]
    for i, pic_url in enumerate(pic_urls):
        i = i+600
        try:
            pic = requests.get(pic_url, timeout=15)
            image_output_path = './images/' + str(i + 1) + '.jpg'
            with open(image_output_path, 'wb') as f:
                f.write(pic.content)
                print('成功下载第%s张图片: %s' % (str(i + 1), str(pic_url)))
        except IOError as e:
            print('下载第%s张图片时失败: %s' % (str(i + 1), str(pic_url)))
            print(e)
            continue


if __name__ == '__main__':
    url_init = url_init_first + quote(keyword, safe='/')
    all_pic_urls = []
    page_urls, next_page_url = get_page_urls(url_init, headers)
    all_pic_urls.extend(page_urls)

    page_count = 0  # 累计翻页数
    if not os.path.exists('./images'):
        os.mkdir('./images')

    # 获取图片链接
    while 1:
        page_urls, next_page_url = get_page_urls(next_page_url, headers)
        page_count += 1
        print('正在获取第%s个翻页的所有图片链接' % str(page_count))
        if next_page_url == '' and page_urls == []:
            print('已到最后一页，共计%s个翻页' % page_count)
            break
        all_pic_urls.extend(page_urls)
        if len(all_pic_urls) >= max_download_images:
            print('已达到设置的最大下载数量%s' % max_download_images)
            break

    down_pic(list(set(all_pic_urls)), max_download_images)
```

## docker文件丢失问题

![image-20210429141806170](https://gitee.com/hqinglau/img/raw/master/img/20210429141813.png)

## opencv ld

![image-20210501221633409](https://gitee.com/hqinglau/img/raw/master/img/20210501221633.png)

## VIA -> coco格式不标准

![image-20210502215537493](https://gitee.com/hqinglau/img/raw/master/img/20210502215537.png)

![image-20210502215555362](https://gitee.com/hqinglau/img/raw/master/img/20210502215555.png)

![image-20210502215604460](https://gitee.com/hqinglau/img/raw/master/img/20210502215604.png)

用notepad++正则处理一下。