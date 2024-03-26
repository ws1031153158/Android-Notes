# Zygote
## init
系统启动时创建，用于 fork 出其他进程。
通过 Socket 进行通信，新进程会继承 Zygote 虚拟机、类加载器等资源，避免重新加载。Zygote 主要工作是预加载和共享进程资源，提高启动速度。
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
# SurfaceFlinger
## foundation
系统中只有一个实例，负责给 C 端分配窗口。
他可以理解为是一个平面，即每个窗口是一个平面，每个平面对应一段内存，即所谓的屏幕缓冲区，缓冲区大小取决于窗口大小，即宽高（一般为宽*高）。
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
# Service
## Launch
## Bind
## Lifecycle
# Broadcast
## foundation
## Send
### Registe
## Receive
# ContentProvider
## Registe
## ContentResolver
## foundation
