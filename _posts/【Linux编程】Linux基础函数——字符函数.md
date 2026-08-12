---
title: 【Linux编程】Linux基础函数——字符函数
date: 2026-08-11 20:33:20
tags:
  - C/C++
  - Linux编程
---

## 概述
`<ctype.h>` 头文件提供了一系列用于测试和映射字符的函数。这些函数接受一个 `int` 类型的参数（通常是一个字符的ASCII值），并返回一个非零值（真）或零（假）。

---

## 一、字符分类函数

### 1. 字母与数字相关

| 函数                  | 功能描述                                     | 示例                 |
| --------------------- | -------------------------------------------- | -------------------- |
| `int isalnum(int c)`  | 测试字符是否为英文字母或数字 (a-z, A-Z, 0-9) | `isalnum('A')` → 真  |
| `int isalpha(int c)`  | 测试字符是否为英文字母 (a-z, A-Z)            | `isalpha('Z')` → 真  |
| `int isdigit(int c)`  | 测试字符是否为阿拉伯数字 (0-9)               | `isdigit('5')` → 真  |
| `int isxdigit(int c)` | 测试字符是否为十六进制数字 (0-9, a-f, A-F)   | `isxdigit('F')` → 真 |

### 2. ASCII字符相关

| 函数                 | 功能描述                          | 示例               |
| -------------------- | --------------------------------- | ------------------ |
| `int isascii(int c)` | 测试字符是否为ASCII码字符 (0-127) | `isascii(65)` → 真 |

### 3. 空白字符相关（重点区分）

| 函数                 | 功能描述                   | 区别说明                                                     | 示例                                         |
| -------------------- | -------------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| `int isblank(int c)` | 测试字符是否为**空白字符** | 仅包含**空格**和**水平制表符** (`' '` 和 `'\t'`)             | `isblank(' ')` → 真<br>`isblank('\t')` → 真  |
| `int isspace(int c)` | 测试字符是否为**空格字符** | 包含更多：空格 `' '`、换页 `'\f'`、换行 `'\n'`、回车 `'\r'`、水平制表 `'\t'`、垂直制表 `'\v'` | `isspace('\n')` → 真<br>`isspace('\r')` → 真 |

**使用建议**：
- 需要严格匹配空格或Tab时，使用 `isblank()`
- 需要匹配所有类型的空白符（包括换行、回车等）时，使用 `isspace()`

### 4. 打印字符相关

| 函数                 | 功能描述                         | 区别说明                               | 示例                                       |
| -------------------- | -------------------------------- | -------------------------------------- | ------------------------------------------ |
| `int isprint(int c)` | 测试字符是否为**可打印字符**     | **包含空格**，因为空格可以在输出时看见 | `isprint(' ')` → 真<br>`isprint('A')` → 真 |
| `int isgraph(int c)` | 测试字符是否为**可图形化字符**。 | **不包含空格**，因为空格不能图形化     | `isgraph(' ')` → 假<br>`isgraph('A')` → 真 |

**注意**：`isprint()` 的范围比 `isgraph()` 多一个空格字符。

### 5. 大小写字母相关

| 函数                 | 功能描述                         | 示例                |
| -------------------- | -------------------------------- | ------------------- |
| `int islower(int c)` | 测试字符是否为小写英文字母 (a-z) | `islower('m')` → 真 |
| `int isupper(int c)` | 测试字符是否为大写英文字母 (A-Z) | `isupper('K')` → 真 |

### 6. 其他

| 函数                 | 功能描述                                                     | 示例                                       |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| `int iscntrl(int c)` | 测试字符是否为ASCII控制字符 (0-31, 127)                      | `iscntrl('\n')` → 真                       |
| `int ispunct(int c)` | 测试字符是否为标点符号或特殊符号（非空格、非字母、非数字的可打印字符） | `ispunct('.')` → 真<br>`ispunct('!')` → 真 |

---

## 二、字符转换函数

| 函数                 | 功能描述                 | 示例                   |
| -------------------- | ------------------------ | ---------------------- |
| `int tolower(int c)` | 将大写字母转换为小写字母 | `tolower('A')` → `'a'` |
| `int toupper(int c)` | 将小写字母转换为大写字母 | `toupper('z')` → `'Z'` |

---

## 三、完整示例代码

```cpp
#include <stdio.h>
#include <ctype.h>
#include <string.h>

// 演示字符分类函数的使用
void demonstrate_char_functions() {
    const char* str = "Hello World\r\n你好世界\r\n2026-01-01";
    int alnum_cnt = 0, alpha_cnt = 0, digit_cnt = 0;
    int blank_cnt = 0, space_cnt = 0;
    int print_cnt = 0, graph_cnt = 0;
    int upper_cnt = 0, lower_cnt = 0, punct_cnt = 0;
    
    size_t len = strlen(str);
    printf("分析字符串: \"%s\"\n", str);
    printf("字符串长度: %zu\n\n", len);
    
    for (size_t i = 0; i < len; i++) {
        unsigned char ch = str[i];  // 避免符号扩展问题
        
        // 字母数字分类
        if (isalnum(ch)) alnum_cnt++;
        if (isalpha(ch)) alpha_cnt++;
        if (isdigit(ch)) digit_cnt++;
        
        // 空白字符分类（重点区分）
        if (isblank(ch)) blank_cnt++;   // 仅空格和Tab
        if (isspace(ch)) space_cnt++;   // 所有空白字符
        
        // 可打印字符分类
        if (isprint(ch)) print_cnt++;   // 包含空格
        if (isgraph(ch)) graph_cnt++;   // 不包含空格
        
        // 大小写统计
        if (isupper(ch)) upper_cnt++;
        if (islower(ch)) lower_cnt++;
        
        // 标点符号
        if (ispunct(ch)) punct_cnt++;
    }
    
    printf("=== 统计结果 ===\n");
    printf("字母数字字符数: %d\n", alnum_cnt);
    printf("字母字符数: %d\n", alpha_cnt);
    printf("数字字符数: %d\n", digit_cnt);
    printf("空白字符数 (isblank): %d\n", blank_cnt);
    printf("空格字符数 (isspace): %d\n", space_cnt);
    printf("可打印字符数 (isprint): %d\n", print_cnt);
    printf("可打印字符数 (isgraph): %d\n", graph_cnt);
    printf("大写字母数: %d\n", upper_cnt);
    printf("小写字母数: %d\n", lower_cnt);
    printf("标点符号数: %d\n", punct_cnt);
}

// 演示字符转换函数
void demonstrate_convert_functions() {
    char text[] = "Hello, World! 2026";
    printf("\n转换前: %s\n", text);
    
    // 转换为大写
    for (int i = 0; text[i]; i++) {
        text[i] = toupper(text[i]);
    }
    printf("转换为大写: %s\n", text);
    
    // 转换为小写
    for (int i = 0; text[i]; i++) {
        text[i] = tolower(text[i]);
    }
    printf("转换为小写: %s\n", text);
}

// 演示 isblank 和 isspace 的区别
void demonstrate_blank_vs_space() {
    printf("\n=== isblank vs isspace 对比 ===\n");
    const char* test_chars = " \t\n\r\v\f";
    
    for (int i = 0; test_chars[i]; i++) {
        unsigned char ch = test_chars[i];
        printf("字符 '\\x%02X': isblank=%d, isspace=%d\n", 
               ch, isblank(ch), isspace(ch));
    }
}

int main() {
    demonstrate_char_functions();
    demonstrate_convert_functions();
    demonstrate_blank_vs_space();
    
    return 0;
}
```

---

## 四、使用注意事项

1. **参数类型**：这些函数接受 `int` 类型参数，但通常传入 `char` 类型。为了安全，应先将 `char` 转换为 `unsigned char`，避免符号扩展问题。

   ```cpp
   // 推荐做法
   isalnum((unsigned char)ch);
   ```

2. **返回值**：当条件为真时返回非零值，为假时返回0。不要假设返回值一定是1。

3. **区域设置**：某些函数（如 `isalpha`, `isalnum`）的行为可能受区域设置影响（如中文字符可能在某些区域设置下被识别为字母）。

4. **性能考虑**：这些函数通常实现为宏或内联函数，性能开销很小。
