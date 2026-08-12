## 异常声明（异常规范，Exception Specifications）

- 可以在函数的**声明**中列出该函数可能抛掷的所有异常类型。
  - 示例：
    ```cpp
    void fun() throw(A, B, C, D);
    ```

- 若**没有异常接口声明**，则该函数可以抛掷**任何类型**的异常。

- 若函数**不抛掷任何类型**的异常，声明方式如下：
  ```cpp
  void fun() throw();
  ```
  或：
  ```cpp
  void fun() noexcept;
  ```

---

## noexcept 异常说明

- 对**明确不会抛出异常**的函数使用 `noexcept` 说明符修饰。
- 声明方式：
  ```cpp
  返回值类型 函数名(形参列表) noexcept;
  ```
- 异常处理机制在编译和运行时会产生额外开销，使用 `noexcept` 可**省去异常处理**，从而优化并加速函数调用。
- 需要保持**一致性**：该函数内部所调用的函数及其定义语句均不得抛出异常。
- 配套有 **`noexcept` 运算符**，可用于判断一个函数是否带有 `noexcept` 说明。

##### 示例：
```cpp
void f() noexcept;
noexcept(f());   // 返回 true，因为 f 有 noexcept 说明
```

---

## 慎用异常声明的情况

在以下情形中，应**谨慎使用或避免使用**异常声明（`exception specifications`）：

1. **函数模板（Function Templates）**
   - 对于带有**类型参数**的函数模板，应尽量避免使用异常规范。
   - 原因：不同类型对同一操作的实现可能不同，抛出的异常类型也可能不同。
   - 因此，模板很难或**几乎不可能**准确预知其具体实例化后可能抛出的所有异常类型。

2. **回调函数（Callback Functions）**
   - 回调函数的行为通常由调用方决定，编译器难以静态分析其异常抛出情况。
   - 使用异常规范可能导致误判或运行时冲突。

3. **系统可能抛出的异常**
   - 系统级操作（如内存分配、I/O 等）可能抛出无法预知或平台相关的异常。
   - 在这些场景中，使用异常规范可能带来风险或不必要的限制

---

## 自动的分析（异常处理流程）

当找到一个匹配的 `catch` 异常处理后，系统会依次执行以下操作：

1. **初始化异常参数**
   - 将抛出的异常对象传递给 `catch` 块的参数（按值、引用或指针方式）。

2. **自动析构对象（栈展开，Stack Unwinding）**
   - 将从对应的 `try` 块开始到异常被抛掷处之间**构造但尚未析构**的所有局部自动对象（栈上对象）进行析构。
   - 析构顺序与构造顺序相反。

3. **恢复执行**
   - 从最后一个匹配的 `catch` 处理块之后开始继续执行程序。

---

## C++ 标准库异常类汇总

| 异常类              | 头文件        | 异常的含义                                                   |
| ------------------- | ------------- | ------------------------------------------------------------ |
| `bad_alloc`         | `<exception>` | 使用 `new` 动态分配内存失败                                  |
| `bad_cast`          | `<typeinfo>`  | 执行 `dynamic_cast` 失败                                     |
| `bad_typeid`        | `<typeinfo>`  | 对空指针执行 `typeid(*p)` 操作                               |
| `bad_exception`     | `<exception>` | 当函数的异常声明不允许某异常，而 `unexpected()` 又抛出了同样不被允许的异常，且该函数的异常声明列表中含有 `bad_exception` 时，会在调用点抛出此异常 |
| `ios_base::failure` | `<ios>`       | C++ 输入输出流操作过程中发生错误                             |
| `underflow_error`   | `<stdexcept>` | 算术运算时发生向下溢出                                       |
| `overflow_error`    | `<stdexcept>` | 算术运算时发生向上溢出                                       |
| `range_error`       | `<stdexcept>` | 内部计算时发生作用域（范围）错误                             |
| `out_of_range`      | `<stdexcept>` | 参数值不在允许的范围之内                                     |
| `length_error`      | `<stdexcept>` | 尝试创建长度超过最大允许值的对象                             |
| `invalid_argument`  | `<stdexcept>` | 向函数传入了无效参数                                         |
| `domain_error`      | `<stdexcept>` | 执行程序所需的先决条件不满足                                 |

#### 补充说明

- `<exception>`、`<new>`、`<typeinfo>`、`<ios>` 中的异常类属于**核心语言支持**类。
- `<stdexcept>` 中的异常类属于**标准逻辑/运行时错误**类，常用于应用程序层。
- 这些异常类均继承自标准基类 `std::exception`，可通过 `what()` 成员函数获取错误描述字符串。

---

## 标准异常类的基础

- **`exception`**  
  - C++ 标准程序库中所有异常类的**公共基类**。

- **`logic_error`**（逻辑错误）  
  - 表示**可以在程序中被预先检测到**的异常。  
  - 如果小心地编写程序（如检查参数合法性），这类异常是**可以避免**的。

- **`runtime_error`**（运行时错误）  
  - 表示**难以被预先检测**的异常。  
  - 这类异常通常与外部环境、系统状态或数据有关，无法在编码阶段完全避免。

---

### 继承层次示意
```
std::exception
├── std::logic_error
│   ├── std::domain_error
│   ├── std::invalid_argument
│   ├── std::length_error
│   └── std::out_of_range
└── std::runtime_error
    ├── std::range_error
    ├── std::overflow_error
    └── std::underflow_error
```

---

下面以这段代码为例讲解 `throw` 的底层原理。

有些类的用途提前声明：`MyException`用于抛出的异常类、`Demo`用于检测异常处理代码前对象是否会正确析构的类

先说一下整体的流程：

1. 异常对象被抛出后，执行流会先在堆区创建一个异常对象
2. 然后依次析构从 `try` 到 `throw` 之间创建的对象。顺序是后构造的先析构。此外要注意的是由普通指针和引用创建的堆区对象，只会调用`operator delete()`释放空间，不会调用对象的析构函数。
3. 中间的对象析构完后，根据异常对象的类型，选择 `catch` 块进入执行
4. 然后继续正常执行 `try-catch` 之后的代码

下面结合汇编讲讲底层原理

### throw的起点

```cpp
int func() 
{
	Demo d1;
    Demo d2; 
    Demo& d = *(new Demo);
    int x = 1;
    if (x == 2)
	    throw MyExcetption("func() 抛出异常");
    return 10;
}
```

```assembly
		sub     esp, 12
        push    sizeof(MyExcetption)
        call    __cxa_allocate_exception
        add     esp, 16
        mov     ebx, eax
        sub     esp, 8
        push    OFFSET FLAT:.LC2
        push    ebx
        call    MyExcetption::MyExcetption(char const*)
        add     esp, 16
        sub     esp, 4
        push    OFFSET FLAT:MyExcetption::~MyExcetption()
        push    OFFSET FLAT:typeinfo for MyExcetption
        push    ebx
        call    __cxa_throw
```

C++中的`throw`展开后的汇编指令就是这样。其中的函数调用顺序：

1. 先调用 `__cxa_allocate_exception` 函数，传入异常对象的大小然后在堆区申请相应大小的内存，返回申请的地址
2. 然后在申请的内存上调用异常对象 `MyException` 的构造函数
3. 然后会调用 `__cxa_throw` 函数，传入的参数有
	- 异常对象的析构函数地址
	- 异常对象的 `typeinfo` 地址
	- 异常对象在堆区的地址

那么 `__cxa_throw`  里面在做什么呢

```cpp
void __cxa_throw(
    void* obj,
    type_info* t,
    void(*dest)(void*)
)
{

    // 找到异常头
    __cxa_exception* header = getHeader(obj);


    header->exceptionType = t;

    header->exceptionDestructor = dest;


    header->handlerCount = 0;


    // 设置当前异常
    __cxa_get_globals()->caughtExceptions = header;


    // 调用libgcc展开器

    _Unwind_RaiseException(
          &header->unwindException
    );


    // 如果返回说明没人catch

    terminate();

}
```

其中最重要的地方就是 `_Unwind_RaiseException` 函数的调用，这个函数是整个C++异常处理逻辑的核心函数

### _Unwind_RaiseException

```cpp
_Unwind_RaiseException(exception)
{

    context = 当前CPU状态;

    /* 根据 eip 从 eh_frame 获取当前 frame */
	while(frame = 当前frame : eh_frame) {
        CFA = frame.CFA;
        ret = mem[CFA - 4];
        
        /* 根据 ret 寻找 landing_pad */
        for (item : gcc_except_table) {
            if (exception == item.exception &&
               ret属于item.eip的范围) {
                ebp = mem[CFA - 8];
                esp = CFA;
                landing_pad = item.landing_pad;
                
                set ebp;
                set esp;
                jmp landing_pad;
            }
        }
    }
    
    while(true)
    {
	
        ret = frame.ret; // 获取当前栈帧的返回地址

        // 根据异常类型和返回地址在.eh_frame查找着陆点对应的栈帧
        // 因为函数的栈帧基本由ebp和esp决定,所以返回这俩就可以了 
		frame = eh_frame(exception, ret); 
        
        					   
        result =
          personality(
             frame,
             SEARCH_PHASE,
             exception
          );


        if(result != HANDLER_FOUND)
            break;
        
        // 根据异常类型返回地址在.gcc_except_table查找着陆点
        landing_pad = gcc_except_table(exception,ret);
        
        set_frame(frame); // 设置着陆点的栈帧
        
        jmp landing_pad // 直接跳转到着陆点
    }


}
```

编译器为了实现异常机制，会预先在可执行文件中添加两个表：`.eh_frame` 和 `.gcc_execpt_table`。

- `.eh_frame`这个表格制定的是一套规则， 它可以根据当前的 `eip` 所在的范围获取 `CFA`（规则制定的栈位置的起点，比如`CFA = ebp + 8`），然后再根据 `CFA` 获取到当前的栈帧中存储的上一个



```cpp
#include <iostream>	

class MyExcetption {
public:
	MyExcetption(const char* message) : message(message) {}
	~MyExcetption() {};
	const char* what() const { return message; }
private:
	const char* message;
};

class Demo {
public:
	Demo() { std::cout << "Demo构造" << std::endl; };
	~Demo() { std::cout << "Demo析构" << std::endl; }
};

int func() 
{
	Demo d;
    Demo& d2 = *(new Demo);
    int x = 1;
    if (x == 2)
	    throw MyExcetption("func() 抛出异常");
    else if (x == 3)
        throw 3;
    return 10;
}

int main() {
	int x = 1;
	try 
    {
		Demo d;
		func();
	} 
    catch (MyExcetption& e) 
    {
        x = 3;
	}
    catch (int e)
    {
        x = 4;
    }
    x = 5;
}

```

```assembly
MyExcetption::MyExcetption(char const*):
        push    ebp
        mov     ebp, esp
        mov     eax, DWORD PTR [ebp+8]
        mov     edx, DWORD PTR [ebp+12]
        mov     DWORD PTR [eax], edx
        nop
        pop     ebp
        ret
        .set    MyExcetption::MyExcetption(char const*),MyExcetption::MyExcetption(char const*)
MyExcetption::~MyExcetption() :
        push    ebp
        mov     ebp, esp
        nop
        pop     ebp
        ret
        .set    MyExcetption::~MyExcetption() ,MyExcetption::~MyExcetption() 
.LC0:
        .string "Demo\346\236\204\351\200\240"
Demo::Demo():
        push    ebp
        mov     ebp, esp
        sub     esp, 8
        sub     esp, 8
        push    OFFSET FLAT:.LC0
        push    OFFSET FLAT:_ZSt4cout
        call    std::basic_ostream<char, std::char_traits<char>>& std::operator<<<std::char_traits<char>>(std::basic_ostream<char, std::char_traits<char>>&, char const*)
        add     esp, 16
        sub     esp, 8
        push    OFFSET FLAT:_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
        push    eax
        call    std::ostream::operator<<(std::ostream& (*)(std::ostream&))
        add     esp, 16
        nop
        leave
        ret
        .set    Demo::Demo(),Demo::Demo()
.LC1:
        .string "Demo\346\236\220\346\236\204"
Demo::~Demo() :
        push    ebp
        mov     ebp, esp
        sub     esp, 8
        sub     esp, 8
        push    OFFSET FLAT:.LC1
        push    OFFSET FLAT:_ZSt4cout
        call    std::basic_ostream<char, std::char_traits<char>>& std::operator<<<std::char_traits<char>>(std::basic_ostream<char, std::char_traits<char>>&, char const*)
        add     esp, 16
        sub     esp, 8
        push    OFFSET FLAT:_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
        push    eax
        call    std::ostream::operator<<(std::ostream& (*)(std::ostream&))
        add     esp, 16
        nop
        leave
        ret
        .set    Demo::~Demo() ,Demo::~Demo() 
.LC2:
        .string "func() \346\212\233\345\207\272\345\274\202\345\270\270"
func():
        push    ebp
        mov     ebp, esp
        push    esi
        push    ebx
        sub     esp, 32
        sub     esp, 12
        lea     eax, [ebp-17]
        push    eax
        call    Demo::Demo()
        add     esp, 16
        sub     esp, 12
        lea     eax, [ebp-18]
        push    eax
        call    Demo::Demo()
        add     esp, 16
        sub     esp, 12
        push    1
        call    operator new(unsigned int)
        add     esp, 16
        mov     ebx, eax
        mov     BYTE PTR [ebp-25], 1
        sub     esp, 12
        push    ebx
        call    Demo::Demo()
        add     esp, 16
        mov     DWORD PTR [ebp-12], ebx
        mov     DWORD PTR [ebp-16], 1
        cmp     DWORD PTR [ebp-16], 2
        jne     .L6
        sub     esp, 12
        push    4
        call    __cxa_allocate_exception
        add     esp, 16
        mov     ebx, eax
        sub     esp, 8
        push    OFFSET FLAT:.LC2
        push    ebx
        call    MyExcetption::MyExcetption(char const*)
        add     esp, 16
        sub     esp, 4
        push    OFFSET FLAT:MyExcetption::~MyExcetption()
        push    OFFSET FLAT:typeinfo for MyExcetption
        push    ebx
        call    __cxa_throw
.L6:
        mov     ebx, 10
        sub     esp, 12
        lea     eax, [ebp-18]
        push    eax
        call    Demo::~Demo()
        add     esp, 16
        sub     esp, 12
        lea     eax, [ebp-17]
        push    eax
        call    Demo::~Demo()
        add     esp, 16
        mov     eax, ebx
        jmp     .L15
        mov     esi, eax
        cmp     BYTE PTR [ebp-25], 0
        je      .L9
        sub     esp, 8
        push    1
        push    ebx
        call    operator delete(void*, unsigned int)
        add     esp, 16
.L9:
        mov     ebx, esi
        jmp     .L10
        mov     ebx, eax
.L10:
        sub     esp, 12
        lea     eax, [ebp-18]
        push    eax
        call    Demo::~Demo()
        add     esp, 16
        jmp     .L11
        mov     ebx, eax
.L11:
        sub     esp, 12
        lea     eax, [ebp-17]
        push    eax
        call    Demo::~Demo()
        add     esp, 16
        mov     eax, ebx
        sub     esp, 12
        push    eax
        call    _Unwind_Resume
.L15:
        lea     esp, [ebp-8]
        pop     ebx
        pop     esi
        pop     ebp
        ret
main:
        lea     ecx, [esp+4]
        and     esp, -16
        push    DWORD PTR [ecx-4]
        push    ebp
        mov     ebp, esp
        push    esi
        push    ebx
        push    ecx
        sub     esp, 28
        mov     DWORD PTR [ebp-28], 1
        sub     esp, 12
        lea     eax, [ebp-37]
        push    eax
        call    Demo::Demo()
        add     esp, 16
        call    func()
        sub     esp, 12
        lea     eax, [ebp-37]
        push    eax
        call    Demo::~Demo() 
        add     esp, 16
.L22:
        mov     DWORD PTR [ebp-28], 5
        mov     eax, 0
        jmp     .L25
        mov     esi, eax
        mov     ebx, edx
        sub     esp, 12
        lea     eax, [ebp-37]
        push    eax
        call    Demo::~Demo()
        add     esp, 16
        mov     eax, esi
        mov     edx, ebx
        jmp     .L19
.L19:
        cmp     edx, 1
        je      .L20
        cmp     edx, 2
        je      .L21
        sub     esp, 12
        push    eax
        call    _Unwind_Resume
.L20:
        sub     esp, 12
        push    eax
        call    __cxa_begin_catch
        add     esp, 16
        mov     DWORD PTR [ebp-36], eax
        mov     DWORD PTR [ebp-28], 3
        call    __cxa_end_catch
        jmp     .L22
.L21:
        sub     esp, 12
        push    eax
        call    __cxa_begin_catch
        add     esp, 16
        mov     eax, DWORD PTR [eax]
        mov     DWORD PTR [ebp-32], eax
        mov     DWORD PTR [ebp-28], 4
        call    __cxa_end_catch
        jmp     .L22
.L25:
        lea     esp, [ebp-12]
        pop     ecx
        pop     ebx
        pop     esi
        pop     ebp
        lea     esp, [ecx-4]
        ret
typeinfo for MyExcetption:
        .long   _ZTVN10__cxxabiv117__class_type_infoE+8
        .long   typeinfo name for MyExcetption
typeinfo name for MyExcetption:
        .string "12MyExcetption"
```



---

### 如何继承异常

就是维护一个字符串`msg`，然后用`what()`返回这个字符串`msg`。

~~~cpp
class exception {
private:
    std::string msg;
public:
   explicit exception(const char* msg) : msg(msg) {}
   const char* what() const {
       return msg.c_str();
   }
}
~~~

就是这样。自己构造一个类继承`exception`也是这样写。

~~~cpp
class MyException : public exception {
private:
	std::string msg;
public:
	explicit MyException(const char* msg) : msg(msg) {}
	const char* what() const override {
		return msg.c_str();
	}
};
~~~

