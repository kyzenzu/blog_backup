---
title: 【Windows编程】第一个Windows程序
date: 2026-07-27 13:08:16
tags:
  - C/C++
  - Windows编程
---

## Windows 窗口程序运行逻辑笔记（含完整代码）

### 1. 整体流程概览

一个 Windows 窗口程序的运行分为以下几个核心阶段：

| 阶段              | 核心操作                                              | 目的                                     |
| :---------------- | :---------------------------------------------------- | :--------------------------------------- |
| **1. 设计窗口类** | 填写 `WNDCLASS` 结构体                                | 定义窗口的"模板"（外观、行为、处理函数） |
| **2. 注册窗口类** | `RegisterClass`                                       | 将模板告知操作系统                       |
| **3. 创建窗口**   | `CreateWindow`                                        | 根据模板在内存中创建实际的窗口对象       |
| **4. 显示窗口**   | `ShowWindow` / `UpdateWindow`                         | 让窗口在屏幕上可见                       |
| **5. 消息循环**   | `GetMessage` → `TranslateMessage` → `DispatchMessage` | 持续接收并分发消息                       |
| **6. 窗口过程**   | `MyWinProc`                                           | 处理具体消息（由系统回调）               |

---

### 2. 为什么需要窗口类（WNDCLASS）？

**核心原因**：Windows 采用**面向对象的设计思想**，但 C 语言不支持类和继承，因此通过结构体来模拟"类"的概念。

`WNDCLASS` 相当于一个**窗口模板/蓝图**，它描述了：
- 窗口长什么样（图标、光标、背景色、样式）
- 窗口的行为由谁处理（`lpfnWndProc` 指向窗口过程函数）
- 窗口属于哪个应用程序（`hInstance`）

#### 2.1 WNDCLASS 结构体详解

```cpp
typedef struct _WNDCLASS {
    UINT      style;          // 窗口类样式，指定窗口类的行为特性
                              // 可选值：CS_HREDRAW（水平重绘）、CS_VREDRAW（垂直重绘）
                              //         CS_DBLCLKS（双击消息）、CS_NOCLOSE（禁用关闭）等
                              // 默认值：0（无特殊样式）
    
    WNDPROC   lpfnWndProc;    // 窗口过程函数指针，处理所有发给该窗口的消息
                              // 必须指定，不能为 NULL
    
    int       cbClsExtra;     // 类额外内存大小（字节），用于存储该类所有窗口共享的数据
                              // 默认值：0（不分配额外内存）
    
    int       cbWndExtra;     // 窗口额外内存大小（字节），用于存储该窗口实例的私有数据
                              // 默认值：0（不分配额外内存）
    
    HINSTANCE hInstance;      // 拥有该窗口类的应用程序实例句柄
                              // 必须指定，不能为 NULL
    
    HICON     hIcon;          // 窗口类的默认图标，显示在标题栏和任务栏
                              // LoadIcon(NULL, IDI_APPLICATION) 获取默认应用程序图标
                              // 默认值：NULL（无图标）
    
    HCURSOR   hCursor;        // 窗口类的默认光标，鼠标移动到客户区时显示
                              // LoadCursor(NULL, IDC_ARROW) 获取标准箭头光标
                              // 默认值：NULL（无光标）
    
    HBRUSH    hbrBackground;  // 窗口类的默认背景画刷，用于擦除客户区背景
                              // GetStockObject(WHITE_BRUSH) 获取白色画刷
                              // 也可用 (HBRUSH)(COLOR_WINDOW + 1)
                              // 默认值：NULL（不擦除背景）
    
    LPCWSTR   lpszMenuName;   // 窗口类默认菜单的资源名称，NULL 表示无菜单
                              // 默认值：NULL（无菜单）
    
    LPCWSTR   lpszClassName;  // 窗口类的名称，用于标识该类
                              // 必须指定，不能为 NULL
} WNDCLASS;
```

---

### 3. 为什么需要注册窗口类（RegisterClass）？

注册窗口类的本质是：**将设计好的模板告知操作系统内核**，让内核记录下来。

```cpp
RegisterClass(&wndcls);
```

- 注册后，系统内核会维护一个**窗口类表**，保存 `WNDCLASS` 中的信息（尤其是 `lpfnWndProc` 函数地址）。
- `CreateWindow` 时，通过类名（`className`）在表中查找对应的窗口类信息，从而创建窗口实例。

> **如果不注册**：`CreateWindow` 找不到类名对应的注册信息，创建失败。

#### 3.1 RegisterClass 参数详解

```cpp
ATOM RegisterClass(
    _In_ CONST WNDCLASS *lpWndClass  // 指向 WNDCLASS 结构体的指针
);
```

| 参数         | 说明                                           |
| :----------- | :--------------------------------------------- |
| `lpWndClass` | 指向已填充好的 `WNDCLASS` 结构体的指针         |
| **返回值**   | 成功返回唯一标识该类的原子（ATOM），失败返回 0 |
| **失败原因** | 类名已注册、参数无效、内存不足等               |

---

### 4. 创建窗口（CreateWindow）发生了什么？

调用 `CreateWindow` 时，系统内核做以下几件事：

| 步骤 | 内核操作                                                    |
| :--- | :---------------------------------------------------------- |
| 1    | 通过 `className` 查找已注册的窗口类                         |
| 2    | 从窗口类中读取 `lpfnWndProc`、背景画刷、图标等属性          |
| 3    | 分配内核对象内存（窗口对象数据结构）                        |
| 4    | 在调用进程的**句柄表**中创建一项，返回 `HWND`（窗口句柄）   |
| 5    | 初始化窗口的引用计数等内核对象成员                          |
| 6    | **发送 `WM_CREATE` 消息**到窗口过程（允许应用程序做初始化） |

> **注意**：此时窗口尚未在屏幕上显示，只是内核中有了这个窗口对象。

#### 4.1 CreateWindow 参数详解

```cpp
HWND CreateWindow(
    _In_opt_ LPCWSTR lpClassName,      // 窗口类名称（必须已注册）
    _In_opt_ LPCWSTR lpWindowName,     // 窗口标题（显示在标题栏）
    _In_ DWORD dwStyle,                // 窗口样式（外观和行为）
    _In_ int X,                        // 窗口左上角 X 坐标（屏幕坐标）
    _In_ int Y,                        // 窗口左上角 Y 坐标
    _In_ int nWidth,                   // 窗口宽度（像素）
    _In_ int nHeight,                  // 窗口高度（像素）
    _In_opt_ HWND hWndParent,          // 父窗口句柄（NULL 表示无父窗口）
    _In_opt_ HMENU hMenu,              // 菜单句柄（NULL 表示无菜单）
    _In_opt_ HINSTANCE hInstance,      // 应用程序实例句柄
    _In_opt_ LPVOID lpParam            // 创建参数（通过 WM_CREATE 的 CREATESTRUCT 传递）
);
```

| 参数                 | 说明                         | 代码中的值                            |
| :------------------- | :--------------------------- | :------------------------------------ |
| `lpClassName`        | 已注册的窗口类名             | `className`（"MyClassName"）          |
| `lpWindowName`       | 窗口标题栏文本               | `windowName`（"MyWindowName"）        |
| `dwStyle`            | 窗口样式，决定窗口外观和行为 | `WS_OVERLAPPEDWINDOW`（标准重叠窗口） |
| `X` / `Y`            | 窗口左上角位置               | `CW_USEDEFAULT`（系统自动选择）       |
| `nWidth` / `nHeight` | 窗口宽高                     | `CW_USEDEFAULT`（系统自动选择）       |
| `hWndParent`         | 父窗口句柄                   | `NULL`（无父窗口，独立窗口）          |
| `hMenu`              | 菜单句柄                     | `NULL`（无菜单）                      |
| `hInstance`          | 应用程序实例句柄             | `hInstance`（WinMain 传入）           |
| `lpParam`            | 额外创建参数                 | `NULL`（无额外参数）                  |

#### 4.2 常用窗口样式（dwStyle）

| 样式常量              | 说明                                                        |
| :-------------------- | :---------------------------------------------------------- |
| `WS_OVERLAPPEDWINDOW` | 标准重叠窗口（含标题栏、系统菜单、边框、最小化/最大化按钮） |
| `WS_POPUP`            | 弹出式窗口（无标题栏边框，常用于对话框）                    |
| `WS_CHILD`            | 子窗口（必须指定父窗口）                                    |
| `WS_VISIBLE`          | 创建后立即可见（相当于自动调用 `ShowWindow`）               |

---

### 5. 显示窗口

```cpp
ShowWindow(hwnd, SW_SHOWNORMAL);  // 在屏幕上绘制窗口
UpdateWindow(hwnd);               // 立即发送 WM_PAINT 消息
```

#### 5.1 ShowWindow 参数详解

```cpp
BOOL ShowWindow(
    HWND hWnd,      // 窗口句柄
    int nCmdShow    // 显示方式
);
```

| `nCmdShow` 值   | 说明                       |
| :-------------- | :------------------------- |
| `SW_SHOWNORMAL` | 正常显示窗口（还原并激活） |
| `SW_SHOW`       | 显示窗口（不改变激活状态） |
| `SW_HIDE`       | 隐藏窗口                   |
| `SW_MAXIMIZE`   | 最大化显示                 |
| `SW_MINIMIZE`   | 最小化显示                 |
| `SW_RESTORE`    | 恢复到正常大小             |

#### 5.2 UpdateWindow 说明

```cpp
BOOL UpdateWindow(HWND hWnd);
```

- 立即发送 `WM_PAINT` 消息到窗口过程，强制刷新客户区
- 如果窗口没有需要重绘的内容，则不发送 `WM_PAINT`

---

### 6. 消息循环（Message Loop）

```cpp
MSG msg;
while (GetMessage(&msg, NULL, NULL, NULL)) {
    TranslateMessage(&msg);
    DispatchMessage(&msg);
}
return msg.wParam;
```

#### 6.1 MSG 结构体详解

```cpp
typedef struct tagMSG {
    HWND   hwnd;     // 接收消息的窗口句柄（目标窗口）
    UINT   message;  // 消息标识符（如 WM_PAINT、WM_CLOSE 等）
    WPARAM wParam;   // 消息的第一个附加参数（含义取决于消息类型）
    LPARAM lParam;   // 消息的第二个附加参数（含义取决于消息类型）
    DWORD  time;     // 消息被投递到队列的时间（GetTickCount 返回值）
    POINT  pt;       // 消息被投递时鼠标在屏幕上的位置（屏幕坐标）
} MSG;
```

#### 6.2 三个核心函数

##### GetMessage

```cpp
BOOL GetMessage(
    _Out_ LPMSG lpMsg,          // 指向 MSG 结构体，接收消息信息
    _In_opt_ HWND hWnd,         // 指定要接收消息的窗口（NULL 表示所有窗口）
    _In_ UINT wMsgFilterMin,    // 消息编号最小值（0 表示不过滤）
    _In_ UINT wMsgFilterMax     // 消息编号最大值（0 表示不过滤）
);
```

| 参数                | 说明                                                     |
| :------------------ | :------------------------------------------------------- |
| `lpMsg`             | 输出参数，接收取出的消息                                 |
| `hWnd`              | 指定只接收特定窗口的消息（NULL = 所有窗口）              |
| `wMsgFilterMin/Max` | 消息范围过滤（0 = 接收所有消息）                         |
| **返回值**          | `TRUE`（收到非 WM_QUIT 消息）<br>`FALSE`（收到 WM_QUIT） |

##### TranslateMessage

```cpp
BOOL TranslateMessage(
    _In_ CONST MSG *lpMsg  // 指向 MSG 结构体
);
```

- 将键盘虚拟键消息（`WM_KEYDOWN`）转换为字符消息（`WM_CHAR`）
- 例如：物理按键 'A' → `WM_KEYDOWN` → `TranslateMessage` → `WM_CHAR`（字符 'a' 或 'A'）

##### DispatchMessage

```cpp
LRESULT DispatchMessage(
    _In_ CONST MSG *lpMsg  // 指向 MSG 结构体
);
```

- 将消息分发给目标窗口的窗口过程函数
- 系统通过 `hwnd` → 窗口对象 → 窗口类 → `lpfnWndProc` 找到 `MyWinProc`
- **返回值**：窗口过程 `MyWinProc` 的返回值

---

### 7. 消息分发机制（DispatchMessage 如何调用 MyWinProc）

```
应用程序调用 DispatchMessage(&msg)
         │
         ▼
    进入内核模式
         │
         ▼
系统内核根据 msg.hwnd 找到目标窗口对象
         │
         ▼
从窗口对象中读取窗口类注册信息
         │
         ▼
找到 lpfnWndProc 的函数地址（即 MyWinProc）
         │
         ▼
    退出内核模式
         │
         ▼
系统调用 MyWinProc(hwnd, uMsg, wParam, lParam)
         │
         ▼
   窗口过程处理消息并返回
```

**关键数据结构关联**：

```
注册时：className → WNDCLASS → { lpfnWndProc: MyWinProc, ... }
创建时：hwnd → 窗口对象 → 引用 WNDCLASS → 指向 MyWinProc
分发时：msg.hwnd → 窗口对象 → WNDCLASS → MyWinProc
```

---

### 8. 窗口过程（MyWinProc）

```cpp
LRESULT CALLBACK MyWinProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
```

#### 8.1 窗口过程参数详解

| 参数       | 类型      | 说明                                       |
| :--------- | :-------- | :----------------------------------------- |
| `hwnd`     | `HWND`    | 接收消息的窗口句柄（用于区分多个窗口实例） |
| `uMsg`     | `UINT`    | 消息标识符（如 `WM_PAINT`、`WM_CLOSE` 等） |
| `wParam`   | `WPARAM`  | 第一个消息参数（含义取决于消息类型）       |
| `lParam`   | `LPARAM`  | 第二个消息参数（含义取决于消息类型）       |
| **返回值** | `LRESULT` | 消息处理结果（含义取决于消息类型）         |

#### 8.2 常见消息详解

| 消息             | 触发时机         | wParam             | lParam                     | 建议处理                               |
| :--------------- | :--------------- | :----------------- | :------------------------- | :------------------------------------- |
| `WM_CHAR`        | 键盘输入产生字符 | 字符代码（如 'A'） | 按键状态信息               | 处理文本输入                           |
| `WM_LBUTTONDOWN` | 鼠标左键按下     | 按键状态标志       | 鼠标坐标（X 低位，Y 高位） | 响应点击操作                           |
| `WM_PAINT`       | 窗口需要重绘     | 0（未使用）        | 0（未使用）                | 绘制客户区内容                         |
| `WM_CLOSE`       | 用户点击关闭按钮 | 0（未使用）        | 0（未使用）                | 询问是否关闭，调用 `DestroyWindow`     |
| `WM_DESTROY`     | 窗口被销毁       | 0（未使用）        | 0（未使用）                | 调用 `PostQuitMessage(0)` 退出消息循环 |
| `WM_CREATE`      | 窗口创建时       | 0（未使用）        | `CREATESTRUCT*`            | 初始化窗口数据                         |

#### 8.3 为什么需要调用 DefWindowProc？

```cpp
default:
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
```

| 函数            | 作用                                                         |
| :-------------- | :----------------------------------------------------------- |
| `DefWindowProc` | 执行 Windows 系统默认的消息处理（窗口移动、尺寸调整、最小化等） |

- 窗口过程只处理**自己关心的消息**（如 `WM_PAINT`、`WM_CHAR`）
- 所有**未处理的消息**必须交给 `DefWindowProc` 执行**默认处理**
- 如果不调用 `DefWindowProc`，窗口将无法正常工作（无法移动、无法调整大小、无法最小化等）

---

### 9. 完整代码

```cpp
#include <Windows.h>
#include <stdio.h>

LPCWSTR className = L"MyClassName";
LPCWSTR windowName = L"MyWindowName";

LRESULT CALLBACK MyWinProc(
    HWND hwnd, // handle to window
    UINT uMsg,  // message identifier
    WPARAM wParam, // first message param
    LPARAM lParam // second message param
) {
    int ret = 0;
    HDC hdc;
    switch (uMsg) {
    case WM_CHAR:
        wchar_t szChar[20];
        swprintf_s(szChar, 20, L"按下字符: %c", (wchar_t)wParam);
        MessageBox(hwnd, (LPCWSTR)szChar, L"窗口标题", NULL);
        break;
    case WM_LBUTTONDOWN:
        MessageBox(hwnd, L"按下鼠标左键", L"窗口标题", NULL);
        break;
    case WM_PAINT:
        PAINTSTRUCT ps;
        hdc = BeginPaint(hwnd, &ps);
        TextOut(hdc, 0, 0, L"输出内容", lstrlenW(L"输出内容"));
        EndPaint(hwnd, &ps);
        MessageBox(hwnd, L"触发绘制事件", L"窗口标题", NULL);
        break;
    case WM_CLOSE:
        ret = MessageBox(hwnd, L"是否关闭窗口", L"窗口标题", MB_YESNO);
        if (ret == IDYES) {
            DestroyWindow(hwnd);
        }
        break;
    case WM_DESTROY:
        PostQuitMessage(0);
        break;
    default:
        return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }
    return ret;
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PSTR lpCmdLine, int nCmdShow) {
    // 设计一个窗口类
    WNDCLASS wndcls;
    wndcls.cbClsExtra = NULL;
    wndcls.cbWndExtra = NULL;
    wndcls.hbrBackground = (HBRUSH)GetStockObject(WHITE_BRUSH);
    wndcls.hCursor = LoadCursor(NULL, IDC_ARROW);
    wndcls.hIcon = LoadIcon(NULL, IDI_APPLICATION);
    wndcls.hInstance = hInstance;

    wndcls.lpfnWndProc = MyWinProc;

    wndcls.lpszClassName = className;
    wndcls.lpszMenuName = NULL;
    wndcls.style = CS_HREDRAW | CS_VREDRAW;

    // 注册窗口类
    RegisterClass(&wndcls);

    // 创建窗口
    HWND hwnd;
    hwnd = CreateWindow(className, windowName, WS_OVERLAPPEDWINDOW, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, NULL, NULL, hInstance, NULL);

    // 显示以及更新窗口
    ShowWindow(hwnd, SW_SHOWNORMAL);
    UpdateWindow(hwnd);

    // 消息循环  GetMessage只有在接收到WM_QUIT才会返回0
    // TranslateMessage 翻译消息 WM_KEYDOWN和WM_KEYUP 合并为WM_CHAR
    MSG msg;
    while (GetMessage(&msg, NULL, NULL, NULL)) {
        TranslateMessage(&msg); // 将机器码翻译为字符 'a' 或 'A'
        DispatchMessage(&msg); // 分发处理消息
    }
    return msg.wParam;
}
```

---

### 10. 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Windows 窗口程序运行全流程                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WinMain 入口                                                               │
│      │                                                                      │
│      ▼                                                                      │
│  1. 设计窗口类（WNDCLASS）                                                  │
│     · 填写 lpfnWndProc = MyWinProc（窗口过程函数地址）                      │
│     · 填写 hInstance、背景画刷、光标、图标等                                │
│      │                                                                      │
│      ▼                                                                      │
│  2. 注册窗口类（RegisterClass）                                             │
│     → 系统内核保存类信息（类名 ↔ WNDCLASS）                                 │
│      │                                                                      │
│      ▼                                                                      │
│  3. 创建窗口（CreateWindow）                                                │
│     → 根据类名查找注册信息，分配窗口内核对象，返回 HWND                     │
│     → 系统发送 WM_CREATE（可在此做初始化）                                  │
│      │                                                                      │
│      ▼                                                                      │
│  4. 显示窗口（ShowWindow / UpdateWindow）                                   │
│     → 屏幕绘制 → 系统发送 WM_PAINT                                          │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                     5. 消息循环（while 循环）                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │  GetMessage（从系统消息队列取消息）                            │  │  │
│  │  │    │ 收到 WM_QUIT → 退出循环                                   │  │  │
│  │  │    ▼                                                          │  │  │
│  │  │  TranslateMessage（键盘扫描码 → WM_CHAR）                     │  │  │
│  │  │    ▼                                                          │  │  │
│  │  │  DispatchMessage（系统根据 hwnd 查找窗口类，调用 MyWinProc）  │  │  │
│  │  │                      │                                        │  │  │
│  │  │                      ▼                                        │  │  │
│  │  │   ┌──────────────────────────────────────────────────────────┐ │  │  │
│  │  │   │            6. 窗口过程 MyWinProc                        │ │  │  │
│  │  │   │  switch (uMsg) {                                        │ │  │  │
│  │  │   │    case WM_CHAR:    处理字符输入                        │ │  │  │
│  │  │   │    case WM_LBUTTONDOWN: 处理鼠标点击                    │ │  │  │
│  │  │   │    case WM_PAINT:   绘制窗口内容                        │ │  │  │
│  │  │   │    case WM_CLOSE:   询问是否关闭                        │ │  │  │
│  │  │   │    case WM_DESTROY: PostQuitMessage(0)  → 退出循环     │ │  │  │
│  │  │   │    default: DefWindowProc（系统默认处理）               │ │  │  │
│  │  │   │  }                                                     │ │  │  │
│  │  │   └──────────────────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  程序退出（返回 msg.wParam）                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 11. 关键点总结

| 要点                              | 说明                                                         |
| :-------------------------------- | :----------------------------------------------------------- |
| **为什么要写窗口类**              | 窗口类相当于"模板"或"蓝图"，定义了窗口的外观和行为。采用面向对象设计，支持多个窗口共享同一类属性。 |
| **为什么要注册**                  | 将窗口类信息存入系统内核，供 `CreateWindow` 查询使用。内核需要保存 `lpfnWndProc` 地址以便后续消息分发。 |
| **创建窗口的本质**                | 在系统内核中分配窗口对象内存，返回句柄 `HWND`。句柄是用户态访问内核对象的唯一标识。 |
| **消息分发机制**                  | `DispatchMessage` 通过 `hwnd` 找到对应的窗口对象，再从窗口对象关联的窗口类中取出 `lpfnWndProc`，最终回调 `MyWinProc`。 |
| **消息循环的作用**                | 持续从系统消息队列取消息并分发。没有消息循环，窗口无法响应用户操作。 |
| **DefWindowProc 的作用**          | 处理所有不感兴趣的消息（如窗口移动、尺寸变化），确保窗口具备标准 Windows 窗口的基本行为。 |
| **WM_DESTROY 与 PostQuitMessage** | 窗口销毁时发送 `WM_DESTROY`，在其中调用 `PostQuitMessage(0)` 向消息队列投递 `WM_QUIT`，让 `GetMessage` 返回 `FALSE` 退出消息循环。 |

