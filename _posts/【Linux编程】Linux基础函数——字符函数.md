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

| 函数                 | 功能描述                         | 区别说明                               | 示例                                                         |
| -------------------- | -------------------------------- | -------------------------------------- | ------------------------------------------------------------ |
| `int isprint(int c)` | 测试字符是否为**可打印字符**     | **包含空格**，因为空格可以在输出时看见 | `isprint(' ')` → 真<br>`isprint('A')` → 真<br>`isprint('\t') `→ 假 |
| `int isgraph(int c)` | 测试字符是否为**可图形化字符**。 | **不包含空格**，因为空格不能图形化     | `isgraph(' ')` → 假<br>`isprint('\t')` → 假<br/>`isgraph('A')` → 真 |

**注意**：`isprint()` 的范围比 `isgraph()` 多一个空格字符。

### 5. 大小写字母相关

| 函数                 | 功能描述                         | 示例                |
| -------------------- | -------------------------------- | ------------------- |
| `int islower(int c)` | 测试字符是否为小写英文字母 (a-z) | `islower('m')` → 真 |
| `int isupper(int c)` | 测试字符是否为大写英文字母 (A-Z) | `isupper('K')` → 真 |

### 6. 其他

| 函数                 | 功能描述                                                     | 示例                                         |
| -------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| `int iscntrl(int c)` | 测试字符是否为ASCII控制字符 (0-31, 127)                      | `iscntrl('\n')` → 真<br>`iscntrl('\t')` → 真 |
| `int ispunct(int c)` | 测试字符是否为标点符号或特殊符号（非空格、非字母、非数字的可打印字符） | `ispunct('.')` → 真<br>`ispunct('!')` → 真   |

---

## 二、字符转换函数

| 函数                 | 功能描述                 | 示例                   |
| -------------------- | ------------------------ | ---------------------- |
| `int tolower(int c)` | 将大写字母转换为小写字母 | `tolower('A')` → `'a'` |
| `int toupper(int c)` | 将小写字母转换为大写字母 | `toupper('z')` → `'Z'` |

---

## 三、完整示例代码

```cpp
#include <iostream>
#include <ctype.h>
#include <string.h>
#include <stdlib.h>



int main()
{
	const char* str = "Hello World\r\n 2026-01-23\t\v";
	size_t len = strlen(str);
	for (size_t i = 0; i < len; i++) {
		if (str[i] == '\r') std::cout << "\\r: ";
		else if (str[i] == '\n') std::cout << "\\n: ";
		else if (str[i] == '\t') std::cout << "\\t: ";
		else if (str[i] == '\v') std::cout << "\\v: ";
		else std::cout << str[i] << ": ";
        
		if (isalnum(str[i])) std::cout << "Alnum ";
		if (isalpha(str[i])) std::cout << "Alpha ";
		if (isdigit(str[i])) std::cout << "Digit ";
		if (isxdigit(str[i])) std::cout << "XDigit ";
        
		if (isascii(str[i])) std::cout << "Ascii ";
        
		if (isblank(str[i])) std::cout << "Blank ";
		if (isspace(str[i])) std::cout << "Space ";
        
		if (isprint(str[i])) std::cout << "Print ";
		if (isgraph(str[i])) std::cout << "Graph ";
        
		if (iscntrl(str[i])) std::cout << "Cntrl ";
		if (ispunct(str[i])) std::cout << "Punct ";

		std::cout << std::endl;
	}

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
