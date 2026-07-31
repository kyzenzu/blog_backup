---
title: 【Windows编程】进程间通信——命名管道
date: 2026-07-26 16:00:21
tags:
  - C/C++
  - Windows编程
---

# Windows 命名管道（Named Pipe）编程笔记

## 一、核心API：CreateNamedPipeW



```c
WINBASEAPI
HANDLE
WINAPI
CreateNamedPipeW(
    _In_ LPCWSTR lpName,                        // 管道名称，格式: \\.\pipe\管道名
    _In_ DWORD dwOpenMode,                      // 打开模式
    _In_ DWORD dwPipeMode,                      // 管道模式
    _In_ DWORD nMaxInstances,                   // 最大实例数
    _In_ DWORD nOutBufferSize,                  // 输出缓冲区大小（字节），0表示系统自动分配
    _In_ DWORD nInBufferSize,                   // 输入缓冲区大小（字节），0表示系统自动分配
    _In_ DWORD nDefaultTimeOut,                 // 默认超时时间（毫秒）
    _In_opt_ LPSECURITY_ATTRIBUTES lpSecurityAttributes  // 安全属性，NULL使用默认权限
);
```



### 参数详解

| 参数                | 说明                                                         |
| :------------------ | :----------------------------------------------------------- |
| **lpName**          | 管道名称，格式必须为 `\\.\pipe\管道名`                       |
| **dwOpenMode**      | `PIPE_ACCESS_DUPLEX` — 双向通信 `PIPE_ACCESS_INBOUND` — 服务端只读 `PIPE_ACCESS_OUTBOUND` — 服务端只写 可组合 `| FILE_FLAG_OVERLAPPED` 启用异步模式 |
| **dwPipeMode**      | `PIPE_TYPE_BYTE` — 字节流模式 `PIPE_TYPE_MESSAGE` — 消息模式 `PIPE_READMODE_MESSAGE` — 按消息读取 `PIPE_WAIT` — 阻塞模式 |
| **nMaxInstances**   | 最大实例数：1~255 或 `PIPE_UNLIMITED_INSTANCES`（最多受系统资源限制） |
| **nDefaultTimeOut** | `0` — 使用系统默认（50ms） `NMPWAIT_WAIT_FOREVER` — 无限等待 |

------

## 二、命名管道内核对象模型



```text
                        内核中的命名管道对象
┌─────────────────────────────────────────────────────────────────────┐
│  命名管道: \\.\pipe\MyPipe                                        │
│  最大实例数: 5  当前实例数: 2                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  实例1 (已连接)        实例2 (已连接)        实例3 (空闲)   │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │ │
│  │  │  共享内存缓冲区   │  │  共享内存缓冲区   │  │  等待连接   │ │ │
│  │  │ [客户端↔服务端]  │  │ [客户端↔服务端]  │  │             │ │ │
│  │  │  服务端引用: 2   │  │  服务端引用: 2   │  │  引用: 1    │ │ │
│  │  │  客户端引用: 1   │  │  客户端引用: 1   │  │             │ │ │
│  │  └────────┬────────┘  └────────┬────────┘  └──────┬──────┘ │ │
│  └───────────┼────────────────────┼───────────────────┼────────┘ │
│              ↑                    ↑                   ↑          │
│         ┌────┴────┐         ┌────┴────┐        ┌────┴────┐     │
│         │服务端句柄│         │服务端句柄│        │服务端句柄│     │
│         │读+写权限 │         │读+写权限 │        │监听权限  │     │
│         └────┬────┘         └────┬────┘        └────┬────┘     │
│              │                    │                   │          │
│         服务端进程持有        服务端进程持有       服务端进程持有  │
└─────────────────────────────────────────────────────────────────────┘
                         ↑                     ↑
                         │ ConnectNamedPipe    │ CreateFile
                         │                     │
                    ┌────┴────┐           ┌────┴────┐
                    │客户端句柄│           │客户端句柄│
                    │读+写权限│           │读+写权限│
                    └────┬────┘           └────┬────┘
                         │                     │
                     客户端进程1            客户端进程2
                     (连接实例1)            (连接实例2)
```



### 核心要点

- **一个命名管道对象** 管理多个实例
- **每个实例** 拥有独立的共享内存缓冲区（双向）
- **服务端进程** 持有所有实例的服务端句柄
- **每个客户端** 持有对应实例的客户端句柄
- **空闲实例** 用于接受新连接

------

## 三、与匿名管道对比

| 对比项       | 匿名管道               | 命名管道                               |
| :----------- | :--------------------- | :------------------------------------- |
| **方向**     | 单向（写→读）          | 双向（客户端↔服务端）                  |
| **句柄数量** | 2个（读写分离）        | 每个连接2个（服务端句柄 + 客户端句柄） |
| **进程关系** | 父进程持有，子进程继承 | 服务端监听，客户端连接                 |
| **通信模式** | 一对一                 | 一对多（多客户端）                     |
| **跨网络**   | 不支持                 | 支持（需指定服务器名）                 |

------

## 四、完整代码示例

### 4.1 服务端代码



```cpp
HANDLE hNamedPipe = NULL;

// ==================== 创建并等待连接 ====================
void CNamedPipeServerDlg::OnBnClickedButtonCreate()
{
    // 1. 创建命名管道
    LPCWSTR szPipeName = TEXT("\\\\.\\pipe\\mypipe");
    hNamedPipe = CreateNamedPipe(
        szPipeName,
        PIPE_ACCESS_DUPLEX | FILE_FLAG_OVERLAPPED,  // 双向 + 异步
        PIPE_TYPE_BYTE,                              // 字节流模式
        1,                                           // 单实例
        1024,                                        // 输出缓冲区
        1024,                                        // 输入缓冲区
        0,                                           // 默认超时
        NULL                                         // 默认安全属性
    );
    if (hNamedPipe == INVALID_HANDLE_VALUE) {
        MessageBox(TEXT("命名管道创建失败"));
        CloseHandle(hNamedPipe);
        hNamedPipe = NULL;
        return;
    }

    // 2. 创建事件对象（用于异步等待）
    HANDLE hEvent = CreateEvent(NULL, TRUE, FALSE, NULL);
    if (hEvent == INVALID_HANDLE_VALUE) {
        MessageBox(TEXT("事件创建失败"));
        CloseHandle(hEvent);
        return;
    }

    // 3. 异步等待客户端连接
    OVERLAPPED overlapped = { 0 };
    overlapped.hEvent = hEvent;

    if (!ConnectNamedPipe(hNamedPipe, &overlapped)) {
        if (GetLastError() != ERROR_IO_PENDING) {
            MessageBox(TEXT("等待客户端连接失败"));
            CloseHandle(hNamedPipe);
            CloseHandle(hEvent);
            hNamedPipe = NULL;
            return;
        }
    }

    // 4. 等待连接完成
    if (WaitForSingleObject(hEvent, INFINITE) == WAIT_FAILED) {
        MessageBox(TEXT("等待对象失败"));
        CloseHandle(hNamedPipe);
        CloseHandle(hEvent);
        hNamedPipe = NULL;
        return;
    }

    CloseHandle(hEvent);
    MessageBox(TEXT("客户端已连接！"));
}

// ==================== 发送数据 ====================
void CNamedPipeServerDlg::OnBnClickedButtonSend()
{
    if (hNamedPipe == NULL) {
        MessageBox(TEXT("请先建立管道连接"));
        return;
    }

    char szSendBuf[100] = "我是服务端";
    DWORD dwWriteNum;

    if (!WriteFile(hNamedPipe, szSendBuf, strlen(szSendBuf) + 1, &dwWriteNum, NULL)) {
        MessageBox(TEXT("发送失败"));
        return;
    }
}

// ==================== 接收数据 ====================
void CNamedPipeServerDlg::OnBnClickedButtonRecv()
{
    if (hNamedPipe == NULL) {
        MessageBox(TEXT("请先建立管道连接"));
        return;
    }

    char szRecvBuf[100] = { 0 };
    DWORD dwReadNum;

    if (!ReadFile(hNamedPipe, szRecvBuf, 100, &dwReadNum, NULL)) {
        MessageBox(TEXT("接收失败"));
        return;
    }

    TRACE("dwReadNum = %d\n", dwReadNum);
    MessageBox(szRecvBuf);
}
```



### 4.2 客户端代码

```cpp
HANDLE hNamedPipe = NULL;

// ==================== 连接管道 ====================
void CNamedPipeClientDlg::OnBnClickedButtonConnect()
{
    LPCWSTR szPipeName = TEXT("\\\\.\\pipe\\mypipe");

    // 1. 等待管道可用
    if (WaitNamedPipe(szPipeName, NMPWAIT_WAIT_FOREVER) == 0) {
        MessageBox(TEXT("当前没有可利用的管道"));
        return;
    }

    // 2. 打开命名管道
    hNamedPipe = CreateFile(
        szPipeName,
        GENERIC_READ | GENERIC_WRITE,  // 读写权限
        0,                              // 独占访问
        NULL,                           // 默认安全属性
        OPEN_EXISTING,                  // 必须已存在
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hNamedPipe == INVALID_HANDLE_VALUE) {
        TRACE("CreateFile failed with %d\n", GetLastError());
        MessageBox(TEXT("打开命名管道失败！"));
        hNamedPipe = NULL;
        return;
    }

    MessageBox(TEXT("已连接到服务端！"));
}

// ==================== 发送数据 ====================
void CNamedPipeClientDlg::OnBnClickedButtonSend()
{
    if (hNamedPipe == NULL) {
        MessageBox(TEXT("请先连接管道"));
        return;
    }

    char szSendBuf[100] = "我是客户端";
    DWORD dwWriteNum;

    if (!WriteFile(hNamedPipe, szSendBuf, strlen(szSendBuf) + 1, &dwWriteNum, NULL)) {
        MessageBox(TEXT("发送失败"));
        return;
    }
}

// ==================== 接收数据 ====================
void CNamedPipeClientDlg::OnBnClickedButtonRecv()
{
    if (hNamedPipe == NULL) {
        MessageBox(TEXT("请先连接管道"));
        return;
    }

    char szRecvBuf[100] = { 0 };
    DWORD dwReadNum;

    if (!ReadFile(hNamedPipe, szRecvBuf, 100, &dwReadNum, NULL)) {
        MessageBox(TEXT("接收失败"));
        return;
    }

    TRACE("dwReadNum = %d\n", dwReadNum);
    MessageBox(szRecvBuf);
}
```



------

## 五、关键流程总结



```text
┌──────────────┐                    ┌──────────────┐
│   服务端      │                    │   客户端      │
├──────────────┤                    ├──────────────┤
│ 1. CreateNamedPipe │              │              │
│    ↓          │                    │              │
│ 2. ConnectNamedPipe│               │ 1. WaitNamedPipe  │
│    (等待连接)  │                    │    (等待管道)  │
│    ↓          │                    │    ↓          │
│ 3. 连接成功    │ ←── 建立连接 ───→ │ 2. CreateFile │
│    ↓          │                    │    (打开管道)  │
│ 4. WriteFile / │ ←── 数据通信 ───→ │ 4. WriteFile / │
│    ReadFile   │                    │    ReadFile   │
│    ↓          │                    │    ↓          │
│ 5. CloseHandle│                    │ 5. CloseHandle│
└──────────────┘                    └──────────────┘
```



------

## 六、注意事项

| 序号 | 注意事项                                                     |
| :--- | :----------------------------------------------------------- |
| 1    | 管道名称必须以 `\\.\pipe\` 开头                              |
| 2    | 服务端和客户端的 `PIPE_TYPE` 模式必须一致                    |
| 3    | `FILE_FLAG_OVERLAPPED` 异步模式下，必须配合 `OVERLAPPED` 结构使用 |
| 4    | 同步模式下 `ConnectNamedPipe` 会阻塞等待，需要单独开线程     |
| 5    | `WriteFile`/`ReadFile` 在异步模式下也需要配合 `OVERLAPPED`   |
| 6    | 多实例场景下，服务端需要为每个客户端创建独立的 `hPipe` 句柄  |
| 7    | 字符串传输时建议包含结尾的 `\0`，以便接收方能正确识别边界    |
