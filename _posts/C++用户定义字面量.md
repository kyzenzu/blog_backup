---
title: C++用户定义字面量
date: 2026-01-29 17:26:04
tags:
  - C++
---

### 用户定义字面量

`C++11`新规定的语法，用户可以自定义作用在字面量上的函数，由编译器完成传参和函数调用。

- **功能**：让程序员自定义字面量后缀
- **语法**：`operator""后缀名(参数)`，这个双引号只是个形式而已不必在调用时真的需要出现双引号
- **位置**：只能是**后缀**（如 `100km`），不能是前缀

作用对象有以下四种基本类型的字面量

#### 1. 整数字面量

语法：

`返回类型 operator""后缀(unsigned long long 值)`

示例：

~~~cpp
int operator""km(unsigned long long m) {
    return m * 1000;  // 千米转米
}
int distance = 5km;  // 5000
~~~

#### 2. 浮点数字面量

语法：

`返回类型 operator""后缀(long double 值)`

示例：

~~~cpp
double operator""deg(long double rad) {
    return rad * 180.0 / 3.1415926535;
}
double angle = 3.141592deg;  // ≈180°
~~~

#### 3. 字符字面量

语法（四种编码方式）：

`返回类型 operator""后缀(char 值)`
`返回类型 operator""后缀(wchar_t 值)`
`返回类型 operator""后缀(char16_t 值)`
`返回类型 operator""后缀(char32_t 值)`

示例：

~~~cpp
char operator""up(char c) {
    return toupper(c);
}
char ch = 'a'up;  // 'A'
~~~

#### 4. 字符串字面量（最常用！）

语法：

`返回类型 operator""后缀(const char* str, size_t 长度)`

示例：

~~~cpp
std::string operator""s(const char* str, size_t len) {
    return std::string(str, len);
}
std::string str = "Hello"s;  // std::string类型
~~~

* 对字符串字面量的处理非常特殊，编译器会不仅进会传入字符串字面量`const char*`地址，还会传入这个字面量的长度(不包括`\0`)