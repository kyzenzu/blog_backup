---
title: C++泛型编程——模板
date: 2026-01-30 22:36:49
tags:
  - C++
---
## 函数模板

#### 1.2.1简单定义一个函数模板

简单定义一个模板函数

~~~cpp
template<typename T>
void swap(T& a, T& b) {
	T tmp = a;
	a = b;
	b = tmp;
}
~~~

使用函数模板有两种方式：自动类型推导、显示指定类型

~~~cpp
int a = 10;
int b = 20;
swap(a, b); // 自动类型推导
swap<int>(a, b) // 显示指定类型
~~~

#### 1.2.2 函数模板注意事项

注意事项：

* 自动类型推导，必须推导出一致的数据类型`T`,才可以使用
	~~~cpp
	int a = 10;
	double b = 20;
	swap(a, b); // 用a推出的T和b推出的T不一致
	~~~

	用`a`推出`T`为`int`，但是用`b`推出T为`double`。产生了二义性，这让编译器无法构造出一个确定的函数。

* 自动类型推导，必须要能确定出T的数据类型，才可以使用

	~~~cpp
	template<typename T>
	void func() {
	    std::cout << "Hello World" << std::endl;
	}
	func(); // 错误,编译器无法知道T的类型
	~~~

	编译器无法知道`T`的类型，自然就无法构造出一个确定的函数。

#### 1.2.3 模板小应用

可以在函数模板中调用另一个模板，前提是能以当前模板的`T`正确推断出另一个模板的`T`。使编译器可以递归的构造出正确的函数。

~~~cpp
template<typename T>
void swap(T& a, T& b) {
	T tmp = a;
	a = b;
	b = tmp;
}

template<typename T>
void sort(T a[], int size) {
	for (int i = 0; i < size; i++) {
		int min = i;
		for (int j = i; j < size; j++)
			if (a[j] < a[min]) min = j;
		swap(a[i], a[min]);
	}
}
~~~

#### 1.2.4 普通函数与函数模板的区别

* 普通函数调用时可以发生自动类型转换（隐式类型转换） 
* 函数模板调用时，如果利用自动类型推导，不会发生隐式类型转换 
* 如果利用显式指定类型的方式，可以发生隐式类型转换

~~~cpp
template<class T>
T Add(T a, T b)  
{
	return a + b;
}
int main() {
    int a = 10;
    char b = 'c';    
}
~~~

错误用法：

~~~cpp
Add(a, b); // 错误 T有二义性
~~~

正确用法：

~~~cpp
 Add<int>(a, b); // 正确
~~~

其实也好想，本来`char`确实可以隐式转为`int`，但前提是`T`确定为`int`。但在自动类型推导根本不确定`T`，推导为`int`也行，推导为`char`也行。产生了**二义性**，这让编译器无法构造出一个确定的函数。

#### 1.2.5 普通函数和函数模板的调用规则

调用规则如下：

1. 如果函数模板和普通函数都可以实现，优先调用普通函数 
2. 可以通过空模板参数列表来强制调用函数模板 
3. 函数模板也可以发生重载 
4. 如果函数模板可以产生更好的匹配,优先调用函数模板

##### 优先调用普通函数

就是如果模板函数与普通函数同名同参数，编译器会优先调用普通函数。毕竟有一个现成确定的函数，编译器也懒得再去构造一个函数。

~~~cpp
#include <iostream>
template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板" << std::endl;
}

void Print(int a, int b) {
	std::cout << "调用普通函数" << std::endl;
}

int main() {
	int a = 10, b = 20;
	Print(a, b); // 调用普通函数
}
~~~

比较神奇的，即使只有一个普通函数的声明，编译器也会优先调用普通函数。因为编译器只管有声明，它相信函数会定义在其它编译单元，导致它没有根据模板构造函数。使得这个函数到最后也没有函数定义，最终没有通过链接阶段。

~~~cpp
#include <iostream>
void Print(int a, int b); // 编译器以为定义在其它编译单元,就没有根据模板再构造函数

template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板" << std::endl;
}

int main() {
	int a = 10, b = 20;
    Print(a, b); // 链接错误
}
~~~

在已经有普通函数声明的情况下，直接使用会产生链接错误

~~~cpp
Print(a, b); // 链接错误
~~~

##### 通过空模板参数列表强制调用函数模板 

调用时可以通过空模板参数列表对模板函数和普通函数的调用加以区分，并强制编译器根据函数模板进行构造。

~~~cpp
#include <iostream>
template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板" << std::endl;
}

void Print(int a, int b) {
	std::cout << "调用普通函数" << std::endl;
}

int main() {
	int a = 10, b = 20;
    Print(a, b); // 链接错误
}
~~~

明确调用模板函数：

~~~cpp
Print<>(a, b); // 调用函数模板
~~~

因为模板函数生成的函数是`void Print<int>(int, int)`，而普通函数是`void Print(int, int)`，两者的**签名**不同，所以不会造成重定义，链接阶段也能正确链接。

##### 函数模板也可以重载 

~~~cpp
template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板1" << std::endl;
}

template<typename T>
void Print(T a, T b, int) {
	std::cout << "调用函数模板2" << std::endl;
}
~~~

##### 如果函数模板可以产生更好的匹配,优先调用函数模板

什么叫更好的匹配？看下面的例子

~~~cpp
#include <iostream>
template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板" << std::endl;
}

void Print(int a, int b) {
	std::cout << "调用普通函数" << std::endl;
}

int main() {
	char a = 10, b = 20;
	Print(a, b); // 调用函数模板
}
~~~

可以发现，既可以调用普通函数（因为`char`类型可以隐式转为`int`）也可以调用函数模板。但是编译器发现可以根据函数模板将`T = char`，生成更直接对应的函数，还能省一步隐式转换。所以编译器选择根据模板构造相应的函数，并调用它。

#### 1.2.6 模板的局限性

局限性： 模板的通用性并不是万能的

例如：

~~~cpp
template<typename T>
void f(T a, T b) { 
	if(a > b) { ... }
}
~~~

在上述代码中，如果T的数据类型传入的是像Person这样的自定义数据类型，也无法正常运行 因此C++为了解决这种问题，提供**模板的重载**，可以为这些特定的类型提供**具体化的模板**

所谓具体化就是对模板中的`T`类型进行指定，这样特定类型的函数调用就会走特定类型的模板。**类似与给模板打补丁**

~~~cpp
#include <iostream>
#include <string>
class Person {
public:
	std::string m_Name;
	int m_Age;
	Person(std::string name, int age) : m_Name(name), m_Age(age) {}
};

template<typename T>
bool compare(T& a, T& b) {
	return a == b;
}
// 具体化，显示具体化的原型和定义以template<>开头，并通过具体类名来指出类型
// 具体化优先于常规模板
template<> bool compare(Person& a, Person& b) {
	if (a.m_Name == b.m_Name && a.m_Age == b.m_Age)
		return true;
	else
		return false;
}

int main() {
	Person a("Tom", 18);
	Person b("Tom", 18);
	bool ret = compare(a, b);
}
~~~

#### 偏特化

~~~cpp
template<typename Signature> 
class Function;  // 主模板

template<typename R, typename... Args>
class Function<R(Args...)> { 
public:
    R operator()(Args... args) { /* 调用内部可调用对象 */ }
};
int main() {
	Function<void(int)> f;
	f(1);
}
~~~

`R(Args..)`是一个整体，C++允许这种方式作为函数类型。



## 类模板

#### 1.3.1简单定义一个类模板

简单定义一个类模板：

~~~cpp
#include <iostream>
#include <string>
template<typename NameType, typename AgeType>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
	void Print() {
		std::cout << m_Name << std::endl;
		std::cout << m_Age << std::endl;
	}
};

int main() {
	Person<std::string, int> p("张三", 18);
	p.Print();
}
~~~

总结：类模板和函数模板语法相似，在声明模板template后面加类，此类称为类模板

#### 1.3.2 类模板与函数模板的区别

类模板与函数模板区别主要有两点： 

1. 只有函数模板可以使用自动类型推导，类模板不支持自动类型推导必须显式指定类型(C++14版本)
2. 函数模板和类模板都可以指定默认类型参数(C++14版本)。区别是，全部类型参数都默认指定后，**类模板用默认参数时必须加`<>`，函数模板可以省略。**

##### 类模板必须显式指定类型

~~~cpp
#include <iostream>
template<typename NameType, typename AgeType>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
};

int main() {
}
~~~

错误用法：

~~~cpp
Person p("张三", 18); // 错误, 类模板没有自动类型推导
~~~

正确用法：

~~~cpp
Person<std::string, int> p("张三", 18); // 正确，类模板必须显式指定类型
~~~

至于为什么类模板不支持自动类型推导这也好想。想用自动类型推导我们得这样用`Person p("张三", 18);`，但是这样顶多只能推出**构造函数的参数**的类型，但是模板类的**成员变量的类型**可能与**构造函数的参数**的类型半毛钱关系也没有，或者只是**成员变量的类型**的组成部分。

而函数模板可以支持自动类型推导是因为实参的类型就是形参的类型，完全绑定的。虽然模板类构造函数的形参的类型和调用时实参的类型是一样的也能推导，但这仍然与类的成员变量的类型没有关系。因此模板类的使用还是需要显式指明类型，尤其是成员变量的类型。

##### 指定默认类型参数

~~~cpp
#include <iostream>
#include <string>
template<typename NameType, typename AgeType = int>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
	void Print() {
		std::cout << m_Name << std::endl;
		std::cout << m_Age << std::endl;
	}
};

template<typename T = int>
void Print() {
	std::cout << "调用函数模板" << std::endl;
}

int main() {
    Person<std::string> p("张三", 18);
	Print(); // 指定默认参数类型后,T就能确定,编译器就能构造
}
~~~

#### 最好的总结例子
~~~cpp
#include <iostream>
#include <string>
template<typename NameType = std::string, typename AgeType = int>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
	void Print() {
		std::cout << m_Name << std::endl;
		std::cout << m_Age << std::endl;
	}
};
template<typename T = int>
void Print() {
	std::cout << "调用函数模板" << std::endl;
}
int main() {
    Person<> p("张三", 18); // 必须有<>
    Print(); // 可以省略<>
}
~~~
即使类模板已经有默认类型，使用时仍然必须有`<>`。因为它是模板类，名字就带有`<>`，与普通类不同

> 这种差异主要是历史原因造成的：
>
> - 类模板出现较早，设计上更保守
> - 函数模板需要更好的用户体验，所以允许更多省略
> - 一致性并不是C++的首要设计目标

#### 1.3.3 类模板中的成员函数的创建时机

类模板中成员函数和普通类中成员函数创建时机是有区别的： 

* 普通类中的成员函数一开始就可以创建
* 类模板中的成员函数在调用时才创建

类模板中的成员函数也是函数，创建时机和**模板函数**一样。都是在调用时才创建

~~~cpp
class Person1 {
public:
	void showPerson1() {}
};

class Person2 {
public:
	void showPerson2() {}
};

template<class T>
class Entity {
public:
	T person;
	void func1() {
		person.showPerson1();
	}
	void func2() {
		person.showPerson2();
	}
};
~~~

这里定义了两个类`Person1`和`Person2`，以及一个模板类`Entity`。按理来说，在模板类中`T`确定的情况下的`peron`只会是一种类型，要么是`Person1`，要么是`Person2`，意味着在类`Entity`被编译器构造出来后，`func1()`和`func2()`不应该共存。因为`person`不可能同时具有`showPerson1()`和`showPerson2()`。

但结果却能够编译通过。这就是因为编译器对模板的处理很特殊：**类模板中的成员函数在调用时才创建**

正确用法：

~~~cpp
int main() {
    Entity<Person1> e;
	e.func1();
}
~~~

这里`e`的类型指定为`Entity<Person1>`，并且调用了`func1()`，于是编译器在`Entity<Person1>`下创建了`Entity::func1()`函数，并且`Person1 e.person`确实有`showPerson1()`函数。所以编译通过了。

错误用法：

~~~cpp
int main() {
    Entity<Person1> e;
	e.func1();
    e.func2();
}
~~~

`e`的类型指定为`Entity<Person1>`，然后调用了`func2()`，于是编译器在`Entity<Person1>`下创建了`Entity::func2()`函数，但是指定为`Person1 person`并没有`showPerson2()`函数。于是编译器报错了。

**总结：类模板中的成员函数并不是一开始就创建的，在调用时才去创建**，编译器对模板的这种做法很大程度上提高了模板的泛型。

#### 模板的语法检测和构造时机

编译器一开始会简单的检查一下模板函数和类模板的语法，所谓简单就是看有没有**原则上的语法错误**，比如

~~~cpp
template<typename T>
void Print(T a, T b) {
	std::cout << "调用函数模板" << std::endl;
    sdfsdfsdfsdf;
}
~~~

像这样子乱写肯定是不通过的。下面还会再涉及一个关于友元的**原则错误**，这个错误还是很难想到的

更深层次的检查则会在`函数调用`和`使用类定义变量`时才会去构造函数和类，然后再检查看看是不是有比如`Entity = int`这样离谱的赋值操作，这个我称之为**逻辑错误**。

#### 1.3.4 类模板对象做函数的参数

现在定义一个类模板

~~~cpp
template<typename NameType, typename AgeType>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
	void ShowPerson() {
		std::cout << "姓名：" << m_Name << " 年龄：" << m_Age << std::endl;
	}
};
~~~

现在，我需要把这个类作为一个函数的参数，这个参数的类型又该怎么写呢？

一共有三种传入方式： 

1. 指定传入的类型   --- 直接显示对象的数据类型 
2. 参数模板化           ---  将对象中的参数变为模板进行传递 
3. 整个类模板化       --- 将这个对象类型 模板化进行传递

##### 指定传入的类型

把模板特化后完完整整的作为一个数据类型。

~~~cpp
void printPerson(Person<std::string, int>& p) {
	p.ShowPerson();
}

int main() {
	Person<std::string, int> p("张三", 18);
	printPerson(p);
}
~~~

##### 参数模板化

把函数改为模板函数，这样参数类型就不需要特化，用`typename`代替就可以了。然后调用时编译器就能根据`p`的类型`Person<std::string, int>`自动推导出`<NameType, AgeType>`

~~~cpp
template<typename NameType, typename AgeType>
void printPerson(Person<NameType, AgeType>& p) {
	p.ShowPerson();
}

int main() {
	Person<std::string, int> p("张三", 18);
	printPerson(p);
}
~~~

##### 整个类模板化

上面的方法是特意提取出`Person`中的两个数据类型作为模板类型，通过`p`的类型`Person<std::string, int>`与函数的参数类型`Person<NameType, AgeType>&`进行对比编译器就能推导出`NameType=std::string, AgeType=int`。

这次直接把整个`Person<NameType, AgeType>`作为`T`，让编译器根据`p`的类型完整的推导出参数类型。

由于已知`p`的类型`Person<std::string, int>`是特化后具体的数据类型，因此编译器就能够推导出`T`应该为`Person<std::string, int>`类型

~~~cpp
template<typename T>
void printPerson(T& p) {
	p.ShowPerson();
}

int main() {
	Person<std::string, int> p("张三", 18);
	printPerson(p);
}
~~~

#### 1.3.5 类模板与继承

当类模板碰到继承时，需要注意一下几点： 

* 当子类继承的父类是一个类模板时，子类在声明的时候，要指定出父类中T的类型 
* 如果不指定，编译器无法给子类分配内存 
* 如果想灵活指定出父类中T的类型，子类也需变为类模板

比如说：

~~~cpp
template<typename T>
class Base {
public:
	T m;
};

class Derived : public Base {
    
};
~~~

像这样去继承肯定是不行的。父类是一个模板，编译器编译时在没有用它定义变量的情况下可以不管它，但是子类不是模板是一个必须要定义的类，它继承父类就得要腾出空间存储父类的变量，但现在父类多大都不知道，子类自然无法定义。所以此时编译器不能让它通过。

解决思路也有两条：

1. 确定父类的大小。也就是子类在声明的时候，指定出父类中T的类型 
	~~~cpp
	template<typename T>
	class Base {
	public:
		T base;
	};
	
	class Derived : public Base<int> {
	    
	};
	~~~

2. 把子类也定义成模板。这样编译器就能视情况是否要放过这个子类。
	~~~cpp
	template<typename T>
	class Base {
	public:
		T base;
	};
	
	template<typename BaseType, typename DerivedType>
	class Derived : public Base<BaseType> {
	public:
		DerivedType derived;
	};
	~~~

	像这个样子就能在使用子类时定义子类的同时顺便还能把父类一起定义出来。编译器也就能够根据提供的类型构造出确定的父类和子类。
	~~~cpp
	int main() {
		Derived<int, char> d;
	}
	~~~

	根据数据类型`Derived<int, char>`，编译器就可以推断出`BaseType=int, DerivedType=char`。然后就可以递归的定义出父类。

	~~~cpp
	class Base {
	public:
		int base;
	}
	class Derived : public Base<int> {
	public:
		char derived;
	};
	~~~

#### 1.3.6 类模板成员函数类外实现

类模板成员函数在类外实现还是需要点讲究，这涉及到一个很关键的细节。

下面写一个错误的示例：

~~~cpp
template<typename NameType, typename AgeType>
class Person {
public:
	NameType m_Name;
	AgeType m_Age;
	Person(NameType, AgeType);
	void ShowPerson();
};

template<typename NameType, typename AgeType>
Person::Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
~~~

编译器在不知道模板的情况下直接看`NameType`和`AgeType`肯定不认识，所以在头上加上`template`关键字很正常。但是为什么错了呢，因为直接用`Person::Person()`去定义

正确的定义方式是：

~~~cpp
template<typename NameType, typename AgeType>
Person<NameType, AgeType>::Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
~~~

为什么要加一个尖括号呢？

##### 一个模板可以对应无数个类

```cpp
Person<int, int>
Person<std::string, int>
Person<const char*, double>
```

你写：

```cpp
Person::Person
```

编译器会反问一句（这是关键）：

> **你是想给哪一个 `Person<?, ?>` 定义构造函数？**

C++ 不允许这种模糊性。

归根结底，模板类还是和普通的类不一样。一个模板可以定义出多个不同的类：`Person<int, int`，`Person<int char>`都是不同的类，它们是可以共存的，不像普通的类`Person`就是它的名字。所以由模板定义出类它的名字带个`<>`还是有必要的。

带上`<>`后上面函数的定义不仅确定了`NameType`和`AgeType`，还能确定这个定义的函数是属于哪个`Person<>`模板类的。像这样：

~~~cpp
class Person {
public:
	std::string m_Name;
	int m_Age;
	Person(std::string, int);
	void ShowPerson();
};
Person<std::string, int>::Person(std::string name, int age) : m_Name(name), m_Age(age) {}
void Person<std::string, int>::ShowPerson() {
	std::cout << "姓名：" << m_Name << " 年龄：" << m_Age << std::endl;
}
~~~

总结：

* 类模板类内实现写到`Person`时都不需要带上`<T>`，不管是构造函数、析构函数、函数参数还是返回类型。因为这是在模板类内，编译器自然知道`Person`就是`Person<T>`
* 类模板中成员函数类外实现时，需要加上模板参数列表`<>`，单单`Peron::Person()`根本不知道是哪个具体模板类的函数。此外，类外也需要`Person<T>`来作为模板类的唯一签名。
* 类内可以直接用`Person`指代；类外最好都用`Person<T>`来表示类的唯一签名。

#### 1.3.7 类模板分文件编写

根本解决办法：**让编译器意识到去构造函数的定义。在一般函数中调用模板函数，就能让编译器意识到。**

编译单元为什么称之为**编译单元**：因为编译器在开始编译当前的单元时，就会清空自己用于编译功能的缓存区，根本不会管上一个单元编译出的东西。编译时期单元和单元之间非常隔离。

假如把类模板的东西分文件编写，比如：

`Person.h`

~~~cpp
#pragma once
#include <iostream>
template<typename NameType, typename AgeType>
class Person {
public:
	NameType m_Name;
	AgeType m_Age = m_Name;
	Person(NameType, AgeType);
	void ShowPerson();
};
~~~

`Person.cpp`

~~~cpp
#include "Person.h"
template<typename NameType, typename AgeType>
Person<NameType, AgeType>::Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}

template<typename NameType, typename AgeType>
void Person<NameType, AgeType>::ShowPerson() {
	std::cout << "姓名：" << m_Name << " 年龄：" << m_Age << std::endl;
}
~~~

`Main.cpp`

~~~cpp
#include <iostream>
#include "Person.h"

int main() {
	Person<std::string, int> p("张三", 18);
    p.ShowPerson();
}
~~~

模拟一下编译器编译流程：

1. `Person.h`，是头文件不是`.cxx`后缀不是编译单元，略过
2. `Person.cpp`，知道了两个模板函数，但是当前编译单元下没有调用，也就不构造了，结束
3. `Main.cpp`，通过头文件包含知道了有`Person`这个类但也仅仅是个声明，认为函数的定义在别的编译单元，也就让它通过了。

仔细一看上面，发现根本没有`void Person<NameType, AgeType>::ShowPerson()`构造函数和`void Person<NameType, AgeType>::ShowPerson()`两个函数的定义。本来这两个函数的定义我们是想要交给编译器来构造，可是编译器在`Person.cpp`这个函数单元中根本没有构造，所以到了链接阶段，这两个函数的定义还是没有。于是出了链接错误。

知道了原因后就知道怎么解决：**让编译器在某个编译单元中把函数的定义构造出来。**

* 比较好的解决办法：把整个模板的声明和定义都放在`.hpp`文件中，让`Main.cpp`来包含就可以了。因为`Main.cpp`中函数的调用，所以编译器会去构造这个类的成员函数。

* 还有一个我自己想的奇怪方法，在`Person.cpp`中自己写一个普通函数来调用这个类的成员函数，让编译器在编译`Person.cpp`单元时，知道需要把模板类的两个函数构造出来，这样就有了函数定义。

`Person.cpp`

~~~cpp
#include "Person.h"
template<typename NameType, typename AgeType>
Person<NameType, AgeType>::Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}

template<typename NameType, typename AgeType>
void Person<NameType, AgeType>::ShowPerson() {
	std::cout << "姓名：" << m_Name << " 年龄：" << m_Age << std::endl;
}

void func() {
	Person<std::string, int> p("张三", 18);
	p.ShowPerson();
}
~~~

总结根本解决办法：**让编译器意识到去构造函数的定义。在一般函数中调用模板函数，就能让编译器意识到。**

#### 1.3.8 类模板与友元

* 全局友元函数类内实现 - 直接在类内声明友元即可 

* 全局友元函数类外实现 - 需要提前让编译器知道全局函数的存在

###### 全局友元函数类内实现

~~~cpp
template<typename NameType, typename AgeType>
class Person {
	friend void PrintPerson(Person<NameType>, AgeType>& p) {
		std::cout << "姓名：" << p.m_Name << " 年龄：" << p.m_Age << std::endl;
	}
private:
	NameType m_Name;
	AgeType m_Age;
public:
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
};

int main() {
	Person<std::string, int> p("张三", 18);
	PrintPerson(p);
}
~~~

全局友元函数类内实现最简单，因为就定义在模板类中，编译器推断出模板`Person<>`的具体类型后，在构造时就能在模板中得知`PrintPerson()`是模板具体类`Person<std::string, int>`的友元，不会乱关联到其它具体类的友元关系。并且由于在模板中，也省去理解`NameType`和`AgeType`。同时还知道了函数的定义。

###### 全局友元函数类外实现

这个过程会经历层层错误，让我们来逐层剖析。

~~~cpp
template<typename NameType, typename AgeType>
class Person {
	friend void PrintPerson(Person<NameType, AgeType>&);
private:
	NameType m_Name;
	AgeType m_Age;
public:
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
};

template<typename NameType, typename AgeType>
void PrintPerson(Person<NameType, AgeType>& p) {
	std::cout << "姓名：" << p.m_Name << " 年龄：" << p.m_Age << std::endl;
}

int main() {
	Person<std::string, int> p("张三", 18);
	PrintPerson(p);
}
~~~

首先模板类的友元函数定义为模板函数这很容易想到。但这样定义还是错的，而且还是链接错误：
编译器检错没有问题，也就是说，没能找到友元函数的定义。

虽然在第二行，`PrintPerson(p);`调用促使编译器构造出了`PrintPerson<std::string, int>(Person<std::string, int>&)`类型的函数，但这个函数是由模板构造出来的函数，它有`<>`。注意模板类中友元的声明没有`<>`，也就是说它认为这个友元是普通函数，这两个因为名字的不同，被视为两个不同的函数。所以在链接阶段，没有`PrintPerson(Person<NameType, AgeType>&)`普通函数的定义。

这个错误的关键在：**没有认识到普通函数和模板函数的签名有所不同**。

再修改一下呢？

~~~cpp
template<typename NameType, typename AgeType>
class Person {
	friend void PrintPerson<NameType, AgeType>(Person<NameType, AgeType>&);
private:
	NameType m_Name;
	AgeType m_Age;
public:
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
};

template<typename NameType, typename AgeType>
void PrintPerson(Person<NameType, AgeType>& p) {
	std::cout << "姓名：" << p.m_Name << " 年龄：" << p.m_Age << std::endl;
}

int main() {
	Person<std::string, int> p("张三", 18);
	PrintPerson(p);
}
~~~

在类中声明友元时标注上它是一个模板函数了。看似能对，但还是错了。而且变成了编译错误，也就是说编译器不认可这个语法。

前面说了，编译器一开始会检查模板的**原则错误**，会在模板类构造出来后再检查**逻辑错误**。这里错了不可能会是逻辑错误，看来是原则错误，那具体是什么原则错误呢。

##### C++ 规定

> **只有在名字已经被声明为模板时，才能使用 `<...>` 作为模板实参列表**

在此之前，我们并没有声明`PrintPerson<NameType, AgeType>()`是模板，所以还要再添加它是模板的声明。

但加了`void PrintPerson<NameType, AgeType>()`是函数模板的声明后，编译器不知道类`Person<NameType, AgeType>`是啥啊，所以还要再加一个类模板`class Person<NameType, AgeType>`的声明。

最终结果

~~~cpp
template<typename NameType, typename AgeType>
class Person;

template<typename NameType, typename AgeType>
void PrintPerson(Person<NameType, AgeType>&);

template<typename NameType, typename AgeType>
class Person {
	friend void PrintPerson<NameType, AgeType>(Person<NameType, AgeType>&);
private:
	NameType m_Name;
	AgeType m_Age;
public:
	Person(NameType name, AgeType age) : m_Name(name), m_Age(age) {}
};

template<typename NameType, typename AgeType>
void PrintPerson(Person<NameType, AgeType>& p) {
	std::cout << "姓名：" << p.m_Name << " 年龄：" << p.m_Age << std::endl;
}
~~~

~~~cpp
int main() {
	Person<std::string, int> p("张三", 18);
	PrintPerson(p);
}
~~~

总结：建议全局函数做类内实现，用法简单，而且编译器可以直接识别

---

### 遇到的小坑

看看以下代码：为什么定义`it`一定要添加`typename`关键字？

~~~cpp
template<typename T, class Cmp>
void PrintSet(const std::set<T, Cmp>& s) {
	for (typename std::set<T, Cmp>::const_iterator it = s.begin(); it != s.end(); it++)
		std::cout << *it << " ";
	std::cout << std::endl;
}
~~~

模板里凡是 `T::xxx`，如果它是类型，就必须写 `typename`

> **模板在“第一次看到代码”时，编译器根本不知道
>  `T::xxx` 到底是不是一个类型。**

所以：
 👉 **默认当“不是类型”处理**，C++编译器默认当成是某个域下的静态变量处理
 👉 除非你显式告诉它：这是类型（`typename`）

### 偏特化和具体化

#### 一、概念区别

| 特性         | 模板偏特化（Partial Specialization）              | 模板显式具体化（Explicit Specialization）      |
| ------------ | ------------------------------------------------- | ---------------------------------------------- |
| **定义**     | 针对模板参数的一部分或特定形式提供专门实现        | 针对模板参数的具体类型提供专门实现             |
| **适用对象** | 只适用于 **类模板**                               | 类模板和函数模板都可以（但函数模板不能偏特化） |
| **匹配方式** | 编译器在实例化时匹配最合适的偏特化                | 编译器在实例化时匹配完全指定的具体化           |
| **语法**     | `template<typename T> class MyClass<T*> { ... };` | `template<> class MyClass<int> { ... };`       |
| **特征**     | 类型参数可以是部分固定、部分泛化                  | 类型参数完全指定，不再泛化                     |

------

#### 二、类模板示例

###### 1️⃣ 主模板

```
template<typename T>
class MyClass {
public:
    void info() { std::cout << "通用类型\n"; }
};
```

###### 2️⃣ 偏特化，针对某种形式

```
template<typename T>
class MyClass<T*> {   // 针对指针类型
public:
    void info() { std::cout << "指针类型\n"; }
};
```

- 这里只是 **匹配 T\* 形式**，不是完全固定某个类型
- 调用：

```
MyClass<int> a;   a.info(); // 通用类型
MyClass<int*> b;  b.info(); // 指针类型
```

✅ 编译器会选择最合适的模板

###### 3️⃣ 显式具体化，完全固定类型

```
template<>
class MyClass<int> {   // 针对 int 类型
public:
    void info() { std::cout << "int 类型\n"; }
};
```

- **完全固定类型**：只有 `MyClass<int>` 使用
- 其他类型用主模板或偏特化

------

#### 三、函数模板示例

> 注意：**函数模板不能偏特化**，只能显式具体化

###### 1️⃣ 主模板

```
template<typename T>
bool compare(T& a, T& b) {
    return a == b;
}
```

###### 2️⃣ 显式具体化

```
template<>
bool compare<std::string>(std::string& a, std::string& b) {
    return a.size() == b.size(); // 特殊处理
}
```

- 只能写 **完全指定类型**
- 不能写 “偏特化某类函数模板”，比如 `template<typename T> compare<T*>` ❌

------

#### 四、总结记忆口诀

1. **偏特化 = 部分固定 + 泛化**
	- 类模板可以
	- 函数模板不行
	- 例子：`MyClass<T*>`，匹配指针类型但 T 泛化
2. **显式具体化 = 完全固定**
	- 类模板、函数模板都可以
	- 例子：`MyClass<int>` 或 `compare<std::string>`
3. **优先级**

- 当模板实例化时，编译器**优先选择显式具体化**
- 然后选择偏特化
- 最后使用主模板

------

#### 五、可视化理解

```
类模板:
        主模板: template<typename T> MyClass<T>
        偏特化: template<typename T> MyClass<T*>  <-- T 可以变化
        显式具体化: template<> MyClass<int>  <-- 完全固定

函数模板:
        主模板: template<typename T> compare<T>
        显式具体化: template<> compare<std::string>  <-- 完全固定
        （偏特化不允许）
```

> ⚠️ 核心区别：偏特化**部分匹配**，显式具体化**完全固定**
