# Zygote
## init
通过 Socket 进行通信，主要工作是预加载和共享进程资源，提高启动速度。
## app_process
负责启动 Zygote 进程和应用进程，主入口点是 main 方法，它是整个进程启动流程的起点，区分 zygote 和非空（className 不为空）进程，最终调用 runtime.start 启动（runtime 为 AppRuntime，继承 AndroidRunTime，也就是 ART）
## AppRuntime
初始化 JNI、初始化虚拟机、注册 JNI 方法，通过 JNI 调用 Zygote.mian，从这里开始进入到 Java 层
## main
完成资源的预加载，通过 Native 方法 fork 出系统服务（子线程中反射调用其 main 方法），为 Launcher 等启动做准备，新进程会继承 Zygote 虚拟机、类加载器等资源，避免重新加载，最后启动 Loop 开启监听
## 为何使用 Socket 不使用 Binder
1.时序：Binder 驱动进程早于 init  进程加载，所需 service 早于 Zygote，但无法保证 Zygote 注册时已经初始化完成 。
2.效率：使用 LocalSocket，减少了数据验证等环节。
3.Binder 拷贝：fork 子进程会拷贝 Binder 对象，占用空间，且无法释放（成对存在，分为 C/S 端，释放 Server 引用需要释放 C 端对象，导致失去 binder ），使用 Socket 应用进程（C 端）会主动关闭。
# Window
## foundation
Window 是个抽象类，实现类为 PhoneWindow。Window 是分层级的：系统 Window（2000-2999，需要权限才能创建，如 Toast、状态栏）、子 Window（1000-1999，需要父 Window，不能单独存在，如 Dialog）和应用 Window（1-99，对应一个 Activity），层级大的覆盖在层级小的上面。
Window 是一种概念，具体以 View 的形式存在（一般为 DecorView），通过 ViewRootImpl 与 View 联系，一个 Window 对应一个 ViewTree（控制 View 显示层级），Window 会拦截并分发事件给 View。
## WindowManager
访问 Window 的入口，实现类为 WindowManagerImpl，WindowManagerImpl 又将具体实现委托给 WindowManagerGlobal（进程单例），通过 WM 访问 WMS 进行 add、remove、update 等操作，此过程为 IPC 过程。
## WindowConfiguration
Configuration 是用来保存系统的各项配置的类，如网络运营商配置、屏幕参数配置、显示模式配置等，其中 windowConfiguration 就是保存的窗口相关的配置。  
 WindowConfiguration 里有两个主要 Rect 区域：  
- mBounds 保存该配置下窗口相对于整个屏幕的可用区域  
- mAppBounds 可以用来保存 mBounds 中去除了一些系统装饰后应用可以使用的区域  
- mMaxBounds （Android 12L）应用可以获取到的最大区域，一般是全屏幕  
mWindowingMode 保存的是该配置下窗口的显示模式，例如全屏、分屏、浮动等模式。    
## WindowContainer
ConfigurationContainer 是 WMS 里设计的配置容器类，一个配置容器可以容纳其他的配置容器（就像 View 那样），它是抽象类，只定义的容纳关系，并没有实现子容器的保存形式，它也是一个泛型类，只能保存对应泛型的子容器。    
WindowContainer 是 WMS 里设计的窗口容器类，它可以容纳其他的窗口容器，同时它继承自配置容器类 ConfigurationContainer，因此每一个窗口容器可以向它管理的窗口附加一份新的配置(主要为 WindowConfiguration相关的配置)，窗口容器将它的子容器保存在一个列表里，用于管理窗口配置，包括窗口的生命周期（创建、销毁）、布局和渲染（调整窗口大小，位置和可见性）等，并实现了添加、删除子容器，以及遍历不同类型子容器的便捷方法，内部持有一个 surface 对象。    
Task、activityRecord、windowState 等都间接/直接继承 windowContainer，在 sf 中都有对应 layer，关系为 task -> activityRecord -> windowState
### WindowToken
窗口 Token，用来做 Binder 通信，同时也是一种标识，应用组件在需要新的窗口时，必须提供 WindowToken 以表明自己的身份，并且窗口的类型必须与所有的 WindowToken 的类型一致
### ActivityRecord
继承 windowToken，对应着应用进程中的 Activity，包含了 activity 所有信息
### WindowState
继承 activityRecord，对应着一个窗口，用于描述窗口的状态信息以及和WindowManagerService进行通信，表示一个窗口的所有属性
### RootWindowContainer
RootWindowContainer 是 WMS 管理所有窗口容器的根容器，系统中只存在一个，由 WMS 服务创建，但是会等待 ATMS 服务启动完成后再初始化，管理的是 DisplayContent（也是一个 windowContainer，对应着显示屏幕，对应唯一 ID，添加窗口时通过指定ID决定显示在哪个屏幕），每一个 DisplayContent 对应的是一个 Display 屏幕（包括物理屏幕和虚拟屏幕），同时实现了 DisplayListener 的接口，可以监听屏幕的添加、删除操作。
### WindowContainerTransaction
表示 WindowContainer上的操作集合，一般配合 WindowContainerTransactionCallback 一起，接收包含同步操作结果的事务。相比 Transaction 是在 SurfaceControl 上的操作集合，WindowContainerTransaction 是在 WindowContainer 上的操作集合，如 setBound 等修改 windowContainer 属性以及层级（如将转换父容器或在当前父容器中的位置）的操作，传入 windowContainer 对应的 token 并发送到服务端，最后由 callback 返回结果给客户端
## ShellTransition
### BlastSyncEngine
ST 的核心，是 WMS 维护的一个成员对象，每次窗口事务切换都围绕他进行  
从 startSyncSet 开始，在这一步创建 SyncGroup 并返回 syncId，维护的 mActiveSync 列表将创建的 set 添加进去，接着将 window 添加到 syncSet（也就是 synGroup） 中，setReady 置好状态，通过 onSurfacePlacement 回调检查 window 是否已经 draw 完了（group.tryFinish），最后调用 finishNow 来提交这次的 transaction，后续由维护的监听 mListener 执行 onTransactionReady 回调到 Transition 中。
### AnimCustom
#### TransitionHandler
ST 通过此接口，定义 start/merge 方法（一般只关注 startAnimation 和 mergeAnimation）等来自定义动画，其他应用进程想要定义窗口动画，需要自己注册一个 remoteTransitionHandler（正常是在 Shell 侧的）       
创建 windowContainerTransaction 并设置参数，调用  startTransaction，这一步创建 activeTransition，并添加到 pendingTransitions 列表中，当 onTransitionReady 回调到来时，添加到 readyTransition 中（移除原来的 transition），接着执行 handler 的 startAnim 发起 ST 动画，最后指定需要定制动画的 transitionHandler。  
非 shell 侧发起的 transition，可通过 addHandler 加入到 mHandlers 中待遍历（active.mHandler 为 null 时 dispatchTransition 遍历）时执行 startAnimation。  
TransitionInfo：transition 信息，是否处理 transition、执行动画是否依赖此 info，主要包括一些  window\action\flags 的 change 列表  
requestStartTransition 时 mHandlers 中 handleRequest 返回非空 WCT 的 handler，onTransitionReady 时执行其 startAnim，重写 handleRequest 可以实现特定的 Transiton 处理  
merege：  
1.若没有 handle 住（activeTransiton 为 null）则在回调中直接执行 playTransition，通过 active.mHandler.startAnimation 回调到 remoteTransitionHandler 中，创建 IRemoteTransitionFinshiedCallback，通过 remote 执行 startAnimation 最终回调到应用进程实现的 remoteAnimationAdapterCompat 中，交由 IRemoteTransition.Stub 来处理 startAnimation。
2. handle 住（activeTransition 不为 null）则直接调用 playing.mHandler.mergeAnimation，同样是回调到 remoteTransitionHandler 中，创建动画结束回调，通过 remote.mergeAnimation 最终回调到应用进程执行 mergeAnimation    
3.应用侧动画结束会执行动画结束回调，call Merge/Main（是 merge 则先调用 mergeCall 再调用 mainCall），回调到 Shell 侧 onTransitionFinished，进一步到 Transitions 的 onMerged/onFinished，遍历 mMerged 列表，merge 所有的 mFinish、mStartTransition 并统一 apply，最终 finishTransition 回调到内核，执行 finishTransition。  
onTransitionConsumed：当 transition 停止或 merge 完成时执行一些清理工作
#### TransitionObserver
负责监听 ST 各个阶段（onTransitionReady 等）
### Transition
是动画 change 的集合，包含 window 的 open/close、bounds/mode 的改变、display 的 rotate/size/density 改变，BSE 的回调会调到此处的 onTransitionReady，这里会从内核回调到 Shell 侧 TransitionPlayerImpl 的 onTransitionReady。  
会在各个回调（如 onTransitionReady）中传递 transitionInfo，info 包含了 trigger type 以及一个 changes list
lifecycle：  
1.Trigger：启动 task  
2.Collecting：记录所有的 change，等待 C 端 接收 chane 并将匹配的每一帧数据和suface 的 change 绘制到同步的 transaction(为 invisible)  
3.Playing:做动画到新的 state  
4.Finished：内核做一些善后处理如  clean 操作以及将使用完毕的 leash 置为 invalide    
Collect：  
1.判断 mOpeningApps/mClosingApps  
2.RootWindowContainer#checkAppTransitionReady，接着调用 AppTransitionController#handleAppTransitionReady，transitionGoodToGo/transitionGoodToGoForTaskFragments    
3.WM change 触发 Transition(trigger) ，WM 判断此次操作为 start\pause\change 哪个 并 collect 参与的 containers(collecting)，随后等待所有的 container redraw(syncing)    
4.Playing：SurfaceAnimationRunner#startAnimationshell 接收 onTransactionReady 回调过来的 message 随后做动画，动画结束后执行 finishTransition 告知内核  
Finising：  
1.一些 FinishCallback  
2.内核在 finishTransition 中做一些 clean 工作
### OverView
1.Shell 侧 TransitionPlayerImpl 实现 ITransitionPlayer 接口，初始化通过 registerTransitionPlayer 注册至 Core 侧 TransitionController 保存。  
2.Transition 开始时 Core 侧通过 ITransitionPlayer.requestStartTransition 通知 Shell 侧（IRemoteTransition 实例也通过此接口参数传递），Shell 侧此方法通知各个 handler.handleRequest。    
3.Shell 侧通过 startTransition 通知 Core 侧开始 Transition。  
4.Core 侧 WM operations(Trigger 加上来自 startTransition 的 WCT) 完成后，BLASTSyncEngine 等待参与的窗口重绘（每次 applySurfaceChangedTransaction 后触发 onSurfacePlacement 检查），全部重绘完毕，BSE 收集所有 SyncTransaction 到 merged transaction，调用 onTransactionReady （完成动画准备工作：更新可见性、创建 transactionInfo 、startTransaction、finishTransaction等）通知 listener，Shell 侧 onTransactionReady 将 transaction 派发至对应 handler 执行动画，remoteTransition 场景动画是远端通过 IRemoteTransition 完成的。  
5.动画完成后 Shell 侧通过 finishTransition 通知 core 同时 core 调用同名接口完成收尾（清理）
## ​BlastBufferQueue
Buffer 申请在 APP 侧，dequeue，queue，acquire，release 操作均由 APP 进行，通过 Transaction 传递给 SF，减少 SF 压力。  
界面不显示会释放 BlastBufferQueue 对象，减少内存。  
将 buffer 和窗口信息更新同步，一次事务中可以传递 buffer 以及 对应的 layer 窗口大小等图层属性给 SF，同时可以将事务跨进程传递给系统服务，系统服务根据需要将窗口的几何修改融入到该事务一并提交，保证在同一帧生效，最后 
relayoutWindow (从 WMS 申请 window layout， 创建 sc 并通过他创建 BBQ)，接着通过 JNI 接口创建 native 对象（创建 bufferQueue 并设置监听），最终创建 suface 对象。  
# SurfaceFlinger
## foundation
系统中只有一个实例，负责给 C 端分配窗口。
他可以理解为是一个平面，即每个窗口是一个平面，每个平面对应一段内存，即所谓的屏幕缓冲区，缓冲区大小取决于窗口大小，即宽高（一般为宽*高）。  
接受多个源（有显示界面）的数据缓冲（BufferQueue，DeQueue 取出 -> Queue 放回），进行合成（acquireBuffer 获取，releaseBuffer 放回）：  
1.不会在应用每次提交缓冲区时都执行操作，在显示设备准备好接收新的缓冲区时（VSYNC 信号到达）才会唤醒，遍历层列表寻找新缓冲区。如果找到会获取该缓冲区，否则继续使用以前的缓冲区。  
2.SurfaceFlinger 必须始终显示内容，会保留一个缓冲区，如果在某个层上没有提交缓冲区，则该层会被忽略。  
3.在收集可见层的所有缓冲区后会询问 Hardware Composer 如何进行合成
## Simple Process
1.APK 需要创建窗口时，会通过 WM.addView 创建一个 ViewRoot（ViewRootImpl） 对象，其中会通过 SF 无参构造函数创建 SF 对象，此时只是一个空壳（窗口需初始化后才对应一个屏幕显示的窗口，本质是给 SF 分配一段屏幕缓冲区的内存），需要向 WMS 请求（将空对象传给 WMS），返回一个完整对象。  
2.WMS 收到请求后，通过 SF 的 JNI 调用到 SF_client 驱动，请求 SF 进程创建指定窗口，SF 创建一段屏幕缓冲区并关联该窗口，将地址返回给 WMS，WMS 通过此地址初始化 SF 对象 返回给 APK。  
3.APK 有了 SF 后就可以进行一系列的绘制操作，如矩阵、文本、图片等
## Tips
1.SF 本质上只表示一个平面，而不是一段数据，Android 使用 Skia 绘图驱动库（C/C++）进行各种平面绘制。  
2.SF 包含 lockCanvas 函数，APK 通过此函数返回的 Canvas 对象的各种绘制函数完成平面绘制。
# PMS
## init
Ztgote fork 出PMS 进程，在 PMS 的 main 函数中完成初始化加载。
首先解析 packages.xml 文件，其中包含了所有安装的 APK 信息，再扫描所有 APK 的目录（一般为 data/app  或 system/app），将解析后的 APK 信息更新到 xml 文件当中，最后向 mSettings 文件（存储 APK 信息以及一些 PMS 的相关设置）中写入。
## install
1.创建一个 session，将 APK 写入到 session 当中，并将 session 提交给 PMS  
2.将 APK 文件拷贝到 data/app 目录下  
3.解析 APK （如包含的四大组件），并扫描 APK 信息（如 package settings）  
3.检查签名是否合法（如测试签名和正式签名等）  
4.更新系统状态以及 PMS 中的内存数据  
5.返回结果通知 UI 变化（如一些 onPackageChange 等回调）
# Activity
## Launch Launcher
1.Zygote fork 出进程（其实 AMS\PMS\WMS 都是这样子），在 AMS 中调用 systemReady  
2.ActivityTaskManagerInternal 调用 startHomeActivity 会拉起 Launcher  
3.rootWindowContainer 调用startprocessasync 来 fork launcher 进程（这里也是使用 Socket 通信）  
4.接着执行 activityThread（真正实现在内部类 ApplicationThread 中，负责 Activity 的启动等工作） 的 main 函数，初始化 looper、handler、创建 instrumentation（用来监控 Activity，调用生命周期函数等） 等对象。  
5.获取 AMS 对象，并执行 attachApplication 函数，创建并 bind application  
6.instrumentation 中来 new 一个 activity，activity 执行 attach，来 bind contextImpl（也就是维护的 mBase 对象）  
7.初始化 window 之后执行 performLaunch、resumeLaunch，最终 makeVisible  
## Launch activity normal
1.context 调用 startActivity，实现类 contextImpl 去执行 startActivity  
2.调用 startActivityForResult ，不过已经被弃用了，可以用 ActivityResult API 来实现  
3.instrumentation 执行  execStartActivity  
4.ATMS 执行 startActivity（此时 AIDL 通信到 S 端了），调用到 resumeTopActivity，如果顶层 activity 为 null，则通过回调 Binder 通信到 ActivityThread 执行 transaction，其中的 handler （内部维护一个对象 H） 执行 sendMessage，由 AT 来 handleMessage，并执行 performLaunchActivity，在这一步获取 activity 信息，需要的话会使用 classLoader 来创建 activity，若第一次启动，则创建 application。  
5.looper.prepareMainLooper 开始消息循环，最终调用 activity 的 onCreate  
## onNewIntent
1.activity launchMode 设置为 singleTop，并处于栈顶时启动，或 startActivity 时设置了 FLAG_ACTIVITY_SINGLE_TOP，不再创建实例，直接使用原来的，并会调用 onNewIntent，后续生命周期中其他方法也会基于此方法中传递的参数调用。（onPause -> onNewIntent -> onResume）。  
2.launchMode 为 singleTask 时，activity 不在栈顶，再次启动会调用 onNewIntent，onStart -> onNewIntent -> onResume，若已经在栈顶，则与 singleTop 类似。  
3.若没有设置 setIntent，那么 onNewIntent 返回的会是旧的 intent。
## Lifecycle
onCreate  
onStart：可见（如透明的）  
onResume：处于前台，可交互  
onPause：处于后台  
onStop：不可见  
onDestroy：异常情况下（Activity被销毁重建，如改变屏幕方向/大小），会走onSaveInstance（在 onStop 之前），获取参数中储存的销毁前数据来恢复 Acitvity 现场(onRestoreInstanceState,在 onStart 之后)
## IntentFilter
1.要求 intent 必须含有 action，满足 manifest 中任一 action 即可  
2.可以没有 category，只要有，则要满足所有 category 才能匹配  
3.data 用于定义一些数据格式、长度等，匹配规则和 action 类似，但不强制要求有
## 通信
1.通过 intent.putExtra 携带一些基本数据类型或者 bundle 对象（封装数据或实现了接口的对象），在跳转时传递，但是有大小限制，因为 Binder 映射内存限制大小为 1M  
2.通过持久化来实现，如写入 SQL、File、SP 等  
3.实现内存共享，如静态成员，或匿名共享内存：将指定的物理内存分别映射到各个进程自己的虚拟地址空间中，从而便捷的实现进程间内存共享，可以通过 SharedMemory 等共享工具实现   
4.也可以通过缓存中介一下，如 LruCache，发送方先把数据放入 LruCache，接收方从缓存中读取
## Fragment
### Lifecycle
onAttach  
onCreate  
onCreateView  
onViewCreated  
onStart  
onResume  
onPause  
onStop  
onDestroyView  
onDestroy  
onDettach
### foundation
1.通过 transaction 操作，通过 add（将所有 fragment 叠加到 frameLayout 中，可能有重叠问题） 或 replace（先将原来的 remove 掉）  
2.一个容器只能添加一种 fragment，通过 add 添加的只能添加一次，hide 和 show 切换，生命周期不变，replace 添加的可以多次  ，执行上一个 fragment 的 onDestroyView、onDestroy、onDetach，和下一个的 onAttach、onCreate、onCreateView、onViewCreated、onStart、onResume  
3.Activity 添加 fragment 默认不会添加到任务栈，transaction 内部维护双向链表，记录 add 和 replace 的fragment，自动实现类似出栈入栈操作
### 通信
1.当然可以使用 eventus，有点大材小用了，基本上不会考虑。   
2.通过 activity，若 activity 持有 fragment 实例，那么可以直接获取到其中所有方法，没持有也可以通过 findfragmentByTag/findFragmentById 等获取，而 fragment 可以直接通过 getActivity 拿到 activity 的引用。此外，在 activity 或 fragment 中创建子 fragment 时添加 callback 可以实现 fragment 间的通信，不过这种属于 android 包中老 fragment 的手段，新的以及从 AOSP 移到了 androidx 中，这在方式不再考虑。  
3.基于 ResultAPI，fragmentManager 成为了 fragmentResult 的持有者，可以通过使用 Fragment 扩展函数，进一步调用 fragmentManager.setFragmentResultListener 方法，为 fragment 设置监听。fragment 在 manager 中注册监听后，子 fragment 返回 result 时就会通知该 fragment。（setResult 多次只返回一次结果）   
4.可以通过 viewModel 实现通信，例如 fragment 都去订阅 activity 的 viewModel 中的 liveData，感知变化。
## LaunchMode
standard：默认，每次启动 Activity 都会创建实例入栈，不可以使用 applicationContext 启动此模式的 activity（不存在任务栈），需要指定 FLAG_ACTIVITY_NEW_TASK 创建任务栈  
singleTop：栈顶复用，启动的 activity 位于栈顶则调用 onNewIntent    
singleTask：栈内复用，启动的 activity 位于栈内（不在栈顶则其余 activity 全部出栈）则调用 onNewIntent    
singleInstance：单一实例，每个 activity 都位于独立的栈中  
## getIntent
1.返回 mIntent 成员变量，在 onCreate 之前的 attach 通过 AMS 返回赋值。  
2.可以在 onNewIntent 中通过 setContent 手动更新，这样 get 到的是最新的值。
## Tips
1.SplashActivity 是打开应用时，有一个欢迎界面，一般做一些初始化工作。  
2.Application 在单进程应用中为单例，onCreate 只执行一次，在 LoadApk.makeApplication 中判断为空才会创建（由 AT.instrumentation.newApplication 实现）。  
3.manifast 可以设置 taskAffinity，标识 Activity 所需要的任务栈名称，默认应用中所有 Activity 所需任务栈名称都为应用包名，一般跟 singleTask 或 allowTaskReparenting 属性结合使用，属性值不能与当前应用包名相同否则等于没设置。    
设置 singleTask，同时指定 Activity 所需要的栈名称，使相应的 Activity 存在于不同的栈中。   
当 allowTaskReparenting 为 true 时，表示 Activity 能从启动的 Task 移动到有 affinity 的 Task。  
# Service
## Launch
一般通过 startService 来启动一个服务，多次调用只会走一次 onCreate（只创建一次），但 onStartCommmand（onStart 弃用了） 每次都会调用。  
1.首先从一个 contextWrapper（context 代理，将调用委托给 contextImpl，也就是 mBase）的 startService 开始  
2.调用 startServiceCommand，在此函数中会通信到 S 端，AMS.getService 获取一个 ActiveService 赋值给 mService，由此对象接着继续后续的 startService  
3.通过 Binder 通信，由 Binder 对象 IapplicationThread 调用方法来创建 Service 并调用 onCreate  
4.和 Activity 类似，发消息通知 H（handler），H.handleMessage，使用 classLoader 来创建 Service，若为运行在其他进程则会创建 Application  
5.创建 contextImpl，调用 attach 建立链接，最终调用 onCreate 并将 service 对象放入 activityThread 维护的列表（其实是一个 arrayMap）当中，执行 onStartCommand 完成启动。
## Bind
一般通过 bindService 来绑定一个服务  
1.bindService 执行会走 onBind（多次 bind 只会调用一次），返回一个 Binder 对象，用于通信   
2.类似 start，从一个 contextWrapper.bindService 开始，接着是 mBase.bindService  
3.通过一个 serviceDispatcher 连接 serviceConnection 和 IServiceConnection（传递给 AMS，不是一对一的，context 和 serviceConnection 不同都会组成不同的 IServiceConnection）    
4.通过维护的 arrayMap 来判断是否存在相同的 connection，若不存在则会创建一个新的 dispatcher（这个connection 是一对一的）并放入 map 中
5.接着就会 IPC 到 AMS 端执行 bindService，交由 ApplicationThread 继续处理  
6.还是使用 H.handleBindService 从返回的 token 当中获取 service 对象，调用 onBind，并返回 binder 对象给 C 端，并回调 serviceConnection.onServiceConnected，在这个回调中通知 AMS 来 publishSerice 通知 C 端连接完成。
## Tips
所有绑定的 C 端都解除了绑定，这个 service 才会被销毁
## Lifecycle
onStartCommand  
onBind（onReBind）  
onUnbind   
onDestroy
# Broadcast
## foundation
分为普通和有序广播，有序指的是上一个拦截广播在释放之后下一个才可以获取，会按照指定的优先级接收本地（当前应用或当前进程）和全局的广播
## Send
1.还是从一个 CW 开始 sendBroadcast，接着 mBase 执行 sendBroadcast  
2.向 AMS 发起请求，由 AMS 执行 broadcastIntent，找到广播接收者（满足 intentFilter），并将此接收者放入到广播列表等待被唤起（对此接收者调用 send）
### Registe
分为 manifest 静态注册（PMS 初始化时自动注册）和代码动态注册（运行时）  
1.四大组件都差不多，还是 CW 到 mBase 挨个调用  
2.通过 LoadApk.Dispatcher 实现一个 IntentReceiver（Binder 对象），并向 AMS 发起请求，由 AMS 来真正注册，并存储 intentFilter 和 receiver
广播需要及时 unRegiste
## Receive
与 send 对应，上述提到的广播列表中轮到该接收者时，会经过 appilcationThread 处理，接着通过 LoadApk.receiverDispatcher 执行 performReceive，通过 H 来 post 消息，最终执行 onReceive
# ContentProvider
## Registe
需要在 manifest 中注册，一般都是单例，访问 CP 需要通过 AMS 实现（获取到的 CP 是一个 Binder 对象）   
所在进程启动时将 CP 启动（在 application.onCreate 前执行 CP.onCreate），并发布到 AMS  
1.既然是随应用进程启动，那么一定会走到应用的入口——AT.main，调用到 attach 后会执行 S 端 AMS.attachApplication，这一步才真正启动了 CP  
2.通过 IPC 获取到 C 端的 applicationThread，并调用 bindApplication，通过 H 来 post 到主线程执行  
3.AT 执行 handle 操作，绑定 mBase 以及 instrumentation，接着创建 application，执行 attachBaseContext  
4.最终执行 CP 的 onCreate  
## ContentResolver
IPC 中通过 contentResolver 获取另一进程的 contentProvider 提供的数据（C 端调用 call 方法，S 端需要重新此方法）
## foundation
1.采用索引表的形式组织数据，会指定唯一提供者以及访问权限  
2.底层是通过 binder 实现的，方法都运行在 binder 线程（如 call 方法）  
3.数据更新类似广播机制，通用一个 contentObserver（也需要注册，通过 URI 描述，告知接收数据类型） 接收数据更新通知，由 provider 来发送通知，数据源由 SQLite 实现，数据也会被封装为 cursor  
4.多线程操作，若针对内存则需要加锁实现同步，若底层为数据库数据则不需要，SQLite 会自己处理，但若是多个 SQLite 则还是需要自己处理同步    
5.多进程操作会由请求队列按顺序执行  
# Application
