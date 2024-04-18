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
## 内存抖动
短时间内频繁大量创建临时对象，会频繁 GC，无论哪种方式实现的GC在执行时都不可避免的需要 STW（Stop The World），STW 意味着所有的工作线程都将被暂停，虽然时间很短，但终究会存在时间成本，一两次内存回收不易被察觉，但多次内存回收集中在短时间内爆发，这就会造成较大程度的界面卡顿风险。   
尽量避免在循环体中创建对象、尽量不要在自定义 View 的 onDraw 方法中创建对象（会被频繁调用）、对于可复用对象，可以考虑使用对象池缓存。  
## Bitmap
图片内存 = 宽高一个像素占用内存（和色彩模式有关，如 ARGB_8888 为 4（8*4 = 32位 = 4字节））。  
Bitmap 对象放在 Java 堆，像素数据放在 Native 内存，8.0 新增 NativeAllocationRegistry 来辅助回收 Native 内存，以及可减少图片内存并提升绘制效率的硬件位图 Hardware Bitmap ，Bitmap 的 Native 内存可和对象一起释放， GC 避免内存滥用。此外，Glide 会自动调整加载的图片大小（根据 Imageview，以及三级缓存优化图片加载）。  
## LeakCanary
hook Android 生命周期，自动检测当 Activity、Fragment 销毁时实例是否回收，销毁的实例传给 RefWatcher（持有它们的弱引用），保留实例（Retained Instance）数量达到阈值会进行堆转储，数据放进 hprof 文件（APP 可见阈值为 5，不可见为 1），会解析 hprof 文件，找出导致 GC 无法回收实例的引用链，就是泄漏踪迹（Leak Trace，最短强引用路径，GC Roots 到实例的路径）。  
监听系统内存状态：  
ComponentCallback2：在 Activity 中实现 ComponentCallback2 接口获取系统内存的相关事件, 在 onTrimMemory(level)  回调针对不同事件做不同释放内存操作  
ActivityManager.getMemoryInfo()：  
返回一个 ActivityManager.MemoryInfo 对象，包含系统当前内存状态（可用内存、总内存、低杀内存阈值， lowMemory 布尔值判断是否处于低内存态）  
## Tips
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
原生采用信号（软中断信号）量中断处理的捕捉方式，程序运行在用户态，系统调用、中断或异常时进入内核态，信号涉及两种状态转换。  
接收：接收信号由内核代理，收到信号会放到对应进程信号队列中，向进程发送一个中断，使其进入内核态。（此时信号只在队列中，对进程来说暂时不知道信号到来）。  
检测：进程进入内核态有两种场景对信号检测：    
1.从内核态返回用户态前进行检测。  
2.在内核态中，从睡眠状态被唤醒时进行检测。    
有新信号时进入信号处理：处理运行在用户态，处理前，内核将当前栈内数据拷贝到用户栈，修改指令寄存器（eip）并指向信号处理函数，之后返回用户态，执行对应处理函数，处理完成后返回内核态，检查是否有信号未处理。所有信号处理完成，恢复内核栈（用户栈拷贝回来），恢复指令寄存器（eip）并指向中断前的运行位置，最后回到用户态继续执行进程。
# System trace
## MainThread/RenderThread
一帧流程（60fps | 16.6ms）：  
1.主线程接到 Vsync 信号（一个 Message 来唤醒），Choreographe r回调 onVsync 开始一帧的绘制  
2.处理 Input 事件  
3.处理 Animation 回调  
4.处理 Traversal 回调  
5.Main 和 Render 进行 Sync，主线程结束绘制，可以进行下一个 Message 及 Idle，或等下一次 Vsync  
6.Render 从 BufferQueue 取出 Buffer（DequeueBuffer），进行真正的渲染（主线程的 draw 没有执行 drawCall，是将内容记录到 DisplayList（RednerNode）中，同步到 RenderThread）  
7.Render 将完成的 Buffer 返还给 BufferQueue，等 Vsync-SF 交由 SF 合成  
tips：renderThread 对应硬件加速（默认开启），开启硬件加速后会初始化 renderThread，使用 GPU 来渲染。
## Input
1.触摸屏每隔几毫秒扫描一次，如果有触摸事件，则将事件上报到对应驱动  
2.InputReader 从 EventHub 读取触摸事件放到 InboundQueue  
3.InputDispatcher 从 InboundQueue 中将触摸事件包装分发给注册了 Input 事件的 App 的 OutBoundQueue，同时将事件记录到各个 App(连接) 的 WaitQueue（两者为 SystemServer 中的 Native线程）  
4.App 拿到事件，记录到 PendingInputEventQueue，进行 Input 事件分发，若此分发过程中，App 的 UI 发生变化，则请求 Vsync 进行一帧的绘制  
5.App 处理完成，回调 InputManagerService 将负责监听的 WaitQueue 中对应 Input 移除
## Vsync
从左到右边，从上到下逐行扫描像素点，在屏幕刷新频率（帧率，fps，GPU一秒绘制的帧数）固定时，一个屏幕内数据来自 2 个不同帧（buffer 在显示过程中被修改），画面会出现撕裂，垂直同步就是为了解决此问题。  
双缓存：绘制和显示器拥有各自  buffer，GPU 始终将完成一帧写入 Back Buffer，显示器使用 Frame Buffer，屏幕刷新时，Frame Buffer 不变，Back buffer 就绪后进行交换  
三缓存：双缓冲基础上增加一个 Graphic Buffer，CPU 空闲时，Back Buffer 被占用，只能等待 GPU 后再写入，此时第三缓存提前准备好数据。  
扫描完屏幕后，需要重新回到第一行进入下次循环，此时有一段时间空隙（VBI），进行缓存交换，Vsync 利用 VBI 出现的垂直同步脉冲来保证交换时间。
### offset
为 0， App 和 SurfaceFlinger 同时收到 Vsync 信号。  
不为 0，App 先收到 Vsync 信号，进行一帧渲染，然 Offset 后，SurfaceFlinger 收到 Vsync 信号开始合成，这时如果 App 的 Buffer 已经 Ready ，那 SurfaceFlinger 这一次合成就可以包含 App 这一帧，用户也会早一点看到。
## ​Choreographer
配合 Vsync ，给上层 App 渲染提供稳定的 Message 处理时机： Vsync 到来，系统对 Vsync 信号周期调整，控制每一帧绘制操作时机，Vsync 信号唤醒 Choreographer 来做 App 的绘制操作（通过 postCallback 设置的回调函数调用使用者）。  
承上：接收和处理 App 的更新消息和回调，等 Vsync 到来统一处理。如集中处理 Input 、Animation、Traversal( measure、layout、draw ) ，判断卡顿掉帧情况，记录 CallBack 耗时等  
启下：接收 Vsync 事件回调，请求 Vsync 信号  
线程单例要和一个 Looper 绑定（内部有一个 Handler 所以需要绑定 Looper ），定义了一个 FrameCallback interface，当 Vsync 到来，doFrame 被调用，在固定的时间中断。
## CPU
1.C0 状态（激活）  
最大工作状态，可以接收指令和处理数据  。  
2.C1 状态（挂起）  
可以通过执行汇编指令“ HLT ”进入，唤醒时间快（只需 10 ns），节省 70% 的 CPU 功耗  。  
3.C2 状态（停止允许）  
处理器时钟频率和 I/O 缓冲被停止(执行引擎和 I/0 缓冲已经没有时钟频率)，可节约 70%  CPU 和平台能耗，从 C2 切到 C0 需要 100 ns以上。  
4.C3 状态（深度睡眠）  
总线频率和 PLL 均被锁定，在多核心系统下，缓存无效，在单核心系统下，内存被关闭，但缓存仍有效。可节省 70%  CPU 功耗，平台功耗比 C2 状态大，唤醒时间需要 50 ns  
# HWC
HWC（Hardware Composer）是 Android 中进行窗口（Layer）合成和显示的 HAL 层模块，其实现是特定于设备的，而且通常由显示设备制造商 (OEM)完成，为 SurfaceFlinger 服务提供硬件支持。   
SurfaceFlinger 可以使用 OpenGL ES 合成 Layer，这需要占用并消耗 GPU 资源。大多数 GPU 都没有针对图层合成进行优化，当 SurfaceFlinger 通过 GPU 合成图层时，应用程序无法使用 GPU 进行自己的渲染。而 HWC 通过硬件设备进行图层合成，可以减轻 GPU 的合成压力 。  
1.SurfaceFlinge r向 HWC 提供所有 Layer 的完整列表，让 HWC 根据其硬件能力，决定如何处理这些 Layer。  
2.HWC 会为每个 Layer 标注合成方式，是通过 GPU 还是通过 HWC 合成。  
3.SurfaceFlinger 负责先把所有注明 GPU 合成的 Layer 合成到一个输出 Buffer，然后把这个输出 Buffer 和其他 Layer（注明 HWC 合成的 Layer）一起交给 HWC，让 HWC 完成剩余 Layer 的合成和显示。
