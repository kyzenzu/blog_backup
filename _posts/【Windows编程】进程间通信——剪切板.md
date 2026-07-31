---
title: 【Windows编程】进程间通信——剪切板
date: 2026-07-27 10:12:03
tags:
  - C/C++
  - Windows编程
---

收到，整理后的剪切板笔记如下（代码中所有原有注释完整保留）：

---

## Windows 剪切板（Clipboard）进程间通信笔记

### 1. 基本概念

Windows 提供了访问操作系统剪切板的 API，因此可以以**操作系统的剪切板**为中间媒介，实现进程间通信：

- **发送端**：将数据写入剪切板
- **接收端**：从剪切板读取数据

---

### 2. 核心 API 流程

| 步骤 | 发送端（写入）                        | 接收端（读取）                          |
| ---- | ------------------------------------- | --------------------------------------- |
| 1    | `OpenClipboard()` 打开剪切板          | `OpenClipboard()` 打开剪切板            |
| 2    | `EmptyClipboard()` 清空剪切板         | `IsClipboardFormatAvailable()` 检查格式 |
| 3    | `GlobalAlloc()` 分配全局内存          | `GetClipboardData()` 获取数据           |
| 4    | `GlobalLock()` 锁定内存并写入数据     | `GlobalLock()` 锁定内存并读取数据       |
| 5    | `GlobalUnlock()` 解锁内存             | `GlobalUnlock()` 解锁内存               |
| 6    | `SetClipboardData()` 将数据放入剪切板 | `SetDlgItemText()` 显示到界面           |
| 7    | `CloseClipboard()` 关闭剪切板         | `CloseClipboard()` 关闭剪切板           |

---

### 3. 发送端代码示例

```cpp
void CClicpBoardDlg::OnBnClickedButtonSend()
{
    /* 1.打开剪切板 */
    if (OpenClipboard()) {
        /* 2.清空剪切板 */
        EmptyClipboard();

        /* 3.获取编辑框内容 */
        CString strEditSend;
        GetDlgItemText(IDC_EDIT_SEND, strEditSend);

        /* 4.申请一块全局内存 */
        HANDLE hGlobalMem = GlobalAlloc(GMEM_MOVEABLE, strEditSend.GetLength() + 1);

        /* 5.把编辑框内容移到全局空间中 */
        char* szSendBuf = (char*)GlobalLock(hGlobalMem); // 锁定内存：为了防止系统为了腾出空间而移动或释放你正在使用的内存, 返回相应空间的指针
        strcpy(szSendBuf, strEditSend);
        GlobalUnlock(hGlobalMem);

        /* 6.把内容从全局空间移到剪切板 */
        SetClipboardData(CF_TEXT, hGlobalMem);

        /* 7.关闭剪切板 */
        CloseClipboard();
    }
}
```

---

### 4. 接收端代码示例

```cpp
void CClicpBoardDlg::OnBnClickedButtonRecv()
{
    /* 1.打开剪切板 */
    if (OpenClipboard()) {
        /* 2.检查剪切板是否可用 */
        if (IsClipboardFormatAvailable(CF_TEXT)) {
            /* 3. 从剪切板获取内容 */
            HANDLE hGlobalMem = GetClipboardData(CF_TEXT);

            /* 4. 将剪切板的内容移到编辑框 */
            char* szRecvBuf = (char*)GlobalLock(hGlobalMem);
            SetDlgItemText(IDC_EDIT_RECV, szRecvBuf);
            GlobalUnlock(hGlobalMem);
        }
        /* 5.关闭剪切板 */
        CloseClipboard();
    }
}
```

---

### 5. 关键点总结

| 要点       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 通信媒介   | 操作系统全局剪切板                                           |
| 通信方向   | **单向**（发送端写入，接收端读取）                           |
| 数据格式   | 通过格式标识符指定（如 `CF_TEXT` 表示文本）                  |
| 内存管理   | 使用 `GlobalAlloc` / `GlobalLock` / `GlobalUnlock` 管理全局内存 |
| 数据所有权 | 调用 `SetClipboardData` 后，剪切板拥有内存对象，发送端**不应**再释放 |
| 注意事项   | • 操作前必须先 `OpenClipboard()`<br>• 操作完成后必须 `CloseClipboard()`<br>• 发送端写入前应先 `EmptyClipboard()` 清空旧数据 |

---

### 6. 注意事项补充

| 注意点     | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 剪切板占用 | 一次只能有一个进程打开剪切板，操作期间其他进程无法访问       |
| 数据格式   | `CF_TEXT` 是 ANSI 文本，如需 Unicode 可使用 `CF_UNICODETEXT` |
| 内存句柄   | `SetClipboardData` 成功后，内存句柄归系统管理，无需手动 `GlobalFree` |
| 读取时机   | 接收端读取时需先检查格式是否可用，避免读取到非预期数据       |

