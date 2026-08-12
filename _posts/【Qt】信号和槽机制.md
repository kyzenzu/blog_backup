```
title: 【Qt】信号和槽机制
date: 2026-08-09 10:09:20
tags:
  - C/C++
  - Qt
```

---

在 `signals:` 区域中声明的函数，**不需要用户提供实现**，Qt 的 **MOC（Meta Object Compiler）** 会在编译期自动生成对应的函数体。

---

## 1. MOC 生成的信号函数长什么样？

**示例声明：**

```cpp
class MyObject : public QObject
{
    Q_OBJECT

signals:
    void valueChanged(int value);
};
```

**Qt 5.8 中 MOC 生成的实现（简化）：**

```cpp
// SIGNAL 0
void MyObject::valueChanged(int _t1)
{
    void *_a[] = {
        nullptr,                                       // 返回值占位（信号无返回值）
        const_cast<void*>(reinterpret_cast<const void*>(&_t1))  // 参数打包
    };

    QMetaObject::activate(this, &staticMetaObject, 0, _a);
}
```

**核心本质：**

> 信号函数本身不执行任何业务逻辑，其唯一职责是**通知 Qt 元对象系统**——"某个信号被触发了"。

---

## 2. `emit` 关键字的作用

`emit` 在 Qt 中被定义为**空宏**，预处理阶段会被完全移除。

```cpp
emit valueChanged(42);   // 预处理后等价于
valueChanged(42);        // 普通的函数调用
```

`emit` 的作用仅在于**提升代码语义的可读性**，让"发射信号"这一行为在代码中更加自然直观。

---

## 3. `QMetaObject::activate` 的内部逻辑

```cpp
void QMetaObject::activate(
    QObject *sender,
    const QMetaObject *m,
    int local_signal_index,
    void **argv)
{
    // 1. 计算全局信号索引
    int signal_index = local_signal_index + signalOffset;

    // 2. 获取 sender 对象的私有数据
    QObjectPrivate *senderData = QObjectPrivate::get(sender);

    // 3. 根据信号索引获取对应的连接列表
    ConnectionList &list = senderData->connectionLists[signal_index];

    // 4. 遍历所有连接，调用对应的槽函数
    for (Connection *c : list)
    {
        if (c->connectionType == DirectConnection)
        {
            // 直接调用槽函数
            call slot;
        }
        else
        {
            // 通过事件队列异步调用
            queue event;
        }
    }
}
```

**执行流程：**

1. 根据信号索引，从 `sender` 对象中取出该信号对应的**连接列表**（`ConnectionList`）
2. 遍历列表中的每一个 `Connection` 节点
3. 根据连接类型（直连/队列连接），同步或异步地调用与之关联的槽函数

> 整个过程**精确到运行时的具体对象**，而非类级别。

---

## 4. 对象内部的数据结构

每个 `QObject` 实例内部都持有一个 `QObjectPrivate` 对象，其中存储着一个 `ConnectionLists[]` 数组：

```
QObject（某个具体对象）
└── QObjectPrivate
    └── connectionLists（QObjectConnectionListVector）
        ├── [0] ConnectionList  →  Connection → Connection → ...
        ├── [1] ConnectionList  →  Connection → ...
        ├── [2] ConnectionList  →  NULL
        ├── [3] ConnectionList  →  Connection → ...
        └── ...

```

- 每个 ConnectionList 对应一个信号索引（signal_index）
- 每个 Connection 是一个链表节点，记录一对 sender.signal → receiver.slot 的连接关系

---

## 5. `connect()` 做了什么？

`connect()` 函数的核心工作：

1. 将当前**对象**的某个 `signal` 与**目标对象**的某个 `slot` 组合成一个 `Connection` 节点
2. 将该节点插入到当前对象对应信号索引的 `ConnectionList` 中

这样，当 `activate()` 在运行时被调用时，就能根据当前对象及其信号索引，找到所有已连接的槽函数并逐一调用。

> ⭐ **关键点**：即使两个对象属于同一个类，它们各自内存中的 `ConnectionLists[]` 也是**完全独立**的。这正是 Qt 信号槽机制能够**精确到对象级别**的根本原因。

---

**示意图：**

![connect 连接示意图](../posts_img/【Qt】Qt初学的两个核心问题/connect.png)

---

