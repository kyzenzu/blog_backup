---
title: 【Windows编程】Socket编程——UDP
date: 2026-07-27 13:00:11
tags:
  - C/C++
  - Windows编程
---

## Windows Socket UDP 编程笔记

### 1. UDP 与 TCP 的核心区别

| 对比项      | UDP                                     | TCP                                         |
| :---------- | :-------------------------------------- | :------------------------------------------ |
| 连接方式    | **无连接**（不需要建立连接）            | 面向连接（需三次握手）                      |
| 通信函数    | `sendto` / `recvfrom`（需指定目标地址） | `send` / `recv`（连接后无需指定地址）       |
| 服务端流程  | socket → bind → recvfrom/sendto         | socket → bind → listen → accept → send/recv |
| 客户端流程  | socket → sendto/recvfrom                | socket → connect → send/recv                |
| 可靠性      | **不可靠**（丢包、乱序）                | 可靠（重传、有序）                          |
| 边界保留    | **保留消息边界**                        | 流式传输，无边界                            |
| socket 类型 | `SOCK_DGRAM`                            | `SOCK_STREAM`                               |

---

### 2. 基本流程

#### 2.1 UDP 服务端流程

| 步骤 | 函数                  | 说明                            |
| :--- | :-------------------- | :------------------------------ |
| 0    | `WSAStartup`          | 初始化 Winsock（必须）          |
| 1    | `socket`              | 创建 UDP socket（`SOCK_DGRAM`） |
| 2    | `bind`                | 绑定本地地址和端口              |
| 3    | `recvfrom` / `sendto` | 收发数据（含客户端地址）        |
| 4    | `closesocket`         | 关闭 socket                     |
| 5    | `WSACleanup`          | 清理 Winsock                    |

#### 2.2 UDP 客户端流程

| 步骤 | 函数                  | 说明                            |
| :--- | :-------------------- | :------------------------------ |
| 0    | `WSAStartup`          | 初始化 Winsock（必须）          |
| 1    | `socket`              | 创建 UDP socket（`SOCK_DGRAM`） |
| 2    | `sendto` / `recvfrom` | 收发数据（需指定服务器地址）    |
| 3    | `closesocket`         | 关闭 socket                     |
| 4    | `WSACleanup`          | 清理 Winsock                    |

> **注意**：UDP 客户端通常不需要 `bind`，系统会自动分配端口。

---

### 3. 核心 API 详解

#### 3.1 创建 Socket（UDP）

```cpp
SOCKET sock = socket(AF_INET, SOCK_DGRAM, 0);
```

| 参数   | 值           | 说明                |
| :----- | :----------- | :------------------ |
| 地址族 | `AF_INET`    | IPv4                |
| 类型   | `SOCK_DGRAM` | 数据报套接字（UDP） |
| 协议   | `0`          | 自动选择 UDP        |

#### 3.2 发送数据：sendto

```cpp
sendto(sockClient, sendBuf, strlen(sendBuf) + 1, 0, (SOCKADDR*)&addrServer, len);
```

| 参数   | 说明                                           |
| :----- | :--------------------------------------------- |
| 第一个 | socket 句柄                                    |
| 第二个 | 发送数据缓冲区                                 |
| 第三个 | 数据长度（含 `\0` 可确保接收端正确识别字符串） |
| 第四个 | 标志（通常为 0）                               |
| 第五个 | 目标地址结构体指针                             |
| 第六个 | 目标地址结构体大小                             |

**与 TCP 的 `send` 区别**：`sendto` 每次调用都需指定目标地址，因为 UDP 是无连接的。

#### 3.3 接收数据：recvfrom

```cpp
recvfrom(sockServer, recvBuf, sizeof(recvBuf), 0, (SOCKADDR*)&addrClient, &len);
```

| 参数   | 说明                                                   |
| :----- | :----------------------------------------------------- |
| 第一个 | socket 句柄                                            |
| 第二个 | 接收数据缓冲区                                         |
| 第三个 | 缓冲区大小                                             |
| 第四个 | 标志（通常为 0）                                       |
| 第五个 | 输出：发送方地址结构体指针                             |
| 第六个 | 输入输出：地址结构体大小（传入最大长度，传出实际长度） |

**与 TCP 的 `recv` 区别**：`recvfrom` 可获取数据发送方的地址信息，便于回复。

---

### 4. 服务端完整代码

```cpp
#include <iostream>
#include <stdlib.h>
#include <WinSock2.h>

#pragma comment(lib, "ws2_32.lib")

int main() {
    std::cout << "UDP Server" << std::endl;

    /*0. 初始化 Winsock（必须！）*/
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

    /*1.创建空socket*/
    SOCKET sockServer = socket(AF_INET, SOCK_DGRAM, 0);
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

    /*3.直接接收*/
    SOCKADDR_IN addrClient;
    int len = sizeof(addrClient);
    char recvBuf[100] = { 0 };
    char sendBuf[100] = { 0 };
    while (true) {
        recvfrom(sockServer, recvBuf, sizeof(recvBuf), 0, (SOCKADDR*)&addrClient, &len);
        std::cout << recvBuf << std::endl;

        sprintf_s(sendBuf, sizeof(sendBuf), "Ack: %s", recvBuf);
        sendto(sockServer, sendBuf, strlen(sendBuf) + 1, 0, (SOCKADDR*)&addrClient, len);
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
#include <iostream>
#include <stdlib.h>
#include <WinSock2.h>

#pragma comment(lib, "ws2_32.lib")

int main() {
    std::cout << "UDP Client" << std::endl;

    /*0. 初始化 Winsock（必须！）*/
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

    /*1.创建空socket*/
    SOCKET sockClient = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockClient == INVALID_SOCKET) {
        printf("socket error: %d\n", GetLastError());
        system("pause");
        return -1;
    }

    /*2.直接发送数据*/
    SOCKADDR_IN addrServer;
    addrServer.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");
    addrServer.sin_family = AF_INET;
    addrServer.sin_port = htons(6000);

    char recvBuf[100] = { 0 };
    char sendBuf[100] = "Hello World";
    int len = sizeof(addrServer);
    sendto(sockClient, sendBuf, strlen(sendBuf) + 1, 0, (SOCKADDR*)&addrServer, len);

    /*3.直接接收*/
    recvfrom(sockClient, recvBuf, sizeof(recvBuf), 0, (SOCKADDR*)&addrServer, &len);
    std::cout << recvBuf << std::endl;

    /*5.关闭*/
    closesocket(sockClient);
    WSACleanup();
    system("pause");
    return 0;
}
```

---

### 6. UDP 特有函数速查

| 函数       | 作用       | 关键参数                                            |
| :--------- | :--------- | :-------------------------------------------------- |
| `sendto`   | 发送数据报 | 目标地址 `(SOCKADDR*)` 和地址长度                   |
| `recvfrom` | 接收数据报 | 输出发送方地址 `(SOCKADDR*)` 和地址长度（传入传出） |

---

### 7. UDP 关键点总结

| 要点            | 说明                                                     |
| :-------------- | :------------------------------------------------------- |
| **无连接**      | UDP 不需要 `connect`/`listen`/`accept`，可直接收发       |
| **消息边界**    | 每次 `recvfrom` 对应一个完整的 `sendto` 数据报           |
| **地址信息**    | `recvfrom` 可获取发送方地址，用于 `sendto` 回复          |
| **不可靠性**    | UDP 不保证交付、顺序，应用层需自行处理丢包和重传         |
| **socket 类型** | `SOCK_DGRAM`（数据报）而非 `SOCK_STREAM`                 |
| **客户端 bind** | 通常不需要显式 `bind`，系统自动分配端口                  |
| **长度含 `\0`** | 发送字符串时 `strlen + 1` 可确保接收端正确识别字符串结束 |

