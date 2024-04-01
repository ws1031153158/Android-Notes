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
进程优先级：前台 -> 可见 -> 服务 -> 后台 -> 空
### Android ClassLoader
Boost：加载Android Framework 层 class 文件  
BaseDex:Path/Dex（只提供构造函数，加载逻辑位于父类 BaseDex）：  
Path：Android 应用程序类加载器，加载指定 dex，以及 jar、zip、apk 中的 classes.dex  
Dex：加载指定的 dex，jar、zip、apk 中的 classes.dex 类
