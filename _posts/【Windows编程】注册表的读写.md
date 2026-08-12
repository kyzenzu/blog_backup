---
title: 【Windows编程】注册表的读写
date: 2026-08-07 14:22:29
tags:
  - C/C++
  - Windows编程
---

## 注册表的读写

Windows 注册表是一个分层级的系统配置数据库，Win32 API 提供了丰富的函数来操作注册表。通过 `regedit` 命令可以打开注册表编辑器进行查看。

---

## 一、注册表基础结构

⭐API名称解释：将文件目录（根键 + 子项）称为 `KEY`，将文件目录下的文件（其实就是一个个键值对）称为 `VALUE`

注册表采用树形结构组织，由**根键 → 子项 → 键值**三级组成：

- **根键（HKEY）**：注册表的顶级节点，共 5 个
- **子项（SubKey）**：类似于文件系统中的文件夹，可多层嵌套
- **键值（Value）**：实际存储配置数据的"文件"，包含名称、类型和数据

### 1.1 根键（HKEY）说明

| 根键                    | 简写 | 用途                                           |
| :---------------------- | :--- | :--------------------------------------------- |
| **HKEY_LOCAL_MACHINE**  | HKLM | 存储计算机硬件和操作系统的全局配置，与用户无关 |
| **HKEY_CURRENT_USER**   | HKCU | 存储当前登录用户的个性化设置                   |
| **HKEY_USERS**          | HKU  | 包含计算机上所有用户的配置文件                 |
| **HKEY_CLASSES_ROOT**   | HKCR | 管理文件扩展名关联和 COM 对象注册信息          |
| **HKEY_CURRENT_CONFIG** | HKCC | 存储当前生效的硬件配置文件信息                 |

> **注意：** 应用程序不能直接在 `HKLM` 或 `HKU` 根键下创建子项，必须在系统定义的子项（如 `HKLM\SOFTWARE`）下进行操作。

### 1.2 键值数据类型

每个键值由**名称**、**类型**和**数据**三部分组成，常见数据类型如下：

| 类型            | 说明                                              |
| :-------------- | :------------------------------------------------ |
| `REG_SZ`        | 固定长度文本字符串，最常用                        |
| `REG_EXPAND_SZ` | 可展开字符串，可包含环境变量（如 `%SystemRoot%`） |
| `REG_DWORD`     | 32 位整数，常用于开关控制或计数                   |
| `REG_BINARY`    | 原始二进制数据，存储硬件信息等                    |
| `REG_MULTI_SZ`  | 多个字符串组成的序列                              |

---

## 二、核心 API

### 2.1 RegCreateKey - 创建注册表项

```cpp
WINADVAPI
LSTATUS
APIENTRY
RegCreateKey(
    _In_ HKEY hKey,          // 根键，如 HKEY_LOCAL_MACHINE
    _In_opt_ LPCSTR lpSubKey, // 子项路径，如 "SOFTWARE\\MYWEIGHT"
    _Out_ PHKEY phkResult    // 返回的操作句柄（准入门票）
    );
```

**功能特点：**
- 如果指定的子项不存在，会**自动创建**（包括路径中所有不存在的父项）
- 成功后返回句柄，后续操作通过句柄进行，无需重复指定完整路径
- 操作完成后需调用 `RegCloseKey` 关闭句柄释放资源

### 2.2 RegSetValueEx - 设置键值数据

```cpp
WINADVAPI
LSTATUS
APIENTRY
RegSetValueExA(
    _In_ HKEY hKey,                        // 注册表项句柄
    _In_opt_ LPCSTR lpValueName,           // 键值名称
    _Reserved_ DWORD Reserved,             // 保留，必须为 0
    _In_ DWORD dwType,                     // 数据类型：REG_DWORD / REG_SZ 等
    _In_reads_bytes_opt_(cbData) CONST BYTE* lpData, // 要写入的数据
    _In_ DWORD cbData                      // 数据大小（字节数）
    );
```

### 2.3 RegOpenKey - 打开注册表项

```cpp
WINADVAPI
LSTATUS
APIENTRY
RegOpenKey(
    _In_ HKEY hKey,          // 根键
    _In_opt_ LPCSTR lpSubKey, // 子项路径
    _Out_ PHKEY phkResult    // 返回的操作句柄
    );
```

**与 `RegCreateKey` 的区别：** `RegOpenKey` 仅打开已存在的项，不会自动创建新项。

### 2.4 RegQueryValueEx - 查询键值数据

```cpp
WINADVAPI
LSTATUS
APIENTRY
RegQueryValueEx(
    _In_ HKEY hKey,                       // 注册表项句柄
    _In_opt_ LPCSTR lpValueName,          // 键值名称
    _Reserved_ LPDWORD lpReserved,        // 保留，必须为 NULL
    _Out_opt_ LPDWORD lpType,             // 返回数据类型（可为 NULL）
    _Out_writes_bytes_to_opt_(*lpcbData, *lpcbData) __out_data_source(REGISTRY) LPBYTE lpData, // 接收数据的缓冲区
    _When_(lpData == NULL, _Out_opt_) _When_(lpData != NULL, _Inout_opt_) LPDWORD lpcbData // 缓冲区大小/实际返回大小
    );
```

---

## 三、注册表重定向（Registry Redirector）

### 3.1 什么是重定向？

在 64 位 Windows 系统上，为了兼容 32 位应用程序，系统会自动将 32 位程序对 `HKLM\SOFTWARE` 的访问重定向到 `HKLM\SOFTWARE\WOW6432Node`：

| 你的代码请求             | 系统实际访问                         |
| :----------------------- | :----------------------------------- |
| `HKLM\SOFTWARE\MYWEIGHT` | `HKLM\SOFTWARE\WOW6432Node\MYWEIGHT` |

### 3.2 为什么需要重定向？

- **64 位应用** → 访问真实路径 `HKLM\SOFTWARE`
- **32 位应用** → 访问被重定向到 `HKLM\SOFTWARE\WOW6432Node`

> **WOW6432Node** 的含义：**W**indows **O**n **W**indows **64**，即 64 位系统上的 32 位子系统。它像是一个"隔离区"，让 32 位程序的数据与 64 位程序的数据互不干扰。

---

## 四、写入注册表示例

在 `HKLM\SOFTWARE\MYWEIGHT` 下创建一个名为 `admin` 的 `REG_DWORD` 类型键值，保存体重数据（80）：

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    HKEY hkResult;
    char szSubKey[MAX_PATH] = "SOFTWARE\\MYWEIGHT";

    // 1. 创建（或打开）注册表项
    DWORD dwRet = RegCreateKey(HKEY_LOCAL_MACHINE, szSubKey, &hkResult);
    if (dwRet != ERROR_SUCCESS) {
        TRACE("[ERROR]: %d\n", dwRet);
        MessageBox("创建注册表失败");
        return;
    }

    // 2. 写入键值数据
    DWORD dwValueData = 80;
    dwRet = RegSetValueEx(
        hkResult,           // 句柄
        "admin",            // 键值名称
        0,                  // 保留，必须为 0
        REG_DWORD,          // 数据类型
        (CONST BYTE*)&dwValueData,  // 数据
        sizeof(DWORD)       // 数据大小
    );
    if (dwRet != ERROR_SUCCESS) {
        MessageBox("写入注册表失败");
        return;
    }

    MessageBox("注册表写入成功");
    RegCloseKey(hkResult);  // 3. 关闭句柄，释放资源
}
```

---

## 五、读取注册表示例

读取刚才写入的 `admin` 键值并显示：

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    HKEY hkResult;
    char szSubKey[MAX_PATH] = "SOFTWARE\\MYWEIGHT";

    // 1. 打开已存在的注册表项
    DWORD dwRet = RegOpenKey(HKEY_LOCAL_MACHINE, szSubKey, &hkResult);
    if (dwRet != ERROR_SUCCESS) {
        TRACE("[ERROR]: %d\n", dwRet);
        MessageBox("打开注册表失败");
        return;
    }

    // 2. 查询键值数据
    DWORD dwValueData;      // 存储读取的数据
    DWORD dwValueType;      // 存储数据类型
    DWORD cbValue = sizeof(DWORD);  // 缓冲区大小，函数返回时会更新为实际大小

    dwRet = RegQueryValueEx(
        hkResult,           // 句柄
        "admin",            // 键值名称
        NULL,               // 保留，必须为 NULL
        &dwValueType,       // 返回数据类型
        (BYTE*)&dwValueData, // 接收数据的缓冲区
        &cbValue            // 缓冲区大小
    );
    if (dwRet != ERROR_SUCCESS) {
        MessageBox("读取注册表失败");
        return;
    }

    // 3. 显示读取结果
    char szDisplayMsg[1024] = { 0 };
    sprintf(szDisplayMsg, "Weight = %d", dwValueData);
    MessageBox(szDisplayMsg);

    RegCloseKey(hkResult);  // 关闭句柄
}
```

---

## 六、常用操作补充

### 6.1 删除注册表项或键值

```cpp
// 删除指定键值
RegDeleteValue(hKey, "admin");

// 删除整个子项（该子项下必须为空）
RegDeleteKey(HKEY_LOCAL_MACHINE, "SOFTWARE\\MYWEIGHT");
```

### 6.2 枚举子项或键值

```cpp
// 枚举子项
DWORD dwIndex = 0;
char szSubKeyName[MAX_PATH];
DWORD dwSize = MAX_PATH;
while (RegEnumKeyEx(hKey, dwIndex++, szSubKeyName, &dwSize, NULL, NULL, NULL, NULL) == ERROR_SUCCESS) {
    // 处理每个子项
    dwSize = MAX_PATH;
}

// 枚举键值
DWORD dwIndex = 0;
char szValueName[MAX_PATH];
DWORD dwSize = MAX_PATH;
DWORD dwType;
BYTE byData[1024];
DWORD dwDataSize = 1024;
while (RegEnumValue(hKey, dwIndex++, szValueName, &dwSize, NULL, &dwType, byData, &dwDataSize) == ERROR_SUCCESS) {
    // 处理每个键值
    dwSize = MAX_PATH;
    dwDataSize = 1024;
}
```

### 6.3 使用技巧与注意事项

1. **权限问题**：向 `HKLM` 写入通常需要管理员权限，否则会返回 `ERROR_ACCESS_DENIED`
2. **路径分隔符**：子项路径中使用 `\\` 转义反斜杠
3. **关闭句柄**：每次操作完成后必须调用 `RegCloseKey` 释放资源
4. **判断返回值**：始终检查返回值是否为 `ERROR_SUCCESS` 来判断操作是否成功
5. **缓冲区大小**：`RegQueryValueEx` 的 `lpcbData` 参数在调用前后含义不同——调用前表示缓冲区大小，调用后表示实际返回的数据大小
