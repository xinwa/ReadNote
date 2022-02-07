#### happens-before

在 JMM 中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必 须要存在 happens-before 关系。这里提到的两个操作既可以是在一个线程之内，也可以 是在不同线程之间。

与程序员密切相关的 happens-before 规则如下。

* 程序顺序规则:一个线程中的每个操作，happens-before 于该线程中的任意后续操作。

* 监视器锁规则:对一个锁的解锁，happens-before 于随后对这个锁的加锁。

*  volatile 变量规则:对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。

*  传递性:如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

重点：happens-before 仅仅要求前一个操作(执行的结果)对后一个操作可见，且 前一个操作按顺序排在第二个操作之前

#### DoubleCheckLocking 单例问题

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20211223000621683.png" alt="image-20211223000621683" style="zoom:50%;" />

在于 instance = new Instance() 这一行上面。

在实例化对象时，可分为三步

memory = allocate(); // 1:分配对象的内存空间

ctorInstance(memory);  // 2:初始化对象

 instance =memory；// 3:设置 instance 指向刚分配的内存地址

如果步骤2和步骤3发生指令重排序后，当线程1执行完了 instance =memory，但还没进行初始化，这时发生线程切换，线程 2 进入

getInstance 方法，判断 instrance 不为null，于是方法返回，但此时 instance 还未进行过初始化，所以会出问题，解决的方法是使用 volatile 修饰 instance 即可。

