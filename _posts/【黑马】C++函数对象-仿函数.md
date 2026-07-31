---
title: C++函数对象(仿函数)
date: 2026-02-04 10:55:24
tags:
  - C++
---

####  函数对象概念  

**概念**： 

* 重载函数调用操作符的类，其对象常称为函数对象 
* 函数对象使用重载的()时，行为类似函数调用，也叫仿函数 

**本质**： 

* 函数对象(仿函数)是一个类，不是一个函数

#### 函数对象使用

**特点**：

* 函数对象在使用时，可以像普通函数那样调用, 可以有参数，可以有返回值 
* 函数对象超出普通函数的概念，函数对象可以有自己的状态，因为有成员变量可以记录状态
* 函数对象可以作为参数传递，企图代替函数指针复杂的类型

**示例：**

~~~cpp
class Print {
public:
	int count = 0;
	void operator()(const char* msg) {
		std::cout << msg << std::endl;
	}
};
void DoPrint(Print& print, const char* msg) {
	print(msg);
}
int main() {
	Print print;
	DoPrint(print, "Hello World");
	DoPrint(print, "Hello World");
	DoPrint(print, "Hello World");
	std::cout << "调用次数：" << print.count << std::endl;
}
~~~

#### 谓词(Predicate)

##### 谓词概念  

**概念**： 

* 返回`bool`类型的仿函数称为谓词 
* 如果`operator()`接受一个参数，那么叫做一元谓词 
* 如果`operator()`接受两个参数，那么叫做二元谓词

**一元谓词**

~~~cpp
struct GreaterFive {
    bool operator()(int val) { // 只会传入一个参数
        return val > 5;
    }
};
int main() {
    std::vector<int> v;
    for (int i = 0; i < 10; i++) v.push_back(i);
    std::vector<int>::iterator it = find_if(v.begin(), v.end(), GreaterFive());
    if (it != v.end())
        std::cout << "找到：" << *it << std::endl;
    else
        std::cout << "没找到" << std::endl;
}
~~~

**二元谓词**

~~~cpp
struct Compare {
    bool operator()(int a, int b) { // 传入两个参数进行比较
        return a > b;
    }
};
template<typename T>
void PrintVector(const std::vector<T>& v) {
	for (typename std::vector<T>::const_iterator it = v.begin(); it != v.end(); it++)
		std::cout << *it << " ";
	std::cout << std::endl;
}
int main() {
    std::vector<int> v;
    for (int i = 0; i < 10; i++) v.push_back((i+1) * 10);
    std::sort<std::vector<int>::iterator, Compare>(v.begin(), v.end(), Compare());
    PrintVector(v);
}
~~~

#### 内建函数对象

##### 内建函数对象意义

概念： 

* STL内建了一些函数对象 

分类：

* 算术仿函数 (加、减、乘、除)
* 关系仿函数 (大于、小于、等于)
* 逻辑仿函数 (逻辑与、或、非)

用法： 

* 这些仿函数所产生的对象，用法和一般函数完全相同 
* 使用内建函数对象，需要引入头文件  `#include<functional>`

这些内建仿函数的内部定义非常简单，它的出现只是为了减少我们自己去手动定义。

##### 算数仿函数

功能描述： 

* 实现四则运算 
* 其中`negate`是一元运算，其他都是二元运算 

仿函数原型： 

* `template<typename T> plus<T>`             //加法仿函数 
* `template<typename T> minus<T>`         //减法仿函数 
* `template<typename T> multiplies<T>`    //乘法仿函数 
* `template<typename T> divides<T>`         //除法仿函数 
* `template<typename T> modulus<T>`        //取模仿函数 
* `template<typename T> negate<T>`           //取反仿函数

~~~cpp
#include <functional>
// plus简单实现
class Plus {
public:
	int operator()(int a, int b) {
		return a + b;
	}
};
// negate简单实现
class Negate {
public:
	int operator()(int a) {
		return -a;
	}
};
int main() {
    std::plus<int> plus;
    std::negate<int> negate;
    std::cout << plus(10, 20) << std::endl;
    std::cout << negate(t0) << std::endl;
}
~~~

#### 关系仿函数

功能描述： 

* 实现关系对比 

仿函数原型： 

* `template<class T> bool equal_to<T>`                  //等于 
* `template<class T> bool not_equal_to<T>`           //不等于

* `template<class T> bool greater<T> `                     // 大于
* `template<class T> bool greater_equal<T> `        // 大于等于
* `template<class T> bool less<T> `                          // 小于
* `template<class T> bool less_equal<T> `              // 小于等于

**示例：**

~~~cpp
#include <functional>
// greater简单实现
class Compare {
public:
	bool operator()(int a, int b) {
		return a > b;
	}
};
template<typename T>
void PrintVector(const std::vector<T>& v) {
	for (typename std::vector<T>::const_iterator it = v.begin(); it != v.end(); it++)
		std::cout << *it << " ";
	std::cout << std::endl;
}
int main() {
    std::vector<int> v;
    for (int i = 0; i < 10; i++) v.push_back((i+1) * 10);
    std::sort<std::vector<int>::iterator, Compare>(v.begin(), v.end(), Compare());
    PrintVector(v);
}
~~~

#### 逻辑仿函数

**功能描述**： 

* 实现逻辑运算 

**函数原型**：

* `template<typename T> bool logical_and<T>`   // 逻辑与
* `template<typename T> bool logical_or<T>`   // 逻辑或
* `template<typename T> bool logical_not<T>`   // 逻辑非

~~~cpp
#include <functional>
// logical_not简单实现
class LogecalNot {
public:
    bool operator(bool f) {
        return !f;
    }
}
template<typename T>
void PrintVector(const std::vector<T>& v) {
	for (typename std::vector<T>::const_iterator it = v.begin(); it != v.end(); it++)
		std::cout << *it << " ";
	std::cout << std::endl;
}
int main() {
	std::vector<bool> v1{true, false, true, false};
	PrintVector(v1);

	std::vector<bool> v2;
	v2.resize(v1.size());
	std::transform(v1.begin(), v1.end(), v2.begin(), std::logical_not<bool>());
	PrintVector(v2);
}
~~~

