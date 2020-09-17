# 深入理解android内核设计思想读书笔记

### 5.1 Android进程和线程

进程是程序的一个运行实例；线程是cpu的基本调度单位

并发实现的操作系统采用分时的方法，为正在运行的多个任务分配合理的、单独的CPU时间片来实现并发

四大组件并不是进程的载体，只是进程的组成部分

**一个只有一个activity的程序，最少有三个Thread（一个main线程，两个binder线程）**

1. main线程是由ActivityThread创建的，两个binder线程一个负责和ams通信，一个负责和wms通信
2. 同理一个之只有service的程序也是一样的，service和activity的主线程都是ActivityThread
3. 如果一个android程序有两个activity，也只有一个main线程，但是会有三个binder线程
4. 一个相同的包里组件默认是同一个进程的，当然也可以通过process属性声明不同的进程

### 5.2 handler，messageQueue，runnable，looper

1. Message和runnable都是可以被压入某个messageQueue中，runnable可以封装成message
2. Looper循环的从MessageQueue取回message
3. Handler真正去处理message

**Looper不断获取MessageQueue中的Message，然后由Handler处理**

#### Handler机制简析

[xref](http://www.aospxref.com/android-10.0.0_r2/xref/): /[frameworks](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/)/[base](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/)/[core](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/)/[java](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/)/[android](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/)/[os](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/os/)/[Handler.java](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/os/Handler.java)

```java
@UnsupportedAppUsage
final Looper mLooper; //每个Thread对应一个looper
final MessageQueue mQueue;//每个looper对应一个MessageQueue，一个MessageQueue里有N个Message，一个Message最多指定一个handler处理，因此Thread与Handler是一对多的关系
@UnsupportedAppUsage
final Callback mCallback;
```

Handler有两个方面的作用：处理Message和将某个Mseeage压入MessageQueue中

##### 处理Message

```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
         handleCallback(msg);  //优先由handlercallback调用
     } else {
         if (mCallback != null) { 
             if (mCallback.handleMessage(msg)) { //其次由mCallback调用
                 return;
             }
         }
         handleMessage(msg); //最后才会调用handleMessage处理Message
     }
 }
```

##### 将Mseeage压入MessageQueue中

post系列：

```java
public final boolean post(@NonNull Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}

public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
//post和send系列方法没有本质的区别，都是将某个消息压入消息队列，区别是post可以接受runnable参数，send接受Message，不过最终post方法都会调用到send方法
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r; //像这样runnable和Message就建立起联系了
    return m;
}
```

send系列：

```java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendEmptyMessage(int what)
{
    return sendEmptyMessageDelayed(what, 0);
}

public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageAtTime(msg, uptimeMillis);
} 

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
} //uptimeMillis时间过后将消息放入消息队列，以在消息循环的下一次迭代中进行处理

public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}//将消息放入消息队列的最前面，以在消息循环的下一次迭代中进行处理
```

无论post还是send方法最终都是执行**MessageQueue**的enqueueMessage方法

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

##### **MessageQueue**简析 

[xref](http://www.aospxref.com/android-10.0.0_r2/xref/): /[frameworks](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/)/[base](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/)/[core](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/)/[java](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/)/[android](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/)/[os](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/os/)/[MessageQueue.](http://www.aospxref.com/android-10.0.0_r2/xref/frameworks/base/core/java/android/os/MessageQueue.java)

```java
//构造方法
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();//本地方法nativeInit()，涉及到JNI调用
}

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
        //如果此队列中头部元素是null(空的队列，一般是第一次)，或者此消息不是延时的消息，则此消息需要被立即处理，此时会将这个消息作为新的头部元素，并将此消息的next指向旧的头部元素，然后判断如果Looper获取消息的线程如果是阻塞状态则唤醒它，让它立刻去拿消息处理
    	msg.next = p; //由此可以看出MessageQueue是链表结构
    	mMessages = msg;
    	needWake = mBlocked;
	} else {
	// Inserted within the middle of the queue.  Usually we don't have to wake
	// up the event queue unless there is a barrier at the head of the queue
	// and the message is the earliest asynchronous message in the queue.
    //插入队列中间。 通常，除非队列的开头有障碍并且消息是队列中最早的异步消息，否则我们不必唤醒事件队列。
    //如果此消息是延时的消息，则将其添加到队列中，原理就是链表的添加新元素，按照when，也就是延迟的时间来插入的，延迟的时间越长，越靠后，这样就得到一条有序的延时消息链表，取出消息的时候，延迟时间越小的，就被先获取了。插入延时消息不需要唤醒Looper线程。
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

//元素出队
Message next() {......}
```

##### Looper

looper包含了一个MessageQueue。

应用程序使用后looper有两种情况，第一种是ActivityThread，第二种是普通线程。

普通线程：

```Java
class LooperThread extend Thread {
    public Handler mHandler;
    public run () {
        Looper.prepare(); //Looper的准备工作
        mHandler = new Handler() {
            public void HandleMessage(Message msg) {
                ......处理消息
            }
        }; //创建处理消息的handler
        Looper.loop(); //进入主循环
    }
}
```

先看Looper.prepare();

```
代码下次补上
```

ThreadLocal是一种特殊的全局变量，只属于自己的线程，外界所有的线程都无法访问到，这也从侧面说明每个线程的Looper是独立的。

ThreadLocal是存放针对当前线程的Looper和其他数据的数据对象的模板

Looper的构造方法

```java
//Looper的构造方法
//Looper和Handler之间的桥梁是MessageQueue
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);//new了一个MessageQueue，Looper创建的时候就创建了MessageQueue
}
```

再来看Looper.loop()；

```java
public static void loop() {
	final Looper me = myLooper();//调用sThreadLocal.get()返回与之匹配的Looper实例
    ...
    final MessageQueue queue = me.mQueue;//Looper中自带一个MessageQueue
    ...
    for (;;) { //消息循环开始
        Message msg = queue.next();//从消息队列中取出一个消息，可能会阻塞。
        if (msg == null) {
            return; //如果消息队列里没有消息，说明线程要推出了
        }
        ...
        msg.target.dispatchMessage(msg); //分派消息，target实际上是Handler，dispatchMessage最终调用的是Handler的处理函数。
        msg.recycle(); //消息处理完毕要回收消息
    }
}
```

loop（）函数的主要工作就是不断从消息队列中取出需要处理的事件，然后分发给相应的负责人。

如果消息队列为空，它很快会进入睡眠以让出CPU资源。

而在具体的事件处理中，程序会post新的事件到队列中，另外，其他进程也很可能会投递新的事件到这个队列中

APK应用程序就是不停的执行“处理队列事件”的工作，直到它退出运行。

### 5.3 UI主线程——ActivityThread

```
代码下次补上
```

主线程和普通线程实际上并没有很大区别，都拥有Looper对象。

普通线程可以获取到主线程的Looper对象，但是主线程无法获取到普通线程的Looper。

ActivityThread通过getHandler（）返回一个 final H mH = new H（）；

mH是整个ActivityThread的事件管家，负责处理主线程的各种消息。

##### main（）函数的经典模型

伪代码实现如下：

```java
main()
{
	initalize(); //必要的初始化工作
	CreateWindows();//创建窗口
	ShowWindows();//显示窗口
	While(GetMessage()){
        //不断获取消息，并不断执行消息，直至推出
		TranslateMessage();//对消息进行必要的前期处理
		DispatchMessage();//分配消息给相应的对象处理
	}
}
//消息是推动整个系统动起来的基础
```

