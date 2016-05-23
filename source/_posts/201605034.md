title: Android线程管理（三）——Thread类的内部原理、休眠及唤醒
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
## 三、Thread类的内部原理、休眠及唤醒
### 3.1 Thread类的内部原理
线程是CPU资源调度的基本单位，属于抽象范畴，Java通过Thread类完成线程管理。Thread类本质其实是“可执行代码”，其实现了Runnable接口，而Runnable接口唯一的方法就是run()。
```
public class Thread implements Runnable {
    ……
}
```
```
public interface Runnable {

    /**
     * Starts executing the active part of the class' code. This method is
     * called when a thread is started that has been created with a class which
     * implements {@code Runnable}.
     */
    public void run();
}
```
 从注释可以看出，调用Thread的start()方法就是间接调用Runnable接口的run()方法。

```
public synchronized void start() {
    checkNotStarted();

    hasBeenStarted = true;

    VMThread.create(this, stackSize);
}
```
start()方法中VMThread.create(this, stackSize)是真正创建CPU线程的地方，换句话说，只有调用start()后的Thread才真正创建CPU线程，而新创建的线程中运行的就是Runnable接口的run()方法。

### 3.2 线程休眠及唤醒
线程通信、同步、协作是多线程编程中常见的问题。线程协作通常是采用线程休眠及唤醒来实现的，线程的休眠通过等待某个对象的锁（monitor）实现（wait()方法），当其他线程调用该对象的notify()方法时，该线程就被唤醒。该对象实现在线程间数据传递，多个线程通过该对象实现协作。

线程协作的经典例子是Java设计模式中的“生产者-消费者模式”，生产者不断往缓冲区写入数据，消费者从缓冲区中取出数据进行消费。在实现上，生产者与消费者分别继承Thread，缓冲区采用优先级队列PriorityQueue来模拟。生产者将数据放入缓冲区的前提是缓冲区有剩余空间，消费者从缓冲区中取出数据的前提是缓冲区中有数据，因此，这就涉及到生成者线程与消费者线程之间的协作。下面通过代码简要说明下。

```
public class TestWait {
    private int size = 5;
    private PriorityQueue<Integer> queue = new PriorityQueue<Integer>(size);

    public static void main(String[] args) {
        TestWait test = new TestWait();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();

        producer.start();
        consumer.start();
    }

    class Consumer extends Thread {

        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == 0) {
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.poll(); // 每次移走队首元素
                       queue.notify();
                    System.out.println("从队列取走一个元素，队列剩余" + queue.size() + "个元素");
                }
            }
        }
    }

    class Producer extends Thread {

        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == size) {
                        try {
                            System.out.println("队列满，等待有空余空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.offer(1); // 每次插入一个元素
                       queue.notify();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："
                            + (size - queue.size()));
                }
            }
        }
    }
}
```
这段代码在很多讲述生产者-消费者模式的地方都会用到，其中，Producer线程首先启动，synchronized关键字使其能够获得queue的锁，其他线程处于等待状态。初始queue为空，通过offer向缓冲区队列写入数据，notify()方法使得等待该缓冲区queue的线程（此处为消费者线程）唤醒，但该线程并不能马上获得queue的锁，只有等生产者线程不断向queue中写入数据直到queue.size() == size，此时缓冲队列充满，生产者线程调用wait()方法进入等待状态。此时，消费者线程处于唤醒并且获得queue的锁，通过poll()方法消费缓冲区中的数据，同理，虽然调用了notify()方法使得生产者线程被唤醒，但其并不能马上获得queue的锁，只有等消费者线程不断消费数据直到queue.size() == 0，消费者线程调用wait()方法进入等待状态，生产者线程重新获得queue的锁，循环上述过程，从而完成生产者线程与消费者线程的协作。

在Android的SystemServer中有多处用到了线程协作的方式，比如WindowManagerService的main()中通过runWithScissors()启动的BlockingRunnable与SystemServer所在线程的协作。WindowManagerService源码地址可参考：https://github.com/android/platform_frameworks_base/blob/master/services/core/java/com/android/server/wm/WindowManagerService.java

### 3.3 线程中断
在Java中“中断”线程是通过interrupt()方法来实现的，之所以加引号，是因为interrupt()并不中断正在运行的线程，只是向线程发送一个中断请求，具体行为依赖于线程的状态，在文档中有如下说明：
```
Posts an interrupt request to this Thread. The behavior depends on the state of this Thread:

Threads blocked in one of Object's wait() methods or one of Thread's join() or sleep() methods will be woken up, their interrupt status will be cleared, and they receive an InterruptedException. 
Threads blocked in an I/O operation of an java.nio.channels.InterruptibleChannel will have their interrupt status set and receive an java.nio.channels.ClosedByInterruptException. Also, the channel will be closed. 
Threads blocked in a java.nio.channels.Selector will have their interrupt status set and return immediately. They don't receive an exception in this case.
```

翻译下：

 - 如果线程处于阻塞状态，即线程被Object.wait()、Thread.join()或 Thread.sleep()阻塞，调用interrupt()方法，将接收到InterruptedException异常，中断状态被清除，结束阻塞状态；
 - 如果线程在进行I/O操作（java.nio.channels.InterruptibleChannel）时被阻塞，那么线程将收到java.nio.channels.ClosedByInterruptException异常，通道被关闭，结束阻塞状态；
 - 如果线程被阻塞在java.nio.channels.Selector中，那么中断状态会被置位并返回，不会抛出异常。
 
```
public void interrupt() {
    // Interrupt this thread before running actions so that other
    // threads that observe the interrupt as a result of an action
    // will see that this thread is in the interrupted state.
    VMThread vmt = this.vmThread;
    if (vmt != null) {
        vmt.interrupt();
    }

    synchronized (interruptActions) {
        for (int i = interruptActions.size() - 1; i >= 0; i--) {
            interruptActions.get(i).run();
        }
    }
}
```

### 3.4 join()和sleep()方法
join()方法也可以理解为线程之间协作的一种方式，当两个线程需要顺序执行时，调用第一个线程的join()方法能使该线程阻塞，其依然通过wait()方法来实现的。
```
/**
 * Blocks the current Thread (<code>Thread.currentThread()</code>) until
 * the receiver finishes its execution and dies.
 *
 * @throws InterruptedException if <code>interrupt()</code> was called for
 *         the receiver while it was in the <code>join()</code> call
 * @see Object#notifyAll
 * @see java.lang.ThreadDeath
 */
public final void join() throws InterruptedException {
    VMThread t = vmThread;
    if (t == null) {
        return;
    }
    synchronized (t) {
        while (isAlive()) {
            t.wait();
        }
    }
}
```
另外，还有带时间参数的join()方法，在超出规定时间后，退出阻塞状态。同样的，其通过带时间参数的wait()方法实现而已。
```
public final void join(long millis) throws InterruptedException{}
public final void join(long millis, int nanos) throws InterruptedException {}
```
sleep()与wait()的相同之处在于它们都是通过等待阻塞线程，不同之处在于sleep()等待的是时间，wait()等待的是对象的锁。
```
public static void sleep(long time) throws InterruptedException {
    Thread.sleep(time, 0);
}
```
```
public static void sleep(long millis, int nanos) throws InterruptedException {
    VMThread.sleep(millis, nanos);
}
```

### 3.5 CountDownLatch
CountDownLatch位于java.util.concurrent.CountDownLatch，实现倒数计数锁存器，当计数减至0时，触发特定的事件。在某些主线程需要等到子线程的应用很实用，以Google的zxing开源库中的一段代码为例进行说明：
```
final class DecodeThread extends Thread {

  ……
  private final CountDownLatch handlerInitLatch;

  DecodeThread(CaptureActivity activity,
               Collection<BarcodeFormat> decodeFormats,
               Map<DecodeHintType,?> baseHints,
               String characterSet,
               ResultPointCallback resultPointCallback) {

    this.activity = activity;
    handlerInitLatch = new CountDownLatch(1);

    ……
  }

  Handler getHandler() {
    try {
      handlerInitLatch.await();
    } catch (InterruptedException ie) {
      // continue?
    }
    return handler;
  }

  @Override
  public void run() {
    Looper.prepare();
    handler = new DecodeHandler(activity, hints);
    handlerInitLatch.countDown();
    Looper.loop();
  }
}
```
在上述例子中，首先在DecodeThread构造器中初始化CountDownLatch对象，并传入初始化参数1。其次，在run()方法中调用CountDownLatch对象的countDown()方法，这很好的保证了外部实例通过getHandler()方法获取handler时，handler不为null。