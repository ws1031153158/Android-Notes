# 包体积优化	
## APK 结构
```
APK 本质是一个 ZIP 文件，解压后：

├── AndroidManifest.xml    （二进制XML）
├── classes.dex            （Java/Kotlin 字节码）
├── classes2.dex           （MultiDex）
├── classes3.dex           ...
├── res/                   （编译后的资源）
│   ├── layout/
│   ├── drawable/
│   ├── drawable-xxhdpi/
│   └── ...
├── resources.arsc         （资源索引表）
├── assets/                （原始资源，不编译）
├── lib/                   （Native so 库）
│   ├── arm64-v8a/
│   ├── armeabi-v7a/
│   └── x86/
└── META-INF/              （签名信息）

典型 App 各部分占比：
┌──────────────┬────────────┬─────────────────────┐
│ 组成部分     │ 典型占比   │ 优化空间             │
├──────────────┼────────────┼─────────────────────┤
│ so 库        │ 30%~50%    │ 大，ABI过滤/压缩     │
│ dex          │ 20%~35%    │ 中，混淆/裁剪        │
│ 资源文件     │ 15%~30%    │ 大，压缩/去重        │
│ resources.arsc│ 3%~8%    │ 中，去除无用配置     │
│ assets       │ 5%~15%    │ 中，压缩             │
└──────────────┴────────────┴─────────────────────┘

// bash
# 分析 APK 中各部分大小
# Android Studio → Build → Analyze APK

# 命令行分析
# apkanalyzer 工具（NDK 自带）

# 查看 dex 中各包的方法数
apkanalyzer dex packages app-release.apk

# 查看具体类的大小
apkanalyzer dex code --class com.xxx.MainActivity app-release.apk

# 查看资源大小
apkanalyzer resources configs app-release.apk

# 对比两个 APK 的差异
apkanalyzer apk compare old.apk new.apk
```
## Dex 优化
### ProGuard/R8 混淆压缩
1.R8：  
```
R8 是 Google 推出的新一代编译器
整合了 ProGuard 的所有功能，并做了更多优化

R8 vs ProGuard：
┌──────────────────┬──────────────┬──────────────────┐
│ 功能             │ ProGuard     │ R8               │
├──────────────────┼──────────────┼──────────────────┤
│ 代码压缩         │ ✅           │ ✅（更激进）      │
│ 代码混淆         │ ✅           │ ✅               │
│ 代码优化         │ 有限         │ ✅（更强）        │
│ Dex 生成         │ 需要 D8      │ 直接生成          │
│ Kotlin 支持      │ 一般         │ 原生支持          │
│ 编译速度         │ 慢           │ 快               │
│ 包体积减少       │ 基准         │ 比ProGuard再小10%│
└──────────────────┴──────────────┴──────────────────┘

ProGuard：源码 → javac → .class → ProGuard(混淆/裁剪) → dex工具 → .dex  
R8：源码 → kotlinc/javac → .class → R8(混淆/裁剪/优化/dex，一步完成，速度更快) → .dex

R8 的额外优化：
├── 内联（Inlining）：小方法直接内联，消除方法调用开销
├── 类合并（Class Merging）：合并只有一个实现的接口
├── 枚举优化：将简单枚举转换为 int 常量
├── Kotlin 特性优化：
│   ├── 移除 data class 未使用的 component 方法
│   ├── 优化 companion object
│   └── 内联 lambda
└── 常量折叠：编译期计算常量表达式
```

2.开启：  
```
// build.gradle 开启 R8 完整模式
android {
    buildTypes {
        release {
            minifyEnabled true      // 开启 R8 压缩混淆
            shrinkResources true    // 开启资源压缩
            proguardFiles getDefaultProguardFile(
                'proguard-android-optimize.txt' // 使用优化版规则
            ), 'proguard-rules.pro'
        }
    }

    // 开启 R8 完整模式（更激进的优化）
    // gradle.properties 中添加：
    // android.enableR8.fullMode=true
}
```

3.ProGuard 规则细化：  
```
# proguard-rules.pro

# ================================
# 基础保留规则
# ================================

# 保留注解（反射需要）
-keepattributes *Annotation*(注释)
-keepattributes Signature(泛型信息，序列化需要) 
-keepattributes SourceFile,LineNumberTable  # 保留行号（崩溃定位）

# 保留Framework 组件
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider

# 保留Native方法（JNI调用）
-keepclasseswithmembernames class * {native <methods>;}

# 保留枚举（Enum有特殊方法）
-keepclassmembers enum * {public static **[] values(); public static ** valueOf(java.lang.String);}

# 保留序列化 Parcelable
-keepclassmembers class * implements android.os.Parcelable {public static final android.os.Parcelable$Creator CREATOR;}

# ================================
# 反射相关保留
# 运行时通过反射访问的类，R8看不懂字符串，会认为不可达而删除
# ================================

# 保留通过反射访问的类/方法/字段
-keepclassmembers class com.xxx.model.** {
    <fields>;
}

# 保留数据模型(JSON 序列化的模型类)
-keep class com.xxx.model.** { *; }
-keep class com.xxx.network.response.** { *; }

# ================================
# 第三方库规则
# ================================

# Retrofit
-keepattributes Exceptions
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

# Gson
-keep class com.google.gson.** { *; }
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# ================================
# 精细化：只保留必要的内容
# ================================

# 不保留 BuildConfig（减少 dex 大小）
-dontwarn com.xxx.BuildConfig

# 移除日志（release 包不需要）
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}

# 移除 Kotlin 的空检查（性能优化，有风险）
# -assumenosideeffects class kotlin.jvm.internal.Intrinsics {
#     static void checkParameterIsNotNull(...);
# }
```
#### 问题排查
```
1. 生成 mapping.txt（混淆映射表）
build/outputs/mapping/release/mapping.txt

2. 还原崩溃堆栈
混淆后的堆栈：
at a.a.a(Unknown Source:3)
使用 retrace 还原：
java -jar retrace.jar mapping.txt stacktrace.txt

3. 查看哪些代码被删除
build.gradle 添加：
android {
    buildTypes {
        release {
            // 生成使用情况报告
        }
    }
}
查看 build/outputs/mapping/release/usage.txt
里面列出了所有被删除的类和方法

4. 查看种子文件（被保留的类）
build/outputs/mapping/release/seeds.txt
```
#### Full Mode
```
// gradle.properties 开启 R8 Full Mode
android.enableR8.fullMode=true

// 注意：Full Mode 可能导致更多兼容性问题
// 需要充分测试
```

与普通的区别：
```
普通 R8：
├── 保守地处理接口默认方法
├── 不合并某些边界情况的类
└── 保留更多"可能被反射访问"的代码

Full Mode 额外优化：
├── 更激进的接口内联
├── 接口默认方法内联
├── 更多类合并
├── 包体积额外减少 5%~10%
├── 删除更多"看起来没用"的代码
└── 对反射的假设更激进（可能误删反射访问的代码）
```
##### 踩坑
Gson 反序列化失败：
```
// 场景：用 Gson 解析服务端 JSON

data class UserResponse(
    @SerializedName("user_id")
    val userId: String,
    @SerializedName("user_name")
    val userName: String,
    val age: Int
)

// 问题：Full Mode 下，UserResponse 的字段可能被删除
// 因为 R8 看不到这些字段被"直接"访问
// Gson 通过反射访问字段，R8 静态分析不到

// 症状：
val user = gson.fromJson(json, UserResponse::class.java)
println(user.userId)   // null！字段被删除了
println(user.userName) // null！
println(user.age)      // 0！

// 解决方案一：keep 规则
-keepclassmembers class com.xxx.model.** {
    <fields>;
}

// 解决方案二：@Keep 注解（更精准）
@Keep
data class UserResponse(
    @SerializedName("user_id")
    val userId: String,
    ...
)

// 解决方案三：换用 Moshi + Kotlin codegen（推荐）
// Moshi 的 @JsonClass(generateAdapter = true)
// 编译期生成适配器，不依赖反射
// R8 可以正确分析，不需要 keep 规则
@JsonClass(generateAdapter = true)
data class UserResponse(
    @Json(name = "user_id") val userId: String,
    @Json(name = "user_name") val userName: String,
    val age: Int
)
```

接口默认方法被错误内联:
```
// 场景：接口有默认方法实现

interface Clickable {
    fun click()
    fun showOff() = println("I'm clickable!") // 默认实现
}

interface Focusable {
    fun setFocus(b: Boolean) = println("I ${if (b) "got" else "lost"} focus.")
    fun showOff() = println("I'm focusable!")
}

class Button : Clickable, Focusable {
    override fun click() = println("I was clicked")
    // 没有 override showOff()，存在歧义！
}

// Full Mode 可能错误处理这种多接口默认方法冲突
// 症状：运行时 AbstractMethodError 或调用了错误的实现

// 解决方案：明确 override
class Button : Clickable, Focusable {
    override fun click() = println("I was clicked")
    override fun showOff() {
        super<Clickable>.showOff() // 明确指定
    }
}
```

枚举被错误优化:
```
// 场景：通过字符串获取枚举值

enum class Direction { NORTH, SOUTH, EAST, WEST }

// 代码中使用
val dir = Direction.valueOf("NORTH") // 反射！

// Full Mode 可能删除枚举的 values() 或 valueOf() 方法
// 症状：NoSuchMethodError

// 解决方案：keep 枚举
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

// 或者避免使用 valueOf（用 when 替代）
fun parseDirection(s: String): Direction = when(s) {
    "NORTH" -> Direction.NORTH
    "SOUTH" -> Direction.SOUTH
    else -> Direction.NORTH
}
// 这样 R8 可以静态分析，不需要 keep
```

Retrofit 接口方法被删除:
```
// 场景：Retrofit 通过动态代理调用接口方法

interface ApiService {
    @GET("/users/{id}")
    suspend fun getUser(@Path("id") id: String): User

    @POST("/login")
    suspend fun login(@Body request: LoginRequest): LoginResponse
}

// Full Mode 可能认为这些接口方法没被直接调用
// （实际是通过动态代理调用的）→ 删除方法！
// 症状：运行时找不到方法，请求失败

// 解决方案
-keep,allowobfuscation interface com.xxx.api.ApiService
-keepclassmembers,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

// 更好的方案：Retrofit 2.9+ 支持 R8 规则自动配置
// 在 retrofit 的 jar 中已包含 consumer-rules.pro
// 确保使用最新版本的 Retrofit
```

ServiceLoader 失效:
```
// 场景：使用 Java SPI 机制（ServiceLoader）
// META-INF/services/com.xxx.Plugin 文件中注册实现类

// Full Mode 可能删除这些实现类（认为没被引用）
// 症状：ServiceLoader.load() 返回空

// 解决方案
-keep class * implements com.xxx.Plugin { *; }

// 或者用 @AutoService 注解 + keep 规则
```
#### 无用代码裁剪（Tree Shaking）
1.原理：  
```
本质是可达性分析，和 JVM GC 的标记算法思路相同，从"根节点"出发，遍历所有可达代码：  
R8 的根节点（不会被删除的起点）：

① AndroidManifest.xml 中声明的组件  
<activity android:name=".MainActivity"/>  
<service android:name=".MyService"/>  
└── 这些类及其生命周期方法是根节点

② -keep 规则声明的类/方法  
-keep class com.xxx.MainActivity { *; }

③ 反射可能访问的类（R8无法静态分析反射）  
Class.forName("com.xxx.SomeClass") → R8 看不懂  
→ 需要手动 -keep，否则可能被误删！

④ 注解处理器生成的代码入口

⑤ JNI 调用的 Java 方法  
JNIEXPORT void Java_com_xxx_NativeLib_init  
→ 对应的 Java native 方法是根节点
```

标记：
白色 = 未访问（初始状态，可能被删除）  
灰色 = 已发现但未完全处理  
黑色 = 已完全处理（确定保留）

① 所有根节点标记为灰色，加入工作队列  

② 取出灰色节点，分析它引用的所有代码：  
   ├── 调用的方法 → 标记为灰色  
   ├── 访问的字段 → 标记为灰色  
   ├── 继承的父类 → 标记为灰色  
   ├── 实现的接口 → 标记为灰色  
   └── 注解 → 标记为灰色  

③ 当前节点标记为黑色（处理完毕）  

④ 重复②③直到工作队列为空   

⑤ 所有仍为白色的节点 → 删除  

// 原始代码
class Utils {
    fun usedMethod() { println("被使用") }
    fun unusedMethod() { println("没人调用我") } // 会被删除
}

class DeadClass { // 整个类没被引用，会被删除
    fun method() {}
}

// R8 分析调用链：
// 入口点（Activity/Application）→ 追踪所有可达代码
// 不可达的代码 → 直接删除

// 配置入口点（keep规则）
// proguard-rules.pro
-keep class com.xxx.MainActivity { *; }
-keep class * extends android.app.Application { *; }
```
```

2.开启： 
```
// 使用 lint 检测无用代码
// ./gradlew lint

// build.gradle 配置
android {
    lintOptions {
        // 检测未使用的资源
        check 'UnusedResources'
        // 检测未使用的代码
        check 'UnusedIds'
        // 将警告变为错误（强制修复）
        warningsAsErrors false
        // 生成报告
        htmlReport true
        htmlOutput file("$buildDir/reports/lint-results.html")
    }
}
```

3.优化：  
```
// ================================
// 优化一：避免不必要的 companion object
// ================================

// ❌ companion object 会生成额外的类
class BadClass {
    companion object {
        const val TAG = "BadClass"
        fun create(): BadClass = BadClass()
    }
}
// 生成：BadClass + BadClass$Companion（两个类！）

// ✅ 使用顶层函数/属性
const val TAG = "GoodClass"  // 顶层常量，直接内联
fun createGoodClass() = GoodClass()  // 顶层函数

// ================================
// 优化二：避免 Kotlin 扩展函数滥用
// ================================

// 每个扩展函数都会生成一个静态方法
// 大量扩展函数 = dex 方法数增加

// ❌ 为每个类型都写扩展函数
fun String.isValidEmail(): Boolean = contains("@")
fun String.isValidPhone(): Boolean = matches(Regex("\\d{11}"))
fun String.isValidIdCard(): Boolean = length == 18
// 生成 3 个静态方法

// ✅ 合并到工具类（如果不需要链式调用）
object StringValidator {
    fun isValidEmail(s: String) = s.contains("@")
    fun isValidPhone(s: String) = s.matches(Regex("\\d{11}"))
}

// ================================
// 优化三：data class 精简
// ================================

// data class 自动生成：equals/hashCode/toString/copy/componentN
// 如果不需要这些方法，考虑用普通 class

// ❌ 不需要 copy/component 但用了 data class
data class Config(val debug: Boolean, val version: String)
// 生成了 copy/component1/component2 等方法（浪费）

// ✅ 只需要 equals/hashCode 时
class Config(val debug: Boolean, val version: String) {
    override fun equals(other: Any?): Boolean { ... }
    override fun hashCode(): Int { ... }
}
```
#### 缩包(Minification)
1.混淆：  
```
// 混淆前
package com.example.payment

class PaymentProcessor {
    private var userBalance: Double = 0.0

    fun processPayment(amount: Double): Boolean {
        return userBalance >= amount
    }
}

// 混淆后（R8生成）
package com.example.payment

class a {           // PaymentProcessor → a
    private var a: Double = 0.0  // userBalance → a

    fun a(a: Double): Boolean {  // processPayment → a
        return this.a >= a
    }
}
```

类名/方法名/字段名 → 短名称：  
类名按字母顺序的最短名称，如：  
第1个类 → a  
第2个类 → b  
...  
第26个类 → z  
第27个类 → aa  
第28个类 → ab  
...  
方法名、字段名同理，但在各自的命名空间内独立计数

包名压缩：
```
// 混淆前
com.example.app.feature.payment.PaymentProcessor
com.example.app.feature.payment.CardValidator
com.example.app.feature.user.UserManager
com.example.app.feature.user.ProfileHelper

// 混淆后（包名压缩）
a.a.a.a  // PaymentProcessor
a.a.a.b  // CardValidator
a.a.b.a  // UserManager
a.a.b.b  // ProfileHelper

// 或者更激进（repackageclasses）
// 所有类放到同一个包下
-repackageclasses 'x'
// 结果：
x.a  // PaymentProcessor
x.b  // CardValidator
x.c  // UserManager
x.d  // ProfileHelper
```

字符串不混淆:
```
// 默认情况：字符串不混淆
class ApiConfig {
    val baseUrl = "https://api.example.com"        // 明文！
    val secretKey = "sk_live_abcd1234"             // 危险！
    val sqlQuery = "SELECT * FROM users"           // 暴露表结构！
}

// 反编译后完全可见：
// const-string v0, "https://api.example.com"
// const-string v1, "sk_live_abcd1234"

// 解决方案一：字符串加密（R8 StringEncryption，需要付费版）

// 解决方案二：手动加密敏感字符串
object SecureConfig {
    // 运行时解密，不在字节码中明文存储
    val secretKey: String
        get() = decrypt("加密后的密文")

    private fun decrypt(cipher: String): String {
        // 解密逻辑
        return XORDecrypt(cipher, BuildConfig.DECRYPT_KEY)
    }
}

// 解决方案三：NDK存储（放到so中）
// 敏感字符串在C++层，更难逆向

// 解决方案四：服务端下发
// 不在客户端存储敏感信息

减少包体积:  
类名从平均20字符 → 1~2字符,包路径从30字符 → 1字符  
删除无用代码  
内联（减少方法数）  
减少 Dex 文件头开销
```

2.内联：  
```
R8 决定是否内联的判断条件：

✅ 可以内联：
├── 方法体足够小（字节码指令数 < 阈值，约5~10条）
├── 方法只被调用一次
├── 非虚方法（private/final/static）
└── 内联后不会导致类循环依赖

❌ 不能内联：
├── 方法体太大（内联会导致调用方膨胀）
├── 递归方法
├── 虚方法（可能有多个实现）
├── 被 -keep 保留的方法
└── 包含异常处理的复杂方法

// 优化前
fun add(a: Int, b: Int) = a + b
fun calculate() = add(1, 2) // 函数调用有开销

// ❌ 普通高阶函数：lambda 会生成匿名类
fun <T> measure(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时: ${System.currentTimeMillis() - start}ms")
    return result
}
// 每次调用都创建一个匿名类对象

// ✅ inline 函数：lambda 直接内联，无匿名类
inline fun <T> measureInline(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时: ${System.currentTimeMillis() - start}ms")
    return result
}
// 编译后：lambda 代码直接插入调用处，无额外类

// 优化后（R8内联）
fun calculate() = 3 // 直接计算结果，消除函数调用
```

3.常量折叠:  
```
val MAX_SIZE = 100
val DOUBLE_SIZE = MAX_SIZE * 2 // R8直接替换为 200

4.无效代码消除:    
// 无效代码消除
fun example(debug: Boolean = false) {
    if (debug) {
        // BuildConfig.DEBUG = false 时
        // R8 直接删除整个if块
        Log.d("TAG", "debug info")
    }
}
```

4.类合并（Class Merging）:  
```
// 只有一个子类的抽象类 → 合并为一个类
// 减少类数量，降低方法数
// 场景：只有一个实现的抽象类/接口

// 合并前
abstract class BaseRepository {
    abstract fun fetchData(): String
    fun logFetch() { println("fetching...") }
}

class UserRepository : BaseRepository() {
    // UserRepository 是 BaseRepository 的唯一子类
    override fun fetchData(): String = "user data"
}

// 使用方
val repo: BaseRepository = UserRepository()
repo.fetchData()

// R8 类合并后：
// BaseRepository 被合并进 UserRepository
// 消除了一层继承
class UserRepository {  // 不再继承 BaseRepository
    fun fetchData(): String = "user data"
    fun logFetch() { println("fetching...") }  // 直接合并进来
}

// 效果：
// ├── 减少一个类（减少方法数）
// ├── 消除虚方法调用（变为直接调用）
// └── 为进一步内联创造条件
```

5.参数移除:  
```
// 未使用的方法参数 → 直接删除
fun process(data: String, unused: Int) { // unused被删除
    println(data)
}
```
#### Dex 分包优化
```
生成Dex文件
.class 文件（JVM字节码）和 .dex 文件的区别：

.class 文件：
├── 每个类一个文件
├── 常量池：每个类独立
├── 指令集：JVM字节码（基于栈）
└── 设计目标：JVM运行

.dex 文件：
├── 多个类合并到一个文件
├── 常量池：所有类共享（去重！）
├── 指令集：Dalvik字节码（基于寄存器）
└── 设计目标：移动设备，内存紧张

R8 Dexing 步骤：
① 字节码转换：JVM字节码 → Dalvik字节码
   
   // JVM字节码（基于栈）：
   ILOAD_1    // 将局部变量1压栈
   ILOAD_2    // 将局部变量2压栈
   IADD       // 弹出两个值，相加，结果压栈
   IRETURN    // 弹出栈顶，返回
   
   // Dalvik字节码（基于寄存器）：
   add-int v0, v1, v2  // v0 = v1 + v2，直接操作寄存器
   return v0

② 共享常量池构建
   // 多个类中相同的字符串/类型引用 → 合并为一份
   // 例：100个类都引用 "java/lang/String"
   // .class：100个常量池各存一份
   // .dex：共享常量池只存一份 → 大幅减小体积

③ 方法数检查与分包
   // 单个 Dex 文件方法数上限：65536（0xFFFF）
   // 超过则分为 classes.dex + classes2.dex + ...
   
   // R8 的分包策略：
   // 将启动相关类优先放入 classes.dex（主Dex）
   // 其余类放入 secondary Dex

④ 优化 Dex 布局
   // 按类的调用关系排列，提高局部性
   // 相关联的类放在相邻位置
   // 减少运行时的缺页中断
```
## 资源优化
### 图片优化
```
图片格式对比：
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ 格式     │ 有损压缩 │ 无损压缩 │ 透明度   │ 支持版本 │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ PNG      │ ❌       │ ✅       │ ✅       │ 全部     │
│ JPEG     │ ✅       │ ❌       │ ❌       │ 全部     │
│ WebP     │ ✅       │ ✅       │ ✅       │ API 14+  │
│ AVIF     │ ✅       │ ✅       │ ✅       │ API 31+  │
│ SVG      │ -        │ -        │ ✅       │ 全部     │
└──────────┴──────────┴──────────┴──────────┴──────────┘

WebP vs PNG/JPEG：
├── 有损 WebP：比 JPEG 小 25%~34%
├── 无损 WebP：比 PNG 小 26%
└── 支持透明度（PNG 的优势 WebP 也有）

推荐策略：
├── 照片类图片：有损 WebP（质量 85）
├── 图标/UI 图片（有透明度）：无损 WebP
├── 简单图形：VectorDrawable（SVG）
└── 动画：Lottie（JSON 格式，比 GIF 小很多）
```

转换脚本：  
```
# 批量转换 PNG 到 WebP
# 使用 cwebp 工具（Google 官方）

# 单个文件转换
cwebp -q 85 input.png -o output.webp

# 批量转换脚本
find ./res -name "*.png" | while read file; do
    output="${file%.png}.webp"
    cwebp -q 85 "$file" -o "$output"
    # 如果 webp 更小，删除原 png
    if [ $(stat -f%z "$output") -lt $(stat -f%z "$file") ]; then
        rm "$file"
        echo "转换成功: $file → $output"
    else
        rm "$output"
        echo "跳过（webp更大）: $file"
    fi
done

# Android Studio 内置转换
# 右键图片 → Convert to WebP
```

图片压缩：  
```
// 图片压缩 Gradle 插件（构建时自动压缩）
// 使用 pngquant/optipng 压缩 PNG

class ImageCompressPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.tasks.register("compressImages") { task ->
            task.doLast {
                val resDir = File(project.projectDir, "src/main/res")
                compressAllImages(resDir)
            }
        }

        // 在 preBuild 之前执行
        project.tasks.named("preBuild").configure {
            it.dependsOn("compressImages")
        }
    }

    private fun compressAllImages(resDir: File) {
        resDir.walkTopDown()
            .filter { it.extension in listOf("png", "jpg", "jpeg") }
            .forEach { imageFile ->
                val originalSize = imageFile.length()
                compressImage(imageFile)
                val newSize = imageFile.length()
                val saved = originalSize - newSize
                if (saved > 0) {
                    println("压缩: ${imageFile.name} " +
                        "${originalSize/1024}KB → ${newSize/1024}KB " +
                        "(节省 ${saved/1024}KB)")
                }
            }
    }

    private fun compressImage(file: File) {
        when (file.extension.lowercase()) {
            "png" -> {
                // 使用 pngquant 有损压缩
                ProcessBuilder("pngquant", "--force", "--quality=65-80",
                    "--output", file.absolutePath, file.absolutePath)
                    .start().waitFor()
            }
            "jpg", "jpeg" -> {
                // 使用 mozjpeg 压缩
                ProcessBuilder("jpegoptim", "--max=85", file.absolutePath)
                    .start().waitFor()
            }
        }
    }
}
```
### 无用资源删除（shrinkResources）
```
// build.gradle 资源裁剪配置
android {
    buildTypes {
        release {
            shrinkResources true  // 删除未使用的资源
            minifyEnabled true    // 必须同时开启代码压缩
        }
    }

    // 只保留中文和英文（删除其他语言资源）
    // 减少 strings.xml 的语言版本
    defaultConfig {
        resConfigs "zh", "zh-rCN", "en"
        // 删除其他语言：fr/de/ja/ko 等
        // 对于只做中文市场的 App，可以只保留 zh
    }

    // 只保留需要的屏幕密度资源
    // 现代手机基本都是 xxhdpi 或 xxxhdpi
    defaultConfig {
        resConfigs "xxhdpi", "xxxhdpi"
        // 删除 ldpi/mdpi/hdpi/xhdpi 的图片资源
        // 系统会自动缩放 xxhdpi 的图片
    }
}
```

自定义：  
```
// 自定义资源裁剪：删除重复资源
// 不同分辨率目录下内容完全相同的图片

object DuplicateResourceFinder {

    fun findDuplicates(resDir: File): Map<String, List<File>> {
        val md5ToFiles = mutableMapOf<String, MutableList<File>>()

        resDir.walkTopDown()
            .filter { it.isFile && it.extension in
                listOf("png", "jpg", "webp", "xml") }
            .forEach { file ->
                val md5 = file.readBytes().md5()
                md5ToFiles.getOrPut(md5) { mutableListOf() }.add(file)
            }

        // 返回有重复的文件组
        return md5ToFiles.filter { it.value.size > 1 }
    }

    fun removeDuplicates(duplicates: Map<String, List<File>>) {
        duplicates.forEach { (_, files) ->
            // 保留分辨率最高的版本，删除其他
            val sorted = files.sortedByDescending { file ->
                // 按目录分辨率排序：xxxhdpi > xxhdpi > xhdpi
                when {
                    file.path.contains("xxxhdpi") -> 4
                    file.path.contains("xxhdpi") -> 3
                    file.path.contains("xhdpi") -> 2
                    file.path.contains("-hdpi") -> 1
                    else -> 0
                }
            }
            // 删除除最高分辨率外的重复文件
            sorted.drop(1).forEach { it.delete() }
        }
    }
}
```

resources.arsc:  
```
resources.arsc：资源索引表
包含所有资源的 ID → 文件路径 映射

优化方向：
├── 删除未使用的配置（语言/密度）
│   → 通过 resConfigs 配置
├── 资源混淆（AndResGuard）
│   → 路径变短，arsc 文件变小
└── 删除重复的字符串
    → 相同的字符串只存储一次

resources.arsc 不能压缩！
Android 7.0+ 要求 resources.arsc 不压缩
（需要内存映射直接访问）
```
### 资源混淆（AndResGuard）
### 重复资源去重
### 语言/分辨率裁剪
## so 库优化
### ABI 过滤（只保留 arm64-v8a）
```
Android 支持的 ABI：
├── arm64-v8a：64位 ARM（主流，覆盖95%+设备）
├── armeabi-v7a：32位 ARM（老设备）
├── x86_64：64位 x86（模拟器）
└── x86：32位 x86（模拟器）

每个 ABI 都有一套 so 文件
保留所有 ABI = so 大小 × 4

优化策略：
只保留 arm64-v8a
├── 覆盖市面上 95%+ 的真实设备
├── so 大小减少约 75%（4套→1套）
└── 64位 ARM 设备可以运行 32位 so（向后兼容）
   但反过来不行（32位设备不能运行64位so）
```

启用：  
```
// build.gradle ABI 过滤
android {
    defaultConfig {
        ndk {
            // 只保留 arm64-v8a
            abiFilters "arm64-v8a"

            // 如果需要兼容老设备（32位ARM）
            // abiFilters "arm64-v8a", "armeabi-v7a"
        }
    }

    // 按 ABI 分包（不同设备下载对应版本）
    // 配合 AAB 使用效果更好
    splits {
        abi {
            enable true
            reset()
            include "arm64-v8a", "armeabi-v7a"
            universalApk false  // 不生成通用包
        }
    }
}
```
### 动态下发
```
// 大型 so 文件（如 AI 模型推理库）动态下发
// 首次使用时才下载，减少安装包大小

object SoDynamicLoader {

    private const val SO_DOWNLOAD_URL = "https://cdn.xxx.com/so/"
    private const val SO_DIR = "dynamic_so"

    // 检查并加载 so
    suspend fun loadSo(
        context: Context,
        soName: String,
        version: String
    ): Boolean {
        val soFile = getSoFile(context, soName, version)

        return if (soFile.exists()) {
            // 本地已有，直接加载
            loadLocalSo(soFile)
        } else {
            // 下载后加载
            downloadAndLoad(context, soName, version)
        }
    }

    private fun getSoFile(
        context: Context,
        soName: String,
        version: String
    ): File {
        return File(
            context.getDir(SO_DIR, Context.MODE_PRIVATE),
            "${soName}_${version}.so"
        )
    }

    private fun loadLocalSo(soFile: File): Boolean {
        return try {
            System.load(soFile.absolutePath)
            true
        } catch (e: UnsatisfiedLinkError) {
            soFile.delete() // 文件损坏，删除重新下载
            false
        }
    }

    private suspend fun downloadAndLoad(
        context: Context,
        soName: String,
        version: String
    ): Boolean = withContext(Dispatchers.IO) {
        val soFile = getSoFile(context, soName, version)
        val url = "$SO_DOWNLOAD_URL${soName}_${version}_arm64.so"

        try {
            // 下载 so 文件
            downloadFile(url, soFile)

            // 验证完整性（MD5 校验）
            if (!verifyMd5(soFile, getExpectedMd5(soName, version))) {
                soFile.delete()
                return@withContext false
            }

            // 加载
            loadLocalSo(soFile)
        } catch (e: Exception) {
            soFile.delete()
            false
        }
    }
}

// 使用示例：AI 推理功能按需加载
class AIFeature {

    private var isLoaded = false

    suspend fun initialize(context: Context): Boolean {
        if (isLoaded) return true

        // 动态加载 AI 推理 so（可能 20MB+）
        isLoaded = SoDynamicLoader.loadSo(
            context,
            soName = "libai_inference",
            version = "1.2.0"
        )
        return isLoaded
    }

    fun predict(input: FloatArray): FloatArray {
        check(isLoaded) { "AI so 未加载" }
        return nativePredict(input)
    }

    private external fun nativePredict(input: FloatArray): FloatArray
}
```
### 符号裁剪（strip）
```
# so 文件包含两种符号表：
# .symtab：完整符号表（调试用）
# .dynsym：动态符号表（运行时需要）

# strip 命令：删除 .symtab，只保留 .dynsym
# NDK 构建时默认会 strip release 版本的 so

# 手动 strip
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/\
llvm-strip --strip-unneeded libnative.so

# 查看 so 大小变化
ls -la libnative.so          # strip 前
llvm-strip libnative.so
ls -la libnative.so          # strip 后（通常减少 30%~60%）

# CMakeLists.txt 配置（确保 release 时 strip）
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_link_options(-Wl,--strip-all)
endif()
```

cmake:  
```
# CMakeLists.txt 完整优化配置
cmake_minimum_required(VERSION 3.18.1)
project("myapp")

# 编译优化选项
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} \
    -O2 \                    # 优化级别
    -fvisibility=hidden \    # 默认隐藏符号（只导出需要的）
    -ffunction-sections \    # 每个函数独立段（配合--gc-sections）
    -fdata-sections"         # 每个数据独立段
)

# 链接优化选项
set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} \
    -Wl,--gc-sections \      # 删除未使用的段（Dead Code Elimination）
    -Wl,--strip-all \        # 删除所有符号（release包）
    -Wl,--icf=all"           # 相同函数合并（Identical Code Folding）
)

add_library(mylib SHARED native.cpp)
```
### 合并 so
## 架构优化
### 插件化（功能动态下发）
### AAB（Android App Bundle）
```
AAB（Android App Bundle）：
Google Play 的新发布格式

传统 APK 发布：
开发者 → 上传一个通用 APK → 所有用户下载同一个 APK
问题：APK 包含所有语言/密度/ABI 的资源，用户只用其中一部分

AAB 发布：
开发者 → 上传 AAB → Google Play 按需生成 APK
├── 用户设备：arm64-v8a + zh + xxhdpi
│   → 只下载 arm64 so + 中文资源 + xxhdpi 图片
└── 用户设备：armeabi-v7a + en + xhdpi
    → 只下载 armeabi so + 英文资源 + xhdpi 图片

效果：
├── 用户下载的 APK 减少 15%~65%
├── 开发者不需要手动管理多个 APK
└── 支持 Dynamic Feature（功能模块按需下载）

AAB 文件结构：
├── base/                  （基础模块）
│   ├── dex/
│   ├── res/
│   ├── assets/
│   ├── lib/
│   └── manifest/
├── feature_payment/       （支付功能模块，按需下载）
│   └── ...
└── feature_camera/        （相机功能模块，按需下载）
    └── ...
```
### 按需下载
```
// 功能模块按需下载（Dynamic Feature）

// 1. 在 build.gradle 中声明 Dynamic Feature
// feature_camera/build.gradle
plugins {
    id 'com.android.dynamic-feature'
}
android {
    // ...
}
dependencies {
    implementation project(':app') // 依赖基础模块
}

// app/build.gradle
android {
    dynamicFeatures = [':feature_camera', ':feature_payment']
}

// 2. 运行时按需下载
class MainActivity : AppCompatActivity() {

    private val splitInstallManager by lazy {
        SplitInstallManagerFactory.create(this)
    }

    fun openCamera() {
        // 检查模块是否已安装
        if (splitInstallManager.installedModules.contains("feature_camera")) {
            startCameraFeature()
            return
        }

        // 请求下载
        val request = SplitInstallRequest.newBuilder()
            .addModule("feature_camera")
            .build()

        splitInstallManager.startInstall(request)
            .addOnSuccessListener { sessionId ->
                // 下载开始
            }
            .addOnFailureListener { exception ->
                // 下载失败
                handleInstallFailure(exception)
            }
    }

    // 监听安装状态
    private val installStateListener = SplitInstallStateUpdatedListener { state ->
        when (state.status()) {
            SplitInstallSessionStatus.DOWNLOADING -> {
                val progress = state.bytesDownloaded() * 100 / state.totalBytesToDownload()
                showProgress(progress.toInt())
            }
            SplitInstallSessionStatus.INSTALLED -> {
                // 安装完成，可以使用功能
                startCameraFeature()
            }
            SplitInstallSessionStatus.FAILED -> {
                showError("安装失败: ${state.errorCode()}")
            }
        }
    }

    override fun onResume() {
        super.onResume()
        splitInstallManager.registerListener(installStateListener)
    }

    override fun onPause() {
        super.onPause()
        splitInstallManager.unregisterListener(installStateListener)
    }
}
```
## 编译优化
### D8
```
// gradle.properties
# 开启 D8 增量编译
android.enableD8.desugaring=true

# 开启 R8 完整模式
android.enableR8.fullMode=true

# 开启 dex 合并优化
android.enableDexingArtifactTransform=true
```
### Baseline Profile
### 压缩算法选择
```
// APK 中不同文件的压缩策略

android {
    packagingOptions {
        // so 文件不压缩（Android 6.0+ 支持直接内存映射执行）
        // 不压缩 so 虽然 APK 变大，但安装后占用空间不变
        // 且加载速度更快（无需解压）
        jniLibs.useLegacyPackaging = false  // false = 不压缩 so

        // 排除重复文件
        excludes += [
            'META-INF/LICENSE',
            'META-INF/LICENSE.txt',
            'META-INF/NOTICE',
            'META-INF/NOTICE.txt',
            '**/*.kotlin_module'
        ]
    }
}
```
## 监控
```
// 包体积监控：防止包体积悄悄增大

// build.gradle 添加包体积检查 task
tasks.register("checkApkSize") {
    doLast {
        val apkFile = fileTree("$buildDir/outputs/apk/release")
            .matching { include("**/*.apk") }
            .singleFile

        val apkSizeMB = apkFile.length() / 1024.0 / 1024.0

        println("APK 大小: ${"%.2f".format(apkSizeMB)} MB")

        // 超过阈值则构建失败
        val maxSizeMB = 50.0
        if (apkSizeMB > maxSizeMB) {
            throw GradleException(
                "APK 大小 (${"%.2f".format(apkSizeMB)}MB) " +
                "超过限制 (${maxSizeMB}MB)！"
            )
        }
    }
}

// 在 assembleRelease 后自动检查
tasks.named("assembleRelease").configure {
    finalizedBy("checkApkSize")
}

// Python
# 包体积分析脚本：对比两个版本的差异
# compare_apk.py

import zipfile
import sys
from pathlib import Path

def analyze_apk(apk_path: str) -> dict:
    """分析 APK 各部分大小"""
    sizes = {
        'dex': 0, 'so': 0, 'res': 0,
        'assets': 0, 'arsc': 0, 'other': 0
    }

    with zipfile.ZipFile(apk_path) as apk:
        for info in apk.infolist():
            size = info.file_size
            name = info.filename

            if name.endswith('.dex'):
                sizes['dex'] += size
            elif name.startswith('lib/') and name.endswith('.so'):
                sizes['so'] += size
            elif name.startswith('res/'):
                sizes['res'] += size
            elif name.startswith('assets/'):
                sizes['assets'] += size
            elif name == 'resources.arsc':
                sizes['arsc'] += size
            else:
                sizes['other'] += size

    return sizes

def compare_apks(old_apk: str, new_apk: str):
    """对比两个 APK 的差异"""
    old_sizes = analyze_apk(old_apk)
    new_sizes = analyze_apk(new_apk)

    print(f"{'组成部分':<15} {'旧版本':>10} {'新版本':>10} {'差异':>10}")
    print("-" * 50)

    total_old = total_new = 0
    for key in old_sizes:
        old = old_sizes[key]
        new = new_sizes.get(key, 0)
        diff = new - old
        total_old += old
        total_new += new

        diff_str = f"+{diff/1024:.1f}KB" if diff > 0 else f"{diff/1024:.1f}KB"
        print(f"{key:<15} {old/1024:>8.1f}KB {new/1024:>8.1f}KB {diff_str:>10}")

    print("-" * 50)
    total_diff = total_new - total_old
    diff_str = f"+{total_diff/1024:.1f}KB" if total_diff > 0 else f"{total_diff/1024:.1f}KB"
    print(f"{'总计':<15} {total_old/1024:>8.1f}KB {total_new/1024:>8.1f}KB {diff_str:>10}")

if __name__ == '__main__':
    compare_apks(sys.argv[1], sys.argv[2])
```
