---
title: 【Windows编程】线程同步——事件对象Event
date: 2026-07-27 11:32:57
tags:
  - C/C++
  - Windows编程
---

## Windows 事件对象（Event）笔记

### 1. 基本概念

事件对象也属于内核对象，它包含以下三个成员：

- 使用计数
- 用于指明该事件是**自动重置**还是**人工重置**的布尔值
- 用于指明该事件处于**已通知状态（有信号）**还是**未通知状态（无信号）**的布尔值

### 2. 两种事件类型对比

| 类型                                     | 行为                                                         | 适用场景                                             |
| :--------------------------------------- | :----------------------------------------------------------- | :--------------------------------------------------- |
| **人工重置事件**（bManualReset = TRUE）  | 事件变为有信号状态时，**所有**等待该事件的线程均变为可调度线程 | 多个线程同时响应同一个事件（如通知所有工作线程退出） |
| **自动重置事件**（bManualReset = FALSE） | 事件变为有信号状态时，**只有一个**等待线程变为可调度线程（系统自动将其重置为无信号） | 资源互斥访问、线程间一对一同步                       |

### 3. 核心 API

#### 3.1 创建事件：`CreateEventW`

```cpp
WINBASEAPI
    _Ret_maybenull_
    HANDLE
    WINAPI
    CreateEventW(
    _In_opt_ LPSECURITY_ATTRIBUTES lpEventAttributes, // 安全属性
    _In_ BOOL bManualReset, // 复位方式　　TRUE 必须用ResetEvent手动复原  FALSE 自动还原为无信号状态
    _In_ BOOL bInitialState, // 初始状态 　　TRUE 初始状态为有信号状态  FALSE 无信号状态
    _In_opt_ LPCWSTR lpName //对象名称 　NULL  无名的事件对象
);
```

#### 3.2 参数说明

| 参数名                | 含义                                                         |
| :-------------------- | :----------------------------------------------------------- |
| **lpEventAttributes** | 安全属性（NULL = 默认，句柄不可继承）                        |
| **bManualReset**      | `TRUE` = 人工重置，需手动调用 `ResetEvent` 恢复无信号；`FALSE` = 自动重置，等待成功后自动恢复无信号 |
| **bInitialState**     | `TRUE` = 初始为有信号状态；`FALSE` = 初始为无信号状态        |
| **lpName**            | 事件名称（NULL = 匿名事件）                                  |

### 4. 常用操作函数

| 函数                                   | 作用                                           |
| :------------------------------------- | :--------------------------------------------- |
| `SetEvent(hEvent)`                     | 将事件设为**有信号**状态                       |
| `ResetEvent(hEvent)`                   | 将事件设为**无信号**状态（仅用于人工重置事件） |
| `WaitForSingleObject(hEvent, timeout)` | 请求/等待事件变为有信号状态                    |

### 5. 代码示例

#### 5.1 示例一：人工重置事件实现线程同步

```cpp
/* 用事件对象实现同步 */

char str[100];
HANDLE hEvent;
unsigned WINAPI NumberOfA(void* arg) {
    WaitForSingleObject(hEvent, INFINITE);
    int cnt = 0;
    for (int i = 0; str[i] != 0; i++) {
        if (str[i] == 'A')
            cnt++;
    }
    printf("A的个数: %d\n", cnt);
    return 0;
}

unsigned WINAPI NumberOfOther(void* arg) {
    int cnt = 0;
    for (int i = 0; str[i] != 0; i++) {
        if (str[i] != 'A')
            cnt++;
    }
    printf("Other的个数: %d\n", cnt-1);
    SetEvent(hEvent);
    return 0;
}



int main() {
    fputs("Input: ", stdout);
    fgets(str, 100, stdin);
    WINBASEAPI
        _Ret_maybenull_
        HANDLE
        WINAPI
        CreateEventW(
        _In_opt_ LPSECURITY_ATTRIBUTES lpEventAttributes, // 安全属性
        _In_ BOOL bManualReset, // 复位方式　　TRUE 必须用ResetEvent手动复原  FALSE 自动还原为无信号状态
        _In_ BOOL bInitialState, // 初始状态 　　TRUE 初始状态为有信号状态  FALSE 无信号状态
        _In_opt_ LPCWSTR lpName //对象名称 　NULL  无名的事件对象
        );
    hEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

    HANDLE hThread1 = (HANDLE)_beginthreadex(NULL, 0, NumberOfA, NULL, 0, NULL);
    HANDLE hThread2 = (HANDLE)_beginthreadex(NULL, 0, NumberOfOther, NULL, 0, NULL);
    WaitForSingleObject(hThread1, INFINITE);
    WaitForSingleObject(hThread2, INFINITE);

    ResetEvent(hEvent);
    CloseHandle(hEvent);
}
```

#### 5.2 示例二：自动重置事件实现售票同步

```cpp
HANDLE g_hEvent;
int iTicket = 1000;
DWORD WINAPI SellTicketA(LPVOID arg) {
    while (TRUE) {
        WaitForSingleObject(g_hEvent, INFINITE);
        if (iTicket > 0) {
            iTicket--;
            printf("A remain %d\n", iTicket);
        } else {
            break;
        }
        SetEvent(g_hEvent);
    }
    return 0;
}

DWORD WINAPI SellTicketB(LPVOID arg) {
    while (TRUE) {
        WaitForSingleObject(g_hEvent, INFINITE);
        if (iTicket > 0) {
            iTicket--;
            printf("B remain %d\n", iTicket);
        } else {
            break;
        }
        SetEvent(g_hEvent);
    }
    return 0;
}

int main() {
    g_hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
    SetEvent(g_hEvent);
    HANDLE ThreadA = CreateThread(NULL, 0, SellTicketA, NULL, 0, NULL);
    HANDLE ThreadB = CreateThread(NULL, 0, SellTicketB, NULL, 0, NULL);

    WaitForSingleObject(ThreadA,INFINITE);
    WaitForSingleObject(ThreadB, INFINITE);

    CloseHandle(ThreadA);
    CloseHandle(ThreadB);
}
```

