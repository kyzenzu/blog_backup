---
title: 【Windows编程】创建线程
date: 2026-07-27 11:07:56
tags:
  - C/C++
  - Windows编程
---

## Windows 创建线程（CreateThread）笔记

### 1. 核心 API

#### 1.1 CreateThread（Windows API）

```cpp
WINBASEAPI
_Ret_maybenull_
HANDLE
WINAPI
CreateThread(
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,   // 线程安全属性（NULL=默认，句柄不可继承）
    _In_ SIZE_T dwStackSize,                             // 线程栈初始大小（0=默认1MB）
    _In_ LPTHREAD_START_ROUTINE lpStartAddress,          // 线程函数入口地址
    _In_opt_ __drv_aliasesMem LPVOID lpParameter,        // 传递给线程函数的参数
    _In_ DWORD dwCreationFlags,                          // 创建标志（0=立即运行，CREATE_SUSPENDED=挂起）
    _Out_opt_ LPDWORD lpThreadId                         // 输出：接收线程ID（可为NULL）
);
```

#### 1.2 _beginthreadex（C 运行时库）

```cpp
_ACRTIMP uintptr_t __cdecl _beginthreadex(
    _In_opt_  void* _Security,                           // 线程安全属性（NULL=默认）
    _In_      unsigned _StackSize,                       // 线程栈大小（0=默认）
    _In_      _beginthreadex_proc_type _StartAddress,    // 线程函数入口（必须使用 __cdecl 调用约定）
    _In_opt_  void* _ArgList,                            // 传递给线程函数的参数
    _In_      unsigned _InitFlag,                        // 创建标志（0=立即运行，CREATE_SUSPENDED=挂起）
    _Out_opt_ unsigned* _ThrdAddr                        // 输出：接收线程ID（可为NULL）
);
```

---

### 2. 详细参数表格

#### 2.1 CreateThread 参数详解

| 参数名                 | 类型                     | 必填         | 含义                             | 默认值（NULL/0时）                                      | 注意事项                                                     |
| :--------------------- | :----------------------- | :----------- | :------------------------------- | :------------------------------------------------------ | :----------------------------------------------------------- |
| **lpThreadAttributes** | `LPSECURITY_ATTRIBUTES`  | 可选         | 线程句柄的安全描述符和继承性设置 | `NULL` → 使用默认安全描述符，返回的线程句柄**不可继承** | 若需子进程继承线程句柄，需设置 `bInheritHandle = TRUE`       |
| **dwStackSize**        | `SIZE_T`                 | 必填         | 线程初始栈大小（字节）           | `0` → 使用默认大小（通常为 1MB）                        | 实际分配的栈大小会四舍五入到页面大小倍数；系统会在需要时自动扩展栈 |
| **lpStartAddress**     | `LPTHREAD_START_ROUTINE` | **必填**     | 线程函数的入口点地址             | ⚠️ **无默认值**，必须指定有效函数指针                    | 线程函数签名：`DWORD WINAPI ThreadProc(LPVOID lpParam);`     |
| **lpParameter**        | `LPVOID`                 | 可选         | 传递给线程函数的自定义参数指针   | `NULL` → 线程函数接收 NULL                              | 可传递任意类型指针（结构体、对象等），但需确保参数生命周期覆盖线程执行期间 |
| **dwCreationFlags**    | `DWORD`                  | 必填         | 控制线程创建的标志               | `0` → 线程创建后立即运行                                | 常用标志：`CREATE_SUSPENDED`（挂起）、`STACK_SIZE_PARAM_IS_A_RESERVATION`（保留栈） |
| **lpThreadId**         | `LPDWORD`                | 可选（输出） | 接收新线程的唯一标识符（TID）    | `NULL` → 不返回线程 ID                                  | 可用于调试或后续操作（如 `OpenThread`）；注意线程 ID 不一定是系统唯一（仅在本进程内唯一） |

#### 2.2 _beginthreadex 参数详解

| 参数名            | 类型                       | 必填         | 含义                                      | 默认值（NULL/0时）                    | 注意事项                                                     |
| :---------------- | :------------------------- | :----------- | :---------------------------------------- | :------------------------------------ | :----------------------------------------------------------- |
| **_Security**     | `void*`                    | 可选         | 同 `CreateThread` 的 `lpThreadAttributes` | `NULL` → 默认安全描述符，句柄不可继承 | 内部会转换为 `SECURITY_ATTRIBUTES*`                          |
| **_StackSize**    | `unsigned`                 | 必填         | 同 `CreateThread` 的 `dwStackSize`        | `0` → 使用默认栈大小（1MB）           | 实际行为与 `CreateThread` 相同                               |
| **_StartAddress** | `_beginthreadex_proc_type` | **必填**     | 线程函数入口点                            | ⚠️ **无默认值**                        | 线程函数签名：`unsigned __stdcall ThreadProc(void* arg);`<br>**注意：必须使用 `__stdcall` 调用约定** |
| **_ArgList**      | `void*`                    | 可选         | 同 `CreateThread` 的 `lpParameter`        | `NULL` → 线程函数接收 NULL            | 同 `CreateThread`                                            |
| **_InitFlag**     | `unsigned`                 | 必填         | 同 `CreateThread` 的 `dwCreationFlags`    | `0` → 立即运行                        | 支持 `CREATE_SUSPENDED` 标志                                 |
| **_ThrdAddr**     | `unsigned*`                | 可选（输出） | 同 `CreateThread` 的 `lpThreadId`         | `NULL` → 不返回线程 ID                | 同 `CreateThread`                                            |

---

### 3. 两者的关键区别与选型建议

| 对比维度           | `CreateThread`（Windows API）                           | `_beginthreadex`（CRT）                                      |
| :----------------- | :------------------------------------------------------ | :----------------------------------------------------------- |
| **所属库**         | `kernel32.dll`                                          | `msvcrt*.dll`（C 运行时库）                                  |
| **调用约定**       | 线程函数：`WINAPI`（`__stdcall`）                       | 线程函数：`__stdcall`（但必须用 `_beginthreadex_proc_type` 类型） |
| **返回值类型**     | `HANDLE`（内核句柄）                                    | `uintptr_t`（可转换为 `HANDLE`）                             |
| **线程 ID 类型**   | `DWORD`                                                 | `unsigned int`                                               |
| **C 运行时初始化** | ❌ **不会**初始化 CRT（如 `errno`、`strtok` 等静态变量） | ✅ **会**正确初始化 CRT，确保线程安全的库函数调用             |
| **内存管理**       | 直接使用系统 API                                        | 会额外分配 CRT 专用的线程本地存储（TLS）块                   |
| **适用场景**       | 纯 Windows API 编程，不依赖 C 运行时库                  | C/C++ 程序，使用了 `errno`、`strtok`、`malloc` 等 CRT 函数   |
| **推荐程度**       | ⚠️ 仅限纯系统编程                                        | ✅ **强烈推荐**（除非项目完全不用 CRT）                       |

---

### 4. 使用要点与注意事项

#### 4.1 通用要点

1. **线程函数原型**：
   - `CreateThread`：`DWORD WINAPI ThreadProc(LPVOID lpParam)`
   - `_beginthreadex`：`unsigned __stdcall ThreadProc(void* arg)`
   - 两者返回值和参数类型略有不同，但功能等价

2. **清理资源**：
   - 线程句柄（`HANDLE` / `uintptr_t`）在使用完后必须调用 `CloseHandle()` 释放
   - `_beginthreadex` 创建线程后，应配套使用 `_endthreadex()`（而非 `ExitThread`）以正确清理 CRT 资源

3. **错误处理**：
   - `CreateThread` 失败返回 `NULL`，用 `GetLastError()` 获取错误码
   - `_beginthreadex` 失败返回 `0`，用 `errno` 或 `_doserrno` 获取错误码

#### 4.2 重要提醒 ⚠️

- **绝对不要**在 C++ 程序中使用 `CreateThread` 创建线程，除非你完全不使用 C 运行时库
- 使用 `_beginthreadex` 时，线程退出应使用 `_endthreadex()`（或让线程函数 `return`），**不要**调用 `ExitThread` 或 `TerminateThread`
- 如果使用 C++ 的标准库特性（如 `std::thread`、`std::async`），直接使用标准库更安全，它们内部已正确封装了 `_beginthreadex`

---

### 5. 代码示例

##### 示例一

```cpp
int main() {
    uintptr_t __cdecl _beginthreadex(
        _In_opt_  void* _Security,
        _In_      unsigned                 _StackSize,
        _In_      _beginthreadex_proc_type _StartAddress,
        _In_opt_  void* _ArgList,
        _In_      unsigned                 _InitFlag,
        _Out_opt_ unsigned* _ThrdAddr
    );
    int xiaohong = 20, ming = 10, laowang = 50;
    unsigned int hong_id, ming_id, wang_id;
    _beginthreadex(NULL, 0, thread_main_hong, (void*)&xiaohong, 0, &hong_id);
    _beginthreadex(NULL, 0, thread_main_ming, (void*)&ming, 0, &ming_id);
    _beginthreadex(NULL, 0, thread_main_wang, (void*)&laowang, 0, &wang_id);

    WINBASEAPI
        _Ret_maybenull_
        HANDLE
        WINAPI
        CreateThread(
        _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,
        _In_ SIZE_T dwStackSize,
        _In_ LPTHREAD_START_ROUTINE lpStartAddress,
        _In_opt_ __drv_aliasesMem LPVOID lpParameter,
        _In_ DWORD dwCreationFlags,
        _Out_opt_ LPDWORD lpThreadId
        );
    
    int m = 10;
    DWORD dwThreadID;
    HANDLE hThread = CreateThread(NULL, 0, ThreadFunc, &m, 0, &dwThreadID);
    printf("我是主线程, PID=%d\n", GetCurrentThreadId());
    if (WaitForSingleObject(hThread, INFINITE) == WAIT_FAILED) {
        puts("thread wait error");
        return -1;
    }
    //CloseHandle(hThread);
    system("pause");
    return 0;
}
```

##### 示例二

```cpp
int main(int argc, char* argv[])
{
    HANDLE tHandles[NUM_THREAD];
    int i;

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
        _In_reads_(nCount) CONST HANDLE * lpHandles,   // 要监测的句柄的组
        _In_ BOOL bWaitAll,  // TRUE 等待所有的内核对象发出信号， FALSE 任意一个内核对象发出信号
        _In_ DWORD dwMilliseconds //等 待时间
        );

    WaitForMultipleObjects(NUM_THREAD, tHandles, TRUE, INFINITE);
    CloseHandle(hMutex);
    printf("result: %lld \n", num);
    system("pause");

    return 0;
}
```

