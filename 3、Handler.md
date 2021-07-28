* 线程中通信
* 主要搞懂从```sendMessage(Message msg)```===>>```public void handleMessage(@NonNull Message msg)```干了啥

### Handler

* sendMessage后调用方法关系

  sendMessage(Message newMessage)-->

  sendMessageDelayed(@NonNull Message msg, long delayMillis)-->

  sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis)-->

  enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,long uptimeMillis)

* 一个```Handler``` 有且只有一个```MessageQueue```

* 每个线程只能创建一个Looper

* 主线程没有必要且不允许手动开启```Loop.prepare```

  ```java
  /**
   * 发送消息的最底层方法
   * 最终调用的是将Message进行入列
   */ 
  public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
      MessageQueue queue = mQueue;
      if (queue == null) {
          RuntimeException e = new RuntimeException(
                  this + " sendMessageAtTime() called with no mQueue");
          Log.w("Looper", e.getMessage(), e);
          return false;
      }
      return enqueueMessage(queue, msg, uptimeMillis);
  }
  ```

####  为什么loop在主线程中不会被卡死？？？

其实死循环与ANR不是一回事，loop的for(;;)会致使Linux挂起。

以下四个条件都可以造成ANR发生：

- **InputDispatching Timeout**：5秒内无法响应屏幕触摸事件或键盘输入事件
- **BroadcastQueue Timeout** ：在执行前台广播（BroadcastReceiver）的`onReceive()`函数时10秒没有处理完成，后台为60秒。
- **Service Timeout** ：前台服务20秒内，后台服务在200秒内没有执行完毕。
- **ContentProvider Timeout** ：ContentProvider的publish在10s内没进行完。

其实相当于10号线绕城转，一直转不会造成ANR，只有在前边一辆上人上的慢，后边车追尾了，这时候才会造成崩溃，ANR.

#### 为什么Handler会造成内存泄漏？？？

handler发送的消息在当前handler的消息队列中，如果此时activity finish掉了，那么消息队列的消息依旧会由handler进行处理，若此时handler声明为内部类（非静态内部类），我们知道内部类天然持有外部类的实例引用，那么就会导致activity无法回收，进而导致activity泄露。

#### 为何handler要定义为static？？？

因为静态内部类不持有外部类的引用，所以使用静态的handler不会导致activity的泄露。

#### 为何handler要定义为static的同时，还要用WeakReference 包裹外部类的对象？？？

如果需要使用外部类的成员，可以通过"activity. "获取变量方法等，如果直接使用强引用，显然会导致activity泄露。当然如果不需要引用外部类成员，就可以不必使用WeakReference。

> 决方案：静态内部类+弱引用



