---
title: 【Linux编程】进程控制函数——进程跳转
date: 2026-08-16 15:57:47
tags:
  - C/C++
  - Linux编程
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
