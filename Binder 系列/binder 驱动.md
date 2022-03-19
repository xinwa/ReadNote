#### 字符设备

Linux系统将设备分为三大类：字符设备、块设备和网络设备。字符设备是其中较为基础的一类，它的读写操作需要一个字节一个字节的进行，不能随机读取设备中的某一数据，即要按照先后顺序。

字符设备驱动所做的工作主要是添加、初始化、删除 cdev 结构体，申请、释放设备号，填充 file_operations 结构体中的功能函数，比如 open()、read()、write()、close() 等。当我们创建一个字符设备时，一般会在 /dev目录下生成一个设备文件，Linux用户层的程序就可以通过这个设备文件来操作这个字符设备。

在 Linux 内核中，使用 cdev 结构体来描述一个字符设备

```c
struct cdev {
        struct kobject kobj;  /* 内嵌的内核对象 */
        struct module *owner;  /* 模块所有者，一般为THIS OWNER */
        const struct file_operations *ops;  /* 文件操作结构体 */
        struct list_head list;  /* 把所有向内核注册的字符设备形成链表 */
        dev_t dev;  /* 设备号，由主设备号和次设备号构成 */
        unsigned int count;  /* 属于同主设备号的次设备号的个数 */
} __randomize_layout;
```

cdev 结构体的另一个重要成员 file_operations 定义了字符设备驱动提供给虚拟文件系统的接口函数。

##### file_operations 结构体

file_operations 结构体中的成员函数是字符设备驱动程序设计的主体内容，主要功能函数都在这里实现。用户层对 Linux系统的调用最终会调用到这些函数

```c
struct file_operations {
  　/* 模块拥有者，一般为THIS MODULE */
　　struct module *owner;　　
		/* 从设备中读取数据，成功时返回读取的字节数，出错返回负值 */
　　ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);　　
		/* 向设备发送数据，成功时该函数返回写入字节数。若未被实现，用户调层用 write() 时系统将返回 */
　　ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);　　　
    /* 将设备内存映射内核空间进程内存中，若未实现，用户层调用mmap()系统将返回 */
　　int (*mmap) (struct file *, struct vm_area_struct *);　　
   /* 提供设备相关控制命令（读写设备参数、状态，控制设备进行读写...）的实现，当调用成功时返回一个非负值 */
	 long (*unlocked_ioctl)(struct file *filp, unsigned int cmd, unsigned long arg);　　	
	 /* 打开设备 */
　　int (*open) (struct inode *, struct file *);　　
  　/* 关闭设备 */
　　int (*release) (struct inode *, struct file *);　　
　　/* 刷新设备 */
   int (*flush) (struct file *, fl_owner_t id);　　
　　...
};
```



需要注意的是，file_operations 中的 read() 和 write() 函数中的第二个参数内存地址指的是用户空间的地址，不能在内核中直接进行读写。这时候需要借助两个函数来完成，copy_from_user() 函数用来完成将数据从用户空间拷贝到内核，copy_to_user() 函数用来将数据从内核拷贝到用户空间。

整体结构框图如下：

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216005742704.png" alt="image-20220216005742704" style="zoom:50%;" />

##### 系统调用

用户态的程序调用 kernel 层驱动是需要陷入内核态的，进行系统调用（syscall），比如打开 Binder 驱动的方法的调用链为 open() -> __open() -> binder_open()。

open() 为用户空间的方法，__open()便是系统调用中相应的处理方法，通过查找，对应调用到内核binder驱动的binder_open() 方法.

#### Binder 驱动

##### 概述

Binder 驱动是 Android 专用的，底层的驱动架构与其他 Linux 驱动一样，以 misc 设备进行注册，作为虚拟字符设备，没有直接操作硬件，只是对设备内存的处理。

包含如下四个关键方法

* binder_init：设备的初始化，创建 /dev/binder 设备节点
* binder_open：获取 Binder Driver 的文件描述符
* binder_mmap: 在内核分配一块内存，用于存放数据
* binder_ioctl:将 IPC 的数据作为参数传递给 Binder Driver

[binder.c](https://android.googlesource.com/kernel/goldfish/+/refs/heads/android-goldfish-3.10/drivers/android/binder.c)

[binder.h](https://android.googlesource.com/kernel/common/+/experimental/android-4.4/include/uapi/linux/android/binder.h)

##### binder_init 方法

```c
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
  
static char *binder_devices_param = CONFIG_ANDROID_BINDER_DEVICES;

struct binder_device {
	struct hlist_node hlist;
	struct miscdevice miscdev;
	struct binder_context context;
};
  
static struct miscdevice binder_miscdev = {
    .minor = MISC_DYNAMIC_MINOR, //次设备号 动态分配
    .name = "binder",     //设备名
    .fops = &binder_fops  //设备的文件操作结构，这是file_operations结构
};

static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};

static int __init binder_init(void) {
  	...
    int ret = 0;
	  char *device_name, *device_names;
	  struct binder_device *device;
   	struct hlist_node *tmp;
    // 在 debugfs 文件系统中创建 binder 和 proc 目录
  	binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
  	if (binder_debugfs_dir_entry_root)
		binder_debugfs_dir_entry_proc = debugfs_create_dir("proc", binder_debugfs_dir_entry_root);
		device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
	
	  strcpy(device_names, binder_devices_param);
		while ((device_name = strsep(&device_names, ","))) {
		ret = init_binder_device(device_name);
		if (ret)
			goto err_init_binder_device_failed;
	}
}

static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;
	struct binder_context *context;
	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	if (!binder_device)
		return -ENOMEM;
	binder_device->miscdev.fops = &binder_fops;
	binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
	binder_device->miscdev.name = name;
	context = &binder_device->context;
	context->binder_context_mgr_uid = INVALID_UID;
	context->name = name;
	mutex_init(&context->binder_main_lock);
	mutex_init(&context->binder_deferred_lock);
	mutex_init(&context->binder_mmap_lock);
	context->binder_deferred_workqueue = create_singlethread_workqueue(name);
	if (!context->binder_deferred_workqueue) {
		ret = -ENOMEM;
		goto err_create_singlethread_workqueue_failed;
	}
	INIT_HLIST_HEAD(&context->binder_procs);
	INIT_HLIST_HEAD(&context->binder_dead_nodes);
	INIT_HLIST_HEAD(&context->binder_deferred_list);
	INIT_WORK(&context->deferred_work, binder_deferred_func);
	ret = misc_register(&binder_device->miscdev);
	if (ret < 0) {
		goto err_misc_register_failed;
	}
	hlist_add_head(&binder_device->hlist, &binder_devices);
	return ret;
  err_create_singlethread_workqueue_failed:
  err_misc_register_failed:
	free_binder_device(binder_device);
	return ret;
}

```

主要流程

在 debugfs 文件系统中创建 binder 和 proc 目录，拿到 binder device_name 为 binder，进行 binder 驱动的初始化。

注册 misc 设备，指定相应文件的操作方法。

```c
binder_device->miscdev.fops = &binder_fops;
binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
binder_device->miscdev.name = name;
```



##### binder_open 方法

```c
// 宏函数，这里面有 ptr, type, member 分别代表指针、类型、成员。   
// container_of() 的作用就是通过一个结构变量中一个成员的地址找到这个结构体变量的首地址。  
#define container_of(ptr, type, member) ({              \         
const typeof( ((type *)0)->member ) *__mptr = (ptr);    \         
(type *)( (char *)__mptr - offsetof(type,member) );})
  
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;
	struct binder_device *binder_dev;
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
	get_task_struct(current->group_leader);
	proc->tsk = current->group_leader;
	INIT_LIST_HEAD(&proc->todo);
	init_waitqueue_head(&proc->wait);
	proc->default_priority = task_nice(current);
	binder_dev = container_of(filp->private_data, struct binder_device, miscdev);
	proc->context = &binder_dev->context;
	binder_lock(proc->context, __func__);
	binder_stats_created(BINDER_STAT_PROC);
	hlist_add_head(&proc->proc_node, &proc->context->binder_procs);
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	filp->private_data = proc;
	binder_unlock(proc->context, __func__);
	...
	return 0;
}
```

创建 binder_proc 对象，并把当前进程等信息保存到 binder_proc 对象，该对象管理 IPC 所需的各种信息并拥有其他结构体的根结构体；再把 binder_proc 对象保存到文件指针filp，以及把 binder_proc 加入到全局链表`binder_procs`。

##### binder_mmap 方法

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area; // 内核虚拟空间
	struct binder_proc *proc = filp->private_data;
	const char *failure_string;
	struct binder_buffer *buffer;
	if (proc->tsk != current->group_leader)
		return -EINVAL;
	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;
	mutex_lock(&proc->context->binder_mmap_lock);
  // 分配内核虚拟空间
	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
  
	proc->buffer = area->addr;
  // 地址偏移量 = 用户虚拟地址空间 - 内核虚拟地址空间
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
	mutex_unlock(&proc->context->binder_mmap_lock);
  // 分配物理页的指针数组，数组大小为 vma 的等效 page 个数；
	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
	// 设置缓存大小
	proc->buffer_size = vma->vm_end - vma->vm_start;
	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;
  // 分配物理页面，同时映射到内核空间和进程空间，先分配1个物理页
	ret = binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma);
	buffer = proc->buffer;
	INIT_LIST_HEAD(&proc->buffers);
	list_add(&buffer->entry, &proc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(proc, buffer);
	proc->free_async_space = proc->buffer_size / 2;
	barrier();
	proc->files = get_files_struct(current);
	proc->vma = vma;
	proc->vma_vm_mm = vma->vm_mm;
	return ret;
}
```

主要流程

首先在内核虚拟地址空间，申请一块与用户虚拟内存相同大小的内存；然后再申请1个 page 大小的物理内存，再将同一块物理内存分别映射到内核虚拟地址空间和用户虚拟内存空间，从而实现了用户空间的 Buffer 和内核空间的 Buffer 同步操作的功能。

分配虚拟内核空间：area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);

记录虚拟用户空间与虚拟内核空间的偏移量：proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;

分配物理页的指针数组：proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);

分配物理页面，同时映射到内核空间和进程空间：binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma);

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216015639148.png" alt="image-20220216015639148" style="zoom:50%;" />

##### binder_ioctl 方法

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_context *context = proc->context;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
  // 进入休眠状态，直到中断唤醒
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

	binder_lock(context, __func__);
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}
	switch (cmd) {
	case BINDER_WRITE_READ:
    // 进行数据读写
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		break;
	case BINDER_SET_MAX_THREADS:
		if (copy_from_user_preempt_disabled(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
	case BINDER_SET_CONTEXT_MGR:
		ret = binder_ioctl_set_ctx_mgr(filp);
		break;
	...
	}
	return ret;
}

```

*ioctl(文件描述符，ioctl命令，数据类型)*

binder_ioctl() 函数负责在两个进程间收发 IPC 数据和 IPC reply 数据。

| ioctl命令                | 数据类型                 | 操作                    |
| :----------------------- | :----------------------- | :---------------------- |
| **BINDER_WRITE_READ**    | struct binder_write_read | 收发Binder IPC数据      |
| BINDER_SET_MAX_THREADS   | __u32                    | 设置Binder线程最大个数  |
| BINDER_SET_CONTEXT_MGR   | __s32                    | 设置Service Manager节点 |
| BINDER_THREAD_EXIT       | __s32                    | 释放Binder线程          |
| BINDER_VERSION           | struct binder_version    | 获取Binder版本信息      |
| BINDER_SET_IDLE_TIMEOUT  | __s64                    | 没有使用                |
| BINDER_SET_IDLE_PRIORITY | __s32                    | 没有使用                |

##### binder_get_thread

从 binder_proc 中查找 binder_thread, 如果当前线程已经加入到 proc 的线程队列则直接返回，如果不存在则构造 binder_thread，并将当前线程添加到当前的 proc 中。

此处创建 binder_thread 的目的, 是为了记录使用 binder 驱动的线程的具体信息。

```c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;
    struct rb_node **p = &proc->threads.rb_node;
    while (*p) {  // 根据当前进程的 pid，从 binder_proc 中查找相应的 binder_thread
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);
        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            break;
    }
    if (*p == NULL) {
        thread = kzalloc(sizeof(*thread), GFP_KERNEL); //新建binder_thread结构体
        if (thread == NULL)
            return NULL;
        binder_stats_created(BINDER_STAT_THREAD);
        thread->proc = proc;
        thread->pid = current->pid;  //保存当前进程(线程)的pid
        init_waitqueue_head(&thread->wait);
        INIT_LIST_HEAD(&thread->todo);
        rb_link_node(&thread->rb_node, parent, p);
        rb_insert_color(&thread->rb_node, &proc->threads);
        thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
        thread->return_error = BR_OK;
        thread->return_error2 = BR_OK;
    }
    return thread;
}
```

##### binder_ioctl_write_read

对于 ioctl() 方法中，传递进来的命令是 cmd = `BINDER_WRITE_READ`时执行该方法，arg 是一个`binder_write_read`结构体

```c
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) { //把用户空间数据ubuf拷贝到bwr
        ret = -EFAULT;
        goto out;
    }

    if (bwr.write_size > 0) {
        //当写缓存中有数据，则执行binder写操作
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) { //当写失败，再将bwr数据写回用户空间，并返回
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) {
        //当读缓存中有数据，则执行binder读操作
        ret = binder_thread_read(proc, thread,
                      bwr.read_buffer, bwr.read_size, &bwr.read_consumed,
                      filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait); //唤醒等待状态的线程
        if (ret < 0) { //当读失败，再将bwr数据写回用户空间，并返回
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }

    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) { //将内核数据bwr拷贝到用户空间ubuf
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216020542265.png" alt="image-20220216020542265" style="zoom:50%;" />

