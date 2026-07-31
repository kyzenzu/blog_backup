---
title: JCC跳转指令和状态寄存器
date: 2024-10-24
tags: 
  - 汇编
---

在x86汇编语言中，JCC指令（条件跳转指令）是一组基于条件标志寄存器（EFLAGS）中的状态来决定是否跳转到某个地址的指令。下面是所有JCC指令及其对应的条件标志和含义。

### 状态寄存器（EFLAGS）

EFLAGS寄存器中包含了一些标志位，这些标志位用于指示上一次操作的结果，并用于控制条件跳转指令。常见的标志位有：

- **CF (Carry Flag)**：进位标志。如果运算结果有进位或者借位，CF置1。
- **PF (Parity Flag)**：奇偶标志。如果结果的低8位中1的个数为偶数，PF置1。
- **AF (Auxiliary Carry Flag)**：辅助进位标志。用于BCD运算。
- **ZF (Zero Flag)**：零标志。如果运算结果为0，ZF置1。
- **SF (Sign Flag)**：符号标志。如果运算结果为负数，SF置1。
- **OF (Overflow Flag)**：溢出标志。如果有符号数运算溢出，OF置1。

### JCC指令详解

JCC指令通常有两个字节，第一个字节是0x0F，第二个字节表示具体的跳转条件。下面是所有的JCC指令及其对应的条件和标志位。

| 指令          | 条件                                 | 标志位         | 描述                  |
| ------------- | ------------------------------------ | -------------- | --------------------- |
| JO            | Overflow                             | OF=1           | 如果溢出              |
| JNO           | Not Overflow                         | OF=0           | 如果不溢出            |
| JB, JNAE, JC  | Below, Not Above or Equal, Carry     | CF=1           | 如果进位              |
| JAE, JNB, JNC | Above or Equal, Not Below, Not Carry | CF=0           | 如果不进位            |
| JE, JZ        | Equal, Zero                          | ZF=1           | 如果相等/结果为零     |
| JNE, JNZ      | Not Equal, Not Zero                  | ZF=0           | 如果不相等/结果不为零 |
| JBE, JNA      | Below or Equal, Not Above            | CF=1 or ZF=1   | 如果低于或等于        |
| JA, JNBE      | Above, Not Below or Equal            | CF=0 and ZF=0  | 如果高于              |
| JS            | Sign                                 | SF=1           | 如果结果为负          |
| JNS           | Not Sign                             | SF=0           | 如果结果为正          |
| JP, JPE       | Parity, Parity Even                  | PF=1           | 如果奇偶为偶数        |
| JNP, JPO      | Not Parity, Parity Odd               | PF=0           | 如果奇偶为奇数        |
| JL, JNGE      | Less, Not Greater or Equal           | SF≠OF          | 如果小于              |
| JGE, JNL      | Greater or Equal, Not Less           | SF=OF          | 如果大于或等于        |
| JLE, JNG      | Less or Equal, Not Greater           | ZF=1 or SF≠OF  | 如果小于或等于        |
| JG, JNLE      | Greater, Not Less or Equal           | ZF=0 and SF=OF | 如果大于              |

### 结合状态寄存器和JCC指令的表格

下表列出了JCC指令与EFLAGS寄存器标志位的关系：

| 指令          | 条件                                 | OF   | CF   | ZF   | SF   | PF   | 描述                  |
| ------------- | ------------------------------------ | ---- | ---- | ---- | ---- | ---- | --------------------- |
| JO            | Overflow                             | 1    |      |      |      |      | 如果溢出              |
| JNO           | Not Overflow                         | 0    |      |      |      |      | 如果不溢出            |
| JB, JNAE, JC  | Below, Not Above or Equal, Carry     |      | 1    |      |      |      | 如果进位              |
| JAE, JNB, JNC | Above or Equal, Not Below, Not Carry |      | 0    |      |      |      | 如果不进位            |
| JE, JZ        | Equal, Zero                          |      |      | 1    |      |      | 如果相等/结果为零     |
| JNE, JNZ      | Not Equal, Not Zero                  |      |      | 0    |      |      | 如果不相等/结果不为零 |
| JBE, JNA      | Below or Equal, Not Above            |      | 1    | 1    |      |      | 如果低于或等于        |
| JA, JNBE      | Above, Not Below or Equal            |      | 0    | 0    |      |      | 如果高于              |
| JS            | Sign                                 |      |      |      | 1    |      | 如果结果为负          |
| JNS           | Not Sign                             |      |      |      | 0    |      | 如果结果为正          |
| JP, JPE       | Parity, Parity Even                  |      |      |      |      | 1    | 如果奇偶为偶数        |
| JNP, JPO      | Not Parity, Parity Odd               |      |      |      |      | 0    | 如果奇偶为奇数        |
| JL, JNGE      | Less, Not Greater or Equal           | 0/1  |      |      | 0/1  |      | 如果小于              |
| JGE, JNL      | Greater or Equal, Not Less           | 0/1  |      |      | 0/1  |      | 如果大于或等于        |
| JLE, JNG      | Less or Equal, Not Greater           | 0/1  |      | 1    | 0/1  |      | 如果小于或等于        |
| JG, JNLE      | Greater, Not Less or Equal           | 0/1  |      | 0    | 0/1  |      | 如果大于              |

### 说明

- 当OF和SF相等时，表示无符号数比较结果的正负号。
- CF和ZF组合起来可以判断无符号数的大小关系。
- PF用于判断结果的奇偶性，主要用于一些特定的算法。

这样，通过结合EFLAGS寄存器的标志位和JCC指令，程序可以实现各种条件的跳转控制。

---

>补码是C语言层面将"-1"存入内存的方式。以什么方式将数字转化为机器码存储，将机器码解释为什么数字，是由C语言来定义的。计算机只处理交过来的机器码，以及交给C语言对应的机器码。
>
>CPU运算时管你有符号，无符号，也不管是否会溢出，只管对应位相加
>
>但会产生两套比较逻辑和溢出逻辑，一个有符号，一个无符号
>
>存储在PSW中
>
>由C语言自己选择JBE(无符号跳转)和JLE(有符号跳转)
>
>来区分有符号比较和无符号比较