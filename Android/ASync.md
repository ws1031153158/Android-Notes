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
配合 Looper 和 Message 以及 MessageQueue 使用，实现子线程和主线程的切换，主要用于更新UI。  
创建 Handler 需要传入 Looper（使用默认 Looper 的构造函数已弃用），可以通过 sendMessage（需要重写 handleMessage 来处理消息，在 sendMessage 执行到 enqueueMessage 时，会将 handler 本身和 message 绑定，传递给 looper 来区分是哪个 handler 的哪个 message）或 post（传入 callback）方法传递消息，此时将消息放入消息队列（当前线程的队列）等待 Looper获取才会执行。  
Message.callback（post 传递的 Runnable） 不为null 则 handleCallback，为null 则检查 mCallback，不为null 则调用 mCallback.handleMessage（返回为 false 则交给自身 handleMessage），为 null 则自己 handleMessage 处理  
IdleHandler：一个回调接口，消息队列没有消息时（如当前获取的 message 为空）或队列中消息没到执行时间时（消息队列将要阻塞等待时）才会执行，存放在 mPendingIdleHandlers 队列（触发时遍历该队列）
## ThreadLocal
存储线程相关的数据，每个线程存的数据只有该线程可以获取，作用域是线程（属于线程变量），不同线程具有不同副本时，需要使用到，比如复杂业务逻辑下获取监听，另外，可以让某个对象或数据成为线程内的全局元素存在，Handler 可以通过 ThreadLocal 获取到该线程的 Looper  
set：values 获取 currentThread 数据，若不存在则赋予初始化值，接着将数据存入一个 table 数组中（values.put，放入 reference 字段标识对象 index 的下一个位置）  
get：取出当前 thread 的 values 对象，返回对象为 null 则给初始值（initialValue，默认返回为 null，可重写）
## MessageQueue
单链表实现，enqueueMessage 插入，next 读取（并移除），next 为无限循环方法，队列没有消息会一直阻塞，也就造成 looper 的阻塞
looper.quit 调用 messageQueue.quit，接着 messageQueue 标记正在退出，最终 next 返回 null  
msg.target（每个 msg 对应一个 target，为对应 handler 的引用，也就是发送消息的 Handler）.dispatchMessage（自身处理，但是切换到创建 Handler 所使用的 Looper 中执行）来处理消息
## Looper
对 MessageQueue 进行轮询，没有新消息会一直堵塞，构造方法中会创建一个 MessageQueue  
Looper.prepare 加上 Looper.loop（执行轮询，死循环，MessageQueue.next 返回 null 才会跳出循环，没有消息阻塞则会在 nativePollOnce 释放 cpu 资源进入休眠）来为线程创建一个 Looper，子线程手动创建 Looper 需要主动调用 quit 退出循环，退出后线程会立刻终止。
一般 ANR 是由于规定时间没有完成指定任务，系统会抛出异常，looper 阻塞时不会占用过多 cpu 资源，也不存在指定时间，不会抛出 anr，但对于业务添加的线程，尽量再确定不再使用时回收资源关闭 looper
# RxJava
异步数据处理库，也是一种扩展的观察者模式。  
四大要素：Observable/Flowable（额外传入 backPresureStrategy 参数（ERROR(抛 missingBackPressureException异常)/BUFFER(扩大缓存池,default 128)/DROP(丢弃)/LATEST(消费者.request 传入需求数量，超过丢弃，可以接收最后一个事件））)（被观察者），Observer/Subscriber (实现了 observe r接口,多了unsubscribe 取消订阅,支持背压，异步中，被观察者发送事件速度快于观察者处理速度时，告诉上游被观察者降低发送速度)  
Schedulers(调度器)：解决多线程问题，io 用于 I/O 操作（IO 密集型任务如网络请求提供 IO 线程池） 、computation 用于计算工作（CPU 密集型任务专用线程池）(默认的调度器)、immediate 立即在当前线程执行指定工作、newThread 创建新线程（线程池有上限）、trampoline 顺序按需处理队列，并运行队列的每一个任务。
AndroidSchedulers：RxAndroid 提供在 Android平 台的调度器，指定观察者在主线程(mainThread,可以直接 post 到主线程执行UI操作)。  
创建：create（传入 observableonsubscribe (回调接口，回调 subscribe，参数为 observableEmitter(Emiiter 子类，提供 onnext/onerror/oncomplete 方法)) -> 用来创建 observablecreate，为实现类,后续调用其实是调用的这个)
大部分操作符，都是新建 Observable 对象，包装上游 Observable 对象，传入新的 OnSubscribe，最后都是调用 create（链式调用基于代理模式，一层一层的包装(包装 onSubscribe））    
订阅：返回 subscribtion，调用 observablecreate.subscribe 接着回调 onSubscribe（传的 disposable 是 createEmitter(实现了disposable 接口,调用 onNext 等方法，任务取消时不再回调观察者)），从下往上递归调用 observable 的 subscribe 方法  
切线程：传入一个 Schedule，IO 由 IoScheduler 切到 IO 线程，Main 由 Handlercheduler 切到主线程。  
subscribeon 指定 subscribe 所在线程，用于 obervable，只第一次生效（链式调用的第一个），subscribeon 返回 observable(observableSubcribeOn) 接着包装 observer 回调 onSubscribe，设置 disposable ，schedualDirect(包含一个worker，work.schedualDirect 来创建一个 runnable) ，oberveron 用于 oberver，指定下游 observer 回调所在线程，返回 observableObserveOn，同样是创建 worker 并创建一个runnable，最终都是将 runnable 通过 handler 去 post
warnning：  
内存泄漏：响应机制通过回调实现，所以可能有内存泄漏，如异步任务未执行完页面 back 了，subscribe 中创建 observer 或者 consumer 属于匿名内部类，要在 destroy 的时候去 unsubscribe（维护一个集合统一处理）
# Tips
## 子线程更新 UI
当前屏幕刷新率较高，需要频繁更新 view，则不能对 UI 更新操作加锁，这样可能会导致多个线程更新同一个 View，ViewRootImpl 中 checkThread 检查当前更新 UI 线程是否和创建 UI 线程相同，否则抛异常  
子线程更新 UI 需要满足：  
1.ViewRootImpl 创建前无线程限制  
2.保证创建和更新 UI 操作在同一线程  
3.对应线程需要创建 Looper 并调用 loop 方法，保证 UI 更新后的效果能够显示（WindowManger.addView  实现类 WindowManageGlobal.addView 调用到 Root 执行 ViewRootImpl.setView ，该方法会调用 ViewRootImpl.requestLayout 来 scheduleTraversals ，该方法会往消息队列中插入一条消息屏障，并调用 Choreographer.postCallback 方法，往 looper 中插入 MSG_DO_SCHEDULE_CALLBACK 异步消息，等 VSync 来后执行。（ViewRootImpl 有一个 Choreographer 成员变量，ViewRootImpl 的构造函数中会调用 Choreographer.getInstance，获取当前线程的 Choreographer 局部实例））
