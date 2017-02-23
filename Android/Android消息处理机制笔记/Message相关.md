# Message相关

判断新创建Message处于队列中的位置，并插入相应位置。

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
              	 *	1.p为null时，说明prev是当前消息队列中的最后一条消息(因为p为null，所以prev不为					 *	  null且prev的触发时间小于传入Message的触发时间，所以传入Message的为消息队列中的				 *	  最后一条消息，prev为传入Message的上一条消息)
              	 *	2.p的触发时间大于传入Message的触发时间(因为p的触发时间大于传入Message的触发时					 *	  间，所以p在消息队列中是传入Message的下一条消息，因为在上一次循环中没有进入if语					 *	  句，所以prev不为null且触发时间小于传入Message对象的触发时间，所以prev在消息队列					* 	 中处于传入Message的上一条)
              	 */	
                msg.next = p;
                prev.next = msg;
            }
        }
        return true;
    }
```