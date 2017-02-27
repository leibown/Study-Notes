# MessageQueue与Looper的由来

前面两篇文章分别讲了Handler的消息发送和Message入列、Message的创建和Message在队列中的存在形式，那么MessageQueue是怎么来的？因为我们在创建Handler和发送Message时并没有创建MessageQueue，那这个消息队列从何而来呢？上源码:

```java
//省略了部分代码
public Handler(Callback callback, boolean async) {
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

以上为Handler的构造方法，Handler的构造方法共有7个，大多数构造方法都指向了这个两个参数的构造方法。从这个构造方法中，我们看到了`mQueue = mLooper.mQueue`说明消息队列MessageQueue为Looper的一个成员变量，mLooper是有Looper类中的静态方法`Looper.myLooper()`获取:

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

在`Looper.myLooper()`方法中调用`sThreadLocal.get()`，源码继续：

```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```

看到这里其实我是懵逼的(相信大家也都有点懵逼吧)。在这个方法中去获取了当前的线程，看到这里我们至少知道Looper对象是和线程相关的。再来看看这段代码:

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

在`Looper.prepare()`方法中我们看到了`sThreadLocal.set(new Looper(quitAllowed));`，结合上面的`sThreadLocal.get()`至少可以看出一个是放入，另一个方法是取出。那为什么我们在没有调用`Looper.prepare()`方法时还能拿到Looper对象呢？准确的说在主线程为什么我们能拿到Looper对象呢。为此查了下资料。以下资料来自[Android中为什么主线程不会因为Looper.loop()里的死循环阻塞？](http://www.jianshu.com/p/72c44d567640)

> 我们知道APP的入口是在ActivityThread,一个Java类,有着main方法,而且main方法中的代码也不是很多.

```java
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

从上述代码中可以看出，APP在启动时就会在主线程中调用Looper.prepareMainLooper()，所以在主线程中一开始就存在着Looper对象，从Looper.prepare()方法中可以看出，一个线程中只能存在一个Looper对象，所以我们在主线程发送消息时并不需要自己创建新的Looper对象，但是在工作线程中使用Handler就必须手动调用`Looper.prepare()`方法。这下我们知道了Looper的来源，MessageQueue的由来也非常明确了。

```java
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

在`Looper.prepare()`方法中调用了Looper的构造方法，而就在Looper的构造方法中创建了MessageQueue。