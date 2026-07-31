---
title: 【汇编学习】PC机键盘的处理过程
date: 2024-3-16
tags:
  - 汇编
---

键盘输入的处理过程大致分为以下三个步骤：

1. 键盘输入
2. 引发9号中断
3. 执行int 9中断例程

### 键盘输入

键盘按下一个键时，键盘的芯片会产生对应的 **扫描码**，送入计算机与键盘的接口的寄存器中，这个接口在这次示例中地址为 60H
按键的松开也会产生一个 **扫描码**，相同的处理

扫描码根据被按下产生和松开产生分为 **通码** 和 **断码** ，同一个按键的通码的第8位为0，断码的第8位为1
一个扫描码的大小似乎为 1 个字节，也就是说同一按键，断码 = 通码 + 80H
例：g键的通码为 22H，断码为 A2H

### 引发中断

键盘的输入到达 60H端口之后，相关的芯片会向CPU发送**可屏蔽中断信号**，这个中断的类型码为 **9**

CPU在执行完一条指令后，如果 IF = 1，就会检测是否有中断信息，有，则响应中断，引发中断过程，转去执行int 9中断例程

### 执行中断例程

> 在了解之前，我们需要先知道输入的字符键值如何保存？
> 在BIOS中有一个键盘缓冲区，这是系统启动后，BIOS用于存放int 9 中断例程所接收的键盘输入的内存区。在这个缓冲区可以存储 15 个键盘输入，一个键盘输入用一个字单元存放，高位字节存放扫描码，低位字节存放 ASCII码。
> 与一般字符键不同的是，控制键和切换键并不存放在BIOS缓冲区。它们存储在专门的地址 **0040:17** 中，这个地址被称为**键盘状态字节**
>
> |   7    |    6     |    5    |     4      |  3   |  2   |    1    |    0    |
> | :----: | :------: | :-----: | :--------: | :--: | :--: | :-----: | :-----: |
> | Insert | CapsLock | NumLock | ScrollLock | Alt  | Ctrl | 左shift | 右shift |

CPU响应9号中断就会执行int 9中断例程：

1. 读出60H端口中的扫描码，根据扫描码类型做不同处理
2. 如果是字符键的扫描码，将该扫描码和它所对应的ASCII码送入内存中的BIOS键盘缓冲区
	如果是控制键（比如 Ctrl ）和切换键（比如CapsLock）的扫描码，则将其转变为状态字节，写入内存中存储状态字节的单元
3. 做一些扫除工作，中断返回

---

前面 2 个过程由硬件自动完成，我们能够操作就是在第3步，也就是根据自己的需求去定制中断的处理过程，处理键盘的输入

~~~assembly
assume cs:code, ds:data, ss:stack

stack segment
	db 128 dup (0)
stack ends

data segment
	dw 0, 0
data ends

code segment
start:	
	mov ax, stack
	mov ss, ax
	mov sp, 128

	mov ax, data
	mov ds, ax

	mov ax, 0
	mov es, ax

	push es:[9*4]
	pop ds:[0]
	push es:[9*4+2]
	pop ds:[2]

	mov word ptr es:[9*4], offset int9
	mov es:[9*4+2], cs

	mov ax, 0B800H
	mov es, ax
	mov ah, 'a'
 s: mov es:[160*12+40*2], ah
 	call delay
 	inc ah
 	cmp ah, 'z'
 	jna s

 	mov ax, 0
 	mov es, ax

 	push ds:[0]
 	pop es:[9*4]
 	push ds:[2]
 	pop es:[9*4+2]

 	mov ax, 4c00H
 	int 21h

delay:
	push ax
	push dx
	mov dx, 5H
	mov ax, 0
 s1:sub ax, 1
 	sbb dx, 0
 	cmp ax, 0
 	jne s1
 	cmp dx, 0
 	jne s1
	pop dx
	pop ax
	ret

int9:
	push ax
	push bx
	push es

	in al, 60H

	pushf
	pushf
	pop bx
	and bh, 11111100b
	push bx
	popf
	call dword ptr ds:[0]

	cmp al, 1
	jne int9ret

	mov ax, 0B800H
	mov es, ax
	inc byte ptr es:[160*12+40*2+1]

int9ret:
	pop es
	pop bx
	pop ax
	iret

code ends
end start
~~~

