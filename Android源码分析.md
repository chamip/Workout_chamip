### Android源码分析

---

![](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.comandroid架构.jpg)

Application层就是一个应用程序；Framework提供一个Java的运行环境以及对功能实现的封装；Runtime/ART是一个java虚拟机，因为Android上层是Java，需要再编译一次成为更低级的语言；Libraries层是功能实现区，有很多函数库，包括多媒体编解码/浏览器渲染/数据库实现等；Kernel层负责和硬件接触，通过它来操作硬件。

#### 阅读源码的思路

1. **Android源码分为功能实现上的纵向，以及功能拓展上的横向。**比如研究音频系统的实现原理，*纵向*：需要从一个音乐的开始播放追踪，然后发现解码库的调用，共享内存的创建和使用，路由的切换，音频输入设备的开启，音频流的开始。比如研究音频系统包括哪些内容，*横向*：通过Framework的接口，发现音频系统主要包括放音，录音，路由切换，音效处理等。
2. **Android的功能模块绝大部分是C/S架构**。Server是需要攻破的地方，**Libraries是精髓**，理解它之后才能向HAL和Kernel学习。
3. **Android的底层是Linux Kernel**。需要熟悉Kernel的基础协议，HAL是对kernel层的封装，方便各硬件接口统一，换硬件不用到Libraries里面改代码，在HAL里改就行。

---

#### 源码阅读 2021.08.18

ServiceManager：是整个应用服务的管理程序。

MediaService/MediaPlayerService：注册提供媒体播放的服务程序。

MediaPlayerClient：与MediaPlayerService交互的客户端程序。



main_mediaserver.cpp

```cpp
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    //获得一个ProcessState实例
    sp<ProcessState> proc(ProcessState::self());
    //获得一个ServiceManager对象
    sp<IServiceManager> sm(defaultServiceManager());
    ALOGI("ServiceManager: %p", sm.get());
    AIcu_initializeIcuOrDie();
    //初始化MediaPlayService服务
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    registerExtensions();
    //启动Process的线程池
    ProcessState::self()->startThreadPool();
    //将自己加入线程池
    IPCThreadState::self()->joinThreadPool();
}
```

**ProcessState**

*location*:![image-20210818152751804](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.comimage-20210818152751804.png)

```cpp
    sp<ProcessState> proc(ProcessState::self());
```

调用ProcessState::self()函数，赋值给proc变量，程序结束后，proc自动delete内部内容，释放资源。

ProcessState::self()实现：

```cpp
sp<ProcessState> ProcessState::self()
{
    //锁保护
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    //创建一个ProcessState对象
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}
```

ProcessState构造函数：

```cpp
//Process::self的作用：1. 打开/dev/binder设备，相当于和内核binder机制有了交互的通道
//                                            2. 映射fd到内存，设备的fd传进去后，其内存和binder设备共享
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    //映射内存的起始地址
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
        //c++中特殊的赋值方法，不用调用this
{
    if (mDriverFD >= 0) {
        //将fd映射为内存，所以内存的memcpy等操作相当于write/read(fd)
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver '%s' could not be opened.  Terminating.", driver);
}
```

ProcessState::self()功能：

1. 打开/dev/binder设备，相当于和内核binder机制有了交互的通道。
2. 映射fd到内存，设备的fd传进去，这块内存和binder设备共享。

**defaultServiceManager**

*location*:![image-20210818152136483](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.comimage-20210818152136483.png)

```cpp
sp<IServiceManager> defaultServiceManager()
{
    static Mutex gDefaultServiceManagerLock;
    static sp<IServiceManager> gDefaultServiceManager;

    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;
	//singleton
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == nullptr) {
            //真正的gDefaultServiceManager是在这里创建的
            //实质：gDefaultServiceManager = interface_cast<IServicerManager>(new BpBinder(0));
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(nullptr));
            if (gDefaultServiceManager == nullptr)
                sleep(1);
        }
    }

    return gDefaultServiceManager;
}
```

```cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}
```

```cpp
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    //从数组中查找对应索引的资源
    handle_entry* e = lookupHandleLocked(handle);
/*
            struct handle_entry {
                IBinder* binder;
                RefBase::weakref_type* refs;
            };
*/
    if (e != nullptr) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  The
        // attemptIncWeak() is safe because we know the BpBinder destructor will always
        // call expungeHandle(), which acquires the same lock we are holding now.
        // We need to do this because there is a race condition between someone
        // releasing a reference on this BpBinder, and a new reference on its handle
        // arriving from the driver.
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.
                //
                // TODO: add a driver API to wait for context manager, or
                // stop special casing handle 0 for context manager and add
                // a driver API to get a handle to the context manager with
                // proper reference counting.

                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
                if (status == DEAD_OBJECT)
                   return nullptr;
            }
            //创建新的BpBinder
            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    //返回刚才创建的BpBinder
    return result;
}

```

调用的结果就是

```cpp
gDefaultServiceManager = interface_cast<IServiceManager>(new BpBinder(0));
```

**BpBinder**

*location*:![image-20210818153102129](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.comimage-20210818153102129.png)

```cpp

BpBinder::BpBinder(int32_t handle, int32_t trackedUid)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(nullptr)
    , mTrackedUid(trackedUid)
{
    ALOGV("Creating BpBinder %p handle %d\n", this, mHandle);

    extendObjectLifetime(OBJECT_LIFETIME_WEAK);
    IPCThreadState::self()->incWeakHandle(handle, this);
}
```

singleton:**单例模式**，这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

1. 只能有一个实例
2. 必须自己创建自己的唯一实例
3. 必须给其他对象提供这一实例

ServiceManager存在的意义：

Android系统中的Service信息先add到ServiceManager中，由ServiceManager集中管理，如果某个服务如MediaPlayerService的客户端要和MediaPlayerService通讯，先在Manager查询，然后返回数据进行交互。





---

*note*:linux截图：shift+ctrl+prtSc