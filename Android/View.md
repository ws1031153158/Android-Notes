# ViewRoot & DecorView
ViewRoot 就是 ViewRootImpl，连接Window 和 View（这里是DecorView），三大流程都由 ViewRootImpl 完成。  
activity 创建完成后 将 decorview 添加到 window 中，创建 ViewRootImpl 并建立和 decorview 的连接（root.setView）
# draw
## process
activity 在 onCreate 时调用 setContentView，这一步将 layout 放入 window 关联 root 的 content 当中，接着 AT 会继续执行 handleResumeActivity，这里会调用 wm.addView，此方法中创建一个 viewRootimpl 来对应 decorView（一对一），并通过 root.setView 进行关联，接下来就到了三大流程环节了（performMeasure、performLayout、performDraw）。首先会走一次 requestLayout，root 会调用 performMeasure，接着走到 decorView 的 measure 方法中进而调用 onMeasure 以及 child 的 onMeasure，后续的 layout 和 draw 按顺序执行。
## getWidth/getHeight
setContentView 在 onCreate 时执行，addView 在 onResume 时执行，所以这两个阶段无法获取宽高，可以通过 view.post 来拿
## onFinishInflate
在 setContentView 之后 onMeasure 之前调用，代表 view 中的子 view、控件等映射完成（从 xml 解析完毕），可以进行初始化了（如 findView 等获取控件和子 view）  
setContentView 阶段 decorView 和容纳 layout 的 content 初始化完毕，将 layout 添加到 content（一个 FrameLayout） 中（window.setContentView）
## invalidate & requestLayout
invalidate：会递归调用父 view 的 invalidateChildInParent（不会调用 root 的 invalidate），通过 dispatchDraw 分发事件，再调用 view 的 draw 方法，最终调用 onDraw  
requestLayout：递归调用父 window 的 requestLayout，不同的是会走 measure 流程，但不一定触发 onDraw（layout 过程 l、t、r、b 变化触发 invalidate，才会走到 onDraw）  
只需要刷新 view 则使用 inva，需要重新 measure 则使用 requestLayout，可以接一个 invalidate 保证重绘
## Measure
1.对于单独的 view 而言，measure 就可以确定大小（宽高）了，已经完成了对整个 view 的测量，但一般针对 viewGroup，还需要进行子 view 的测量  
2.measure 是 final 类型的，所以直接看 onMeasure 就可以。首先会 getDefaultSize 设置一下测量大小，获取 MeasureSpec 的模式和数值来指定测量大小。view 宽高很大程度由 specSize 决定，重写 onMeasure 需要设置 wrap_content 时自身大小，指定 mWidth、mHeight 并设置给 wrap_content，否则和 match_parent 没区别。  
3.viewGroup 没有实现 onMeasure，只有 measureChild，因为各个 layout 的方式都不同，需要各自处理。  
4.一般测量大小和最终大小是相同的，但有的场景会多次 measure，所以最好在 layout 时获取 view 宽高。  
5.层级加深 measure 会执行很多次，比如父 view 设置 wrap_content，子 view 设置 match_parent，父 view 会先以 0 作为强制宽度测量子 view，并正常测量其他子 view，再用最宽子 view 宽度，二次测量这个子 view ，得出尺寸并作为自己最终的宽度。多个子 view 设置 match_parent ，需要对每一个进行二次测量。
### MeasureSpec
父 view 会影响子 view 的 MeasureSpec 创建。  
32 位 Int，高 2 位标记 SpecMode，低 30 位标记 SpecSize。  
UNSPECIFIED：不对子 view 限制，一般用于系统内部标识一种测量状态。  
EXACTLY：指定精确数值，对应 match_parent。   
AT_MOST：指定子 view 可用大小，不大于即可，对应 wrap_content。  
EXACTLY + dp/px ：EXACTLY-childSize  
EXACTLY + match_parent：EXACTLY-parentSize  
EXACTLY +wrap_content：AT_MOST-parentSize  
AT_MOST + dp/px ：EXACTLY-childSize  
AT_MOST + match_parent：AT_MOST-parentSize  
AT_MOST + wrap_content：AT_MOST-parentSize  
UNSPECIFIED + dp/px ：EXACTLY-childSize  
UNSPECIFIED + match_parent：UNSPECIFIED-0  
UNSPECIFIED + wrap_content：UNSPECIFIED-0  
此外，若给 View 设置 LayoutParams 则在测量时由父容器的约束转换为对应的 MeasureSpec，即 MeasureSpec 由父容器的 MeasureSpec 和自身的 LayoutParams 共同决定
## Layout
确定 view 位置（ViewGroup 则是确定子元素位置，左上右下两点坐标），初始化四个顶点位置，调用 onLayout（确定子元素位置），因为和具体布局有关，所以 View 和 ViewGroup 都没实现 onLayout，都要子类去实现。getwidth 和 getHeight 获取的是顶点坐标的差，若在 layout 时加了 padding 或 margin 则最终宽高和测量宽高不一致。
## Draw
1.首先绘制 background  
2.接着调用自身的 onDraw  
3.再通过 dispathDraw 将事件分发给子 view  
4.最后绘制 view 的一些 decoration（如onDrawScrollBars）
# Touch
## onTouch
View 有，ViewGroup 没有的回调，需要绑定监听 onTouchListener，优先级高于 onTouchEvent，在返回 false 的情况下才会走 onTouchEvent，view 只有这个和 dispatchTouchEvent
## onTouchEvent
处理 down 事件后，后续所有事件都由自身处理，若不处理则后续事件不会再走到这里，此外，若不处理 up 事件则此次事件会丢失
## onDispatchTouchEvent
返回 true 则说明自身处理事件，返回 false 会分发给上层 view 处理
## onIntercepTouchEvent
返回 true 为自身处理事件，false 则分发给子 view 处理，viewGroup 独有回调
## process
1.清空异常及已有状态，给所有之前选择的 Target 发送 Cancel 事件，确保之前 Target 能收到完整的事件周期；清除已有T arget，复位所有标志位（如 PFLAG_CANCEL_NEXT_UP_EVENT、FLAG_DISALLOW_INTERCEPT 等）  
2.首先调用 activity 的 dispatchTouchEvent 分发（activity 只有这个和 onTouchEvent），从 window 分发到 viewGroup，调用 onInterceptTouchEvent 确定当前 ViewGroup 是否拦截 Down 事件，拦截则事件不会传递给子 View，调用 onTouchEvent 本身消费事件；不拦截则遍历所有子 View 寻找是否有子 View 需要消费该事件（调用子 view 的 dispatchTouchEvent）；若有子 View 需要消费该事件，则设定该事件的处理Target 为该子 View；若无子 View 需要消费该事件，则调用 super.dispatchEvent 判断该 ViewGroup 本身是否需要处理该事件  
3.若事件处理 Target 不为空或该 ViewGroup 消费该事件，则返回 true；否则返回 false。返回值将决定该 ViewGroup 的上级 ViewGroup 是否需要继续询问其他子 View 是否需要消费该事件；对于处于顶层的 DecorView 来说，其返回值会决定包含该 DecorView 的 Activity 是否需要调用 Activity.onTouchEvent 进行处理。    
4.对up、cancel、move 判断在 Down 事件的处理中是否找到可处理该事件的 Target，存在 Target，则调用 onInterceptTouchEvent 以确定当前 ViewGroup 是否拦截该事件，拦截直接调用 super.dispatchEvent 判断该 ViewGroup 本身是否需要消费事件；不拦截，传递该事件至所有已有 Target；不存在 Target，直接调用 super.dispatchEvent 判断该 ViewGroup 本身是否需要消费事件，若事件为 Up 或 Cancel，表明一个完整事件周期结束，则清除已有 Target，复位被置位的标志位（如 PFLAG_CANCEL_NEXT_UP_EVENT、FLAG_DISALLOW_INTERCEPT 等）
## PointerDown
支持多 Pointer(调用 setMotionEventSplittingEnabled 将 FLAG_SPLIT_MOTION_EVENTS 置位)的情况下，有新的 Pointer 按下时产生，会重新遍历 View 层级，寻找可以处理新 Pointer 事件的 Target，具体流程参考 Down 事件的分发逻辑；遍历结束若仍没有找到处理该事件的  Target，则会将新 Pointer 的处理权设置给已有 Target 中最早被添加的 Target。完成 Target 的寻找之后，会将该事件通过 dispatchTransformedTouchEvent 传递至所有已有 Target 进行处理
## PointerUp
相对于 Up 事件，当传递至所有已有 Target 结束之后不能标记以 Down 事件起始的整个事件周期结束，仅能标记其关联 Pointer（以 PointerDown 事件起始）的事件周期结束，不会清除所有状态，仅从已有 Target 中移除掉与该 Pointer 相关的部分。
## MotionEvent
MotionEvent 作为 Touch 事件的载体，采用时间片管理 Touch 事件所有相关行为的数据。  
纵向上，MotionEvent 在一段时间内多次采样合成为一个事件进行处理，每一次采样对应 Touch 数据，事件中就包含多份 Touch 数据，将最近采样数据作为当前数据，其他数据存储为历史数据(批量处理出于效率)。
横向上，一个时间片对应特定时间点（可通过 getEventTime 获取）的采样数据数据，该采样点可能包含多个触摸点，MotionEvent 中采用 Pointer 标记每一个采样点，每一个 Pointer 的激活周期为从 Down 事件至 Up 或 Cancel 事件，激活周期内分配一个在不同 MotionEvent 中保持唯一的 PointerId，但是在不同 MotionEvent 中 Pointer 的排序会不断调整，因此 Pointer 在不同 MotionEvent 中对应的 PointerIndex 也会不断变化，根据 PointerId 可以找到该 Pointer 在某一 MotionEvent 中的 PointerIndex，根据 PointerIndex 则可以获取该 Pointer 在 MotionEvent 中相关 Touch 数据   
rawX：相对于屏幕坐标系的原始X坐标，getRawX  
rawY：相对于屏幕坐标系的原始Y坐标，getRawY  
x：相对于事件处理主体坐标系的X坐标，getX(int index)  
y：相对于事件处理主体坐标系的Y坐标，getY(int index)  
size：按压区域大小，getSize(int index)  
pressure：按压压力，getPressure(int index)  
orientation：按压时屏幕方向，getOrientation(int index)  
touchMajor：按压椭圆区域长边长，getTouchMajor(int index)  
touchMinor：按压椭圆区域短边长，getTouchMinor(int index)  
MotionEvent 将触发当前事件的 Pointe r作为主 Pointer，PointerIndex 为 0，MotionEvent 通过提供 getX() 这类不带 index 参数的接口方便的操作主 Pointer 数据。  
MotionEvent 通过 getAction 接口获取事件 Action，Action 中低 8 位地址存储事件类型（包括 Down、Move、Up、Cancel、PointerDown、PointerUp），高 8 位地址存储 PointerId（当事件类型为 PointerDown、PointerUp 时）。  
FLAG_WINDOW_IS_OBSCURED：是否被透明 View 遮挡  
FLAG_TAINTED：事件是否出现不一致  
FLAG_TARGET_ACCESSIBILITY_FOCUS：事件是否需要先触发辅助功能 View
# Scroll
scrollTo：基于传入参数的绝对滑动  
scrollBy：实际上也是调用了 scrollTo，基于当前位置的相对滑动  
mScrollX = View 左边缘和 View 内容左边缘水平方向距离  
mScrollY = View 上边缘和 View 内容上边缘水平方向距离  
View 边缘为 View 的位置（四个顶点）  
View 内容边缘为 View 内部的内容边缘  
scrollTo/scrollBy 只改变 View 内容位置不改变 View 在布局中位置  
也可以使用动画、改变位置参数等实现滑动，特殊的弹性滑动，可以通过Scroller（自定义）实现，需要配合 View 的 computeScroll（不断让 View 重绘，每次重绘进行小幅度滑动），根据时间流逝百分比计算 scrollX 和 scrollY 改变的百分比，并计算当前值（类似属性动画插值器）
# ListView
只支持竖直方向滑动，recyclerview 包含三种布局（线性、网格、瀑布）notifyDataSetChanged 实现全局刷新（也可以通过写 callback 实现某个 item 的刷新），recyclerview 可以 notifyItemChanged 局部刷新，不支持嵌套滑动（子 view 处理事件后父 view 不再参与，需要手动在事件传递时进行处理）只有两级缓存（ActiveViews：layout 开始时显示的 view，结束后,所有 ActiveViews 的 view 移到 ScrapViews；ScrapViews 中 view 可能被 adapter 重用）
# RecyclerView
替代 ListView 的产品，需要通过 Adapter 初始化列表重写 onCreateViewHolder 以及 onBindViewHolder，可指定 LayoutManager，如 GridLayoutManager，ListLayoutManager 等。  
有独特的三级缓存机制，用户可主动加第四层缓存：  
mScrapView：还未消失但将要消失   
mCachedView：已经消失，默认大小为2用户指定缓存逻辑  
RecyclePool：默认大小为5  
采用差分刷新提高效率：DiffUtil 工具类实现，将新旧数据集传递给 Recyclerview，告知 adapter 哪些改变的数据需要刷新 view，可通过维护一个 list（AsyncListDiffer，异步操作的差分列表，构造传入两个参数，listupdatecallback（重写增删改方法）和 differconfig（重写 areItemsTheSame 和 areContentTheSame，并设置主、子线程 Executor）），向 Recyclerview 提交该列表刷新 view
# Animator
只改变显示位置，内部数据和事件不会改变（需要手动同步）  
帧动画：播放图片，图片较多/较大容易OOM  
属性动画：通过（时间）插值器和（类型）估值器来定义动画，通过 objectAnimator 的工厂方法获取对象（一般 ofFloat），调用 start 开启动画（ValueAnimator 为 objectAnimator 父类，动态计算目标对象属性的值并设置），可通过 AnimatorSet 实现多个动画的播放：AnimatorSet 调用 apply，内部去 play 以及统一执行 start（一次），动画过程可以通过 AnimatorUpdateListener 和 AnimatorListener 来监听，重写 start、end 等方法（属性必须具有 set 形式的 setter 函数，如 text 需要 setText）  
插值器：根据时间流逝的百分比计算出当前属性值改变的百分比，自定义需要实现 TimeInterpolator 接口的 getInterpolation 方法  
估值器：根据当前属性改变的百分比来计算改变后的属性值，自定义要重写 evaluate 方法（对应 ofObejct，其他都有默认 SDK 的插值器）  
此外可以通过关键帧代替自定义，Keyframe 指定属性百分比时对象的属性值。
# attribute
px：pixel，像素，分辨率（density）描述为像素相乘  
dp：density independent pixel，密度无关像素，为适配不同 dpi，定义 px = dp * （dpi / 160），即 160dpi 时的一个像素大小  
dpi：dots per inch，每英寸像素数，像素密度  
sw：屏幕宽度 / （dpi / 160）  
# progress
include:布局重用  
merge:减少布局层级，与 include 配合，include 的布局和当前布局 layout 一致，当前可使用 merge  
viewStub:不可见的没有大小的 View，按需加载，inflate 只调用一次，之后 viewStub 移除，替换为对应 layout（在 inflate 调用或 setVisibility 后执行）  
自定义 view 一定要重写两个参数的构造函数：setContentView -> createView -> clazz.getConstructor(mConstructorSignature)，通过反射获取根节点 view，并获取构造方法，这里的 mConstructorSignature 为两个参数的数组
