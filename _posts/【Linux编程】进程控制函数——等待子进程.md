---
title: 【Linux编程】进程控制函数——等待子进程
date: 2026-08-18 16:51:39
tags:
  - C/C++
  - Linux编程
---
### 等待子进程函数详解

---

#### 1. `system` —— 执行 Shell 命令（高级封装）

##### 函数原型
```c
#include <stdlib.h>

int system(const char *command);
```

##### 功能说明
创建一个子进程，在子进程中调用 `/bin/sh -c command` 来解析并执行传入的字符串命令。这是 C 标准库提供的便捷函数，内部自动调用了 `fork()`、`exec()` 和 `waitpid()`。

##### 返回值（极易混淆）
| 返回值 | 含义 |
|--------|------|
| `-1` | `fork()` 失败或 `waitpid()` 返回异常错误 |
| `127` | Shell 执行失败（如找不到命令） |
| 其他值 | 子进程的退出状态码（需用 `WIFEXITED` 等宏解析） |

##### 所需头文件

| 头文件 | 必要性 | 说明 |
|--------|--------|------|
| `#include <stdlib.h>` | **必须** | C 标准库规定 `system` 函数的声明位于 `<stdlib.h>` 中。 |

##### 致命缺陷（不推荐在正式项目中使用）
1. **安全漏洞**：启动 Shell 环境，存在被注入恶意命令的风险（Shellshock 漏洞与此有关）。
2. **性能开销**：每次调用都要启动两个进程（先启动 Shell，再启动命令）。
3. **信号干扰**：调用期间 `SIGCHLD` 信号的处理方式会被临时改变，可能干扰父进程其他部分的信号处理逻辑。

---

#### 2. `wait` —— 阻塞等待任意子进程

##### 函数原型
```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *status);
```

##### 功能说明
**阻塞**当前进程，直到它的**任意一个**子进程终止。这是最基础的子进程回收函数。

##### 返回值
- **成功**：返回被回收的子进程 PID。
- **失败**：返回 `-1`（如没有任何子进程需要等待，`errno` 设为 `ECHILD`）。

##### 所需头文件

| 头文件 | 必要性 | 说明 |
|--------|--------|------|
| `#include <sys/types.h>` | **强烈建议** | 提供 `pid_t` 类型定义。 |
| `#include <sys/wait.h>` | **必须** | 提供 `wait` 和 `waitpid` 的函数声明，以及所有状态解析宏（`WIFEXITED`、`WEXITSTATUS`、`WIFSIGNALED`、`WTERMSIG` 等）和选项常量（`WNOHANG`、`WUNTRACED` 等）的定义。POSIX 标准规定进程等待相关函数的声明位于此头文件中。 |

##### 参数详解
| 参数 | 说明 |
|------|------|
| `status` | 输出型参数（整型指针）。传入 `NULL` 表示不关心子进程退出详情，仅回收资源。 |

##### 核心限制
- 无法指定等待哪个子进程。
- 必须**阻塞**等待。若子进程一直不退出，父进程永久卡死在 `wait` 调用处。

---

#### 3. `waitpid` —— 可定制化等待（工业级首选）

##### 函数原型
```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *status, int options);
```

##### 功能说明
`wait` 的增强版，提供了**指定子进程**和**非阻塞**的能力，是生产环境中的标准选择。

##### 返回值
- **成功**：返回被回收的子进程 PID。
- **失败**：返回 `-1`，并设置 `errno`。
- **`WNOHANG` 模式下无子进程退出**：返回 `0`。

##### 所需头文件

| 头文件 | 必要性 | 说明 |
|--------|--------|------|
| `#include <sys/types.h>` | **强烈建议** | 提供 `pid_t` 类型定义。 |
| `#include <sys/wait.h>` | **必须** | 提供 `waitpid` 函数声明及所有相关宏定义。 |

##### 参数 `pid`：精确控制等待目标

| 传入值 | 效果 |
|--------|------|
| **`> 0`** | 精确等待 PID 等于该值的特定子进程 |
| **`-1`** | 等待**任意**子进程（等同于 `wait`） |
| **`0`** | 等待与调用者在**同一进程组**中的任意子进程 |
| **`< -1`** | 等待进程组 ID（PGID）等于 `pid` 绝对值的任意子进程 |

##### 参数 `options`：控制行为模式

通过按位或 `|` 组合多个选项：

| 选项 | 说明 |
|------|------|
| **`WNOHANG`**（最常用） | **非阻塞模式**。若子进程尚未退出，立即返回 `0`，不挂起父进程。是实现高性能服务器（如 Nginx、Redis）主循环的关键。 |
| **`WUNTRACED`** | 除了终止的进程，也捕捉因信号（如 `SIGTTIN`）而**暂停**（挂起）的子进程。 |
| **`WCONTINUED`** | 捕捉因 `SIGCONT` 信号而从暂停状态**恢复运行**的子进程（通常在作业控制 `fg`/`bg` 命令时发生）。 |

---

#### 4. 解析 `status` 状态值（位掩码宏详解）

`status` 是一个位掩码（Bit Mask），不能直接当整数看，必须用以下宏解析：

| 宏函数 | 判断条件 | 提取具体值 |
|--------|----------|------------|
| **`WIFEXITED(status)`** | 子进程是否**正常退出**（调用了 `exit` 或 `return`） | `WEXITSTATUS(status)` 提取退出码（如 `exit(5)` 返回 5） |
| **`WIFSIGNALED(status)`** | 子进程是否因**收到未捕获的信号**而异常终止（如段错误） | `WTERMSIG(status)` 提取导致终止的**信号编号**（如 `SIGSEGV` 为 11） |
| **`WIFSTOPPED(status)`** | 子进程是否因信号而**暂停**（需启用 `WUNTRACED`） | `WSTOPSIG(status)` 提取导致暂停的**信号编号**（如 `SIGSTOP`） |
| **`WIFCONTINUED(status)`** | 子进程是否**恢复运行**（需启用 `WCONTINUED`） | （无额外提取函数） |

---

#### 5. 核心难点：为什么必须调用 `wait`/`waitpid`？（僵尸进程）

##### 经典流程
1. 子进程退出时，内核**不会立即释放**它的所有资源。
2. 内核保留一个最小的数据结构（包含 PID、退出状态、CPU 时间等），并发送 `SIGCHLD` 信号通知父进程。
3. **父进程的职责**：必须通过 `wait`/`waitpid` 来“读取”这个状态信息。
4. **只有完成这个读操作**，内核才会彻底删除该子进程记录，释放其 PID。

##### 后果
如果父进程**一直不调用** `wait`（如有死循环且未处理信号），子进程会一直处于**僵尸状态**（`<defunct>`）。

- 僵尸进程**不占用内存**，但**占用 PID 资源**。
- 当 PID 耗尽时，系统将**无法创建任何新进程**，导致系统瘫痪。

---

#### 6. 最佳实战：优雅处理 `SIGCHLD` 并避免阻塞

在真实服务器编程中，标准做法是：

**在信号处理函数中循环调用 `waitpid`（非阻塞）**：

```c
#include <sys/wait.h>
#include <unistd.h>
#include <signal.h>
#include <stdio.h>

void sigchld_handler(int signo) {
    pid_t pid;
    int status;
    // 关键：使用 WNOHANG 非阻塞，循环回收直到没有子进程退出
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            printf("Child %d exited with code %d\n", pid, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child %d killed by signal %d\n", pid, WTERMSIG(status));
        }
    }
}

int main() {
    signal(SIGCHLD, sigchld_handler);
    // 业务逻辑...
    return 0;
}
```

##### 为什么必须循环调用？
- `SIGCHLD` 信号是**不可靠信号**（传统 Unix），多个子进程同时退出时可能只触发一次信号处理。
- 循环调用确保所有已退出的子进程都被回收，避免留下僵尸进程。

##### 双重保险
如果系统不支持可靠信号，可在 `sigaction` 中设置 `SA_RESTART` 标志，防止阻塞的系统调用被信号中断（`EINTR` 错误）。

---



