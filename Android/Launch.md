# 启动状态
1.冷启动：应用自设备启动后或系统终止应用后首次启动，耗时最多，是衡量启动耗时的标准       
2.温启动：当启动应用时，后台已有该应用的进程，但是 Activity 需要重新创建（退出应用后又重新启动应用。进程可能还在运行/系统因内存不足等原因将应用回收，然后用户又重新启动这个应用）   
3.热启动：Activity 只需要走 onStart，但是如果一些内存为响应内存整理事件（如 onTrimMemory()）而被完全清除，则需要为了响应热启动而重新创建相应的对象，热启动显示的屏幕上行为和冷启动场景相同。系统进程显示空白屏幕，直到应用完成 Activity 呈现  
# 启动阶段
![image](https://github.com/user-attachments/assets/5ec9b45f-770c-4560-be5b-c8be4af1e3a5)  
## Application
在 Application 阶段，可以在 attachBaseContext，installProvider 和 app:onCreate 三个时间段进行相关优化      
bindApplication：APP 进程由 zygote 进程 fork 出来后会执行 ActivityThread 的 main 方法，该方法最终触发执行 bindApplication()，这也是 Application 阶段的起点   
attachBaseContext：在应用中最早能触达到的生命周期，本阶段也是最早的预加载时机   
installProvider：很多三方 sdk 借助该时机来做初始化操作，很可能导致启动耗时的不可控情形，需要按具体 case 优化   
onCreate：这里有很多三方库和业务的初始化操作，是通过异步、按需、预加载等手段做优化的主要时机，它也是 Application 阶段的末尾   
## Activity
创建主 Activity 并且执行相关生命周期方法      
Activity 阶段最关键的生命周期是 onCreate()，这个阶段中包含了大量的 UI 构建、首页相关业务、对象初始化等耗时任务，是我们在优化启动过程中非常重要的一环，我们可以通过异步、预加载、延迟执行等手段做各方面的优化   
## 绘制
来到 View 构建的阶段，该阶段也是比较耗时，可采用异步 Inflate 配合 X2C（编译期将 xml 布局转代码）并提升相应异步线程优先级的方法综合优化   
View 的整体渲染阶段，涵盖 measure、layout、draw 三部分，这里可尝试从层级、布局、渲染上取得优化收益   
最后是首屏数据加载阶段，这部分涵盖非常多数据相关的操作，也需要综合性优化，可尝试预加载、三级缓存或网络优先级调度等手段进行优化   
## 首帧
我们在应用中能触达到的 attachBaseContext 阶段，这是最早的预加载时机   
可以把这个方法的回调时间当作启动开始时间，因为 attachBaseContext() 是应用进程的第一个生命周期，但是准确来说，应用的启动时间包括进程创建，应该在冷启动时用户点击应用 Icon 开始计算   
Activity#onWindowFocusChanged() 这个方法的调用时机是用户与 Activity 交互的最佳时间点，当 Activity 中的 View 测量绘制完成之后会回调 Activity 的 onWindowFocusChanged() 方法，可以选择它来当作时间结束的时间点   
但是大部分数据是通过请求接口回来之后，才能填充页面才能显示出来，当执行到 onWindowFocusChanged() 的时候，请求数据还没完成，页面上依旧是没有数据的，用户仅仅可以交互写死在 XML 布局当中的视图，更多的内容还是不可见，不可交互的   
所以结束时间点通常选择在列表上面第一个 itemView 的 perDrawCallback() 方法的回调时机当作时间结束点，也就是首帧时间。当列表上面第一个 itemView 被显示出来的时候说明网络请求已经完成。页面上的 View 已经填充了数据，并且开始重新渲染了。此时用户是可以交互的，这个才是比较有意义的时间节点，可以通过监听 itemView.viewTreeObserver.addOnPreDrawListener(object :    
 ViewTreeObserver.OnPreDrawListener {  
    override fun onPreDraw(): Boolean {  
        return false  
    }  
})  
## 耗时分析
速度测量 && 启动耗时：adb shell dumpsys AMS 耗时信息（AMS）、attachBaseContext 记录启动时间，在 View 绘制完记录结束时间（埋点）（onWindowFocusChanged 是首帧绘制，还未完成）、Debug SDK、Trace SDK
