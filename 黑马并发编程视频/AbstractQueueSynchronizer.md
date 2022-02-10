#### AbstractQueueSynchronizer 定义

是堵塞式锁和相关的同步器工具的框架

#### 特点

* 用 state 属性来表示资源的状态（分独占模式和共享模式）
  * 独占模式是只有一个线程能够访问资源，而共享模式可以允许多个线程访问资源
  * compareAndSetState 机制来设置 state 状态
  * getState - 获取 state 状态
  * setState - 设置 state 状态
* 提供了基于 FIFO 的等待队列，类似于 Monitor 的 EntryList
* 条件变量来实现等待、唤醒机制、支持多个条件变量、类似于 Monitor 的 WaitSet

子类需要实现的一些方法

* tryAcquire
* tryRelease
* tryAcquireShared
* tryReleaseShared
* isHeldExcllusively

##### 

#### ReentrantLock 

![image-20220209223934999](/Users/xinwa/Library/Application Support/typora-user-images/image-20220209223934999.png)

