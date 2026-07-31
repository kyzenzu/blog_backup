---
title: C++类和对象的运算符重载
date: 2026-01-23 19:45:22
tags:
  - C++
---

### 类声明

~~~cpp
class Integer {
	friend std::ostream& operator<<(std::ostream&,const Integer&);
private:
	int m_Data;
public:
	Integer(): m_Data(0);
    Integer operator+(const Integer&); // 加号运算符重载
	Integer& operator++(); // 前置++运算符重载
	Integer operator++(int); // 后置++运算符重载
    Integer& operator=(const Integer&); // 赋值运算符重载 深拷贝
    bool operator==(const Integer&); // 关系运算符重载
    void operator()(const char*); // 函数调用运算符重载
};
~~~

#### 加号+运算符重载

~~~cpp
// 全局运算符重载 operator+(i1, i2) => i1 + i2
Integer operator+(const Integer& i1, const Integer& i2) {
    Integer tmp;
    tmp.m_Data = i1.m_Data + i2.m_Data;
    return tmp;
}
// 类内运算符重载 i1.operator+(i2) => i1 + i2
Integer Integer::operator+(const Integer& other) { // 返回临时对象
    Integer tmp;
    tmp.m_Data = this->m_Data + other.m_Data;
    return tmp;
}
~~~

#### 左移输出<<运算符重载

~~~cpp
// 全局运算符重载
std::ostream& operator<<(std::ostream& ostream,const Integer& integer) {
    ostream << integer.m_Data;
    return ostream;
}
~~~

#### 前置++递增运算符重载

~~~cpp
Integer& Integer::operator++() { // 实现++(++a)
    m_Data++;
	return *this;
}
~~~

#### 后置++递增运算符重载

~~~cpp
Integer Integer::operator++(int) { // 只能 a++
	Integer tmp = *this;
	m_Data++;
	return tmp;
}
~~~

#### 赋值=运算符重载

~~~cpp
// 深拷贝
Integer& Integer::operator=(const Integer& other) { // 实现 a = b = c
    if (m_Data != nullptr) {
        delete m_Data;
        m_Data = nullptr;
    }
    m_Data = new int(*other.m_Data);
    return *this;
}
~~~

#### 关系运算符重载

~~~cpp
bool Integer::operator==(const Integer& other) {
    bool res = this->m_Data == other.m_Data ? true : false;
    return res;
}
~~~

#### 函数调用运算符重载

~~~cpp
// 必须是类内函数
void Integer::operator()(const char* str) {
    std::cout << str << std::endl;
}
~~~

赋值（`=`）、下标（`[]`）、函数调用（`()`）和成员访问箭头（`->`）运算符必须是类内成员函数。

像这些操作数明显有左右固定顺序的操作符，比如下标运算符左边必须是数组对象，里面必须是整型，像这样：`Array[5]`。如果允许定义全局的运算符重载，那么程序员可以这样定义：`void operator[](int x, Array array)`，那么这个函数可以这样子调用`6[a]`，**就非常的诡异**。因此直接禁止掉全局的定义方式，只允许类内运算符重载，**使得自定义对象类型固定在运算符的左侧**。