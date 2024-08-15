# Input
input 核心是 InputReader 和 InputDispatcher，InputReader 和 InputDispatcher 是跑在 SystemServer进程中的两个 native 循环线程，负责读取和分发 Input 事件，整个处理过程大致流程如下：  
1.InputReader 负责从 EventHub 里面把 Input 事件读取出来，然后交给 InputDispatcher 进行事件分发  
2.InputDispatcher 在拿到 InputReader 获取的事件之后，将事件放入 iq ，寻找并分发到目标窗口    
3.InboundQueue队列（iq）中放着 InputDispatcher 从 InputReader 中拿到的 input 事件  
4.OutboundQueue（oq）队列里面放的是即将要被派发给各个目标窗口 App 的事件  
5.WaitQueue 队列里面记录的是已经派发给 App（wq），但是 App 还在处理没有返回处理成功的事件  
6.PendingInputEventQueue 队列（aq）中记录的是应用需要处理的 Input 事件，这里 input 事件已经传递到了应用进程  
7.deliverInputEvent 标识 App UI Thread 被 Input 事件唤醒  
8.InputResponse 标识 Input 事件区域，一个 Input_Down 事件 + 若干个 Input_Move 事件 + 一个 Input_Up 事件的处理阶段都被算到了这里  
9.App 响应处理 Input 事件，内部会在其界面 View 树中传递处理，由 DecorView 遍历所有子 View  
10.事件处理完成后会调用 finishInputEvent 结束应用对触控事件处理逻辑，通过 JNI 调用到 native 层通知 InputDispatcher 事件处理完成，从 wq 队列中及时移除待处理事件以免 ANR
# Zygote
## init
通过 Socket 进行通信，主要工作是预加载和共享进程资源，提高启动速度。  
## app_process
负责启动 Zygote 进程和应用进程，主入口点是 main 方法，它是整个进程启动流程的起点，区分 zygote 和非空（className 不为空）进程，其中创建服务端s ocket，并预加载系统资源和框架类，  并且也会进入一个死循环进行监听 AMS 的请求
最终调用 runtime.start 启动（runtime 为 AppRuntime，继承 AndroidRunTime，也就是 ART）
## AppRuntime
初始化 JNI、初始化虚拟机、注册 JNI 方法，通过 JNI 调用 Zygote.mian，从这里开始进入到 Java 层
## main
完成资源的预加载，通过 Native 方法 fork 出系统服务（子线程中反射创建 ActivityThread，并调用其 main 方法，注意，这里的线程是应用进程的 binder 线程），为 Launcher 等启动做准备，新进程会继承 Zygote 虚拟机、类加载器等资源，避免重新加载，最后启动 Loop 开启监听
## 为何使用 Socket 不使用 Binder
1.时序：Binder 驱动进程早于 init  进程加载，所需 service 早于 Zygote，但无法保证 Zygote 注册时已经初始化完成    
2.效率：使用 LocalSocket，减少了数据验证等环节   
3.Binder 拷贝：fork 子进程会拷贝 Binder 对象，占用空间，且无法释放（成对存在，分为 C/S 端，释放 Server 引用需要释放 C 端对象，导致失去 binder ），使用 Socket 应用进程（C 端）会主动关闭   
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
## Window 显示
### 添加窗口
1.app 端 activity resume 后，viewRootImpl 会通知 system server 添加窗口   
2.新添加窗口默认为 NO_SURFACE，还没有绘制   
![image](https://github.com/user-attachments/assets/170b474e-b070-4427-b94b-33fea2a23bb0)  
### 创建 Surface
1.App 调用 system server 端 relayoutWindow 开始创建 surface，等待绘制  
2.system server 会创建一个空白 surface，绘制完成前是隐藏的  
3.app 端在 Vsync 信号到来时执行 doFrame，以及下一次的 sheduleVsync（post 同步屏障到消息队列，阻塞同步消息执行；post traversalsRunnable 到 Choreographer 的 mCallbackQueue 中，等待下次 Vsync 信号到来执行回调）     
4.surface 为 DRAW_PENDING  
![image](https://github.com/user-attachments/assets/34b7f075-edff-43c5-9005-2150d110fa26)  
### 绘制 Surface
1.绘制完成，surface 状态为 COMMIT_DRAW_PENDING，等待系统提交  
![image](https://github.com/user-attachments/assets/8a9c01ef-a618-479d-90ab-bb545d28faa7)
### 提交窗口
1.此时为 READY_TO_SHOW，等待同一 windowToken 绘制完成一起 show  
![image](https://github.com/user-attachments/assets/0d36216b-9492-4f6d-bbc1-416895ad4e28)

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
启动/请求 transition 时 添加到 pending -> core 通知时转至 ready -> 开始动画时为 active  
1.Trigger：启动 task  
2.Collecting：记录所有的 change，等待 C 端 接收 change 并将匹配的每一帧数据和 suface 的 change 绘制到同步的 transaction(为 invisible)，一次只有一个 transition 能成为 collectingTransition，实际上可以分为两阶段，阶段一是实际参与 suface 的 change，  收集 transition 封装为 collectingTransition，阶段二就是等待 transition ready，这一阶段不会改变 surface，所以可以同时进行多个  
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
整体流程：  
![image](https://github.com/user-attachments/assets/9eea6395-2c38-4c64-90cb-f5483f95cfe2)  
创建并收集 transition：  
![image](https://github.com/user-attachments/assets/9eed3e1b-2e65-402a-9384-ba794ef31071)   
![image](https://github.com/user-attachments/assets/91b622b0-dea1-42e0-bae0-beb916a9aedd)  
![image](https://github.com/user-attachments/assets/a9fe3834-0cad-431a-8d64-ba8583a9193f)
### Track & SYNC
1.shell transition 一次只能 play 一个动画，因此引入 track，支持多个 track play，core 为 track 分配 ID，shell 可以使用 ID 并行 play，或机型 merge，相同 track ID 的 transition 都会在同一 track 中顺序 play，但 track 之间是独立的  
2.但一些极端情况，transition 可能参与多个 track ，因此又引入 SYNC，在开始前结束所有正在运行的 transition/track
### OverView
1.Shell 侧 TransitionPlayerImpl 实现 ITransitionPlayer 接口，初始化通过 registerTransitionPlayer 注册至 Core 侧 TransitionController 保存。  
2.Transition 开始时 Core 侧通过 ITransitionPlayer.requestStartTransition 通知 Shell 侧（IRemoteTransition 实例也通过此接口参数传递），Shell 侧此方法通知各个 handler.handleRequest。    
3.Shell 侧通过 startTransition 通知 Core 侧开始 Transition。  
4.Core 侧 WM operations(Trigger 加上来自 startTransition 的 WCT) 完成后，BLASTSyncEngine 等待参与的窗口重绘（每次 applySurfaceChangedTransaction 后触发 onSurfacePlacement 检查），全部重绘完毕，BSE 收集所有 SyncTransaction 到 merged transaction，调用 onTransactionReady （完成动画准备工作：更新可见性、创建 transactionInfo 、startTransaction、finishTransaction等）通知 listener，Shell 侧 onTransactionReady 将 transaction 派发至对应 handler 执行动画，remoteTransition 场景动画是远端通过 IRemoteTransition 完成的。  
5.动画完成后 Shell 侧通过 finishTransition 通知 core 同时 core 调用同名接口完成收尾（清理）
### Tips
Shell 也有可能导致内存泄露，基本上都是某个动画未正常结束，执行时间太久导致后续动画堆积或被 merge 到异常动画，相关 surface 无法释放造成的  
## Transition 耗时过长导致后续动画堆积
看 visible layer 中 transition root（每个动画都会创建一个） 相关 layer，shell 端新的 transition 一直在等待执行，而某个动画执行太久，导致后续动画一直未执行，transition root 一直未释放
## Transition 未正常 finish 导致后续动画被 merge
transition 未 finish，会导致后续动画都被 merge 到这个 transition 上来，这些动画一直不能结束，导致内存泄露
## ​BlastBufferQueue
Buffer 申请在 APP 侧，dequeue，queue，acquire，release 操作均由 APP 进行，通过 Transaction 传递给 SF，减少 SF 压力。  
界面不显示会释放 BlastBufferQueue 对象，减少内存。  
将 buffer 和窗口信息更新同步，一次事务中可以传递 buffer 以及 对应的 layer 窗口大小等图层属性给 SF，同时可以将事务跨进程传递给系统服务，系统服务根据需要将窗口的几何修改融入到该事务一并提交，保证在同一帧生效，最后 
relayoutWindow (从 WMS 申请 window layout， 创建 sc 并通过他创建 BBQ)，接着通过 JNI 接口创建 native 对象（创建 bufferQueue 并设置监听），最终创建 suface 对象。  
## Tips
在应用冷启动的 transition 流程中，应用的展示 window 是这样子的：首先桌面添加一层遮罩，充当 surface，如果应用对 Application 配置了 splash screen，在自身绘制完成，内容准备好之后会通知 WM 去 remove 掉 startWindow 来展示这个，往往伴随着应用的 finish draw locked，或者 WM 侧在 finishDraw 之后，也会返回给桌面 surface 用来展示
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
## RenderThread
Activity 初始化流程中，ViewRootImpl 完成了对界面的 measure、layout 和 draw 等绘制流程后，用户依然还是看不到屏幕上显示的应用界面内容，界面还需要经过 RenderThread 线程的渲染处理，渲染完成后，还需要通过 Binder 调用，上帧给 surfaceflinger 进程中进行合成后送显才能最终显示到屏幕上  
### 硬件加速
1.从 DecorView 出发，递归遍历 View 控件树，记录每个 View 节点的 drawOp 绘制操作命令，完成绘制命令树的构建（遍历时记录 onDraw 中的绘制操作，创建命令节点保存到 List 中）  
2.JNI 调用同步Java 层构建的绘制命令树到 Native 层的 RenderThread 渲染线程，并唤醒渲染线程利用 OpenGL 执行渲染任务
### 渲染
UI 线程利用 RenderProxy 向 RenderThread 线程发送一个任务请求，RenderThread 被唤醒，开始渲染，大致流程如下：  
1.遍历 View 树上每一个命令节点，执行 prepareTreeImpl 函数，实现同步绘制命令树的操作  
2.调用 OpenGL 库 API 使用 GPU，按照构建好的绘制命令完成界面的渲染  
3.将前面已经绘制渲染好的图形缓冲区 Binder 上帧给 SurfaceFlinger 合成和显示
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
## Init APP Launch activity
![image](https://github.com/user-attachments/assets/ae1bca38-7b6e-4773-875d-86a1c2e1a080)
### Init Process
![image](https://github.com/user-attachments/assets/797ba746-1e92-4e7e-a361-b05865ae39c4)  
### Init Activity
AMS 收到应用进程的 attachApplication 注册请求后，先通过 binder 调用 IApplicationThread.bindApplication 接口，触发应用进程在主线程执行 handleBindeApplication 初始化操作，然后继续执行启动应用Activity，应用进程这边在收到系统 binder 调用后，做进一步初始化和创建流程（Create 和 Resume）    
### Create
1.创建 Activity 的 Context  
2.通过反射创建 Activity 对象  
3.执行 Activity.attach 动作，其中会创建应用窗口的 PhoneWindow 对象并设置 WindowManager    
4.执行应用 Activity.onCreate 生命周期函数，并在 setContentView 中创建窗口的 DecorView   
### Resume
1.执行应用 Activity.onResume  
2.执行 WindowManager.addView  开启视图绘制逻辑  
3.创建 Activity 的 ViewRootImpl 对象    
4.执行 ViewRootImpl.setView 开启 UI 界面绘制动作
## Launch activity normal
1.context 调用 startActivity，实现类 contextImpl 去执行 startActivity  
2.调用 startActivityForResult ，不过已经被弃用了，可以用 ActivityResult API 来实现  
3.instrumentation 执行  execStartActivity  
4.ATMS 执行 startActivity（此时 AIDL 通信到 S 端了），调用到 resumeTopActivity，如果顶层 activity 为 null，则通过回调 Binder 通信到 ActivityThread 执行 transaction，其中的 handler （内部维护一个对象 H） 执行 sendMessage，由 AT 来 handleMessage，并执行 performLaunchActivity，在这一步获取 activity 信息，需要的话会使用 classLoader 来创建 activity，若第一次启动，则创建 application。  
5.looper.prepareMainLooper 开始消息循环，最终调用 activity 的 onCreate  
## Home Launcher & APP Activity
1.Home 有自启机制，TouchInteractionService 绑定在 systemUI，在 onServiceConnected 回调中，会有重启的逻辑，也就是说 SystemUI 设置了死亡监听，在死亡监听中看到重新绑定的逻辑，进而在桌面被杀的时候重新拉起来，三方目前基本是没有保活能力的   
## System APP & Normal APP
### 存放位置不同
/system/priv-app/ 存放系统应用，/system/app/ 存放普通系统应用，/data/app/ 存放普通应用目录（预装应用）   
### 权限不同
对于 manifest 权限级别而言，普通应用为 normal，普通系统应用为 signature，系统应用为 signatureOrSystem 或者 signature|privileged（系统应用不可卸载）  
### 签名不一致
系统应用通过打包生成正式 ap k时，会使用公司的密钥进行签名，应用在安装的时候会校验密钥，调用系统 installer 进行安装的时候会解析这类信息  
系统应用(桌面)可以访问系统中的一些 hide API（非 SDK API）  
signature 现在主要用来表示跟平台签名一致
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
## 前台Service
高进程优先级，避免由于低内存或其他厂商定制策略被系统查杀，会在通知栏显示通知
### ANR 
默认 anr 超时时间为 20s，以 startForegroundService 方式启动的 Service，则需要在 10s 内调用 startForeground 方法，否则会发生 ANR。一般会将该方法放到 Service 的 onStartCommand 中去执行，因为这种类型的 ANR 超时时间是从 framework 层开始通知 app 端执行 Service 的 onStartCommand 开始计算的。如果将该方法放到 Service 的 onCreate 中去执行，onStartCommand 方法返回值是 START_NOT_STICKY，进程被杀后 Service 重启不会回调 onCreate 方法，但是还会走 ANR 超时逻辑   
### 开启流程
![image](https://github.com/user-attachments/assets/ea147127-a210-467f-9faa-450029092161)
### 取消流程
![image](https://github.com/user-attachments/assets/2d30047c-b6a1-49be-895c-725123d4ccfa)
## 后台Service
没有通知，优先级较低；anr 超时时间为 200s，android 对后台进程管控比较严格，app 退到后台超过 1min 就会进入 idle 状态，并尝试 stop 相关的 Service   
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
onCreate -> onStartCommand -> onDestroy  
onCreate ->   onBind -> onUnbind -> onDestroy
# Broadcast
## foundation
1.分为普通和有序广播，有序指的是上一个拦截广播在释放之后下一个才可以获取，会按照指定的优先级接收本地（当前应用或当前进程）和全局的广播  ，但是是串行分发，效率低，可以通过 abort 截断，入队会根据优先级对 receiver 排序，普通就是无序的，也是默认的，是并行分发，不可拦截、中止或修改，数据传递通过 intent.putExtra    
2.有序广播和无序广播主要区别在于可以指定广播的最后一个接收者 resultReceiver，通过最后一个 receiver 可以知道广播派发完了，做一些收尾工作   
3.按照处理类型可以分为前台广播（发生时添加 tag 为 intent.FLAG_RECEIVER_FOREGROUND，10s 超时）和后台广播（默认就是后台，60s 超时），各自有一个广播队列互不干扰  
### Registe
分为 manifest 静态注册（PMS 初始化时自动注册）和代码动态注册（运行时）  
1.四大组件都差不多，还是 CW 到 mBase 挨个调用  
2.通过 LoadApk.Dispatcher 实现一个 IntentReceiver（Binder 对象），并向 AMS 发起请求，由 AMS 来真正注册，并存储 intentFilter 和 receiver
广播需要及时 unRegiste
## Send
1.还是从一个 CW 开始 sendBroadcast，接着 mBase 执行 sendBroadcast  
2.向 AMS 发起请求，由 AMS 执行 broadcastIntent，找到广播接收者（满足 intentFilter），并将此接收者放入到广播列表等待被唤起（对此接收者调用 send）
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
IPC 中通过 contentResolver 获取另一进程的 contentProvider 提供的数据（C 端调用 call 方法，S 端需要重写此方法）
## foundation
1.采用索引表的形式组织数据，会指定唯一提供者以及访问权限  
2.底层是通过 binder 实现的，方法都运行在 binder 线程（如 call 方法）  
3.数据更新类似广播机制，通过一个 contentObserver（也需要注册，通过 URI 描述，告知接收数据类型） 接收数据更新通知，由 provider 来发送通知，数据源由 SQLite 实现，数据也会被封装为 cursor  
4.多线程操作，若针对内存则需要加锁实现同步，若底层为数据库数据则不需要，SQLite 会自己处理，但若是多个 SQLite 则还是需要自己处理同步    
5.多进程操作会由请求队列按顺序执行  
## C 端查询
1.若有 ANR，卡在 acquireProvider 则是与 system server 通信；卡在访问 IContentProvider 接口则是跟 S 端通信  
2.query 方式失败，会建立 stable 连接  
3.call、insert、delete、update 都是建立 stable 连接，对端挂掉，也会级联查杀 client  
4.query 建立 unstable 连接，client 不绑定对端的查杀状态  
5.ActivityThread.acquireProvider 执行时，若本地有对于 IContentProvider 实例，则直接获取；不存在，则需要向 AMS 查询，若对端 provider 已发布，直接执行本地 install，未发布则会先 wait，等待 20 s，超时会 ANR，此外，正常流程为一端先 install，完成后 binder 到 AMS，来 publish provider，这里在 S 端发布完成后，AMS 会 notify C 端，解除 wait 状态  
## AMS 查询
运行在 system server 进程  
1.S 进程存在，且已发布 provider，则直接建立连接  
2.S 进程存在，未发布，则通知 S 端发布  
3.S 进程不存在，则先创建进程
## 返回 null
1.对端进程死亡或不存在  
2.provider 不在运行状态  
3.pms 解析 provider 所在 application 失败 
## 引用计数
1.对于 app 而言，C 端获取到 S 端 provider 后建立连接，引用计数增加，在 call 结束后，release provider，引用计数减少  
2.对 system server 而言，ActivityThread.complete remove provider 后，到 AMS 去 removeContentProvider，最终 adjustCounts
## 死亡移除
目标 CP 进程结束后，AMS 会根据连接的引用计数，决定是否杀死 C 端进程  
1.stableCount 不为 0，则会杀死进程  
2.stableCount 为 0，unStableCount 不为 0，则通知 C 端，并移除对应连接
# Application
应用初始化启动时由系统创建，存储一些系统信息
## Init
1.在 ActivityThread.mian 中，开启主线程 loop 之前，会通过 binder 调用 AMS.attachApplication，将自己注册进去  
2.之后切到 AMS 进程，执行 bindApplication，随后向应用主线程 post 消息通知 bind 完成  
3.回到应用进程，根据框架传入的 ApplicationInfo 信息创建应用 APK 对应的 LoadedApk 对象，创建 Application 的 Context 对象  
4.创建类加载器 ClassLoader 对象并触发 Art 虚拟机执行加载应用 APK 的 Dex 文件  
5.通过 LoadedApk 加载应用 APK 的 Resource 资源  
6.调用 LoadedApk.makeApplication，创建应用的 Application 对象  
7.执行应用 Application.onCreate
## attachBaseContext
在 onCreate 之前被调用，作用是向 Context 中添加或修改一些信息，一般执行一些初始化如创建全局对象（数据库等）、设置默认语言等
## LifeCycle
1.onConfiguration：配置更改时回调  
2.onCreate：创建时回调  
3.onTerminal：结束时回调  
4.onLowMemory：内存不足时回调  
5.onTrimMemory：内存清理时回调
# Configuration
## 整体时序
![image](https://github.com/user-attachments/assets/0c96fdf4-21e3-4c10-a479-c7bda90f602c)  
![image](https://github.com/user-attachments/assets/681327bd-06ef-4b21-88c9-487172cea83d)
## 进程
主要是更新 APP 进程中 Resource 的 Config 并触发 Application、Service、ContentProvider 的 onConfigurationChanged  
## System server
![image](https://github.com/user-attachments/assets/46c5c767-d673-45a5-9eaa-68e17a30b9b6)  
## App
![image](https://github.com/user-attachments/assets/52fcea5b-11e9-4c1e-8d9b-90050398285c)
## Activity
### relaunch
manifest 文件没有配置了 configChanges，则回 relaunch  
![image](https://github.com/user-attachments/assets/79fa5c99-02eb-4457-b37a-fefa57aaa3c3)    
一般 top activity 会 resume，底层的走 onPause
### onConfigurationChanged
配置 configChanges，则会走 onConfigurationChanged 回调  
![image](https://github.com/user-attachments/assets/6310dcba-3236-446e-bf77-b973a2d59540)
### top activity
![image](https://github.com/user-attachments/assets/4d2e94ca-e0aa-4c1b-9b52-95882727c024)
### 其他可见 activity
![image](https://github.com/user-attachments/assets/cdb17ff7-9fb6-4d79-9b3c-af55d286d6b0)
### 不可见 activity
![image](https://github.com/user-attachments/assets/1f86c82a-53c9-4109-a539-2e83a1595c34)  
会在下次 resume 的时候去执行 relaunch，因此跟上面两种不在同一线程，不在同一时间，若需要看 relaunch 原因，需要往前查找 configuration changed 相关 log





