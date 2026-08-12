---
title: 【Qt】QMainWindow
date: 2026-08-09 14:03:24
tags:
  - C/C++
  - Qt
---

### 一、QMainWindow概述

`QMainWindow` 是 Qt 中用于构建应用程序主窗口的基类，许多常见的应用程序（如文本编辑器、图片编辑器、IDE等）都基于它实现。

一个标准的 `QMainWindow` 界面由以下五个部分组成：

| 部件                           | 说明                                   | 数量       |
| ------------------------------ | -------------------------------------- | ---------- |
| **菜单栏（Menu Bar）**         | 位于窗口顶部，包含文件、编辑等下拉菜单 | 只能有一个 |
| **工具栏（Tool Bar）**         | 可放置常用操作按钮，可拖动、悬浮       | 可以有多个 |
| **状态栏（Status Bar）**       | 位于窗口底部，显示状态信息             | 只能有一个 |
| **铆接部件（Dock Widget）**    | 浮动窗口，可停靠在窗口四周             | 可以有多个 |
| **中心部件（Central Widget）** | 占据窗口中央区域，通常为主要工作区     | 只能有一个 |

> **记忆技巧**：使用 `set` 方法设置的部件只能有一个（菜单栏、状态栏、中心部件），使用 `add` 方法添加的部件可以有多个（工具栏、铆接部件）。

---

### 二、添加菜单栏（Menu Bar）

#### 基本概念

菜单栏位于主窗口的最顶部，用于组织应用程序的各种功能菜单。通过 `menuBar()` 方法可以获取或创建菜单栏。

#### 创建步骤

**第一步：获取或创建菜单栏**

```cpp
QMenuBar* menuBar = this->menuBar();  // 获取现有菜单栏，若无则自动创建
// 或手动创建
QMenuBar* menuBar = new QMenuBar(this);
```

**第二步：添加菜单**

```cpp
// 添加"文件"菜单
QMenu* fileMenu = menuBar->addMenu("文件");
// 添加"编辑"菜单
QMenu* editMenu = menuBar->addMenu("编辑");
// 添加"构建"菜单
QMenu* buildMenu = menuBar->addMenu("构建");
```

**第三步：向菜单中添加菜单项**

```cpp
// 在"文件"菜单中添加操作项
QAction* newFileAction = fileMenu->addAction("新建");
QAction* openFileAction = fileMenu->addAction("打开");
QAction* saveFileAction = fileMenu->addAction("保存");

// 添加分割线
fileMenu->addSeparator();

// 添加子菜单（嵌套菜单）
QMenu* lastFileMenu = fileMenu->addMenu("最近访问的文件");
lastFileMenu->addAction("1.txt");
lastFileMenu->addAction("2.txt");
lastFileMenu->addAction("3.txt");
```

**第四步：设置菜单栏到主窗口**

```cpp
this->setMenuBar(menuBar);
```

> **注意**：如果菜单栏为空（没有任何菜单），则不会显示。添加菜单后才会出现。

---

### 三、添加工具栏（Tool Bar）

#### 基本概念

工具栏提供了快速访问常用功能的入口，可以包含按钮、下拉框等控件。与菜单栏不同，工具栏可以有多个，且可以拖动、浮动或停靠在窗口的四边。

#### 创建步骤

**第一步：创建工具栏对象**

```cpp
QToolBar* toolBar = new QToolBar(this);  // parent可加可不加，addToolBar时会自动设置
```

**第二步：设置工具栏属性**

```cpp
// 设置允许停靠的区域（左侧和右侧）
toolBar->setAllowedAreas(Qt::LeftToolBarArea | Qt::RightToolBarArea);

// 设置是否可浮动（鼠标拖拽时变为独立窗口）
toolBar->setFloatable(false);

// 设置是否可移动（用户拖动改变位置）
toolBar->setMovable(false);
```

**第三步：添加操作项到工具栏**

可以直接将菜单栏中的 `QAction` 对象添加到工具栏，实现菜单和工具栏共用同一个操作：

```cpp
toolBar->addAction(newFileAction);   // 添加"新建"操作
toolBar->addAction(openFileAction);  // 添加"打开"操作
toolBar->addAction(saveFileAction);  // 添加"保存"操作
```

**第四步：将工具栏添加到主窗口**

```cpp
// 将工具栏停靠在左侧
this->addToolBar(Qt::ToolBarArea::LeftToolBarArea, toolBar);
```

> **常用停靠区域**：
> - `Qt::LeftToolBarArea`：左侧
> - `Qt::RightToolBarArea`：右侧
> - `Qt::TopToolBarArea`：顶部
> - `Qt::BottomToolBarArea`：底部

---

### 四、添加状态栏（Status Bar）

#### 基本概念

状态栏位于主窗口底部，用于显示程序运行状态、提示信息等。通常左侧显示临时信息，右侧显示永久信息（如时间、余额等）。

#### 创建步骤

**第一步：创建状态栏对象**

```cpp
QStatusBar* statusBar = new QStatusBar(this);
```

**第二步：添加显示控件**

```cpp
// 左侧区域：使用 addWidget 添加（从左到右排列）
QLabel* RMBLabel = new QLabel("人民币余额：100");
statusBar->addWidget(RMBLabel);

// 右侧区域：使用 addPermanentWidget 添加（从右到左排列，即永久区域）
QLabel* USDLabel = new QLabel("美元余额：100");
statusBar->addPermanentWidget(USDLabel);
```

**第三步：设置状态栏到主窗口**

```cpp
this->setStatusBar(statusBar);
```

> **区别说明**：
> - `addWidget()`：添加到状态栏左侧，按添加顺序从左到右排列
> - `addPermanentWidget()`：添加到状态栏右侧，从右到左排列，通常用于显示固定信息

---

### 五、添加铆接部件（Dock Widget）

#### 基本概念

铆接部件（也称浮动窗口）是一种可以悬浮在主窗口之上，或停靠在窗口四周的子窗口。常用于显示辅助信息面板（如属性面板、图层列表等）。

#### 创建步骤

**第一步：创建铆接部件对象**

```cpp
QDockWidget* dockWidget = new QDockWidget(this);
```

**第二步：设置铆接部件属性**

```cpp
// 设置允许停靠的区域（右侧和顶部）
dockWidget->setAllowedAreas(Qt::RightDockWidgetArea | Qt::TopDockWidgetArea);

// 设置是否默认悬浮（true为悬浮状态）
dockWidget->setFloating(true);
```

**第三步：向铆接部件中添加内容**

```cpp
// 可以添加任意QWidget作为子控件
QLabel* label = new QLabel("这是铆接部件", dockWidget);
dockWidget->setWidget(label);
```

**第四步：将铆接部件添加到主窗口**

```cpp
this->addDockWidget(Qt::RightDockWidgetArea, dockWidget);
```

> **常用停靠区域**：
> - `Qt::LeftDockWidgetArea`：左侧
> - `Qt::RightDockWidgetArea`：右侧
> - `Qt::TopDockWidgetArea`：顶部
> - `Qt::BottomDockWidgetArea`：底部

---

### 六、添加中心部件（Central Widget）

#### 基本概念

中心部件是主窗口中最重要的区域，占据了除菜单栏、工具栏、状态栏和铆接部件之外的所有空间。每个 `QMainWindow` 只能有一个中心部件。

#### 创建步骤

```cpp
// 创建一个文本编辑框作为中心部件
QTextEdit* textEdit = new QTextEdit(this);

// 设置为中心部件
this->setCentralWidget(textEdit);
```

> **说明**：中心部件可以是任何 `QWidget` 的子类，如 `QTextEdit`（文本编辑器）、`QGraphicsView`（图形视图）、`QWidget`（自定义面板）等。

---

### 七、完整示例代码

以下是完整的 `MainWindow` 构造函数代码，包含了上述所有部件的创建和使用：

```cpp
MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    // ========== 1. 添加顶部菜单栏 ==========
    QMenuBar* menuBar = new QMenuBar(this);

    // 文件菜单
    QMenu* fileMenu = menuBar->addMenu("文件");
    QAction* newFileAction = fileMenu->addAction("新建");
    QAction* openFileAction = fileMenu->addAction("打开");
    QAction* saveFileAction = fileMenu->addAction("保存");
    fileMenu->addSeparator();  // 分割线

    // 文件菜单下的子菜单
    QMenu* lastFileMenu = fileMenu->addMenu("最近访问的文件");
    lastFileMenu->addAction("1.txt");
    lastFileMenu->addAction("2.txt");
    lastFileMenu->addAction("3.txt");

    // 编辑和构建菜单
    QMenu* editMenu = menuBar->addMenu("编辑");
    QMenu* buildMenu = menuBar->addMenu("构建");

    this->setMenuBar(menuBar);

    // ========== 2. 添加侧边工具栏 ==========
    QToolBar* toolBar = new QToolBar;
    toolBar->setAllowedAreas(Qt::LeftToolBarArea | Qt::RightToolBarArea);
    toolBar->setFloatable(false);
    toolBar->setMovable(false);

    // 将菜单栏的操作添加到工具栏（共用操作）
    toolBar->addAction(newFileAction);
    toolBar->addAction(openFileAction);
    toolBar->addAction(saveFileAction);

    this->addToolBar(Qt::ToolBarArea::LeftToolBarArea, toolBar);

    // ========== 3. 添加底部状态栏 ==========
    QStatusBar* statusBar = new QStatusBar(this);

    QLabel* RMBLabel = new QLabel("人民币余额：100");
    statusBar->addWidget(RMBLabel);  // 左侧

    QLabel* USDLabel = new QLabel("美元余额：100");
    statusBar->addPermanentWidget(USDLabel);  // 右侧永久区域

    this->setStatusBar(statusBar);

    // ========== 4. 添加悬浮铆接部件 ==========
    QDockWidget* dockWidget = new QDockWidget(this);
    dockWidget->setAllowedAreas(Qt::RightDockWidgetArea | Qt::TopDockWidgetArea);
    dockWidget->setFloating(true);  // 初始为悬浮状态

    // 向铆接部件中添加内容
    QLabel* dockLabel = new QLabel("浮动面板内容", dockWidget);
    dockWidget->setWidget(dockLabel);

    this->addDockWidget(Qt::RightDockWidgetArea, dockWidget);

    // ========== 5. 添加中间中心部件 ==========
    QTextEdit* textEdit = new QTextEdit(this);
    this->setCentralWidget(textEdit);
}
```

---

### 八、总结

| 部件     | 类名           | 数量 | 添加方式             | 说明               |
| -------- | -------------- | ---- | -------------------- | ------------------ |
| 菜单栏   | `QMenuBar`     | 一个 | `setMenuBar()`       | 顶部，包含下拉菜单 |
| 工具栏   | `QToolBar`     | 多个 | `addToolBar()`       | 可拖动、浮动、停靠 |
| 状态栏   | `QStatusBar`   | 一个 | `setStatusBar()`     | 底部，显示状态信息 |
| 铆接部件 | `QDockWidget`  | 多个 | `addDockWidget()`    | 浮动窗口，可停靠   |
| 中心部件 | `QWidget` 子类 | 一个 | `setCentralWidget()` | 中央主要工作区     |

**记忆口诀**：
- **set** 开头的 → 只有一个（菜单栏、状态栏、中心部件）
- **add** 开头的 → 可以有多个（工具栏、铆接部件）
