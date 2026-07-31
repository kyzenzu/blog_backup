---
title: 【Windows编程】线程同步——信号量Semaphore
date: 2026-07-27 11:40:50
tags:
  - C/C++
  - Windows编程
---

## Windows 信号量（Semaphore）笔记

### 1. 基本概念

信号量是用于资源计数的内核对象，用于控制多个线程对共享资源的访问。

#### 1.1 信号量的状态

| 状态                         | 含义                               |
| :--------------------------- | :--------------------------------- |
| **有信号状态（触发状态）**   | 当前资源计数 > 0，表示有可用资源   |
| **无信号状态（未触发状态）** | 当前资源计数 = 0，表示没有可用资源 |

#### 1.2 信号量的组成

| 成员                                | 说明                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| **计数器**                          | 该内核对象被使用的次数                                       |
| **最大资源数量**（`lMaximumCount`） | 信号量可以控制的最大资源数量（带符号的32位），即计数器的上限值 |
| **当前资源数量**（`lInitialCount`） | 标识当前可用资源的数量（带符号的32位）。表示**当前开放资源的个数**，**只有开放的资源才能被线程所申请** |

> **注意**：`lInitialCount` 表示的是**当前可用资源的数量**，而非剩余资源的数量。每次线程成功等待信号量，当前资源计数减 `1`；每次调用 `ReleaseSemaphore`，当前资源计数增加指定的数量。

---

### 2. 核心 API

#### 2.1 创建信号量：`CreateSemaphoreW`

```cpp
HANDLE
WINAPI
CreateSemaphoreW(
    _In_opt_ LPSECURITY_ATTRIBUTES lpSemaphoreAttributes,  // 安全属性，NULL表示使用默认安全描述符且句柄不可继承
    _In_ LONG lInitialCount,                               // 初始可用资源计数（0=无信号状态，>0=有信号状态）
    _In_ LONG lMaximumCount,                               // 最大资源计数（信号量计数的上限值）
    _In_opt_ LPCWSTR lpName                                // 信号量名称，NULL表示创建匿名信号量
);
```

**参数说明：**

| 参数名                    | 含义                                                         |
| :------------------------ | :----------------------------------------------------------- |
| **lpSemaphoreAttributes** | 安全属性（`NULL` = 默认，句柄不可继承）                      |
| **lInitialCount**         | 初始可用资源数量（`0` = 无信号状态；`>0` = 有信号状态），不能超过 `lMaximumCount` |
| **lMaximumCount**         | 信号量计数的最大值（即最大资源数量）                         |
| **lpName**                | 信号量名称（`NULL` = 匿名信号量）                            |

#### 2.2 释放信号量：`ReleaseSemaphore`

```cpp
WINBASEAPI
BOOL
WINAPI
ReleaseSemaphore(
    _In_ HANDLE hSemaphore,         // 信号量对象的句柄
    _In_ LONG lReleaseCount,        // 要增加的信号量计数（释放的资源数量）
    _Out_opt_ LPLONG lpPreviousCount // 输出释放前的原始计数，可为 NULL
);
```

---

### 3. 代码示例

```cpp
/* 用信号量实现同步 */
static HANDLE semOne;
static HANDLE semTwo;
static int num;

DWORD WINAPI Read(void* arg) {
    while (TRUE) {
        fputs("\nRead\n", stdout);
        WaitForSingleObject(semTwo, INFINITE);
        printf("begin read: ");
        scanf("%d", &num);
        ReleaseSemaphore(semOne, 1, NULL);
    }
    return 0;
}
DWORD WINAPI Accu(void* arg) {
    while (TRUE) {
        fputs("\nAccu\n", stdout);
        WaitForSingleObject(semOne, INFINITE);
        printf("begin accu: ");
        scanf("%d", &num);
        ReleaseSemaphore(semTwo, 1, NULL);
    }
    return 0;
}

int main() {
    WINBASEAPI BOOL WINAPI ReleaseSemaphore(
        _In_ HANDLE hSemaphore,   //信号量的句柄
        _In_ LONG lReleaseCount,   //将lReleaseCount值加到信号量的当前资源计数上面 0-> 1
        _Out_opt_ LPLONG lpPreviousCount  //当前资源计数的原始值
        );

    HANDLE hThread1, hThread2;
    semOne = CreateSemaphore(NULL, 0, 1, NULL);
    semTwo = CreateSemaphore(NULL, 1, 1, NULL);
    hThread1 = CreateThread(NULL, 0, Read, NULL, 0, NULL);
    hThread2 = CreateThread(NULL, 0, Accu, NULL, 0, NULL);
    WaitForSingleObject(hThread1, INFINITE);
    WaitForSingleObject(hThread2, INFINITE);
    CloseHandle(semOne);
    CloseHandle(semOne);
    return 0;
}
```

