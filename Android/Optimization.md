
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
