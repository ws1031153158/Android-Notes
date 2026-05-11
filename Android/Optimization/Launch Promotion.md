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
