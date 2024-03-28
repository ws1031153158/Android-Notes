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
## MotionEvent
# Scroll
# ListView
# RecyclerView
# Animator
# attribute
# progress
