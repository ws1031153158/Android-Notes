# AsyncTask
异步框架，现已弃用，一些老的项目会较多使用  
需要重写 doInBackground 等回调，业务逻辑复杂时会陷入回调地狱  
底层是对 Executor 的封装，内部持有 2-4 个线程大小的线程池，执行 execute 将任务提交到内部的 ArrayDeque 容器封装为一个 runnable，之后从容器中取出任务串行执行最终会封装为 futureTask 由线程池来处理，在 doInBackground 回调，postresult 返回结果（获取 AsyncTask 创建时传入的 Handler 来处理，不传则默认主线程）  
# FutureTask
更加轻量级的异步框架，配合Executor使用，可以主动调用 cancel 中断任务，可以通过 get 方法获取期望值
# HandlerThread
继承 Thread，封装 Handler，实现 runnable 接口，创建 HT 对象等于创建 Thread（即新开一个工作线程） 对象加上设置线程优先级（构造函数传入线程名），run 方法中创建 looper（looper 创建 Handler）以及消息队列，为一个无限循环的方法，不需要时要调用 quit 终止线程。  
通过 Handler 向 HT 消息队列发送消息来执行任务，主要用于 IntentService（抽象类，执行高优先级后台耗时任务，执行完毕自动停止，优先级高于一般线程（本身是 Service，但执行在工作线程），不易被杀死，封装了 Handler 和 HT（onCreate 中创建），每次启动调用 onStartCommand，接着调用 onStart（针对本身），由 Handler.sendMessage 发送消息，交给 HT 处理，将 intent 对象交给 onHandleIntent（执行结束调用 stopSelf）处理，内容和 startService 传递的 intent 一致，解析启动 IntentService 传递的参数）
存在问题：在获得 HandlerThread 工作线程的 Looper 对象时存在一个同步的问题：只有当线程创建成功 & 其对应的 Looper 对象也创建成功后才能获得 Looper 的值，才能将创建的 Handler 与 工作线程的 Looper 对象绑定，从而将 Handler 绑定工作线程。  
解决方案：即保证同步的解决方案 ，同步锁、wait 和 notifyAll，即 在 run 中成功创建 Looper 对象后，立即调用 notifyAll 通知 getLooper 中的 wait 结束等待 & 返回 run 中成功创建的 Looper 对象，使得 Handler 与该 Looper 对象绑定。
# Executor
线程调度器，配合 ExecutorService（子类）使用，常用的子类有：  
ThreadPoolExecutor：线程池调度，需要手动给定参数，如核心线程数，非核心线程数，超时计数，任务队列（有默认），自定义线程工厂（有默认）等  
newFixThreadPool：固定大小  
newSingleThreadExecutor：只有一个线程  
未达到核心线程数启动核心线程直接启动，否则插入任务队列等待，若队列已满线程数未达到最大线程数则开启非核心线程，若达到最大线程数则拒绝任务并通知调用者。  
线程 getTask 返回 null 会回收工作线程，正常任务全部执行完，存在非核心线程，超时唤醒后 CAS 返回 null 来回收；此外，调用 shutdown，全部返回 null 全回收，还有没执行完的任务，会向任一空闲线程发送中断信号。
# CountDownLatch
一种信号量，作为多线程倒计时的计数器，强制等待其他任务执行完成，不会复原，创建 CountDownLatch 并设置计数值，启动多线程并调用 CountDownLatch 实例的 countDown，主线程调用 await，count 为 0 停止阻塞（此后再调用 await 不会进入等待，所以是一次性的）
## CyclicBarrier
多个线程需要互相等待对方代码中的某个地方时使用。  
await 实现等待的线程叫参与方，除最后一个执行 CyclicBarrier.await 方法的线程外，其他执行该方法的线程都会被暂停，可以重复使用（进行多轮的等待）。  
CyclicBarrier 内部有一个用于实现等待/通知的 Condition（条件变量）类型的变量 trip ，还有一个分代（Generation）对象，表示 CyclicBarrier  实例可以重复使用，当前分代初始状态是 parties（参与方总数），await 每执行一次，parties 值减 1，调用了 CyclicBarrier 方法的参与方相当于等待线程，最后一个参与方相当于通知线程。
通知线程调用 await 时，先执行 barrierAction.run，再执行 trip.signalAll 唤醒所有等待线程，接着开始下一个分代（parties 恢复为初始值）
Generation 中有一个布尔值 broken，调用 await 线程被中断时变为 true，会抛出一个 BrokenBarrierExcetpion 异常，表示当前分代已被破坏，无法完成任务（使用 CyclicBarrier 的线程不能被中断（interrupt 方法被调用）
# Handler
## ThreadLocal
## MessageQueue
## Looper
# RxJava
# Tips
