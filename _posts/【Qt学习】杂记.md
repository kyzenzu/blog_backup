---
title: 【Qt学习】杂记
date: 2026-04-24 22:14:46
tags:
  - C++
  - Qt
---

ui文件编译出cpp文件的前后对比

~~~xml
// widget.ui
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>Widget</class>
 <widget class="QWidget" name="Widget">
     
  <!-- 控件1：标签 -->
  <widget class="QLabel" name="infoLabel">
  </widget>
  
  <!-- 控件2：按钮 -->
  <widget class="QPushButton" name="clickButton">
  </widget>
  
  <!-- 控件3：文本输入框 -->
  <widget class="QLineEdit" name="inputEdit">
  </widget>
  
 </widget>
</ui>
~~~

~~~cpp
class Ui_Widget
{
public:
    // 成员变量：UI文件中定义的所有子控件（不包括顶层Widget）
    QLabel *infoLabel;
    QPushButton *clickButton;
    QLineEdit *inputEdit;

    // 设置UI的主要函数
    void setupUi(QWidget *Widget)
    {
        // 1. 设置顶层控件的属性
        if (Widget->objectName().isEmpty())
            Widget->setObjectName(QString::fromUtf8("Widget"));
        
        // 2. 创建并设置 infoLabel
        infoLabel = new QLabel(Widget);
        infoLabel->setObjectName(QString::fromUtf8("infoLabel"));
        
        // 3. 创建并设置 clickButton
        clickButton = new QPushButton(Widget);
        clickButton->setObjectName(QString::fromUtf8("clickButton"));
        
        // 4. 创建并设置 inputEdit
        inputEdit = new QLineEdit(Widget);
        inputEdit->setObjectName(QString::fromUtf8("inputEdit"));
        
        // 5. 设置所有控件的文本内容
        retranslateUi(Widget);
        
        // 6. 自动连接信号槽（基于命名约定）
        QMetaObject::connectSlotsByName(Widget);
    }
    
    // 多语言支持函数
    void retranslateUi(QWidget *Widget)
    {
        // 设置控件显示文本
        infoLabel->setText(QCoreApplication::translate("Widget", "欢迎使用Qt程序", nullptr));
        clickButton->setText(QCoreApplication::translate("Widget", "点击我", nullptr));
        // inputEdit 的文本不需要设置，placeholder已经在setupUi中设置
        // 窗口标题已经在setupUi中设置
        (void)Widget; // 防止未使用参数警告
    }
};

// 命名空间封装，方便使用
namespace Ui {
    class Widget: public Ui_Widget {};
} // namespace Ui
~~~

* ui文件提供了创建`Ui::Widget`类的模板，`uic`根据这个文件编写`Ui::Widget`类的`cpp`源码

* ui文件中的`<class>`标签用于表明，当前ui文件描绘的是`Ui`域下的`Ui::Widget`类。

* ui文件中的顶层`<widget>`控件并不会生成创建对象代码，它只是描绘了`setupUi`要接收的参数类型，并设置参数名。顶层`<widget>`对象由用户创建，传递给`setUi`函数

* 后面的`<widget>`控件在`setupUi`中创建，在`setupUi`中指向父对象为用户创建的`Widget`对象。

使用示例：

~~~cpp
namespace Ui {
class Widget;
}

class Widget : public QWidget // 作为顶层父控件
{
    Q_OBJECT

public:
    explicit Widget(QWidget *parent = 0);
    ~Widget();

private:
    Ui::Widget *ui;
};
~~~

~~~cpp
Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget) // 在堆区创建各个子控件
{
    ui->setupUi(this); // 设置子控件的parent为当前Widget
}

Widget::~Widget()
{
    delete ui;
}
~~~



---

总结我在上面提到的connect的具体用法，讲讲connect的实现原理，以及信号和槽的实现原理。最好能够结合一些Qt5.8.0的源码

## 1. `connect` 的几种具体用法

### 老式字符串写法

```C++
connect(sender, SIGNAL(valueChanged(int)),
        receiver, SLOT(setValue(int)));
```

宏展开后近似是：

```C++
connect(sender, "2valueChanged(int)",
        receiver, "1setValue(int)");
```

`SIGNAL(x)` 通过 `#x` 把参数字符串化，`"2"` 表示 signal；`SLOT(x)` 类似，`"1"` 表示 slot。

------

### Qt5 新式函数指针写法

```c++
connect(sender, &Sender::valueChanged,
        receiver, &Receiver::setValue);
```

优点是：

```c++
&Sender::valueChanged
&Receiver::setValue
```

都是真实成员函数指针，能做编译期检查。

------

### 连接 lambda / 匿名函数

```c++
connect(sender, &Sender::valueChanged, [](int v) {
    qDebug() << v;
});
```

更推荐带 context：

```c++
connect(sender, &Sender::valueChanged, receiver, [](int v) {
    qDebug() << v;
});
```

带 `receiver/context` 的好处是：`receiver` 析构时自动断开连接。Qt 文档源码注释也明确说，连接到 functor/lambda 时，sender 或 context 被销毁会自动断开，但 lambda 捕获的对象仍要自己保证存活。

------

## 2. `connect` 本质做什么

不管哪种写法，`connect` 的核心目的都是建立一条记录：

```
sender 的某个 signal
        ↓
receiver 的某个 slot / lambda
```

更底层一点：

```
(sender, signal_index) → (receiver, method_index / slotObj)
```

也就是说，`connect` **不是马上调用函数**，只是把“以后信号发出时该调用谁”登记到连接表里。

------

## 3. 老式 `SIGNAL/SLOT` 的实现思路

老式写法：

```c++
connect(a, SIGNAL(sig(int)),
        b, SLOT(foo(int)));
```

宏展开：

```c++
connect(a, "2sig(int)",
        b, "1foo(int)");
```

然后 Qt 会：

```
"sig(int)" → 在 sender 的 QMetaObject 方法表中查 signal_index
"foo(int)" → 在 receiver 的 QMetaObject 方法表中查 method_index
```

所以旧写法是：

```
字符串 → 元对象查表 → index → 建立连接
```

缺点也很明显：

```c++
SIGNAL(sigg(int))  // 拼错
```

编译器不知道，只有运行时才可能报错。

------

## 4. 新式函数指针写法内部做了什么

新式写法：

```c++
connect(a, &A::sig, b, &B::foo);
```

你可能以为 Qt 会直接保存 `&B::foo`，以后直接：

```c++
(b->*&B::foo)();
```

但 QObject slot 的主路径不是这么简单。

Qt 5.8 这一类实现里，函数指针主要用于：

```
编译期检查类型
        ↓
找到 signal_index / slot 调用包装
        ↓
登记 Connection
```

Qt 源码里的 `QObject::connectImpl` 会通过 `static_metacall(QMetaObject::IndexOfMethod, ...)` 从函数指针对应的信号找到 `signal_index`，然后转交给 `QObjectPrivate::connectImpl`。

------

## 5. Connection 数据结构大概长什么样

Qt 5 的私有结构里，`QObjectPrivate::Connection` 很关键。源码里能看到它包含：

```c++
QObject *sender;
QAtomicPointer<QObject> receiver;
QAtomicPointer<QThreadData> receiverThreadData;

union {
    StaticMetaCallFunction callFunction;
    QtPrivate::QSlotObjectBase *slotObj;
};

QAtomicPointer<const int> argumentTypes;
ushort method_offset;
ushort method_relative;
signed int signal_index : 27;
ushort connectionType : 3;
ushort isSlotObject : 1;
```

也就是说一个连接节点既能表示普通 QObject 槽，也能表示 lambda/functor。普通槽用 `callFunction + method_offset + method_relative`，lambda/functor 用 `slotObj`。

连接表也不是全局乱搜。Qt 的 `ConnectionData` 中有 `signalVector`，每个 signal index 对应一条连接链表；同时每个 connection 也在 receiver 的 `senders` 链表里，方便析构和断开连接。源码注释写得很直白：`signalVector` 存每个 signal 的连接列表，`senders` 存连接到该对象 slot 的所有连接。

可以简化成：

```
sender
 └── ConnectionData
      └── signalVector
           ├── signal 0 → Connection → Connection → ...
           ├── signal 1 → Connection → ...
           └── signal 2 → ...
```

------

## 6. 信号函数是怎么实现的

你写：

```c++
signals:
    void sig(int value);
```

它不是普通函数体由你写。MOC 会生成信号函数实现，核心类似：

```c++
void A::sig(int value)
{
    void *argv[] = { nullptr, &value };
    QMetaObject::activate(this, &staticMetaObject, signal_index, argv);
}
```

所以：

```c++
emit sig(123);
```

里的 `emit` 只是空宏，本质就是调用：

```c++
sig(123);
```

而 `sig(123)` 内部真正做的是：

```c++
QMetaObject::activate(...)
```

Qt 5 源码中 `QMetaObject::activate` 会计算真实的 `signal_index`，然后进入 `doActivate`。

------

## 7. `activate` 如何调用槽函数

`activate` 的核心就是：

```
找到 sender 当前 signal_index 对应的连接链表
逐个遍历 Connection
根据连接类型决定直接调用还是投递事件
```

Qt 源码中能看到它遍历连接节点，取出 `receiver` 和 `receiverThreadData`，判断是否同线程；如果是 `AutoConnection` 且跨线程，或者显式 `QueuedConnection`，就走 `queued_activate`。

简化伪代码：

```c++
for (Connection *c : connectionsOf(sender, signal_index)) {
    if (需要排队调用) {
        postEvent(receiver, new QMetaCallEvent(...));
    } else {
        if (c->isSlotObject)
            c->slotObj->call(receiver, argv);
        else
            c->callFunction(receiver, InvokeMetaMethod, method_relative, argv);
    }
}
```

源码中对应逻辑也很清楚：

```c++
if (c->isSlotObject)
    c->slotObj->call(receiver, argv);
else
    callFunction(receiver, QMetaObject::InvokeMetaMethod, method_relative, argv);
```

也就是：lambda/functor 走 `slotObj->call`，QObject 槽走 MOC 生成的静态调用函数。

------

## 8. 槽函数是怎么被调用的

对于普通 QObject slot，Qt 不一定保存“槽函数地址”本身，而是保存：

```
method_offset
method_relative
callFunction
```

`callFunction` 本质上指向 MOC 生成的静态分发函数。它根据 method id 做分发：

```c++
switch (id) {
case 0:
    obj->foo(...);
    break;
case 1:
    obj->bar(...);
    break;
}
```

所以更准确的模型是：

```
不是：函数名 → 函数地址 → 调用

而是：method index → static_metacall / qt_metacall → switch → 调用真实函数
```

这也是为什么 private slot 也能被信号槽机制调用：它不是外部代码直接写 `receiver->foo()`，而是 MOC 生成的类内部调用代码在执行。

------

## 9. lambda 情况为什么不一样

lambda 没有元对象方法表，也没有 method index。

所以 Qt 会把 lambda 包成一个 `QSlotObjectBase` 派生对象，存在 `Connection::slotObj` 里。源码里 `Connection` 的 union 同时支持：

```c++
StaticMetaCallFunction callFunction;
QtPrivate::QSlotObjectBase *slotObj;
```

并用 `isSlotObject` 区分。

调用时就是：

```c++
slotObj->call(receiver, argv);
```

这条路径不靠 `qt_metacall` 的 method index。

------

## 10. 最终总图

```
你写 connect
   ↓
老语法：SIGNAL/SLOT 字符串 → 查 QMetaObject → signal_index / method_index
新语法：函数指针 → 模板检查 → 找 signal_index / 生成调用包装
lambda：生成 QSlotObjectBase 包装对象
   ↓
Qt 创建 Connection 节点
   ↓
挂到 sender->ConnectionData->signalVector[signal_index]
   ↓

emit sig(...)
   ↓
调用 MOC 生成的信号函数
   ↓
QMetaObject::activate(...)
   ↓
遍历该 signal 的 Connection 链表
   ↓
同线程 Direct：立即调用
跨线程 Queued：post QMetaCallEvent
   ↓
普通 slot：static_metacall / qt_metacall → switch → 真函数
lambda：slotObj->call(...)
```

一句话总结：

> Qt 信号槽不是 C++ 原生机制，而是 MOC 生成代码 + QMetaObject 元信息 + Connection 链表 + activate 分发器共同实现的“半反射回调系统”。

---

## Qt为什么类中可以定义`slots`和`signals`关键字

~~~cpp
// 在 Qt 头文件中搜索
// qt5/qtbase/src/corelib/kernel/qobjectdefs.h
#ifndef QT_NO_KEYWORDS
# define signals public
# define slots
# define emit
#else
// 如果定义了 QT_NO_KEYWORDS，则使用带下划线的版本
# define Q_SIGNALS public
# define Q_SLOTS
# define Q_EMIT
#endif
~~~

---

### 为什么QSqlQuery不需要指定我前面创建的QDataBase

QSqlQuery会自己去QSqlDatabase中找默认连接。

## 一、QSqlQuery 如何找到数据库连接

### 默认行为



```cpp
// 创建默认数据库连接
QSqlDatabase db = QSqlDatabase::addDatabase("QMYSQL");
db.setHostName("localhost");
db.setDatabaseName("mydb");
db.open();

// 不需要指定数据库，自动使用默认连接
QSqlQuery query;  // 使用默认连接
query.exec("SELECT * FROM students");

// 等价
QSqlQuery query(db);  // 显式指定
```



### QSqlQuery 的查找规则



```cpp
// QSqlQuery 的构造函数
QSqlQuery::QSqlQuery() {
    // 内部查找逻辑：
    // 1. 查找当前线程的默认数据库连接
    // 2. 如果没有，查找应用程序的默认连接
    // 3. 如果还没有，创建一个无效的查询对象
}

// 实际上相当于：
QSqlQuery query(QSqlDatabase::database());  // 默认参数
```



## 二、多数据库连接的场景

### 场景1：只有一个连接（最常用）

cpp

```cpp
// main.cpp
int main() {
    // 创建默认连接（不指定连接名）
    QSqlDatabase db = QSqlDatabase::addDatabase("QMYSQL");
    db.setDatabaseName("testdb");
    db.open();
    
    // 这里 QSqlQuery 自动使用上面的连接 ✅
    QSqlQuery query;
    query.exec("SELECT * FROM users");
    
    return 0;
}
```



### 场景2：多个命名连接

cpp

```cpp
// 创建多个数据库连接
void createConnections() {
    // 默认连接
    QSqlDatabase db1 = QSqlDatabase::addDatabase("QMYSQL", "default_conn");
    db1.setDatabaseName("db1");
    db1.open();
    
    // 第二个连接
    QSqlDatabase db2 = QSqlDatabase::addDatabase("QMYSQL", "second_conn");
    db2.setDatabaseName("db2");
    db2.open();
}

void queryDatabases() {
    // 方式1：使用默认连接（需要设置为默认）
    QSqlDatabase::database("default_conn");  // 激活某个连接
    
    // ❌ 这样不行，不知道用哪个
    // QSqlQuery query;  // 如果没设置默认连接，会失败
    
    // ✅ 正确：显式指定
    QSqlQuery query1(QSqlDatabase::database("default_conn"));
    QSqlQuery query2(QSqlDatabase::database("second_conn"));
    
    // ✅ 或者设置默认连接
    QSqlDatabase::setDefaultConnection("default_conn");
    QSqlQuery query;  // 现在使用 default_conn
}
```



## 三、内部实现原理

### QSqlQuery 的简化内部实现

cpp

```cpp
// Qt 源码简化版
class QSqlQuery {
private:
    QSqlDatabase m_db;
    
public:
    // 默认构造函数
    QSqlQuery() {
        // 获取当前线程的默认数据库
        m_db = QSqlDatabase::database();
        
        if (m_db.isValid()) {
            // 使用找到的数据库驱动
            m_driver = m_db.driver();
        } else {
            qWarning() << "QSqlQuery: No database connection available";
        }
    }
    
    // 显式指定数据库
    explicit QSqlQuery(const QSqlDatabase& db) 
        : m_db(db) {
        m_driver = m_db.driver();
    }
    
    // 执行查询
    bool exec(const QString& query) {
        if (!m_driver) {
            qWarning() << "QSqlQuery: No database driver";
            return false;
        }
        return m_driver->exec(query);
    }
};
```



### QSqlDatabase::database() 的实现逻辑

cpp

```cpp
// QSqlDatabase 获取默认连接的逻辑
QSqlDatabase QSqlDatabase::database(const QString& name, bool open) {
    // 内部维护一个全局的数据库连接池
    static QHash<QString, QSqlDatabase> connections;
    
    if (name.isEmpty()) {
        // 查找默认连接
        // 1. 查找线程局部存储中的默认连接
        // 2. 查找全局默认连接
        // 3. 如果没有，返回第一个创建的连接（历史原因）
    }
    
    return connections.value(name);
}
```



## 四、实际测试验证

cpp

```cpp
#include <QCoreApplication>
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlError>
#include <QDebug>

void testDefaultConnection() {
    // 创建数据库连接
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");
    db.setDatabaseName("test.db");
    
    if (!db.open()) {
        qDebug() << "Open failed:" << db.lastError().text();
        return;
    }
    
    // 创建表
    QSqlQuery query;
    query.exec("CREATE TABLE users (id INT, name TEXT)");
    
    // 插入数据
    query.exec("INSERT INTO users VALUES (1, 'Alice')");
    
    // 查询
    query.exec("SELECT * FROM users");
    while (query.next()) {
        qDebug() << query.value(0).toInt() 
                 << query.value(1).toString();
    }
    
    // 证明使用的是同一个连接
    qDebug() << "Connection name:" << db.connectionName();
    qDebug() << "Default connection:" 
             << QSqlDatabase::database().connectionName();
    // 输出相同的连接名
}

void testMultipleConnections() {
    // 创建两个连接
    QSqlDatabase db1 = QSqlDatabase::addDatabase("QSQLITE", "conn1");
    db1.setDatabaseName("db1.db");
    db1.open();
    
    QSqlDatabase db2 = QSqlDatabase::addDatabase("QSQLITE", "conn2");
    db2.setDatabaseName("db2.db");
    db2.open();
    
    // 这里有问题：默认是哪个？
    QSqlQuery query1;
    query1.exec("CREATE TABLE test (id INT)");  // 会用到哪个？不确定！
    
    // ✅ 正确做法：显式指定
    QSqlQuery query2(db1);
    query2.exec("INSERT INTO test VALUES (1)");  // 确定是 db1
    
    QSqlQuery query3(db2);
    query3.exec("INSERT INTO test VALUES (2)");  // 确定是 db2
}
```



## 五、您的代码为什么能工作？

cpp

```cpp
// 您之前的代码
void insertData() {
    // 假设在前面创建并打开了数据库
    QSqlDatabase db = QSqlDatabase::addDatabase("QMYSQL");
    db.setHostName("localhost");
    db.setDatabaseName("students");
    db.open();
    
    // 这里没有指定数据库
    QSqlQuery query;  // ✅ 自动使用上面创建的默认连接
    
    // 执行 SQL
    query.exec(sql);  // ✅ 使用默认连接
}
```



**工作原因**：

1. 您之前使用 `QSqlDatabase::addDatabase()` 时**没有指定连接名**
2. 这会创建一个**默认连接**
3. `QSqlQuery` 默认构造函数自动使用**默认连接**
