---
title: 【Windows编程】进程间通信——WM_COPYDATA
date: 2026-07-26 17:24:58
tags:
  - C/C++
  - Windows编程
---

# Windows进程间通信：WM_COPYDATA 实现笔记

## 一、核心数据结构：COPYDATASTRUCT

```c
typedef struct tagCOPYDATASTRUCT {
    ULONG_PTR dwData;   // 自定义数据类型标识（由接收方定义有效类型）
    DWORD cbData;        // lpData 指向的数据大小（字节数）
    PVOID lpData;        // 要传递的数据指针（可为 NULL）
} COPYDATASTRUCT, *PCOPYDATASTRUCT;
```

---

## 二、通信原理

1. **查找目标窗口**：发送方通过窗口标题获取接收方窗口句柄（`FindWindow`）
2. **封装与发送**：发送方将数据封装为 `COPYDATASTRUCT`，通过 `::SendMessage()` 发送 `WM_COPYDATA` 消息
3. **系统调度与接收**：Windows 操作系统根据窗口句柄将消息路由到目标窗口，调用其 `OnCopyData()` 回调函数

---

## 三、发送方实现

```cpp
void CWMCOPYDATASenderDlg::OnBnClickedButtonSend()
{
    CString strWindowTitle(_T("接收方"));
    CString strMsg(_T("我是发送方"));

    // 1. 获取接收方窗口句柄
    HWND hWnd = ::FindWindow(NULL, strWindowTitle);
    
    if (hWnd != INVALID_HANDLE_VALUE && IsWindow(hWnd)) 
    {
        // 2. 构造 COPYDATASTRUCT
        COPYDATASTRUCT copyDataStruct;
        copyDataStruct.dwData = 0;   // 自定义类型标识
        copyDataStruct.cbData = strMsg.GetLength() * sizeof(TCHAR);
        copyDataStruct.lpData = (PVOID)strMsg.GetString();

        // 3. 发送 WM_COPYDATA 消息
        ::SendMessage(hWnd, 
                      WM_COPYDATA, 
                      (WPARAM)(AfxGetApp()->m_pMainWnd), 
                      (LPARAM)&copyDataStruct);
    }
}
```

---

## 四、接收方实现

### 添加消息映射（两种方式）

**方式一（推荐）**：在接收方对话框设计窗口 → 右键 → 类向导 → 消息 → 选择 `WM_COPYDATA` → 双击自动生成

**方式二**：手动添加消息映射宏

```cpp
// 头文件中声明
afx_msg BOOL OnCopyData(CWnd* pWnd, COPYDATASTRUCT* pCopyDataStruct);

// 消息映射中添加
BEGIN_MESSAGE_MAP(CWMCOPYDATARecverDlg, CDialogEx)
    ON_WM_COPYDATA()
END_MESSAGE_MAP()
```

### 接收处理函数

```cpp
BOOL CWMCOPYDATARecverDlg::OnCopyData(CWnd* pWnd, COPYDATASTRUCT* pCopyDataStruct)
{
    // 提取数据
    LPCTSTR szData = (LPCTSTR)(pCopyDataStruct->lpData);
    DWORD dwLength = (DWORD)pCopyDataStruct->cbData;
    
    // 复制数据（防止指针失效）
    TCHAR szRecvText[1024] = { 0 };
    memcpy(szRecvText, szData, dwLength);
    
    // 业务处理
    MessageBox(szRecvText, _T("消息接收"), MB_OK);
    
    return CDialogEx::OnCopyData(pWnd, pCopyDataStruct);
}
```

---

## 五、关键注意事项

| 要点             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| **线程安全**     | 建议使用 `SendMessage`（阻塞直到接收方处理完成），`PostMessage` 可能导致数据被提前释放 |
| **Unicode/ANSI** | 使用 `TCHAR` 系列保证字符集兼容，计算大小时务必包含 `sizeof(TCHAR)` 因子 |
| **数据生命周期** | `lpData` 指向的数据在消息处理期间必须有效（发送方栈上或堆上均可） |
| **窗口查找**     | 确保窗口标题精确匹配，可通过 Spy++ 工具验证                  |
| **错误处理**     | 检查 `FindWindow` 返回值及 `IsWindow` 有效性                 |

---

## 六、适用场景

- ✅ 轻量级、低频率的进程间数据交换
- ✅ 同机进程通信（不支持跨机器）
- ✅ 自定义结构体或字符串传递
- ❌ 不适合大数据量或高频通信（有性能开销）



> WM_COPY_DATA 封装数据和解析数据。非常方便。如果数据量大，建议用命名管道。
