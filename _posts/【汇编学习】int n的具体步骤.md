---
title: 【汇编学习】int n的具体步骤
date: 2024-3-16
tags:
  - 汇编
---

int n：

1. get n
2. pushf
3. TF = 0, IF = 0
4. push cs
5. push IP
6. (IP) = 4 * n, (CS) = 4 * n + 2

​	; 中断处理程序

7. pop IP
8. pop CS
9. popf



4,5,6可归为 call dword ptr ds:[0]

7,8,9可归为 iret

3的具体步骤为

~~~assembly
pushf
pop ax
and ah, 11111100b
push ax
popf
~~~

