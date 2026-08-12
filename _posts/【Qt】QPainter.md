---
title: 【Qt】QPainter
date: 2026-08-09 16:06:28
tags:
  - C/C++
  - Qt
---

# Qt 中 QPainter 的完整绘制指南

## 一、设计动机：Qt 的绘制模型是什么？

### 1.1 绘制系统在 GUI 框架中的位置

Qt 的绘制系统基于 **Scenic（场景）** 和 **Retained Mode（保留模式）** 的混合设计：

- **保留模式**：窗口/控件维护自己的绘制状态，系统只在需要时重绘。
- **立即模式**：`QPainter` 提供类似"画布"的接口，开发者通过调用绘制函数立即将图形渲染到设备上。

Qt 选择**事件驱动 + 立即绘制**的模型，核心原因是：

- GUI 程序是**事件驱动**的，绘制请求作为事件放入队列，统一处理。
- 绘制引擎的初始化（如 OpenGL 上下文、窗口缓冲区）需要特定的时机，不能在任意时刻创建。

### 1.2 为什么不能在任意地方创建 QPainter？

`QPainter` 需要一个 **QPaintDevice**（如 `QWidget`、`QPixmap`）作为绘制目标。但这个设备**只有在绘制事件处理期间才处于就绪状态**。

原因如下：

1. **双缓冲机制**：Qt 使用双缓冲减少闪烁，绘制时先绘制到后台缓冲区，再交换到前台。后台缓冲区只在 `paintEvent` 期间分配。
2. **平台差异**：不同平台（Windows/X11/macOS）的绘制上下文创建时机不同，Qt 在 `QWidget::event()` 中统一处理。
3. **事件优先级**：多个 `update()` 可能被合并为一次绘制，如果提前创建 `QPainter` 会导致绘制内容被覆盖。

## 二、事件处理与绘制流程

### 2.1 核心流程说明

1. `Label::update` 槽会触发 Label 的 Paint 事件，QApplication 调用 `Label::event()` 函数
2. 由于 Label 安装了 MainWindow 的事件过滤器
3. 触发了 Paint 事件的 Label，在 `Label::event()` 函数中会先调用 `MainWindow::eventFilter()`
4. `Label::event()` 处理好 eventFilter 后会继续调用真正的事件处理函数 `paintEvent()`
5. 绘制引擎只有在 `Label::event()` 中准备好，所以不能在自己写的函数中直接创建 QPainter

### 2.2 QWidget::event() 源码解析

```cpp
bool QWidget::event(QEvent *event)
{
    /* 
        ① 准备好绘制引擎（QPaintDevice初始化）
        包括：分配后台缓冲区、设置OpenGL上下文等
    */
    
    /* ② 遍历事件过滤器列表 */
    for (filter : eventFilters)
        if (filter->eventFilter(this, event))
            return;  // 被拦截，不再继续处理
    
    /* ③ 根据事件类型调用对应的处理函数 */
    switch (event->type()) {
    case QEvent::MouseMove:
        mouseMoveEvent((QMouseEvent*)event);
        break;
    case QEvent::MouseButtonPress:
        mousePressEvent((QMouseEvent*)event);
        break;
    case QEvent::KeyPress:
        keyPressEvent((QKeyEvent*)event);
        break;
    case QEvent::Paint:
        paintEvent((QPaintEvent*)event);  // 绘制事件被分发
        break;
    // ... 处理几十种其他事件类型 ...
    default:
        return QObject::event(event);
    }
    return true;
}
```

## 三、核心类职责

| 类             | 职责                                            |
| :------------- | :---------------------------------------------- |
| `QWidget`      | 所有控件的基类，拥有 `event()` 作为事件入口     |
| `QPaintEvent`  | 绘制事件类型，包含需要绘制的区域                |
| `QPainter`     | 绘制工具，提供绘制图形、文本、图像的 API        |
| `QPaintDevice` | 绘制目标抽象，如 `QWidget`、`QPixmap`、`QImage` |

## 四、两种绘制方式对比

### 4.1 方式一：重写 `paintEvent()`（标准方式）

```cpp
class MyLabel : public QLabel {
protected:
    void paintEvent(QPaintEvent* event) override {
        QPainter painter(this);
        painter.drawText(10, 10, "Hello");
        QLabel::paintEvent(event);  // 如果需要调用父类绘制
    }
};
```

**适用场景**：自定义控件、继承现有控件添加额外绘制。

### 4.2 方式二：使用 `eventFilter()` 拦截 Paint 事件

```cpp
bool MainWindow::eventFilter(QObject* watched, QEvent* event) {
    if (watched == ui->labelSunRiseAndSet && event->type() == QEvent::Paint) {
        QPainter painter(ui->labelSunRiseAndSet);
        // 绘制内容
        return true;  // 拦截，不再调用 label 自身的 paintEvent
    }
    return QWidget::eventFilter(watched, event);
}
```

**优点**：
- 无需继承 `QLabel`，减少类数量。
- 可为多个控件统一管理绘制逻辑。
- 可以**完全替换**控件的默认绘制。

**缺点**：
- 失去了父类已有的绘制功能（如背景、文本等），需全部重新实现。

## 五、绘制流程逐步解析

### 5.1 初始化时安装事件过滤器

```cpp
ui->labelSunRiseAndSet->installEventFilter(this);
ui->labelTempCurve->installEventFilter(this);
```

**为什么需要 `installEventFilter`？**
- 只有安装了过滤器，控件的 `event()` 才会在绘制前调用 `MainWindow::eventFilter()`。
- 一个对象可以安装多个过滤器，按添加顺序依次调用。

### 5.2 定时器触发更新

```cpp
sunTimer = new QTimer(ui->labelSunRiseAndSet);
connect(sunTimer, SIGNAL(timeout()), ui->labelSunRiseAndSet, SLOT(update()));
sunTimer->start(1000);
```

**为什么用 `update()` 而非直接调用绘制函数？**
- `update()` 是异步的，会合并连续的重绘请求，避免频繁刷新导致性能问题。
- 直接调用绘制函数会破坏 Qt 的绘制事件机制，可能导致绘制内容被下一次事件覆盖。

### 5.3 在 `eventFilter()` 中触发绘制

```cpp
ui->labelSunRiseAndSet->installEventFilter(this);
ui->labelTempCurve->installEventFilter(this);

bool MainWindow::eventFilter(QObject *watched, QEvent *event) {
    if (watched == ui->labelSunRiseAndSet && event->type() == QEvent::Paint) {
        drawSunRiseAndSet();   // 在事件处理中创建 QPainter
    }
    if (watched == ui->labelTempCurve && event->type() == QEvent::Paint) {
        drawTempCurve();
    }
    return QWidget::eventFilter(watched, event);
}
```

**关键点**：

- ⭐`MainWindow::eventFilter ` 被安装在 `labelSunRiseAndSet` 和 `labelTempCurve` 的事件过滤器中。这两个 `Label` 的事件一旦被触发，就会调用 `MainWindow::eventFilter ` 这里来进行分流处理。
- `eventFilter` 在 `QWidget::event()` **内部**被调用，此时绘制引擎已准备完毕。
- 返回 `true` 表示事件被拦截，后续不会再调用 `paintEvent()`。
- 如果返回 `false`，则继续调用 `paintEvent()`。

### 5.4 绘制函数的实现结构

```cpp
void MainWindow::drawSunRiseAndSet() {
    QPainter painter(ui->labelSunRiseAndSet);
    painter.setRenderHint(QPainter::Antialiasing, true);

    // ① 保存当前画笔状态
    painter.save();
    // 修改画笔...
    painter.drawLine(...);
    painter.restore();

    // ② 再次保存/恢复，绘制文字
    painter.save();
    painter.setFont(...);
    painter.drawText(...);
    painter.restore();

    // ③ 绘制圆弧，使用笔刷填充
    painter.setPen(Qt::NoPen);
    painter.setBrush(QColor(...));
    painter.drawPie(...);
}
```

**为什么频繁使用 `save()` / `restore()`？**
- `QPainter` 的状态（画笔、画刷、字体、变换矩阵等）是全局的。
- 每次 `save()` 将当前状态压栈，`restore()` 恢复，避免不同绘制模块相互影响。
- 这是一种**防御性编程**，提高代码的可维护性。

## 六、绘制坐标系与变换

### 6.1 坐标原点

`QPainter` 的默认原点在绘制设备的**左上角**，x 轴向右，y 轴向下。

示例代码中的硬编码坐标：
```cpp
const QPoint MainWindow::sun[2] = {
    QPoint(20, 75),
    QPoint(130, 75)
};
```
这些坐标是相对于 `ui->labelSunRiseAndSet` 左上角的。

### 6.2 圆弧与角度

```cpp
painter.drawArc(rect[0], 0 * 16, 180 * 16);
```

**Qt 的角度单位为什么是 1/16 度？**
- 这是为了兼容早期图形系统的整数运算精度。
- `0 * 16` 表示 0 度，`180 * 16` 表示 180 度。
- 逆时针为正方向

**Qt中圆弧的绘制思路**：

- 先确定一个起始角度，再确定角度的范围
- 从起始角度开始绘制，按照角度的范围画出一段圆弧
- ⭐这与数学中在 x-y 坐标系里绘制圆弧的思路一样。从 0 点出发，往 x 轴正方向引一条射线，逆时针为正角度方向，然后定角度绘制弧线

### 6.3 动态计算绘制参数

```cpp
int sunrise = sunriseHour.toInt() * 60 + sunriseMinute.toInt();
int sunset = sunsetHour.toInt() * 60 + sunsetMinute.toInt();
int now = QTime::currentTime().hour() * 60 + QTime::currentTime().minute();

startAngle = ((double)(sunset - now) / (sunset - sunrise)) * 180 * 16;
spanAngle = ((double)(now - sunrise) / (sunset - sunrise)) * 180 * 16;
```

这里用时间比例动态计算饼图的起止角度，实现了日出日落进度可视化。

## 七、常见陷阱与建议

| 陷阱                            | 说明                         | 解决方案                                             |
| :------------------------------ | :--------------------------- | :--------------------------------------------------- |
| 在构造函数中创建 `QPainter`     | 设备未初始化，绘制无效       | 必须在 `paintEvent` 或 `eventFilter` 中创建          |
| 忘记调用 `save()` / `restore()` | 状态污染导致后续绘制异常     | 每次修改状态前 `save()`，修改后 `restore()`          |
| 未检查坐标范围                  | 绘制超出设备区域可能导致裁剪 | 使用 `painter.viewport()` 和 `painter.window()` 控制 |
| 频繁 `update()` 导致性能下降    | 每帧都触发重绘               | 使用定时器控制刷新频率，或合并绘制请求               |
| 在 `eventFilter` 中返回 `false` | 会同时调用父类绘制，造成重叠 | 明确需要拦截时返回 `true`                            |

## 八、总结：QPainter 的核心原则

1. **只能在 Paint 事件中创建 `QPainter`** → 保证绘制引擎已初始化。
2. **用 `update()` 触发绘制** → 利用事件队列的合并机制提高性能。
3. **用 `save()` / `restore()` 隔离状态** → 防止绘制模块相互干扰。
4. **坐标系以绘制设备左上角为原点** → 布局时始终基于控件的相对位置。
5. **事件过滤器优先于 `paintEvent()`** → 提供一种无需继承的绘制扩展方式。
