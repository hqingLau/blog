## 文件

存储虚拟化两个抽象：文件和目录。在 UNIX 系统中，文件系统提供了一种统一的方式来访问磁盘、 U 盘、 CD-ROM、许多其他设备上的文件，事实上还有很多其他的东西，都位于单一目录树下。  

`O_TRUNC`：truncation（截断），截断为零字节大小，删除现有内容。

> `strace`：跟踪程序在运行时所做的每个系统调用。

```shell
$ strace -e trace=open,close,read cat foo 
open("tls/x86_64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
# ... 一些共享库
open("foo", O_RDONLY)                   = 3
read(3, "hi\n", 65536)                  = 3
hi
read(3, "", 65536)                      = 0
close(3)                                = 0
# ...
+++ exited with 0 +++
$ 

```

常用系统调用函数：

```c
int open(const char *pathname, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t count);
// write告诉文件系统，你有空了写一下。
// 性能原因，会在内存缓冲一段时间再写入磁盘
ssize_t write(int fd, const void *buf, size_t count);
// 根据偏移量定位
// lseek()调用只是在 OS 内存中更改一个变量，该变量跟踪特定进
// 程的下一个读取或写入开始的偏移量，磁头并不会真的移过去
off_t lseek(int fd, off_t offset, int whence);
// 强制写入磁盘
int fsync(int fd);
```

> C库的`fflush`写入到内核缓冲区，`fsync`把内核缓冲到磁盘上（阻塞）。
>
> C库 - ffush - 内核缓冲 - fsync - 磁盘。

```shell
$ strace mv foo foo.txt
# ... 一些操作
geteuid()                               = 1000
ioctl(0, TCGETS, {B38400 opost isig icanon echo ...}) = 0
stat("foo.txt", 0x7ffff8abd4f0)         = -1 ENOENT (No such file or directory)
lstat("foo", {st_mode=S_IFREG|0664, st_size=3, ...}) = 0
lstat("foo.txt", 0x7ffff8abd1a0)        = -1 ENOENT (No such file or directory)
renameat2(AT_FDCWD, "foo", AT_FDCWD, "foo.txt", 0) = 0
lseek(0, 0, SEEK_CUR)                   = -1 ESPIPE (Illegal seek)
close(0)                                = 0
# ...
+++ exited with 0 +++
$ 
```

> `lstat`如果是符号链接，会返回符号链接信息。

`rename`：文件重命名

```c
int rename(const char *oldpath, const char *newpath);
```

通常是一个原子调用，可用于原子文件更新，先写入临时文件，再`rename`。

```c
int fd = open("foo.txt.tmp",O_WRONLY|O_CREAT|O_TRUNC);
char buffer[64];
sprintf(buffer,"%s","haha");
write(fd,buffer,strlen(buffer));
fsync(fd);
close(fd);
rename("foo.txt.tmp","foo.txt");
```

结果：

```shell
$ gcc filedir.c 
$ ./a.out 
$ ls foo.txt*
foo.txt
$ cat foo.txt 
haha
```

获取文件信息`stat`，之前记录过，[链接](https://orzlinux.cn/blog/file_stat.html)。

```shell
$ stat foo.txt 
  File: ‘foo.txt’
  Size: 4               Blocks: 8          IO Block: 4096   regular file
Device: fc01h/64513d    Inode: 917765      Links: 1
Access: (1670/-rw-rwx--T)  Uid: ( 1000/hqinglau)   Gid: ( 1000/hqinglau)
Access: 2021-08-29 00:35:12.792958656 +0800
Modify: 2021-08-29 00:35:03.558728757 +0800
Change: 2021-08-29 00:35:03.567728981 +0800
 Birth: -
$
```

## 目录

不能直接写入目录，只能间接更新目录。

创建：`mkdir()`，创建空目录，包含两个条目，当前引用`.`和父目录`..`（点-点）。

使用：`opendir()`，`closedir()`，`readdir()`读取条目结构体`dirent`（dir entry）。

错误尝试：

```c
int fd = open("foodir",O_RDONLY);
char buf[64];
read(fd,buf,63);
printf("%s\n",buf);
// 结果为空
// write也为空，但是测试显示没报错
```

正确用法：

```c
DIR *dp = opendir("foodir");
struct dirent *d;
while ((d=readdir(dp))!=NULL) {
    printf("%d %s\n",(int)d->d_ino,d->d_name);
}
closedir(dp);
// 输出
// 917767 footxt
// 917766 .
// 917759 ..
```

目录只是将信息映射到`inode`号，具体的用`stat()`获取详细信息。

`rmdir()`要求目录是空的。

硬链接：文件系统取消链接文件时，或检查inode号的引用计数，`unlink()`删除文件会删除文件名和inode的链接，减少引用计数，删文件要用`unlink()`。

软链接：大小为文件名大小

```shell
$ ln -s foodir/footxt foolink
$ ls -l foolink 
... 13 Aug 29 01:02 foolink -> foodir/footxt
```

所以软链接可能指向空。

最后一个小练习：遍历目录下所有文件。

一个关键点就是路径问题。

```c
// 递归
void dilistdir(char *root,char *prefix)
{
    char pre[128];
    char ro[128];
    sprintf(pre,"%s%s","    ",prefix);
    DIR *dp = opendir(root);
    struct dirent *d;
    struct stat buf;
    while ((d=readdir(dp))!=NULL) {
        if(strcmp(d->d_name,".")==0||strcmp(d->d_name,"..")==0)
            continue;
        printf("%s%s\n",prefix,d->d_name);
        sprintf(ro,"%s/%s",root,d->d_name);
        stat(ro,&buf);
        if(S_ISDIR(buf.st_mode)==1)
        {
            dilistdir(ro,pre);
            continue;
        }
    }

    closedir(dp);
}


int main()
{
    printf("foodir/\n");
    dilistdir("foodir","|-- ");
    return 0;
}
```

输出示例：

```shell
$ ./a.out 
foodir/
|-- footxt
|-- foodir3
    |-- foo.txt3
|-- foodir2
    |-- foo.txt2
    |-- foo.txt22
|-- bar
```

