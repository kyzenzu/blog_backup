---
title: 【The Cherno】杂记
date: 2026-01-15 20:08:57
tags:
  - C/C++
---



#### struct 和 class 的区别

* `struct` 和 `class` 的唯一区别就是默认的可见性。`struct`默认的可见性是`public`，`class`默认的可见性是`private`

##### 全局函数和全局变量的 static

* `static`作用在全局函数或全局变量时`static int s_Varible`，**当前编译单元**在使用全局变量时会链接到当前编译单元的**静态全局变量**；同时，**其它编译单元**无法观测到这个**静态全局变量**，也就无法链接

##### 函数重载中参数的 const

* 允许`void f(int& a)`和`void f(const int& a)`重载，但是不允许`void f(int a)`和`void f(const int a)`重载。是因为前面引用的`const`是**底层const**，编译器编译后保留`const`，最终判断函数内的参数是不同类型，允许重载；但是后面的`const`属于**顶层const**，编译器编译会去除`const`，最终判断函数内的参数是相同的，不允许重载。
* 所谓顶层`const`指这个`const`修饰**变量本身**，而底层`const`指这个`const`修饰变量**指向的那个对象**。C++规定编译后去掉顶层`const`（不影响代码行为和最终效果），保留底层`const`作为不同的数据类型

##### 括号法调用无参构造，拷贝构造生成匿名对象

* `Person p()`，被编译器认为是函数声明

* `Person(other)`，被编译器认为是声明定义`other`造成重定义

* 第一个没什么解决办法。第二个本意是作为表达式生成一个拷贝`other`的匿名对象，但是编译器认为是一个变量的声明，这是因为：

	而 **C++ 语法规则规定**：

	> **当一条语句“既可以被解析为声明，又可以被解析为表达式”时，必须优先解析为声明。**

* 解决办法就是外面套一层括号`(Person(other))`，因为声明不允许以 `(` 开头，编译器只能把它当成表达式，因此调用拷贝构造函数，生成匿名对象

#### 构造函数默认情况是否存在

* 拷贝构造函数总是有的，默认或者自定义

* 自定义有参构造函数会**顶替**默认构造(/无参构造)函数

* 自定义拷贝构造函数也会**顶替**默认构造(/无参构造)函数
	
	~~~cpp
	class Person {
	private:
		int m_Age;
	public:
		Person() {
	    }
	    Person(const Person& other) {
			m_Age = other.m_Age;
		}
		~Person() {	
		}
	    Person& operator(const Person& rhs) {
	        m_Age = rhs.m_Age;
	    }
	};    
	~~~

![](../posts_img/杂记/默认构造/默认情况.png)

#### const修饰成员函数

* `non-static`成员函数中的参数`this`是`Entity* const this`类型属于自带顶层`const`。但如果我们想要`const Entity* const this`底层`const`类型怎么办，由于`this`指针不表现在函数定义中，`const`放在参数列表里或者函数名左边都不合适，于是放在了函数的右边。

	~~~cpp
	class Entity {
	private:
		int m_Age;
	public:
		void func() const {
	        
	    }
	};
	~~~

	函数右边的`const`用于修饰`this`参数的底层`const`，使得`const Entity* const this`类型

* 为什么常对象只能调用常函数？防止一般的函数修改常对象的值，这个解释固然合理，但我觉得还能有另一个解释

* 常对象`const Entity e`取地址生成的指针是`const Entity*`类型，有底层`const`修饰。但是一般函数的`this`指针属于`Entity* const`类型，`void func(Entity* const this)`
	我们知道底层`const`不能忽视，不能把`const Entity*`赋值给`Entity* const this`，因此调用不成立。而常函数`void func(const Entity* const this) const`有底层`const`修饰，因此常对象只能调用常函数。

* 总结来说，带底层`const`修饰的指针不能赋值给一般指针，但是一般指针可以赋值给带底层`const`修饰的指针

#### 类对象作为成员变量

在继承中，父类在子类中也相当于类对象，仍然是一样的构造和析构顺序

~~~cpp
#include <iostream>
class Entity {
public:
	Entity() {}
    ~Entity() {}
};
class Person {
public:
    int x;
    int y;
    Entity e;
	Person()
    { // 在这里调用Entity::Entity(),构造Entity e
        // 子类可能会用到父类的东西，所以先构造父类
    }
    ~Person() 
    {
        // 子类可能还有用到父类的东西，所以后析构父类
    } // 在这里调用Entity::~Entity(),析构Entity e
};

int main() {
    Person p;
} 
~~~

~~~assembly
Person::Person()
push	ebp
mov		ebp, esp
sub		esp, 8

; { 左括号展开为以下指令
mov		eax, DWORD PTR [this]
add 	eax, 8
sub		esp, 12
push 	eax
call	Entity::Entity()
add		esp, 16

nop
leave
ret
~~~

~~~assembly
Person::~Person()
push    ebp
mov     ebp, esp
sub     esp, 8

;} 右括号展开为以下指令
mov     eax, DWORD PTR [this]
add     eax, 8
sub     esp, 12
push    eax
call	Entity::~Entity()

add     esp, 16
nop
leave
ret
~~~



#### 利用VS自带cmd查看类的结构

~~~plaintext
cl /d1 reportSingleClassLayout<ClassName> main.cpp
~~~



#### 继承方式

~~~cpp
class Base {
public: 
	int m_A; // 继承后视继承方式分配到不同的访问域
protected: 
	int m_B; // 继承后视继承方式分配到不同的访问域
private: // 不可访问无法撼动
	int m_C;
};
~~~

* 基类的` private` 成员在派生类中不管怎么样都不可直接访问；
* `public` 和 `protected` 成员会根据继承方式，在派生类中重新分配访问域
* `public继承`保持不变（统一收紧为`public`），`protected继承`统一收紧为 `protected`，`private继承`统一收紧为`private`。

不同的继承方式，父类的成员在子类中访问权限不同的分配情况：

~~~cpp
class Derived : public Base {
public:
	int m_A;
protected:
	int m_B;    
不可访问:
	int m_C;
};
~~~

~~~cpp
class Derived : protected Base {
protected:
	int m_A;
	int m_B;
不可访问:
	int m_C;
};
~~~

~~~cpp
class Derived : private Base {
private:
	int m_A;
	int m_B;
不可访问:
	int m_C;
};
~~~

---

#### 关于函数签名的问题（非常重要）

> * ==函数签名：函数的唯一标识符。组成部分只有函数名、参数列表、`const`修饰符==
>
> * <u>区别于编译器视角下对函数的自定义命名</u>
> * ==返回类型不参与函数签名，但是作为编译器自定义命名函数的一部分。==
>
> ==总结来讲，编译器判断一些函数行为就是根据函数签名。编译器视角下对函数的自定义命名或许只有在最后编译器帮助链接器链接时才会用到。==

---

#### 继承中同名成员

* 当子类与父类拥有同名的成员函数，子类会隐藏父类中所有重载的同名成员函数。仅仅只需要同名即可，无关乎其它 
* 成员变量也是同理

#### 虚函数小问题

* 编译器判断子类的函数是不是虚函数的重写，只会根据**函数签名**来判断。不根据返回类型判断。

* 一旦某个函数在继承链中被声明为 `virtual`，它在**所有派生类中都保持虚函数特性**，无论后续的派生类是否显式使用 `virtual` 关键字。**原因**应该是派生类继承父类虚函数表，派生类中与父类相同签名的函数必须注册在虚函数表中，因此即使没有虚函数标签这个函数也是虚函数。
* **不能"去虚拟化"**：无法在派生类中将虚函数变回非虚函数。
* 为什么静态函数不能是虚函数？因为虚函数存在于虚函数表中，而虚函数表只存在于具体对象中，但是静态函数是被规定可以通过类名调用`Entity::staticFunc()`，这两者明显矛盾了。
* ⭐构造函数不能是虚函数，因为虚函数需要虚函数表，在调用构造函数时对象还没创建好不可能有虚函数表的地址。并且虚函数表的地址是在调用构造函数后在构造函数的开头进行填写的。
* 析构函数最好是虚函数。

#### 虚析构和纯虚析构

* 虚析构：`virtual ~Animal() {}`，纯虚析构：`virtual ~Animal() = 0`

* 为什么含有纯虚析构的抽象基类被继承后必须实现这个纯虚析构函数，而一般的函数却不需要？
	~~~cpp
	class Animal {
	public:
		virtual ~Animal() = 0;
		virtual void func() = 0;
	};
	Animal::~Animal() {
		std::cout << "Animal析构了" << std::endl;
	}
	class Cat : public Animal {
	public:
		int* m_Age;
		void speak() override {
			std::cout << "Cat在说话" << std::endl;
		}
		void func() override {
			std::cout << "Cat Func" << std::endl;
		}
		~Cat() {
			delete m_Age;
			std::cout << "Cat析构了" << std::endl;
		}
	};
	int main() {
		Animal* a = new Cat;
		a->speak();
		a->func();
		delete a;
	}
	~~~

	首先，C++中抽象类`Animal`不允许实例化，也就是说不可能有`Animal a`对象存在，这个对象的虚函数表`vfptr`中的`func`按理来说是`Animal::func`，但由于这个对象不可能存在，因此源码中不可能出现`a.func()`的调用。
	为什么会提到调用？因为不实现`Animal`纯虚析构函数产生的报错是`error LNK2019: unresolved external symbol "public: virtual __thiscall Animal::~Animal(void)"`，这是链接错误，也就是说编译器调用了`Animal::~Animal(void)`但没有找到这个函数的定义。
	这就要回到前面提到的**类对象作为成员变量**这块内容，在继承中父类`Animal`也类似类对象作为`Cat`的成员变量，析构`Cat`时在`Cat::~Cat()`的最后一部分编译器会在汇编代码中插入插入调用`call Entity::~Entity()`的部分。这个函数没定义啊，所以产生了链接错误。
	**总结：不需要实现一般的纯虚函数是因为不会调用到`Animal::func`，逃不过编译期的检测；但是纯虚析构函数就算我们不主动调用，编译器也会在编译期自主添加`Animal::~Animal`的调用，我们没有实现它的话在链接器就会出问题。**
	PS：编译器会在编译期主动添加`Animal::~Animal`调用的原因是：基类也有可能会在堆区开辟空间，如果不析构子类中的基类就会发送内存泄漏问题。

#### const正确用法

~~~cpp
class Entity {
private:
	std::string m_Name;
	int m_Age;
public:
	explicit Entity(const std::string& name) : m_Name(name), m_Age(-1) {}
	explicit Entity(int age) : m_Name("Unknown"), m_Age(age) {
		Entity *const p = this; // 这个const必须在*右边作为顶层const修饰*
		Entity *const &b = this; // 这个const必须在*右边作为底层const修饰*，不能放在Entity左边或者Entity和*中间。也就是不能放在*左边。因为const优先修饰左边。虽然放在左边语法正确但是与this的类型对不上
	}
};
~~~

#### 智能指针基本原理

当我们在一个作用域内在堆区开辟内存，如果想要在离开作用域后释放堆区就需要时刻记住在作用域的最后几行写上`delete`。但我们总是忘记，所以有了智能指针。

它的基本原理就是，智能指针定义在栈上，作用域结束**自动触发**智能指针的析构函数。所以我们在智能指针的析构函数上写上`delete`。

* 栈上的对象在作用域结束后**自动触发**析构函数，然后释放栈。
* 堆上的对象需要**主动使用**`delete`调用析构函数，然后释放堆。

~~~cpp
class SmartPtr {
private:
	Entity* m_Ptr;
public:
	SmartPtr(Entity* p) : m_Ptr(p) {}
	~SmartPtr() { delete m_Ptr; }
};

int main() {
    {
        SmartPtr ptr = new Entity();
    } // SmartPtr::~SmartPtr(&ptr) -> delete m_Ptr -> Entity::~Entity(m_Ptr)
}
~~~

#### 箭头操作符小技巧

使用箭头操作符获取成员变量在结构体中的偏移量。

~~~cpp
struct Vertex3 {
	int x, y, z;
};
int main() {
	int offset_x = (int) & ((Vertex3*)0)->x;
	int offset_y = (int) & ((Vertex3*)0)->y;
	int offset_z = (int) & ((Vertex3*)0)->z;
}
~~~

简单来说就是对`0`地址的结构体中的成员变量取地址就能得到偏移量。

#### 友元小知识

能作为类的友元的函数一定是类外的函数。也就是说在类中声明为`friend`的函数就会被编译器认为是这个类之外的函数，即使这个函数是在类内定义的。

~~~cpp
class A {
    friend void foo() {
    }
};
~~~

###### 这时 `foo` 的本质是：

- 函数名：`::foo`
- 是否成员：❌ 否
- 是否全局：✅ 是
- 是否有 `this`：❌ 没有
- 是否能访问 `A` 的 `private`：✅ 可以（friend）

调用方式：

```
foo();     // 正确
A::foo();  // ❌ 错误
```

因为`friend`声明的函数被归属在类外，又没有规定`foo()`的域，所以认为它是`::foo`，也就是全局函数了。

#### 迭代器

| 种类           | 功能                                                         | 支持运算                                        |
| -------------- | ------------------------------------------------------------ | ----------------------------------------------- |
| 输入迭代器     | 对数据的只读访问                                             | 只读，支持++、==、!=                            |
| 输出迭代器     | 对数据的只写访问                                             | 只写，支持++、==、!=                            |
| 前向迭代器     | 读写操作，并能向前推进迭代器                                 | 读写，支持++、==、!=                            |
| 双向迭代器     | 读写操作，既能向前推进迭代器也能向后                         | 读写，支持++、--、==、!=                        |
| 随机访问迭代器 | 读写操作，可以以跳跃的方式随机访问任意数据。功能最强大的迭代器 | 读写，支持++、--、[n]、+n、<、<=、>、>=、==、!= |

#### vector

* `resize()`函数`re`的只有`size`，除非扩容不然不会去动`capacity`。
* `vector`拷贝构造函数只会拷贝`size`大小的数据，不会连同`capacity`一起拷贝。开辟的`capacity`与`size`相等。
* `swap()`是完全的交换，包括`size`和`capacity`。
* 利用拷贝构造和交换的原理可以拷贝构造一个匿名对象，在变量`resize()`变小后收缩`capacity`。
	`std::vector<int>(v).swap(v)`。最后达到`v`的`capacity`和`resize()`后的`size`一样大。

| 运算符重载     | 函数接口    |
| -------------- | ----------- |
| T& operator=() | T& assign() |
| T& operator+=  | T& append() |
| E& operator[]  | E& at()     |

#### 仿函数对象对于模板的存在意义

为什么要专门设计一个仿函数类去重载它的`operator()`呢？这与模板的定义有关。

比如说`std::set`在使用时可以指定它的比较器比如`std::set<int, Comp> s`这样的方式去定义一个类，这个`class Comp`是一个仿函数类，为什么不能是`bool Comp(int a, int b)`这样的函数呢？

虽然我们在定义`std::Array<int, 5>`时可以在`<>`内使用非类型的东西，但是这并不代表就能使用`Comp`这个函数名。函数名在编译器看来会降级为函数指针，好像也能变成**地址数字**这样的字面量，但是函数的地址只有在运行时才会确定，在编译期不可能知道，因此函数名在编译器看来就是个变量一样的存在，不确定的东西不可能让编译器去构造出一个确定的类。

但是如果是仿函数类就不一样了，它是一个确定类，编译器可以根据它构造一个确定的模板类。至于在模板类中调用`仿函数::operaotr()`，它就是个成员函数调用，编译器标记成一个符号之后交给链接器就行了。

再仔细一想，其实非要用函数也是可以的，但这需要在`<>`内使用函数降级为函数指针后的类型。比如说定义的比较器为`bool Compare(int a, int b)`，那么使用时就得知道函数指针的类型为`bool (*)(int, int)`，然后像这样定义：`std::set<int, bool(*)(int, int)>`。这样其实也行，只要传入的东西可以使用`()`调用就行。毕竟模板的类型参数是很万能的。但这么一看，麻烦不说还很抽象，如此想来仿函数的出现应该就是为了避免我们出现去寻找函数指针的类型这样的行为。

---

#### auto推导原则

auto只会推导出基本的必要的数据类型（包括指针类型和底层const），是否需要引用类型和const属性由用户自己决定添加

##### 完整的推导规则表格

| 原始类型           | auto 推导    | 想要的效果         | 需要怎么写                                         |
| :----------------- | :----------- | :----------------- | :------------------------------------------------- |
| `int`              | `int`        | `int`              | `auto x = expr;`                                   |
| `int&`             | `int`        | `int&`             | `auto& x = expr;`                                  |
| `const int`        | `int`        | `const int`        | `const auto x = expr;`                             |
| `const int&`       | `int`        | `const int&`       | `const auto& x = expr;`                            |
| `int*`             | `int*`       | `int*`             | `auto x = expr;`                                   |
| `int* const`       | `int*`       | `int* const`       | `auto const x = expr;` (或 `const auto x = expr;`) |
| `const int*`       | `const int*` | `const int*`       | `auto x = expr;` (底层 const 保留)                 |
| `const int* const` | `const int*` | `const int* const` | `const auto x = expr;`                             |

---

#### `const ` 和 `constexpr`

`const`更多起到的是变量类型修饰符的作用，用来修饰某个变量只读；而`constexpr`着重表明变量的初值在编译时由编译器来运算。

也就是说`constexpr`修饰变量是为了让编译器在编译时计算这个变量的初值并赋值



constexpr变量 = 编译时常量 + 只读



对`constexpr`修饰函数的理解就是上面给`constexpr`变量赋值时只能调用`constexpr`函数（常量表达式）

对常量表达式的理解：编译时就能确定值 的 表达式

---

#### 定义变量时，是否初始化为0的编译器行为

大致分为三类：

1. 全局变量
2. 基本数据类型的局部变量
3. 类对象：
	1. 使用编译器提供的默认构造函数（没有无参构造函数）
	2. 有默认（/无参）构造函数

#### 全局变量

对于全局变量（包括静态变量）如果没有初值，都会初始化为0。

#### 基本数据类型

对于基本数据类型，定义一个局部变量时，只要变量名后面不跟括号那么编译器就不会对其进行初始化，其值就是一个垃圾值。
但只要后面**跟着括号**编译器就会将其初始化，即使括号里没有数，也会将其初始化为0。

~~~cpp
int a; // 垃圾值
int b{}; // 初始化为0
int c[5]; // 垃圾值
int d[5] = {} // 全部初始化为0

int* p1 = new int; // 垃圾值
int* p2 = new int(); // 初始化为0
int* p3 = new int[5]; // 垃圾值
int* p4 = new int[5]{}; // 全部初始化为0
~~~

特别注意，在类构造函数的初始化列表中，也会初始化为0

~~~cpp
struct Entity {
	int x, y;
	Entity():x(), y() {} // 全部初始化为0
};
~~~

总结：带括号其实就是告诉编译器多一个`mov 0`操作

---

#### 类对象

* 没有构造函数的结构体

同基本数据类型一样。如果有括号，里面的成员就会初始化（为0）；如果没有括号，成员的值就是垃圾值

~~~cpp
struct Entity {
	int x, y;
};
	Entity e1; // 垃圾值
	Entity e2{}; // 全部初始化为0
	Entity* p1 = new Entity; // 垃圾值
	Entity* p2 = new Entity(); // 全部初始化为0
~~~

总结：带括号其实就是告诉编译器多一个`mov {0, 0}`操作

* 无参构造函数为`default`

就和没有写构造函数一样。如果有括号，就是`mov {0, 0}`；如果没有括号，那就什么都不做。

~~~cpp
class Entity {
private:
	int x, y;
public:
	Entity() = default;
};

	Entity e1; // 垃圾值
	Entity e2{}; // 全部初始化为0
	Entity* p1 = new Entity; // 垃圾值
	Entity* p2 = new Entity(); // 全部初始化为0
~~~

从上面两个来看，都是使用的编译器提供的默认的构造函数。**而编译器提供的默认构造函数本质就是编译器对汇编指令的插入，或者说编译器行为：有括号就插入`mov {0, 0}`，没括号就什么都不做**。

* 带构造函数的类

对于带构造函数的类。如果有括号，那肯定会调用无参构造；但如果没有括号，同样也会调用无参构造

~~~cpp
class Entity {
private:
	int x, y;
public:
	Entity() {} // 不做任何处理，肯定都是垃圾值
};
	Entity e1; // 垃圾值
	Entity e2{}; // 垃圾值
	Entity* p1 = new Entity; // 垃圾值
	Entity* p2 = new Entity(); // 垃圾值
~~~

**特别注意**，在类构造函数的初始化列表中，也会初始化为0

~~~cpp
struct Entity {
	int x, y;
	Entity():x(), y() {} // 全部初始化为0
};
~~~

---

#### std::set

1. C++中`std::set`就是**红黑树**，而红黑树是排序树，排序树就是不允许有重复的数据
2. `std::multiset`就是对`std::set`做了特殊处理，允许有重复数据

#### std::map

1. C++中`std::map`就是**键值对的红黑树**，所以不允许有重复的键
2. `std::multimap`就是对`std::map`做了特殊处理，允许有重复的键

---

#### NVO/RNVO一句话总结

去掉返回类型（改`void`），去掉`return`语句，拿返回的对象指代临时对象`tmp`。

`RNVO`的原则是：返回的对象是函数体内**唯一**确定的**局部对象**。不能有多个`return`，不能是函数参数对象。
