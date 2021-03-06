指令集告诉我们 CPU 可以执行什么指令，每种指令需要提供什么样的操纵数。

CPU 由多个（二极管、三极管）电路组成，每个电路可以表示 0 和 1，基于此来实现 与、或、非 逻辑。寄存器也是由多个电路组成，专门用于存储数据。

#### 编译器

编译器是将一个高级语言翻译成低级语言的程序

词法分析：将源代码中的「关键词」提取出来

语法树：编译器在扫描出各个 token 后根据规则将其用树的形式表示出来，这颗树就被称为语法树。

语义分析之后编译器遍历语法数转化为「中间代码」表示，生成中间代码之后对其进行优化，最终转换为机器指令。

#### 解释器

对于 C 和 C++而言，编译器会把C / C++ 程序直接翻译成 01 二进制机器指令。

在写 Python、Java、C# 程序时，需要安装运行时环境，这里的运行时环境，其实就是解释器。**解释器是一个可以运行程序的程序**

通常解释器其实使用 C / C++ 写出来的。然后用 C / C++ 写的程序专门用来执行你写的 Java 、Python 之类的程序。

这就是解释器语言效率不如编译语言效率高的原因。

C/C++程序最终被编译器编译成01机器指令，CPU 可以直接运行机器指令，而对于解释型语言来说 CPU 首先执行的是解释器程序，在由解释器程序在执行你写的程序。

#### CPU 是如何工作的

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20211128011143294.png" alt="image-20211128011143294" style="zoom:25%;" />

可执行程序保存在磁盘中，运行时需要拷贝到内存中，这被称为程序加载。然后操作系统告诉 CPU 机器指令在内存中的起始地址，然后 CPU 从该地址开始执行程序。

#### 寄存器

CPU 在执行过程中会产生临时数据，如果这些临时数据保存借助内存的话无疑会降低 CPU 执行指令的速度。为了加快指令的处理速度，CPU需要自己的「工作内存」。故寄存器可以简单的理解为读写速度飞快的内存，是制作在 CPU 中的高速存储介质，故寄存器造价特别昂贵。

#### 程序计数器

CPU 中有一个专门的寄存器来存放下一条指令的内存地址。当 CPU 执行完一个指令后，就会根据这个寄存器中的内存地址取出下一条指令，取出指令的同时会更新下一条指令的地址，地址 + 1 或者是设定为指定的地址（if和for就是跳转的指令地址）

所以执行程序的过程是：把程序从磁盘拷贝到内存，找到第一条指令的内存地址，然后设置给程序计数器，这样程序就可以开始运行啦。

#### CPU 的工作模式

CPU 可以执行的指令从等级上分为俩类，一类是「特权指令」这类指令多涉及硬件操作，另一类是普通指令。CPU 在执行指令时工作在俩种模式下："内核模式"以及"用户模式"。

CPU 在用户模式下只能执行部分指令集，比如用户模式下不能执行与 I/O有关的指令，不能访问整个的内存空间，不可以执行特权指令。

当执行用户程序时，CPU工作在用户模式，当 CPU 开始运行操纵系统时，CPU 工作在内核模式。

