---
title: 【哈工大操作系统】如何在linux0.11上添加内核函数
date: 2024-5-26
tags:
  - Linux
  - 操作系统
sticky: 100
---

原理不多说了，直接进入正题

### 编写内核函数

这里就简单的编写一个打印函数，用到内核中的`printk()`函数，所以加上头文件

~~~C
/* linux0.11/kernel/who.c */
#include <linux/kernel.h>

int sys_whoami()
{
    printk("I'm KYZEN.\n");
    return 0;
}
~~~

编写好了一个`.c`文件，但是这样还是不够的，在最终`make`的时候还需要将其编译为`.o`文件，以供链接。

修改同目录下的`Makefile`文件，在`OBJS`下添加`who.o`，并在结尾处添加`who`相关的规则

~~~makefile
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
	panic.o printk.o vsprintf.o sys.o exit.o \
	signal.o mktime.o who.o
...
who.s who.o: who.c ../include/linux/kernel.h
~~~

这样就可以将`.c`文件变成二进制文件编入操作系统中了

### 注册内核函数

上面还只是定义内核函数，并没有告诉操作系统有这么个函数，所以还需要在系统调用表中注册这个函数。所谓注册，其实就是在头文件中添加函数的声明，这样我们在后面编写的系统调用函数能够通过这个头文件中的`sys_call_table[]`函数调用表链接到前面写的函数定义。

写在数组上面的声明主要是写给数组看的。

~~~C
/* linux0.11/include/linux/sys.h */
...
extern int sys_setreuid();
extern int sys_setregid();
extern int sys_whoami(); /* 添加内核函数声明 */

fn_ptr sys_call_table[] = { ..., sys_setreuid,sys_setregid, sys_whoami }; /* 添加函数指针 */
~~~

### 编写系统调用函数前的准备

由于系统调用函数是通过输入一个**常量符号**作为`sys_call_table[]`数组的下标去调用内核函数的，所以还需要将这个常量符号定义一下。

~~~C
/* linux0.11/include/unistd.h */
#ifdef __LIBRARY__

#define __NR_setreuid	70
#define __NR_setregid	71
#define __NR_whoami		72 /* 在这里添加常量符号 */

#define _syscall0(type,name) \
...
#define _syscall1(type,name,atype,a) \
...
#define _syscall2(type,name,atype,a,btype,b) \
...
#define _syscall3(type,name,atype,a,btype,b,ctype,c) \
...
#endif /* __LIBRARY__ */
~~~

在`int 0x80`中程序会比较系统调用号和系统调用总数的关系，所以还需要修改一下这个总数的数值

~~~assembly
/* linux0.11/kernel/system_call.s */
nr_system_calls = 72->73 /* 有几个系统调用函数就写几个 */
...
system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
~~~

### 编写系统调用函数

系统调用函数是作为调用内核函数的接口提供给用户使用的，它的作用就是走`int 0x80`然后调用相应的内核函数

这一块其实很简单，`linux`中已经为我们提供好了函数模板。就是`linux0.11/include/unistd.h`下的`_syscall0`、`_syscall1`、`_syscall3`宏函数

我们只需要调用这个宏函数就可以完成系统调用函数的定义

起初我本来想像`write()`函数一样定义在`linux0.11/lib`目录下的，但是不知道为什么会出现`error: undefined symbol _whoami referred from texe segment`报错。

没办法只能在测试程序中定义了。

~~~C
/* ~/test.c */
#define _LIBRARY_
#include <unistd.h>

_syscall0(int,whoami)
int main()
{
    whoami();
    return 0;
}
~~~

注意一定要在测试程序中定义`_LIBRARY_`，否则是找不到`unistd.h`文件中的宏定义和系统调用号

不过可能会出现系统调用号未定义的行为，这是因为`Bochs`虚拟机中的`unistd.h`没有定义，在`/usr/include/unistd.h`里面再加上就好了