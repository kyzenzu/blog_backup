---
title: 【Windows编程】配置文件访问与读写
date: 2026-08-07 14:20:26
tags:
  - C/C++
  - Windows编程
---

## 配置文件访问与读写

Windows 提供了一套专门用于操作 INI 配置文件的 API，方便程序对配置信息进行持久化存储。

---

## 一、核心 API

### 1.1 WritePrivateProfileString - 写入配置项

```cpp
WINBASEAPI
BOOL
WINAPI
WritePrivateProfileString(
    _In_opt_ LPCSTR lpAppName,   // 节名称（Section），如 "[Param1]"，不存在则自动创建
    _In_opt_ LPCSTR lpKeyName,   // 键名称（Key），不存在则自动创建；若为 NULL，则删除整个节
    _In_opt_ LPCSTR lpString,    // 要写入的值（字符串），若为 NULL 则删除该键
    _In_opt_ LPCSTR lpFileName   // .ini 文件路径（完整路径）
    );
```

### 1.2 GetPrivateProfileString - 读取配置项

```cpp
WINBASEAPI
DWORD
WINAPI
GetPrivateProfileStringA(
    _In_opt_ LPCSTR lpAppName,          // 节名称，若为 NULL 则返回文件中所有节名
    _In_opt_ LPCSTR lpKeyName,          // 键名称，若为 NULL 则返回该节下所有键名
    _In_opt_ LPCSTR lpDefault,          // 默认值（当键不存在时返回此值）
    _Out_writes_to_opt_(nSize, return + 1) LPSTR lpReturnedString, // 接收结果的缓冲区
    _In_     DWORD nSize,               // 缓冲区大小（字符数）
    _In_opt_ LPCSTR lpFileName          // .ini 文件路径（完整路径）
    );
```

> **返回值：** 实际复制到缓冲区的字符数（不包括终止符 `\0`）。若文件或节不存在，返回 `nSize - 1`（缓冲区被填满）。

---

## 二、写入配置文件示例

将配置参数分节写入 `test.ini` 文件：

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    // 1. 获取当前程序所在目录
    char szDirPath[MAX_PATH] = { 0 };
    GetCurrentDirectory(MAX_PATH, szDirPath);

    // 2. 构造完整的 INI 文件路径
    char szFilePath[MAX_PATH] = { 0 };
    sprintf(szFilePath, "%s\\test.ini", szDirPath);

    // 3. 写入配置项
    WritePrivateProfileString(
        "Param1",          // 节名
        "QueryInterval",   // 键名
        "3600",            // 值
        szFilePath         // 文件路径
    );

    WritePrivateProfileString(
        "Param1",          // 同一节
        "CheckInterval",   // 键名
        "4000",            // 值
        szFilePath
    );

    WritePrivateProfileString(
        "Param2",          // 不同的节
        "PopupInterval",   // 键名
        "3000",            // 值
        szFilePath
    );
}
```

**生成的文件内容（test.ini）：**

```ini
[Param1]
QueryInterval=3600
CheckInterval=4000

[Param2]
PopupInterval=3000
```

---

## 三、读取配置文件示例

从 `test.ini` 中读取所有配置项并拼接显示：

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    // 1. 获取程序目录并构造文件路径
    char szDirPath[MAX_PATH] = { 0 };
    GetCurrentDirectory(MAX_PATH, szDirPath);

    char szFilePath[MAX_PATH] = { 0 };
    sprintf(szFilePath, "%s\\test.ini", szDirPath);

    // 2. 准备接收各配置项的缓冲区
    char szQueryInterval[1024] = { 0 };
    char szCheckInterval[1024] = { 0 };
    char szPopupInterval[1024] = { 0 };

    // 3. 读取各配置项
    GetPrivateProfileString(
        "Param1",            // 节名
        "QueryInterval",     // 键名
        NULL,                // 默认值（键不存在时返回空）
        szQueryInterval,     // 接收缓冲区
        1024,                // 缓冲区大小
        szFilePath           // 文件路径
    );

    GetPrivateProfileString(
        "Param1",
        "CheckInterval",
        NULL,
        szCheckInterval,
        1024,
        szFilePath
    );

    GetPrivateProfileString(
        "Param2",
        "PopupInterval",
        NULL,
        szPopupInterval,
        1024,
        szFilePath
    );

    // 4. 拼接并显示所有配置信息
    char szDisplayMsg[1024] = { 0 };
    sprintf(szDisplayMsg, 
            "QueryInterval = %s\nCheckInterval = %s\nPopupInterval = %s", 
            szQueryInterval, 
            szCheckInterval, 
            szPopupInterval);
    
    MessageBox(szDisplayMsg);
}
```

---

## 四、实用补充

### 4.1 常用场景

- **程序启动时加载配置**：在 `InitInstance` 或窗口初始化时读取 INI 文件
- **保存用户设置**：程序退出时调用 `WritePrivateProfileString` 保存配置
- **多语言支持**：根据不同语言环境读取不同的 INI 文件
- **模块化配置**：将不同模块的配置分节存储

### 4.2 注意事项

1. **文件路径**：建议使用完整的绝对路径，避免因工作目录变化导致找不到文件
2. **缓冲区大小**：确保缓冲区足够大，避免读取被截断
3. **默认值**：合理设置 `lpDefault` 参数，当配置项缺失时程序仍能正常工作
4. **字符编码**：上述 API 为 ANSI 版本（`A` 后缀），如需 Unicode 支持，可使用 `WritePrivateProfileStringW` / `GetPrivateProfileStringW`

### 4.3 删除配置项

```cpp
// 删除指定键
WritePrivateProfileString("Param1", "QueryInterval", NULL, szFilePath);

// 删除整个节
WritePrivateProfileString("Param1", NULL, NULL, szFilePath);
```
