### 引言
Handler是Android中线程间通信的一种机制，它在APP的整个生命周期过程中都发挥着重要的作用，是Android中最重要的机制之一。
无论是初中高级程序员，它都是面试必问项。同时它也是基础中的基础，必须掌握。

### 原理总结
Handler消息通信过程中主要涉及以下几个对象：Handler、Looper、Message、MessageQueue
- Handler主要负责消息的发送和处理，通过sendMessage()、sendMessageDelayed()等方法把message发送到MessageQueue
中，同时在handleMessage回调中拿到Looper从MessageQueue中取出的消息进行处理
- Looper开启了一个死循环，不断的从MessageQueue中取出消息让Handler处理。一个线程对应一个Looper，同时一个Looper对应
一个MessageQueue
- Message就是消息，存放我们要处理的任务信息
- MessageQueue是消息队列，它通过链表的方式存放message，并且插入链表的消息都是根据时间进行排序的

### 源码
#### Handler部分

带着几个问题去看源码
- 我们知道在主线程中可以直接创建Handler，那么在子线程呢？
- 发送消息时都是怎么创建message的？
- handler发送消息的方式有哪些？
- 分发消息的时候为什么可以发送到指定线程？

首先从创建handler开始，代码如下，注意注释信息。
```
    /**
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     * 如果当前线程没有一个looper，则就不能收到消息，并且会抛出一个异常
     */
    public Handler() {}
    
    /**
     * Use the provided {@link Looper} instead of the default one.
     * 用传入的looper代替默认的looper
     * @param looper The looper, must not be null. 不可以为null
     */
    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }
```
看代码知道创建handler之前必须要有一个looper才可以。其它几个构造方法也是一样，注意传入looper参数的构造方法，在子线程创建
handler的时候我们可以用这个，并且looper一定不要为null，否则就会抛出异常了。

下面看一下message创建部分，开始学习的时候我们都是通过new Message()来创建message。它还有另外一种获取方式就是通过
obtainMessage(),代码如下
```
    /**
     * Returns a new {@link android.os.Message Message} from the global message pool. More efficient than
     * creating and allocating new instances. The retrieved message has its handler set to this instance (Message.target == this).
     *  If you don't want that facility, just call Message.obtain() instead.
     * 会从message池中获取一个message返回，比直接创建更高效
     */
    @NonNull
    public final Message obtainMessage()
    {
        return Message.obtain(this);
    }
```
其它获取的方法与此类似，只不过加了一些参数条件。这儿是享元模式的运用，通过创建一个缓存池来重复利用message，避免了频繁重复
创建message的消耗及内存抖动的发生。

handler发送消息的方式有sendMessage(@NonNull Message msg)、sendEmptyMessage(int what)、
sendMessageDelayed(Message msg, long delayMillis)、sendMessageAtTime(Message msg, long uptimeMillis)、
sendMessageAtFrontOfQueue(@NonNull Message msg)等等方式，虽然各自的参数不同，但是最终都会走到enqueueMessage()
方法，稍微看下代码
```
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        //这儿给target赋值，this即当前handler对象，这也是为什么消息能发送到正确的线程
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
最后看一下removeCallbacksAndMessages()方法，代码如下
```
    /**
     * Remove any pending posts of callbacks and sent messages whose
     * <var>obj</var> is <var>token</var>.  If <var>token</var> is null,
     * all callbacks and messages will be removed.
     * 移除所有对象为token的待处理的回调和已发送消息，如果传入的token为null，则移除所有
     */
    public final void removeCallbacksAndMessages(@Nullable Object token) {
        mQueue.removeCallbacksAndMessages(this, token);
    }
```
#### Looper部分

问题：
- 为什么不可以在外部直接new Looper()对象？
- 一个线程可以有几个Looper，几个MessageQueue，怎么保证？
- 怎么停止looper循环，有什么区别？

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
首先看构造方法，如上代码所示，其为私有，所以我们不能通过new对象的方式在外部调用，只能通过静态方法prepare()来获得Looper
代码如下
```
    /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      * 注意后面部分：调用prepare()之后，不要忘记调用loop()开启循环，并且可以通过quit()来停止
      */
    public static void prepare() {
        prepare(true);
    }
    
    private static void prepare(boolean quitAllowed) {
        //做校验，一个线程只能创建一个looper
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //通过ThreadLocal来存储当前线程的looper对象
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
可以看到Looper中通过ThreadLocal这么一个线程本地存储机制来保证一个线程只能有一个Looper，而Looper创建时同时创建了一个
MessageQueue，所以一个Looper也对应一个MessageQueue。

接下来看一下Looper开启循环后都做了什么，省略一些不关心的部分
```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper(); //获取Looper
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue; //通过Looper获取MessageQueue

        ...

        for (;;) { //死循环
            Message msg = queue.next(); // might block
            if (msg == null) { //如果没有消息就返回
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            // 这儿发现如果printer不为null就会打印一个log，每个消息都会走这一段逻辑。
            // 利用这个机制我们可以查看每个消息的耗时
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            
            ...
            
            try {
                //消息分发 msg.target获取到handler发送到指定线程
                msg.target.dispatchMessage(msg); 
                ...
            } catch (Exception exception) {
                ...
                throw exception;
            } finally {
                ...
            }
            ...

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            ...
            // 结束后对msg进行回收
            msg.recycleUnchecked();
        }
    }
```
最后看一下停止方法的区别
```
    /**
     * Causes the {@link #loop} method to terminate without processing any
     * more messages in the message queue.
     * 调用quit后就停止了，不在处理消息队列中的消息
     *
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * 调用quit后就无法在消息队列中插入消息了
     *
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * 调用quit停止可能会不安全，可能会有一些消息还未发送，可以通过调用quitSafely保证所有消息都按顺序处理完成
     */
    public void quit() {
        mQueue.quit(false);
    }

    /**
     * Causes the {@link #loop} method to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * However pending delayed messages with due times in the future will not be
     * delivered before the loop terminates.
     * 调用quitSafely不会立马停止，会将当前消息队列中的消息发送完之后再停止
     * 
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * 同上
     */
    public void quitSafely() {
        mQueue.quit(true);
    }
```
#### Message部分

问题：
- message是通过什么方式进行复用的？

Message就是一个存储消息的对象，我们主要看下它是怎么复用的
```
    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     * 从全局缓存池中返回一个message对象实例，如果没有才会创建对象。让我们避免了重复创建对象
     */
    public static Message obtain() {
        // 这儿通过加锁，保证了多线程的安全
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    @UnsupportedAppUsage
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details. 清空msg中的信息
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
            // 如果缓存池的容量小于50就放入，它是一个链表结构
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
    
```
#### MessageQueue部分

问题：
- looper是怎么从MessageQueue中取出消息的？
- MessageQueue中没有消息时为什么不会卡死？

从取出消息开始，代码如下
```
    @UnsupportedAppUsage
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // 如果消息循环已经停止或者调用了disposed()方法就返回null。调用disposed后mPtr会重置为0
        // This can happen if the application tries to restart a looper after quit
        // which is not supported. 
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0; // ‘休眠’时间
        for (;;) { //死循环
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //调用native方法，底层采用linux的epoll机制，当有新的消息插入时会通过nativeWake方法唤醒，所以并不会卡死
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) { //处理异步消息
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        // 下一个消息还没有到执行时间，计算继续等待时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message. 获取message
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages. -1说明没有消息需要处理了，通过nativePollOnce进行休眠直到有消息插入
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
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

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            // 有时候我们为了不占用主线程时间，会添加一个IdleHander接口回调，当没有消息需要处理时就会遍历这些接口
            // 回调其中的quueIdle()方法，在里面我们可以处理自己的一些任务
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                // queueIdle()有一个布尔返回值，如果返回true，会在下次空闲时继续回调。如果返回false，则会从
                // Idlehandler接口集合中移除，也就是说只会回调一次
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
需要注意的点已经加了注释，不再啰嗦，接下来看一个平时不太注意的方法，代码如下
```
    /**
     * Posts a synchronization barrier to the Looper's message queue.
     *
     * Message processing occurs as usual until the message queue encounters the
     * synchronization barrier that has been posted.  When the barrier is encountered,
     * later synchronous messages in the queue are stalled (prevented from being executed)
     * until the barrier is released by calling {@link #removeSyncBarrier} and specifying
     * the token that identifies the synchronization barrier.
     *
     * This method is used to immediately postpone execution of all subsequently posted
     * synchronous messages until a condition is met that releases the barrier.
     * Asynchronous messages (see {@link Message#isAsynchronous} are exempt from the barrier
     * and continue to be processed as usual.
     *
     * This call must be always matched by a call to {@link #removeSyncBarrier} with
     * the same token to ensure that the message queue resumes normal operation.
     * Otherwise the application will probably hang!
     *
     * @return A token that uniquely identifies the barrier.  This token must be
     * passed to {@link #removeSyncBarrier} to release the barrier.
     *
     * @hide 这个方法是隐藏的，但是我们可以通过调用有参数的方法加入同步屏障消息
     */
    @TestApi
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }
```
官方注释很详细，这个方法就是加入一个同步屏障消息。在遇到屏障之前，message消息处理跟往常一样。当遇到同步屏障后，同步消息
暂停处理，优先处理异步消息，直到通过removeSyncBarrier移除同步屏障。

### 感谢读者
感谢所有的读者，如果有什么理解错误，欢迎指正，我将非常感激。

### 告诫自己
这只是写这篇文章时对Handler源码的理解，并且记录了一些人为较为重要的点，肯定不能包含所有，可能之后阅读时认为的遗漏在当时认为
没有必要写出来。有疑问时还是要查阅官方源码。**尽信书不如无书**。