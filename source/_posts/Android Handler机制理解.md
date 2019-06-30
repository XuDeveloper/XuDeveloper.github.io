---
title: Android Handler机制理解
tags: Android知识点
categories: Android
abbrlink: 8cda8bbd
date: 2017-10-21 14:51:23
---

#### 1.何谓Handler机制？

一般来说，当你的应用被创建的时候，会创建一条应用的主线程。因为效率的考虑，所有的 View 和 Widget 都不是线程安全的，所以相关操作强制放在同一个线程，这样就可以避免多线程带来的问题。这个线程就是主线程，也即 UI 线程。

<!--more-->


当然，你可以创建自己的线程去做操作，但如何应用的主线程通信呢。那就要使用到 Handler 机制了。如果你将一个 Handler 和你的 UI 线程连接，处理消息的代码就将会在 UI 线程中执行。新线程和UI线程的通信是通过从你的新线程调用和主线程相关的 Handler 对象的相关方法实现的。

那接下来就要介绍一下这个消息通讯机制 Handler，涉及到三个主要的类：Looper，Handler 和 Message 类。

#### 2.Looper

重点方法为：prepare()和loop()

##### Looper#prepare()：

```java
private static final ThreadLocal sThreadLocal = new ThreadLocal(); 
public static final void prepare() {  
    if (sThreadLocal.get() != null) {  
        throw new RuntimeException("Only one Looper may be created per thread");  
    }  
    sThreadLocal.set(new Looper()); 
}
```
解释：首先它创建了一个ThreadLocal对象，它是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据。然后在prepare方法中将looper存储在线程里面。

##### Looper#Looper()：
```java 
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed); 
    mRun = true; 
    mThread = Thread.currentThread();
}
```

而在构造方法中，Looper创建了一个MessageQueue，虽然是叫queue但其实内部实现是一个单链表。

##### Looper#loop()：
```java
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

public static Looper myLooper() {
    return sThreadLocal.get();
}
```

解释：方法直接返回了前面sThreadLocal存储的Looper实例，如果me为null则抛出异常，也就是说looper方法必须在prepare方法之后运行。拿到该looper实例中的mQueue即消息队列后进入了无限循环，不断从队列中取出一条消息，如果没有消息则阻塞。如果取得消息使用调用msg.target.dispatchMessage(msg);把消息交给msg的target的dispatchMessage方法去处理。而msg的target是什么呢？其实就是前面讲到的handler对象，最后会释放消息占据的资源。

Looper类总结：
1. 与当前线程绑定，保证一个线程只会有一个Looper实例，同时一个Looper实例也只有一个MessageQueue。
2. loop()方法，不断从MessageQueue中去取消息，交给message的target的dispatchMessage去处理。

接下来就要讲发送消息的对象了，这个对象就是Handler。

#### 3.Handler
主要作用是将一个任务切换到某个指定的线程中去执行，同时为了解决在子线程中无法访问UI的矛盾。

所以我们首先看Handler的构造方法，看其如何与MessageQueue联系上的。

##### Handler#Handler()：
```java 
public Handler() {  
    this(null, false);  
}  
public Handler(Callback callback, boolean async) {  
    if (FIND_POTENTIAL_LEAKS) {  
        final Class<? extends Handler> klass = getClass();  
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) && 
                (klass.getModifiers() & Modifier.STATIC) == 0) {  
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());  
        }  
    }    
    mLooper = Looper.myLooper();  
    if (mLooper == null) {  
        throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");  
    }  
    mQueue = mLooper.mQueue;  
    mCallback = callback;  
    mAsynchronous = async;  
}
```
解释：在构造的时候会检查当前的Handler是否为静态类，不是静态声明的话会打印Log，提示会有内存泄漏现象的产生，然后通过Looper.myLooper()方法获取到当前线程的Looper实例（mLooper）并进一步获取到当前线程的消息队列（mQueue），这样就保证了handler的实例与我们Looper实例中MessageQueue关联上了。

使用的时候我们会经常使用到sendMessage方法，我们来看看源码实现：
```java 
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
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

一路跳到最后的enqueueMessage方法，enqueueMessage中首先为msg.target赋值为this，因为Looper中的loop方法会取出每个msg然后交给msg,target.dispatchMessage(msg)去处理消息，也就是把当前的handler作为msg的target属性。最终会调用queue的enqueueMessage的方法，保存到消息队列中去。

现在已经很清楚了Looper会调用prepare()和loop()方法，在当前执行的线程中保存一个Looper实例，这个实例会保存一个MessageQueue对象，然后当前线程进入一个无限循环中去，不断从MessageQueue中读取Handler发来的消息。然后再回调创建这个消息的handler中的dispatchMessage方法。

##### Handler#dispatchMessage()：

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // 如果message设置了callback，即runnable消息，处理callback！
        handleCallback(msg);  // 并直接调用callback的run方法！
    } else {
        // 如果handler本身设置了callback，则执行callback 
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 如果message没有callback，则调用handler的钩子方法handleMessage
        handleMessage(msg);
    }
}
```
几个变量和方法的解释：
1. callback：message携带的Runnable对象，实际上就是Handler的post方法所传递的Runnable参数。

我们来看一下Handler的post方法源码实现：
```java 
mHandler.post(new Runnable()  {
    @Override 
    public void run() { 
        // code
    }
});
```
其实这个Runnable并没有创建什么线程，而是发送了一条消息：

```java 
public final boolean post(Runnable r) {
    return sendMessageDelayed(getPostMessage(r), 0); 
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain(); 
    m.callback = r; 
    return m;
}
```
在getPostMessage中，得到了一个Message对象，然后将我们创建的Runable对象作为callback属性，赋值给了此message。

注意：产生一个Message对象，可以new，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new重新分配内存。

2. mCallback：可通过Handler handler = new Handler(callback); 可以用来创建一个Handler实例但不需要派生Handler子类。它可用来拦截消息！当mCallback的handleMessage返回true的时候可以拦截消息，具体的逻辑看上面的代码很容易理解！

3. handleMessage(msg)：它是一个空方法，为什么呢，因为消息的最终回调是由我们控制的，我们在创建handler的时候都是复写handleMessage方法，然后根据msg.what进行消息处理。

到此，这个流程已经解释完毕，总结一下：

1. 首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程中只会存在一个。
2. Looper.loop()会让当前线程进入一个无限循环，不端从MessageQueue的实例中读取消息，然后回调msg.target.dispatchMessage(msg)方法。
3. Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与MessageQueue相关联。
4. Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。
5. 在构造Handler实例时，我们会重写handleMessage方法，也就是msg.target.dispatchMessage(msg)。
6. 在Activity中，我们并没有显示的调用Looper.prepare()和Looper.loop()方法，是因为在Activity的启动代码中，已经在当前UI线程调用了Looper.prepare()和Looper.loop()方法。

下面是个人认为在 Activity 中一个合格的 Handler 该有的样子：

```java 
private static class MyHandler extends Handler {
    private WeakReference<CustomActivity> activityWeakReference;
    public MyHandler(CustomActivity activity) {
        activityWeakReference = new WeakReference<CustomActivity>(activity);
    }
    @Override
    public void handleMessage(Message msg) {
        CustomActivity activtiy = activityWeakReference.get();
        if (activity != null) {
            // code               
        }
    }
} 
```