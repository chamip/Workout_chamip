#### Android四大组件：Activity/Service/Provider/BroadcastReceiver

---

**Intent**

Intent是一个桥梁，完成四大组件之间的信息的传递，

同类或者不同类的组件无法直接传递对象，需要沟通就要利用Intent，同时通过静态变量或静态方法传递数据，容易造成数据异常、内存泄露等问题。

**生命周期**

生命周期是指组件的实例对象从创建到销毁可能会被系统调用的一些方法，每个方法的调用都有特定的条件，可以根据需要重写生命周期方法来达到在某些特定时刻执行特定任务的目的。生命周期方法不建议自行调用，应由系统管理。

**注册组件**

四大组件都需要通过项目中的`AndroidManifest.xml`文件进行静态注册后才可正常使用，其中`BroadcastReceiver`可以在其他组件中动态注册

**响应时间**

应用主线程未在规定的时间内执行完任务，系统会报ANR（应用程序无响应）错误，因此应将耗时任务交由子线程完成。

**Activity**

Activity是应用最直观的入口，一个应用可以没有其他组件，但是不能没有Activity。用户的IO操作都由Activity进行处理，应用的数据展示、吸引人的动画、优秀的界面设计等都需要Activity进行展示。

生命周期：

![生命周期](/Users/chamip/Desktop/生命周期.png)

![生命周期1](/Users/chamip/Desktop/生命周期1.png)

方法有：onCreate()/onStart()/onResume()/onPause()/onStop()/onResta rt()/onDestroy()。

**两个Activity跳转时的生命周期**

常见的场景就是两个Activity跳转，如A跳转到B，此时A和B的生命周期交织在一起，并不是简单的A执行完所有的生命周期再从头执行B的生命周期。

1. 当B无实例，且不是透明或半透明时：A.onPause() > B.onCreate() > B.onStart() > B.onResume() > A.onStop()。
   此时从B返回A：B.onPause() > A.onRestart() > A.onStart() > A.onResume() > B.onStop() > B.onDestroy()。
2. 当B无实例，且为透明或半透明时：A.onPause() > B.onCreate() > B.onStart() > B.onResume()。
   此时从B返回A：B.onPause() > A.onResume() > B.onStop() > B.onDestroy()。透明的B将导致A被B覆盖时不执行A.onStop()方法，B返回A时也不执行A.onRestart() > A.onStart()这两个方法。接下来的就不考虑透明的情况了，举一反三。
3. 当B有实例，B为SingleTop模式，且B位于栈顶（B不位于栈顶时相当于无实例，参考1、2两点）：此时A不在栈顶，其生命周期并不执行，B无生命周期变化，但系统会调用B.onNewIntent()方法告知B接收到了新的Intent。
4. 当B有实例，B为SingleTask模式：A.onPause() > B.onRestart() > B.onStart() > B.onNewIntent() > B.onResume()（onNewIntent()与onResume()无确定先后顺序，位置可能交换）> A.onStop() > 原本栈内位于B之上的Activity进入销毁流程。

---

**Service**

它的作用时承担大部分的数据处理方式，为其他组件提供繁重耗时的后台服务，可监控其他组件的运行状态。

**Service超时ANR/启动子线程**

通常出现ANR时，应用在主线程执行了较重的耗时任务，所以需要将耗时任务交给子线程进行。Service面临需要启动子线程的情况时，直接使用多线程也是可以的，但是如果Service必然是需要子线程的情况，建议使用IntentService完成任务。

IntentService是Service的子类，它和Service的不同点在于：

1. 内部有一个工作线程来完成耗时的操作，只需实现onHandleIntent方法即可。
2. 完成全部工作任务后，会自动终止服务。
3. 如果同时执行多个任务时，会以工作队列的方式依次执行。

ntentService非常适合用于处理应用中的耗时任务，当然如果需要并行执行还是考虑线程池，建议将应用中可串行执行的耗时任务都交由IntentService来完成。

---

**ContentProvider**

它的作用是，Content Provider使一个应用程序的指定数据集提供给其他应用程序。其他应用可以通过ContentResolver类从该内容提供者中获取或存入数据。开发人员通常不会使用ContentProvider类的对象，大多数是通过ContentResolver对象实现对ContentProvider的操作。

（1）android平台提供了ContentProvider使一个应用程序的指定数据集提供给其他应用程序。其他应用可以通过ContentResolver类从该内容提供者中获取或存入数据。

（2）只有需要在多个应用程序间共享数据是才需要内容提供者。例如，通讯录数据被多个应用程序使用，且必须存储在一个内容提供者中。它的好处是统一数据访问方式。

（3）ContentProvider实现数据共享。ContentProvider用于保存和获取数据，并使其对所有应用程序可见。这是不同应用程序间共享数据的唯一方式，因为android没有提供所有应用共同访问的公共存储区。

（4）开发人员不会直接使用ContentProvider类的对象，大多数是通过ContentResolver对象实现对ContentProvider的操作。

（5）ContentProvider使用URI来唯一标识其数据集，这里的URI以content://作为前缀，表示该数据由ContentProvider来管理。

---

**BroadcastReceiver**

它的作用是广播可以用于异步处理消息，此时作用类似于Handler，比Handler更强大的是广播可以在整个设备内互通消息，基于这个特性可以用来解决进程间通信的问题。广播的生命周期从调用开始到onReceive()执行完毕结束，需要注意的是，一般广播的生命周期都极短，需要在10秒内处理完onReceive()中的所有工作，所以，一般不进行耗时长的工作，如果有耗时长的工作，应当通过Intent传递给Service进行处理。

**两种注册形式**

广播接收者的注册有两种方法，分别是程序动态注册和AndroidManifest文件中进行静态注册。

动态注册广播接收器特点是当用来注册的Activity关掉后，广播也就失效了。静态注册无需担忧广播接收器是否被关闭，只要设备是开启状态，广播接收器也是打开着的。也就是说哪怕app本身未启动，该app订阅的广播在触发时也会对它起作用。

---

AMS是android中SystemServer进程中的一个线程，AMS是管理四大组件运行状态的系统服务线程。

![ams](/Users/chamip/Desktop/ams.png)

**SystemServer进程启动AMS**

SystemServer.java中的源码如下：

```java
private void startBootstrapServices() {
            // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
}
```

SystemServiceManager.startService(Class clazz）通过反射的技术的启动Service,关键源码如下:

```java
public <T extends SystemService> T startService(Class<T> serviceClass) {
       //...
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        //...        
        // Register it.
        mServices.add(service);

        // Start it.
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + name
                    + ": onStart threw an exception", ex);
        }
        return service;
    }
```

**AMS线程采用循环机制来处理统一进程内的请求**

ServiceThread是HandlerThread的子类，其run方法源码如下：

```java
public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

AMS线程也是采用Hander,Looper,MessageQueue的循环机制来处理客户的请求。

**四大组件都是由AMS管理的**

AMS的职责不仅仅是Activity，还包括Service、BroadCast Receiver、ContentProvider。 应用程序通过Binder机制与AMS通信，进而由AMS管理四大组件，包括启动、停止等。

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        //...
        public int startActivity(...)  {...}
        public final void activityStopped(...) {...}
        public ComponentName startService(...){...}
        public Intent registerReceiver(...) {...}
        public final ContentProviderHolder getContentProvider(...}
        //...

}
```

**ActivityStack管理当前系统所有Activity**

主要的几个变量，mTaskHistory记录着历史的activity，mLRUActivities记录的activity是根据LRU排序的，mPausingActivity切换activity时，正在暂停的activity。其源码如下:

```java
	 private ArrayList<TaskRecord> mTaskHistory = new ArrayList<TaskRecord>();

   final ArrayList<TaskGroup> mValidateAppTokens = new ArrayList<TaskGroup>();

   final ArrayList<ActivityRecord> mLRUActivities = new ArrayList<ActivityRecord>();

   ActivityRecord mPausingActivity = null;

   ActivityRecord mLastPausedActivity = null;

   ActivityRecord mLastNoHistoryActivity = null;

   ActivityRecord mResumedActivity = null;

   ActivityRecord mLastStartedActivity = null;
```

**ActivityServives管理当前系统所有的Service和状态**

```java
public final class ActiveServices {
    //...

    // How long we wait for a service to finish executing.
    static final int SERVICE_TIMEOUT = 20*1000;

    // How long we wait for a service to finish executing.
    static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
    //...

    final SparseArray<ServiceMap> mServiceMap = new SparseArray<ServiceMap>();

    /**
     * All currently bound service connections.  Keys are the IBinder of
     * the client's IServiceConnection.
     */
    final ArrayMap<IBinder, ArrayList<ConnectionRecord>> mServiceConnections
            = new ArrayMap<IBinder, ArrayList<ConnectionRecord>>();
/**
     * List of services that we have been asked to start,
     * but haven't yet been able to.  It is used to hold start requests
     * while waiting for their corresponding application thread to get
     * going.
     */
    final ArrayList<ServiceRecord> mPendingServices
            = new ArrayList<ServiceRecord>();

    /**
     * List of services that are scheduled to restart following a crash.
     */
    final ArrayList<ServiceRecord> mRestartingServices
            = new ArrayList<ServiceRecord>();

    /**
     * List of services that are in the process of being destroyed.
     */
    final ArrayList<ServiceRecord> mDestroyingServices
            = new ArrayList<ServiceRecord>();

    static final class DelayingProcess extends ArrayList<ServiceRecord> {
        long timeoout;
    }
```

**管理当前系统所有BroadCastReceiver和状态**

AMS中保存着系统BroadCastReceiver，管理着广播接收器的调度和运行的超时时间等，其关键源码如下：

```java
   // How long we allow a receiver to run before giving up on it.
    static final int BROADCAST_FG_TIMEOUT = 10*1000;
    static final int BROADCAST_BG_TIMEOUT = 60*1000;

    /**
     * Keeps track of all IIntentReceivers that have been registered for
     * broadcasts.  Hash keys are the receiver IBinder, hash value is
     * a ReceiverList.
     */
    final HashMap<IBinder, ReceiverList> mRegisteredReceivers =
            new HashMap<IBinder, ReceiverList>();

    BroadcastQueue mFgBroadcastQueue;
    BroadcastQueue mBgBroadcastQueue;
    // Convenient for easy iteration over the queues. Foreground is first
    // so that dispatch of foreground broadcasts gets precedence.
    final BroadcastQueue[] mBroadcastQueues = new BroadcastQueue[2];
```

