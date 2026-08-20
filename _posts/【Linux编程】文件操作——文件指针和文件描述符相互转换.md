---
title: 【Linux编程】文件操作——文件指针和文件描述符相互转换
date: 2026-08-19 11:38:01
tags:
  - C/C++
  - Linux编程
---

### 文件指针与文件描述符转换函数详解

---

#### 使用背景

在 Linux 系统编程中，存在两套文件操作接口：

| 层级             | 接口类型                | 代表函数                                  | 特点                                                         |
| ---------------- | ----------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| **操作系统层面** | 文件描述符 `fd`（整数） | `open`、`read`、`write`、`ioctl`、`fcntl` | 底层接口，功能强大，可精细控制，符合 Linux "一切皆文件" 的设计哲学 |
| **C 标准库层面** | 文件指针 `FILE*`（流）  | `fopen`、`fprintf`、`fgets`、`fscanf`     | 高级接口，带用户态缓冲区，使用方便，跨平台可移植性好         |

**两种接口各有优势**：
- 有时需要使用操作系统提供的接口（`fd`）对文件做更精细的操作（如 `fcntl`、`ioctl`、`fsync`）。
- 有时需要使用 C 标准库提供的接口（`fp`）做更抽象高级的操作（如 `fprintf` 格式化输出、`fgets` 按行读取）。

因此，C 语言提供了让文件指针和文件描述符**相互转换**的接口，使开发者可以根据需要在两种接口之间灵活切换。

---

#### 1. `fdopen` —— 文件描述符 → 文件指针

##### 函数原型
```c
#include <stdio.h>

FILE *fdopen(int fd, const char *mode);
```

##### 功能说明
将一个已经打开的文件描述符（`fd`）**包装**成一个标准 I/O 流（`FILE*`），使其能够使用 `fprintf`、`fgets` 等高级缓冲 I/O 函数进行操作。

##### 返回值
- **成功**：返回指向 `FILE` 结构的指针。
- **失败**：返回 `NULL`，并设置 `errno`。

##### 所需头文件

| 头文件               | 必要性   | 说明                                                         |
| -------------------- | -------- | ------------------------------------------------------------ |
| `#include <stdio.h>` | **必须** | C 标准库规定 `fdopen` 函数的声明及 `FILE` 类型定义位于 `<stdio.h>` 中。 |

##### 参数详解
| 参数   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| `fd`   | 已经打开的文件描述符（由 `open`、`pipe`、`socket`、`dup` 等返回） |
| `mode` | 流的打开模式，与 `fopen` 相同（见下方说明）                  |

##### `mode` 参数关键区别（与 `fopen` 对比）

| mode 值        | 与 `fopen` 的关键区别                                        |
| -------------- | ------------------------------------------------------------ |
| `"w"` / `"w+"` | **不会截断文件**。文件已在 `open` 时确定是否截断（由 `O_TRUNC` 标志决定） |
| `"a"` / `"a+"` | **无法创建文件**。文件描述符已存在，说明文件已打开           |

> **重要提示**：传入的 `mode` **必须**与文件描述符 `fd` 本身的读写权限兼容。例如，`O_RDONLY` 的描述符不能用 `"w"` 模式调用 `fdopen`，否则返回 `NULL`。

##### 核心行为与注意事项
1. **文件位置**：新流继承文件描述符的当前偏移量，并清空错误和 EOF 指示器。
2. **关闭行为**：调用 `fclose` 关闭流时，**会同时关闭底层文件描述符** `fd`。`fclose` 之后不要再次使用该 `fd`。
3. **缓冲问题**：`fdopen` 创建的流使用标准 I/O 缓冲。若混合使用流操作（`fprintf`）和系统调用（`write`），需注意缓冲区刷新，否则可能导致数据顺序混乱。

##### 典型应用场景
- **管道和网络套接字**：`pipe()` 和 `socket()` 返回的是 `fd`，无法用 `fopen` 直接打开。`fdopen` 可将它们封装成流，方便使用 `fprintf` 等函数。
- **重定向标准流**：将 `open` 返回的描述符封装为 `FILE*`，用于重定向 `stdin`、`stdout`、`stderr`。

##### 使用示例
```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 1. 用 open 打开文件，获得文件描述符
    int fd = open("example.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }

    // 2. 用 fdopen 将 fd 转换为 FILE* 流
    FILE *stream = fdopen(fd, "w");
    if (stream == NULL) {
        perror("fdopen failed");
        close(fd);  // fdopen 失败，需手动关闭 fd
        return 1;
    }

    // 3. 使用标准 I/O 函数操作流
    fprintf(stream, "Hello, world!\n");

    // 4. 关闭流（自动关闭底层 fd）
    fclose(stream);
    // 注意：此时不要再使用 fd
    return 0;
}
```

---

#### 2. `fileno` —— 文件指针 → 文件描述符

##### 函数原型
```c
#include <stdio.h>

int fileno(FILE *stream);
```

##### 功能说明
提取一个标准 I/O 流（`FILE*`）底层所关联的文件描述符（`fd`）。与 `fdopen` 方向相反。

##### 返回值
- **成功**：返回与流关联的文件描述符（非负整数）。
- **失败**：返回 `-1`。

##### 所需头文件

| 头文件               | 必要性   | 说明                                                         |
| -------------------- | -------- | ------------------------------------------------------------ |
| `#include <stdio.h>` | **必须** | POSIX 标准规定 `fileno` 函数的声明位于 `<stdio.h>` 中。注意：`fileno` 不是 ISO C 标准函数，而是 POSIX 扩展。 |

##### 核心行为与注意事项
1. **共享同一文件表项**：`fileno` 返回的是流内部维护的 `fd`，与流**共享同一个文件表项**（文件位置、打开模式等）。对流或描述符的任何操作都会互相影响。
2. **关于缓冲的陷阱（非常重要）**：
   - 标准 I/O 流默认带有用户态缓冲区。
   - 若先用 `fprintf` 写入，再通过 `fileno` 获取的 `fd` 调用 `write`，数据顺序可能混乱（`fprintf` 的数据可能还在缓冲区中）。
   - **解决方案**：混用前调用 `fflush(stream)` 刷新缓冲区，或用 `setbuf(stream, NULL)` 禁用缓冲。
3. **关闭顺序的影响**：
   - 先 `close(fd)` → 流变得不可用。
   - 先 `fclose(stream)` → `fd` 也被关闭，之后不能再使用。

##### 典型应用场景
- **需要底层系统调用时**：如设置非阻塞模式（`fcntl(fd, F_SETFL, O_NONBLOCK)`）、强制同步到磁盘（`fsync(fd)`）。
- **与 `select`/`poll`/`epoll` 配合**：这些 I/O 多路复用函数需要 `fd`，可从流中提取。
- **处理网络套接字**：用 `socket()` 创建套接字并用 `fdopen` 包装成流后，如需调用 `setsockopt` 或 `shutdown`，需用 `fileno` 取回原始 `fd`。

##### 使用示例
```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    // 1. 用 fopen 打开文件流
    FILE *stream = fopen("example.txt", "w+");
    if (stream == NULL) {
        perror("fopen failed");
        return 1;
    }

    // 2. 写入数据（暂存在用户态缓冲区）
    fprintf(stream, "This data is buffered.\n");

    // 3. 获取底层文件描述符
    int fd = fileno(stream);
    if (fd == -1) {
        perror("fileno failed");
        fclose(stream);
        return 1;
    }

    // 4. 关键：混用系统调用前，先刷新流缓冲区
    fflush(stream);

    // 5. 通过 fd 执行底层操作：强制同步到磁盘
    if (fsync(fd) == -1) {
        perror("fsync failed");
    }

    // 6. 继续使用流写入
    fprintf(stream, "Another line after fsync.\n");

    // 7. 关闭流（自动关闭 fd）
    fclose(stream);
    return 0;
}
```

---

#### 对比总结

| 函数         | 方向        | 输入            | 输出            | 主要用途                                                |
| ------------ | ----------- | --------------- | --------------- | ------------------------------------------------------- |
| **`fdopen`** | 描述符 → 流 | 文件描述符 `fd` | `FILE*` 流      | 让底层 `fd` 享受标准 I/O 的便利（如 `fprintf`）         |
| **`fileno`** | 流 → 描述符 | `FILE*` 流      | 文件描述符 `fd` | 让标准 I/O 流能参与底层系统调用（如 `fcntl`、`select`） |

---

#### 特别提醒
- **不要滥用**：大部分文件操作可用标准 I/O 完成。只在确实需要底层控制时（如非阻塞 I/O、多路复用、直接 `write`/`read`）才使用 `fileno`。
- **可移植性**：`fileno` 是 POSIX 扩展，非 ISO C 标准。Windows 环境下对应函数为 `_fileno`（带下划线），使用前需注意平台差异。

---

#### 快速参考

| 函数     | 头文件      | 核心用途       | 是否 POSIX     |
| -------- | ----------- | -------------- | -------------- |
| `fdopen` | `<stdio.h>` | `fd` → `FILE*` | 是             |
| `fileno` | `<stdio.h>` | `FILE*` → `fd` | 是（非 ISO C） |

