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

3.人工优化：  
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
const val TAG = "GoodClass"  // const 直接内联,无 getter，此外可以直接作为顶层常量，完全消除 Companion

@JvmStatic
fun createGoodClass() = GoodClass()  // 生成真正的静态方法，而非通过 Companion 访问

object GoodClass {} // 单例，无 Companion

// ================================
// 优化二：避免 Kotlin 扩展函数滥用
// ================================

// 每个扩展函数都会生成一个静态方法(带接收者参数的静态方法)
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

// ① 减少方法调用开销（无需压栈/出栈）
// ② 减少方法数（内联后原方法可能被删除）
```

3.常量折叠:  
```
val MAX_SIZE = 100
val DOUBLE_SIZE = MAX_SIZE * 2 // R8直接替换为 200
// 效果：减少字段引用，dex 指令更少
```

4.无效代码消除:    
```
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

6.枚举优化：  
```
// ================================
// 枚举的字节码开销
// ================================

// 原始枚举
enum class Status {
    LOADING, SUCCESS, ERROR, EMPTY
}

// 生成的字节码（等价于）：
// public final class Status extends Enum<Status> {
//     public static final Status LOADING = new Status("LOADING", 0);
//     public static final Status SUCCESS = new Status("SUCCESS", 1);
//     public static final Status ERROR = new Status("ERROR", 2);
//     public static final Status EMPTY = new Status("EMPTY", 3);
//     private static final Status[] $VALUES = {LOADING, SUCCESS, ERROR, EMPTY};
//
//     public static Status[] values() { ... }
//     public static Status valueOf(String name) { ... }
// }
// 问题：
// ① 每个枚举值都是一个对象（内存开销）
// ② values() 每次调用都创建新数组（内存抖动）
// ③ 方法数：values + valueOf + 每个枚举的方法

// ================================
// R8 的枚举优化（自动）
// ================================
// R8 会将简单枚举转换为 int 常量（EnumUnboxing）
// 条件：枚举没有自定义方法/字段，没有被反射使用

// R8 优化后（等价于）：
// LOADING = 0, SUCCESS = 1, ERROR = 2, EMPTY = 3
// 直接用 int，消除枚举类！

// ================================
// 开发者配合：避免阻止 R8 枚举优化
// ================================

// ❌ 阻止优化：枚举有自定义方法
enum class Status(val displayName: String) {  // 自定义字段
    LOADING("加载中"),
    SUCCESS("成功"),
    ERROR("失败"),
    EMPTY("空数据");

    fun isTerminal() = this == SUCCESS || this == ERROR  // 自定义方法
}
// R8 无法将这个枚举 unbox 为 int

// ✅ 如果只需要状态标识，用 @IntDef 或 sealed class
@IntDef(Status.LOADING, Status.SUCCESS, Status.ERROR, Status.EMPTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Status {
    companion object {
        const val LOADING = 0
        const val SUCCESS = 1
        const val ERROR = 2
        const val EMPTY = 3
    }
}
// 完全没有枚举类，直接用 int 常量

// 或者用 sealed class（类型安全 + 可以携带数据）
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: String) : UiState()
    data class Error(val message: String) : UiState()
    object Empty : UiState()
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

WebP 压缩：  
```
WebP 压缩原理：

有损压缩（类似 JPEG）：
├── 将图片分成 4×4 或 8×8 的块
├── 对每个块做 DCT（离散余弦变换）
├── 量化（丢弃人眼不敏感的高频信息）
└── 熵编码（Huffman/算术编码）

无损压缩（类似 PNG）：
├── 预测编码：用相邻像素预测当前像素
├── 变换：对预测误差做变换
├── 颜色空间变换：RGB → YCbCr
└── 熵编码

WebP vs JPEG/PNG 压缩率：
├── 有损 WebP vs JPEG（相同质量）：小 25%~34%
├── 无损 WebP vs PNG：小 26%
└── 带透明度 WebP vs PNG：小 26%（PNG 没有有损压缩选项）

为什么 WebP 更小？
├── 更先进的预测算法（比 JPEG 的 DCT 更精确）
├── 更好的熵编码（算术编码 vs JPEG 的 Huffman）
└── 支持有损+透明度（JPEG 不支持透明度）
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

VectorDrawable:  
```
原理：
├── 基于 SVG 路径描述图形
├── 运行时由 CPU 渲染成 Bitmap
├── 任意尺寸无失真（矢量）
└── 一个文件适配所有分辨率

vs 多套 PNG：
┌──────────────────┬──────────────────┬──────────────────┐
│                  │ 多套 PNG         │ VectorDrawable   │
├──────────────────┼──────────────────┼──────────────────┤
│ 文件数量         │ 4套×N个图标      │ 1个文件           │
│ 文件大小         │ 每套 10~50KB     │ 1~5KB            │
│ 渲染质量         │ 固定分辨率        │ 任意尺寸无失真    │
│ 渲染性能         │ 直接显示         │ 需要 CPU 渲染     │
│ 适用场景         │ 复杂图片/照片     │ 图标/简单图形     │
└──────────────────┴──────────────────┴──────────────────┘

适合用 VectorDrawable 的场景：
├── App 图标（各种尺寸都需要）
├── 导航栏图标
├── 功能图标（分享/收藏/设置等）
└── 简单的装饰图形

不适合的场景：
├── 照片（复杂图片无法用路径描述）
├── 复杂渐变图片
└── 需要极高渲染性能的场景（VectorDrawable 有 CPU 开销）

// 问题：VectorDrawable 每次绘制都需要 CPU 渲染
// 解决：对于静态图标，缓存渲染结果

// 方法一：在代码中预渲染为 Bitmap（一次性开销）
fun vectorToBitmap(context: Context, vectorResId: Int, size: Int): Bitmap {
    val drawable = AppCompatResources.getDrawable(context, vectorResId)!!
    val bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(bitmap)
    drawable.setBounds(0, 0, size, size)
    drawable.draw(canvas)
    return bitmap
}

// 方法二：使用 Hardware Layer 缓存
imageView.setLayerType(View.LAYER_TYPE_HARDWARE, null)
// VectorDrawable 渲染结果缓存为 GPU 纹理
// 后续绘制直接用纹理，无需重新渲染

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
```
APK 资源系统：

resources.arsc 结构：
┌─────────────────────────────────────────────────────┐
│  StringPool（字符串池）                               │
│  ├── "res/layout/activity_main"                     │
│  ├── "res/drawable/ic_launcher"                     │
│  └── "res/layout/fragment_home"                     │
├─────────────────────────────────────────────────────┤
│  Package（包信息）                                    │
│  └── TypeSpec（类型规格）                             │
│      ├── layout → [activity_main, fragment_home...] │
│      ├── drawable → [ic_launcher, ic_back...]       │
│      └── string → [app_name, hello_world...]        │
└─────────────────────────────────────────────────────┘

资源访问流程：
R.layout.activity_main（0x7F040001）
  → resources.arsc 查找 0x7F040001
  → 找到路径 "res/layout/activity_main.xml"
  → 从 APK 中读取该文件

AndResGuard 做了什么：
① 将路径混淆：
   "res/layout/activity_main" → "r/l/a"
   "res/drawable/ic_launcher" → "r/d/a"
   resources.arsc 中的路径字符串变短

② 将文件重命名：
   res/layout/activity_main.xml → r/l/a.xml

③ 用 7zip 重新压缩 APK：
   比系统默认的 zip 压缩率更高

效果：
├── resources.arsc 变小（路径字符串变短）
├── APK 整体压缩率提升（7zip 更好）
└── 资源路径被混淆（防止逆向分析）
```

配置：  
```
// build.gradle 完整配置
apply plugin: 'AndResGuard'

andResGuard {
    // mapping 文件：保持混淆一致性（多版本间）
    // 首次构建：null（自动生成）
    // 后续构建：使用上次生成的 mapping
    mappingFile = file("andresguard-mapping.txt")

    use7zip = true
    useSign = true
    keepRoot = false  // 是否保留 res 根目录名

    // 白名单：这些资源不混淆
    // 原因：通过反射/字符串访问的资源必须保留原名
    whiteList = [
        // 桌面图标（系统通过固定路径访问）
        "R.mipmap.ic_launcher",
        "R.mipmap.ic_launcher_round",

        // 通知图标（系统访问）
        "R.drawable.ic_notification",

        // 通过 Resources.getIdentifier() 访问的资源
        // 例：resources.getIdentifier("ic_" + type, "drawable", packageName)
        "R.drawable.ic_*",  // 通配符

        // 第三方 SDK 要求保留的资源
        "R.string.wechat_*",
        "R.layout.umeng_*",
    ]

    // 压缩配置
    compressFilePattern = [
        "*.png",
        "*.jpg",
        "*.jpeg",
        "*.gif",
        "*.webp",
        "*.xml",    // 布局文件也压缩
    ]

    // 7zip 配置
    sevenzip {
        artifact = 'com.tencent.mm:SevenZip:1.2.20'
        // 或指定本地 7zip 路径
        // path = "/usr/local/bin/7za"
    }

    // 最终 APK 输出路径
    finalApkBackupPath = "${project.rootDir}/final_apk"
}
```
### 重复资源去重
来源：  
```
重复资源来源：
├── 不同分辨率目录下内容相同的图片
│   drawable-hdpi/bg.png 和 drawable-xxhdpi/bg.png 内容一样
├── 第三方库带来的重复资源
│   多个库都包含相同的默认图标
├── 开发者手动复制的资源
│   ic_back_arrow.png 和 ic_arrow_left.png 内容完全相同
└── 历史遗留的重复文件
```

去重：  
```
// 完整的重复资源检测工具
import java.io.File
import java.security.MessageDigest

object DuplicateResourceDetector {

    data class ResourceFile(
        val file: File,
        val md5: String,
        val sizeBytes: Long,
        val resType: String,   // layout/drawable/string
        val qualifier: String  // hdpi/xxhdpi/zh/en
    )

    data class DuplicateGroup(
        val md5: String,
        val files: List<ResourceFile>,
        val wastedBytes: Long  // 重复浪费的字节数
    )

    fun detectDuplicates(resDir: File): List<DuplicateGroup> {
        // 1. 收集所有资源文件
        val allFiles = resDir.walkTopDown()
            .filter { it.isFile }
            .filter { it.extension in listOf("png", "jpg", "webp", "xml", "9.png") }
            .map { file ->
                val md5 = calculateMd5(file)
                val parts = file.parentFile.name.split("-")
                ResourceFile(
                    file = file,
                    md5 = md5,
                    sizeBytes = file.length(),
                    resType = parts[0],
                    qualifier = parts.drop(1).joinToString("-")
                )
            }
            .toList()

        // 2. 按 MD5 分组
        val groups = allFiles.groupBy { it.md5 }

        // 3. 找出有重复的组
        return groups
            .filter { (_, files) -> files.size > 1 }
            .map { (md5, files) ->
                DuplicateGroup(
                    md5 = md5,
                    files = files,
                    wastedBytes = files.sumOf { it.sizeBytes } - files[0].sizeBytes
                )
            }
            .sortedByDescending { it.wastedBytes }
    }

    fun generateReport(duplicates: List<DuplicateGroup>): String {
        val totalWasted = duplicates.sumOf { it.wastedBytes }
        val sb = StringBuilder()

        sb.appendLine("=== 重复资源报告 ===")
        sb.appendLine("共发现 ${duplicates.size} 组重复资源")
        sb.appendLine("总浪费空间：${totalWasted / 1024}KB")
        sb.appendLine()

        duplicates.forEach { group ->
            sb.appendLine("MD5: ${group.md5} (浪费 ${group.wastedBytes / 1024}KB)")
            group.files.forEach { file ->
                sb.appendLine("  ${file.file.relativeTo(file.file.parentFile.parentFile)}")
            }
            sb.appendLine()
        }

        return sb.toString()
    }

    fun removeDuplicates(
        duplicates: List<DuplicateGroup>,
        strategy: DeduplicateStrategy = DeduplicateStrategy.KEEP_HIGHEST_DENSITY
    ) {
        duplicates.forEach { group ->
            val filesToKeep = when (strategy) {
                DeduplicateStrategy.KEEP_HIGHEST_DENSITY -> {
                    // 保留分辨率最高的版本
                    listOf(group.files.maxByOrNull { densityScore(it.qualifier) }!!)
                }
                DeduplicateStrategy.KEEP_NODPI -> {
                    // 如果有 nodpi 版本，只保留它
                    group.files.filter { it.qualifier == "nodpi" }
                        .ifEmpty {
                            listOf(group.files.maxByOrNull { densityScore(it.qualifier) }!!)
                        }
                }
            }

            // 删除不保留的文件
            group.files
                .filter { it !in filesToKeep }
                .forEach { resourceFile ->
                    println("删除重复资源: ${resourceFile.file.path}")
                    resourceFile.file.delete()
                }
        }
    }

    private fun densityScore(qualifier: String): Int = when {
        qualifier.contains("xxxhdpi") -> 4
        qualifier.contains("xxhdpi") -> 3
        qualifier.contains("xhdpi") -> 2
        qualifier.contains("hdpi") -> 1
        qualifier.contains("nodpi") -> 5  // nodpi 优先级最高（适配所有密度）
        qualifier.isEmpty() -> 0           // 默认目录
        else -> -1
    }

    private fun calculateMd5(file: File): String {
        val md = MessageDigest.getInstance("MD5")
        file.inputStream().use { input ->
            val buffer = ByteArray(8192)
            var read: Int
            while (input.read(buffer).also { read = it } != -1) {
                md.update(buffer, 0, read)
            }
        }
        return md.digest().joinToString("") { "%02x".format(it) }
    }

    enum class DeduplicateStrategy {
        KEEP_HIGHEST_DENSITY,
        KEEP_NODPI
    }
}

// 使用
fun main() {
    val resDir = File("src/main/res")
    val duplicates = DuplicateResourceDetector.detectDuplicates(resDir)
    println(DuplicateResourceDetector.generateReport(duplicates))

    // 确认后执行去重
    DuplicateResourceDetector.removeDuplicates(duplicates)
}
```
### 语言/分辨率裁剪
语言：  
```
// build.gradle
android {
    defaultConfig {
        // 只保留中文和英文
        // 其他语言（法语/德语/日语等）的 strings.xml 全部删除
        resConfigs "zh", "zh-rCN", "zh-rTW", "en"
    }
}

// 效果分析：
// 第三方库（如 Google Play Services）包含 80+ 种语言的字符串
// 裁剪后只保留 zh 和 en
// 通常可以减少 500KB~2MB

// 注意：
// ① 只影响字符串资源（strings.xml）
// ② 不影响图片/布局资源
// ③ 如果 App 支持多语言，需要保留对应语言
```

分辨率：  
```
android {
    defaultConfig {
        // 只保留高密度资源
        // 现代手机基本都是 xxhdpi（480dpi）或 xxxhdpi（640dpi）
        resConfigs "xxhdpi", "xxxhdpi"

        // 系统会自动缩放：
        // 如果设备是 xhdpi，但没有 xhdpi 资源
        // 系统会用 xxhdpi 资源缩小显示
        // 视觉效果稍有影响，但可以接受
    }
}

// 更激进：只保留 xxhdpi
// resConfigs "xxhdpi"
// 减少约 25% 的图片资源大小

// 注意：
// ① ldpi/mdpi 设备（老设备）会用 xxhdpi 资源缩小，可能有性能影响
// ② 如果 App 有大量低端机用户，谨慎使用
// ③ 矢量图不受影响（VectorDrawable 适配所有密度）
```
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
问题：  
```
多个小 So 文件的开销

每个 So 文件的固定开销：
├── ELF 文件头：约 64 字节
├── 段表（Section Header Table）：每个段 64 字节
├── 动态链接信息（.dynamic）：约 1KB
├── 符号表（.dynsym）：每个导出符号约 24 字节
└── 字符串表（.dynstr）：符号名字符串

如果有 10 个小 So，每个 100KB：
├── 10 × 固定开销 ≈ 10 × 5KB = 50KB 额外开销
├── 10 次 dlopen 调用（加载时间增加）
└── 10 个独立的符号表（内存占用增加）

合并后：
├── 1 次 dlopen
├── 1 个符号表
└── 固定开销只有 1 份
```

实现：  
```
# CMakeLists.txt：将多个模块合并为一个 So

cmake_minimum_required(VERSION 3.18.1)
project("myapp")

# 原来：多个独立的 So
# add_library(module_network SHARED network.cpp)
# add_library(module_crypto SHARED crypto.cpp)
# add_library(module_image SHARED image.cpp)
# add_library(module_audio SHARED audio.cpp)

# 优化：合并为一个 So
add_library(
    myapp_native  # 合并后的 So 名称
    SHARED
    # 所有模块的源文件
    network/network.cpp
    network/http_client.cpp
    crypto/aes.cpp
    crypto/rsa.cpp
    image/decoder.cpp
    image/encoder.cpp
    audio/player.cpp
    audio/recorder.cpp
)

target_link_libraries(
    myapp_native
    android
    log
    z
    OpenSLES
)

# 只导出必要的 JNI 函数（隐藏内部符号）
# 减少符号表大小
set_target_properties(myapp_native PROPERTIES
    CXX_VISIBILITY_PRESET hidden      # 默认隐藏所有符号
    VISIBILITY_INLINES_HIDDEN ON
)

# 只有 JNI 函数需要导出（用 __attribute__((visibility("default"))) 标记）

# CMakeLists.txt：将多个模块合并为一个 So

cmake_minimum_required(VERSION 3.18.1)
project("myapp")

# 原来：多个独立的 So
# add_library(module_network SHARED network.cpp)
# add_library(module_crypto SHARED crypto.cpp)
# add_library(module_image SHARED image.cpp)
# add_library(module_audio SHARED audio.cpp)

# 优化：合并为一个 So
add_library(
    myapp_native  # 合并后的 So 名称
    SHARED
    # 所有模块的源文件
    network/network.cpp
    network/http_client.cpp
    crypto/aes.cpp
    crypto/rsa.cpp
    image/decoder.cpp
    image/encoder.cpp
    audio/player.cpp
    audio/recorder.cpp
)

target_link_libraries(
    myapp_native
    android
    log
    z
    OpenSLES
)

# 只导出必要的 JNI 函数（隐藏内部符号）
# 减少符号表大小
set_target_properties(myapp_native PROPERTIES
    CXX_VISIBILITY_PRESET hidden      # 默认隐藏所有符号
    VISIBILITY_INLINES_HIDDEN ON
)

# 只有 JNI 函数需要导出（用 __attribute__((visibility("default"))) 标记）
```
## 架构优化
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
// gradle.properties 完整优化配置

# 开启 R8 完整模式（更激进的优化）
android.enableR8.fullMode=true

# 开启 D8 desugaring（语法糖脱糖，支持低版本）
android.enableD8.desugaring=true

# 并行编译（加快编译速度）
org.gradle.parallel=true
org.gradle.workers.max=4

# 增量编译
org.gradle.configureondemand=true

# JVM 堆大小（避免编译 OOM）
org.gradle.jvmargs=-Xmx4g -XX:MaxPermSize=512m \
    -XX:+HeapDumpOnOutOfMemoryError \
    -Dfile.encoding=UTF-8

// build.gradle 编译优化
android {
    compileOptions {
        // 开启核心库脱糖（支持 Java 8+ API 在低版本系统上运行）
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }

    kotlinOptions {
        jvmTarget = '11'
        // 开启 Kotlin 编译器优化
        freeCompilerArgs += [
            '-Xopt-in=kotlin.RequiresOptIn',
            // 开启内联类优化
            '-Xinline-classes',
            // 开启协程优化
            '-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi'
        ]
    }

    buildFeatures {
        // 关闭不需要的构建特性（减少生成代码）
        buildConfig = false  // 如果不需要 BuildConfig
        aidl = false         // 如果不用 AIDL
        renderScript = false // 如果不用 RenderScript
        resValues = false    // 如果不用 resValue
        shaders = false      // 如果不用 shader
    }
}

dependencies {
    // 核心库脱糖依赖
    coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:2.0.4'
}
```
### Baseline Profile
问题：  
```
Baseline Profile 解决的问题：

Android 运行 Java/Kotlin 代码的方式：
├── 解释执行（Interpreter）：最慢，逐条解释字节码
├── JIT 编译（Just-In-Time）：运行时编译热点代码
│   └── 首次运行慢，多次运行后变快
└── AOT 编译（Ahead-Of-Time）：安装时编译为机器码
    └── 运行最快，但编译耗时长

问题：
App 安装后，ART 不知道哪些代码是热点
→ 首次启动：全部解释执行（慢！）
→ 运行一段时间后：JIT 编译热点代码（变快）
→ 用户体验：首次启动卡顿，后续流畅

Baseline Profile 的作用：
开发者提前告诉 ART：这些方法是热点代码
→ 安装时提前 AOT 编译这些方法
→ 首次启动就很快！

效果：
├── 启动速度提升 30%~40%
├── 首帧渲染时间减少
└── 减少 JIT 编译的 CPU 开销（省电）
```

生成：  
```
// 1. 添加依赖
// build.gradle
dependencies {
    implementation 'androidx.profileinstaller:profileinstaller:1.3.1'
}

// macrobenchmark 模块的 build.gradle
dependencies {
    implementation 'androidx.benchmark:benchmark-macro-junit4:1.2.3'
}

// 2. 编写 Baseline Profile 生成器
// macrobenchmark/src/androidTest/java/BaselineProfileGenerator.kt

@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() {
        rule.collect(
            packageName = "com.xxx.app",
            // 定义用户关键路径（这些路径的代码会被收集）
            profileBlock = {
                // 启动 App
                pressHome()
                startActivityAndWait()

                // 模拟用户操作：首页加载
                device.waitForIdle()

                // 模拟滑动列表
                val list = device.findObject(By.res("com.xxx.app:id/recycler_view"))
                list.fling(Direction.DOWN)
                device.waitForIdle()

                // 模拟点击进入详情页
                val firstItem = device.findObject(By.res("com.xxx.app:id/item_title"))
                firstItem?.click()
                device.waitForIdle()

                // 模拟返回
                device.pressBack()
                device.waitForIdle()
            }
        )
    }
}

// 3. 运行生成器
// ./gradlew :macrobenchmark:generateBaselineProfile
// 生成文件：app/src/main/baseline-prof.txt

// baseline-prof.txt 内容示例：
// HSPLcom/xxx/app/MainActivity;->onCreate(Landroid/os/Bundle;)V
// HSPLcom/xxx/app/HomeFragment;->onViewCreated(Landroid/view/View;Landroid/os/Bundle;)V
// HSPLcom/xxx/app/ArticleAdapter;->onBindViewHolder(...)V
// ...
// H = Hot（热点方法，AOT 编译）
// S = Startup（启动时调用）
// P = Post-startup（启动后调用）
// L = Library（库方法）
```
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
