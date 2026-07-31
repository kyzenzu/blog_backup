---
title: 【Qt】QPainter小记
date: 2026-06-30 14:15:14
tags:	
  - C++
  - Qt
---

1. Label::update槽会触发label的Paint事件。QApplication就会调用Label::event()函数
2. 由于label安装了MainWindow的事件过滤器
3. 因此触发了Paint事件的label，在Label::event()函数中会先调用MainWindow::eventFilter()
4. Label::event()处理好eventFilter后会继续调用真正的事件处理函数paintEvent()
5. 又由于绘制引擎只有在Label::event()中准备好,所以不能在自己写的函数中直接创建QPainter 

~~~cpp
bool QWidget::event(QEvent *event)
{
    /* 
    	准备好绘制引擎 
    */
    
    /* 先查看过滤器 */
    for (filter : eventFilters)
    	if (filter->eventFilter(this, event))
        	return;  // 被拦截
    
	/* 再根据事件调用处理函数 */
    switch (event->type()) {
    case QEvent::MouseMove:
        mouseMoveEvent((QMouseEvent*)event); // 鼠标移动事件被分发
        break;
    case QEvent::MouseButtonPress:
        mousePressEvent((QMouseEvent*)event); // 鼠标按下事件被分发
        break;
    case QEvent::KeyPress:
        keyPressEvent((QKeyEvent*)event); // 键盘按下事件被分发
        break;
    case QEvent::Paint:
        paintEvent((QPaintEvent*)event); // 绘制事件被分发
        break;
    // ... 处理几十种其他事件类型 ...
    default:
        // 对于未处理的事件，调用父类 QObject::event
        return QObject::event(event);
    }
    return true;
}
~~~

