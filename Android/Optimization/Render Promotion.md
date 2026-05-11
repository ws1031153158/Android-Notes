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
