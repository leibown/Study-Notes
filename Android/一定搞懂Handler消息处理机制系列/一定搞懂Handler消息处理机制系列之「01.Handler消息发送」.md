## 一定搞懂Handler消息处理机制系列之「Handler消息发送」

### Handler消息发送的方式有两种：

- **Post**

  ```java
  public final boolean post(Runnable r){
         return  sendMessageDelayed(getPostMessage(r), 0);
    }
  /**
  @param delayMillis 这个参数表示延迟多少毫秒之后触发回调接口
  */
  public final boolean postDelayed(Runnable r, long delayMillis){
          return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
  /**
  此方法会把产生的Message直接放入消息队列中的首位，官方不建议使用该方法，因为这样操作容易使消息队列匮乏，导致排序问题或具有其他意外的副作用
  */
  public final boolean postAtFrontOfQueue(Runnable r){
          return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
  /**
  @param uptimeMillis 这个参数表示当前时间为uptimeMillis毫秒时触发回调接口
  */
  public final boolean postAtTime(Runnable r, long uptimeMillis){
          return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }
  public final boolean postAtTime(Runnable r, Object token, long uptimeMillis){
          return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
  ```
  我们可以看到，每个Post方法有一个共同的参数，那就是接收了一个Runnable接口，这个参数是用于回调的。每个使用Post发送消息时都去调用了sendMessage的相关方法，每个sendMessage相关方法都需要一个Message对象，所以就拿着Runnable对象去调用getPostMessage方法，

  ```java
  private static Message getPostMessage(Runnable r) {
          Message m = Message.obtain();
          m.callback = r;
          return m;
      };
  private static Message getPostMessage(Runnable r, Object token) {
          Message m = Message.obtain();
          m.obj = token;
          m.callback = r;
          return m;
      };
  ```

  getPostMessage会返回一个Message对象，拿到Message对象去调用相关sendMessage方法。

- **Send**

  ```java
  public final boolean sendMessage(Message msg){
          return sendMessageDelayed(msg, 0);
      };
  public final boolean sendEmptyMessage(int what){
          return sendEmptyMessageDelayed(what, 0);
      };
  /**
  @param uptimeMillis 这个参数表示当前时间为uptimeMillis毫秒时发送此消息
  */
  public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
          Message msg = Message.obtain();
          msg.what = what;
          return sendMessageAtTime(msg, uptimeMillis);
      };
  /**
  @param delayMillis 这个参数表示延迟多少毫秒之后发送此消息
  */
  public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
          Message msg = Message.obtain();
          msg.what = what;
          return sendMessageDelayed(msg, delayMillis);
      };
  public final boolean sendMessageDelayed(Message msg, long delayMillis){
          if (delayMillis < 0) {
              delayMillis = 0;
          }
          return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
      };
  public final boolean sendMessageAtFrontOfQueue(Message msg) {
          MessageQueue queue = mQueue;
          if (queue == null) {
              RuntimeException e = new RuntimeException(
                  this + " sendMessageAtTime() called with no mQueue");
              Log.w("Looper", e.getMessage(), e);
              return false;
          }
          return enqueueMessage(queue, msg, 0);
      };
   public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
          MessageQueue queue = mQueue;
          if (queue == null) {
              RuntimeException e = new RuntimeException(
                      this + " sendMessageAtTime() called with no mQueue");
              Log.w("Looper", e.getMessage(), e);
              return false;
          }
          return enqueueMessage(queue, msg, uptimeMillis);
      };
  ```
  可以看到除了sendMessageAtFrontOfQueue方法，其他方法最终都会产生一个Message对象去调用sendMessageAtTime方法，sendMessageAtTime中获取当前的消息队列，然后enqueueMessage方法把新产生的Message对象放入队列当中。

  ```java
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
          msg.target = this;
          if (mAsynchronous) {
              msg.setAsynchronous(true);
          }
          return queue.enqueueMessage(msg, uptimeMillis);
      }
  ```
  关于如何**Message入列**请移步[一定搞懂Handler消息处理机制系列之「02.Message入列」](https://github.com/leibown/Study-Notes/blob/master/Android/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97/%E4%B8%80%E5%AE%9A%E6%90%9E%E6%87%82Handler%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E7%B3%BB%E5%88%97%E4%B9%8B%E3%80%8C02.Message%E5%85%A5%E5%88%97%E3%80%8D.md)

