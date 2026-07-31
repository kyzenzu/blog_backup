---
title: C++文件操作
date: 2026-03-09 11:29:12
tags:
  - C++
---

## 文件操作



程序运行时产生的数据都属于临时数据，程序一旦运行结束都会被释放

通过**文件可以将数据持久化**

C++中对文件操作需要包含头文件 ==&lt; fstream &gt;==



文件类型分为两种：

1. **文本文件**     -  文件以文本的**ASCII码**形式存储在计算机中
2. **二进制文件** -  文件以文本的**二进制**形式存储在计算机中，用户一般不能直接读懂它们



操作文件的三大类:

1. ofstream：写操作
2. ifstream： 读操作
3. fstream ： 读写操作

---

* 

---





### 5.1文本文件

#### 5.1.1写文件

   写文件步骤如下：

1. 包含头文件   

	\#include <fstream\>

2. 创建流对象  

	ofstream ofs;

3. 打开文件

	ofs.open("文件路径",打开方式);

4. 写数据

	ofs << "写入的数据";

5. 关闭文件

	ofs.close();

	

文件打开方式：

| 打开方式                 | 解释                                                     |
| ------------------------ | -------------------------------------------------------- |
| ios::in                  | 为读文件而打开文件                                       |
| ios::out                 | 为写文件而打开文件                                       |
| ios::ate(**atend**)      | 初始位置：文件尾                                         |
| ios::app(**append**)     | 追加方式写文件                                           |
| ios::trunc(**truncate**) | 如果文件成功打开，并且文件已经存在，则把文件长度截断为 0 |
| ios::binary              | 二进制方式                                               |

**注意：** 文件打开方式可以配合使用，利用|操作符

**例如：**用二进制方式写文件 `ios::binary |  ios:: out`

#### 文件流默认打开方式

| 流类型     | 默认打开方式                        | 文件存在时的行为        | 文件不存在时的行为 |
| :--------- | :---------------------------------- | :---------------------- | :----------------- |
| `ofstream` | `std::ios::out | (std::ios::trunc)` | 清空文件内容（截断为0） | 创建新文件         |
| `ifstream` | `std::ios::in`                      | 正常打开（保留内容）    | 打开失败           |
| `fstream`  | `std::ios::in | std::ios::out`      | 正常打开（保留内容）    | 打开失败           |

> **`ofstream` 会自动补充 `ios::out`。**
>
> 也就是说：
>
> ```
> std::ofstream out("a.txt", std::ios::app);
> ```
>
> 实际等价于：
>
> ```
> std::ofstream out("a.txt", std::ios::out | std::ios::app);
> ```
>
> 注意：
>
> - **不会再自动加 `trunc`**
> - 因为 `app` 和 `trunc` 语义冲突

对于 `ofstream`：

```
不指定 mode      → out | trunc
指定 mode      → out | mode
```

也就是：

```
ofstream 总会自动加 out
但不会自动加 trunc（如果你自己写了 mode）
```

**示例：**

```C++
#include <fstream>

void test01()
{
	ofstream ofs;
	ofs.open("test.txt", ios::out);

	ofs << "姓名：张三" << endl;
	ofs << "性别：男" << endl;
	ofs << "年龄：18" << endl;

	ofs.close();
}

int main() {

	test01();

	system("pause");

	return 0;
}
```

总结：

* 文件操作必须包含头文件 fstream
* 读文件可以利用 ofstream  ，或者fstream类
* 打开文件时候需要指定操作文件的路径，以及打开方式
* 利用<<可以向文件中写数据
* 操作完毕，要关闭文件



#### 5.1.2读文件



读文件与写文件步骤相似，但是读取方式相对于比较多



读文件步骤如下：

1. 包含头文件   

	\#include <fstream\>

2. 创建流对象  

	ifstream ifs;

3. 打开文件并判断文件是否打开成功

	ifs.open("文件路径",打开方式);

4. 读数据

	四种方式读取

5. 关闭文件

	ifs.close();



**示例：**

```C++
#include <fstream>
#include <string>
void test01()
{
	ifstream ifs;
	ifs.open("test.txt", ios::in);

	if (!ifs.is_open())
	{
		cout << "文件打开失败" << endl;
		return;
	}

	//第一种方式
	char buf[1024] = { 0 };
	while (ifs >> buf) // 一次读取一个单词
	{
		cout << buf << endl;
	}

	//第二种
	//char buf[1024] = { 0 };
	//while (ifs.getline(buf,sizeof(buf)))
	//{
	//	cout << buf << endl;
	//}

	//第三种
	//string buf;
	//while (getline(ifs, buf))
	//{
	//	cout << buf << endl;
	//}

	char c;
	while ((c = ifs.get()) != EOF)
	{
		cout << c;
	}

	ifs.close();


}

int main() {

	test01();

	system("pause");

	return 0;
}
```

总结：

- 读文件可以利用 ifstream  ，或者fstream类
- 利用is_open函数可以判断文件是否打开成功
- close 关闭文件 

### 5.2 二进制文件

以二进制的方式对文件进行读写操作

打开方式要指定为 ==ios::binary==



#### 5.2.1 写文件

二进制方式写文件主要利用流对象调用成员函数write

函数原型 ：`ostream& write(const char * buffer,int len);`

参数解释：字符指针buffer指向内存中一段存储空间。len是读写的字节数



**示例：**

```C++
#include <fstream>
#include <string>

class Person
{
public:
	char m_Name[64];
	int m_Age;
};

//二进制文件  写文件
void test01()
{
	//1、包含头文件

	//2、创建输出流对象
	ofstream ofs("person.txt", ios::out | ios::binary);
	
	//3、打开文件
	//ofs.open("person.txt", ios::out | ios::binary);

	Person p = {"张三"  , 18};

	//4、写文件
	ofs.write((const char *)&p, sizeof(p));

	//5、关闭文件
	ofs.close();
}

int main() {

	test01();

	system("pause");

	return 0;
}
```

总结：

* 文件输出流对象 可以通过write函数，以二进制方式写数据











#### 5.2.2 读文件

二进制方式读文件主要利用流对象调用成员函数read

函数原型：`istream& read(char *buffer,int len);`

参数解释：字符指针buffer指向内存中一段存储空间。len是读写的字节数

示例：

```C++
#include <fstream>
#include <string>

class Person
{
public:
	char m_Name[64];
	int m_Age;
};

void test01()
{
	ifstream ifs("person.txt", ios::in | ios::binary);
	if (!ifs.is_open())
	{
		cout << "文件打开失败" << endl;
	}

	Person p;
	ifs.read((char *)&p, sizeof(p));

	cout << "姓名： " << p.m_Name << " 年龄： " << p.m_Age << endl;
}

int main() {

	test01();

	system("pause");

	return 0;
}
```



- 文件输入流对象 可以通过read函数，以二进制方式读数据



---

#### 关于文件流对象和打开方式

* **基础`mode`**：`ios::in`、`ios::out`。**修饰`mode`**：`ios::ate`、`ios::app`、`ios::trunc`。
* `ofstream`自带`ios::out | ios::trunc`、`ifstream`自带`ios::in`、`fstream`自带`ios::out | ios::in`
* 自己指定`mode`不会覆盖最基础的`mode`（比如`ios::in`、`ios::out`），只会追加覆盖有歧义的修饰`mode`，比如`ios::app`会覆盖`ios::trunc`。

---

#### 写文件

* `ofstream`自带基础模式：`ios::out`。

* `ofstream`可以再指定修饰模式：`ios:trunc`或者`ios:app`。这两个属性不能共存，会相互覆盖。而`ios:ate`这个模式没什么用就别去管了。
* `ofstream`默认属性是`ios::out | ios::trunc`

#### `operator<<`

使用`ofstream`写**文本文件**时，其实写的来源就是`char buf[]`。用`operator<<`输出运算符就行了。这个函数每次输出会识别字符数组末尾的`\0`，就当输出一个字符串。

~~~cpp
string names[] = { "Zoot", "Jimmy", "AI", "Stan" };
ofstream txtout("test.txt");
for (int i = 0; i < 4; i++) {
	txtout << names[i] << endl;
}
txtout.close();
~~~

#### `write`

使用`ofstream`写**二进制文件**时，其实就是把某块内存内容写到文件中去。不再有什么`\r\n`之分，也不会再受`\0`限制，就是无脑写字节。只用`write`函数就行，指定一下**内存起始地址**以及**要写的字节数**。

~~~cpp
struct Date {
	int mon, day, year;
};
Date date = { 6, 10, 92 };
ofstream ofs("test", ios::binary);
ofs.write((char*)&date, sizeof(date));
ofs.close();
~~~

文本模式和二进制模式最大区别：

1. 文本模式在C源码中碰到`\n`会解析成`\r\n`再输出到文件中
2. 文本模式输出就看`\0`在哪里就停在哪里
3. 二进制模式管你什么字节的特殊含义，字节就是字节直接原模原样的输出到文件中。

---

#### 读文件

文件内容

~~~
hello world
cpp
~~~

#### `operator>>` 格式化输入

读取代码

~~~cpp
ifstream ifs;
ifs.open("test.txt");

char buf[1024];
while (ifs >> buf) {
	cout << buf << endl;
}
~~~

* 操作符格式化读取返回值类型是`ifstream`，但是这个类型自己有类型转换函数转为`bool`类型。
* 读取本意：跳过空白字符，读取连续非空白字符（即一次读取一个单词到缓冲区）
* 额外操作：自动在缓冲区添加`\0`

读取结果

~~~
buf[]:hello\0
buf[]:world\0
buf[]:cpp\0
~~~

#### `getline()`按行读取

文件内容

~~~
hello world
cpp
~~~

读取代码

~~~cpp
ifstream ifs;
ifs.open("test.txt");

char buf[1024];
while (ifs.getline(buf, sizeof(buf))) {
	cout << buf << endl;
}
~~~

* 函数读取返回值类型是`ifstream`，但是这个类型自己有类型转换函数转为`bool`类型。
* 读取本意：一次读取一行，以换行符为界但不读取换行符
* 额外操作：自动在缓冲区添加`\0`

读取结果：

~~~
buf[]:hello world\0
buf[]:cpp\0
~~~

#### `std::getline()`按行读取

文件内容

~~~
hello world
cpp
~~~

读取代码

~~~cpp
ifstream ifs;
ifs.open("test.txt");

string buf;
while (std::getline(ifs, buf)) {
	cout << buf << endl;
}
~~~

* 函数读取返回值类型是`ifstream`，但是这个类型自己有类型转换函数转为`bool`类型。
* 读取本意：一次读取一行，以换行符为界但不读取换行符
* 额外操作：自动在缓冲区添加`\0`

读取结果：

~~~
buf[]:hello world\0
buf[]:cpp\0
~~~

#### `get`按字符读取

文件内容

~~~
hello world
cpp
~~~

读取代码

~~~cpp
char c;
while ((c = ifs.get()) != EOF) {
	cout << c;
}
~~~

* 读取本意：一次读取一个字符（字节）

读取结果：

~~~
hello world\n
cpp
~~~

