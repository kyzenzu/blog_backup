---
title: 【The Cherno】枚举的使用
date: 2026-01-17 19:11:27
tags:
  - C++
---

### 枚举

基本语法

~~~cpp
enum Color {
	RED, YELLOW, BLUE
};
int main() {
	Color color = RED;
	return 0;
}
~~~

1. `enum`基本含义就是定义一种数据类型`Color`，这种类型的变量的值只能是`RED`，`YELLOW`或者`BLUE`，并且只能用这三个关键字赋值，`0`、`1`、`2`这些数字常量都不行

2. `RED`的实际值默认为`0`，也可以改变`YELLOW`的值，后面的`BLUE`依次递增

  ~~~cpp
  enum Color {
  	RED /* 0 */, YELLOW = 35, BLUE /* 36 */
  };
  ~~~

3. 直接用`enum`定义的枚举类型会打破域污染全局关键字。同时可能造成重定义问题

  ~~~cpp
  enum Color1 {
  	RED, YELLOW, BLUE
  };
  enum Color2 {
  	RED, YELLOW, BLUE
  };
  int main() {
  	Color1 color = RED; // 报错, RED重定义,编译器分不清是谁的 RED
  	Color1 color = Color1::RED; // 同样报错,也是RED重定义
  	return 0;
  }
  ~~~
根本问题不是有没有使用`Color1::`域的问题。而是`enum`这种枚举定义方式本身就有问题，它里面的关键字就是会打破域跑出来，因此会造成`RED、YELLOW、BLUE`的重定义问题

4. `enum`定义的枚举类型默认是`4`字节大小的带符号`int`，可以用基本数据类型规定其类型以及占用的内存空间

  ~~~cpp
  enum Color : char {
  	RED, YELLOW, BLUE
  };
  ~~~
  ~~~cpp
  #include <iostream>
  class Log {
  public:
  	enum Level {
  		ERROR = 0, WARNING, INFO
  	};
  private:
  	Level m_LogLevel = INFO;
  public:
  	void SetLogLevel(Level level) {
  		m_LogLevel = level;
  	}
  	void Error(const char* message) {
  		if (m_LogLevel >= ERROR)
  			std::cout << "[ERROR]: " << message << std::endl;
  	}
  	void Warning(const char* message) {
  		if (m_LogLevel >= WARNING)
  			std::cout << "[WARNING]: " << message << std::endl;
  	}
  	void Info(const char* message) {
  		if (m_LogLevel >= INFO)
  			std::cout << "[INFO]: " << message << std::endl;
  	}
  };
  int main() {
  	Log log;
  	log.SetLogLevel(Log::WARNING); // 打破域的体现
      log.SetLogLevel(log.INFO); // 打破域后变成类似staitc的静态成员变量
  	log.Warning("Hello World");
  	return 0;
  }
  ~~~


### 枚举类

由于上面的第3个问题，C++推出新的关键字`enum class`，给枚举下的三个关键字划定了域。枚举类不是类，只不过恰好用了`class`关键字，而`class`也有**域**的意思

~~~cpp
enum class Color1 {
	RED = -20, YELLOW, BLUE
};
enum class Color2 : char {
	RED = 97, YELLOW, BLUE
};
int main() {
	Color1 color1 = Color1::RED;
    Color2 color2 = Color2::RED;
    std::cout << color1 << std::endl; // 需要函数重载
	std::cout << color2 << std::endl; // 需要函数重载
	return 0;
}
~~~

