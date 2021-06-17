# Java部分

## 多线程

### Java部分

***

### Android部分

***

#### Handler

1. Handler

   * Handler是用来将Message和Runnable对象向Looper中的MessageQueue队列中发送，并处理事件；
   * 创建Handler时会绑定到当前线程的Looper或者指定一个Looper；  
   * Message和Runnable对象在Looper所属线程中处理。

2. Looper

   * Looper是用于为线程运行处理消息的循环；  
   * 默认情况下Looper没有与线程相关联，需要调用Looper.prepare()方法为当前线程提供绑定，调用Looper.loop()方法开始循环
   * 大多数与消息循环的交互是通过Handler进行的；
   * Looper运行时无限遍历MessageQueue队列来处理循环的。

3. MessageQueue

   * MessageQueue是保存需要Looper处理的消息列表；
   * 消息不会直接添加MessageQueue的，而是通过Handler向其所属的Looper中MessageQueue添加的；
   * MessageQueue是通过链表来保存的。

4. HandlerThread

   * HandlerThread是创建一个带Looper的线程，Looper所属HandlerThread的线程；  

   * 使用方法  

     ```java
     HandlerThread handlerThread = new HandlerThread("HandlerThread-1");
     handlerThread.start(); // 必须调用start方法
     Handler handler = new Handler(handlerThread.getLooper());
     handler.post(() -> {
     	appendLog(Thread.currentThread().getName()); // HandlerThread-1线程
     });
     ```

     1. 创建一个HandlerThread对象
     2. 启动HandlerThread（<font color="RED">** 必须要先调用start()方法 **</font>）
     3. 创建一个Handler对象
     4. 之后的操作在handler中进行，handler post的线程就是在HandlerThread下运行

5. ThreadLocal

   * ThreadLocal是给当前线程提供局部变量，每个线程包含一个ThreadLocalMap，ThreadLocal则保存在ThreadLocalMap的Entry[]数组中；  
   * 通过ThreadLocal变量的set()、get()方法可以获取和修改当前线程的ThreadLocal中的值。
***

