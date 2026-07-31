---
title: 【汇编学习】C与汇编
date: 2024-3-18
tags:
  - C/C++
  - 汇编
---

编写好的**main**函数的地址总是在`CS:01FA`或者说`076A:01FA`处

~~~assembly
* (char *) 0x2000 = 'a';
mov byte ptr [2000], 61

* (int *) 0x2000 = 0xf;
mov word ptr [2000], 0000f

* (char far *) 0x20001000 = 'a';
mov bx, 2000
mov es, bx
mov bx, 1000
mov byte ptr es:[bx], 61

* (char *) _AX = 'b';
mov bx, ax
mov abyte ptr [bx], 62

_BX = 0x1000;
* (char *) (_BX + _BX) = 'a';
mov bx, 1000
add bx, bx
mov byte ptr [bx], 61

* (char far *) (0x20001000 + _BX) = * (char *)_AX;
mov bx, ax
mov al, [bx]
xor cx, cx
add bx, 1000
adc cx, 2000
mov es, cx
mov es:[bx], al
~~~



