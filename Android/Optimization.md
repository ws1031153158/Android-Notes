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
##### BaseProfile
问题：  
App 安装后首次启动 → JIT 解释执行 → 慢  
多次启动后 → 系统后台 dex2oat → AOT编译 → 快  

优化：  
告诉系统："这些类/方法是启动关键路径，安装时就AOT编译"  
效果：首次启动即享受AOT性能，启动速度提升 20%~40%  

接入：  
```
// add dependencies
// app/build.gradle
dependencies {
    implementation "androidx.profileinstaller:profileinstaller:1.3.1"

    // 测试模块依赖
    "baselineProfile"(project(":baselineprofile"))
}

// baselineprofile/build.gradle（新建模块）
plugins {
    id("com.android.test")
    id("androidx.baselineprofile")
}

android {
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true
}

dependencies {
    implementation("androidx.test.ext:junit:1.1.5")
    implementation("androidx.test.uiautomator:uiautomator:2.2.0")
    implementation("androidx.benchmark:benchmark-macro-junit4:1.2.3")
}
```

```
//Profile Generator
// baselineprofile/src/main/.../BaselineProfileGenerator.kt
@RunWith(AndroidJUnit4::class)
@LargeTest
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(
        packageName = "com.xxx.app",
        // 包含哪些交互路径
        includeInStartupProfile = true
    ) {
        // ================================
        // 描述用户启动路径
        // 框架会记录这些路径中加载的所有类/方法
        // ================================

        // 1. 冷启动
        pressHome()
        startActivityAndWait() // 等待首帧渲染完成

        // 2. 首页关键交互
        device.waitForIdle()

        // 滑动首页列表（触发列表渲染相关代码）
        device.findObject(By.res("com.xxx.app:id/home_recycler"))
            ?.let { recycler ->
                recycler.scroll(Direction.DOWN, 3f)
                recycler.scroll(Direction.UP, 3f)
            }

        // 3. 点击进入详情页（高频路径）
        device.findObject(By.res("com.xxx.app:id/item_card"))
            ?.click()
        device.waitForIdle()

        // 4. 返回
        device.pressBack()
        device.waitForIdle()

        // 5. 搜索路径
        device.findObject(By.res("com.xxx.app:id/search_btn"))
            ?.click()
        device.waitForIdle()
    }
}
```

```
# 连接真机（需要 rooted 设备或 Google Pixel）
# 或使用 Android Studio 的 Managed Device

# 运行生成器
./gradlew :app:generateReleaseBaselineProfile

# 生成的文件位置：
# app/src/{$Flavor}Release/baseline-prof.txt

# 文件内容示例：
# HSPLcom/xxx/MainActivity;-><init>()V
# HSPLcom/xxx/MainActivity;->onCreate(Landroid/os/Bundle;)V
# HSPLcom/xxx/network/OkHttpManager;->init(Landroid/content/Context;)V
# ...
# H = Hot（热方法，AOT编译）
# S = Startup（启动时加载）
# P = Post-startup（启动后加载）
# L = 类加载
```

```
// check effect
// macrobenchmark/src/.../StartupBenchmark.kt
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    // 没有 Baseline Profile 的冷启动
    @Test
    fun coldStartWithoutProfile() = benchmarkRule.measureRepeated(
        packageName = "com.xxx.app",
        metrics = listOf(
            StartupTimingMetric(),
            FrameTimingMetric()
        ),
        iterations = 10,
        startupMode = StartupMode.COLD,
        setupBlock = {
            // 清除 Profile 编译
            device.executeShellCommand(
                "cmd package compile -m interpret-only -f com.xxx.app"
            )
        }
    ) {
        pressHome()
        startActivityAndWait()
    }

    // 有 Baseline Profile 的冷启动
    @Test
    fun coldStartWithProfile() = benchmarkRule.measureRepeated(
        packageName = "com.xxx.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 10,
        startupMode = StartupMode.COLD,
        setupBlock = {
            // 应用 Profile 编译
            device.executeShellCommand(
                "cmd package compile -m speed-profile -f com.xxx.app"
            )
        }
    ) {
        pressHome()
        startActivityAndWait()
    }
}

// 典型结果：
// 无Profile：timeToInitialDisplay P50 = 850ms
// 有Profile：timeToInitialDisplay P50 = 580ms
// 提升约 32% ✅
```

```
# CI Integration
# .git/workflows/baseline-profile.yml
name: Generate Baseline Profile

on:
  # 每次发版前生成
  push:
    branches: [ release/* ]

jobs:
  generate-profile:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Android SDK
        uses: android-actions/setup-android@v2

      - name: Generate Baseline Profile
        run: |
          ./gradlew :baselineprofile:generateBaselineProfile \
            -Pandroid.testoptions.manageddevices.emulator.gpu=swiftshader_indirect

      - name: Commit Profile
        run: |
          git add app/src/main/baseline-prof.txt
          git commit -m "Update Baseline Profile"
          git push
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
```

```
// 任务状态定义
sealed class TaskState {
    object Pending   : TaskState()
    object Running   : TaskState()
    data class Success(val costMs: Long) : TaskState()
    data class Failed(val error: Throwable) : TaskState()
}

data class StartupProgress(
    val taskId: String,
    val state: TaskState,
    val totalTasks: Int,
    val completedTasks: Int
) {
    val progressPercent: Int
        get() = (completedTasks * 100f / totalTasks).toInt()
}
```

调度器，拓扑排序检测循环依赖：  
```
class StartupScheduler(private val context: Context) {

    private val taskMap = mutableMapOf<String, StartupTask>()
    private val taskStates = mutableMapOf<String, MutableStateFlow<TaskState>>()

    // 对外暴露的进度流
    private val _progressFlow = MutableSharedFlow<StartupProgress>(
        replay = 0,
        extraBufferCapacity = 64
    )
    val progressFlow: SharedFlow<StartupProgress> = _progressFlow

    // 所有任务完成的信号
    private val _completedFlow = MutableStateFlow(false)
    val completedFlow: StateFlow<Boolean> = _completedFlow

    fun register(vararg tasks: StartupTask): StartupScheduler {
        tasks.forEach { task ->
            taskMap[task.id] = task
            taskStates[task.id] = MutableStateFlow(TaskState.Pending)
        }
        return this
    }

    // 注册完成后，start() 前先做检测
    suspend fun start() = coroutineScope {
        // 第一步：检测所有问题，有问题直接抛异常
        validateGraph()

        val totalTasks = taskMap.size
        val completedCount = AtomicInteger(0)
        val dependencyCount = buildDependencyCount()
        val dependents = buildDependents()

        // 找出所有无依赖的任务，立即启动
        val readyTasks = dependencyCount
            .filter { it.value.get() == 0 }
            .keys

        readyTasks.forEach { taskId ->
            launchTask(
                taskId, this,
                dependencyCount, dependents,
                totalTasks, completedCount
            ) 
        }

       //正常调度
       schedule()
    }

    private fun launchTask(
        taskId: String,
        scope: CoroutineScope,
        dependencyCount: Map<String, AtomicInteger>,
        dependents: Map<String, List<String>>,
        totalTasks: Int,
        completedCount: AtomicInteger
    ) {
        val task = taskMap[taskId] ?: return
        val stateFlow = taskStates[taskId] ?: return

        val dispatcher = if (task.runOnMainThread)
            Dispatchers.Main else Dispatchers.IO

        scope.launch(dispatcher) {
            // 更新状态：Running
            stateFlow.value = TaskState.Running
            _progressFlow.emit(
                StartupProgress(
                    taskId = taskId,
                    state = TaskState.Running,
                    totalTasks = totalTasks,
                    completedTasks = completedCount.get()
                )
            )

            val startTime = SystemClock.elapsedRealtime()

            try {
                task.run(context)

                val cost = SystemClock.elapsedRealtime() - startTime
                val completed = completedCount.incrementAndGet()

                // 更新状态：Success
                val successState = TaskState.Success(cost)
                stateFlow.value = successState
                _progressFlow.emit(
                    StartupProgress(
                        taskId = taskId,
                        state = successState,
                        totalTasks = totalTasks,
                        completedTasks = completed
                    )
                )

                // 检查是否全部完成
                if (completed == totalTasks) {
                    _completedFlow.value = true
                }

                // 通知后续任务
                dependents[taskId]?.forEach { depId ->
                    val remaining = dependencyCount[depId]?.decrementAndGet()
                    if (remaining == 0) {
                        launchTask(
                            depId, scope,
                            dependencyCount, dependents,
                            totalTasks, completedCount
                        )
                    }
                }
            } catch (e: Exception) {
                stateFlow.value = TaskState.Failed(e)
                _progressFlow.emit(
                    StartupProgress(
                        taskId = taskId,
                        state = TaskState.Failed(e),
                        totalTasks = totalTasks,
                        completedTasks = completedCount.get()
                    )
                )
            }
        }
    }

    // 图合法性全量检测
    private fun validateGraph() {
        // 检测1：依赖的任务是否都已注册
        checkMissingDependencies()
        
        // 检测2：是否存在循环依赖（Kahn拓扑排序）
        checkCyclicDependencies()
    }

    // 获取特定任务的状态流
    fun getTaskState(taskId: String): StateFlow<TaskState>? =
        taskStates[taskId]

    // 等待特定任务完成（挂起）
    suspend fun waitForTask(taskId: String) {
        taskStates[taskId]
            ?.first { it is TaskState.Success || it is TaskState.Failed }
    }

    private fun buildDependencyCount() = taskMap.mapValues { (id, task) ->
        AtomicInteger(task.dependencies.size)
    }.toMutableMap()

    private fun buildDependents(): Map<String, List<String>> {
        val map = mutableMapOf<String, MutableList<String>>()
        taskMap.values.forEach { task ->
            task.dependencies.forEach { depId ->
                map.getOrPut(depId) { mutableListOf() }.add(task.id)
            }
        }
        return map
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

    private suspend fun schedule(){...}
}
```

```
//Consume
class SplashActivity : AppCompatActivity() {

    private val scheduler = FlowStartupScheduler(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_splash)

        // 注册任务
        scheduler.register(
            ConfigTask(),
            NetworkTask(),
            DatabaseTask(),
            UserInfoTask(),
            HomeDataTask()
        )

        // 观察启动进度，更新骨架屏
        lifecycleScope.launch {
            scheduler.progressFlow
                .flowOn(Dispatchers.Main)
                .collect { progress ->
                    // 更新进度条
                    progressBar.progress = progress.progressPercent

                    // 更新任务状态文字（调试模式）
                    if (BuildConfig.DEBUG) {
                        statusText.text =
                            "${progress.taskId}: ${progress.state}"
                    }
                }
        }

        // 观察全部完成信号
        lifecycleScope.launch {
            scheduler.completedFlow
                .filter { it } // 等 true
                .first()       // 只取第一次
            // 所有任务完成，跳转主页
            navigateToMain()
        }

        // 启动调度
        lifecycleScope.launch {
            scheduler.start()
        }
    }

    // 也可以等待特定任务完成后做某件事
    // 比如：网络初始化完成后才发请求
    private suspend fun waitNetworkThenFetch() {
        scheduler.waitForTask("network")
        // network 完成，可以发网络请求了
        fetchSplashAd()
    }
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
## 低端机优化
### 特殊性
低端机特征：  
1.CPU：4核以下，主频 1.4GHz 以下  
2.RAM：2GB 以下，可用内存可能只有 500MB  
3.存储：eMMC 5.0（随机IO比高端机慢 3~5倍）  
4.GPU：Mali-400 等老旧GPU  
5.Android版本：可能是 Android 8/9  

导致的问题：  
1.类加载慢（CPU弱）  
2.IO慢（存储慢）→ Dex加载、资源加载更慢  
3.内存紧张 → 频繁GC → 启动期间GC更多  
4.系统本身负载高 → Binder响应更慢  
5.多进程竞争资源更激烈  
### 优化
#### IO 优化
```
// 低端机 IO 比高端机慢 3~5 倍
// 所有 IO 操作必须异步

class LowEndIOOptimizer {

    // 策略：预判设备等级，动态调整策略
    private val isLowEnd = isLowEndDevice()

    fun optimizeStartup(context: Context) {
        if (isLowEnd) {
            // 低端机：更激进的异步策略
            // 减少启动时的IO操作数量
            applyLowEndStrategy(context)
        } else {
            applyNormalStrategy(context)
        }
    }

    private fun applyLowEndStrategy(context: Context) {
        // 1. 减少启动时读取的SP数量
        //    只预加载最关键的1个SP，其他延迟
        context.getSharedPreferences("critical_only", MODE_PRIVATE)

        // 2. 数据库WAL模式（减少IO等待）
        //    WAL允许读写并发(修改写入独立的.wal文件，内存共享，在此读最新数据，写仅追加到文件末尾，n对1，系统会定期写会.db，清空.wal)，减少锁等待
        val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)
            .build()

        // 3. 图片只加载低分辨率版本
        ImageLoader.setLowEndConfig(
            maxBitmapSize = 720,  // 高端机可能是1080
            memCacheSize = 32     // MB，高端机可能是128MB
        )
    }

    private fun isLowEndDevice(): Boolean {
        val am = context.getSystemService(ActivityManager::class.java)
        val memInfo = ActivityManager.MemoryInfo()
        am.getMemoryInfo(memInfo)
        return am.isLowRamDevice ||
               memInfo.totalMem < 3L * 1024 * 1024 * 1024 || // 3GB
               Runtime.getRuntime().availableProcessors() <= 4
    }
}
```
#### 内存优化
```
class LowEndMemoryStrategy {

    fun apply(application: Application) {
        application.registerComponentCallbacks(object : ComponentCallbacks2 {
            override fun onTrimMemory(level: Int) {
                when (level) {
                    // 低端机更激进地释放内存
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW -> {
                        // 系统内存不足，立即释放非必要缓存
                        ImageLoader.clearMemoryCache()
                        clearNonCriticalCache()
                    }
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
                        // 极度紧张，释放所有能释放的
                        releaseAllCache()
                    }
                }
            }
            override fun onConfigurationChanged(config: Configuration) {}
            override fun onLowMemory() {
                releaseAllCache()
            }
        })
    }

    // 低端机启动时，主动控制对象创建
    fun controlObjectCreation() {
        // 减少启动期间的大对象创建
        // 避免触发GC

        // 例：图片缓存大小按内存动态调整
        val maxMemory = Runtime.getRuntime().maxMemory()
        val cacheSize = when {
            maxMemory < 64 * 1024 * 1024  -> 8   // <64MB heap → 8MB cache
            maxMemory < 128 * 1024 * 1024 -> 16  // <128MB heap → 16MB cache
            else                          -> 32  // 正常 → 32MB cache
        }
        ImageLoader.setMemCacheSize(cacheSize)
    }
}
```
#### 任务裁剪
```
// 低端机启动时，砍掉非必要任务
class AdaptiveStartupScheduler(context: Context) 
    : FlowStartupScheduler(context) {

    private val isLowEnd = isLowEndDevice()

    override fun register(vararg tasks: StartupTask): FlowStartupScheduler {
        val filteredTasks = tasks.filter { task ->
            if (isLowEnd) {
                // 低端机过滤掉非关键任务
                task.priority >= PRIORITY_CRITICAL
            } else {
                true
            }
        }
        return super.register(*filteredTasks.toTypedArray())
    }

    companion object {
        const val PRIORITY_CRITICAL = 10  // 必须执行
        const val PRIORITY_NORMAL   = 5   // 正常执行
        const val PRIORITY_LOW      = 1   // 低端机跳过
    }
}

// 任务声明时标注优先级
class AnalyticsTask : StartupTask() {
    override val id = "analytics"
    override val priority = AdaptiveStartupScheduler.PRIORITY_LOW
    // 低端机启动时跳过统计SDK初始化
    // 延迟到用户真正操作时再初始化
}
```
#### 首屏简化
```
// 低端机展示简化版首屏
class AdaptiveHomeActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        if (isLowEndDevice()) {
            // 低端机：加载简化布局
            // 减少View数量，降低measure/layout耗时
            setContentView(R.layout.activity_home_simple)
            loadSimpleContent()
        } else {
            // 高端机：完整布局
            setContentView(R.layout.activity_home_full)
            loadFullContent()
        }
    }

    private fun loadSimpleContent() {
        // 低端机首屏策略：
        // 1. 不加载Banner（动画耗GPU）
        // 2. 列表只加载前5条（减少首屏数据量）
        // 3. 图片延迟加载（先展示占位图）
        // 4. 不播放任何动画
    }
}
```
#### 进程优先级
```
# 低端机内存紧张，App进程容易被LMK杀死
# 启动期间提升进程优先级

# 代码层面：
# 启动时设置前台Service，提升进程优先级
# 避免启动过程中被系统杀死

// 启动期间临时提升优先级
class StartupForegroundService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // 创建通知，提升为前台Service
        val notification = buildStartupNotification()
        startForeground(NOTIFICATION_ID, notification)

        // 启动完成后停止
        lifecycleScope.launch {
            waitForStartupComplete()
            stopSelf()
        }
        return START_NOT_STICKY
    }
}
```
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
### 冷启 Binder 耗时
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
    private val cache = mutableMapOf<String, Boolean>()
    
    fun preloadAsync(context: Context, packages: List<String>) {
        thread {
            packages.forEach { pkg ->
                cache[pkg] = try {
                    context.packageManager.getPackageInfo(pkg, 0)
                    true
                } catch (e: Exception) { false }
            }
        }
    }
    
    fun isInstalled(pkg: String) = cache[pkg] ?: false
}
```

3.异步 CP 查询，使用 CursorLoader 或协程：启动时查询 ContentProvider（如联系人、媒体库）,每次 query 都是跨进程 Binder 调用:  
```
class SplashActivity : AppCompatActivity() {
    override fun onCreate(...) {
        lifecycleScope.launch(Dispatchers.IO) {
            // 子线程执行 Binder 调用，不阻塞主线程
            val cursor = contentResolver.query(...)
            withContext(Dispatchers.Main) {
                updateUI(cursor)
            }
        }
    }
}
```
### 调用 Binder 线程池耗尽
每个进程的 Binder 线程池默认最多 15 个线程，大量异步任务同时发起 Binder 调用 -> 每个调用都在等待 system_server 响应 -> 新的 Binder 请求：只能排队等待 -> 主线程的 Binder 调用也被阻塞 -> ANR

```
// 检测 Binder 线程池状态
// adb shell cat /proc/[pid]/status | grep Threads
// 观察线程数是否接近 15（Binder线程上限）

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
R8 = 编译器 + 混淆器 + 优化器 的三合一工具  
AGP 3.4+ 默认替代 ProGuard  
ProGuard：源码 → javac → .class → ProGuard(混淆/裁剪) → dex工具 → .dex  
R8：源码 → kotlinc/javac → .class → R8(混淆/裁剪/优化/dex，一步完成，速度更快) → .dex

R8 核心能力：
1.Tree Shaking（摇树，删除无用代码）  
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

2. Minification（混淆，缩短名称）  
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
```

减少包体积:  
类名从平均20字符 → 1~2字符,包路径从30字符 → 1字符  
删除无用代码  
内联（减少方法数）  
减少 Dex 文件头开销  

3. Optimization（优化，改写字节码）  
```
// ① 内联（Inlining）
// 优化前
fun add(a: Int, b: Int) = a + b
fun calculate() = add(1, 2) // 函数调用有开销

// 优化后（R8内联）
fun calculate() = 3 // 直接计算结果，消除函数调用

// ② 常量折叠
val MAX_SIZE = 100
val DOUBLE_SIZE = MAX_SIZE * 2 // R8直接替换为 200

// ③ 无效代码消除
fun example(debug: Boolean = false) {
    if (debug) {
        // BuildConfig.DEBUG = false 时
        // R8 直接删除整个if块
        Log.d("TAG", "debug info")
    }
}

// ④ 类合并（Class Merging）
// 只有一个子类的抽象类 → 合并为一个类
// 减少类数量，降低方法数

// ⑤ 参数移除
// 未使用的方法参数 → 直接删除
fun process(data: String, unused: Int) { // unused被删除
    println(data)
}
```

4. Dexing（生成Dex文件）
```
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
```

Dexing 步骤：  
```
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
#### 内联
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
```
#### 类合并
```
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
#### 需要保留内容
1.参数：
-keepattributes *Annotation*(注释)/SourceFile/LineNumberTable(行号，看堆栈需要)/Signature(泛型信息，序列化需要)  
2.Framework 组件：
-keep class * extends android.app/android.content/ ... ...
3.数据模型(JSON序列化用到反射):
-keep class com.xxx.model.** { *; }(特定包下所有类)
-keep @com.xxx.annotation.KeepModel class * { *; }(带注解的类,更精准)
4.三方SDK(按需)：
如Retrofit、Gson、OkHttp
5.反射相关（运行时通过反射访问的类，R8看不懂字符串，会认为不可达而删除）：
-keep class com.xxx.plugin.** { *; }
6.Native方法（JNI调用）：
-keepclasseswithmembernames class * {native <methods>;}
7.枚举（Enum有特殊方法）：
-keepclassmembers enum * {public static **[] values(); public static ** valueOf(java.lang.String);}
8.Parcelable：
-keepclassmembers class * implements android.os.Parcelable {public static final android.os.Parcelable$Creator CREATOR;}
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
