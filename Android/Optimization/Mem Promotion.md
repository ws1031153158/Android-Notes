# 内存优化	
## 内存模型
```
Android 进程内存构成：

VSS（Virtual Set Size）虚拟内存
├── 进程申请的所有虚拟地址空间
├── 包含未实际使用的内存
└── 通常远大于实际使用

RSS（Resident Set Size）常驻内存
├── 实际在物理内存中的页
├── 包含共享库的内存
└── 比 PSS 大

PSS（Proportional Set Size）比例内存
├── 私有内存 + 共享内存按比例分摊
├── 最能反映进程实际占用
└── adb dumpsys meminfo 显示的主要指标

USS（Unique Set Size）独占内存
├── 只属于该进程的私有内存
├── 进程退出后释放的内存
└── 排查内存泄漏的关键指标

┌─────────────────────────────────────────────┐
│              进程内存布局                     │
├─────────────────────────────────────────────┤
│ Java Heap    │ Java/Kotlin 对象              │
│ Native Heap  │ C/C++ malloc 分配             │
│ Code         │ DEX/so 代码段                 │
│ Stack        │ 线程栈                        │
│ Graphics     │ GL纹理/Surface缓冲            │
│ Other        │ mmap文件/匿名内存等            │
└─────────────────────────────────────────────┘
```

## 内存限制
```
不同设备的内存限制：

// 获取当前 App 的内存限制
val am = getSystemService(ActivityManager::class.java)
val memoryClass = am.memoryClass         // 标准堆大小限制（MB）
val largeMemoryClass = am.largeMemoryClass // largeHeap 限制（MB）

典型值：
┌──────────────┬──────────────┬────────────────┐
│ 设备级别     │ memoryClass  │ largeMemoryClass│
├──────────────┼──────────────┼────────────────┤
│ 低端机       │ 64MB~128MB   │ 128MB~256MB    │
│ 中端机       │ 128MB~256MB  │ 256MB~512MB    │
│ 高端机       │ 256MB~512MB  │ 512MB~1GB      │
└──────────────┴──────────────┴────────────────┘

// largeHeap：在 Manifest 中申请更大堆
// <application android:largeHeap="true">
// 不推荐！治标不治本，且影响系统整体性能
```
## 内存泄漏
原因：对象不再使用，但无法被GC回收  
问题：内存持续增长 → OOM  
特征：USS/PSS 随时间单调递增  

```
GC Root（不会被回收的根节点）：
├── 静态变量
├── 活跃线程
├── JNI 全局引用
└── 系统类

泄漏路径：
GC Root → 长生命周期对象 → 泄漏的短生命周期对象
                              （Activity/Fragment/View）
```
### 单例/静态对象引起的内存泄漏  
生命周期与程序生命周期一样，若有对象不再使用，却被单例/静态对象持有，就会导致其无法回收，进而导致内存泄漏。  
如传入了Activity作为context对象，就会导致 Activity 销毁时无法被回收，传入 Application 上下文可解决。  

非静态对象，每次 Activity 重启都会创建一次  
静态对象，只会创建一次（和 APP 同生命周期），但主动创建多次，虽然static 变量只有一份，但等于反复给它赋新值，每次 new 的还是是新对象（相当于地址指向换了）
### 非静态内部类引起的内存泄漏 
```
// ❌ 泄漏：非静态内部类隐式持有外部类引用
class BadActivity : AppCompatActivity() {

    // 非静态内部类：隐式持有 BadActivity.this
    inner class DataLoader : AsyncTask<Void, Void, String>() {
        override fun doInBackground(vararg params: Void?): String {
            Thread.sleep(10000) // 耗时操作
            return "result"
        }
        override fun onPostExecute(result: String) {
            updateUI(result) // 访问外部类
        }
    }

    // 问题：Activity 销毁后，DataLoader 还在运行
    // DataLoader 持有 Activity 引用 → Activity 无法回收
}

// ✅ 修复：静态内部类 + 弱引用
class GoodActivity : AppCompatActivity() {

    private val loader = DataLoader(this)

    // 静态内部类：不持有外部类引用
    class DataLoader(activity: GoodActivity) {
        // 弱引用：不阻止 Activity 被回收
        private val weakActivity = WeakReference(activity)

        fun load() {
            GlobalScope.launch(Dispatchers.IO) {
                val result = doHeavyWork()
                withContext(Dispatchers.Main) {
                    // Activity 可能已销毁，检查弱引用
                    weakActivity.get()?.updateUI(result)
                }
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        // 取消正在进行的任务
    }
}
```
### Handler引起的内存泄漏：  
若 Handler 通过内部类创建，内部类持有外部类引用（Activity），而队列中的消息 target 指向 Handler，即消息持有 Handler 引用，若 Activity 销毁时队列中的消息还未处理完，这些未处理完的消息会持有 Activity 引用，导致 Activity 无法回收。  
Handler 改为静态内部类，Activity#onDestroy 中移除队列中消息，Handler 内弱引用 Activity 对象，让内部类不再持有外部类的引用时，程序就不允许 Handler 操作 Activity 对象。    


此外，其他类似的异步任务（如：Asynctask，内部类持有外部类引用）也可能会有同样的原因引起内存泄漏。  
同样可以通过改为静态内部类并在 onDestroy 中取消任务解决问题。  

```
// ❌ 泄漏
class BadActivity : AppCompatActivity() {
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            // 匿名内部类持有 BadActivity.this
            updateUI()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 发送延迟消息
        handler.sendEmptyMessageDelayed(0, 60000) // 60秒后执行
        // Activity 销毁后，handler 还在 MessageQueue 中
        // handler 持有 Activity → 泄漏！
    }
}

// ✅ 修复
class GoodActivity : AppCompatActivity() {

    private val handler = SafeHandler(this)

    private class SafeHandler(activity: GoodActivity)
        : Handler(Looper.getMainLooper()) {

        private val ref = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            ref.get()?.updateUI() // 弱引用检查
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null) // 清除所有消息！
    }
}
```
### Coroutine/RxJava 引起的内存泄漏
```
// ❌ 泄漏：GlobalScope 启动的协程持有 Activity
class BadActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        GlobalScope.launch { // GlobalScope 生命周期是整个App！
            delay(60000)
            updateUI() // 持有 Activity 引用
        }
    }
}

// ✅ 修复：使用 lifecycleScope
class GoodActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch { // Activity 销毁时自动取消
            delay(60000)
            updateUI()
        }
    }
}

// ViewModel 中使用 viewModelScope
class GoodViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch { // ViewModel 清除时自动取消
            fetchData()
        }
    }
}

```
### WebView 引起的内存泄漏  
```
// WebView 通过 XML inflate 时，会绑定 Activity 的 Context

// 动态创建 WebView，使用 ApplicationContext
// 不要用 this（Activity Context），用 ApplicationContext

WebView mWebView = new WebView(getApplicationContext());
LinearLayout mll = findViewById(R.id.webview_container);
mll.addView(mWebView);


// onDestroy 中显式销毁

@Override
protected void onDestroy() {
    super.onDestroy();
    if (mWebView != null) {
        mWebView.removeAllViews();  // 移除所有子View
        mWebView.destroy();         // 销毁WebView
        mWebView = null;
    }
}
```
### 集合持有对象引起的内存泄漏
```
// ❌ 泄漏：全局集合持有 Activity/View
object GlobalCache {
    val activityCache = mutableListOf<Activity>() // 严重泄漏！
    val viewCache = mutableMapOf<String, View>()  // 泄漏！
}

// ✅ 修复：使用 WeakReference 或及时清理
object SafeCache {
    private val cache = mutableMapOf<String, WeakReference<View>>()

    fun put(key: String, view: View) {
        cache[key] = WeakReference(view)
    }

    fun get(key: String): View? {
        return cache[key]?.get().also {
            if (it == null) cache.remove(key) // 清理已回收的引用
        }
    }
}
```
### 监听器/回调未注销引起的内存泄漏
```
// ❌ 泄漏
class BadFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // 注册监听器（EventBus/LiveData/BroadcastReceiver等）
        EventBus.getDefault().register(this)
        LocalBroadcastManager.getInstance(requireContext())
            .registerReceiver(receiver, IntentFilter("action"))
        SensorManager.registerListener(sensorListener, sensor, delay)
    }

    // 忘记在 onDestroyView/onDestroy 中注销！
    // Fragment 被销毁，但监听器还持有 Fragment 引用
}

// ✅ 修复：成对注册/注销
class GoodFragment : Fragment() {

    private val receiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            handleBroadcast(intent)
        }
    }

    override fun onStart() {
        super.onStart()
        EventBus.getDefault().register(this)
        LocalBroadcastManager.getInstance(requireContext())
            .registerReceiver(receiver, IntentFilter("action"))
    }

    override fun onStop() {
        super.onStop()
        EventBus.getDefault().unregister(this)
        LocalBroadcastManager.getInstance(requireContext())
            .unregisterReceiver(receiver)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        // View 相关的监听器在这里注销
        sensorManager.unregisterListener(sensorListener)
    }
}
```
## OOM
原因：申请内存超过系统限制  
问题：进程崩溃  
特征：堆内存/图片内存/线程栈等超限  

JVM 在面临内存资源不足时的一种自我保护机制，遭遇 OOM 时，不会立即终止执行，无法为对象分配内存空间时，抛出 OOM（Error 子类，标志着一种通常不可恢复的、严重的运行时问题，本身不会直接导致 JVM 退出），JVM 会将这个错误传递给当前正在运行的代码，这给予了应用程序一个机会去捕获并处理这个异常，尽管在常规情况下并不推荐捕获和处理这种严重错误，但如果确实进行了这样的操作，程序可能会尝试继续执行。    
### 类型堆
① Java 堆 OOM（最常见）：  
```
   java.lang.OutOfMemoryError: Java heap space
   原因：Java 堆超过 memoryClass 限制，对象的持续创建，如果它们因为某些原因（如内存泄漏）而无法被垃圾收集器有效回收，那么堆内存最终会被消耗殆尽，往往是因为代码中存在内存管理不当
   解决：
   object MemoryManager {

    // 内存压力监控
    fun registerMemoryWatcher(context: Context) {
        val am = context.getSystemService(ActivityManager::class.java)

        // 定期检查内存使用情况
        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(30_000) // 每30秒检查一次

                val memInfo = ActivityManager.MemoryInfo()
                am.getMemoryInfo(memInfo)

                val usedMemMB = getUsedMemory() / 1024 / 1024
                val maxMemMB = Runtime.getRuntime().maxMemory() / 1024 / 1024
                val usageRate = usedMemMB.toFloat() / maxMemMB

                when {
                    usageRate > 0.9f -> {
                        // 内存使用超过 90%，紧急释放
                        Log.w("MemoryManager", "内存告警：${usedMemMB}MB/${maxMemMB}MB")
                        emergencyRelease()
                    }
                    usageRate > 0.75f -> {
                        // 内存使用超过 75%，主动释放
                        normalRelease()
                    }
                }
            }
        }
    }

    private fun getUsedMemory(): Long {
        val runtime = Runtime.getRuntime()
        return runtime.totalMemory() - runtime.freeMemory()
    }

    private fun emergencyRelease() {
        // 清空图片缓存
        Glide.get(AppContext.get()).clearMemory()
        // 清空业务缓存
        DataCache.clearAll()
        // 触发 GC
        System.gc()
    }

    private fun normalRelease() {
        // 清理过期缓存
        DataCache.clearExpired()
    }
}
```

② 图片 OOM:  
```
   java.lang.OutOfMemoryError: bitmap size exceeds VM budget
   原因：单张图片或图片总量过大
   解决：参考上方图片优化
```

③ 线程 OOM:  
```
   java.lang.OutOfMemoryError: pthread_create failed
   原因：线程数量过多，每个线程需要 512KB~8MB 栈空间
   解决：
   // 问题：无限制创建线程
// ❌ 每次都创建新线程
fun badThreadUsage() {
    Thread { doWork() }.start() // 无限制创建！
}

// ✅ 使用线程池，限制线程数量
object ThreadPoolManager {

    // IO 密集型：线程数 = CPU核心数 * 2
    val ioExecutor: ExecutorService = ThreadPoolExecutor(
        2,                          // 核心线程数
        Runtime.getRuntime().availableProcessors() * 2, // 最大线程数
        60L, TimeUnit.SECONDS,      // 空闲线程存活时间
        LinkedBlockingQueue(128),   // 任务队列（限制队列大小！）
        ThreadFactory { r ->
            Thread(r, "io-pool-${threadCount.incrementAndGet()}")
                .apply { isDaemon = true }
        },
        ThreadPoolExecutor.CallerRunsPolicy() // 队列满时，调用者线程执行
    )

    // CPU 密集型：线程数 = CPU核心数 + 1
    val cpuExecutor: ExecutorService = ThreadPoolExecutor(
        Runtime.getRuntime().availableProcessors() + 1,
        Runtime.getRuntime().availableProcessors() + 1,
        0L, TimeUnit.MILLISECONDS,
        LinkedBlockingQueue(64),
        ThreadFactory { r ->
            Thread(r, "cpu-pool-${threadCount.incrementAndGet()}")
                .apply { isDaemon = true }
        }
    )

    private val threadCount = AtomicInteger(0)
}
```

④ FD（文件描述符）OOM:  
```
   java.lang.OutOfMemoryError: Could not allocate JNI Env
   原因：FD 数量超过系统限制（通常 1024）
   解决：
   // FD 泄漏：打开文件/Socket/Cursor 后未关闭
// ❌ 泄漏
fun readFileBad(file: File): String {
    val fis = FileInputStream(file)
    return fis.bufferedReader().readText()
    // 忘记 close()！FD 泄漏
}

// ✅ 使用 use{}（自动关闭）
fun readFileGood(file: File): String {
    return FileInputStream(file).use { fis ->
        fis.bufferedReader().readText()
        // use{} 结束时自动调用 close()
    }
}

// 监控 FD 数量
object FDMonitor {
    fun getFDCount(): Int {
        return try {
            File("/proc/${Process.myPid()}/fd").listFiles()?.size ?: 0
        } catch (e: Exception) { 0 }
    }

    fun checkFDLeak() {
        val fdCount = getFDCount()
        if (fdCount > 800) { // 接近1024限制
            Log.e("FDMonitor", "FD数量告警：$fdCount")
            // 打印所有FD信息，找出泄漏来源
            dumpFDInfo()
        }
    }

    private fun dumpFDInfo() {
        File("/proc/${Process.myPid()}/fd").listFiles()?.forEach { fd ->
            try {
                val link = Os.readlink(fd.absolutePath)
                Log.d("FDMonitor", "${fd.name} -> $link")
            } catch (e: Exception) { }
        }
    }
}
```

⑤ Native 堆 OOM:  
```
   原因：JNI/so 中 malloc 失败
```

⑥ 方法栈 OOM:  
```
   java.lang.OutOfMemoryError: StackOverflowError
   原因：线程请求的栈大小超出了JVM 所允许的最大值，通常与线程的设计和实现有关
   解决： 一般开发阶段就是必现问题，最常见的是死循环
```

⑦ 元空间/方法区 OOM:  
```
   java.lang.OutOfMemoryError: PermGen space/Metaspace
   原因：系统加载大量的类和方法时，这部分内存资源可能会变得紧张，通常发生在应用程序需要动态加载大量代码的场景中
   解决： 尽可能避免同一时间，大量动态加载代码
```
## 图片优化
### 图片格式
PNG：无损压缩图片方式，支持 Alpha 通道，切图素材大多用这种格式。  
JPEG：有损压缩图片格式，不支持背景透明和多帧动画，适用于色彩丰富的图片压缩，不适合于 logo。  
WEBP：支持有损和无损压缩，支持完整的透明通道，也支持多帧动画，是一种比较理想的图片格式。  
.9图：点九图实际上仍然是 png 格式图片，它是针对 Andorid 平台特殊的图片格式，体积小，拉伸变形，能指定位置拉伸或者填充，能很好的适配机型。  
无损 webp 平均比 png 小26%，有损 jpeg 平均比 webp 少24%-35%，无损 webp 支持 Alpha 通道，有损 webp 在一定条件下也支持。采用 webp 在保持图片清晰情况下，可以优先减少磁盘空间大小。可以将 drawable 中的 png、jpg 格式图片转换为 webp 格式图片。  
### 像素格式
ALPHA_8，内存 1B，只有透明度，比较少用到。  
RGB_565，内存2B，只有颜色，不需要 Alpha 通道的，特别是 .JPG 格式的，无透明度，质量稍低。  
ARGB_4444，内存2B，颜色+透明度，内存只占 ARGB_8888 的一半，已被废弃。  
ARGB_8888，内存4B，颜色+透明度，默认格式，最高质量。  
通过替换系统 drawable 默认色彩通道（BitmapFactory.Options.inPreferredConfig），将部分没有透明通道的图片格式由 ARGB_8888 替换为 RGB565，在图片质量上的损失几乎肉眼不可见，而每张图片可以节省1/2的内存。但不通用，取决于图片是否有透明度需求。  
### Bitmap优化
图片内存：  
宽高一个像素占用内存 = 宽 × 高 × 每像素字节数（和色彩模式有关，如 ARGB_8888 为 4）。  
```
1080×1920 的图片
├── ARGB_8888：1080 × 1920 × 4 = 8.29MB
├── RGB_565：1080 × 1920 × 2 = 4.15MB
└── 如果图片实际是 100×100，但被放大显示
    → 仍然按 1080×1920 分配内存！

关键认知：
Bitmap 内存大小 = 解码后的像素数据大小
与原始文件大小（jpg/png）无关！
一张 100KB 的 jpg，解码后可能占 8MB 内存
```

采样率:  
计算出一个合适的 inSampleSize 缩放比例，降低图片像素，来达到降低图片占用内存大小的目的，避免不必要的大图载入。    
采样率 inSampleSize 只能是整数(只能是2的次方)，不能很好保证图片质量。如果 inSampleSize=2，则最终内存占用就会是原来的1/4（宽高都为原来的1/2），适用于图片过大的情况。    

压缩：  
```
// 选择合适的像素格式

fun loadBitmapOptimized(path: String, hasAlpha: Boolean): Bitmap {
    return BitmapFactory.Options().run {
        // 无透明度需求：使用 RGB_565，内存减半
        inPreferredConfig = if (hasAlpha) {
            Bitmap.Config.ARGB_8888
        } else {
            Bitmap.Config.RGB_565
        }
        BitmapFactory.decodeFile(path, this)
    }
}


// 按需加载合适尺寸

fun decodeSampledBitmap(
    resources: Resources,
    resId: Int,
    reqWidth: Int,   // 目标宽度（View的实际宽度）
    reqHeight: Int   // 目标高度
): Bitmap {
    return BitmapFactory.Options().run {
        // 第一次：只读取尺寸信息，不分配内存
        inJustDecodeBounds = true
        BitmapFactory.decodeResource(resources, resId, this)

        // 计算采样率（必须是2的幂次）
        inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)

        // 第二次：按采样率解码
        inJustDecodeBounds = false
        BitmapFactory.decodeResource(resources, resId, this)
    }
}

fun calculateInSampleSize(
    options: BitmapFactory.Options,
    reqWidth: Int,
    reqHeight: Int
): Int {
    val (height, width) = options.run { outHeight to outWidth }
    var inSampleSize = 1

    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        // 找到最大的 inSampleSize（2的幂次）
        // 使得缩放后的尺寸仍然大于等于目标尺寸
        while (halfHeight / inSampleSize >= reqHeight &&
               halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

复用（inBitmap）：  
```
class BitmapPool(private val maxSize: Int = 10) {

    // 按尺寸分组的 Bitmap 池
    private val pool = LruCache<String, Bitmap>(maxSize)

    fun getBitmap(width: Int, height: Int, config: Bitmap.Config): Bitmap? {
        val key = "${width}x${height}_${config}"
        return pool.get(key)?.takeIf { !it.isRecycled }
    }

    fun putBitmap(bitmap: Bitmap) {
        if (bitmap.isRecycled) return
        val key = "${bitmap.width}x${bitmap.height}_${bitmap.config}"
        pool.put(key, bitmap)
    }
}

val bitmapPool = BitmapPool()

fun decodeBitmapWithReuse(path: String, width: Int, height: Int): Bitmap {
    val options = BitmapFactory.Options().apply {
        inMutable = true // 允许复用

        // 尝试从池中获取可复用的 Bitmap
        val reusable = bitmapPool.getBitmap(width, height, Bitmap.Config.ARGB_8888)
        if (reusable != null) {
            inBitmap = reusable // 复用内存，不重新分配
        }

        inSampleSize = 1
    }

    return try {
        BitmapFactory.decodeFile(path, options)
    } catch (e: IllegalArgumentException) {
        // inBitmap 复用失败（尺寸不匹配），重新解码
        options.inBitmap = null
        BitmapFactory.decodeFile(path, options)
    }
}
```

BitmapRegionDecoder：  

释放：    
```
// Bitmap 在内存中的存储分两部分 ：一部分是 Bitmap 对象，另一部分为对应的像素数据，前者占据的内存较小，而后者才是内存占用的大头。 
// bitmap.recycle 用于释放与当前 Bitmap 对象相关联的 Native 对象，并清理对像素数据的引用。但不能同步地释放像素数据，而是在没有其它引用的时候，简单地允许像素数据被作为垃圾回收掉。  

// 调用 recycle，并且将维护的局部/全局对象等置为 null

class ImageHolder {
    private var bitmap: Bitmap? = null

    fun updateBitmap(newBitmap: Bitmap) {
        // 回收旧 Bitmap 再赋值新的
        bitmap?.takeIf { !it.isRecycled }?.recycle()
        bitmap = newBitmap
    }

    fun release() {
        bitmap?.takeIf { !it.isRecycled }?.recycle()
        bitmap = null
    }
}
```
## 内存抖动
原因：短时间内大量创建销毁（临时）对象  
问题：频繁GC → 卡顿  
特征：内存曲线锯齿形波动  
### 频繁 GC
无论哪种方式实现的GC在执行时都不可避免的需要 STW（Stop The World），STW 意味着所有的工作线程都将被暂停，虽然时间很短，但终究会存在时间成本，一两次内存回收不易被察觉，但多次内存回收集中在短时间内爆发，这就会造成较大程度的界面卡顿风险。   
尽量避免在循环体中创建对象、尽量不要在自定义 View 的 onDraw 方法中创建对象（会被频繁调用） 
### 对象池复用
对于可复用对象，可以考虑使用对象池缓存。
### 检测
```
// 检测工具：
// Android Studio Profiler → Memory → Record allocations
// 观察：
// ├── 短时间内大量对象创建（锯齿形内存曲线）
// ├── GC 频率（Profiler 中的 GC 事件）
// └── 哪些类被频繁创建

// ================================
// 常见内存抖动场景
// ================================

// 场景一：字符串拼接
// 在JVM将java代码翻译成字节码的时候，String对象的每次拼接都会创建StringBuilder对象。
fun buildLogBad(items: List<String>): String {
    var result = ""
    items.forEach { result += it + "," } // 每次 += 创建新 String
    return result
}

// 场景二：onDraw 中创建对象
class BadCustomView(context: Context) : View(context) {
    override fun onDraw(canvas: Canvas) {
        // ❌ 每帧都创建 Paint 和 RectF！
        val paint = Paint() // 每帧创建
        val rect = RectF(0f, 0f, width.toFloat(), height.toFloat()) // 每帧创建
        canvas.drawRect(rect, paint)
    }
}

// ================================
// 通用对象池
// ================================
class ObjectPool<T : Any>(
    private val maxPoolSize: Int = 20,
    private val create: () -> T,
    private val reset: T.() -> Unit = {},
    private val validate: T.() -> Boolean = { true }
) {
    private val pool = ArrayDeque<T>(maxPoolSize)
    private val lock = Object()

    // 获取对象
    fun acquire(): T {
        synchronized(lock) {
            while (pool.isNotEmpty()) {
                val obj = pool.removeFirst()
                if (obj.validate()) return obj
                // 验证失败（对象已损坏），丢弃继续找
            }
        }
        return create() // 池空了，创建新对象
    }

    // 归还对象
    fun release(obj: T) {
        obj.reset() // 重置状态
        synchronized(lock) {
            if (pool.size < maxPoolSize) {
                pool.addLast(obj)
            }
            // 池满了，让 GC 回收
        }
    }

    // 使用完自动归还（类似 use{}）
    inline fun <R> use(block: (T) -> R): R {
        val obj = acquire()
        return try {
            block(obj)
        } finally {
            release(obj)
        }
    }

    fun clear() {
        synchronized(lock) { pool.clear() }
    }
}

// ================================
// 实际使用场景
// ================================

// Rect 对象池（自定义View中频繁使用）
val rectPool = ObjectPool(
    maxPoolSize = 20,
    create = { Rect() },
    reset = { setEmpty() }
)

val rectFPool = ObjectPool(
    maxPoolSize = 20,
    create = { RectF() },
    reset = { setEmpty() }
)

// Paint 对象池
val paintPool = ObjectPool(
    maxPoolSize = 10,
    create = { Paint(Paint.ANTI_ALIAS_FLAG) },
    reset = { reset() }
)

// 使用示例
class ChartView(context: Context) : View(context) {

    override fun onDraw(canvas: Canvas) {
        rectFPool.use { rect ->
            rect.set(0f, 0f, width.toFloat(), height.toFloat())
            paintPool.use { paint ->
                paint.color = Color.BLUE
                canvas.drawRect(rect, paint)
            }
        }
    }
}

// ================================
// 检测：自定义 GC 监控
// ================================
object GCMonitor {

    private var lastGCTime = 0L
    private var gcCount = 0

    fun startMonitoring() {
        // 通过 Debug.getRuntimeStat 监控 GC 次数
        thread(name = "gc-monitor", isDaemon = true) {
            while (true) {
                Thread.sleep(1000)
                checkGCFrequency()
            }
        }
    }

    private fun checkGCFrequency() {
        val gcInfo = Debug.getRuntimeStat("art.gc.gc-count")
        val currentGCCount = gcInfo?.toIntOrNull() ?: return

        val gcDelta = currentGCCount - gcCount
        gcCount = currentGCCount

        if (gcDelta > 5) { // 1秒内GC超过5次
            Log.w("GCMonitor", "GC频繁：1秒内触发 $gcDelta 次GC，可能存在内存抖动")
        }
    }
}
```
## 内存碎片
原因：空闲内存不连续，无法分配大块内存  
问题：OOM（即使总空闲内存足够）  
特征：free内存多但仍然OOM  
### Java 堆碎片
产生原理：  
```
ART 堆结构：
┌─────────────────────────────────────────────┐
│  Zygote Space  │  分配给所有App的共享只读空间  │
├─────────────────────────────────────────────┤
│  Image Space   │  预加载的类和对象            │
├─────────────────────────────────────────────┤
│  Large Object  │  大对象（>12KB）独立管理     │
│  Space         │                             │
├─────────────────────────────────────────────┤
│  Non-Moving    │  不可移动对象（如JNI引用的）  │
│  Space         │                             │
├─────────────────────────────────────────────┤
│  Moving Space  │  普通对象，GC时可移动整理     │
│  (Semi-Space)  │                             │
└─────────────────────────────────────────────┘

碎片产生过程：
时刻1：[A:100B][B:200B][C:50B][D:300B][E:100B]
时刻2：B和D被回收（标记清除算法）
       [A:100B][空:200B][C:50B][空:300B][E:100B]
时刻3：尝试分配 400B 对象
       总空闲 = 200 + 300 = 500B > 400B
       但最大连续空闲 = 300B < 400B
       → 分配失败！OOM！（即使总空闲足够）

ART 的应对策略：
├── Moving Space：GC 时将存活对象紧凑排列（消除碎片）
├── Large Object Space：大对象单独管理，减少碎片影响
└── Compaction GC：主动整理堆内存（但有 STW 开销）
```

检测：  
```
// Java 堆碎片检测
object JavaHeapFragmentDetector {

    data class HeapFragmentInfo(
        val totalHeapMB: Long,
        val usedHeapMB: Long,
        val freeHeapMB: Long,
        val fragmentRatio: Float,  // 碎片率
        val largestFreeChunkMB: Long
    )

    fun analyze(): HeapFragmentInfo {
        val runtime = Runtime.getRuntime()
        val totalHeap = runtime.totalMemory()
        val freeHeap = runtime.freeMemory()
        val usedHeap = totalHeap - freeHeap

        // 通过 Debug.MemoryInfo 获取详细信息
        val memInfo = Debug.MemoryInfo()
        Debug.getMemoryInfo(memInfo)

        // 碎片率估算：
        // 触发 GC 前后的 free 内存差值 / GC 前 free 内存
        val freeBeforeGC = freeHeap
        System.gc()
        val freeAfterGC = runtime.freeMemory()

        // GC 后释放的内存 = 真正的垃圾
        // GC 前后 free 差值小 = 碎片多（free 内存无法被整合）
        val fragmentRatio = 1f - (freeBeforeGC.toFloat() / freeAfterGC)

        return HeapFragmentInfo(
            totalHeapMB = totalHeap / 1024 / 1024,
            usedHeapMB = usedHeap / 1024 / 1024,
            freeHeapMB = freeAfterGC / 1024 / 1024,
            fragmentRatio = fragmentRatio.coerceIn(0f, 1f),
            largestFreeChunkMB = estimateLargestFreeChunk()
        )
    }

    // 通过尝试分配不同大小的数组来估算最大连续空闲块
    private fun estimateLargestFreeChunk(): Long {
        var low = 1L
        var high = Runtime.getRuntime().freeMemory() / 1024 / 1024

        while (low < high) {
            val mid = (low + high + 1) / 2
            try {
                // 尝试分配 mid MB 的数组
                @Suppress("unused")
                val probe = ByteArray((mid * 1024 * 1024).toInt())
                low = mid
            } catch (e: OutOfMemoryError) {
                high = mid - 1
            }
        }
        return low
    }
}
```
### Native 堆碎片
```
Native 堆碎片比 Java 堆更严重：
├── Java 堆：GC 可以移动对象，整理碎片
└── Native 堆：malloc/free 分配的内存无法移动
              一旦产生碎片，只能等进程重启

Native 内存分配器（Android 默认）：
├── Android 5.0 以前：dlmalloc
├── Android 5.0+：jemalloc（更好的碎片控制）
└── Android 10+：Scudo（安全增强版）

目标：
1.迅速地申请内存
2.迅速地释放内存
3.高效地使用内存
4.安全地访问内存

jemalloc 的碎片控制策略：
1.分级管理：小对象（<8B, 16B, 32B...）固定大小 bin；中对象（<4KB）按页管理；大对象（>4KB）直接 mmap，间隔：8B → 16B → 32B（跳跃式增长）
2.Thread Cache：每个线程有本地缓存，减少锁竞争
3.Arena：多个独立的内存池（不同大小的对象混放在同一个），数量固定（CPU核心数），减少碎片扩散

Scudo 的碎片控制策略：
1.按大小隔离：Primary 处理小/中对象(<256KB)；Secondary 处理大对象(>=256KB)，间隔更小（8B递增），内部碎片更少      
2.TSD 固定上限：每个线程独立缓存，设置上限
3.Region机制：每个 SizeClass 有独立的 Region（大小相同对象放一起），free 的槽位可以被任何同大小对象复用
4.Checksum 安全校验：hash(地址, 随机Cookie），free时重新计算hash，与存储对比，不一致则abort
```
### 碎片整理策略
```
object FragmentationStrategy {

    // ================================
    // 策略一：主动触发 GC + Heap Trim
    // ================================
    fun trimJavaHeap() {
        // 触发 GC，让 ART 整理 Moving Space
        System.gc()
        System.runFinalization()
        System.gc()

        // 通知 ART 将空闲内存归还给 OS
        // Android 4.1+ 支持
        try {
            val vmRuntime = Class.forName("dalvik.system.VMRuntime")
            val getRuntime = vmRuntime.getDeclaredMethod("getRuntime")
            val runtime = getRuntime.invoke(null)
            val trim = vmRuntime.getDeclaredMethod("trimHeap")
            trim.invoke(runtime)
        } catch (e: Exception) {
            Log.w("FragmentationStrategy", "trimHeap 失败: ${e.message}")
        }
    }

    // ================================
    // 策略二：Native 堆碎片整理
    // ================================
    fun trimNativeHeap() {
        // malloc_trim：将 Native 堆顶部的空闲内存归还给 OS
        // 减少 Native 堆的 RSS
        try {
            val libC = Class.forName("android.system.Os")
            // 通过 JNI 调用 malloc_trim(0)
            NativeHeapTrimmer.trim()
        } catch (e: Exception) { }
    }

    // ================================
    // 策略三：对象分配策略优化（预防碎片）
    // ================================

    // 大对象复用：避免频繁分配/释放大内存块
    class LargeObjectPool<T>(
        private val create: () -> T,
        private val reset: (T) -> Unit
    ) {
        // 大对象（>12KB）在 Large Object Space
        // 不参与 Moving GC，容易产生碎片
        // 通过复用减少大对象的分配/释放频率
        private val pool = ArrayDeque<T>(3) // 大对象池不宜太大

        fun acquire(): T = pool.removeFirstOrNull() ?: create()
        fun release(obj: T) {
            reset(obj)
            if (pool.size < 3) pool.addLast(obj)
        }
    }

    // ================================
    // 策略四：后台时主动整理
    // ================================
    fun onAppBackground() {
        GlobalScope.launch(Dispatchers.IO) {
            // App 进入后台，用户不感知，可以做耗时整理
            trimJavaHeap()
            trimNativeHeap()

            // 清理各级缓存
            ImageCache.trimToSize(0)
            DataCache.clearNonEssential()
        }
    }
}
```
## 后台进程内存占用
### 主动内存收缩
```
// App 进入后台时，主动释放内存
// 目标：降低 oom_adj，减少被 LMK 杀死的概率

class BackgroundMemoryManager(private val app: Application) {

    // 内存收缩级别
    enum class ShrinkLevel {
        LIGHT,    // 轻度：释放图片缓存
        MODERATE, // 中度：释放业务缓存
        AGGRESSIVE // 激进：释放几乎所有缓存
    }

    fun shrink(level: ShrinkLevel) {
        when (level) {
            ShrinkLevel.LIGHT -> {
                // 释放图片内存缓存（磁盘缓存保留）
                Glide.get(app).clearMemory()
                // 释放非当前页面的 View 缓存
                ViewCache.clearInactiveViews()
            }

            ShrinkLevel.MODERATE -> {
                // 在 LIGHT 基础上
                // 释放业务数据内存缓存
                DataCache.clearAll()
                // 释放 WebView 缓存
                WebView(app).clearCache(false)
                // 触发 GC
                System.gc()
            }

            ShrinkLevel.AGGRESSIVE -> {
                // 在 MODERATE 基础上
                // 释放所有可释放内存
                trimJavaHeap()
                trimNativeHeap()
                // 通知各模块释放内存
                MemoryEventBus.post(MemoryEvent.AGGRESSIVE_TRIM)
            }
        }
    }

    // 根据系统内存压力自动选择收缩级别
    fun onTrimMemory(level: Int) {
        val shrinkLevel = when (level) {
            ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN ->
                ShrinkLevel.LIGHT

            ComponentCallbacks2.TRIM_MEMORY_BACKGROUND,
            ComponentCallbacks2.TRIM_MEMORY_MODERATE ->
                ShrinkLevel.MODERATE

            ComponentCallbacks2.TRIM_MEMORY_COMPLETE,
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL ->
                ShrinkLevel.AGGRESSIVE

            else -> return
        }
        shrink(shrinkLevel)
    }
}
```
### 内存快照对比分析
```
// 通过对比前后台内存快照，找出后台内存增长点

object MemorySnapshotComparator {

    data class MemorySnapshot(
        val timestamp: Long,
        val javaHeapMB: Long,
        val nativeHeapMB: Long,
        val graphicsMB: Long,
        val stackMB: Long,
        val codeMB: Long,
        val otherMB: Long,
        val totalPssMB: Long,
        // 对象级别快照（需要 Debug.dumpHprofData）
        val topObjectCounts: Map<String, Int> = emptyMap()
    ) {
        val totalMB get() = javaHeapMB + nativeHeapMB + graphicsMB
    }

    private var foregroundSnapshot: MemorySnapshot? = null
    private var backgroundSnapshot: MemorySnapshot? = null

    fun takeSnapshot(context: Context): MemorySnapshot {
        val debug = Debug.MemoryInfo()
        Debug.getMemoryInfo(debug)

        return MemorySnapshot(
            timestamp = System.currentTimeMillis(),
            javaHeapMB = debug.getMemoryStat("summary.java-heap")
                .toLongOrNull()?.div(1024) ?: 0,
            nativeHeapMB = debug.getMemoryStat("summary.native-heap")
                .toLongOrNull()?.div(1024) ?: 0,
            graphicsMB = debug.getMemoryStat("summary.graphics")
                .toLongOrNull()?.div(1024) ?: 0,
            stackMB = debug.getMemoryStat("summary.stack")
                .toLongOrNull()?.div(1024) ?: 0,
            codeMB = debug.getMemoryStat("summary.code")
                .toLongOrNull()?.div(1024) ?: 0,
            otherMB = debug.getMemoryStat("summary.system")
                .toLongOrNull()?.div(1024) ?: 0,
            totalPssMB = debug.totalPss.toLong() / 1024
        )
    }

    fun onForeground(context: Context) {
        foregroundSnapshot = takeSnapshot(context)
    }

    fun onBackground(context: Context) {
        backgroundSnapshot = takeSnapshot(context)
        analyzeGrowth()
    }

    private fun analyzeGrowth() {
        val fg = foregroundSnapshot ?: return
        val bg = backgroundSnapshot ?: return

        val javaGrowth = bg.javaHeapMB - fg.javaHeapMB
        val nativeGrowth = bg.nativeHeapMB - fg.nativeHeapMB
        val graphicsGrowth = bg.graphicsMB - fg.graphicsMB

        // 后台内存不应该增长，增长说明有泄漏
        if (javaGrowth > 10) {
            Log.w("MemorySnapshot", "后台Java堆增长 ${javaGrowth}MB，可能有泄漏")
            APMReporter.reportBackgroundMemoryGrowth("java", javaGrowth)
        }
        if (nativeGrowth > 10) {
            Log.w("MemorySnapshot", "后台Native堆增长 ${nativeGrowth}MB")
            APMReporter.reportBackgroundMemoryGrowth("native", nativeGrowth)
        }
    }
}
```
### 子进程内存预算管控
```
// 多进程 App 的内存管理
// 每个进程都有独立的内存限制

object MultiProcessMemoryBudget {

    // 进程内存预算配置
    data class ProcessBudget(
        val processName: String,
        val maxHeapMB: Int,      // 最大堆内存
        val targetHeapMB: Int,   // 目标堆内存（正常运行）
        val criticalHeapMB: Int  // 临界值（超过则报警）
    )

    val budgets = mapOf(
        ":main" to ProcessBudget(":main", 256, 150, 200),
        ":push" to ProcessBudget(":push", 64, 30, 50),
        ":download" to ProcessBudget(":download", 128, 60, 100),
        ":webview" to ProcessBudget(":webview", 200, 100, 160)
    )

    fun getCurrentProcessBudget(): ProcessBudget? {
        val processName = getCurrentProcessName()
        return budgets[processName]
    }

    fun checkBudget() {
        val budget = getCurrentProcessBudget() ?: return
        val used = (Runtime.getRuntime().totalMemory() -
                Runtime.getRuntime().freeMemory()) / 1024 / 1024

        when {
            used > budget.maxHeapMB -> {
                // 超过最大预算，强制释放
                Log.e("MemoryBudget",
                    "${budget.processName} 超过最大内存预算: ${used}MB > ${budget.maxHeapMB}MB")
                forceRelease()
            }
            used > budget.criticalHeapMB -> {
                // 超过临界值，上报告警
                APMReporter.reportMemoryBudgetExceeded(
                    budget.processName, used, budget.criticalHeapMB
                )
            }
        }
    }

    private fun getCurrentProcessName(): String {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            Application.getProcessName()
        } else {
            try {
                File("/proc/self/cmdline").readText().trim { it <= ' ' }
            } catch (e: Exception) { "" }
        }
    }
}
```
### 进程保活与内存代价
```
进程保活方案的内存代价分析：

┌──────────────────┬──────────────┬──────────────────────┐
│ 保活方案         │ 内存代价     │ 说明                  │
├──────────────────┼──────────────┼──────────────────────┤
│ 前台Service      │ +5~10MB      │ 需要常驻通知栏        │
│ 1像素Activity    │ +2~5MB       │ 灰色地带，不推荐      │
│ JobScheduler     │ 极低         │ 合规，但不保证及时    │
│ WorkManager      │ 极低         │ 推荐，系统管理        │
│ 双进程守护       │ +20~50MB     │ 两个进程互相拉起      │
│ 账号同步         │ +3~8MB       │ 系统级，相对可靠      │
└──────────────────┴──────────────┴──────────────────────┘

内存代价 vs 保活收益 的权衡：
├── 用户价值高的功能（IM/导航）：前台Service，代价可接受
├── 普通后台任务：WorkManager，内存代价最低
└── 双进程守护：内存代价太高，且高版本系统已限制，不推荐

后台进程内存优化原则：
① 后台进程不应该持有大量业务数据
② 后台进程只保留必要的功能（推送/定位）
③ 进入后台立即释放 UI 相关内存
④ 使用 onTrimMemory 响应系统内存压力
```
## 匿名内存（Anonymous Memory）增长异常
```
匿名内存（Anonymous Memory）：
没有对应文件的内存映射
通过 mmap(MAP_ANONYMOUS) 或 malloc 分配

在 /proc/[pid]/maps 中显示为：
7f8a000000-7f8b000000 rw-p 00000000 00:00 0  [anon]
                                              ↑ 匿名内存

匿名内存异常的来源：
├── Native 内存泄漏（malloc 未 free）
├── 线程数膨胀（每个线程栈 = 匿名内存）
├── JNI GlobalRef 未释放
└── mmap 映射未释放
```
### Native 内存泄漏
```
// 方案一：重载 malloc/free，记录分配信息

#include <malloc.h>
#include <unordered_map>
#include <mutex>
#include <execinfo.h>

struct AllocInfo {
    size_t size;
    void* backtrace[16];
    int backtraceSize;
    long timestamp;
};

static std::unordered_map<void*, AllocInfo> gAllocMap;
static std::mutex gAllocMutex;
static bool gTracking = false;

// 重载 malloc
void* tracked_malloc(size_t size) {
    void* ptr = malloc(size);
    if (ptr && gTracking) {
        AllocInfo info;
        info.size = size;
        info.backtraceSize = backtrace(info.backtrace, 16);
        info.timestamp = time(nullptr);

        std::lock_guard<std::mutex> lock(gAllocMutex);
        gAllocMap[ptr] = info;
    }
    return ptr;
}

// 重载 free
void tracked_free(void* ptr) {
    if (ptr && gTracking) {
        std::lock_guard<std::mutex> lock(gAllocMutex);
        gAllocMap.erase(ptr);
    }
    free(ptr);
}

// 获取泄漏报告
extern "C" JNIEXPORT jstring JNICALL
Java_com_xxx_NativeLeakDetector_getLeakReport(JNIEnv* env, jobject) {
    std::lock_guard<std::mutex> lock(gAllocMutex);

    std::string report;
    size_t totalLeaked = 0;

    for (auto& [ptr, info] : gAllocMap) {
        totalLeaked += info.size;
        report += "地址: " + std::to_string((uintptr_t)ptr) +
                  " 大小: " + std::to_string(info.size) + "B\n";

        // 解析调用栈
        char** symbols = backtrace_symbols(info.backtrace, info.backtraceSize);
        if (symbols) {
            for (int i = 0; i < info.backtraceSize; i++) {
                report += "  " + std::string(symbols[i]) + "\n";
            }
            free(symbols);
        }
    }

    report = "总泄漏: " + std::to_string(totalLeaked) + "B\n" + report;
    return env->NewStringUTF(report.c_str());
}
```
### 线程数量膨胀
```
// 线程数膨胀：线程不断创建，不被回收
// 每个线程默认栈大小：8MB（64位）/ 512KB（32位）
// 100个线程 = 800MB 内存！

object ThreadLeakDetector {

    fun getThreadCount(): Int {
        return Thread.getAllStackTraces().size
    }

    fun getThreadDetails(): Map<String, List<String>> {
        return Thread.getAllStackTraces()
            .entries
            .groupBy { (thread, _) ->
                // 按线程名前缀分组
                thread.name.split("-").firstOrNull() ?: "unknown"
            }
            .mapValues { (_, entries) ->
                entries.map { (thread, _) ->
                    "${thread.name} [${thread.state}]"
                }
            }
    }

    // 监控线程数增长
    fun startMonitoring() {
        var lastCount = getThreadCount()

        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(10_000) // 每10秒检查

                val currentCount = getThreadCount()
                val growth = currentCount - lastCount

                if (currentCount > 100) {
                    Log.e("ThreadLeak", "线程数过多: $currentCount")
                    val details = getThreadDetails()
                    details.forEach { (group, threads) ->
                        Log.e("ThreadLeak", "$group: ${threads.size}个线程")
                    }
                    APMReporter.reportThreadLeak(currentCount, details)
                }

                if (growth > 20) {
                    Log.w("ThreadLeak", "线程数快速增长: +$growth (总计$currentCount)")
                }

                lastCount = currentCount
            }
        }
    }

    // 检查线程池配置是否合理
    fun auditThreadPools() {
        // 通过反射获取所有 ThreadPoolExecutor 实例
        // 检查是否有无界队列（LinkedBlockingQueue 无容量限制）
        // 无界队列 = 任务堆积 = 内存泄漏
    }
}
```
### JNI 层 GlobalRef 未释放
```
// JNI GlobalRef 泄漏：
// NewGlobalRef 创建全局引用，阻止 GC 回收 Java 对象
// 如果忘记 DeleteGlobalRef → Java 对象永远无法回收

// ❌ 泄漏
jobject gCallback = nullptr;

void setCallback(JNIEnv* env, jobject callback) {
    gCallback = env->NewGlobalRef(callback); // 创建全局引用
    // 忘记在不需要时调用 DeleteGlobalRef！
}

// ✅ 正确管理
class GlobalRefManager {
private:
    JNIEnv* env;
    jobject ref;

public:
    GlobalRefManager(JNIEnv* e, jobject obj) : env(e) {
        ref = env->NewGlobalRef(obj);
    }

    ~GlobalRefManager() {
        if (ref != nullptr) {
            env->DeleteGlobalRef(ref); // RAII：析构时自动释放
            ref = nullptr;
        }
    }

    jobject get() const { return ref; }
};

// ================================
// 监控 GlobalRef 数量
// ================================
extern "C" JNIEXPORT jint JNICALL
Java_com_xxx_JNIMonitor_getGlobalRefCount(JNIEnv* env, jobject) {
    // 通过 /proc/[pid]/status 获取 JNI 引用数
    // 或通过 JVM TI 接口获取
    jclass vmDebug = env->FindClass("dalvik/system/VMDebug");
    jmethodID method = env->GetStaticMethodID(
        vmDebug, "countInstancesOfClass",
        "(Ljava/lang/Class;Z)J"
    );
    // 返回全局引用数量
    return 0; // 实际实现需要 JVM TI
}

// Kotlin 层监控 JNI 引用
object JNIRefMonitor {

    fun getJNIRefCount(): Int {
        return try {
            val vmDebug = Class.forName("dalvik.system.VMDebug")
            val method = vmDebug.getDeclaredMethod("getGlobalRefCount")
            method.isAccessible = true
            method.invoke(null) as? Int ?: 0
        } catch (e: Exception) {
            // 通过 /proc/self/status 解析
            parseJNIRefsFromStatus()
        }
    }

    private fun parseJNIRefsFromStatus(): Int {
        return try {
            File("/proc/self/status").readLines()
                .find { it.startsWith("VmRSS:") }
                ?.split("\\s+".toRegex())
                ?.getOrNull(1)
                ?.toIntOrNull() ?: 0
        } catch (e: Exception) { 0 }
    }

    fun startMonitoring() {
        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(30_000)
                val count = getJNIRefCount()
                if (count > 1000) {
                    Log.e("JNIRefMonitor", "JNI GlobalRef 过多: $count")
                    APMReporter.reportJNIRefLeak(count)
                }
            }
        }
    }
}
```
## Native 内存
### malloc/free 泄漏
```
Native 内存泄漏检测工具链：

开发阶段：
├── AddressSanitizer (ASan)：检测内存越界/UAF/泄漏
│   编译时插桩，运行时检测，性能开销约 2x
├── LeakSanitizer (LSan)：专门检测内存泄漏
│   进程退出时报告所有未释放内存
└── Valgrind：全面的内存检测，但性能开销极大（10x~50x）

线上阶段：
├── malloc_debug：Android 系统内置，可动态开启
├── 自定义 malloc hook：重载 malloc/free 记录分配
└── 统计：通过 mallinfo() 获取统计信息

# 开启 malloc_debug（调试阶段）
adb shell setprop libc.debug.malloc.options backtrace
adb shell setprop libc.debug.malloc.program com.xxx.app
adb shell stop && adb shell start

# 获取 Native 内存泄漏报告
adb shell am dumpheap -n com.xxx.app /data/local/tmp/native_heap.txt
adb pull /data/local/tmp/native_heap.txt
```
### 内存映射（mmap）优化
```
// mmap 内存映射的使用场景与优化

object MmapOptimizer {

    // ================================
    // 场景一：大文件读取（避免全量加载到内存）
    // ================================
    fun readLargeFileWithMmap(file: File): ByteBuffer {
        val channel = FileInputStream(file).channel
        return channel.use {
            // mmap 映射：文件内容映射到虚拟内存
            // 实际读取时才触发缺页中断，按需加载
            // 不需要将整个文件读入内存！
            it.map(
                FileChannel.MapMode.READ_ONLY,
                0,
                file.length()
            )
        }
    }

    // ================================
    // 场景二：进程间共享内存
    // ================================
    fun createSharedMemory(name: String, size: Int): MemoryFile {
        // MemoryFile 基于 ashmem（Android 匿名共享内存）
        // 底层是 mmap(MAP_SHARED)
        // 多个进程可以映射同一块内存，实现零拷贝通信
        return MemoryFile(name, size)
    }

    // ================================
    // mmap 内存泄漏检测
    // ================================
    fun getMmapUsage(): Long {
        return try {
            // 解析 /proc/self/maps 统计匿名 mmap 大小
            File("/proc/self/maps").readLines()
                .filter { line ->
                    // 匿名映射：没有文件路径，且不是栈/堆
                    line.contains("00:00") &&
                    !line.contains("[stack]") &&
                    !line.contains("[heap]")
                }
                .sumOf { line ->
                    val parts = line.split(" ")[0].split("-")
                    if (parts.size == 2) {
                        val start = parts[0].toLongOrNull(16) ?: 0
                        val end = parts[1].toLongOrNull(16) ?: 0
                        end - start
                    } else 0L
                }
        } catch (e: Exception) { 0L }
    }
}
```
## 分级缓存
### LruCache
原理：  
```
LruCache（Least Recently Used）：
最近最少使用缓存，基于 LinkedHashMap 实现

LinkedHashMap 的关键特性：
├── 维护插入顺序 或 访问顺序
├── LruCache 使用访问顺序（accessOrder = true）
└── 每次 get/put 后，该节点移到链表尾部

数据结构：
HashMap（O(1)查找）+ 双向链表（维护访问顺序）

链表结构：
[最久未访问] ← → [较久未访问] ← → [最近访问]
     ↑                                   ↑
   淘汰端                              保留端

put 操作：
① 插入 HashMap
② 将节点移到链表尾部（最近访问）
③ 如果超过容量，移除链表头部节点（最久未访问）

get 操作：
① 从 HashMap 查找
② 找到后，将节点移到链表尾部
③ 返回值
```

实现：  
```
// LruCache 源码解析与自定义实现
class CustomLruCache<K, V>(private val maxSize: Int) {

    // LinkedHashMap：accessOrder=true 表示按访问顺序排序
    private val map = object : LinkedHashMap<K, V>(
        (maxSize / 0.75f + 1).toInt(),
        0.75f,
        true // accessOrder = true：按访问顺序
    ) {
        // 当 size > maxSize 时，自动移除最老的元素
        override fun removeEldestEntry(eldest: Map.Entry<K, V>): Boolean {
            return size > maxSize
        }
    }

    private var hitCount = 0
    private var missCount = 0
    private var putCount = 0
    private var evictionCount = 0

    @Synchronized
    fun get(key: K): V? {
        val value = map[key]
        if (value != null) hitCount++ else missCount++
        return value
    }

    @Synchronized
    fun put(key: K, value: V): V? {
        putCount++
        val previous = map.put(key, value)
        return previous
    }

    @Synchronized
    fun remove(key: K): V? = map.remove(key)

    @Synchronized
    fun trimToSize(maxSize: Int) {
        while (map.size > maxSize) {
            val eldest = map.entries.iterator().next()
            map.remove(eldest.key)
            evictionCount++
        }
    }

    fun hitRate(): Float = hitCount.toFloat() / (hitCount + missCount)
}

// ================================
// Bitmap LruCache（带大小计算）
// ================================
class BitmapLruCache(maxSizeBytes: Int) : LruCache<String, Bitmap>(maxSizeBytes) {

    // 重写 sizeOf：返回每个 Bitmap 的实际大小（字节）
    override fun sizeOf(key: String, bitmap: Bitmap): Int {
        return bitmap.byteCount
    }

    // 重写 entryRemoved：Bitmap 被淘汰时回收内存
    override fun entryRemoved(
        evicted: Boolean, key: String,
        oldValue: Bitmap, newValue: Bitmap?
    ) {
        if (evicted && !oldValue.isRecycled) {
            // 被淘汰的 Bitmap 放入 BitmapPool 复用
            // 而不是直接 recycle
            BitmapPool.put(oldValue)
        }
    }
}
```
### DiskLruCache
原理：  
```
DiskLruCache：磁盘级 LRU 缓存

核心文件：
├── journal 文件：记录所有缓存操作的日志
│   DIRTY key    ← 开始写入
│   CLEAN key size ← 写入完成
│   READ key     ← 读取
│   REMOVE key   ← 删除
└── 缓存文件：key.0, key.1（支持多个值）

journal 文件示例：
libcore.io.DiskLruCache
1                    ← 版本号
1                    ← appVersion
2                    ← valueCount（每个key对应几个文件）

DIRTY abc123
CLEAN abc123 1024    ← 文件大小1024字节
READ abc123
DIRTY def456
REMOVE def456        ← 写入失败，删除

工作原理：
① 写入：先写 DIRTY，写完写 CLEAN
   如果写入中断（App崩溃），重启后 DIRTY 条目被清理
② 读取：写 READ 日志，更新访问顺序
③ 淘汰：按 journal 中的顺序，删除最旧的 CLEAN 条目
④ 压缩：journal 过大时，重写 journal（只保留最终状态）
```

使用：  
```
// DiskLruCache 使用
class ImageDiskCache(cacheDir: File, maxSizeBytes: Long) {

    private val cache = DiskLruCache.open(
        cacheDir,
        1,           // appVersion
        1,           // valueCount（每个key一个文件）
        maxSizeBytes
    )

    fun put(key: String, bitmap: Bitmap) {
        val editor = cache.edit(sanitizeKey(key)) ?: return
        try {
            editor.newOutputStream(0).use { output ->
                bitmap.compress(Bitmap.CompressFormat.WEBP, 85, output)
            }
            editor.commit() // 写入成功
        } catch (e: Exception) {
            editor.abort() // 写入失败，回滚
        }
    }

    fun get(key: String): Bitmap? {
        val snapshot = cache.get(sanitizeKey(key)) ?: return null
        return try {
            snapshot.getInputStream(0).use { input ->
                BitmapFactory.decodeStream(input)
            }
        } finally {
            snapshot.close()
        }
    }

    // key 必须是 [a-z0-9_-]{1,120}
    private fun sanitizeKey(key: String): String {
        return key.lowercase()
            .replace(Regex("[^a-z0-9_-]"), "_")
            .take(120)
    }

    fun flush() = cache.flush()
    fun close() = cache.close()
    fun delete() = cache.delete()
}
```
### 三级缓存
```
// 完整的三级缓存架构
// 内存缓存（LruCache）→ 磁盘缓存（DiskLruCache）→ 网络

class ThreeLevelCache<V : Any>(
    memoryCacheSize: Int,
    diskCacheDir: File,
    diskCacheSize: Long,
    private val serializer: CacheSerializer<V>
) {
    // 一级：内存缓存（最快，但容量小，进程重启丢失）
    private val memoryCache = LruCache<String, V>(memoryCacheSize)

    // 二级：磁盘缓存（较快，容量大，持久化）
    private val diskCache = ImageDiskCache(diskCacheDir, diskCacheSize)

    // 三级：网络（最慢，但数据最新）
    // 由调用方提供

    suspend fun get(
        key: String,
        networkFetcher: suspend () -> V?
    ): V? {
        // 1. 查内存缓存
        memoryCache.get(key)?.let {
            return it // 命中内存缓存，直接返回
        }

        // 2. 查磁盘缓存
        diskCache.get(key)?.let { diskData ->
            val value = serializer.deserialize(diskData)
            // 回填内存缓存
            memoryCache.put(key, value)
            return value
        }

        // 3. 从网络获取
        val networkData = withContext(Dispatchers.IO) {
            networkFetcher()
        } ?: return null

        // 写入磁盘缓存和内存缓存
        withContext(Dispatchers.IO) {
            diskCache.put(key, serializer.serialize(networkData))
        }
        memoryCache.put(key, networkData)

        return networkData
    }

    fun invalidate(key: String) {
        memoryCache.remove(key)
        diskCache.remove(key)
    }

    // 缓存命中率统计
    fun getHitRate(): CacheHitRate {
        return CacheHitRate(
            memoryHitRate = memoryCache.hitRate(),
            // diskHitRate 需要自己统计
        )
    }
}

interface CacheSerializer<V> {
    fun serialize(value: V): Bitmap  // 示例：序列化为Bitmap存磁盘
    fun deserialize(bitmap: Bitmap): V
}
```
## LMK 机制
### Low Memory Killer 原理
```
LMK（Low Memory Killer）：
Android 的内存回收机制，在内存不足时杀死进程

演进历史：
├── 旧版 LMK（Android 9 以前）
│   ├── 内核驱动：/drivers/staging/android/lowmemorykiller.c
│   ├── 直接在内核层杀进程
│   └── 配置：/sys/module/lowmemorykiller/parameters/
│
└── LMKD（Android 9+）
    ├── 用户空间守护进程（lmkd）
    ├── 监听 PSI（Pressure Stall Information）内存压力
    ├── 根据内存压力级别决定杀哪个进程
    └── 更灵活，支持更多策略

LMKD 工作流程：
① 监听 /proc/pressure/memory（PSI 接口）
② 内存压力超过阈值 → 触发 LMKD
③ 读取所有进程的 oom_adj 值
④ 选择 oom_adj 最高的进程杀死
⑤ 如果内存仍然不足，继续杀下一个
```
### 进程优先级
```
// 通过生命周期管理优化 oom_adj

class ProcessPriorityManager(private val app: Application) {

    private var foregroundActivityCount = 0

    fun register() {
        app.registerActivityLifecycleCallbacks(
            object : Application.ActivityLifecycleCallbacks {

                override fun onActivityStarted(activity: Activity) {
                    foregroundActivityCount++
                    onForeground()
                }

                override fun onActivityStopped(activity: Activity) {
                    foregroundActivityCount--
                    if (foregroundActivityCount == 0) {
                        onBackground()
                    }
                }

                override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {}
                override fun onActivityResumed(activity: Activity) {}
                override fun onActivityPaused(activity: Activity) {}
                override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
                override fun onActivityDestroyed(activity: Activity) {}
            }
        )
    }

    private fun onForeground() {
        // App 进入前台：oom_adj 自动降低（系统管理）
        // 我们需要做的：
        // 恢复被后台释放的资源
        ResourceManager.restore()
    }

    private fun onBackground() {
        // App 进入后台：oom_adj 升高
        // 主动释放内存，降低被 LMK 杀死的概率
        // oom_adj 不变，但内存占用减少
        // → 系统有更多内存 → LMK 触发阈值更难达到
        BackgroundMemoryManager(app).shrink(
            BackgroundMemoryManager.ShrinkLevel.MODERATE
        )
    }
}
```
### oom_adj 管理
原理：  
```
oom_adj 值范围：-1000 ~ 1000
值越高 = 越容易被杀死

进程优先级与 oom_adj 对应关系：

┌──────────────────────────┬───────────┬────────────────────┐
│ 进程状态                  │ oom_adj   │ 说明               │
├──────────────────────────┼───────────┼────────────────────┤
│ Native进程（system_server）│ -1000    │ 永不被杀            │
│ 前台Activity              │ 0         │ 最高优先级          │
│ 前台Service               │ 1~2       │ 几乎不被杀          │
│ 可见但非焦点Activity       │ 100       │ 低内存才被杀        │
│ 后台Service               │ 200       │ 内存紧张时被杀      │
│ 后台Activity              │ 400~500   │ 较容易被杀          │
│ 空进程                    │ 900~1000  │ 最容易被杀          │
└──────────────────────────┴───────────┴────────────────────┘

查看进程 oom_adj：
adb shell cat /proc/[pid]/oom_adj
adb shell cat /proc/[pid]/oom_score_adj  # 新版本
```

检测：  
```
object OomAdjManager {

    // 读取当前进程的 oom_adj
    fun getCurrentOomAdj(): Int {
        return try {
            File("/proc/${Process.myPid()}/oom_adj")
                .readText().trim().toInt()
        } catch (e: Exception) {
            try {
                File("/proc/${Process.myPid()}/oom_score_adj")
                    .readText().trim().toInt()
            } catch (e2: Exception) { 0 }
        }
    }

    // 监控 oom_adj 变化（了解进程优先级变化）
    fun startMonitoring() {
        var lastAdj = getCurrentOomAdj()

        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(5000)
                val currentAdj = getCurrentOomAdj()

                if (currentAdj != lastAdj) {
                    Log.d("OomAdj", "oom_adj 变化: $lastAdj → $currentAdj")

                    // oom_adj 升高 = 优先级降低 = 更容易被杀
                    if (currentAdj > 400) {
                        Log.w("OomAdj", "进程优先级较低，可能被LMK杀死")
                    }
                    lastAdj = currentAdj
                }
            }
        }
    }

    // 提高进程优先级的合规方式
    fun improveProcessPriority(context: Context) {
        // 方式一：启动前台 Service（最有效）
        // oom_adj 降到 1~2

        // 方式二：保持 Activity 可见
        // oom_adj = 0（最高）

        // 方式三：使用 JobScheduler/WorkManager
        // 系统调度，不需要常驻进程
    }
}
```
## 内存监控
### Cuctom
原理：  
```
// 线上 OOM 监控方案

核心问题：
传统方案（LeakCanary）在线上不可用：
├── dump hprof 文件太大（几百MB）
├── dump 期间 App 暂停（STW），用户感知明显
└── 分析 hprof 耗时耗内存

解决方案：
├── 子进程 dump：fork 子进程执行 dump，主进程不暂停
├── hprof 裁剪：只保留必要信息，大幅减小文件
└── 增量上报：只上报新增的泄漏，不重复上报

fork 子进程 dump 原理：
主进程
  │
  ├── fork() → 子进程（内存是主进程的 CoW 副本）
  │              │
  │              └── 执行 Debug.dumpHprofData()
  │                  子进程 dump，主进程继续运行！
  │
  └── 继续正常运行（不暂停！）

CoW（Copy-on-Write）：
fork 后，父子进程共享物理内存页
只有当某个进程修改内存时，才复制该页
→ fork 瞬间完成，子进程 dump 不影响父进程
```

核心流程：  
```
// 核心流程实现（简化）
object KOOMCore {

    // 触发条件：Java 堆使用率超过阈值
    private const val HEAP_THRESHOLD = 0.85f

    fun startMonitoring() {
        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(10_000) // 每10秒检查

                val heapUsage = getHeapUsageRate()
                if (heapUsage > HEAP_THRESHOLD) {
                    triggerDump()
                }
            }
        }
    }

    private fun getHeapUsageRate(): Float {
        val runtime = Runtime.getRuntime()
        val used = runtime.totalMemory() - runtime.freeMemory()
        val max = runtime.maxMemory()
        return used.toFloat() / max
    }

    private fun triggerDump() {
        // 方式一：直接 dump（会暂停 App）
        // val hprofFile = File(cacheDir, "heap_${System.currentTimeMillis()}.hprof")
        // Debug.dumpHprofData(hprofFile.absolutePath)

        // 方式二：fork 子进程 dump（KOOM 方案）
        forkAndDump()
    }

    private fun forkAndDump() {
        // 通过 JNI 调用 fork()
        val pid = NativeFork.fork()

        if (pid == 0) {
            // 子进程：执行 dump
            try {
                val hprofFile = File(
                    AppContext.get().cacheDir,
                    "heap_${System.currentTimeMillis()}.hprof"
                )
                Debug.dumpHprofData(hprofFile.absolutePath)
                // dump 完成后，裁剪 hprof
                HprofStripper.strip(hprofFile)
                // 上报
                uploadHprof(hprofFile)
            } finally {
                // 子进程退出
                System.exit(0)
            }
        }
        // 父进程：继续正常运行
    }
}
```
### LeakCanary
原理：  
```

// ① Hook Activity/Fragment 生命周期，检测当 Activity、Fragment 销毁时实例是否回收
// ② 对象销毁时（onDestroy），用 WeakReference 包装（销毁的实例传给 RefWatcher（持有它们的弱引用））
// ③ 保留实例（Retained Instance）数量达到阈值会进行堆转储，数据放进 hprof 文件（APP 可见阈值为 5，不可见为 1）
// ④ 手动触发 GC
// ⑤ GC 后检查 WeakReference 是否被清除
//    WeakReference.get() == null → 已回收，无泄漏
//    WeakReference.get() != null → 未回收，可能泄漏
// ⑥ 确认泄漏：dump heap，解析 hprof 文件，找出导致 GC 无法回收实例的引用链，就是泄漏踪迹（Leak Trace，最短强引用路径，GC Roots 到实例的路径）。  

// 简化实现原理：
class LeakDetector {

    private val watchedObjects = mutableMapOf<String, WeakReference<Any>>()
    private val retainedKeys = mutableSetOf<String>()

    fun watch(obj: Any, description: String) {
        val key = UUID.randomUUID().toString()
        watchedObjects[key] = WeakReference(obj)

        // 5秒后检查是否已回收
        Handler(Looper.getMainLooper()).postDelayed({
            checkRetained(key, description)
        }, 5000)
    }

    private fun checkRetained(key: String, description: String) {
        // 触发 GC
        Runtime.getRuntime().gc()
        System.runFinalization()
        Runtime.getRuntime().gc()

        // 检查是否回收
        if (watchedObjects[key]?.get() != null) {
            // 对象还在！可能泄漏
            retainedKeys.add(key)
            dumpHeapAndAnalyze(key, description)
        }
    }

    private fun dumpHeapAndAnalyze(key: String, description: String) {
        // dump hprof 文件
        val heapDumpFile = File(cacheDir, "leak_${key}.hprof")
        Debug.dumpHprofData(heapDumpFile.absolutePath)

        // 在子进程分析 hprof（避免影响主进程）
        // 找到从 GC Root 到泄漏对象的最短引用路径
        analyzeHeapDump(heapDumpFile, key, description)
    }
}
```

监听系统内存状态：  
ComponentCallback2：在 Activity 中实现 ComponentCallback2 接口获取系统内存的相关事件, 在 onTrimMemory(level)  回调针对不同事件做不同释放内存操作  
ActivityManager.getMemoryInfo()：  
返回一个 ActivityManager.MemoryInfo 对象，包含系统当前内存状态（可用内存、总内存、低杀内存阈值， lowMemory 布尔值判断是否处于低内存态）  
### Hprof 裁剪上报
```
// hprof 文件结构：
// ├── Header（文件头）
// ├── Records（记录列表）
//     ├── STRING（字符串）
//     ├── LOAD_CLASS（类信息）
//     ├── HEAP_DUMP_SEGMENT（堆数据）
//     │   ├── CLASS_DUMP（类定义）
//     │   ├── INSTANCE_DUMP（对象实例）← 包含对象字段值
//     │   ├── OBJECT_ARRAY_DUMP（对象数组）
//     │   └── PRIMITIVE_ARRAY_DUMP（基本类型数组）← 包含原始数据
//     └── ...

// 裁剪策略：
// 泄漏分析只需要：对象引用关系
// 不需要：对象的具体字段值、数组的原始数据
// → 裁剪掉 PRIMITIVE_ARRAY_DUMP 中的原始数据
// → 裁剪掉 INSTANCE_DUMP 中的字段值（保留引用）
// → 文件大小可以减少 50%~80%

object HprofStripper {

    fun strip(hprofFile: File): File {
        val strippedFile = File(
            hprofFile.parent,
            "${hprofFile.nameWithoutExtension}_stripped.hprof"
        )

        hprofFile.inputStream().buffered().use { input ->
            strippedFile.outputStream().buffered().use { output ->
                // 读取并验证文件头
                val header = readHeader(input)
                writeHeader(output, header)

                // 逐条处理 Record
                while (input.available() > 0) {
                    val recordType = input.read()
                    val timestamp = readInt(input)
                    val length = readInt(input)

                    when (recordType) {
                        HEAP_DUMP, HEAP_DUMP_SEGMENT -> {
                            // 处理堆数据，裁剪不必要的内容
                            val heapData = stripHeapData(input, length)
                            writeRecord(output, recordType, timestamp, heapData)
                        }
                        else -> {
                            // 其他 Record 直接复制
                            val data = input.readNBytes(length)
                            writeRecord(output, recordType, timestamp, data)
                        }
                    }
                }
            }
        }

        return strippedFile
    }

    private fun stripHeapData(input: InputStream, length: Int): ByteArray {
        val output = ByteArrayOutputStream()

        var remaining = length
        while (remaining > 0) {
            val subRecordType = input.read()
            remaining--

            when (subRecordType) {
                PRIMITIVE_ARRAY_DUMP -> {
                    // 裁剪：只保留数组头信息，丢弃原始数据
                    val serialNumber = readLong(input); remaining -= 8
                    val stackSerial = readInt(input); remaining -= 4
                    val numElements = readInt(input); remaining -= 4
                    val elementType = input.read(); remaining--

                    val elementSize = getPrimitiveSize(elementType)
                    val dataSize = numElements * elementSize
                    input.skip(dataSize.toLong()) // 跳过原始数据
                    remaining -= dataSize

                    // 只写入头信息，数组长度置为0
                    writePrimitiveArrayHeader(
                        output, serialNumber, stackSerial, 0, elementType
                    )
                }

                INSTANCE_DUMP -> {
                    // 保留实例引用，裁剪基本类型字段值
                    // 实现略...
                    copyInstanceDump(input, output)
                    remaining -= getInstanceDumpSize(input)
                }

                else -> {
                    val size = getSubRecordSize(subRecordType, input)
                    val data = input.readNBytes(size)
                    output.write(subRecordType)
                    output.write(data)
                    remaining -= size
                }
            }
        }

        return output.toByteArray()
    }

    private fun getPrimitiveSize(type: Int) = when (type) {
        2 -> 4; 4 -> 1; 5 -> 2; 6 -> 4
        7 -> 8; 8 -> 1; 9 -> 2; 10 -> 4; 11 -> 8
        else -> 0
    }

    // Record 类型常量
    private const val HEAP_DUMP = 0x1C
    private const val HEAP_DUMP_SEGMENT = 0x1D
    private const val PRIMITIVE_ARRAY_DUMP = 0x23
    private const val INSTANCE_DUMP = 0x21
}
```
### 内存水位线告警
```
// 完整的内存水位线告警体系

object MemoryWaterlineMonitor {

    // 水位线配置
    data class Waterline(
        val name: String,
        val heapUsageRate: Float,   // Java堆使用率
        val nativeHeapMB: Long,     // Native堆大小
        val action: WaterlineAction
    )

    enum class WaterlineAction {
        LOG,           // 只记录日志
        REPORT,        // 上报监控
        TRIM,          // 主动释放缓存
        DUMP_HPROF,    // dump hprof（最高级别）
        FORCE_GC       // 强制GC
    }

    private val waterlines = listOf(
        Waterline("low", 0.60f, 200, WaterlineAction.LOG),
        Waterline("medium", 0.75f, 300, WaterlineAction.REPORT),
        Waterline("high", 0.85f, 400, WaterlineAction.TRIM),
        Waterline("critical", 0.92f, 500, WaterlineAction.DUMP_HPROF)
    )

    private var lastTriggeredLevel = -1

    fun startMonitoring(context: Context) {
        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(15_000) // 每15秒检查

                val heapUsage = getHeapUsageRate()
                val nativeHeap = Debug.getNativeHeapAllocatedSize() / 1024 / 1024

                // 找到当前触发的最高水位线
                val triggeredIndex = waterlines.indexOfLast { waterline ->
                    heapUsage >= waterline.heapUsageRate ||
                    nativeHeap >= waterline.nativeHeapMB
                }

                if (triggeredIndex > lastTriggeredLevel) {
                    val waterline = waterlines[triggeredIndex]
                    handleWaterline(context, waterline, heapUsage, nativeHeap)
                    lastTriggeredLevel = triggeredIndex
                } else if (triggeredIndex < lastTriggeredLevel) {
                    // 内存压力降低，重置水位线
                    lastTriggeredLevel = triggeredIndex
                }
            }
        }
    }

    private fun handleWaterline(
        context: Context,
        waterline: Waterline,
        heapUsage: Float,
        nativeHeapMB: Long
    ) {
        Log.w("MemoryWaterline",
            "触发水位线[${waterline.name}]: " +
            "Java堆=${(heapUsage * 100).toInt()}%, " +
            "Native堆=${nativeHeapMB}MB"
        )

        when (waterline.action) {
            WaterlineAction.LOG -> { /* 已记录日志 */ }

            WaterlineAction.REPORT -> {
                APMReporter.reportMemoryWaterline(
                    level = waterline.name,
                    heapUsage = heapUsage,
                    nativeHeapMB = nativeHeapMB
                )
            }

            WaterlineAction.TRIM -> {
                APMReporter.reportMemoryWaterline(waterline.name, heapUsage, nativeHeapMB)
                // 主动释放缓存
                Glide.get(context).clearMemory()
                DataCache.clearAll()
                System.gc()
            }

            WaterlineAction.DUMP_HPROF -> {
                APMReporter.reportMemoryWaterline(waterline.name, heapUsage, nativeHeapMB)
                // 触发 hprof dump（fork子进程，不影响主进程）
                KOOMCore.triggerDump()
            }

            WaterlineAction.FORCE_GC -> {
                repeat(3) {
                    System.gc()
                    System.runFinalization()
                }
            }
        }
    }

    private fun getHeapUsageRate(): Float {
        val runtime = Runtime.getRuntime()
        val used = runtime.totalMemory() - runtime.freeMemory()
        return used.toFloat() / runtime.maxMemory()
    }
}
```
## 小妙招
### 自动装箱  
尽量使用基本数据类型来代替封装数据类型，int 比 Integer 要更加有效，其它数据类型也是一样。自动装箱的核心是把基础数据类型转换成对应的复杂类型。自动装箱转化时，会产生一个新的对象，这样就会产生更多的内存和性能开销。如 int 只占4字节，而 Integer 对象有16字节，特别是 HashMap 这类容器，进行增、删、改、查操作时，都会产生大量的自动装箱操作。   
### 择优数据结构  
1.SparseArray与ArrayMap    
Android 自身还提供了一系列优化过后的数据集合工具类，如 SparseArray、SparseBooleanArray、LongSparseArray，使用这些 API 可以让我们的程序更加高效。  
HashMap 工具类会相对比较低效，因为它需要为每一个键值对都提供一个对象入口，而 SparseArray 就避免掉了基本数据类型转换成对象数据类型的时间。  
ArrayMap 提供了和 HashMap 一样的功能，但避免了过多的内存开销，方法是使用两个小数组，而不是一个大数组。并且 ArrayMap 在内存上是连续不间断的。总体来说，在 ArrayMap 中执行插入或者删除操作时，从性能角度上看，比 HashMap 还要更差一些，但如果只涉及很小的对象数，比如1000以下，就不需要担心这个问题了。因为此时 ArrayMap 不会分配过大的数组。  
2.避免使用枚举类型    
枚举最大的优点是类型安全，但在 Android 平台上，枚举的内存开销是直接定义常量的三倍以上。  
每一个枚举值都是一个单例对象，在使用它时会增加额外的内存消耗，所以枚举相比与 Integer 和 String 会占用更多的内存。大量使用 Enum 会增加 DEX 文件的大小，会造成运行时更多的 IO 开销，使我们的应用需要更多的空间。特别是分 Dex 多的大型 App，枚举的初始化很容易导致 ANR。
### 避免对象创建
1.我们可以在字符串拼接的时候尽量少用 +=，多使用 StringBuffer，StringBuilder。    
2.不要在 onMeause、onLayout、onDraw 中去刷新 UI（requestLayout）。    
3.自定义 View 的onLayout、onDraw、列表遍历等频繁调用的方法里创建对象。这些方法会被多次调用，在其内部创建对象会导致系统频繁申请存储空间并触发 GC，导致内存抖动，严重会导致 OOM。
### 慎用 Service
如果应用程序当中需要使用 Service 来执行后台任务，一定注意只有当任务正在执行的时候才让 Service 运行起来。另外，当任务执行完之后去停止 Service 时，要小心 Service 停止失败导致内存泄漏的情况。  
启动一个 Service 时，系统会倾向于将这个 Service 所依赖的进程进行保留，这样就会导致这个进程变得非常消耗内存。并且系统可以在 LruCache 当中缓存的进程数量也会减少，导致切换应用程序的时候耗费更多性能，严重的话，甚至有可能会导致崩溃。因为系统在内存非常吃紧的时候可能已无法维护所有正在运行的 Service 所依赖的进程了。  
为了能够控制 Service 的生命周期，Android 官方推荐的最佳解决方案就是使用 IntentService，这种 Service 的最大特点就是当后台任务执行结束后会自动停止，从而极大程度上避免了 Service 内存泄漏的可能性。  
通常避免使用持久性服务，它们会持续请求使用可用内存。建议采用 WorkManager等替代实现方式。  
