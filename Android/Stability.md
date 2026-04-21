# Crash
## Java Crash
### 全局 UncaughtExceptionHandler
### 崩溃信息采集、堆栈还原
## Native Crash
原生采用信号（软中断信号）量中断处理的捕捉方式，程序运行在用户态，系统调用、中断或异常时进入内核态，信号涉及两种状态转换。  
接收：接收信号由内核代理，收到信号会放到对应进程信号队列中，向进程发送一个中断，使其进入内核态。（此时信号只在队列中，对进程来说暂时不知道信号到来）。  
检测：进程进入内核态有两种场景对信号检测：    
1.从内核态返回用户态前进行检测。  
2.在内核态中，从睡眠状态被唤醒时进行检测。    
有新信号时进入信号处理：处理运行在用户态，处理前，内核将当前栈内数据拷贝到用户栈，修改指令寄存器（eip）并指向信号处理函数，之后返回用户态，执行对应处理函数，处理完成后返回内核态，检查是否有信号未处理。所有信号处理完成，恢复内核栈（用户栈拷贝回来），恢复指令寄存器（eip）并指向中断前的运行位置，最后回到用户态继续执行进程。
### Breakpad / LLVM libunwind
### tombstone 分析
### addr2line 符号还原
## OOM Crash
### 内存快照（Hprof）采集
### 大对象追踪
### 内存兜底策略
## 归因分析
### 线上监控
### 版本/机型/系统分布分析
### 聚合策略
### 报警策略
### 相似崩溃合并
## 崩溃恢复
### 进程重启策略
### 数据保护
### 降级方案
## 热修复
### Tinker
### Robust
### Sophix
### 字节码插桩方案
# ANR
## 应用侧归因
1.死锁  
2.主线程调用 thread 的 join()方法、sleep()方法、wait()方法或者等待线程锁的时候  
3.主线程阻塞在 nSyncDraw  
4.主线程耗时操作，如复杂的 layout，庞大的 for 循环，IO 等  
5.主线程被子线程同步锁 block  
6.主线程等待子线程超时  
7.主线程 Activity 生命周期函数执行超时  
8.主线程 Service 生命周期函数执行超时  
9.主线程 Broadcast.onReceive 函数执行超时（即使调用了 goAsync ）  
10.渲染线程耗时  
11.耗时的网络访问  
12.大量的数据读写  
13.数据库操作  
14.硬件操作（比如 Camera)  
15.service binder 的数量达到上限  
16.其它线程终止或崩溃导致主线程一直等待  
17.Dump 内存操作  
18.大量 SharedPerference 同时读写  
## 系统侧归因
1.与 SystemServer 进行 Binder 通信，SystemServer 执行耗时：方法本身执行耗时导致超时，或者 SystemServer Binder 锁竞争太多，导致等锁超时  
2.等待其他进程返回超时，比如从其他进程的 ContentProvider 中获取数据超时（10s，但是这个场景很少见）  
3.Window 错乱导致 Input 超时（5s）  
4.ContentProvider 对端的进程频繁崩溃，也会杀掉当前进程  
5.整机低内存  
6.整机 CPU 占用高：查看是总使用率是否过高，同时需要关注缺页次数（访问的页面不在内存时，会产生一次缺页中断，需要调入主存，一次调入次数加一），xxx minor 表示高速缓存中的缺页次数，可以理解为进程在做内存访问，xxx major 表示内存的缺页次数，可以理解为进程在做 IO 操作   
7.整机 IO 使用率高  
8.SurfaceFlinger 超时  
9.系统冻结功能出现 Bug：应用是否处于 D 状态（TASK_UNINTERRUPTIBLE，不可中断的睡眠状态，通常是由于它正在执行一个不能中断的 I/O 操作，如磁盘读写、网络传输等）    
10.System Server 中 WatchDog 出现 ANR  
11.整机触发温控限制频率  
12.Service 前台 20 s，后台 200 s 超时（system server 在 service 流程开始前会发生 delay 消息到 main handler，若 app 端在 delay 消息内未通知 system server 移除消息，则超时）  
1.Broadcast 前台 10 s，后台 60 s 超时（同样，也是发生 delay 消息，等待 receiver 去移除，contentProvider 一样的机制）
## ANR 监控
### 主线程 WatchDog
### Signal 监听（SIGQUIT）
### ANR-WatchDog 库
## Binder 阻塞
### Binder 线程耗尽
### 系统服务响应慢
## Broadcast ANR
### 前台 10s / 后台 60s 超时
### onReceive 耗时
## ContentProvider ANR
### publish 超时
## Service ANR
### 前台 20s / 后台 200s 超时
## Input 超时
### 主线程 5s 无响应
### 触摸事件分发耗时
## 系统资源不足
### CPU 负载
### IO wait
### 内存压力导致的 ANR
