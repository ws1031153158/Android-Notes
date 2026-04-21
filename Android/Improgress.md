# 启动优化
## AppStarter
不同的库使用不同的 ContentProvider 进行初始化，导致 ContentProvider 太多，管理杂乱，影响耗时  
可以移除三方库的 ContentProvider 启动时候自动初始化的步骤，手动通过 LazyLoad 的方式启动，此外，应用开发者可以控制各个库的初始化时机或者初始化顺序  
![image](https://github.com/user-attachments/assets/8fbae98c-a7a8-404a-bcc4-3e00f03082ad)  
meta-data 当中加入了一个 tools:node= remove
## IdleHandler
可以在 MessageQueue 空闲的时候执行任务  
![image](https://github.com/user-attachments/assets/9f4a2fbb-11f5-4495-bd93-c6eee56e9e0d)  
1.在启动的过程中，可以借助 idleHandler 来做一些延迟加载的事情， 比如在启动过程中 Activity 的 onCreate 里面 addIdleHandler，这样在 Message 空闲的时候，可以执行这个任务  
2.进行启动时间统计：比如在页面完全加载之后，调用 activity.reportFullyDrawn 来告知系统这个 Activity 已经完全加载，用户可以使用了，比如在主页的 List 加载完成后，调用 activity.reportFullyDrawn  
# 内存优化
## Foundation
### meminfo
当出现内存偏高或其他内存相关的问题时，我们可以通过 meminfo 来缩小范围、查找潜在的风险点  
获取方式：adb shell dumpsys meminfo <pkg>  
meminfo统计：  
![image](https://github.com/user-attachments/assets/2c26f42e-2494-4f9b-80ef-0460746b48ef)  
对照 meminfo 来看：  
  ![image](https://github.com/user-attachments/assets/594eb71d-63d4-4701-9ff7-118565932f28)  
纵轴：  
![image](https://github.com/user-attachments/assets/52028878-2de4-4f24-8b63-7c674513f70e)  
横轴：  
![image](https://github.com/user-attachments/assets/95f30ffb-ce1a-4321-a7f1-851bd3f5fc59)  
App Summary:  
![image](https://github.com/user-attachments/assets/2a57c902-f440-455b-bd04-0697365c48e7)
### hprof
heap profile，是某一时间点，应用进程堆转存生成的文件，包含了这一时间点的内存快照；当需要进一步拆解内存、定位问题时，进行获取并分析    
![image](https://github.com/user-attachments/assets/4ba978f7-7d5d-44a4-b06a-24d684ff6145)

## 内存抖动
短时间内频繁大量创建临时对象，会频繁 GC，无论哪种方式实现的GC在执行时都不可避免的需要 STW（Stop The World），STW 意味着所有的工作线程都将被暂停，虽然时间很短，但终究会存在时间成本，一两次内存回收不易被察觉，但多次内存回收集中在短时间内爆发，这就会造成较大程度的界面卡顿风险。   
尽量避免在循环体中创建对象、尽量不要在自定义 View 的 onDraw 方法中创建对象（会被频繁调用）、对于可复用对象，可以考虑使用对象池缓存。  
## OOM
JVM 在面临内存资源不足时的一种自我保护机制，遭遇 OOM 时，不会立即终止执行，无法为对象分配内存空间时，抛出 OOM（Error 子类，标志着一种通常不可恢复的、严重的运行时问题，本身不会直接导致 JVM 退出），JVM 会将这个错误传递给当前正在运行的代码，这给予了应用程序一个机会去捕获并处理这个异常，尽管在常规情况下并不推荐捕获和处理这种严重错误，但如果确实进行了这样的操作，程序可能会尝试继续执行。   
1.堆内存不足（OutOfMemoryError: Java heap space）：    
最常见，随着对象的持续创建，如果它们因为某些原因（如内存泄漏）而无法被垃圾收集器有效回收，那么堆内存最终会被消耗殆尽，这种情况往往是因为代码中存在内存管理不当的问题。   
2.元空间/方法区空间不足（OutOfMemoryError: PermGen space/Metaspace）：    
当系统加载大量的类和方法时，这部分内存资源可能会变得紧张，通常发生在应用程序需要动态加载大量代码的场景中。   
3.本地方法栈空间不足（StackOverflowError）：   
如果线程请求的栈大小超出了JVM 所允许的最大值，就会导致本地方法栈溢出，通常与线程的设计和实现有关。   
4.请求内存超过物理和虚拟内存：   
请求的内存超过了物理内存和虚拟内存的限制时，也会触发 OutOfMemoryError，不仅仅与 JVM 的内存设置有关，还受到整个系统配置的影响。   
5.解决：  
通过调整堆内存大小、选择合适的垃圾收集器等手段，可以更好地适应应用程序的内存需求，减少OOM的发生。  
利用缓存技术可以有效减少内存使用，避免创建过多的大型对象也可以降低OOM的风险。  
## 内存泄露
程序中已动态分配的堆内存由于某种原因未被释放或无法释放，造成系统内存浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。  
1.单例引起的内存泄漏：  
生命周期与程序生命周期一样，若有对象不再使用，却被单例持有，就会导致其无法回收，进而导致内存泄漏。  
如 context 对象传入了Activity，就会导致 Activity 销毁时无法被回收，传入 Application 上下文可解决。  
2.非静态内部类创建静态实例引起的内存泄漏：  
非静态内部类会默认持有外部类引用，若在外部类中去创建这个内部类对象，当频繁打开关闭Activity时，会导致重复创建对象，造成资源浪费。  
可把该实例设为静态，解决重复创建实例问题，但会引起其他问题（静态成员变量生命周期和程序生命周期一样），静态成员变量持有 Activity 引用，所以 Activity 销毁时，无法回收。  
3.Handler引起的内存泄漏：  
若 Handler 通过内部类创建，内部类持有外部类引用（Activity），而队列中的消息 target 指向 Handler，即消息持有 Handler 引用，若 Activity 销毁时队列中的消息还未处理完，这些未处理完的消息会持有 Activity 引用，导致 Activity 无法回收。  
Handler 改为静态内部类，Activity#onDestroy 中移除队列中消息，Handler 内弱引用 Activity 对象，让内部类不再持有外部类的引用时，程序就不允许 Handler 操作 Activity 对象。  
4.Asynctask引起的内存泄漏：  
和 Handler 类似，Asynctask 内部类持有外部类引用。  
改为静态内部类并在 onDestroy 中取消任务即可。  
5.WebView 引起的内存泄漏：  
不同的版本上和不同的机型上会存在不同的问题。  
不在 xml 中定义，定义一个 view 容器（例 FrameLayout），在代码中动态添加（context 可使用弱引用），onDestroy 销毁时直接 layout.removeAllViews，移除WebView。  
6.资源对象未关闭引起的内存泄漏：  
广播、服务、数据库cursor、多媒体、文件、套接字等。  
不使用时需关闭。  
7.三方注册资源未取消注册：  
如 EventBus。  
为便于管理 Activity，将其添加到 Activity 栈中，销毁时未从栈中移除等。  
## Bitmap
图片内存 = 宽高一个像素占用内存（和色彩模式有关，如 ARGB_8888 为 4（8*4 = 32位 = 4字节））。  
Bitmap 对象放在 Java 堆，像素数据放在 Native 内存，8.0 新增 NativeAllocationRegistry 来辅助回收 Native 内存，以及可减少图片内存并提升绘制效率的硬件位图 Hardware Bitmap ，Bitmap 的 Native 内存可和对象一起释放， GC 避免内存滥用。此外，Glide 会自动调整加载的图片大小（根据 Imageview，以及三级缓存优化图片加载）。  
## LeakCanary
hook Android 生命周期，自动检测当 Activity、Fragment 销毁时实例是否回收，销毁的实例传给 RefWatcher（持有它们的弱引用），保留实例（Retained Instance）数量达到阈值会进行堆转储，数据放进 hprof 文件（APP 可见阈值为 5，不可见为 1），会解析 hprof 文件，找出导致 GC 无法回收实例的引用链，就是泄漏踪迹（Leak Trace，最短强引用路径，GC Roots 到实例的路径）。  
监听系统内存状态：  
ComponentCallback2：在 Activity 中实现 ComponentCallback2 接口获取系统内存的相关事件, 在 onTrimMemory(level)  回调针对不同事件做不同释放内存操作  
ActivityManager.getMemoryInfo()：  
返回一个 ActivityManager.MemoryInfo 对象，包含系统当前内存状态（可用内存、总内存、低杀内存阈值， lowMemory 布尔值判断是否处于低内存态）  
## 可优化点
### 自动装箱  
尽量使用基本数据类型来代替封装数据类型，int 比 Integer 要更加有效，其它数据类型也是一样。自动装箱的核心是把基础数据类型转换成对应的复杂类型。自动装箱转化时，会产生一个新的对象，这样就会产生更多的内存和性能开销。如 int 只占4字节，而 Integer 对象有16字节，特别是 HashMap 这类容器，进行增、删、改、查操作时，都会产生大量的自动装箱操作。   
### 内存复用   
1.资源复用：通用的字符串、颜色定义、简单页面布局的复用。  
2.对象池：显示创建对象池，实现复用逻辑，对相同的类型数据使用同一块内存空间。  
3.Bitmap对象的复用：使用 inBitmap 属性可以告知 Bitmap 解码器尝试使用已经存在的内存区域，新解码的 Bitmap 会尝试使用之前那张 Bitmap 在 heap 中占据的 pixel data 内存区域。  
### 择优数据结构  
1.SparseArray与ArrayMap    
Android 自身还提供了一系列优化过后的数据集合工具类，如 SparseArray、SparseBooleanArray、LongSparseArray，使用这些 API 可以让我们的程序更加高效。  
HashMap 工具类会相对比较低效，因为它需要为每一个键值对都提供一个对象入口，而 SparseArray 就避免掉了基本数据类型转换成对象数据类型的时间。  
ArrayMap 提供了和 HashMap 一样的功能，但避免了过多的内存开销，方法是使用两个小数组，而不是一个大数组。并且 ArrayMap 在内存上是连续不间断的。总体来说，在 ArrayMap 中执行插入或者删除操作时，从性能角度上看，比 HashMap 还要更差一些，但如果只涉及很小的对象数，比如1000以下，就不需要担心这个问题了。因为此时 ArrayMap 不会分配过大的数组。  
2.避免使用枚举类型    
枚举最大的优点是类型安全，但在 Android 平台上，枚举的内存开销是直接定义常量的三倍以上。  
每一个枚举值都是一个单例对象，在使用它时会增加额外的内存消耗，所以枚举相比与 Integer 和 String 会占用更多的内存。大量使用 Enum 会增加 DEX 文件的大小，会造成运行时更多的 IO 开销，使我们的应用需要更多的空间。特别是分 Dex 多的大型 App，枚举的初始化很容易导致 ANR。
### 避免对象创建
1.我们可以在字符串拼接的时候尽量少用 +=，多使用 StringBuffer，StringBuilder。    
2.不要在 onMeause、onLayout、onDraw 中去刷新 UI（requestLayout）。    
3.自定义 View 的onLayout、onDraw、列表遍历等频繁调用的方法里创建对象。这些方法会被多次调用，在其内部创建对象会导致系统频繁申请存储空间并触发 GC，导致内存抖动，严重会导致 OOM。
### 慎用 Service
如果应用程序当中需要使用 Service 来执行后台任务，一定注意只有当任务正在执行的时候才让 Service 运行起来。另外，当任务执行完之后去停止 Service 时，要小心 Service 停止失败导致内存泄漏的情况。  
启动一个 Service 时，系统会倾向于将这个 Service 所依赖的进程进行保留，这样就会导致这个进程变得非常消耗内存。并且系统可以在 LruCache 当中缓存的进程数量也会减少，导致切换应用程序的时候耗费更多性能，严重的话，甚至有可能会导致崩溃。因为系统在内存非常吃紧的时候可能已无法维护所有正在运行的 Service 所依赖的进程了。  
为了能够控制 Service 的生命周期，Android 官方推荐的最佳解决方案就是使用 IntentService，这种 Service 的最大特点就是当后台任务执行结束后会自动停止，从而极大程度上避免了 Service 内存泄漏的可能性。  
通常避免使用持久性服务，它们会持续请求使用可用内存。建议采用 WorkManager等替代实现方式。  
## 图片优化
### 图片格式
PNG：无损压缩图片方式，支持 Alpha 通道，切图素材大多用这种格式。  
JPEG：有损压缩图片格式，不支持背景透明和多帧动画，适用于色彩丰富的图片压缩，不适合于 logo。  
WEBP：支持有损和无损压缩，支持完整的透明通道，也支持多帧动画，是一种比较理想的图片格式。  
.9图：点九图实际上仍然是 png 格式图片，它是针对 Andorid 平台特殊的图片格式，体积小，拉伸变形，能指定位置拉伸或者填充，能很好的适配机型。  
无损 webp 平均比 png 小26%，有损 jpeg 平均比 webp 少24%-35%，无损 webp 支持 Alpha 通道，有损 webp 在一定条件下也支持。采用 webp 在保持图片清晰情况下，可以优先减少磁盘空间大小。可以将 drawable 中的 png、jpg 格式图片转换为 webp 格式图片。  
### 像素格式
ALPHA_8，内存 1B，色彩组成为透明度，比较少用到。  
RGB_565，内存2B，色彩组成为颜色，不需要 Alpha 通道的，特别是 .JPG 格式的。  
ARGB_4444，内存2B，色彩组成为颜色+透明度，内存只占 ARGB_8888 的一半，已被废弃。  
ARGB_8888，内存4B，色彩组成为颜色+透明度，系统默认的像素点格式。  
通过替换系统 drawable 默认色彩通道（BitmapFactory.Options.inPreferredConfig），将部分没有透明通道的图片格式由 ARGB_8888 替换为 RGB565，在图片质量上的损失几乎肉眼不可见，而每张图片可以节省1/2的内存。但不通用，取决于图片是否有透明度需求。
### 采样率
在把图片载入内存之前，我们需要先计算出一个合适的 inSampleSize 缩放比例，降低图片像素，来达到降低图片占用内存大小的目的，避免不必要的大图载入。  
采样率 inSampleSize 只能是整数(只能是2的次方)，不能很好保证图片质量。如果 inSampleSize=2，则最终内存占用就会是原来的1/4（宽高都为原来的1/2），适用于图片过大的情况。  
## Bitmap
### 释放 bitmap 对象  
Bitmap 使用完后需要调用 recycle() 方法回收资源，否则会发生内存泄漏。bitmap.recycle 用于释放与当前 Bitmap 对象相关联的 Native 对象，并清理对像素数据的引用。但不能同步地释放像素数据，而是在没有其它引用的时候，简单地允许像素数据被作为垃圾回收掉。  
Bitmap 在内存中的存储分两部分 ：一部分是 Bitmap 对象，另一部分为对应的像素数据，前者占据的内存较小，而后者才是内存占用的大头。
### 复用  
可以通过 LruCache 等缓存机制来管理 Bitmap 对象的复用。除此之外，可以使用 BitmapFactory.Options 的 inBitmap 属性来指定一个可复用的 Bitmap 对象。
### 释放 imageview 资源
bitmap 资源和 background 资源都要回收，对其调用 recycle，并且将维护的局部/全局对象等置为 null。
## GC 优化
1.锁屏 GC  
2.需要资源的特殊场景不 GC  
3.后台 GC  
但是，正常来讲 GC 是系统决定的，非必要应用侧不应该主动调用显式 GC，而是去分析内存问题
## Tips
1.SparseArray：只有 integer 类型属性，避免了自动装箱（基本数据类型和包装器类型（引用类型）转换，包装器 .valueof 装箱，xxxValue 拆箱，实现基本类型和引用类直接运算）的开销，可以代替 hashMap。  
2.使用 static final 优化成员变量，static 会由编译器调用 clinit 方法进行初始化，之后访问的时候会需要先到它那里查找，然后才返回数据。static final 不需做多余的查找动作，打包在 dex 文件中可以直接调用，并不会在类初始化申请内存。基本数据类型的成员，可以全写成 static final。   
3.﻿adb shell dumpsys meminfo pid 可以获取内存信息：  
PSS/RSS：实际使用物理内存（包括进程独占（USS）和共享（PSS按进程占用共享等分））  
Private：进程独占内存  
SWAP PSS：释放后其他进程可以使用的内存，所以只能看到 Dirty，包含在 PSS  
Native Heap：PSS + SWAP PSS DIRETY  
详细内存分布 -> APP 内存（图片、java堆、）-> 总内存 -> 对象内存（view、activity、viewimpl，可当做内存泄漏的依据）  
# ANR
ANR 问题本质是一个性能问题。通过消息机制监测，发生超时消息、移除超时消息、处理超时消息等都在 system server 侧处理  
ANR 机制实际上对应用程序主线程的限制，要求主线程在限定的时间内处理完一些最常见的操作(启动服务、处理广播、处理输入)， 如果处理超时，则认为主线程已经失去了响应其他操作的能力。  
主线程中的耗时操作，譬如密集 CPU 运算、大量 IO、复杂界面布局等，都会降低应用程序的响应能力。  
部分 ANR 问题是很难分析的，有时候由于系统底层的一些影响，导致消息调度失败，出现问题的场景又难以复现。 这类 ANR 问题往往需要花费大量的时间去了解系统的一些行为，超出了 ANR 机制本身的范畴。有一些 ANR 问题很难调查清楚，因为整个系统不稳定的因素很多，例如 Linux Kernel 本身的 Bug 引起的内存碎片过多、硬件损坏等。这类比较底层的原因引起的 ANR 问题往往无从查起。  
## Application
1.死锁  
2.主线程调用 thread 的 join()方法、sleep()方法、wait()方法或者等待线程锁的时候  
3.主线程阻塞在 nSyncDraw  
4.主线程耗时操作，如复杂的 layout，庞大的 for 循环，IO 等  
5.主线程被子线程同步锁 block  
6.主线程等待子线程超时  
7.主线程 Activity 生命周期函数执行超时  
8.主线程 Service 生命周期函数执行超时  
9.主线程 Broadcast.onReceive 函数执行超时（即使调用了 goAsync ）  
10.渲染线程耗时  
11.耗时的网络访问  
12.大量的数据读写  
13.数据库操作  
14.硬件操作（比如 Camera)  
15.service binder 的数量达到上限  
16.其它线程终止或崩溃导致主线程一直等待  
17.Dump 内存操作  
18.大量 SharedPerference 同时读写  
## System Server
1.与 SystemServer 进行 Binder 通信，SystemServer 执行耗时：方法本身执行耗时导致超时，或者 SystemServer Binder 锁竞争太多，导致等锁超时  
2.等待其他进程返回超时，比如从其他进程的 ContentProvider 中获取数据超时（10s，但是这个场景很少见）  
3.Window 错乱导致 Input 超时（5s）  
4.ContentProvider 对端的进程频繁崩溃，也会杀掉当前进程  
5.整机低内存  
6.整机 CPU 占用高：查看是总使用率是否过高，同时需要关注缺页次数（访问的页面不在内存时，会产生一次缺页中断，需要调入主存，一次调入次数加一），xxx minor 表示高速缓存中的缺页次数，可以理解为进程在做内存访问，xxx major 表示内存的缺页次数，可以理解为进程在做 IO 操作   
7.整机 IO 使用率高  
8.SurfaceFlinger 超时  
9.系统冻结功能出现 Bug：应用是否处于 D 状态（TASK_UNINTERRUPTIBLE，不可中断的睡眠状态，通常是由于它正在执行一个不能中断的 I/O 操作，如磁盘读写、网络传输等）    
10.System Server 中 WatchDog 出现 ANR  
11.整机触发温控限制频率  
12.Service 前台 20 s，后台 200 s 超时（system server 在 service 流程开始前会发生 delay 消息到 main handler，若 app 端在 delay 消息内未通知 system server 移除消息，则超时）  
1.Broadcast 前台 10 s，后台 60 s 超时（同样，也是发生 delay 消息，等待 receiver 去移除，contentProvider 一样的机制）
## Native Crash
原生采用信号（软中断信号）量中断处理的捕捉方式，程序运行在用户态，系统调用、中断或异常时进入内核态，信号涉及两种状态转换。  
接收：接收信号由内核代理，收到信号会放到对应进程信号队列中，向进程发送一个中断，使其进入内核态。（此时信号只在队列中，对进程来说暂时不知道信号到来）。  
检测：进程进入内核态有两种场景对信号检测：    
1.从内核态返回用户态前进行检测。  
2.在内核态中，从睡眠状态被唤醒时进行检测。    
有新信号时进入信号处理：处理运行在用户态，处理前，内核将当前栈内数据拷贝到用户栈，修改指令寄存器（eip）并指向信号处理函数，之后返回用户态，执行对应处理函数，处理完成后返回内核态，检查是否有信号未处理。所有信号处理完成，恢复内核栈（用户栈拷贝回来），恢复指令寄存器（eip）并指向中断前的运行位置，最后回到用户态继续执行进程。
## Tips
主线程 sleep 不一定 ANR，休眠期间没有其他消息需要处理则不会，若此时有点击事件或其他线程传来的更新 UI 请求则可能会 ANR  
## Deal
1. am_anr 确定 ANR 发生时间、进程以及原因
2. /data/anr 对应时间点 traces 文件，优先看主线程状态
3. ANR in 看对应 pid 相关信息，如生命周期（am_lifecycle，会展示执行时长）
4. 看 lock 状态（dvm_lock_sample，会展示等锁时长）
5. 看 binder 耗时（binderTransact，会展示 binder 时长），再看对端进程状态（等锁情况）  
# Runtime Crach
app 不会闪退，但进程被杀掉，不会接受任何事件，可以通过 getDefaultUncaughtExceptionHandler 交给系统处理（如捕获到异常为空）。  
try-catch 可以捕获主线程异常。UncaughtExceptionHandler 可以捕获子线程异常，异常发生回调 uncaughtException，捕获到的异常为 Throwable。  
自定义 crashHandler 继承 Thread.UncaughtExceptionHandler，初始化进行，Thread.setDefaultUncaughtExceptionHandler(this) 操作，在 unCaughtException 回调中处理上报异常（有可能此时已未响应，需要创建 looper）。
