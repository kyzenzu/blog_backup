---
title: 【Windows编程】进程间通信——管道
date: 2026-07-27 09:51:18
tags:
  - C/C++
  - Windows编程
---

## Windows 管道（Anonymous Pipe）笔记

### 1. 基本概念

- Windows 的匿名管道，就是从操作系统教科书上学到的那个最经典、最传统的“管道”概念。从它的API名就可以看出`CreatePipe()`。**匿名管道就是传统意义上的管道**，下面用“管道”指代匿名管道
- 其本质是**父进程与子进程之间的一块共享内存**，用于实现进程间的单向数据通信。
- 不是说数据必须从父进程流向子进程，管道不是这样设计的。管道就是一块共享内存，只不过这块共享内存只能从写句柄写入，从读句柄读取。子进程和父进程一样都能通过句柄读写管道。
- 命名管道是管道的增强版，支持更多功能，但不同操作系统的实现有所差异（Windows 与 Linux 不同）。

> **注意**：下文中的“管道”如无特殊说明，均指匿名管道。

---

### 2. 管道对象的内部结构

Windows 内核中的管道对象可以抽象为以下结构：

```
内核中的管道对象（共享内存）：
┌─────────────────────────────┐
│  管道缓冲区                  │
│  [数据流向: 写端 → 读端]     │
│                             │
│  写端引用计数: 2            │
│  读端引用计数: 2            │
└─────────────────────────────┘
         ↑              ↑
         │              │
    ┌────┴────┐   ┌────┴────┐
    │ 写句柄   │   │ 读句柄   │
    │hWritePipe│   │hReadPipe │
    │只写权限   │   │只读权限   │
    └────┬────┘   └────┬────┘
         │              │
    父进程持有      父进程持有
    子进程继承      子进程继承
```

- 管道的读写两端均有**引用计数**，父进程和子进程各持有一份句柄。
- 当所有写句柄被关闭时，读操作会失败或返回 EOF；同理，所有读句柄关闭后，写操作将失败。

---

### 3. 核心 API

#### 3.1 创建管道：`CreatePipe`
- 用于创建匿名管道，返回读句柄和写句柄。
- 可通过 `SECURITY_ATTRIBUTES` 控制句柄是否可被子进程继承。

#### 3.2 创建子进程：`CreateProcessW`

函数原型（关键参数说明）：

```c
WINBASEAPI
BOOL
WINAPI
CreateProcessW(
    _In_opt_ LPCWSTR lpApplicationName,          // 可执行文件路径（可选）
    _Inout_opt_ LPWSTR lpCommandLine,            // 命令行字符串（会被修改）
    _In_opt_ LPSECURITY_ATTRIBUTES lpProcessAttributes, // 进程安全属性，控制句柄继承
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,  // 线程安全属性
    _In_ BOOL bInheritHandles,                   // 是否继承父进程的可继承句柄
    _In_ DWORD dwCreationFlags,                  // 创建标志（如 CREATE_NO_WINDOW）
    _In_opt_ LPVOID lpEnvironment,               // 环境块
    _In_opt_ LPCWSTR lpCurrentDirectory,         // 当前工作目录
    _In_ LPSTARTUPINFOW lpStartupInfo,           // 启动信息（含标准句柄重定向）
    _Out_ LPPROCESS_INFORMATION lpProcessInformation // 返回进程/线程句柄和 ID
);
```

> 使用 `STARTUPINFOW` 结构时，必须设置 `dwFlags = STARTF_USESTDHANDLES`，并将 `hStdInput`、`hStdOutput`、`hStdError` 重定向到管道句柄，以实现子进程的标准 I/O 重定向。

| 参数                   | 类型                    | 方向              | 说明                                                         |
| :--------------------- | :---------------------- | :---------------- | :----------------------------------------------------------- |
| `lpApplicationName`    | `LPCWSTR`               | 输入（可选）      | 要执行的 `.exe` 文件完整路径。若为 `NULL`，则从 `lpCommandLine` 中提取可执行文件名。若包含路径则直接使用，否则按系统搜索顺序查找。 |
| `lpCommandLine`        | `LPWSTR`                | 输入/输出（可选） | 命令行字符串（**会被修改**，不能传常量）。可包含程序名和参数。若 `lpApplicationName` 为 `NULL`，则此字符串的第一个标记作为可执行文件名。 |
| `lpProcessAttributes`  | `LPSECURITY_ATTRIBUTES` | 输入（可选）      | 进程的安全描述符，决定返回的**进程句柄**是否可被子进程继承。为 `NULL` 时使用默认安全性且句柄不可继承。 |
| `lpThreadAttributes`   | `LPSECURITY_ATTRIBUTES` | 输入（可选）      | 主线程的安全描述符，决定返回的**线程句柄**是否可被子进程继承。为 `NULL` 时使用默认安全性且句柄不可继承。 |
| `bInheritHandles`      | `BOOL`                  | 输入              | 若为 `TRUE`，子进程将继承父进程中**所有标记为可继承**的句柄。 |
| `dwCreationFlags`      | `DWORD`                 | 输入              | 控制进程创建方式，常用值：`CREATE_SUSPENDED`（挂起主线程）、`CREATE_NEW_CONSOLE`（新建控制台窗口）等。 |
| `lpEnvironment`        | `LPVOID`                | 输入（可选）      | 新进程的环境块（以 `\0\0` 结尾的字符串列表）。为 `NULL` 时使用调用进程的环境块。 |
| `lpCurrentDirectory`   | `LPCWSTR`               | 输入（可选）      | 新进程的当前工作目录路径。为 `NULL` 时使用调用进程的当前工作目录。 |
| `lpStartupInfo`        | `LPSTARTUPINFOW`        | 输入（**必须**）  | 指定新进程的主窗口外观、标准句柄重定向等启动信息。调用前必须用 `ZeroMemory` 初始化并设置 `cb` 成员。 |
| `lpProcessInformation` | `LPPROCESS_INFORMATION` | 输出（**必须**）  | 返回新进程和主线程的句柄及 ID。调用成功后**必须调用 `CloseHandle`** 关闭 `hProcess` 和 `hThread`，避免资源泄漏。 |

---

### 4. 代码示例

#### 4.1 父进程代码（服务端）

```cpp
HANDLE hReadPipe;
HANDLE hWritePipe;

// 创建管道并启动子进程
void CAnonyPipeServerDlg::OnBnClickedButtonCreate()
{
    // 1. 创建管道
    SECURITY_ATTRIBUTES pipeSecurityAttributes;
    pipeSecurityAttributes.bInheritHandle = TRUE;
    pipeSecurityAttributes.lpSecurityDescriptor = NULL;
    pipeSecurityAttributes.nLength = sizeof(SECURITY_ATTRIBUTES);

    BOOL bRet = CreatePipe(&hReadPipe, &hWritePipe, &pipeSecurityAttributes, 0);
    if (!bRet) {
        MessageBox(TEXT("匿名管道创建失败"));
        return;
    }

    // 2. 配置子进程启动信息
    STARTUPINFO strStartUpInfo = {0};
    strStartUpInfo.cb = sizeof(STARTUPINFO);
    strStartUpInfo.dwFlags = STARTF_USESTDHANDLES;
    strStartUpInfo.hStdInput = hReadPipe;    // 子进程标准输入 ← 管道读端
    strStartUpInfo.hStdOutput = hWritePipe;  // 子进程标准输出 → 管道写端
    strStartUpInfo.hStdError = GetStdHandle(STD_ERROR_HANDLE); // 错误输出沿用父进程

    PROCESS_INFORMATION processInformation = {0};

    // 3. 创建子进程
    bRet = CreateProcess(
        NULL,
        TEXT("AnonyPipeChildProcess.exe"),
        NULL,
        NULL,
        TRUE,               // 必须为 TRUE，子进程才能继承管道句柄
        CREATE_NO_WINDOW,
        NULL,
        NULL,
        &strStartUpInfo,
        &processInformation
    );

    if (bRet) {
        // 关闭父进程持有的子进程句柄，减少引用计数
        CloseHandle(processInformation.hProcess);
        CloseHandle(processInformation.hThread);
        processInformation.dwProcessId = 0;
        processInformation.dwThreadId = 0;
        processInformation.hThread = NULL;
        processInformation.hProcess = NULL;
    } else {
        CloseHandle(hReadPipe);
        CloseHandle(hWritePipe);
        hReadPipe = NULL;
        hWritePipe = NULL;
        MessageBox(TEXT("创建子进程失败"));
    }
}

// 向管道写入数据（父进程发送）
void CAnonyPipeServerDlg::OnBnClickedButtonSend()
{
    char szSendBuf[100] = "我是父进程";
    DWORD dwWriteNum;
    if (!WriteFile(hWritePipe, szSendBuf, strlen(szSendBuf) + 1, &dwWriteNum, NULL)) {
        MessageBox("Write Failed");
    }
}

// 从管道读取数据（父进程接收）
void CAnonyPipeServerDlg::OnBnClickedButtonRecv()
{
    char szRecvBuf[100] = {0};
    DWORD dwReadNum;
    if (!ReadFile(hReadPipe, szRecvBuf, 100, &dwReadNum, NULL)) {
        MessageBox("Read Failed");
        return;
    }
    TRACE("dwReadNum = %d\n", dwReadNum);
    MessageBox(szRecvBuf);
}
```

---

#### 4.2 子进程代码（客户端）

```cpp
// 向管道写入数据（向标准输出句柄写入）
void CAnonyPipeClientDlg::OnBnClickedButtonSend()
{
    HANDLE hWritePipe = GetStdHandle(STD_OUTPUT_HANDLE);
    char szSendBuf[100] = "我是子进程";
    DWORD dwWriteNum;
    if (!WriteFile(hWritePipe, szSendBuf, strlen(szSendBuf) + 1, &dwWriteNum, NULL)) {
        MessageBox("Write Failed");
        return;
    }
}

// 从管道读取数据（从标准输入句柄读取）
void CAnonyPipeClientDlg::OnBnClickedButtonRecv()
{
    HANDLE hReadPipe = GetStdHandle(STD_INPUT_HANDLE);
    char szRecvBuf[100] = {0};
    DWORD dwReadNum;
    if (!ReadFile(hReadPipe, szRecvBuf, 100, &dwReadNum, NULL)) {
        MessageBox("Read Failed");
        return;
    }
    TRACE("dwReadNum = %d\n", dwReadNum);
    MessageBox(szRecvBuf);
}
```

---

### 5. 关键点总结

| 要点            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| 通信机制        | 基于共享内存，读写端分离                                     |
| 句柄继承        | 必须将 `SECURITY_ATTRIBUTES.bInheritHandle` 设为 `TRUE`，且 `CreateProcess` 的 `bInheritHandles` 参数为 `TRUE` |
| 标准 I/O 重定向 | 通过 `STARTUPINFO` 的 `hStdInput` / `hStdOutput` 实现        |
| 资源管理        | 父进程创建子进程后，应及时关闭 `PROCESS_INFORMATION` 中的句柄，避免资源泄漏 |
| 双向通信        | 管道本身是单向的，若需要双向通信，可创建两个管道（各负责一个方向） |

