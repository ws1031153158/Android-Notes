# 启动优化	     
## 测量
```
# 测量冷启动
adb shell am start -W -S com.xxx.app/.MainActivity
# -S 表示先 force-stop，确保冷启动

# 测量温启动（进程存在，Activity 销毁）
# 先按返回键退出 App（不要 force-stop）
adb shell am start -W com.xxx.app/.MainActivity

# 输出：
# Status: ok
# LaunchState: COLD / WARM / HOT  ← 系统自动判断
# ThisTime: 234ms
# TotalTime: 456ms
# WaitTime: 478ms
```
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
流程：Activity创建(onCreate)+布局(setContentView / inflate)+首帧渲染     
```
触发场景：用户按返回键退出，进程还在

用户再次点击图标：
    ① AMS 检查进程 → 存在 ✅
    ② AMS 检查 Activity 栈 → 栈为空 或 Activity已销毁 ❌
    ③ 需要重新创建 Activity

App 进程收到消息：
    跳过：进程创建（已存在）
    跳过：Application.onCreate（已执行）
    执行：Activity.onCreate()
          setContentView() → inflate 布局
          onStart()
          onResume()
          首帧渲染

用户看到界面 ← 温启动结束
```

问题：  
```
// 进程级单例状态丢失
// 进程还在，但 Activity 被销毁重建
// 进程级单例（内存中的数据）还在
// 但 Activity 相关的状态已经丢失

// 常见错误：依赖 Activity 传递的数据，温启动后为空
object UserSession {
    // 进程级单例，温启动后依然存在 ✅
    var currentUser: User? = null
    var authToken: String? = null
}

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ❌ 错误：假设 UserSession 一定有数据
        // 冷启动时：Application.onCreate 中初始化了 UserSession
        // 温启动时：Application.onCreate 不执行！
        //           但 UserSession 是进程级单例，数据还在 ✅
        //           所以这里其实是安全的

        // ❌ 真正的问题：依赖 Activity 传递的临时数据
        // 比如：上一个 Activity 通过 Intent 传来的数据
        // 温启动重建时，需要重新处理这些数据
        val userId = intent.getStringExtra("userId") ?: run {
            // 温启动时可能没有这个 extra
            // 需要从其他地方恢复
            UserSession.currentUser?.id ?: run {
                // 真的没有，跳转登录
                startActivity(Intent(this, LoginActivity::class.java))
                finish()
                return
            }
        }
    }
}

// onSaveInstanceState 恢复
class HomeActivity : AppCompatActivity() {

    private var scrollPosition = 0
    private var selectedTabIndex = 0

    // 系统销毁 Activity 前调用（温启动场景）
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // 保存需要恢复的状态
        outState.putInt("scroll_position", scrollPosition)
        outState.putInt("selected_tab", selectedTabIndex)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)

        // 温启动恢复状态
        savedInstanceState?.let { state ->
            scrollPosition = state.getInt("scroll_position", 0)
            selectedTabIndex = state.getInt("selected_tab", 0)

            // 恢复UI状态
            recyclerView.scrollToPosition(scrollPosition)
            tabLayout.selectTab(tabLayout.getTabAt(selectedTabIndex))
        }
    }
}

// 温启动每次都要重新 inflate 布局
// 这是温启动的主要耗时来源

// 优化方案一：ViewStub 延迟加载非首屏内容
class HomeActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)

        // 首屏必要内容立即加载
        setupToolbar()
        setupMainContent()

        // 非首屏内容延迟加载
        // 温启动时用户看到首屏后，再加载其他内容
        viewStubSidePanel.setOnInflateListener { _, inflated ->
            setupSidePanel(inflated)
        }
        // 用户点击侧边栏时才 inflate
    }
}

// 优化方案二：Fragment 懒加载
class HomeActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)

        // 只加载首个 Tab 的 Fragment
        if (savedInstanceState == null) {
            supportFragmentManager.beginTransaction()
                .add(R.id.container, HomeFragment())
                .commit()
            // 其他 Tab 的 Fragment 在用户切换时才创建
        }
    }
}
```

2.热启动：进程存在，Activity 在后台，只需要走 onStart，但是如果一些内存为响应内存整理事件（如 onTrimMemory()）而被完全清除，则需要为了响应热启动而重新创建相应的对象，热启动显示的屏幕上行为和冷启动场景相同。系统进程显示空白屏幕，直到应用完成 Activity 呈现  
流程：onResume+首帧渲染    
```
用户按 Home 键：
    Activity.onPause()
    Activity.onStop()
    进程继续存活，Activity 实例保留在内存

用户点击图标回来：
    ① AMS 检查进程 → 存在 ✅
    ② AMS 检查 Activity 栈 → Activity 实例存在 ✅
    ③ 发送 Resume 消息到 App 进程

App 进程收到消息：
    Activity.onRestart()
    Activity.onStart()
    Activity.onResume()
    ViewRootImpl 触发重绘
    首帧渲染上屏

用户看到界面 ← 热启动结束
```

问题：  
```
① onResume 中的耗时操作
② 数据刷新（重新请求网络/数据库）
③ 动画重新启动
④ 首帧重绘耗时
⑤ 图片重新加载（如果被回收）
```
### 温启动优化
1.Application 数据预热：  
```
// 温启动时 Application 不重新执行
// 但进程级缓存还在
// 可以在冷启动时多做一些预热，温启动直接用

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 预热数据缓存（温启动时直接命中）
        GlobalScope.launch(Dispatchers.IO) {
            // 预加载用户数据到内存
            UserCache.preload(this@MyApplication)
            // 预加载配置到内存
            ConfigCache.preload(this@MyApplication)
        }
    }
}

// 温启动时 Activity 直接从内存缓存取数据
// 无需重新请求网络或读取数据库
object UserCache {
    private var cachedUser: User? = null

    suspend fun preload(context: Context) {
        cachedUser = withContext(Dispatchers.IO) {
            AppDatabase.getInstance(context).userDao().getCurrentUser()
        }
    }

    fun getUser(): User? = cachedUser
}
```

2.进程保活策略:  
```
// 让进程存活更长时间
// 用户下次启动时走温启动而非冷启动

// 方案：前台 Service（最强保活，但影响用户体验，谨慎使用）
// 方案：1像素 Activity（灰色地带，不推荐）
// 方案：JobScheduler 定期唤醒（合规方案）

// 推荐方案：合理使用 WorkManager 保持进程活跃
class DataSyncWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 定期同步数据，顺带保持进程活跃
        syncLatestData()
        return Result.success()
    }
}

// 注册定期任务
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "data_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    PeriodicWorkRequestBuilder<DataSyncWorker>(
        15, TimeUnit.MINUTES  // 最小间隔15分钟
    ).build()
)
```

### 热启动优化
1.onResume 瘦身  
```
    override fun onResume() {
        super.onResume()

        // 只做必须在 onResume 做的事
        // 比如：恢复动画、注册监听器
        resumeAnimations()
        registerSensorListener()

        // 数据刷新：判断是否真的需要
        refreshIfNeeded()
    }

    private fun refreshIfNeeded() {
        val now = System.currentTimeMillis()
        val lastRefreshTime = PreferenceManager
            .getDefaultSharedPreferences(this)
            .getLong("last_refresh", 0)

        // 超过5分钟才刷新，避免每次热启动都请求网络
        if (now - lastRefreshTime > 5 * 60 * 1000) {
            lifecycleScope.launch {
                refreshData()
            }
        }
    }
```

2.数据缓存  
```
class HomeViewModel : ViewModel() {

    // 内存缓存：热启动直接用，无需重新请求
    private var cachedData: HomeData? = null
    private var cacheTime: Long = 0

    fun loadHomeData(forceRefresh: Boolean = false) {
        viewModelScope.launch {

            // 热启动场景：内存缓存还在，直接展示
            cachedData?.let { cache ->
                val cacheAge = System.currentTimeMillis() - cacheTime
                if (!forceRefresh && cacheAge < CACHE_VALID_DURATION) {
                    // 直接用缓存，热启动无感知
                    _uiState.value = UiState.Success(cache)
                    return@launch
                }
            }

            // 缓存失效或强制刷新：先展示旧数据，后台更新
            cachedData?.let {
                _uiState.value = UiState.Success(it) // 先展示旧数据
            }

            // 后台静默更新
            try {
                val newData = repository.fetchHomeData()
                cachedData = newData
                cacheTime = System.currentTimeMillis()
                _uiState.value = UiState.Success(newData)
            } catch (e: Exception) {
                // 更新失败，旧数据依然可用
                if (cachedData == null) {
                    _uiState.value = UiState.Error(e)
                }
            }
        }
    }

    companion object {
        private const val CACHE_VALID_DURATION = 5 * 60 * 1000L // 5分钟
    }
}
```

3.图片缓存  
```
// 热启动时图片被回收是常见问题
// Glide/Coil 有内存缓存，但内存紧张时会被回收

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 配置 Glide 内存缓存大小
        // 确保热启动时图片还在内存中
        GlideBuilder()
            .setMemoryCache(
                LruResourceCache(
                    // 根据设备内存动态调整
                    calculateMemoryCacheSize(this)
                )
            )
    }

    private fun calculateMemoryCacheSize(context: Context): Long {
        val am = context.getSystemService(ActivityManager::class.java)
        val memInfo = ActivityManager.MemoryInfo()
        am.getMemoryInfo(memInfo)

        return when {
            memInfo.totalMem > 4L * 1024 * 1024 * 1024 -> // >4GB
                128 * 1024 * 1024L  // 128MB 图片缓存

            memInfo.totalMem > 2L * 1024 * 1024 * 1024 -> // >2GB
                64 * 1024 * 1024L   // 64MB

            else ->
                32 * 1024 * 1024L   // 32MB
        }
    }
}
```

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
```
类的首次加载流程：
① ClassLoader 查找类定义（在 Dex 文件中查找）
② 读取类的字节码
③ 验证字节码（Class Verify）
④ 分配 Class 对象内存
⑤ 执行静态初始化块 <clinit>

首次加载耗时：0.1ms ~ 5ms（取决于类的复杂度）
启动阶段需要加载数百个类 → 累计耗时显著

预加载：在用户看 Splash 时，提前触发类加载
        进入首页时类已在内存中，直接使用
```
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

4.运行时预加载：  
```
object ClassPreloader {

    // 启动阶段需要的关键类列表
    // 通过 Perfetto/Systrace 分析得出
    private val criticalClasses = listOf(
        "com.xxx.home.HomeActivity",
        "com.xxx.home.HomeViewModel",
        "com.xxx.home.HomeAdapter",
        "com.xxx.network.OkHttpManager",
        "com.xxx.image.GlideManager",
        "com.xxx.db.AppDatabase"
    )

    fun preload() {
        GlobalScope.launch(Dispatchers.IO) {
            criticalClasses.forEach { className ->
                try {
                    // 触发类加载（不实例化）
                    Class.forName(className)
                } catch (e: ClassNotFoundException) {
                    // 类不存在，忽略
                }
            }
        }
    }
}

// 更精准的方式：
// 通过插桩记录线上用户的类加载顺序
// 生成 criticalClasses 列表
// 类似 Dex 重排的思路，但这里是运行时预加载
```
## 资源预加载
1.图片：  
```
object ImagePreloader {

    // 启动阶段需要的关键图片
    private val startupImages = listOf(
        R.drawable.home_banner_placeholder,
        R.drawable.default_avatar,
        R.drawable.tab_home_selected,
        R.drawable.tab_home_normal,
        R.drawable.tab_mine_selected,
        R.drawable.tab_mine_normal
    )

    fun preload(context: Context) {
        // 在子线程预加载，不阻塞主线程
        GlobalScope.launch(Dispatchers.IO) {
            startupImages.forEach { resId ->
                // 方式一：通过 Glide 预加载到内存缓存
                Glide.with(context)
                    .load(resId)
                    .preload() // 加载到缓存，不显示到View

                // 方式二：直接解码到内存（不依赖Glide）
                // BitmapFactory.decodeResource(context.resources, resId)
            }
        }
    }

    // 网络图片预加载（首屏Banner等）
    fun preloadNetworkImages(context: Context, urls: List<String>) {
        GlobalScope.launch(Dispatchers.IO) {
            urls.forEach { url ->
                Glide.with(context)
                    .load(url)
                    .diskCacheStrategy(DiskCacheStrategy.ALL)
                    .preload()
            }
        }
    }
}

// 在 Application.onCreate 中调用
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 子线程预加载图片
        // Splash展示期间完成，进入首页时直接命中缓存
        ImagePreloader.preload(this)
    }
}
```

2.字体：  
```
// 自定义字体首次加载耗时明显（需要解析字体文件）
// 预加载后，TextView 使用字体时直接命中缓存

object FontPreloader {

    fun preload(context: Context) {
        GlobalScope.launch(Dispatchers.IO) {
            try {
                // 方式一：ResourcesCompat 预加载
                // 内部有缓存，预加载后后续使用直接命中
                ResourcesCompat.getFont(context, R.font.custom_regular)
                ResourcesCompat.getFont(context, R.font.custom_bold)
                ResourcesCompat.getFont(context, R.font.custom_medium)

            } catch (e: Exception) {
                // 字体加载失败，降级使用系统字体
                Log.w("FontPreloader", "字体预加载失败: ${e.message}")
            }
        }
    }
}

// 更好的方式：使用 PrecomputedText 预计算文本布局
// 适用于首屏有大量固定文本的场景
object TextPreloader {

    fun precomputeTexts(
        context: Context,
        texts: List<String>,
        textViewParams: PrecomputedTextCompat.Params
    ): List<PrecomputedTextCompat> {
        // 在子线程预计算文本布局
        // 主线程直接使用计算结果，跳过 measure 中的文本测量
        return texts.map { text ->
            PrecomputedTextCompat.create(text, textViewParams)
        }
    }
}
```

3.数据库：  
```
// Room 数据库首次 getDatabase() 耗时较长
// 原因：数据库文件打开 + schema 验证 + migration 检查

object DatabasePreloader {

    @Volatile
    private var preloadJob: Job? = null

    fun preload(context: Context) {
        preloadJob = GlobalScope.launch(Dispatchers.IO) {
            // 触发数据库初始化
            // 后续真正使用时直接命中已初始化的实例
            AppDatabase.getInstance(context)

            // 预热关键查询（可选）
            // 让 SQLite 的页缓存预热
            AppDatabase.getInstance(context)
                .userDao()
                .getCurrentUser()
        }
    }

    // 确保预加载完成（在需要使用数据库之前调用）
    suspend fun awaitPreload() {
        preloadJob?.join()
    }
}
```

5.SP:  
```
// SP 首次 getSharedPreferences 会触发文件 IO
// 预加载到内存后，后续读取直接命中内存缓存

object SPPreloader {

    // 需要预加载的 SP 文件名列表
    private val spFiles = listOf(
        "user_config",
        "app_settings",
        "feature_flags"
    )

    fun preload(context: Context) {
        // 必须在子线程！避免主线程 IO
        GlobalScope.launch(Dispatchers.IO) {
            spFiles.forEach { name ->
                // 触发 SP 文件读取，加载到内存
                context.getSharedPreferences(name, Context.MODE_PRIVATE)
            }
        }
    }
}

// Application 中统一管理预加载
class MyApplication : Application() {

    override fun attachBaseContext(base: Context) {
        super.attachBaseContext(base)
        // attachBaseContext 阶段就开始预加载 SP
        // 比 onCreate 更早，争取更多并行时间
        SPPreloader.preload(base)
    }

    override fun onCreate() {
        super.onCreate()
        ImagePreloader.preload(this)
        FontPreloader.preload(this)
        DatabasePreloader.preload(this)
    }
}
```

## so加载 
1.加载耗时：  
```
System.loadLibrary("xxx") 内部流程：

① 查找 so 文件路径
   /data/app/com.xxx/lib/arm64/libxxx.so

② 打开文件，读取 ELF 头
   验证文件格式、架构匹配

③ mmap 映射到内存
   将 so 文件映射到进程地址空间

④ 动态链接
   解析符号表
   重定位（修正函数地址）
   处理依赖的其他 so

⑤ 执行 JNI_OnLoad
   so 的初始化逻辑

耗时分布：
├── 小型 so（<1MB）：5~20ms
├── 中型 so（1~5MB）：20~100ms
└── 大型 so（>5MB，如 Flutter engine）：100~500ms
```

2.加载优化：  
```
// 策略一：按需加载（最重要）
object SoLoader {

    private val loadedLibs = mutableSetOf<String>()

    fun loadIfNeeded(libName: String) {
        if (libName !in loadedLibs) {
            System.loadLibrary(libName)
            loadedLibs.add(libName)
        }
    }
}

// 策略二：异步预加载（Splash期间）
object SoPreloader {

    fun preloadAsync(vararg libNames: String) {
        // so 加载必须在主线程？
        // 不是！so 加载可以在子线程
        // 但 JNI_OnLoad 中如果调用了 JNI 函数需要注意线程安全
        GlobalScope.launch(Dispatchers.IO) {
            libNames.forEach { name ->
                try {
                    System.loadLibrary(name)
                } catch (e: UnsatisfiedLinkError) {
                    Log.e("SoPreloader", "加载失败: $name")
                }
            }
        }
    }
}

// 策略三：so 合并（减少加载次数）
// 将多个小 so 合并为一个大 so
// 减少动态链接次数
// 通过 CMakeLists.txt 配置：
// add_library(merged_lib SHARED
//     lib_a.cpp
//     lib_b.cpp
//     lib_c.cpp
// )

// 策略四：只保留必要的 ABI
// build.gradle
android {
    defaultConfig {
        ndk {
            // 只保留 arm64-v8a
            // 覆盖市面上 95%+ 的设备
            // 包体积减少约 50%（相比保留全部ABI）
            abiFilters "arm64-v8a"
        }
    }
}
```
## 网络连接
1.DNS 预解析：  
```
// 冷启动后第一个网络请求的耗时：
// DNS解析（20~200ms）+ TCP握手（20~100ms）+ TLS握手（20~100ms）+ 请求响应
// 预连接可以消除前三项耗时

object NetworkPreconnect {

    // App 的主要域名
    private val domains = listOf(
        "api.example.com",
        "cdn.example.com",
        "static.example.com"
    )

    fun preconnect(context: Context) {
        GlobalScope.launch(Dispatchers.IO) {

            // 方式一：DNS 预解析
            // 提前解析域名，缓存 IP
            domains.forEach { domain ->
                try {
                    InetAddress.getAllByName(domain) // 触发DNS解析并缓存
                } catch (e: Exception) {
                    // DNS 失败不影响启动
                }
            }

            // 方式二：TCP 预连接（OkHttp连接池预热）
            preWarmOkHttpConnectionPool()
        }
    }

    private fun preWarmOkHttpConnectionPool() {
        // 发起一个轻量级请求，建立连接并放入连接池
        // 后续真正的业务请求直接复用连接
        try {
            val request = Request.Builder()
                .url("https://api.example.com/ping") // 轻量级ping接口
                .head() // HEAD请求，无响应体，流量最小
                .build()

            OkHttpClient.getInstance()
                .newCall(request)
                .execute()
                .close()
        } catch (e: Exception) {
            // 预连接失败不影响业务
        }
    }
}
```

2.接口预请求：  
```
// 在 Splash 展示期间，提前请求首页数据
// 进入首页时数据已经准备好，直接展示

object DataPrefetcher {

    // 预取的数据缓存
    private var prefetchedHomeData: Deferred<HomeData?>? = null

    fun prefetch(context: Context) {
        // 使用 async 发起请求，但不等待结果
        prefetchedHomeData = GlobalScope.async(Dispatchers.IO) {
            try {
                ApiService.getInstance().getHomeData()
            } catch (e: Exception) {
                null // 预取失败，后续正常请求
            }
        }
    }

    // 首页 ViewModel 中获取预取数据
    suspend fun getPrefetchedData(): HomeData? {
        return try {
            // 如果预取完成，立即返回
            // 如果还在请求中，等待完成（通常 Splash 时间足够）
            prefetchedHomeData?.await()
        } catch (e: Exception) {
            null
        } finally {
            prefetchedHomeData = null // 用完清除
        }
    }
}

// HomeViewModel 中使用
class HomeViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            // 优先使用预取数据
            val prefetched = DataPrefetcher.getPrefetchedData()
            if (prefetched != null) {
                // 直接展示，无需等待网络
                _uiState.value = UiState.Success(prefetched)
                return@launch
            }
            // 降级：正常请求
            fetchFromNetwork()
        }
    }
}
```
## CPU 调度
### 升频
拉高 CPU 频率，绑定大核，约定时间（1s）进行关键初始化。  
避免启动阶段大量 IO（IO 不受 CPU Boost 影响）  

```
// 大核 vs 小核：
// 现代 ARM 处理器采用 big.LITTLE 架构
// 大核：高性能，高功耗
// 小核：低性能，低功耗
//
// 启动阶段：希望使用大核加速
// 厂商通常会在点击图标时短暂提升 CPU 频率（Boost）
// 开发者可以通过以下方式配合：

object CpuBoostHelper {

    fun requestBoost() {
        // 方式一：通过厂商 SDK 申请 Boost（需要合作）
        // 小米：MiuiBoostFramework
        // 华为：HwBoostFramework
        // OPPO：OplusBoostFramework

        // 方式二：通过 PowerManager 申请性能模式
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            val powerManager = context.getSystemService(PowerManager::class.java)
            // 申请高性能模式（需要权限）
            // 仅在启动关键路径上使用，用完立即释放
        }

        // 方式三：确保在 Boost 窗口内完成关键初始化
        // 系统 Boost 通常持续 1~2 秒
        // 关键初始化必须在这个窗口内完成
    }
}
```
## GC 抑制
adb shell logcat | grep "GC_" 查看启动期间 GC 情况  
```
// GC 卡顿原因：
// ART GC 分为：
// ├── 并发GC（Concurrent GC）：大部分工作与App并发，STW极短
// ├── 半空间GC（Semi-space GC）：STW较长，内存紧张时触发
// └── 标记清除GC（Mark-Sweep）：STW中等

// 触发GC的主要原因：
// ├── 频繁创建短生命周期对象（内存抖动）
// ├── 大对象分配（直接进入老年代）
// └── 内存不足
```
### 对象预分配
预创建并复用的对象（对象池）  
```
class ObjectPool<T>(
    private val maxSize: Int = 10,
    private val factory: () -> T,
    private val reset: (T) -> Unit = {}
) {
    private val pool = ArrayDeque<T>(maxSize)

    fun acquire(): T {
        return if (pool.isEmpty()) {
            factory() // 池空了，创建新对象
        } else {
            pool.removeFirst() // 从池中取
        }
    }

    fun release(obj: T) {
        if (pool.size < maxSize) {
            reset(obj) // 重置状态
            pool.addLast(obj) // 归还到池
        }
        // 池满了，让GC回收
    }
}

// 使用对象池
val rectPool = ObjectPool(
    factory = { RectF() },
    reset = { it.setEmpty() }
)
```
### 大对象优化
```
// 大对象直接进入老年代，触发 Full GC（STW 更长）

// ================================
// Bitmap 内存优化
// ================================
object BitmapOptimizer {

    // 使用 BitmapPool 复用 Bitmap 内存
    // Glide 内部已实现，直接使用 Glide 加载图片即可

    // 手动复用 Bitmap（inBitmap）
    fun decodeBitmapWithReuse(
        path: String,
        reuseBitmap: Bitmap?
    ): Bitmap {
        val options = BitmapFactory.Options().apply {
            inMutable = true // 允许复用
            inBitmap = reuseBitmap // 复用已有 Bitmap 的内存
        }
        return BitmapFactory.decodeFile(path, options)
    }

    // 按需加载合适分辨率
    fun decodeSampledBitmap(
        path: String,
        reqWidth: Int,
        reqHeight: Int
    ): Bitmap {
        val options = BitmapFactory.Options().apply {
            inJustDecodeBounds = true
            BitmapFactory.decodeFile(path, this)

            inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)
            inJustDecodeBounds = false
            // RGB_565：每像素2字节（vs ARGB_8888的4字节）
            // 无透明度需求时使用，内存减半
            inPreferredConfig = Bitmap.Config.RGB_565
        }
        return BitmapFactory.decodeFile(path, options)
    }

    private fun calculateInSampleSize(
        options: BitmapFactory.Options,
        reqWidth: Int,
        reqHeight: Int
    ): Int {
        val (height, width) = options.run { outHeight to outWidth }
        var inSampleSize = 1
        if (height > reqHeight || width > reqWidth) {
            val halfHeight = height / 2
            val halfWidth = width / 2
            while (halfHeight / inSampleSize >= reqHeight &&
                   halfWidth / inSampleSize >= reqWidth) {
                inSampleSize *= 2
            }
        }
        return inSampleSize
    }
}
```
### Tips
1. 减少临时对象创建  
2. 避免大对象分配（直接触发 GC）  
3. 预分配关键对象（启动前置）
4.锁屏 GC  
5.需要资源的特殊场景不 GC  
6.后台 GC  
但是，正常来讲 GC 是系统决定的，非必要应用侧不应该主动调用显式 GC，而是去分析内存问题
## 监控
```
┌─────────────────────────────────────────────────────────┐
│                    启动监控体系                           │
├─────────────────────────────────────────────────────────┤
│  数据采集层                                               │
│  ├── 启动类型识别（冷/温/热）                             │
│  ├── 分阶段耗时采集                                       │
│  ├── 设备信息采集                                         │
│  └── 异常信息采集                                         │
├─────────────────────────────────────────────────────────┤
│  数据上报层                                               │
│  ├── 实时上报（关键指标）                                  │
│  ├── 批量上报（详细数据）                                  │
│  └── 采样策略（控制上报量）                               │
├─────────────────────────────────────────────────────────┤
│  数据分析层                                               │
│  ├── P50 / P90 / P99 分位数                              │
│  ├── 多维度拆分（机型/系统/版本）                         │
│  └── 趋势分析（版本对比）                                  │
├─────────────────────────────────────────────────────────┤
│  报警层                                                   │
│  ├── 阈值报警                                             │
│  ├── 环比劣化报警                                         │
│  └── 灰度监控                                             │
└─────────────────────────────────────────────────────────┘
```
### 核心指标
1.冷启动 P50 / P90 / P99 耗时
2.首屏展示时间（reportFullyDrawn）
3.启动成功率（启动期间 Crash 率）
4.启动期间 ANR 率
#### 分段指标
1.Application 耗时
2.首个 Activity onCreate 耗时
3.布局 inflate 耗时
4.首帧渲染耗时
### 数据采集
1.启动节点：  
```
T0: 进程创建时间
    └── Application.attachBaseContext() 中记录

T1: Application.onCreate() 开始
T2: Application.onCreate() 结束
    └── T2 - T1 = Application 初始化耗时

T3: 首个 Activity.onCreate() 开始
T4: setContentView() 完成
    └── T4 - T3 = 布局加载耗时

T5: onResume() 完成
T6: 首帧上屏（ViewTreeObserver.OnPreDrawListener）
    └── T6 - T5 = 首帧渲染耗时

T7: reportFullyDrawn()（真实内容展示完成）
    └── T7 - T6 = 数据加载耗时

关键指标：
├── 总启动耗时 = T7 - T0（冷启动）
├── Application 耗时 = T2 - T1
├── Activity 耗时 = T6 - T3
└── 首屏完整耗时 = T7 - T0
```

2.采集：  
```
// ================================
// 启动时间记录器
// ================================
object StartupTracer {

    // 所有时间点（使用 elapsedRealtime，不受系统时间调整影响）
    private val timePoints = ConcurrentHashMap<String, Long>()

    // 分阶段耗时（任务级别）
    private val stageCosts = ConcurrentHashMap<String, Long>()

    // 启动类型
    var startupType: StartupType = StartupType.COLD

    // ================================
    // 记录时间点
    // ================================
    fun mark(point: TimePoint) {
        timePoints[point.key] = SystemClock.elapsedRealtime()
    }

    fun markWithValue(point: TimePoint, timeMs: Long) {
        timePoints[point.key] = timeMs
    }

    // ================================
    // 记录阶段耗时（用于任务级监控）
    // ================================
    fun recordStage(stageName: String, costMs: Long) {
        stageCosts[stageName] = costMs
    }

    inline fun <T> traceStage(stageName: String, block: () -> T): T {
        val start = SystemClock.elapsedRealtime()
        return try {
            block()
        } finally {
            val cost = SystemClock.elapsedRealtime() - start
            recordStage(stageName, cost)
        }
    }

    // ================================
    // 计算各阶段耗时
    // ================================
    fun buildReport(): StartupReport {
        val t0 = timePoints[TimePoint.PROCESS_START.key] ?: 0L
        val t1 = timePoints[TimePoint.APP_CREATE_START.key] ?: 0L
        val t2 = timePoints[TimePoint.APP_CREATE_END.key] ?: 0L
        val t3 = timePoints[TimePoint.ACTIVITY_CREATE_START.key] ?: 0L
        val t4 = timePoints[TimePoint.SET_CONTENT_VIEW_END.key] ?: 0L
        val t5 = timePoints[TimePoint.ON_RESUME_END.key] ?: 0L
        val t6 = timePoints[TimePoint.FIRST_FRAME.key] ?: 0L
        val t7 = timePoints[TimePoint.FULLY_DRAWN.key] ?: 0L

        return StartupReport(
            startupType = startupType,

            // 核心指标
            totalCostMs = t7 - t0,
            ttidMs = t6 - t0,
            ttfdMs = t7 - t0,

            // 分阶段指标
            processCreateMs = t1 - t0,
            appCreateMs = t2 - t1,
            activityCreateMs = t4 - t3,
            firstFrameMs = t6 - t5,
            dataLoadMs = t7 - t6,

            // 任务级明细
            stageCosts = stageCosts.toMap(),

            // 设备信息
            deviceInfo = DeviceInfo.collect()
        )
    }

    fun reset() {
        timePoints.clear()
        stageCosts.clear()
    }
}

// 时间点枚举
enum class TimePoint(val key: String) {
    PROCESS_START("process_start"),
    APP_CREATE_START("app_create_start"),
    APP_CREATE_END("app_create_end"),
    ACTIVITY_CREATE_START("activity_create_start"),
    SET_CONTENT_VIEW_END("set_content_view_end"),
    ON_RESUME_END("on_resume_end"),
    FIRST_FRAME("first_frame"),
    FULLY_DRAWN("fully_drawn")
}

enum class StartupType { COLD, WARM, HOT }
```

3.埋点：  
```
// ================================
// 启动时间记录器
// ================================
object StartupTracer {

    // 所有时间点（使用 elapsedRealtime，不受系统时间调整影响）
    private val timePoints = ConcurrentHashMap<String, Long>()

    // 分阶段耗时（任务级别）
    private val stageCosts = ConcurrentHashMap<String, Long>()

    // 启动类型
    var startupType: StartupType = StartupType.COLD

    // ================================
    // 记录时间点
    // ================================
    fun mark(point: TimePoint) {
        timePoints[point.key] = SystemClock.elapsedRealtime()
    }

    fun markWithValue(point: TimePoint, timeMs: Long) {
        timePoints[point.key] = timeMs
    }

    // ================================
    // 记录阶段耗时（用于任务级监控）
    // ================================
    fun recordStage(stageName: String, costMs: Long) {
        stageCosts[stageName] = costMs
    }

    inline fun <T> traceStage(stageName: String, block: () -> T): T {
        val start = SystemClock.elapsedRealtime()
        return try {
            block()
        } finally {
            val cost = SystemClock.elapsedRealtime() - start
            recordStage(stageName, cost)
        }
    }

    // ================================
    // 计算各阶段耗时
    // ================================
    fun buildReport(): StartupReport {
        val t0 = timePoints[TimePoint.PROCESS_START.key] ?: 0L
        val t1 = timePoints[TimePoint.APP_CREATE_START.key] ?: 0L
        val t2 = timePoints[TimePoint.APP_CREATE_END.key] ?: 0L
        val t3 = timePoints[TimePoint.ACTIVITY_CREATE_START.key] ?: 0L
        val t4 = timePoints[TimePoint.SET_CONTENT_VIEW_END.key] ?: 0L
        val t5 = timePoints[TimePoint.ON_RESUME_END.key] ?: 0L
        val t6 = timePoints[TimePoint.FIRST_FRAME.key] ?: 0L
        val t7 = timePoints[TimePoint.FULLY_DRAWN.key] ?: 0L

        return StartupReport(
            startupType = startupType,

            // 核心指标
            totalCostMs = t7 - t0,
            ttidMs = t6 - t0,
            ttfdMs = t7 - t0,

            // 分阶段指标
            processCreateMs = t1 - t0,
            appCreateMs = t2 - t1,
            activityCreateMs = t4 - t3,
            firstFrameMs = t6 - t5,
            dataLoadMs = t7 - t6,

            // 任务级明细
            stageCosts = stageCosts.toMap(),

            // 设备信息
            deviceInfo = DeviceInfo.collect()
        )
    }

    fun reset() {
        timePoints.clear()
        stageCosts.clear()
    }
}

// 时间点枚举
enum class TimePoint(val key: String) {
    PROCESS_START("process_start"),
    APP_CREATE_START("app_create_start"),
    APP_CREATE_END("app_create_end"),
    ACTIVITY_CREATE_START("activity_create_start"),
    SET_CONTENT_VIEW_END("set_content_view_end"),
    ON_RESUME_END("on_resume_end"),
    FIRST_FRAME("first_frame"),
    FULLY_DRAWN("fully_drawn")
}

enum class StartupType { COLD, WARM, HOT }


// ================================
// Activity 生命周期自动埋点
// ================================
class StartupLifecycleCallback : Application.ActivityLifecycleCallbacks {

    // 只监控启动的第一个 Activity
    private var isFirstActivity = true

    override fun onActivityCreated(
        activity: Activity,
        savedInstanceState: Bundle?
    ) {
        if (!isFirstActivity) return
        isFirstActivity = false

        // 判断启动类型
        StartupTracer.startupType = when {
            savedInstanceState != null -> StartupType.WARM
            else -> StartupType.COLD
            // 热启动在 onRestart 中判断
        }

        StartupTracer.mark(TimePoint.ACTIVITY_CREATE_START)

        // 监听 setContentView 完成
        // 通过 OnContentChangedCallback 实现
        if (activity is AppCompatActivity) {
            activity.addContentChangedCallback()
        }
    }

    override fun onActivityStarted(activity: Activity) {
        // 热启动检测
        if (activity.isResumedFromBackground()) {
            StartupTracer.startupType = StartupType.HOT
            StartupTracer.mark(TimePoint.ACTIVITY_CREATE_START)
        }
    }

    override fun onActivityResumed(activity: Activity) {
        if (!isMonitoringActivity(activity)) return
        StartupTracer.mark(TimePoint.ON_RESUME_END)

        // 注册首帧回调
        registerFirstFrameCallback(activity)
    }

    private fun registerFirstFrameCallback(activity: Activity) {
        activity.window.decorView.viewTreeObserver
            .addOnPreDrawListener(object : ViewTreeObserver.OnPreDrawListener {
                override fun onPreDraw(): Boolean {
                    activity.window.decorView.viewTreeObserver
                        .removeOnPreDrawListener(this)

                    StartupTracer.mark(TimePoint.FIRST_FRAME)

                    // 首帧完成，可以上报 TTID
                    reportTTID()
                    return true
                }
            })
    }

    // 其他回调省略
    override fun onActivityPaused(activity: Activity) {}
    override fun onActivityStopped(activity: Activity) {}
    override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
    override fun onActivityDestroyed(activity: Activity) {}
}


// ================================
// Activity 中的 TTFD 上报
// ================================
class HomeActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)
        StartupTracer.mark(TimePoint.SET_CONTENT_VIEW_END)

        showSkeleton()

        viewModel.uiState.observe(this) { state ->
            if (state is UiState.Success) {
                renderContent(state.data)
                hideSkeleton()

                // T7：真实内容展示完成
                StartupTracer.mark(TimePoint.FULLY_DRAWN)
                reportFullyDrawn() // 通知系统

                // 上报完整启动数据
                reportStartup()
            }
        }
    }

    private fun reportStartup() {
        val report = StartupTracer.buildReport()
        StartupReporter.report(report)
        StartupTracer.reset() // 重置，准备下次启动
    }
}
```

4.设备信息：  
```
// ================================
// 设备信息（用于多维度分析）
// ================================
data class DeviceInfo(
    val model: String,           // 机型
    val brand: String,           // 品牌
    val androidVersion: Int,     // Android 版本
    val sdkVersion: Int,         // SDK 版本
    val cpuCores: Int,           // CPU 核心数
    val totalRamMB: Long,        // 总内存
    val availableRamMB: Long,    // 可用内存
    val isLowEndDevice: Boolean, // 是否低端机
    val networkType: String,     // 网络类型
    val appVersion: String,      // App 版本
    val isFirstLaunch: Boolean,  // 是否首次启动
    val batteryLevel: Int        // 电量（低电量影响性能）
) {
    companion object {
        fun collect(): DeviceInfo {
            val context = AppContext.get()
            val am = context.getSystemService(ActivityManager::class.java)
            val memInfo = ActivityManager.MemoryInfo()
            am.getMemoryInfo(memInfo)

            return DeviceInfo(
                model = Build.MODEL,
                brand = Build.BRAND,
                androidVersion = Build.VERSION.RELEASE.toIntOrNull() ?: 0,
                sdkVersion = Build.VERSION.SDK_INT,
                cpuCores = Runtime.getRuntime().availableProcessors(),
                totalRamMB = memInfo.totalMem / 1024 / 1024,
                availableRamMB = memInfo.availMem / 1024 / 1024,
                isLowEndDevice = am.isLowRamDevice,
                networkType = getNetworkType(context),
                appVersion = BuildConfig.VERSION_NAME,
                isFirstLaunch = isFirstLaunch(context),
                batteryLevel = getBatteryLevel(context)
            )
        }

        private fun getNetworkType(context: Context): String {
            val cm = context.getSystemService(ConnectivityManager::class.java)
            return when {
                cm.activeNetwork == null -> "none"
                cm.getNetworkCapabilities(cm.activeNetwork)
                    ?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true -> "wifi"
                else -> "cellular"
            }
        }
    }
}
```
### 数据上报
1.数据结构：  
```
data class StartupReport(
    // 基础信息
    val startupType: StartupType,
    val timestamp: Long = System.currentTimeMillis(),
    val sessionId: String = UUID.randomUUID().toString(),

    // 核心指标（毫秒）
    val totalCostMs: Long,    // 总启动耗时
    val ttidMs: Long,         // Time To Initial Display
    val ttfdMs: Long,         // Time To Full Display

    // 分阶段指标
    val processCreateMs: Long,  // 进程创建耗时
    val appCreateMs: Long,      // Application 初始化耗时
    val activityCreateMs: Long, // Activity 创建耗时
    val firstFrameMs: Long,     // 首帧渲染耗时
    val dataLoadMs: Long,       // 数据加载耗时

    // 任务级明细
    val stageCosts: Map<String, Long>,

    // 设备信息
    val deviceInfo: DeviceInfo,

    // 异常信息
    val hasException: Boolean = false,
    val exceptionMsg: String? = null
)
```

2.采样策略：  
```
// ================================
// 采样上报（控制上报量，节省流量和服务器压力）
// ================================
object StartupReporter {

    // 采样率配置（从服务端下发，可动态调整）
    private var samplingRate = 0.1f  // 默认10%采样

    // 低端机全量上报（低端机问题更多，需要更多数据）
    private var lowEndSamplingRate = 1.0f

    fun report(report: StartupReport) {
        val rate = if (report.deviceInfo.isLowEndDevice) {
            lowEndSamplingRate
        } else {
            samplingRate
        }

        // 采样判断
        if (Math.random() > rate) return

        // 慢启动全量上报（不受采样限制）
        val isSlowStartup = when (report.startupType) {
            StartupType.COLD -> report.totalCostMs > 3000
            StartupType.WARM -> report.totalCostMs > 1500
            StartupType.HOT  -> report.totalCostMs > 500
        }

        if (isSlowStartup || Math.random() <= rate) {
            doReport(report)
        }
    }

    private fun doReport(report: StartupReport) {
        // 序列化为 JSON
        val json = buildReportJson(report)

        // 批量上报（积累一定数量后一次性发送）
        ReportBatcher.add("startup", json)
    }

    private fun buildReportJson(report: StartupReport): JSONObject {
        return JSONObject().apply {
            put("type", report.startupType.name)
            put("total_ms", report.totalCostMs)
            put("ttid_ms", report.ttidMs)
            put("ttfd_ms", report.ttfdMs)
            put("app_create_ms", report.appCreateMs)
            put("activity_create_ms", report.activityCreateMs)
            put("first_frame_ms", report.firstFrameMs)
            put("data_load_ms", report.dataLoadMs)

            // 分阶段明细
            put("stages", JSONObject(report.stageCosts))

            // 设备信息
            put("device_model", report.deviceInfo.model)
            put("android_version", report.deviceInfo.androidVersion)
            put("ram_mb", report.deviceInfo.totalRamMB)
            put("is_low_end", report.deviceInfo.isLowEndDevice)
            put("network", report.deviceInfo.networkType)
            put("app_version", report.deviceInfo.appVersion)
        }
    }
}
```

3.批量上报：  
```
// ================================
// 批量上报器（减少网络请求次数）
// ================================
object ReportBatcher {

    private val buffer = ConcurrentLinkedQueue<Pair<String, JSONObject>>()
    private const val MAX_BATCH_SIZE = 20
    private const val FLUSH_INTERVAL_MS = 30_000L // 30秒

    init {
        // 定期 flush
        GlobalScope.launch(Dispatchers.IO) {
            while (true) {
                delay(FLUSH_INTERVAL_MS)
                flush()
            }
        }
    }

    fun add(type: String, data: JSONObject) {
        buffer.offer(Pair(type, data))

        // 达到批量大小立即上报
        if (buffer.size >= MAX_BATCH_SIZE) {
            GlobalScope.launch(Dispatchers.IO) { flush() }
        }
    }

    private suspend fun flush() {
        if (buffer.isEmpty()) return

        val batch = mutableListOf<Pair<String, JSONObject>>()
        while (batch.size < MAX_BATCH_SIZE) {
            batch.add(buffer.poll() ?: break)
        }

        try {
            APMApi.reportBatch(batch)
        } catch (e: Exception) {
            // 上报失败，重新入队
            batch.forEach { buffer.offer(it) }
        }
    }
}
```
### 数据分析
1.核心维度：  
```
线上数据分析维度：

① 时间维度
   ├── 按小时/天/周趋势
   ├── 版本发布前后对比
   └── 灰度版本 vs 全量版本

② 设备维度
   ├── 机型 Top 20 分布
   ├── Android 版本分布
   ├── 高/中/低端机分层
   └── 内存大小分层

③ 网络维度
   ├── WiFi vs 4G vs 5G
   └── 弱网场景

④ 业务维度
   ├── 新用户 vs 老用户
   ├── 首次安装 vs 更新后
   └── 不同入口（图标/通知/外链）
```

2.分位计算：  
```
// 为什么用分位数而不是平均值？
// 平均值会被极端值拉偏
// 例：99个用户800ms，1个用户30000ms
//     平均值 = (99*800 + 30000) / 100 = 1092ms
//     P99 = 30000ms（真实反映最差体验）
//     P50 = 800ms（反映大多数用户体验）

// 服务端计算分位数（伪代码）
fun calculatePercentiles(values: List<Long>): PercentileResult {
    val sorted = values.sorted()
    val size = sorted.size

    return PercentileResult(
        p50 = sorted[(size * 0.50).toInt()],
        p75 = sorted[(size * 0.75).toInt()],
        p90 = sorted[(size * 0.90).toInt()],
        p95 = sorted[(size * 0.95).toInt()],
        p99 = sorted[(size * 0.99).toInt()]
    )
}

// 上报时需要上报原始值，服务端计算分位数
// 客户端不做分位数计算（需要大量数据才有意义）
```
### 告警
1.告警策略：  
```
// ================================
// 报警规则配置
// ================================
data class AlertRule(
    val name: String,
    val startupType: StartupType,
    val metric: String,          // 监控指标
    val percentile: Int,         // 分位数（50/90/99）
    val threshold: Long,         // 绝对阈值（ms）
    val degradationRate: Float,  // 环比劣化率（0.1 = 10%）
    val minSampleSize: Int       // 最小样本量（样本太少不报警）
)

// 报警规则示例
val alertRules = listOf(
    // 冷启动 P90 超过 3000ms 报警
    AlertRule(
        name = "cold_start_p90_threshold",
        startupType = StartupType.COLD,
        metric = "total_ms",
        percentile = 90,
        threshold = 3000,
        degradationRate = Float.MAX_VALUE, // 不检查环比
        minSampleSize = 1000
    ),

    // 冷启动 P50 环比劣化超过 10% 报警
    AlertRule(
        name = "cold_start_p50_degradation",
        startupType = StartupType.COLD,
        metric = "total_ms",
        percentile = 50,
        threshold = Long.MAX_VALUE, // 不检查绝对阈值
        degradationRate = 0.1f,     // 10% 劣化
        minSampleSize = 1000
    ),

    // Application 初始化 P90 超过 500ms 报警
    AlertRule(
        name = "app_create_p90_threshold",
        startupType = StartupType.COLD,
        metric = "app_create_ms",
        percentile = 90,
        threshold = 500,
        degradationRate = Float.MAX_VALUE,
        minSampleSize = 500
    )
)
```

2.灰度监控：  
```
// ================================
// 灰度发布期间的监控策略
// ================================
object GrayMonitor {

    // 灰度期间提高采样率
    fun onGrayRelease() {
        StartupReporter.samplingRate = 1.0f  // 全量采样
        StartupReporter.lowEndSamplingRate = 1.0f
    }

    // 灰度结束恢复正常采样
    fun onGrayEnd() {
        StartupReporter.samplingRate = 0.1f
    }

    // 灰度版本 vs 老版本实时对比
    // 服务端按 version_name 分组计算指标
    // 如果灰度版本 P50 劣化 > 5% → 立即报警
    // 可以快速回滚，避免影响全量用户
}
```
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
-keepattributes *Annotation*
-keepattributes Signature
-keepattributes SourceFile,LineNumberTable  # 保留行号（崩溃定位）

# 保留四大组件
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider

# ================================
# 反射相关保留
# ================================

# 保留通过反射访问的类/方法/字段
-keepclassmembers class com.xxx.model.** {
    <fields>;
}

# 保留 JSON 序列化的模型类
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

4.无用代码分析：  
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

5.Kotlin 代码优化：  
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
// 优化二：内联函数
// ================================

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

// ================================
// 优化三：避免 Kotlin 扩展函数滥用
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
// 优化四：data class 精简
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
### 无用代码裁剪（Tree Shaking）
### 内联优化
### Dex 分包优化
## 资源优化
### 图片压缩（WebP/AVIF）
### 无用资源删除（shrinkResources）
### 资源混淆（AndResGuard）
### 重复资源去除
### 语言/分辨率裁剪
## so 库优化
### ABI 过滤（只保留 arm64-v8a）
### So 压缩与动态下发
### 符号裁剪（strip）
### 合并 so
## 架构优化
### 插件化（功能动态下发）
### AAB（Android App Bundle）
### 按需下载资源
## 编译优化
### D8/R8 编译器
### Baseline Profile
### 压缩算法选择
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
