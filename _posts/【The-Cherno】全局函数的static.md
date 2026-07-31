---
title: 【The Cherno】全局函数的static
date: 2026-01-11 17:01:39
tags: 
  - C/C++
---
题外话：==函数返回类型不参与函数重载，但仍然作为编译器构成函数签名的一部分==

实验文件：`Math.cpp`

~~~cpp
#include <iostream>
void Log(const char* message);
int Multiply(int a, int b) {
	Log("Multiply");
	return a * b;
}
int main()
{
	std::cout << Multiply(3, 5) << std::endl;
	return 0;
}
~~~

此时`Log(const char*)`函数未定义。构建后当然会出现报错：

`Math.obj : error LNK2019: 无法解析的外部符号 "void __cdecl Log(char const *)" ，函数 "int __cdecl Multiply(int,int)" 中引用了该符号`

#### Step1

此时如果我们注释掉`Multiply()`中的`Log()`函数调用，文件就能正常构建。

~~~cpp
#include <iostream>
void Log(const char* message);
int Multiply(int a, int b) {
	//Log("Multiply");
	return a * b;
}
int main()
{
	std::cout << Multiply(3, 5) << std::endl;
	return 0;
}
~~~

因为`Multiply()`不再调用`Log()`函数，自然不会在`Multiply()`中出现**链接问题**

#### Step2

现在只注释掉`main()`函数中对`Multiply()`函数的调用。直觉上似乎没错

~~~cpp
#include <iostream>
void Log(const char* message);
int Multiply(int a, int b) {
	Log("Multiply");
	return a * b;
}
int main()
{
	//std::cout << Multiply(3, 5) << std::endl;
	return 0;
}
~~~

但结果还是报错了，同样的报错

`Math.obj : error LNK2019: 无法解析的外部符号 "void __cdecl Log(char const *)" ，函数 "int __cdecl Multiply(int,int)" 中引用了该符号`

问题还是`Multiply()`调用了`Log()`函数，虽然`main()`函数不再`Multiply()`函数，实际运行时看似不会有链接问题。但是，并不保证会有其它**翻译单元**是否会调用这里的`Multiply()`函数，所以还是会出现**链接问题**

#### Step3

注意到上面会出问题的关键是**其它翻译单元**，如果其它翻译单元无法调用`Multiply()`函数，这个问题似乎就能得到解决。也就是说，让`Multiply()`函数让其它翻译单元不可见即可。

~~~cpp
#include <iostream>
void Log(const char* message);
static int Multiply(int a, int b) {
	Log("Multiply");
	return a * b;
}
int main()
{
	//std::cout << Multiply(3, 5) << std::endl;
	return 0;
}
~~~

这样就能成功构建，没人调用`Multiply()`，也就不会有链接问题。

* **这里就体现了`static`关键字使函数让其它翻译单元不可见的能力**
