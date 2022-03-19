##### binder 通信协议

Client 进程通过 RPC 与 Server 通信，可以简单地划分为三层，驱动层、IPC层、业务层。

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216234419934.png" alt="image-20220216234419934" style="zoom:50%;" />

##### binder 通信模型

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216234525482.png" alt="image-20220216234525482" style="zoom:50%;" />

Binder 协议包含在 IPC 数据中

1. `BINDER_COMMAND_PROTOCOL`：binder请求码，以 BC_ 开头，简称 BC 码，用于从 IPC 层传递到 Binder Driver 层；
2. `BINDER_RETURN_PROTOCOL` ：binder响应码，以 BR_ 开头，简称 BR 码，用于从 Binder Driver 层传递到 IPC 层；

Binder IPC 通信至少是两个进程的交互：

- client 进程执行 binder_thread_write，根据 BC_XXX 命令，生成相应的 binder_work；
- server 进程执行 binder_thread_read，根据 binder_work.type 类型，生成 BR_XXX，发送到用户空间处理。

##### binder 通信过程

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220216234809821.png" alt="image-20220216234809821" style="zoom:70%;" />

binder.work.type 类型

```c
BINDER_WORK_TRANSACTION //最常见类型
BINDER_WORK_TRANSACTION_COMPLETE
BINDER_WORK_NODE
BINDER_WORK_DEAD_BINDER
BINDER_WORK_DEAD_BINDER_AND_CLEAR
BINDER_WORK_CLEAR_DEATH_NOTIFICATION
```

##### binder_thread_write

请求处理过程是通过 binder_thread_write() 方法，该方法用于处理 binder 协议中的请求码。当 binder_buffer 存在数据，binder 线程的写操作循环执行

```c
binder_thread_write(){
    while (ptr < end && thread->return_error == BR_OK) {
        get_user(cmd, (uint32_t __user *)ptr)；// 获取IPC数据中的Binder协议(BC码)
        switch (cmd) {
            case BC_INCREFS: ...
            case BC_ACQUIRE: ...
            case BC_RELEASE: ...
            case BC_DECREFS: ...
            case BC_INCREFS_DONE: ...
            case BC_ACQUIRE_DONE: ...
            case BC_FREE_BUFFER: ... break;
            
            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;
                copy_from_user(&tr, ptr, sizeof(tr))； // 拷贝用户空间 tr 到内核
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;

            case BC_REGISTER_LOOPER: ...
            case BC_ENTER_LOOPER: ...
            case BC_EXIT_LOOPER: ...
            case BC_REQUEST_DEATH_NOTIFICATION: ...
            case BC_CLEAR_DEATH_NOTIFICATION:  ...
            case BC_DEAD_BINDER_DONE: ...
            }
        }
    }
}
```

对于请求码为`BC_TRANSACTION`或`BC_REPLY`时，会执行 binder_transaction() 方法，这是最为频繁的操作。 对于其他命令则不同。

###### binder_transaction

```c
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    // 根据各种判定，获取以下信息：
    struct binder_thread *target_thread； // 目标线程
    struct binder_proc *target_proc；    // 目标进程
    struct binder_node *target_node；    // 目标binder节点
    struct list_head *target_list；      // 目标TODO队列
    wait_queue_head_t *target_wait；     // 目标等待队列
      
    // 分配两个结构体内存
    struct binder_transaction *t = kzalloc(sizeof(*t), GFP_KERNEL);
    struct binder_work *tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
  
  	..
    if (reply) {
      // BR 过程
			in_reply_to = thread->transaction_stack;
			binder_set_nice(in_reply_to->saved_priority);
			thread->transaction_stack = in_reply_to->to_parent;
			target_thread = in_reply_to->from;
			target_proc = target_thread->proc;
	  } else {
      // BC 过程
			if (tr->target.handle) {
				struct binder_ref *ref;
        // 通过 target.handle 获取目标进程
				ref = binder_get_ref(proc, tr->target.handle, true);
				target_node = ref->node;
		} else {
			target_node = context->binder_context_mgr_node;
		}
			e->to_node = target_node->debug_id;
			target_proc = target_node->proc;
		}
  
    // 从 target_proc 分配一块 buffer
	  t->buffer = binder_alloc_buf(target_proc, tr->data_size,
		tr->offsets_size, extra_buffers_size,
		!reply && (t->flags & TF_ONE_WAY));

    for (; offp < off_end; offp++) {
        switch (fp->type) {
        case BINDER_TYPE_BINDER: ...
        case BINDER_TYPE_WEAK_BINDER: ...
        case BINDER_TYPE_HANDLE: ...
        case BINDER_TYPE_WEAK_HANDLE: ...
        case BINDER_TYPE_FD: ...
        }
    }
    // 向目标进程的 target_list 添加 BINDER_WORK_TRANSACTION 事务
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    // 向当前线程的 todo 队列添加 BINDER_WORK_TRANSACTION_COMPLETE 事务
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
```

###### BC_PROTOCOL

binder 请求码，是用`enum binder_driver_command_protocol`来定义的，是用于应用程序向 binder 驱动设备发送请求消息，应用程序包含 Client 端和 Server 端，以 BC_ 开头，总17条；(-代表目前不支持的请求码)

| 请求码                        | 参数类型                | 作用                           |
| :---------------------------- | :---------------------- | :----------------------------- |
| BC_TRANSACTION                | binder_transaction_data | Client向Binder驱动发送请求数据 |
| BC_REPLY                      | binder_transaction_data | Server向Binder驱动发送请求数据 |
| BC_FREE_BUFFER                | binder_uintptr_t(指针)  | 释放内存                       |
| BC_INCREFS                    | __u32(descriptor)       | binder_ref弱引用加1操作        |
| BC_DECREFS                    | __u32(descriptor)       | binder_ref弱引用减1操作        |
| BC_ACQUIRE                    | __u32(descriptor)       | binder_ref强引用加1操作        |
| BC_RELEASE                    | __u32(descriptor)       | binder_ref强引用减1操作        |
| BC_ACQUIRE_DONE               | binder_ptr_cookie       | binder_node强引用减1操作       |
| BC_INCREFS_DONE               | binder_ptr_cookie       | binder_node弱引用减1操作       |
| BC_REGISTER_LOOPER            | 无参数                  | 创建新的looper线程             |
| BC_ENTER_LOOPER               | 无参数                  | 应用线程进入looper             |
| BC_EXIT_LOOPER                | 无参数                  | 应用线程退出looper             |
| BC_REQUEST_DEATH_NOTIFICATION | binder_handle_cookie    | 注册死亡通知                   |
| BC_CLEAR_DEATH_NOTIFICATION   | binder_handle_cookie    | 取消注册的死亡通知             |
| BC_DEAD_BINDER_DONE           | binder_uintptr_t(指针)  | 已完成binder的死亡通知         |
| BC_ACQUIRE_RESULT             | -                       | -                              |
| BC_ATTEMPT_ACQUIRE            | -                       | -                              |

##### binder_thread_read

响应处理过程是通过`binder_thread_read()`方法，该方法根据不同的`binder_work->type`以及不同状态，生成相应的响应码。

```c
binder_thread_read（）{
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
        //当 &thread->todo和&proc->todo 都为空时，goto 到retry  标志处，否则往下执行：
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
done:
    *consumed = ptr - buffer;
    // 当满足请求线程加已准备线程数等于0，已启动线程数小于最大线程数(15)，
    // 且 looper 状态为已注册或已进入时创建新的线程。
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc->requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
```

当 transaction 堆栈为空，且线程 todo 链表为空，且 non_block=false 时，意味着没有任何事务需要处理的，会进入等待客户端请求的状态。当有事务需要处理时便会进入循环处理过程，并生成相应的响应码。在Binder驱动层，只有在进入binder_thread_read() 方法时，同时满足以下条件， 才会生成`BR_SPAWN_LOOPER`命令，当用户态进程收到该命令则会创建新线程：

######  BR_PROTOCOL

binder 响应码，是用`enum binder_driver_return_protocol`来定义的，是binder设备向应用程序回复的消息，，应用程序包含 Client 端和 Server 端，以 BR_开头，总18条；

| 响应码                           | 参数类型                | 作用                                        |
| :------------------------------- | :---------------------- | :------------------------------------------ |
| BR_ERROR                         | __s32                   | 操作发生错误                                |
| BR_OK                            | 无参数                  | 操作完成                                    |
| BR_NOOP                          | 无参数                  | 不做任何事                                  |
| BR_SPAWN_LOOPER                  | 无参数                  | 创建新的Looper线程                          |
| BR_TRANSACTION                   | binder_transaction_data | Binder驱动向Server端发送请求数据            |
| BR_REPLY                         | binder_transaction_data | Binder驱动向Client端发送回复数据            |
| BR_TRANSACTION_COMPLETE          | 无参数                  | 对请求发送的成功反馈                        |
| BR_DEAD_REPLY                    | 无参数                  | 回复失败，往往是线程或节点为空              |
| BR_FAILED_REPLY                  | 无参数                  | 回复失败，往往是transaction出错导致         |
| BR_INCREFS                       | binder_ptr_cookie       | binder_ref弱引用加1操作（Server端）         |
| BR_DECREFS                       | binder_ptr_cookie       | binder_ref弱引用减1操作（Server端）         |
| BR_ACQUIRE                       | binder_ptr_cookie       | binder_ref强引用加1操作（Server端）         |
| BR_RELEASE                       | binder_ptr_cookie       | binder_ref强引用减1操作（Server端）         |
| BR_DEAD_BINDER                   | binder_uintptr_t(指针)  | Binder驱动向client端发送死亡通知            |
| BR_CLEAR_DEATH_NOTIFICATION_DONE | binder_uintptr_t(指针)  | BC_CLEAR_DEATH_NOTIFICATION命令对应的响应码 |
| BR_ACQUIRE_RESULT                | -                       | -                                           |
| BR_ATTEMPT_ACQUIRE               | -                       | -                                           |
| BR_FINISHED                      | -                       | -                                           |



##### binder 数据转换图

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220217002730749.png" alt="image-20220217002730749" style="zoom:70%;" />

##### binder 内存转移图

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220217000830416.png" alt="image-20220217000830416" style="zoom:70%;" />

