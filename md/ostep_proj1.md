> OSTEP项目一：[github测试文件代码](https://github.com/remzi-arpacidusseau/ostep-projects)

代码都比较简短，都是大致原理的实现，不再写注释。

## linux cat实现

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc,char *argv[])
{
    char *buf = malloc(sizeof(char)*1025);
    size_t nread;
    ssize_t readlen;
    FILE *fp = stdin;

    int i = 1;
    if(argc==1)
    {
        while((readlen=getline(&buf,&nread,fp))>0)
        {
            buf[readlen] = 0;
            printf("%s",buf);
        }
        return 0;
    }
    for(;i<argc;i++)
    {
        fp = fopen(argv[i],"rb");
        if(fp == NULL)
        {
            printf("wcat: cannot open file\n");
            return 1;
        }

        while((readlen=getline(&buf,&nread,fp))>0)
        {
            buf[readlen] = 0;
            printf("%s",buf);
        }
        fclose(fp);
    }
    return 0;
}
```

![image-20210816162744383](https://gitee.com/hqinglau/img/raw/master/img/20210816162746.png)

## linux grep实现

这个也是比较简单，没有正则，只是简单地匹配，每行查找一个子串而已。

```c
// 同理，输入有两种形式，stdin和文件
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc,char **argv)
{
    char *buf = malloc(sizeof(char)*1025*64);
    size_t nread;
    ssize_t readlen;
    FILE *fp = stdin;

    int i = 2;
    if(argc==1)
    {
        printf("wgrep: searchterm [file ...]\n");
        return 1;
    }
    if(argc==2)
    {
        while((readlen=getline(&buf,&nread,fp))>0)
        {
            buf[readlen] = 0;
            if(strstr(buf,argv[1]))
                printf("%s",buf);
        }
        return 0;
    }
    for(;i<argc;i++)
    {
        fp = fopen(argv[i],"rb");
        if(fp == NULL)
        {
            printf("wgrep: cannot open file\n");
            return 1;
        }

        while((readlen=getline(&buf,&nread,fp))>0)
        {
            buf[readlen] = 0;
            if(strstr(buf,argv[1]))
                printf("%s",buf);
        }
        fclose(fp);
    }
    return 0;
}
```

![image-20210816164655897](https://gitee.com/hqinglau/img/raw/master/img/20210816164657.png)

## linux zip实现

有点像刷题，几个例子，盲打，几个组合拳下来才能全部通过。

```c
//这个的规则就是aaaabcc -> 4ab2c
#include <stdio.h>
#include <stdlib.h>

void diwrite(int n,char c)
{
    fwrite(&n,4,1,stdout);
    fwrite(&c,1,1,stdout);
}

int main(int argc,char *argv[])
{
    char lastc = 0;
    char curc;
    int nread;
    int count = 0;
    FILE *fp = stdin;

    int i = 1;
    if(argc==1)
    {
        printf("wzip: file1 [file2 ...]\n");
        return 1;
    }
    for(;i<argc;i++)
    {
        fp = fopen(argv[i],"rb");
        if(fp == NULL)
        {
            printf("wzip: cannot open file\n");
            return 1;
        }

        while((nread = fread(&curc,1,1,fp))==1)
        {
            if(curc==lastc)
            {
                count++;
                continue;
            }
            if(count!=0)
                diwrite(count,lastc);
            count = 1;
            lastc = curc;
        }
        fclose(fp);
    }
    diwrite(count,lastc);
    return 0;
}
```

![image-20210816172511621](https://gitee.com/hqinglau/img/raw/master/img/20210816172512.png)

