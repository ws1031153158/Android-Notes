# System trace
## 抓取
push 配置文件：  
adb push config.pbtx /data/misc/perfetto-configs/config.pbtx  
抓 SysTrace：  
adb shell perfetto --txt -c /data/misc/perfetto-configs/config.pbtx -o /data/misc/perfetto-traces/trace.perfetto-trace    

Android 12 以下（直接通过文件路径传递配置文件在非 root 设备上会失败）：  
adb push config.pbtx /data/local/tmp/config.pbtx   
adb shell 'cat /data/local/tmp/config.pbtx | perfetto -c - -o /data/misc/perfetto-traces/trace.perfetto-trace'  
## 自定义标记
Trace.beginSection + Trace.endSection()，定位代码耗时
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
### 流程
应用一帧渲染的整体流程，从执行顺序的角度来看是从 Choreographer 收到 Vsync 开始，到 SurfaceFlinger/HWC 合成一帧结束（后面还包含屏幕显示部分）  
![image](https://github.com/user-attachments/assets/02ca4c5f-4096-49e6-ba34-fa01907fb8c0)  
从 System 角度看是：  
![image](https://github.com/user-attachments/assets/d8bf43fd-63a3-4272-8371-b2144f93f30a)

## CPU
1.C0 状态（激活）  
最大工作状态，可以接收指令和处理数据  。  
2.C1 状态（挂起）  
可以通过执行汇编指令“ HLT ”进入，唤醒时间快（只需 10 ns），节省 70% 的 CPU 功耗  。  
3.C2 状态（停止允许）  
处理器时钟频率和 I/O 缓冲被停止(执行引擎和 I/0 缓冲已经没有时钟频率)，可节约 70%  CPU 和平台能耗，从 C2 切到 C0 需要 100 ns以上。  
4.C3 状态（深度睡眠）  
总线频率和 PLL 均被锁定，在多核心系统下，缓存无效，在单核心系统下，内存被关闭，但缓存仍有效。可节省 70%  CPU 功耗，平台功耗比 C2 状态大，唤醒时间需要 50 ns  
## Task
### Sleep
白色，有两种情况：  
1.nativePoll 这种，一般属于主动 Sleep，因为没有消息处理了，所以进入 Sleep 状态等待 Message，一般是正常的，不需要关注，比如两帧之间的那段，就是主动 sleep  
2.被动 Sleep 一般是由用户主动调用 sleep，或者用 Binder 与其他进程进行通信，这个是我们最常见的，也是分析性能问题的时候经常会遇到的，需要重点关注，通过 binder transaction 来查看 Binder 调用信息，或者如果没有 binder 信息，那么就需要去看在等待什么线程
### Running
绿色，需要关注两点：  
1.是否应用的本身逻辑耗时，比如某些代码逻辑导致  
2.是否跑在了对应的核心上，可能跑在了小核上，一般 UI 线程和 RenderThread 都是在大核上  
### Runnable
蓝色，一般线程的状态转换是这样子的：  
![image](https://github.com/user-attachments/assets/e78d2c1a-1c3c-468a-a0b9-914c4ad62b16)    
正常情况下，应用进入 Runnable 状态之后，会马上被调度器调度，进入 Running 状态，开始干活；但是在系统繁忙的时候，应用就会有大量的时间在 Runnable 状态，因为 cpu 已经跑满，各种任务都需要排队等待调度，如果应用启动的时候出现大量的 Runnable 任务，那么需要查看系统的状态
# HWC
HWC（Hardware Composer）是 Android 中进行窗口（Layer）合成和显示的 HAL 层模块，其实现是特定于设备的，而且通常由显示设备制造商 (OEM)完成，为 SurfaceFlinger 服务提供硬件支持。   
SurfaceFlinger 可以使用 OpenGL ES 合成 Layer，这需要占用并消耗 GPU 资源。大多数 GPU 都没有针对图层合成进行优化，当 SurfaceFlinger 通过 GPU 合成图层时，应用程序无法使用 GPU 进行自己的渲染。而 HWC 通过硬件设备进行图层合成，可以减轻 GPU 的合成压力 。  
1.SurfaceFlinge r向 HWC 提供所有 Layer 的完整列表，让 HWC 根据其硬件能力，决定如何处理这些 Layer。  
2.HWC 会为每个 Layer 标注合成方式，是通过 GPU 还是通过 HWC 合成。  
3.SurfaceFlinger 负责先把所有注明 GPU 合成的 Layer 合成到一个输出 Buffer，然后把这个输出 Buffer 和其他 Layer（注明 HWC 合成的 Layer）一起交给 HWC，让 HWC 完成剩余 Layer 的合成和显示。
# meminfo
当出现内存偏高或其他内存相关的问题时，我们可以通过 meminfo 来缩小范围、查找潜在的风险点  
获取方式：adb shell dumpsys meminfo <pkg>  
meminfo统计：  
![image](https://github.com/user-attachments/assets/2c26f42e-2494-4f9b-80ef-0460746b48ef)  
对照 meminfo 来看：  
  ![image](https://github.com/user-attachments/assets/594eb71d-63d4-4701-9ff7-118565932f28)  
纵轴：  
![image](https://github.com/user-attachments/assets/52028878-2de4-4f24-8b63-7c674513f70e)  
横轴：  
![image](https://github.com/user-attachments/assets/95f30ffb-ce1a-4321-a7f1-851bd3f5fc59)  
App Summary:  
![image](https://github.com/user-attachments/assets/2a57c902-f440-455b-bd04-0697365c48e7)
# hprof
heap profile，是某一时间点，应用进程堆转存生成的文件，包含了这一时间点的内存快照；当需要进一步拆解内存、定位问题时，进行获取并分析    
![image](https://github.com/user-attachments/assets/4ba978f7-7d5d-44a4-b06a-24d684ff6145)
# 埋点体系
## 目标
1.精准定位每个阶段耗时  
2.线上可采集（低开销）  
3.与业务代码解耦  
4.支持多维度分析（启动/页面/网络/渲染）  
## 分层
Layer 1: 系统级（Perfetto 自动采集，无需代码）  
Layer 2: 框架级（SDK/基础库关键路径）  
Layer 3: 业务级（具体业务模块）  
## 埋点实现
```
// ================================
// 核心封装：TraceCompat
// 同时支持线上（轻量）和 Perfetto（详细）
// ================================
object AppTrace {
    
    // 是否开启详细追踪（Debug/测试环境开启）
    private var detailEnabled = BuildConfig.DEBUG
    
    // 线上轻量追踪（始终开启，极低开销）
    private val onlineTracer = OnlineTracer()
    
    inline fun section(tag: String, block: () -> Unit) {
        begin(tag)
        try {
            block()
        } finally {
            end()
        }
    }
    
    fun begin(tag: String) {
        // 1. Perfetto/Systrace 追踪（仅 Debug）
        if (detailEnabled) {
            TraceCompat.beginSection(tag)
        }
        // 2. 线上时间戳记录（始终执行，开销极小）
        onlineTracer.begin(tag)
    }
    
    fun end() {
        if (detailEnabled) {
            TraceCompat.endSection()
        }
        onlineTracer.end()
    }
    
    // 异步追踪（用于跨线程的事件）
    fun beginAsync(tag: String, cookie: Int) {
        if (detailEnabled) {
            TraceCompat.beginAsyncSection(tag, cookie)
        }
        onlineTracer.beginAsync(tag, cookie)
    }
    
    fun endAsync(tag: String, cookie: Int) {
        if (detailEnabled) {
            TraceCompat.endAsyncSection(tag, cookie)
        }
        onlineTracer.endAsync(tag, cookie)
    }
}
```

```
// 线上环境：不使用 Perfetto（用户设备无法抓trace）
// 改为：记录时间戳，启动结束后批量上报
class OnlineTracer {
    
    data class TraceEvent(
        val tag: String,
        val startTime: Long,
        var endTime: Long = 0,
        val threadName: String = Thread.currentThread().name
    )
    
    // 使用 ThreadLocal 避免锁竞争
    private val threadLocalStack = ThreadLocal<ArrayDeque<TraceEvent>>()
    
    // 已完成的事件（线程安全）
    private val completedEvents = ConcurrentLinkedQueue<TraceEvent>()
    
    fun begin(tag: String) {
        val stack = threadLocalStack.get() ?: ArrayDeque<TraceEvent>().also {
            threadLocalStack.set(it)
        }
        stack.addLast(TraceEvent(
            tag = tag,
            startTime = SystemClock.elapsedRealtime()
        ))
    }
    
    fun end() {
        val stack = threadLocalStack.get() ?: return
        val event = stack.removeLastOrNull() ?: return
        event.endTime = SystemClock.elapsedRealtime()
        completedEvents.add(event)
    }
    
    // 生成上报数据
    fun generateReport(): Map<String, Long> {
        return completedEvents
            .groupBy { it.tag }
            .mapValues { (_, events) ->
                events.sumOf { it.endTime - it.startTime }
            }
    }
}
```

```
// ================================
// 启动阶段标准埋点
// ================================
class MyApplication : Application() {
    
    override fun attachBaseContext(base: Context) {
        // 进程创建时间（最早的时间点）
        StartupMonitor.markProcessStart()
        super.attachBaseContext(base)
        
        AppTrace.begin("App:attachBaseContext")
        // MultiDex 等操作
        AppTrace.end()
    }
    
    override fun onCreate() {
        AppTrace.begin("App:Application.onCreate")
        super.onCreate()
        
        AppTrace.section("App:initCrashSDK") {
            CrashSDK.init(this)
        }
        
        AppTrace.section("App:initRouter") {
            Router.init(this)
        }
        
        AppTrace.section("App:initImageLoader") {
            ImageLoader.init(this)
        }
        
        // 异步任务用异步追踪
        val cookie = System.identityHashCode(this)
        AppTrace.beginAsync("App:asyncInit", cookie)
        GlobalScope.launch(Dispatchers.IO) {
            PushSDK.init(this@MyApplication)
            AppTrace.endAsync("App:asyncInit", cookie)
        }
        
        AppTrace.end() // Application.onCreate 结束
    }
}
```

```
// ================================
// Activity 生命周期自动埋点
// 通过 ActivityLifecycleCallbacks 统一注入
// 无需每个 Activity 手动添加
// ================================
class PerformanceLifecycleCallback : Application.ActivityLifecycleCallbacks {
    
    private val activityStartTime = WeakHashMap<Activity, Long>()
    
    override fun onActivityCreated(activity: Activity, bundle: Bundle?) {
        val tag = "Activity:${activity.javaClass.simpleName}.onCreate"
        AppTrace.begin(tag)
        activityStartTime[activity] = SystemClock.elapsedRealtime()
    }
    
    override fun onActivityStarted(activity: Activity) {
        // onCreate 结束，onStart 开始
        AppTrace.end() // 结束 onCreate 的 section
        AppTrace.begin("Activity:${activity.javaClass.simpleName}.onStart")
    }
    
    override fun onActivityResumed(activity: Activity) {
        AppTrace.end() // 结束 onStart
        AppTrace.begin("Activity:${activity.javaClass.simpleName}.onResume")
    }
    
    override fun onActivityWindowFocusChanged(
        activity: Activity, 
        hasFocus: Boolean
    ) {
        if (hasFocus) {
            AppTrace.end() // 结束 onResume
            
            // 首帧时间（最关键指标）
            val startTime = activityStartTime[activity] ?: return
            val firstFrameTime = SystemClock.elapsedRealtime() - startTime
            
            StartupMonitor.reportFirstFrame(
                activityName = activity.javaClass.simpleName,
                costMs = firstFrameTime
            )
        }
    }
    
    // 其他回调省略...
}

// Application 中注册
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        registerActivityLifecycleCallbacks(PerformanceLifecycleCallback())
    }
}
```

```
object StartupMonitor {
    
    private var processStartTime = 0L
    private var appCreateEndTime = 0L
    private var firstFrameTime = 0L
    
    fun markProcessStart() {
        processStartTime = SystemClock.elapsedRealtime()
    }
    
    fun markAppCreateEnd() {
        appCreateEndTime = SystemClock.elapsedRealtime()
    }
    
    fun reportFirstFrame(activityName: String, costMs: Long) {
        firstFrameTime = SystemClock.elapsedRealtime()
        
        val report = StartupReport(
            // 总冷启动耗时
            totalCost = firstFrameTime - processStartTime,
            // Application 阶段耗时
            appCost = appCreateEndTime - processStartTime,
            // Activity 到首帧耗时
            activityCost = costMs,
            // 各阶段明细
            details = AppTrace.onlineTracer.generateReport(),
            // 设备信息
            deviceModel = Build.MODEL,
            androidVersion = Build.VERSION.SDK_INT,
            // 是否低端机
            isLowEndDevice = isLowEndDevice()
        )
        
        // 上报到 APM 平台
        APMReporter.report("cold_start", report)
    }
    
    private fun isLowEndDevice(): Boolean {
        val am = context.getSystemService(ActivityManager::class.java)
        return am.isLowRamDevice || 
               getTotalRam() < 3 * 1024L * 1024 * 1024 // 3GB以下
    }
}
```
