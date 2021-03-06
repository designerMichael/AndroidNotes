---
title: Handler解析
date: 2016-06-10 21:48:06
categories: 源码解析
tags: [Handler,源码解析]
---
  


在 Android 中，只有主线程才能操作 UI，但是主线程不能进行耗时操作，否则会阻塞线程，产生 ANR 异常，所以常常把耗时操作放到其它子线程进行。如果在子线程中需要更新 UI，一般是通过 Handler 发送消息，主线程接受消息并且进行相应的逻辑处理。除了直接使用 Handler，还可以通过 View 的 post 方法以及 Activity 的 runOnUiThread 方法来更新 UI，当然它们内部也是利用了Handler来实现的。


<!--more-->



# 基本使用

一个最简单的例子：

```java
public class MainActivity extends AppCompatActivity {
    private static final int MESSAGE_TEXT_VIEW = 0;
    
    private TextView mTextView;
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_TEXT_VIEW:
                    mTextView.setText("UI成功更新");
                default:
                    super.handleMessage(msg);
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        mTextView = (TextView) findViewById(R.id.text_view);

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                mHandler.obtainMessage(MESSAGE_TEXT_VIEW).sendToTarget();
            }
        }).start();

    }
}
```

handler可以发送Message对象，也可以发送Runnable对象。

```java

//Message msg = new Message();一般不建议直接new
Message msg = handler.obtainMessage();
msg.what = 1;//msg.arg1/arg2/obj
//有多种sendMessage方法：
handler.sendEmptyMessage(1);
handler.sendMessage(msg);
handler.sendMessageAtTime(msg,123223);
handler.sendMessageDelayed(msg,2000);
```

也可以发送Runnable对象

```java
private Runnable myRunnable = new Runnable() {
    @Override
    public void run() {
        //延迟1秒后又启动这个runnable，就循环了
        handler.postDelayed(this , 1000);
    }
};

handler.post(myRunnable);
handler.postAtTime(myRunnable,32332);
handler.postDelayed(myRunnable,2000);

```




# 子线程创建Looper
主线程系统默认创建了Looper，子线程则要自己创建。
```java
new Thread(new Runnable() {  
        @Override  
        public void run() {  
            Looper.prepare()
            handler2 = new Handler(){
				//重写handleMessage方法。
			}; 
            Looper.loop() 
        }  
    }).start();
```



# HandlerThread

`HandlerThread`继承于`Thread`，只不过它比普通的`Thread`多了一个`Looper`,可以在这个线程中分发和处理消息。

创建`HandlerThread`：
```java
HandlerThread handlerThread = new HandlerThread("jerome");  
handlerThread.start();  
```
然后将HandlerThread创建的looper传递给threadHandler，即完成绑定：

```java
handler = new Handler(handlerThread.getLooper()) {  
  
    @Override  
    public void handleMessage(Message msg) {  
        super.handleMessage(msg);  
		//这儿可以做耗时的操作；  
}
```


# 一些注意点

- `Handler`跟哪个线程绑定，要看`Handler`的`looper`来源哪个线程
调用Handler的无参构造`（Handler mHandler = new Handler()）`默认会用当前线程的`Looper`来构建内部的消息循环机制，如果当前线程没有`Looper`，就会报错。
但是你也可以指定Handler的Looper，如：
```java
Handler mHandler = new Handler(Looper.getMainLooper()){
	    @Override
	    public void handleMessage(Message msg) {
	        ImageView imageView = (ImageView) msg.obj;
	        imageView.setBackgroundColor(Color.BLUE);
	    }
	};
```
你可以在任意线程调用主线程的Looper来创建Handler，这样创建的Handler就与UI线程绑定，可以在里面更新UI。

通俗一点讲就是用线程A的Looper创建了mHandler，在线程B中拿到mHanlder，调用mHandler.sendMessage方法，这个消息就会传到线程A中，在线程A中处理。这样业务逻辑就从线程B切换到线程A中了，而线程A往往是主线程。

- 一个线程可以有多个Handler，但只有一个Looper，一个MessageQueue

- 停止Looper用quit()方法

- 注意是android.os.Handler包下，别导错了。


# 源码解析

## MessageQueue的工作原理
`MessageQueue`主要包含两个主要操作`enqueueMessage`和`next`，前者是插入消息，后者是取出一条消息并移除。`MessageQueue`的内部其实是通过单链表来维护消息列表的，单链表在插入和删除上比较有优势。
`next`方法是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next方法会返回这条消息并将它从链表中移除。
在`enqueueMessage`中，会把 `msg.target `设置为当前的 `Handler` 对象

## Looper的工作原理
Hanlder的工作需要Looper，想要获得 Looper 需要调用` prepare()` 方法，主线程在创建时系统会默认调用，其他的线程就需要我们手动prepare了。源码如下：
```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
解释：
`ThreadLocal`是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。
这里首先判断`sThreadLocal`中是否已经存在`Looper`了，如果还没有则创建一个新的Looper设置进去。同时也可以看出每个线程中最多只会有一个`Looper`对象。

接下来看一下`Looper`的构造，做了两件事：创建一个`MessageQueue`；将当前线程对象保存起来。
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
接下来会调用Looper.loop()方法，只有调用了loop方法消息循环系统才会真正的起作用，源码如下：
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
    
    //无限循环
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
```

这个方法的代码有点长，不去追究细节，只看整体逻辑。可以看出，在这个方法内部有个死循环，里面通过 `MessageQueue `的 `next()` 方法获取下一条消息，没有获取到会阻塞。如果成功获取新消息，便调用 `msg.target.dispatchMessage(msg)`，其中的`msg.target`就是发送这条消息的`Handler`对象。handler的`dispatchMessage`方法会在创建handler时使用的`Looper`中执行，这样就切换了线程。

关于`Looper`还有一些要注意的点:


- 除了`prepare`方法，Looper还有`prepareMainLooper`方法，主要是给主线程也就是`ActivityThread`创建Looper使用的，本质也是调用了`prepare`方法。
- Looper提供了`getMainLooper`，在任何地方都可以获取到主线程的`Looper`。
- `Looper`的`quit`和`quitSafely`方法的区别是：前者会直接退出Looper，后者只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才安全地退出。Looper退出之后，通过Handler发送的消息就会失败，这个时候Handler的send方法会返回false。
- 在子线程中，如果手动为其创建了`Looper`，那么在所有的事情完成以后应该调用quit方法来终止消息循环，否则这个子线程就会一直处于等待的状态，而如果退出Looper以后，这个线程就会立刻终止，因此建议不需要的时候终止Looper。
- `Looper`的`loop`方法是个死循环，唯一跳出循环的方式是MessageQueue的next方法返回null，Looper调用quit后会调用MessageQueue的quit，此时MessageQueue就会被标记为退出状态，MessageQueue的next返回null，loop方法也就停止了。


## Handler的工作原理
首先看一下Handler的构造：
```java
public Handler() {
     this(null, false);
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
```
在构造方法中，通过调用` Looper.myLooper() `获得了 Looper 对象。如果 `mLooper `为空，那么会抛出异常：`"Can't create handler inside thread that has not called Looper.prepare()"`，意思是：不能在未调用 `Looper.prepare()` 的线程创建 `handler`。

看一下`Looper.myLooper()`源码：
```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
代码很简单，就是从 `sThreadLocal` 中获取 Looper 对象。

在得到`Looper`对象之后，又获取了它的内部变量 `mQueue`， 这是 `MessageQueue `对象，也就是消息队列，用于保存 Handler 发送的消息。


`Handler`的工作主要包含：消息的发送和接收之后的处理。
发送有多种方式：`sendMessage(Message msg)`；`sendMessageDelayed(Message msg，long delayMillis)`;`sendMessageAtTime(Messgae msg ,long uptimeMills)`
最终都会调用`enqueueMessage`方法向`MessageQueue`中插入这条消息。`MessageQueue`的next方法就会返回消息给Looper，Looper最后交给Handler处理，即`dispatchMessage`方法会被调用，实现如下：
```java
public void dispatchMessage(Message msg) {
if (msg.callback != null) {
    handleCallback(msg);//当message是runnable的情况，也就是Handler的post方法传递的参数，这种情况下直接执行runnable的run方法
} else {
    if (mCallback != null) {//如果创建Handler的时候是给Handler设置了Callback接口的实现，那么此时调用该实现的handleMessage方法
        if (mCallback.handleMessage(msg)) {
            return;
        }
    }
    handleMessage(msg);//如果是派生Handler的子类，就要重写handleMessage方法，那么此时就是调用子类实现的handleMessage方法
}
}
private static void handleCallback(Message message) {
    message.callback.run();
}
/**
* Subclasses must implement this to receive messages.
*/
public void handleMessage(Message msg) {
}
```
注释讲的很清楚了。

Handler还有一个特殊的构造方法，它可以通过特定的Looper来创建Handler
```
public Handler(Looper looper){
    this(looper, null, false);
}
```

# 更多源码解析文章

<http://www.feeyan.cn/?p=17>

<http://www.jianshu.com/p/e04698eaba88>

<http://www.jianshu.com/p/1e5640e6bef9>

<http://blog.csdn.net/guolin_blog/article/details/9991569>

<https://segmentfault.com/a/1190000004866028>