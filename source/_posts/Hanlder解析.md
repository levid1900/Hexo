---
title: Hanlder原理解析
date: 2017-03-02 15:34:43
tags: Android
---

## 相关类
Looper:在线程中不断循环Message。

ThreadLocal:线程内部存储类。所有的线程都使用同一个ThreadLocal对象，但是对应不同值，修改里面内容也不会影响其它线程。在线程中用来存储Looper对象。

MessageQueue:消息队列。存储等待分发的消息。

Message:消息对象。

**类图**
//TODO

## Looper
```java
class LooperThread extends Thread {
       public Handler mHandler;
       
      public void run() {
          Looper.prepare();

           mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```
Looper类在线程中持有一个Message循环，每个线程都会有一个Looper对象。

Looper类中的对象

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static Looper sMainLooper;  // guarded by Looper.class

final MessageQueue mQueue;
final Thread mThread;
```
静态ThreadLocal中存储的是每个线程的Looper对象。

静态Looper对象sMainLooper，应用主线程的Looper，即UI线程的Looper对象。它的初始化如下：

```java
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

/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

/**

```
注释中可以看到sMainLooper的声明是由Android环境调用，在每个应用程序初始化时调用此方法，声明主线程的Looper对象.
_sThreadLocal.get()_是从当前线程中取Looper对象，如果没有就new一个。

Looper使用前必须调用_prepare()_方法(主线程调用_prepareMainLooper()_)。
看一下_prepare()_方法中做了什么。

```java
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
```
初始化一个Looper对象放入sThreadLocal中，如果已经初始化过就会抛出异常_"Only one Looper may be created per thread"_，所以_prepare()_在每个线程中只允许调用一次，每个线程中只能有一个Looper对象。
看一下Looper的初始化方法：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
初始化MessageQueue对象，以及保存当前线程。
初始化完了调用_Looper.looper()_开始循环消息队列。

```java
public static void loop() {
        final Looper me = myLooper();
        //先判断有没有Looper对象，即有没有调用过prepare()方法，没有就抛出异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            //取下一条message，如果没有，则线程会阻塞在这里，具体实现在MessageQueue里
            Message msg = queue.next(); // might block
            if (msg == null) {
                // 如果取出null，此循环就会退出
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            //msg.target是个Handler，在Handler对象中调用dispatchMessage()，分发消息，具体实现在下面
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

            msg.recycleUnchecked();
        }
    }
```
_Looper.looper()_不断的从_mQueue_中取出Message，如果暂时没有Message，线程就会阻塞。取出Message后，在Handler中进行消息的分发处理。

## MessageQueue
Looper里有个MessageQueue对象，保存了待处理的消息队列。
MessageQueue有几个主要的方法：_next()_ _enqueueMessage()_ _removeMessages()_
下一条消息、加入消息、移除消息。

上面Looper循环中在不断的取下一条消息，即_next()_

```java
Message next() {
        // ptr是一个类似指针的变量，主要在native方法中调用
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            //native方法，阻塞nextPollTimeoutMillis毫秒，当nextPollTimeoutMillis = -1时，会持续阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                //mMessages是当前消息
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 取下一条异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 下一条消息还没有到执行时间，会先阻塞
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // message符合要求
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // 没有message，会一直阻塞线程
                    nextPollTimeoutMillis = -1;
                }

                if (mQuitting) {
                    dispose();
                    return null;
                }

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            pendingIdleHandlerCount = 0;

            nextPollTimeoutMillis = 0;
        }
    }
```

如果message的执行时间没有到，线程会一直阻塞；如果当前没有message，线程也会阻塞，直到新message进入队列，_enqueueMessage()_被调用

```java
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 如果没有当前message或者当前message的执行时间在传入message的后面
                //传入消息就插入到当前message前面变为当前message
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                //循环找到传入message合适位置插入队列
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            if (needWake) {
                //如果之前有阻塞的话 native方法唤醒线程
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
由上面可知，Looper.looper()方法里对线程的唤醒阻塞是靠MessageQueue的插入消息以及取消息来完成的。

## Handler
 Handler是在线程中运行一个Message循环的类。先看下构造方法：

 ```java

 public Handler() {
         this(null, false);
     }

public Handler(Callback callback) {
         this(callback, false);
    }

public Handler(Looper looper) {
         this(looper, null, false);
    }

public Handler(Looper looper, Callback callback) {
         this(looper, callback, false);
    }

public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

 ```
 可以看到，初始化时如果没有传入Looper对象，会使用当前线程的Looper,而在此之前，必须调用_Looper.prepare()_，否则会抛出异常"Can't create handler inside thread that has not called Looper.prepare()"。

 Callback是个接口
 ```java
 public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
实例化后必须重写_handleMessage(Message msg)_方法来处理消息。

初始化Handler时我们一般也会重写_dispatchMessage(Message msg)_或者_handlerMessage(Message msg)_方法

```java
public void handleMessage(Message msg) {
    }
    
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
这两个方法都是来处理循环中的消息，前面在Looer类中_looper()_方法

```java
msg.target.dispatchMessage(msg);
```
调用了_dispatchMessage()_方法，而后_dispatchMessage()_中再调用_handleMessage()_，所以子类继承这两个任意一个都行。

处理消息看完了来看一下发送消息_sendMessage()_

```java
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
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
    
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
    

```
_sendMessage()_最后走向_queue.enqueueMessage()_，将Message扔到Looper中的MessageQueue中，然后Looper去循环消息队列发送到Handler子类实现的_dispatchMessage()_中处理。

这就是Handler处理消息的流程。

## 使用Handler更新UI
大家都知道，在子线程中更新UI必须是用Handler,否则会报错。那么，子线程使用Handler为什么可以更新UI？

看前面的例子：

```java
class LooperThread extends Thread {
       public Handler mHandler;
       
      public void run() {
          Looper.prepare();

           mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```
这里的Hanlder处理消息可不可以更新UI呢？答案是不能。
上面代码中，_new Handler()_初始化没有代入任何参数，而从前面分析可知，Handler初始化如果没有传入Looper对象会使用当前线程的Looper对象，这里就是子线程的Looper对象,Handler中消息也会加入到子线程的MessageQueue。子线程的Looper循环还是无法更新主线程的UI。

可以更新UI的实例是这样的：

```java
class LooperThread extends Thread {
       public Handler mHandler;
       
      public void run() {

           mHandler = new Handler(Looper.getMainLooper()) {
              public void handleMessage(Message msg) {
                  // process incoming messages here
                  updateUI();
              }
          };

      }
  }
```
这个实例中Handler初始化时传入了_Looper.getMainLooper()_，主线程即UI线程的Looper()。此时，Handler中的Looper对象是_sMainLooper_，主线程中的初始化的Looper对象。Handler消息会插入主线程的消息队列，此时可以更新UI。





    




