---
title: Makefile学习
date: 2024-5-24
tags:
  - C/C++
---

# Makefile学习笔记



## 1个规则

~~~makefile
目标: 依赖条件
[tab] 命令
~~~

~~~makefile
hello: hello.c
	gcc -o hello hello.c
~~~

基本原则：

1. 若想生成**目标**，检查**规则**中的**依赖条件**是否存在，如不存在，则寻找是否有**规则**用来生成该**依赖条件**
2. 检查规则中的**目标**是否需要更新，必须先检查它的所有**依赖**，**依赖**中有任意一个被更新，则**目标**才能更新
	* 分析各个目标和依赖之间的关系
	* 根据依赖关系自底向上执行命令
	* 根据**依赖条件**和**目标**的修改时间来判断**目标**是否需要更新
	* 如果目标不依赖任何条件，则执行对应命令，以示更新
3. `make`工具检查`makefile`文件，默认将其中第1条规则的目标作为**终极目标**
	除非用 `ALL` 命令指定终极目标
	`make`生成**终极目标**就自动退出了

~~~makefile
ALL: main
hello.o: hello.c
     gcc -c hello.c -o hello.o
add.o: add.c
     gcc -c add.c -o add.o
sub.o: sub.c
     gcc -c sub.c -o sub.o
div.o: div.c
     gcc -c div.c -o div.o
main: hello.o add.o sub.o div.o
     gcc hello.o add.o sub.o div.o -o main 
~~~



## 2个函数

### $(wildcard *.c)

~~~makefile
src = $(wildcard *.c) # src = hello.c add.c sub.c div.c
~~~

匹配当前工作目录下所有`.c`文件。将文件名组成列表，赋值给变量 src

### $(patsubst %.c, %.o,\$(src))

~~~makefile
obj = $(patsubst %.c, %.o,$(src)) # obj = hello.o add.o sub.o div.o
~~~

将变量**参数3**中包含**参数1**的部分替换为**参数2**，并赋值给变量 `obj`

~~~makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL: main

main: $(obj)
	gcc hello.o add.o sub.o div.o -o main 
hello.o: hello.c
	gcc -c hello.c -o hello.o
add.o: add.c
	gcc -c add.c -o add.o
sub.o: sub.c
	gcc -c sub.c -o sub.o
div.o: div.c
	gcc -c div.c -o div.o
~~~

### clean

~~~makefile
clean: (没有依赖)
	-rm -rf $(obj) main # rm前的'-'作用是删除不存在文件时不报错，继续执行
~~~



## 3个自动变量

| 自动变量 |                             作用                             |
| :------: | :----------------------------------------------------------: |
|    $@    |               在规则的命令中，表示规则中的目标               |
|    $<    |          在规则的命令中，表示规则中的第一个依赖条件          |
|    $^    | 在规则的命令中，表示规则中的所有依赖条件，组成一个列表，用空格隔开，删除重复项 |

~~~makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL: main

main: $(obj)
	gcc $^ -o $@
hello.o: hello.c
	gcc -c $< -o $@
add.o: add.c
	gcc -c $< -o $@
sub.o: sub.c
	gcc -c $< -o $@
div.o: div.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) main
~~~



## 模式规则

将上面一组相似的规则定义了一个模式

~~~makefile
%.o: %.c
	gcc -c $< -o $@
~~~



~~~makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL: main

main: $(obj)
	gcc $^ -o $@

%.o: %.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) main
~~~

~~~shell
> make
gcc -c hello.c -o hello.o
gcc -c add.c -o add.o
gcc -c div.c -o div.o
gcc -c sub.c -o sub.o
gcc hello.o add.o div.o sub.o -o main
>
~~~

这样可以随意的添加或删除一些源文件而不需要去动`makefile`

## 静态模式规则

给一个变量指定了一个模式规则

~~~makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL: main

main: $(obj)
	gcc $^ -o $@

$(obj): %.o: %.c
	gcc -c $< -o $@

%.o: %.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) main
~~~



## 伪目标

不管目标文件是否是最新的，都再去执行一次**目标**的**规则**

~~~makefile
.PHONY: clean
clean:
	-rm -rf $(obj) main
~~~



## 其它

还可以用变量去定义一些在命令中会用到的参数

~~~makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

FLAGS = -Wall -g

ALL: main

main: $(obj)
	gcc $^ -o $@ $(FLAGS)

$(obj): %.o: %.c
	gcc -c $< -o $@ $(FLAGS)

clean:
	-rm -rf $(obj) main
~~~



## 小测试

小测试的项目结构

~~~shell
maketest
├── include
│   └── head.h
├── Makefile
├── obj
└── src
    ├── add.c
    ├── div.c
    ├── main.c
    └── sub.c

3 directories, 6 files
~~~

`Makefile`文件内容

~~~makefile
src = $(wildcard ./src/*.c)
obj = $(patsubst ./src/%.c, ./obj/%.o, $(src))

INCLUDE_PATH = ./include

ALL: main

main: $(obj) # 链接阶段就不需要include了
	gcc $^ -o $@

$(obj): ./obj/%.o: ./src/%.c # 编译阶段需要include
	gcc -c $< -I $(INCLUDE_PATH) -o $@

.PHONY: clean
clean:
	-rm -rf $(obj) main
~~~

`main.c`文件内容

~~~c
#include "head.h"
int main()
{
	int a = 10, b = 2;
	printf("%d+%d=%d\n", a, b, add(a, b));
	printf("%d-%d=%d\n", a, b, sub(a, b));
	printf("%d/%d=%d\n", a, b, div(a, b));
	return 0;
}
~~~

`head.h`文件内容

~~~c
#ifndef _HEAD_H_
#define _HEAD_H_

#include <stdio.h>

int add(int, int);
int sub(int, int);
int div(int, int);

#endif
~~~

执行

~~~shell
> make
gcc -c src/main.c -I ./include -o obj/main.o
gcc -c src/add.c -I ./include -o obj/add.o
gcc -c src/div.c -I ./include -o obj/div.o
gcc -c src/sub.c -I ./include -o obj/sub.o
gcc obj/main.o obj/add.o obj/div.o obj/sub.o -o main
~~~

最终执行项目结构图

~~~shell
maketest
├── include
│   └── head.h
├── main
├── Makefile
├── obj
│   ├── add.o
│   ├── div.o
│   ├── main.o
│   └── sub.o
└── src
    ├── add.c
    ├── div.c
    ├── main.c
    └── sub.c

3 directories, 11 files
~~~

