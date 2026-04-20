# Compose VS XML
XML：                        Compose：  
描述"长什么样"               描述"状态下长什么样"  
View是对象，可以直接操作      UI是函数，状态变了自动重绘  
findViewById拿到View改它     改状态，UI自动跟着变  
布局和逻辑分离               布局和逻辑在一起  
## XML
静态配置文件，缺点多：  
### 命令式、静态化
XML 只写 “长什么样”，状态更新、条件显示、列表逻辑必须靠 Java/Kotlin 代码手动，findViewById、setText、setVisibility，容易漏更、不一致。  
### 分离割裂
布局（XML）、逻辑（代码）、样式（styles.xml）分散在多个文件，维护、跳转、重构极麻烦。  
### 嵌套性能差
多层 <LinearLayout>/<RelativeLayout> 嵌套会导致递归测量 / 布局深度变深、过度绘制、内存开销大。    
LinearLayout ↓ 测量 RelativeLayout ↓ 测量 ConstraintLayout ↓ 测量 TextView  
每一层都是独立 ViewGroup，都要走完整的：onMeasure → onLayout → 父通知子 → 子回报父  
### 逐层渲染 
LayoutInflater 解析 XML →  创建真实 View 对象（重量级） →  ViewGroup 嵌套 →  onMeasure（测量自己和孩子） →  onLayout（布局：摆放孩子位置） →  onDraw（绘制内容） → 刷新 invalidate → 交给 Android Render 渲染 → 反复触发   
最终渲染到屏幕
### 无类型安全
ID、字符串、尺寸都是硬编码，编译不检查，运行时才报错。  
### 无法利用 Kotlin
不能用高阶函数、DSL、条件表达式、扩展函数等现代特性。  
## Compose
纯 Kotlin 声明式，解决XML缺点：  
### UI= f(State)
界面 ＝ 函数 (状态)，直接在 Kotlin 里写界面结构 + 逻辑 + 样式，状态变自动重组，不用手动更新。    
核心机制：快照系统（Snapshot）+ 重组（Recomposition）：  
1.定义一个可观察状态：（var count by remember { mutableStateOf(0) }）  
2.建立依赖关系：Compose 在执行 @Composable 时，会自动记录哪些函数用到了这个 state。  
3.执行count++：Snapshot 系统检测到状态写入。  
4.系统找到所有用到这个 state 的函数，把它们标记为需要重组。  
5.在下一帧，Compose 只重新执行这些函数，刷新对应 UI。
### 一体化
布局、逻辑、样式、动画在同一个函数 / 文件，内聚度极高。  
### 扁平高效
Compose 自带 Column/Row/Box/LazyColumn，天然扁平、少嵌套、测量布局更快。  
内部是单层 LayoutNode 树，一次测量完成，不需要多层 ViewGroup 嵌套。  
逻辑嵌套，不是测量嵌套：  
1.底层只有 LayoutNode 树   
2.所有节点在同一个测量阶段完成   
3.没有多层 ViewGroup 来回通信   
4.没有多次重复测量  
5.布局指令更紧凑，天然扁平化  
### 类型安全、编译检查
所有参数强类型，IDE 实时提示、报错。  
### Kotlin 原生
充分利用函数式、DSL、协程、Flow 等特性。  
### 底层实现
全新的 runtime + 渲染管线，不基于传统 View 体系，也不解析任何 XML：  
1.核心三层（无 XML）：
Runtime：组合（Composition）、重组（Recomposition）、状态（Snapshot）、SlotTable（UI 树数据结构）。   
UI：LayoutNode、测量 / 布局 / 绘制、Modifier。   
Foundation / Material：组件库（Text、Button、Column 等）。  
<img width="451" height="689" alt="image" src="https://github.com/user-attachments/assets/57211d99-6a6c-4045-b5f0-fb06bdb950d7" />
2.渲染三阶段：  
组合（Composition）：执行 @Composable 函数 → 生成 SlotTable → 构建 LayoutNode 树  
布局（Layout）：测量每个节点大小 → 确定坐标位置  
绘制（Draw）：用 Canvas/GPU 直接渲染，不经过 View/ViewGroup  
3.与 XML/View 体系的关系：  
底层无关：不解析任何 XML，不继承 View，不使用 LayoutInflate。  
上层互通（兼容层）：XML 里可以放 ComposeView 加载 Compose ，Compose 里可以用 AndroidView 包裹传统 View。  
<img width="998" height="482" alt="image" src="https://github.com/user-attachments/assets/066fc5fd-e024-4b31-9315-eca72d8cf855" />
#### UI引擎
1.Compose 自己不用 View/ViewGroup，内部是：LayoutNode 、Modifier 、LayoutCoordinates ，自己的测量、布局、绘制，抛弃了 View 体系回调，全部用 Modifier + Scope 替代：    
compose全部整合进一个 Layout() 组件 + Modifier.drawBehind：   
测量 = measurable.measure(constraints)   
布局 = placeable.place(x,y) 绘制 = Modifier.drawBehind 点击 = pointerInput  
onLayout = onGloballyPositioned   
### 直接渲染
跳过 View 体系，直接测量布局绘制：  
不解析XML、不生成View树（无View对象）、构建LayoutNode（只有轻量数据结构：SlotTable + LayoutNode） 、自己测量/布局（自顶向下一次完成）、直接调用GraphicBuffer / Canvas绘制（直接提交给引擎，不经View体系、无ViewRoot、invalidate）：  
执行 @Composable 函数 生成 SlotTable 结构 → 构建 LayoutNode 树 → Compose 自己测量（MeasureScope） → Compose 自己布局（LayoutCoordinates） → Compose 自己绘制（DrawScope） → 直接提交渲染指令
## 系统生命周期
onConfigurationChanged 等依然存在：  
1.在 Activity 里正常写逻辑  
2.在 Compose 里用：val configuration = LocalConfiguration.current，状态一变，Compose 自动重组

