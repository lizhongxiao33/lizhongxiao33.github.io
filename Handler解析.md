[toc]

## Hnadler机制基本流程：
![image][pic1]


## Handler机制相关类

类的基本作用
· Handler：用于发送也接收消息
· Message：消息实体
· Looper：用于轮询消息队列，一个线程只有一个Looper
· MessageQueue：消息队列，用于存储消息和管理消息

### Handler

先来看一下Handler的构造方法，根据是否传入Looper，最后会执行到两个不同的构造方法

构造1：
```java

 public Handler(@Nullable Callback callback, boolean async) {
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
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async; //设置是否同步处理消息，默认为false
    }

```
该构造中因为没有传入Looper，所以要从Looper中的ThreadLocal<Looper>静态常量中获取获取当前线程的Looper（ThreadLocal中的ThreadLocalMap保存着Looper），如果当前线程没有Looper，或者说当前线程的Looper没有prepare，则直接抛出异常。

构造2：
```java
  public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```

Handler各种发送message方法最后都会归并到enqueueMessage()函数中，将message加入队列中
```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this; 此处Handler会将自身设置为Message的target参数变量
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) { //判断handler处理方式，设置Message的flag
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
构建Message的obtainMessage方法中使用Message的obtain方法初始化Message相关参数，此处同样会将Handler自身传入Message中，这也是为什么Handler会造成内存泄漏的地方，因为Message持有了handler，如果当Message被设置为10分钟后执行则持有Handler的对象10分钟内则无法被回收。

```java
 public final Message obtainMessage(int what, int arg1, int arg2, @Nullable Object obj) {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
```

处理接收数据的handleMessage
```java
public void dispatchMessage(@NonNull Message msg) {
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

Handler中有一个静态内部类BlockingRunnable
```java
private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }

        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {
                return false;
            }

            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                        try {
                            wait(delay);//限时等待
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                        try {
                            wait(); //无限期等待
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
```

用于实现Handler发送同步消息，既发送消息的线程在发送消息后处于休眠状态，等待消息回调后唤醒。
在 Runnable 执行完前会通过调用 wait()方法来使发送者线程转为阻塞等待状态，当任务执行完毕后再通过notifyAll()来唤醒发送者线程，从而实现了在 Runnable 被执行完之前发送者线程都会一直处于等待状态

该Runnable只在runWithScissors方法中使用，而runWithScissors是隐藏函数(@hide)

此处的等待要和后续提到的Looper等待区分开

### Message
```java
public final class Message implements Parcelable {
	
	public int what;
	public int arg1;
	public int arg2;
	public Object obj;
    public Messenger replyTo;
	public static final int UID_NONE = -1;
	public int sendingUid = UID_NONE;
	public int workSourceUid = UID_NONE;
	static final int FLAG_IN_USE = 1 << 0;
	static final int FLAG_ASYNCHRONOUS = 1 << 1;
	static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;
	@UnsupportedAppUsage  //不受限制的灰名单
	int flags;
	@UnsupportedAppUsage
    @VisibleForTesting(visibility = VisibleForTesting.Visibility.PACKAGE)
    public long when;
	Bundle data;
	@UnsupportedAppUsage
    Handler target;
	@UnsupportedAppUsage
    Runnable callback;
    @UnsupportedAppUsage
    Message next;
    /** @hide */
    public static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;
	private static final int MAX_POOL_SIZE = 50;//链表最大值
	private static boolean gCheckRecycle = true;

	...

}
```
Message类就是消息载体，本质可以理解为一个单链表结构的节点，实现了Parcelable接口，使得它可以序列化和反序列化。

```java
 	public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0;
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    ......

 	@UnsupportedAppUsage
    void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {//当前链表size小于最大值时将回收的Message放入pool
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```
Message的创建会先从回收的pool中获取，如果pool为空则新建一个。
Message自身实现了回收方法，用于Message对象的回收再利用，防止内存溢出、内存抖动（享元模式）

### Looper
```java
public final class Looper {
 	private static final String TAG = "Looper";

    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    @UnsupportedAppUsage
    private static Looper sMainLooper;//主线程的Looper
    private static Observer sObserver;

    @UnsupportedAppUsage
    final MessageQueue mQueue; 消息队列
    final Thread mThread;

    @UnsupportedAppUsage
    private Printer mLogging;
    private long mTraceTag;

    //如果设置了此选项，则如果消息调度的时间超过此值，循环器将显示警告日志。
    private long mSlowDispatchThresholdMs;

    //如果设置，如果消息传递（实际传递时间-发布时间）花费的时间超过此时间，循环器将显示警告日志。
    private long mSlowDeliveryThresholdMs;

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    //此方法创建主线的looper，如果重复调用会抛出异常，
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
	
	......
    
	private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
	
	......
    
	 public void quit() {
        mQueue.quit(false);
    }

	public void quitSafely() {
        mQueue.quit(true);
    }
}
```
可以看到Looper的构造方法是私有的，创建Looper需要通过prepare方法，quitAllowed字段用于标识消息队列是否可以主动清空操作。
此外，sThreadLocal和sMainLooper都是静态常量，可以理解为全局只有一个sThreadLocal和sMainLooper，prepareMainLooper方法会在ActivityThread中被调用，所以正常使用时调用prepareMainLooper会抛出异常，保证主线程Looper的唯一性。

接下来看一下Looper的核心工作
```java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) { //首先判断Looper是否初始化
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        ......

        for (;;) { //死循环从队列里获取消息
            Message msg = queue.next(); // might block
            if (msg == null) {//如果msg为空说明queue调用了上述的quit方法，队列已经被清空，退出循环遍历。
                return;
            }

            ......

            final Observer observer = sObserver;

            final long traceTag = me.mTraceTag; 跟踪标识，将跟踪事件写入系统跟踪缓冲区。这些跟踪事件可以使用Systrace工具进行收集和可视化
            
			......

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            Object token = null;
            if (observer != null) {
                token = observer.messageDispatchStarting();
            }
            long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
            try {
                msg.target.dispatchMessage(msg); //此处的target就是上述提到的msg构建时将自身传入msg的handler。
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } catch (Exception exception) {
                if (observer != null) {
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
           

           .....

           msg.recycleUnchecked();//回收msg对象，具体实现已在上述Message介绍中给出。
        }
    }
```

### MessageQueue

可以看到Looper只是循环的拿数据分发出去，而消息的各种处理，比如延时消息处理，同步屏障处理等则主要在MessageQueue中实现

首先看一下上面提到的入队函数enqueueMessage
```java
 boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {//没有绑定handler，抛出异常
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {//该message对象已经入队，抛出异常
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {//消息队列已经quit，代表以及无法处理消息，抛出异常
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake; //是否唤起
            if (p == null || when == 0 || when < p.when) {//如果原本Message链表队头为空，或者when=0，或者when小于当前队头，则直接将入队的message设为队头。
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; //mBlocked是否阻塞了Looper的轮询，会在后续的next方法中设置为true/false，如果为true则说明Looper需要唤醒
            } else {   //根据when进行拆入排序
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
                msg.next = p;
                prev.next = msg;
            }
            if (needWake) {
                nativeWake(mPtr); //唤醒Looper，mPtr是native层中MessageQueue对象的引用地址
            }
        }
        return true;
    }

	//android_os_MessageQueue.cpp中nativeWake的调用流程
	static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    	NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    	nativeMessageQueue->wake();
	}

    void NativeMessageQueue::wake() {
    	mLooper->wake();
	}

    //Looper.cpp
    void Looper::wake() {//往mWakeEventFd中write 1唤醒
    	uint64_t inc = 1;
    	ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
	}
```

enqueueMessage方法主要实现了消息的插入排序后入队，此处还会根据needWake字段，判断是否调用native的方法唤醒Looper，而Looper的休眠则是在后面next方法中nativePollOnce中实现。

```java
Message next() {
        final long ptr = mPtr;
        if (ptr == 0) { //quit后会将mPtr赋值为0
            return null;
        }

        int pendingIdleHandlerCount = -1;
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
			//nextPollTimeoutMillis为等待事件，-1时则无限等待
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 如果队头Message得到target为null，遍历队列，如果存在同步消息则立刻执行（同步屏障）
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {//如果消息还没到时间则计算出时间差，循环到上面的nativePollOnce方法让Looper开始等待
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    nextPollTimeoutMillis = -1;
                }

                // 此处dispose会将mPtr置为0
                if (mQuitting) {
                    dispose();
                    return null;
                }

                //后面主要是在没消息空闲时设置IdleHandler了的逻辑
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) { //没有消息也没设置IdleHandler则阻塞
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
                    Log.wtf(TAG, "IdleHandler threw exception", t);
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

	//android_os_MessageQueue.cpp
	static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    	NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    	nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
	}

    void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    	......
    	mLooper->pollOnce(timeoutMillis);
        ......
	}

	//Looper.cpp
    int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    	......
        result = pollInner(timeoutMillis);
    	......
    }


    int Looper::pollInner(int timeoutMillis) {
		......
        //使用epoll_wait进行等待，timeoutMillis就是等待的时间
  		int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
		......
    }
```

在这个next方法中主要做了一下几点事情：
· 获取一个Message，如果存在同步屏障（target为null的Message）则说明存在需要立刻执行的Message，遍历找到消息后立刻执行，具体插入屏障的方法postSyncBarrier（应用：view的刷新ViewRootImpl.java）。
· 遍历完取得一个Message后计算消息是否已经到了可执行时间，没有则计算出nextPollTimeoutMillis，利用nativePollOnce方法让Looper进行等待。
· 如果当前没有消息处理，则判断是否有IdleHandler，IdleHandler 主要用在我们希望能够在当前线程 消息队列空闲时 做些事情（例如UI线程在显示完成后，如果线程空闲我们就可以提前准备其他内容）的情况下，不过最好不要做耗时操作。
使用场景：
```java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                //do somethings
            	 return false;
            }
});
```
· 如果已经调用quit，调用dispose方法销毁native中的MessageQueue
```java
	private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }
```

## 思考

· 一个线程有几个 Looper？
· Handler 内存泄漏原因？
· Looper死循环为什么不会导致应用卡死？（Choreographer.java）
· HnadlerThread是什么？
创建子线程的Handler：
```java
	Handler handler; //全局变量
    private void testThread(){
        Thread thread = new Thread(new Runnable() {
            Looper looper;
            @Override
            public void run() {
                Looper.prepare();
                looper = Looper.myLooper();
                handler = new Handler(looper);
                Looper.loop();
            }
        });
        thread.start();
        //Handler handler = new Handler(thread.getLooper()); //Thread并没有getLooper()
    }

    private void testHandlerThread(){
        HandlerThread handlerThread = new HandlerThread("testHandlerThread");
        handlerThread.start();
        Handler handler = new Handler(handlerThread.getLooper());
        handler.post(new Runnable() {
            @Override
            public void run() {
               //do something
            }
        });
    }

```
优点：
· 方便初始化，方便获取Looper
· 保证线程安全

[pic1]:data:image/png;base64,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3Bt5vBExt+PlmbfW0rqOhrrBzxdmr8ltBq9Vt36/TIVOz9UqnQqeit5VV10lb7zxxpyq3ttvvy379u0Lm77++usRvfpgolepVG7wPO+mIAhuqtfrl/SbI46DwDAEen1eDNMm58yPQFo3qPlFmczZ4+PjS6anp0/F0TpzOw6K+W0jreuo8KK3YcMGmZqakomJCRkdHQ1nxKOPPhoK4EsvvRT+38DW7z/xxBPhMevWrWvJoZFI/f4VV1whjzzySLgU/MADD8hzzz0XHq9SeeONN7baN+3cdtttcscdd4TfV8HUP7/77rthO9dcc428//77rZ/b/VxyySVhnBqzC0u3q1atuvL8+fM3eZ73ZRH5782lVR9QEvN7SRJ5VgS4GWZFvnO/ad2g3Bt5+Cv5ZjzPO9JoNI4sWLDg5SNHjkwPGydze1hyxTgvreuo8KKngvf666/LO++8EwqVypYu295///3y5JNPtkRv//794cwxsqZ/13PWr18fSpwtinqc+bmRODPtTBVSE2iqinruyMhIuEy8ZcuWUCzNkvKqVatacWmVUY9VibTjvOuuu0TbHfR5wl6V0V6TbGxs7PIFCxbcJCI3i8hn2l1aiF4xPnBcHkWaN0NzXZrr1Oai//DSr+g1Hwc7+7NC/3Gn1/vTTz/d9vniOPqbbxu9Pjvm277L55v52IzxnIgcCYLgVc/zflKv1388SOxxzO1O96JB4hjkWLu/6Lzt1o65tvQYLZYsXLjwgsP1Gvvxj3/cKnIMElcej03rOiqF6OkEuPfee+W+++6TZ599Vi677LJQ6Exlb+XKlXOqc2bCmKre448/Hlb6bNEy1bmxsbELPoztyp22pefpV/SD275g9M8qn/aXqeo9+OCDqYmeLks0l2V/z/O863pdPIheL0L8fL4E4rgZ9hsDotcfqbRuUP1Fk+5REdGb07nnece10iciz33sYx976oUXXuj6DHMccztvonf06NE5K2AGoF573/nOd+TnP/95eK82K3DpZjfd3tK6jkohejphVOouvvhimZmZaVXnoqJ38803d3xeTzdw6FLta6+9NudfG2a5VZdoN23aFB6jX7rTV7/079puP6Jnqo7RqZb00q3nef8gCIIve573e0EQ3DLIVEf0BqHFscMQiONm2G+/iF5/pNK6QfUXTbpHdRO9SCS/DILgiO/73xeRp2q12jvR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