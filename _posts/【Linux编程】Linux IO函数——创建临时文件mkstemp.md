---
title: 【Linux编程】Linux IO函数——创建临时文件mkstemp
date: 2026-08-14 21:12:07
tags:
  - C/C++
  - Linux编程
---

### `mkstemp` 函数详解

#### 1. 函数原型

```c
int mkstemp(char *template);
```

---

#### 2. 功能说明

生成一个**唯一的临时文件名**，并**原子地创建并打开**该文件，返回文件描述符。该文件以读写模式打开，权限为 `0600`（仅所有者可读写），且文件是临时的，不保证数据长期有效。

---

#### 3. 返回值

- **成功**：返回文件描述符（非负整数），可用于 `read`、`write`、`close` 等操作。
- **失败**：返回 `-1`，并设置全局 `errno` 以指示具体错误。

---

#### 4. 所需头文件

| 头文件                | 必要性   | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| `#include <stdlib.h>` | **必须** | 提供 `mkstemp` 函数声明。此头文件是 C 标准库的一部分，包含内存分配、进程控制、字符串转换等实用函数。`mkstemp` 的 POSIX 标准声明即位于此头文件中。 |

---

#### 5. `template` 参数格式

- **格式**：前缀 + `XXXXXX`（六个大写的 X）
- 前缀可以为任意字符串（含路径），但**后缀必须恰好是六个 `X`**
- 函数会**替换**这六个 `X` 为随机字符，以生成唯一文件名

**示例**：

```c
char template[] = "/tmp/myapp_temp_XXXXXX";
mkstemp(template);
// 执行后，template 可能变为 "/tmp/myapp_temp_AbC3Xy"
```

---

#### 6. 历史渊源：`mktemp()` 与安全问题

在 `mkstemp()` 出现之前，创建一个临时文件的“标准”做法是使用 **`mktemp()`** 函数（注意没有 `s`）：

```c
char *mktemp(char *template);
```

- `mktemp()` **只生成一个不存在的文件名**，但**不会实际创建文件**
- 这会导致一个严重的安全问题：**竞态条件（TOCTOU）**

---

#### 7. 竞态条件问题（TOCTOU）

典型的错误用法：

```c
char name[] = "/tmp/fileXXXXXX";
mktemp(name);          // 生成一个不存在的文件名
int fd = open(name, O_CREAT | O_RDWR, 0600);  // 再创建文件
```

**问题**：在 `mktemp()` 和 `open()` 之间，攻击者可以：

1. 预测你将要创建的文件名
2. 抢先创建一个同名文件（可能是指向敏感文件的符号链接）
3. 当你 `open()` 时，可能覆盖或破坏攻击者指定的文件

这就是典型的 **TOCTOU（Time Of Check To Time Of Use）** 漏洞。

---

#### 8. `mkstemp()` 的安全性

```c
int mkstemp(char *template);
```

- `mkstemp()` 中的 **`s` = `secure`**
- 它**原子地**完成两件事：
	1. 生成一个唯一的文件名
	2. **立即创建并打开该文件**（返回文件描述符）
- 两步操作之间没有时间窗口，从根本上杜绝了竞态条件

此外，`mkstemp()` 还会：

- 以 **读写模式** 打开文件
- 设置权限为 **0600**（仅所有者可读写）
- 确保文件**不会被其他进程劫持**

---

#### 9. 相关函数对比

| 函数      | 含义                                   | 安全性                                              |
| --------- | -------------------------------------- | --------------------------------------------------- |
| `mktemp`  | `make temp`（生成临时文件名）          | **不安全**，仅生成名字不创建文件，存在 TOCTOU 漏洞  |
| `mkstemp` | `make secure temp`（安全创建临时文件） | **安全**，原子创建+打开，返回 fd                    |
| `tmpfile` | 标准库函数，自动创建临时文件（流式）   | **安全**，返回 `FILE*` 流，程序关闭或结束时自动删除 |

---

#### 10. 使用示例

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    char template[] = "/tmp/myapp_XXXXXX";
    
    int fd = mkstemp(template);
    if (fd < 0) {
        perror("mkstemp");
        return 1;
    }
    
    printf("临时文件已创建：%s\n", template);
    
    // 使用临时文件...
    write(fd, "temp data", 9);
    
    // 关闭并删除临时文件（可选）
    close(fd);
    unlink(template);  // 删除文件
    // 即使 unlink 后，只要 fd 还开着，进程仍可继续读写
    
    return 0;
}
```

---

#### 11. 总结

**`s` = `secure`**，表示这个函数是 `mktemp()` 的安全增强版本，通过**原子创建+打开**来避免 TOCTOU 竞态条件漏洞。

> **现代编程中，应该始终使用 `mkstemp()` 而不是 `mktemp()`**——实际上，`mktemp()` 在 Linux 手册中已经被标记为 **DEPRECATED（废弃）** 了。
