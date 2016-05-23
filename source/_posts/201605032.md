title: Android线程管理（一）——线程通信
tags:
- Android线程
categories: Android线程管理

---

线程通信、ActivityThread及Thread类是理解Android线程管理的关键。

线程，作为CPU调度资源的基本单位，在Android等针对嵌入式设备的操作系统中，有着非常重要和基础的作用。本小节主要从以下三个方面进行分析：

《Android线程管理（一）——线程通信》
《Android线程管理（二）——ActivityThread》
《Android线程管理（三）——Thread》

----------
## 一、Handler、MessageQueue、Message及Looper四者的关系
在开发Android多线程应用时，Handler、MessageQueue、Message及Looper是老生常谈的话题。但想彻底理清它们之间的关系，却需要深入的研究下它们各自的实现才行。首先，给出一张它们之间的关系图：
![](/img/20160113-1.png)

 - Looper依赖于MessageQueue和Thread，因为每个Thread只对应一个Looper，每个Looper只对应一个MessageQueue。
 - MessageQueue依赖于Message，每个MessageQueue对应多个Message。即Message被压入MessageQueue中，形成一个Message集合。
 - Message依赖于Handler进行处理，且每个Message最多指定一个Handler来处理。Handler依赖于MessageQueue、Looper及Callback。
 
从运行机制来看，Handler将Message压入MessageQueue，Looper不断从MessageQueue中取出Message（当MessageQueue为空时，进入休眠状态），其target handler则进行消息处理。因此，要彻底弄清Android的线程通信机制，需要了解以下三个问题：


 - Handler的消息分发、处理流程
 - MessageQueue的属性及操作
 - Looper的工作原理
 
### 1.1 Handler的消息分发、处理流程
Handler主要完成Message的入队（MessageQueue）和处理，下面将通过Handler的源码分析其消息分发、处理流程。首先，来看下Handler类的方法列表：
![](/img/20160113-2.png)
从上图中可以看出，Handler类核心的方法包括：1）构造器；2）分发消息；3）处理消息；4）post发送消息；5）send发送消息；6）remove消息和回调。

首先，从构造方法来看，构造器的多态最终通过调用如下方法实现，即将实参赋值给Handler类的内部域。
```
final MessageQueue mQueue;
final Looper mLooper;
final Callback mCallback;
final boolean mAsynchronous;

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
其次，消息的入队是通过post方法和send方法来实现的。
```
public final boolean postAtTime(Runnable r, long uptimeMillis) {
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}
```
```
public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageAtTime(msg, uptimeMillis);
}
```
```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
两者的区别在于参数类型不同，post方法传入的实例对象实现了Runnable接口，然后在内部通过getPostMessage方法将其转换为Message，最终通过send方法发出；send方法传入的实例对象为Message类型，在实现中，将Message压入MessageQueue。
```
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
通过Handler将Message压入MessageQueue之后，Looper将其轮询后交由Message的target handler处理。Handler首先会对消息进行分发。首先判断Message的回调处理接口Callback是否为null，不为null则调用该Callback进行处理；否判断Handler的回调接口mCallback是否为null，不为null则调用该Callback进行处理；如果上述Callback均为null，则调用handleMessage方法处理。
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
handleMessage方法在Handler的子类中必须实现。即消息具体的处理交由应用软件实现。
```
/**
 * Subclasses must implement this to receive messages.
 */
public void handleMessage(Message msg) {

}
```
回到Activity（Fragment），在Handler的子类中实现handleMessage方法。这里需要注意一个内存泄露的问题，比较下述两种实现方式，第一种直接定义Handler的实现，第二种通过静态内部类继承Handler，定义继承类的实例。
```
Handler mHandler = new Handler() {
            
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
            
        // 根据msg调用Activity的方法
    }
};
```
```
static class MyHandler extends Handler {

    WeakReference<DemoActivity> mActivity;

    public MyHandler(DemoActivity demoActivity) {
        mActivity = new WeakReference<DemoActivity>(demoActivity);
    }

    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        DemoActivity theActivity = mActivity.get();

        // 根据msg调用theActivity的方法
    }
}
```
不绕弯子，直接说明为什么第一种方式会引起内存泄露，而第二种不会。

在第一种方式中，mHandler通过匿名内部类方式实例化，在Java中，内部类会强持有外部类的引用（handleMessage方法中可以直接调用Activity的方法），在外部Activity调用onDestroy()方法之后，如果Handler的MessageQueue依然有未处理的消息，那么由于Handler持有Activity的引用导致Activity无法被系统GC回收，从而引起内存泄露。

在第二种方式中，首先继承Handler定义静态内部类，由于MyHandler为静态类，即使定义在Activity的内部，也与Activity没有逻辑上的联系，即不会持有外部Activity的引用；其次，在静态类内部，定义外部Activity的弱引用，弱引用在系统资源紧张时会被系统优先回收。最后，在handleMessage()方法中，通过WeakReference的get方法获取外部Activity的引用，如果该弱引用已被回收，则get方法返回null。
```
struct GcSpec {
  /* If true, only the application heap is threatened. */
  bool isPartial;
  /* If true, the trace is run concurrently with the mutator. */
  bool isConcurrent;
  /* Toggles for the soft reference clearing policy. */
  bool doPreserve;
  /* A name for this garbage collection mode. */
  const char *reason;
};
```
这段代码定义在dalvik/vm/alloc/Heap.h中，其中doPreserve为true时，表示在执行GC的过程中，不回收软引用引用的对象；为false时，表示在执行GC的过程中，回收软引用引用的对象。

最后，使用Handler的过程中，还需要注意一点，在前面的方法列表图中已经提到。为避免Activity调用onDestroy后，Handler的MessageQueue中仍存在Message，一般会在onDestroy中调用removeCallbacksAndMessages()方法。
```
@Override
protected void onDestroy() {
    super.onDestroy();
    // 清空Message队列
    myHandler.removeCallbacksAndMessages(null);
}
```
```
public final void removeCallbacksAndMessages(Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}
```
removeCallbacksAndMessages()方法会移除obj为token的由post发送的callback和send发送的message，当token为null时，会移除所有callback和message。

### 1.2 MessageQueue的属性及操作
MessageQueue，消息队列，其属性与常规队列相似，包括入队、出队等，这里简要介绍一下MessageQueue的实现。

首先，MessageQueue新建队列的工作是通过在其构造器中调用本地方法nativeInit实现的。nativeInit会创建NativeMessageQueue对象，然后赋值给MessageQueue成员变量mPtr。mPtr是int类型数据，代表NativeMessageQueue的内存指针。
```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```
其次，Message入队的通过enqueueMessage方法实现。首先检查message是否符合入队要求（是否正在使用，target handler是否为null），符合要求后通过设置prev.next = msg队列的指针完成入队操作。
```
boolean enqueueMessage(Message msg, long when);
```
再次，出队是通过next()方法完成的。涉及到同步、锁等问题，这里不详细展开了。

再次，删除元素有两个实现。即分别通过p.callback == r和p.what == what来进行消息识别。
```
void removeMessages(Handler h, int what, Object object);
void removeMessages(Handler h, Runnable r, Object object);
```
最后，销毁队列和创建队列一样，是通过本地函数完成的。传入的参数为MessageQueue的内存指针。
```
private native static void nativeDestroy(int ptr);
```
### 1.3 Looper的工作原理
Looper是线程通信的关键，正是因为Looper，整个线程通信机制才真正实现“通”。

在应用开发过程中，一般当主线程需要传递消息给用户自定义线程时，会在自定义线程中定义Handler进行消息处理，并在Handler实现的前后分别调用Looper的prepare()方法和loop()方法。大致实现如下：

```
new Thread(new Runnable() {
            
    private Handler mHandler;
            
    @Override
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                        
            }
        };
        Looper.loop();
    }
});
```

> 这里重点说明prepare()方法和loop()方法，实际项目中不建议定义匿名线程。

```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
可以看出，prepare方法的重点是sThreadLocal变量，sThreadLocal变量是什么呢？
```
// sThreadLocal.get() will return null unless you've called prepare().
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
ThreadLocal实现了线程本地存储。简单看一下它的类注解文档，ThreadLocal是一种特殊的全局变量，全局性在于它存储于自己所在线程相关的数据，而其他线程无法访问。
```
/**
 * Implements a thread-local storage, that is, a variable for which each thread
 * has its own value. All threads share the same {@code ThreadLocal} object,
 * but each sees a different value when accessing it, and changes made by one
 * thread do not affect the other threads. The implementation supports
 * {@code null} values.
 *
 * @see java.lang.Thread
 * @author Bob Lee
 */
public class ThreadLocal<T> {

}
```
回到prepare方法中，sThreadLocal添加了一个针对当前线程的Looper对象。并且prepare方法只能调用一次，否则会抛出运行时异常。

初始化完毕之后，Handler通过post和send方法如何保证消息投递到Looper所持有的MessageQueue中呢？其实，MessageQueue是Handler和Looper的桥梁。在前面Handler章节中提到Handler的初始化方法，Handler的mLooper对象是通过Looper的静态方法myLooper()获取的，而myLooper()是通过调用sThreadLocal.get()来得到的，即Handler的mLooper就是当前线程的Looper对象，Handler的mQueue就是mLooper.mQueue。
```
……
mLooper = Looper.myLooper();
if (mLooper == null) {
   throw new RuntimeException(
        "Can't create handler inside thread that has not called Looper.prepare()");
}
mQueue = mLooper.mQueue;
……
```
```
public static Looper myLooper() {
    return sThreadLocal.get();
}
```
      