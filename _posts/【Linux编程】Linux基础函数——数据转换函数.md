---
title: 【Linux编程】Linux基础函数——数据转换函数
date: 2026-08-12 15:37:25
tags:
  - C/C++
  - Linux编程
---

## 概述
数据转换函数主要用于**字符串与数值**之间的相互转换。这些函数分为两大类：
1. **字符串 → 数值**（解析转换）
2. **数值 → 字符串**（格式化转换）

---

## 一、函数命名规则详解

### 1. 简称释义表

| 缩写     | 全称               | 含义                        |
| -------- | ------------------ | --------------------------- |
| **a**    | ASCII              | ASCII字符串（早期命名习惯） |
| **to**   | to                 | 转换为...                   |
| **i**    | integer            | 整数                        |
| **l**    | long               | 长整数                      |
| **ll**   | long long          | 长长整数（64位）            |
| **f**    | float              | 单精度浮点数                |
| **d**    | double             | 双精度浮点数                |
| **ld**   | long double        | 长双精度浮点数              |
| **str**  | string             | 字符串                      |
| **ul**   | unsigned long      | 无符号长整数                |
| **ull**  | unsigned long long | 无符号长长整数              |
| **base** | base/radix         | 进制基数                    |

### 2. 函数命名拆解示例

```
atof  = ASCII to float
atoi  = ASCII to integer
atol  = ASCII to long
strtol = string to long

ecvt   = 指数形式 (exponential) convert value
fcvt   = 固定小数 (fixed) convert value  
gcvt   = 通用格式 (general) convert value
```

---

## 二、字符串 → 数值 转换函数

### 1. 简单转换函数（`ato*` 系列）

这些函数**没有错误检测**机制，遇到非法字符会返回0或未定义行为。

| 函数原型                           | 功能               | 命名拆解               |
| ---------------------------------- | ------------------ | ---------------------- |
| `double atof(const char* str)`     | 字符串 → double    | ASCII → to → float     |
| `int atoi(const char* str)`        | 字符串 → int       | ASCII → to → integer   |
| `long atol(const char* str)`       | 字符串 → long      | ASCII → to → long      |
| `long long atoll(const char* str)` | 字符串 → long long | ASCII → to → long long |

**示例代码**：
```cpp
#include <stdio.h>
#include <stdlib.h>

void demo_ato_functions() {
    const char* num_str = "  123.45abc";
    
    printf("atoi: %d\n", atoi(num_str));      // 输出: 123
    printf("atol: %ld\n", atol(num_str));     // 输出: 123
    printf("atoll: %lld\n", atoll(num_str));  // 输出: 123
    printf("atof: %f\n", atof(num_str));      // 输出: 123.450000
}
```

⚠️ **注意**：`ato*` 函数遇到 `"abc"` 会返回0，无法区分是真正的"0"还是解析失败。

---

### 2. 安全转换函数（`strto*` 系列）

这些函数**支持错误检测**，可以通过 `endptr` 参数判断转换是否完全成功。

#### 参数说明

| 参数     | 类型          | 说明                                            |
| -------- | ------------- | ----------------------------------------------- |
| `str`    | `const char*` | 待转换的字符串（输入）                          |
| `endptr` | `char**`      | 指向第一个无法转换字符的指针（输出），可为 NULL |
| `base`   | `int`         | 进制基数（2-36），0表示自动检测                 |

**`endptr` 使用示例**：
```cpp
char str[] = "123abc";
char* endptr;
long val = strtol(str, &endptr, 10);

printf("转换值: %ld\n", val);   // 123
printf("剩余字符串: %s\n", endptr); // "abc"
if (*endptr != '\0') {
    printf("⚠️ 字符串未完全转换!\n");
}
```

#### 函数列表

| 函数原型                                                     | 功能                        | 命名拆解                         |
| ------------------------------------------------------------ | --------------------------- | -------------------------------- |
| `long strtol(const char* str, char** endptr, int base)`      | 字符串 → long               | string → to → long               |
| `unsigned long strtoul(const char* str, char** endptr, int base)` | 字符串 → unsigned long      | string → to → unsigned long      |
| `long long strtoll(const char* str, char** endptr, int base)` | 字符串 → long long          | string → to → long long          |
| `unsigned long long strtoull(const char* str, char** endptr, int base)` | 字符串 → unsigned long long | string → to → unsigned long long |

#### 浮点数转换函数

| 函数原型                                              | 功能                 | 命名拆解                  |
| ----------------------------------------------------- | -------------------- | ------------------------- |
| `float strtof(const char* str, char** endptr)`        | 字符串 → float       | string → to → float       |
| `double strtod(const char* str, char** endptr)`       | 字符串 → double      | string → to → double      |
| `long double strtold(const char* str, char** endptr)` | 字符串 → long double | string → to → long double |

---

### 3. 完整示例：`strto*` 系列

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

void demo_strto_functions() {
    const char* numbers[] = {
        "123",
        "  456abc",
        "0x1A",      // 十六进制
        "0777",      // 八进制
        "3.14159xyz",
        "not a number"
    };
    
    printf("=== strtol 演示 ===\n");
    for (int i = 0; i < 6; i++) {
        char* endptr;
        errno = 0;  // 重置错误标志
        
        long val = strtol(numbers[i], &endptr, 0);  // base=0 自动检测进制
        
        printf("\n输入: \"%s\"\n", numbers[i]);
        printf("  转换值: %ld\n", val);
        printf("  剩余: \"%s\"\n", endptr);
        
        if (errno == ERANGE) {
            printf("  ⚠️ 数值超出范围!\n");
        } else if (*endptr == '\0') {
            printf("  ✅ 完全转换成功\n");
        } else if (endptr == numbers[i]) {
            printf("  ❌ 没有转换任何字符\n");
        }
    }
}

void demo_strtod_functions() {
    const char* float_str[] = {"3.14159", "   -2.718e-5abc", "1.0e308"};
    
    printf("\n=== strtod 演示 ===\n");
    for (int i = 0; i < 3; i++) {
        char* endptr;
        errno = 0;
        
        double val = strtod(float_str[i], &endptr);
        
        printf("\n输入: \"%s\"\n", float_str[i]);
        printf("  转换值: %f\n", val);
        printf("  剩余: \"%s\"\n", endptr);
        
        if (errno == ERANGE) {
            printf("  ⚠️ 数值超出范围! (溢出或下溢)\n");
        }
    }
}

int main() {
    demo_strto_functions();
    demo_strtod_functions();
    return 0;
}
```

---

### 4. `ato*` vs `strto*` 对比

| 特性     | `ato*` 系列        | `strto*` 系列      |
| -------- | ------------------ | ------------------ |
| 错误检测 | ❌ 无法检测         | ✅ 通过 endptr 检测 |
| 进制支持 | ❌ 仅十进制         | ✅ 支持 2-36 进制   |
| 溢出检测 | ❌ 未定义行为       | ✅ 通过 errno 检测  |
| 使用场景 | 确定格式正确的输入 | 需要验证的用户输入 |

---

## 三、数值 → 字符串 转换函数（老式）

> ⚠️ **注意**：`ecvt`、`fcvt`、`gcvt` 是**已过时**的函数（POSIX标准，非ANSI C），不推荐在新代码中使用。推荐使用 `sprintf` / `snprintf` 替代。

### 1. `ecvt()` - 指数形式转换，总共保留多少位

```cpp
char* ecvt(double value, int ndigit, int *decpt, int *sign);
```

**参数详解**：

| 参数     | 说明                                      |
| -------- | ----------------------------------------- |
| `value`  | 要转换的双精度数                          |
| `ndigit` | **全部有效位数**（包括整数部分+小数部分） |
| `decpt`  | 输出参数，小数点的位置索引                |
| `sign`   | 输出参数，0=正数，非0=负数                |

**示例**：
```cpp
#include <stdio.h>
#include <stdlib.h>

void demo_ecvt() {
    	double num = 123.456789;
	int decpt, sign;

	char* str1 = ecvt(num, 6, &decpt, &sign);  // 总共保留6位有效数字
	printf("ecvt(123.456, 6): %s\n", str1);
	printf("小数点位置: %d\n", decpt);
	printf("符号: %d\n", sign);
}
```

**输出**：

```text
ecvt(123.456, 6): 123457
小数点位置: 3
符号: 0
```



### 2. `fcvt()` - 固定小数位转换，小数点后保留多少位

```cpp
char* fcvt(double value, int ndigit, int *decpt, int *sign);
```

**参数详解**：

| 参数     | 说明                               |
| -------- | ---------------------------------- |
| `value`  | 要转换的双精度数                   |
| `ndigit` | **小数点后的有效位数**（小数位数） |
| `decpt`  | 输出参数，小数点的位置索引         |
| `sign`   | 输出参数，0=正数，非0=负数         |

**示例**：
```cpp
void demo_fcvt() {
    double num = 123.456;
    int decpt, sign;
    char* str = fcvt(num, 2, &decpt, &sign);  // 保留2位小数
    
    // 结果: "12346", decpt=3 (四舍五入，456→46)
    // 实际值: 123.456 ≈ 123.46
    printf("\nfcvt(123.456, 2):\n");
    printf("  字符串: %s\n", str);
    printf("  小数点位置: %d\n", decpt);
}
```

**输出**：

```text
fcvt(-123.456789, 9): 123456789000
小数点位置: 3
符号: 1
```



### 3. `gcvt()` - 通用格式转换，总共最多保留多少位

```cpp
char* gcvt(double value, int ndigit, char *buf);
```

**参数详解**：

| 参数     | 说明                           |
| -------- | ------------------------------ |
| `value`  | 要转换的双精度数               |
| `ndigit` | **最大有效总共位数**           |
| `buf`    | 输出缓冲区，存储转换后的字符串 |

**示例**：

```cpp
void demo_gcvt() {
	double nums[] = { 123.456, 0.000123, 1234567.89, -0.012, 123456789, 10 };
	char buf[64];

	printf("\n=== gcvt 演示 ===\n");
	for (int i = 0; i < sizeof(nums)/sizeof(double); i++) {
   		gcvt(nums[i], 6, buf);  // 最大6位有效数字
    	printf("gcvt(%.2f, 6): %s\n", nums[i], buf);
	}
}
```

**输出**：

```text
=== gcvt 演示 ===
gcvt(123.46, 6): 123.456
gcvt(0.00, 6): 0.000123
gcvt(1234567.89, 6): 1.23457e+06
gcvt(-0.01, 6): -0.012
gcvt(123456789.00, 6): 1.23457e+08
gcvt(10.00, 6): 10
```



### 4. `ecvt` vs `fcvt` vs `gcvt` 对比

`ecvt` 和 `fcvt` 更多的是对 **浮点数** 做**格式上的解析**。

更多的还是用`gcvt`将 浮点数 转为 字符串

| 函数   | ndigit 含义  | 输出格式         | 典型用途 |
| ------ | ------------ | ---------------- | -------- |
| `ecvt` | 总有效位数   | 自动科学计数法   | 科学计算 |
| `fcvt` | 小数位数     | 固定小数位       | 财务计算 |
| `gcvt` | 最大有效位数 | 自动选择最简格式 | 通用输出 |

---

### 5.`ecvt()` 和 `fcvt()` 使用后是否需要释放空间

**不需要，也绝对不能**对 `ecvt()` 和 `fcvt()` 返回的指针调用 `free()`。

这是由这两个函数的工作方式决定的：

⭐**它们返回指向静态内存的指针**

`ecvt()` 和 `fcvt()` 函数返回的指针，指向一个**函数内部静态分配的缓冲区**。

- **没有动态分配内存**：因为内存不是通过 `malloc`、`calloc` 或 `realloc` 在堆上分配的，所以使用 `free()` 来释放它是错误的，会导致未定义行为（通常是程序崩溃）。
- **缓冲区会被覆盖**：这个静态缓冲区只有一份，每次调用 `ecvt()` 或 `fcvt()` 时，里面的内容都会被新的调用结果覆盖。

---

### 6. 现代替代方案：`snprintf()`

```cpp
void modern_alternative() {
    double num = 123.456789;
    char buffer[64];
    
    // 等价于 ecvt(num, 6, ...)
    snprintf(buffer, sizeof(buffer), "%.6g", num);
    printf("科学格式: %s\n", buffer);  // 123.457
    
    // 等价于 fcvt(num, 2, ...)
    snprintf(buffer, sizeof(buffer), "%.2f", num);
    printf("固定格式: %s\n", buffer);  // 123.46
    
    // 通用格式
    snprintf(buffer, sizeof(buffer), "%g", num);
    printf("通用格式: %s\n", buffer);  // 123.457
}
```

---

## 四、关于 `long` 在不同平台的大小

> **背景**：早期16位系统中，`int` 为2字节，`long` 为4字节。随着系统演进，不同平台对 `long` 的定义出现了差异。

| 平台             | int (32位) | long (32/64位)         |
| ---------------- | ---------- | ---------------------- |
| **Windows 32位** | 4 字节     | 4 字节                 |
| **Windows 64位** | 4 字节     | 4 字节（兼容性考虑）   |
| **Linux 32位**   | 4 字节     | 4 字节                 |
| **Linux 64位**   | 4 字节     | **8 字节**（LP64模型） |

**实际影响**：
```cpp
// Linux 64位下：sizeof(int)=4, sizeof(long)=8
// Windows 64位下：sizeof(int)=4, sizeof(long)=4

// 如果需要跨平台固定大小，使用 <stdint.h>
#include <stdint.h>
int32_t   // 固定32位
int64_t   // 固定64位
```

---

## 五、完整综合示例

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

void comprehensive_demo() {
    printf("========== 1. ato* 系列 (无错误检测) ==========\n");
    printf("atoi(\"42abc\") = %d\n", atoi("42abc"));
    printf("atoi(\"abc\") = %d (⚠️ 无法检测错误)\n\n", atoi("abc"));
    
    printf("========== 2. strto* 系列 (有错误检测) ==========\n");
    char str[] = "  123.45rem";
    char* endptr;
    double val = strtod(str, &endptr);
    printf("strtod(\"%s\")\n", str);
    printf("  转换值: %f\n", val);
    printf("  剩余部分: \"%s\"\n", endptr);
    
    printf("\n========== 3. 进制转换演示 ==========\n");
    printf("strtol(\"0x1A\", NULL, 0) = %ld (自动检测十六进制)\n", 
           strtol("0x1A", NULL, 0));
    printf("strtol(\"0777\", NULL, 0) = %ld (自动检测八进制)\n", 
           strtol("0777", NULL, 0));
    printf("strtol(\"1010\", NULL, 2) = %ld (二进制)\n", 
           strtol("1010", NULL, 2));
    
    printf("\n========== 4. 数值转字符串 (现代方法) ==========\n");
    double pi = 3.1415926535;
    char buffer[64];
    snprintf(buffer, sizeof(buffer), "%.6f", pi);
    printf("snprintf: %s\n", buffer);
}

int main() {
    comprehensive_demo();
    return 0;
}
```

---

## 六、快速参考卡片

### 字符串 → 数值

| 功能需求           | 推荐函数               | 注意事项             |
| ------------------ | ---------------------- | -------------------- |
| 简单整数转换       | `atoi`                 | 无错误检测           |
| 需要错误检测的整数 | `strtol`               | 使用 endptr 和 errno |
| 需要进制转换       | `strtol` 或 `strtoul`  | base 参数指定进制    |
| 浮点数转换         | `strtod`               | 支持科学计数法       |
| 64位整数           | `strtoll` / `strtoull` | 跨平台固定大小       |

### 数值 → 字符串

| 功能需求     | 推荐函数         | 说明        |
| ------------ | ---------------- | ----------- |
| 通用格式化   | `snprintf`       | 标准C，安全 |
| 科学计数法   | `snprintf("%e")` | 替代 ecvt   |
| 固定小数位   | `snprintf("%f")` | 替代 fcvt   |
| 自动选择格式 | `snprintf("%g")` | 替代 gcvt   |

