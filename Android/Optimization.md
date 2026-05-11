# 卡顿优化
## 主线程耗时	
### StrictMode 检测
```
// StrictMode：开发阶段检测主线程违规操作
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()      // 检测主线程磁盘读
                    .detectDiskWrites()     // 检测主线程磁盘写
                    .detectNetwork()        // 检测主线程网络请求
                    .detectCustomSlowCalls() // 检测自定义慢调用
                    .penaltyLog()           // 打印日志
                    .penaltyDialog()        // 弹窗提示（开发阶段）
                    .build()
            )

            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()  // 检测 Cursor 泄漏
                    .detectLeakedClosableObjects() // 检测 IO 流泄漏
                    .detectActivityLeaks()         // 检测 Activity 泄漏
                    .penaltyLog()
                    .build()
            )
        }
    }
}

// 标记自定义慢调用
StrictMode.noteSlowCall("heavy_computation")
```
### 主线程 IO/锁等待
1.IO:  
```
// ================================
// 检测：StrictMode（开发阶段）
// ================================
StrictMode.setThreadPolicy(
    StrictMode.ThreadPolicy.Builder()
        .detectDiskReads()
        .detectDiskWrites()
        .detectNetwork()
        .penaltyLog()
        .build()
)

// ================================
// 常见主线程 IO 场景与优化
// ================================

// 场景一：主线程读取 SharedPreferences
// ❌ 问题
val sp = getSharedPreferences("config", MODE_PRIVATE)
val value = sp.getString("key", "") // 首次可能触发文件IO

// ✅ 优化：提前异步预加载（见启动优化章节）
// 或使用 DataStore（基于协程，天然异步）
val dataStore = context.createDataStore("config")
lifecycleScope.launch {
    val value = dataStore.data.map { it[KEY] }.first()
}

// 场景二：主线程数据库操作
// ❌ 问题
val user = db.userDao().getUser(id) // 主线程DB查询！

// ✅ 优化：Room 强制异步（suspend函数）
lifecycleScope.launch {
    val user = withContext(Dispatchers.IO) {
        db.userDao().getUser(id)
    }
    updateUI(user)
}

// 场景三：主线程文件操作
// ❌ 问题
val config = File(filesDir, "config.json").readText()

// ✅ 优化
lifecycleScope.launch {
    val config = withContext(Dispatchers.IO) {
        File(filesDir, "config.json").readText()
    }
    parseConfig(config)
}
```

2.锁等待：  
```
// ================================
// 场景：主线程等待锁
// ================================

// ❌ 问题：主线程持有/等待锁
class DataManager {
    private val lock = Object()
    private var cachedData: Data? = null

    // 子线程持有锁做耗时操作
    fun updateDataInBackground() {
        thread {
            synchronized(lock) {
                Thread.sleep(500) // 耗时操作持有锁
                cachedData = fetchNewData()
            }
        }
    }

    // 主线程等待同一把锁 → 主线程阻塞500ms！
    fun getDataOnMainThread(): Data? {
        synchronized(lock) { // 等待子线程释放锁
            return cachedData
        }
    }
}

// ✅ 优化方案一：读写分离（CopyOnWrite）
class DataManagerV2 {

    // volatile 保证可见性，无需锁
    @Volatile
    private var cachedData: Data? = null

    fun updateDataInBackground() {
        thread {
            val newData = fetchNewData() // 耗时操作不持锁
            cachedData = newData         // 原子赋值，volatile保证可见性
        }
    }

    fun getDataOnMainThread(): Data? {
        return cachedData // 直接读，无锁
    }
}

// ✅ 优化方案二：无锁数据结构
class DataManagerV3 {
    // AtomicReference：无锁线程安全
    private val dataRef = AtomicReference<Data?>(null)

    fun updateDataInBackground() {
        thread {
            val newData = fetchNewData()
            dataRef.set(newData) // 原子操作，无锁
        }
    }

    fun getDataOnMainThread(): Data? {
        return dataRef.get() // 原子读，无锁
    }
}

// ✅ 优化方案三：主线程不等待，用回调
class DataManagerV4 {
    private val executor = Executors.newSingleThreadExecutor()

    fun getDataAsync(callback: (Data?) -> Unit) {
        executor.execute {
            val data = fetchData() // 子线程执行
            Handler(Looper.getMainLooper()).post {
                callback(data) // 回调到主线程
            }
        }
    }
}
```
## 卡顿监控	
### 死锁检测
```
// ================================
// 死锁检测
// ================================
object DeadlockDetector {

    fun detect(): List<ThreadInfo> {
        val threadMXBean = ManagementFactory.getThreadMXBean()

        // 检测死锁线程
        val deadlockedIds = threadMXBean.findDeadlockedThreads()
            ?: return emptyList()

        return threadMXBean.getThreadInfo(deadlockedIds, true, true)
            .map { info ->
                ThreadInfo(
                    threadName = info.threadName,
                    state = info.threadState.name,
                    lockName = info.lockName ?: "unknown",
                    lockOwnerName = info.lockOwnerName ?: "none",
                    stackTrace = info.stackTrace.joinToString("\n") { "\tat $it" }
                )
            }
    }

    data class ThreadInfo(
        val threadName: String,
        val state: String,
        val lockName: String,
        val lockOwnerName: String,
        val stackTrace: String
    )
}

// 定期检测死锁
thread(name = "deadlock-detector", isDaemon = true) {
    while (true) {
        Thread.sleep(5000)
        val deadlocks = DeadlockDetector.detect()
        if (deadlocks.isNotEmpty()) {
            Log.e("DeadlockDetector", "检测到死锁！")
            deadlocks.forEach { Log.e("DeadlockDetector", it.toString()) }
        }
    }
}
```
### Looper 消息监控
```
// 原理：主线程所有任务都通过 Looper 执行
// 监控每条消息的处理时间
// 超过阈值 → 卡顿

// 原理：Looper 处理每条消息前后调用 Printer
// 通过时间差判断消息处理是否超时

class LooperMonitor private constructor() : MessageQueue.IdleHandler {

    companion object {
        val instance by lazy { LooperMonitor() }
        private const val JANK_THRESHOLD = 100L    // 轻微卡顿阈值
        private const val BLOCK_THRESHOLD = 3000L  // 严重卡顿阈值
    }

    private var msgStartTime = 0L
    private var msgStartWallTime = 0L
    private var isMonitoring = false

    // 堆栈采集器（异步采集，不影响主线程）
    private val stackSampler = StackSampler()

    private val printer = object : Printer {
        override fun println(msg: String) {
            if (!isMonitoring) return

            if (msg.startsWith(">>>>>")) {
                // 消息开始处理
                onMessageStart()
            } else if (msg.startsWith("<<<<<")) {
                // 消息处理完成
                onMessageEnd(msg)
            }
        }
    }

    private fun onMessageStart() {
        msgStartTime = SystemClock.elapsedRealtime()
        msgStartWallTime = System.currentTimeMillis()

        // 开始异步采集堆栈
        // 在消息处理期间，每隔一段时间采集一次主线程堆栈
        stackSampler.startSampling(Looper.getMainLooper().thread)
    }

    private fun onMessageEnd(msg: String) {
        val cost = SystemClock.elapsedRealtime() - msgStartTime
        stackSampler.stopSampling()

        when {
            cost >= BLOCK_THRESHOLD -> {
                // 严重卡顿：上报完整堆栈
                val stacks = stackSampler.getCollectedStacks()
                reportBlock(cost, msg, stacks)
            }
            cost >= JANK_THRESHOLD -> {
                // 轻微卡顿：上报最后一次堆栈
                val lastStack = stackSampler.getLastStack()
                reportJank(cost, msg, lastStack)
            }
        }
    }

    fun install() {
        isMonitoring = true
        Looper.getMainLooper().setMessageLogging(printer)
        // 注册 IdleHandler：主线程空闲时的回调
        Looper.myQueue().addIdleHandler(this)
    }

    // IdleHandler：主线程空闲时触发
    // 可以在这里做一些低优先级的工作
    override fun queueIdle(): Boolean {
        // 返回 true = 保持注册，下次空闲继续回调
        return true
    }
}

// ================================
// 堆栈采样器（子线程定时采集主线程堆栈）
// ================================
class StackSampler {

    private val sampleInterval = 20L // 每20ms采集一次
    private val maxSamples = 100     // 最多保留100次采样
    private val samples = ArrayDeque<StackEntry>()

    @Volatile private var isSampling = false
    private var samplingThread: Thread? = null
    private var targetThread: Thread? = null

    data class StackEntry(
        val timestamp: Long,
        val stack: String,
        val cpuTime: Long
    )

    fun startSampling(thread: Thread) {
        targetThread = thread
        isSampling = true
        samples.clear()

        samplingThread = thread(name = "stack-sampler", isDaemon = true) {
            while (isSampling) {
                try {
                    val stack = thread.stackTrace
                        .joinToString("\n") { "\tat $it" }

                    synchronized(samples) {
                        if (samples.size >= maxSamples) {
                            samples.removeFirst()
                        }
                        samples.addLast(StackEntry(
                            timestamp = SystemClock.elapsedRealtime(),
                            stack = stack,
                            cpuTime = Debug.threadCpuTimeNanos()
                        ))
                    }
                    Thread.sleep(sampleInterval)
                } catch (e: InterruptedException) {
                    break
                }
            }
        }
    }

    fun stopSampling() {
        isSampling = false
        samplingThread?.interrupt()
    }

    fun getCollectedStacks(): List<StackEntry> {
        return synchronized(samples) { samples.toList() }
    }

    fun getLastStack(): StackEntry? {
        return synchronized(samples) { samples.lastOrNull() }
    }
}

// 安装监控
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Looper.getMainLooper().setMessageLogging(LooperMonitor())
    }
}
```
### Choreographer 帧监控
```
// 原理：Choreographer 每帧回调
// 计算相邻两帧的时间差
// 超过 N 个 VSync 周期 → 丢帧

class ChoreographerMonitor : Choreographer.FrameCallback {

    private var lastFrameTimeNs = 0L
    private val vsyncPeriodNs = 16_666_666L // 60fps

    override fun doFrame(frameTimeNanos: Long) {
        if (lastFrameTimeNs != 0L) {
            val costNs = frameTimeNanos - lastFrameTimeNs
            val droppedFrames = (costNs / vsyncPeriodNs).toInt() - 1

            if (droppedFrames > 0) {
                // 丢帧了！
                onFrameDropped(droppedFrames, costNs / 1_000_000)
            }
        }
        lastFrameTimeNs = frameTimeNanos

        // 持续监控
        Choreographer.getInstance().postFrameCallback(this)
    }

    private fun onFrameDropped(droppedFrames: Int, costMs: Long) {
        when {
            droppedFrames >= 25 -> // 冻帧（>416ms）
                reportFrozenFrame(droppedFrames, costMs)
            droppedFrames >= 3  -> // 严重卡顿
                reportSeriousJank(droppedFrames, costMs)
            else                -> // 轻微卡顿
                reportJank(droppedFrames, costMs)
        }
    }
}

// 安装
Choreographer.getInstance().postFrameCallback(ChoreographerMonitor())
```
### FrameMetrics API
```
// 最精准，可以拿到每帧各阶段耗时

class FrameMetricsMonitor(private val activity: Activity) {

    private val listener = Window.OnFrameMetricsAvailableListener {
        _, frameMetrics, _ ->

        // 各阶段耗时（纳秒）
        val totalDuration = frameMetrics
            .getMetric(FrameMetrics.TOTAL_DURATION) / 1_000_000

        val layoutMeasure = frameMetrics
            .getMetric(FrameMetrics.LAYOUT_MEASURE_DURATION) / 1_000_000

        val draw = frameMetrics
            .getMetric(FrameMetrics.DRAW_DURATION) / 1_000_000

        val gpuDuration = frameMetrics
            .getMetric(FrameMetrics.GPU_DURATION) / 1_000_000

        val inputHandling = frameMetrics
            .getMetric(FrameMetrics.INPUT_HANDLING_DURATION) / 1_000_000

        val animationDuration = frameMetrics
            .getMetric(FrameMetrics.ANIMATION_DURATION) / 1_000_000

        // 判断是否卡顿
        if (totalDuration > 16) {
            // 上报详细的帧耗时数据
            PerformanceReporter.reportSlowFrame(
                totalMs = totalDuration,
                layoutMs = layoutMeasure,
                drawMs = draw,
                gpuMs = gpuDuration,
                inputMs = inputHandling,
                animMs = animationDuration
            )
        }
    }

    fun start() {
        activity.window.addOnFrameMetricsAvailableListener(
            listener,
            Handler(Looper.getMainLooper())
        )
    }

    fun stop() {
        activity.window.removeOnFrameMetricsAvailableListener(listener)
    }
}
```
### WatchDog 机制
思路：  
```
开启一个子线程，定期向主线程发送任务
如果任务在规定时间内没有被执行
→ 说明主线程被阻塞了 → 卡顿！

时序：
子线程：[发送任务] ----等待N毫秒----> [检查是否执行]
主线程：[正常处理消息]...[处理WatchDog任务]
                              ↑
                    如果这里超时 = 卡顿
```

实现：  
```
// 在渲染优化章节的基础上，增加更多能力

class EnhancedWatchDog(
    private val blockThreshold: Long = 3000L,
    private val anrThreshold: Long = 5000L
) {
    private val mainHandler = Handler(Looper.getMainLooper())
    private val stackSampler = StackSampler()

    @Volatile private var tickCount = 0L
    @Volatile private var lastTickCount = -1L
    @Volatile private var blockStartTime = 0L

    private val tickRunnable = Runnable { tickCount++ }

    fun start() {
        mainHandler.post(tickRunnable)

        thread(name = "enhanced-watchdog", isDaemon = true) {
            while (true) {
                Thread.sleep(1000L)
                checkMainThread()
            }
        }
    }

    private fun checkMainThread() {
        if (tickCount != lastTickCount) {
            // 主线程正常
            lastTickCount = tickCount
            blockStartTime = 0L
            stackSampler.stopSampling()
            mainHandler.post(tickRunnable)
            return
        }

        // 主线程可能阻塞
        val now = SystemClock.elapsedRealtime()

        if (blockStartTime == 0L) {
            // 刚开始阻塞，启动堆栈采集
            blockStartTime = now
            stackSampler.startSampling(Looper.getMainLooper().thread)
            return
        }

        val blockDuration = now - blockStartTime

        when {
            blockDuration >= anrThreshold -> {
                // ANR 级别
                val stacks = stackSampler.getCollectedStacks()
                reportAnrLevel(blockDuration, stacks)
                // 同时读取 /data/anr/traces.txt（如果有权限）
                readAnrTraces()
            }
            blockDuration >= blockThreshold -> {
                // 严重卡顿
                val stacks = stackSampler.getCollectedStacks()
                reportBlock(blockDuration, stacks)
            }
        }
    }

    private fun readAnrTraces() {
        try {
            val tracesFile = File("/data/anr/traces.txt")
            if (tracesFile.exists() && tracesFile.canRead()) {
                val content = tracesFile.readText()
                // 解析 traces 文件，提取关键信息
                parseAnrTraces(content)
            }
        } catch (e: Exception) {
            // 无权限，忽略
        }
    }
}

// 使用
class MyApplication : Application() {
    private val watchDog = WatchDog(threshold = 3000L)

    override fun onCreate() {
        super.onCreate()
        if (!BuildConfig.DEBUG) { // 线上开启
            watchDog.start()
        }
    }
}
```

WatchDog vs Looper Printer:  
```
┌──────────────────┬─────────────────┬──────────────────┐
│                  │ Looper Printer  │ WatchDog         │
├──────────────────┼─────────────────┼──────────────────┤
│ 检测原理         │ 消息处理前后回调 │ 定时检测心跳      │
│ 精准度           │ 高（每条消息）  │ 中（有检测间隔）  │
│ 性能开销         │ 低              │ 极低             │
│ 能否检测死锁     │ 否              │ 能               │
│ 堆栈采集时机     │ 消息结束时      │ 阻塞期间随时      │
│ 适用场景         │ 开发/测试       │ 线上             │
└──────────────────┴─────────────────┴──────────────────┘
```
## Binder
### 主线程 Binder 耗时
1.异步预加载 SP：主线程读取 SP，可能触发 Binder 到 system_server：
```
class MyApplication : Application() {
    override fun attachBaseContext(base: Context) {
        super.attachBaseContext(base)
        // 尽早在子线程触发 SP 加载
        // 等主线程需要时，大概率已加载完毕
        thread {
            base.getSharedPreferences("config", MODE_PRIVATE)
            base.getSharedPreferences("user", MODE_PRIVATE)
        }
    }
}
```

2.缓存PMS查询结果，批量查询：每次 getPackageInfo 都是 Binder 调用:
```
object PackageCache {
    private val packageInfoCache = ConcurrentHashMap<String, Boolean>()

    fun isPackageInstalled(packageName: String): Boolean {
        // 先查内存缓存
        packageInfoCache[packageName]?.let { return it }

        // 缓存未命中，子线程查询（不能在主线程）
        // 这里返回默认值，异步更新缓存
        GlobalScope.launch(Dispatchers.IO) {
            val installed = try {
                context.packageManager.getPackageInfo(packageName, 0)
                true
            } catch (e: PackageManager.NameNotFoundException) {
                false
            }
            packageInfoCache[packageName] = installed
        }
        return false // 返回默认值
    }

    // 启动时批量预查询（子线程）
    fun preloadPackageInfo(packages: List<String>) {
        GlobalScope.launch(Dispatchers.IO) {
            packages.forEach { pkg ->
                val installed = try {
                    context.packageManager.getPackageInfo(pkg, 0)
                    true
                } catch (e: Exception) { false }
                packageInfoCache[pkg] = installed
            }
        }
    }
}
```

3.异步 CR 查询，使用 CursorLoader 或协程：查询 ContentProvider（如联系人、媒体库）,每次 query 都是跨进程 Binder 调用:  
```
suspend fun queryContacts(): List<Contact> =
        withContext(Dispatchers.IO) {
            val contacts = mutableListOf<Contact>()
            context.contentResolver.query(
                ContactsContract.Contacts.CONTENT_URI,
                arrayOf(
                    ContactsContract.Contacts._ID,
                    ContactsContract.Contacts.DISPLAY_NAME
                ),
                null, null, null
            )?.use { cursor ->
                while (cursor.moveToNext()) {
                    contacts.add(
                        Contact(
                            id = cursor.getLong(0),
                            name = cursor.getString(1)
                        )
                    )
                }
            }
            contacts
        }
```

4. ActivityManager 优化：  
```
    private var cachedMemoryClass = -1

    fun getMemoryClass(): Int {
        if (cachedMemoryClass == -1) {
            // 只查询一次，缓存结果
            val am = context.getSystemService(ActivityManager::class.java)
            cachedMemoryClass = am.memoryClass
        }
        return cachedMemoryClass
    }
```
### 调用 Binder 线程池耗尽
每个进程的 Binder 线程池默认最多 15 个线程，大量异步任务同时发起 Binder 调用 -> 每个调用都在等待 system_server 响应 -> 新的 Binder 请求：只能排队等待 -> 主线程的 Binder 调用也被阻塞 -> ANR

```
// adb shell cat /proc/[pid]/status | grep Threads
// 观察线程数是否接近 15（Binder线程上限）

// 监控 Binder 线程池使用情况
object BinderThreadMonitor {
    fun checkBinderThreadPool(): BinderThreadInfo {
        return try {
            // /proc/[pid]/status 包含线程数信息
            val statusFile = File("/proc/${Process.myPid()}/status")
            val content = statusFile.readText()

            // 解析线程数
            val threads = content.lines()
                .find { it.startsWith("Threads:") }
                ?.split("\\s+".toRegex())
                ?.getOrNull(1)
                ?.toIntOrNull() ?: 0

            // Binder 线程通常命名为 Binder:pid_N
            val binderThreadCount = Thread.getAllStackTraces()
                .keys
                .count { it.name.startsWith("Binder:") }

            BinderThreadInfo(
                totalThreads = threads,
                binderThreads = binderThreadCount,
                isPoolExhausted = binderThreadCount >= 15 // 默认上限15
            )
        } catch (e: Exception) {
            BinderThreadInfo(0, 0, false)
        }
    }

    data class BinderThreadInfo(
        val totalThreads: Int,
        val binderThreads: Int,
        val isPoolExhausted: Boolean
    )
}

// 优化方案：控制并发 Binder 调用数量
class BinderCallLimiter {
    // 限制同时进行的 Binder 调用数量
    private val semaphore = Semaphore(5) // 最多5个并发
    
    suspend fun <T> execute(block: suspend () -> T): T {
        semaphore.acquire()
        return try {
            block()
        } finally {
            semaphore.release()
        }
    }
}
```
### 跨进程调用耗时监控
动态代理：  
```
// 自定义 Binder 耗时监控
// 通过动态代理拦截 Binder 调用
object BinderMonitor {
    
    fun install() {
        // Hook ServiceManager，拦截所有 getService 调用
        // 在每次 Binder 调用前后记录时间
        
        val serviceManagerClass = Class.forName("android.os.ServiceManager")
        val getServiceMethod = serviceManagerClass
            .getDeclaredMethod("getService", String::class.java)
        
        // 使用反射替换为代理实现
        // 记录每次 Binder 调用的耗时和调用栈
    }
    
    data class BinderRecord(
        val serviceName: String,
        val costMs: Long,
        val callStack: String,
        val threadName: String
    )
    
    private val records = ConcurrentLinkedQueue<BinderRecord>()
    
    fun report(): List<BinderRecord> {
        return records
            .sortedByDescending { it.costMs }
            .take(20) // 上报最慢的20次 Binder 调用
    }
}
```

缺点：  
兼容性：  
需要适配各Android版本。  
反射的性能开销：  
1.Class.forName()：类查找，有缓存，第一次慢  
2.getDeclaredMethod()：方法查找，每次都有开销  
3.method.invoke()：比直接调用慢 10~50 倍  
4.在启动关键路径上用反射 = 拆东墙补西墙  

编译期插桩：  
```
// 原理：编译时通过 ASM 字节码插桩
// 在每个 Binder 调用前后自动插入监控代码
// 运行时没有反射，只是普通方法调用

// Gradle Transform 插件
class BinderMonitorTransform : Transform() {

    override fun transform(invocation: TransformInvocation) {
        invocation.inputs.forEach { input ->
            input.jarInputs.forEach { jarInput ->
                // 扫描所有字节码
                processJar(jarInput, invocation.outputProvider)
            }
        }
    }

    private fun processClass(classBytes: ByteArray): ByteArray {
        val reader = ClassReader(classBytes)
        val writer = ClassWriter(reader, ClassWriter.COMPUTE_FRAMES)

        // ASM 访问者模式，拦截方法调用
        val visitor = BinderCallVisitor(writer)
        reader.accept(visitor, ClassReader.EXPAND_FRAMES)

        return writer.toByteArray()
    }
}

// ASM 方法访问者：拦截 Binder 调用
class BinderCallVisitor(cv: ClassVisitor) : ClassVisitor(ASM9, cv) {

    override fun visitMethod(...): MethodVisitor {
        val mv = super.visitMethod(...)
        return BinderMethodVisitor(mv)
    }
}

class BinderMethodVisitor(mv: MethodVisitor) 
    : MethodVisitor(ASM9, mv) {

    override fun visitMethodInsn(
        opcode: Int, owner: String,
        name: String, descriptor: String, isInterface: Boolean
    ) {
        // 检测是否是 Binder 的 transact 调用
        if (owner == "android/os/IBinder" && name == "transact") {
            // 在 transact 调用前插入：记录开始时间
            mv.visitMethodInsn(
                INVOKESTATIC,
                "com/xxx/monitor/BinderMonitor",
                "onBinderCallBegin",
                "()V", false
            )

            // 原始 transact 调用
            super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)

            // 在 transact 调用后插入：记录结束时间
            mv.visitMethodInsn(
                INVOKESTATIC,
                "com/xxx/monitor/BinderMonitor",
                "onBinderCallEnd",
                "()V", false
            )
        } else {
            super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)
        }
    }
}

// 运行时监控类（被插桩代码调用）
// 没有任何反射！只是普通静态方法
object BinderMonitor {
    private val startTimeThreadLocal = ThreadLocal<Long>()

    @JvmStatic
    fun onBinderCallBegin() {
        startTimeThreadLocal.set(SystemClock.elapsedRealtime())
    }

    @JvmStatic
    fun onBinderCallEnd() {
        val cost = SystemClock.elapsedRealtime() -
                   (startTimeThreadLocal.get() ?: return)
        if (cost > THRESHOLD_MS) {
            recordSlowBinder(cost, getCallStack())
        }
    }

    private const val THRESHOLD_MS = 10L
}
```

方法签名解析：  
```
// 你想拦截所有 Binder.transact() 调用
// 但 transact 的签名是：
// transact(int code, Parcel data, Parcel reply, int flags): Boolean

// ASM 中方法签名是描述符格式：
// (ILandroid/os/Parcel;Landroid/os/Parcel;I)Z
// I = int, L...;= 对象类型, Z = boolean

// 难点：你需要精确匹配描述符
// 否则：
// ① 匹配错误的方法（误插桩）
// ② 遗漏目标方法（漏插桩）

class BinderCallVisitor(mv: MethodVisitor) : MethodVisitor(ASM9, mv) {
    override fun visitMethodInsn(
        opcode: Int,
        owner: String,    // 类名：android/os/IBinder
        name: String,     // 方法名：transact
        descriptor: String, // 描述符：(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z
        isInterface: Boolean
    ) {
        // 必须同时匹配 owner + name + descriptor
        // 只匹配 name 会误伤其他类的同名方法！
        if (owner == "android/os/IBinder"
            && name == "transact"
            && descriptor == "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z"
        ) {
            // 插桩
        } else {
            super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)
        }
    }
}
```

栈帧计算：  
```
插桩前的字节码栈帧：
... [arg1] [arg2] → 调用 transact → [result]

插桩后需要：
... [arg1] [arg2] → 调用 monitorBegin → [arg1] [arg2] → 调用 transact → [result] → 调用 monitorEnd → [result]

问题：
① 插入 monitorBegin 时，栈上已有参数
   如果 monitorBegin 消费了栈上的值 → 参数丢失！
   需要在调用前 DUP（复制）参数

② 不同方法的参数数量不同
   transact 有4个参数，其他方法可能有1个或10个
   DUP 逻辑需要适配不同参数数量

③ COMPUTE_FRAMES 标志
   ClassWriter(ClassWriter.COMPUTE_FRAMES) 让 ASM 自动计算
   但有时会出错，需要手动处理

// 正确的插桩方式（不破坏栈）
override fun visitMethodInsn(...) {
    if (isTargetMethod(owner, name, descriptor)) {
        
        // 方案：在调用前后插入，不修改参数
        // 调用前：记录开始时间（不需要参数）
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/monitor/BinderMonitor",
            "begin",
            "()V",  // 无参数，不影响栈
            false
        )
        
        // 原始调用（参数还在栈上，不受影响）
        super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)
        
        // 调用后：记录结束时间（result还在栈上）
        // 注意：result 是 boolean，在栈顶
        // 如果 monitorEnd 需要 result，要先 DUP
        mv.visitInsn(DUP) // 复制栈顶的 boolean result
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/monitor/BinderMonitor",
            "end",
            "(Z)V",  // 接收 boolean，消费复制的那个
            false
        )
        // 原始 result 还在栈上，正常返回给调用方
    } else {
        super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)
    }
}
```
缺点：  
需要兼容AGP版本  
实现难度大  

/proc 文件监控：  
```
// 线上最佳方案：不用反射，不用插桩
// 利用系统的 Binder 统计接口

object BinderStats {

    // 读取系统提供的 Binder 统计信息
    // /proc/[pid]/wchan 显示线程等待原因
    // binder_transaction_pending = 在等待Binder
    fun getBinderWaitInfo(): String {
        return try {
            File("/proc/${Process.myPid()}/wchan").readText()
        } catch (e: Exception) { "" }
    }

    // 结合 Choreographer 监控
    // 在每帧回调中检查主线程是否在等待 Binder
    fun startMonitor() {
        Choreographer.getInstance().postFrameCallback(object :
            Choreographer.FrameCallback {
            override fun doFrame(frameTimeNanos: Long) {
                checkMainThreadBinder()
                Choreographer.getInstance().postFrameCallback(this)
            }
        })
    }

    private fun checkMainThreadBinder() {
        // 通过 /proc/[pid]/task/[mainThreadTid]/wchan
        // 判断主线程是否在等待 Binder
        val mainTid = Looper.getMainLooper().thread.id
        val wchan = try {
            File("/proc/${Process.myPid()}/task/$mainTid/wchan").readText()
        } catch (e: Exception) { return }

        if (wchan.contains("binder")) {
            // 主线程正在等待 Binder！记录调用栈
            val stackTrace = Looper.getMainLooper().thread.stackTrace
            reportBinderBlock(stackTrace)
        }
    }
}
```

```
// /proc 文件是内核虚拟文件系统（procfs）
// 读取它不是真正的磁盘IO！
// 而是：内核直接从内存数据结构生成内容

// 实测开销：
// /proc/[pid]/wchan 读取：约 0.05ms ~ 0.2ms
// 反射调用：约 0.5ms ~ 5ms
// 相差 10~100 倍

// 但频繁读取仍有开销，需要控制频率
object ProcMonitor {
    
    // 不要每帧都读，用采样策略
    private var lastCheckTime = 0L
    private const val CHECK_INTERVAL_MS = 100L // 每100ms检查一次
    
    fun checkMainThreadBinder() {
        val now = SystemClock.elapsedRealtime()
        if (now - lastCheckTime < CHECK_INTERVAL_MS) return
        lastCheckTime = now
        
        // 异步读取，不阻塞主线程
        // 注意：读取的是主线程的状态，但读取操作在子线程
        thread {
            val mainTid = Looper.getMainLooper().thread.id
            val wchan = readWchan(mainTid)
            if (wchan.contains("binder")) {
                // 主线程在等待Binder，上报
                reportBinderBlock()
            }
        }
    }
    
    private fun readWchan(tid: Long): String {
        return try {
            // 这是内核虚拟文件，不是真正的磁盘IO
            File("/proc/${Process.myPid()}/task/$tid/wchan").readText()
        } catch (e: Exception) { "" }
    }
}
```
缺点：  
/proc文件读取有开销（远小于反射）  
#### ASM 耗时监控插件
项目结构:  
```
project/
├── app/
│   └── build.gradle
├── plugin-timing/              ← 插件模块
│   ├── build.gradle
│   └── src/main/
│       ├── kotlin/
│       │   └── com/timing/
│       │       ├── TimingPlugin.kt        ← 插件入口
│       │       ├── TimingTransform.kt     ← AGP7以下
│       │       ├── TimingClassVisitorFactory.kt ← AGP7+
│       │       ├── TimingClassVisitor.kt  ← 类访问者
│       │       └── TimingMethodVisitor.kt ← 方法访问者
│       └── resources/
│           └── META-INF/gradle-plugins/
│               └── com.timing.plugin.properties
├── timing-runtime/             ← 运行时库模块
│   └── src/main/kotlin/
│       └── com/timing/
│           └── MethodTimer.kt  ← 被插桩代码调用
└── settings.gradle
```

运行时库（被插桩代码调用，无反射）:  
```
// timing-runtime/src/main/kotlin/com/timing/MethodTimer.kt

object MethodTimer {

    // 使用 ThreadLocal 存储每个线程的调用栈
    // 避免多线程竞争
    private val callStack = ThreadLocal<ArrayDeque<Long>>()

    // 阈值：超过此时间才记录
    private const val THRESHOLD_MS = 16L

    // 回调：外部注册，收到慢方法通知
    var onSlowMethod: ((methodName: String, costMs: Long) -> Unit)? = null

    // 方法开始时调用（由插桩代码调用）
    @JvmStatic
    fun onMethodEnter() {
        val stack = callStack.get() ?: ArrayDeque<Long>().also {
            callStack.set(it)
        }
        stack.addLast(System.currentTimeMillis())
    }

    // 方法结束时调用（由插桩代码调用）
    @JvmStatic
    fun onMethodExit(methodName: String) {
        val stack = callStack.get() ?: return
        val startTime = stack.removeLastOrNull() ?: return
        val costMs = System.currentTimeMillis() - startTime

        if (costMs >= THRESHOLD_MS) {
            onSlowMethod?.invoke(methodName, costMs)
            // 默认打印日志
            android.util.Log.w("MethodTimer",
                "慢方法: $methodName 耗时 ${costMs}ms")
        }
    }
}
```

MethodVisitor（核心插桩逻辑）:  
```
// TimingMethodVisitor.kt
import org.objectweb.asm.*
import org.objectweb.asm.Opcodes.*

class TimingMethodVisitor(
    methodVisitor: MethodVisitor,
    private val className: String,   // 如：com/xxx/MainActivity
    private val methodName: String,  // 如：onCreate
    private val descriptor: String   // 如：(Landroid/os/Bundle;)V
) : MethodVisitor(ASM9, methodVisitor) {

    // 方法的完整标识（用于日志显示）
    private val fullMethodName = "$className.$methodName$descriptor"

    // visitCode：方法体开始
    // 在第一条指令前插入 onMethodEnter()
    override fun visitCode() {
        super.visitCode()

        // 插入：MethodTimer.onMethodEnter()
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/timing/MethodTimer",  // 类名（/分隔）
            "onMethodEnter",           // 方法名
            "()V",                     // 描述符：无参数，无返回值
            false                      // 不是接口方法
        )
    }

    // visitInsn：访问无操作数指令
    // 在所有 RETURN 指令前插入 onMethodExit()
    override fun visitInsn(opcode: Int) {
        // 检测所有类型的 return 指令
        val isReturn = opcode in setOf(
            RETURN,   // void 方法返回
            IRETURN,  // int/boolean/byte/char/short 返回
            LRETURN,  // long 返回
            FRETURN,  // float 返回
            DRETURN,  // double 返回
            ARETURN,  // 对象引用返回
            ATHROW    // 抛出异常（也算方法结束）
        )

        if (isReturn) {
            insertMethodExit()
        }

        super.visitInsn(opcode)
    }

    private fun insertMethodExit() {
        // 插入：MethodTimer.onMethodExit("com/xxx/MainActivity.onCreate...")
        
        // 1. 将方法名字符串压栈
        mv.visitLdcInsn(fullMethodName)
        
        // 2. 调用 onMethodExit(String)
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/timing/MethodTimer",
            "onMethodExit",
            "(Ljava/lang/String;)V",  // 接收一个String参数
            false
        )
    }

    // 处理 try-catch：异常路径也需要插桩
    // visitTryCatchBlock 不需要修改
    // 但要确保 ATHROW 也被处理（上面已处理）
}
```

ClassVisitor（过滤哪些类/方法需要插桩）:  
```
// TimingClassVisitor.kt
import org.objectweb.asm.*
import org.objectweb.asm.Opcodes.*

class TimingClassVisitor(
    cv: ClassVisitor,
    private val config: TimingConfig
) : ClassVisitor(ASM9, cv) {

    private lateinit var currentClassName: String
    private var isInterface = false

    override fun visit(
        version: Int, access: Int, name: String,
        signature: String?, superName: String?,
        interfaces: Array<out String>?
    ) {
        currentClassName = name
        // 判断是否是接口（接口方法不插桩）
        isInterface = (access and ACC_INTERFACE) != 0
        super.visit(version, access, name, signature, superName, interfaces)
    }

    override fun visitMethod(
        access: Int, name: String, descriptor: String,
        signature: String?, exceptions: Array<out String>?
    ): MethodVisitor {

        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)

        // 过滤不需要插桩的方法
        if (shouldSkip(access, name)) {
            return mv  // 直接返回原始 mv，不插桩
        }

        // 返回插桩后的 MethodVisitor
        return TimingMethodVisitor(mv, currentClassName, name, descriptor)
    }

    private fun shouldSkip(access: Int, methodName: String): Boolean {
        // 跳过接口
        if (isInterface) return true

        // 跳过抽象方法（没有方法体）
        if (access and ACC_ABSTRACT != 0) return true

        // 跳过 native 方法
        if (access and ACC_NATIVE != 0) return true

        // 跳过构造方法（可选，根据需求）
        if (methodName == "<init>" || methodName == "<clinit>") return true

        // 跳过合成方法（编译器生成的）
        if (access and ACC_SYNTHETIC != 0) return true

        // 检查类名是否在白名单中
        if (!config.shouldInstrument(currentClassName)) return true

        return false
    }
}

// 插桩配置
data class TimingConfig(
    // 需要插桩的包名前缀
    val includePackages: List<String> = listOf("com/xxx/"),
    // 排除的包名前缀
    val excludePackages: List<String> = listOf(
        "com/timing/",          // 排除监控库本身
        "kotlin/",
        "kotlinx/",
        "androidx/",
        "android/"
    )
) {
    fun shouldInstrument(className: String): Boolean {
        // 必须在包含列表中
        val included = includePackages.any { className.startsWith(it) }
        if (!included) return false

        // 不能在排除列表中
        val excluded = excludePackages.any { className.startsWith(it) }
        return !excluded
    }
}
```

ClassVisitorFactory:  
```
// TimingClassVisitorFactory.kt
import com.android.build.api.instrumentation.*
import org.objectweb.asm.ClassVisitor

abstract class TimingClassVisitorFactory
    : AsmClassVisitorFactory<TimingParameters> {

    override fun createClassVisitor(
        classContext: ClassContext,
        nextClassVisitor: ClassVisitor
    ): ClassVisitor {
        val config = TimingConfig(
            includePackages = parameters.get().includePackages.get(),
            excludePackages = parameters.get().excludePackages.get()
        )
        return TimingClassVisitor(nextClassVisitor, config)
    }

    override fun isInstrumentable(classData: ClassData): Boolean {
        // 快速过滤，避免处理不需要插桩的类
        // 这里的过滤比 ClassVisitor 中更早，性能更好
        val className = classData.className.replace('.', '/')
        return parameters.get().includePackages.get()
            .any { className.startsWith(it) }
    }
}

// 插件参数接口
interface TimingParameters : InstrumentationParameters {
    @get:Input
    val includePackages: ListProperty<String>

    @get:Input
    val excludePackages: ListProperty<String>
}
```

Gradle 插件入口:  
```
// TimingPlugin.kt
import com.android.build.api.extension.AndroidComponentsExtension
import org.gradle.api.Plugin
import org.gradle.api.Project

class TimingPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        // 获取插件配置扩展
        val extension = project.extensions.create(
            "methodTiming",
            TimingExtension::class.java
        )

        // 等 Android 插件配置完成后再处理
        val androidComponents = project.extensions
            .getByType(AndroidComponentsExtension::class.java)

        androidComponents.onVariants { variant ->
            // 只在 debug 或指定变体中插桩
            if (!extension.enabledVariants.contains(variant.name)
                && !extension.enableAll) {
                return@onVariants
            }

            // 注册 ASM 转换
            variant.instrumentation.transformClassesWith(
                TimingClassVisitorFactory::class.java,
                InstrumentationScope.PROJECT  // 只处理项目代码，不处理依赖
            ) { params ->
                // 配置参数
                params.includePackages.set(extension.includePackages)
                params.excludePackages.set(extension.excludePackages)
            }

            // 让 AGP 自动计算栈帧
            variant.instrumentation.setAsmFramesComputationMode(
                FramesComputationMode.COMPUTE_FRAMES_FOR_INSTRUMENTED_METHODS
            )
        }
    }
}

// 插件配置 DSL
open class TimingExtension {
    var enableAll: Boolean = false
    var enabledVariants: List<String> = listOf("debug")
    var includePackages: List<String> = listOf()
    var excludePackages: List<String> = listOf(
        "kotlin/", "kotlinx/", "androidx/", "android/"
    )
    var thresholdMs: Long = 16L
}
```

注册：  
```
# plugin-timing/src/main/resources/
# META-INF/gradle-plugins/com.timing.plugin.properties

implementation-class=com.timing.TimingPlugin
```

```
// plugin-timing/build.gradle
plugins {
    id 'kotlin'
    id 'java-gradle-plugin'
}

dependencies {
    implementation gradleApi()
    implementation 'com.android.tools.build:gradle:7.4.0'
    implementation 'org.ow2.asm:asm:9.4'
    implementation 'org.ow2.asm:asm-commons:9.4'
    implementation 'org.ow2.asm:asm-util:9.4'  // 调试用
}

gradlePlugin {
    plugins {
        timingPlugin {
            id = 'com.timing.plugin'
            implementationClass = 'com.timing.TimingPlugin'
        }
    }
}
```

使用:  
```
// settings.gradle
includeBuild('plugin-timing')

// app/build.gradle
plugins {
    id 'com.android.application'
    id 'com.timing.plugin'  // 应用插件
}

// 配置插桩
methodTiming {
    enableAll = false
    enabledVariants = ['debug', 'release']
    includePackages = [
        'com/xxx/app/',      // 只插桩自己的代码
    ]
    excludePackages = [
        'com/xxx/app/generated/',  // 排除生成代码
    ]
    thresholdMs = 16  // 超过16ms才记录
}

dependencies {
    implementation project(':timing-runtime')
}
```

```
// Application 中配置回调
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 配置慢方法回调
        MethodTimer.onSlowMethod = { methodName, costMs ->
            // 上报到 APM 平台
            APMReporter.reportSlowMethod(
                method = methodName,
                costMs = costMs,
                thread = Thread.currentThread().name
            )
        }
    }
}
```

验证：  
```
// 插桩前的字节码（用 ASM Bytecode Viewer 查看）：
// public onCreate(Landroid/os/Bundle;)V
//   ALOAD 0
//   ALOAD 1
//   INVOKESPECIAL AppCompatActivity.onCreate
//   RETURN

// 插桩后的字节码：
// public onCreate(Landroid/os/Bundle;)V
//   INVOKESTATIC MethodTimer.onMethodEnter ()V   ← 插入
//   ALOAD 0
//   ALOAD 1
//   INVOKESPECIAL AppCompatActivity.onCreate
//   LDC "com/xxx/MainActivity.onCreate(Landroid/os/Bundle;)V"  ← 插入
//   INVOKESTATIC MethodTimer.onMethodExit (Ljava/lang/String;)V ← 插入
//   RETURN

// 运行时输出：
// W/MethodTimer: 慢方法: com/xxx/MainActivity.onCreate 耗时 234ms
// W/MethodTimer: 慢方法: com/xxx/HomeFragment.onViewCreated 耗时 89ms
```

完整流程：  
```
编译期：
源码(.kt/.java)
    ↓ kotlinc/javac
字节码(.class)
    ↓ AGP调用 TimingClassVisitorFactory
    ↓ ClassReader 读取每个 .class
    ↓ TimingClassVisitor 判断是否插桩
    ↓ TimingMethodVisitor 在方法入口/出口插入调用
    ↓ ClassWriter 生成新 .class
修改后的字节码(.class)
    ↓ R8/D8
.dex 文件（包含插桩代码）

运行期：
方法被调用
    → MethodTimer.onMethodEnter() 记录开始时间
    → 原始方法逻辑执行
    → MethodTimer.onMethodExit("方法名") 计算耗时
    → 超过阈值 → 触发回调 → 上报APM
```
## 系统调用	
### 主线程SP读写
1.问题：  
```
SP 的实现原理：
├── 数据存储：XML 文件（/data/data/包名/shared_prefs/xxx.xml）
├── 读取：首次 getSharedPreferences → 解析整个 XML 文件到内存
├── 写入：commit() → 同步写磁盘（主线程阻塞！）
│         apply()  → 异步写磁盘（但有隐患）
└── 内存：整个文件常驻内存（即使只用一个key）

具体问题：
┌────────────────────────────────────────────────────┐
│ 问题一：首次加载阻塞主线程                           │
│ getSharedPreferences() 在主线程调用                 │
│ → 触发文件 IO 解析 XML                              │
│ → 文件越大越慢（可能 10ms~100ms）                   │
├────────────────────────────────────────────────────┤
│ 问题二：commit() 同步写磁盘                          │
│ editor.commit() 直接在调用线程写磁盘                 │
│ → 主线程调用 = 主线程阻塞                            │
├────────────────────────────────────────────────────┤
│ 问题三：apply() 的隐患                               │
│ apply() 看似异步，但：                               │
│ Activity.onStop/onPause 时系统会等待                 │
│ 所有 apply() 的写入完成                              │
│ → 可能导致 onStop 卡顿甚至 ANR                       │
├────────────────────────────────────────────────────┤
│ 问题四：多进程不安全                                 │
│ MODE_MULTI_PROCESS 已废弃                           │
│ 多进程读写同一个 SP → 数据损坏                       │
└────────────────────────────────────────────────────┘
```

2.MMKV：  
```
MMKV（腾讯开源）原理：
├── 存储格式：Protocol Buffers（二进制，比XML快）
├── 读写机制：mmap 内存映射文件
│   └── 写入 = 写内存（OS异步刷盘，无需等待）
├── 增量更新：只追加变化的key，不重写整个文件
└── 多进程：文件锁保证安全

传统文件 IO 流程：
                    用户空间          内核空间
write(data) →  [用户缓冲区]  →  [内核缓冲区]  →  [磁盘]
                    ↑                  ↑
              数据在这里           数据复制到这里
              
问题：
① 用户空间 → 内核空间：一次数据拷贝
② 内核空间 → 磁盘：一次磁盘 IO
③ 必须等待 ② 完成，write() 才返回（同步）

─────────────────────────────────────────────────

mmap 流程：
                    用户空间          内核空间
mmap() →       [映射内存区域]  ←→  [内核页缓存]  →  [磁盘]
                    ↑                  ↑
              直接操作这块内存    OS自动同步到磁盘
              
优势：
① 写入 = 写内存（纳秒级，无需等待磁盘）
② OS 负责将页缓存异步刷到磁盘（不阻塞App）
③ 进程崩溃？OS 仍然会将页缓存刷盘，数据不丢失
④ 读取 = 读内存（如果页在内存中，无磁盘 IO）

// mmap 在 Android/Linux 的系统调用
// MMKV 的 C++ 层核心代码（简化）：

// 1. 打开文件
int fd = open(path, O_RDWR | O_CREAT, S_IRWXU);

// 2. 设置文件大小（mmap 需要预先分配空间）
ftruncate(fd, fileSize); // 初始 4KB，不够时扩容

// 3. 内存映射
void* ptr = mmap(
    nullptr,          // 让OS选择映射地址
    fileSize,         // 映射大小
    PROT_READ | PROT_WRITE, // 可读可写
    MAP_SHARED,       // 共享映射（写入对其他进程可见）
    fd,               // 文件描述符
    0                 // 文件偏移量
);

// 4. 写入数据（就是写内存！）
memcpy(ptr + offset, data, dataSize);
// 这一行完成后，数据就"写入"了
// OS 会在合适时机（通常几秒内）将内存页刷到磁盘
// 不需要等待！

// 5. 主动刷盘（可选，需要强一致性时调用）
msync(ptr, fileSize, MS_ASYNC); // 异步刷盘
msync(ptr, fileSize, MS_SYNC);  // 同步刷盘（会阻塞）

// Protobuf 编码原理

XML（SP的格式）：
<map>
    <string name="user_id">12345</string>
    <boolean name="is_login" value="true" />
    <int name="age" value="25" />
</map>

存储大小：约 100 字节
解析：字符串解析，需要遍历每个字符

─────────────────────────────────────────────────

Protobuf（MMKV的格式）：
每个 key-value 编码为：
[key长度(varint)][key内容][value类型+长度(varint)][value内容]

"user_id" + "12345" 的编码：
07 75 73 65 72 5F 69 64  ← "user_id"（7字节）
05 31 32 33 34 35        ← "12345"（5字节）
总共：12字节（vs XML的 35字节）

优势：
├── 体积更小（约 XML 的 1/3~1/5）
├── 解析更快（直接读二进制，无需字符串解析）
└── 增量追加（只追加新的key-value，不重写整个文件）

// 增量写入

SP 写入：改1个key → 重写整个文件
┌─────────────────────────────────┐
│ user_id=12345                   │
│ is_login=true                   │  ← 改了这个
│ age=25                          │
│ theme=dark                      │
└─────────────────────────────────┘
整个文件重新写入！

─────────────────────────────────────────────────

MMKV 写入：改1个key → 追加到文件末尾
┌─────────────────────────────────┐
│ [user_id][12345]                │ ← 旧数据保留
│ [is_login][true]                │ ← 旧数据保留
│ [age][25]                       │ ← 旧数据保留
│ [theme][dark]                   │ ← 旧数据保留
│ [is_login][false]               │ ← 新数据追加到末尾
└─────────────────────────────────┘

读取时：从后往前扫描，取最新的值
写入时：直接追加，O(1) 复杂度

文件满了怎么办？
→ 触发 trim：重新整理文件，去掉旧值，只保留最新值
→ trim 操作耗时，但频率很低（文件满才触发）

// 多进程安全

多进程场景：
进程A 和 进程B 同时写入同一个 MMKV 文件

解决方案：文件锁（flock）

进程A 写入：
① flock(fd, LOCK_EX) // 获取排他锁
② 追加数据到 mmap 内存
③ flock(fd, LOCK_UN) // 释放锁

进程B 同时写入：
① flock(fd, LOCK_EX) // 等待进程A释放锁
② 获得锁后，先检查文件是否被修改（CRC校验）
③ 重新加载变化的数据
④ 追加自己的数据
⑤ 释放锁

读取：使用共享锁（LOCK_SH），允许并发读

// 多进程 MMKV 初始化
val mmkv = MMKV.mmkvWithID(
    "shared_prefs",
    MMKV.MULTI_PROCESS_MODE // 开启多进程模式
)

性能对比：
┌──────────┬───────────┬───────────┬──────────────┐
│          │ 写入耗时  │ 读取耗时  │ 主线程安全    │
├──────────┼───────────┼───────────┼──────────────┤
│ SP       │ ~1ms/次   │ ~0.1ms/次 │ commit不安全  │
│ MMKV     │ ~0.01ms/次│ ~0.01ms/次│ 完全安全      │
└──────────┴───────────┴───────────┴──────────────┘


// ================================
// MMKV 接入
// ================================

// build.gradle
// implementation 'com.tencent:mmkv:1.3.1'

// Application 初始化
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        MMKV.initialize(this) // 初始化，指定存储目录
    }
}

// ================================
// 封装 MMKV（兼容 SP 接口）
// ================================
class MMKVPreferences(
    private val mmkv: MMKV = MMKV.defaultMMKV()
) : SharedPreferences {

    // 读操作（直接读内存，极快）
    override fun getString(key: String, defValue: String?): String? =
        mmkv.decodeString(key, defValue)

    override fun getInt(key: String, defValue: Int): Int =
        mmkv.decodeInt(key, defValue)

    override fun getLong(key: String, defValue: Long): Long =
        mmkv.decodeLong(key, defValue)

    override fun getBoolean(key: String, defValue: Boolean): Boolean =
        mmkv.decodeBool(key, defValue)

    override fun getFloat(key: String, defValue: Float): Float =
        mmkv.decodeFloat(key, defValue)

    override fun getStringSet(key: String, defValues: Set<String>?): Set<String>? =
        mmkv.decodeStringSet(key, defValues)

    override fun contains(key: String): Boolean = mmkv.containsKey(key)

    override fun getAll(): Map<String, *> {
        val result = mutableMapOf<String, Any?>()
        mmkv.allKeys()?.forEach { key ->
            result[key] = mmkv.decodeString(key)
        }
        return result
    }

    override fun edit(): SharedPreferences.Editor = MMKVEditor(mmkv)

    override fun registerOnSharedPreferenceChangeListener(
        listener: SharedPreferences.OnSharedPreferenceChangeListener
    ) { /* MMKV 不支持，可用其他方式实现 */ }

    override fun unregisterOnSharedPreferenceChangeListener(
        listener: SharedPreferences.OnSharedPreferenceChangeListener
    ) { }
}

class MMKVEditor(private val mmkv: MMKV) : SharedPreferences.Editor {

    private val pendingChanges = mutableMapOf<String, Any?>()
    private val pendingRemovals = mutableSetOf<String>()
    private var clearAll = false

    override fun putString(key: String, value: String?) = apply {
        pendingChanges[key] = value
    }
    override fun putInt(key: String, value: Int) = apply {
        pendingChanges[key] = value
    }
    override fun putLong(key: String, value: Long) = apply {
        pendingChanges[key] = value
    }
    override fun putBoolean(key: String, value: Boolean) = apply {
        pendingChanges[key] = value
    }
    override fun putFloat(key: String, value: Float) = apply {
        pendingChanges[key] = value
    }
    override fun putStringSet(key: String, values: Set<String>?) = apply {
        pendingChanges[key] = values
    }
    override fun remove(key: String) = apply {
        pendingRemovals.add(key)
    }
    override fun clear() = apply { clearAll = true }

    // commit/apply 都是写内存（mmap），极快，主线程安全
    override fun commit(): Boolean {
        applyChanges()
        return true
    }

    override fun apply() {
        applyChanges() // MMKV 的写入本身就是异步安全的
    }

    private fun applyChanges() {
        if (clearAll) mmkv.clearAll()
        pendingRemovals.forEach { mmkv.removeValueForKey(it) }
        pendingChanges.forEach { (key, value) ->
            when (value) {
                is String -> mmkv.encode(key, value)
                is Int -> mmkv.encode(key, value)
                is Long -> mmkv.encode(key, value)
                is Boolean -> mmkv.encode(key, value)
                is Float -> mmkv.encode(key, value)
                is Set<*> -> @Suppress("UNCHECKED_CAST")
                    mmkv.encode(key, value as Set<String>)
                null -> mmkv.removeValueForKey(key)
            }
        }
    }
}

// ================================
// SP 迁移到 MMKV（一次性迁移）
// ================================
object SPMigration {

    fun migrate(context: Context, spName: String) {
        val sp = context.getSharedPreferences(spName, Context.MODE_PRIVATE)
        val mmkv = MMKV.mmkvWithID(spName)

        // 检查是否已迁移
        if (mmkv.containsKey("__migrated__")) return

        // 迁移所有数据
        sp.all.forEach { (key, value) ->
            when (value) {
                is String -> mmkv.encode(key, value)
                is Int -> mmkv.encode(key, value)
                is Long -> mmkv.encode(key, value)
                is Boolean -> mmkv.encode(key, value)
                is Float -> mmkv.encode(key, value)
                is Set<*> -> @Suppress("UNCHECKED_CAST")
                    mmkv.encode(key, value as Set<String>)
            }
        }

        // 标记已迁移
        mmkv.encode("__migrated__", true)

        // 清除旧 SP（可选）
        sp.edit().clear().apply()
    }
}
```

3.DataStore:  
```
├── 完全异步（基于协程和 Flow）
├── 事务性写入（原子操作）
├── 错误处理（IO 异常通过 Flow 传递）
└── 强制异步（没有同步 API，不可能主线程调用）

DataStore 底层使用 Protocol Buffers 存储
（Preferences DataStore 也是，只是封装了类型安全的 API）

文件位置：
/data/data/包名/files/datastore/文件名.preferences_pb

存储格式：Protobuf 二进制
（比 SP 的 XML 更紧凑，解析更快）

但 DataStore 的优化重点不在存储格式
而在于：读写机制的正确性

// DataStore 内部核心：SingleProcessDataStore

// ================================
// 写入机制（事务性）
// ================================

// dataStore.edit { } 的内部流程：

// 1. 获取当前数据（从内存缓存或文件）
// 2. 在协程中执行 transform 块（你的修改逻辑）
// 3. 将修改后的数据序列化为 Protobuf
// 4. 写入临时文件（.tmp 后缀）
// 5. 原子重命名（rename tmp → 正式文件）
//    rename 是原子操作，要么成功要么失败
//    不会出现"写了一半"的情况
// 6. 更新内存缓存
// 7. 通知所有 Flow 收集者

// 关键：步骤 3~5 在 IO 线程执行
// 步骤 2（transform）在调用协程的上下文执行

// ================================
// 读取机制（Flow + 内存缓存）
// ================================

// dataStore.data 是一个 Flow<Preferences>
// 内部实现：

class SingleProcessDataStore {

    // 内存缓存（StateFlow）
    private val _data = MutableStateFlow<Preferences?>(null)

    // 对外暴露的 Flow
    val data: Flow<Preferences> = flow {
        // 首次收集：从文件读取
        val initialData = readFromFile()
        _data.value = initialData
        emit(initialData)

        // 后续：监听内存缓存变化
        _data.filterNotNull().collect { emit(it) }
    }.flowOn(Dispatchers.IO) // IO 线程读取文件

    // 写入
    suspend fun edit(transform: suspend (MutablePreferences) -> Unit) {
        // 在专用的单线程调度器上执行（串行化写入）
        withContext(singleThreadContext) {
            val current = _data.value ?: readFromFile()
            val mutable = current.toMutablePreferences()

            transform(mutable) // 执行用户的修改

            val updated = mutable.toPreferences()

            // 原子写入文件
            writeToFileAtomically(updated)

            // 更新内存缓存，触发 Flow 通知
            _data.value = updated
        }
    }

    private fun writeToFileAtomically(data: Preferences) {
        val tmpFile = File(file.parent, "${file.name}.tmp")
        // 写入临时文件
        tmpFile.outputStream().use { output ->
            PreferencesProto.serialize(data, output)
        }
        // 原子重命名
        if (!tmpFile.renameTo(file)) {
            throw IOException("写入失败")
        }
    }
}

// 不会 ANR

SP apply() 的 ANR 原因：

SP 内部有一个 QueuedWork 机制：
apply() → 将写入任务加入 QueuedWork 队列
Activity.onStop() → 系统调用 QueuedWork.waitToFinish()
                  → 主线程等待所有 apply() 写入完成
                  → 如果写入慢 → 主线程阻塞 → ANR

─────────────────────────────────────────────────

DataStore 为什么没有这个问题：

① 没有 apply() 这个 API
   只有 suspend fun edit()
   必须在协程中调用
   不可能在主线程同步调用

② 写入完全在 IO 协程中
   不依赖 QueuedWork
   不会在 onStop 时阻塞主线程

③ 即使 App 进程被杀
   写入的是临时文件 + 原子重命名
   要么写入成功，要么保留旧文件
   不会出现数据损坏

// DataStore 两种类型：
// ├── Preferences DataStore：类似 SP，key-value
// └── Proto DataStore：强类型，基于 Protobuf

// ================================
// Preferences DataStore
// ================================

// 创建 DataStore（全局单例）
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(
    name = "user_settings"
)

// 定义 key
object PreferencesKeys {
    val USER_ID = longPreferencesKey("user_id")
    val USER_NAME = stringPreferencesKey("user_name")
    val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
    val THEME_MODE = intPreferencesKey("theme_mode")
}

// Repository 封装
class UserPreferencesRepository(private val context: Context) {

    // 读取（Flow，响应式）
    val userIdFlow: Flow<Long> = context.dataStore.data
        .catch { exception ->
            // 读取异常处理
            if (exception is IOException) {
                emit(emptyPreferences())
            } else throw exception
        }
        .map { preferences ->
            preferences[PreferencesKeys.USER_ID] ?: 0L
        }

    val isLoggedIn: Flow<Boolean> = context.dataStore.data
        .map { it[PreferencesKeys.IS_LOGGED_IN] ?: false }

    // 写入（suspend，在协程中调用）
    suspend fun saveUserId(userId: Long) {
        context.dataStore.edit { preferences ->
            preferences[PreferencesKeys.USER_ID] = userId
        }
    }

    suspend fun saveLoginState(isLoggedIn: Boolean) {
        context.dataStore.edit { preferences ->
            preferences[PreferencesKeys.IS_LOGGED_IN] = isLoggedIn
        }
    }

    // 批量写入（原子操作）
    suspend fun saveUserInfo(userId: Long, userName: String) {
        context.dataStore.edit { preferences ->
            preferences[PreferencesKeys.USER_ID] = userId
            preferences[PreferencesKeys.USER_NAME] = userName
            preferences[PreferencesKeys.IS_LOGGED_IN] = true
            // 以上三个操作是原子的
        }
    }

    // 清除所有数据
    suspend fun clearAll() {
        context.dataStore.edit { it.clear() }
    }
}

// ViewModel 中使用
class SettingsViewModel(
    private val repo: UserPreferencesRepository
) : ViewModel() {

    // 直接暴露 Flow 给 UI
    val isLoggedIn = repo.isLoggedIn
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = false
        )

    fun logout() {
        viewModelScope.launch {
            repo.clearAll() // 自动在 IO 线程执行
        }
    }
}
```
### 文件 IO 异步化
1.问题：  
```
文件 IO 耗时来源：
├── 磁盘寻道时间（HDD：10ms，SSD：0.1ms，eMMC：0.5ms）
├── 文件系统操作（open/close/stat）
├── 数据传输（读写速度）
└── 文件锁竞争

典型耗时（低端机 eMMC）：
├── 读取 1KB 文件：0.5ms~5ms
├── 读取 1MB 文件：5ms~50ms
├── 写入 1KB 文件：1ms~10ms
└── 创建/删除文件：1ms~20ms

主线程执行以上操作 → 直接导致卡顿
```

2.异步：  
```
// ================================
// 封装异步文件操作工具类
// ================================
object AsyncFileIO {

    // 异步读取文件（返回 Flow，支持进度）
    fun readFileAsFlow(file: File): Flow<ByteArray> = flow {
        emit(file.readBytes())
    }.flowOn(Dispatchers.IO)

    // 异步读取文本
    suspend fun readText(file: File): String =
        withContext(Dispatchers.IO) {
            file.readText(Charsets.UTF_8)
        }

    // 异步写入文本
    suspend fun writeText(file: File, content: String) =
        withContext(Dispatchers.IO) {
            file.writeText(content, Charsets.UTF_8)
        }

    // 异步追加写入
    suspend fun appendText(file: File, content: String) =
        withContext(Dispatchers.IO) {
            file.appendText(content, Charsets.UTF_8)
        }

    // 带缓冲的大文件读取
    suspend fun readLargeFile(
        file: File,
        bufferSize: Int = 8192,
        onProgress: ((Float) -> Unit)? = null
    ): ByteArray = withContext(Dispatchers.IO) {
        val totalSize = file.length()
        val outputStream = ByteArrayOutputStream(totalSize.toInt())
        var bytesRead = 0L

        file.inputStream().buffered(bufferSize).use { input ->
            val buffer = ByteArray(bufferSize)
            var count: Int
            while (input.read(buffer).also { count = it } != -1) {
                outputStream.write(buffer, 0, count)
                bytesRead += count
                onProgress?.invoke(bytesRead.toFloat() / totalSize)
            }
        }
        outputStream.toByteArray()
    }

    // 安全写入（先写临时文件，再重命名，防止写入中断导致文件损坏）
    suspend fun safeWriteText(file: File, content: String) =
        withContext(Dispatchers.IO) {
            val tempFile = File(file.parent, "${file.name}.tmp")
            try {
                tempFile.writeText(content, Charsets.UTF_8)
                // 原子重命名（同目录下）
                if (!tempFile.renameTo(file)) {
                    // renameTo 失败，手动复制
                    file.writeText(content)
                    tempFile.delete()
                }
            } catch (e: Exception) {
                tempFile.delete()
                throw e
            }
        }
}

// ================================
// 日志文件异步写入（高频写入场景）
// ================================
class AsyncLogger(private val logFile: File) {

    // 使用 Channel 作为缓冲队列
    private val logChannel = Channel<String>(capacity = Channel.UNLIMITED)
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    init {
        // 启动消费者协程，批量写入
        scope.launch {
            val buffer = StringBuilder()
            var lastFlushTime = System.currentTimeMillis()

            for (log in logChannel) {
                buffer.appendLine(log)

                val now = System.currentTimeMillis()
                val shouldFlush = buffer.length > 4096 || // 缓冲区满
                        now - lastFlushTime > 1000        // 超过1秒

                if (shouldFlush) {
                    flushBuffer(buffer.toString())
                    buffer.clear()
                    lastFlushTime = now
                }
            }
            // Channel 关闭时，写入剩余内容
            if (buffer.isNotEmpty()) {
                flushBuffer(buffer.toString())
            }
        }
    }

    // 主线程调用，非阻塞
    fun log(message: String) {
        val timestamp = SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault())
            .format(Date())
        logChannel.trySend("[$timestamp] $message")
    }

    private suspend fun flushBuffer(content: String) {
        withContext(Dispatchers.IO) {
            logFile.appendText(content)
        }
    }

    fun close() {
        logChannel.close()
        scope.cancel()
    }
}
```
## 线程调度	
### 单线程协程调度
```
class SingleThreadDataManager {
    // 所有操作在同一个协程上下文中串行执行
    // 天然线程安全，无需锁
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)
    private var data: String = ""

    suspend fun updateData(newData: String) = withContext(dispatcher) {
        data = newData // 单线程执行，无竞争
    }

    suspend fun getData(): String = withContext(dispatcher) {
        data
    }
}
```
### 超时
```
class TimeoutLockExample {
    private val lock = ReentrantLock()

    fun tryExecute(timeoutMs: Long, action: () -> Unit): Boolean {
        // 尝试获取锁，超时则放弃
        if (lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
            try {
                action()
                return true
            } finally {
                lock.unlock()
            }
        }
        // 获取锁超时，上报卡顿
        reportLockTimeout(timeoutMs)
        return false
    }
}
```
## 消息队列	
### Handler 消息积压
```
// ================================
// 问题：消息队列积压导致卡顿
// ================================

// 场景：高频事件（传感器/触摸）产生大量消息
// 主线程处理不过来 → 消息积压 → 延迟响应

class SensorDataProcessor {

    private val mainHandler = Handler(Looper.getMainLooper())

    // ❌ 问题：每次传感器数据都 post 到主线程
    // 传感器频率 100Hz = 每10ms一条消息
    // 主线程处理一条消息需要 16ms
    // 消息积压越来越多！
    fun onSensorDataBad(data: SensorData) {
        mainHandler.post {
            updateUI(data) // 主线程处理
        }
    }

    // ✅ 优化一：合并消息（只处理最新的）
    private var pendingData: SensorData? = null
    private val processRunnable = Runnable {
        pendingData?.let { updateUI(it) }
        pendingData = null
    }

    fun onSensorDataGood(data: SensorData) {
        pendingData = data
        // 移除旧的消息，只保留最新的
        mainHandler.removeCallbacks(processRunnable)
        mainHandler.post(processRunnable)
    }

    // ✅ 优化二：降低处理频率（采样）
    private var lastProcessTime = 0L
    private val minInterval = 16L // 最多60fps

    fun onSensorDataSampled(data: SensorData) {
        val now = SystemClock.elapsedRealtime()
        if (now - lastProcessTime < minInterval) return // 丢弃过于频繁的数据

        lastProcessTime = now
        mainHandler.post { updateUI(data) }
    }
}
```
### Handler 消息优化
```
// ================================
// 优化一：使用 Message 对象池
// ================================

// ❌ 直接创建 Message 对象
handler.post(Runnable { doSomething() })
// 每次都创建新的 Message 和 Runnable 对象

// ✅ 使用 Message.obtain() 复用对象池中的 Message
val msg = Message.obtain(handler) {
    doSomething()
}
handler.sendMessage(msg)
// Message 处理完后自动归还到对象池

// ================================
// 优化二：避免内存泄漏（弱引用）
// ================================

// ❌ 匿名内部类 Handler 持有 Activity 引用
class BadActivity : AppCompatActivity() {
    // 内部类持有外部类引用 → 内存泄漏！
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            // 这里隐式持有 BadActivity.this
            updateUI()
        }
    }
}

// ✅ 静态内部类 + 弱引用
class GoodActivity : AppCompatActivity() {

    private val handler = SafeHandler(this)

    class SafeHandler(activity: GoodActivity) :
        Handler(Looper.getMainLooper()) {

        // 弱引用，不阻止 Activity 被回收
        private val weakActivity = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            val activity = weakActivity.get() ?: return
            // Activity 已被回收，不执行
            activity.updateUI()
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null) // 清除所有消息
    }
}
```
### IdleHandler 合理使用
```
// IdleHandler：主线程空闲时执行任务
// 不影响主线程正常任务，利用空闲时间做低优先级工作

class IdleTaskScheduler {

    private val pendingTasks = ArrayDeque<() -> Unit>()

    fun scheduleWhenIdle(task: () -> Unit) {
        pendingTasks.addLast(task)

        Looper.myQueue().addIdleHandler(object : MessageQueue.IdleHandler {
            override fun queueIdle(): Boolean {
                // 主线程空闲时执行一个任务
                val nextTask = pendingTasks.removeFirstOrNull()
                nextTask?.invoke()

                // 返回 true = 还有任务，继续注册
                // 返回 false = 任务完成，取消注册
                return pendingTasks.isNotEmpty()
            }
        })
    }
}

// 使用场景：
// ① 预加载下一页数据
// ② 预创建 ViewHolder
// ③ 清理缓存
// ④ 上报日志

val scheduler = IdleTaskScheduler()

// 首帧渲染完成后，利用空闲时间预加载
scheduler.scheduleWhenIdle { preloadNextPage() }
scheduler.scheduleWhenIdle { warmupImageCache() }
scheduler.scheduleWhenIdle { reportAnalytics() }
```
# 渲染优化	
```
App 进程                    系统进程
┌─────────────────┐        ┌──────────────────┐
│   主线程         │        │  SurfaceFlinger   │
│   measure       │        │  合成所有Layer     │
│   layout        │        │  送显到屏幕        │
│   draw          │        └──────────────────┘
│   构建DrawOp列表 │               ↑
├─────────────────┤        ┌──────────────────┐
│   RenderThread  │──GPU──►│   Hardware       │
│   执行DrawOp    │        │   Composer       │
│   提交GPU       │        └──────────────────┘
└─────────────────┘

关键认知：
├── 主线程：负责 measure/layout/draw（构建渲染指令）
├── RenderThread：负责执行渲染指令，与GPU通信
├── GPU：实际执行渲染
└── SurfaceFlinger：合成多个窗口，最终送显
```
## 帧率优化
### 首帧优化
首帧定义：  
```
首帧 ≠ Activity.onCreate() 完成
首帧 ≠ setContentView() 完成
首帧 = 用户真正看到有意义内容的那一帧上屏

几个关键时间点：

时间轴：
0ms    Activity.onCreate()
  ↓    setContentView()
  ↓    onStart() / onResume()
  ↓    ViewRootImpl.performTraversals()  ← measure/layout/draw
  ↓    RenderThread 提交 GPU
  ↓    SurfaceFlinger 合成
  ↓    屏幕显示 ← 这才是首帧！

指标定义：
├── TTID（Time To Initial Display）
│     系统定义，Activity 首次渲染完成
│     对应 Logcat: "Displayed com.xxx/.MainActivity: +1s234ms"
│
└── TTFD（Time To Full Display）
      开发者定义，真正有意义的内容展示完成
      对应 reportFullyDrawn() 调用时机
```

TTID & TTFD：  
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // ↑ TTID 从这里开始计算首帧

        // 此时界面可能只有骨架屏/占位图
        // 用户看到的不是真正的内容

        viewModel.data.observe(this) { data ->
            renderContent(data)
            // ↑ 真正的内容渲染完成

            reportFullyDrawn()
            // ↑ TTFD：告诉系统"现在才是真正展示完成"
        }
    }
}

// TTID 和 TTFD 的差距 = 数据加载时间
// 优化目标：
// ① 缩短 TTID（让界面尽快出现，哪怕是骨架屏）
// ② 缩短 TTFD（让真实内容尽快展示）
// ③ 缩短 TTFD - TTID 的差值（骨架屏展示时间）
```

完整流程：  
```
setContentView(R.layout.activity_main)
         │
         ▼
① XML 解析 + View 创建
   XmlPullParser 解析 XML
   反射创建每个 View 对象
   设置 LayoutParams 和属性
         │
         ▼
② ViewTree 构建完成
   DecorView
   └── LinearLayout（系统）
         └── FrameLayout（content）
               └── 你的根布局
                     └── 子View树
         │
         ▼
③ onResume() 执行完毕
   此时 ViewRootImpl 才开始工作
   （onResume 之前不会触发绘制！）
         │
         ▼
④ Choreographer 等待 VSync 信号
   VSync 到来（16.6ms 一次，60fps）
         │
         ▼
⑤ ViewRootImpl.performTraversals()
   ├── performMeasure()   → View.measure()
   ├── performLayout()    → View.layout()
   └── performDraw()      → View.draw()
         │
         ▼
⑥ RenderThread（硬件加速）
   构建 DisplayList
   提交给 GPU 渲染
         │
         ▼
⑦ SurfaceFlinger
   合成多个 Layer
   送显到屏幕
         │
         ▼
⑧ 用户看到首帧 ✅
```
#### 耗时
1.布局 inflate 耗时：  
```
├── IO：从磁盘/内存读取 XML 文件
├── XML 解析：XmlPullParser 逐字符解析
├── 反射：Class.forName() + newInstance() 创建 View
└── 属性设置：解析并设置每个 XML 属性

典型耗时（低端机）：
├── 简单布局（10个View）：~10ms
├── 中等布局（30个View）：~30ms
├── 复杂布局（100个View）：~100ms+
└── 嵌套RecyclerView首屏：~200ms+
```

2.measure/layout 耗时：  
```
measure:
├── 布局层级深：每层都要 measure，N层 = N次递归
├── RelativeLayout：子View measure 两次！
├── ConstraintLayout：复杂约束计算
└── wrap_content + 多层嵌套：最坏情况 O(2^n)

layout：
├── 复杂位置计算
└── 频繁的 requestLayout（触发重新 measure/layout）
```

3.draw 耗时：  
```
├── 过度绘制：同一像素被绘制多次
├── 复杂自定义 View 的 onDraw
├── 大量 Canvas 操作
└── 软件渲染（未开启硬件加速）
```
#### 优化
1.层级优化：  
```
<!-- ❌ 错误：深层嵌套 -->
<LinearLayout>
    <LinearLayout>
        <RelativeLayout>
            <LinearLayout>
                <TextView/>
                <ImageView/>
            </LinearLayout>
        </RelativeLayout>
    </LinearLayout>
</LinearLayout>

<!-- ✅ 正确：ConstraintLayout 一层搞定 -->
<ConstraintLayout>
    <TextView
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"/>
    <ImageView
        app:layout_constraintTop_toBottomOf="@id/textView"
        app:layout_constraintStart_toStartOf="parent"/>
</ConstraintLayout>

// 用 Layout Inspector 检测层级
// Android Studio → View → Tool Windows → Layout Inspector

// 用 Hierarchy Viewer 检测 measure/layout/draw 耗时
// 红色节点 = 该层级耗时最长，重点优化
```

2.异步 inflate：  
```
// 方案一：AsyncLayoutInflater
// 适用场景：非首屏页面预加载（上文已讲过局限性）

// 方案二：X2C（编译期 XML → Java Code）
// 彻底消除 XML 解析 + 反射

// X2C 生成的代码示例：
// 原 XML：
// <TextView
//     android:id="@+id/title"
//     android:layout_width="match_parent"
//     android:layout_height="wrap_content"
//     android:textSize="16sp"
//     android:textColor="#333333"/>

// X2C 生成：
class XC_ActivityMain : IViewCreator {
    override fun createView(context: Context, tag: String?): View? {
        return null
    }

    fun createView(context: Context): View {
        val textView = TextView(context)
        textView.id = R.id.title

        val lp = LinearLayout.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.WRAP_CONTENT
        )
        textView.layoutParams = lp
        textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, 16f)
        textView.setTextColor(0xFF333333.toInt())

        return textView
    }
}

// 接入方式：
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 替换 setContentView
        X2C.setContentView(this, R.layout.activity_main)
    }
}
```

3.ViewStub 延迟加载：  
```
// Xml
<!-- activity_main.xml -->
<LinearLayout>

    <!-- 首屏必要内容：立即加载 -->
    <include layout="@layout/header"/>
    <include layout="@layout/main_content"/>

    <!-- 非首屏内容：用 ViewStub 占位 -->
    <ViewStub
        android:id="@+id/stub_side_panel"
        android:layout="@layout/side_panel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

    <ViewStub
        android:id="@+id/stub_bottom_sheet"
        android:layout="@layout/bottom_sheet"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</LinearLayout>

// Java
class MainActivity : AppCompatActivity() {

    private var sidePanelInflated = false

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 此时 side_panel 和 bottom_sheet 都没有 inflate
        // 首帧渲染更快！

        setupMainContent() // 只处理首屏内容
    }

    // 用户点击侧边栏按钮时才 inflate
    fun onSidePanelClick() {
        if (!sidePanelInflated) {
            val sidePanel = findViewById<ViewStub>(R.id.stub_side_panel)
                .inflate() // 此时才真正 inflate
            sidePanelInflated = true
            setupSidePanel(sidePanel)
        }
        // 显示侧边栏
    }
}
```

4.骨架屏：  
```
// 骨架屏的本质：
// 用简单的占位布局替代复杂的真实布局
// 让用户感知到"有东西了"，降低等待焦虑

class HomeActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)

        // 立即显示骨架屏（本地资源，无需网络）
        showSkeleton()

        // 异步加载真实数据
        viewModel.loadData()
        viewModel.uiState.observe(this) { state ->
            when (state) {
                is UiState.Loading -> showSkeleton()
                is UiState.Success -> {
                    hideSkeleton()
                    renderContent(state.data)
                    reportFullyDrawn() // 真实内容展示完成
                }
                is UiState.Error -> showError(state.error)
            }
        }
    }

    private fun showSkeleton() {
        skeletonView.visibility = View.VISIBLE
        contentView.visibility = View.GONE
        // 骨架屏动画（shimmer效果）
        shimmerLayout.startShimmer()
    }

    private fun hideSkeleton() {
        shimmerLayout.stopShimmer()
        skeletonView.visibility = View.GONE
        contentView.visibility = View.VISIBLE
    }
}
```

5.onResume 前置优化：  
```
// onResume 执行完毕后，Choreographer 才开始等 VSync
// onResume 之前的任何耗时都会延迟首帧

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ❌ 错误：onCreate 中做耗时操作
        val config = loadConfigFromDisk()      // IO，50ms
        val user = queryUserFromDB()           // DB，80ms
        setupComplexAnimation()               // 复杂计算，30ms
        // 以上160ms全部在首帧之前！

        setContentView(R.layout.activity_main)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // ✅ 正确：先展示骨架屏，再异步加载数据
        showSkeleton()

        lifecycleScope.launch {
            // 异步执行，不阻塞首帧
            val config = withContext(Dispatchers.IO) {
                loadConfigFromDisk()
            }
            val user = withContext(Dispatchers.IO) {
                queryUserFromDB()
            }
            // 数据准备好后更新UI
            hideSkeleton()
            renderContent(config, user)
            reportFullyDrawn()
        }
    }
}
```

6.Compose首帧优化：  
```
// Compose 没有 XML inflate，首帧天然更快
// 但 Compose 有自己的首帧问题：首次 Composition 耗时

@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()

    // ❌ 错误：首帧直接渲染复杂内容
    when (uiState) {
        is UiState.Loading -> ComplexLoadingScreen() // 复杂加载界面
        is UiState.Success -> ComplexHomeContent(uiState.data)
        is UiState.Error -> ErrorScreen()
    }

    // ✅ 正确：首帧用简单骨架屏
    when (uiState) {
        is UiState.Loading -> {
            // 简单的骨架屏，Composition 快
            SkeletonScreen()
        }
        is UiState.Success -> {
            // 真实内容，数据准备好后再渲染
            HomeContent(data = uiState.data)
            LaunchedEffect(Unit) {
                reportFullyDrawn() // 通知系统真实内容已展示
            }
        }
        is UiState.Error -> ErrorScreen()
    }
}

// Compose 重组优化：避免首帧不必要的重组
@Composable
fun HomeContent(data: HomeData) {
    // 用 remember 缓存计算结果，避免重组时重复计算
    val processedData = remember(data) {
        processData(data) // 只在 data 变化时重新计算
    }

    LazyColumn {
        items(
            items = processedData.items,
            key = { it.id } // 提供 key，优化重组
        ) { item ->
            ItemCard(item = item)
        }
    }
}
```

7.预渲染：  
```
// 预渲染：在用户进入页面之前，提前创建并渲染 View
// 适用场景：高频访问的页面（如首页、详情页）

object PreRenderer {

    private var preRenderedView: View? = null
    private val mainHandler = Handler(Looper.getMainLooper())

    // 在 Splash 页面展示期间，提前渲染首页
    fun preRenderHome(context: Context) {
        mainHandler.post {
            // 必须在主线程创建 View
            val view = LayoutInflater.from(context)
                .inflate(R.layout.activity_home, null)

            // 提前 measure（使用屏幕尺寸）
            val width = context.resources.displayMetrics.widthPixels
            val height = context.resources.displayMetrics.heightPixels
            view.measure(
                View.MeasureSpec.makeMeasureSpec(width, View.MeasureSpec.EXACTLY),
                View.MeasureSpec.makeMeasureSpec(height, View.MeasureSpec.EXACTLY)
            )
            view.layout(0, 0, width, height)

            preRenderedView = view
        }
    }

    // HomeActivity 直接使用预渲染的 View
    fun getPreRenderedHome(): View? {
        return preRenderedView?.also {
            preRenderedView = null // 用完清除
        }
    }
}

class SplashActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_splash)

        // Splash 展示期间预渲染首页
        PreRenderer.preRenderHome(applicationContext)

        Handler(Looper.getMainLooper()).postDelayed({
            startActivity(Intent(this, HomeActivity::class.java))
        }, 2000)
    }
}

class HomeActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 尝试使用预渲染的 View
        val preRendered = PreRenderer.getPreRenderedHome()
        if (preRendered != null) {
            // 直接使用，跳过 inflate + measure + layout
            setContentView(preRendered)
        } else {
            // 降级：正常流程
            setContentView(R.layout.activity_home)
        }
    }
}
```

首帧渲染监控：  
```
// 精准监控首帧时间
class FirstFrameMonitor {

    companion object {
        // 精准的首帧回调
        fun registerFirstFrameCallback(
            activity: AppCompatActivity,
            onFirstFrame: (costMs: Long) -> Unit
        ) {
            val startTime = SystemClock.elapsedRealtime()

            // OnPreDrawListener（最早的绘制回调）
            activity.window.decorView.viewTreeObserver
                .addOnPreDrawListener(object : ViewTreeObserver.OnPreDrawListener {
                    override fun onPreDraw(): Boolean {
                        activity.window.decorView.viewTreeObserver
                            .removeOnPreDrawListener(this)

                        val cost = SystemClock.elapsedRealtime() - startTime
                        onFirstFrame(cost)
                        return true
                    }
                })
        }
    }
}

// 使用
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        FirstFrameMonitor.registerFirstFrameCallback(this) { costMs ->
            Log.d("FirstFrame", "首帧耗时: ${costMs}ms")
        }

        setContentView(R.layout.activity_main)
    }
}


# 系统自动打印 TTID
adb logcat | grep "Displayed"

# 输出：
# ActivityManager: Displayed com.xxx/.MainActivity: +1s234ms
#                                                    ↑ TTID
```
### Janky Frame 分析
帧率 & VSync:  
```
屏幕刷新率：60Hz = 每16.6ms刷新一次
           90Hz = 每11.1ms刷新一次
           120Hz = 每8.3ms刷新一次

VSync信号：屏幕每次刷新前发出的同步信号
           App 必须在下一个 VSync 到来前准备好一帧

16ms内必须完成：
├── 主线程：measure + layout + draw（构建DrawOp）
├── RenderThread：执行DrawOp + 提交GPU
└── GPU：渲染完成

任何一个环节超时 → 丢帧（Jank）
连续丢帧 → 用户感知到卡顿

严重程度分级：
├── Jank：丢1帧（16ms~32ms）
├── Serious Jank：丢2帧以上（32ms~700ms）
└── Frozen Frame：冻帧（>700ms，接近ANR）
```

FrameMetrics:  
```
// FrameMetrics 提供每帧各阶段的精确耗时
// Android 7.0+

class JankyFrameAnalyzer(private val activity: Activity) {

    data class FrameDetail(
        val totalMs: Long,
        val inputMs: Long,        // 输入事件处理
        val animationMs: Long,    // 动画计算
        val layoutMeasureMs: Long,// measure + layout
        val drawMs: Long,         // draw（构建DisplayList）
        val syncMs: Long,         // 主线程→RenderThread同步
        val commandMs: Long,      // RenderThread执行命令
        val gpuMs: Long,          // GPU渲染
        val swapMs: Long,         // 缓冲区交换
        val isJanky: Boolean
    )

    private val frameDetails = mutableListOf<FrameDetail>()

    private val listener = Window.OnFrameMetricsAvailableListener {
        _, metrics, _ ->

        val total = metrics.getMetric(FrameMetrics.TOTAL_DURATION) / 1_000_000
        val input = metrics.getMetric(FrameMetrics.INPUT_HANDLING_DURATION) / 1_000_000
        val animation = metrics.getMetric(FrameMetrics.ANIMATION_DURATION) / 1_000_000
        val layoutMeasure = metrics.getMetric(FrameMetrics.LAYOUT_MEASURE_DURATION) / 1_000_000
        val draw = metrics.getMetric(FrameMetrics.DRAW_DURATION) / 1_000_000
        val sync = metrics.getMetric(FrameMetrics.SYNC_DURATION) / 1_000_000
        val command = metrics.getMetric(FrameMetrics.COMMAND_ISSUE_DURATION) / 1_000_000
        val gpu = metrics.getMetric(FrameMetrics.GPU_DURATION) / 1_000_000
        val swap = metrics.getMetric(FrameMetrics.SWAP_BUFFERS_DURATION) / 1_000_000
        val isJanky = metrics.getMetric(FrameMetrics.FIRST_DRAW_FRAME) == 0L

        val detail = FrameDetail(
            total, input, animation, layoutMeasure,
            draw, sync, command, gpu, swap, total > 16
        )

        if (detail.isJanky) {
            analyzeJankCause(detail)
            frameDetails.add(detail)
        }
    }

    // 根据各阶段耗时分析卡顿原因
    private fun analyzeJankCause(frame: FrameDetail) {
        val cause = when {
            frame.inputMs > 8 ->
                "输入事件处理过慢（${frame.inputMs}ms）：onTouchEvent耗时"

            frame.layoutMeasureMs > 8 ->
                "measure/layout过慢（${frame.layoutMeasureMs}ms）：布局层级过深或复杂约束"

            frame.drawMs > 8 ->
                "draw过慢（${frame.drawMs}ms）：onDraw耗时，可能有对象创建或复杂绘制"

            frame.syncMs > 4 ->
                "主线程同步RenderThread过慢（${frame.syncMs}ms）：可能有大Bitmap上传GPU"

            frame.gpuMs > 8 ->
                "GPU渲染过慢（${frame.gpuMs}ms）：过度绘制或复杂shader"

            frame.commandMs > 8 ->
                "RenderThread执行命令过慢（${frame.commandMs}ms）：DisplayList过于复杂"

            else ->
                "综合耗时（total=${frame.totalMs}ms）"
        }

        Log.w("JankyFrame", "卡顿原因：$cause")
        reportJankCause(cause, frame)
    }

    fun start() {
        activity.window.addOnFrameMetricsAvailableListener(
            listener, Handler(Looper.getMainLooper())
        )
    }

    // 生成卡顿报告
    fun generateReport(): JankReport {
        val jankyFrames = frameDetails.filter { it.isJanky }
        return JankReport(
            totalFrames = frameDetails.size,
            jankyFrames = jankyFrames.size,
            jankyRate = jankyFrames.size.toFloat() / frameDetails.size,
            avgLayoutMs = jankyFrames.map { it.layoutMeasureMs }.average(),
            avgDrawMs = jankyFrames.map { it.drawMs }.average(),
            avgGpuMs = jankyFrames.map { it.gpuMs }.average(),
            worstFrameMs = jankyFrames.maxOfOrNull { it.totalMs } ?: 0
        )
    }
}
```

adb:  
```
# 获取帧率统计
adb shell dumpsys gfxinfo com.xxx.app

# 输出关键部分：
# Stats since: 123456789ns
# Total frames rendered: 1200
# Janky frames: 45 (3.75%)       ← 卡顿帧数和比例
# 50th percentile: 8ms
# 90th percentile: 16ms
# 95th percentile: 24ms          ← 5%的帧超过24ms
# 99th percentile: 48ms          ← 1%的帧超过48ms
# Number Missed Vsync: 12        ← 错过VSync次数
# Number High input latency: 8   ← 输入延迟次数
# Number Slow UI thread: 23      ← 主线程慢次数
# Number Slow bitmap uploads: 3  ← Bitmap上传GPU慢
# Number Slow issue draw commands: 5 ← 绘制命令慢

# 重置统计
adb shell dumpsys gfxinfo com.xxx.app reset

# 获取详细帧数据（每帧各阶段耗时）
adb shell dumpsys gfxinfo com.xxx.app framestats
```
## 布局优化
setContentView → XML 解析 → 反射创建 View → 耗时
### 异步 Inflate
AsyncLayoutInflater：子线程执行 XML 解析 + View 创建 → 完成后回调主线程 → 主线程 addView/setContentView。    
如果 inflate 完成前，主线程就需要这个 View，主线程只能等回调，所以其实一般不用于加速渲染，而且用来预加载：  
```
class HomeActivity : AppCompatActivity() {
    private var detailView: View? = null
    
    override fun onCreate(...) {
        setContentView(R.layout.activity_home)
        // 主线程展示首页的同时，子线程预加载详情页布局
        // 用户还没点击，有充足时间
        AsyncLayoutInflater(this).inflate(
            R.layout.activity_detail, null
        ) { view, _, _ ->
            detailView = view // 提前准备好
        }
    }
    
    fun onItemClick() {
        // 用户点击时，布局已经准备好了，直接用
        detailView?.let {
            // 直接展示，无需等待 inflate
        }
    }
}
```
### 测量次数
1.measure 流程：  
```
父View调用子View.measure()
  → 子View.onMeasure()
    → 测量自身大小
    → 如果有子View，递归测量子View
  → 返回测量结果给父View

MeasureSpec（测量规格）：
├── EXACTLY：父View指定了精确大小（match_parent / 具体dp值）
├── AT_MOST：父View指定了最大值（wrap_content）
└── UNSPECIFIED：父View不限制（ScrollView内的子View）

性能关键：
wrap_content = AT_MOST
  → 子View需要先测量自身内容大小
  → 父View再根据结果决定自身大小
  → 可能触发多次测量！

RelativeLayout 的双重测量问题：
RelativeLayout 需要测量两次：
  第一次：水平方向测量
  第二次：垂直方向测量
嵌套 RelativeLayout → 指数级测量次数！
```


2.优化：  
```
// ❌ 问题：嵌套 RelativeLayout
// 外层 RelativeLayout measure 2次
//   内层 RelativeLayout measure 2次
//   = 总共 4次 measure（每层嵌套翻倍）

// ✅ 优化：ConstraintLayout 替代嵌套布局
// ConstraintLayout 只 measure 1次（大多数情况）
// 且可以表达复杂的约束关系，不需要嵌套

// ConstraintLayout 性能关键点：
// 1. 避免过多 chain（链式约束计算复杂）
// 2. 避免循环依赖约束
// 3. 使用 Barrier 替代多层嵌套
```
### 减少层级
1.ConstraintLayout 一层解决嵌套问题  
2.避免 RelativeLayout 嵌套 LinearLayout  
### Merge/ViewStub/Barrier
```
<!-- include：复用布局，但会增加层级 -->
<include layout="@layout/header"/>
<!-- 等价于把 header.xml 的内容直接放在这里 -->
<!-- 如果 header.xml 根节点是 LinearLayout -->
<!-- 则会多一层 LinearLayout -->

<!-- merge：消除多余层级 -->
<!-- header.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- merge 的子View直接合并到父容器 -->
    <TextView android:id="@+id/title"/>
    <ImageView android:id="@+id/logo"/>
</merge>

<!-- 使用 include 引入 merge 布局 -->
<!-- 父容器是 LinearLayout -->
<LinearLayout>
    <include layout="@layout/header"/>
    <!-- 等价于直接写 TextView 和 ImageView -->
    <!-- 没有多余的 LinearLayout 层级 -->
</LinearLayout>

<!-- 注意：merge 只能作为根节点使用 -->
<!-- 且必须通过 include 引入，不能直接 inflate -->



// merge 布局的正确 inflate 方式
class HeaderView(context: Context, attrs: AttributeSet?) 
    : LinearLayout(context, attrs) {
    
    init {
        // 必须传入 this 作为 parent，且 attachToRoot = true
        LayoutInflater.from(context).inflate(
            R.layout.header,
            this,      // parent = this（LinearLayout）
            true       // attachToRoot = true
        )
        // merge 的子View会直接添加到 this（LinearLayout）
        // 不会多一层包裹
    }
}
```

Barrier:  
```
<!-- Barrier：动态边界，替代复杂嵌套 -->
<androidx.constraintlayout.widget.ConstraintLayout>

    <TextView android:id="@+id/label1" .../>
    <TextView android:id="@+id/label2" .../>

    <!-- Barrier：取 label1 和 label2 中较宽的那个的右边界 -->
    <androidx.constraintlayout.widget.Barrier
        android:id="@+id/barrier"
        app:barrierDirection="end"
        app:constraint_referenced_ids="label1,label2"/>

    <!-- content 对齐到 Barrier 右侧 -->
    <TextView
        android:id="@+id/content"
        app:layout_constraintStart_toEndOf="@id/barrier"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```
## 过度绘制
```
过度绘制（Overdraw）：
同一个像素在同一帧内被绘制多次

示例：
┌─────────────────┐
│  Activity背景    │ ← 第1次绘制
│  ┌───────────┐  │
│  │Fragment背景│  │ ← 第2次绘制（覆盖了Activity背景）
│  │ ┌───────┐ │  │
│  │ │卡片背景│ │  │ ← 第3次绘制
│  │ │ ┌───┐ │ │  │
│  │ │ │文字│ │ │  │ ← 第4次绘制
│  │ │ └───┘ │ │  │
│  │ └───────┘ │  │
│  └───────────┘  │
└─────────────────┘

底层的绘制完全被上层覆盖 → 浪费 GPU 资源
```
### GPU 过度绘制检测
```
开启 GPU 过度绘制调试：
设置 → 开发者选项 → 调试 GPU 过度绘制

颜色含义：
┌──────────┬──────────┬─────────────────────────┐
│ 颜色     │ 过度绘制  │ 说明                     │
├──────────┼──────────┼─────────────────────────┤
│ 白色     │ 0次      │ 理想状态                  │
│ 蓝色     │ 1次      │ 可接受                    │
│ 绿色     │ 2次      │ 需要关注                  │
│ 粉色     │ 3次      │ 需要优化                  │
│ 红色     │ 4次+     │ 严重问题，必须优化         │
└──────────┴──────────┴─────────────────────────┘

优化目标：界面大部分区域为蓝色，无红色区域
```

### 优化
1.移除不必要的背景:  
```
// ❌ 问题：多层背景叠加
// styles.xml
<style name="AppTheme">
    <item name="android:windowBackground">@color/white</item>
    <!-- Activity 有背景 -->
</style>

// activity_main.xml
<LinearLayout
    android:background="@color/white">  <!-- Layout 又设置了相同背景 -->
    <RecyclerView
        android:background="@color/white"/> <!-- RecyclerView 又设置了背景 -->
</LinearLayout>

// ✅ 优化：移除冗余背景
// 方式一：主题中移除 windowBackground
<style name="AppTheme">
    <item name="android:windowBackground">@null</item>
</style>

// 方式二：代码中移除
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 移除 window 背景（如果 DecorView 上层有背景）
        window.setBackgroundDrawable(null)
        setContentView(R.layout.activity_main)
    }
}
```

2.clipRect 裁剪绘制区域:  
```
// 场景：卡片叠加效果（如扑克牌）
class OverlapCardView(context: Context) : View(context) {

    private val cards = listOf(/* 多张卡片数据 */)

    override fun onDraw(canvas: Canvas) {
        cards.forEachIndexed { index, card ->
            canvas.save()

            if (index < cards.size - 1) {
                // 不是最后一张：只绘制露出的部分
                // 使用 clipRect 限制绘制区域
                canvas.clipRect(
                    card.left,
                    card.top,
                    card.left + VISIBLE_WIDTH, // 只露出这么宽
                    card.bottom
                )
            }
            // 在裁剪区域内绘制卡片
            drawCard(canvas, card)

            canvas.restore()
        }
    }
}
```

3.自定义 View :  
```
class OptimizedCardView(context: Context) : View(context) {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val bgPaint = Paint().apply {
        color = Color.WHITE
        style = Paint.Style.FILL
    }

    override fun onDraw(canvas: Canvas) {
        // ❌ 错误：先绘制背景，再绘制内容
        // 如果内容完全覆盖背景 → 背景绘制浪费
        canvas.drawRect(bounds, bgPaint)  // 背景
        canvas.drawBitmap(bitmap, 0f, 0f, paint) // 内容（完全覆盖背景）

        // ✅ 优化：判断是否需要绘制背景
        // 如果内容不透明且完全覆盖 → 跳过背景绘制
        if (!isContentOpaque()) {
            canvas.drawRect(bounds, bgPaint)
        }
        canvas.drawBitmap(bitmap, 0f, 0f, paint)
    }

    private fun isContentOpaque(): Boolean {
        return bitmap?.hasAlpha() == false
    }
}
```
### quickReject
原理：  
```
quickReject：Canvas 的快速裁剪判断
在真正绘制之前，先判断要绘制的区域是否在
当前裁剪区域（clip region）内

如果完全在裁剪区域外 → 直接跳过绘制
不需要执行实际的绘制操作

作用：
├── 跳过不可见区域的绘制
├── 减少 GPU 绘制指令
└── 降低 DisplayList 复杂度
```

使用：  
```
class OptimizedView(context: Context) : View(context) {

    private val items = mutableListOf<DrawItem>()
    private val paint = Paint()
    private val rectF = RectF()

    override fun onDraw(canvas: Canvas) {
        items.forEach { item ->

            // ================================
            // quickReject：绘制前先判断是否可见
            // ================================
            rectF.set(item.left, item.top, item.right, item.bottom)

            // canvas.quickReject 返回 true = 不可见，跳过
            // 返回 false = 可能可见，继续绘制
            if (canvas.quickReject(rectF, Canvas.EdgeType.BW)) {
                return@forEach // 跳过这个item的绘制
            }

            // 通过 quickReject 检查，执行实际绘制
            drawItem(canvas, item)
        }
    }

    // 自定义 ViewGroup 中优化子 View 绘制
    override fun dispatchDraw(canvas: Canvas) {
        for (i in 0 until childCount) {
            val child = getChildAt(i)

            // 判断子 View 是否在可见区域内
            val childRect = RectF(
                child.left.toFloat(),
                child.top.toFloat(),
                child.right.toFloat(),
                child.bottom.toFloat()
            )

            if (!canvas.quickReject(childRect, Canvas.EdgeType.BW)) {
                // 子 View 可见，才绘制
                drawChild(canvas, child, drawingTime)
            }
            // 不可见的子 View 直接跳过
        }
    }
}
```

RV:  
```
// RecyclerView 的 ItemDecoration 中使用 quickReject
class OptimizedDividerDecoration : RecyclerView.ItemDecoration() {

    private val paint = Paint().apply {
        color = Color.LTGRAY
        strokeWidth = 1f
    }

    override fun onDraw(canvas: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        val left = parent.paddingLeft.toFloat()
        val right = (parent.width - parent.paddingRight).toFloat()

        for (i in 0 until parent.childCount) {
            val child = parent.getChildAt(i)
            val top = child.bottom.toFloat()
            val bottom = top + 1f

            // quickReject 判断分割线是否在可见区域
            if (!canvas.quickReject(left, top, right, bottom, Canvas.EdgeType.BW)) {
                canvas.drawLine(left, top, right, top, paint)
            }
        }
    }
}
```

硬件加速:  
```
注意：硬件加速开启时，quickReject 行为有变化

软件渲染：
quickReject 精确判断裁剪区域
完全在裁剪区域外 → 返回 true

硬件加速：
quickReject 判断的是 View 的绘制边界
不是精确的像素级裁剪
但仍然有效，可以跳过明显不可见的绘制

建议：
├── 自定义 View 的 onDraw 中使用 quickReject
├── 复杂 ViewGroup 的 dispatchDraw 中使用
└── 有大量子元素的自定义布局中使用
```
## 列表优化
### RecyclerView 
#### 四级缓存
```
┌─────────────────────────────────────────────────────┐
│ 一级缓存：mAttachedScrap / mChangedScrap             │
│ 存放：当前屏幕上正在显示的 ViewHolder                 │
│ 大小：无限制                                          │
│ 特点：直接复用，不需要 onBindViewHolder               │
├─────────────────────────────────────────────────────┤
│ 二级缓存：mCachedViews                               │
│ 存放：刚滑出屏幕的 ViewHolder（保留数据）             │
│ 大小：默认 2 个                                       │
│ 特点：直接复用，不需要 onBindViewHolder               │
├─────────────────────────────────────────────────────┤
│ 三级缓存：ViewCacheExtension（自定义缓存）            │
│ 存放：开发者自定义                                    │
│ 特点：适用于特殊场景                                  │
├─────────────────────────────────────────────────────┤
│ 四级缓存：RecycledViewPool                           │
│ 存放：回收的 ViewHolder（数据已清除）                 │
│ 大小：每种 viewType 默认 5 个                         │
│ 特点：需要 onBindViewHolder 重新绑定数据              │
│ 优势：可以跨 RecyclerView 共享                        │
└─────────────────────────────────────────────────────┘

滑动时的查找顺序：
一级 → 二级 → 三级 → 四级 → 创建新的
```
#### 性能优化
1.setHasFixedSize:  
```
recyclerView.setHasFixedSize(true)
// 告诉 RecyclerView：数据变化不影响 RV 自身大小
// 跳过 requestLayout，只重绘变化的 item
// 适用：RecyclerView 是 match_parent 时
```

2.增大 mCachedViews 大小:  
```
recyclerView.setItemViewCacheSize(5) // 默认2，增大到5
// 更多 ViewHolder 保留在二级缓存
// 快速滑动回来时直接复用，无需 onBindViewHolder
```

3.共享 RecycledViewPool:  
```
// 场景：ViewPager 中多个 Tab 都有 RecyclerView, 且 itemType 相同

val sharedPool = RecyclerView.RecycledViewPool()
sharedPool.setMaxRecycledViews(0, 20)
// viewType=0，最多缓存20个

tabRecyclerView1.setRecycledViewPool(sharedPool)
tabRecyclerView2.setRecycledViewPool(sharedPool)
tabRecyclerView3.setRecycledViewPool(sharedPool)
// 切换 Tab 时，ViewHolder 可以跨 RV 复用
```

4.预取（Prefetch）:
```
// 嵌套 RecyclerView 的预取配置
class OuterAdapter : RecyclerView.Adapter<OuterViewHolder>() {
    override fun onBindViewHolder(holder: OuterViewHolder, position: Int) {
        val innerLayoutManager = LinearLayoutManager(
            holder.itemView.context,
            LinearLayoutManager.HORIZONTAL,
            false
        )
        // 设置预取数量（提前创建几个 item）
        innerLayoutManager.initialPrefetchItemCount = 4

        holder.innerRecyclerView.layoutManager = innerLayoutManager
        holder.innerRecyclerView.adapter = InnerAdapter(data[position])
    }
}
```

5.DiffUtil:  
```
class HomeAdapter : ListAdapter<Item, ItemViewHolder>(ItemDiffCallback()) {

    // DiffUtil 在后台线程计算差异
    // 只更新真正变化的 item，而不是 notifyDataSetChanged
}

class ItemDiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(old: Item, new: Item): Boolean {
        return old.id == new.id // 判断是否同一个item
    }

    override fun areContentsTheSame(old: Item, new: Item): Boolean {
        return old == new // 判断内容是否变化（data class自动实现）
    }

    // 局部更新（只更新变化的字段）
    override fun getChangePayload(old: Item, new: Item): Any? {
        val payload = Bundle()
        if (old.title != new.title) payload.putString("title", new.title)
        if (old.price != new.price) payload.putString("price", new.price)
        return if (payload.isEmpty) null else payload
    }
}

// ViewHolder 中处理局部更新
class ItemViewHolder(view: View) : RecyclerView.ViewHolder(view) {

    fun bind(item: Item) {
        titleView.text = item.title
        priceView.text = item.price
    }

    // 局部更新：只刷新变化的字段
    fun bindPayload(payload: Bundle) {
        payload.getString("title")?.let { titleView.text = it }
        payload.getString("price")?.let { priceView.text = it }
    }
}
```

6.onBindViewHolder:  
```
class ItemViewHolder(view: View) : RecyclerView.ViewHolder(view) {

    private val titleView: TextView = view.findViewById(R.id.title)
    private val imageView: ImageView = view.findViewById(R.id.image)
    private val priceView: TextView = view.findViewById(R.id.price)

    fun bind(item: Item) {
        // ❌ 错误：onBindViewHolder 中做耗时操作
        val formattedPrice = formatPrice(item.price) // 字符串格式化
        val processedTitle = processTitle(item.title) // 文本处理
        val bitmap = decodeBitmap(item.imageUrl)      // 图片解码！

        // ✅ 正确：onBindViewHolder 只做赋值
        titleView.text = item.title
        priceView.text = item.formattedPrice // 数据层预处理好
        Glide.with(imageView).load(item.imageUrl).into(imageView)

        // ✅ 使用 PrecomputedText 优化文本测量
        // 在数据层预计算文本布局
        titleView.setTextFuture(
            PrecomputedTextCompat.getTextFuture(
                item.title,
                TextViewCompat.getTextMetricsParams(titleView),
                null
            )
        )
    }
}
```

7.嵌套RV:  
```
// 场景：垂直列表中嵌套水平列表（常见于首页）

class OuterAdapter(
    private val sharedPool: RecyclerView.RecycledViewPool
) : RecyclerView.Adapter<OuterViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int)
        : OuterViewHolder {
        val holder = OuterViewHolder(
            LayoutInflater.from(parent.context)
                .inflate(R.layout.item_outer, parent, false)
        )

        // 关键优化：内层 RV 共享 RecycledViewPool
        holder.innerRecyclerView.setRecycledViewPool(sharedPool)

        // 内层 RV 的 LayoutManager 复用
        holder.innerRecyclerView.layoutManager =
            LinearLayoutManager(parent.context, HORIZONTAL, false).apply {
                initialPrefetchItemCount = 4
            }

        return holder
    }

    override fun onViewRecycled(holder: OuterViewHolder) {
        super.onViewRecycled(holder)
        // 外层 item 回收时，保存内层滚动位置
        holder.saveScrollState()
    }

    override fun onBindViewHolder(holder: OuterViewHolder, position: Int) {
        // 恢复内层滚动位置
        holder.restoreScrollState(position)
        holder.bind(data[position])
    }
}
```
#### ItemAnimator 优化
问题：  
```
RecyclerView 默认使用 DefaultItemAnimator
├── 每次 notifyItemChanged 触发淡入淡出动画
├── 动画期间 ViewHolder 被占用，无法复用
├── 大量数据更新时：动画堆积 → 卡顿
└── 某些场景动画完全不需要（如后台静默更新）
```

优化：  
// ================================
// 方案一：禁用动画（数据频繁更新场景）
// ================================
recyclerView.itemAnimator = null
// 完全禁用，notifyItemChanged 直接刷新，无动画
// 适用：股票行情、实时数据等高频更新场景

// ================================
// 方案二：禁用特定动画
// ================================
(recyclerView.itemAnimator as? SimpleItemAnimator)
    ?.supportsChangeAnimations = false
// 只禁用 change 动画（notifyItemChanged触发的）
// 保留 add/remove 动画
// 适用：列表数据更新但不需要闪烁效果

// ================================
// 方案三：自定义轻量 ItemAnimator
// ================================
class LightItemAnimator : DefaultItemAnimator() {

    override fun animateChange(
        oldHolder: RecyclerView.ViewHolder,
        newHolder: RecyclerView.ViewHolder,
        preInfo: ItemHolderInfo,
        postInfo: ItemHolderInfo
    ): Boolean {
        // 直接完成，不播放动画
        // 但通知框架动画已结束（必须调用！）
        dispatchChangeFinished(oldHolder, true)
        if (oldHolder !== newHolder) {
            dispatchChangeFinished(newHolder, false)
        }
        return false // false = 不需要runPendingAnimations
    }

    // 缩短动画时长
    init {
        addDuration = 120   // 默认250ms
        removeDuration = 120
        moveDuration = 120
        changeDuration = 0  // change动画直接0ms
    }
}

recyclerView.itemAnimator = LightItemAnimator()

// ================================
// 方案四：Payload 局部更新避免动画
// ================================
// 使用 payload 更新时，DefaultItemAnimator
// 会跳过 change 动画，直接刷新变化的部分

adapter.notifyItemChanged(position, "price_changed") // 带payload

// Adapter 中处理
override fun onBindViewHolder(
    holder: ViewHolder,
    position: Int,
    payloads: List<Any>
) {
    if (payloads.isEmpty()) {
        // 完整绑定
        bind(holder, getItem(position))
    } else {
        // 局部更新，无动画
        payloads.forEach { payload ->
            if (payload == "price_changed") {
                holder.updatePrice(getItem(position).price)
            }
        }
    }
}
### Paging3
核心架构：  
```
三层架构：
┌─────────────────────────────────────┐
│  UI 层                               │
│  PagingDataAdapter                  │
│  └── 自动处理加载状态/错误/重试       │
├─────────────────────────────────────┤
│  ViewModel 层                        │
│  Pager → Flow<PagingData>           │
│  └── 管理分页数据流                  │
├─────────────────────────────────────┤
│  数据层                              │
│  PagingSource                       │
│  └── 定义如何加载数据（网络/数据库）  │
│  RemoteMediator（可选）              │
│  └── 网络+数据库组合                 │
└─────────────────────────────────────┘
```

完整实现：  
```
// ================================
// Step1：PagingSource（数据来源）
// ================================
class ArticlePagingSource(
    private val api: ArticleApi,
    private val query: String
) : PagingSource<Int, Article>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1 // 默认第一页

        return try {
            val response = api.getArticles(
                query = query,
                page = page,
                pageSize = params.loadSize // Paging3自动管理loadSize
            )

            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.articles.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e) // 自动触发错误状态
        }
    }

    // 刷新时从哪一页开始（支持恢复滚动位置）
    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}

// ================================
// Step2：ViewModel
// ================================
class ArticleViewModel(private val api: ArticleApi) : ViewModel() {

    private val currentQuery = MutableStateFlow("Android")

    val articles: Flow<PagingData<Article>> = currentQuery
        .flatMapLatest { query ->
            Pager(
                config = PagingConfig(
                    pageSize = 20,           // 每页数量
                    prefetchDistance = 5,    // 距底部5条时预加载
                    enablePlaceholders = false, // 是否显示占位符
                    initialLoadSize = 40     // 首次加载数量（通常2倍pageSize）
                ),
                pagingSourceFactory = {
                    ArticlePagingSource(api, query)
                }
            ).flow
        }
        .cachedIn(viewModelScope) // 缓存在ViewModel，旋转屏幕不重新加载
}

// ================================
// Step3：Adapter
// ================================
class ArticleAdapter : PagingDataAdapter<Article, ArticleViewHolder>(
    ArticleDiffCallback()
) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int)
        : ArticleViewHolder {
        return ArticleViewHolder(
            LayoutInflater.from(parent.context)
                .inflate(R.layout.item_article, parent, false)
        )
    }

    override fun onBindViewHolder(holder: ArticleViewHolder, position: Int) {
        getItem(position)?.let { holder.bind(it) }
        // getItem 可能返回 null（enablePlaceholders=true时）
    }
}

class ArticleDiffCallback : DiffUtil.ItemCallback<Article>() {
    override fun areItemsTheSame(old: Article, new: Article) =
        old.id == new.id
    override fun areContentsTheSame(old: Article, new: Article) =
        old == new
}

// ================================
// Step4：加载状态处理
// ================================
class ArticleFragment : Fragment() {

    private val viewModel: ArticleViewModel by viewModels()
    private val adapter = ArticleAdapter()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // 添加加载状态 Footer
        recyclerView.adapter = adapter.withLoadStateFooter(
            footer = LoadStateAdapter { adapter.retry() }
        )

        // 观察加载状态
        lifecycleScope.launch {
            adapter.loadStateFlow.collect { loadState ->
                // 首次加载
                progressBar.isVisible =
                    loadState.refresh is LoadState.Loading

                // 首次加载错误
                errorView.isVisible =
                    loadState.refresh is LoadState.Error

                // 显示错误信息
                (loadState.refresh as? LoadState.Error)?.let {
                    errorText.text = it.error.message
                }
            }
        }

        // 收集分页数据
        lifecycleScope.launch {
            viewModel.articles.collectLatest { pagingData ->
                adapter.submitData(pagingData)
            }
        }
    }
}

// ================================
// LoadState Footer Adapter
// ================================
class LoadStateAdapter(
    private val retry: () -> Unit
) : LoadStateAdapter<LoadStateViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, loadState: LoadState)
        : LoadStateViewHolder {
        return LoadStateViewHolder(
            LayoutInflater.from(parent.context)
                .inflate(R.layout.item_load_state, parent, false),
            retry
        )
    }

    override fun onBindViewHolder(holder: LoadStateViewHolder, loadState: LoadState) {
        holder.bind(loadState)
    }
}

class LoadStateViewHolder(view: View, retry: () -> Unit)
    : RecyclerView.ViewHolder(view) {

    private val progressBar = view.findViewById<ProgressBar>(R.id.progress)
    private val retryButton = view.findViewById<Button>(R.id.retry)
    private val errorText = view.findViewById<TextView>(R.id.error_msg)

    init {
        retryButton.setOnClickListener { retry() }
    }

    fun bind(loadState: LoadState) {
        progressBar.isVisible = loadState is LoadState.Loading
        retryButton.isVisible = loadState is LoadState.Error
        errorText.isVisible = loadState is LoadState.Error
        (loadState as? LoadState.Error)?.let {
            errorText.text = it.error.message
        }
    }
}
```

Paging3 + Room（RemoteMediator):  
```
// 网络数据缓存到本地，离线可用
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, Article>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, Article>
    ): MediatorResult {

        val page = when (loadType) {
            LoadType.REFRESH -> 1  // 刷新从第一页开始
            LoadType.PREPEND -> return MediatorResult.Success(
                endOfPaginationReached = true // 不支持向上加载
            )
            LoadType.APPEND -> {
                // 计算下一页页码
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(
                        endOfPaginationReached = true
                    )
                getNextPageForItem(lastItem)
            }
        }

        return try {
            val response = api.getArticles(page = page, pageSize = state.config.pageSize)

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.articleDao().clearAll() // 刷新时清空旧数据
                }
                db.articleDao().insertAll(response.articles)
            }

            MediatorResult.Success(
                endOfPaginationReached = response.articles.isEmpty()
            )
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```
## 自定义 View
### onDraw
```
class OptimizedCustomView(context: Context) : View(context) {

    // ✅ Paint 在类初始化时创建，不在 onDraw 中创建
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLUE
        strokeWidth = 4f
        style = Paint.Style.STROKE
    }

    // ✅ Path 复用，不在 onDraw 中 new
    private val path = Path()

    // ✅ RectF 复用
    private val rectF = RectF()

    // ✅ 数据变化时才重新计算，不在 onDraw 中计算
    private var cachedPoints: List<PointF> = emptyList()

    fun updateData(newData: List<Float>) {
        // 数据变化时预计算绘制路径
        cachedPoints = calculatePoints(newData)
        path.reset()
        cachedPoints.forEachIndexed { index, point ->
            if (index == 0) path.moveTo(point.x, point.y)
            else path.lineTo(point.x, point.y)
        }
        invalidate() // 触发重绘
    }

    override fun onDraw(canvas: Canvas) {
        // ✅ onDraw 只做绘制，不做计算
        canvas.drawPath(path, paint)
    }

    // ✅ onSizeChanged 中处理尺寸相关计算
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        rectF.set(0f, 0f, w.toFloat(), h.toFloat())
        // 尺寸变化时重新计算
    }
}
```
### 硬件加速
```
// 硬件加速：默认开启（Android 4.0+）
// 将 View 的绘制指令转换为 GPU 指令

// 某些 Canvas 操作不支持硬件加速
// 需要关闭硬件加速（软件渲染）
class SpecialView(context: Context) : View(context) {
    init {
        // 关闭这个 View 的硬件加速（只影响此View）
        setLayerType(LAYER_TYPE_SOFTWARE, null)
    }
}

// View Layer 优化动画
// 场景：View 做透明度/位移/缩放动画
class AnimatedView(context: Context) : View(context) {

    fun startAnimation() {
        // 动画开始前：将 View 缓存为 GPU 纹理
        // 动画过程中：直接操作纹理，不需要重新绘制 View
        setLayerType(LAYER_TYPE_HARDWARE, null)

        ObjectAnimator.ofFloat(this, "alpha", 1f, 0f).apply {
            duration = 300
            doOnEnd {
                // 动画结束后：移除 Layer，释放 GPU 内存
                setLayerType(LAYER_TYPE_NONE, null)
            }
            start()
        }
    }
}

// 注意：LAYER_TYPE_HARDWARE 会占用 GPU 内存
// 动画结束后必须移除！否则内存泄漏
```
## RenderThread
### 减少 DisplayList 构建耗时
```
DisplayList：View.draw() 生成的渲染指令列表
主线程：执行 View.draw() → 生成 DisplayList
RenderThread：执行 DisplayList 中的指令 → 提交GPU

DisplayList 耗时来源：
├── onDraw 中复杂的 Canvas 操作
├── 大量 View 的 draw 递归调用
├── 频繁 invalidate 导致 DisplayList 重建
└── 大 Bitmap 上传 GPU（sync阶段）
```

优化：  
```
// ================================
// 优化一：减少 invalidate 范围
// ================================
class ChartView(context: Context) : View(context) {

    private var highlightIndex = -1

    fun highlightItem(index: Int) {
        val oldIndex = highlightIndex
        highlightIndex = index

        // ❌ 全局 invalidate：整个 View 重建 DisplayList
        // invalidate()

        // ✅ 局部 invalidate：只重建变化区域的 DisplayList
        // 计算旧高亮区域
        if (oldIndex >= 0) {
            val oldRect = getItemRect(oldIndex)
            invalidate(oldRect) // 只刷新旧区域
        }
        // 计算新高亮区域
        val newRect = getItemRect(index)
        invalidate(newRect) // 只刷新新区域
    }

    private fun getItemRect(index: Int): Rect {
        val itemWidth = width / itemCount
        return Rect(
            index * itemWidth, 0,
            (index + 1) * itemWidth, height
        )
    }
}

// ================================
// 优化二：View Layer 缓存 DisplayList
// ================================
// 场景：View 内容不变，只做位移/缩放/透明度变化
// 使用 Hardware Layer 缓存 DisplayList 为 GPU 纹理
// 动画时直接操作纹理，不重建 DisplayList

class AnimatedCardView(context: Context) : View(context) {

    fun startSlideAnimation() {
        // 缓存为 GPU 纹理
        setLayerType(LAYER_TYPE_HARDWARE, null)

        animate()
            .translationX(200f)
            .setDuration(300)
            .withEndAction {
                // 动画结束释放纹理
                setLayerType(LAYER_TYPE_NONE, null)
            }
            .start()
        // 动画过程中 DisplayList 不重建
        // 直接操作 GPU 纹理，极低开销
    }
}

// ================================
// 优化三：避免大 Bitmap 频繁上传 GPU
// ================================
// sync 阶段：主线程将新的 Bitmap 上传到 GPU 纹理
// 大 Bitmap 上传耗时 = sync 阶段卡顿

class ImageManager {

    // ❌ 问题：每次都创建新 Bitmap，每次都上传 GPU
    fun updateImage(view: ImageView, newBitmap: Bitmap) {
        view.setImageBitmap(newBitmap) // 触发 GPU 上传
    }

    // ✅ 优化：复用 Bitmap（使用 inBitmap）
    fun updateImageWithReuse(view: ImageView, imageUrl: String) {
        Glide.with(view)
            .load(imageUrl)
            // Glide 内部复用 Bitmap，减少 GPU 上传
            .into(view)
    }

    // ✅ 优化：降低 Bitmap 分辨率
    fun loadScaledBitmap(context: Context, resId: Int, targetWidth: Int): Bitmap {
        val options = BitmapFactory.Options().apply {
            inJustDecodeBounds = true
            BitmapFactory.decodeResource(context.resources, resId, this)

            // 计算采样率
            inSampleSize = calculateInSampleSize(this, targetWidth)
            inJustDecodeBounds = false
            inPreferredConfig = Bitmap.Config.RGB_565 // 减少内存占用
        }
        return BitmapFactory.decodeResource(context.resources, resId, options)
    }
}
```
### 属性动画优化
问题：  
```
属性动画（ObjectAnimator/ValueAnimator）：
每一帧：
① Choreographer 回调
② 计算当前动画值（插值器计算）
③ 调用 setter（如 setTranslationX）
④ 触发 invalidate
⑤ 重建 DisplayList
⑥ RenderThread 渲染

├── 每帧都在主线程执行 ①②③④
├── 复杂动画 = 每帧主线程耗时增加
└── 多个动画同时运行 = 主线程压力倍增
```

优化：  
```
// ================================
// 优化一：使用 RenderThread 动画
// ViewPropertyAnimator 部分操作在 RenderThread 执行
// ================================

// ❌ ObjectAnimator：完全在主线程
ObjectAnimator.ofFloat(view, "translationX", 0f, 200f).start()

// ✅ ViewPropertyAnimator：translationX/Y/Z、rotation、
//    scaleX/Y、alpha 等变换在 RenderThread 执行
//    不占用主线程！
view.animate()
    .translationX(200f)
    .alpha(0.5f)
    .rotation(45f)
    .setDuration(300)
    .start()

// ================================
// 优化二：使用 Animator 而非 Animation
// ================================
// Animation（补间动画）：只改变绘制位置，不改变实际属性
//   → 点击区域不跟着移动，有歧义
//   → 性能和 Animator 差不多，但功能受限

// Animator（属性动画）：真正改变 View 属性
//   → 推荐使用

// ================================
// 优化三：减少动画对象创建
// ================================
class AnimationHelper {

    // ❌ 每次动画都创建新的 ObjectAnimator
    fun startBadAnimation(view: View) {
        ObjectAnimator.ofFloat(view, "alpha", 0f, 1f)
            .setDuration(300)
            .start()
    }

    // ✅ 复用 ObjectAnimator
    private val alphaAnimator = ObjectAnimator.ofFloat(
        null, "alpha", 0f, 1f
    ).apply {
        duration = 300
    }

    fun startGoodAnimation(view: View) {
        alphaAnimator.target = view
        alphaAnimator.start()
    }
}

// ================================
// 优化四：AnimatorSet 替代多个独立动画
// ================================
// ❌ 多个独立动画：各自触发 invalidate
view1.animate().alpha(0f).start()
view2.animate().translationY(100f).start()
view3.animate().scaleX(0.8f).start()

// ✅ AnimatorSet：协调多个动画，减少重绘
AnimatorSet().apply {
    playTogether(
        ObjectAnimator.ofFloat(view1, "alpha", 0f),
        ObjectAnimator.ofFloat(view2, "translationY", 100f),
        ObjectAnimator.ofFloat(view3, "scaleX", 0.8f)
    )
    duration = 300
    start()
}

// ================================
// 优化五：插值器选择
// ================================
// 复杂插值器（如 BounceInterpolator）每帧计算量大
// 简单动画用 LinearInterpolator 或 AccelerateDecelerateInterpolator
ObjectAnimator.ofFloat(view, "translationX", 200f).apply {
    interpolator = LinearInterpolator() // 最简单，计算量最小
    duration = 200
    start()
}
```
## 窗口动画
### 共享元素动画
原理：  
```
共享元素动画（Shared Element Transition）：
两个 Activity/Fragment 之间，某个 View 看起来
从一个位置"飞"到另一个位置

① 记录起始 View 的位置、大小、外观
② 在目标 Activity/Fragment 中找到对应 View
③ 创建过渡动画：从起始状态变化到目标状态
④ 动画期间：View 在 DecorView 上层绘制（覆盖两个界面）
```

实现：  
```
// ================================
// 基础实现
// ================================

// 列表页
class ArticleListActivity : AppCompatActivity() {

    fun onItemClick(article: Article, imageView: ImageView) {
        val intent = Intent(this, ArticleDetailActivity::class.java)
        intent.putExtra("article", article)

        // 定义共享元素
        val options = ActivityOptionsCompat.makeSceneTransitionAnimation(
            this,
            imageView,                    // 共享的 View
            "article_image"              // transitionName（必须唯一）
        )
        startActivity(intent, options.toBundle())
    }
}

// 列表 item 布局
// <ImageView
//     android:transitionName="article_image_${article.id}"
//     .../>

// 详情页
class ArticleDetailActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 延迟过渡动画，等数据加载完成
        postponeEnterTransition()

        setContentView(R.layout.activity_detail)

        // 设置相同的 transitionName
        detailImageView.transitionName = "article_image"

        // 图片加载完成后开始动画
        Glide.with(this)
            .load(article.imageUrl)
            .listener(object : RequestListener<Drawable> {
                override fun onResourceReady(...): Boolean {
                    // 图片加载完成，开始过渡动画
                    startPostponedEnterTransition()
                    return false
                }
                override fun onLoadFailed(...): Boolean {
                    startPostponedEnterTransition() // 失败也要开始，否则卡住
                    return false
                }
            })
            .into(detailImageView)
    }
}
```

优化：  
```
// ================================
// 优化一：postponeEnterTransition 防止动画错位
// ================================
// 问题：图片未加载完就开始动画 → 动画结束后图片才出现
// 解决：postponeEnterTransition 延迟动画

// ================================
// 优化二：减少共享元素数量
// ================================
// 每个共享元素都需要截图和动画计算
// 共享元素越多，性能开销越大
// 建议：最多 2~3 个共享元素

// ================================
// 优化三：自定义过渡动画
// ================================
class DetailActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 自定义共享元素过渡
        window.sharedElementEnterTransition = buildEnterTransition()
        window.sharedElementReturnTransition = buildReturnTransition()
    }

    private fun buildEnterTransition(): Transition {
        return TransitionSet().apply {
            // 组合多种过渡效果
            addTransition(ChangeBounds())    // 位置和大小变化
            addTransition(ChangeImageTransform()) // ImageView的ScaleType变化
            addTransition(ChangeClipBounds())    // 裁剪边界变化
            duration = 300
            interpolator = FastOutSlowInInterpolator()
        }
    }

    private fun buildReturnTransition(): Transition {
        return TransitionSet().apply {
            addTransition(ChangeBounds())
            addTransition(ChangeImageTransform())
            duration = 250 // 返回动画稍快
            interpolator = FastOutLinearInInterpolator()
        }
    }
}

// ================================
// 优化四：Fragment 共享元素（更流畅）
// ================================
// Activity 间共享元素：需要跨进程通信，有额外开销
// Fragment 间共享元素：在同一个 Activity 内，更流畅

class ListFragment : Fragment() {
    fun navigateToDetail(item: Item, sharedView: View) {
        val detailFragment = DetailFragment.newInstance(item.id)

        // 设置共享元素过渡
        detailFragment.sharedElementEnterTransition =
            TransitionInflater.from(context)
                .inflateTransition(android.R.transition.move)

        parentFragmentManager.beginTransaction()
            .addSharedElement(sharedView, "shared_image")
            .replace(R.id.container, detailFragment)
            .addToBackStack(null)
            .commit()
    }
}
```
## Compose 优化
### 重组（Recomposition）控制
```
// 重组：状态变化时，重新执行 Composable 函数
// 问题：不必要的重组 = 性能浪费

@Composable
fun ParentComposable() {
    var count by remember { mutableStateOf(0) }

    // ❌ count 变化时，整个 ParentComposable 重组
    // ChildComposable 也会重组（即使它不依赖 count）
    Column {
        Text("Count: $count")
        ChildComposable() // 不依赖 count，但也被重组了！
        Button(onClick = { count++ }) { Text("增加") }
    }
}

// ✅ 优化：将状态下沉，缩小重组范围
@Composable
fun ParentComposable() {
    Column {
        CounterSection() // 只有这部分重组
        ChildComposable() // 不受影响
    }
}

@Composable
fun CounterSection() {
    var count by remember { mutableStateOf(0) }
    Text("Count: $count")
    Button(onClick = { count++ }) { Text("增加") }
}
```
### remember/derivedStateOf
```
@Composable
fun OptimizedList(items: List<Item>) {

    // ❌ 每次重组都重新计算
    val expensiveResult = items.filter { it.isActive }
        .sortedBy { it.score }
        .take(10)

    // ✅ remember：缓存计算结果
    // 只在 items 变化时重新计算
    val filteredItems = remember(items) {
        items.filter { it.isActive }
            .sortedBy { it.score }
            .take(10)
    }

    // ✅ derivedStateOf：状态派生
    // 只在派生结果真正变化时触发重组
    val hasActiveItems by remember {
        derivedStateOf { items.any { it.isActive } }
    }
    // items 中非 active 状态变化时，hasActiveItems 不变
    // 不会触发使用 hasActiveItems 的 Composable 重组

    LazyColumn {
        items(filteredItems, key = { it.id }) { item ->
            ItemCard(item = item)
        }
    }
}
```
### LazyColumn 优化
1.key 参数:  
```
LazyColumn {
    items(
        items = articleList,
        key = { article -> article.id } // 提供稳定的key
        // 作用：
        // 1. 数据更新时，Compose 知道哪些item变了
        // 2. 只重组变化的item，不重组全部
        // 3. 支持item动画（添加/删除/移动）
    ) { article ->
        ArticleItem(article = article)
    }
}
```

2.contentType :  
```
LazyColumn {
    items(
        items = mixedList,
        key = { it.id },
        contentType = { item ->
            // 告诉Compose不同类型的item不能复用
            when (item) {
                is BannerItem -> "banner"
                is ArticleItem -> "article"
                is AdItem -> "ad"
                else -> "unknown"
            }
        }
    ) { item ->
        when (item) {
            is BannerItem -> BannerCard(item)
            is ArticleItem -> ArticleCard(item)
            is AdItem -> AdCard(item)
        }
    }
}
```

3.避免在 item 中创建 State:  
```
// ❌ 错误：每次重组都创建新State
@Composable
fun ArticleItem(article: Article) {
    var isExpanded by remember { mutableStateOf(false) }
    // 问题：LazyColumn 回收item时，State丢失
    // 滑回来时 isExpanded 重置为 false
}

// ✅ 正确：State 提升到 ViewModel
@Composable
fun ArticleItem(
    article: Article,
    isExpanded: Boolean,           // 从外部传入
    onExpandChange: (Boolean) -> Unit // 状态变化回调
) {
    // item 只负责展示，不持有状态
}
```

4.图片加载优化:  
```
@Composable
fun ArticleItem(article: Article) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(article.imageUrl)
            .size(200, 150)          // 指定目标尺寸，避免加载大图
            .crossfade(true)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build(),
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = Modifier.size(200.dp, 150.dp)
    )
}
```

5.减少重组范围:  
```
// ❌ 整个item都会因为任何状态变化而重组
@Composable
fun ArticleItem(article: Article, viewModel: ArticleViewModel) {
    val likeCount by viewModel.getLikeCount(article.id).collectAsState()

    Column {
        // likeCount 变化时，整个Column重组
        Text(article.title)    // 不依赖likeCount，但也重组了
        Text(article.content)  // 不依赖likeCount，但也重组了
        LikeButton(count = likeCount) // 依赖likeCount
    }
}

// ✅ 状态下沉，缩小重组范围
@Composable
fun ArticleItem(article: Article, viewModel: ArticleViewModel) {
    Column {
        Text(article.title)    // 不重组
        Text(article.content)  // 不重组
        LikeSection(           // 只有这部分重组
            articleId = article.id,
            viewModel = viewModel
        )
    }
}

@Composable
fun LikeSection(articleId: String, viewModel: ArticleViewModel) {
    val likeCount by viewModel.getLikeCount(articleId).collectAsState()
    LikeButton(count = likeCount)
}
```

6.预加载配置:  
```
LazyColumn(
    // 预加载距离：提前加载屏幕外的item
    // 默认：一个屏幕高度
    // 增大：滑动更流畅，但内存占用增加
    state = rememberLazyListState(),
    // 通过 LazyListState 可以监听滚动状态
) {
    // ...
}

// 监听滚动，实现无限加载
@Composable
fun InfiniteList(viewModel: ListViewModel) {
    val listState = rememberLazyListState()
    val items by viewModel.items.collectAsState()

    // 检测是否滚动到底部
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()
            val totalItems = listState.layoutInfo.totalItemsCount
            lastVisibleItem?.index != null &&
            lastVisibleItem.index >= totalItems - 5 // 距底部5条时加载
        }
    }

    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore) viewModel.loadMore()
    }

    LazyColumn(state = listState) {
        items(items, key = { it.id }) { item ->
            ItemCard(item)
        }
    }
}
```
### 稳定性
```
// Compose 重组优化依赖「稳定性」判断
// 稳定类型：基本类型、String、@Stable/@Immutable 标注的类
// 不稳定类型：普通 class（Compose 无法判断是否变化）

// ❌ 不稳定：普通 data class
data class UserState(
    val name: String,
    val avatar: String,
    val followers: Int
)
// Compose 认为 UserState 可能随时变化
// 每次父组件重组，使用 UserState 的子组件都会重组

// ✅ 稳定：添加 @Stable 或 @Immutable 注解
@Immutable // 告诉 Compose：这个类的属性不会变化
data class UserState(
    val name: String,
    val avatar: String,
    val followers: Int
)
// 现在 Compose 知道 UserState 是不可变的
// 只有引用变化时才重组

// ✅ 使用 ImmutableList 替代 List
// List 是不稳定的（Compose 不知道内容是否变化）
// ImmutableList（kotlinx.collections.immutable）是稳定的
@Composable
fun ItemList(
    items: ImmutableList<Item> // 而不是 List<Item>
) {
    LazyColumn {
        items(items) { ItemCard(it) }
    }
}
```
# 包体积优化	
## APK 结构
```
APK 本质是一个 ZIP 文件，解压后：

├── AndroidManifest.xml    （二进制XML）
├── classes.dex            （Java/Kotlin 字节码）
├── classes2.dex           （MultiDex）
├── classes3.dex           ...
├── res/                   （编译后的资源）
│   ├── layout/
│   ├── drawable/
│   ├── drawable-xxhdpi/
│   └── ...
├── resources.arsc         （资源索引表）
├── assets/                （原始资源，不编译）
├── lib/                   （Native so 库）
│   ├── arm64-v8a/
│   ├── armeabi-v7a/
│   └── x86/
└── META-INF/              （签名信息）

典型 App 各部分占比：
┌──────────────┬────────────┬─────────────────────┐
│ 组成部分     │ 典型占比   │ 优化空间             │
├──────────────┼────────────┼─────────────────────┤
│ so 库        │ 30%~50%    │ 大，ABI过滤/压缩     │
│ dex          │ 20%~35%    │ 中，混淆/裁剪        │
│ 资源文件     │ 15%~30%    │ 大，压缩/去重        │
│ resources.arsc│ 3%~8%    │ 中，去除无用配置     │
│ assets       │ 5%~15%    │ 中，压缩             │
└──────────────┴────────────┴─────────────────────┘

// bash
# 分析 APK 中各部分大小
# Android Studio → Build → Analyze APK

# 命令行分析
# apkanalyzer 工具（NDK 自带）

# 查看 dex 中各包的方法数
apkanalyzer dex packages app-release.apk

# 查看具体类的大小
apkanalyzer dex code --class com.xxx.MainActivity app-release.apk

# 查看资源大小
apkanalyzer resources configs app-release.apk

# 对比两个 APK 的差异
apkanalyzer apk compare old.apk new.apk
```
## Dex 优化
### ProGuard/R8 混淆压缩
1.R8：  
```
R8 是 Google 推出的新一代编译器
整合了 ProGuard 的所有功能，并做了更多优化

R8 vs ProGuard：
┌──────────────────┬──────────────┬──────────────────┐
│ 功能             │ ProGuard     │ R8               │
├──────────────────┼──────────────┼──────────────────┤
│ 代码压缩         │ ✅           │ ✅（更激进）      │
│ 代码混淆         │ ✅           │ ✅               │
│ 代码优化         │ 有限         │ ✅（更强）        │
│ Dex 生成         │ 需要 D8      │ 直接生成          │
│ Kotlin 支持      │ 一般         │ 原生支持          │
│ 编译速度         │ 慢           │ 快               │
│ 包体积减少       │ 基准         │ 比ProGuard再小10%│
└──────────────────┴──────────────┴──────────────────┘

ProGuard：源码 → javac → .class → ProGuard(混淆/裁剪) → dex工具 → .dex  
R8：源码 → kotlinc/javac → .class → R8(混淆/裁剪/优化/dex，一步完成，速度更快) → .dex

R8 的额外优化：
├── 内联（Inlining）：小方法直接内联，消除方法调用开销
├── 类合并（Class Merging）：合并只有一个实现的接口
├── 枚举优化：将简单枚举转换为 int 常量
├── Kotlin 特性优化：
│   ├── 移除 data class 未使用的 component 方法
│   ├── 优化 companion object
│   └── 内联 lambda
└── 常量折叠：编译期计算常量表达式
```

2.开启：  
```
// build.gradle 开启 R8 完整模式
android {
    buildTypes {
        release {
            minifyEnabled true      // 开启 R8 压缩混淆
            shrinkResources true    // 开启资源压缩
            proguardFiles getDefaultProguardFile(
                'proguard-android-optimize.txt' // 使用优化版规则
            ), 'proguard-rules.pro'
        }
    }

    // 开启 R8 完整模式（更激进的优化）
    // gradle.properties 中添加：
    // android.enableR8.fullMode=true
}
```

3.ProGuard 规则细化：  
```
# proguard-rules.pro

# ================================
# 基础保留规则
# ================================

# 保留注解（反射需要）
-keepattributes *Annotation*(注释)
-keepattributes Signature(泛型信息，序列化需要) 
-keepattributes SourceFile,LineNumberTable  # 保留行号（崩溃定位）

# 保留Framework 组件
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider

# 保留Native方法（JNI调用）
-keepclasseswithmembernames class * {native <methods>;}

# 保留枚举（Enum有特殊方法）
-keepclassmembers enum * {public static **[] values(); public static ** valueOf(java.lang.String);}

# 保留序列化 Parcelable
-keepclassmembers class * implements android.os.Parcelable {public static final android.os.Parcelable$Creator CREATOR;}

# ================================
# 反射相关保留
# 运行时通过反射访问的类，R8看不懂字符串，会认为不可达而删除
# ================================

# 保留通过反射访问的类/方法/字段
-keepclassmembers class com.xxx.model.** {
    <fields>;
}

# 保留数据模型(JSON 序列化的模型类)
-keep class com.xxx.model.** { *; }
-keep class com.xxx.network.response.** { *; }

# ================================
# 第三方库规则
# ================================

# Retrofit
-keepattributes Exceptions
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

# Gson
-keep class com.google.gson.** { *; }
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# ================================
# 精细化：只保留必要的内容
# ================================

# 不保留 BuildConfig（减少 dex 大小）
-dontwarn com.xxx.BuildConfig

# 移除日志（release 包不需要）
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}

# 移除 Kotlin 的空检查（性能优化，有风险）
# -assumenosideeffects class kotlin.jvm.internal.Intrinsics {
#     static void checkParameterIsNotNull(...);
# }
```
#### 问题排查
```
1. 生成 mapping.txt（混淆映射表）
build/outputs/mapping/release/mapping.txt

2. 还原崩溃堆栈
混淆后的堆栈：
at a.a.a(Unknown Source:3)
使用 retrace 还原：
java -jar retrace.jar mapping.txt stacktrace.txt

3. 查看哪些代码被删除
build.gradle 添加：
android {
    buildTypes {
        release {
            // 生成使用情况报告
        }
    }
}
查看 build/outputs/mapping/release/usage.txt
里面列出了所有被删除的类和方法

4. 查看种子文件（被保留的类）
build/outputs/mapping/release/seeds.txt
```
#### Full Mode
```
// gradle.properties 开启 R8 Full Mode
android.enableR8.fullMode=true

// 注意：Full Mode 可能导致更多兼容性问题
// 需要充分测试
```

与普通的区别：
```
普通 R8：
├── 保守地处理接口默认方法
├── 不合并某些边界情况的类
└── 保留更多"可能被反射访问"的代码

Full Mode 额外优化：
├── 更激进的接口内联
├── 接口默认方法内联
├── 更多类合并
├── 包体积额外减少 5%~10%
├── 删除更多"看起来没用"的代码
└── 对反射的假设更激进（可能误删反射访问的代码）
```
##### 踩坑
Gson 反序列化失败：
```
// 场景：用 Gson 解析服务端 JSON

data class UserResponse(
    @SerializedName("user_id")
    val userId: String,
    @SerializedName("user_name")
    val userName: String,
    val age: Int
)

// 问题：Full Mode 下，UserResponse 的字段可能被删除
// 因为 R8 看不到这些字段被"直接"访问
// Gson 通过反射访问字段，R8 静态分析不到

// 症状：
val user = gson.fromJson(json, UserResponse::class.java)
println(user.userId)   // null！字段被删除了
println(user.userName) // null！
println(user.age)      // 0！

// 解决方案一：keep 规则
-keepclassmembers class com.xxx.model.** {
    <fields>;
}

// 解决方案二：@Keep 注解（更精准）
@Keep
data class UserResponse(
    @SerializedName("user_id")
    val userId: String,
    ...
)

// 解决方案三：换用 Moshi + Kotlin codegen（推荐）
// Moshi 的 @JsonClass(generateAdapter = true)
// 编译期生成适配器，不依赖反射
// R8 可以正确分析，不需要 keep 规则
@JsonClass(generateAdapter = true)
data class UserResponse(
    @Json(name = "user_id") val userId: String,
    @Json(name = "user_name") val userName: String,
    val age: Int
)
```

接口默认方法被错误内联:
```
// 场景：接口有默认方法实现

interface Clickable {
    fun click()
    fun showOff() = println("I'm clickable!") // 默认实现
}

interface Focusable {
    fun setFocus(b: Boolean) = println("I ${if (b) "got" else "lost"} focus.")
    fun showOff() = println("I'm focusable!")
}

class Button : Clickable, Focusable {
    override fun click() = println("I was clicked")
    // 没有 override showOff()，存在歧义！
}

// Full Mode 可能错误处理这种多接口默认方法冲突
// 症状：运行时 AbstractMethodError 或调用了错误的实现

// 解决方案：明确 override
class Button : Clickable, Focusable {
    override fun click() = println("I was clicked")
    override fun showOff() {
        super<Clickable>.showOff() // 明确指定
    }
}
```

枚举被错误优化:
```
// 场景：通过字符串获取枚举值

enum class Direction { NORTH, SOUTH, EAST, WEST }

// 代码中使用
val dir = Direction.valueOf("NORTH") // 反射！

// Full Mode 可能删除枚举的 values() 或 valueOf() 方法
// 症状：NoSuchMethodError

// 解决方案：keep 枚举
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

// 或者避免使用 valueOf（用 when 替代）
fun parseDirection(s: String): Direction = when(s) {
    "NORTH" -> Direction.NORTH
    "SOUTH" -> Direction.SOUTH
    else -> Direction.NORTH
}
// 这样 R8 可以静态分析，不需要 keep
```

Retrofit 接口方法被删除:
```
// 场景：Retrofit 通过动态代理调用接口方法

interface ApiService {
    @GET("/users/{id}")
    suspend fun getUser(@Path("id") id: String): User

    @POST("/login")
    suspend fun login(@Body request: LoginRequest): LoginResponse
}

// Full Mode 可能认为这些接口方法没被直接调用
// （实际是通过动态代理调用的）→ 删除方法！
// 症状：运行时找不到方法，请求失败

// 解决方案
-keep,allowobfuscation interface com.xxx.api.ApiService
-keepclassmembers,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

// 更好的方案：Retrofit 2.9+ 支持 R8 规则自动配置
// 在 retrofit 的 jar 中已包含 consumer-rules.pro
// 确保使用最新版本的 Retrofit
```

ServiceLoader 失效:
```
// 场景：使用 Java SPI 机制（ServiceLoader）
// META-INF/services/com.xxx.Plugin 文件中注册实现类

// Full Mode 可能删除这些实现类（认为没被引用）
// 症状：ServiceLoader.load() 返回空

// 解决方案
-keep class * implements com.xxx.Plugin { *; }

// 或者用 @AutoService 注解 + keep 规则
```
#### 无用代码裁剪（Tree Shaking）
1.原理：  
```
本质是可达性分析，和 JVM GC 的标记算法思路相同，从"根节点"出发，遍历所有可达代码：  
R8 的根节点（不会被删除的起点）：

① AndroidManifest.xml 中声明的组件  
<activity android:name=".MainActivity"/>  
<service android:name=".MyService"/>  
└── 这些类及其生命周期方法是根节点

② -keep 规则声明的类/方法  
-keep class com.xxx.MainActivity { *; }

③ 反射可能访问的类（R8无法静态分析反射）  
Class.forName("com.xxx.SomeClass") → R8 看不懂  
→ 需要手动 -keep，否则可能被误删！

④ 注解处理器生成的代码入口

⑤ JNI 调用的 Java 方法  
JNIEXPORT void Java_com_xxx_NativeLib_init  
→ 对应的 Java native 方法是根节点
```

标记：
白色 = 未访问（初始状态，可能被删除）  
灰色 = 已发现但未完全处理  
黑色 = 已完全处理（确定保留）

① 所有根节点标记为灰色，加入工作队列  

② 取出灰色节点，分析它引用的所有代码：  
   ├── 调用的方法 → 标记为灰色  
   ├── 访问的字段 → 标记为灰色  
   ├── 继承的父类 → 标记为灰色  
   ├── 实现的接口 → 标记为灰色  
   └── 注解 → 标记为灰色  

③ 当前节点标记为黑色（处理完毕）  

④ 重复②③直到工作队列为空   

⑤ 所有仍为白色的节点 → 删除  

// 原始代码
class Utils {
    fun usedMethod() { println("被使用") }
    fun unusedMethod() { println("没人调用我") } // 会被删除
}

class DeadClass { // 整个类没被引用，会被删除
    fun method() {}
}

// R8 分析调用链：
// 入口点（Activity/Application）→ 追踪所有可达代码
// 不可达的代码 → 直接删除

// 配置入口点（keep规则）
// proguard-rules.pro
-keep class com.xxx.MainActivity { *; }
-keep class * extends android.app.Application { *; }
```
```

2.开启： 
```
// 使用 lint 检测无用代码
// ./gradlew lint

// build.gradle 配置
android {
    lintOptions {
        // 检测未使用的资源
        check 'UnusedResources'
        // 检测未使用的代码
        check 'UnusedIds'
        // 将警告变为错误（强制修复）
        warningsAsErrors false
        // 生成报告
        htmlReport true
        htmlOutput file("$buildDir/reports/lint-results.html")
    }
}
```

3.优化：  
```
// ================================
// 优化一：避免不必要的 companion object
// ================================

// ❌ companion object 会生成额外的类
class BadClass {
    companion object {
        const val TAG = "BadClass"
        fun create(): BadClass = BadClass()
    }
}
// 生成：BadClass + BadClass$Companion（两个类！）

// ✅ 使用顶层函数/属性
const val TAG = "GoodClass"  // 顶层常量，直接内联
fun createGoodClass() = GoodClass()  // 顶层函数

// ================================
// 优化二：避免 Kotlin 扩展函数滥用
// ================================

// 每个扩展函数都会生成一个静态方法
// 大量扩展函数 = dex 方法数增加

// ❌ 为每个类型都写扩展函数
fun String.isValidEmail(): Boolean = contains("@")
fun String.isValidPhone(): Boolean = matches(Regex("\\d{11}"))
fun String.isValidIdCard(): Boolean = length == 18
// 生成 3 个静态方法

// ✅ 合并到工具类（如果不需要链式调用）
object StringValidator {
    fun isValidEmail(s: String) = s.contains("@")
    fun isValidPhone(s: String) = s.matches(Regex("\\d{11}"))
}

// ================================
// 优化三：data class 精简
// ================================

// data class 自动生成：equals/hashCode/toString/copy/componentN
// 如果不需要这些方法，考虑用普通 class

// ❌ 不需要 copy/component 但用了 data class
data class Config(val debug: Boolean, val version: String)
// 生成了 copy/component1/component2 等方法（浪费）

// ✅ 只需要 equals/hashCode 时
class Config(val debug: Boolean, val version: String) {
    override fun equals(other: Any?): Boolean { ... }
    override fun hashCode(): Int { ... }
}
```
#### 缩包(Minification)
1.混淆：  
```
// 混淆前
package com.example.payment

class PaymentProcessor {
    private var userBalance: Double = 0.0

    fun processPayment(amount: Double): Boolean {
        return userBalance >= amount
    }
}

// 混淆后（R8生成）
package com.example.payment

class a {           // PaymentProcessor → a
    private var a: Double = 0.0  // userBalance → a

    fun a(a: Double): Boolean {  // processPayment → a
        return this.a >= a
    }
}
```

类名/方法名/字段名 → 短名称：  
类名按字母顺序的最短名称，如：  
第1个类 → a  
第2个类 → b  
...  
第26个类 → z  
第27个类 → aa  
第28个类 → ab  
...  
方法名、字段名同理，但在各自的命名空间内独立计数

包名压缩：
```
// 混淆前
com.example.app.feature.payment.PaymentProcessor
com.example.app.feature.payment.CardValidator
com.example.app.feature.user.UserManager
com.example.app.feature.user.ProfileHelper

// 混淆后（包名压缩）
a.a.a.a  // PaymentProcessor
a.a.a.b  // CardValidator
a.a.b.a  // UserManager
a.a.b.b  // ProfileHelper

// 或者更激进（repackageclasses）
// 所有类放到同一个包下
-repackageclasses 'x'
// 结果：
x.a  // PaymentProcessor
x.b  // CardValidator
x.c  // UserManager
x.d  // ProfileHelper
```

字符串不混淆:
```
// 默认情况：字符串不混淆
class ApiConfig {
    val baseUrl = "https://api.example.com"        // 明文！
    val secretKey = "sk_live_abcd1234"             // 危险！
    val sqlQuery = "SELECT * FROM users"           // 暴露表结构！
}

// 反编译后完全可见：
// const-string v0, "https://api.example.com"
// const-string v1, "sk_live_abcd1234"

// 解决方案一：字符串加密（R8 StringEncryption，需要付费版）

// 解决方案二：手动加密敏感字符串
object SecureConfig {
    // 运行时解密，不在字节码中明文存储
    val secretKey: String
        get() = decrypt("加密后的密文")

    private fun decrypt(cipher: String): String {
        // 解密逻辑
        return XORDecrypt(cipher, BuildConfig.DECRYPT_KEY)
    }
}

// 解决方案三：NDK存储（放到so中）
// 敏感字符串在C++层，更难逆向

// 解决方案四：服务端下发
// 不在客户端存储敏感信息

减少包体积:  
类名从平均20字符 → 1~2字符,包路径从30字符 → 1字符  
删除无用代码  
内联（减少方法数）  
减少 Dex 文件头开销
```

2.内联：  
```
R8 决定是否内联的判断条件：

✅ 可以内联：
├── 方法体足够小（字节码指令数 < 阈值，约5~10条）
├── 方法只被调用一次
├── 非虚方法（private/final/static）
└── 内联后不会导致类循环依赖

❌ 不能内联：
├── 方法体太大（内联会导致调用方膨胀）
├── 递归方法
├── 虚方法（可能有多个实现）
├── 被 -keep 保留的方法
└── 包含异常处理的复杂方法

// 优化前
fun add(a: Int, b: Int) = a + b
fun calculate() = add(1, 2) // 函数调用有开销

// ❌ 普通高阶函数：lambda 会生成匿名类
fun <T> measure(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时: ${System.currentTimeMillis() - start}ms")
    return result
}
// 每次调用都创建一个匿名类对象

// ✅ inline 函数：lambda 直接内联，无匿名类
inline fun <T> measureInline(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时: ${System.currentTimeMillis() - start}ms")
    return result
}
// 编译后：lambda 代码直接插入调用处，无额外类

// 优化后（R8内联）
fun calculate() = 3 // 直接计算结果，消除函数调用
```

3.常量折叠:  
```
val MAX_SIZE = 100
val DOUBLE_SIZE = MAX_SIZE * 2 // R8直接替换为 200

4.无效代码消除:    
// 无效代码消除
fun example(debug: Boolean = false) {
    if (debug) {
        // BuildConfig.DEBUG = false 时
        // R8 直接删除整个if块
        Log.d("TAG", "debug info")
    }
}
```

4.类合并（Class Merging）:  
```
// 只有一个子类的抽象类 → 合并为一个类
// 减少类数量，降低方法数
// 场景：只有一个实现的抽象类/接口

// 合并前
abstract class BaseRepository {
    abstract fun fetchData(): String
    fun logFetch() { println("fetching...") }
}

class UserRepository : BaseRepository() {
    // UserRepository 是 BaseRepository 的唯一子类
    override fun fetchData(): String = "user data"
}

// 使用方
val repo: BaseRepository = UserRepository()
repo.fetchData()

// R8 类合并后：
// BaseRepository 被合并进 UserRepository
// 消除了一层继承
class UserRepository {  // 不再继承 BaseRepository
    fun fetchData(): String = "user data"
    fun logFetch() { println("fetching...") }  // 直接合并进来
}

// 效果：
// ├── 减少一个类（减少方法数）
// ├── 消除虚方法调用（变为直接调用）
// └── 为进一步内联创造条件
```

5.参数移除:  
```
// 未使用的方法参数 → 直接删除
fun process(data: String, unused: Int) { // unused被删除
    println(data)
}
```
#### Dex 分包优化
```
生成Dex文件
.class 文件（JVM字节码）和 .dex 文件的区别：

.class 文件：
├── 每个类一个文件
├── 常量池：每个类独立
├── 指令集：JVM字节码（基于栈）
└── 设计目标：JVM运行

.dex 文件：
├── 多个类合并到一个文件
├── 常量池：所有类共享（去重！）
├── 指令集：Dalvik字节码（基于寄存器）
└── 设计目标：移动设备，内存紧张

R8 Dexing 步骤：
① 字节码转换：JVM字节码 → Dalvik字节码
   
   // JVM字节码（基于栈）：
   ILOAD_1    // 将局部变量1压栈
   ILOAD_2    // 将局部变量2压栈
   IADD       // 弹出两个值，相加，结果压栈
   IRETURN    // 弹出栈顶，返回
   
   // Dalvik字节码（基于寄存器）：
   add-int v0, v1, v2  // v0 = v1 + v2，直接操作寄存器
   return v0

② 共享常量池构建
   // 多个类中相同的字符串/类型引用 → 合并为一份
   // 例：100个类都引用 "java/lang/String"
   // .class：100个常量池各存一份
   // .dex：共享常量池只存一份 → 大幅减小体积

③ 方法数检查与分包
   // 单个 Dex 文件方法数上限：65536（0xFFFF）
   // 超过则分为 classes.dex + classes2.dex + ...
   
   // R8 的分包策略：
   // 将启动相关类优先放入 classes.dex（主Dex）
   // 其余类放入 secondary Dex

④ 优化 Dex 布局
   // 按类的调用关系排列，提高局部性
   // 相关联的类放在相邻位置
   // 减少运行时的缺页中断
```
## 资源优化
### 图片优化
```
图片格式对比：
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ 格式     │ 有损压缩 │ 无损压缩 │ 透明度   │ 支持版本 │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ PNG      │ ❌       │ ✅       │ ✅       │ 全部     │
│ JPEG     │ ✅       │ ❌       │ ❌       │ 全部     │
│ WebP     │ ✅       │ ✅       │ ✅       │ API 14+  │
│ AVIF     │ ✅       │ ✅       │ ✅       │ API 31+  │
│ SVG      │ -        │ -        │ ✅       │ 全部     │
└──────────┴──────────┴──────────┴──────────┴──────────┘

WebP vs PNG/JPEG：
├── 有损 WebP：比 JPEG 小 25%~34%
├── 无损 WebP：比 PNG 小 26%
└── 支持透明度（PNG 的优势 WebP 也有）

推荐策略：
├── 照片类图片：有损 WebP（质量 85）
├── 图标/UI 图片（有透明度）：无损 WebP
├── 简单图形：VectorDrawable（SVG）
└── 动画：Lottie（JSON 格式，比 GIF 小很多）
```

转换脚本：  
```
# 批量转换 PNG 到 WebP
# 使用 cwebp 工具（Google 官方）

# 单个文件转换
cwebp -q 85 input.png -o output.webp

# 批量转换脚本
find ./res -name "*.png" | while read file; do
    output="${file%.png}.webp"
    cwebp -q 85 "$file" -o "$output"
    # 如果 webp 更小，删除原 png
    if [ $(stat -f%z "$output") -lt $(stat -f%z "$file") ]; then
        rm "$file"
        echo "转换成功: $file → $output"
    else
        rm "$output"
        echo "跳过（webp更大）: $file"
    fi
done

# Android Studio 内置转换
# 右键图片 → Convert to WebP
```

图片压缩：  
```
// 图片压缩 Gradle 插件（构建时自动压缩）
// 使用 pngquant/optipng 压缩 PNG

class ImageCompressPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.tasks.register("compressImages") { task ->
            task.doLast {
                val resDir = File(project.projectDir, "src/main/res")
                compressAllImages(resDir)
            }
        }

        // 在 preBuild 之前执行
        project.tasks.named("preBuild").configure {
            it.dependsOn("compressImages")
        }
    }

    private fun compressAllImages(resDir: File) {
        resDir.walkTopDown()
            .filter { it.extension in listOf("png", "jpg", "jpeg") }
            .forEach { imageFile ->
                val originalSize = imageFile.length()
                compressImage(imageFile)
                val newSize = imageFile.length()
                val saved = originalSize - newSize
                if (saved > 0) {
                    println("压缩: ${imageFile.name} " +
                        "${originalSize/1024}KB → ${newSize/1024}KB " +
                        "(节省 ${saved/1024}KB)")
                }
            }
    }

    private fun compressImage(file: File) {
        when (file.extension.lowercase()) {
            "png" -> {
                // 使用 pngquant 有损压缩
                ProcessBuilder("pngquant", "--force", "--quality=65-80",
                    "--output", file.absolutePath, file.absolutePath)
                    .start().waitFor()
            }
            "jpg", "jpeg" -> {
                // 使用 mozjpeg 压缩
                ProcessBuilder("jpegoptim", "--max=85", file.absolutePath)
                    .start().waitFor()
            }
        }
    }
}
```
### 无用资源删除（shrinkResources）
```
// build.gradle 资源裁剪配置
android {
    buildTypes {
        release {
            shrinkResources true  // 删除未使用的资源
            minifyEnabled true    // 必须同时开启代码压缩
        }
    }

    // 只保留中文和英文（删除其他语言资源）
    // 减少 strings.xml 的语言版本
    defaultConfig {
        resConfigs "zh", "zh-rCN", "en"
        // 删除其他语言：fr/de/ja/ko 等
        // 对于只做中文市场的 App，可以只保留 zh
    }

    // 只保留需要的屏幕密度资源
    // 现代手机基本都是 xxhdpi 或 xxxhdpi
    defaultConfig {
        resConfigs "xxhdpi", "xxxhdpi"
        // 删除 ldpi/mdpi/hdpi/xhdpi 的图片资源
        // 系统会自动缩放 xxhdpi 的图片
    }
}
```

自定义：  
```
// 自定义资源裁剪：删除重复资源
// 不同分辨率目录下内容完全相同的图片

object DuplicateResourceFinder {

    fun findDuplicates(resDir: File): Map<String, List<File>> {
        val md5ToFiles = mutableMapOf<String, MutableList<File>>()

        resDir.walkTopDown()
            .filter { it.isFile && it.extension in
                listOf("png", "jpg", "webp", "xml") }
            .forEach { file ->
                val md5 = file.readBytes().md5()
                md5ToFiles.getOrPut(md5) { mutableListOf() }.add(file)
            }

        // 返回有重复的文件组
        return md5ToFiles.filter { it.value.size > 1 }
    }

    fun removeDuplicates(duplicates: Map<String, List<File>>) {
        duplicates.forEach { (_, files) ->
            // 保留分辨率最高的版本，删除其他
            val sorted = files.sortedByDescending { file ->
                // 按目录分辨率排序：xxxhdpi > xxhdpi > xhdpi
                when {
                    file.path.contains("xxxhdpi") -> 4
                    file.path.contains("xxhdpi") -> 3
                    file.path.contains("xhdpi") -> 2
                    file.path.contains("-hdpi") -> 1
                    else -> 0
                }
            }
            // 删除除最高分辨率外的重复文件
            sorted.drop(1).forEach { it.delete() }
        }
    }
}
```

resources.arsc:  
```
resources.arsc：资源索引表
包含所有资源的 ID → 文件路径 映射

优化方向：
├── 删除未使用的配置（语言/密度）
│   → 通过 resConfigs 配置
├── 资源混淆（AndResGuard）
│   → 路径变短，arsc 文件变小
└── 删除重复的字符串
    → 相同的字符串只存储一次

resources.arsc 不能压缩！
Android 7.0+ 要求 resources.arsc 不压缩
（需要内存映射直接访问）
```
### 资源混淆（AndResGuard）
### 重复资源去重
### 语言/分辨率裁剪
## so 库优化
### ABI 过滤（只保留 arm64-v8a）
```
Android 支持的 ABI：
├── arm64-v8a：64位 ARM（主流，覆盖95%+设备）
├── armeabi-v7a：32位 ARM（老设备）
├── x86_64：64位 x86（模拟器）
└── x86：32位 x86（模拟器）

每个 ABI 都有一套 so 文件
保留所有 ABI = so 大小 × 4

优化策略：
只保留 arm64-v8a
├── 覆盖市面上 95%+ 的真实设备
├── so 大小减少约 75%（4套→1套）
└── 64位 ARM 设备可以运行 32位 so（向后兼容）
   但反过来不行（32位设备不能运行64位so）
```

启用：  
```
// build.gradle ABI 过滤
android {
    defaultConfig {
        ndk {
            // 只保留 arm64-v8a
            abiFilters "arm64-v8a"

            // 如果需要兼容老设备（32位ARM）
            // abiFilters "arm64-v8a", "armeabi-v7a"
        }
    }

    // 按 ABI 分包（不同设备下载对应版本）
    // 配合 AAB 使用效果更好
    splits {
        abi {
            enable true
            reset()
            include "arm64-v8a", "armeabi-v7a"
            universalApk false  // 不生成通用包
        }
    }
}
```
### 动态下发
```
// 大型 so 文件（如 AI 模型推理库）动态下发
// 首次使用时才下载，减少安装包大小

object SoDynamicLoader {

    private const val SO_DOWNLOAD_URL = "https://cdn.xxx.com/so/"
    private const val SO_DIR = "dynamic_so"

    // 检查并加载 so
    suspend fun loadSo(
        context: Context,
        soName: String,
        version: String
    ): Boolean {
        val soFile = getSoFile(context, soName, version)

        return if (soFile.exists()) {
            // 本地已有，直接加载
            loadLocalSo(soFile)
        } else {
            // 下载后加载
            downloadAndLoad(context, soName, version)
        }
    }

    private fun getSoFile(
        context: Context,
        soName: String,
        version: String
    ): File {
        return File(
            context.getDir(SO_DIR, Context.MODE_PRIVATE),
            "${soName}_${version}.so"
        )
    }

    private fun loadLocalSo(soFile: File): Boolean {
        return try {
            System.load(soFile.absolutePath)
            true
        } catch (e: UnsatisfiedLinkError) {
            soFile.delete() // 文件损坏，删除重新下载
            false
        }
    }

    private suspend fun downloadAndLoad(
        context: Context,
        soName: String,
        version: String
    ): Boolean = withContext(Dispatchers.IO) {
        val soFile = getSoFile(context, soName, version)
        val url = "$SO_DOWNLOAD_URL${soName}_${version}_arm64.so"

        try {
            // 下载 so 文件
            downloadFile(url, soFile)

            // 验证完整性（MD5 校验）
            if (!verifyMd5(soFile, getExpectedMd5(soName, version))) {
                soFile.delete()
                return@withContext false
            }

            // 加载
            loadLocalSo(soFile)
        } catch (e: Exception) {
            soFile.delete()
            false
        }
    }
}

// 使用示例：AI 推理功能按需加载
class AIFeature {

    private var isLoaded = false

    suspend fun initialize(context: Context): Boolean {
        if (isLoaded) return true

        // 动态加载 AI 推理 so（可能 20MB+）
        isLoaded = SoDynamicLoader.loadSo(
            context,
            soName = "libai_inference",
            version = "1.2.0"
        )
        return isLoaded
    }

    fun predict(input: FloatArray): FloatArray {
        check(isLoaded) { "AI so 未加载" }
        return nativePredict(input)
    }

    private external fun nativePredict(input: FloatArray): FloatArray
}
```
### 符号裁剪（strip）
```
# so 文件包含两种符号表：
# .symtab：完整符号表（调试用）
# .dynsym：动态符号表（运行时需要）

# strip 命令：删除 .symtab，只保留 .dynsym
# NDK 构建时默认会 strip release 版本的 so

# 手动 strip
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/\
llvm-strip --strip-unneeded libnative.so

# 查看 so 大小变化
ls -la libnative.so          # strip 前
llvm-strip libnative.so
ls -la libnative.so          # strip 后（通常减少 30%~60%）

# CMakeLists.txt 配置（确保 release 时 strip）
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_link_options(-Wl,--strip-all)
endif()
```

cmake:  
```
# CMakeLists.txt 完整优化配置
cmake_minimum_required(VERSION 3.18.1)
project("myapp")

# 编译优化选项
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} \
    -O2 \                    # 优化级别
    -fvisibility=hidden \    # 默认隐藏符号（只导出需要的）
    -ffunction-sections \    # 每个函数独立段（配合--gc-sections）
    -fdata-sections"         # 每个数据独立段
)

# 链接优化选项
set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} \
    -Wl,--gc-sections \      # 删除未使用的段（Dead Code Elimination）
    -Wl,--strip-all \        # 删除所有符号（release包）
    -Wl,--icf=all"           # 相同函数合并（Identical Code Folding）
)

add_library(mylib SHARED native.cpp)
```
### 合并 so
## 架构优化
### 插件化（功能动态下发）
### AAB（Android App Bundle）
```
AAB（Android App Bundle）：
Google Play 的新发布格式

传统 APK 发布：
开发者 → 上传一个通用 APK → 所有用户下载同一个 APK
问题：APK 包含所有语言/密度/ABI 的资源，用户只用其中一部分

AAB 发布：
开发者 → 上传 AAB → Google Play 按需生成 APK
├── 用户设备：arm64-v8a + zh + xxhdpi
│   → 只下载 arm64 so + 中文资源 + xxhdpi 图片
└── 用户设备：armeabi-v7a + en + xhdpi
    → 只下载 armeabi so + 英文资源 + xhdpi 图片

效果：
├── 用户下载的 APK 减少 15%~65%
├── 开发者不需要手动管理多个 APK
└── 支持 Dynamic Feature（功能模块按需下载）

AAB 文件结构：
├── base/                  （基础模块）
│   ├── dex/
│   ├── res/
│   ├── assets/
│   ├── lib/
│   └── manifest/
├── feature_payment/       （支付功能模块，按需下载）
│   └── ...
└── feature_camera/        （相机功能模块，按需下载）
    └── ...
```
### 按需下载
```
// 功能模块按需下载（Dynamic Feature）

// 1. 在 build.gradle 中声明 Dynamic Feature
// feature_camera/build.gradle
plugins {
    id 'com.android.dynamic-feature'
}
android {
    // ...
}
dependencies {
    implementation project(':app') // 依赖基础模块
}

// app/build.gradle
android {
    dynamicFeatures = [':feature_camera', ':feature_payment']
}

// 2. 运行时按需下载
class MainActivity : AppCompatActivity() {

    private val splitInstallManager by lazy {
        SplitInstallManagerFactory.create(this)
    }

    fun openCamera() {
        // 检查模块是否已安装
        if (splitInstallManager.installedModules.contains("feature_camera")) {
            startCameraFeature()
            return
        }

        // 请求下载
        val request = SplitInstallRequest.newBuilder()
            .addModule("feature_camera")
            .build()

        splitInstallManager.startInstall(request)
            .addOnSuccessListener { sessionId ->
                // 下载开始
            }
            .addOnFailureListener { exception ->
                // 下载失败
                handleInstallFailure(exception)
            }
    }

    // 监听安装状态
    private val installStateListener = SplitInstallStateUpdatedListener { state ->
        when (state.status()) {
            SplitInstallSessionStatus.DOWNLOADING -> {
                val progress = state.bytesDownloaded() * 100 / state.totalBytesToDownload()
                showProgress(progress.toInt())
            }
            SplitInstallSessionStatus.INSTALLED -> {
                // 安装完成，可以使用功能
                startCameraFeature()
            }
            SplitInstallSessionStatus.FAILED -> {
                showError("安装失败: ${state.errorCode()}")
            }
        }
    }

    override fun onResume() {
        super.onResume()
        splitInstallManager.registerListener(installStateListener)
    }

    override fun onPause() {
        super.onPause()
        splitInstallManager.unregisterListener(installStateListener)
    }
}
```
## 编译优化
### D8
```
// gradle.properties
# 开启 D8 增量编译
android.enableD8.desugaring=true

# 开启 R8 完整模式
android.enableR8.fullMode=true

# 开启 dex 合并优化
android.enableDexingArtifactTransform=true
```
### Baseline Profile
### 压缩算法选择
```
// APK 中不同文件的压缩策略

android {
    packagingOptions {
        // so 文件不压缩（Android 6.0+ 支持直接内存映射执行）
        // 不压缩 so 虽然 APK 变大，但安装后占用空间不变
        // 且加载速度更快（无需解压）
        jniLibs.useLegacyPackaging = false  // false = 不压缩 so

        // 排除重复文件
        excludes += [
            'META-INF/LICENSE',
            'META-INF/LICENSE.txt',
            'META-INF/NOTICE',
            'META-INF/NOTICE.txt',
            '**/*.kotlin_module'
        ]
    }
}
```
## 监控
```
// 包体积监控：防止包体积悄悄增大

// build.gradle 添加包体积检查 task
tasks.register("checkApkSize") {
    doLast {
        val apkFile = fileTree("$buildDir/outputs/apk/release")
            .matching { include("**/*.apk") }
            .singleFile

        val apkSizeMB = apkFile.length() / 1024.0 / 1024.0

        println("APK 大小: ${"%.2f".format(apkSizeMB)} MB")

        // 超过阈值则构建失败
        val maxSizeMB = 50.0
        if (apkSizeMB > maxSizeMB) {
            throw GradleException(
                "APK 大小 (${"%.2f".format(apkSizeMB)}MB) " +
                "超过限制 (${maxSizeMB}MB)！"
            )
        }
    }
}

// 在 assembleRelease 后自动检查
tasks.named("assembleRelease").configure {
    finalizedBy("checkApkSize")
}

// Python
# 包体积分析脚本：对比两个版本的差异
# compare_apk.py

import zipfile
import sys
from pathlib import Path

def analyze_apk(apk_path: str) -> dict:
    """分析 APK 各部分大小"""
    sizes = {
        'dex': 0, 'so': 0, 'res': 0,
        'assets': 0, 'arsc': 0, 'other': 0
    }

    with zipfile.ZipFile(apk_path) as apk:
        for info in apk.infolist():
            size = info.file_size
            name = info.filename

            if name.endswith('.dex'):
                sizes['dex'] += size
            elif name.startswith('lib/') and name.endswith('.so'):
                sizes['so'] += size
            elif name.startswith('res/'):
                sizes['res'] += size
            elif name.startswith('assets/'):
                sizes['assets'] += size
            elif name == 'resources.arsc':
                sizes['arsc'] += size
            else:
                sizes['other'] += size

    return sizes

def compare_apks(old_apk: str, new_apk: str):
    """对比两个 APK 的差异"""
    old_sizes = analyze_apk(old_apk)
    new_sizes = analyze_apk(new_apk)

    print(f"{'组成部分':<15} {'旧版本':>10} {'新版本':>10} {'差异':>10}")
    print("-" * 50)

    total_old = total_new = 0
    for key in old_sizes:
        old = old_sizes[key]
        new = new_sizes.get(key, 0)
        diff = new - old
        total_old += old
        total_new += new

        diff_str = f"+{diff/1024:.1f}KB" if diff > 0 else f"{diff/1024:.1f}KB"
        print(f"{key:<15} {old/1024:>8.1f}KB {new/1024:>8.1f}KB {diff_str:>10}")

    print("-" * 50)
    total_diff = total_new - total_old
    diff_str = f"+{total_diff/1024:.1f}KB" if total_diff > 0 else f"{total_diff/1024:.1f}KB"
    print(f"{'总计':<15} {total_old/1024:>8.1f}KB {total_new/1024:>8.1f}KB {diff_str:>10}")

if __name__ == '__main__':
    compare_apks(sys.argv[1], sys.argv[2])
```
# 电量优化	
## WakeLock	
### 非必要 WakeLock 释放
### 超时保护
## 后台任务	
### JobScheduler/WorkManager 合并任务
### Doze 模式适配
## 定位优化	
### 按需请求定位
### 降低定位频率
### Geofencing 替代持续定位
## 传感器	
### 及时注销传感器监听
### 降低采样率
## 网络电量	
### 减少轮询
### Push 替代 Pull
### 批量网络请求
## 渲染电量	
### 降低帧率（非交互场景）
### 深色模式（OLED 省电）
## Battery Historian	
### 电量分析工具使用
### 耗电归因
# 存储优化	
## SharedPreferences	
sharedPreferences 是一个 xml 的读取和存储操作，在使用前都会调用 getSharedPreferences 方法，这时它会去异步加载文件当中的配置文件，load 到内存当中，调用 get 或 put 属性时，如果 load 内存的操作没有执行完成，那么就会一直阻塞进行等待，都是拿同一把锁，它既然是 IO 操作，如果这文件存在很久，这个时间就会很长,如果项目比较大，有几十个类使用 SharedPreferences 文件，里面的文件也非常多   
1.在 Application 中 MultiDex 之前加载 SharedPreferences（如果其他类在 Multidex 之前加载进行操作，会因为一些类不在主 dex 当中，导致崩溃，Sharedpreferences 是系统类，不会报错）   
2.创建 SharedPreferences 并且保存到 Map 中，那么需要的时候可以在 SP_MAP 中直接获取
### ANR 风险
### MMKV/DataStore 替代方案
## 数据库优化	
### SQLite 索引
### 事务批量操作
### Room 查询优化
### WAL 模式
## 文件 IO	
### 异步 IO
### BufferedStream
### 内存映射文件（MappedByteBuffer）
## 序列化优化	
### Protobuf/FlatBuffers 替代 Java 序列化
## 磁盘缓存	
### 缓存分级
### 缓存淘汰策略
### 缓存大小控制
# 并发优化
## 线程池管理	
### 统一线程池
### 避免线程无限创建
### 核心线程数配置
## 协程优化	
### Kotlin Coroutines 调度器选择
### 结构化并发、Flow 背压
## 线程泄漏	
### 未终止线程检测
### 线程生命周期管理
## 死锁检测	
### 锁顺序规范
### DeadlockDetector
### synchronized 优化
### 锁粒度控制
## 异步框架	
### RxJava 线程切换
### LiveData/Flow 线程安全
# 错误处理
## 错误分类
1.网络错误:    
无网络连接  
请求超时  
DNS 解析失败  

2.HTTP错误:  
401 未登录/Token 过期  
403 无权限  
404 资源不存在  
500 服务器错误  


3.业务错误:  
success: false  
服务器返回的业务异常  

4.本地错误:  
数据解析失败  
本地数据库异常
## 统一处理
统一错误模型：  
```
sealed class AppError {
    // 网络错误
    data object NoNetwork : AppError()
    data object Timeout : AppError()

    // HTTP 错误
    data object Unauthorized : AppError()   // 401
    data object Forbidden : AppError()      // 403
    data class HttpError(
        val code: Int,
        val message: String
    ) : AppError()

    // 业务错误
    data class BusinessError(
        val message: String
    ) : AppError()

    // 未知错误
    data class Unknown(
        val message: String
    ) : AppError()
}
```

1.OkHttp 拦截器捕获网络层错误  
```
class ErrorInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = try {
            chain.proceed(chain.request())
        } catch (e: Exception) {
            // 网络层异常转换
            throw when (e) {
                is UnknownHostException -> NoNetworkException()
                is SocketTimeoutException -> TimeoutException()
                else -> e
            }
        }

        // HTTP 错误码处理
        when (response.code) {
            401 -> throw UnauthorizedException()
            403 -> throw ForbiddenException()
            500 -> throw ServerException()
        }

        return response
    }
}
```

2.Repository 层统一转换为 AppError  
```
suspend fun getWatchlist(): Result<List<WatchlistItem>> {
    return runCatching {
        api.getWatchlist(bearerToken())
    }.mapError { e ->
        // 统一转换为 AppError
        when (e) {
            is NoNetworkException -> AppError.NoNetwork
            is TimeoutException -> AppError.Timeout
            is UnauthorizedException -> AppError.Unauthorized
            else -> AppError.Unknown(e.message ?: "未知错误")
        }
    }
}
```

3.ViewModel 层根据错误类型差异化处理  
```
fun loadWatchlist() {
    viewModelScope.launch {
        repository.getWatchlist()
            .onSuccess { items ->
                _uiState.update {
                    it.copy(items = items, isLoading = false)
                }
            }
            .onFailure { error ->
                when (error) {
                    is AppError.NoNetwork -> {
                        // 加载本地缓存
                        loadFromLocal()
                        _uiState.update {
                            it.copy(
                                errorMessage = "网络不可用，显示缓存数据",
                                isOffline = true
                            )
                        }
                    }
                    is AppError.Unauthorized -> {
                        // Token 过期，跳转登录
                        _uiState.update {
                            it.copy(navigateToLogin = true)
                        }
                    }
                    is AppError.Timeout -> {
                        // 超时重试
                        retryLoad()
                    }
                    else -> {
                        _uiState.update {
                            it.copy(errorMessage = error.message)
                        }
                    }
                }
            }
    }
}
```
## 用户体验
1.无网络 → 显示缓存数据 + 离线提示  
```
// 不同错误展示不同 UI
when {
    uiState.isOffline -> {
        // 顶部显示离线提示条
        OfflineBanner()
    }
    uiState.errorMessage != null -> {
        // Snackbar 提示
        LaunchedEffect(uiState.errorMessage) {
            snackbarHostState.showSnackbar(
                message = uiState.errorMessage,
                actionLabel = "重试"
            )
        }
    }
    uiState.items.isEmpty() && !uiState.isLoading -> {
        // 空状态页
        EmptyState(onRetry = vm::loadWatchlist)
    }
}
```

2.Token过期 → 自动跳转登录   
3.超时 → 自动重试  
```
// Retrofit 配置重试
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(RetryInterceptor(maxRetry = 3))
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(10, TimeUnit.SECONDS)
    .build()

class RetryInterceptor(private val maxRetry: Int) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var retryCount = 0
        var response: Response
        do {
            response = chain.proceed(chain.request())
            retryCount++
        } while (!response.isSuccessful && retryCount < maxRetry)
        return response
    }
}
```

4.业务错误 → Snackbar 提示具体原因
## 监控运维
1.客户端上报错误日志  
2.结合后端监控告警  
3.形成完整的错误闭环

# Pending
```
# 网络优化	
## 请求优化	
### 请求合并
### 批量上报
### 接口聚合（BFF）
## 协议优化	
### HTTP/2 多路复用
### QUIC/HTTP3
### Protobuf 替代 JSON
## 缓存策略	
### 强缓存/协商缓存
### 离线缓存、预请求
## 连接优化	
### 连接池复用
### DNS 预解析、IP 直连
## 弱网优化	
### 超时重试策略
### 断点续传
### 网络质量感知
## 安全优化	
### HTTPS 证书校验
### SSL Pinning
### 数据加密
## 网络监控	
### 请求耗时
### 成功率
### 流量统计
### OkHttp Interceptor
```
