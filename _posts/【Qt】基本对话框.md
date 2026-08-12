---
title: 【Qt】基本对话框
date: 2026-08-09 14:52:53
tags:
  - C/C++
  - Qt
---

### 一、基本对话框概述

基本对话框是Qt系统内置的一系列通用对话框，拿来即可使用，无需重复开发。常见的如颜色选择对话框、文件选择对话框、字体选择对话框等。

> **特点**：这些对话框都是**模态**的，弹出后必须处理完才能操作其他窗口。

---

### 二、颜色选择对话框（QColorDialog）

#### 功能介绍

`QColorDialog` 用于让用户从调色板中选择一种颜色，并返回选中的颜色值。典型应用场景：绘图软件中的颜色选择、文本编辑器中的字体颜色设置等。

#### 使用前的准备

```cpp
#include <QColorDialog>
#include <QColor>
#include <QDebug>
```

#### 核心方法：getColor()

`QColorDialog` 的使用方式与 `QMessageBox` 类似，都通过**静态方法**调用。

```cpp
QColor getColor(
    const QColor &initial = Qt::white,  // 默认颜色
    QWidget *parent = nullptr,          // 父窗口
    const QString &title = QString(),   // 对话框标题
    ColorDialogOptions options = 0      // 选项（可忽略）
);
```

#### 参数说明

| 参数      | 说明                       | 默认值              |
| --------- | -------------------------- | ------------------- |
| `initial` | 对话框打开时默认选中的颜色 | `Qt::white`（白色） |
| `parent`  | 父窗口指针                 | `nullptr`           |
| `title`   | 对话框标题                 | 空字符串            |
| `options` | 额外选项                   | 0（忽略即可）       |

#### QColor 类型

`QColor` 是Qt中表示颜色的类，通过RGB三原色（红绿蓝）和Alpha通道（透明度）来描述颜色：

- `red()` → 红色分量（0-255）
- `green()` → 绿色分量（0-255）
- `blue()` → 蓝色分量（0-255）
- `alpha()` → 透明度（0透明 ~ 255不透明）

#### 使用示例

```cpp
// 弹出颜色选择对话框，默认选中青色
QColor color = QColorDialog::getColor(Qt::cyan, this, "选择颜色");

// 输出选中的颜色信息
qDebug() << "红色:" << color.red();
qDebug() << "绿色:" << color.green();
qDebug() << "蓝色:" << color.blue();
qDebug() << color;  // 直接输出，格式为 QColor(ARGB)
```

#### 运行效果

点击按钮后，弹出颜色选择面板：
- 左侧为基本颜色选择区域
- 右侧为自定义调色板
- 底部显示当前选中颜色的RGB数值
- 点击"OK"确认选择，点击"Cancel"取消

---

### 三、文件选择对话框（QFileDialog）

#### 功能介绍

`QFileDialog` 用于让用户选择文件或目录，返回文件路径。典型应用场景：记事本中的"打开文件"、"保存文件"功能。

#### 使用前的准备

```cpp
#include <QFileDialog>
```

#### 核心方法

文件对话框主要有两个常用静态方法：

| 方法                     | 说明                       | 返回值        |
| ------------------------ | -------------------------- | ------------- |
| `getOpenFileName()`      | 选择**单个**文件           | `QString`     |
| `getOpenFileNames()`     | 选择**多个**文件           | `QStringList` |
| `getSaveFileName()`      | 选择保存路径（获取文件名） | `QString`     |
| `getExistingDirectory()` | 选择目录                   | `QString`     |

#### getOpenFileName() 参数详解

```cpp
QString getOpenFileName(
    QWidget *parent = nullptr,      // 父窗口
    const QString &caption = QString(), // 对话框标题
    const QString &dir = QString(), // 默认打开目录
    const QString &filter = QString(), // 文件筛选器
    QString *selectedFilter = nullptr, // 选中的筛选器（可忽略）
    Options options = 0             // 选项（可忽略）
);
```

| 参数      | 说明                       | 示例                 |
| --------- | -------------------------- | -------------------- |
| `parent`  | 父窗口指针                 | `this`               |
| `caption` | 对话框标题                 | `"打开文件"`         |
| `dir`     | 默认打开的目录             | `"C:\\"` 或 `"D:\\"` |
| `filter`  | 文件筛选器，只显示指定类型 | `"*.txt *.cpp"`      |

#### 文件筛选器的写法

```cpp
// 筛选多个文件类型（用空格或;;分隔）
"*.txt *.cpp *.h"           // 空格分隔
"*.txt;;*.cpp;;*.h"         // 双分号分隔

// 带描述文字
"文本文件 (*.txt);;C++文件 (*.cpp);;头文件 (*.h)"
```

#### 使用示例

```cpp
// 打开文件对话框
QString fileName = QFileDialog::getOpenFileName(
    this, 
    "打开文件", 
    "D:\\", 
    "*.cpp *.h *.txt"
);
qDebug() << fileName;  // 输出文件路径，如 "D:/project/main.cpp"
```

```cpp
// 保存文件对话框（选择保存路径）
QString saveName = QFileDialog::getSaveFileName(
    this, 
    "选择保存路径", 
    "C:\\", 
    "*.txt;;*.cpp;;*.h"
);
qDebug() << saveName;
```

---

### 四、完整示例代码

```cpp
Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);

    // ========== 颜色选择对话框 ==========
    connect(ui->buttonSelectColor, &QPushButton::clicked, this, [=]() {
        // 弹出颜色选择对话框，默认选中青色
        QColor color = QColorDialog::getColor(Qt::cyan, this, "选择颜色");
        
        // 如果用户点击了取消，返回的颜色是无效的
        if (color.isValid()) {
            qDebug() << "选中的颜色:" << color;
            qDebug() << "RGB值:" << color.red() << color.green() << color.blue();
        } else {
            qDebug() << "用户取消了颜色选择";
        }
    });

    // ========== 文件保存对话框 ==========
    connect(ui->buttonSelectFile, &QPushButton::clicked, this, [=]() {
        // 弹出保存文件对话框，默认打开C盘
        QString fileName = QFileDialog::getSaveFileName(
            this, 
            "选择文件", 
            "C:\\", 
            "*.txt;;*.cpp;;*.h"
        );
        
        // 如果用户点击了取消，返回的字符串为空
        if (!fileName.isEmpty()) {
            qDebug() << "选择的文件路径:" << fileName;
        } else {
            qDebug() << "用户取消了文件选择";
        }
    });
}
```

---

### 五、其它常用基本对话框

除了颜色和文件对话框，Qt还提供了以下内置对话框：

| 类名               | 功能         | 示例场景           |
| ------------------ | ------------ | ------------------ |
| `QFontDialog`      | 选择字体     | 编辑器中的字体设置 |
| `QInputDialog`     | 获取用户输入 | 输入名称、密码等   |
| `QProgressDialog`  | 显示进度     | 文件复制、下载进度 |
| `QPrintDialog`     | 打印设置     | 文档打印           |
| `QPageSetupDialog` | 页面设置     | 打印页面配置       |

#### QFontDialog 简单示例

```cpp
#include <QFontDialog>

bool ok;
QFont font = QFontDialog::getFont(&ok, this);
if (ok) {
    qDebug() << "字体名称:" << font.family();
    qDebug() << "字体大小:" << font.pointSize();
}
```

#### QInputDialog 简单示例

```cpp
#include <QInputDialog>

QString text = QInputDialog::getText(
    this, 
    "输入对话框", 
    "请输入您的名字:"
);
qDebug() << "输入的内容:" << text;
```

---

### 六、总结

#### 使用基本对话框的通用步骤

1. **包含头文件**：`#include <对应类名>`
2. **查看帮助文档**：了解静态方法和参数
3. **调用静态方法**：`类名::方法名(参数)`
4. **处理返回值**：根据返回值执行相应操作
5. **检查有效性**：判断用户是否点击了取消

#### 常用静态方法速查

| 对话框         | 常用方法             | 返回值        |
| -------------- | -------------------- | ------------- |
| `QColorDialog` | `getColor()`         | `QColor`      |
| `QFileDialog`  | `getOpenFileName()`  | `QString`     |
| `QFileDialog`  | `getOpenFileNames()` | `QStringList` |
| `QFileDialog`  | `getSaveFileName()`  | `QString`     |
| `QFontDialog`  | `getFont()`          | `QFont`       |
| `QInputDialog` | `getText()`          | `QString`     |

#### 注意事项

- 所有基本对话框都是**模态**的
- 用户点击"取消"时，返回的对象通常处于**无效状态**（如颜色无效、字符串为空）
- 建议在使用返回值前进行**有效性检查**

