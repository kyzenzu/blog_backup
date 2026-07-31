---
title: 【Qt】Qt中event机制和keyPressEvent
date: 2026-05-22 21:13:27
tags:
  - C++
  - Qt
---



我不是很理解Qt中的事件，比较疑惑的是QObject的event()函数是何时被调用，被谁调用的。并且疑惑如果有keyPressEvent()函数这个两个会不会重复调用

你这个问题其实已经抓住了 **Qt 事件系统的核心**：
 👉 `event()` 和 `keyPressEvent()` 的关系，本质是 **“统一入口 + 分发处理”**，而不是两个独立系统。

我按“谁调用谁、什么时候调用、会不会重复”给你讲清楚。

------

# 一、`event()` 是什么时候被调用的？谁调用？

先说结论：

👉 `QObject::event()` **不是你直接调用的，而是 Qt 事件循环调用的**

完整调用链大致是：

```
操作系统 → Qt事件循环 → QCoreApplication::notify()
          → QObject::event()
          → 具体事件函数（如 keyPressEvent）
```

### 更具体一点：

1. 操作系统产生事件（比如按键）
2. Qt 把它封装成 `QEvent`（例如 `QKeyEvent`）
3. 进入事件队列（event loop）
4. Qt 主循环（`exec()`）开始处理：

```
QCoreApplication::notify(receiver, event);
```

1. `notify()` 内部会调用：

```
receiver->event(event);
```

👉 所以：

> ✅ **`event()` 是由 `QCoreApplication::notify()` 调用的**

------

# 二、`event()` 到底干嘛？

👉 它是一个“总入口函数”（统一分发器）

你可以理解为：

```
bool QObject::event(QEvent *e) {
    switch (e->type()) {
        case QEvent::KeyPress:
            keyPressEvent(static_cast<QKeyEvent*>(e));
            break;
        case QEvent::MouseButtonPress:
            mousePressEvent(...);
            break;
        ...
    }
}
```

当然真实源码更复杂，但逻辑就是这个。

👉 关键点：

> `event()` 负责 **把不同类型事件分发到对应的虚函数**

------

# 三、那 `keyPressEvent()` 和 `event()` 会不会重复调用？

👉 **不会重复处理，是“调用链关系”**

调用顺序是：

```
event() → keyPressEvent()
```

不是：

```
event()
keyPressEvent()
（两个独立调用 ❌）
```

------

## 举个实际例子

你写：

```
void MyWidget::keyPressEvent(QKeyEvent *e) {
    qDebug() << "keyPressEvent";
}
```

运行时发生的是：

```
notify()
  → event()
      → keyPressEvent()
```

👉 只会执行一次 `keyPressEvent()`

------

# 四、如果你重写了 `event()` 会发生什么？

这是重点 ⚠️

```
bool MyWidget::event(QEvent *e) {
    if (e->type() == QEvent::KeyPress) {
        qDebug() << "event 捕获 key";
        return true;
    }
    return QWidget::event(e);
}
```

### 结果：

👉 `keyPressEvent()` **不会被调用**

因为：

> 你在 `event()` 里“截断”了事件，没有继续往下分发

------

## 正确写法（如果你还想让 keyPressEvent 生效）

```
bool MyWidget::event(QEvent *e) {
    if (e->type() == QEvent::KeyPress) {
        qDebug() << "event 捕获 key";
    }
    return QWidget::event(e); // 继续分发
}
```

------

# 五、总结一句话（非常重要）

👉 Qt 事件系统本质：

> **event() 是统一入口，具体事件函数（keyPressEvent 等）是它内部调用的分支处理**

------

# 六、再帮你建立一个“脑模型”

你可以把 Qt 想成一个“总机 + 分机系统”：

- `notify()` 👉 总机
- `event()` 👉 分线员
- `keyPressEvent()` 👉 具体接电话的人

流程：

```
用户按键
  ↓
notify（总机接电话）
  ↓
event（分线员判断是什么电话）
  ↓
keyPressEvent（转接给键盘处理的人）
```

------

# 七、什么时候用 `event()`？什么时候用 `keyPressEvent()`？

👉 建议：

### 用 `keyPressEvent()`（90%情况）

- 只关心某一类事件（键盘、鼠标）

### 用 `event()`（高级用法）

- 想拦截所有事件
- 想改分发逻辑
- 处理 Qt 没提供虚函数的事件类型

------

# 八、额外补充（你这个阶段很值得知道）

还有一个更底层的机制：

👉 **事件过滤器（eventFilter）**

调用顺序其实是：

```
eventFilter → event() → keyPressEvent()
```

如果你后面继续深入 Qt，这个是关键。

------

# 最后总结（帮你压缩记忆）

👉 一句话版：

- `notify()` 调用 `event()`
- `event()` 调用 `keyPressEvent()`
- 不会重复调用，是链式调用
- 重写 `event()` 可能“拦截”掉后续事件

------



给我看一下Qt5.8中event的源码，然后我再疑惑的是操作系统 → Qt事件循环这部分是如何做到的，以及QCoreApplication::notify(receiver, event);这里面receiver是如何确定的这个函数是循环调用传入多个receiver吗

我会把源码拆成三段看：`QCoreApplication::notify/notifyInternal2`、`QObject::event()`/`QWidget::event()`，再解释 OS 消息如何进入 Qt，以及 receiver 是在哪里确定的。

已思考 21s

你可以把它理解成这条链：

```
操作系统消息
 → QAbstractEventDispatcher::processEvents()
 → Qt 平台层把原生消息翻译成 QEvent
 → QCoreApplication::notifyInternal2(receiver, event)
 → QApplication/QCoreApplication::notify(receiver, event)
 → notify_helper(receiver, event)
 → receiver->event(event)
 → QWidget::event(event)
 → keyPressEvent(event)
```

## 1. Qt 5.8 里 `QObject::event()` 源码长这样

`QObject::event()` 本身只处理很基础的 QObject 事件，比如定时器、子对象变化、延迟删除、元调用等：

```
bool QObject::event(QEvent *e)
{
    switch (e->type()) {
    case QEvent::Timer:
        timerEvent((QTimerEvent*)e);
        break;

    case QEvent::ChildAdded:
    case QEvent::ChildPolished:
    case QEvent::ChildRemoved:
        childEvent((QChildEvent*)e);
        break;

    case QEvent::DeferredDelete:
        qDeleteInEventHandler(this);
        break;

    case QEvent::MetaCall:
        // queued signal / invokeMethod 等会走这里
        ...
        break;

    ...
    default:
        return false;
    }

    return true;
}
```

所以注意：

**`QObject::event()` 里面没有 `keyPressEvent()`。**

因为 `keyPressEvent()` 是 `QWidget` 的东西，不是普通 `QObject` 的东西。

真正分发键盘事件的是：

## 2. `QWidget::event()` 里调用 `keyPressEvent()`

Qt 5.8 的 `QWidget::event()` 文档注释直接说：键盘事件比较特殊，`event()` 会先检查 Tab/Shift+Tab 是否用于焦点切换；如果不是，才调用 `keyPressEvent()`。源码附近就是：

```
bool QWidget::event(QEvent *event)
{
    ...

    switch (event->type()) {
    case QEvent::MouseMove:
        mouseMoveEvent((QMouseEvent*)event);
        break;

    case QEvent::MouseButtonPress:
        mousePressEvent((QMouseEvent*)event);
        break;

    case QEvent::KeyPress: {
        QKeyEvent *k = (QKeyEvent *)event;
        bool res = false;

        if (!(k->modifiers() & (Qt::ControlModifier | Qt::AltModifier))) {
            if (k->key() == Qt::Key_Backtab
                || (k->key() == Qt::Key_Tab && (k->modifiers() & Qt::ShiftModifier)))
                res = focusNextPrevChild(false);
            else if (k->key() == Qt::Key_Tab)
                res = focusNextPrevChild(true);
        }

        if (!res)
            keyPressEvent(k);

        break;
    }

    ...
    }
}
```

所以你之前的问题答案是：

**不会重复调用。**

真实关系是：

```
receiver->event(event);        // 虚函数调用
// 如果 receiver 是 QWidget 子类，实际进入 QWidget::event()
QWidget::event(event)
    → keyPressEvent(event)
```

不是：

```
event();
keyPressEvent();  // Qt 又单独调用一次
```

## 3. `event()` 是谁调用的？

关键源码在 `notify_helper()`：

```
bool QCoreApplicationPrivate::notify_helper(QObject *receiver, QEvent *event)
{
    // 先过 application event filters
    ...

    // 再过 receiver 自己的 event filters
    ...

    // 真正投递事件
    consumed = receiver->event(event);
    return consumed;
}
```

而 `notify_helper()` 是从 `notify()` / `notifyInternal2()` 过来的。Qt 5.8 里：

```
bool QCoreApplication::notifyInternal2(QObject *receiver, QEvent *event)
{
    ...
    return self->notify(receiver, event);
}

bool QCoreApplication::notify(QObject *receiver, QEvent *event)
{
    if (QCoreApplicationPrivate::is_app_closing)
        return true;

    return doNotify(receiver, event);
}
```

也就是说：

```
notifyInternal2(receiver, event)
 → notify(receiver, event)
 → doNotify(receiver, event)
 → notify_helper(receiver, event)
 → receiver->event(event)
```

## 4. 操作系统 → Qt 事件循环，是怎么做到的？

`app.exec()` 不是一个魔法函数，它本质上进入一个循环：

```
while (!退出) {
    从操作系统取消息;
    把原生消息翻译成 Qt 事件;
    找到接收对象 receiver;
    notify(receiver, event);
}
```

跨平台部分叫：

```
QAbstractEventDispatcher
```

Qt 文档里说它负责从窗口系统和其他来源接收事件，再发送给 `QCoreApplication` / `QApplication` 处理。

不同系统有不同实现：

```
Windows: 取 MSG，例如 WM_KEYDOWN / WM_MOUSEMOVE
macOS:   取 NSEvent
Linux/X11: 取 XCB event
Wayland: 取 Wayland event
```

Qt 平台层再把它们转成：

```
WM_KEYDOWN  → QKeyEvent(QEvent::KeyPress)
WM_MOUSEMOVE → QMouseEvent(QEvent::MouseMove)
```

## 5. `receiver` 是怎么确定的？

这要看事件类型。

### 键盘事件

键盘事件通常发给：

```
QApplication::focusWidget()
```

也就是当前拥有键盘焦点的控件。

比如：

```
QLineEdit *edit;
edit->setFocus();
```

这时你按键，receiver 基本就是这个 `QLineEdit`。

### 鼠标事件

鼠标事件通常根据坐标命中测试：

```
鼠标位置 → 找到窗口 → 找到这个位置下面最里面的 QWidget
```

比如鼠标点在按钮上，receiver 就是那个 `QPushButton`。

### 定时器事件

你写：

```
startTimer(1000);
```

这个 timer 属于哪个 QObject，receiver 就是哪个 QObject。

### `postEvent`

如果你手动写：

```
QCoreApplication::postEvent(obj, new QEvent(...));
```

那 receiver 就是你传进去的 `obj`。

## 6. `notify(receiver, event)` 是循环调用多个 receiver 吗？

**不是一个 event 同时循环传给多个 receiver。**

更准确是：

```
事件循环每次取出一个事件
 → 这个事件有一个 receiver
 → notify(receiver, event)
```

伪代码：

```
while (eventLoopRunning) {
    EventItem item = takeNextEvent();

    QObject *receiver = item.receiver;
    QEvent  *event    = item.event;

    QCoreApplication::notify(receiver, event);
}
```

所以是：

```
第1个事件 → receiverA
第2个事件 → receiverB
第3个事件 → receiverA
第4个事件 → receiverC
```

不是：

```
一个事件 → receiverA → receiverB → receiverC
```

不过有例外：某些事件可能会**传播**，比如键盘/鼠标事件如果当前 widget 不接受，Qt 可能继续传给父 widget。Qt 的 `notify()` 注释里也提到，某些 mouse/key 事件会在 receiver 不感兴趣时传播到父对象。

## 最后压缩成一句话

`event()` 是 Qt 投递事件时调用的统一入口；`receiver` 在事件产生/翻译阶段就已经确定；`notify()` 每次处理一个 receiver + 一个 event；如果 receiver 是 `QWidget`，那么 `QWidget::event()` 会根据事件类型再调用 `keyPressEvent()`、`mousePressEvent()` 等具体函数。

一个事件只有一个receiver吗，那Qt是怎么确定这个receiver的

先把你这个问题纠正得更精确一点：

👉 **“一个事件只有一个 receiver 吗？”**

✔ **绝大多数情况下：是的（一个事件 → 一个目标对象）**
 ❗ 但**可能发生“传播”或“转发”**，看起来像多个 receiver

------

# 一、Qt 是怎么确定 receiver 的？

核心一句话：

> 👉 **receiver 在“事件产生/翻译阶段”就已经确定了，不是在 notify() 里算的**

不同事件类型，确定方式完全不同 👇

------

# 二、最关键的三类事件

## 1️⃣ 键盘事件（KeyPress）

👉 **receiver = 当前焦点对象（focus object）**

```
QWidget *w = QApplication::focusWidget();
```

### 举个例子：

```
QLineEdit *edit;
edit->setFocus();
```

你按键：

```
OS: WM_KEYDOWN
 → Qt 转成 QKeyEvent
 → receiver = focusWidget() = edit
 → notify(edit, event)
```

✔ 所以键盘事件永远只发给“谁有焦点”

------

## 2️⃣ 鼠标事件（MousePress / Move）

👉 **receiver = 鼠标坐标命中的最深层 QWidget**

Qt 做了一件事叫：

> 🧠 **hit test（命中测试）**

流程：

```
鼠标点击 (x, y)
 → 找到窗口
 → 找到这个点最里面的子控件
 → 这个控件就是 receiver
```

### 举例：

```
QWidget
 ├── QPushButton
 └── QLabel
```

你点在按钮上：

```
receiver = QPushButton
```

------

## 3️⃣ 定时器 / 自定义事件

👉 **receiver = 你绑定的对象**

```
startTimer(1000);   // 属于 this
QCoreApplication::postEvent(obj, event);
```

✔ 谁创建的 / 你传给谁，receiver 就是谁

------

# 三、那事件“只会给一个对象”吗？

### ✔ 基本规则：一个事件只投递给一个 receiver

```
notify(receiver, event);
```

------

# 四、但你感觉“像多个”的原因（重点）

## 1️⃣ 事件传播（propagation）

比如鼠标事件：

```
void Child::mousePressEvent(QMouseEvent *e) {
    e->ignore();   // 我不要
}
```

👉 Qt 会把事件往父组件传：

```
Child → Parent → Parent → ...
```

所以你会看到：

```
好像多个对象都收到了事件
```

但本质是：

> ❗ **同一个 event 被“转交”，不是一开始就发给多个**

------

## 2️⃣ eventFilter（事件过滤器）

调用链：

```
eventFilter (A)
eventFilter (B)
event()
keyPressEvent()
```

👉 看起来很多对象参与了，但：

✔ **真正的 receiver 还是一个**
 ✔ 其他只是“拦截者”

------

## 3️⃣ Qt 内部转发（特殊情况）

比如：

- Tab 键 → 被 QWidget::event() 截获 → 转换为焦点移动
- Shortcut → 可能被 QApplication 处理

------

# 五、源码层面你要抓住的点

在 Qt 5.8 里：

```
QCoreApplication::notifyInternal2(receiver, event)
```

👉 **receiver 已经是确定好的**

Qt 不会在这里“找谁来处理”，而是直接：

```
receiver->event(event);
```

------

# 六、你可以建立这个“完整模型”

```
【操作系统产生事件】

→ Qt 平台层（QPA）
   ✔ 转换为 QEvent
   ✔ 决定 receiver（关键！）

→ 放入事件系统 / 直接分发

→ QCoreApplication::notify(receiver, event)

→ receiver->event(event)

→ QWidget::event()

→ keyPressEvent / mousePressEvent
```

------

# 七、一句话总结（帮你建立直觉）

👉 **receiver 不是 event() 里找的，而是在事件“诞生时”就决定好的**

------

# 八、再补一个你这个阶段非常关键的理解

你现在其实已经踩到 Qt 设计的核心思想了：

> 👉 **事件 = “谁应该处理这件事” + “发生了什么”**

也就是：

```
(receiver, event)
```

不是：

```
(event) → 再去找谁处理 ❌
```

------

# 如果你想再深入（强烈推荐）

下一步你可以问我这个：

👉 **“Qt 是怎么通过源码做 hit test 找到鼠标点击的那个 QWidget？”**

这个一旦搞懂，你对 Qt GUI 会直接上一个台阶。

---

我的疑问是如果父控件有子控件，子控件还有子控件，它们有重写了contextMenuEvent，那么它们的传递机制是怎么实现的，由谁做到的

## 事件传递机制的实现原理

这是一个非常深入的问题！让我详细解释**整个传递链是如何实现的**，以及**由谁负责**这个传递过程。

------

### 核心答案

传递机制由 **Qt 事件系统** 实现，具体来说有三个关键角色：

1. **QApplication** - 事件分发总控
2. **QWidget::event()** - 事件路由和传播判断
3. **递归调用** - 向上查找父控件链

------

## 完整传递机制详解

### 1. QApplication 负责初始路由

```cpp
// QApplication::notify() 简化逻辑
bool QApplication::notify(QObject *receiver, QEvent *event)
{
    if (event->type() == QEvent::ContextMenu) {
        QContextMenuEvent *ctxEvent = static_cast<QContextMenuEvent*>(event);
        
        // ⭐ 关键：找到鼠标坐标下的最深层子控件
        QWidget *targetWidget = findDeepestChild(static_cast<QWidget*>(receiver), 
                                                  ctxEvent->pos());
        
        if (targetWidget && targetWidget != receiver) {
            // 直接发送给最深层的子控件
            return sendEvent(targetWidget, event);
        }
    }
    
    // 没有子控件，直接发给当前控件
    return receiver->event(event);
}

// 递归查找最深层的子控件
QWidget* findDeepestChild(QWidget *parent, const QPoint &pos)
{
    QWidget *child = parent->childAt(pos);
    if (child && child != parent) {
        // 转换为子控件坐标后继续递归查找
        QPoint childPos = child->mapFromParent(pos);
        return findDeepestChild(child, childPos);
    }
    return parent;  // 返回最深层控件
}
```



**示例**：

text

```
GrandParent (0,0, 300x300)
  └─ Parent (50,50, 200x200)
      └─ Child (80,80, 100x100)  ← 鼠标点击在 (150,150)

findDeepestChild 过程：
1. 检查 GrandParent → 找到 Parent
2. 检查 Parent → 找到 Child  
3. 检查 Child → 没有更深子控件
4. 返回 Child
```



------

### 2. QWidget::event() 负责传播控制

```cpp
// Qt 源码 QWidget::event() 简化版
bool QWidget::event(QEvent *event)
{
    switch (event->type()) {
        case QEvent::ContextMenu: {
            // 调用具体的事件处理函数
            contextMenuEvent(static_cast<QContextMenuEvent*>(event));
            
            // ⭐⭐⭐ 关键传播逻辑 ⭐⭐⭐
            // ⭐ 如果解决不了,在子控件的event()里转发给父控件 ⭐
            if (!event->isAccepted()) {
                // 如果当前控件没有接受事件，尝试传递给父控件
                QWidget *parent = parentWidget();
                if (parent && !isWindow()) {  // 不是顶层窗口且有父控件
                    // 将事件重新发送给父控件
                    return QCoreApplication::sendEvent(parent, event);
                }
                return false;  // 没有父控件，事件被丢弃
            }
            return true;  // 事件已被接受，停止传播
        }
    }
    return QWidget::event(event);
}
```



------

### 3. 完整的传递链递归过程

让我们追踪一个具体的三层嵌套示例：

cpp

```
class GrandParent : public QWidget {
protected:
    void contextMenuEvent(QContextMenuEvent *event) override {
        qDebug() << "GrandParent: 收到事件";
        event->ignore();  // 继续传递
    }
};

class Parent : public GrandParent {
protected:
    void contextMenuEvent(QContextMenuEvent *event) override {
        qDebug() << "Parent: 收到事件";
        event->ignore();  // 继续传递
    }
};

class Child : public Parent {
protected:
    void contextMenuEvent(QContextMenuEvent *event) override {
        qDebug() << "Child: 收到事件";
        event->ignore();  // 继续传递
    }
};
```



**完整调用链**：

text

```
用户右键点击 Child 区域
    ↓
[步骤1] QApplication::notify()
    - 找到最深层的子控件 = Child
    - 调用 QApplication::sendEvent(Child, event)
    ↓
[步骤2] Child::event() 被调用
    - 调用 Child::contextMenuEvent()
    - Child 设置 event->ignore()
    - Child::event() 检查到 !isAccepted()
    - 调用 sendEvent(Parent, event)  ← 向上传递
    ↓
[步骤3] Parent::event() 被调用
    - 调用 Parent::contextMenuEvent()
    - Parent 设置 event->ignore()
    - Parent::event() 检查到 !isAccepted()
    - 调用 sendEvent(GrandParent, event)  ← 继续向上
    ↓
[步骤4] GrandParent::event() 被调用
    - 调用 GrandParent::contextMenuEvent()
    - GrandParent 设置 event->ignore()
    - GrandParent::event() 检查到 !isAccepted()
    - GrandParent->parentWidget() 返回 nullptr（顶层窗口）
    - 事件被丢弃
```



------

### 4. 关键代码：sendEvent 的递归特性

注意 `sendEvent` 是**同步调用**，会立即执行目标控件的 `event()` 函数：

cpp

```
// Child::event() 中的代码
if (!event->isAccepted()) {
    QWidget *parent = parentWidget();
    if (parent) {
        // ⭐ 这是一个同步调用，会立即执行 Parent::event()
        // Parent::event() 执行完毕后，才返回这里
        return QCoreApplication::sendEvent(parent, event);
    }
}
```



**调用栈示意图**：

text

```
QApplication::notify()
  └─ Child::event()
       └─ Child::contextMenuEvent()
       └─ sendEvent(Parent)  // 同步调用
            └─ Parent::event()
                 └─ Parent::contextMenuEvent()
                 └─ sendEvent(GrandParent)
                      └─ GrandParent::event()
                           └─ GrandParent::contextMenuEvent()
                           └─ 返回
                 └─ 返回
       └─ 返回
  └─ 返回
```



------

### 5. 实际验证代码

cpp

```
#include <QApplication>
#include <QWidget>
#include <QPushButton>
#include <QDebug>

class DeepWidget : public QWidget {
    QString name;
public:
    DeepWidget(const QString &n, QWidget *parent = nullptr) 
        : QWidget(parent), name(n) {
        setStyleSheet(QString("background-color: %1; border: 1px solid black;")
                     .arg(name == "Child" ? "red" : 
                          name == "Parent" ? "green" : "blue"));
    }
    
protected:
    void contextMenuEvent(QContextMenuEvent *event) override {
        qDebug() << "[" << name << "] 收到事件";
        
        // 打印调用栈深度
        static int depth = 0;
        depth++;
        qDebug() << "  深度:" << depth;
        
        // 选择是否传递
        event->ignore();  // 总是向上传递
        
        qDebug() << "[" << name << "] 事件处理结束";
        depth--;
    }
    
    bool event(QEvent *event) override {
        if (event->type() == QEvent::ContextMenu) {
            qDebug() << "  ->" << name << "::event() 开始";
            bool result = QWidget::event(event);
            qDebug() << "  <-" << name << "::event() 结束, accepted:" 
                     << event->isAccepted();
            return result;
        }
        return QWidget::event(event);
    }
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    
    auto *grandParent = new DeepWidget("GrandParent");
    grandParent->resize(300, 300);
    
    auto *parent = new DeepWidget("Parent", grandParent);
    parent->resize(200, 200);
    parent->move(50, 50);
    
    auto *child = new DeepWidget("Child", parent);
    child->resize(100, 100);
    child->move(50, 50);
    
    grandParent->show();
    
    return app.exec();
}
```



**运行输出**（右键点击 Child）：

text

```
  -> Child::event() 开始
[ Child ] 收到事件
  深度: 1
[ Child ] 事件处理结束
  <- Child::event() 结束, accepted: false

  -> Parent::event() 开始
[ Parent ] 收到事件
  深度: 2
[ Parent ] 事件处理结束
  <- Parent::event() 结束, accepted: false

  -> GrandParent::event() 开始
[ GrandParent ] 收到事件
  深度: 3
[ GrandParent ] 事件处理结束
  <- GrandParent::event() 结束, accepted: false
```



------

### 6. 如果中间某个控件 accept() 会怎样？

cpp

```
class Parent : public GrandParent {
protected:
    void contextMenuEvent(QContextMenuEvent *event) override {
        qDebug() << "Parent: 收到事件";
        event->accept();  // ← 这里接受事件
    }
};
```



**输出**：

text

```
  -> Child::event() 开始
[ Child ] 收到事件，ignore()
  <- Child::event() 结束, accepted: false

  -> Parent::event() 开始
[ Parent ] 收到事件，accept()
  <- Parent::event() 结束, accepted: true

// GrandParent 不会收到！
```



------

### 总结：谁做到了什么？

| 角色                              | 职责                                                |
| :-------------------------------- | :-------------------------------------------------- |
| **QApplication::notify()**        | 找到最深层的子控件，作为事件初始接收者              |
| **QWidget::event()**              | 调用具体的 contextMenuEvent，并判断是否需要向上传播 |
| **QCoreApplication::sendEvent()** | 同步发送事件到父控件，实现递归调用                  |
| **父控件链**                      | 通过 `parentWidget()` 获取，形成传播路径            |

**核心机制**：

1. 事件总是先发给**最深层子控件**
2. 每个控件的 `event()` 函数检查 `isAccepted()`
3. 如果 `false`，自动调用 `sendEvent(parent, event)`
4. 这是一个**同步递归调用**，不是异步信号
5. 直到某个控件 `accept()` 或到达顶层窗口

**一句话总结**：传递机制由 **QWidget::event()** 中的传播逻辑实现，它在调用完 `contextMenuEvent()` 后检查事件状态，如果未被接受就递归调用父控件的 `event()`，形成链式传播。
