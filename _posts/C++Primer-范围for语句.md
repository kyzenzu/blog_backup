---
title: C++范围for语句
date: 2024-9-28
tags:
  - C/C++
  - C++
---

# 范围for语句

`C++11`标准引入了一种更简单的 for 语句，也就是**范围 for 语句**，或者说 **for-in 循环**，其基本的用法与其它语言差不多：

~~~c++
for (Type element : container)
    // do sth. on element
~~~

从结果上来说，这种循环语句是`C++11`提供的语法糖，想要正确的使用它，我们还需要了解其底层机制是怎么样的。

在**《C++ Primer》第5版**，**第168页5.4.3**中提给出如下示例：

~~~c++
vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
// 范围变量必须是引用类型，这样才能对元素执行写操作
for (auto &r : v)
    r *= 2; // 将v中每个元素的值翻倍
~~~

范围 for 语句的定义来源于与之等价的传统 for 语句：

~~~c++
for (auto beg = v.begin(), end = v.end(); beg != end; ++beg) {
    auto &r = *beg; // r 必须是引用类型，这样才能对元素执行写操作
    r *= 2; // 将v中每个元素的值翻倍
}
~~~

这样我们也就清楚了，为什么原来的范围 for 循环在定义 **r** 变量时需要使用引用。从传统 for 循环来看，原来在范围 for 小括号里定义的 **r** 变量被移到循环体内定义，在头部重新定义 2 个容器的迭代器 **beg **和 **end** ，分别指向容器的开头和”结尾“，整个循环由这 2 个迭代器来控制，每次循环变量 **r** 都会根据迭代器(**beg**)指向的元素重新赋值，然后对变量 **r** 执行一些操作。

为了加深印象和验证这种转换，我们可以自己实现一个可以用于范围 for 循环的类。

~~~c++
#include <iostream>

class Container {
private:
    static const size_t size = 5;
    int arr[size] = {0, 1, 2, 3, 4};
public:
	class Iterator {
	private:
		int* ptr;
	public:
		Iterator(int* ptr) {
            this->ptr = ptr;
        }
        int& operator*() {
            return *ptr;
        }
		Iterator operator++() {
		    ptr++;
			return ptr;
		}
		bool operator!=(const Iterator& other) const {
			return ptr != other.ptr;
		}
	};
	Iterator begin() {
        return Iterator(arr);
    }
    Iterator end() {
        return Iterator(arr + size);
    }
};

int main()
{
    Container container;
    for (int element : container)
        std::cout << element << std::endl;
}
~~~

编写这样的类可以让我们更加清晰的认识到范围 for 循环是如何转换为传统 for 循环的，分析这个转换流程，我们就可得出自己实现的类需要满足什么条件才能用于范围 for 循环：

1. 有一个内部类 **Iterator** 作为迭代器
2. 迭代器实现操作符重载，包括以下操作符`*，!=，++`
3. **Container** 类内置`begin()`和`end()`函数，且两者返回该容器的迭代器

其中需要特别注意的是，迭代器的`++`运算符需要重载的是前置自增，也就是`++iterator`，如上面示例中所示，否则编译器马上就会报错

后置自增重载是这样的

~~~c++
iterator operator++(int) { // 小括号里有个int表示后置自增
    iterator temp = *this;
    ++ptr;
    return temp;
}
~~~

其实在没有翻阅书籍前，我对传统 for 循环的猜测是这样的：

~~~c++
for (auto beg = v.begin(); beg != v.end(); ++beg) {
    auto &r = *beg; // r 必须是引用类型，这样才能对元素执行写操作
    r *= 2; // 将v中每个元素的值翻倍
}
~~~

注意到了吗，关于 `end()`函数我把它移到了循环条件的位置。乍一看这样想也挺正确，写个代码验证以下就行了，为了方便我拿 `std::vector` 容器做演示

~~~c++
std::vector<int> v = {0, 1, 2, 4};
std::vector<int>::iterator it = v.end() - 1;
for (int r : v) {
	if (r == 0)
	v.erase(it);
	std::cout << r << std::endl;
}
~~~

这段代码大概的意思是：

1. 用列表初始化的方式定义了一个`std::vector`变量 **v**，其包含 4 个元素
2. 定义了一个迭代器 **it**，指向了容器的最后一个元素 **4**
3. 进入循环的第一步就是将容器最后一个元素 **4** 擦除
4. 然后依次输出容器内的各个元素

如果按照我的之前的猜测，这段代码的结果应该只是`0 1 2 3`，因为每次循环条件都会重新判断，第一次循环后 `v.end()`就已经减一，输出 **3** 之后应该就会停止。

但事实并非如此，实际上它还将 **4** 输出了：

~~~shell
[Running] cd "d:\cppcode" && g++ test.cpp -o test && "d:\cppcode"test
0
1
2
4
[Done] exited with code=0 in 1.176 seconds
~~~

这也侧面体现出了范围 for 循环转换后的传统 for 循环就是书上的这种。

按照书上的这种，程序在循环时对 `end` 的判断是具有延时性的，它不是实时的，这也因此表现出了范围 for 循环的不安全性。

书上在后面提到，如果我们使用了范围 for 循环，我们不应该在循环体内对容器进行添加或删除操作。或者说一个更广泛的问题，在定义一个迭代器后，我们在使用这个迭代器的期间不应该对容器执行添加或删除。

以`vector`容器为例，`vector`之所以能够实现动态数组，就是依赖于一种算法，根据当前容器的`size`来适当的申请和释放内存。如果我们在定义了一个 `iterator`后往容器添加数据，这个`vector`可能会释放当前存储元素的空间，换个更大的空间来存储。那么这时`iterator`指向的就是一个被释放后不应该被利用的地址，对这块内存空间进行操作无疑是十分危险的。

**原版**

~~~C++
#include <iostream>
#include <vector>
using namespace std;

vector<int> v{ 1,2,3,4,5,6 };
vector<int>& getRange()
{
    cout << "get vector range..." << endl;
    return v;
}

int main(void)
{
    for (auto val :v)
    {
        cout << val << " ";
    }
    cout << endl;

    return 0;
}
~~~

**C++ Insights**

~~~C++
#include <iostream>
#include <vector>
using namespace std;

std::vector<int, std::allocator<int> > v = std::vector<int, std::allocator<int> >{std::initializer_list<int>{1, 2, 3, 4, 5, 6}, std::allocator<int>()};
std::vector<int, std::allocator<int> > & getRange()
{
  std::operator<<(std::cout, "get vector range...").operator<<(std::endl);
  return v;
}

int main()
{
  {
    std::vector<int, std::allocator<int> > & __range1 = v;
    __gnu_cxx::__normal_iterator<int *, std::vector<int, std::allocator<int> > > __begin1 = __range1.begin();
    __gnu_cxx::__normal_iterator<int *, std::vector<int, std::allocator<int> > > __end1 = __range1.end();
    for(; __gnu_cxx::operator!=(__begin1, __end1); __begin1.operator++()) {
      int val = __begin1.operator*();
      std::operator<<(std::cout.operator<<(val), " ");
    }
    
  }
  std::cout.operator<<(std::endl);
  return 0;
}
~~~

