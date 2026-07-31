---
title: pwn-heap malloc函数
date: 2024-1-4
tags:
  - CTF
  - PWN
---

那么当我们在程序中第一次申请了一块空间时到底发生了什么？

> ==提前科普==：
>
> 为了防止你会懵 B 所以对于有些下文会提到但是已经涉及到的东西讲解一下：
>
> 1. **Top chunk**: 当程序第一次进行 malloc 的时候， heap 会被分为两块，一块给用户，剩下的那块就是 Top chunk ，再次申请堆块要是没有合适的空间便会使用 Top chunk 的空间。
> 2. 你申请到的一块堆内存的起始地址你可以写入数据的起始地址，因为堆块头部会记录一些信息，所以你会看到下面的示例中有 `0x10 `大小差距。
> 3. 你申请的大小 ≠ 实际申请大小，他会有一个的取整的步骤。

 **malloc() 函数:**

分配所需的内存空间，并返回一个指向它的指针

~~~C
/*
  malloc(size_t n)
  Returns a pointer to a newly allocated chunk of at least n bytes, or null
  if no space is available. Additionally, on failure, errno is
  set to ENOMEM on ANSI C systems.
  If n is zero, malloc returns a minumum-sized chunk. (The minimum
  size is 16 bytes on most 32bit systems, and 24 or 32 bytes on 64bit
  systems.)  On most systems, size_t is an unsigned type, so calls
  with negative arguments are interpreted as requests for huge amounts
  of space, which will often fail. The maximum supported value of n
  differs across systems, but is in all cases less than the maximum
  representable value of a size_t.
*/
void* malloc(size_t size);
~~~

参数 size ：要分配的字节大小，是无符号整数，如果你输入了 `-1` ，理论上会有一个很大很大的堆，实际上会报错，因为确实很大！

返回值：如果分配成功，则返回指向堆数据可写位置的指针；如果分配失败，则返回`NULL`。

可以看出，malloc 函数返回对应大小字节的内存块的指针。此外，该函数还对一些异常情况进行了处理

- 当 size=0 时，返回当前系统允许的堆的最小内存块。
- 当 size 为负数时，由于在大多数系统上，**size_t 是无符号数（这一点非常重要）**，所以程序就会申请很大的内存空间，但通常来说都会失败，因为系统没有那么多的内存可以分配。

**free() 函数：**

释放之前调用 calloc 、 malloc 或 realloc 所分配的内存空间。

~~~C
void free(void* ptr);
~~~

参数 ptr ：指针指向一个要释放内存的内存块。如果传递的参数是一个空指针，则不会执行任何动作，**<u>注意 free() 不会清除内存块的数据</u>**。

无返回值。

- **当 p 为空指针时，函数不执行任何操作。**
- 当 p 已经被释放之后，再次释放会出现乱七八糟的效果，这其实就是 `double free`。
- 除了被禁用 (mallopt) 的情况下，当释放很大的内存空间时，程序会将这些内存空间还给系统，以便于减小程序所使用的内存空间。

> 提前科普：
>
> 一般使用 free() 函数释放的堆块不会立刻被回收，它们会变成一种叫Free Chunk 的东西并且加上了一种类似 xxx bin 的名字，一般这类堆块释放后如果挨着一个也被释放的堆块或者是 Top Chunk 会合并，当然请记住 Fast Bin 是一个特例一它不会轻易合并。

**calloc() 函数：**

分配所需的内存空间，并返回一个（一组）指向它（它们）的指针。**<u>malloc 和 calloc 之间的不同点是， malloc不会设置内存为零，而calloc 会设置分配的内存为零。</u>**

分配`nitems`个`size`大小的堆块

~~~C
void* calloc(size_t nitems, size_t size);
~~~

参数 nitems ：需要的堆块数量，也是无符号数。

返回值：如果分配成功，则返回一个（一组）指向分配的堆块（堆块们）的指针；如果分配失败，则返回 NULL。

**realloc()函数：**

更改已经配置的内存空间，即更改由 malloc() 函数分配的内存空间的大小。`realloc() = malloc() + free()`

~~~C
void* realloc(void* ptr, size_t size);
~~~

参数 *ptr ：一个指针，指向堆块或者为null

返回值：看情况，下文会说。

1. size > ptr→size，申请扩大堆块

	1. 当前内存段后面有空闲空间（指已经申请过但没有被利用的空间），则直接吞并部分空闲空间，返回原指针。
	2. 当前内存段后面没有空闲空间，或者空闲空间不够，那么就使用堆中第一个满足这一要求的内存块，将目前的数据复制到新的位置，并将原来的数据块释放掉，返回新的内存块位置相当于 `malloc(size) + memcpy(dst, ptr, size) + free(ptr)`。

2. size < ptr→size，申请缩小堆块。堆块会直接缩小，被削减的内存会释放。释放的内存块可能会被合并可能不会被合并。

	> 提示： 
	>
	> 这里的释放不是和 free()函数一样的释放，而是区别于free() 函数的释放内存方式，不过具体是什么以后再说吧。

3. `ptr == NULL` and `size != 0`，如果传入了一个空指针，但是 size 不为 0，那么`realloc()`只会申请内存，返回申请的内存地址，就相当于`malloc(size)`。

4. `isValid(ptr)` and `size == 0`，如果传入了一个正常的堆块地址，但是 size 为 0那么`realloc()`只会释放空间，就相当于`free(ptr)`。且释放的堆会变为`fastbins`，可能内存也不会清除，效果与`free()`一样。

5. 如果申请失败，将返回 NULL ，此时，原来的指针仍然有效。

