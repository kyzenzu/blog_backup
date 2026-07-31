---
title: C++中的引用和移动语义
date: 2024-9-8
tags: 
  - C++
  - C/C++
sticky: 100
---

**引用**是C++基于C语言新提出的一个概念，很多教科书或者网上都说引用是变量的一个别名。但是，这种说法未免有些太玄乎了，这样引出了很多问题，引用本身是一个变量吗，它会占据内存空间吗等等，这些问题使得引用变得更加玄乎了。

为了深入了解引用，我决定在汇编中寻找答案，毕竟汇编才是C/C++的根本。

话不多说，直接给出例子。

~~~C++
int main()
{
    int x = 0;
    int* p = &x;
    *p = 5;
    int& ref = x;
    ref = 10;
    return 0;
}
~~~

~~~assembly
main:
        push    rbp
        mov     rbp, rsp
        
        ; int x = 0
        mov     DWORD PTR [rbp-20], 0 ; [rbp-20]为x
        
        ; int* p = &x
        lea     rax, [rbp-20]
        mov     QWORD PTR [rbp-8], rax ; [rbp-8]为p
        ; *p = 5
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax], 5
        
        ; int& ref = x
        lea     rax, [rbp-20]
        mov     QWORD PTR [rbp-16], rax ; [rbp-16]为ref
        ; ref = 10
        mov     rax, QWORD PTR [rbp-16]
        mov     DWORD PTR [rax], 10
        
        ; return 0
        mov     eax, 0
        
        pop     rbp
        ret
~~~

在这个例子中我先定义了一个指针`p`指向变量`x`，解引用该指针去修改`x`的值；然后定义了一个引用`ref`，用引用取修改`x`的值。就做了两套相同的事情，定义然后修改。如果看得懂一些汇编，就能够发现指针和引用的定义以及用法竟然是一样一样的。于是我们得出了一个结论：**引用的本质就是指针**。

为了更好的理解，写一个等效`C++`代码吧

~~~C++
int x;
int& ref = x; // int* const ref = &x
ref = 5; // *ref = 5
~~~

定义一个引用实际上就是在定义一个指针，这样写的好处就是能够省略很多**声明符**。注意到，为什么等效代码会有`const`呢，其实在汇编代码中`const`的性质并不会体现出来，它是给编译器看的，仅仅通过汇编我们很难反推出等效的C++代码到底有没有`const`。但是，想想我们在C++中是如何使用引用的，是不是直接用引用名`ref`，这样其实就是类似于`*ref`，并不是修改`ref`本身的内存空间，而是修改`ref`引用的那个内存空间，也就是变量`x`。通过C++代码是很难去动`ref`内存里的值，也就很难改动`ref`的指向，这样不也侧面说明了这个`const`了吗。

以上就是传统意义上引用的本质，在其它语言上也可以这么解释。

但是，在C++中，为了配合左值和右值，将引用分为了**左值引用**和**右值引用**。其中左值引用就是上面讲的，那么右值引用又是什么东西呢？

为了更好的解释这两个引用，需要知道什么是左值，什么是右值。

网上说可以取地址的对象就是左值，反之则是右值。说的很有道理，但是我认为还没有说到点上，因为严格来说不管怎么样我们写的任何代码任何数据最终都是在内存里的，凭什么说有的能取址，有的不能取地址。你说`int x = 10`中这个`10`是右值因为它不能取地址，真要说起来，（以下是示例）这条代码转为汇编指令不就是` mov DWORD PTR [rbp-0x4],0xa`吗，对应机器指令就是`c7 45 fc 0a 00 00 00`，将其存储在内存里，我同样能对`0a 00 00 00`这4个字节进行寻址和取址。所以用这种方式定义左值和右值难免有些矛盾。

所以这是我对左值和右值做出一个笼统且感性的概念：

* 左值：一个对象，或者说一片内存空间，只不过这片内存空间可以让程序员能够在C++代码中用一个明确的**标识符**表示出来，基本上这片可标识的内存空间除了不在`.txt`代码段其它段都可以存在
* 右值：同样是一个对象，一片内存空间，但是这片内存空间程序员不能够用一个明确的**标识符**在C++代码表示出来，程序员自然也就不能用C++操作这片空间。这片内存空间可能存在于`.txt`代码段、栈空间甚至是寄存器中。比如说一个常量`10`、一个临时对象，我们都不能在C++代码中用一个标识符来表示这块空间吧。

请注意，上面说的**标识符**都是从程序员的角度，基于C++代码层面上定义。并不是编译器基于汇编代码层面的，对于编译器来说不管是左值还是右值对象的内存空间位置都是了如指掌的，是全知全能的，所以我们不应该看编译器来定义左值右值。

有了这个概念后让我们来看一下一个简单的右值引用

~~~C++
int&& ref = 10;
ref = 5;
~~~

右值引用从语法上只是比左值引用多了一个`&`符号，其引用的对象是一个右值。但是，前面说过，引用本质上是个指针，现在这个指针指向`10`，而这个`10`其实是存在于代码区的，难道我们可以操控代码区了？这无疑于C++往自己身上绑了颗定时炸弹，C++的设计者当然不会允许这种事情发生。这背后到底发生了什么，看看汇编就知道了

~~~assembly
; int&& ref = 10
mov     DWORD PTR [rbp-12], 10
lea     rax, [rbp-12]
mov     QWORD PTR [rbp-8], rax

; ref = 5
mov     rax, QWORD PTR [rbp-8]
mov     DWORD PTR [rax], 5
~~~

 不难发现，这套逻辑与前面的左值引用可以说非常相似，甚至是一模一样。就是最开始的一行多了一段`mov     DWORD PTR [rbp-12], 10`，这是为什么？前面说了，原来的`10`存与代码区，是万万不能动的，为了能够在C++中正常使用这个引用，编译器特地把`10`移到了栈当中，即`[rbp-12]`，然后令`ref`指向这个地址，这个引用的定义也就完成了。从这里不难看出，右值引用的本质其实与左值引用别无二样，只有在最开始选择引用的对象不同的罢了（一个是左值，一个是右值）。

值得注意的是，这个右值引用在被定义后，被引用的对象其实是个左值，也就是说引用变量`ref`被定义后`*ref`（等效代码，实际是正常使用就是`ref=5`这样的）是一个左值。为什么？结合前面左值右值的定义，现在`ref`指向的那个对象，那片内存空间，在C++代码中已经可以用`ref`来表示了，程序员可以用一个特定的标识符来操作那片空间，所以此后`ref`指向的对象就是一个左值。

什么？你说右值引用不应该引用的的是一个右值吗？是的，右值引用在最开始**想要**引用一个右值，但是受现实所迫，它在最后又去引用了一个左值。

右值，一开始基本上被认为是一个常量，我们不应该去修改它，比如说`10 = 5;`这样的代码就很诡异吧。所以在`C++11`提出右值引用之前，都是用被`const`修饰的左值引用去引用右值

~~~C++
const int& ref = 10;
ref = 5; // error
~~~

汇编代码：

~~~assembly
mov     DWORD PTR [rbp-12], 10
lea     rax, [rbp-12]
mov     QWORD PTR [rbp-8], rax
~~~

与前面右值引用时完全一模一样的。不同的是我们不能在C++代码中去修改它了。

那么，右值引用这个新概念的提出又有什么用呢？在接下来 **移动语义（move semantics）**中它会有大作用。

在这之前，先聊聊平常又简单的概念：**强制类型转换**

我们知道在等号赋值时，等号右边表达式结果的类型必须与等号左边表达式的类型相同才能赋值，否则在`Visual Studio 2022`中它就会报以下错：`"某某"类型的值不能用于初始化"某某"类型的实体`

比如说下面这样是不行的

~~~C++
int x = 3;
int* p = &x;
int y = p; // error: "int *"类型的值不能用于初始化"int"类型的实体
~~~

但是我们改成这样就可以了

~~~C++
int x = 3;
int* p = &x;
int y = (int)p; // 做一个强制类型转换
~~~

这里的转换我是在`x86`的环境下做的，指针的大小与`int`类型相同都是4个字节。如果在`x64`环境下注意把`int`改为`long long`，否则会报丢失精度的错误。

看下汇编

~~~assembly
;int x = 3
005B185F  mov         dword ptr [ebp-0Ch],3  

;int* p = &x
005B1866  lea         eax,[ebp-0Ch]  
005B1869  mov         dword ptr [ebp-18h],eax  

;int y = (int)p
005B186C  mov         eax,dword ptr [ebp-18h]  
005B186F  mov         dword ptr [ebp-24h],eax
~~~

一般情况下的强制类型转换无非也**就是把改变一个数据的解读方式，数据的值依然视不变（不考虑长类型变短类型的精度丢失）**，至少在C语言中都是这样的

你可能会说`int x = 3.14;`我这样做也是正确的，那是因为编译器帮你做了一个**隐式类型转换**，它实际上是`int x = (int)3.14;`至少从结果上来说都是`mov DWORD PTR [rbp-4], 3`，把小数点后面的数都砍掉了

也许你已经注意到了，难道我们上面写的左值引用和右值引用也是**隐式类型转换**吗？是的，没错

~~~C++
int x = 5;
int& ref1 = x;
int& ref2 = (int&)x;
~~~

~~~assembly
mov     DWORD PTR [rbp-20], 5

lea     rax, [rbp-20]
mov     QWORD PTR [rbp-8], rax

lea     rax, [rbp-20]
mov     QWORD PTR [rbp-16], rax
~~~

~~~C++
int&& ref1 = 10;
int&& ref2 = (int&&)10;
~~~

~~~assembly
mov     DWORD PTR [rbp-20], 10
lea     rax, [rbp-20]
mov     QWORD PTR [rbp-8], rax

mov     DWORD PTR [rbp-24], 10
lea     rax, [rbp-24]
mov     QWORD PTR [rbp-16], rax
~~~

你会发现，不管我有没有显式写出强制类型转换，它们最后的汇编代码都是一样的。结合上面所说的，编译会做出适当的隐式类型转换

但是又出现了一个问题，前面的强制类型转换都是数据数值的迁移，比如说`int y = (int)p `至少从数值上来说它们是相等的，这里引用的类型转换竟然并不是数据的迁移，**而是将目标的地址迁移了过去**，也就是说从`mov`变成了`lea`。

这是为什么呢？从C++设计者的角度来看，他就是为了实现`int& ref = x`这个简单的语法，实现用`ref`就像在用`x`本身一样，而`ref`本质上又是个指针，到时候操作`ref`的时候会自动的解引用`ref`内存中存储的地址。如果，这里在定义`ref`时，`x`的类型转换还是像C语言中那样简单的把`x`的数值迁移过去，那`ref`指向`x`的意义就完全没有了，`ref`就像个野指针一样，那么引用概念的提出也就完全没有意义了。由此，只能去改变编译器的行为了。新概念的提出需要做出新的规则，这也能理解，不过这也是C++变得如此臃肿的原因之一。

说了这么多，到底移动语义是什么呢，且看下面的代码

~~~C++
int x = 10;
int&& ref = (int&&)x;
~~~

像这样就是一个移动语义了，将**一个左值强制转换为一个右值引用(指针)**这就是**移动语义**，怎么样是不是很简单。只不过为了使这样的语法更具有规范性，可以使其适用于任何类型，比如所有的基本数据类型和自定义类。C++提供了一个模板函数`std::move()`，于是上面的写法就可以改写成这样

~~~C++
int x = 10;
int&& ref = std::move(x);
~~~

这个函数就会返回一个强制类型转换`x`为右值引用的变量（存储了`x`的地址，是个右值），类似于这样吧（我猜的，有错误请指正

~~~C++
int&& move(int& x) // move的作用仅仅只是做一步强制类型转换
{
    return (int&&)x;
}

int main()
{
    int x = 4;
    int&& ref = move(x);
    return 0;
}
~~~

~~~assembly
move(int&):
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8] ; 返回 x 的地址
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-12], 4
        lea     rax, [rbp-12]
        mov     rdi, rax ; 传入 x 的地址
        call    move(int&)
        mov     QWORD PTR [rbp-8], rax ; x 的地址赋值给了ref
        mov     eax, 0
        leave
        ret
~~~

移动语义到底有什么用呢，我们说右值引用都是**想要**来引用右值的，在这里用右值引用去引用左值，其实就是将这个左值**标记**为一个右值，所谓标记，就是从`ref`来看`x`是一个右值，实际上`x`仍然是一个左值，这是为了告诉程序员将这个左值看作一个右值来对待，今后这块内存区域不会再用到了。

想想平常对右值的定义，是不是一个临时的对象，一个用完就丢的一次性用品，但是将这个一次性用品直接就丢了会不会太可惜了，这是对资源的浪费，所以**我们要将这个一次性用品掏空并利用好其中的资源然后再把它丢弃**，这就是`移动语义`的真正**内涵**。

用一个整形的变量解释还是太难了，下面还是用一个类去解释会更清晰

 ~~~C++
 class String
 {
 public:
 	String() = default;
     
 	String(const char* string) {
 		printf("String Created\n");
 		m_Size = strlen(string);
 		m_Data = new char[m_Size];
 		memcpy(m_Data, string, m_Size);
 	}
     
 	String(const String& other) {
 		printf("String Copied\n");
 		m_Size = other.m_Size;
 		m_Data = new char[m_Size];
 		memcpy(m_Data, other.m_Data, m_Size);
 	}
 
 	~String() {
 		printf("String Destroyed\n");
 		delete m_Data;
 	}
 private:
 	char* m_Data;
 	uint32_t m_Size;
 };
 
 class Entity
 {
 public:
 	Entity() = default;
 	Entity(const String& name) : m_Name(name) {}
 private:
 	String m_Name;
 };
 ~~~

这里我定义了一个`String`类和一个`Entity`类，一个Entity有一个名字很正常吧。

`String`类里我只定义了一个有参构造和一个拷贝构造，功能还没完善，我会细细说明

然后写了测试函数

~~~C++
int main()
{
	Entity e("Hello World");
	std::cin.get();
}
~~~

很简单，用字符串为名字定义了一个`Entity`对象

控制台输出如下：

~~~shell
String Created
String Copied
String Destroyed
~~~

有点乱，先说明一下，本来`Entity`类中没有关于字符串字面量的构造函数，但是它有关于`String`类的构造函数`Entity(const String& name)`，于是编译器做了一步隐式转换`(String)"Hello World"`，这样的出来的`String`对象是临时对象，姑且称之为`tmp`，它是一个右值，用`const String&`引用一个右值，前面说了，也没问题，但是要注意，这个`tmp`被引用了之后就变成了左值。然后写一下这其中的调用链

~~~C++
"Hello World" -> (String)"Hello World" -> tmp.String("Hello World") -> tmp ->
e.Entity(tmp) -> m_Name.String(tmp) -> e -> tmp.~String()
~~~

控制台输出的内容分别在`tmp.String("Hello World")`、`m_Name.String(tmp)`和`tmp.~String()`中输出

乍一看，这好像没什么问题。如果真没问题那么这篇文章我也白写了。

我写这段代码的目的是什么，是不是用`"Hello World"`创建一个`String`类（重点是在堆区分配这个字符串数组），然后再用这个`String`对象初始化`e.m_Name`，就需要`e.m_Name`去拷贝这个临时对象，这个过程又涉及到了一次堆区的内存分配，仔细想想，我在堆区分配了两次内存，而第一次分配的内存又要马上释放掉，这真的合理吗，很明显这是资源的浪费。

第一次创建的临时对象就好比前面提到的一次性用品，仅仅只是为了拷贝就把它丢了未免太可惜了。为什么我们不直接把它在堆区创建的内存拿来用呢

接下来，让我们写**移动构造函数**

~~~C++
String(String&& other) { 
	printf("String Moved\n");
	m_Size = other.m_Size;
	m_Data = other.m_Data;
}
~~~

与拷贝构造函数的定义很像，不过对`other`对象的操作方式有些不一样。

不同的是，它并没像拷贝构造函数那样在堆区分配一片空间，取而代之的是直接将`this->m_Data`指向了由`other`申请分配的那片堆区的内存空间。只是这样还没完，因为`other`已经被视为一个右值，或者本身就是个右值，它一会可能是要被销毁的，那么它指向的堆区也会被跟着释放，这样的话新构造出来的对象不就指向了一片无效区吗，这不行，还得再改改

~~~C++
String(String&& other) { 
	printf("String Moved\n");
	m_Size = other.m_Size;
	m_Data = other.m_Data;
    
    other.m_Size = 0;
    other.m_Data = nullptr;
}
~~~

很简单，将`other`的成员变量改一下就好了，其中`other.m_Size`的重置无所谓，反正后面也不会用到，重点是将`other.m_Data`置为`nullptr`，因为`String`类的析构函数`~String()`会涉及到`delete m_Data`，而`delete nullptr`这个操作既能避免原先分配的那片空间被释放，又不会引发错误，可谓一举两得。

> 这是因为 `delete` 操作符实际上并不执行任何动作来检查或释放空指针。它仅用于释放已分配的内存。当你传递一个空指针给 `delete` 时，它会简单地忽略这个操作，并且不会引发运行时错误或异常。

到这里，可以说新构造出来的对象不仅将`other`的数据全部窃取了过来，还把`other`扒得连裤衩子都不剩。这就是**移动**得核心思想。

将这个移动构造函数放在`String`类中，然后在`Entity`类中添加合适的构造函数

~~~C++
Entity(String&& name) : m_Name(name) {}
~~~

还是同样的测试代码，让我们看看结果怎么样

~~~shell
String Created
String Copied
String Destroyed
~~~

纳尼？怎么还是用拷贝构造函数，别急仔细分析一下

`(String)"Hello World"`构造出一个`String`类的临时对象`tmp`，它是个右值不假，也成功调用了`e.Entity(String&& name)`没错，但是重点在后面的列表初始化`m_Name(name)`

在这个`tmp`没有被`name`引用前是个右值没错，但是被`name`引用了之后它就变成了左值呀，因为可以用`name`标识符在让程序员在C++中表示`tmp`的内存空间了呀。所以编译器在构造`e.m_Name`时选择利用传入参数是左值和右值都可以的拷贝构造函数（反正不可能选择传入参数是右值的移动构造函数），怎么办呢

强制类型转换呀，虽然引用类型`name`本质是一个指针，但是它在后续的使用中都会自动解引用，解引用它指向的的那个对象，那片空间，是个左值。把它强制类型转换为右值引用就行了嘛。于是把`Entity`的构造函数修改一下

~~~C++
Entity(String&& name) : m_Name((Entity&&)name) {}
~~~

或者选择更优雅的方式，用`std::move()`

~~~C++
Entity(String&& name) : m_Name(std::move(name)) {}
~~~

`String`类和`Entity`类就变成了如下这样，为了便于之后的演示，我还在两个类中各自添加`Print()`函数。

~~~C++
class String
{
public:
	String() = default;
	String(const char* string) {
		printf("String Created\n");
		m_Size = strlen(string);
		m_Data = new char[m_Size];
		memcpy(m_Data, string, m_Size);
	}
	String(const String& other) {
		printf("String Copied\n");
		m_Size = other.m_Size;
		m_Data = new char[m_Size];
		memcpy(m_Data, other.m_Data, m_Size);
	}
	String(String&& other) { 
		printf("String Moved\n");
		m_Size = other.m_Size;
		m_Data = other.m_Data;
		
		other.m_Size = 0;
		other.m_Data = nullptr;
	}
	~String() {
		printf("String Destroyed\n");
		delete m_Data;
	}
	void Print() {
		if (m_Data == nullptr) {
			printf("NULL\n");
			return;
		}
		for (uint32_t i = 0; i < m_Size; i++)
			printf("%c", m_Data[i]);
		printf("\n");
	}
private:
	char* m_Data;
	uint32_t m_Size;
};

class Entity
{
public:
	Entity() = default;
	Entity(const String& name) : m_Name(name) {}
	Entity(String&& name) : m_Name(std::move(name)) {}
	void PrintName() {
		m_Name.Print();
	}
private:
	String m_Name;
};

int main()
{
	Entity e("Hello World");
    e.PrintName();
	std::cin.get();
}
~~~

再来看看结果怎么样

~~~shell
String Created
String Moved
String Destroyed
Hello World
~~~

完美！这样就只在堆区分配了一次内存空间，而且`tmp`被销毁了也不会对`e`产生任何影响

到这里其实已经讲的已经差不多了，但是还有一个**移动赋值函数**没讲，这并没有什么什么稀奇的，就像**移动构造函数**是对**拷贝构造函数**的重载一样，**移动赋值函数**同样也是对**拷贝赋值函数**也就是`ClassType& operator=(const ClassType& other)`的重载

为了方便直接拿`String`类做演示吧

~~~C++
class String
{
public:
	String() = default;
	String(const char* string) {
		printf("String Created\n");
		m_Size = strlen(string);
		m_Data = new char[m_Size];
		memcpy(m_Data, string, m_Size);
	}
	String(const String& other) {
		printf("String Copied\n");
		m_Size = other.m_Size;
		m_Data = new char[m_Size];
		memcpy(m_Data, other.m_Data, m_Size);
	}
	String(String&& other) { 
		printf("String Moved\n");
		m_Size = other.m_Size;
		m_Data = other.m_Data;
		
		other.m_Size = 0;
		other.m_Data = nullptr;
	}
	~String() {
		printf("String Destroyed\n");
		delete m_Data;
	}
	String& operator=(String&& other) {
		printf("String Moved\n");
		delete[] m_Data;

		m_Size = other.m_Size;
		m_Data = other.m_Data;

		other.m_Size = 0;
		other.m_Data = nullptr;
        
		return *this;
	}
	void Print() {
		if (m_Data == nullptr) {
			printf("NULL\n");
			return;
		}
		for (uint32_t i = 0; i < m_Size; i++)
			printf("%c", m_Data[i]);
		printf("\n");
	}
private:
	char* m_Data;
	uint32_t m_Size;
};

int main()
{
	String apple = "apple";
	String fruit;

	std::cout << "apple: ";
	apple.Print();
	std::cout << "fruit: ";
	fruit.Print();

	fruit = std::move(apple);

	std::cout << "apple: ";
	apple.Print();
	std::cout << "fruit: ";
	fruit.Print();

	std::cin.get();
}
~~~

如果我已经构造出了一个左值对象`apple`，它有在堆区分配一片空间。同时我有一个没有分配好堆区空间的左值对象`fruit`。这时我想要`fruit`也有片空间但是不想去申请分配空间怎么办？去窃取`apple`的喽。用`std::move()`返回`apple`强制类型转换后的右值引用（`(String&&)apple`），将左值`apple`标记为右值（其实还是个左值），然后 `=` 符号就会自动调用 `String& operator=(String&& other)`，将`apple`的数据完全移动到`fruit`，之后`apple`里面就什么都没有了。

看一下结果

~~~shell
String Created
apple: apple
fruit: NULL
String Moved
apple: NULL
fruit: apple
~~~

很完美，`fruit`成功指向了`"apple"`

但是，这个移动赋值函数还有点小瑕疵，如果我们做出了`apple = apple;`这样的行为，就会造成`apple`的自杀，这样就不符合原本的意思了，所以还需要做一个条件判断，改成下面这样

~~~C++
String& operator=(String&& other) {
	printf("String Moved\n");
	if (this != &other) {
		delete[] m_Data;

		m_Size = other.m_Size;
		m_Data = other.m_Data;

		other.m_Size = 0;
		other.m_Data = nullptr;
	}
	return *this;
}
~~~

就可以啦！

1. 左值不能**隐式类型转化**为右值引用，所以
	~~~C++
	int x = 10;
	int&& ref = x; // 不成立
	int&& ref = (int&&)x // 成立
	~~~

	但是左值是可以**隐式类型转化**为左值引用的

2. 右值引用本身是左值的情况下，其被使用时自动解引用为其引用的对象
	右值引用本身是右值的情况下，其被使用于右值定义时就像指针赋值一样，其它情况下会自动解引用为其引用的对象

	~~~C++
	int&& move(int& x) {
	    return (int&&)x;
	}
	int main() {
	    int x = 3;
	    int&& ref = move(x); // move(x)是右值，类型是右值引用
	                         // ref的内存中会直接存储x的地址，这是直接定义时赋值的情况下
	    int&& ref1 = move(x) + 1; // move(x)是右值，类型是右值引用
	                              // 但是会接引用为x，因为要用到'+'上
	                              // 最后ref1会引用到4上
	}
	~~~

