## Linux设备分类

设备的驱动程序也要像裸机程序那样进行一些硬件操作，不同的是驱动程序需要"融合进内核里"，因此需要在驱动程序中加入操作系统规定的接口，这些接口都是独立于设备的。虽然操作系统为驱动程序设计者带来了"麻烦"，却为应用程序设计者带来了"便利"。

Linux下设备分为三类：字符设备、块设备、网络设备。

字符设备是指必须以串行顺序访问的设备，比如触屏；块设备是指可以以任意顺序访问的设备，即以块为单位进行操作，比如键盘；

字符设备不经过Cache，块设备数据经过Cache。两者的驱动程序设计差异较大。除了网络设备外，字符设备和块设备的驱动程序都被映射到文件系统中，通过调用open、read、write、close就能访问。需要说明一点，C语言的fopen、fread、fwrite、fclose实际上也是做相应的系统调用。下图是一个Linux下不同驱动种类的结构关系图：

![img](https://ask.qcloudimg.com/http-save/yehe-7043923/1g2wyvtudw.png?imageView2/2/w/1620)

## 驱动开发所需知识储备

做好驱动程序开发，需要开发者有良好的硬件基础、C语言基础、Linux内核基础以及多任务并发和控制的基础。

Linux上浏览内核源码，推荐使用的工具是vim+cscope或者vim+ctags。

## 有无操作系统的驱动程序区别

下面以led驱动为例，来说明有无操作系统的区别。

一般处理器有GPIO有两个寄存器，即控制寄存器和数据寄存器。

无操作系统时，一般需要的函数有三个，即

```javascript
LightInit()//设置控制寄存器为输出模式

LightOn()//打开Led

LightOff()//熄灭Led
```

Linux操作系统下，可以使用字符设备驱动程序框架来编写，根据Linux下的编程习惯，可以重新将其命名为light_init(),light_on,light_off

代码结构如下：

```javascript
//定义设备结构体

struct light_dev{

    struct cdev cdev;//字符设备结构体

    unsigned char value;

}

struct light_dev *light_devp;

int light_major = LIGHT_MAJOR;

 

MODULE_AUTHOR(" ");

MODULE_LICENSE(" ");

 

int light_open(struct inode *inode, struct file *filp){}

int light_release(struct inode *inode, struct file *filp){}

//读写设备

ssize_t light_read(struct file *filp, char _user *buf, size_t count, loff_t *f_pos){}

ssize_t light_write(struct file *filp, const char _user *buf, size_t count, loff_t *f_pos){}

//ioctl

int light_ioctl(struct inode *inode, struct file *flip, unsigned int cmd, unsigned long arg){}

struct file_operations light_fops={

    .ower = THIS_MODULE;

    .read = light_read;

    .write = light_write;

    .open = light_open;

    .release = light_release;

    .ioctl = light_ioctl;

}

//设置字符设备cdev结构体

static void light_setup_cdev(struct light_dev *dev, int index){}

//模块加载函数

int light_init(void)

//模块卸载函数

int light_cleanup(void)

 

module_init(light_init)

module_exit(light_cleanup)
```

这只是一个程序的结构，可以看出，与裸机的驱动程序相比，Linux下的驱动程序代码复杂很多。

## Linux设备驱动开发的硬件基础

RISC和CISC计算机的区别：RISC指令周期短，代码量大；CISC指令复杂，指令周期长，代码量小。

目前ROM基本上都使用Flash，NOR（或非）和NAND（与非）是两种主流的Flash。

NOR Flash的特点是可在芯片内执行代码，接口简单，不需要再加额外的控制芯片；NAND Flash的特点是块访问，接口需要再加入控制芯片，不能在芯片内部执行代码。NAND 的发生位反转的几率大于NOR，在使用时，应采用错误检测、错误改正算法（EDC/ECC）。Flash都是只能将1写为0，在烧写前，需要将Flash全置位，所有字节都为0xff。

DRAM以电荷形式存储在电容器中，需要定期刷新；SDRAM也是DRAM的范畴。CAM是以内容寻址的特殊RAM，输入需要查询的数据，输出数据地址和匹配标识，在数据检索中有很大优势。

*开漏输出、集电极开路输出是指需要上拉电阻才能输出高电平。

驱动工程师对硬件比IC工程师要更宏观。驱动工程师一般不需要分析时序图，但是许多企业的驱动工程师还需要承担电路板的调试工作，因此还需要了解一些电路时序的分析。

真实的电路必须满足芯片手册上的建立时间和保持时间的最低要求。查看datasheet时，没有必要通读全屏，要学会查看主要的信息内容。

## Linux内核代码结构

- arch：与不同CPU架构相关的代码
- block：块设备驱动IO调度
- crypto：相关算法，包括加密、散列、压缩、CRC校验等算法
- Document：内核各部分的注释与解释
- drivers：驱动，不同驱动有不同的子文件夹
- fs:文件系统
- include:头文件
- init:初始化相关代码
- ipc:进程通信
- kernel:内核部分
- lib:库文件代码
- mm:内存管理代码
- net:网络相关代码，实现各种协议
- scripts:配置内核脚本
- security:安全相关
- sound:音频设备驱动
- usr:用于打包压缩的cpio

Linux内核主要由进程调度、内存管理、虚拟文件系统、网络接口、进程通信组成。

## 内核空间和用户空间

CPU内部往往都实现了不同的操作模式。

比如ARM的七种工作模式：

- 用户模式（usr）绝大多数应用程序运行在此模式
- 快速中断模式（fiq）用于高速数据传输
- 外部中断模式（irp）用于通用中断处理
- 管理模式（svc）
- 数据访问模式（abt）
- 系统模式（sys）
- 未定义指令终止模式（und）

ARM+Linux采用SWI，从usr模式进入svc模式；x86处理器包含4个不同的特权级（0-3）下，Linux的用户代码运行在特权级3，系统内核运行在特权级0

Linux只能通过系统调用或者硬件中断完成从用户空间到内核空间的控制转换。

## 内核的编译与加载

在linux内核中增加程序需要完成以下3项工作：

将代码加入到linux的相应目录；

在目录的Kconfig中加入相应的编译配置选项；

在目录的Makefile中增加新项目的编译条目。

## Linux下的C编码风格

Windows下，宏全部大写，变量第一个单词小写，其后每一个单词的首字母都大写，函数名每个单词的首字母都大写。

例如：

```javascript
#define MYLINUX

int myLinux;

int MyLinux(void);
```

 而Linux 下，采用如下的风格：

```javascript
#define MYLINUX

int my_linux;

int my_linux(void);
```

Linux代码缩进使用8个字符，对于结构体、if等{不另起一行，函数另起一行。

do{}while(0)主要用于宏定义中，其使用完全是为了保证宏定义无错误的编译。

goto只用于出现错误解决错误时。

参考资料：

《Linux设备驱动开发详解》 宋宝华