#### 概述

ServiceManager 是Binder IPC 通信过程中的守护进程，本身也是一个Binder服务，但并没有采用 libbinder 中的多线程模型来与 Binder 驱动通信，而是自行编写了 binder.c 直接和 Binder 驱动来通信，并且只有一个循环 binder_loop 来进行读取和处理事务，这样的好处是简单而高效。 

ServiceManager 本身工作相对简单，其功能：查询和注册服务，流程图如下

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220217004631424.png" alt="image-20220217004631424" style="zoom:67%;" />

#### 启动过程

ServiceManager 是由[init进程](http://gityuan.com/2016/02/05/android-init/)通过解析 init.rc 文件而创建的，其所对应的可执行程序 /system/bin/servicemanager，所对应的源文件是 service_manager.c，进程名为 /system/bin/servicemanager。

[service_manager.c](https://android.googlesource.com/platform/frameworks/base/+/6215d3ff4b5dfa52a5d8b9a42e343051f31066a5/cmds/servicemanager/service_manager.c)

#### main 方法

```c
int main(int argc, char **argv)
{
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;
    bs = binder_open(128*1024);
    if (binder_become_context_manager(bs)) {
        LOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }
    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

#### binder_open 方法

```c
struct binder_state
{
    int fd; // dev/binder的文件描述符
    void *mapped; // 指向mmap的内存地址
    size_t mapsize; // 分配的内存大小，默认为 128KB
};

// mapsize = 128 * 1024
struct binder_state *binder_open(unsigned mapsize)
{
    struct binder_state *bs;
    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return 0;
    }
  	// 通过系统调用陷入内核打开 binder 驱动
    bs->fd = open("/dev/binder", O_RDWR);

    bs->mapsize = mapsize;
    // 通过系统调用，mmap 内存映射，mmap必须是 page 的整数倍
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
        /* TODO: check version */
    return bs;
}
```

##### binder_open 的过程

调用 open 方法会通过系统调用，执行到 binder 驱动层对应的的 binder_open() 方法，驱动层会创建一个 binder_proc 对象代表当前的进程，并且 binder_proc 地址存储到 fd-> private_data 中，同时放入全局链表 binder_procs。

##### binder_mmap 的过程

mmap 方法经过系统调用会执行 binder_mmap 方法中，在 binder 驱动层中进行「内核虚拟地址空间」和「用户虚拟地址空间」的创建，然后分配物理内存，将同一块物理内存分别映射到「内核虚拟地址空间」和「用户虚拟内存空间」。创建

Binder_Buffer 对象，放入到当前进程的 binder_proc 中的 proc->buffers 链表。

#### binder_become_context_manager 方法

```c
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    binder_lock(__func__);
    switch (cmd) {
      case BINDER_SET_CONTEXT_MGR:
          ret = binder_ioctl_set_ctx_mgr(filp);
          break;
      }
      case :...
    }
    binder_unlock(__func__);
}
```

该方法目的是让当前进行成为上下文的管理者。经过系统调用执行到 binder_ioctl 方法

```c
// service manager所对应的binder_node;
static struct binder_node *binder_context_mgr_node;
// 运行service manager的线程uid
static kuid_t binder_context_mgr_uid = INVALID_UID;

static int binder_ioctl_set_ctx_mgr(struct file *filp)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	struct binder_context *context = proc->context;
	kuid_t curr_euid = current_euid();
	
	ret = security_binder_set_context_mgr(proc->tsk);
	if (uid_valid(context->binder_context_mgr_uid)) {
			...
		}
	} else {
   // 设置当前线程 euid 作为 Service Manager的uid
		context->binder_context_mgr_uid = curr_euid;
	}
	context->binder_context_mgr_node = binder_new_node(proc, 0, 0);
	context->binder_context_mgr_node->local_weak_refs++;
	context->binder_context_mgr_node->local_strong_refs++;
	context->binder_context_mgr_node->has_strong_ref = 1;
	context->binder_context_mgr_node->has_weak_ref = 1;
	return ret;
}
```

binder_ioctl_set_ctx_mgr 执行过程中，通过 binder_new_node 方法创建 binder_node 并保存到 binder_context_mgr_node 静态变量中。

##### binder_new_node

```c
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       binder_uintptr_t ptr,
                       binder_uintptr_t cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;
    // 首次进来为空
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }

    // 给新创建的 binder_node 分配内核空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
    // 将新创建的 node 对象添加到 proc 红黑树；
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
    node->work.type = BINDER_WORK_NODE; //设置binder_work的type
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    return node;
}
```

ServiceManager 创建的 binder_node 结点时，传入的 prt 与 cookie 都为 0。并且 binder_node 加入到 binder_proc 的红黑树中了。

#### binder_loop 方法

```c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    unsigned readbuf[32];
    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(unsigned));
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (unsigned) readbuf;    
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
    }
}
```

binder_loop 方法构造 binder_write_read 结构，先执行 binder_write 方法，然后进入循环，通过 BINDER_WRITE_READ 命令，执行 ioctl 传递数据给驱动层之后，binder_parse 进行解析，然后继续循环。下面先看下 binder_write 过程

##### binder_write 过程

```c#
int binder_write(struct binder_state *bs, void *data, size_t len) {
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data; // 此处 data 为 BC_ENTER_LOOPER
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    return res;
}

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

`binder_write`通过 ioctl() 将BC_ENTER_LOOPER 命令发送给 binder 驱动，此时 bwr 只有 write_buffer 有数据，进入

binder_thread_write 方法。

##### binder_ioctl 过程

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    ...

    binder_lock(__func__);
    thread = binder_get_thread(proc); // 获取binder_thread
    switch (cmd) {
      case BINDER_WRITE_READ:  // 进行 binder 的读写操作
          ret = binder_ioctl_write_read(filp, cmd, arg, thread);
          if (ret)
              goto err;
          break;
      case ...
    }
    ret = 0;
    return ret;
}
```

获取当前进程和线程，进行 binder 的读写操作

##### binder_ioctl_write_read 过程

```c
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) { // 把用户空间数据 ubuf 拷贝到 bwr
        ret = -EFAULT;
        goto out;
    }

    if (bwr.write_size > 0) { // 此时写缓存有数据
        ret = binder_thread_write(proc, thread,
                  bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
        ...
    }

    if (bwr.read_size > 0) { // 此时读缓存无数据
        ...
    }

    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) { //将内核数据bwr拷贝到用户空间ubuf
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

用户层的数据，不能直接使用，需要通过 copy_from_user 方法拷贝到内核中 bwr 结构中。

##### binder_thread_write 过程

```c
static int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread, binder_uintptr_t binder_buffer, size_t size, binder_size_t *consumed) {
  uint32_t cmd;
  void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;
  
  while (ptr < end && thread->return_error == BR_OK) {
    get_user(cmd, (uint32_t __user *)ptr); // 获取命令
    switch (cmd) {
      case BC_ENTER_LOOPER:
          // 设置该线程的looper状态
          thread->looper |= BINDER_LOOPER_STATE_ENTERED;
          break;
      case ...;
    }
  }
}
```

拿到 bwr 中用户空间设置的「数据地址」，然后通过 get_user 方法获取具体的内容。解析内容为BC_ENTER_LOOPER，

接着更新  thread->looper 为 BINDER_LOOPER_STATE_ENTERED;

##### binder_write_read 数据传递说明

首先在用户空间时，进行 binder_write_read 结构更新，以写操作为例，将需要「传递数据的地址」 data 覆值给  bwr.write_buffer 中，然后进行 ioctl 进行数据传递。

```c
ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

通过上面的方法，我们会把 bwr 的地址传递给 binder 驱动层。驱动层拿到之后会进行数据的拷贝

```c
struct binder_write_read bwr;
copy_from_user(&bwr, ubuf, sizeof(bwr)
```

拷贝完成之后，通过 bwr.write_size > 0 判断，得到有写数据，于是开始 binder_thread_write 的流程

```c
  void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;
	get_user(cmd, (uint32_t __user *)ptr);
```

上面的 binder_buffer 为用户空间需要「传递的数据的地址」。通过 get_user 方法拿到用户空间的数据给 cmd。然后就是

更新 thread 的 looper 状态了。

##### binder_loop 中循环说明

```c
bwr.read_size = sizeof(readbuf);
bwr.read_consumed = 0;
bwr.read_buffer = (unsigned) readbuf;    
res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
```

进行循环中后，就去 binder 驱动层中去读取数据了

```c
static int binder_thread_read(
  	struct binder_proc *proc, struct binder_thread *thread,
		binder_uintptr_t binder_buffer, size_t size, 
		binder_size_t *consumed, int non_block) 
{
    retry:
    wait_for_proc_work = thread->transaction_stack == NULL &&
            list_empty(&thread->todo);
    // 根据 wait_for_proc_work 来决定 wait 在当前线程还是进程的等待队列
    if (wait_for_proc_work) {
        ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
        ...
    } else {
        ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
        ...
    }
    
   	 while (1) {
        // 当 &thread->todo 和 &proc->todo 都为空时，goto 到 retry 标志处，否则往下执行：
      if (!list_empty(&thread->todo)) {
			   w = list_first_entry(&thread->todo, struct binder_work,entry);
		  } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
			   w = list_first_entry(&proc->todo, struct binder_work, entry);
		   } else {
			   /* no data added */
			   if (ptr - buffer == 4 &&
			      !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
				   goto retry;
			   break;
		}

   struct binder_transaction_data tr;
   struct binder_transaction *t = NULL;
	 switch (w->type) {
        case BINDER_WORK_TRANSACTION: ...
        case BINDER_WORK_TRANSACTION_COMPLETE: ...
        case BINDER_WORK_NODE: ...
        case BINDER_WORK_DEAD_BINDER: ...
        case BINDER_WORK_DEAD_BINDER_AND_CLEAR: ...
        case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
    }
        ...
}
```

当 &thread->todo 和 &proc->todo 都为空时，意味着没有任何事务需要处理的，goto 到 retry 标志处，然后会进入等待客户端请求的状态。根据 wait_for_proc_work 来决定 wait 在当前线程还是进程的等待队列

#### binder_parse

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        switch(cmd) {
        case BR_NOOP:  //无操作，退出循环
            break;
        case BR_TRANSACTION_COMPLETE:
            break;
        case BR_INCREFS:
        case BR_ACQUIRE:
        case BR_RELEASE:
        case BR_DECREFS:
            ptr += sizeof(struct binder_ptr_cookie);
            break;
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            ...
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg; 
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn); // 从 txn 解析出 binder_io 信息

                res = func(bs, txn, &msg, &reply);

                binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
            }
            ptr += sizeof(*txn);
            break;
        }
        case BR_REPLY: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            ...
            binder_dump_txn(txn);
            if (bio) {
                bio_init_from_txn(bio, txn);
                bio = 0;
            }
            ptr += sizeof(*txn);
            r = 0;
            break;
        }
        case BR_DEAD_BINDER: {
            struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
            ptr += sizeof(binder_uintptr_t);
            // binder死亡消息
            death->func(bs, death->ptr);
            break;
        }
        case BR_FAILED_REPLY:
            r = -1;
            break;
        case BR_DEAD_REPLY:
            r = -1;
            break;
        default:
            return -1;
        }
    }
    return r;
}
```

##### svcmgr_handler 过程

```c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si; 
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    ...
    
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    ...

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE: 
        s = bio_get_string16(msg, &len); // 服务名
        handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);
        //【见小节3.1.2】
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE: 
        s = bio_get_string16(msg, &len); // 服务名
        handle = bio_get_ref(msg); // handle
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
         // 注册指定服务 
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;

    case SVC_MGR_LIST_SERVICES: {  
        uint32_t n = bio_get_uint32(msg);

        if (!svc_can_list(txn->sender_pid)) {
            return -1;
        }
        si = svclist;
        while ((n-- > 0) && si)
            si = si->next;
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}

struct svcinfo
{
    struct svcinfo *next;
    uint32_t handle; // 服务的 handle 值
    struct binder_death death;
    int allow_isolated;
    size_t len; // 名字长度
    uint16_t name[0]; // 服务名
};
```



