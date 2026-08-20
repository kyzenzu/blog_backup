---
title: 【Linux编程】Linux进程控制函数
date: 2026-08-15 12:43:14
tags:
  - C/C++
  - Linux编程
---

## Linux进程启动函数笔记

### 一、进程启动的两种主要方式

在Linux中，启动一个新程序（进程）通常通过两类函数组合使用：

1. **`fork()` 系列**：创建当前进程的一个副本（子进程）。
2. **`exec()` 系列**：在当前进程中加载并执行一个全新的程序，替换当前进程的代码、数据、堆栈等。

> **核心区别**：`fork` 是“复制自己”，`exec` 是“替换自己”。通常两者结合使用——`fork` 创建子进程，然后在子进程中调用 `exec` 来运行新程序。

#### `fork` 和 `exec` 函数族所需头文件

##### `fork` 函数

| 头文件                   | 必要性       | 说明                                                         |
| ------------------------ | ------------ | ------------------------------------------------------------ |
| `#include <unistd.h>`    | **必须**     | 提供 `fork` 函数声明。POSIX 标准明确规定 `fork` 的原型位于此头文件中。 |
| `#include <sys/types.h>` | **强烈建议** | 提供 `pid_t` 类型定义。虽然 `<unistd.h>` 在多数系统中已间接包含，但显式引入可提升代码可移植性，符合 POSIX 规范。 |

---

##### `exec` 函数族

| 头文件                | 必要性   | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| `#include <unistd.h>` | **必须** | 提供所有 `exec` 族函数的声明（`execl`、`execlp`、`execle`、`execv`、`execvp`、`execvpe`）。POSIX 标准规定这些函数原型位于此头文件中。 |

> **注意**：`exec` 族函数不需要 `<sys/types.h>`，因为它们不直接使用 `pid_t` 等类型。但如果代码中同时使用 `fork` 和 `exec`，则仍需包含 `<sys/types.h>` 以确保 `pid_t` 可见。

---

### 二、`exec` 函数族详解（共8个）

| 函数     | 原型                                                         | 特点                                                         |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `execl`  | `int execl(const char *path, const char *arg, ...);`         | ⭐参数以列表形式逐个给出（可变参数），以 `NULL` 作为最后一个参数 |
| `execlp` | `int execlp(const char *file, const char *arg, ...);`        | 参数传递同 `execl`，但使用 `PATH` 环境变量搜索可执行文件     |
| `execle` | `int execle(const char *path, const char *arg, ..., char *const envp[]);` | 参数传递同 `execl`，但可自定义环境变量数组 `envp`            |

| 函数          | 原型                                                         | 特点                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `execv`       | `int execv(const char *path, char *const argv[]);`           | ⭐参数以指针数组 `argv` 传递，同样要以 `NULL` 作为数组的最后一个元素 |
| `execvp`      | `int execvp(const char *file, char *const argv[]);`          | 参数传递同 `execv`，但使用 `PATH` 搜索可执行文件             |
| **`execve`**  | `int execve(const char *path, char *const argv[], char *const envp[]);` | 参数传递同 `execv`，但可自定义环境变量数组 `envp`<br />**系统调用**，所有 `exec` 函数的底层实现 |
| `execvpe`     | `int execvpe(const char *file, char *const argv[], char *const envp[]);` | 参数传递同 `execv`，不仅使用 `PATH` 搜索可执行文件，而且可自定义环境变量 |
| **`fexecve`** | `int fexecve(int fd, char *const argv[], char *const envp[]);` | 通过**文件描述符**（而非路径）来执行程序，适用于已打开或已验证的文件 |

---

### 三、`exec` 函数族命名规律（记忆口诀）

函数名的构成可以拆解为三部分，方便记忆：

| 后缀/中缀                 | 含义                                               |
| ------------------------- | -------------------------------------------------- |
| **`l`** (list)            | 参数以**列表**形式逐个传递（可变参数）             |
| **`v`** (vector)          | 参数以**数组（向量）**形式传递                     |
| **`p`** (path)            | 使用 `PATH` 环境变量搜索文件路径                   |
| **`e`** (environment)     | 可以**自定义环境变量**（通过 `envp` 参数）         |
| **`f`** (file descriptor) | 通过**文件描述符**指定要执行的程序（仅 `fexecve`） |

> 组合规则：`exec` + `[l/v]` + 可选 `[p]` + 可选 `[e]`，另有独立的 `execve` 和 `fexecve`

- **带 `p`**：只需传文件名，系统去 `PATH` 找（如 `ls` → `/bin/ls`）
- **不带 `p`**：必须传完整路径（如 `/bin/ls`）
- **带 `e`**：由你传入完整的环境变量数组，不继承父进程环境
- **不带 `e`**：子进程继承父进程的环境变量
- **`execve`**：标准系统调用，不带 `p`，必须指定完整路径，且必须带 `envp`
- **`fexecve`**：通过文件描述符执行，可用于在权限检查或文件锁定后安全执行

---

### 四、八个 `exec` 函数的具体区别对比

| 函数          | 路径搜索            | 参数形式 | 环境变量           | 备注                             |
| ------------- | ------------------- | -------- | ------------------ | -------------------------------- |
| `execl`       | 需完整路径          | 列表     | 继承父进程         | 库函数                           |
| `execlp`      | 使用 `PATH`         | 列表     | 继承父进程         | 库函数                           |
| `execle`      | 需完整路径          | 列表     | 用户自定义         | 库函数                           |
| `execv`       | 需完整路径          | 数组     | 继承父进程         | 库函数                           |
| `execvp`      | 使用 `PATH`         | 数组     | 继承父进程         | 库函数                           |
| `execvpe`     | 使用 `PATH`         | 数组     | 用户自定义         | 库函数（非标准）                 |
| **`execve`**  | 需完整路径          | 数组     | **必须**用户自定义 | **系统调用**，所有 `exec` 的底层 |
| **`fexecve`** | 无需路径（通过 fd） | 数组     | **必须**用户自定义 | 系统调用，通过文件描述符执行     |

---

#### `exec` 函数使用示例与注意事项

##### 示例代码

```c
#include <unistd.h>
#include <stdio.h>

int main(int argc, char* argv[])
{
    printf("%s\n", argv[0]);      // 打印当前程序自身的 argv[0]
    printf("执行前\n");
    execl("/usr/bin/ls", "xx", "-l", NULL);
    printf("执行后\n");           // 这行不会被执行
    return 0;
}
```

---

##### 执行结果

```bash
root@aliyun:~# ./projects/LinuxConsole/bin/x64/Debug/LinuxConsole.out -c
./projects/LinuxConsole/bin/x64/Debug/LinuxConsole.out
执行前
total 16
drwxr-xr-x 2 root root 4096 Aug  8 21:50 code
drwxr-xr-x 3 root root 4096 Aug  9 18:20 projects
drwx------ 3 root root 4096 Jul 23 14:00 snap
drwxr-xr-x 2 root root 4096 Aug  8 21:27 workspace
```

---

##### 注意事项与解析

##### 1. `exec` 执行后，后续代码不再执行

从执行结果可以看到，`"执行后"` 这条信息**没有输出**。这说明 `execl()` 成功后，当前进程的代码段、数据段、堆栈等被新程序（`ls`）完全替换，因此 `main` 函数中 `execl()` 之后的代码永远不会被执行。

> 只有当 `execl()` **执行失败**时（如路径错误、权限不足），才会返回 `-1`，此时才会继续执行后面的代码。

---

##### 2. `execl` 的参数含义：为什么第二个参数是 `"xx"`？

`execl("/usr/bin/ls", "xx", "-l", NULL);` 的第二个参数 `"xx"` 看起来有些奇怪，实际上它与程序启动时的 `argv[0]` 有关。

**回忆**：我们自己的程序 `LinuxConsole.out` 启动时：

- `argv[0]` = `"./projects/LinuxConsole/bin/x64/Debug/LinuxConsole.out"`（命令行输入的第一个字符串）
- `argv[1]` = `"-c"`（命令行输入的第二个字符串，即选项）

**对于新程序 `ls` 也是一样**：当 `exec` 加载 `ls` 时，它会把自己的参数列表传递给新程序的 `main` 函数，规则完全相同：

| 参数位置            | 对应新程序的 `argv[]` | 说明                                                 |
| ------------------- | --------------------- | ---------------------------------------------------- |
| 第1个参数（`path`） | —                     | 可执行文件的**路径**，不属于 `argv`                  |
| 第2个参数           | `argv[0]`             | 新程序“自己”的名字（通常为程序名，可以是任意字符串） |
| 第3个参数           | `argv[1]`             | 新程序的第一个选项/参数                              |
| 第4个参数           | `argv[2]`             | 新程序的第二个选项/参数                              |
| ...                 | ...                   | ...                                                  |
| 最后一个 `NULL`     | —                     | 参数列表结束标记                                     |

**因此**：对于 `execl("/usr/bin/ls", "xx", "-l", NULL)`：

- `"/usr/bin/ls"` → 告诉系统要执行哪个程序
- `"xx"` → 作为 `ls` 的 `argv[0]`（`ls` 自身可以忽略它，或者用它来显示程序名）
- `"-l"` → 作为 `ls` 的 `argv[1]`，被 `ls` 解析为“以长格式列出文件”

之所以第二个参数写 `"xx"` 可以正常工作，是因为 `ls` 程序**不依赖 `argv[0]` 的内容来做决定**，它的行为完全由 `argv[1]` 及之后的参数决定。

而之所以要把`"-l"`移到第三个参数，是因为 `ls` 程序执行时是从 `argv[1] `获取 `-l` 选项发挥选项的作用。(代码就这么写的，没办法)

如果想让 `ps` 或 `top` 等工具看到正确的程序名，通常会传入真实的程序名（如 `"ls"`）。

---

#### 3. 核心要点总结

| 要点                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| **`exec` 替换进程**  | 执行成功后，原程序后续代码不再执行，进程被新程序完全覆盖     |
| **参数传递规则**     | `exec` 的参数列表直接对应新程序的 `argv[]`，从第2个参数开始对应 `argv[0]`、`argv[1]`…… |
| **`argv[0]` 可随意** | 新程序的 `argv[0]` 通常传入程序名，但很多程序并不依赖它做逻辑判断，所以可以传任意字符串 |
| **以 `NULL` 结尾**   | 可变参数列表必须以 `NULL` 结尾，表示参数传递结束             |

---

### 五、`fork` 函数（复制当前进程）

`fork` 用于创建一个子进程，子进程是父进程的副本（复制代码、数据、堆栈等）。

**函数原型**：

```c
pid_t fork(void);
```

**返回值**（三种情况）：

| 返回值     | 含义                                     |
| :--------- | :--------------------------------------- |
| **大于 0** | 当前处于**父进程**，返回值为子进程的 PID |
| **等于 0** | 当前处于**子进程**                       |
| **小于 0** | 调用失败，未创建子进程                   |

> **补充说明**：系统允许的进程数量是有限的（受系统资源限制），典型范围如 **1~32768**、**1~32767** 或 **1~65535**（取决于系统配置）。

---

### 六、`exec` 与 `fork` 的核心区别（重点）

| 对比维度         | `fork()`                           | `exec()`                                 |
| ---------------- | ---------------------------------- | ---------------------------------------- |
| **功能**         | 复制当前进程，创建子进程           | 在当前进程中加载并运行一个新程序         |
| **进程是否变化** | 新进程是**新创建的**（PID 不同）   | **不创建新进程**，PID 保持不变           |
| **进程映像**     | 子进程复制父进程的代码、数据、堆栈 | 完全替换为新的程序映像，旧内容被丢弃     |
| **返回值**       | 父进程返回子进程 PID，子进程返回 0 | 成功不返回，失败返回 -1                  |
| **典型用途**     | 创建一个执行环境（子进程）         | 在子进程中执行新程序（配合 `fork` 使用） |

**典型组合用法**：

```c
pid_t pid = fork();
if (pid == 0) {
    // 子进程
    execlp("ls", "ls", "-l", NULL);
    exit(1); // 如果 exec 失败才执行
} else if (pid > 0) {
    // 父进程等待子进程
    wait(NULL);
}
```

> 如果不调用 `fork` 直接 `exec`，当前进程就会被新程序“覆盖”，一般用于“单进程转换”场景（如 `init` 启动 `getty`，或 shell 执行命令前的自我替换）。

---

### 七、补充：`system()` 和 `popen()` 的简要说明

| 函数       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| `system()` | 内部调用 `fork` + `exec` + `wait`，直接执行一个 shell 命令，等待命令结束返回。简单但安全性较低。 |
| `popen()`  | 也是 `fork` + `exec`，但会建立管道，用于与子进程的输入/输出流进行通信。 |

---

### 八、一句话总结

- **`fork`** = 复制，产生新进程（克隆体）；
- **`exec`** = 替换，当前进程改头换面（新程序）；
- **`exec` 的8个变体** = 不同参数组织方式 + 是否搜索 PATH + 是否自定义环境变量 + 是否通过文件描述符执行；
- 底层都是 **`execve`** 系统调用，`fexecve` 是通过 fd 执行的特殊版本；
- 实际开发中，最常用的是 **`fork` + `execvp`（或 `execlp`）** 组合，简洁且灵活。

---

## Linux进程终止函数笔记

总结：

- `atexit()`/`on_exit()`用于注册程序退出时的钩子函数
- `exit()`程序正常退出用的函数，它会在调用`atexit()`和`on_exit()`注册的钩子函数后，调用`_exit()`进入内核
- `_exit()`系统调用，直接进入内核退出进程
- `abort()`程序异常终止用的函数，它会输出一行字
- `assert()`程序断言终止用的函数，它会输出自己的一行内容后再调用`abort`函数。

### 进程终止的两种方式

| 方式         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| **正常退出** | 程序主动调用退出函数，或 `main` 函数返回，进程有序清理后退出 |
| **异常终止** | 程序遇到致命错误、收到信号或主动调用 `abort`，进程立即终止，不保证资源清理 |

---

### 一、进程正常退出前的钩子函数（注册清理回调）

| 函数      | 原型                                                    | 说明                                                         |
| --------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| `atexit`  | `int atexit(void (*func)(void));`                       | 注册一个**无参**函数，在 `exit` 正常退出时按 **注册顺序的逆序** 调用。可注册多个 |
| `on_exit` | `int on_exit(void (*function)(int, void*), void *arg);` | 注册一个**带参**函数，在 `exit` 正常退出时调用。第一个参数为 `exit` 的 `status`，第二个参数为注册时传入的 `arg`（非标准，常见于 GNU/Linux） |

**两者的区别**：

| 对比维度         | `atexit`                                   | `on_exit`                                                    |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
| **标准性**       | C 标准库 / POSIX 标准                      | **非标准**（GNU 扩展），移植性较差                           |
| **回调函数参数** | `void (*func)(void)`，无参数               | `void (*function)(int, void*)`，可接收退出状态码和自定义参数 |
| **调用时机**     | 仅 `exit` 正常退出时调用                   | 仅 `exit` 正常退出时调用（与 `atexit` 相同）                 |
| **注册数量**     | 可注册多个，调用顺序为逆序（后注册先调用） | 可注册多个，调用顺序为逆序                                   |

> **注意**：`_exit`、`_Exit` 和 `abort` **不会**调用这些钩子函数。

---

### 正常退出进程的函数（3个）

| 函数    | 原型                      | 说明                                                         |
| ------- | ------------------------- | ------------------------------------------------------------ |
| `exit`  | `void exit(int status);`  | **标准库函数**，执行完整的退出流程：调用 `atexit`/`on_exit` 注册的清理函数 → 刷新 `stdio` 缓冲区 → 关闭所有标准 I/O 流 → 调用 `_exit` 进入内核 |
| `_exit` | `void _exit(int status);` | **系统调用**，直接进入内核终止进程，**不执行** `atexit` 钩子，**不刷新** `stdio` 缓冲区（但内核会关闭所有文件描述符） |
| `_Exit` | `void _Exit(int status);` | C99 标准提供，行为与 `_exit` **完全等价**，用于跨平台兼容（POSIX 和非 POSIX 系统） |

> **三者的关系**：`exit` → 库层清理 → 调用 `_exit` → 内核终止。`_exit` 和 `_Exit` 是直接进入内核，跳过库层清理。

---

### 二异常终止进程的函数

| 函数     | 原型                           | 说明                                                         |
| -------- | ------------------------------ | ------------------------------------------------------------ |
| `abort`  | `void abort(void);`            | 向当前进程发送 `SIGABRT` 信号，默认行为是终止进程并产生 core dump。**不调用** `atexit`/`on_exit` 注册的函数，但会刷新 `stdio` 缓冲区（取决于实现） |
| `assert` | `void assert(int expression);` | 若 `expression` 为 **假（0）**，则向 `stderr` 输出错误信息（包含文件名、行号、函数名），然后调用 `abort()` 终止程序。仅在 **未定义 `NDEBUG`** 时生效；定义 `NDEBUG` 后，`assert` 被预处理器展开为空操作 |

---

### 三`abort` 与 `assert` 的区别（重点）

| 对比维度       | `abort()`                                    | `assert()`                                                 |
| -------------- | -------------------------------------------- | ---------------------------------------------------------- |
| **本质**       | 函数（库函数）                               | 宏（预处理阶段展开）                                       |
| **触发条件**   | 主动调用，无条件终止                         | 仅当传入的表达式为 **假** 时才触发                         |
| **是否可禁用** | 不可禁用，调用即终止                         | 可通过 `#define NDEBUG` 在编译时禁用（宏展开为空）         |
| **输出信息**   | 默认无额外信息（但可由信号处理器捕获）       | 自动输出文件名、行号、函数名、失败的表达式                 |
| **适用场景**   | 程序遇到不可恢复的致命错误（如内存分配失败） | **调试期间**检查程序逻辑假设（如指针非空、数组索引越界等） |
| **核心哲学**   | 明确告知程序要终止                           | “程序员的断言”——条件不成立意味着代码逻辑有 bug             |

> **一句话总结**：`assert` 是用于**调试**的“契约检查”，条件失败时调用 `abort`；`abort` 是用于**生产环境**的“紧急制动”，无条件终止程序。

#### 输出对比

##### `assert(0)` 输出

```bash
LinuxConsole.out: /root/projects/LinuxConsole/main.cpp:29: int main(int, char**): Assertion `0' failed.
Aborted (core dumped)
```

##### `abort()` 输出

```bash
Aborted (core dumped)
```

可见 `assert()` 会调用 `abort()`，并且多一行自己的输出。

---

### 四、进程终止函数所需头文件

| 函数                          | 头文件                | 必要性   | 说明                                                         |
| ----------------------------- | --------------------- | -------- | ------------------------------------------------------------ |
| `atexit` / `on_exit` / `exit` | `#include <stdlib.h>` | **必须** | C 标准库规定 `exit` 和 `atexit` 的函数声明位于 `<stdlib.h>` 中；`on_exit` 是 glibc 扩展，其声明同样位于 `<stdlib.h>` 中。 |
| `_exit`                       | `#include <unistd.h>` | **必须** | POSIX 标准规定 `_exit` 的函数声明位于 `<unistd.h>` 中。      |
| `abort`                       | `#include <stdlib.h>` | **必须** | C 标准库规定 `abort` 的函数声明位于 `<stdlib.h>` 中。        |
| `assert`                      | `#include <assert.h>` | **必须** | C 标准库规定 `assert` 宏的定义位于 `<assert.h>` 中。         |

---

### 五、所有函数 / 宏的对比总览

| 函数/宏            | 终止方式             | 调用 `atexit`/`on_exit` | 刷新 `stdio` 缓冲区    | 产生 core dump       | 是否可禁用             |
| ------------------ | -------------------- | ----------------------- | ---------------------- | -------------------- | ---------------------- |
| `exit`             | 正常                 | ✅ 是                    | ✅ 是                   | ❌ 否                 | 不可禁用               |
| `_exit` / `_Exit`  | 正常（直接进内核）   | ❌ 否                    | ❌ 否（内核关闭 fd）    | ❌ 否                 | 不可禁用               |
| `abort`            | 异常（发送 SIGABRT） | ❌ 否                    | 取决于实现（通常刷新） | ✅ 是（可配置）       | 不可禁用               |
| `assert`（条件假） | 异常（调用 `abort`） | ❌ 否                    | 取决于实现             | ✅ 是（经由 `abort`） | ✅ 可定义 `NDEBUG` 禁用 |

---

### 六、典型使用场景与建议

| 场景                                           | 推荐函数                                                    |
| ---------------------------------------------- | ----------------------------------------------------------- |
| 普通程序正常退出                               | `exit(0)` 或 `return 0`（`main` 中 `return` 等效于 `exit`） |
| 子进程（`fork` 后 `exec` 失败）                | `_exit(status)`，避免调用 `atexit` 钩子干扰父进程           |
| 快速终止，不做任何清理                         | `_exit(status)` 或 `_Exit(status)`                          |
| 检测到不可恢复的致命错误                       | `abort()`，配合 core dump 事后分析                          |
| 调试期间检查逻辑假设                           | `assert(expression)`                                        |
| 程序退出前执行资源清理（如关闭文件、释放锁）   | `atexit(func)` 注册清理函数                                 |
| 退出前需要向清理函数传递退出状态码或自定义数据 | `on_exit(func, arg)`（注意非标准）                          |

---

### 七、一句话总结

- **`atexit` / `on_exit`** = 注册 `exit` 退出前的回调函数，`on_exit` 更灵活但非标准。
- **`exit`** = 正常退出，执行注册的钩子 + 刷新缓冲区；
- **`_exit` / `_Exit`** = 立即退出，跳过钩子和缓冲区刷新，直接进内核；
- **`abort`** = 异常终止，发送 `SIGABRT` 信号，通常产生 core dump；
- **`assert`** = 调试期断言，表达式为假时调用 `abort`，生产环境可禁用；

---
### 一、`setjmp` 和 `longjmp` 函数详解

#### 1. 函数原型
```c
#include <setjmp.h>

int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int val);
```

---

#### 2. 功能说明
- **`setjmp(env)`**：保存当前程序的**堆栈环境**（包括 CPU 寄存器、栈指针、程序计数器等）到 `env` 缓冲区中，用于后续的跨函数跳转。
	⭐`setjmp` 会将执行 `call setjpm` 指令前所有寄存器（包括栈寄存器、程序计数器）的状态都保存起来。然后返回一个值放在 `eax` 里。
- **`longjmp(env, val)`**：恢复 `env` 中保存的堆栈环境，从 `setjmp` 的调用点**继续执行**，实现非本地跳转（non-local jump），类似于"跨越多个函数调用栈的 goto"。
	⭐`longjmp`  会恢复 `setjmp` 保存的所有寄存器，只修改 `eax` 的值，然后跳转到 `call setjmp` 的下一条指令。

---

#### 3. 返回值
- **`setjmp`**：
  - **首次直接调用**时返回 `0`。
  - **从 `longjmp` 跳转回来**时返回 `val`（若 `val` 为 `0`，则会加一返回 `1`）。
- **`longjmp`**：无返回值（一旦执行，控制流直接跳转到 `setjmp` 的返回点，不会返回到调用者）。

---

#### 4. 所需头文件

| 头文件                | 必要性   | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| `#include <setjmp.h>` | **必须** | 提供 `setjmp`、`longjmp`、`sigsetjmp`、`siglongjmp` 的函数声明，以及 `jmp_buf` 和 `sigjmp_buf` 类型定义。C 标准库和 POSIX 标准均规定此头文件为非本地跳转相关函数声明的所在地。 |

---

#### 5. `jmp_buf` 类型详解

```c
typedef struct __jmp_buf_tag jmp_buf[1];
```

- `jmp_buf` 是一个长度为 `1` 的数组，本质上是一个**单元素数组类型**。
- **妙用**：传递 `jmp_buf` 变量给函数时，直接传变量名即可（数组自动退化为指针），无需显式取地址 `&env`。
- 用于保存：通用寄存器（`eax`、`ebx`、`ecx`、`edx`、`esi`、`edi`）、栈指针（`esp`）、帧指针（`ebp`）、指令指针（`eip`）等。

---

#### 6. 为什么 `setjmp/longjmp` 必须用汇编实现？

`setjmp` 和 `longjmp` 的函数定义均由汇编实现

1. **需要直接操作 CPU 寄存器**：C 语言无法直接读写寄存器（如 `ebp`、`esp`、`eip`），而 `setjmp` 需要保存所有寄存器的当前状态。
2. **需要精确控制栈帧布局**：C 编译器会为函数插入额外的序言（prologue）和结尾（epilogue）代码，这会破坏栈帧结构。`longjmp` 恢复现场时若栈帧被编译器修改，会导致程序崩溃。

---

#### 7. `setjmp` 原理（i386 汇编实现分析）

`setjmp` 将执行 `call setjmp` 指令**前**的所有寄存器状态保存到 `jmp_buf` 中：

```assembly
** jmp_buf 中寄存器的偏移量:
**  eax ebx ecx edx esi edi ebp esp eip
**   0   4   8   12  16  20  24  28  32
*/
.globl setjmp
setjmp:
    push    ebp
    mov     ebp, esp
    push    edi                 ; 保存函数调用前的 edi
    mov     edi, [ebp+8]        ; edi = jmp_buf 结构的地址

    ; 保存通用寄存器
    mov     [edi+0], eax
    mov     [edi+4], ebx
    mov     [edi+8], ecx
    mov     [edi+12], edx
    mov     [edi+16], esi

    ; 保存 EDI（从栈中恢复原值）
    mov     eax, [ebp-4]
    mov     [edi+20], eax

    ; 保存调用者的 EBP（帧指针）
    mov     eax, [ebp+0]
    mov     [edi+24], eax

    ; 保存 ESP（栈指针），调整到调用 setjmp 之前的状态
    mov     eax, esp
    add     eax, 12             ; 跳过 setjmp 自身的栈帧
    mov     [edi+28], eax

    ; 保存 EIP（指令指针）—— 关键！
    mov     eax, [ebp+4]        ; [ebp+4] 就是 call setjmp 的下一条指令地址
    mov     [edi+32], eax       ; 保存到 jmp_buf 的 EIP 位置

    pop     edi
    mov     eax, 0              ; setjmp 首次调用返回 0
    leave
    ret
```

---

#### 8. `longjmp` 原理（i386 汇编实现分析）

`longjmp` 是 `setjmp` 的逆过程：**从 `jmp_buf` 中恢复所有被保存的寄存器，然后跳转回 `setjmp` 被调用时的返回地址**，让程序"以为"自己刚刚从 `setjmp` 返回。

```assembly
** jmp_buf 中寄存器的偏移量:
**  eax ebx ecx edx esi edi ebp esp eip
**   0   4   8   12  16  20  24  28  32
*/
.globl longjmp
longjmp:
    push    ebp
    mov     ebp, esp

    mov     edx, [ebp+8]        ; edx = jmp_buf 地址
    mov     eax, [ebp+12]       ; eax = 返回值 (val) ⭐
    test    eax, eax            
    jnz     .L_val_ok
    inc     eax                 ; val=0 时改为 1（保证 setjmp 返回非零）
.L_val_ok:

    ; 按顺序恢复寄存器（EDX 最后恢复，因为它还保存着 jmp_buf 地址）
    mov     ecx, [edx+8]        ; 恢复 ECX
    mov     ebx, [edx+4]        ; 恢复 EBX
    mov     esi, [edx+16]       ; 恢复 ESI
    mov     edi, [edx+20]       ; 恢复 EDI
    mov     ebp, [edx+24]       ; 恢复 EBP
    mov     esp, [edx+28]       ; 恢复栈指针

    push    dword ptr [edx+32]  ; 将 EIP 压栈，为 ret 做准备
    mov     edx, [edx+12]       ; 最后恢复 EDX（此时已不需要 jmp_buf 地址）

    ret                         ; 跳转到压入的返回地址
```

**关键点**：
- `longjmp` 恢复所有寄存器，**只修改 `eax` 的值**（设置为 `val`）。
- 通过 `ret` 指令跳转到 `call setjmp` 的下一条指令，实现"穿越式"返回。

---

### 二、 `sigsetjmp` 和 `siglongjmp` —— 信号安全增强版

#### 1. 函数原型
```c
#include <setjmp.h>

int sigsetjmp(sigjmp_buf env, int savesigs);
void siglongjmp(sigjmp_buf env, int val);
```

`sig` 前缀是 **signal（信号）** 的缩写。这两个函数是 `setjmp`/`longjmp` 的**信号安全增强版**，专门解决跨信号处理函数跳转时**信号掩码（signal mask）**的恢复问题。

#####  `savesigs` 参数

| 参数值   | 行为                                              |
| -------- | ------------------------------------------------- |
| **非零** | 保存当前信号掩码到 `env`，`siglongjmp` 时自动恢复 |
| **零**   | 不保存信号掩码（等价于普通 `setjmp`）             |

#### 2. 普通 `setjmp` 的缺陷

**问题背景**：

当一个信号处理函数执行时，内核会**自动阻塞当前正在处理的信号**（防止递归中断造成混乱）。正常流程中，信号处理函数返回后，内核自动解除阻塞。但如果信号处理函数内部调用 `longjmp` 跳转出去：

```
1. 程序正常执行
2. 信号到达 → 内核阻塞该信号，跳转到信号处理函数
3. 信号处理函数执行 longjmp
4. 跳转到 setjmp 的调用点继续执行
5. ❌ 信号仍被阻塞！程序永远不会再收到这个信号
```

**根本原因**：普通 `setjmp` 只保存和恢复 CPU 寄存器，**不保存信号掩码**。跳转时跳过了信号处理函数的返回，内核无法执行"解除阻塞"的步骤，信号掩码位永远为 `1`。⭐信号掩码的作用就是屏蔽信号，所以跳转后不恢复信号掩码，信号会一直处于阻塞状态。

#### 3.什么是信号处于阻塞状态

##### 3.1 信号的生命周期（前置知识）

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  信号产生   │ -> │  信号挂起   │ -> │  信号递送   │
│ (generated) │    │ (pending)   │    │ (delivered) │
└─────────────┘    └─────────────┘    └─────────────┘
     ↓                    ↓                    ↓
  键盘 Ctrl+C        等待处理中           执行处理函数
  kill() 调用       被阻塞则停留         或默认动作
  硬件异常          在pending队列
```

**阻塞（blocked）** 发生在"信号挂起"阶段：内核有信号要交给进程，但进程暂时不想接收，信号就卡在 pending 队列里等待，**不会触发处理函数**。

##### 3.2为什么信号处理函数执行时，当前信号会被阻塞？

这是内核的**默认安全机制**，防止递归中断造成混乱：

```c
// 伪代码：内核在调用信号处理函数时的行为
void kernel_deliver_signal(int sig, void (*handler)(int)) {
    sigprocmask(SIG_BLOCK, &sig, NULL);  // 1. 阻塞当前信号
    handler(sig);                         // 2. 执行用户处理函数
    sigprocmask(SIG_UNBLOCK, &sig, NULL); // 3. 解除阻塞
}
```

```c
// 假设用户注册了 SIGINT 处理函数
signal(SIGINT, my_handler);

// 正常的 SIGINT 处理函数
void my_handler(int sig) {
    /* ⭐当执行到这里时，内核已经自动阻塞了 SIGINT
       如果此时又一个 SIGINT 到达，它不会打断当前处理函数
       而是排队等待，直到处理函数返回后才递送 */
    
    /* 执行一些操作... */
    printf("处理 SIGINT...\n");
    
    // ⭐函数返回后，内核自动解除 SIGINT 的阻塞
}
```

##### 问题演示 `longjmp` 如何导致信号永久阻塞？

```c
jmp_buf env;

void handler(int sig) {
    longjmp(env, 1);  // ❌ 跳转出去，跳过了解除阻塞的步骤！
}

int main() {
    setjmp(env);
    signal(SIGINT, handler);
    while(1) pause();
}
```

**后果**：第一次按 `Ctrl+C` 后，`SIGINT` 永远被阻塞，后续按 `Ctrl+C` 无效。

**"信号处于阻塞状态"的核心**：

- 内核中有个**信号掩码（signal mask）**，每位代表一个信号
- 位为 1 → 该信号被阻塞，暂不递送
- 阻塞不是"丢弃"，信号还在 pending 队列里
- `longjmp` 跳过函数返回 → 跳过内核自动解除阻塞的步骤 → 掩码位永远为 1

#### 4. `sigsetjmp`/`siglongjmp` 的解决方案

`siglongjmp` 在跳转前**主动恢复信号掩码**：

```c
// siglongjmp 的伪代码
void siglongjmp(sigjmp_buf env, int val) {
    if (env 中保存了信号掩码) {
        sigprocmask(SIG_SETMASK, &env.mask, NULL);  // 恢复掩码，解除阻塞
    }
    longjmp(env, val);  // 然后才跳转
}
```

`sigsetjmp` 保存信号掩码到 `sigjmp_buf`（当 `savesigs != 0` 时），`siglongjmp` 在跳转前将其恢复，从而解决信号永久阻塞的问题。

#### 5. 源码实现（glibc i386 版本）

**`sigjmp_buf` 结构**：
```
[sigjmp_buf]
   [0-31]   = 普通 jmp_buf (寄存器)
   [32]     = 信号掩码保存标志 (savesigs)
   [33]     = 信号掩码
```

**`sigsetjmp` 实现**：

```assembly
.globl sigsetjmp
sigsetjmp:
    call    setjmp              ; 先调用普通 setjmp 保存寄存器
    test    eax, eax
    jnz     .L_return           ; 如果是 siglongjmp 返回，直接跳回

    ; 首次调用：检查 savesigs 参数
    mov     4(%esp), %eax       ; 获取 savesigs 参数
    test    %eax, %eax
    jz      .L_no_mask          ; savesigs == 0，不保存掩码

    ; 保存当前信号掩码到 sigjmp_buf
    lea     4(%esp), %eax
    add     $32, %eax           ; 跳到掩码存储位置
    push    %eax
    call    sigprocmask         ; 获取当前信号掩码
    add     $4, %esp

    ; 标记掩码已保存
    mov     4(%esp), %eax
    movl    $1, 32(%eax)        ; 设置保存标志 = 1

.L_no_mask:
    xor     %eax, %eax          ; 返回 0
.L_return:
    ret
```

**`siglongjmp` 实现**：

```assembly
.globl siglongjmp
siglongjmp:
    push    %ebp
    mov     %esp, %ebp

    mov     8(%ebp), %eax       ; eax = sigjmp_buf 地址

    ; 检查是否需要恢复信号掩码
    cmpl    $1, 32(%eax)        ; 检查保存标志
    jne     .L_no_restore

    ; 需要恢复掩码
    lea     33(%eax), %eax      ; eax = 掩码数据地址
    push    %eax
    push    $0
    push    $SIG_SETMASK
    call    sigprocmask         ; sigprocmask(SIG_SETMASK, &mask, NULL)
    add     $12, %esp

.L_no_restore:
    ; 恢复参数准备调用 longjmp
    mov     8(%ebp), %eax
    push    %eax
    call    longjmp             ; 跳转到普通 longjmp
    ; 永远不会返回
```

---

### 示例代码

### 11. 应用示例：利用 `setjmp`/`longjmp` 实现异常捕获机制

下面的示例代码展示了如何利用 `setjmp`/`longjmp` 在 C 语言中模拟类似 C++/Java 的异常捕获（try-catch）机制，包括对段错误（`SIGSEGV`）的捕获和处理。

```c
#include <setjmp.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

jmp_buf jmpbuf;

void func1() {
    // 业务逻辑函数1（可在此调用 longjmp）
}

void func2() {
    longjmp(jmpbuf, 2);  // 主动抛出异常，错误码为 2
}

void signalHandler(int sig) {
    if (sig == SIGSEGV) {
        longjmp(jmpbuf, SIGSEGV);  // 将段错误信号转换为异常跳转，错误码为 SIGSEGV
    }
}

int main(int argc, char* argv[]) {
    // 注册信号处理函数
    signal(SIGSEGV, signalHandler);

    int ret = setjmp(jmpbuf);

    if (ret == 0) {
        // try 块：执行可能出现异常的业务逻辑
        /* func1(); */
        /* func2(); */
        *(int*)(NULL) = 0;  // 故意触发段错误（SIGSEGV）
    } else if (ret == 1) {
        // catch: 错误码 1
        printf("ERROR: %d\n", 1);
    } else if (ret == 2) {
        // catch: 错误码 2
        printf("ERROR: %d\n", 2);
    } else if (ret == SIGSEGV) {
        // catch: 段错误异常
        printf("ERROR: SIGSEGV\n");
    }

    return 0;
}
```

---

#### 执行流程分析

| 步骤 | 事件                                            | 说明                                                        |
| ---- | ----------------------------------------------- | ----------------------------------------------------------- |
| 1    | `setjmp(jmpbuf)` 首次调用                       | 保存当前堆栈环境到 `jmpbuf`，返回 `0`，进入 `ret == 0` 分支 |
| 2    | 执行 `*(int*)(NULL) = 0`                        | 触发段错误（SIGSEGV），内核跳转到 `signalHandler`           |
| 3    | 内核自动阻塞 `SIGSEGV`                          | 防止递归中断（详见第 9 节）                                 |
| 4    | `signalHandler` 调用 `longjmp(jmpbuf, SIGSEGV)` | 跳转回 `setjmp` 的调用点                                    |
| 5    | `setjmp` 第二次返回                             | 返回值为 `SIGSEGV`（即信号编号），进入对应的 `else if` 分支 |
| 6    | 打印 `"ERROR: SIGSEGV"`                         | 异常被成功捕获并处理                                        |

---

#### 关键设计思想

1. **`ret == 0` 作为 try 块入口**：首次调用 `setjmp` 返回 `0`，进入受保护的代码区域。
2. **`longjmp` 作为 throw 语句**：通过第二个参数携带不同的“错误码”，区分异常类型。
3. **`ret` 值作为异常类型匹配**：`setjmp` 的返回值决定了进入哪个 `else if` 分支，实现类似 `catch` 的效果。
4. **信号处理函数作为异常转换层**：将硬件异常（如段错误）转换为 `longjmp` 跳转，使其能被上层捕获。

---

### 三、总结：`setjmp` vs `sigsetjmp`

| 特性                   | `setjmp`/`longjmp` | `sigsetjmp`/`siglongjmp` |
| ---------------------- | ------------------ | ------------------------ |
| 保存寄存器             | ✓                  | ✓                        |
| 保存信号掩码           | ✗                  | ✓（当 `savesigs != 0`）  |
| 信号处理函数中跳转安全 | ❌ 导致信号永久阻塞 | ✓ 恢复掩码后安全         |
| 跨平台兼容性           | C 标准库           | POSIX                    |

> **核心原则**：在信号处理函数中使用非本地跳转时，**必须使用 `sigsetjmp`/`siglongjmp`** 并设置 `savesigs` 为非零值，否则会导致信号永久阻塞。

