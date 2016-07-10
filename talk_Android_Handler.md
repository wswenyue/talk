# 深入解析Handler原理以及Handler的使用姿势

Handler作为一个老朋友，对我们来说既熟悉又陌生。初学Android会接触到它，进阶、高级也需要接触它。Handler在Android的世界里也是极其的重要，子线程更新UI、线程之间通信都会用到它。下面就由我有深入浅慢慢道来。


一般初学者使用Handler基本都是这样：

```java
public class HandlerActivity extends Activity {
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == 0x123) {
                //.... 更新UI
                mTv.setText(msg.obj.toString());
            }
        }
    };
    private TextView mTv;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        mTv = (TextView) findViewById(R.id.mTv);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //执行耗时任务
                    Thread.sleep(1000 * 60);
                    //发送消息
                    Message msg = mHandler.obtainMessage();
                    msg.what = 0x123;
                    msg.obj = "hello text";
                    mHandler.sendMessage(msg);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```
从上面熟悉的代码，我们就知道Handler是这样被用来做UI的更新。但是这样代码写了好久，我们就会有这样的疑问，Handler到底是什么？   
 
Handler是什么呢？我们查阅资料发现：有人说它是Android中更新UI的一种机制；也有人说它是Android中线程间一套消息机制。   

有了答案我们就去探究。如果说它是更新UI的一种机制，那么为什么要使用Handler来更新UI？不使用会怎么样。然后我们做了尝试发现用子线程去访问UI会报错，Android只允许UI线程去访问UI，那么怎么办。如果不用Handler，让UI线程去更新，比如我们去请求服务器获取到数据，然后把数据展示在UI上。你会发现数据还没有请求回来，程序就会报ANR错误即臭名昭著的应用程序无响应，Android又建议不要在主线程中进行耗时操作，否则会导致程序无法响应。感觉挺矛盾的，既不让搞耗时任务又不让在子线程中访问UI，那怎么办。初学Android的时候遇到这样的问题，我一脸懵逼。。。   
 
说了半天。。还是得用Handler。。但是Android为什么不让非UI线程访问UI呢？？？   
查阅资料，我找到AndroidUI处理使用的是**单线程模型**，为啥不用多线程呢，是因为**多线程并发访问UI可能会导致UI控件处于不可预期的状态**。如果加锁，虽然能解决，但是缺点也很明显：1.**锁机制让UI访问逻辑变得复杂**；2.**加锁导致效率低下**。所以简单高效的做法就是使用单线程模型，然后通过Handler这种机制去更新UI。

看到这里，似乎好像Handler就是Android中更新UI的一种机制？？其实深入了解，你会发现它实质是Android中线程间的一种消息机制。只是子线程在更新UI时使用的是发送消息，并且Handler也常常被开发者用来更新UI。

Handler的运行机制==Android消息机制   
Android的消息机制即Handler的运行机制，它所涉及到的类有：

- Message 消息的封装，消息体
- MessageQueue 消息队列
- Handler 消息机制的上层接口。封装了消息的发送（主要包括消息发送给谁）
- Looper 消息机制的发动机

先来看看Android源码中用到Handler消息机制的地方：   
Activity的生命周期的回调方法都是通过Handler机制发送相应消息，根据不同的Message做出不同的分支处理。AMS（ActivityManageService）发送消息给ActivityThread，然后由ActivityThread对相应消息做出相应方法的回调。


下面讲解一下Android消息机制的源码实现

在这之前先来认识一下：**ThreadLocal**    
定义：不是线程，是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。 Android源码中使用到地方有：Looper、ActivityThread、AMS。   
具体的使用例子：

```java 
public class MainActivity extends Activity {
 private ThreadLocal<Boolean> mThreadLocal = new ThreadLocal<>();

 public void threadClick(View view) {
 		//UI线程
        mThreadLocal.set(true);
        Log.e(TAG_THREADLOCAL, Thread.currentThread().getName() + "##Value>>" + mThreadLocal.get());//结果 true
        new Thread("Thread#1#") {
            @Override
            public void run() {
            	//子线程Thread#1#
                mThreadLocal.set(false);
                Log.e(TAG_THREADLOCAL, Thread.currentThread().getName() + "##Value>>" + mThreadLocal.get());//结果 false
            }
        }.start();
        new Thread("Thread#2#") {
            @Override
            public void run() {
            //子线程Thread#2#"
                Log.e(TAG_THREADLOCAL, Thread.currentThread().getName() + "##Value>>" + mThreadLocal.get());//结果 null
            }
        }.start();

    }
}
    
```
通过这个例子我们就可以清晰的知道ThreadLocal是一个线程相关的存储类。

了解完ThreadLocal，我们来通过一个子线程间使用Handler通信的例子来看看Android消息机制的源码实现。

这个是子线程A

```java
public class ThreadA extends Thread {
    private static final String TAG = "ThreadA";
    public static Handler mHandlerA;
    @Override
    public void run() {
        Looper.prepare();
        mHandlerA = new Handler(Looper.myLooper()) {
            @Override
            public void handleMessage(Message msg) {
                if (msg.what == 0x110) {
                    Log.e(TAG, "收到消息>>" + msg.obj.toString());
                }
            }
        };
        Looper.loop();
    }

}
```
子线程B

```java
public class ThreadB extends Thread {
    private static final String TAG = "ThreadB";
    @Override
    public void run() {
        Log.e(TAG, "send message to threadA");
        Message msg = ThreadA.mHandlerA.obtainMessage();
        msg.what = 0x110;
        msg.obj = "Hello ThreadA, I am ThreadB";
        msg.sendToTarget();
    }
}
```
 线程间通信
 
 ```java
public void testHandlerThread(View view) {
    ThreadA threadA = new ThreadA();
    threadA.start();

    ThreadB threadB = new ThreadB();
    threadB.start();
}

结果Log：
1969-2421/com.wswenyue.handlerdemo E/ThreadB: send message to threadA
1969-2419/com.wswenyue.handlerdemo E/ThreadA: 收到消息>>Hello ThreadA, I am ThreadB    
 ```
 
 通过这个B线程给A线程发送消息的过程，我们来过一遍源码~~~
 
 **Looper.prepare()**
 
```java
#Looper.prepare()--->new Looper 
private static void prepare(boolean quitAllowed) {
    //...
    sThreadLocal.set(new Looper(quitAllowed));
}
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
#new Hanlder 
public Handler(Callback callback, boolean async) {
        //...
        mLooper = Looper.myLooper();//sThreadLocal.get();
        //...
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}
```

**Looper.loop()**

```java
public static void loop() {
    //...
    final MessageQueue queue = me.mQueue;
    //...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }        
        //...
        msg.target.dispatchMessage(msg);
        //...
    }
}
```

**Handler.dispatchMessage()**

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg); //message.callback.run();
    } else {
        if (mCallback != null) {//可以new Handler 传入Callback
            if (mCallback.handleMessage(msg)) {// Handler(Callback callback, boolean async)
                return;
            }
        }
        handleMessage(msg);//子类复写该方法
    }
}  
//Subclasses must implement this to receive messages.    
public void handleMessage(Message msg) { }
```

**MessageQueue.next()**

```java
Message next() {
  //...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        //...
        synchronized (this) {
            //...
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
           //...
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;//返回符合当前时间的消息
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
// Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            //...
        }
    }
```

**Handler send message**

```java
post(Runnable)
postAtTime(Runnable,long)
postDelayed(Runnable,long)

sendMessage(Message)
sendMessageDelayed(Message,long)
sendMessageAtTime(Message,long)
sendEmptyMessage(int)
sendEmpty...

上面所有的方法都会调用到sendMessageAtTime(Message msg, long uptimeMillis)
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue; // mQueue = mLooper.mQueue;
    //...
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;//this 就是当前Handler自己
   //...
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

**MessageQueue.enqueueMessage()**

```java
boolean enqueueMessage(Message msg, long when) {
   //...
    synchronized (this) {
        if (mQuitting) {
            //...
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
            // ...
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                //...
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        //...
    }
    return true;
}

```
上面我把重点涉及到的方法都详细的提出来了，方法体中省略掉了一些不相干的部分，关键部分有注释说明。
其实流程上已经没有可说的了，摘出来的源码这块大家都能看懂。    
说到底其实就是通过它可以轻松地将一个任务切换到Handler所在的线程中去执行，总结起来就是：handler负责发送消息，Looper负责接收Handler发送的消息，并直接把消息回传给Handler自己；
MessageQueue就是一个存储消息的容器。

仔细看源码的人会发现在MessageQueue的next()方法中有一个nativePollOnce(mPtr,nextPollTimeoutMillis)方法调用。好奇的你接着寻找答案发现，这个是一个native方法，查看源码，你会发现，好像Native层也实现了一套Native的Handler。这个mPtr就是存储了Native层的消息队列对象。查阅底层源码Looper.cpp，我们发现了底层Handler机制的实现原理：
![](http://wswenyue.win/talk/handler_files/looper_cpp.png)
底层通信的机制是使用Linux的管道机制。

那么Android为什么要有两套消息机制？思考一下你就会明白：
   
- Android支持纯Native开发，所以需要Native实现一套消息机制
- Android核心组件运行在Native层，它们之间的通信需要消息机制

看完源码，我们来聊聊Android消息机制源码设计的艺术：

从源码可以看出Handler的target是Handler类型，实际就是转了一圈，通过Handler将消息传递给消息队列，消息队列又将消息分发给Handler来处理。这样的设计其实是设计模式中的**命令模式**。     
**Message就是一条命令，Handler就是处理人，通过命令模式将操作与执行者解耦。**      
关于命令模式的定义：    
命令模式是对命令的封装。 命令模式把发出命令的责任和执行命令的责任分割开，委派给不同的对象。 每一个命令都是一个操作：请求的一方发出请求要求执行一个操作；接收的一方收到请求，并执行操作。   

另外一个就是Message的设计，**Message对象池使用了享元模式**    
关于享元模式的定义：是对象池的一种实现。享元模式用来尽可能减少内存的使用量，它适用于可能存在大量重复对象的场景，来缓解可共享的对象，达到对象共享、避免创建过多对象的效果，作用是提示性能。避免内存溢出。    
下面通过简单的图来说明Message对象池

<div><img src="http://wswenyue.win/talk/handler_files/m_1.jpg" height="260" width="400"><img src="http://wswenyue.win/talk/handler_files/m_2.jpg" height="260" width="400"></div>

sPool是一个指针，指向当前可用的Message对象，如果添加一个Message，那么只需要将新的Message的next指针指向之前的可用的Message上，sPool指针指向当前的新Message。插入和删除都在头。这其实算是一个单链表。


了解了Handler、以及消息机制，我们来谈谈Handler使用中的一些问题：

常见的问题有：

- CalledFromWrongThreadException：这种异常是因为尝试在子线程中去更新UI，进而产生异常
- Can't create handle inside thread that ha not called Looper.prepared：是因为我们在子线程中去创建Handler，而产生的异常
- Handler当做内部类，使用不当导致内存泄露的问题

这些问题中前两个都是不理解或者使用不规范造成的，在这里我们不做讲解。现在说说使用Handler造成内存泄漏的问题，我们先来看一段代码：

```java
/**
 * 当activity被finish的时候，延迟发送的消息仍然会存活在UI线程的消息队列中，
 * 直到10分钟后它被处理掉。这个消息持有activity的Handler的引用，
 * Handler又隐式的持有它的外部类(这里就是LeakActivity)的引用。
 * 这个引用会一直存在直到这个消息被处理，所以垃圾回收机制就没法回收这个activity，
 * 内存泄露就发生了.
 * 非静态的匿名类会隐式的持有外部类的引用，所以context会被泄露掉
 * <p/>
 * 在Java中，非静态的内部类或者匿名类会隐式的持有其外部类的引用，而静态的内部类则不会。
 */
public class LeakActivity extends Activity {
    private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            //...
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);

        mLeakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                //...
            }
        }, 1000 * 60 * 10);
        //...
        finish();

    }
}
```
那么我推荐的解决方案如下：

```java
/**
 * 实现Handler的子类或者使用static修饰内部类。
 * 静态的内部类不会持有外部类的引用，所以activity不会被泄露。
 * 如果你要在Handler内调用外部activity类的方法的话，
 * 可以让Handler持有外部activity类的弱引用，这样也不会有泄露activity的风险。
 * <p/>
 * 如果要实例化一个超出activity生命周期的内部类对象，避免使用非静态的内部类。
 * 建议使用静态内部类并且在内部类中持有外部类的弱引用。
 */
public class SecureActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_secure);
        mHandler.postDelayed(sRunnable, 1000 * 60 * 10);
        finish();
    }
    private final SecureHandler mHandler = new SecureHandler(this);
    /**
     * 关于匿名类造成的泄露问题，我们可以用static修饰这个匿名类对象解决这个问题，
     * 因为静态的匿名类也不会持有它外部类的引用。
     */
    private static final Runnable sRunnable = new Runnable() {
        @Override
        public void run() {
            //...
        }
    };
    private static class SecureHandler extends Handler {
        private final WeakReference<SecureActivity> mActivity;
        public SecureHandler(SecureActivity activity) {
            mActivity = new WeakReference<SecureActivity>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            SecureActivity activity = mActivity.get();
            if (activity != null) {
                // ...
            }
        }
    }
}
```

本文关于Handler的讲解不清楚的地方大家可以查看相应的PPT和Demo    
戳我查看：[PPT](http://wswenyue.win/talk/handler.htm)
[Demo](https://github.com/wswenyue/talk/tree/master)