# Message是如何触发的

还是ActivityThread这段代码。来自[Android中为什么主线程不会因为Looper.loop()里的死循环阻塞？](http://www.jianshu.com/p/72c44d567640)

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

在上篇文章[MessageQueue与Looper的由来](https://github.com/leibown/Study-Notes/blob/master/Android/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97%E4%B9%8B%E3%80%8C03.MessageQueue%E4%B8%8ELooper%E7%9A%84%E7%94%B1%E6%9D%A5%E3%80%8D%20.md)我们得知这段代码中的`Looper.prepareMainLooper()`在主线程中创建了一个Looper对象，然后在这段代码的末尾处，调用了`Looper.loop()`方法，我们来看看`Looper.loop()`源码：

```java
 //删除部分代码
public static void loop() {
  		//获取当前线程的Looper对象
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
  		//获取Looper对象中的消息队列
        final MessageQueue queue = me.mQueue;
		
   		...

        for (;;) {
          	//不断的从消息队列拿出消息队列的第一条消息，直到没有消息为止
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
	
          	...
          
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

          	...
          
            msg.recycleUnchecked();
        }
    }
```

从这个`loop()`方法中逻辑就比较明了了，里面有一个死循环，从消息队列中不断的取出消息，然后调用这个方法`msg.target.dispatchMessage(msg)`，`msg.tagre`为当初你创建的Handler对象，因为在Handler把消息放入消息队列的时候执行了以下代码:

```java
//Handler类
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

可以看到这里把创建的Handler赋值给了`msg.target`。然后`msg.target.dispatchMessage(msg)`就相当于`handler.dispatchMessage(msg)`,所以我们来看看Handler的dispatchMessage()方法：

```java
 public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
          	//mCallback为当初创建Handler时传入的Callback对象
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

 private static void handleCallback(Message message) {
        message.callback.run();
    }
```

如果看过第一篇[Handler消息发送](https://github.com/leibown/Study-Notes/blob/master/Android/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97%E4%B9%8B%E3%80%8C01.Handler%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E3%80%8D.md)我们就可以了解到Handler是如何发送消息的，所以看到这段代码就应该知道这些消息或者是回调是如何发送的了。

根据自己的理解，绘制了以下流程图，如有错误，请及时提醒我改正：![](https://docs.google.com/drawings/d/1ECMjUaIEk7DLrsMm4YpgxIgoIGIS48TvP3QDHMeQLzQ/pub?w=1739&h=1085)

这个系列是本人写的第一篇比较完成的博客，肯定会有非常多的不足，希望大家能够多留言批评，希望能明确指出文章中可能有的错误，我会及时更正，谢谢。