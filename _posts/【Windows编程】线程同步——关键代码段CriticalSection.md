---
title: 【Windows编程】线程同步——关键代码段CriticalSection
date: 2026-07-27 11:44:57
tags:
  - C/++
  - Windows编程
---

收到，整理后的关键代码段笔记如下（代码中所有原有注释完整保留）：

---

## Windows 关键代码段（Critical Section）笔记

### 1. 基本概念

关键代码段（也称为**临界区**）是 Windows 中最常用的线程同步方式，因为它**工作在用户模式**下，不涉及内核对象，因此使用效率更高。

关键代码段是指一小段代码，在代码能够执行前，它必须**独占**对某些资源的访问权。通常把多线程中访问同一种资源的那部分代码定义为关键代码段。

### 2. 核心 API

| 函数                        | 作用                                             |
| :-------------------------- | :----------------------------------------------- |
| `InitializeCriticalSection` | 初始化一个关键代码段                             |
| `EnterCriticalSection`      | 进入关键代码段（获得所有权，若被占用则阻塞等待） |
| `LeaveCriticalSection`      | 退出关键代码段（释放所有权）                     |
| `DeleteCriticalSection`     | 删除关键代码段（释放资源）                       |

#### 2.1 初始化关键代码段

```cpp
InitializeCriticalSection(
    _Out_ LPCRITICAL_SECTION lpCriticalSection
);
```

在调用 `InitializeCriticalSection` 之前，需要先构造一个 `CRITICAL_SECTION` 结构体对象，然后将该对象的地址传递给函数。

#### 2.2 进入关键代码段

```cpp
VOID
WINAPI
EnterCriticalSection(
    _Inout_ LPCRITICAL_SECTION lpCriticalSection
);
```

调用 `EnterCriticalSection` 以获得指定临界区对象的所有权。如果该所有权已赋予其他线程，则调用线程会一直等待，直到获得所有权为止。

#### 2.3 退出关键代码段

```cpp
VOID
WINAPI
LeaveCriticalSection(
    _Inout_ LPCRITICAL_SECTION lpCriticalSection
);
```

线程使用完临界区所保护的资源后，调用 `LeaveCriticalSection` 释放所有权。之后，其他等待该临界区的线程可以获得所有权并进入关键代码段。

#### 2.4 删除临界区

```cpp
WINBASEAPI
VOID
WINAPI
DeleteCriticalSection(
    _Inout_ LPCRITICAL_SECTION lpCriticalSection
);
```

当临界区不再需要时，调用 `DeleteCriticalSection` 释放该对象占用的所有资源。**该临界区必须没有被任何线程所拥有**。

### 3. 代码示例

```cpp
/* 用CriticalSection实现互斥 */
int iTickets = 100;
CRITICAL_SECTION g_cs;

DWORD WINAPI SellTicketA(LPVOID arg) {
    while (TRUE) {
        EnterCriticalSection(&g_cs);
        if (iTickets > 0)
        {
            iTickets--;
            printf("A remain %d\n", iTickets);
            LeaveCriticalSection(&g_cs);
            Sleep(1);
        } else {
            LeaveCriticalSection(&g_cs);
            break;
        }
    }
    return 0;
}

DWORD WINAPI SellTicketB(LPVOID arg) {
    while (TRUE) {
        EnterCriticalSection(&g_cs);
        if (iTickets > 0)
        {
            iTickets--;
            printf("B remain %d\n", iTickets);
            LeaveCriticalSection(&g_cs);
            Sleep(1);
        } else {
            LeaveCriticalSection(&g_cs);
            break;
        }
    }
    return 0;
}

int main() {
    HANDLE hThreadA, hThreadB;
    InitializeCriticalSection(&g_cs);
    hThreadA = CreateThread(NULL, 0, SellTicketA, NULL, 0, NULL);
    hThreadB = CreateThread(NULL, 0, SellTicketB, NULL, 0, NULL);

    WaitForSingleObject(hThreadA, INFINITE);
    WaitForSingleObject(hThreadB, INFINITE);

    DeleteCriticalSection(&g_cs);
    return 0;
}
```

### 4. 关键点总结

| 要点     | 说明                                                         |
| :------- | :----------------------------------------------------------- |
| 工作模式 | **用户模式**，不涉及内核对象，效率高                         |
| 适用场景 | 同一进程内多线程互斥访问共享资源                             |
| 跨进程   | 不支持跨进程同步（如需跨进程，应使用互斥对象 Mutex）         |
| 所有权   | 同一线程可多次进入同一临界区（递归锁），需对应相同次数的 `LeaveCriticalSection` |
| 等待超时 | 不支持超时等待（若临界区被占用，线程会无限等待）             |
| 资源释放 | 使用完毕后必须调用 `DeleteCriticalSection` 释放资源          |

