---
title: 『Android』源码级探索Handler对象
date: 2018-3-8 14:03:00
categories: 
tags: Android
---


## Handler介绍

> A Handler allows you to send and process Message and Runnable objects associated with a thread's MessageQueue.  Each Handler instance is associated with a single thread and that thread's message queue.  When you create a new Handler, it is bound to the thread / message queue of the thread that is creating it -- from that point on, it will deliver messages and runnables to that message queue and execute them as they come out of the message queue.

Handler允许您发送和处理与线程的MessageQueue关联的Message和Runnable对象。每个Handler实例都与单个线程和该线程的消息队列相关联。当您创建一个新的Handler时，它将绑定到正在创建它的线程的线程/消息队列 -- 从那时起，它将消息(messages)和可运行消息(runnables)传递到该消息队列(message queue)，并在消息队列出来时执行它们。

<!-- more -->

> There are two main uses for a Handler: 
> (1) to schedule messages and runnables to be executed as some point in the future; 
> (2) to enqueue an action to be performed on a different thread than your own.

Handler的两个主要用途：

(1)、在某一时刻执行某些事情
(2)、在不同的线程中执行操作 (也就是线程间通信)

## Looper、MessageQueue、Message介绍

### Looper

Looper字面意思是一个轮询器，是用来检测MessageQueue中是否有Message消息，如果有消息则把消息分发出去，没有消息就阻塞在那里直到有消息过来为止。
一个线程对应一个Looper,一个Looper对应一个MessageQueue.

> 默认情况下，线程是没有Looper的，所以要调用 Looper.prepare()来给线程创建消息队列，然后再通过，Looper.loop()来不停（死循环）的处理消息消息队列的消息

### MessageQueue

是用来存放消息的，通过Looper.prepare()来初始化。MessageQueue从字面意思来看是一个消息队列，但是内部实现并不是基于队列的。是基于一个单链表的数据结构来维护消息列表的，链表插入和删除优势很明显。

### Message

传递信息的载体。

## Handler简单用法

Step1.初始化Handler对象，重写handleMessage方法

```java
Looper.prepare();
Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        // TODO 
    }
};
Looper.loop();
```

Step2.发送消息通知Handler处理

```java
Message message = new Message();
// Message message = mHandler.obtainMessage();
message.what = 0;
mHandler.sendMessage(message);
```

## 源码探索

### handler的构造方法

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    // ...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

从该构造方法可以看出，获取当前线程的Looper对象赋值给mLooper变量，接着将Looper中的消息队列赋值给mQueue变量，这里Handler就与当前线程中的Looper对象绑定了。

### handler.sendMessage()

`mHandler.sendMessage(message);`将消息发送出去，此过程中经历了什么？

从源码处可看到执行了`sendMessageDelayed(msg, 0)`

```java
public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
}
```

在`sendMessageDelayed`方法中执行`sendMessageAtTime`

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

在`sendMessageAtTime`中执行`enqueueMessage`方法传递`MessageQueue`，`Message`及`运行时间`uptimeMillis

``` java
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

`enqueueMessage`中`msg.target = this`，这个this指的就是Handler对象，意思是指这个Message中的target属性是当前这个Handler对象，之后调用`MessageQueue`的`enqueueMessage`方法。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

`enqueueMessage()`此方法将Message插入到MessageQueue中。

`MessageQueue.java`

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
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
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
        
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

由此可见，handler.sendMessage()最终将消息插入到MessageQueue中。

### Looper

1.`Looper.prepare();`创建Looper对象并初始化消息队列

```java
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

由此可见，最终初始化了mQueue和mThread这两个变量。

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

已经有了消息队列，也了解到sendMessage是将消息插入到消息队列中，那如何管理MessageQueue中的消息呢，这时候就需要依靠Looper.loop()方法了。

2.`Looper.loop();`轮询获取消息并分发

```java
public static void loop() {
    // 从sThreadLocal中取出当前线程的Looper对象
    final Looper me = myLooper();
    if (me == null) {
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 获取消息队列
    final MessageQueue queue = me.mQueue;
    // ...省略部分代码
    for (;;) {
        Message msg = queue.next();
        if (msg == null) {
            return;
        }
        // ...省略部分代码
        // 分发消息
        msg.target.dispatchMessage(msg);
        // ...省略部分代码
        // 回收消息
        msg.recycleUnchecked();
    }
}
```

由此可以看到，Looper.loop()方法是循环取出MessageQueue中的消息然后调用Handler的dispatchMessage方法将消息分发出去。

然后我们再接着看下dispatchMessage()方法

```java
public void dispatchMessage(Message msg) {
    // 如果msg的callback不为空的话
    if (msg.callback != null) {
        // 执行handleCallback(msg)
        handleCallback(msg);
    } else {
        // msg.callback == null && mCallback != null 时
        // 执行handleMessage(msg)
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
            handleMessage(msg);
        }
    }
}
```

这是我们看到了熟悉的handleMessage(msg)方法。

``` java
// 初始化消息队列
Looper.prepare();
Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        // 处理发送过来的消息
    }
};
// 循环监听消息队列，读取消息队列中的消息，
// 调用handler.dispatchMessage方法，最后调用
// handler的handlerMessage(Message msg)方法处理消息。
Looper.loop();
```

此时重写其handleMessage方法，可以处理发送过来的消息。

## Activity中的Looper

我们在平时开发过程中，并没有在Activity中初始化Looper对象(Looper.prepare()和Looper.loop())，那为什么可以使用Handler呢?

Activity所在的主线程其实是由ActivityThread管理的，我们来看一下ActivityThread的源码。

```java
// 入口main方法
public static void main(String[] args) {
    // ...
    Looper.prepareMainLooper();
    // ...
    Looper.loop();
}
```

由此我们就知道了，为什么我们并没有在Activity中使用Looper.prepare()和Looper.loop()却可以使用Handler。所以Activity默认有一个Looper，Service同样。

## Handler引起的内存泄漏，如何避免？

泄漏原因：非静态内部类会持有外部类的引用

解决方案：避免使用非静态内部类

```java
static class MyHandler extends Handler {
      @Override
      public void handleMessage(Message msg) {
        
    }
}
```

完整代码：

```java
public class MyActivity extends AppCompatActivity {
    private MyHandler mHandler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mHandler = new MyHandler(this);
    }
    @Override
    protected void onDestroy() {
        // Remove all Runnable and Message.
        mHandler.removeCallbacksAndMessages(null);
        super.onDestroy();
    }
    static class MyHandler extends Handler {
        // WeakReference to the outer class's instance.
        private WeakReference<MyActivity> mOuter;
        public MyHandler(MyActivity activity) {
            mOuter = new WeakReference<MyActivity>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            MyActivity outer = mOuter.get();
            if (outer != null) {
                // Do something with outer as your wish.
            }
        }
    }
}
```

