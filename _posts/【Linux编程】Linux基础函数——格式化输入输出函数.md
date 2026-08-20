---
title: 【Linux编程】Linux基础函数——格式化输入输出函数
date: 2026-08-13 14:32:44
tags:
  - C/C++
  - Linux编程
---

# Linux 基础函数——格式化输入输出函数

## 一、格式化输出函数族

### 1. 函数分类总览

C 语言提供了两组功能相似、但参数形式不同的格式化输出函数：

- **`printf` 族**：使用直接参数列表（变长参数 `...`）
- **`vprintf` 族**：使用 `va_list` 类型的参数列表（通常用于封装函数内部）

这两族函数又可以按输出目标分为三类：

| 输出目标     | `printf` 族           | `vprintf` 族            |
| ------------ | --------------------- | ----------------------- |
| 标准输出     | `printf`              | `vprintf`               |
| 字符串缓冲区 | `sprintf`、`snprintf` | `vsprintf`、`vsnprintf` |
| 文件流       | `fprintf`             | `vfprintf`              |

---

### 2. `printf` 族（直接参数版本）

#### （1）输出到标准输出

```c
int printf(const char *format, ...);
```

- **功能**：格式化输出到 `stdout`
- **返回值**：成功时输出字符数，失败返回负数

#### （2）输出到字符串缓冲区

```c
int sprintf(char *str, const char *format, ...);
int snprintf(char *str, size_t size, const char *format, ...);
```

- `sprintf`：将格式化结果写入 `str`，**不检查缓冲区大小**，可能导致缓冲区溢出（不安全）
- `snprintf`：最多写入 `size` 个字符（包括结尾 `\0`），**保证安全**，推荐使用
- **返回值**：`snprintf` 返回**实际所需字符数**（不包括 `\0`），若返回值 ≥ `size` 则说明发生了截断

#### （3）输出到文件流

```c
int fprintf(FILE *stream, const char *format, ...);
```

- **功能**：格式化输出到指定文件流

#### 示例：`printf` 族函数的使用

```cpp
#include <cstdio>
#include <iostream>

int main() {
    // 1. printf - 标准输出
    printf("printf: %d %ld %lld %f %e %g\n", 1, 2, 3, 0.00001, 0.00001, 0.00001);

    // 2. sprintf - 写入字符串（不安全，不检查长度）
    char buffer[128];
    sprintf(buffer, "sprintf: %s %d %g", "Hello World", 10, 0.00001);
    std::cout << buffer << std::endl;

    // 3. snprintf - 安全写入字符串（推荐）
    snprintf(buffer, sizeof(buffer), "snprintf: %s %d %g", "Hello World", 10, 0.00001);
    std::cout << buffer << std::endl;

    // 4. fprintf - 写入文件
    FILE* pFile = fopen("test.txt", "w+");
    if (pFile) {
        fprintf(pFile, "fprintf: %s %d %g\n", "Hello World", 10, 0.00001);
        fclose(pFile);
    }
    return 0;
}
```

---

### 3. `vprintf` 族（`va_list` 版本）

这些函数用于在自定义函数内部封装格式化输出，接受 `va_list` 类型的参数。

#### （1）输出到标准输出

```c
int vprintf(const char *format, va_list arg);
```

#### （2）输出到字符串缓冲区

```c
int vsprintf(char *str, const char *format, va_list arg);
int vsnprintf(char *s, size_t n, const char *format, va_list arg);
```

- `vsprintf`：不安全（不检查长度）
- `vsnprintf`：安全版本，限制最大写入 `n` 个字符

#### （3）输出到文件流

```c
int vfprintf(FILE *stream, const char *format, va_list arg);
```

#### 示例：`vprintf` 族函数的使用（封装测试函数）

```cpp
#include <cstdio>
#include <cstdarg>
#include <iostream>

void test(const char* fmt1, const char* fmt2, const char* fmt3, const char* fmt4, ...) {
    va_list ap;

    // 1. vprintf - 输出到标准输出
    va_start(ap, fmt4);
    vprintf(fmt1, ap);
    va_end(ap);

    // 2. vsprintf - 写入字符串（不安全）
    char buffer[128];
    va_start(ap, fmt4);
    vsprintf(buffer, fmt2, ap);
    va_end(ap);
    std::cout << buffer << std::endl;

    // 3. vsnprintf - 安全写入字符串（推荐）
    va_start(ap, fmt4);
    vsnprintf(buffer, sizeof(buffer), fmt3, ap);
    va_end(ap);
    std::cout << buffer << std::endl;

    // 4. vfprintf - 写入文件
    FILE* pFile = fopen("test1.txt", "w+");
    if (pFile) {
        va_start(ap, fmt4);
        vfprintf(pFile, fmt4, ap);
        va_end(ap);
        fclose(pFile);
    }
}

int main() {
    test("vprintf: %s %d %g\n",
         "vsprintf: %s %d %g",
         "vsnprintf: %s %d %g",
         "vfprintf: %s %d %g\n",
         "Hello World", 10, 0.00001);
    return 0;
}
```

---

### 4. 输出函数对照总结

| 函数        | 参数类型  | 目标     | 安全性     |
| ----------- | --------- | -------- | ---------- |
| `printf`    | `...`     | 标准输出 | 安全       |
| `sprintf`   | `...`     | 字符串   | **不安全** |
| `snprintf`  | `...`     | 字符串   | **安全**   |
| `fprintf`   | `...`     | 文件流   | 安全       |
| `vprintf`   | `va_list` | 标准输出 | 安全       |
| `vsprintf`  | `va_list` | 字符串   | **不安全** |
| `vsnprintf` | `va_list` | 字符串   | **安全**   |
| `vfprintf`  | `va_list` | 文件流   | 安全       |

**使用建议**：

- 任何时候优先使用 `snprintf` / `vsnprintf` 替代 `sprintf` / `vsprintf`
- 使用 `va_start` 和 `va_end` 必须成对出现，每次使用 `va_list` 前应重新调用 `va_start`
- 以上所有函数在格式化字符串时，都支持 `%d`、`%f`、`%s`、`%e`、`%g` 等常见格式化符号

---

## 二、格式化输入函数族

### 1. 函数分类总览

与 `printf` 族类似，`scanf` 族也分为两组：

- **`scanf` 族**：使用直接参数列表（变长参数 `...`）
- **`vscanf` 族**：使用 `va_list` 类型的参数列表

按数据来源分为三类：

| 数据来源 | `scanf` 族 | `vscanf` 族 |
| -------- | ---------- | ----------- |
| 标准输入 | `scanf`    | `vscanf`    |
| 字符串   | `sscanf`   | `vsscanf`   |
| 文件流   | `fscanf`   | `vfscanf`   |

---

### 2. `scanf` 族（直接参数版本）

#### （1）从标准输入读取

```c
int scanf(const char *format, ...);
```

- **功能**：从 `stdin` 读取格式化数据
- **返回值**：成功匹配并赋值的参数个数，失败返回 `EOF`

```c
int i;
float f;
scanf("%d %f", &i, &f);  // 输入：10 1.234
printf("i=%d, f=%f\n", i, f);
```

#### （2）从字符串读取

```c
int sscanf(const char *str, const char *format, ...);
```

- **功能**：从字符串 `str` 中解析格式化数据

```c
const char* data = "42 3.14159";
int i;
double d;
sscanf(data, "%d %lf", &i, &d);
printf("i=%d, d=%lf\n", i, d);
```

#### （3）从文件流读取

```c
int fscanf(FILE *stream, const char *format, ...);
```

- **功能**：从指定文件流中读取格式化数据

```c
FILE* fp = fopen("data.txt", "r");
int i;
char str[64];
if (fp) {
    fscanf(fp, "%s %d", str, &i);
    printf("str=%s, i=%d\n", str, i);
    fclose(fp);
}
```

---

### 3. `vscanf` 族（`va_list` 版本）

#### （1）从标准输入读取

```c
int vscanf(const char *format, va_list arg);
```

```c
#include <stdarg.h>

void read_int(const char* fmt, int* p) {
    va_list ap;
    va_start(ap, p);
    vscanf(fmt, ap);
    va_end(ap);
}

int main() {
    int i;
    read_int("%d", &i);
    printf("i=%d\n", i);
    return 0;
}
```

#### （2）从字符串读取

```c
int vsscanf(const char *s, const char *format, va_list arg);
```

```c
#include <stdarg.h>

void parse_data(const char* str, const char* fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    vsscanf(str, fmt, ap);
    va_end(ap);
}

int main() {
    int i;
    float f;
    parse_data("100 2.718", "%d %f", &i, &f);
    printf("i=%d, f=%f\n", i, f);
    return 0;
}
```

#### （3）从文件流读取

```c
int vfscanf(FILE *stream, const char *format, va_list arg);
```

```c
#include <stdarg.h>

void read_from_file(FILE* fp, const char* fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    vfscanf(fp, fmt, ap);
    va_end(ap);
}

int main() {
    FILE* fp = fopen("data.txt", "r");
    int i;
    float f;
    if (fp) {
        read_from_file(fp, "%d %f", &i, &f);
        printf("i=%d, f=%f\n", i, f);
        fclose(fp);
    }
    return 0;
}
```

---

### 4. `scanf` 系列使用注意事项

#### （1）空格是默认分隔符

`scanf` 默认以空白字符（空格、制表符、换行）作为输入项之间的分隔符。

#### （2）格式字符串中的空格可有可无

格式字符串中的空格是宽松的，即使没有空格，输入时也可以用空格分隔。

```c
int i; float f;
scanf("%d%f", &i, &f);   // 输入：10 1.234  ✅ 正确
scanf("%d%f", &i, &f);   // 输入：10,1.234 ❌ 错误（逗号不被识别）
```

#### （3）部分格式能自动分割，部分不能自动分割

- 整数和字符串之间可自动分割：`%d%s` → `10hello` 会自动分割为 `10` 和 `"hello"`
- **整数和小数之间必须手动用空格分隔**：`%d%f` → `10 1.234` ✅ 必须加空格

#### （4）自定义分隔符必须严格匹配

如果格式字符串中使用了非空白分隔符，输入时必须原样输入。

```c
int i; float f;
scanf("%d,%f", &i, &f);  // 格式中用逗号分隔
// 输入：10,1.234 ✅  输入：10 1.234 ❌
```

#### （5）取地址符是必需的

`scanf` 需要修改变量的值，因此必须传递变量的地址。

```c
int i;
scanf("%d", &i);   // ✅ 正确
scanf("%d", i);    // ❌ 错误（传值而非地址）
```

#### （6）`%f` 和 `%lf` 在 `scanf` 中严格区分

> 与 `printf` 不同，`printf` 中 `float` 传入时自动提升为 `double`，所以 `%f` 和 `%lf` 都对应 `double`。而 `scanf` 中 `%f` 对应 `float*`，`%lf` 对应 `double*`，必须严格匹配。

```c
float f;
double d;
scanf("%f", &f);   // ✅ 正确
scanf("%lf", &d);  // ✅ 正确
scanf("%f", &d);   // ❌ 类型不匹配（未定义行为）
scanf("%lf", &f);  // ❌ 类型不匹配（未定义行为）
```

---

### 5. 输入函数对照总结

| 函数      | 参数类型  | 数据来源 | 返回值     |
| --------- | --------- | -------- | ---------- |
| `scanf`   | `...`     | 标准输入 | 成功匹配数 |
| `sscanf`  | `...`     | 字符串   | 成功匹配数 |
| `fscanf`  | `...`     | 文件流   | 成功匹配数 |
| `vscanf`  | `va_list` | 标准输入 | 成功匹配数 |
| `vsscanf` | `va_list` | 字符串   | 成功匹配数 |
| `vfscanf` | `va_list` | 文件流   | 成功匹配数 |

**使用要点总结**：

1. 始终为变量取地址（`&`）
2. 整数和小数之间必须用空格分隔
3. 自定义分隔符输入时必须原样输入
4. `%f` 配 `float*`，`%lf` 配 `double*`，不可混用
5. 返回值可用于检测输入是否成功

---

## 三、`printf` 函数 format 详解

### 1. 格式字符

| 格式字符  | 意义                                               |
| --------- | -------------------------------------------------- |
| `d`       | 以十进制形式输出带符号整数（正数不输出符号）       |
| `o`       | 以八进制形式输出无符号整数（不输出前缀 0）         |
| `x` / `X` | 以十六进制形式输出无符号整数（不输出前缀 0x）      |
| `u`       | 以十进制形式输出无符号整数                         |
| `f`       | 以小数形式输出单、双精度实数                       |
| `E` / `e` | 以指数形式输出单、双精度实数                       |
| `G` / `g` | 以 `%f` 或 `%e` 中较短的输出宽度输出单、双精度实数 |
| `c`       | 输出单个字符                                       |
| `s`       | 输出字符串                                         |
| `p`       | 输出指针地址                                       |
| `lu`      | 32 位无符号整数                                    |
| `llu`     | 64 位无符号整数                                    |

### 2. flags（标识）

| flags | 描述                                                         |
| ----- | ------------------------------------------------------------ |
| `-`   | 在给定的字段宽度内左对齐，默认是右对齐                       |
| `+`   | 强制在结果之前显示加号或减号，即正数前面会显示 `+` 号。默认情况下，只有负数前面会显示 `-` 号 |
| 空格  | 如果没有写入任何符号，则在该值前面插入一个空格               |
| `#`   | 与 `o`、`x` 或 `X` 一起使用时，非零值前面会分别显示 `0`、`0x` 或 `0X`；与 `e`、`E` 和 `f` 一起使用时，会强制输出包含一个小数点；与 `g` 或 `G` 一起使用时，结果与使用 `e` 或 `E` 时相同，但尾部的零不会被移除 |
| `0`   | 在指定填充的数字左边放置零（`0`），而不是空格                |

### 3. width（宽度）

| width      | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| `(number)` | 要输出的字符的最小数目。如果输出的值短于该数，结果会用空格填充；如果长于该数，结果不会被截断 |
| `*`        | 宽度在 format 字符串中未指定，作为附加整数值参数放置于要被格式化的参数之前 |

### 4. .precision（精度）

| .precision | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| `.number`  | 对于整数说明符（`d`、`i`、`o`、`u`、`x`、`X`）：指定要写入的数字的最小位数，不足时用前导零填充，超出不截断。精度为 0 意味着不写入任何字符。<br>对于 `e`、`E` 和 `f`：小数点后要输出的小数位数。<br>对于 `g` 和 `G`：要输出的最大有效位数。<br>对于 `s`：要输出的最大字符数，默认输出所有字符直到遇到空字符。<br>对于 `c`：没有任何影响。<br>未指定精度时默认为 1；指定时不带显式值则假定为 0 |
| `.*`       | 精度在 format 字符串中未指定，作为附加整数值参数放置于要被格式化的参数之前 |

### 5. length（长度）

| length        | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `h`           | 参数被解释为短整型或无符号短整型（仅适用于整数说明符：`i`、`d`、`o`、`u`、`x` 和 `X`） |
| `l`（小写 L） | 参数被解释为长整型或无符号长整型，适用于整数说明符（`i`、`d`、`o`、`u`、`x` 和 `X`）及说明符 `c`（宽字符）和 `s`（宽字符字符串） |
| `L`           | 参数被解释为长双精度型（仅适用于浮点数说明符：`e`、`E`、`f`、`g` 和 `G`） |

**附加参数**：根据不同的 format 字符串，函数可能需要一系列的附加参数，每个参数包含了一个要被插入的值，替换了 format 参数中指定的每个 `%` 标签。参数的个数应与 `%` 标签的个数相同。

---

## 四、`scanf` 函数 format 详解

### 1. 格式参数

| 参数        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `*`         | 可选的星号，表示数据是从流中读取的，但可以被忽视，即不存储在对应的参数中 |
| `width`     | 指定当前读取操作中读取的最大字符数                           |
| `modifiers` | 为对应的附加参数所指向的数据指定大小：`h`：短整型（针对 `d`、`i` 和 `n`），或无符号短整型（针对 `o`、`u` 和 `x`）；`l`：长整型（针对 `d`、`i` 和 `n`），或无符号长整型（针对 `o`、`u` 和 `x`），或双精度型（针对 `e`、`f` 和 `g`）；`L`：长双精度型（针对 `e`、`f` 和 `g`） |
| `type`      | 一个字符，指定要被读取的数据类型以及数据读取方式             |

### 2. 类型字符

| 类型                               | 合格的输入                                                   | 参数的类型       |
| ---------------------------------- | ------------------------------------------------------------ | ---------------- |
| `%a`、`%A`                         | 读入一个浮点值（仅 C99 有效）                                | `float *`        |
| `%c`                               | 单个字符：读取下一个字符。如果指定了不为 1 的宽度 width，函数会读取 width 个字符，存储在数组连续位置，末尾不会追加空字符 | `char *`         |
| `%d`                               | 十进制整数：数字前面的 `+` 或 `-` 号是可选的                 | `int *`          |
| `%e`、`%E`、`%f`、`%F`、`%g`、`%G` | 浮点数：包含一个小数点、可选的前置符号 `+` 或 `-`、可选的后置字符 `e` 或 `E`，以及一个十进制数字 | `float *`        |
| `%lf`                              | 双精度数输入                                                 | `double *`       |
| `%i`                               | 读入十进制、八进制、十六进制整数                             | `int *`          |
| `%o`                               | 八进制整数                                                   | `int *`          |
| `%s`                               | 字符串：读取连续字符，直到遇到空白字符（空格、换行和制表符） | `char *`         |
| `%u`                               | 无符号的十进制整数                                           | `unsigned int *` |
| `%x`、`%X`                         | 十六进制整数                                                 | `int *`          |
| `%p`                               | 读入一个指针                                                 | —                |
| `%[]`                              | 扫描字符集合                                                 | —                |
| `%%`                               | 读 `%` 符号                                                  | —                |

**附加参数**：根据不同的 format 字符串，函数可能需要一系列的附加参数，每个参数包含了一个要被插入的值，替换了 format 参数中指定的每个 `%` 标签。参数的个数应与 `%` 标签的个数相同。
