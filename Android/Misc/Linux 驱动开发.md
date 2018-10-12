---
title: Linux 驱动开发
date: 2016-05-16 18:57:39
tags: [Linux, 驱动]
categories: Android
---

## 背景
本章说明里面说本章的内容会注重于 Application 层，看了几天 Binder 的源码，发现已经掉进 Library 层的坑了，悲剧的是，不看一下 Binder 驱动，貌似连 Binder 这关都过不了，没办法，只能看看驱动到底怎么写。

网上教学写入门级 Linux 驱动的例子很多，比如[【驱动】linux设备驱动·入门](http://liucw.blog.51cto.com/6751239/1218461)、[linux驱动编写（虚拟字符设备编写）](http://jp.51studyit.com/article/details/53732.htm)，这两个例子步骤比较全，但是例子本身太简单，所以又找到一枚：[Linux内存映射（mmap）](http://www.cnblogs.com/lknlfy/archive/2012/04/27/2473804.html)。这篇文章简直就是为学习Binder而生，但是它的步骤很简略。因此结合前面两个例子提供的步骤以及第三篇文章的内容，这边讲述一下最终的实践经历。<!--more-->

## 实践
__实践环境如下：Mac OS + VirtualBox 5.0.14 + Ubuntu-12.04.5__
>在Mac本身环境下貌似不行，目录结构不一致。

#### 1. 驱动程序
在Ubuntu系统桌面上新建文件`mymap.c`，将[Linux内存映射（mmap）](http://www.cnblogs.com/lknlfy/archive/2012/04/27/2473804.html)中如下内容拷贝进去:

```C
#include <linux/miscdevice.h>
#include <linux/delay.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/mm.h>
#include <linux/fs.h>
#include <linux/types.h>
#include <linux/delay.h>
#include <linux/moduleparam.h>
#include <linux/slab.h>
#include <linux/errno.h>
#include <linux/ioctl.h>
#include <linux/cdev.h>
#include <linux/string.h>
#include <linux/list.h>
#include <linux/pci.h>
#include <linux/gpio.h>

#define DEVICE_NAME "mymap"

static unsigned char array[10]={0,1,2,3,4,5,6,7,8,9};
static unsigned char *buffer;

static int my_open(struct inode *inode, struct file *file)
{
    return 0;
}


static int my_map(struct file *filp, struct vm_area_struct *vma)
{    
    unsigned long page;
    unsigned char i;
    unsigned long start = (unsigned long)vma->vm_start;
    //unsigned long end =  (unsigned long)vma->vm_end;
    unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start);

    //得到物理地址
    page = virt_to_phys(buffer);    
    //将用户空间的一个vma虚拟内存区映射到以page开始的一段连续物理页面上
    if(remap_pfn_range(vma,start,page>>PAGE_SHIFT,size,PAGE_SHARED))//第三个参数是页帧号，由物理地址右移PAGE_SHIFT得到
        return -1;

    //往该内存写10字节数据
    for(i=0;i<10;i++)
        buffer[i] = array[i];
    
    return 0;
}


static struct file_operations dev_fops = {
    .owner    = THIS_MODULE,
    .open    = my_open,
    .mmap   = my_map,
};

static struct miscdevice misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = DEVICE_NAME,
    .fops = &dev_fops,
};


static int __init dev_init(void)
{
    int ret;    
	 printk(KERN_ALERT "(init)Hello,World!\n");
    //注册混杂设备
    ret = misc_register(&misc);
    //内存分配
    buffer = (unsigned char *)kmalloc(PAGE_SIZE,GFP_KERNEL);
    //将该段内存设置为保留
    SetPageReserved(virt_to_page(buffer));

    return ret;
}

static void __exit dev_exit(void)
{
    //注销设备
    misc_deregister(&misc);
    //清除保留
    ClearPageReserved(virt_to_page(buffer));
    //释放内存
    kfree(buffer);
}

module_init(dev_init);
module_exit(dev_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("LKN@SCUT");
```
完整拷贝，不需要任何的修改。
>比起原博客的代码，在`dev_init()`函数中增加了一行代码：`printk(KERN_ALERT "(init)Hello,World!\n");`，用于检测安装。

#### 2. Makefile文件
在桌面上新建一个文件`Makefile`，内容如下：

```C
obj-m := mymap.o
PWD := $(shell pwd)
KDIR := /lib/modules/$(shell uname -r)/build
all:
    $(MAKE) -C $(KDIR) M=$(PWD) modules
```
__【注意！】__ 最后一行开头是一个Tab，不要用空格代替，实践的时候被坑了。具体问题可见[这里](http://ubuntuforums.org/showthread.php?t=1795434)。

>KDIR也有赋值为`/usr/src/linux-source-2.6.24/`的，即目录是在`/usr/src/`下面，读者可作参考，我这个目录下面是找不到source目录的。

#### 3. 配置环境
在编译出驱动文件之前，需要首先在系统命令行shell下安装当前版本的linux内核源代码：`sudo apt-get install linux-source`。

#### 4. 编译
进入桌面文件夹，执行如下命令：`make`。即可得到如下输入：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/driver-make.png" width="400" alt="Make驱动"/></div>

这个命令执行完毕后，会在桌面上生成很多文件，其中一个是`mymap.ko`，这个就是驱动。

#### 5. 安装卸载
执行`sudo insmod ./mymap.ko`命令就能安装该驱动，执行`dmesg | tail -n 1`则可以看到输入"(init)Hello, world!"的输出，如下图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/driver-install.png" width="400" alt="驱动安装"/></div>

另外我们可以执行`lsmod`命令，是可以找到一行`mymap`驱动的输出的。

卸载驱动的命令是`rmmod`。

#### 6. 验证
安装卸载中已经提到一些验证方法，但是[Linux内存映射（mmap）](http://www.cnblogs.com/lknlfy/archive/2012/04/27/2473804.html)则是用程序去验证的。

在桌面上新建一个`test.c`文件，内容如下：

```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <linux/fb.h>
#include <sys/mman.h>
#include <sys/ioctl.h> 

#define PAGE_SIZE 4096


int main(int argc , char *argv[])
{
    int fd;
    int i;
    unsigned char *p_map;
    
    //打开设备
    fd = open("/dev/mymap",O_RDWR);
    if(fd < 0)
    {
        printf("open fail\n");
        exit(1);
    }

    //内存映射
    p_map = (unsigned char *)mmap(0, PAGE_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED,fd, 0);
    if(p_map == MAP_FAILED)
    {
        printf("mmap fail\n");
        goto here;
    }

    //打印映射后的内存中的前10个字节内容
    for(i=0;i<10;i++)
        printf("%d\n",p_map[i]);
    

here:
    munmap(p_map, PAGE_SIZE);
    return 0;
}
```
不用做任何的更改。

执行命令`gcc -o test test.c`即可在桌面上生成`test`可执行文件。执行该文件，就可以得到如下输出:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/driver-verify.png" width="400" alt="驱动安装验证"/></div>
与原博客的结果一致。

>如果打印出来的是"open fail"，可以看一下权限是否足够，可以在root权限下尝试执行。

## 总结
这个实践非常简单（就是环境和执行过程坑），对于Makefile里面的配置，以及驱动的写法，这里都没有做过多的解释，但是推荐的文章里面有表述，做这个实践主要的目的是：

1. 感受一下写驱动的过程；
2. 体验一下虚拟设备（挂载、读写）、mmap内存映射；

了解这些对于理解Binder驱动非常有用，详细的驱动相关的知识可以阅读《Linux设备驱动程序》这本书。

PS：关于文件描述符和进程的关系，可以参考这篇文章: [Linux中的文件描述符与打开文件之间的关系](http://blog.csdn.net/cywosp/article/details/38965239)。