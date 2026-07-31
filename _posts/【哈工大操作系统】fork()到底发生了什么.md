---
title: 【哈工大操作系统】fork()到底发生了什么
date: 2024-5-12
tags:
  - 操作系统
sticky: 110
---

`fork()`函数起源于`linux-0.11`下的`main()`函数中

~~~C
// linux-0.11/init/main.c

static inline _syscall0(int,fork)
void main(void)
{
    /* ... */
    if (!fork()) {
        init();
    }
    for(;;) pause();
}
~~~

在这里调用了`fork()`函数，但是`fork()`具体长什么样，将上面的宏展开也就得出了答案

`_syscall0(type,name)`的宏定义如下：

~~~C
// linux-0.11/include/unistd.h

int fork(void);

#define __NR_fork	2

#define _syscall0(type,name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name)); \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}
~~~

简单代入一下，`fork()`这个系统调用函数也就长这样

~~~C
int fork(void)
{
    long __res;
    __asm__ volatile (
        "int $0x80"
        : "=a" (__res)
        : "0" (__NR_fork)
    );
    if (__res >= 0)
        return (int) __res;
    errno = -__res;
    return -1;
}
~~~

核心就是用了`int 0x80`中断，传入`__NR_fork`作为参数，并将结果通过`eax`寄存器传给变量`__res`

`int 0x80`中断会去调用`system_call`函数，但是在这之前还有更重要但是容易忽略的过程——**保护现场**。这个过程由`CPU`硬件自动完成，不需要我们操心，它所保存的变量在后面会有大用，下面来详细说说这个过程。

每个进程在被创建时，都会创建一个`TSS`和一个`内核栈`在内存当中，`TSS`中会有变量`es0`和`ss0`指向这个`内核栈`。触发中断时，`CPU`会根据`TR`寄存器在`GDT`表中找到`TSS段描述符`，然后根据这个段描述符找到当前进程的`TSS`。`CPU`会在`TSS`中在相应位置检索出`esp0`和`ss0`，找到其指向的内核栈，接下来做的就是**保护现场**，将`ss`、`esp`、`EFLAGS`、`cs`、`eip`这五个寄存器的值一次存入内核栈中，然后修改`ss:esp`为当前内核栈。由此，不仅在内核栈当中保存了用户栈的地址，还将栈从用户栈切换到内核栈。接下来，`int 0x80`就会跳转到`system_call`这个内核函数。正式进入了**内核态**。

~~~assembly
; linux-0.11/kernel/system_call.s

_system_call:
	push ds, es, fs
	pushl edx, ecx, ebx
	call _sys_fork
	pushl eax
	
; 调度函数	
	movl _current, eax
	cmpl $0, state(eax)
	jne reschedule
	cmpl $0, counter(eax)
	je reschedule
	
ret_from_sys_call:
	; ...
	popl eax
	popl ebx, ecx, edx
	pop fs, es, ds
	iret
	
_sys_fork:
	; ...
	push gs
	push esi, edi, ebp, eax
	call _copy_process
	addl $20, esp ; 这一步很妙，用加法相当于弹出了20个寄存器
	ret
~~~

从上面可知，`_system_call`跳转到`_sys_fork`后又马上调用了`_copy_process`这个函数

~~~C
// linux-0.11/kernel/fork.c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
    struct task_struct* p;
    /* .. */
    p = (struct task_struct*)get_free_page(); // 申请了一个Page，然后以task_struct的方式对待它，剩下的内存拿来当内核栈了
    if (!p)
        return -EAGAIN;
    
    /* ... */
    
    // tss指向子进程内核栈
    p->tss.esp0 = PAGE_SIZE + (long)p;
    p->tss.ss0 = 0x10; // 内核数据段
    
    /* ... */
    
    // tss设置用户栈(与父进程共用)
    p->tss.esp = esp;
    p->tss.ss = ss & 0xffff;
    
    /* ... */
    
    p->tss.ip = ip;
    p->tss.cs = cs & 0xffff;
    
    /* ... */
    
    p->tss.ecx = ecx;
    p->tss.eax = 0;
    
    /* 内存操作... */
    
    p->state = TASK_RUNNING;
    return last_pid; // 返回子进程的pid
}
// p->tss作为任务状态段,其实任务状态寄存器TR指向的地方就是这里,就是get_free_page()得到的地址
~~~

这个函数需要一堆参数，从哪来呢。回顾一下，在调用`_sys_fork`之前，我们 push 了一堆寄存器，包括还原现场时保存的寄存器，这些都是当前父进程的状态，作为这个函数的参数。这个函数具体做的就是为子进程申请一块空间，然后把里面的父进程的当前的状态(所有寄存器的值)拷贝给子进程`TSS`，这样子进程与父进程的状态大体一致，指向的用户栈也是同一个。但是后来子进程又做了一些自定义操作，比如把`eax`置为 0，用来当作`fork()`函数的返回值。**在这个过程中，感觉老师上课讲的什么子进程内核栈拷贝父进程的内核栈可能是错的说法，正确说法应该是，子进程的`TSS`以父进程的内核栈为媒介拷贝父进程在用户态时的`TSS`。此时子进程的内核栈应该是空的，而且子进程起手就是在用户态运行。**（后记：这段代码是原先linux0.11根据TSS做进程切换需要而创建进程的方式，但是后来老师提出的切换内核栈做到进程切换的方式不再这样做，而是通过`*(--kernstack) = register`方式做内核栈的`push`，所以老师说的其实是对的。

> 在C语言中，函数参数是从右往左依次入栈的，换句话说右边的参数所在地址会高一点

在这个函数前面申请内存那块，如果申请失败了，子进程创建失败，那么返回`-EAGAIN`放在`eax`中，这个值定义在`linux-0.11/include/errno.h`中`#define EAGAIN    11`，可见是个负值。

`_sys_fork`执行好了之后，就会执行下面的调度函数。如果前面子进程创建成功，调度函数会把当前进程调度为子进程，此时`eax`存储的是 0；如果创建失败，调度函数仍然调度父进程，此时`eax`存储 -11。最后`eax`的值会保存在`fork()`函数中的`__res`变量中用作返回值

这样，回到原来`if(!fork())`进行判断时，就能根据返回的结果决定走`if`里的语句作为子进程，还是跳过继续执行父进程。



将`_sys_fork`函数的作用概括一下：

将当前父进程所有寄存器的值入栈，作为函数参数赋值给创建的子进程的`TSS`，注意这个创建的子进程还没立刻执行，它只不过是先被创建了放在那，只有在进程调度后才会被执行，虽然父进程和子进程谁先被调度不知道但总归是会被执行的。在`copy_process()`中时，`CPU`仍然还在为父进程服务。

在进入`copy_process()`前，父进程的内核栈的`cs:ip`指向的是`fork()`函数中的指令，大致应该是`mov [__res], eax`。而区分父进程还是父进程的关键点就在这。

对于父进程来说，`CPU`在执行`copy_process()`函数时服务的进程还是父进程，这个函数执行完后会`return last_pid`即将子进程的`pid`存入`eax`中。而父进程退出内核态返回用户态时执行的第一条指令就是`mov [__res], eax`，这就使得`fork()`的返回值是子进程的`pid`。

当然了，如果子进程创建失败了，`eax`里存的就是一个负值，`fork()`的返回值也变成负值

对于子进程来说，`CPU`在父进程中执行`copy_process()`函数时，就将子进程的`eax`强制修改为了`0`，在调度到子进程执行时，因为子进程`TSS`里的`cs:ip`是父进程很早就压栈进来的，子进程的第一条指令就是`mov [__res], eax`这一条在用户态的代码，所以此时`fork()`的返回值就是`0`

完。