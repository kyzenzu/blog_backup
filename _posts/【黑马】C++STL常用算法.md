---
title: C++STL常用算法
date: 2026-02-04 16:22:17
tags:
  - C++
---

### STL- 常用算法



#### 常用遍历算法

**算法简介：**

* `for_each`     //遍历容器
* `transform`   //搬运容器到另一个容器中

#### for_each

**功能描述：**

* 实现遍历容器
* 遍历v的元素交给func处理

**函数原型：**

* `for_each(iterator beg, iterator end, _func);  `

	// 遍历算法 遍历容器元素

	// beg 开始迭代器

	// end 结束迭代器

	// _func 函数或者函数对象

**示例：**

~~~cpp
void PrintFunc(int a) {
	std::cout << a << std::endl;
}
class PrintClass {
public:
	void operator()(int a) {
		std::cout << a << std::endl;
	}
};
int main() {
	std::vector<int> v;
	for (int i = 0; i < 10; i++) v.push_back(i);
	std::for_each<std::vector<int>::iterator, void(*)(int)>(v.begin(), v.end(), PrintFunc); // 传函数
	std::for_each<std::vector<int>::iterator, PrintClass>(v.begin(), v.end(), PrintClass()); // 传仿函数
}
~~~

~~~cpp
// for_each简单实现
Func ForEach(iterator begin, iterator end, Func func) {
    for (; begin != end; begin++)
        func(*begin); // 把每个元素送到func中处理
    return func;
}
~~~

#### transform

**功能描述：**

* 搬运容器到另一个容器中
* v1的元素经过func()处理后交给v2

**函数原型：**

* `transform(iterator beg1, iterator end1, iterator beg2, _func);`

//beg1 源容器开始迭代器

//end1 源容器结束迭代器

//beg2 目标容器开始迭代器

//_func 函数或者函数对象

~~~cpp
class Transform {
public:
	int operator()(int x) {
		return x * 10;
	}
};
int main() {
	std::vector<int> v1;
	for (int i = 0; i < 10; i++) v1.push_back(i);
	std::vector<int> v2;
	v2.resize(v1.size()); // 一定要预留空间，不然运行会报错
    // 必须要用resize(), 用reserve()运行后还是报错
	std::transform<>(v1.begin(), v1.end(), v2.begin(), Transform());
}
~~~

~~~cpp
// transform简单实现
iterator transform(iterator begin, iterator end, iterator dest, Func func) {
    for (; begin != end; begin++, dest++) {
        *dest = func(*begin); // v1的元素先送给func()处理再交给v2
    }
    return dest;
}
~~~

关于一些算法使用前必须用`v.resize()`而不是`v.reserve()`的原因：

一开始定义的`std::vector<int> v`都是没有初始化的，这也就意味着`v.size`为0，那么此时`v.begin()`返回的迭代器指向哪里呢？调试后发现`v.begin()`返回的迭代器指向`nullptr`，这就有很大的问题了。而`v.capacity()`只会扩大容器的容量，`v.size()`仍然是`0`。这就使得`v.begin()`返回的迭代器就是指向`nullptr`的。而不管是 `transform` 还是 `merge`，它们的代码模型都是：

```cpp
*result = value; // 此时result == nullptr
++result;
```

这肯定在算法运行的一开始就解引用`nullptr`，直接报错。

### 常用查找算法

**算法简介：**

- `find`                     //查找元素
- `find_if`               //按条件查找元素
- `adjacent_find`    //查找相邻重复元素
- `binary_search`    //二分查找法
- `count`                   //统计元素个数
- `count_if`             //按条件统计元素个数




#### find

**功能描述：**

* 查找指定元素，找到返回指定元素的迭代器，找不到返回结束迭代器end()

**函数原型：**

- `find(iterator beg, iterator end, value);  `

	// 按值查找元素，找到返回指定位置迭代器，找不到返回结束迭代器位置

	// beg 开始迭代器

	// end 结束迭代器

	// value 查找的元素

~~~cpp
int main() {
	std::vector<int> v1;
	for (int i = 0; i < 10; i++) v1.push_back(i);
	std::vector<int>::iterator it = std::find<std::vector<int>::iterator, int>(v1.begin(), v1.begin() + 2, 5);
	std::cout << *it << std::endl; // *it == 2
}
~~~

~~~cpp
// find简单实现
iterator Find(iterator begin, iterator end, int v) {
    iterator it;
    for (; begin != end; begin++)
        if (*begin == v) break;
    return begin; // 走到哪返回哪
}
~~~

#### find_if

**功能描述：**

- 按条件查找元素

**函数原型：**

- `find_if(iterator beg, iterator end, _Pred);  `

	// 按值查找元素，找到返回指定位置迭代器，找不到返回结束迭代器位置

	// beg 开始迭代器

	// end 结束迭代器

	// _Pred 函数或者谓词（返回bool类型的仿函数）

~~~cpp
iterator find_if(iterator _First, const iterator _Last, _Pr _Pred) { // find first satisfying _Pred
    for (; _UFirst != _ULast; ++_UFirst) {
        if (_Pred(*_UFirst)) { // 遍历元素传入Pred，返回是true的元素
            break;
        }
    }
    return _First;
}
~~~

~~~cpp
class GreaterTwenty {
public:
	bool operator()(Person& p) {
		return p.m_Age > 20;
	}
};

int main() {
	std::vector<Person> v;

	std::vector<Person>::iterator it = std::find_if(v.begin(), v.end(), GreaterTwenty());
	if (it != v.end()) std::cout << it->m_Name << std::endl;
}
~~~

#### adjacent_find

**功能描述：**

- 查找相邻重复元素

**函数原型：**

- `adjacent_find(iterator beg, iterator end);  `

	// 查找**相邻且重复**元素,返回相邻元素的第一个位置的迭代器

	// beg 开始迭代器

	// end 结束迭代器

~~~cpp
int main() {
	std::vector<int> v;
	for (int i = 0; i < 10; i++) v.push_back(i);
	std::vector<int>::iterator it = std::find(v.begin(), v.end(), 7);
	v.insert(it, *it);
	it = std::adjacent_find(v.begin(), v.end());
	std::cout << *it << " " << *(it + 1) << " " << *(it + 2); // 7 7 8
}
~~~

#### binary_search

**功能描述：**

- 查找指定元素是否存在

**函数原型：**

- `bool binary_search(iterator beg, iterator end, value);  `

	// 查找指定的元素，查到 返回true 否则false

	// 注意: 在**无序序列中不可用**

	// beg 开始迭代器

	// end 结束迭代器

	// value 查找的元素

#### count

**功能描述：**

- 统计元素个数

**函数原型：**

- `count(iterator beg, iterator end, value);  `

	// 统计元素出现次数

	// beg 开始迭代器

	// end 结束迭代器

	// value 统计的元素

~~~cpp
int count(const _InIt _First, const _InIt _Last, const _Ty& _Val) {
    int _Count = 0;
    for (; _UFirst != _ULast; ++_UFirst) {
        if (*_UFirst == _Val) {
            ++_Count;
        }
    }

    return _Count;
}
~~~

~~~cpp
class Person {
public:
	std::string m_Name;
	int m_Age;
	Person(std::string name, int age) : m_Name(name), m_Age(age) {}
	bool operator==(const Person& other) {
		return m_Age == other.m_Age;
	}
};

int main() {
	std::vector<Person> v;
	Person p("诸葛亮", 35);
	int count = std::count(v.begin(), v.end(), p);
	std::cout << count << std::endl;
}
~~~

**总结：** 统计自定义数据类型时候，需要配合重载 `operator==`

#### count_if

**功能描述：**

- 按条件统计元素个数

**函数原型：**

- `count_if(iterator beg, iterator end, _Pred);  `

	// 按条件统计元素出现次数

	// beg 开始迭代器

	// end 结束迭代器

	// _Pred 谓词

~~~cpp
int count_if(_InIt _First, _InIt _Last, _Pr _Pred) {
    // count elements satisfying _Pred
    int _Count = 0;
    for (; _UFirst != _ULast; ++_UFirst) {
        if (_Pred(*_UFirst)) { // ++能返回true的元素
            ++_Count;
        }
    }

    return _Count;
}
~~~

~~~cpp
class GreaterFour {
public:
	bool operator()(int x) {
		return x > 4;
	}
};

int main() {
	std::vector<int> v;
	int count = std::count_if(v.begin(), v.end(), GreaterFour());
	std::cout << count << std::endl;
}
~~~

### 常用排序算法

**算法简介：**

- `sort` //对容器内元素进行排序
- `random_shuffle` //洗牌 指定范围内的元素随机调整次序
- `merge ` // 容器元素合并，并存储到另一容器中
- `reverse` // 反转指定范围的元素

#### sort

**功能描述：**

- 对容器内元素进行排序

**函数原型：**

- `sort(iterator beg, iterator end, _Pred);  `

	// 按值查找元素，找到返回指定位置迭代器，找不到返回结束迭代器位置

	// beg 开始迭代器

	// end 结束迭代器

	// _Pred 谓词

~~~cpp
template<class _Ty = void>
struct less {
    bool operator()(const _Ty& _Left, const _Ty& _Right) const {
        return _Left < _Right;
    }
}
template<class _Ty = void>
struct greater {
    bool operator()(const _Ty& _Left, const _Ty& _Right) const {
        return _Left > _Right;
    }
}
~~~

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v;
    // 从小到大排序
	std::sort(v.begin(), v.end(), std::less<int>());
	std::for_each(v.begin(), v.end(), Print);
	std::cout << std::endl;
    // 从大到小排序
	std::sort(v.begin(), v.end(), std::greater<int>());
	std::for_each(v.begin(), v.end(), Print);
}
~~~

#### random_shuffle

**功能描述：**

- 洗牌 指定范围内的元素随机调整次序

**函数原型：**

- `random_shuffle(iterator beg, iterator end);  `

	// 指定范围内的元素随机调整次序

	// beg 开始迭代器

	// end 结束迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v;
	for (int i = 0; i < 10; i++) v.push_back(i);
	std::for_each(v.begin(), v.end(), Print);
	std::cout << std::endl;
	std::random_shuffle(v.begin(), v.end()); // 打乱顺序
	std::for_each(v.begin(), v.end(), Print);
}
~~~

**总结：**random_shuffle洗牌算法比较实用，使用时记得加随机数种子

#### merge

**功能描述：**

- **两个容器**元素合并，并存储到**另一容器**中

**函数原型：**

- `merge(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);  `

	// 容器元素合并，并存储到另一容器中

	// 注意: 两个容器必须是**有序的**

	// beg1 容器1开始迭代器 // end1 容器1结束迭代器 // beg2 容器2开始迭代器 // end2 容器2结束迭代器 // dest 目标容器开始迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v1, v2;
	for (int i = 0; i < 10; i++) {
		if (i % 2) v1.push_back(i);
		else v2.push_back(i);
	}
	std::vector<int> v3;
	v3.resize(v1.size() + v2.size()); // 一定要预留空间，不然运行会报错
    // 必须要用resize(), 用reserve()运行后还是报错
	std::merge(v1.begin(), v1.end(), v2.begin(), v2.end(), v3.begin());
	std::for_each(v3.begin(), v3.end(), Print);
}
~~~

关于一些算法使用前必须用`v.resize()`而不是`v.reserve()`的原因：

一开始定义的`std::vector<int> v`都是没有初始化的，这也就意味着`v.size`为0，那么此时`v.begin()`返回的迭代器指向哪里呢？调试后发现`v.begin()`返回的迭代器指向`nullptr`，这就有很大的问题了。而`v.capacity()`只会扩大容器的容量，`v.size()`仍然是`0`。这就使得`v.begin()`返回的迭代器就是指向`nullptr`的。而不管是 `transform` 还是 `merge`，它们的代码模型都是：

```cpp
*result = value;
++result;
```

这肯定在算法运行的一开始就解引用`nullptr`，直接报错。

#### reverse

**功能描述：**

- 将容器内元素进行反转

**函数原型：**

- `reverse(iterator beg, iterator end);  `

	// 反转指定范围的元素

	// beg 开始迭代器

	// end 结束迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v;
	std::for_each(v.begin(), v.end(), Print);
	std::cout << std::endl;
	std::reverse(v.begin(), v.end()); // 反转
	std::for_each(v.begin(), v.end(), Print);
	std::cout << std::endl;
}
~~~

### 常用拷贝和替换算法

**算法简介：**

- `copy` // 容器内指定范围的元素拷贝到另一容器中
- `replace` // 将容器内指定范围的旧元素修改为新元素
- `replace_if ` // 容器内指定范围满足条件的元素替换为新元素
- `swap` // 互换两个容器的元素

#### copy

**功能描述：**

- 容器内指定范围的元素拷贝到另一容器中

**函数原型：**

- `copy(iterator beg, iterator end, iterator dest);  `

	// 按值查找元素，找到返回指定位置迭代器，找不到返回结束迭代器位置

	// beg 开始迭代器

	// end 结束迭代器

	// dest 目标起始迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v1, v2;
	for (int i = 0; i < 10; i++) v1.push_back(i);
	v2.resize(v1.size()); // 一定要预留空间，不然运行会报错
    // 必须要用resize(), 用reserve()运行后还是报错
	std::copy(v1.begin(), v1.end(), v2.begin());
	std::for_each(v2.begin(), v2.end(), Print);
}
~~~

#### replace

**功能描述：**

- 将容器内指定范围的旧元素修改为新元素

**函数原型：**

- `replace(iterator beg, iterator end, oldvalue, newvalue);  `

	// 将区间内旧元素 替换成 新元素

	// beg 开始迭代器

	// end 结束迭代器

	// oldvalue 旧元素

	// newvalue 新元素

 ~~~cpp
 int main() {
 	std::vector<int> v;
 	std::for_each(v.begin(), v.end(), Print);
 	std::cout << std::endl;
 	std::replace(v.begin(), v.end(), 20, 200); // 替换
 	std::for_each(v.begin(), v.end(), Print);
 }
 ~~~

#### replace_if

**功能描述:**

- 将区间内满足条件的元素，替换成指定元素

**函数原型：**

- `replace_if(iterator beg, iterator end, _pred, newvalue);  `

	// 按条件替换元素，满足条件的替换成指定元素

	// beg 开始迭代器

	// end 结束迭代器

	// _pred 谓词

	// newvalue 替换的新元素

~~~cpp
int main() {
	std::vector<int> v;
	std::for_each(v.begin(), v.end(), Print);
	std::cout << std::endl;
	std::replace_if(v.begin(), v.end(), GreaterThirty(), 3000); // 大于30的替换为3000
	std::for_each(v.begin(), v.end(), Print);
}
~~~

#### swap

**功能描述：**

- 互换两个容器，包括size、capacity、以及所有元素

**函数原型：**

- `swap(container c1, container c2);  `

	// 互换两个容器的元素

	// c1容器1

	// c2容器2

~~~cpp
int main() {
	std::vector<int> v1, v2;
	for (int i = 0; i < 10; i++) v1.push_back(i);
	for (int i = 0; i < 5; i++) v2.push_back(i * 10);
	std::swap(v1, v2);
}
~~~

### 常用算术生成算法

**注意：**

- 算术生成算法属于小型算法，使用时包含的头文件为 `#include <numeric>`

**算法简介：**

- `accumulate` // 计算容器元素累计总和
- `fill` // 向容器中添加元素

#### accumulate

**功能描述：**

- 计算区间内 容器元素累计总和

**函数原型：**

- `accumulate(iterator beg, iterator end, value);  `

	// 计算容器元素累计总和

	// beg 开始迭代器

	// end 结束迭代器

	// value 起始值

~~~cpp
#include <iostream>
#include <vector>
#include <numeric>
int main() {
	std::vector<int> v;
	for (int i = 0; i <= 100; i++) v.push_back(i);
	int sum = std::accumulate(v.begin(), v.end(), 0);
	std::cout << sum << std::endl;
}
~~~

#### fill

**功能描述：**

- 向容器中填充指定的元素

**函数原型：**

- `fill(iterator beg, iterator end, value);  `

	// 向容器中填充元素

	// beg 开始迭代器

	// end 结束迭代器

	// value 填充的值

~~~cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>
int main() {
	std::vector<int> v;
	v.resize(10); // 因为涉及到v.begin(),所以用reserve()
	std::fill(v.begin(), v.end(), 255);
	std::for_each(v.begin(), v.end(), Print);
}
~~~

### 常用集合算法

**算法简介：**

- `set_intersection` // 求两个容器的交集
- `set_union` // 求两个容器的并集
- `set_difference ` // 求两个容器的差集

**注意点：**

* 集合算法要求两个容器都必须是有序的，且是同序
* 使用算法前，要保证保存结果的容器有足够`resize()`
* 算法返回的是结果的最后一个元素在结果容器的位置(迭代器)。因为`resize()`有可能开太大了，所以不能用`v3.end()`作为遍历结果的终点

#### set_intersection

**功能描述：**

- 求两个容器的交集

**函数原型：**

- `set_intersection(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);  `

	// 求两个集合的交集

	// **注意:两个集合必须是有序序列**

	// beg1 容器1开始迭代器 // end1 容器1结束迭代器 // beg2 容器2开始迭代器 // end2 容器2结束迭代器 // dest 目标容器开始迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v1, v2;
	for (int i = 0; i < 10; i++) {
		v1.push_back(i);
		v2.push_back(i + 5);
	}
    std::sort(v1.begin(), v1.end()); // 保证有序序列
    std::sort(v2.begin(), v2.end()); // 保证有序序列
	std::vector<int> v3;
	v3.resize(std::min(v1.size(), v2.size()));
	std::vector<int>::iterator it = std::set_intersection(v1.begin(), v1.end(), v2.begin(), v2.end(), v3.begin());
	std::for_each(v3.begin(), it, Print); // 注意结束位置不是v3.end()
}
~~~

**总结：**

- 求交集的两个集合必须的**有序序列**
- 目标容器开辟空间需要从**两个容器中取小值**
- **set_intersection返回值是交集中最后一个元素的位置**

####  set_union

**功能描述：**

- 求两个集合的并集

**函数原型：**

- `set_union(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);  `

	// 求两个集合的并集

	// **注意:两个集合必须是有序序列**

	// beg1 容器1开始迭代器 // end1 容器1结束迭代器 // beg2 容器2开始迭代器 // end2 容器2结束迭代器 // dest 目标容器开始迭代器

	~~~cpp
	void Print(int x) {
		std::cout << x << " ";
	}
	int main() {
		std::vector<int> v1, v2;
		for (int i = 0; i < 10; i++) {
			v1.push_back(i);
			v2.push_back(i + 5);
		}
		std::sort(v1.begin(), v1.end()); // 保证有序序列
		std::sort(v2.begin(), v2.end()); // 保证有序序列
		std::vector<int> v3;
		v3.resize(v1.size() + v2.size());
		std::vector<int>::iterator it = std::set_union(v1.begin(), v1.end(), v2.begin(), v2.end(), v3.begin());
		std::for_each(v3.begin(), it, Print);
	}
	~~~

**总结：**

- 求并集的两个集合必须的有序序列
- 目标容器开辟空间需要**两个容器相加**
- set_union返回值是并集中最后一个元素的位置

#### set_difference

**功能描述：**

- 求两个集合的差集

**函数原型：**

- `set_difference(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);  `

	// 求两个集合的差集

	// **注意:两个集合必须是有序序列**

	// beg1 容器1开始迭代器 // end1 容器1结束迭代器 // beg2 容器2开始迭代器 // end2 容器2结束迭代器 // dest 目标容器开始迭代器

~~~cpp
void Print(int x) {
	std::cout << x << " ";
}
int main() {
	std::vector<int> v1, v2;
	for (int i = 0; i < 10; i++) {
		v1.push_back(i);
		v2.push_back(i + 5);
	}
	std::sort(v1.begin(), v1.end()); // 保证有序序列
	std::sort(v2.begin(), v2.end()); // 保证有序序列
	std::vector<int> v3;
	v3.resize(std::max(v1.size(), v2.size()));
	std::vector<int>::iterator it = std::set_difference(v1.begin(), v1.end(), v2.begin(), v2.end(), v3.begin());
	std::for_each(v3.begin(), it, Print);
}
~~~

**总结：**

- 求差集的两个集合必须的有序序列
- 目标容器开辟空间需要从**两个容器取较大值**
- set_difference返回值是差集中最后一个元素的位置
