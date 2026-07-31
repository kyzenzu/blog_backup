---
title: C++中的类型转换
date: 2026-07-19 23:25:38
tags:
  - C/C++
---
### 前言

C语言中有强制类型转换，C++也有自己设计的一套类型转换。总结来说，C++的类型转换是对C语言的类型转换更细致的种类划分；

C风格的强制类型转换是C++类型转换的总和，在C++中使用C风格的强制类型转换时，编译器会自己挑选合适类型转换（static_cast、reinterpret_cast、const_cast

注：C++的类型转换从文字上看很像模板函数的调用，比如`static_cast<int>`，但它其实并不是模板函数。它和`new/delete`一样是C++**语法**层面上的**关键字**。所以它的使用不需要`#include <stl>`

### 各转换运算符及详细规则

##### 1.`static_cast`

这应该是**最常用**转换运算符，因为它的使用规则都是平常在C语言中会用的到且非常符合逻辑的（让人一看就很合理的那种。

* ✅ **数值 ↔ 数值**（`int`、`double`、`char` 等之间的转换）
	~~~cpp
	double d = static_cast<double>(5);     // int → double
	int i = static_cast<int>(5.25);        // double → int（截断）
	~~~

* ✅  **父类指针 ↔ 子类指针**（方便多态的使用，合理
	~~~cpp
	Derived d;
	Base* pb = static_cast<Base*>(&d);     // 派生类指针 → 基类指针
	Derived* pd = static_cast<Derived*>(pb); // 基类指针 → 派生类指针
	~~~

* ✅ **void*  ↔  具体类型指针**（多线程传函数参数会经常用到，合理
	~~~cpp
	void* pv = &d;
	Derived* pd2 = static_cast<Derived*>(pv); // void* → 具体类型指针
	~~~

* ✅ **调用用户定义的转换**（构造函数或转换运算符）
	~~~cpp
	class MyInt { public: operator int() { return 42; } };
	MyInt obj;
	int i = static_cast<int>(obj);  // 调用 operator int()
	~~~



##### 2.`reinterpret_cast`—— "暴力的位重解释"

它的名字就很有意思，reinterpret(重新解释)。更多的是用在**无厘头**但实际可能有用的场景。

它的转换原理更多的是在**位**上，也就是说它的转换就类似于暴力的 `mov`，把内存上的 bit 原样移过去然后按照用户的想法重新解释。

所以需要解释的是`reinterpret_cast`不可以做数值 ↔ 数值的转换比如（double ↔ int）。因为它的转换是暴力的 `mov`，首先字节数就不对，其次编码格式不一样，double 是 `IEEE`浮点数格式，直接暴力的 `mov`得到的数据完全没有意义。所以（double ↔ int）就需要用 `static_cast` 解析玩浮点数格式后做数值上的转换。

* ✅ **指针 ↔ 指针**（任意类型，完全不相关也行）
	~~~cpp
	double* pd;
	int* pi = reinterpret_cast<int*>(pd);  // ✅ 强行把 double 指针当 int 指针用
	~~~

	~~~cpp
	class Entity {};
	
	class Player : public Entity{};
	class Enemy : public Entity{};
	
	int main() {
		Player* player = new Player;
		Entity* entity = static_cast<Entity*>(player); // 有厘头的转换
		Enemy* enemy = reinterpret_cast<Enemy*>(player); // 无厘头的转换,只能用reinterpret
		return 0;
	}
	~~~

* ✅ **指针 ↔ 整数**
	~~~cpp
	int* p = &i;
	int addr = reinterpret_cast<int>(p);  // ✅ 指针↔整数（保存地址）
	~~~

* ✅ **数值类型 → 引用**（任意类型）
	~~~cpp
	double d = 5.25;
	int& ri = reinterpret_cast<int&>(d);  // ✅ 强行把 double 的 8 字节当 int 引用
	~~~



##### 3.`const_cast` —— "唯一的去 const 工具"

* ✅ **添加或去除 `const`/`volatile` 限定符**（指针或引用）
	~~~cpp
	const int a = 10;
	const int* pca = &a;
	int* pa = const_cast<int*>(pca);  // ✅ 去除 const
	
	const int& ra = a;
	int& r = const_cast<int&>(ra);    // ✅ 去除 const
	~~~

这里乍一看，如果我用`pa`对`a`进行了修改怎么办。事实上，语法不会禁止这种操作，运行后`a`的数据在内存中也确实会被修改，但是在编译时编译器会把所有`a`用到的地方统统换成了`10`。所以不会有太大影响



##### 4.`dynamic_cast` ——"安全的运行时转换"

这个转换的**目的**只有一个：让子类指针和父类指针能够**安全**的转换。

能否成功转换的**本质**：被转换方的值能否成功赋值到目标上。`dynamic_cast` 做的就是检测它们是否真的是父子关系，如果是则赋值成功，如果不是则赋值为`nullptr`。

~~~cpp
class Entity {
	virtual void func() {};
};

class Player : public Entity {};
class Enemy : public Entity {};

int main() {
	Entity* entity = new Player;
	Player* player = dynamic_cast<Player*>(entity); // player ⬅ entity,转换成功
	Enemy* enemy = dynamic_cast<Enemy*>(entity); // enemy == nullptr,转换失败
	return 0;
}
~~~

如何**安全**的实现转换，重点就在这个**动态**上。编译器在识别到`dynamic_cast`会像`new`一样展开为一段指令，这段指令就用于实现检测`entity`基类指针指向的对象是否就是`enemy`子类对象，如果是则正常赋值，如果不是则赋值为`nullptr`。

~~~assembly
Enemy* enemy = dynamic_cast<Enemy*>(entity):
        mov     eax, DWORD PTR [entity]
        test    eax, eax
        je      .L5
        push    0
        push    OFFSET FLAT:typeinfo for Enemy
        push    OFFSET FLAT:typeinfo for Entity
        push    eax
        call    __dynamic_cast
        add     esp, 16
        jmp     .L6
.L5:
        mov     eax, 0
.L6:
        mov     DWORD PTR [enemy], eax
        mov     eax, 0

vtable for Player:
		.long   0
        .long   typeinfo for Player
        .long   Entity::func()
        
vtable for Entity:
        .long   0
        .long   typeinfo for Entity
        .long   Entity::func()
        
typeinfo for Enemy:
        .long   _ZTVN10__cxxabiv120__si_class_type_infoE+8
        .long   typeinfo name for Enemy
        .long   typeinfo for Entity
typeinfo name for Enemy:
        .string "5Enemy"
        
typeinfo for Player:
        .long   _ZTVN10__cxxabiv120__si_class_type_infoE+8
        .long   typeinfo name for Player
        .long   typeinfo for Entity
typeinfo name for Player:
        .string "6Player"
        
typeinfo for Entity:
        .long   _ZTVN10__cxxabiv117__class_type_infoE+8
        .long   typeinfo name for Entity
typeinfo name for Entity:
        .string "6Entity"
~~~

可以简单看出`__dynamic_cast`做的就是通过对比`typeinfo`来实现运行运行时检测。

这也解释了为什么使用`dynamic_cast`要启用`RTTI(run-time type info)`。并且父类需要实现虚函数，因为有了虚函数才会有虚函数表，然后对象里才会有存储`typeinfo`。

##### `dynamic_cast` 小技巧

基于`__dynamic_cast`返回值的原理，`dynamic_cast`还有一个使用小技巧实现`Java`那样的运行时类型判断。

~~~cpp
Player* p0 = dynamic_cast<Player*>(entity);
if (p0) {
	/* 如果entity真的是Player类 */
} else {
	/* 如果不是 */
}
~~~

用于实现Java中：
~~~java
if (entity instanceof Player) {
    
}
~~~



