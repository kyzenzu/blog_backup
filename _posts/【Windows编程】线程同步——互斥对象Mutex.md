---
title: 【Windows编程】线程同步——互斥对象Mutex
date: 2026-07-27 11:29:01
tags:
  - C/C++
  - Windows编程
---

收到，整理后的互斥对象笔记如下（代码中所有原有注释完整保留）：

---

## Windows 互斥对象（Mutex）笔记

### 1. 核心 API：`CreateMutexW`

```cpp
WINAPI
CreateMutexW(
    _In_opt_ LPSECURITY_ATTRIBUTES lpMutexAttributes,  // 安全属性（NULL=默认，句柄不可继承）
    _In_ BOOL bInitialOwner,                          // TRUE=当前线程立即拥有互斥体（变成无信号状态），FALSE=当前线程不拥有（为有信号状态）
    _In_opt_ LPCWSTR lpName                           // 互斥体名称（NULL=匿名，有名称=全局/局部命名对象）
);
```

### 2. 详细参数表格

| 参数名                | 类型                    | 必填 | 含义                                                 | 默认值（NULL/FALSE时）                                       | 注意事项                                                     |
| :-------------------- | :---------------------- | :--- | :--------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **lpMutexAttributes** | `LPSECURITY_ATTRIBUTES` | 可选 | 指向安全属性结构，用于设置互斥体的安全描述符和继承性 | `NULL` → 使用默认安全描述符，返回的互斥体句柄**不可继承**    | 若需要子进程继承互斥体句柄，需设置 `bInheritHandle = TRUE` 并传入有效结构指针 |
| **bInitialOwner**     | `BOOL`                  | 必填 | 指定调用线程是否立即拥有互斥体的所有权               | `FALSE` → 互斥体处于**有信号状态**（未拥有），任何线程都可以等待获取 | `TRUE`：互斥体处于**无信号状态**，调用线程成为拥有者 `FALSE`：互斥体处于**有信号状态**，未被任何线程拥有 |
| **lpName**            | `LPCWSTR`               | 可选 | 互斥体的名称（区分大小写的宽字符串），用于跨进程共享 | `NULL` → 创建**匿名**互斥体，仅在本进程内可用                | • 名称不能包含反斜杠 `\` • 若名称已存在，则打开现有互斥体（`GetLastError()` 返回 `ERROR_ALREADY_EXISTS`） • 全局名称：`"Global\\MutexName"` • 局部名称：`"Local\\MutexName"` |

---

### 3. 核心概念

#### 3.1 互斥体的信号状态与所有权

| 状态                           | 含义                                                     | 适用场景                                                     |
| :----------------------------- | :------------------------------------------------------- | :----------------------------------------------------------- |
| **有信号状态**（Signaled）     | 互斥体未被任何线程拥有，`WaitForSingleObject` 会立即返回 | `bInitialOwner = FALSE` 时创建，或拥有者调用 `ReleaseMutex` 后 |
| **无信号状态**（Non-signaled） | 互斥体被某线程拥有，其他线程等待时会阻塞                 | `bInitialOwner = TRUE` 时创建，或某线程成功等待到互斥体后    |

#### 3.2 返回值与错误处理

| 返回值              | 含义                 | 处理方式                                                     |
| :------------------ | :------------------- | :----------------------------------------------------------- |
| `HANDLE`（非 NULL） | 成功创建或打开互斥体 | 使用 `GetLastError()` 判断：`ERROR_ALREADY_EXISTS`（183）→ 打开已有对象；`NO_ERROR`（0）→ 新建对象 |
| `NULL`              | 创建失败             | 调用 `GetLastError()` 获取详细错误码                         |

---

### 4. 代码示例

```cpp
/* Mutex互斥对象 */

#define NUM_THREAD	50
unsigned WINAPI threadInc(void* arg);
unsigned WINAPI threadDes(void* arg);
long long num = 0;
HANDLE hMutex;


int main(int argc, char* argv[])
{
    HANDLE tHandles[NUM_THREAD];
    int i;

    HANDLE
        WINAPI
        CreateMutexW(
        _In_opt_ LPSECURITY_ATTRIBUTES lpMutexAttributes,   //指向安全属性
        _In_ BOOL bInitialOwner,   //初始化互斥对象的所有者  TRUE 立即拥有互斥体，false表示创建的这个mutex不属于任何线程；所以处于激发状态，也就是有信号状态
        _In_opt_ LPCWSTR lpName    //指向互斥对象名的指针  L“Bingo”
        );

    printf("sizeof long long: %d \n", sizeof(long long));
    hMutex = CreateMutex(NULL, FALSE, NULL);

    for (i = 0; i < NUM_THREAD; i++)
    {
        if (i % 2)
            tHandles[i] = (HANDLE)_beginthreadex(NULL, 0, threadInc, NULL, 0, NULL);
        else
            tHandles[i] = (HANDLE)_beginthreadex(NULL, 0, threadDes, NULL, 0, NULL);
    }

    WINBASEAPI
        DWORD
        WINAPI
        WaitForMultipleObjects(
        _In_ DWORD nCount,    // 要监测的句柄的组的句柄的个数
        _In_reads_(nCount) CONST HANDLE * lpHandles,   //要监测的句柄的组
        _In_ BOOL bWaitAll,  // TRUE 等待所有的内核对象发出信号， FALSE 任意一个内核对象发出信号
        _In_ DWORD dwMilliseconds //等待时间
        );

    WaitForMultipleObjects(NUM_THREAD, tHandles, TRUE, INFINITE);
    CloseHandle(hMutex);
    printf("result: %lld \n", num);
    system("pause");

    return 0;
}


unsigned WINAPI threadInc(void* arg)
{
    int i;
    WaitForSingleObject(hMutex, INFINITE);
    for (i = 0; i < 500000; i++)
        num += 1;
    ReleaseMutex(hMutex);
    return 0;
}
unsigned WINAPI threadDes(void* arg)
{
    int i;
    WaitForSingleObject(hMutex, INFINITE);
    for (i = 0; i < 500000; i++)
        num -= 1;
    ReleaseMutex(hMutex);
    return 0;
}
```

