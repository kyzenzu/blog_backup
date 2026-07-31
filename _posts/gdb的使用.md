---
title: gdb的使用
date: 2024-7-17
tags: 
  - C/C++
  - gdb
---

在使用`gdb`调试程序之前，需要用`gcc`根据源代码生成可调试的二进制程序。一般生成的可执行程序因为没有调试信息所以`gdb`无法调试，因为有了调试信息，所以可调式的二进制程序会比没有调试信息的二进制程序的大小要大一些。

~~~shell
> gcc -g *.c -o a
~~~

像这样就可生成可调试的二进制程序，但是可以用`-O0 -Wall`参数把生成的代码优化参数调到最低，并显示一些警告信息。

~~~shell
gcc -g *.c -o a -O0 -Wall
~~~

这个过程其实和正常编译一个`C`文件差不多。如果是`C++`文件的话，将`gcc`换成`g++`就好了。

使用`gdb a`命令就可以进入`gdb`调试前面生成的程序`a`

~~~shell
> gdb a
GNU gdb (Debian 13.2-1+b2) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from a...
(gdb)
~~~

注意这只是进入了`gdb`，并将程序`a`加载到了`gdb`中，这时候`a`程序还没有被执行。

有些程序执行时需要输入命令行参数，但是用`gdb`调试时没有输入命令行参数，这时用以下两个命令分别设置命令行参数和查看命令行参数

~~~shell
(gdb) set args 1 2 3 4 # 想放几个参数放几个
(gdb) show args
Argument list to give program being debugged when it is started is "1 2 3 4".
~~~

* `start`: 启动程序, 最终会阻塞在main函数的第一行，等待输入后续其它 `gdb` 指令

* `run`: 可以缩写为 r, 如果程序中设置了断点会停在第一个断点的位置, 如果没有设置断点, 程序就执行完了。与`continue`不同的是，这时程序执行之前调用的命令

* `quit`: 可以缩写为`q`，退出 `gdb`

`list`命令用于显示源代码，具体使用如下

~~~shell
# 使用 list 和使用 l 都可以
# 从第一行开始显示
(gdb) list 

# 列值这行号对应的上下文代码, 默认情况下只显示10行内容
(gdb) list 行号

# 显示这个函数的上下文内容, 默认显示10行
(gdb) list 函数名
~~~

此外，`gdb`默认定位在`main`函数所在的文件中，用`list`命令不仅可以显示其它文件中的代码，还可以切换到其它文件，格式如下

~~~shell
# 切换到指定的文件，并列出这行号对应的上下文代码, 默认情况下只显示10行内容
(gdb) l 文件名:行号

# 切换到指定的文件，并显示这个函数的上下文内容, 默认显示10行
(gdb) l 文件名:函数名
~~~

注意这个冒号就固定的跟文件名搭配使用好了，虽然放在函数名后也能生效，但其实跟不加冒号一样。

`gdb`默认`list`显示的行数为`10`行，用以下命令可以设置显示的行数和查看当前显示的行数

~~~shell
# 以下两个命令中的 listsize 都可以写成 list
(gdb) set listsize 行数

# 查看当前list一次显示的行数
(gdb) show listsize
~~~

---

* `break`，缩写为`b`，用来打断点，具体使用情况如下

~~~shell
# 在当前文件的某一行上设置断点
(gdb) break 行号
(gdb) b 函数名		# 停止在函数的第一行

# 在非当前文件的某一行上设置断点
(gdb) b 文件名:行号
(gdb) b 文件名:函数名		# 停止在函数的第一行

# 必须要满足某个条件, 程序才会停在这个断点的位置上
# 通常情况下, 在循环中条件断点用的比较多
(gdb) b 行数 if 变量名==某个值
~~~

* `info`，缩写为`i`，可以查看一些信息。比如`break`会为每个断点打上一个编号，用`info break`可以查看我们之前给哪些地方打了断点，并查看其详细信息

~~~shell
(gdb) i b   #info break

# 举例
(gdb) i b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000400cb5 in main() at test.cpp:12
2       breakpoint     keep y   0x0000000000400cbd in main() at test.cpp:13
3       breakpoint     keep y   0x0000000000400cec in main() at test.cpp:18
4       breakpoint     keep y   0x00000000004009a5 in insertionSort(int*, int) 
                                                   at insert.cpp:8
5       breakpoint     keep y   0x0000000000400cdd in main() at test.cpp:16
6       breakpoint     keep y   0x00000000004009e5 in insertionSort(int*, int) 
                                                   at insert.cpp:16
~~~

1. `Num`: 断点的编号, 删除断点或者设置断点状态的时候都需要使用

2. `Enb`: 当前断点的状态, y表示断点可用, n表示断点不可用

3. `What:` 描述断点被设置在了哪个文件的哪一行或者哪个函数上

* `delete`，缩写为`del`或`d`，可以用来删除某个断点，注意这个操作是不可逆的，而且断点编号不会因此调整，也就是说断点的编号中间可能会空了一个数

~~~shell
#举例
(gdb) d 1          # 删除第1个断点
(gdb) d 2 4 6      # 删除第2,4,6个断点

# 删除一个范围, 断点编号 num1 - numN 是一个连续区间
(gdb) d num1-numN
# 举例, 删除第1到第5个断点
(gdb) d 1-5
~~~

* `disable`，缩写为`dis`，如果不想让某个断点生效，又不想删除这个断点就可以使用这个命令让断点失效，它会修改断点的`Enb`属性为`n`
* `enable`，缩写为`ena`，与`disable`对应，让失效的断点恢复有效状态

~~~shell
# 设置某一个或者某几个断点无效
(gdb) dis 断点1的编号 [断点2的编号 ...]
(gdb) disable 2 3 5
# 设置某个区间断点无效
(gdb) dis 断点1编号-断点n编号
(gdb) disable 3-5

# 设置某一个或者某几个断点有效
(gdb) ena 断点1的编号 [断点2的编号 ...]
(gdb) enable 2 3 5
# 设置某个区间断点有效
(gdb) ena 断点1编号-断点n编号
(gdb) enable 3-5
~~~

---

* `continue`，缩写为`c`，让程序从当前断点处继续运行下去，直到遇到下一个有效的断点。与`run`命令不同的是，这是程序进入执行状态之后用的命令。
* `print`，缩写为`p`，用`/fmt`指定要输出的变量的格式

| 格式化字符(/fmt) |                 说明                 |
| :--------------: | :----------------------------------: |
|       `/x`       |     以十六进制的形式打印出整数。     |
|       `/d`       |  以有符号、十进制的形式打印出整数。  |
|       `/u`       |  以无符号、十进制的形式打印出整数。  |
|       `/o`       |      以八进制的形式打印出整数。      |
|       `/t`       |      以二进制的形式打印出整数。      |
|       `/f`       | 以浮点数的形式打印变量或表达式的值。 |
|       `/c`       |   以字符形式打印变量或表达式的值。   |

~~~shell
# print == p
(gdb) p 变量名

# 如果变量是一个整形, 默认对应的值是以10进制格式输出, 其他格式请参考上表
(gdb) p/fmt 变量名

# 举例
(gdb) p i       # 10进制
$5 = 3
(gdb) p/x i     # 16进制
$6 = 0x3
(gdb) p/o i     # 8进制
$7 = 03
~~~

* `ptype`，用来打印变量的类型

~~~shell
# 语法格式
(gdb) ptype 变量名

# 打印变量类型
(gdb) ptype i
type = int
(gdb) ptype array[i]
type = int
(gdb) ptype array
type = int [12]
~~~

* `display`，没有缩写，在每一次执行一条指令后就会打印变量的值，每个被标记要被打印的变量会被打上编号和相应的属性，和`break`管理断点一样

~~~shell
# 在变量的有效取值范围内, 自动打印变量的值(设置一次, 以后就会自动显示)
(gdb) display 变量名

# 以指定的整形格式打印变量的值, 关于 fmt 的取值, 请参考 print 命令
(gdb) display/fmt 变量名

# 举例
(gdb) display/c i
(gdb) display arr
~~~

同样可以用`info`命令查看`display`的详细信息

~~~shell
(gdb) info display
Auto-display expressions now in effect:
Num Enb Expression
1:   y  i
2:   y  array[i]
3:   y  /x array[i]
~~~

1. `Num` : 变量或表达式的编号，GDB 调试器为每个变量或表达式都分配有唯一的编号
2. `Enb` : 表示当前变量（表达式）是处于激活状态还是禁用状态，如果处于激活状态（用 y 表示），则每次程序停止执行，该变量的值都会被打印出来；反之，如果处于禁用状态（用 n 表示），则该变量（表达式）的值不会被打印。
3. `Expression` ：被自动打印值的变量或表达式的名字。

* `undisplay`，和删除断点一样，会把我们不想要显示的变量删除掉，不可逆。还可以用`delete display`来代替

~~~shell
# 命令中的 num 是通过 info display 得到的编号, 编号可以是一个或者多个
(gdb) undisplay num [num1 ...]
(gdb) undisplay 2 3 5
# num1 - numN 表示一个范围
(gdb) undisplay num1-numN
(gdb) undisplay 3-5
~~~

* `disable display`，如果不想删除只是想让其失效就可以用这个命令，它会修改`display`管理的变量的`Ena`属性为`n`
* `enable display`，遇上面对命令相对应，恢复变量的有效性

~~~shell
# 命令中的 num 是通过 info display 得到的编号, 编号可以是一个或者多个
(gdb) disable display num [num1 ...]
(gdb) disable display 2 3 5
# num1 - numN 表示一个范围
(gdb) disable display num1-numN
(gdb) disable display 3-5

# 命令中的 num 是通过 info display 得到的编号, 编号可以是一个或者多个
(gdb) enable  display num [num1 ...]
# num1 - numN 表示一个范围
(gdb) disable display num1-numN
~~~

---

`step`，缩写为`s`，单步执行命令，但是遇到函数时会进入函数内部执行，即 **步入(step into)**

`next`，缩写为`n`，执行指令，但是会跳过函数调用，不会进入函数内部，即 **步过(step over)**

`finish`，进入函数后，如果不想待在函数内就可以使用这个命令执行函数剩下的指令，跳出函数，即 **步出(step out)**。如果想要跳出函数体必须要保证函数体内不能有有效断点，否则无法跳出，会停在函数体内部的下一个断点处。

`until`，通过 until 命令可以直接跳出某个循环体，这样就能提高调试效率了。如果想直接从循环体中跳出, 必须要满足以下的条件，否则命令不会生效：

1. 要跳出的循环体内部不能有有效的断点，如果循环体内部有断点可以用`disable`命令使其失效或者直接`delete`掉，否则无效。
2. 必须要在循环体的开始/结束行执行该命令，也就是说只有停在`for (i = 0; i < len; i++)`这个地方才能使用`until`，否则无效。

`set var 变量名=值`， 可以直接修改某个变量的值。

~~~shell
(gdb) set var i = 5
~~~

---

~~~gdb
set assembly-flavor intel # 将汇编风格改为intel
disassemble $rip # 反汇编rip所指的地址处
display/2i $rip # 这样可以在每次执行指令后查看当前指令
set $rax=0x61 # 修改寄存器的值
x/20i address # 查看该地址开始的20条指令
# x指令是查看内存的，输入寄存器会去寄存器指向的内存
~~~

