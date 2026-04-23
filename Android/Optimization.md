# 启动优化	     
## 冷启动
冷启动：进程不存在，应用自设备启动后或系统终止应用后首次启动，耗时最多，是衡量启动耗时的标准  
### 完整流程
![image](https://github.com/user-attachments/assets/5ec9b45f-770c-4560-be5b-c8be4af1e3a5)    

用户点击图标  
     │  
     ▼  
① Launcher 进程  
   └─ startActivity() → AMS (Binder 跨进程)  
     │  
     ▼  
② system_server 进程  
   └─ AMS 处理请求  
   └─ 检查进程是否存在 → 不存在  
   └─ 通知 Zygote fork 进程  
     │  
     ▼  
③ Zygote fork 新进程  
   └─ 复制 Zygote 预加载资源（类、资源、so）  
   └─ 创建 App 进程 ← APP 进程由 zygote 进程 fork 出来后会执行 ActivityThread 的 main 方法，该方法最终触发执行 bindApplication()，这也是 Application 阶段的起点  
     │  
     ▼  
④ App 进程初始化  
   └─ ActivityThread.main()  
   └─ 创建 Application  
   └─ attachBaseContext() ← 在应用中最早能触达到的生命周期，本阶段也是最早的预加载时机    
   └─ installContentProviders() ← 很多三方 sdk 借助该时机来做初始化操作  
   └─ Application.onCreate() ← 这里有很多三方库和业务的初始化操作，是 Application 阶段的末尾
     │  
     ▼  
⑤ Activity 创建    
   └─ Activity.onCreate()  
   └─ setContentView() → inflate 布局  
   └─ onCreate() → onStart() → onResume() ← Activity 阶段最关键的生命周期是 onCreate()，这个阶段中包含了大量的 UI 构建、首页相关业务、对象初始化等耗时任务  
     │  
     ▼  
⑥ 首帧渲染  
   └─ measure → layout → draw  
   └─ RenderThread 提交 GPU  
   └─ SurfaceFlinger 合成上屏  
     │  
     ▼  
⑦ 用户看到第一帧 ← 冷启动结束  
### 系统指标
adb 测量冷启动时间：  
adb shell am start -W -n com.xxx/.MainActivity  
输出结果：  
ThisTime:   xxx ms  ← 最后一个 Activity 启动耗时  
TotalTime:  xxx ms  ← 整个启动过程耗时（重点关注）  
WaitTime:   xxx ms  ← 包含 Launcher 响应时间  
### 埋点
我们在应用中能触达到的 attachBaseContext 阶段，这是最早的预加载时机：    
可以把这个方法的回调时间当作启动开始时间，因为 attachBaseContext() 是应用进程的第一个生命周期，但是准确来说，应用的启动时间包括进程创建，应该在冷启动时用户点击应用 Icon 开始计算。  
```
override fun attachBaseContext(base: Context) {  
     super.attachBaseContext(base)  
     appStartTime = SystemClock.elapsedRealtime()  
}
```
  
Activity#onWindowFocusChanged() 这个方法的调用时机是用户与 Activity 交互的最佳时间点，当 Activity 中的 View 测量绘制完成之后会回调 Activity 的 onWindowFocusChanged() 方法，可以选择它来当作时间结束的时间点：   
```
override fun onWindowFocusChanged(hasFocus: Boolean) {  
   super.onWindowFocusChanged(hasFocus)  
      if (hasFocus) {  
          val cost = SystemClock.elapsedRealtime() - MyApplication.appStartTime  
          // 上报启动耗时  
          Logger.report("cold_start_cost", cost)  
     }  
}
```

但是大部分数据是通过请求接口回来之后，才能填充页面才能显示出来，当执行到 onWindowFocusChanged() 的时候，请求数据还没完成，页面上依旧是没有数据的，用户仅仅可以交互写死在 XML 布局当中的视图，更多的内容还是不可见，不可交互的   
所以结束时间点通常选择在列表上面第一个 itemView 的 perDrawCallback() 方法的回调时机当作时间结束点，也就是首帧时间。当列表上面第一个 itemView 被显示出来的时候说明数据加载已经完成。页面上的 View 已经填充了数据，并且开始重新渲染了。此时用户是可以交互的，这个才是比较有意义的时间节点，可以通过监听 
```
itemView.viewTreeObserver.addOnPreDrawListener(object :    
 ViewTreeObserver.OnPreDrawListener {  
    override fun onPreDraw(): Boolean {  
        return false  
    }  
})
```

或者通过 vm 监听：  
```
viewModel.dataReady.observe(this) { ready ->  
    if (ready) {  
        reportFullyDrawn() // 上报真实首屏时间  
    }  
}
```
### 优化策略
#### 页面合并
![image](https://github.com/user-attachments/assets/09208ec8-740e-4656-9fc0-a2d092fe5943)    
利用读取开屏信息和等待广告的时间，做一些与 Activity 强关联的并发任务，比如异步 View 预加载，数据加载等  
![image](https://github.com/user-attachments/assets/2e245352-19fd-4d3f-a980-2107d941acb7)  
MainAcitvity 的 launch mode 需要设置为 singleTop，否则会出现 App 从后台进前台，非 MainActivity 走生命周期的现象  
跳转到 MainAcitvity 之后，其他二级页面需要全部都关闭掉，站内跳转到 MainActivity 则附带 CLEAR_TOP | NEW_TASK 的标记 
#### Zygote 阶段
APP 开发者优化空间有限：  
系统启动时 Zygote 预加载了：  
1.常用 Framework 类（/system/etc/preloaded-classes：几千个）  
2.系统资源（framework-res.apk：主题、图片）  
3.常用 so 库  

App 启动时 fork Zygote：  
1.子进程直接继承父进程内存页（Copy-on-Write）  
2.已预加载的类/资源 → 零成本复用  
3.App 自己的类/资源 → 需要重新加载 ← 这里是瓶颈  
  
可优化点：  
减少 App 自身在此阶段的依赖（需要重新加载的自己的资源/类）：
三方 so 库：  
System.loadLibrary("xxx")  
每个 so 的加载 = 文件读取 + 符号解析 + 重定位  
大型 so (如 Flutter engine ~8MB) 耗时明显  

三方 SDK 的类：  
不在 Zygote 预加载列表中  
首次访问触发类加载 + 验证 + 初始化  

App 自定义资源：  
非系统资源，Zygote 不会预加载  

JNI_OnLoad 调用链：  
so 加载后自动触发，可能有大量初始化逻辑  
  
所以避免过早加载大型 so 库、SDK等：  
1.按需加载：用到时再加载，例如进入某页面前、使用某工具前... ...  
2.异步加载：子线程提前加载非主线程依赖的so
#### Application onCreate 优化
##### 视觉
原生逻辑是冷启动过程中会创建一个空白的 Window，等到应用创建第一个 Activity 后才将该 Window 替换    
如果 Application 或 Activity 启动的过程太慢，导致系统的 BackgroundWindow 没有及时被替换，就会出现启动时白屏或黑屏的情况      
可以设置预览图、自定义 Theme 等优化用户等待加载时的观感
#### ContentProvider 优化
很多 SDK 通过 ContentProvider 自动初始化，AndroidManifest 合并后，这些都会在 Application.onCreate 之前执行，系统对每个 ContentProvider：  
1.类加载：加载其所在 Dex/类  
2.对象实例化：实例化 ContentProvider 对象  
3.Binder 注册到 AMS：调用 ContentProvider.attachInfo()  
4.调用 ContentProvider.onCreate()  

1.adb shell dumpsys activity providers com.xxx 查看总数量。  
2.Jetpack App Startup 合并 ContentProvider，所有 SDK 初始化合并到一个 ContentProvider 中  
<img width="543" height="254" alt="image" src="https://github.com/user-attachments/assets/152ef4fc-466f-4ded-89d0-6556aedf82a7" />  

只有 1 个 ContentProvider（InitializationProvider）：  
1.类加载：1次  
2.对象实例化：1次    
3.Binder 注册：1次  
4.在这1个CP的onCreate中，按序调用各 Initializer    

但 SDK 的初始化代码该跑还是跑，节省的是ContentProvider 本身的创建开销、节省的是ContentProvider类加载开销、多次Binder IPC开销  
此外，可以在 Initializer 中实现懒加载/异步（原生CP做不到）：  
```
// App Startup 支持异步初始化（这才是更大的价值）
class HeavySDKInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        // 可以在这里选择异步执行
        CoroutineScope(Dispatchers.IO).launch {
            HeavySDK.init(context) // 不阻塞 CP 的 onCreate
        }
    }
    override fun dependencies() = emptyList<...>()
}
```

#### MultiDex 优化
异步加载 Secondary Dex：  
在主线程加载 main.Dex(主 Dex，包含启动必需类) ,开启子线程异步加载 Secondary Dex,若主线程需要 Secondary Dex 中的类，则等待加载完成，Android 5.0 以上此缺陷几乎可以忽略了，价值不大。

dex2oat：  
Dalvik Executable to Optimized ART file：  
1.将 DEX 字节码预编译为本地机器码。应用程序不再完全依赖解释器执行，而是直接运行机器码，从而提高代码运行速度。  
2.将输入的 dex 文件预编译为 .oat 的本地镜像文件（），加快应用程序的启动速度。    

触发场景：  
1.OTA升级：发布Baseline Profile 让系统触发 dex2oat(Android 11 开始 Google 禁止应用侧触发 dex2oat)。    
2.应用首次安装：系统自动调用 dex2oat 对 DEX 字节码进行预编译，提高后续使用时的流畅度。      
3.系统空闲：当手机充电且空闲时，后台运行 dex2oat 进行更深度的优化。  

```
// Baseline Profile
// 在 macrobenchmark 测试中生成
@ExperimentalBaselineProfilesApi
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(
        packageName = "com.xxx.app"
    ) {
        // 描述启动路径，让编译器提前AOT编译这些代码
        pressHome()
        startActivityAndWait()
        // 模拟用户启动后的关键操作路径
        device.findObject(By.text("首页")).click()
    }
}
// 生成的 baseline-prof.txt 打包进 APK
// 安装时系统会对这些类/方法做 AOT 编译
// 首次启动即享受 AOT 性能
```
## 热启动/温启动
1.温启动：进程存在，但是 Activity 需要重新创建（Activity 被销毁：如退出应用后又重新启动应用。进程可能还在运行/系统因内存不足等原因将应用回收，然后用户又重新启动这个应用）   
2.热启动：进程存在，Activity 在后台，只需要走 onStart，但是如果一些内存为响应内存整理事件（如 onTrimMemory()）而被完全清除，则需要为了响应热启动而重新创建相应的对象，热启动显示的屏幕上行为和冷启动场景相同。系统进程显示空白屏幕，直到应用完成 Activity 呈现  
### Activity 复用
### 进程保活策略
## 启动任务编排
### 有向无环图（DAG）任务依赖
根据依赖关系排序生成有向无环图，最后由优先级执行 task。任务可以分为（init 前、后，空闲，子、主线程等 task）       
![image](https://github.com/user-attachments/assets/850fa3c3-02b0-456f-90d4-96fd73ba98c6)    

例如：  
<img width="477" height="485" alt="image" src="https://github.com/user-attachments/assets/4c6d27c3-4e52-49ed-b0a5-2d75232e6095" />

```
# 任务基类
abstract class StartupTask {
    abstract val id: String
    open val dependencies: List<String> = emptyList()
    open val runOnMainThread: Boolean = false
    open val priority: Int = 0
    abstract suspend fun run(context: Context)
}

调度器，拓扑排序检测循环依赖：  
```
class StartupScheduler(private val context: Context) {

    private val taskMap = mutableMapOf<String, StartupTask>()

    fun register(vararg tasks: StartupTask): StartupScheduler {
        tasks.forEach { taskMap[it.id] = it }
        return this
    }

    // 注册完成后，start() 前先做检测
    suspend fun start() {
        // 第一步：检测所有问题，有问题直接抛异常
        validateGraph()
        
        // 第二步：正常调度
        schedule()
    }

    // 图合法性全量检测
    private fun validateGraph() {
        // 检测1：依赖的任务是否都已注册
        checkMissingDependencies()
        
        // 检测2：是否存在循环依赖（Kahn拓扑排序）
        checkCyclicDependencies()
    }

    // 检测1：缺失依赖
    private fun checkMissingDependencies() {
        val errors = mutableListOf<String>()
        
        taskMap.values.forEach { task ->
            task.dependencies.forEach { depId ->
                if (!taskMap.containsKey(depId)) {
                    errors.add(
                        "任务 [${task.id}] 依赖 [$depId]，" +
                        "但 [$depId] 未注册！"
                    )
                }
            }
        }
        
        if (errors.isNotEmpty()) {
            throw IllegalStateException(
                "启动任务依赖缺失：\n${errors.joinToString("\n")}"
            )
        }
    }

    // 检测2：循环依赖（Kahn 算法）
    private fun checkCyclicDependencies() {
        // 构建入度表
        val inDegree = mutableMapOf<String, Int>()
        taskMap.keys.forEach { inDegree[it] = 0 }
        
        taskMap.values.forEach { task ->
            task.dependencies.forEach { depId ->
                // task 依赖 depId
                // 即 depId → task 这条边
                // task 的入度 +1
                inDegree[task.id] = (inDegree[task.id] ?: 0) + 1
            }
        }

        // 将所有入度为 0 的节点加入队列
        val queue: ArrayDeque<String> = ArrayDeque()
        inDegree.forEach { (id, degree) ->
            if (degree == 0) queue.add(id)
        }

        // Kahn 算法主循环
        var processedCount = 0
        val topoOrder = mutableListOf<String>() // 拓扑顺序（调试用）

        while (queue.isNotEmpty()) {
            val current = queue.removeFirst()
            topoOrder.add(current)
            processedCount++

            // 找到所有依赖 current 的任务
            // 将它们的入度 -1
            taskMap.values
                .filter { it.dependencies.contains(current) }
                .forEach { dependent ->
                    val newDegree = (inDegree[dependent.id] ?: 0) - 1
                    inDegree[dependent.id] = newDegree
                    if (newDegree == 0) {
                        queue.add(dependent.id)
                    }
                }
        }

        // 如果处理的节点数 < 总节点数
        // 说明有节点的入度永远不为0 → 存在循环依赖
        if (processedCount < taskMap.size) {
            // 找出参与循环的节点
            val cycleNodes = inDegree
                .filter { it.value > 0 }
                .keys

            // 还原循环路径
            val cyclePath = findCyclePath(cycleNodes)

            throw IllegalStateException(
                """
                ❌ 启动任务存在循环依赖！
                参与循环的任务：$cycleNodes
                循环路径：$cyclePath
                请检查任务依赖声明！
                """.trimIndent()
            )
        }

        // 打印合法的拓扑顺序（方便调试）
        println("✅ 依赖图检测通过，拓扑顺序：$topoOrder")
    }

    // 还原具体循环路径（DFS）
    // 方便开发者定位问题
    private fun findCyclePath(cycleNodes: Set<String>): String {
        // 只在循环节点中做 DFS
        val visited = mutableSetOf<String>()
        val path = mutableListOf<String>()

        fun dfs(nodeId: String): Boolean {
            if (nodeId in path) {
                // 找到环，截取环的部分
                val cycleStart = path.indexOf(nodeId)
                val cycle = path.subList(cycleStart, path.size) + nodeId
                path.clear()
                path.addAll(cycle)
                return true
            }
            if (nodeId in visited) return false

            path.add(nodeId)
            visited.add(nodeId)

            val task = taskMap[nodeId] ?: return false
            for (depId in task.dependencies) {
                if (depId in cycleNodes && dfs(depId)) {
                    return true
                }
            }

            path.removeLast()
            return false
        }

        cycleNodes.forEach { node ->
            if (dfs(node)) return path.joinToString(" → ")
        }

        return cycleNodes.toString()
    }

    private suspend fun schedule() { /* 正常调度逻辑 */ }
}
```
### IdleHandler
可以在 MessageQueue 空闲的时候执行任务  
![image](https://github.com/user-attachments/assets/9f4a2fbb-11f5-4495-bd93-c6eee56e9e0d)  
1.在启动的过程中，可以借助 idleHandler 来做一些延迟加载的事情， 比如在启动过程中 Activity 的 onCreate 里面 addIdleHandler，这样在 Message 空闲的时候，可以执行这个任务  
2.进行启动时间统计：比如在页面完全加载之后，调用 activity.reportFullyDrawn 来告知系统这个 Activity 已经完全加载，用户可以使用了，比如在主页的 List 加载完成后，调用 activity.reportFullyDrawn  
### 异步并行初始化
#### 异步 init
![image](https://github.com/user-attachments/assets/00093f59-fd7c-44fb-9fdc-fc4f116f56c2)   
将 init 分为多个子 task 放在子 thread（提交到线程池），task 之间无依赖关系     
此外，若 init 在 Application.onCreate 结束前完成，可用 CountDownLatch 等待 task 执行  
![image](https://github.com/user-attachments/assets/1d9c5cb5-9197-4da4-9843-7efb5e08b8f2)    
#### 延迟 init
优先级不高，可在启动完成后执行的 init task延迟到启动完成后执行（如 handler.postdelay），但延迟后的页面加载完成可能造成手势失效（如滑动）。  
此外，init launcher 利用 IdleHandler 实现主线程空闲时执行任务，不影响用户操作。    
![image](https://github.com/user-attachments/assets/4b555acb-2c7d-4545-8327-8a93b4b853a6)
## 类加载优化
1.首次安装：JIT 编译，类需要解释执行  
2.类加载顺序不优化 → 缺页中断频繁  
3.类验证（Class Verify）耗时（但建议保留，确保类字节码合法、类型安全、文件完整等）
### 类预加载
一个类的加载耗时不多，但是在几百上千的基数上，也会延迟启动时间，将进入首页的 class 对象，使用线程池提前预加载进来，在类下次使用时则可以直接使用而不需要触发类加载   
Class.forName() 只加载类本身以及静态变量的引用类，new 类实例可以额外加载类成员变量的引用类  
确定哪些类需要提前加载，可以切换系统的 ClassLoader，在自定义 ClassLoader 里面每个类 load 时加一个 log，在项目中运行一次，这样就可以拿到所有 log，也就是需要异步加载的类  
### Dex 重排（Facebook ReDex / 字节码插桩）
将启动相关类集中到同一 Dex 页，减少缺页中断，主要影响 bindApplication 阶段。  
#### 缺页中断
Dex 文件存储在磁盘上，按类的编译顺序排列，为了节省物理内存，通常不会一次性将整个 DEX 文件读取到内存中，使用 mmap 系统调用将 DEX 文件映射到虚拟地址空间。  

默认 Dex 布局（按编译顺序）：  
[HomeActivity][UserModel][PaymentSDK][SettingActivity]  
[NetworkUtils][DatabaseHelper][SplashActivity][......]  

启动时需要的类：  
SplashActivity、HomeActivity、NetworkUtils...  

这些类分散在 Dex 文件的不同位置：    
→ 当 CPU 访问 DEX 文件中尚未加载到物理内存中的类或代码时，会触发内核的缺页中断（Page Fault）  
→ 内核此时会暂停应用程序，向磁盘（Flash）发起真正的 I/O 操作，将数据从磁盘加载到物理内存  
→ 每次缺页中断约 1ms，几百次 = 几百ms 损耗  

重拍后，按启动顺序：  
[SplashActivity][HomeActivity][NetworkUtils][UserModel]  
[...启动相关类集中在前几个内存页...]  
[PaymentSDK][SettingActivity][...非启动类...]  
#### 实现路径
1.收集启动阶段类加载顺序：  
```
// 通过 Hook ClassLoader 记录启动期间加载的类
class TraceClassLoader(parent: ClassLoader) : ClassLoader(parent) {
    
    private val loadedClasses = mutableListOf<String>()
    
    override fun loadClass(name: String, resolve: Boolean): Class<*> {
        // 记录类加载顺序
        if (isStartupPhase) {
            loadedClasses.add(name)
        }
        return super.loadClass(name, resolve)
    }
    
    fun getStartupClassOrder(): List<String> = loadedClasses
}

// 启动结束后，将顺序写入文件
// startup_classes.txt
// com.xxx.SplashActivity
// com.xxx.HomeActivity  
// com.xxx.network.OkHttpManager
// ...
```

2.根据顺序重排 Dex（编译期）:  
```
// build.gradle 中接入 ReDex 或自定义插件
android {
    buildTypes {
        release {
            // 方案A：Facebook ReDex
            // ReDex 读取 startup_classes.txt
            // 重新排列 Dex 中类的顺序
        }
    }
}

// 方案B：自定义 Gradle Transform（字节码插桩阶段重排）
class DexReorderTransform : Transform() {
    override fun transform(invocation: TransformInvocation) {
        // 1. 读取启动类顺序文件
        val startupOrder = readStartupClassOrder()
        
        // 2. 将启动相关类排到 Dex 文件前面
        reorderDexClasses(invocation, startupOrder)
    }
}
```

3.验证效果:  
统计缺页中断：simpleperf stat -p [pid] --event major-faults,minor-faults
## 资源预加载
### 主题资源/字体/关键图片预加载
## CPU 升频
拉高 CPU 频率，绑定大核，约定时间（1s）进行关键初始化。  
避免启动阶段大量 IO（IO 不受 CPU Boost 影响）
## GC 抑制
adb shell logcat | grep "GC_" 查看启动期间 GC 情况  
### 启动期间抑制 GC
1. 减少临时对象创建  
2. 避免大对象分配（直接触发 GC）  
3. 预分配关键对象（启动前置）
### 对象预分配
预创建并复用的对象（对象池）  
```
private val bufferPool = ArrayDeque<ByteArray>()
bufferPool.add(ByteArray(1024 * 1024)) // 预分配1MB缓冲

fun acquireBuffer(): ByteArray = bufferPool.removeFirstOrNull() ?: ByteArray(1024 * 1024)
    
fun releaseBuffer(buffer: ByteArray) {
     // 用完归还
     bufferPool.addLast(buffer)
}
```
### GC 优化
1.锁屏 GC  
2.需要资源的特殊场景不 GC  
3.后台 GC  
但是，正常来讲 GC 是系统决定的，非必要应用侧不应该主动调用显式 GC，而是去分析内存问题
## 监控
### 核心指标
1.冷启动 P50 / P90 / P99 耗时
2.首屏展示时间（reportFullyDrawn）
3.启动成功率（启动期间 Crash 率）
4.启动期间 ANR 率
### 分段指标
1.Application 耗时
2.首个 Activity onCreate 耗时
3.布局 inflate 耗时
4.首帧渲染耗时
## Tips
1.异步任务过多会抢占首帧渲染的 CPU 资源，需要平衡  
2.Splash 展示期间用户已经在等待，不是"免费"时间，也不要刻意放入过多耗时任务  
3.关注 P90/P99，低端机用户体验同样重要
# 内存优化	
## 内存泄漏
程序中已动态分配的堆内存由于某种原因未被释放或无法释放，造成系统内存浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。  
### 单例引起的内存泄漏  
生命周期与程序生命周期一样，若有对象不再使用，却被单例持有，就会导致其无法回收，进而导致内存泄漏。  
如 context 对象传入了Activity，就会导致 Activity 销毁时无法被回收，传入 Application 上下文可解决。  
### 非静态内部类创建静态实例引起的内存泄漏
非静态内部类会默认持有外部类引用，若在外部类中去创建这个内部类对象，当频繁打开关闭Activity时，会导致重复创建对象，造成资源浪费。  
可把该实例设为静态，解决重复创建实例问题，但会引起其他问题（静态成员变量生命周期和程序生命周期一样），静态成员变量持有 Activity 引用，所以 Activity 销毁时，无法回收。  
### Handler引起的内存泄漏：  
若 Handler 通过内部类创建，内部类持有外部类引用（Activity），而队列中的消息 target 指向 Handler，即消息持有 Handler 引用，若 Activity 销毁时队列中的消息还未处理完，这些未处理完的消息会持有 Activity 引用，导致 Activity 无法回收。  
Handler 改为静态内部类，Activity#onDestroy 中移除队列中消息，Handler 内弱引用 Activity 对象，让内部类不再持有外部类的引用时，程序就不允许 Handler 操作 Activity 对象。  
### Asynctask引起的内存泄漏  
和 Handler 类似，Asynctask 内部类持有外部类引用。  
改为静态内部类并在 onDestroy 中取消任务即可。  
### WebView 引起的内存泄漏  
不同的版本上和不同的机型上会存在不同的问题。  
不在 xml 中定义，定义一个 view 容器（例 FrameLayout），在代码中动态添加（context 可使用弱引用），onDestroy 销毁时直接 layout.removeAllViews，移除WebView。  
### 资源对象未关闭引起的内存泄漏  
广播、服务、数据库cursor、多媒体、文件、套接字等。  
不使用时需关闭。  
### 三方注册资源未取消注册  
如 EventBus。  
为便于管理 Activity，将其添加到 Activity 栈中，销毁时未从栈中移除等。
### Activity/Fragment 泄漏
### 静态引用
### Handler 泄漏
### 监听器未注销
## OOM
JVM 在面临内存资源不足时的一种自我保护机制，遭遇 OOM 时，不会立即终止执行，无法为对象分配内存空间时，抛出 OOM（Error 子类，标志着一种通常不可恢复的、严重的运行时问题，本身不会直接导致 JVM 退出），JVM 会将这个错误传递给当前正在运行的代码，这给予了应用程序一个机会去捕获并处理这个异常，尽管在常规情况下并不推荐捕获和处理这种严重错误，但如果确实进行了这样的操作，程序可能会尝试继续执行。    
### 类型堆
1.内存不足（OutOfMemoryError: Java heap space）：  
最常见，随着对象的持续创建，如果它们因为某些原因（如内存泄漏）而无法被垃圾收集器有效回收，那么堆内存最终会被消耗殆尽，这种情况往往是因为代码中存在内存管理不当的问题。     
2.元空间/方法区空间不足（OutOfMemoryError: PermGen space/Metaspace）：    
当系统加载大量的类和方法时，这部分内存资源可能会变得紧张，通常发生在应用程序需要动态加载大量代码的场景中。   
3.本地方法栈空间不足（StackOverflowError）：  
如果线程请求的栈大小超出了JVM 所允许的最大值，就会导致本地方法栈溢出，通常与线程的设计和实现有关。   
4.请求内存超过物理和虚拟内存：   
请求的内存超过了物理内存和虚拟内存的限制时，也会触发 OutOfMemoryError，不仅仅与 JVM 的内存设置有关，还受到整个系统配置的影响。   
### 大图压缩
### Bitmap 复用（inBitmap）
### 堆外内存管理
### Large Heap 策略
## 内存抖动
### 频繁 GC
短时间内频繁大量创建临时对象，会频繁 GC，无论哪种方式实现的GC在执行时都不可避免的需要 STW（Stop The World），STW 意味着所有的工作线程都将被暂停，虽然时间很短，但终究会存在时间成本，一两次内存回收不易被察觉，但多次内存回收集中在短时间内爆发，这就会造成较大程度的界面卡顿风险。   
尽量避免在循环体中创建对象、尽量不要在自定义 View 的 onDraw 方法中创建对象（会被频繁调用） 
### 对象池复用
对于可复用对象，可以考虑使用对象池缓存。
## 图片优化
### 图片格式
PNG：无损压缩图片方式，支持 Alpha 通道，切图素材大多用这种格式。  
JPEG：有损压缩图片格式，不支持背景透明和多帧动画，适用于色彩丰富的图片压缩，不适合于 logo。  
WEBP：支持有损和无损压缩，支持完整的透明通道，也支持多帧动画，是一种比较理想的图片格式。  
.9图：点九图实际上仍然是 png 格式图片，它是针对 Andorid 平台特殊的图片格式，体积小，拉伸变形，能指定位置拉伸或者填充，能很好的适配机型。  
无损 webp 平均比 png 小26%，有损 jpeg 平均比 webp 少24%-35%，无损 webp 支持 Alpha 通道，有损 webp 在一定条件下也支持。采用 webp 在保持图片清晰情况下，可以优先减少磁盘空间大小。可以将 drawable 中的 png、jpg 格式图片转换为 webp 格式图片。  
### 像素格式
ALPHA_8，内存 1B，色彩组成为透明度，比较少用到。  
RGB_565，内存2B，色彩组成为颜色，不需要 Alpha 通道的，特别是 .JPG 格式的。  
ARGB_4444，内存2B，色彩组成为颜色+透明度，内存只占 ARGB_8888 的一半，已被废弃。  
ARGB_8888，内存4B，色彩组成为颜色+透明度，系统默认的像素点格式。  
通过替换系统 drawable 默认色彩通道（BitmapFactory.Options.inPreferredConfig），将部分没有透明通道的图片格式由 ARGB_8888 替换为 RGB565，在图片质量上的损失几乎肉眼不可见，而每张图片可以节省1/2的内存。但不通用，取决于图片是否有透明度需求。
## 采样率
在把图片载入内存之前，我们需要先计算出一个合适的 inSampleSize 缩放比例，降低图片像素，来达到降低图片占用内存大小的目的，避免不必要的大图载入。  
采样率 inSampleSize 只能是整数(只能是2的次方)，不能很好保证图片质量。如果 inSampleSize=2，则最终内存占用就会是原来的1/4（宽高都为原来的1/2），适用于图片过大的情况。  
## Bitmap优化
图片内存 = 宽高一个像素占用内存（和色彩模式有关，如 ARGB_8888 为 4（8*4 = 32位 = 4字节））。  
Bitmap 对象放在 Java 堆，像素数据放在 Native 内存，8.0 新增 NativeAllocationRegistry 来辅助回收 Native 内存，以及可减少图片内存并提升绘制效率的硬件位图 Hardware Bitmap ，Bitmap 的 Native 内存可和对象一起释放， GC 避免内存滥用。
### 释放对象  
Bitmap 使用完后需要调用 recycle() 方法回收资源，否则会发生内存泄漏。bitmap.recycle 用于释放与当前 Bitmap 对象相关联的 Native 对象，并清理对像素数据的引用。但不能同步地释放像素数据，而是在没有其它引用的时候，简单地允许像素数据被作为垃圾回收掉。  
Bitmap 在内存中的存储分两部分 ：一部分是 Bitmap 对象，另一部分为对应的像素数据，前者占据的内存较小，而后者才是内存占用的大头。  
bitmap 资源和 background 资源都要回收，对其调用 recycle，并且将维护的局部/全局对象等置为 null。
### inSampleSize
### RGB_565
### BitmapRegionDecoder
### Glide/Coil 配置
Glide 会自动调整加载的图片大小（根据 Imageview，以及三级缓存优化图片加载）。 
## 内存碎片化
### Java 堆碎片化
### Native 堆碎片化
### 碎片化检测
### 碎片整理策略
## 后台进程内存占用
### 主动内存收缩
### 内存快照对比分析
### 子进程内存预算管控
### 进程保活与内存代价
## 匿名内存（Anonymous Memory）增长异常
### Native 内存泄漏
### 线程数量膨胀
### JNI 层 GlobalRef 未释放
### 匿名 mmap 滥用
## Native 内存
### malloc/free 泄漏
### Jemalloc
### 内存映射（mmap）优化
## 分级缓存
### LruCache + DiskLruCache 二级缓存策略
## LMK 机制
### Low Memory Killer 原理
### 进程优先级
### oom_adj 管理
## 内存监控
### KOOM
### LeakCanary
hook Android 生命周期，自动检测当 Activity、Fragment 销毁时实例是否回收，销毁的实例传给 RefWatcher（持有它们的弱引用），保留实例（Retained Instance）数量达到阈值会进行堆转储，数据放进 hprof 文件（APP 可见阈值为 5，不可见为 1），会解析 hprof 文件，找出导致 GC 无法回收实例的引用链，就是泄漏踪迹（Leak Trace，最短强引用路径，GC Roots 到实例的路径）。  
监听系统内存状态：  
ComponentCallback2：在 Activity 中实现 ComponentCallback2 接口获取系统内存的相关事件, 在 onTrimMemory(level)  回调针对不同事件做不同释放内存操作  
ActivityManager.getMemoryInfo()：  
返回一个 ActivityManager.MemoryInfo 对象，包含系统当前内存状态（可用内存、总内存、低杀内存阈值， lowMemory 布尔值判断是否处于低内存态）  
### Hprof 裁剪上报
### 内存水位线告警
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
# 卡顿优化
## 主线程耗时	
### StrictMode 检测
### 主线程 IO/网络/锁等待
## 卡顿监控	
### Looper 消息监控
### WatchDog 机制
## Binder 
### 调用	Binder 线程池耗尽
### 跨进程调用耗时监控
## 系统调用	
### 主线程SP读写（MMKV 替代）
### 文件 IO 异步化
## 线程调度	
### 线程优先级
### CPU 亲和性
### 线程饥饿问题
## 消息队列	
### Handler 消息积压
### IdleHandler 合理使用
# 渲染优化	
## 帧率优化
### Choreographer 帧监控
### FrameMetrics API
### Janky Frame 分析
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


### 减少层级
1.ConstraintLayout 一层解决嵌套问题  
2.避免 RelativeLayout 嵌套 LinearLayout
### Merge/ViewStub
1.非首屏必要的 View 用 ViewStub 占位，按需加载  
2.<merge> + <include> ，去除多余的根布局节点，将内容直接添加到父容器中（同类型，如都是LinearLayout）
## 过度绘制
### GPU 过度绘制检测
### 背景清理
### clipRect/quickReject
## 列表优化
### RecyclerView 预取
### DiffUtil
### ItemAnimator 优化
### Paging3
## 自定义 View
### onDraw 避免对象创建
### 硬件加速
### Path/Paint 复用
## RenderThread
### 减少 DisplayList 构建耗时
### 属性动画优化
## 窗口动画
### 共享元素动画
### 过渡动画优化
## Compose 优化
### 重组（Recomposition）控制
### remember/derivedStateOf
### LazyColumn 优化
# 包体积优化	
## 代码裁剪	
### ProGuard/R8 混淆
### 无用代码删除
### 方法数优化
## 资源优化	
### 资源混淆（AndResGuard）
### 无用资源删除、lint 检查
## 图片压缩	
### WebP 转换
### 矢量图（VectorDrawable）替代
## So 库优化	
### ABI 过滤（只保留 arm64-v8a）
### So 动态下发
## 动态化方案	
### 插件化
### 热修复（Tinker/Robust）
### RN/Flutter 
### 动态下发
#### AAB 构建	App Bundle 按需下发
#### 语言/屏幕密度分包
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
