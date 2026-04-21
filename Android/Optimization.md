# 启动优化	
## 冷启动
### 页面合并
![image](https://github.com/user-attachments/assets/09208ec8-740e-4656-9fc0-a2d092fe5943)    
利用读取开屏信息和等待广告的时间，做一些与 Activity 强关联的并发任务，比如异步 View 预加载，数据加载等  
![image](https://github.com/user-attachments/assets/2e245352-19fd-4d3f-a980-2107d941acb7)  
MainAcitvity 的 launch mode 需要设置为 singleTop，否则会出现 App 从后台进前台，非 MainActivity 走生命周期的现象  
跳转到 MainAcitvity 之后，其他二级页面需要全部都关闭掉，站内跳转到 MainActivity 则附带 CLEAR_TOP | NEW_TASK 的标记 
### Application onCreate 优化
#### 视觉
原生逻辑是冷启动过程中会创建一个空白的 Window，等到应用创建第一个 Activity 后才将该 Window 替换    
如果 Application 或 Activity 启动的过程太慢，导致系统的 BackgroundWindow 没有及时被替换，就会出现启动时白屏或黑屏的情况      
可以设置预览图、自定义 Theme 等优化用户等待加载时的观感
### ContentProvider 优化
### MultiDex 优化
## 热启动/温启动
### Activity 复用
### 进程保活策略
## 启动任务编排
### 有向无环图（DAG）任务依赖
### IdleHandler
可以在 MessageQueue 空闲的时候执行任务  
![image](https://github.com/user-attachments/assets/9f4a2fbb-11f5-4495-bd93-c6eee56e9e0d)  
1.在启动的过程中，可以借助 idleHandler 来做一些延迟加载的事情， 比如在启动过程中 Activity 的 onCreate 里面 addIdleHandler，这样在 Message 空闲的时候，可以执行这个任务  
2.进行启动时间统计：比如在页面完全加载之后，调用 activity.reportFullyDrawn 来告知系统这个 Activity 已经完全加载，用户可以使用了，比如在主页的 List 加载完成后，调用 activity.reportFullyDrawn  
### 异步并行初始化
#### 异步 init
![image](https://github.com/user-attachments/assets/00093f59-fd7c-44fb-9fdc-fc4f116f56c2)   
将 init 分为多个子 task 放在子 thread（提交到线程池），task 之间无依赖关系     
此外，若 init 在 Application.onCreate 结束前完成，可用 CountDownLatch 等待 task 执行  
![image](https://github.com/user-attachments/assets/1d9c5cb5-9197-4da4-9843-7efb5e08b8f2)    
根据依赖关系排序生成有向无环图，最后由优先级执行 task。任务可以分为（init 前、后，空闲，子、主线程等 task）       
![image](https://github.com/user-attachments/assets/850fa3c3-02b0-456f-90d4-96fd73ba98c6)  
#### 延迟 init
优先级不高，可在启动完成后执行的 init task延迟到启动完成后执行（如 handler.postdelay），但延迟后的页面加载完成可能造成手势失效（如滑动）。  
此外，init launcher 利用 IdleHandler 实现主线程空闲时执行任务，不影响用户操作。    
![image](https://github.com/user-attachments/assets/4b555acb-2c7d-4545-8327-8a93b4b853a6)
### 启动框架（Alpha/Anchors） 
## 闪屏优化
### WindowBackground 预加载
### 避免白屏/黑屏
## 类加载优化
### 类预加载
一个类的加载耗时不多，但是在几百上千的基数上，也会延迟启动时间，将进入首页的 class 对象，使用线程池提前预加载进来，在类下次使用时则可以直接使用而不需要触发类加载   
Class.forName() 只加载类本身以及静态变量的引用类，new 类实例可以额外加载类成员变量的引用类  
确定哪些类需要提前加载，可以切换系统的 ClassLoader，在自定义 ClassLoader 里面每个类 load 时加一个 log，在项目中运行一次，这样就可以拿到所有 log，也就是需要异步加载的类  
### Dex 重排（Facebook ReDex / 字节码插桩）
## 资源预加载
### 主题资源/字体/关键图片预加载
## GC 抑制
### 启动期间抑制 GC
### 对象预分配
# 内存优化	
## 内存泄漏
### Activity/Fragment 泄漏
### 静态引用
### Handler 泄漏
### 监听器未注销
## OOM
### 大图压缩
### Bitmap 复用（inBitmap）
### 堆外内存管理
### Large Heap 策略
## 内存抖动
### 频繁 GC
### 对象池复用
### 避免循环内创建对象
## Bitmap优化
### inSampleSize
### RGB_565
### BitmapRegionDecoder
### Glide/Coil 配置
## Large Object 频繁分配
## 内存碎片化
### Java 堆碎片化
### Native 堆碎片化
### 碎片化检测
### 碎片整理策略
## 后台进程内存占用
### 主动内存收缩
### 内存快照对比分析
### 子进程内存预算管控
### 进程保活与内存代价
## 匿名内存（Anonymous Memory）增长异常
### Native 内存泄漏
### 线程数量膨胀
### JNI 层 GlobalRef 未释放
### 匿名 mmap 滥用
## Native 内存
### malloc/free 泄漏
### Jemalloc
### 内存映射（mmap）优化
## 分级缓存
### LruCache + DiskLruCache 二级缓存策略
## LMK 机制
### Low Memory Killer 原理
### 进程优先级
### oom_adj 管理
## 内存监控
### KOOM
### LeakCanary
### Hprof 裁剪上报
### 内存水位线告警
# 卡顿优化
## 主线程耗时	
### StrictMode 检测
### 主线程 IO/网络/锁等待
## 卡顿监控	
### Looper 消息监控
### WatchDog 机制
## Binder 
### 调用	Binder 线程池耗尽
### 跨进程调用耗时监控
## 系统调用	
### 主线程SP读写（MMKV 替代）
### 文件 IO 异步化
## 线程调度	
### 线程优先级
### CPU 亲和性
### 线程饥饿问题
## 消息队列	
### Handler 消息积压
### IdleHandler 合理使用
# 渲染优化	
## 帧率优化
### Choreographer 帧监控
### FrameMetrics API
### Janky Frame 分析
## 布局优化
### ConstraintLayout 减少层级
### Merge/ViewStub
### 布局扁平化
## 过度绘制
### GPU 过度绘制检测
### 背景清理
### clipRect/quickReject
## 列表优化
### RecyclerView 预取
### DiffUtil
### ItemAnimator 优化
### Paging3
## 自定义 View
### onDraw 避免对象创建
### 硬件加速
### Path/Paint 复用
## RenderThread
### 减少 DisplayList 构建耗时
### 属性动画优化
## 窗口动画
### 共享元素动画
### 过渡动画优化
## Compose 优化
### 重组（Recomposition）控制
### remember/derivedStateOf
### LazyColumn 优化
# 包体积优化	
## 代码裁剪	
### ProGuard/R8 混淆
### 无用代码删除
### 方法数优化
## 资源优化	
### 资源混淆（AndResGuard）
### 无用资源删除、lint 检查
## 图片压缩	
### WebP 转换
### 矢量图（VectorDrawable）替代
## So 库优化	
### ABI 过滤（只保留 arm64-v8a）
### So 动态下发
## 动态化方案	
### 插件化
### 热修复（Tinker/Robust）
### RN/Flutter 动态下发
AAB 构建	App Bundle 按需下发、语言/屏幕密度分包
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
## 网络监控	请求耗时、成功率、流量统计、OkHttp Interceptor
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
