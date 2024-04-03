# Build
Runnable：无返回值  
Callable：有返回值  
new Thread  新建线程对象  
ThreadPool：可控制并发条件，一般是通过 Executor 来构建的，也可获取系统的线程池使用  
线程优先级和属性会被继承，线程优先级和属性和开启他的线程的相同（父子关系）  
start 开启线程，无需主动 run（jvm 会调，主动调则相当于调了两次，也会执行两次）  ，join 用于等待其他线程执行结束（如 A 调用 B 的 join，A 进入等待状态直到 B 结束）
# ThreadPool
# Lock
## Condition
# Interrupt
# Class
# Automic
