---
title: C++构造函数的调用方式
date: 2024-3-4
tags: 
  - C++
  - C/C++
---
~~~c++
#include <iostream>
class Person {
private:
	int m_Age;
public:
	Person() {
		std::cout << "无参构造/默认构造函数" << std::endl;
	}
	~Person() {
		std::cout << "析构函数" << std::endl;
	}
	Person(int a) {
		std::cout << "有参构造函数" << std::endl;
		m_Age = a;
	}
	Person(const Person& other) {
		std::cout << "拷贝构造函数" << std::endl;
		m_Age = other.m_Age;
	}
};

int main() {
    // 1.括号法
    Person p1(); // 函数声明 注意事项1 无参构造用括号法
    Person p2(18);
    Person p3(other);
    
    // 2.显式法
    Person p1 = Person();
    Person p2 = Person(18);
    Person p3 = Person(other);
    
    // 3.隐式法
    Person p1;
    Person p2 = 18;
    Person p3 = other;
   ----------------------------------------------------------------------------- 
    // 匿名对象
    Person();
    Person(18);
    Person(p2); // 重定义 Person(p2) == Person p2 注意事项2 匿名对象用拷贝
    而 C++ 语法规则规定：

	当一条语句“既可以被解析为声明，又可以被解析为表达式”时，必须优先解析为声明。
   ----------------------------------------------------------------------------- 
    //无参构造
    Person p1(); // 函数声明 注意事项1 无参构造用括号法
    Person p2 = Person();
    Person p3;
    
    // 有参构造
    Person p1(18);
    Person p2 = Person(18);
    Person p3 = 18;
    
    // 拷贝构造
    Person p1(other);
    Person p2 = Person(other);
    Person p3 = other;
}
~~~

