---
layout: post
title: "sleep & wait"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/wait-notify.png
keywords: "线程同步"
tags: [java]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/wait-notify.png)

## 前言

编码的时候经常会遇到sleep，比如模拟耗时，我们会写一个Thead.sleep(5*1000);

在线程同步的时候也经常用wait/notify; 很明显我们在不同场景使用了不同方法，并且习以为常了。

> 那么sleep和wait的区别到底是什么？


## sleep是怎么回事

![sleep](http://7u2jir.com1.z0.glb.clouddn.com/img/sleep.png)

首先sleep是Thread的静态方法，其次一定要try/catch, 因为sleep可能会抛异常，根据方法定义的说明文档，传入负数会抛出`IllegalArgumentException`,线程被中断会抛出`InterruptedException`


```java
/**
 * Causes the currently executing thread to sleep (temporarily cease
 * execution) for the specified number of milliseconds, subject to
 * the precision and accuracy of system timers and schedulers. The thread
 * does not lose ownership of any monitors.
 *
 * @param  millis
 *         the length of time to sleep in milliseconds
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public static native void sleep(long millis) throws InterruptedException;
```

根据上述说明，我们可以得出两点结论：

1. sleep对当前线程生效；
2. 仅仅让出了CPU资源，这个动作和同步，锁之间没有任何影响，该持有的仍旧持有，锁的状态不变；
3. 睡眠时间到后，继续执行后续代码逻辑；

## wait是怎么回事

![wait-notify](http://7u2jir.com1.z0.glb.clouddn.com/img/wait-notify.png)

![wait-alone](http://7u2jir.com1.z0.glb.clouddn.com/img/wait-alone.png)

wait本身是Object的一个成员方法，因此可以在任意对象上调用；

```java
/**
 * Causes the current thread to wait until another thread invokes the
 * {@link java.lang.Object#notify()} method or the
 * {@link java.lang.Object#notifyAll()} method for this object.
 * In other words, this method behaves exactly as if it simply
 * performs the call {@code wait(0)}.
 * <p>
 * The current thread must own this object's monitor. The thread
 * releases ownership of this monitor and waits until another thread
 * notifies threads waiting on this object's monitor to wake up
 * either through a call to the {@code notify} method or the
 * {@code notifyAll} method. The thread then waits until it can
 * re-obtain ownership of the monitor and resumes execution.
 * <p>
 * As in the one argument version, interrupts and spurious wakeups are
 * possible, and this method should always be used in a loop:
 * <pre>
 *     synchronized (obj) {
 *         while (&lt;condition does not hold&gt;)
 *             obj.wait();
 *         ... // Perform action appropriate to condition
 *     }
 * </pre>
 * This method should only be called by a thread that is the owner
 * of this object's monitor. See the {@code notify} method for a
 * description of the ways in which a thread can become the owner of
 * a monitor.
 *
 * @exception  IllegalMonitorStateException  if the current thread is not
 *               the owner of the object's monitor.
 * @exception  InterruptedException if any thread interrupted the
 *             current thread before or while the current thread
 *             was waiting for a notification.  The <i>interrupted
 *             status</i> of the current thread is cleared when
 *             this exception is thrown.
 * @see        java.lang.Object#notify()
 * @see        java.lang.Object#notifyAll()
 */
public final void wait() throws InterruptedException {
    wait(0);
}
```

wait的注释就比较长了，基本描述了一个实例如何使用wait，如何唤醒
使用wait必须包裹在同步代码块之内，并且同步块持有对象的锁，然后根据业务逻辑执行wait等待；

```java
Object lock = new Object();
synchronized (lock) {
    try {
    	// TODO business logic
        lock.wait(1000);
    } catch (InterruptedException e1) {
        e1.printStackTrace();
    }
}
System.out.println("After wait 1000ms");
```

wait支持参数传入，表示超时时间，在上述代码中，1000ms之后即使没有其他线程调用notify/notifyAll，这里也会往后执行，输出`After wait 1000ms`；如果希望永久等待，可以不传时间，或者传入0，这样将不会看到最后一个日志输出。

从示例代码可以知道，首先synchronized获取了lock对象锁，接着通过lock.wait方法，释放了CPU资源，也释放了lock锁，这就意味着，其他线程如果尝试获取lock锁是可以成功的。

如果lock锁呗其他线程占用，那么即使超时时间到达了，也不会继续往下执行；

下面的例子可以验证这个结论：

```java
new Thread(new Runnable() {
  @Override public void run() {
    System.out.println("[1]execute");
    synchronized (lock) {
      try {
        new Thread(new Runnable() {
          @Override public void run() {
            System.out.println("[2]New Thread executed " + Thread.currentThread());
            synchronized (lock) {
              System.out.println("[2]lock obtained by " + Thread.currentThread());
              try {
                Thread.sleep(5000);
                System.out.println("[2]Job done,release lock " + Thread.currentThread());
                lock.notify();
              } catch (InterruptedException e1) {
                e1.printStackTrace();
              }
            }
          }
        }).start();
        System.out.println("[1]Sleep 1000ms");
        Thread.sleep(1000);
        System.out.println("[1]lock wait 1000");
        lock.wait(1000);
      } catch (InterruptedException e1) {
        e1.printStackTrace();
      }
    }
    System.out.println("[1]After wait 1000ms");
  }
}).start();
```

输出日志:

```
[1]execute
[1]Sleep 1000ms
[2]New Thread executed Thread[Thread-1,5,main]
[1]lock wait 1000
[2]lock obtained by Thread[Thread-1,5,main]
[2]Job done,release lock Thread[Thread-1,5,main]
[1]After wait 1000ms
```
## 小结

wait与sleep都可以休眠一定时间后恢复，但是sleep在休眠期间任然持有原来已经获得状态，锁持有不变，仅仅让出CPU资源供其他线程使用；二wait这是直接释放了持有锁，因此其他请求相同锁的线程会获取成功，并且如果锁被持有的后，只有等待再次释放，原线程才能继续得到执行；
