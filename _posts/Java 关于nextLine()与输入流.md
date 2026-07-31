---
title: Java 关于nextLine()与输入流
date: 2023-6-12
tags:
  - Java
---

#### Java 关于nextLine()与输入流

* in.next()、in.nextInt()等函数会读取输入流中==空格==和==回车键==前面的数据，并不会将这两个读进去

* 而in.nextLine()函数会读取整个输入流中的数据，包括==空格==和==回车键==，并在读取后将回车键剔除



如果在in.nextInt()函数后紧跟in.nextLine()函数：

1. 比如输入13

2. 那么，in.nextInt()函数读取了输入流中的13，留下回车键，到了in.nextLine()函数后，发现输入流中还有东西，那么in.nextLine()函数会读取这个回车键，然后就当读完了，读取后将回车键剔除，留下==空==。