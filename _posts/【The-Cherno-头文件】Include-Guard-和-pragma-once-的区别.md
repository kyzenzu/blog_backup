---
title: 【The Cherno 头文件】Include Guard 和 #pragma once 的区别
date: 2026-01-12 17:36:21
tags:
 - C/;C++
---

先说结论：`Include Guard`是基于**宏**的，而`#pragma once`是基于**文件**的。

乍一看，这两个没什么区别，都是为了防止**头文件**的内容被同一个**编译单元**包含两次而使用。

但硬要说区别的话就是上面的结论。

* 实验文件：`main.c`、`header1.h`、`header2.h`
* 实验命令：`gcc -E main.c -o main.i`，得到**预编译文件**
* 实验目的：观察`header2.h`中`header2_1()`函数与`header2_2()`函数的包含次数

~~~c
/* main.c */
#include "header2.h"
#include "header1.h"
int main()
{
	return 0;
}

~~~

目的非常简单，就是要看结果会包含两个头文件`header1.h`和`header2.h`的哪些内容。

~~~c
/* header1.h */
void header1() {
    
}
#include "header2.h"
~~~

~~~c
/* header2.h */
void header2_1() {
    
}
void header2_2() {

}
~~~

#### Step1 正常情况

~~~c
# 0 "Main.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "Main.c"
# 1 "header2.h" 1
void header2_1() {

}
void header2_2() {

}
# 2 "Main.c" 2
# 1 "header1.h" 1
void header1() {

}
# 1 "header2.h" 1
void header2_1() {

}
void header2_2() {

}
# 5 "header1.h" 2
# 3 "Main.c" 2
int main()
{
 return 0;
}

~~~

可以发现`header2.h`被包含了两次。这会导致重定义问题

接下来直接展示`Include Guard` 和 `#pragma once `的区别。

对`header2.h`做具体修改（重复包含保护）

#### Step2 Include Guard

~~~c
/* header2.h */
void header2_1() {
    
}
#ifndef _HEADER2_H
#define _HEADER2_H
void header2_2() {

}
#endif
~~~

* 注意我只对`header2_2()`做了**保护**

然后预处理得到结果

~~~c
# 0 "Main.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "Main.c"
# 1 "header2.h" 1
void header2_1() {

}
void header2_2() {

}
# 2 "Main.c" 2
# 1 "header1.h" 1
void header1() {

}
# 1 "header2.h" 1
void header2_1() { /* 被重复包含 */

}
# 5 "header1.h" 2
# 3 "Main.c" 2
int main()
{
 return 0;
}
~~~

可以发现`header2_1()`被重复包含了。毕竟我只对`header2_2()`部分做了**保护**

#### Step3 #pragma once

~~~c
/* header2.h */
void header2_1() {
    
}
#pragma once
void header2_2() {

}
~~~

* 注意预编译指令的位置

然后预处理得到结果

~~~c
# 0 "Main.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "Main.c"
# 1 "header2.h" 1
void header2_1() {

}     
void header2_2() {

}
# 2 "Main.c" 2
# 1 "header1.h" 1
void header1() {

}
# 3 "Main.c" 2
int main()
{
 return 0;
}
~~~

可以发现整个`header2.h`只被包含了一次。

---

## 原理分析

#### Include Guard

预编译器在碰到`#ifndef`后就会检查**宏表**是否有相应的宏定义

如果有，就会掠过`#ifndef`和`#endif`所包含的内容，预编译`#endif`后面的内容。

#### #pragma once

预编译器在碰到`#pragma once`就会标记当前文件，比如本实验中就是`header2.h`

这样预编译器下一次在别的文件中再次执行`#include "header2.h"`就会直接跳过不执行。
