---
title: gcc和g++的区别
date: 2024-7-10
tags: 
  - C/C++
  - gcc
---

`GCC`，全称**GNU Compile Collection**，也就是一个编译套装。通常来看，`gcc`是用来编译C的，`g++`是用来编译C++的。从用法上来看这是没错的，但这种认识是比较片面的。

需要认识到的第一个原理是，`g++`最终调用的还是`gcc`，换句话说，`gcc`才是最终的编译器。

接下来从三个方面来讲述两者的区别：

1. 对待源文件(编译阶段)，`gcc`对待`.c`文件将其当作C程序，对待`.cpp`文件将其当作C++程序；`g++`对待`.c`和`.cpp`文件一律当作C++程序处理
2. 链接标准库(链接阶段)，`gcc`默认只会连接到标准C库；`g++`可以自动连接到标准C库和标准C++库。但是`gcc`可以用`-l`参数添加C++标准库，像这样`-lstdc++`，这样的话`gcc`也可以编译C++了。
3. 关于`__cpluscplus`宏定义。`gcc`会根据源文件后缀名去判断是否要定义这个宏；`g++`会直接定义这个宏，但是这并不影响它可以正常编译C程序

~~~shell
gcc a.cpp -o a
/usr/bin/ld: /tmp/cc6TWnM5.o: warning: relocation against `_ZSt4cout' in read-only section `.text'
/usr/bin/ld: /tmp/cc6TWnM5.o: in function `main':
a.cpp:(.text+0x11): undefined reference to `std::cout'
/usr/bin/ld: a.cpp:(.text+0x19): undefined reference to `std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)'
/usr/bin/ld: a.cpp:(.text+0x20): undefined reference to `std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)'
/usr/bin/ld: a.cpp:(.text+0x2b): undefined reference to `std::ostream::operator<<(std::ostream& (*)(std::ostream&))'
/usr/bin/ld: warning: creating DT_TEXTREL in a PIE
collect2: error: ld returned 1 exit status

gcc a.cpp -l stdc++ -o a

./a
Hello World
~~~

---

## **GCC 各阶段的实际程序**

| 阶段         | 程序名称         | 描述             |
| :----------- | :--------------- | :--------------- |
| **预处理器** | `cpp`            | C PreProcessor   |
| **编译器**   | `cc1`、`cc1plus` | C/C++ 编译器前端 |
| **汇编器**   | `as`             | GNU Assembler    |
| **链接器**   | `ld`             | GNU Linker       |

~~~shell
# 分步执行
gcc -E hello.c -o hello.i    # 预处理, 调用cpp
gcc -S hello.i -o hello.s    # 编译, 调用cc1
gcc -c hello.s -o hello.o    # 汇编, 调用as
gcc hello.o -o hello         # 链接, 调用ld

# 一步完成（常用方式）
gcc hello.c -o hello
~~~
---
`cpp`  预处理器处理`.c`源文件在交给编译器之前所做的任务：

1. 识别并**递归**处理预处理指令，比如`#include`、`#define`等。其它无法识别的原样保留
2. 保留特殊的预处理指令`#pragma`，因为这个指令的具体执行效果由编译器决定
3. 删除源代码中的注释内容
4. **不进行语法检测**

---

## **#include 实际搜索顺序**

1. `#include "header.h"`
	1. **当前源文件所在目录**
	2. `-iquote` 指定的目录
	3. `-I` 指定的目录
	4. 系统头文件目录（如 `/usr/include` 等）
2. `#include <header>`
	1. `-I` 指定的目录
	2. 系统头文件目录（如 `/usr/include` 等）

C 标准并不强制规定**具体路径列表**，只规定**语义差异**：

- `#include "h"`
	 → 先在**实现定义的位置**查找（通常是当前文件目录），失败后再按 `<h>` 的规则查找
- `#include <h>`
	 → 只在**实现定义的系统头文件位置**查找

