#### 概述

获取 Service Manager是通过`defaultServiceManager()`方法来完成，当进程注册服务和获取服务的过程之前，都需要先调用 defaultServiceManager() 方法来获取`gDefaultServiceManager`对象。对于 gDefaultServiceManager对象，如果存在则直接返回；如果不存在则创建该对象，创建过程包括调用 open() 打开 binder 驱动设备，利用 mmap() 映射内核的地址空间。 

<img src="/Users/xinwa/Library/Application Support/typora-user-images/image-20220217230745607.png" alt="image-20220217230745607" style="zoom:67%;" />

[IServiceManager.cpp 地址](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IServiceManager.cpp;l=1?q=IServiceManager.cpp&sq=)

#### defaultServiceManager 方法

```c++
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    {
        AutoMutex _l(gDefaultServiceManagerLock); //加锁
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    return gDefaultServiceManager;
}
```

ProcessState::self() 用于获取 ProcessState 对象，每个进程有且仅有一个 ProcessState 对象，存在即返回

#####  ProcessState::self() 过程

[ProcessState.cpp](http://aospxref.com/android-7.1.2_r39/xref/frameworks/native/libs/binder/ProcessState.cpp)

```c++
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }

    //实例化ProcessState 【见小节2.2】
    gProcess = new ProcessState;
    return gProcess;
}

ProcessState::ProcessState()
    : mDriverFD(open_driver()) 
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // 采用内存映射函数 mmap，给 binder 分配一块虚拟地址空间,用来接收事务
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            close(mDriverFD); // 没有足够空间分配给/dev/binder,则关闭驱动
            mDriverFD = -1;
        }
    }
}
```

##### open_driver 过程

```c++
static int open_driver()
313  {
314      int fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
315      if (fd >= 0) {
316          int vers = 0;
317          status_t result = ioctl(fd, BINDER_VERSION, &vers);32
328          size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
329          result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
333      } else {
334          ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
335      }
336      return fd;
337  }
```

打开 binder 驱动，保存 df 文件描述符，然后设定 binder 支持的最大线程数。

##### getContextObject 过程

```c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);  
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    // 查找 handle 对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                // 通过 ping 操作测试 binder 是否准备就绪
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }
            // 当 handle 值所对应的 IBinder 不存在或弱引用无效时，则创建 BpBinder 对象
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}

BpBinder::BpBinder(int32_t handle)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK); // 延长对象的生命时间
    IPCThreadState::self()->incWeakHandle(handle); // handle所对应的bindle弱引用 + 1
}
```

getContextObject 过程是获取 handle 值为 0 的 上下文对象。内部会创建 BpBinder(0)

#### interface_cast 方法

```c++
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj); //【见小节4.2】
}
```

interface_cast 为模版函数，这里的 INTERFACE 为 IServiceManager 类型。

```c++
gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
```

上面的方法等价于 gDefaultServiceManager = IServiceManager::asInterface(BpBinder(0))

#####  IServiceManager::asInterface 过程

对于 asInterface() 函数，通过搜索代码，你会发现根本找不到这个方法是在哪里定义这个函数的, 其实是通过模板函数来定义的，通过下面两个代码完成的：

```c++
// 位于IServiceManager.h文件 
 class IServiceManager : public IInterface
{
	public:
		DECLARE_META_INTERFACE(ServiceManager);
 }

// 位于IServiceManager.cpp文件 
IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")
```

DECLARE_META_INTERFACE 与 IMPLEMENT_META_INTERFACE 为事先定义好的宏

###### DECLARE_META_INTERFACE  结构

```c++
#define DECLARE_META_INTERFACE(INTERFACE) \
    static const android::String16 descriptor;                          \
    static android::sp<I##INTERFACE> asInterface( \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    I##INTERFACE(); \
    virtual ~I##INTERFACE(); \
    
    
```

宏替换之后的内容如下

```c++
 class IServiceManager : public IInterface
{
	public:
		static const android::String16 descriptor;
		static android::sp< IServiceManager > asInterface(const android::sp<android::IBinder>& obj)
		virtual const android::String16& getInterfaceDescriptor() const;

		IServiceManager ();
		virtual ~IServiceManager();
 }

```

###### IMPLEMENT_META_INTERFACE 结构

```c++
IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME) \
    const android::String16 I##INTERFACE::descriptor(NAME); \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const { \
        return I##INTERFACE::descriptor; \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface( \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr; \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>( \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get()); \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj); \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { } \
    I##INTERFACE::~I##INTERFACE() { } \
```

宏替换后的内容为

```c++
const android::String16 IServiceManager::descriptor(“android.os.IServiceManager”);

const android::String16& IServiceManager::getInterfaceDescriptor() const
{
     return IServiceManager::descriptor;
}

 android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
{
       android::sp<IServiceManager> intr;
        if(obj != NULL) {
           intr = static_cast<IServiceManager *>(
               obj->queryLocalInterface(IServiceManager::descriptor).get());
           if (intr == NULL) {
               intr = new BpServiceManager(obj);  //【见小节4.5】
            }
        }
       return intr;
}

IServiceManager::IServiceManager () { }
IServiceManager::~ IServiceManager() { }
```

##### 最终过程

```c++
gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
// 等价于                
gDefaultServiceManager = IServiceManager::asInterface(BpBinder(0))
// 等价于
gDefaultServiceManager = new BpServiceManager(BpBinder(0))
```

##### BpServiceManager 实例化

```c++
BpServiceManager(const sp<IBinder>& impl): BpInterface<IServiceManager>(impl)
  
inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote): BpRefBase(remote)
  
BpRefBase::BpRefBase(const sp<IBinder>& o): mRemote(o.get()), mRefs(NULL), mState(0)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);

    if (mRemote) {
        mRemote->incStrong(this);
        mRefs = mRemote->createWeak(this);
    }
}
 
class BpRefBase : public virtual RefBase
{
		protected:
       BpRefBase(const sp<IBinder>& o);
	  	 inline  IBinder*        remote()                { return mRemote; }
       inline  IBinder*        remote() const          { return mRemote; }
 
    private:
			IBinder* const          mRemote;
};
```

`new BpServiceManager()`，在初始化过程中，比较重要工作的是类 BpRefBase 的 mRemote 指向 new BpBinder(0)，从而 BpServiceManager 能够利用Binder进行通过通信。

#### 总结

defaultServiceManager 等价于 new BpServiceManager(new BpBinder(0));

ProcessState::self()主要工作：

- 调用 open()，打开/dev/binder 驱动设备；
- 再利用 mmap()，创建大小为`1M-8K`的内存地址空间；
- 设定当前进程最大的最大并发Binder线程个数为`16`。

BpServiceManager 巧妙将通信层与业务层逻辑合为一体，

- 通过继承接口`IServiceManager`实现了接口中的业务逻辑函数；
- 通过成员变量`mRemote`= new BpBinder(0) 进行Binder通信工作。
- BpBinder 通过 handler 来指向所对应BBinder, 在整个Binder系统中`handle=0`代表 ServiceManager 所对应的BBinder