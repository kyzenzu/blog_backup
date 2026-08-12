---
title: 【Qt】QTextEdit控件
date: 2026-08-09 13:56:16
tags:
  - C/C++
  - Qt
---



### 一、QTextEdit简介

#### 基本概念

`QTextEdit` 是 Qt 提供的多行文本编辑控件，用于接收和显示用户输入的文本内容。它是程序与用户进行数据交互的重要控件之一。

#### 常见应用场景

- 登录界面的账号/密码输入框
- 聊天软件的输入区域
- 文本编辑器的主编辑区
- 表单中的多行文本输入

#### 使用前的准备

使用 `QTextEdit` 需要包含对应的头文件：

```cpp
#include <QTextEdit>
```

> **注意**：`QTextEdit` 属于 `widgets` 模块，由于我们在创建 Qt 项目时默认已包含该模块，因此无需在 `.pro` 文件中额外添加配置。

---

### 二、创建QTextEdit控件

在窗口构造函数中创建一个 `QTextEdit` 对象，并设置其位置和大小：

```cpp
Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);

    // 创建多行文本编辑框
    QTextEdit* edit = new QTextEdit(this);
    edit->resize(200, 50);  // 设置宽200，高50
}
```

---

### 三、获取文本框中的内容

#### 需求描述

新建一个"输出"按钮，点击时将文本框中的内容输出到控制台。

#### 核心方法

使用 `toPlainText()` 获取纯文本内容，配合 `toUtf8().data()` 去除 Qt 默认的双引号。

#### 代码示例

```cpp
// 创建"输出"按钮
QPushButton* button = new QPushButton(this);
button->move(0, 50);
button->setText("输出");

// 点击按钮，输出文本框内容
connect(button, &QPushButton::clicked, [=]() {
    // toPlainText() 获取纯文本
    // toUtf8().data() 去除双引号，输出 char*
    qDebug() << edit->toPlainText().toUtf8().data();
});
```

> **方法链说明**：
> - `edit->toPlainText()` → 返回 `QString`
> - `.toUtf8()` → 转换为 `QByteArray`（UTF-8编码）
> - `.data()` → 转换为 `char*`，输出时不带双引号

---

### 四、设置文本框中的内容

#### 需求描述

新建一个"输入"按钮，点击时程序自动向文本框中写入内容，并设置文本样式。

#### 核心方法

- `setText()`：设置文本内容
- `setTextColor()`：设置文字颜色
- `setTextBackgroundColor()`：设置文字背景色
- `setFontPointSize()`：设置字体大小

#### 代码示例

```cpp
// 创建"输入"按钮
QPushButton* input = new QPushButton(this);
input->move(100, 50);
input->setText("输入");

// 点击按钮，程序自动填充文本
connect(input, &QPushButton::clicked, [=]() {
    // 设置文本颜色为品红
    edit->setTextColor(Qt::GlobalColor::magenta);
    // 设置文本背景色为绿色
    edit->setTextBackgroundColor(Qt::GlobalColor::green);
    // 设置字体大小为20
    edit->setFontPointSize(20);
    // 设置文本内容
    edit->setText("程序自动输入文本");
});
```

点击"输入"按钮后，编辑框中会显示品红色文字、绿色背景、字号为20的"程序自动输入文本"。

---

### 五、QTextEdit的常用信号

#### 信号列表

| 信号                      | 触发时机                       |
| ------------------------- | ------------------------------ |
| `textChanged()`           | 文本内容发生任何变化时         |
| `copyAvailable(bool)`     | 文本可被复制时（有选中内容时） |
| `undoAvailable(bool)`     | 撤销操作可用状态变化时         |
| `redoAvailable(bool)`     | 重做操作可用状态变化时         |
| `cursorPositionChanged()` | 光标位置发生变化时             |
| `selectionChanged()`      | 选中的文本发生变化时           |

---

### 六、实战示例：实时同步文本内容

#### 需求描述

在第一个文本框中输入内容时，第二个文本框实时同步显示相同的内容。

#### 实现思路

连接第一个文本框的 `textChanged` 信号，在槽函数中将第一个文本框的内容设置到第二个文本框中。

#### 完整代码示例

```cpp
Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);

    // ---------- 第一个文本框 ----------
    QTextEdit* edit = new QTextEdit(this);
    edit->resize(200, 50);

    // ---------- "输出"按钮 ----------
    QPushButton* button = new QPushButton(this);
    button->move(0, 50);
    button->setText("输出");
    connect(button, &QPushButton::clicked, [=]() {
        qDebug() << edit->toPlainText().toUtf8().data();
    });

    // ---------- "输入"按钮 ----------
    QPushButton* input = new QPushButton(this);
    input->move(100, 50);
    input->setText("输入");
    connect(input, &QPushButton::clicked, [=]() {
        edit->setTextColor(Qt::GlobalColor::magenta);
        edit->setTextBackgroundColor(Qt::GlobalColor::green);
        edit->setFontPointSize(20);
        edit->setText("程序自动输入文本");
    });

    // ---------- 第二个文本框（同步显示） ----------
    QTextEdit* copy = new QTextEdit(this);
    copy->resize(200, 50);
    copy->move(0, 70);

    // 核心：textChanged信号 → 实时同步
    connect(edit, &QTextEdit::textChanged, [=]() {
        copy->setText(edit->toPlainText());
    });
}
```

#### 运行效果

- 在 `edit` 中输入任何内容，`copy` 会实时同步显示相同内容
- 点击"输出"按钮，控制台显示 `edit` 中的内容
- 点击"输入"按钮，`edit` 被自动填充样式化文本，`copy` 同步更新

---

### 七、其他信号的测试示例

除了 `textChanged`，还可以测试其他信号，例如：

```cpp
// 文本被选中时触发
connect(edit, &QTextEdit::copyAvailable, [=](bool available) {
    if (available) {
        qDebug() << "文本可被复制";
    }
});

// 光标位置改变时触发
connect(edit, &QTextEdit::cursorPositionChanged, [=]() {
    qDebug() << "光标位置已改变";
});

// 选中内容改变时触发
connect(edit, &QTextEdit::selectionChanged, [=]() {
    qDebug() << "选中内容已改变";
});
```

---

### 八、总结

| 功能           | 核心方法/信号              |
| -------------- | -------------------------- |
| 获取纯文本     | `toPlainText()`            |
| 设置文本       | `setText()`                |
| 设置文字颜色   | `setTextColor()`           |
| 设置背景色     | `setTextBackgroundColor()` |
| 设置字号       | `setFontPointSize()`       |
| 文本变化信号   | `textChanged()`            |
| 文本可复制信号 | `copyAvailable(bool)`      |
| 光标移动信号   | `cursorPositionChanged()`  |
| 选中变化信号   | `selectionChanged()`       |

