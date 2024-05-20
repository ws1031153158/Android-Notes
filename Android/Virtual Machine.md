# DVM
针对 Android 创造的 VM，JVM 使用栈存取信息，是对内存的存取，DVM 使用寄存器来进行，是对 CPU 的一块区域进行存取，效率更高，Zygote（本身也是一个 DVM 进程）会在新应用启动时，去 fork 进程并创建 DVM 实例，此外，DVM 允许多个进程共享一个内存  
JVM：.java -> .class  
DVM：.java -> .dex  
Dex 文件最多存储 65536 个方法（所有方法提取 id 索引存在链表中，长度用 short 保存），multidex 在打包时，把一个应用分成多个 dex，加载时追加到类加载器中的 DexPathList 对应数组(存储 dexfile)，解决方法数限制（application.attachBaseContext 调用 MultiDex.install(this)加载后续的 dex，从文件解压（第一次启动或覆盖安装后启动）；从缓存中加载（启动后有缓存，不用解压）），创建 DexFile 对象通过 DexFile 的 Native 方法 openDexFile 打开 dex 文件，做一些优化并生成对应 odex 文件（App 加载类通过 odex 文件进行）
ODEX:  
分离程序资源和可执行文件以及预编译处理，加快加载速度和开机速度  
adb shell getprop dalvik.vm.heapsize 命令：  
heapStartSize 为初始值大小，越小内存消耗越慢，越大应用越流畅，但可运行应用数量会减少  
heapGrowthLimit 为单个应用可用最大内存，若 largeHeap 为 true 则 app 使用内存到 heapsize 才会 OOM，否则到此就会 OOM  
heapsize 为堆内存最大值，超过必 OOM
## ART
DVM 支持 JIT（运行时的即时编译，将字节码转换为机器码）    
ART在 JIT 之外支持预编译，在安装时将部分应用字节码转换为机器码（为了后续热加载）保存在本地，无需每次运行时执行编译  
DVM 的 GC 为标记-清除，有两次 SWT（遍历和标记），ART 只有一次暂停，且通过 pre-cleaning 在暂停前做预处理，减少暂停工作量  
DVM 为 32 位（4 字节）CPU，ART 支持 64 位（8 字节）也兼容 32 位（一次性处理数据量）  
DVM 仅固定一种回收算法，ART 回收算法可运行期选择。ART 具备内存整理能力，减少内存空洞。       
进程优先级：前台 -> 可见 -> 服务 -> 后台 -> 空
### Android ClassLoader
Boost：加载Android Framework 层 class 文件  
BaseDex:Path/Dex（只提供构造函数，加载逻辑位于父类 BaseDex）：  
Path：Android 应用程序类加载器，加载指定 dex，以及 jar、zip、apk 中的 classes.dex  
Dex：加载指定的 dex，jar、zip、apk 中的 classes.dex 类
# GC
## Algorithm
Android 的内存堆是分代的，它会根据分配对象的预期寿命和大小跟踪不同的分配存储分区。例如，最近分配的对象属于「新生代」。当某个对象保持活动状态达足够长的时间时，可将其提升为「老年代」，然后是「永久代」。
### Young
1.新生成的对象首先都是放在年轻代中，目标是尽可能快速地回收生命周期短的对象。  
2.由一个 Eden 区和两个 Survivor 区按照 8:1:1 比例组成，大部分新对象都在 Eden 区中，当 Eden 区满时，存活的对象将被复制到 From 区，然后清空 Eden 区；当 From 区满时，将 Eden 和 From 区中存活对象复制到 To 区，然后清空 Eden 和 From 区，将 To 区和 From 区交换，即保持 From 区为空，如此往复。  
3.通常情况下，在 Survivor 区连续存活超过一定 GC 次数（默认：15）的对象，会升级到老年代存放；但如果一个对象在新生代 Minor GC 后依然无法存放（Eden 和 Survivor 都放不下），也会直接存放到老年代。新生代发生的 GC 也叫做 Minor GC，Minor GC 发生频率比较高，不一定等 Eden 区满了才会触发。
### Old
1.存放的都是一些生命周期较长的对象，当老年代中存满时触发 Full GC，Full GC 发生频率比较低，年老代对象存活时间较长，存活率比较高。  
2.收采用了标记-清理算法，相对比较耗时，并且为了保证在清理过程中引用不发生改变，通常需要暂停所有其他的用户线程（STW，Stop The World），这也是为什么 Full GC 会对进程产生较大的性能影响。
### Permanent
1.用于存放静态的类和方法，该区域比较稳定，不属于堆区，不会进行垃圾回收，对 GC 没有显著影响，这一部分也被称为运行时常量。（在 JDK 1.8 及之后的版本，在本地内存中实现的元空间（Meta-space）已经代替了持久代）  
## Memory
### Dispatch
1.为不同类型的进程分配了不同的内存使用上限，同设备的确切堆大小上限取决于设备的总体可用 RAM 大小，默认给每个 App 分配的内存大小为 16M，分配值和最大值受具体设备影响，AndroidManifest 的 application 节点中设置属性 Android:largeHeap="true" 来突破上限，运行过程中出现了内存泄漏的而造成应用进程使用的内存超过了这个上限，则会被系统视为内存泄漏（OOM）。  
### Recycle
Low Memory Killer 机制：内存不足，针对所有进程回收，低优先级进程优先回收。进程分类，回收收益，能保证进程大部分情况下不会出现内存不足的情况，虚拟机使用分页和内存映射来管理内存。  
如果应用修改的任何内存，无论修改的方式是分配新对象还是清除内存映射的页面，都会一直驻留在 RAM 中，并且无法换出。若要从应用中释放内存，只能释放应用保留的对象引用，使内存可供垃圾回收器回收，一旦确定程序不再使用某块内存，它就会将该内存重新释放到堆中，无需开发者进行任何干预。尽管垃圾回收速度非常快，但仍会影响应用的性能。
