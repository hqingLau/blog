`stat`用于获取文件信息。

```shell
struct stat {
    dev_t     st_dev;         /* ID of device containing file */
    ino_t     st_ino;         /* Inode number */
    mode_t    st_mode;        /* File type and mode */
    nlink_t   st_nlink;       /* Number of hard links */
    uid_t     st_uid;         /* User ID of owner */
    gid_t     st_gid;         /* Group ID of owner */
    dev_t     st_rdev;        /* Device ID (if special file) */
    off_t     st_size;        /* Total size, in bytes */
    blksize_t st_blksize;     /* Block size for filesystem I/O */
    blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */

    /* Since Linux 2.6, the kernel supports nanosecond
    precision for the following timestamp fields.
    For the details before Linux 2.6, see NOTES. */

    struct timespec st_atim;  /* Time of last access */
    struct timespec st_mtim;  /* Time of last modification */
    struct timespec st_ctim;  /* Time of last status change */

    #define st_atime st_atim.tv_sec      /* Backward compatibility */
    #define st_mtime st_mtim.tv_sec
    #define st_ctime st_ctim.tv_sec
};
```

这里记录一下`stat`判断文件是否是文件夹的方法，在[OSTEP中reverse反转文件的实现](https://orzlinux.cn/blog/ostep_proj2.html)也用到了`stat`判断文件节点是否是同一个文件。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>

int main(int argc,char **argv)
{
    FILE *fp = fopen(argv[1],"r");
    struct stat statbuf;
    int fileStat = stat(argv[1],&statbuf);
    // 读文件状态出错，如文件不存在，核心内存不足等等问题
    if(fileStat!=0)
    {
        printf("404\n");
        exit(1);
    }
    // 宏定义，判断是文件夹
    if(S_ISDIR(statbuf.st_mode)==1)
    {
        printf("is_dir\n");
        exit(1);
    }
    // 宏定义，判断是普通文件
    if(S_ISREG(statbuf.st_mode)==1)
    {
        printf("is_file\n");
    }
    if(fp==NULL)
    {
        printf("null\n");
        exit(1);
    }
    char buf[65];
    int nread = fread(buf,1,64,fp);
    buf[nread] = 0;
    printf("nread:%d %s\n",nread,buf);
    fclose(fp);
    return 0;
}
```

测试结果：

```shell
$ ./a.out tests  # tests是一个文件夹
is_dir
$ ./a.out test   # test不存在
404
$ ./a.out a.txt  # a.txt是一个文本文件
is_file
nread:64 aaa
bbbbbb
c
dddd

fffffffffffffffffffffffffffffffffffffffffffff
```

