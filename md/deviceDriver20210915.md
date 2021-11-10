## rename 批量文件重命名

```shell
$ touch file{1..5}.txt                                 
$ ls                                                   
file1.txt  file2.txt  file3.txt  file4.txt  file5.txt
$ rename 's/\.txt/\.c/' *.txt                          
$ ls                                              
file1.c  file2.c  file3.c  file4.c  file5.c
$ 
```

## 添加自定义系统调用

系统调用表，用于关联系统调用号及其相对应的服务历程入口地址。例如系统调用 `read` 在系统调用表中结构如下：

![image-20210915194816123](https://gitee.com/hqinglau/img/raw/master/img/20210915194816.png)

path: /arch/x86/entry/syscalls/syscall_64.tbl(32位系统是syscall_32.tbl)

**修改源程序**

```shell
$ cd linux-4.16.1 //进入linux解压包（我下的版本是4.16.1）
$ vim arch/x86/entry/syscalls/syscall_64.tbl //进入该文件分配系统调用号 （注意别写在最后面，x64的系统调用共300多行，注意别写到后半部分的x32那一块里面）
...
333     common  io_pgetevents           sys_io_pgetevents
334     common  rseq                    sys_rseq
335     64      mysyscall               sys_mysyscall
```

**然后修改头文件**

```shell
$ vim include/linux/syscalls.h 进入该文件，添加服务例程的原型声明（shift+g快速跳到最后一行）
添加：
asmlinkage long sys_mysyscall(void);
```

**源文件**

```shell
$ vim kernel/sys.c 实现系统调用服务例程
# SYSCALL_DEFINE后的数字代表参数个数，这里0个参数（void）
SYSCALL_DEFINE0(mysyscall)
{
        printk("Hello, this is lhq's syscall test!\n");
        return 0;
}

```

**编译安装内核**

```shell
make menuconfig 配置内核
make –j2 编译内核
make modules 编译模块
make modules_install 和 make install 安装模块和安装内核
update-grub2（好像虚拟机不需要这一步）
reboot –n 立即重启
```

**新系统调用测试**

这里编写一个test.c文件来测试（文件存放位置可以任意）

```c
$ cat testMySysCall.c 
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>

int main() {
	syscall(335);
}
```


编译执行，查看信息

**dmesg**

```shell
[   88.945373] audit: type=1400 audit(1631695730.333:52): apparmor="DENIED" operation="open" profile="snap.snap-store.ubuntu-software" name="/etc/PackageKit/Vendor.conf" pid=1761 comm="snap-store" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0
[   99.626425] Hello, this is lhq's syscall test!
```

可见系统调用成功执行。

## 设备驱动程序

**设备驱动程序**的作用在于提供机制，而不是提供策略。

区分机制和策略是 Unix 设计背后隐含的最好思想之一。大部分编程问题都可以分成两部分：

机制：需要提供什么功能
策略：如何使用这些功能

我们应当尽可能做到让驱动程序不带策略。

编写驱动程序时，特别注意：编写访问硬件的内核代码时，不要给用户强加任何特定策略。因为不同的用户有不同的需求，驱动程序应该处理如何使硬件可用的问题，而将怎样使用硬件的问题留给应用程序。

驱动程序可以看作是应用程序和实际设备之间的一个软件层。

驱动程序设计要综合考虑下面三个问题的因素：提供给用户尽量多的选项、编写驱动程序要占用的时间、尽量保持程序简单。

**模块的退出函数必须撤销初始化函数所做的一切，保证没有多余内容残留在系统中。**

内核编程和应用程序编程的区别在于**对并发的处理**。内核代码不能假定给定代码段能够独占处理器。

**内核代码不能实现浮点运算。**

`current`在 <asm/current.h> 中定义，是一个指向`struct task_struct`的指针。`current`指针指向当前正在运行的进程。可以通过访问`struct task_struct`的某些成员来打印当前进程的进程 ID 和命令名：

```c
printk(KERN_INFO "The process is \"%s\" (pid %i)\n", 
        current->comm, current->pid);
```

存储在`current->comm`成员中的命令名是当前进程所执行的程序文件的基本名称，裁剪在15个字符以内。

### 简单示例

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) 
{
	printk(KERN_ALERT "hello");
	return 0;
}

static void __exit hello_exit(void)
{
	printk(KERN_ALERT "Goodbye, cruel world\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("Dual BSD/GPL");
```

**makefile**

```shell
obj-m := main.o
PWD := $(shell pwd)
KERNEL_INFO := $(uname -r)
KERNEL_LIB := /home/l/Desktop/linux-5.11.1/
# 关闭模块数字签名，因为Linux内核模块是默认需要签名的
CONFIG_MODULE_SIG=n
all:
	make -C $(KERNEL_LIB) M=$(PWD) modules
clean:
	make -C $(KERNEL_LIB) M=$(PWD) clean
```

**输出**

```shell
$ sudo insmod main.ko 
$ lsmod | grep main.
main                   16384  0
$ dmesg
[15962.061405] hello
[16164.828692] Goodbye, cruel world
$ sudo rmmod main 
$ lsmod | grep main
$ 
```

模块加载后，在 /proc/modules 文件夹下回有个目录：

```shell
l@vm:/sys/module/main$ tree -a
.
├── coresize
├── holders
├── initsize
├── initstate
├── notes
│   ├── .note.gnu.build-id
│   └── .note.Linux
├── refcnt
├── sections
│   ├── .exit.text
│   ├── .gnu.linkonce.this_module
│   ├── .init.text
│   ├── __mcount_loc
│   ├── .note.gnu.build-id
│   ├── .note.Linux
│   ├── .rodata.str1.1
│   ├── .strtab
│   └── .symtab
├── srcversion
├── taint
└── uevent

3 directories, 18 files
l@vm:/sys/module/main$ pwd
/sys/module/main
l@vm:/sys/module/main$ 
```

### 模块组成

一个Linux内核模块主要由如下几个部分组成：

模块加载函数（一般需要）当通过insmod或modprobe命令加载内核模块时，模块的加载函数会自动被内核执行，完成本模块的相关初始化工作。

模块卸载函数（一般需要）当通过rmmod命令卸载某模块时，模块的卸载函数会自动被内核执行，完成与模块卸载函数相反的功能。

模块许可证声明（必须）

许可证（LICENSE）声明描述内核模块的许可权限，如果不声明LICENSE，模块被加载时，将收到内核被污染 （kernel tainted）的警告。在Linux 2.6内核中，可接受的LICENSE包括“GPL”、“GPL v2”、“GPL and additional rights”、“Dual BSD/GPL”、“Dual MPL/GPL”和“Proprietary”。大多数情况下，内核模块应遵循GPL兼容许可权。Linux 2.6内核模块最常见的是以MODULE_LICENSE( “Dual BSD/GPL” )语句声明模块采用BSD/GPL双LICENSE。

模块参数（可选）模块参数是模块被加载的时候可以被传递给它的值，它本身对应模块内部的全局变量。

模块导出符号（可选）内核模块可以导出符号（symbol，对应于函数或变量），这样其它模块可以使用本模块中的变量或函数。

模块作者等信息声明（可选）用于申明模块作者的相关信息，一般用于备注作者姓名、邮箱等。

### 参数设置及模块信息

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("Dual BSD/GPL");

static char *book_name = "linux device Driver";
static int num = 4000;

static __init int book_init(void)
{
	printk(KERN_INFO "book name: %s\n",book_name);
	printk(KERN_INFO "book num:%d\n",num);
	return 0;
}

static __exit void book_exit(void)
{
	printk(KERN_INFO "book module exit\n");
}

module_init(book_init);
module_exit(book_exit);
module_param(num,int,S_IRUGO);
// $ sudo insmod main.ko num=12
// $ sudo dmesg -c
// [19497.624466] book name: linux device Driver
// [19497.624470] book num:12


// $ ls /sys/module/main/parameters/num -al
// -r--r--r-- 1 root root 4096 9月  15 22:14 /sys/module/main/parameters/num
// Try 'cat --help' for more information.
// $ cat /sys/module/main/parameters/num
// 12

module_param(book_name,charp,S_IRUGO);
MODULE_AUTHOR("yadi, yadi@gmail.com");
MODULE_DESCRIPTION("A simple module for testing module params");
MODULE_VERSION("V1.0");

// $ modinfo main.ko
// filename:       /home/l/deviceDriver/main.ko
// version:        V1.0
// description:    A simple module for testing module params
// author:         yadi, yadi@gmail.com
// license:        Dual BSD/GPL
// srcversion:     BF9E9FC89C53701BFCB41FE
// depends:        
// retpoline:      Y
// name:           main
// vermagic:       5.11.1 SMP mod_unload modversions 
// parm:           num:int
// parm:           book_name:charp
```

### 字符设备驱动

字符设备是3大类设备（字符设备、块设备、网络设备）中较简单的一类设备、其驱动程序中完成的主要工作是初始化、添加和删除 `struct cdev` 结构体，申请和释放设备号，以及填充 `struct file_operations` 结构体中断的操作函数，**实现 `struct file_operations` 结构体中的`read()`、`write()`和`ioctl()`等函数是驱动设计的主体工作**。

如图，在 Linux 内核代码中：

- 使用`struct cdev`结构体来抽象一个字符设备；
- 通过一个`dev_t`类型的设备号（分为主（major）、次设备号（minor））一确定字符设备唯一性；
- 通过`struct file_operations`类型的操作方法集来定义字符设备提供 VFS 的接口函数。 

![image-20210915222806670](https://gitee.com/hqinglau/img/raw/master/img/20210915222806.png)

具体如图：

![image-20210915222921472](https://gitee.com/hqinglau/img/raw/master/img/20210915222921.png)

使用`cdev`结构体来描述一个字符设备：

```c
struct cdev {
	struct kobject kobj; // 内嵌的内核对象
	struct module *owner; // 该字符设备所在的内核模块（所有者）的对象指针，一般为THIS_MODULE主要用于模块计数
	const struct file_operations *ops; // 该结构描述了字符设备所能实现的操作集（打开、关闭、读/写、...），是极为关键的一个结构体
	struct list_head list; // //用来将已经向内核注册的所有字符设备形成链表
	dev_t dev; //  //字符设备的设备号，由主设备号和次设备号构成（如果是一次申请多个设备号，此设备号为第一个）
	unsigned int count; //隶属于同一主设备号的次设备号的个数
} __randomize_layout;

void cdev_init(struct cdev *, const struct file_operations *);
// 动态申请（构造）cdev内存（设备对象）
struct cdev *cdev_alloc(void);
// 释放cdev内存
void cdev_put(struct cdev *p);
// 注册cdev设备对象（添加到系统字符设备列表中）
int cdev_add(struct cdev *, dev_t, unsigned);

void cdev_set_parent(struct cdev *p, struct kobject *kobj);
int cdev_device_add(struct cdev *cdev, struct device *dev);
void cdev_device_del(struct cdev *cdev, struct device *dev);
// 将cdev对象从系统中移除（注销 ）
void cdev_del(struct cdev *);

void cd_forget(struct inode *);
```

一个字符设备或块设备都有一个主设备号和一个次设备号。**主设备号用来标识与设备文件相连的驱动程序**，用来反映设备类型。**次设备号**被驱动程序用来辨别操作的是哪个设备，用来区分同类型的设备。

编写一个**字符设备驱动**，并利用对字符设备的同步操作，设计实现一个聊天程序。可以有一个读，一个写进程共享该字符设备，进行聊天；也可以由多个读和多个写进程共享该字符设备，进行聊天。代码和注释来自[Linux字符设备驱动实现](https://www.cnblogs.com/yueshangzuo/p/8078687.html)。

```c
/*
参考：深入浅出linux设备驱动开发
*/
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
#include <linux/wait.h>
#include <linux/semaphore.h>
#include <linux/sched.h>
#include <linux/cdev.h>
#include <linux/types.h>
#include <linux/kdev_t.h>
#include <linux/device.h>

#define MAXNUM 100
#define MAJOR_NUM 506 //主设备号 ,没有被使用

struct dev{
    struct cdev devm; //字符设备
    struct semaphore sem;
    wait_queue_head_t outq;//等待队列,实现阻塞操作
    int flag; //阻塞唤醒标志
    char buffer[MAXNUM+1]; //字符缓冲区
    char *rd,*wr,*end; //读,写,尾指针
}globalvar;
static struct class *my_class;
int major=MAJOR_NUM;

static ssize_t globalvar_read(struct file *,char *,size_t ,loff_t *);
static ssize_t globalvar_write(struct file *,const char *,size_t ,loff_t *);
static int globalvar_open(struct inode *inode,struct file *filp);
static int globalvar_release(struct inode *inode,struct file *filp);
/*
结构体file_operations在头文件 linux/fs.h中定义，用来存储驱动内核模块提供的对设备进行各种操作的函数的指针。
该结构体的每个域都对应着驱动内核模块用来处理某个被请求的事务的函数的地址。
设备"gobalvar"的基本入口点结构变量gobalvar_fops 
*/
struct file_operations globalvar_fops =
{
    /*
    标记化的初始化格式这种格式允许用名字对这类结构的字段进行初始化,这就避免了因数据结构发生变化而带来的麻烦。
    这种标记化的初始化处理并不是标准 C 的规范,而是对 GUN 编译器的一种(有用的)特殊扩展
    */
    //用来从设备中获取数据. 在这个位置的一个空指针导致 read 系统调用以 -EINVAL("Invalid argument") 失败. 一个非负返回值代表了成功读取的字节数( 返回值是一个 "signed size" 类型, 常常是目标平台本地的整数类型).
    .read=globalvar_read,
    //发送数据给设备. 如果 NULL, -EINVAL 返回给调用 write 系统调用的程序. 如果非负, 返回值代表成功写的字节数.
    .write=globalvar_write,
    //尽管这常常是对设备文件进行的第一个操作, 不要求驱动声明一个对应的方法. 如果这个项是 NULL, 设备打开一直成功, 但是你的驱动不会得到通知.
    .open=globalvar_open,
    //当最后一个打开设备的用户进程执行close ()系统调用时，内核将调用驱动程序的release () 函数：release 函数的主要任务是清理未结束的输入/输出操作、释放资源、用户自定义排他标志的复位等。
    .release=globalvar_release,
};
//内核模块的初始化
static int globalvar_init(void)
{

    /*
    int register_chrdev(unsigned int major, unsigned int baseminor,unsigned int count, const char *name,const struct file_operations *fops)
    返回值提示操作成功还是失败。负的返回值表示错误;0 或正的返回值表明操作成功。
    major参数是被请求的主设备号,name 是设备的名称,该名称将出现在 /proc/devices 中, 
    fops是指向函数指针数组的指针,这些函数是调用驱动程序的入口点,
    在 2.6 的内核之后，新增了一个 register_chrdev_region 函数，
    它支持将同一个主设备号下的次设备号进行分段，每一段供给一个字符设备驱动程序使用，使得资源利用率大大提升，
    */    
    int result = 0;
    int err = 0;
    /*
    宏定义：#define MKDEV(major,minor) (((major) << MINORBITS) | (minor))
    成功执行返回dev_t类型的设备编号，dev_t类型是unsigned int 类型，32位，用于在驱动程序中定义设备编号，
    高12位为主设备号，低20位为次设备号,可以通过MAJOR和MINOR来获得主设备号和次设备号。
    在module_init宏调用的函数中去注册字符设备驱动
    major传0进去表示要让内核帮我们自动分配一个合适的空白的没被使用的主设备号
    内核如果成功分配就会返回分配的主设备号；如果分配失败会返回负数
    */
    dev_t dev = MKDEV(major, 0);
    if(major)
    {
        //静态申请设备编号
        result = register_chrdev_region(dev, 1, "charmem");
    }
    else
    {
        //动态分配设备号
        result = alloc_chrdev_region(&dev, 0, 1, "charmem");
        major = MAJOR(dev);
    }
    if(result < 0)
        return result;
    /*
    file_operations这个结构体变量，让cdev中的ops成员的值为file_operations结构体变量的值。
    这个结构体会被cdev_add函数想内核注册cdev结构体，可以用很多函数来操作他。
    如：
    cdev_alloc：让内核为这个结构体分配内存的
    cdev_init：将struct cdev类型的结构体变量和file_operations结构体进行绑定的
    cdev_add：向内核里面添加一个驱动，注册驱动
    cdev_del：从内核中注销掉一个驱动。注销驱动
    */
    //注册字符设备驱动，设备号和file_operations结构体进行绑定
    cdev_init(&globalvar.devm, &globalvar_fops);
    /*
    #define THIS_MODULE (&__this_module)是一个struct module变量，代表当前模块，
    与那个著名的current有几分相似，可以通过THIS_MODULE宏来引用模块的struct module结构，
    比如使用THIS_MODULE->state可以获得当前模块的状态。
    现在你应该明白为啥在那个岁月里，你需要毫不犹豫毫不迟疑的将struct usb_driver结构里的owner设置为THIS_MODULE了吧，
    这个owner指针指向的就是你的模块自己。
    那现在owner咋就说没就没了那？这个说来可就话长了，咱就长话短说吧。
    不知道那个时候你有没有忘记过初始化owner，
    反正是很多人都会忘记，
    于是在2006年的春节前夕，在咱们都无心工作无心学习等着过春节的时候，Greg坚守一线，去掉了 owner，
    于是千千万万个写usb驱动的人再也不用去时刻谨记初始化owner了。
    咱们是不用设置owner了，可core里不能不设置，
    struct usb_driver结构里不是没有owner了么，
    可它里面嵌的那个struct device_driver结构里还有啊，设置了它就可以了。
    于是Greg同时又增加了usb_register_driver()这么一层，
    usb_register()可以通过将参数指定为THIS_MODULE去调用它，所有的事情都挪到它里面去做。
    反正usb_register() 也是内联的，并不会增加调用的开销。
    */
    globalvar.devm.owner = THIS_MODULE;
    err = cdev_add(&globalvar.devm, dev, 1);
    if(err)
        printk(KERN_INFO "Error %d adding char_mem device", err);
    else
    {
        printk("globalvar register success\n");
        sema_init(&globalvar.sem,1); //初始化信号量
        init_waitqueue_head(&globalvar.outq); //初始化等待队列
        globalvar.rd = globalvar.buffer; //读指针
        globalvar.wr = globalvar.buffer; //写指针
        globalvar.end = globalvar.buffer + MAXNUM;//缓冲区尾指针
        globalvar.flag = 0; // 阻塞唤醒标志置 0
    }
    /*
    定义在/include/linux/device.h
    创建class并将class注册到内核中，返回值为class结构指针
    在驱动初始化的代码里调用class_create为该设备创建一个class，再为每个设备调用device_create创建对应的设备。
    省去了利用mknod命令手动创建设备节点
    */
    my_class = class_create(THIS_MODULE, "chardev0");
    device_create(my_class, NULL, dev, NULL, "chardev0");
    return 0;
}
/*
在大部分驱动程序中,open 应完成如下工作:
● 递增使用计数。--为了老版本的可移植性
● 检查设备特定的错误(诸如设备未就绪或类似的硬件问题)。
● 如果设备是首次打开,则对其进行初始化。
● 识别次设备号,并且如果有必要,更新 f_op 指针。
● 分配并填写被置于 filp->private_data 里的数据结构。
*/
static int globalvar_open(struct inode *inode,struct file *filp)
{
    try_module_get(THIS_MODULE);//模块计数加一
    printk("This chrdev is in open\n");
    return(0);
}
/*
release都应该完成下面的任务:
● 释放由 open 分配的、保存在 filp->private_data 中的所有内容。
● 在最后一次关闭操作时关闭设备。字符设备驱动程序
● 使用计数减 1。
如果使用计数不归0,内核就无法卸载模块。
并不是每个 close 系统调用都会引起对 release 方法的调用。
仅仅是那些真正释放设备数据结构的 close 调用才会调用这个方法,
因此名字是 release 而不是 close。内核维护一个 file 结构被使用多少次的计数器。
无论是 fork 还是 dup 都不创建新的数据结构(仅由 open 创建),它们只是增加已有结构中的计数。
*/
static int globalvar_release(struct inode *inode,struct file *filp)
{
    module_put(THIS_MODULE); //模块计数减一
    printk("This chrdev is in release\n");
    return(0);
}
static void globalvar_exit(void)
{
    device_destroy(my_class, MKDEV(major, 0));
    class_destroy(my_class);
    cdev_del(&globalvar.devm);
    /*
    参数列表包括要释放的主设备号和相应的设备名。
    参数中的这个设备名会被内核用来和主设备号参数所对应的已注册设备名进行比较,如果不同,则返回 -EINVAL。
    如果主设备号超出了所允许的范围,则内核同样返回 -EINVAL。
    */
    unregister_chrdev_region(MKDEV(major, 0), 1);//注销设备
}
/*
ssize_t read(struct file *filp, char *buff,size_t count, loff_t *offp);
参数 filp 是文件指针,参数 count 是请求传输的数据长度。
参数 buff 是指向用户空间的缓冲区,这个缓冲区或者保存将写入的数据,或者是一个存放新读入数据的空缓冲区。
最后的 offp 是一个指向“long offset type(长偏移量类型)”对象的指针,这个对象指明用户在文件中进行存取操作的位置。
返回值是“signed size type(有符号的尺寸类型)”

主要问题是,需要在内核地址空间和用户地址空间之间传输数据。
不能用通常的办法利用指针或 memcpy来完成这样的操作。由于许多原因,不能在内核空间中直接使用用户空间地址。
内核空间地址与用户空间地址之间很大的一个差异就是,用户空间的内存是可被换出的。
当内核访问用户空间指针时,相对应的页面可能已不在内存中了,这样的话就会产生一个页面失效
*/
static ssize_t globalvar_read(struct file *filp,char *buf,size_t len,loff_t *off)
{
    if(wait_event_interruptible(globalvar.outq,globalvar.flag!=0)) //不可读时 阻塞读进程
    {
        return -ERESTARTSYS;
    }
    /*
    down_interruptible 可以由一个信号中断,但 down 不允许有信号传送到进程。
    大多数情况下都希望信号起作用;否则,就有可能建立一个无法杀掉的进程,并产生其他不可预期的结果。
    但是,允许信号中断将使得信号量的处理复杂化,因为我们总要去检查函数(这里是 down_interruptible)是否已被中断。
    一般来说,当该函数返回 0 时表示成功,返回非 0 时则表示出错。
    如果这个处理过程被中断,它就不会获得信号量 , 因此,也就不能调用 up 函数了。
    因此,对信号量的典型调用通常是下面的这种形式:
    if (down_interruptible (&sem))
        return -ERESTARTSYS;
    返回值 -ERESTARTSYS通知系统操作被信号中断。
    调用这个设备方法的内核函数或者重新尝试,或者返回 -EINTR 给应用程序,这取决于应用程序是如何设置信号处理函数的。
    当然,如果是以这种方式中断操作的话,那么代码应在返回前完成清理工作。

    使用down_interruptible来获取信号量的代码不应调用其他也试图获得该信号量的函数,否则就会陷入死锁。
    如果驱动程序中的某段程序对其持有的信号量释放失败的话(可能就是一次出错返回的结果),
    那么其他任何获取该信号量的尝试都将阻塞在那里。
    */
    if(down_interruptible(&globalvar.sem)) //P 操作
    {
        return -ERESTARTSYS;
    }
    globalvar.flag = 0;
    printk("into the read function\n");
    printk("the rd is %c\n",*globalvar.rd); //读指针
    if(globalvar.rd < globalvar.wr)
        len = min(len,(size_t)(globalvar.wr - globalvar.rd)); //更新读写长度
    else
        len = min(len,(size_t)(globalvar.end - globalvar.rd));
    printk("the len is %d\n",len);
    /*
    read 和 write 代码要做的工作,就是在用户地址空间和内核地址空间之间进行整段数据的拷贝。
    这种能力是由下面的内核函数提供的,它们用于拷贝任意的一段字节序列,这也是每个 read 和 write 方法实现的核心部分:
    unsigned long copy_to_user(void *to, const void *from,unsigned long count);
    unsigned long copy_from_user(void *to, const void *from,unsigned long count);
    虽然这些函数的行为很像通常的 memcpy 函数,但当在内核空间内运行的代码访问用户空间时,则要多加小心。
    被寻址的用户空间的页面可能当前并不在内存,于是处理页面失效的程序会使访问进程转入睡眠,直到该页面被传送至期望的位置。
    例如,当页面必须从交换空间取回时,这样的情况就会发生。对于驱动程序编写人员来说,
    结果就是访问用户空间的任何函数都必须是可重入的,并且必须能和其他驱动程序函数并发执行。
    这就是我们使用信号量来控制并发访问的原因.
    这两个函数的作用并不限于在内核空间和用户空间之间拷贝数据,它们还检查用户空间的指针是否有效。
    如果指针无效,就不会进行拷贝;另一方面,如果在拷贝过程中遇到无效地址,则仅仅会复制部分数据。
    在这两种情况下,返回值是还未拷贝完的内存的数量值。
    如果发现这样的错误返回,就会在返回值不为 0 时,返回 -EFAULT 给用户。
    负值意味着发生了错误,该值指明发生了什么错误,错误码在<linux/errno.h>中定义。
    比如这样的一些错误:-EINTR(系统调用被中断)或 -EFAULT (无效地址)。
    */
    if(copy_to_user(buf,globalvar.rd,len))
    {
        printk(KERN_ALERT"copy failed\n");
        /*
        up递增信号量的值,并唤醒所有正在等待信号量转为可用状态的进程。
        必须小心使用信号量。被信号量保护的数据必须是定义清晰的,并且存取这些数据的所有代码都必须首先获得信号量。
        */
        up(&globalvar.sem);
        return -EFAULT;
    }
    printk("the read buffer is %s\n",globalvar.buffer);
    globalvar.rd = globalvar.rd + len;
    if(globalvar.rd == globalvar.end)
        globalvar.rd = globalvar.buffer; //字符缓冲区循环
    up(&globalvar.sem); //V 操作
    return len;
}
static ssize_t globalvar_write(struct file *filp,const char *buf,size_t len,loff_t *off)
{
    if(down_interruptible(&globalvar.sem)) //P 操作
    {
        return -ERESTARTSYS;
    }
    if(globalvar.rd <= globalvar.wr)
        len = min(len,(size_t)(globalvar.end - globalvar.wr));
    else
        len = min(len,(size_t)(globalvar.rd-globalvar.wr-1));
    printk("the write len is %d\n",len);
    if(copy_from_user(globalvar.wr,buf,len))
    {
        up(&globalvar.sem); //V 操作
        return -EFAULT;
    }
    printk("the write buffer is %s\n",globalvar.buffer);
    printk("the len of buffer is %d\n",strlen(globalvar.buffer));
    globalvar.wr = globalvar.wr + len;
    if(globalvar.wr == globalvar.end)
    globalvar.wr = globalvar.buffer; //循环
    up(&globalvar.sem);
    //V 操作
    globalvar.flag=1; //条件成立,可以唤醒读进程
    wake_up_interruptible(&globalvar.outq); //唤醒读进程
    return len;
}
module_init(globalvar_init);
module_exit(globalvar_exit);
MODULE_LICENSE("GPL");
```

**设备查看**：

```shell
$ cat /proc/devices 
Character devices:
  1 mem
  4 /dev/vc/0
  4 tty
  4 ttyS
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  5 ttyprintk
  6 lp
...
506 charmem
# c中 #define MAJOR_NUM 506 //主设备号 ,没有被使用
Block devices:
  7 loop
  8 sd
...

$ ls /dev/chardev0 -al
crw------- 1 root root 506, 0 9月  15 22:48 /dev/chardev0
```

**写入**：

```shell
$ sudo ./test_write 
[sudo] password for l: 
Please input the globar:
yadi
Please input the globar:
lei yadi
Please input the globar:
Please input the globar:
love
Please input the globar:
l 
Please input the globar:
exit
Please input the globar:
quit
$
```

**读**：

```shell
$ sudo ./test_read 
[sudo] password for l: 
yadi
leiyadi
love
l
exit
quit
$
```



## 参考

[linux内核编译及添加系统调用(详细版)](https://blog.csdn.net/weixin_43641850/article/details/104906726)

[Linux设备驱动程序学习----目录](https://www.jianshu.com/p/c71eef23873c)

[linux内核驱动模块开发步骤及实例入门介绍](https://blog.csdn.net/LEON1741/article/details/78165507)

[Linux字符设备驱动](https://www.cnblogs.com/chen-farsight/p/6155518.html)

[Linux字符设备驱动实现](https://www.cnblogs.com/yueshangzuo/p/8078687.html)


