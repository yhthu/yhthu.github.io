title: Android线程管理（二）——ActivityThread
tags:
- Android线程
categories: Android线程管理

---

线程通信、ActivityThread及Thread类是理解Android线程管理的关键。

线程，作为CPU调度资源的基本单位，在Android等针对嵌入式设备的操作系统中，有着非常重要和基础的作用。本小节主要从以下三个方面进行分析：

 1. 《Android线程管理（一）——线程通信》
 2. 《Android线程管理（二）——ActivityThread》
 3. 《Android线程管理（三）——Thread类的内部原理、休眠及唤醒》

----------
## 二、ActivityThread的主要工作及实现机制

ActivityThread是Android应用的主线程（UI线程），说起ActivityThread，不得不提到Activity的创建、启动过程以及ActivityManagerService，但本文将仅从线程管理的角度来分析ActivityThread。ActivityManagerService、ActivityStack、ApplicationThread等会在后续文章中详细分析，敬请期待喔~~不过为了说清楚ActivityThread的由来，还是需要简单介绍下。

以下引用自罗升阳大师的博客：《Android应用程序的Activity启动过程简要介绍和学习计划》

> Step 1. 无论是通过Launcher来启动Activity，还是通过Activity内部调用startActivity接口来启动新的Activity，都通过Binder进程间通信进入到ActivityManagerService进程中，并且调用ActivityManagerService.startActivity接口； 
Step 2. ActivityManagerService调用ActivityStack.startActivityMayWait来做准备要启动的Activity的相关信息；
Step 3. ActivityStack通知ApplicationThread要进行Activity启动调度了，这里的ApplicationThread代表的是调用ActivityManagerService.startActivity接口的进程，对于通过点击应用程序图标的情景来说，这个进程就是Launcher了，而对于通过在Activity内部调用startActivity的情景来说，这个进程就是这个Activity所在的进程了；
Step 4. ApplicationThread不执行真正的启动操作，它通过调用ActivityManagerService.activityPaused接口进入到ActivityManagerService进程中，看看是否需要创建新的进程来启动Activity；
Step 5. 对于通过点击应用程序图标来启动Activity的情景来说，ActivityManagerService在这一步中，会调用startProcessLocked来创建一个新的进程，而对于通过在Activity内部调用startActivity来启动新的Activity来说，这一步是不需要执行的，因为新的Activity就在原来的Activity所在的进程中进行启动；
Step 6. ActivityManagerServic调用ApplicationThread.scheduleLaunchActivity接口，通知相应的进程执行启动Activity的操作；
Step 7. ApplicationThread把这个启动Activity的操作转发给ActivityThread，ActivityThread通过ClassLoader导入相应的Activity类，然后把它启动起来。

大师的这段描述把ActivityManagerService、ActivityStack、ApplicationThread及ActivityThread的调用关系讲的很清楚，本文将从ActivityThread的main()方法开始分析其主要工作及实现机制。

ActivityThread源码来自：https://github.com/android/platform_frameworks_base/blob/master/core/java/android/app/ActivityThread.java

```
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    SamplingProfilerIntegration.start();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());

    AndroidKeyStoreProvider.install();

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
上述代码中，红色部分之前的代码主要用于环境初始化、AndroidKeyStoreProvider安装等，这里不做重点说明。红色部分的代码主要分为两个功能块：1）绑定应用进程到ActivityManagerService；2）主线程Handler消息处理。

关于线程通信机制，Handler、MessageQueue、Message及Looper四者的关系请参考上一篇文章《Android线程管理——线程通信》。

### 2.1 应用进程绑定
main()方法通过thread.attach(false)绑定应用进程。ActivityManagerNative通过getDefault()方法返回ActivityManagerService实例，ActivityManagerService通过attachApplication将ApplicationThread对象绑定到ActivityManagerService，而ApplicationThread作为Binder实现ActivityManagerService对应用进程的通信和控制。
```
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ……            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
            ……        } else {
    ……
    }
}
```
在ActivityManagerService内部，attachApplication实际是通过调用attachApplicationLocked实现的，这里采用了synchronized关键字保证同步。
```
@Override
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
```
attachApplicationLocked的实现较为复杂，其主要功能分为两部分：

 - thread.bindApplication
 - mStackSupervisor.attachApplicationLocked(app)
 
```
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

    // Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }
   // ……
    try {
       // ……
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                enableTrackAllocation, isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked());
        updateLruProcessLocked(app, false, null);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        // todo: Yikes!  What should we do?  For now we will try to
        // start another process, but that could easily get us in
        // an infinite loop of restarting processes...
        Slog.wtf(TAG, "Exception thrown during bind of " + app, e);

        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        startProcessLocked(app, "bind fail", processName);
        return false;
    }

    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
// ……
}
```
thread对象其实是ActivityThread里ApplicationThread对象在ActivityManagerService的代理对象，故此执行thread.bindApplication，最终会调用ApplicationThread的bindApplication方法。该bindApplication方法的实质是通过向ActivityThread的消息队列发送BIND_APPLICATION消息，消息的处理调用handleBindApplication方法，handleBindApplication方法比较重要的是会调用如下方法：
```
mInstrumentation.callApplicationOnCreate(app);
```
callApplicationOnCreate即调用应用程序Application的onCreate()方法，说明Application的onCreate()方法会比所有activity的onCreate()方法先调用。

mStackSupervisor为ActivityManagerService的成员变量，类型为ActivityStackSupervisor。
```
/** Run all ActivityStacks through this */
ActivityStackSupervisor mStackSupervisor;
```
从注释可以看出，mStackSupervisor为Activity堆栈管理辅助类实例。ActivityStackSupervisor的attachApplicationLocked()方法的调用了realStartActivityLocked()方法，在realStartActivityLocked()方法中，会调用scheduleLaunchActivity()方法：
```
final boolean realStartActivityLocked(ActivityRecord r,
        ProcessRecord app, boolean andResume, boolean checkConfig)
        throws RemoteException {
 
    //...  
    try {
        //...
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,
                new Configuration(mService.mConfiguration),
                r.compat, r.icicle, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profileFile, profileFd,
                profileAutoStop);
 
        //...
 
    } catch (RemoteException e) {
        //...
    }
    //...    
    return true;
}
```

app.thread也是ApplicationThread对象在ActivityManagerService的一个代理对象，最终会调用ApplicationThread的scheduleLaunchActivity方法。

```
// we use token to identify this activity without having to send the
// activity itself back to the activity manager. (matters more with ipc)
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
    ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
    CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
    int procState, Bundle state, PersistableBundle persistentState,
    List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
    boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        updateProcessState(procState, false);

        ActivityClientRecord r = new ActivityClientRecord();

        ……
        sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
同bindApplication()方法，最终是通过向ActivityThread的消息队列发送消息，在ActivityThread完成实际的LAUNCH_ACTIVITY的操作。
```
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            r.packageInfo = getPackageInfoNoCheck(
                r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
    ……
}
```
handleLaunchActivity()用于启动Activity。具体的启动流程不在这里详述了，这里重点说明ApplicationThread及ActivityThread的线程通信机制。

### 2.2 主线程消息处理
 在《Android线程管理——线程通信》中谈到了普通线程中Handler、MessageQueue、Message及Looper四者的关系，那么，ActivityThread中的线程通信又有什么不同呢？不同之处主要表现为两点：1）Looper的初始化方式；2）Handler生成。

首先，ActivityThread通过Looper.prepareMainLooper()初始化Looper，为了直观比较ActivityThread与普通线程初始化Looper的区别，把两种初始化方法放在一起：

```
/** Initialize the current thread as a looper.
  * This gives you a chance to create handlers that then reference
  * this looper, before actually starting the loop. Be sure to call
  * {@link #loop()} after calling this method, and end it by calling
  * {@link #quit()}.
  */
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

/**
 * Initialize the current thread as a looper, marking it as an
 * application's main looper. The main looper for your application
 * is created by the Android environment, so you should never need
 * to call this function yourself.  See also: {@link #prepare()}
 */
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
 * 普通线程的prepare()方法默认quitAllowed参数为true，表示允许退出，ActivityThread在prepareMainLooper()方法中调用prepare()方法，参数为false，表示主线程不允许退出。
 * 普通线程只调用prepare()方法，ActivityThread在调用完prepare()方法之后，会通过myLooper()方法将本地线程<ThreadLocal>的Looper对象的引用交给sMainLooper。myLooper()其实就是调用sThreadLocal的get()方法实现的。
 
```
/**
 * Return the Looper object associated with the current thread.  Returns
 * null if the calling thread is not associated with a Looper.
 */
public static Looper myLooper() {
    return sThreadLocal.get();
}
```
 * 之所以要通过sMainLooper指向ActivityThread的Looper对象，就是希望通过getMainLooper()方法将主线程的Looper对象开放给其他线程。
 
```
/** Returns the application's main looper, which lives in the main thread of the application.
 */
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```
其次，ActivityThread与普通线程的Handler生成方式也不一样。普通线程生成一个与Looper绑定的Handler即可，ActivityThread通过sMainThreadHandler指向getHandler()的返回值，而getHandler()方法返回的其实是一个继承Handler的H对象。

```
private class H extends Handler {
    ……
}

final H mH = new H();

final Handler getHandler() {
    return mH;
}
```
真正实现消息机制“通”信的其实是Looper的loop()方法，loop()方法的核心实现如下：

```
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycle();
    }
}
```
大致流程如下：

 * 首先通过上述myLooper()方法获取Looper对象，取出Looper持有的MessageQueue；
 * 然后从MessageQueue取出Message，如果Message为null，说明线程正在退出；
 * Message不为空，则调用Message的target handler对该Message进行分发，具体分发、处理流程可参考《Android线程管理——线程通信》；
 * 消息处理完毕，调用recycle()方法进行回收。