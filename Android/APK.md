# 多渠道
## Variant
变体，指应用可以构建不同的版本，还可以针对不同的目标 API 或设备类型，由多个构建类型组合而成，如 debug 与 release，以及构建脚本中定义的产品变种
## Flavor
1.productFlavors 产品变种，定义不同变体，每个变体可以有不同的配置和资源  
2.通过 create 或直接声明渠道名称来创建渠道，并在渠道中按需配置属性参数，如 applicationId（所有渠道保持一致可以保证一个设备只有一个应用安装）、dimension（根据维度划分，同一个维度属于一类集合） 等  
3.在 BuildConfig 中可以判断当前 flavor  
## buildType
构建类型，用来定义构建类型配置，比如是否开启代码混淆、是否开启调试模式等，通常包含 debug 和 release 两种
## Source
1.app/src/main/ 为主目录，相当于默认渠道，可以新增其他渠道目录与 main 区分  
2.渠道构建时，渠道变体会跟主变体目录下的资源进行合并  
3.如有同名配置资源，例如 strings.xml 文件中的 app_name，则优先取渠道配置资源进行覆盖，其他不同名的则进行合并  
4.layout 文件、assets 文件则是替换，渠道资源优先于主变体资源
## Code
1.代码文件是不支持合并的，也不支持同名  
2.main 作为主变体，渠道可以引用 main 中的代码类，main 也可以引用渠道中的代码类，但是当渠道变换时，main 则会出现找不到之前渠道代码类的异常，因为渠道中的代码为该渠道专属，只有在该渠道编译时才会与主变体 main 中的代码进行融合
## SourceSets
1.可以为渠道指定代码路径，以及 res、manifest 等资源文件路径  
2.不同渠道可以相互配置其资源  
3.代码文件、资源代码文件都支持
## Implementation
2.默认依赖：implementation  
3.渠道依赖：变体 + Implementation，如 huaweiImplementation  
4.构建类型：类型 + Implementation，如 debugImplementation  
5.组合变体：变体 + 类型 + Implementation，如 huaweiDebugImplementation
## 打点
1.meta-data 标签通常在 AndroidManifest.xml 文件中使用，通过键值对的方式为组件提供附加配置信息，在 AndroidManifest.xml 中使用标签通过占位符的方式，来存储渠道信息  
2.productFlavors 中通过 manifestPlaceholders 来替换 AndroidManifest.xml 文件中 value 的值    
3.代码中通过  context.getPackageManager().getApplicationInfo(context.getPackageName(), PackageManager.GET_META_DATA) 来获取 ActivityInfo 中的 metadata
## BuildConfig
BuildConfig 通常用来存储一些常量信息，比如版本号，或者在 buildTypes 中根据构建环境来定义接口请求的域名地址等，BuildConfig 会在编译时生成 class 文件，实际上在配置多渠道信息的时候，已经默认会在BuildConfig 中注入渠道信息(FLAVOR)了
## Tips
把渠道相关配置抽成一个 channel.gradle 文件，然后在 app/build.gradle 文件中 apply 依赖进来，这样可以更好的管理和维护渠道项目的渠道配置
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
```
热修复核心问题：
如何在不重新安装 APK 的情况下
修复线上的 Bug？

技术路线：
┌─────────────────────────────────────────────────────┐
│                   热修复技术路线                      │
├──────────────┬──────────────┬───────────────────────┤
│ Java 层修复  │ Native 层修复│ 资源修复               │
├──────────────┼──────────────┼───────────────────────┤
│ 类加载替换   │ so 替换      │ AssetManager 重建      │
│ 方法替换     │ 函数 Hook    │ 资源重新打包           │
│ 字节码插桩   │              │                        │
└──────────────┴──────────────┴───────────────────────┘
```
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
## Robust
原理：  
```
方法代理（插桩）

编译期：在每个方法入口插入代码
运行期：通过 patch 替换方法实现

插桩后的方法结构：
原始方法：
public String getUserName() {
    return this.name;
}

插桩后：
public String getUserName() {
    // 插入的代码：检查是否有 patch
    ChangeQuickRedirect redirect = $change;
    if (redirect != null &&
        PatchProxy.isSupport(args, this, redirect, false)) {
        return (String) PatchProxy.accessDispatch(
            args, this, redirect, false);
    }
    // 原始逻辑
    return this.name;
}

修复时：
① 下载 patch.dex（包含新的方法实现）
② 加载 patch.dex
③ 设置 $change = new PatchedImplementation()
④ 下次调用该方法时，走 patch 逻辑
⑤ 即时生效！不需要重启！
```

实现：  
```
// Robust 插桩的 Gradle 插件原理（简化）
// 使用 ASM 字节码操作框架

class RobustTransform : Transform() {

    override fun transform(invocation: TransformInvocation) {
        invocation.inputs.forEach { input ->
            input.jarInputs.forEach { jarInput ->
                // 处理 jar 文件中的类
                transformJar(jarInput, invocation.outputProvider)
            }
            input.directoryInputs.forEach { dirInput ->
                // 处理目录中的类文件
                transformDirectory(dirInput, invocation.outputProvider)
            }
        }
    }

    private fun transformClass(classBytes: ByteArray): ByteArray {
        val classReader = ClassReader(classBytes)
        val classWriter = ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
        val classVisitor = RobustClassVisitor(classWriter)
        classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES)
        return classWriter.toByteArray()
    }
}

// ASM ClassVisitor：在每个方法入口插入检查代码
class RobustClassVisitor(cv: ClassVisitor) : ClassVisitor(ASM7, cv) {

    private lateinit var className: String

    override fun visit(version: Int, access: Int, name: String, ...) {
        className = name
        // 添加 $change 字段
        cv.visitField(
            ACC_PUBLIC or ACC_STATIC,
            "\$change",
            "Lcom/meituan/robust/ChangeQuickRedirect;",
            null, null
        )
        super.visit(version, access, name, ...)
    }

    override fun visitMethod(
        access: Int, name: String, descriptor: String, ...
    ): MethodVisitor {
        val mv = super.visitMethod(access, name, descriptor, ...)

        // 跳过构造方法和静态初始化块
        if (name == "<init>" || name == "<clinit>") return mv

        // 跳过不需要插桩的方法
        if (shouldSkip(name, descriptor)) return mv

        return RobustMethodVisitor(mv, className, name, descriptor, access)
    }
}

// ASM MethodVisitor：在方法入口插入代理检查
class RobustMethodVisitor(
    mv: MethodVisitor,
    private val className: String,
    private val methodName: String,
    private val descriptor: String,
    private val access: Int
) : MethodVisitor(ASM7, mv) {

    override fun visitCode() {
        // 插入代码：
        // ChangeQuickRedirect redirect = $change;
        // if (redirect != null && PatchProxy.isSupport(...)) {
        //     return PatchProxy.accessDispatch(...);
        // }

        val isStatic = (access and ACC_STATIC) != 0

        // 加载 $change 字段
        mv.visitFieldInsn(
            GETSTATIC,
            className,
            "\$change",
            "Lcom/meituan/robust/ChangeQuickRedirect;"
        )

        val skipLabel = Label()

        // if ($change == null) goto skipLabel
        mv.visitJumpInsn(IFNULL, skipLabel)

        // 调用 PatchProxy.isSupport()
        // ... 省略参数准备代码 ...
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/meituan/robust/PatchProxy",
            "isSupport",
            "([Ljava/lang/Object;Ljava/lang/Object;" +
            "Lcom/meituan/robust/ChangeQuickRedirect;Z)Z",
            false
        )

        // if (!isSupport) goto skipLabel
        mv.visitJumpInsn(IFEQ, skipLabel)

        // 调用 PatchProxy.accessDispatch()
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/meituan/robust/PatchProxy",
            "accessDispatch",
            "([Ljava/lang/Object;Ljava/lang/Object;" +
            "Lcom/meituan/robust/ChangeQuickRedirect;Z)Ljava/lang/Object;",
            false
        )

        // return（根据返回类型处理）
        insertReturn(descriptor)

        mv.visitLabel(skipLabel)

        super.visitCode()
    }

    private fun insertReturn(descriptor: String) {
        val returnType = Type.getReturnType(descriptor)
        when (returnType.sort) {
            Type.VOID -> mv.visitInsn(RETURN)
            Type.BOOLEAN, Type.BYTE, Type.CHAR,
            Type.SHORT, Type.INT -> {
                mv.visitTypeInsn(CHECKCAST, "java/lang/Integer")
                mv.visitMethodInsn(INVOKEVIRTUAL,
                    "java/lang/Integer", "intValue", "()I", false)
                mv.visitInsn(IRETURN)
            }
            Type.LONG -> {
                mv.visitTypeInsn(CHECKCAST, "java/lang/Long")
                mv.visitMethodInsn(INVOKEVIRTUAL,
                    "java/lang/Long", "longValue", "()J", false)
                mv.visitInsn(LRETURN)
            }
            Type.OBJECT, Type.ARRAY -> {
                mv.visitTypeInsn(CHECKCAST, returnType.internalName)
                mv.visitInsn(ARETURN)
            }
        }
    }
}
```

Patch 生成：  
```
// Patch 类的结构
// 每个需要修复的类，生成对应的 Patch 实现

// 原始类（有 Bug）
class UserManager {
    fun getUserName(userId: String): String {
        return database.query(userId) // Bug：可能 NPE
    }
}

// 生成的 Patch 类
class UserManagerPatch : ChangeQuickRedirect {

    override fun isSupport(
        methodName: String,
        paramTypes: Array<Class<*>>
    ): Boolean {
        // 声明支持哪些方法的修复
        return methodName == "getUserName"
    }

    override fun accessDispatch(
        methodName: String,
        args: Array<Any?>
    ): Any? {
        if (methodName == "getUserName") {
            val userId = args[0] as String
            // 修复后的逻辑
            return database.query(userId) ?: "Unknown User"
        }
        return null
    }
}

// 加载 Patch
object RobustPatchLoader {

    fun loadPatch(context: Context, patchApkPath: String) {
        // 加载 patch dex
        val patchClassLoader = DexClassLoader(
            patchApkPath,
            context.cacheDir.absolutePath,
            null,
            context.classLoader
        )

        // 读取 patch 配置（哪些类需要修复）
        val patchInfo = loadPatchInfo(patchClassLoader)

        patchInfo.forEach { (targetClass, patchClass) ->
            // 加载 patch 实现类
            val patchInstance = patchClassLoader
                .loadClass(patchClass)
                .newInstance() as ChangeQuickRedirect

            // 将 patch 注入到目标类的 $change 字段
            val targetClazz = Class.forName(targetClass)
            val changeField = targetClazz.getDeclaredField("\$change")
                .apply { isAccessible = true }
            changeField.set(null, patchInstance)

            // 立即生效！不需要重启
        }
    }
}
```
# multidex
单个 dex 文件引用的 Java 方法总数不能超过 65536(short 范围，通过索引查找具体对象(在指令中占 2 字节)) ，拆分成多个 dex,一般 Dalvik 只执行优化后的 odex 文件  
拆分：multiDexEnabled 为 true 则打 apk 时 dex 拆分  
解压&压缩：将 apk 中的除主 dex 以外的解压（load 第一次冷启动才会解压，后续获取缓存文件），压缩 zip  
反射填充：将除主 dex 以外的加载，将加载后的内容通过反射的填充到原 classloader（DexPathList 维护的集合）（installSecondaryDexs 安装）（install 方法为入口，APK 文件中的 dex（优化为 odex）添加到应用的类加载器 PathClassLoader 中的 DexPathList 的 Element 数组）
# 加密
## 对称加密
加密解密都使用同一个密钥，优点是方便快速，缺点是易受攻击，常见算法如 AES
## 非对称加密
加解密使用不同密钥的加密方式，优点是具有更高的安全性与可靠性，缺点是加解密速度较慢，常见算法如 RSA 
