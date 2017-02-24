# Message入列

- **判断新创建Message处于队列中的位置，并插入相应位置**

```java
//截取自MessageQueue.enqueueMessage()方法来举例(删除了部分与此次无关代码)
boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
          	//标记传入的msg被使用
            msg.markInUse();
            msg.when = when;
          	//创建临时变量来储存消息队列中的Message对象
            Message p = mMessages;
            boolean needWake;
          	   /*	
          		* 当消息队列中没有消息
          		* 或传入Message的触发时间为0时
                * 或传入Message的触发时间小于当前消息队列中的Message的触发时间
                */
            if (p == null || when == 0 || when < p.when) {
                //把传入的Message放入当前消息队列中的Message之前
                msg.next = p;
              	//把当前消息队列中的Message对象重置为传入的Message对象
                mMessages = msg;
                needWake = mBlocked;
            }  else {
               /*	
          		* 当消息队列中有消息
          		* 且传入Message的触发时间不为0时
                * 且传入Message的触发时间大于当前消息队列中的Message的触发时间
                */
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
            	//创建一个临时变量
                Message prev;
                for (;;) {
                  	//储存临时变量p(当前消息队列中的Messge)
                    prev = p;
                  	//让p指向自己在消息队列中的下一条消息
                    p = p.next;
                  	//当p为null时，说明prev是当前消息队列中的最后一条消息
                  	//或者传入Message的触发时间小于p的触发时间时终止循环
                    if (p == null || when < p.when) {
                        break;
                    }
                }
              	/* 此时的p满足以下两个条件中的一个：
              	 *	1.p为null时，说明prev是当前消息队列中的最后一条消息(因为p为null，所以prev不为
              	 null且prev的触发时间小于传入Message的触发时间，所以传入Message的为消息队列中的最
              	 后一条消息，prev为传入Message的上一条消息)
              	 *	2.p的触发时间大于传入Message的触发时间(因为p的触发时间大于传入Message的触发时
              	 间，所以p在消息队列中是传入Message的下一条消息，因为在上一次循环中没有进入if语句，
              	 所以prev不为null且触发时间小于传入Message对象的触发时间，所以prev在消息队列中处于
              	 传入Message的上一条)
              	 */	
                msg.next = p;
                prev.next = msg;
            }
        }
        return true;
    }
```
- #### **Message的获取方式**

  Message的获取方式除了new Message这种方式，Message类还提供了obtain方法来获取Message

  ```java
  //Message类中有一个静态全局变量来储存空闲或者回收的Message对象
  private static Message sPool;
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
  ```

  这种方式是把静态全局变量*sPool*(这里可以把这个Message看做当前消息池中的第一条消息)标记为未使用然后返回，如果sPool为null才会创建新的Message对象，这样不会造成资源的浪费，避免创建太多Message对象。关于为什么*sPool*会是被回收的Message对象，上源码:

  ```java
    	//此方法为Message的回收方法
  	public void recycle() {
        	//在回收的方法
          recycleUnchecked();
      };
  	void recycleUnchecked() {
        	//这里在做一些重置的工作
          flags = FLAG_IN_USE;
          what = 0;
          arg1 = 0;
          arg2 = 0;
          obj = null;
          replyTo = null;
          sendingUid = -1;
          when = 0;
          target = null;
          callback = null;
          data = null;
        
          synchronized (sPoolSync) {
            //当消息池里消息的数量小于消息池的最大容量时
              if (sPoolSize < MAX_POOL_SIZE) {
                	//重点！！！
                	/*把当前消息池中第一条消息（也就是sPool）置为当前消息的下一条消息(sPool为
                全局静态变量，所有Message都共用这一个sPool)*/
                  next = sPool;
                	/*把当前消息置为消息池中第一条消息(因为上一步骤已经把原来消息池中的第一条消息置为
                了当前消息的下一条消息，现在把当前消息置为消息池中的第一条消息，所以sPool永远代表
                消息池中的第一条消息)*/
                  sPool = this;
                  sPoolSize++;
              }
          }
      }
  ```

  可以看出Message在回收过程中，只要消息池的数量小于消息池的最大容量时，就是把当前Message放入消息池中。

- #### **Message在MessageQueue队列中存在的形式**

  从Message入列方式我们也看出，再有新消息进入队列时，是先判断新消息的触发时间，找出消息应该插入消息队列的位置，把这个位置的消息的next置为本条新消息，然后把新消息的next置为这个位置的消息的下一条消息。类似以下结构(如果我理解有错，欢迎指出)。

  ![message](https://raw.githubusercontent.com/leibown/Study-Notes/master/img/message.png)
