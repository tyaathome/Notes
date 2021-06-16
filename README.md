# Java部分

## 多线程

1. Java部分

2. Android部分

   1. Handler

   2. Looper

   3. MessageQueue

   4. HandlerThread

      * HandlerThread是一个带Looper的线程；  

      * 使用方法  

        ```java
        HandlerThread handlerThread = new HandlerThread("HandlerThread-1");
        handlerThread.start();
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
