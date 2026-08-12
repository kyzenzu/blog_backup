---
title: 【Windows编程】文件操作
date: 2026-08-06 19:55:43
tags:
  - C/C++
  - Windows编程
---

## 一、C语言文件操作

### 1.1 打开文件

```cpp
FILE* __cdecl fopen(
        _In_z_ char const* _FileName,  // 文件名
        _In_z_ char const* _Mode       // 打开模式
        );
```

```cpp
errno_t __cdecl fopen_s(
        _Outptr_result_nullonfailure_ FILE**      _Stream,   // 输出文件指针
        _In_z_                        char const* _FileName, // 文件名
        _In_z_                        char const* _Mode      // 打开模式
        );
```

**文件打开模式说明：**

| 模式 | 说明                                                         |
| :--: | :----------------------------------------------------------- |
| `r`  | 只读方式打开，文件必须存在，否则打开失败                     |
| `w`  | 只写方式打开，若文件存在则清空内容，若不存在则创建新文件     |
| `a`  | 追加写入，若文件存在则在末尾追加，若不存在则创建；写入前不移除 EOF 标记 |
| `r+` | 读写方式打开，文件必须存在                                   |
| `w+` | 读写方式打开，若文件存在则清空内容，若不存在则创建新文件     |
| `a+` | 读写追加模式，若文件存在则在末尾追加，若不存在则创建；追加前移除 EOF 标记，写入完成后恢复 |

### 1.2 写入文件

```cpp
size_t __cdecl fwrite(
        _In_reads_bytes_(_ElementSize * _ElementCount) void const* _Buffer,  // 数据缓冲区
        _In_                                           size_t      _ElementSize,   // 单个元素字节数
        _In_                                           size_t      _ElementCount,  // 元素个数
        _Inout_                                        FILE*       _Stream         // 文件指针
        );
```

**写文件示例：**

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    FILE* pFile = fopen("1.txt", "w");
    if (pFile == NULL) {
        MessageBox("文件打开失败");
        return;
    }
    char szBuf[1024] = "C语言文件操作";
    int iLen = strlen(szBuf);
    if (fwrite(szBuf, sizeof(char), iLen + 1, pFile) <= 0) {
        MessageBox("文件写入失败");
        return;
    }
    fclose(pFile);
}
```

### 1.3 读取文件

**文件指针定位函数：**

```cpp
int __cdecl fseek(
        _Inout_ FILE* _Stream,  // 文件指针
        _In_    long  _Offset,  // 偏移量（字节数）
        _In_    int   _Origin   // 起始参照位置
        );

long __cdecl ftell(
        _Inout_ FILE* _Stream   // 文件指针
        );
```

**`_Origin` 参数取值：**

| 常量       |  值  | 含义               |
| :--------- | :--: | :----------------- |
| `SEEK_SET` |  0   | 从文件开头开始偏移 |
| `SEEK_CUR` |  1   | 从当前位置开始偏移 |
| `SEEK_END` |  2   | 从文件末尾开始偏移 |

**返回值：** `fseek` 返回 `0` 表示定位成功，非 `0` 表示定位失败。

> **使用技巧：** 利用 `fseek` + `ftell` 组合获取文件大小——先将文件指针移到文件末尾，再通过 `ftell` 获取当前偏移量即为文件大小。

**读文件示例：**

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    FILE* pFile = fopen("1.txt", "r");
    if (pFile == NULL) {
        MessageBox("文件打开失败");
        return;
    }
    
    // 获取文件长度
    char szBuf[1024] = { 0 };
    fseek(pFile, 0, SEEK_END);
    int iSize = ftell(pFile);
    // 将文件指针重置到开头
    fseek(pFile, 0, SEEK_SET);
    
    if (fread(szBuf, sizeof(char), iSize, pFile) <= 0) {
        MessageBox("文件读取失败");
        return;
    }

    fclose(pFile);
    MessageBox(szBuf);
}
```

---

## 二、C++ 文件操作

### 2.1 写入文件

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    std::ofstream ofs("2.txt");
    char szBuf[1024] = "C++文件操作";
    int iLen = strlen(szBuf);
    ofs.write(szBuf, iLen + 1);
    ofs.close();
}
```

### 2.2 读取文件

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    std::ifstream ifs("2.txt");
    char szBuf[1024] = { 0 };
    ifs.read(szBuf, 1024);
    ifs.close();
    MessageBox(szBuf);
}
```

### 2.3 打开模式说明

| 模式             | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| `ios::app`       | 追加模式，所有写入操作都在文件末尾进行，即使使用 `seekp` 移动指针也无效 |
| `ios::ate`       | 打开时定位到文件末尾，后续写入从当前位置开始                 |
| `ios::in`        | 以输入（读取）方式打开，文件内容不会被截断                   |
| `ios::out`       | 以输出（写入）方式打开，是 `ofstream` 的默认模式             |
| `ios::trunc`     | 若文件存在则清空内容。当指定 `ios::out` 且**未同时指定** `ios::app`、`ios::ate` 或 `ios::in` 时，该模式会被默认启用 |
| `ios::nocreate`  | 若文件不存在则打开失败                                       |
| `ios::noreplace` | 若文件已存在则打开失败                                       |
| `ios::binary`    | 以二进制模式打开（默认为文本模式）                           |

### 2.4 共享模式（`filebuf`）

| 取值                 | 说明                 |
| :------------------- | :------------------- |
| `filebuf::sh_compat` | 兼容共享模式         |
| `filebuf::sh_none`   | 独占模式，不允许共享 |
| `filebuf::sh_read`   | 允许其他进程读取     |
| `filebuf::sh_write`  | 允许其他进程写入     |

---

## 三、Windows API 文件操作

> Windows API 是对 C 语言文件操作的底层封装，支持更多功能（文件、管道、邮槽、通信资源、磁盘设备、控制台、目录等）。

### 3.1 CreateFile

```cpp
WINBASEAPI
HANDLE
WINAPI
CreateFile(
    _In_ LPCSTR lpFileName,                    // 文件名或设备名
    _In_ DWORD dwDesiredAccess,                // 访问权限：GENERIC_READ / GENERIC_WRITE / 0（查询）
    _In_ DWORD dwShareMode,                    // 共享模式：0 表示独占
    _In_opt_ LPSECURITY_ATTRIBUTES lpSecurityAttributes, // NULL 表示不能被子进程继承
    _In_ DWORD dwCreationDisposition,          // 创建方式：CREATE_NEW / CREATE_ALWAYS / OPEN_EXISTING
    _In_ DWORD dwFlagsAndAttributes,           // 文件属性与标志：FILE_ATTRIBUTE_NORMAL
    _In_opt_ HANDLE hTemplateFile              // 模板文件句柄，通常为 NULL
    );
```

### 3.2 WriteFile

```cpp
WINBASEAPI
BOOL
WINAPI
WriteFile(
    _In_ HANDLE hFile,                                     // 文件句柄
    _In_reads_bytes_opt_(nNumberOfBytesToWrite) LPCVOID lpBuffer, // 要写入的数据缓冲区
    _In_ DWORD nNumberOfBytesToWrite,                      // 要写入的字节数
    _Out_opt_ LPDWORD lpNumberOfBytesWritten,              // 实际写入的字节数
    _Inout_opt_ LPOVERLAPPED lpOverlapped                  // 重叠 I/O，NULL 表示同步操作
    );
```

### 3.3 ReadFile

```cpp
WINBASEAPI
_Must_inspect_result_
BOOL
WINAPI
ReadFile(
    _In_ HANDLE hFile,                                     // 文件句柄
    _Out_writes_bytes_to_opt_(nNumberOfBytesToRead, *lpNumberOfBytesRead) __out_data_source(FILE) LPVOID lpBuffer, // 读取缓冲区
    _In_ DWORD nNumberOfBytesToRead,                       // 要读取的字节数
    _Out_opt_ LPDWORD lpNumberOfBytesRead,                 // 实际读取的字节数
    _Inout_opt_ LPOVERLAPPED lpOverlapped                  // 重叠 I/O，NULL 表示同步操作
    );
```

### 3.4 写文件示例

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    HANDLE hFile = CreateFile(
        "3.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );
    if (hFile == INVALID_HANDLE_VALUE) {
        TRACE("[ERROR]: %d\n", GetLastError());
        MessageBox("文件打开失败");
        return;
    }
    
    char szBuf[1024] = "Windows文件操作";
    DWORD dwWrite;
    WriteFile(hFile, szBuf, strlen(szBuf) + 1, &dwWrite, NULL);
    CloseHandle(hFile);
    TRACE("写入%d个字节\n", dwWrite);
}
```

### 3.5 读文件示例

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    HANDLE hFile = CreateFile(
        "3.txt",
        GENERIC_READ,
        0,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );
    if (hFile == INVALID_HANDLE_VALUE) {
        MessageBox("文件打开失败");
        return;
    }
    
    char szBuf[1024] = { 0 };
    DWORD dwRead;
    ReadFile(hFile, szBuf, 1024, &dwRead, NULL);
    CloseHandle(hFile);
    TRACE("读取%d个字节\n", dwRead);
    MessageBox(szBuf);
}
```

---

## 四、MFC 文件操作

> MFC 的 `CFile` 类是对 Windows API 文件操作的封装，提供了更简洁的面向对象接口。

### 4.1 基本读写操作

**写入文件：**

```cpp
void CMFCFileDlg::OnBnClickedButtonWrite()
{
    CFile file("4.txt", CFile::modeCreate | CFile::modeWrite);
    char szBuf[1024] = "MFC文件操作";
    int iLen = strlen(szBuf);
    file.Write(szBuf, iLen + 1);
    file.Close();
}
```

**读取文件：**

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    CFile file("4.txt", CFile::modeRead);
    char szBuf[1024] = { 0 };
    file.Read(szBuf, 1024);
    file.Close();
    MessageBox(szBuf);
}
```

### 4.2 使用文件对话框

```cpp
void CMFCFileDlg::OnBnClickedButtonRead()
{
    CFileDialog fileDlg(TRUE);
    fileDlg.m_ofn.lpstrTitle = "标题";
    fileDlg.m_ofn.lpstrFilter = "Text Files(*.txt)\0*.txt\0All Files(*.*)\0*.*\0\0";
    
    if (IDOK == fileDlg.DoModal())
    {
        CFile file(fileDlg.GetFileName(), CFile::modeRead);
        char szBuf[1024] = { 0 };
        DWORD dwFileLen = file.GetLength();
        file.Read(szBuf, dwFileLen);
        file.Close();
        MessageBox(szBuf);
    }
}
```

---

## 五、总结对比

| 方案            | 封装层次     | 特点                                             |
| :-------------- | :----------- | :----------------------------------------------- |
| **C 标准库**    | 最底层       | 跨平台、函数式接口，需要手动管理文件指针         |
| **C++ 标准库**  | 中等         | 面向对象、RAII 管理，使用 `iostream` 风格        |
| **Windows API** | 底层系统接口 | 功能最强大，支持高级特性（重叠 I/O、安全属性等） |
| **MFC CFile**   | 高层封装     | 面向对象、简洁易用，适合 MFC 应用程序开发        |

