##### struct file

是字符设备驱动相关重要结构。

struct file 代表一个打开的文件描述符，它不是专门给驱动程序使用的，系统中每一个打开的文件在内核中都有一个关联的 struct file。 它由内核在 open时创建，并传递给在文件上操作的任何函数。

struct file 有个成员void *private_data 文档上说明该成员是系统调用时保存状态信息非常有用的资源

##### vm_area_struct

表示一块连续的虚拟内存区域，给用户空间使用，结构所描述的虚存空间以 vm_start、vm_end 成员表示，它们分别保存了该虚存空间的首地址和末地址后第一个字节的地址，以字节为单位，所以虚存空间范围可以用[vm_start, vm_end)表示。

##### vm_struct

表示一块连续的虚拟地址空间区域。给内核使用，地址空间范围是**(3G + 896M + 8M) ~ 4G**，对应的物理页面都可以是不连续的

##### 内核源码阅读工具

http://aospxref.com/android-8.1.0_r81/xref/frameworks/native/cmds/servicemanager/service_manager.c

https://android.googlesource.com/kernel/goldfish/+/refs/heads/android-goldfish-3.10/drivers/android/binder.c

https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;drc=master;l=99?q=ProcessState.cpp

