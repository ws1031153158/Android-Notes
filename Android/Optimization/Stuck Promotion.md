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
