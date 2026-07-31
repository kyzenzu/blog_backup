---
title: 【Windows编程】Socket编程——TCP
date: 2026-07-27 12:52:03
tags:
  - C/C++
  - Windows编程
---

## Windows Socket TCP 编程笔记

### 1. 基本流程

#### 1.1 TCP 服务端流程

| 步骤 | 函数            | 说明                   |
| :--- | :-------------- | :--------------------- |
| 0    | `WSAStartup`    | 初始化 Winsock（必须） |
| 1    | `socket`        | 创建空 socket          |
| 2    | `bind`          | 绑定本地地址和端口     |
| 3    | `listen`        | 开始监听               |
| 4    | `accept`        | 接收客户端连接（阻塞） |
| 5    | `send` / `recv` | 收发数据               |
| 6    | `closesocket`   | 关闭 socket            |
| 7    | `WSACleanup`    | 清理 Winsock           |

#### 1.2 TCP 客户端流程

| 步骤 | 函数            | 说明                   |
| :--- | :-------------- | :--------------------- |
| 0    | `WSAStartup`    | 初始化 Winsock（必须） |
| 1    | `socket`        | 创建空 socket          |
| 2    | `connect`       | 连接服务器             |
| 3    | `send` / `recv` | 收发数据               |
| 4    | `closesocket`   | 关闭 socket            |
| 5    | `WSACleanup`    | 清理 Winsock           |

---

### 2. 核心 API 详解

#### 2.1 初始化与清理（含版本校验）

```cpp
WSADATA wsaData;
int iResult;
iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
if (iResult != 0) {
    printf("WSAStartup failed: %d\n", iResult);
    return 1;
}
// 校验请求的 Winsock 版本是否被支持
if (LOBYTE(wsaData.wVersion) != 2 || HIBYTE(wsaData.wVersion) != 2) {
    printf("Could not find a usable version of Winsock.dll\n");
    WSACleanup();
    return 1;
}
```

```cpp
// 清理 Winsock
WSACleanup();
```

**说明：**
- `MAKEWORD(2, 2)` 表示请求 Winsock 2.2 版本
- `WSAStartup` 成功后，通过 `wsaData.wVersion` 检查实际加载的版本
- `LOBYTE` / `HIBYTE` 分别取出版本号的低字节（主版本）和高字节（副版本）
- 版本不符时应调用 `WSACleanup` 清理后再返回

#### 2.2 创建 Socket

```cpp
SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);
```

| 参数   | 值            | 说明              |
| :----- | :------------ | :---------------- |
| 地址族 | `AF_INET`     | IPv4              |
| 类型   | `SOCK_STREAM` | 流式套接字（TCP） |
| 协议   | `0`           | 自动选择 TCP      |

#### 2.3 地址结构体

```cpp
SOCKADDR_IN addr;
addr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);     // 服务端：任意本机IP
addr.sin_addr.S_un.S_addr = inet_addr("127.0.0.1"); // 客户端：指定服务器IP
addr.sin_family = AF_INET;                          // IPv4
addr.sin_port = htons(6000);                       // 端口号（网络字节序）
```

**字节序转换：**

| 函数        | 作用                           |
| :---------- | :----------------------------- |
| `htonl`     | 32位主机序 → 网络序            |
| `htons`     | 16位主机序 → 网络序            |
| `ntohl`     | 32位网络序 → 主机序            |
| `ntohs`     | 16位网络序 → 主机序            |
| `inet_addr` | 点分十进制 → 32位网络字节序 IP |
| `inet_ntoa` | 32位网络字节序 IP → 点分十进制 |

#### 2.4 服务端核心函数

##### bind（绑定）

```cpp
bind(sockServer, (sockaddr*)&addrServer, sizeof(addrServer))
```

将 socket 绑定到本地地址和端口。

##### listen（监听）

```cpp
listen(sockServer, 5)
```

- 第一个参数：已绑定的 socket
- 第二个参数：**未决连接队列的最大长度**（已完成三次握手但尚未被 accept 的连接数）

##### accept（接收连接）

```cpp
SOCKET sockClient = accept(sockServer, (sockaddr*)&addrClient, &len)
```

- **返回值**：一个新的 socket，用于与客户端通信（称为**通信 socket**）
- `sockServer` 仍用于监听新的连接
- `sockServer` 的 TCP 四元组：`(本机IP, 本机端口, 0, 0)` → 不完整，仅用于监听
- `sockClient` 的 TCP 四元组：`(本机IP, 本机端口, 客户端IP, 客户端端口)` → 完整，用于通信

#### 2.5 客户端核心函数

##### connect（连接）

```cpp
connect(sockClient, (sockaddr*)&addrServer, sizeof(addrServer))
```

主动连接服务器，发起三次握手。

#### 2.6 收发数据

```cpp
// 发送
send(sockClient, sendBuf, strlen(sendBuf), 0);

// 接收
recv(sockClient, recvBuf, sizeof(recvBuf), 0);
```

| 参数   | 说明             |
| :----- | :--------------- |
| 第一个 | socket 句柄      |
| 第二个 | 数据缓冲区       |
| 第三个 | 数据长度         |
| 第四个 | 标志（通常为 0） |

---

### 3. 循环收发（处理数据不完整问题）

**核心问题**：`send` 和 `recv` 不保证一次发送/接收完整数据，需要循环处理。

```cpp
int MySocketRecv(int sock, char* buf, int dataSize) {
    // 循环接收
    int numsRecvSoFar = 0;
    int numsRemainingToRecv = dataSize;
    printf("enter MySocketRecv\n");
    while (true) {
        int bytesRead = recv(sock, &buf[numsRecvSoFar], numsRemainingToRecv, 0);
        printf("###bytesRead = %d,numsRecvSoFar = %d, numsRemainingToRecv = %d\n",
               bytesRead, numsRecvSoFar, numsRemainingToRecv);
        if (bytesRead == numsRemainingToRecv) {
            return 0;
        } else if (bytesRead > 0) {
            numsRecvSoFar += bytesRead;
            numsRemainingToRecv -= bytesRead;
            continue;
        } else if (bytesRead < 0 && errno == EAGAIN) {
            // 非阻塞模式：没有数据可读，继续循环
            continue;
        } else {
            return -1;
        }
    }
}
```

```cpp
int MySocketSend(int socketNum, unsigned char* data, unsigned dataSize)
{
    unsigned numBytesSentSoFar = 0;
    unsigned numBytesRemainingToSend = dataSize;

    while (1) {
        int bytesSend = send(socketNum, (char const*)(&data[numBytesSentSoFar]), numBytesRemainingToSend, 0/*flags*/);
        if (bytesSend == numBytesRemainingToSend) {
            return 0;
        } else if (bytesSend > 0) {
            numBytesSentSoFar += bytesSend;
            numBytesRemainingToSend -= bytesSend;
            continue;
        } else if ((bytesSend < 0) && (errno == 11)) {
            // 非阻塞模式：发送缓冲区满，继续循环
            continue;
        } else {
            return -1;
        }
    }
}
```

---

### 4. 服务端完整代码

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <WinSock2.h>

#pragma comment(lib, "ws2_32.lib")

int main() {
    printf("TCP Server\n");

    /*0. 初始化 Winsock（必须！）*/
    WSADATA wsaData;
    int iResult;
    iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup failed: %d\n", iResult);
        return 1;
    }
    if (LOBYTE(wsaData.wVersion) != 2 || HIBYTE(wsaData.wVersion) != 2) {
        printf("Could not find a usable version of Winsock.dll\n");
        WSACleanup(); // 清理资源
        return 1;
    }

    /*1.创建空socket*/
    SOCKET sockServer = socket(AF_INET, SOCK_STREAM, 0);
    if (sockServer == INVALID_SOCKET) {
        printf("socket error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*2.绑定本地地址*/
    SOCKADDR_IN addrServer;
    addrServer.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    addrServer.sin_family = AF_INET;
    addrServer.sin_port = htons(6000);
    if (bind(sockServer, (sockaddr*)&addrServer, sizeof(addrServer)) == SOCKET_ERROR) {
        printf("bind error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*3.监听*/
    if (listen(sockServer, 5) == SOCKET_ERROR) {
        // 用于listen的socket因为没有客户IP/port因此是不完整的四元组
        printf("listen error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*4.接收*/
    SOCKADDR_IN addrClient;
    int len = sizeof(addrClient);

    while (true) {
        // accept创建新的socket，用sockServer填充本地IP/port，用新客户填充目标IP/port。形成新的四元组
        SOCKET sockClient = accept(sockServer, (sockaddr*)&addrClient, &len);

        char sendBuf[100] = { 0 };
        sprintf_s(sendBuf, sizeof(sendBuf), "Welcome %s\n", inet_ntoa(addrClient.sin_addr));
        int iLen = send(sockClient, sendBuf, strlen(sendBuf), 0);

        char recvBuf[100] = { 0 };
        iLen = recv(sockClient, recvBuf, sizeof(recvBuf), 0);
        printf("recvBuf: %s\n", recvBuf);

        closesocket(sockClient);
    }

    /*5.关闭*/
    closesocket(sockServer);
    WSACleanup();
    system("pause");

    return 0;
}
```

---

### 5. 客户端完整代码

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <WinSock2.h>

#pragma comment(lib, "ws2_32.lib")

int main() {
    printf("TCP Client\n");

    /*0. 初始化 Winsock（必须！）*/
    WSADATA wsaData;
    int iResult;
    iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup failed: %d\n", iResult);
        return 1;
    }
    if (LOBYTE(wsaData.wVersion) != 2 || HIBYTE(wsaData.wVersion) != 2) {
        printf("Could not find a usable version of Winsock.dll\n");
        WSACleanup(); // 清理资源
        return 1;
    }

    /*1.创建空socket*/
    SOCKET sockClient = socket(AF_INET, SOCK_STREAM, 0);
    if (sockClient == INVALID_SOCKET) {
        printf("socket error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*2.直接连接服务器*/
    SOCKADDR_IN addrServer;
    addrServer.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");
    addrServer.sin_family = AF_INET;
    addrServer.sin_port = htons(6000);
    if (connect(sockClient, (sockaddr*)&addrServer, sizeof(addrServer)) == SOCKET_ERROR) {
        printf("connect error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*3.收发数据*/
    char recvBuf[100] = { 0 };
    int iLen = recv(sockClient, recvBuf, sizeof(recvBuf), 0);
    printf("recvBuf: %s\n", recvBuf);

    char sendBuf[100] = "Hello World";
    iLen = send(sockClient, sendBuf, strlen(sendBuf), 0);

    closesocket(sockClient);
    WSACleanup();
    system("pause");
    return 0;
}
```

---

### 6. WSAStartup 版本校验说明

| 字段                       | 含义                                           |
| :------------------------- | :--------------------------------------------- |
| `MAKEWORD(2, 2)`           | 构造版本号 2.2（高字节=副版本，低字节=主版本） |
| `wsaData.wVersion`         | 系统实际加载的 Winsock 版本                    |
| `LOBYTE(wsaData.wVersion)` | 获取主版本号（低字节）                         |
| `HIBYTE(wsaData.wVersion)` | 获取副版本号（高字节）                         |
| 版本不符时的处理           | 调用 `WSACleanup()` 清理后返回                 |

---

### 7. 关键点总结

| 要点                   | 说明                                                         |
| :--------------------- | :----------------------------------------------------------- |
| **WSAStartup**         | **必须**在所有 Socket 操作前调用，并校验版本号               |
| **WSAStartup 失败**    | 返回非 0 值，不应继续执行 Socket 操作                        |
| **版本校验**           | 请求的版本与实际版本不一致时需调用 `WSACleanup` 后退出       |
| **字节序转换**         | IP 和端口号需使用 `hton*` / `ntoh*` 转换                     |
| **服务端 socket 类型** | `sockServer` 用于监听，`sockClient` 用于通信（由 `accept` 创建） |
| **TCP 四元组**         | 完整连接由 `(源IP, 源端口, 目标IP, 目标端口)` 唯一标识       |
| **send/recv 不完整**   | 不能假设一次 `send`/`recv` 发送/接收完整数据，需要循环处理   |
| **errno 检查**         | 非阻塞模式下需检查 `errno == EAGAIN`（无数据可读）或 `errno == 11`（缓冲区满） |

