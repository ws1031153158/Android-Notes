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
### MeasureSpec
## Layout
## Draw
# Touch
## MotionEvent
# Scroll
# ListView
# RecyclerView
# Animator
# attribute
# progress
