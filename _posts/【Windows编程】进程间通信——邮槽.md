---
title: 【Windows编程】进程间通信——邮槽
date: 2026-07-27 10:03:42
tags:
  - C/C++
  - Windows编程
---



## Windows 邮槽（Mailslot）笔记

### 1. 基本概念

- **邮槽**是 Windows 提供的一种**单向**进程间通信（IPC）机制。
- 通信双方分为**服务端**和**客户端**：
  - **服务端**：创建邮槽，获得邮槽句柄，**只能接收数据**。
  - **客户端**：通过邮槽名打开邮槽，获得句柄后，**只能发送数据**。
  - 邮槽通信是单向的，服务端只能接收数据，客户端只能发送数据。
- 消息传递遵循 **FIFO（先入先出）** 规则，客户端先写入的消息在服务端优先被读取。
- 邮槽支持**任意格式**的数据，但**单条消息最大不能超过 424 字节**。
- 邮槽不仅支持**本机**进程间通信，也支持**跨主机**通信。跨主机时数据通过 **UDP 协议**传输，属于**不可靠**通信，客户端需知道服务端的主机名或域名。

---

### 2. 核心 API：`CreateMailslotW`

#### 2.1 函数原型

```cpp
WINBASEAPI
HANDLE
WINAPI
CreateMailslotW(
    _In_     LPCWSTR lpName,                          // 邮槽名称（必须指定）
    _In_     DWORD nMaxMessageSize,                   // 单条消息的最大字节数
    _In_     DWORD lReadTimeout,                     // 读取操作超时时间（毫秒）
    _In_opt_ LPSECURITY_ATTRIBUTES lpSecurityAttributes // 安全属性（可选）
);
```

#### 2.2 参数详解

| 参数                   | 类型                    | 说明                                    | 默认值/取值范围               | 备注                                                         |
| ---------------------- | ----------------------- | --------------------------------------- | ----------------------------- | ------------------------------------------------------------ |
| `lpName`               | `LPCWSTR`               | 邮槽全局唯一名称                        | **无默认值，必须指定**        | 本地：`\\.\mailslot\<名称>`<br>远程：`\\<服务器名>\mailslot\<名称>`<br>不区分大小写 |
| `nMaxMessageSize`      | `DWORD`                 | 允许写入的单条消息最大字节数            | `0` 表示无限制                | 实际建议 ≤ 424 字节；超过此值写入失败                        |
| `lReadTimeout`         | `DWORD`                 | `ReadFile` 读取时的等待超时时间（毫秒） | `MAILSLOT_WAIT_FOREVER`（-1） | • `-1`：无限等待<br>• `0`：立即返回<br>• 正值：等待指定毫秒数 |
| `lpSecurityAttributes` | `LPSECURITY_ATTRIBUTES` | 安全描述符和句柄继承属性                | `NULL`                        | `NULL` = 默认安全属性，句柄不可继承                          |

#### 2.3 返回值

| 情况     | 返回值                 | 后续处理                            |
| -------- | ---------------------- | ----------------------------------- |
| 创建成功 | 有效句柄（`HANDLE`）   | 可用于 `ReadFile`、`CloseHandle` 等 |
| 创建失败 | `INVALID_HANDLE_VALUE` | 调用 `GetLastError()` 获取错误码    |

---

### 3. 服务端代码示例（接收端）

```cpp
void CMailSlotServerDlg::OnBnClickedButtonRecv()
{
    // 0. 定义邮槽名称
    // 格式必须为: \\.\mailslot\[路径\]名称
    // \\. 表示本机
    LPCTSTR szMailSlotName = TEXT("\\\\.\\mailslot\\MyMailSlot");

    /* 1.服务端通过这个接口创建邮槽 */
    HANDLE hMailSlot = CreateMailslot(
        szMailSlotName,
        0,                      // 可以写入mailslot的单个邮件的最大大小（以字节为单位）。若要指定消息可以是任意大小，请将此值设置为0。
        MAILSLOT_WAIT_FOREVER,  // 无限等待
        NULL                    // 默认安全属性
    );

    if (hMailSlot == INVALID_HANDLE_VALUE) {
        TRACE("CreateMailslot Failed with: %d", GetLastError());
        return;
    }

    /* 2.用文件API从邮槽读取消息 */
    char szRecvBuf[100] = { 0 };
    DWORD dwReadNum;
    if (!ReadFile(hMailSlot, szRecvBuf, 100, &dwReadNum, NULL)) {
        MessageBox("Read Failed");
        CloseHandle(hMailSlot);
        return;
    }

    TRACE("dwReadNum = %d\n", dwReadNum);
    MessageBox(szRecvBuf);

    // 关闭句柄
    CloseHandle(hMailSlot);
}
```

---

### 4. 客户端代码示例（发送端）

```cpp
void CMailSlotClientDlg::OnBnClickedButtonSend()
{
    // 指定服务端邮槽名称（本机示例，跨主机时替换计算机名）
    LPCTSTR szMailSlotName = TEXT("\\\\.\\mailslot\\MyMailSlot");

    // 打开已存在的邮槽
    HANDLE hMailSlot = CreateFile(
        szMailSlotName,
        GENERIC_WRITE,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hMailSlot == INVALID_HANDLE_VALUE) {
        TRACE("Open Mailslot Failed with: %d", GetLastError());
        return;
    }

    // 向邮槽写入数据
    char szSendBuf[100] = "我是客户端消息";
    DWORD dwWriteNum;
    if (!WriteFile(hMailSlot, szSendBuf, strlen(szSendBuf) + 1, &dwWriteNum, NULL)) {
        MessageBox("Write Failed");
        CloseHandle(hMailSlot);
        return;
    }

    // 关闭句柄
    CloseHandle(hMailSlot);
}
```

---

### 5. 关键点总结

| 特性           | 说明                                            |
| -------------- | ----------------------------------------------- |
| 通信方向       | **单向**：服务端只能接收，客户端只能发送        |
| 消息顺序       | **FIFO**（先入先出）                            |
| 消息大小限制   | 单条消息 ≤ **424 字节**                         |
| 跨主机支持     | 支持，基于 **UDP 协议**，**不可靠**             |
| 服务端创建方式 | `CreateMailslot`                                |
| 客户端打开方式 | `CreateFile`（指定 `OPEN_EXISTING`）            |
| 数据读写方式   | 使用 `ReadFile` / `WriteFile`（与文件操作一致） |

