---
title: Python的import
date: 2023-7-18
tags:
  - Python
---
~~~python
import my_module
~~~

1. 寻找my_module.py文件
2. 将my_module.py文件执行后放进缓存中，形成一个命名空间，并在主程序中生成一个变量命名为**my_module**，指向该命名空间
3. 然后就能通过命名空间的方式使用里面的变量、函数或者类

![import01](D:\代码笔记\images\import01.png)

![import00](D:\代码笔记\images\import00.png)



~~~python
from my_module import test1
~~~

1. 运行my_module.py文件
2. 从my_module.py文件里导入函数test1()，赋值给变量test1
3. 但是my_module的命名空间并不会生成，也就没有my_module变量

![import02](D:\代码笔记\images\import02.png)

![import03](D:\代码笔记\images\import03.png)



~~~python
import package
~~~

首先，package即包是一种特殊的模块，在进行导入时

1. 搜索是否有\_\_init\_\_.py文件
2. 如果有，将该文件运行后生成有关该文件的命名空间，然后以**package**命名一个module变量指向这个命名空间，这个命名空间中不会有任何该包下的模块信息，也就是module1和module2，如果涉及这两个，程序会报错，找不到这两个变量
3. 如果没有，那就啥事没有

![import04](D:\代码笔记\images\import04.png)

![import05](D:\代码笔记\images\import05.png)

![import06](D:\代码笔记\images\import06.png)



~~~python
import package.module1
~~~

1. 运行\_\_init\_\_.py和该包下的module1.py文件后生成2个命名空间
2. 由package做为主程序的变量指向\_\_init\_\_.py所生成的命名空间，而在init生成的命名空间中会多出一个module1变量，指向module1.py生成的命名空间

![import08](D:\代码笔记\images\import08.png)

![import07](D:\代码笔记\images\import07.png)



~~~python
from package import module1
~~~

1. 运行init.py文件和module1.py文件，但是只生成一个由module1.py组成的命名空间
2. 在主程序也只会有生成一个module1变量指向module1.py生成的命名空间
3. 并不会有package变量的命名空间

![import09](D:\代码笔记\images\import09.png)

![import10](D:\代码笔记\images\import10.png)



~~~python
from package.module1 import test
~~~

1. 运行init.py和module1.py文件后不生成命名空间
2. 将module1.py里的test()函数导入后生成test变量来指向这个函数

![import11](D:\代码笔记\images\import11.png)



总结：<u>导入包或者模块，执行文件后生成module变量指向命名空间</u>

<u>导入变量、函数、或者类执行文件后会直接生成对应的变量</u> 



关于as

~~~python
import module as m
~~~

就是将导入的变量换个名称，比如将module变量改名为m，指向的对象还是一样的

不过

~~~python
import package.module1 as m
~~~

会直接将m指向package.module1所指向的命名空间，并不会生成init.py的空间，也就不会有package变量