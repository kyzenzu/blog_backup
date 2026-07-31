---
title: 【C++】const int和int究竟是不是同一种类型
date: 2026-06-29 22:48:40
tags:
  - C++
---

### 结论

我认为，`const int`和`int`严格意义上不是同一种类型，`deepseek`给出的结论中编译器也不把它们视为同一种类型。

这一点变量的声明和定义分离中体现的尤为明显。

#### 声明和定义分离

~~~cpp
// MyClass.h
class MyClass {
    static const int count;
};

// MyClass.cpp
int MyClass::count = 10;
~~~

头文件中负责声明，源文件中负责定义。定义中少了`const`会报错：`conflicting declaration int MyClass::count`。

意为：不兼容的声明，或者说声明冲突了。

所以编译器在严格意义上也会认为它们是不同的类型。

#### 函数重载的特殊情况

既然编译器认为它们是不相同的类型，为什么在函数重载时`void f(int x)`和`void f(const int x)`编译器不通过，视为重定义呢。

这是因为编译器在判断函数重载，生成函数签名时将函数参数内的`const`去掉了。将`void f(const int)`视为`void f(int)`。

为什么要去掉呢，从下面几个角度思考：

1. 如果`const`可以保留住视为不同的类型，真正调用的时候应该调用哪个函数呢。因为这时函数参数是拷贝赋值，`const`的重点是形参变量`x`，调用方传入`const int`或者`int`都可以，这都允许赋值。造成了二义性
2. 编译器只需要检测函数内部是否有修改`x`的行为就可以保证`const`的修饰属性，然后在最后生成函数签名判断函数重载时将`const`去掉就能避免二义性。

综上，函数重载编译不通过不是说就能得出 `const int`和`int`是相同类型的结论，只是因为在生成函数签名时`const`被去掉了。