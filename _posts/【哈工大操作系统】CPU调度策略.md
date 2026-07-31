---
title: 【哈工大操作系统】CPU调度策略
date: 2024-5-1
tags:
  - 操作系统
---

### 如何让进程满意？

* 尽快结束任务：周转时间（从任务进入到任务结束）短
* 用户操作尽快响应：响应时间（从操作发生到响应）短
* 系统内耗时间短：吞吐量（完成的任务量）大

讲4个算法，重点是后面3个算法

1. `First Come,First Served(FCFS)`，先到先得算法，比较公平，但是不切实际，差不多意思一下
2. `SJF`，短作业优先算法，让`CPU区间`短或者说执行时间短的作业优先执行，这样子可以达到整体的平均的周转时间最短(任务的周转时间是从任务提交时开始计时，因为是多个任务同时提交，所以从第一个任务开始执行时开始计时)。为什么可以使周转时间最短，可以大致这么想，周转时间体现的是**总体进程的满意度**，在相同时间内，执行的进程越多，那么总体的满意度也就越高了。用公式证明一下
  假设最后的调度结果序列为 {p1, p2, p3, ..., p~n~}，那么 p1的周转时间为 p1; p2的周转时间为 p1+p2，p3 的周转时间为 p1+p2+p3 ; 最终 p~n~ 的周转时间为 p1+p2+p3+...+pn
  由此得出平均周转时间为
  $$(p1)+(p1+p2)+(p1+p2+p3)+... = \sum (n+1-i)p_i $$
  可以发现越靠前的任务在公式中出现的次数越多，所以只要执行时间短的任务尽量往前靠，优  先执行，就能达成平均周转时间最短 
3. `Round Robin(RR)`，**时间片**轮转调度算法，这样的算法可以实现并发，只要控制好时间片的大小，和进程的数量，那么一个进程的最大响应时间就是`(n-1) * T`，就可以控制响应时间了。可以满足受IO控制的前台任务对响应时间的需求
4. 设置优先级，让前台任务实现`RR`算法，后台任务实现`SJF`算法。前台任务的优先级高于后台任务，
  但是弊端十分明显，后台任务可能永远会不执行

我们最终的目的就是结合上面最后3个算法

1. **轮转调度**为核心，让CPU可以在前台任务和后台任务之间也可跳转，
2. 让前台任务有需要的时候能够优先执行前台任务。这里就需要设置一下优先级
3. 让后台任务能够实现短作业优先，实现平均周转时间最短

在`linux-0.11/kernel/sched.c`给出了一个调度算法

~~~C
#define TASK_RUNNING 0
#define TASK_INTERRUPTIBLE 1
#define TASK_UNINTERRUPTIBLE 2
#define TASK_ZOMBIE 3
#define TASK_STOPPED 4
#define FIRST_TASK task[0]
#define LAST_TASK task[NR_TASKS-1]

void schedule(void)
{
	int i,next,c;
	struct task_struct ** p; // p 是一个指向任务结构指针的指针
    					   // 因为下面涉及的任务数组中的元素是任务结构指针

/* check alarm, wake up any interruptible tasks that have got a signal */
	...
/* this is the scheduler proper: */

	while (1) {
		c = -1; // c接下来被定义为counter的最大值
		next = 0;
		i = NR_TASKS; // NUMBER_TASKS,任务总数
		p = &task[NR_TASKS]; // 数组最后一个元素再后一个位置
		while (--i) { // 开从后往前遍历
			if (!(*(--p))) // 如果这个数组中的元素指向的是NULL
				continue; // 就跳过这个任务
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c) {
                /* 如果任务是就绪态并且该任务的时间片比c大的话 */
                c = (*p)->counter, next = i;
                /* 就把下一个要调度的任务设置为该任务 */
            }
				
		}
		if (c) break; // 如果 c > 0 的话就确定要调度的任务
        
        /* 如果 c == 0 说明所有就绪态任务的时间片都用完了
           这时就需要重新分配一下时间片
           其中task[0]是系统的init进程，
           这个进程需要在开机时运行一次就好了
           后续的调度不需要考虑它
        */
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p) {
                (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
            }
	}
	switch_to(next);
}
~~~

~~~c
for(p = &LAST_TASK ; p > &FIRST_TASK ; --p) {
        if (*p) { // 如果*p不指向NULL的话
         	(*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
         }
}
~~~

这段代码需要单独拎出来讲，

首先，`counter`的设计满足了上面提到的第1点，前台任务和后台任务不再有特别区分，它们之间可以用时间片来实现轮转调度，同时还根据`counter`的大小来判断优先级

其次，`counter = counter / 2 + priority`的设计，让时间片的大小得到不断增加，但是有一个阈值。没错，这涉及到一个收敛的数列。
$$
C_{0}=P\\由上述代码得出递推公式:\\C_{n}=\frac{C_{n-1}}{2}+P\\变形一下，得C_{n}-2P=\frac{1}{2}(C_{n-1}-2P)\\得出一个等比数列\\令a_{n}=C_{n}-2P\\则 a_{0}=C_{0}-2P=-P\\\therefore C_{n}-2P=a_{n}=-\frac{P}{2^{n}}\\\therefore C_{n}=2P-\frac{P}{2^{n}}
$$


这样的设计不仅保证了前台任务(IO进程)要求的响应时间，同时因为前面就绪态所有的时间片都为0了，这个算法使阻塞进程的`counter`不断变大，阻塞态经历的时间越久，`counter`就会越大，优先级也就越高，由此当进程从阻塞态回来时，就可以优先执行它了，满足了上面的第2个条件

至于第3个条件，我觉得应该是使用`priority`来决定的，执行时间短的任务自然`priority`大就行了

`counter`的设计承担时间片的作用，不仅满足了轮转调度，还满足了优先级的需求
