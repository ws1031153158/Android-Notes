# 启动优化
视觉：设置预览图、自定义 Theme 等优化用户等待加载时的观感。  
绑大核：处理速度不同，绑大核就是让优先级（PR/IN，越低越好）高（容易获取时间片）线程、进程运行在频率高的 CPU上（一般 0-3 小核，4-7 大核）。  
GC抑制：ART 不会时停，但也很消耗资源，原生处理为 2s 后提高 GC 阈值，可以减少次数。  
onCreate：  
异步init，将 init 分为多个子 task 放在子 thread（提交到线程池），task 之间无依赖关系。此外，若 ini t在 Application.onCreate 结束前完成，可用 CountDownLatch 等待 task 执行，或者根据依赖关系排序生成有向无环图，最后由优先级执行 task。任务可以分为（init 前、后，空闲，子、主线程等 task）。
延迟init：优先级不高，可在启动完成后执行的 init task延迟到启动完成后执行（如 handler.postdelay），但延迟后的页面加载完成可能造成手势失效（如滑动）。  
此外，init launcher 利用 IdleHandler 实现主线程空闲时执行任务，不影响用户操作。    
状态：  
冷启动：先创建进程，再启动应用，最后绘制UI。  
暖启动：只重走 Activity 的生命周期。  
热启动：通过 onStop 或 onPause 后直接 onResume。  
速度测量 && 启动耗时：adb shell dumpsys AMS 耗时信息（AMS）、attachBaseContext 记录启动时间，在 View 绘制完记录结束时间（埋点）（onWindowFocusChanged 是首帧绘制，还未完成）、Debug SDK、Trace SDK
# 内存优化
OOM：见OOM。  
内存抖动：  
短时间内频繁大量创建临时对象（频繁GC），尽量避免在循环体中创建对象、尽量不要在自定义 View 的 onDraw 方法中创建对象（会被频繁调用）、对于可复用对象，可以考虑使用对象池缓存。  
Bitmap：  
图片内存 = 宽高一个像素占用内存（和色彩模式有关，如 ARGB_8888 为 4（8*4 = 32位 = 4字节））。  
Bitmap 对象放在 Java 堆，像素数据放在 Native 内存，8.0 新增 NativeAllocationRegistry 来辅助回收 Native 内存，以及可减少图片内存并提升绘制效率的硬件位图 Hardware Bitmap ，Bitmap 的 Native 内存可和对象一起释放， GC 避免内存滥用。此外，Glide 会自动调整加载的图片大小（根据 Imageview，以及三级缓存优化图片加载）。  
LeakCanary：  
hook Android 生命周期，自动检测当 Activity、Fragment 销毁时实例是否回收，销毁的实例传给 RefWatcher（持有它们的弱引用），保留实例（Retained Instance）数量达到阈值会进行堆转储，数据放进 hprof 文件（APP 可见阈值为 5，不可见为 1），会解析 hprof 文件，找出导致 GC 无法回收实例的引用链，就是泄漏踪迹（Leak Trace，最短强引用路径，GC Roots 到实例的路径）。  
监听系统内存状态：  
ComponentCallback2：在 Activity 中实现 ComponentCallback2 接口获取系统内存的相关事件, 在 onTrimMemory(level)  回调针对不同事件做不同释放内存操作  
ActivityManager.getMemoryInfo()：  
返回一个 ActivityManager.MemoryInfo 对象，包含系统当前内存状态（可用内存、总内存、低杀内存阈值， lowMemory 布尔值判断是否处于低内存态）  
tips：  
SparseArray：只有 integer 类型属性，避免了自动装箱（基本数据类型和包装器类型（引用类型）转换，包装器 .valueof 装箱，xxxValue 拆箱，实现基本类型和引用类直接运算）的开销，可以代替 hashMap。  
﻿adb shell dumpsys meminfo pid 可以获取内存信息：  
PSS/RSS：实际使用物理内存（包括进程独占（USS）和共享（PSS按进程占用共享等分））  
Private：进程独占内存  
SWAP PSS：释放后其他进程可以使用的内存，所以只能看到 Dirty，包含在 PSS  
Native Heap：PSS + SWAP PSS DIRETY  
详细内存分布 -> APP 内存（图片、java堆、）-> 总内存 -> 对象内存（view、activity、viewimpl，可当做内存泄漏的依据）  
# ANR
主线程 sleep 不一定 ANR，休眠期间没有其他消息需要处理则不会，若此时有点击事件或其他线程传来的更新 UI 请求则可能会 ANR  
# Runtime Crach
app 不会闪退，但进程被杀掉，不会接受任何事件，可以通过 getDefaultUncaughtExceptionHandler 交给系统处理（如捕获到异常为空）。  
try-catch 可以捕获主线程异常。 ，UncaughtExceptionHandler 可以捕获子线程异常，异常发生回调 uncaughtException，捕获到的异常为 Throwable。  
自定义 crashHandler 继承 Thread.UncaughtExceptionHandler，初始化进行，Thread.setDefaultUncaughtExceptionHandler(this) 操作，在 unCaughtException 回调中处理上报异常（有可能此时已未响应，需要创建 looper）。
## Native Crash
# System trace
## MainThread/RenderThread
## Input
## Vsync
### offset
## ​Choreographer
## CPU
