---
title: 【Qt】信号和槽的扩展
date: 2026-08-09 13:07:15
tags:
  - C/C++
  - Qt
---

### 一、示例类定义

以下笔记均基于 `Boy` 和 `Girl` 两个类，它们分别定义了重载的信号和槽：

**Boy 类**：

```cpp
class Boy : public QObject
{
    Q_OBJECT
public:
    explicit Boy(QObject *parent = 0);

signals:
    void love();              // 无参信号
    void love(QString);       // 带参信号
public slots:
};
```

**Girl 类**：

```cpp
class Girl : public QObject
{
    Q_OBJECT
public:
    explicit Girl(QObject *parent = 0);

signals:

public slots:
    void ack_love();          // 无参槽
    void ack_love(QString);   // 带参槽
};
```

---

### 二、信号连接信号

#### 概念说明

信号不仅可以连接槽函数，还可以连接另一个信号。当第一个信号被触发时，它会自动触发与之相连的第二个信号，从而实现信号的链式传递。

本质上是信号函数之间的调用关系：第一个信号被 `emit` 后，其内部会调用第二个信号函数，而第二个信号函数内部又会调用所连接的槽函数。

#### 应用场景

假设我们需要在按钮点击后才让 `Boy` 发出表白信号，进而调用 `Girl` 的响应槽函数。此时可以将按钮的 `clicked` 信号连接到 `Boy` 的 `love` 信号，形成一条完整的调用链：`按钮点击 → Boy 表白 → Girl 回应`。

#### 代码示例

```cpp
// 使用函数指针明确指定无参版本
void (Boy::* love)() = &Boy::love;
void (Girl::* ack_love)() = &Girl::ack_love;

// 信号连接信号：按钮点击 → Boy 发出 love 信号
QObject::connect(&button, &QPushButton::clicked, &boy, love);
// 信号连接槽：Boy 的 love 信号 → Girl 的 ack_love 槽
QObject::connect(&boy, love, &girl, ack_love);
```

此时点击按钮，`Boy` 会发出无参的 `love()` 信号，`Girl` 会响应无参的 `ack_love()` 槽函数。

---

### 三、断开信号连接

#### 概念说明

使用 `disconnect()` 函数可以解除已建立的信号与槽（或信号与信号）之间的连接。其参数与 `connect()` 完全一致：

```cpp
disconnect(发送者, 信号, 接收者, 槽/信号)
```

断开连接后，信号触发时将不再调用对应的槽函数或触发后续信号。

#### 代码示例

```cpp
// 断开按钮与 Boy 信号的连接
QObject::disconnect(&button, &QPushButton::clicked, &boy, love);
// 断开 Boy 信号与 Girl 槽的连接
QObject::disconnect(&boy, love, &girl, ack_love);
```

执行断开操作后，即使点击按钮，也不会再触发 `Boy` 的 `love` 信号，`Girl` 也不会收到任何响应。

---

### 四、一个信号连接多个槽函数

#### 概念说明

Qt 允许同一个信号同时连接多个槽函数。当该信号被触发时，所有与之相连的槽函数会按照连接顺序依次执行。

#### 应用场景

例如，点击按钮时不仅希望 `Boy` 向 `Girl` 表白，还希望同时关闭当前窗口。只需将按钮的 `clicked` 信号再连接一个关闭窗口的槽函数即可。

#### 代码示例

```cpp
// 连接表白功能
QObject::connect(&button, &QPushButton::clicked, &boy, love);
QObject::connect(&boy, love, &girl, ack_love);

// 再连接关闭窗口功能
QObject::connect(&button, &QPushButton::clicked, &w, &Widget::close);
```

此时点击按钮，`Girl` 会响应表白，同时窗口也会被关闭，两个槽函数都会被执行。

---

### 五、多个信号连接同一个槽函数

#### 概念说明

多个不同的信号也可以连接到同一个槽函数。无论哪个信号被触发，都会调用这个共同的槽函数。

#### 应用场景

最常见的情况是：窗口的关闭按钮和标题栏的关闭按钮（`×`）都连接到 `QWidget::close()` 槽函数，从而实现点击任意一个都能关闭窗口的功能。

#### 示例扩展

```cpp
// 按钮点击关闭窗口
QObject::connect(&button, &QPushButton::clicked, &w, &Widget::close);
// 窗口的关闭事件也会触发 close 槽
// （Qt 内部已实现，此处仅作概念说明）
```

---

### 六、信号和槽的参数匹配规则

#### 核心规则

1. **参数类型必须一致**：信号发送的参数类型与槽接收的参数类型必须完全匹配。
2. **信号参数个数可以多于槽参数个数**：但要求信号的前 N 个参数类型与槽的 N 个参数类型一一对应。
3. **信号参数个数不能少于槽参数个数**：否则槽函数无法获取到所需数据。

用公式表示即为：`Signal.paraNum >= Slot.paraNum`

#### 实际案例

假设按钮的 `clicked()` 信号是无参的，而 `Boy` 的 `love(QString)` 信号是带参的，两者无法直接连接：

```cpp
// 错误写法：无参信号无法直接连接有参信号
QObject::connect(&button, &QPushButton::clicked, &boy, (void (Boy::*)(QString))&Boy::love);
```

#### 解决方案：使用匿名函数（Lambda）做中转

```cpp
// 在无参的匿名函数中发射有参信号
QObject::connect(&button, &QPushButton::clicked, &boy, [&]() {
    emit boy.love("我们去看电影吧");
});

// 然后连接有参信号和有参槽
QObject::connect(&boy, (void (Boy::*)(QString))&Boy::love, 
                 &girl, (void (Girl::*)(QString))&Girl::ack_love);
```

这样点击按钮时，匿名函数被触发，内部发射带参的 `love(QString)` 信号，`Girl` 的带参槽函数就能正常响应。

---

### 七、完整示例代码

```cpp
int main(int argc, char *argv[])
{
    if(QT_VERSION >= QT_VERSION_CHECK(5,6,0))
        QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling, true);
    
    QApplication a(argc, argv);
    Widget w;

    QPushButton button;
    button.setParent(&w);
    button.setText("表白");

    Boy boy;
    Girl girl;

    // 使用函数指针明确指定无参版本
    void (Boy::* love)() = &Boy::love;
    void (Girl::* ack_love)() = &Girl::ack_love;

    // 信号连接信号
    QObject::connect(&button, &QPushButton::clicked, &boy, love);
    QObject::connect(&boy, love, &girl, ack_love);

    // 断开连接
    QObject::disconnect(&button, &QPushButton::clicked, &boy, love);
    QObject::disconnect(&boy, love, &girl, ack_love);

    // Lambda 中转：无参信号 → 无参槽 → 有参信号
    QObject::connect(&button, &QPushButton::clicked, &boy, [&]() {
        emit boy.love("我们去看电影吧");
    });
    // 有参信号连接有参槽
    QObject::connect(&boy, (void (Boy::*)(QString))&Boy::love, 
                     &girl, (void (Girl::*)(QString))&Girl::ack_love);

    w.show();
    return a.exec();
}
```

---

### 八、小结

| 特性         | 说明                                          |
| ------------ | --------------------------------------------- |
| 信号连接信号 | 信号之间可以链式传递，一个信号触发另一个信号  |
| 断开连接     | 使用 `disconnect()`，参数与 `connect()` 一致  |
| 一信号对多槽 | 一个信号可连接多个槽，按顺序执行              |
| 多信号对一槽 | 多个信号共享同一个槽函数                      |
| 参数匹配     | 信号参数数量 ≥ 槽参数数量，且前置类型必须一致 |

