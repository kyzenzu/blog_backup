---
title: C++函数返回对象时的拷贝构造函数与临时变量
date: 2024-2-23
tags: 
  - C++
  - C/C++
---

本次测试文件`test.cpp`

为了避免**RVO（return value optimization）**带来的影响，本次测试用`g++`编译文件时都会带上`-fno-elide-constructors`选项来关闭`RVO`

每次的编译命令均为

~~~shell
> g++ test.cpp -o test -fno-elide-constructors
~~~

本次测试包含头文件如下

~~~cpp
#include <iostream>
~~~

测试所使用的类如下

~~~cpp
class Entity
{
public:
    Entity() {
        std::cout << "Default constructor\t" << this << std::endl;
    }
	Entity(int x) {
    	std::cout << "Parameter constructor\t" << this << std::endl;
	}
    Entity(const Entity& other) {
        std::cout << this << "Copy" << &other << std::endl;
    }
    ~Entity() {
        std::cout << "Destructor function\t" << this << std::endl;
    }
    Entity& operator=(const Entity& other) {
        std::cout << this << " = " << &other << std::endl;
        return *this;
    }
};
~~~

测试代码及测试结果将根据不同情况分开处理



~~~cpp
void fun(Entity e)
{
	std::cout << "fun()..." << std::endl;
}

int main()
{
	fun(1);

	return 0;
}
~~~

~~~
Parameter constructor   0xc2a39ffbbf
fun()...
Destructor function     0xc2a39ffbbf
~~~

直接用参数传递对象时并不会先产生临时对象，再拷贝构造，而是类似于`Entity e = 1`隐式使用有参构造。

下面看一下同样的`fun()`函数但是不同的调用方式会有什么不一样的结果

~~~cpp
Entity fun()
{
    Entity e;
    return e;
}
~~~



~~~cpp
int main()
{
	Entity e1;
    e1 = fun();
	return 0;
}
~~~

~~~
Default constructor     0xc42fdff88e // e1
Default constructor     0xc42fdff83f // e
Copy constructor        0xc42fdff88f // tmp
0xc42fdff88f Copy 0xc42fdff83f // tmp copy e
Destructor function     0xc42fdff83f // e
0xc42fdff88e = 0xc42fdff88f e1 = tmp
Destructor function     0xc42fdff88f // tmp
Destructor function     0xc42fdff88e // e1
~~~

~~~cpp
int main()
{
	Entity e1 = fun();
	return 0;
}
~~~

~~~
Default constructor     0xf880fff84f // e
Copy constructor        0xf880fff89f // e1
0xf880fff89f Copy 0xf880fff84f // e1 copy e
Destructor function     0xf880fff84f // e
Destructor function     0xf880fff89f // e1
~~~

区别就在于`e1`对象在一开始有没有被创建

第一种情况，先无参构造出了`e1`，在用等号赋值时产生了临时对象`tmp`。

第二种情况，`e1`一开始没有被创建。这种情况下，是隐式的调用拷贝构造函数，等同于`Entity e1(fun())`，这里应该是没有问题的。我猜测，拷贝构造不会产生临时对象所以这里直接以`e`作为`e1`的拷贝对象

~~~cpp
Entity fun2()
{
    Entity e(2);
    std::cout << "fun2()..." << std::endl;
    return e;
}

int main()
{
    std::cout << "before fun2()..." << std::endl;
    fun2();
    std::cout << "after fun2()..." << std::endl;
    return 0;
}
~~~

~~~
before fun2()...
Parameter constructor   0xe7aa9ffa9f // e
fun2()...
Copy constructor        0xe7aa9ffaef // tmp
0xe7aa9ffaef Copy 0xe7aa9ffa9f // tmp copy  e
Destructor function     0xe7aa9ffa9f
Destructor function     0xe7aa9ffaef
after fun2()...
~~~

只是简单的调用，也会在主函数产生临时对象

下面是比较综合例子

~~~cpp
void fun1(Entity e3)
{
    std::cout << "fun1()..." << std::endl;
}

Entity fun2()
{
    Entity e(2);
    std::cout << "fun2()..." << std::endl;
    return e;
}

int main()
{
    Entity e1(1);
    Entity e2 = e1;
    std::cout << "before fun1()..." << std::endl;
    fun1(e1);
    std::cout << "after fun1()..." << std::endl;
    std::cout << "before fun2()..." << std::endl;
    Entity e4 = fun2();
    e4 = fun2();
    std::cout << "after fun2()..." << std::endl;
    return 0;
}
~~~



~~~
Parameter constructor   0xf0d1bffbad // e1
Copy constructor        0xf0d1bffbac // e2
0xf0d1bffbac Copy 0xf0d1bffbad // e2 copy e1
before fun1()...
Copy constructor        0xf0d1bffbae // e3
0xf0d1bffbae Copy 0xf0d1bffbad // e3 copy e1
fun1()...
Destructor function     0xf0d1bffbae // e3
after fun1()...
before fun2()...
Parameter constructor   0xf0d1bffb5f // e
fun2()...
Copy constructor        0xf0d1bffbab // e4
0xf0d1bffbab Copy 0xf0d1bffb5f e4 copy e
Destructor function     0xf0d1bffb5f // e
Parameter constructor   0xf0d1bffb5f // e
fun2()...
Copy constructor        0xf0d1bffbaf // tmp
0xf0d1bffbaf Copy 0xf0d1bffb5f tmp copy e
Destructor function     0xf0d1bffb5f // e
0xf0d1bffbad = 0xf0d1bffbaf // e4 = tmp
Destructor function     0xf0d1bffbaf // tmp
after fun2()...
Destructor function     0xf0d1bffbab // e4
Destructor function     0xf0d1bffbac // e2
Destructor function     0xf0d1bffbad // e1
~~~

接下来看一下，在`RVO`的情况下会发生什么有趣的事情

编译命令改为：`g++ test.cpp -o test`

~~~cpp
Entity fun2(Entity e)
{
    std::cout << "fun2()..." << std::endl;
    Entity e3(2);
    std::cout << "e3 Address: " << &e3 << std::endl;
    return e3;
}

int main()
{
    Entity e1;
    std::cout << "before fun2()..." << std::endl;
    Entity e2 = fun2(1);
    std::cout << "after fun2()..." << std::endl;
    std::cout << "e2 Address: " << &e2 << std::endl;
    return 0;
}
~~~

~~~
Default constructor     0xb95e1ff61e // e1
before fun2()...
Parameter constructor   0xb95e1ff61f // e
fun2()...
Parameter constructor   0xb95e1ff61d // e3
e3 Address: 0xb95e1ff61d
Destructor function     0xb95e1ff61f // e
after fun2()...
e2 Address: 0xb95e1ff61d
Destructor function     0xb95e1ff61d // e3
Destructor function     0xb95e1ff61e // e1
~~~

发现`e2`的地址竟然跟`e3`一样，为什么呢？以下是我的猜测

编译器发现，最后返回的时候，`e2`跟`e3`是几乎是同时创建而且完完全全一模一样的，反正`e3`最后都是要销毁的，那为什么不直接把`e3`的内存交给`e2`呢？所以，在生成汇编代码时，并没有给`e3`分配内存，对e3的操作就是直接在`e2`的内存上进行的

这种操作只有在类型为对象时会发生，如果是基本数据类型的话就是比较一般的拷贝操作

### 参考文章

[C++ 函数返回对象时并没有调用拷贝构造函数_在初始化列表中未调用拷贝构造函数-CSDN博客](https://blog.csdn.net/nbu_dahe/article/details/119142610)

[C++：函数返回值与临时变量_.c++中将临时变量作为返回值的写法-CSDN博客](https://blog.csdn.net/qq_22660775/article/details/89854545)

### 结语

了解临时变量产生的机制，可以让我们在生产中避免不必要的性能开销。对C++的运行机制也会更加了解。此外，我还发现，形参变量的地址与函数中局部变量的地址似乎并不在同一栈帧上。。。（有待考究