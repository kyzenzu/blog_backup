---
title: C++中字符串字面量
date: 2023-2-5
tags:
  - C/C++
  - C++
---

### C++中字符串字面量

* 在C/C++中用双引号括起来的字符序列就叫做字符串字面量,如<u>"abc"</u>

* 在C++中字符串字面量的本质就是 ==const char[] 类型==,如"hello"就是const char[6]型
  在必要的时候，可以将字符串字面量转化为 ==const char*类型==的指针,该指针指向字符串的首字符
  所以,C++的字符串字面量是不允许被修改的,且指针赋值时也必须有规定
  ~~~C++
  const char* str = "hello"; //这样赋值才对
  char* str = "hello"; //报错
  ~~~
  
* 但是,C中的字符串字面的的本质是 ==char[] 类型== ,也就是说理论上C的字符串字面量是   可以被修改的，但这是==未定义行为==,一旦由此操作程序或许会崩溃

​      下面以"hello"为例

* "hello"是数组类型的左值，且是不可被修改的左值,也就是说&"hello"是合法的
* "hello"在必要时可以转化为指针,等同于 &("hello"[0])
~~~ C++
#include <iostream>
int main()
{
	printf("%p\n", "hello");//00007ff6295f9000
	printf("%p\n", &"hello");//00007ff6295f9000
	printf("%c\n", *"hello");//h
	printf("%c\n", "hello"[0]);//h
	printf("%c\n", *("hello"+1));//e
	const char* p = "hello";
	printf("p的地址:%p,p指向%p", &p, p);//p的地址:00000008813ffdc8
                                       //p指向00007ff6295f9000
	return 0;
}
~~~

此外，因为数组类型左值无法修改,所以 "hello" += 1 是不允许的

~~~ C++
	char s[] = "hello";
~~~

这里的"hello"是作为左值使用来为s数组赋初值,所以不会转化为指针，所以s是独立的并不会指向"hello"的地址

