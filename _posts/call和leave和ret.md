---
title: call和leave和ret
date: 2025-10-2
tags: 
  - 汇编
---
~~~assembly
call func 等价于
push eip
mov eip, func
----------
函数开头常用指令
push ebp
mov ebp, esp
sub esp, N   ; 为局部变量分配空间
----------
leave 等价于
mov esp, ebp
pop ebp
----------
ret 等价于
pop eip
~~~

