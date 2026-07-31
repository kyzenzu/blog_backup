---
title: 【Windows编程】创建进程
date: 2026-07-27 10:26:08
tags:
  - C/C++
  - Windows编程
---

收到，整理后的创建进程笔记如下（代码中所有原有注释完整保留）：

---

## Windows 创建进程（CreateProcess）笔记

### 1. 核心 API：`CreateProcessW`

#### 1.1 函数原型

```cpp
WINBASEAPI
BOOL
WINAPI
CreateProcessW(
    _In_opt_ LPCWSTR lpApplicationName,           // 可执行文件完整路径（可为NULL）
    _Inout_opt_ LPWSTR lpCommandLine,             // 命令行字符串（会被修改，不能为常量）
    _In_opt_ LPSECURITY_ATTRIBUTES lpProcessAttributes, // 进程句柄安全属性（NULL则不可继承）
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,  // 线程句柄安全属性（NULL则不可继承）
    _In_ BOOL bInheritHandles,                    // TRUE=继承所有可继承句柄，FALSE=不继承
    _In_ DWORD dwCreationFlags,                   // 创建标志（如 CREATE_NEW_CONSOLE）
    _In_opt_ LPVOID lpEnvironment,                // 环境块（NULL则使用父进程环境）
    _In_opt_ LPCWSTR lpCurrentDirectory,          // 当前工作目录（NULL则使用父进程目录）
    _In_ LPSTARTUPINFOW lpStartupInfo,            // 启动信息结构（必须初始化cb成员）
    _Out_ LPPROCESS_INFORMATION lpProcessInformation // 输出：进程和主线程的句柄+ID
);
```

#### 1.2 详细参数表格

| 参数名                   | 类型                     | 必填             | 含义                                             | 默认值（NULL/0时）                                      | 注意事项                                                     |
| :----------------------- | :----------------------- | :--------------- | :----------------------------------------------- | :------------------------------------------------------ | :----------------------------------------------------------- |
| **lpApplicationName**    | `LPCWSTR`                | 可选             | 要执行的 `.exe` 文件的完整路径（含扩展名）       | 若为 `NULL`，则从 `lpCommandLine` 中解析可执行文件名    | 若指定了路径但不含扩展名，不会自动补 `.exe`；建议传 `NULL`，用命令行携带程序名 |
| **lpCommandLine**        | `LPWSTR`                 | 可选             | 完整的命令行字符串（程序名 + 参数）              | 若为 `NULL`，则新进程无命令行参数                       | ⚠️ **必须可写**（不能是字符串字面量），函数会修改此缓冲区     |
| **lpProcessAttributes**  | `LPSECURITY_ATTRIBUTES`  | 可选             | 进程句柄的安全描述符和继承性设置                 | `NULL` → 使用默认安全描述符，返回的进程句柄**不可继承** | 若需子进程继承进程句柄，需设置 `bInheritHandle = TRUE`       |
| **lpThreadAttributes**   | `LPSECURITY_ATTRIBUTES`  | 可选             | 线程句柄的安全描述符和继承性设置                 | `NULL` → 使用默认安全描述符，返回的线程句柄**不可继承** | 若需子进程继承线程句柄，需设置 `bInheritHandle = TRUE`       |
| **bInheritHandles**      | `BOOL`                   | 必填             | 是否让新进程继承父进程中所有**可继承**的句柄     | 无默认值，必须显式指定                                  | 仅影响标记为可继承的句柄                                     |
| **dwCreationFlags**      | `DWORD`                  | 必填             | 控制进程创建方式、优先级、窗口风格等             | 无默认值，必须显式指定                                  | 常用标志见下方表格                                           |
| **lpEnvironment**        | `LPVOID`                 | 可选             | 新进程的环境变量块（以 `\0\0` 结尾的键值对列表） | `NULL` → 使用父进程的环境块                             | 若自定义，需手动构建并确保双 `\0` 结尾                       |
| **lpCurrentDirectory**   | `LPCWSTR`                | 可选             | 新进程的当前工作目录路径                         | `NULL` → 继承父进程的当前工作目录                       | 若指定，路径必须存在                                         |
| **lpStartupInfo**        | `LPSTARTUPINFOW*`        | **必填**         | 指定新进程的主窗口样式、标准句柄重定向等启动信息 | ⚠️ **无默认值**，必须传入有效指针                        | 调用前必须初始化 `cb` 成员                                   |
| **lpProcessInformation** | `LPPROCESS_INFORMATION*` | **必填（输出）** | 接收新进程的句柄、ID 和主线程的句柄、ID          | ⚠️ **无默认值**，必须传入有效指针                        | 成功后必须 `CloseHandle` 关闭 `hProcess` 和 `hThread`        |

#### 1.3 `dwCreationFlags` 常用标志

| 标志常量                | 说明                                     |
| :---------------------- | :--------------------------------------- |
| `0`                     | 默认创建方式                             |
| `CREATE_NEW_CONSOLE`    | 新进程使用新的控制台窗口                 |
| `CREATE_SUSPENDED`      | 主线程被挂起，需调用 `ResumeThread` 恢复 |
| `DETACHED_PROCESS`      | 新进程不附加到父进程的控制台             |
| `HIGH_PRIORITY_CLASS`   | 高优先级进程                             |
| `NORMAL_PRIORITY_CLASS` | 正常优先级进程                           |
| `CREATE_NO_WINDOW`      | 不显示控制台窗口                         |

#### 1.4 返回值和补充说明

| 项目                     | 说明                                                         |
| :----------------------- | :----------------------------------------------------------- |
| **返回值**               | 成功 → 非 `0`（`TRUE`）；失败 → `0`（`FALSE`），详细错误通过 `GetLastError()` 获取 |
| **字符集**               | `CreateProcessW` 使用 **Unicode（UTF-16）** 宽字符；对应 ANSI 版本为 `CreateProcessA` |
| **lpCommandLine 可写性** | ⚠️ 即使不需要修改命令行，也必须传递可写缓冲区（如 `wchar_t buf[256]`），不能传 `L"..."` 字面量 |
| **进程/线程句柄关闭**    | `hProcess` 和 `hThread` 是内核句柄，**必须关闭**，否则导致句柄泄漏 |
| **典型调用模式**         | 先定义 `PROCESS_INFORMATION pi = {0};` 和 `STARTUPINFOW si = { sizeof(si) };`，调用成功后处理 |

---

### 2. `STARTUPINFOW` 结构体

`STARTUPINFOW` 结构体用于在调用 `CreateProcess` 时，精确控制新进程的主窗口外观、位置、以及标准输入输出句柄等启动信息。

#### 2.1 结构体定义与成员概览

```cpp
typedef struct _STARTUPINFOW {
  DWORD  cb;               // 结构体大小，必须初始化
  LPWSTR lpReserved;       // 保留，必须为 NULL
  LPWSTR lpDesktop;        // 桌面和窗口站名称
  LPWSTR lpTitle;          // 控制台窗口标题
  DWORD  dwX;              // 窗口左上角 X 坐标
  DWORD  dwY;              // 窗口左上角 Y 坐标
  DWORD  dwXSize;          // 窗口宽度
  DWORD  dwYSize;          // 窗口高度
  DWORD  dwXCountChars;    // 控制台屏幕缓冲区宽度（字符列）
  DWORD  dwYCountChars;    // 控制台屏幕缓冲区高度（字符行）
  DWORD  dwFillAttribute;  // 控制台初始文本/背景颜色
  DWORD  dwFlags;          // 位标志，控制哪些成员有效
  WORD   wShowWindow;      // 窗口首次显示状态
  WORD   cbReserved2;      // 保留，必须为 0
  LPBYTE lpReserved2;      // 保留，必须为 NULL
  HANDLE hStdInput;        // 标准输入句柄
  HANDLE hStdOutput;       // 标准输出句柄
  HANDLE hStdError;        // 标准错误句柄
} STARTUPINFOW, *LPSTARTUPINFOW;
```

#### 2.2 详细参数表格

| 参数                                       | 说明与用法                                                   |
| :----------------------------------------- | :----------------------------------------------------------- |
| **`cb`**                                   | **必须初始化**。指定结构体大小（以字节为单位），用于版本控制。应设置为 `sizeof(STARTUPINFOW)`。 |
| **`lpReserved`**                           | 保留字段，**必须为 `NULL`**。                                |
| **`lpDesktop`**                            | 指定进程要运行的**桌面**或**窗口站**名称。若为 `NULL`，则新进程继承父进程的桌面和窗口站。 |
| **`lpTitle`**                              | **控制台进程**：指定新控制台窗口的标题。若为 `NULL`，则使用可执行文件名作为标题。**GUI进程**：必须为 `NULL`。 |
| **`dwX`, `dwY`**                           | 当 `dwFlags` 包含 `STARTF_USEPOSITION` 时有效。指定新窗口左上角相对于屏幕左上角的 **X** 和 **Y** 偏移量（像素）。 |
| **`dwXSize`, `dwYSize`**                   | 当 `dwFlags` 包含 `STARTF_USESIZE` 时有效。指定新窗口的**宽度**和**高度**（像素）。 |
| **`dwXCountChars`, `dwYCountChars`**       | 当 `dwFlags` 包含 `STARTF_USECOUNTCHARS` 时有效。**仅用于控制台**，指定屏幕缓冲区的宽度（字符列）和高度（字符行）。 |
| **`dwFillAttribute`**                      | 当 `dwFlags` 包含 `STARTF_USEFILLATTRIBUTE` 时有效。**仅用于控制台**，指定初始文本和背景颜色。 |
| **`dwFlags`**                              | **核心控制位域**。通过组合标志来指示哪些成员有效（详见下文 `dwFlags` 详解）。 |
| **`wShowWindow`**                          | 当 `dwFlags` 包含 `STARTF_USESHOWWINDOW` 时有效。指定窗口首次显示状态，可取 `SW_*` 常量（如 `SW_SHOW`、`SW_HIDE`），但不能用 `SW_SHOWDEFAULT`。 |
| **`cbReserved2`**                          | 保留供 C 运行时库使用，**必须为 0**。                        |
| **`lpReserved2`**                          | 保留供 C 运行时库使用，**必须为 `NULL`**。                   |
| **`hStdInput`, `hStdOutput`, `hStdError`** | 当 `dwFlags` 包含 `STARTF_USESTDHANDLES` 时有效。指定新进程的标准输入、输出和错误句柄。使用此功能时，`bInheritHandles` 必须为 `TRUE`，且句柄必须可继承。 |

#### 2.3 `dwFlags` 常用标志详解

| 标志常量                  | 值           | 作用                                                         |
| :------------------------ | :----------- | :----------------------------------------------------------- |
| `STARTF_USESHOWWINDOW`    | `0x00000001` | 使用 `wShowWindow` 成员                                      |
| `STARTF_USESIZE`          | `0x00000002` | 使用 `dwXSize` 和 `dwYSize` 成员                             |
| `STARTF_USEPOSITION`      | `0x00000004` | 使用 `dwX` 和 `dwY` 成员                                     |
| `STARTF_USECOUNTCHARS`    | `0x00000008` | 使用 `dwXCountChars` 和 `dwYCountChars` 成员（控制台）       |
| `STARTF_USEFILLATTRIBUTE` | `0x00000010` | 使用 `dwFillAttribute` 成员（控制台）                        |
| `STARTF_RUNFULLSCREEN`    | `0x00000020` | **仅对 x86 控制台应用**，强制以全屏模式运行                  |
| `STARTF_FORCEONFEEDBACK`  | `0x00000040` | 在进程启动时，强制开启“沙漏”或“后台运行”指针反馈             |
| `STARTF_FORCEOFFFEEDBACK` | `0x00000080` | 在进程启动时，强制关闭指针反馈                               |
| `STARTF_USESTDHANDLES`    | `0x00000100` | 使用 `hStdInput`、`hStdOutput` 和 `hStdError` 成员重定向标准句柄 |
| `STARTF_USEHOTKEY`        | `0x00000200` | 使用 `hStdInput` 作为热键值（极少使用，不能与 `STARTF_USESTDHANDLES` 共用） |
| `STARTF_TITLEISLINKNAME`  | `0x00000800` | `lpTitle` 包含的是启动进程的 `.lnk` 快捷方式路径             |
| `STARTF_TITLEISAPPID`     | `0x00001000` | `lpTitle` 包含的是应用程序用户模型 ID (AppUserModelID)       |
| `STARTF_PREVENTPINNING`   | `0x00002000` | 阻止进程创建的窗口被固定到任务栏（需与 `STARTF_TITLEISAPPID` 共用） |

#### 2.4 核心使用要点

1. **必须初始化 `cb`**：在使用该结构体前，务必先清零并设置 `cb` 成员，这是最常见的错误来源。
   ```cpp
   STARTUPINFOW si = { 0 };
   si.cb = sizeof(si);
   ```

2. **标准句柄重定向**：`STARTF_USESTDHANDLES` 与 `CreateProcess` 的 `bInheritHandles` 参数紧密配合。父进程需将标准句柄（如管道、文件）设置为可继承，并将 `bInheritHandles` 设为 `TRUE`，子进程才能正确继承和使用这些句柄。

3. **GUI 与控制台的区别**：大多数 GUI 应用程序只需设置 `cb` 和 `dwFlags`（如果需要），其他字段应保持为 `NULL`。控制台应用程序则可以通过此结构体更精细地控制其窗口外观。

---

### 3. 代码示例

```cpp
void RunExe() {
    WCHAR szCommandLine[] = L"\"C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe\" https://www.baidu.com";

    STARTUPINFO strStartupInfo = { 0 };
    strStartupInfo.cb = sizeof(STARTUPINFO);

    PROCESS_INFORMATION szProcessInformation = { 0 };

    BOOL bRet = CreateProcess(NULL,
                  szCommandLine, 
                  NULL, 
                  NULL, 
                  FALSE, 
                  CREATE_NEW_CONSOLE, 
                  NULL, 
                  NULL, 
                  &strStartupInfo,
                  &szProcessInformation
    );
    if (bRet) {
        printf("Create Success: %d\n", bRet);
        printf("szProcessInformation.hProcess: %p\n", szProcessInformation.hProcess);
        WaitForSingleObject(szProcessInformation.hProcess, 3000);
        CloseHandle(szProcessInformation.hProcess);
        CloseHandle(szProcessInformation.hThread);
        szProcessInformation.hProcess = NULL;
        szProcessInformation.hThread = NULL;
        szProcessInformation.dwProcessId = 0;
        szProcessInformation.dwThreadId = 0;
    } else {
        printf("Create Failed: %d\n", bRet);
        printf("Error Code: %d", GetLastError());
    }
}
```

