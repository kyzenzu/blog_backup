---
title: 【Qt】QMessageBox对话框
date: 2026-08-09 14:46:15
tags:
  - C/C++
  - Qt
---

### 一、QMessageBox概述

#### 什么是QMessageBox

`QMessageBox` 是Qt提供的标准消息对话框，用于在程序中显示信息、弹出询问、发出警告或错误提示。它是应用程序与用户交互的重要方式之一。

#### 典型应用场景

我们每天都在和各种消息对话框打交道：

- 在记事本中编辑文本后，未保存就点击关闭 → 弹出"是否保存"的询问对话框
- 删除文件时的确认提示 → "确定要删除此文件吗？"
- 操作成功或失败的通知 → "保存成功！"或"保存失败！"

#### 重要特点

`QMessageBox` 都是**模态对话框**，弹出时必须处理消息才能进行其他操作。

---

### 二、使用前的准备

使用 `QMessageBox` 需要包含头文件：

```cpp
#include <QMessageBox>
```

---

### 三、QMessageBox的四种类型

`QMessageBox` 提供了四种静态成员方法，分别对应四种不同用途的消息框：

| 方法            | 图标   | 用途                           |
| --------------- | ------ | ------------------------------ |
| `critical()`    | 红色×  | 错误提示，表示程序出现严重问题 |
| `information()` | 蓝色 i | 信息提示，普通通知消息         |
| `question()`    | 蓝色 ? | 询问用户，需要用户做出选择     |
| `warning()`     | 黄色 ! | 警告提示，提醒用户注意潜在问题 |

> **注意**：这些方法都是**静态成员函数**，直接通过 `QMessageBox::方法名()` 调用，无需创建对象。

---

### 四、基本使用（前三个参数）

所有类型的消息框都至少需要三个参数：

```cpp
QMessageBox::类型(this, "标题", "内容");
```

#### 参数说明

| 参数位置 | 参数名   | 说明                       |
| -------- | -------- | -------------------------- |
| 第1个    | `parent` | 父窗口指针，通常写 `this`  |
| 第2个    | `title`  | 对话框的标题               |
| 第3个    | `text`   | 对话框中显示的主要文字内容 |

#### 四种类型的用法对比

```cpp
// 1. 错误消息框（红色×）
QMessageBox::critical(this, "错误", "这是一个错误消息框");

// 2. 信息消息框（蓝色 i）
QMessageBox::information(this, "信息", "这是一个信息提示框");

// 3. 警告消息框（黄色 !）
QMessageBox::warning(this, "警告", "这是一个警告消息框");

// 4. 询问消息框（蓝色 ?）
QMessageBox::question(this, "询问", "这是一个询问消息框");
```

运行效果：前三种只显示一个 **OK** 按钮，而 `question()` 默认显示 **Yes** 和 **No** 两个按钮。

---

### 五、进阶使用（后两个参数）

`question()` 和 `information()` 等方法其实有五个参数，后两个参数用于自定义按钮：

```cpp
QMessageBox::StandardButton ret = QMessageBox::方法名(
    parent,      // 父窗口
    title,       // 标题
    text,        // 内容
    buttons,     // 显示的按钮（可组合）
    defaultButton // 默认按钮
);
```

#### 参数详解

**参数4 - `buttons`（按钮组合）**

指定对话框中显示哪些按钮，类型为 `QMessageBox::StandardButton` 枚举。

常用按钮枚举值：

| 枚举值                | 显示文字 |
| --------------------- | -------- |
| `QMessageBox::Ok`     | OK       |
| `QMessageBox::Save`   | Save     |
| `QMessageBox::Cancel` | Cancel   |
| `QMessageBox::Yes`    | Yes      |
| `QMessageBox::No`     | No       |
| `QMessageBox::Close`  | Close    |
| `QMessageBox::Apply`  | Apply    |

多个按钮用 `|`（或运算符）组合：

```cpp
QMessageBox::Save | QMessageBox::Cancel  // 同时显示 Save 和 Cancel
```

**参数5 - `defaultButton`（默认按钮）**

指定哪个按钮为默认按钮（即按回车键时触发的按钮）。如果不指定，系统会自动选择第一个。

```cpp
QMessageBox::question(this, "询问", "请问是否保存", 
                      QMessageBox::Save | QMessageBox::Cancel,
                      QMessageBox::Cancel);  // 默认选中 Cancel
```

---

### 六、获取用户的选择结果

`QMessageBox` 的静态方法会返回用户点击的按钮，我们可以通过判断返回值来执行不同操作：

```cpp
// 接收返回值
QMessageBox::StandardButton ret = QMessageBox::information(
    this, 
    "信息", 
    "你吃了吗?",
    QMessageBox::Yes | QMessageBox::No,
    QMessageBox::Yes
);

// 判断用户点击了哪个按钮
if (ret == QMessageBox::StandardButton::Yes) {
    qDebug() << "是的吃过饭了";
} else {
    qDebug() << "还没吃过饭呢";
}
```

---

### 七、完整示例代码

```cpp
Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);

    // 注意：MessageBox 都是模态对话框

    // ---------- 1. 信息对话框 ----------
    connect(ui->pushButton_1, &QPushButton::clicked, this, [=]() {
        // 方式一：不关心返回值
        // QMessageBox::information(this, "信息", "你吃了吗?");

        // 方式二：获取用户选择
        QMessageBox::StandardButton ret = QMessageBox::information(
            this, 
            "信息", 
            "你吃了吗?",
            QMessageBox::Yes | QMessageBox::No,
            QMessageBox::Yes
        );

        if (ret == QMessageBox::StandardButton::Yes) {
            qDebug() << "是的吃过饭了";
        } else {
            qDebug() << "还没吃过饭呢";
        }
    });

    // ---------- 2. 询问对话框（使用默认按钮） ----------
    connect(ui->pushButton_2, &QPushButton::clicked, this, [=]() {
        QMessageBox::question(this, "询问", "你吃午饭了吗");
    });

    // ---------- 3. 警告对话框 ----------
    connect(ui->pushButton_3, &QPushButton::clicked, this, [=]() {
        QMessageBox::warning(this, "警告", "你千万要吃饭");
    });

    // ---------- 4. 错误对话框 ----------
    connect(ui->pushButton_4, &QPushButton::clicked, this, [=]() {
        QMessageBox::critical(this, "危险", "不吃饭会饿死");
    });
}
```

---

### 八、总结

#### 核心要点

1. **头文件**：`#include <QMessageBox>`
2. **调用方式**：使用静态成员函数，`QMessageBox::类型()`
3. **模态特性**：所有消息框都是模态的，弹出后不能操作其他窗口
4. **五种参数**：
   - 前3个必填：父窗口、标题、内容
   - 后2个可选：按钮组合、默认按钮
5. **返回值**：返回用户点击的按钮，可用于判断用户选择

#### 使用流程图

```
包含头文件 → 选择消息框类型 → 设置参数 → 接收返回值 → 处理用户选择
```

#### 常用枚举速查

| 枚举                  | 用途       |
| --------------------- | ---------- |
| `QMessageBox::Yes`    | "是"按钮   |
| `QMessageBox::No`     | "否"按钮   |
| `QMessageBox::Ok`     | "确定"按钮 |
| `QMessageBox::Cancel` | "取消"按钮 |
| `QMessageBox::Save`   | "保存"按钮 |
| `QMessageBox::Close`  | "关闭"按钮 |

