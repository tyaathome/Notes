# Java部分

## 多线程

### Android的Handler

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

### RxJava
1. 订阅原理

   * 示例代码

     ```java
     Observable.just(1)
             .subscribeOn(Schedulers.io())
             .observeOn(AndroidSchedulers.mainThread())
             .map((Function<Integer, String>) integer -> String.valueOf(integer))
             .subscribe(new Observer<String>() {});
     ```
     
   * 流程图

     ![流程图](https://github.com/tyaathome/Notes/blob/main/images/rxjava%E6%B5%81%E7%A8%8B%E5%9B%BE.png?raw=true)
     
   * 具体流程

     ① 通过链式调用将被观察者(observable)进行包装(ObservableJust->ObservableSubscribeOn->ObservableObserveOn->ObservableMap)，每一步的observable中都包含上一步的source;

     ② 从下至上递归调用observable.subscribe(observer)方法，将观察者(observer)进行包装(Observer<String>->MapObserver->ObserveOnObserver->SubscribeOnObserver)，每一步的subscribe中都保存上一步的downstream对象;

     ③ 再至上至下递归调用下游的observer.onNext(t)、observer.onComplete()等方法(SubscribeOnObserver->ObserveOnObserver->MapObserver->Observer)结束任务。
     
     ④onSubscribe()方法大体来说是由上至下递归调用onSubscribe()方法，例外情况比如ObservableSubscribeOn对象被订阅时(`subscribe()`)会调用`observer.onSubscribe(parent)`方法，导致这时会向下游递归调用onSubscribe()方法。以我们例子的onSubscribe()方法调用步骤是ObserveOnObserver->MapObserver->Observer<String>->最上游订阅时再调用SubscribeOnObserver。

2. 线程切换原理

   * subscribeOn()是在subscribe()中将task切换至scheduler线程中运行，而通过流程图可知subscribe()方法是在第②步进行的，流程是由下至上，所以subscribeOn()是控制上部的线程；

     ` ObservableSubscribeOn.java`:

     ```java
     @Override
     public void subscribeActual(final Observer<? super T> observer) {
         final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer); // 包装上一步的observer
         observer.onSubscribe(parent);
         parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent))); // 执行subscribe
     }
     ```

     `Scheduler.java`:

     ``` java
     public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
         final Worker w = createWorker(); // 创建worker
         final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
         DisposeTask task = new DisposeTask(decoratedRun, w);
         w.schedule(task, delay, unit); // 在schedule所属的线程运行run
         return task;
     }
     ```

     从上面的代码可知run中的线程就是subscribeOn中所设置的，而下一步是继续执行上面的流程，所以可知subscribeOn所设置的线程是控制顶部操作的线程。

   * observeOn是在onNext()、onError()、onComplete()等方法中将task切换至scheduler线程中运行，而通过流程图可知observer.onNext()等方法是在第③步进行，流程是由上至下，所以observeOn控制下部的线程。

     `ObservableObserveOn.java`:

     ``` java
     public void onNext(T t) {
         if (done) {
             return;
         }
         if (sourceMode != QueueDisposable.ASYNC) {
             queue.offer(t);
         }
         schedule(); // 调用downstream.onNext()
     }
     ```

     从上面代码可知schedule()中的线程是observeOn中所设置的，也就是onNext()的线程，所以observeOn设置的是map及以下的流程的线程。

3. Disposable原理

     * disposable的作用是可以主动解除上下游的订阅关系；
     * 所有的observer都实现了Disposable接口，observer都会持有上下游的对象，`onSubscribe()`传递的Disposable对象其实就是observer本身；
     * 订阅的时候上游对象会向下游递归调用`onSubscribe()`方法，最后得到disposable对象；
     * 解除订阅的时候是调用disposable.dispose()方法，而会从下至上递归调用上游的`dispose()`方法来解除订阅。

4. Scheduler原理

     * Scheduler作用是将任务放在指定线程执行，而且还能执行dispose()操作，用来取消订阅；
     * Schedulers.io()
       * 用于IO操作的调度器；
       * 原理是`IoScheduler`自身维护一个包含`ThreadWorker`的 `CachedWorkerPool`线程池，线程池还存在过期检查队列和一个过期检查的线程；
       * `ThreadWorker`本质上是一个大小为1定长的`ExecutorService`，也就是由Executors.newScheduledThreadPool(1, factory)方法创建的；
       * 所有由`Schedulers.io()`创建的`ThreadWorker`工作完了之后会被release(threadworker)方法，该方法作用是将threadworker加入到过期检查队列，而且会为ThreadWorker延续过期时间(now() + keepAliveTime)以供线程池继续使用，而不会频繁初始化ThreadWorker，线程池初始化时会运行一个单一线程用来检查缓存队列里的ThreadWorker已经过期；
     * Schedulers.newThread()
       * 该调度器每次使用都会创建一个新线程
       * 原理是`NewthreadScheduler`每次使用都会生成一个大小为1定长的`ExecutorService`，也就是由Executors.newScheduledThreadPool(1, factory)方法创建的。
     * AndroidSchedulers.mainThread()
       * 用于在主线程操作的调度器；
       * `HandlerScheduler`内部包含一个`Handler`对象，而任务在`Handler`中进行，所以AndroidSchedulers.mainThread()创建的调度器是包含一个主线程`Looper`的`Handler`，所以所有的任务会在主线程中进行。

### Retrofit

1. 动态代理
   * 静态代理是在编译时期已经被编译成class文件，然后加入到jvm里，最后调用实际的类方法；
   * 动态代理则是在运行时生成的，在运行时动态生成类字节码，然后通过classloader将该类加载到jvm里，最后实现接口对象，而调用具体方法则是被代理成InvocationHandler类的invoke()方法达到调用目的。

2. 源码解析

   * 使用动态代理实现Service类的实现

     ```java
       public <T> T create(final Class<T> service) {
         ...
         return (T)
             Proxy.newProxyInstance(
                 service.getClassLoader(),
                 new Class<?>[] {service},
                 new InvocationHandler() {
                   private final Platform platform = Platform.get();
                   private final Object[] emptyArgs = new Object[0];
     
                   @Override
                   public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                       throws Throwable {
                     ...
                     return platform.isDefaultMethod(method)
                         ? platform.invokeDefaultMethod(method, service, proxy, args)
                         : loadServiceMethod(method).invoke(args);
                   }
                 });
       }
     ```
     
   * loadServiceMethod实现

     * 通过RequestFactory.parseAnnotations(retrofit, method)解析Service类中具体方法的配置；
     * 通过HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory)解析在retrofit配置的callAdapter和responseConverter等参数。
     
   * loadServiceMethod.invoke(args)实现
   
     * 通过使用在上一步创建好的requestFactory、args、 callFactory、responseConverter等参数创建出Call<T>对象；
     * 然后通过调用`callAdapter.adapt(call)`来完成从Service接口方法的调用适配为网络请求(Call)调用，从而完成整个网络请求。
