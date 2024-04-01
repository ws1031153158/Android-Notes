# flavor
flavorDimensions 定义 flavor 的 dimension，productFlavors 配置不同的 flavor 并指定 dimension 和 applicationid，在 BuildConfig 中来判断当前 flavor
# proguard
增大反编译难度，同时减少包体大小（实际字符串替换为混淆后的字符串，可设置 minifyEnabledproguardFiles），此外，可能被外部调用的子类不需要混淆、本地 native 方法、android-support 包、序列化的类等
# resource
可设置 shrinkresource 来去掉无用资源、图片格式采用 webp 格式等减小包体积
## 加载
resource 实现类为 resourceimpl（委托机制），通过 resource 访问 res，使用 AssetManager 访问 assets（resource 也是通过 AssetManager 访问被编译过的资源，但是会先通过 id 找对应文件名，AssetManager 同时可以访问未编译的资源），apk 文件路径通过 assets.addAssetPath 方法设置，可以通过反射修改这个地方将 插件的 apk 文件添加到宿主中。  
创建 AssetManager 对象，调用 addAssetPath 将插件 apk 路径作为参数传入 ，随后将 AssetManager  对象作为参数，创建 Resources 对象，返回给插件
# 打包
首先是资源，将各种图片等 resource、assets 以及 manifest 变为二进制文件，接着将 AIDL 文件转化为 Java 文件，再将 Java、Kotlin 文件转换为 .class 文件，并进一步变为 .dex（odex）文件，编成 APK 后进行 zipAlign 将未压缩的数据对齐，最后发布 key。
# hotfix
## 热修复代码层面：  
一、底层替换法：  
AOP 编程，直接在 Native 层修改原有类（无需重启），但限制多，不能增减原有类的方法和字段（破坏类结构，索引变化，也就无法进行功能迭代）, 加载快，实时生效，但是兼容性差  
1.替换 ArtMethod 结构体中的字段，但有兼容问题（zfb.AndFix）  
2.同时使用类加载和底层替换，针对小修改，在底层替换方案限制范围内，判断是否支持，是就采用底层替换（替换整个 ArtMethod 结构体，不存在兼容问题），否则使用类加载替（tb.Sophix）  
二、Instant Run 法：  
代码改动后进行增量构建，仅构建改变代码，以补丁形式增量部署到设备，进行代码热替换，实时生效，无需重启，破坏结构（侵入式修改）  
类加载：修改指定类后打包成含 dex 的 patch.jar，放在 dexElements 数组第一个元素，先加在修改后的 class 后面的同名 class 不会再加载，但需重启来重新加载类，不即时生效  
.java -> .class -> folder -> .jar -> .dex
## 热修复资源层面：
对 AssetManager 进行修改（替换为自定义的 AssetManager），参考 Instant Run 原理：反射构建 AssetManager，获取到 addAssetPath 方法来获取 外部资源（如 SD 卡），activity.res.mAssets.set 以及 activity.theme.mAssets.set 设置 AssetManager，通过不同的 sdk 版本获取资源的 WeakReference 集合， mAssets.set 将 AssetManager 设置
## 热修复 so 库：
一、加载 so 方法替换：  
System.load 有下发则加载自定义路径 so 调用 System.loadLibaray，无下发则加载已安装 APK.so  
二、反射注入 so 路径：  
System.loadLibaray 调用 Runtime.loadLibrary ：classloader 为 null 则去遍历 libPath 返回 path 配置的路径数组，拼接出 so 路径并传入 nativeLoad，classloader 不为 null 则 findLibrary（nativeLibraryPathElements 中元素对应一个 so库） 获取参数（so 路径）， 使用 nativeLoad
# multidex
单个 dex 文件引用的 Java 方法总数不能超过 65536(short 范围，通过索引查找具体对象(在指令中占 2 字节)) ，拆分成多个 dex,一般 Dalvik 只执行优化后的 odex 文件  
拆分：multiDexEnabled 为 true 则打 apk 时 dex 拆分  
解压&压缩：将 apk 中的除主 dex 以外的解压（load 第一次冷启动才会解压，后续获取缓存文件），压缩 zip  
反射填充：将除主 dex 以外的加载，将加载后的内容通过反射的填充到原 classloader（DexPathList 维护的集合）（installSecondaryDexs 安装）（install 方法为入口，APK 文件中的 dex（优化为 odex）添加到应用的类加载器 PathClassLoader 中的 DexPathList 的 Element 数组）
