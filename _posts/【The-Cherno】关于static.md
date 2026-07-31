---
title: 【The Cherno】关于static
date: 2026-01-16 16:08:03
tags:
  - C/C++
---
`static`的作用分为三个领域：

1. 作用在**全局变量/全局函数**上
2. 作用在类的**成员变量/成员函数**上
3. 作用在函数内的**局部变量**上

这三种作用方式作用在变量上（先不谈函数），都有相同的本质：被<u>`static`修饰的变量在程序运行后存储在内存的`Data Segment`区域（重点不是栈段）</u>。由此引申出这些变量的生存周期就是程序的运行周期

但是，变量的从属对象和作用域终归有些不同，所以三种表现形式还是有些不同。

### 全局变量/全局函数的 static

作用在**全局变量**或者**全局函数**上的`static`有两个作用：

1. 当前编译单元会优先调用（或者说直接调用）本单元的静态全局变量
2. 其它编译单元无法观测到本单元的静态全局变量

* 总的来说就是将`static`全局变量标记为本单元私有的全局变量

实验文件：`a.cpp`，`b.cpp`，`c.cpp`

实验目的：观察两次`s_Varible`的输出值

~~~cpp
/* a.cpp 主文件 */ 
#include <iostream>
static int s_Varible = 2;
void Varible();
int main() {
	Log("Hello");
	std::cout << s_Varible << std::endl;
	Varible();
	return 0;
}
~~~

~~~cpp
/* b.cpp */ 
int s_Varible = 5;
~~~

~~~cpp
/* c.cpp */
#include <iostream>
extern int s_Varible;
void Varible() {
	std::cout << s_Varible << std::endl;
}
~~~

最后的结果：

~~~shell
2
5
~~~

说明：

* 第一次输出的是`a.cpp`中的`static int s_Varible`，表明`static`的第一个性质，当前编译单元会优先调用（或者说直接调用）本单元的静态全局变量。
* 第二次输出的是`b.cpp`中的全局`int s_Varible`，表明`static`的第二个性质，其它编译单元无法观测到`a.cpp`单元的静态全局变量。

### 类和结构体中的 static

首先说明，在C++中类和结构体除了**默认的可见性(visibility)**之外，其它完全相同。

类中的成员变量和方法经过`static`属性修饰后，这个成员变量也就变成了所有实例对象共有的全局变量，同时也变成了该类域下的变量。

~~~cpp
class Entity {
public:
	static int x, y;
private:
	static int speed;
public:
	static void Print() {
		std::cout << x << " " << y << " " << speed << std::endl;
	}
};

int Entity::x;
int Entity::y;
int Entity::speed = 5;
~~~

这里面最最让人疑惑，也是最值得一讲的点就是：**类内声明，类外定义。**

正如上面的用法一样，在类中被`static`修饰的变量必须在类中声明，然后在类外定义。不能直接在类中赋值，或者下面的定义不写。

~~~cpp
class Entity {
public:
	static int x = 2; // 错误写法
};
/* 没有定义 */
~~~

然后再来讲讲为什么还需要在类外定义。

在C++中，类一般是写在头文件中的，而头文件又可能被多个编译单元包含。前面说了，`static`修饰的变量在程序运行后分配在内存的`Data Segment`区域，具有全局的概念，如果允许类内定义，就会造成多个编译单元的重定义。因此需要把`static`修饰的变量的定义单独分离出来，在一个单独编译单元中定义。

但是为什么`static`修饰的函数允许在类内定义呢，这不会到这多个文件包含的重定义问题吗？答案是会的。但这时又有一个新的概念`inline`。

`inline`关键字修饰的函数允许：

1. 允许函数调用展开为函数体，注意是允许，具体会不会展开还要看编译器心情
2. 允许函数重定义

第二条性质就是关键，它虽然允许函数进行重定义，但是C++的`ODR(One Definition Rule)`原则规定编译器随机选择一个定义。所以这要求我们在编写`inline`修饰的函数时要完全一致，虽然编译器允许定义不同，但这也导致未定义行为，因为选择一个定义是随机的。

编译器允许函数在类内定义而不会出现重定义问题的答案就是：所有在类内声明定义的函数都是隐式由`inline`修饰的函数。

~~~cpp
class Entity {
public:
	static int x, y;
private:
	static int speed;
public:
	(inline) static void Print() {
		std::cout << x << " " << y << " " << speed << std::endl;
	}
};

int Entity::x;
int Entity::y;
int Entity::speed = 5;
~~~

##### 类的`static`在单例中的应用

~~~cpp
class Singleton {
private:
	static Singleton* s_Instance;
public:
	static Singleton& Get() {
		return *s_Instance;
	}
	void Hello() {
		std::cout << "Hello World" << std::endl;
	}
};
Singleton* Singleton::s_Instance = nullptr;

int main() {
	Singleton::Get().Hello();

}
~~~

#### 函数内的局部变量static

~~~cpp
class Singleton {
public:
	static Singleton& Get() {
		static Singleton instance;
		return instance;
	}
	void Hello() {
		std::cout << "Hello World" << std::endl;
	}
};
int main() {
	Singleton::Get().Hello();

}
~~~

