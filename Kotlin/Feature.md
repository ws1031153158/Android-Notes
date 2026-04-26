# 扩展函数
## 内联
inline 关键字用于声明内联函数，它的作用是告诉编译器在调用处直接将函数的代码插入，而不是通过常规的函数调用来执行，执行时，程序会按顺序执行内联函数体的代码和 Lambda 表达式的代码。   
可以减少函数调用带来的性能开销，并且可以避免创建匿名对象和类（如 lambda 表达式，实际上是创建了一个内部类，调用 invoke 方法）。内联函数通常用于高阶函数、Lambda 表达式等场景，可以提高程序的执行效率，但如果代码过多，迁移代码成本也会很大。   
let：一般与 .？一起使用，代码块内部是对调用对象的各个方法的访问，it 替代该对象，底层为 inLine 的扩展函数 + lambda 结构，返回值为函数块的最后一行或指定 return 表达式  
with：和 let 类似，内部 this 指代调用对象（可忽略），该结构返回最后一行或指定的 return 表达式。传入两个参数，调用对象和一个 lambda 体，若最后一个参数是函数则可以放在括号外，常用于调用同一个对象的多个方法。（tip：with 不是扩展函数）  
apply：和 run 类似，返回值不同，返回传入对象本身（返回 this），一般用于对象初始化，对属性复制，或动态 inflate 一个 view 给 view 绑定数据  
also：和 let 类似，it 指代，返回当前的这个对象（返回 this），一般可用于多个扩展函数链式调用  
run：可看做 let 和 with 的结合体，只接受一个 lambda 函数作为参数。可以判空，且也是由 this 指代（可省略），以闭包形式返回最后一行代码的值。
### 函数调用
① 参数压栈（或放入寄存器）  
   push arg1, arg2, arg3...  

② 保存返回地址  
   push return_address  

③ 跳转到函数地址  
   jmp function_address  
  
④ 函数内：创建栈帧  
   push rbp  
   mov rbp, rsp  
   sub rsp, local_var_size  

⑤ 执行函数体  

⑥ 销毁栈帧，恢复寄存器  
   mov rsp, rbp  
   pop rbp  

⑦ 返回  
   ret → 弹出返回地址，跳转回去  

1.栈操作（push/pop）  
2.跳转（破坏CPU流水线预测）  
3.栈帧创建/销毁  
4.对于虚方法：还有虚表查找（vtable lookup）  

底层实现：  
```
// 内联前
fun add(a: Int, b: Int): Int = a + b

fun calculate(): Int {
    val x = add(1, 2)  // 函数调用
    val y = add(3, 4)  // 函数调用
    return x + y
}

// 字节码（未内联）：
// ICONST_1
// ICONST_2
// INVOKESTATIC add(II)I  ← 函数调用指令
// ISTORE x
// ICONST_3
// ICONST_4
// INVOKESTATIC add(II)I  ← 函数调用指令
// ISTORE y
// ILOAD x
// ILOAD y
// IADD
// IRETURN

// R8 内联后（字节码层面直接替换）：
// ICONST_1
// ICONST_2
// IADD              ← 直接计算，消除函数调用
// ISTORE x
// ICONST_3
// ICONST_4
// IADD              ← 直接计算
// ISTORE y
// ILOAD x
// ILOAD y
// IADD
// IRETURN

// 更进一步：常量折叠
// ICONST_3          ← 1+2=3，编译期算好
// ISTORE x
// ICONST_7          ← 3+4=7，编译期算好
// ISTORE y
// ICONST_10         ← 3+7=10，编译期算好
// IRETURN
```
### noinline & crossinline
1.函数类型参数本质是一个对象，而 inline 函数会将这个对象的代码展开铺平，消除其对象属性。   
2.noinline 用于标记 inline 函数中的函数类型参数，被标记的函数类型参数不会被内联。   
3.crossinline 是交叉内联，也是作用于内联函数的函数类型参数上，用途是强化函数类型参数的内联优化，使能被间接调用，并且被 crossinline 标记的 Lambda 中不能使用 return。
### 风险
1.代码膨胀  
```
// 被内联的函数被调用100次
// → 函数体被复制100份
// → Dex 文件变大
// → 方法数可能增加（内联后的代码可能超过方法大小限制）

// R8 的内联阈值控制：
// 方法体 > N 条字节码指令 → 不内联
// 但 Full Mode 下阈值更宽松 → 更多大方法被内联 → 包体积可能反而增大

// 示例：
fun formatDate(timestamp: Long): String {
    val sdf = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
    return sdf.format(Date(timestamp))
}

// 被调用50次，Full Mode 可能全部内联
// 50份 SimpleDateFormat 创建代码 → 代码膨胀
```

2.栈溢出  
```
// 内联消除了函数调用栈帧
// 但如果内联层级过深，反而可能影响栈深度计算

// 更常见的问题：
// 内联后的方法体超过 Dex 方法大小限制（64KB字节码）
// → 编译失败！

// R8 Full Mode 遇到这种情况会：
// ① 回退：不内联这个方法
// ② 或者报错（极少数情况）
```

3.调试困难  
```
// 内联后，堆栈信息丢失

// 内联前的崩溃堆栈：
// at com.xxx.Utils.validateInput(Utils.kt:42)
// at com.xxx.PaymentProcessor.process(PaymentProcessor.kt:18)

// 内联后（validateInput 被内联进 process）：
// at com.xxx.PaymentProcessor.process(PaymentProcessor.kt:18)
// validateInput 消失了！不知道是哪行出的问题

// 解决：保留行号信息
// -keepattributes SourceFile,LineNumberTable
// 即使内联，R8 也会尽量保留行号映射
```

4.接口内联破坏多态（Full Mode 特有）  
```
// 场景：接口只有一个实现，Full Mode 内联接口调用

interface PaymentGateway {
    fun charge(amount: Double): Boolean
}

class StripeGateway : PaymentGateway {
    override fun charge(amount: Double): Boolean {
        return stripeApi.charge(amount)
    }
}

// 使用
val gateway: PaymentGateway = StripeGateway()
gateway.charge(100.0)

// Full Mode 分析：PaymentGateway 只有一个实现 StripeGateway
// → 内联接口调用，直接调用 StripeGateway.charge()
// → 消除虚方法查找

// 风险：如果运行时通过反射或动态加载了另一个实现
// class AlipayGateway : PaymentGateway  ← 插件化动态加载
// → 内联后的代码永远调用 Stripe，不会调用 Alipay！
// → 功能错误，且极难排查

// 解决：
-keep interface com.xxx.PaymentGateway { *; }
-keep class * implements com.xxx.PaymentGateway { *; }
```

5.Kotlin inline 函数与 R8 内联的叠加   
```
// Kotlin 的 inline 关键字：编译期内联（开发者控制）
inline fun <T> measureTime(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时: ${System.currentTimeMillis() - start}ms")
    return result
}

// R8 再次内联：对已经内联的代码再次优化
// 双重内联可能导致：
// ① 代码膨胀更严重
// ② 局部变量名冲突（R8 会处理，但偶有bug）
// ③ 异常处理逻辑错乱（try-catch 范围变化）

// 建议：对复杂的 inline 函数，添加 keep 规则防止 R8 再次内联
```
### 编译时常量
变量的值固定不变，在编译时我们就可拿到这个变量的值。具体到代码中，就是变量使用 final 修饰，类型只能是字符串或基本类型，在声明时就赋值；在编译时，会直接拿这个变量值去替换变量名，是一种内联优化。
## 自定义
在类的外面定义扩展函数，可在类内部/外部调用，无需修改类本身即可实现功能的扩展，不会作为成员添加到类中，可在接口中定义扩展，可定义成员扩展（不支持引用，且多个隐式访问可读性差）  
Class.function，本质是静态函数，编译器将扩展函数转化为静态函数的调用。  
可以扩展原生类，在 java 中调用会转化为静态函数的调用
## Tips
1.只有作为内联函数的参数的 Lambda 表达式中可以直接使用 return  
2.Lambda 表达式中的 return 结束的不是内联函数自身，而是内联函数的调用函数  
# When
等同于 switch，且类型推导让我们无需指定返回值类型也不需要写 break，可以匹配任一对象，没有限制
# init
## init block
可以为变量赋值，执行一些检查相关的代码，init 初始化块在创建类的实例对象时执行 
## 顺序
1.主构造函数中属性赋值  
2.类中的属性赋值
3.init 初始化块中的代码执行  
4.次构造函数中的代码执行  
# By
一种委托机制：  
类委托：类的具体实现委托给另一个类（by 之后的类）完成，通常用于指定一个实现类，去实现接口中的公共方法，类的构造函数要传入这个要实现类对象，一般为了解耦和复用，将一个接口交给一个对象（类）实现。  
属性委托：属性值不是在类中直接进行定义，而是将其托付给一个代理类，属性的 get/set 方法被委托给 by 之后的对象的 getValue/setValue 方法（必须提供），特殊的有 by lazy，懒加载，传入一个加载模式，一般为 LazyThreadSafetyMode.SYNCHRONIZED，保证线程安全，只会在第一次使用时初始化一次。  
lateinit：延迟加载，只是编译器不会对初始化做检测，而 lazy 是一个内联高阶函数，传入自身来做一些初始化判断
# Constructor
支持类命名时指定构造函数，子构造函数需要依托主构造函数，会直接或者间接调用主构造函数，是继承的，可以通过 super 调用主构造函数  
主构造函数：在类的标题中，不手动表示系统会生成默认一个无参的  
子构造函数：在类内部，用 constructor 表示，没有注解和可见性修饰符时可以省略关键字，但为 private 时不可省略  
# data class
数据类（只保存数据），自动生成：
equals；hashCode；toString；copy；componentN，kotlin特有解构声明，通过该方法可智能获取到传入的参数；属性的 get()/set()（用var修饰），@JvmField 可忽略对属性的方法生成；constructor，@JvmOverloads 自动生成多个构造函数的重载
# Null Safety
?（可空，不崩溃）；!!（非空）等符号，保证空指针安全，可以智能识别对象类型
# CallBack
若要重写的只有一个方法，则 lambda 表达式可以直接写方法体，此外若不使用传入的参数，则可以直接省略
# sealed class
密封类，枚举类的扩展，子类可以有多个对象，可以持有其他引用数据，可以自定义方法
# Componion
object：修饰的类为静态类，方法和变量均为静态，可声明静态内部类  
componion object：伴生对象，类中只存在一个，类似 java 静态方法、静态成员，调用的也必须是静态成员
# Coroutine
协程不是所谓“轻量级线程”、“在子线程执行”、“可以并行”：  
是可挂起、可恢复 的计算单元，本质是对 回调（Callback）的语法糖封装，运行在线程上，但不绑定特定线程。  
## 实现
通过编译技术，不依赖任何 os/vm，编译时 suspend 函数传入 Continuation 接口，持有 context 和一个 resumeWith 作为参数，其中 resumeWith 会调用 invokeSuspend 在协程启动或挂起时恢复，协程运行完成回调。  
这些代表之后要执行的方法，编译器将协程体编译成匿名内部类，挂起函数调用位置对应挂起点，。  
类中实现 create 来创建协程体类实例，使用状态机，维护一个 label，控制 invokeSuspend 方法执行不同条件分支，挂起函数调用分布在不同分支，挂起函数传参为 this，即协程体自身。    
## 优点
线程会占用内存，且切换开销较大，线程的切换是在内核态下完成的，所以用户态中发起线程的切换会中断，进入内核态，所以需要内核的支持，用户级的线程切换时，才需要内核的支持，用户态和内核态其实就是CPU指令集权限级别的区别，用户态相当于应用程序，内核态相当于操作系统，多个协程可以共用一个线程，切换位于用户态。  
## launch
是一种通过挂起机制实现可中断的方法，通过栈实现层级调用
.launch(Dispatchers.IO/Main/Default/Unconfined）{  
}    
//基于线程池实现，四种调度器，父类为	CoroutineScheduler，通过 dispatch 分配运行的线程  
//launch 并不是一个顶层函数，它必须在一个对象中使用，是	CoroutineScope 接口（提供contexat，通过扩展函数管理协程）的扩展函数  
此结构体实现一个协程，通过 withContext 切换运行线程，并在完成后返回当前线程，该方法为一个 suspend 方法，需要在协程或另一个 suspend 方法中调用。  
## withContext
withContext 是挂起函数，做两件事：  
1. 切换协程运行的调度器（线程，但目标调度器与当前相同时，不切换线程）  
2. 执行完后自动切回原调度器  
3. 有返回值（这是和launch的核心区别）

执行流程：  
1.主线程执行到 withContext(Dispatchers.IO)  
2.当前协程挂起（保存Continuation状态机）  
3.主线程释放，去处理其他消息  
4.block 被提交到 IO 线程池执行  
5.IO线程执行完毕  
6.将"恢复"任务投递回主线程队列  
7.主线程从队列取出"恢复"任务  
8.继续执行 withContext 之后的代码，拿到返回值    

一个IO线程可以处理多个协程，不是每个 withContext/launch 都新建线程。  
线程池复用机制：  
```
// Dispatchers.IO 内部是一个线程池
// 默认最大线程数：max(64, CPU核心数)

// 场景：同时启动10个协程
repeat(10) { index ->
    launch(Dispatchers.IO) {
        println("协程$index 开始: ${Thread.currentThread().name}")
        delay(1000) // 挂起，释放线程
        println("协程$index 结束: ${Thread.currentThread().name}")
    }
}

// 输出可能是：
// 协程0 开始: worker-1
// 协程1 开始: worker-2
// 协程2 开始: worker-3
// ... 最多同时用到 min(10, 64) 个线程
// delay期间，worker-1/2/3 被释放回线程池
// 1000ms后恢复时，可能用的是 worker-5/6/7（任意空闲线程）
// 协程0 结束: worker-5  ← 和开始时不同的线程！
```

嵌套：  
```
suspend fun nestedExample() {
    // 假设当前在主线程

    withContext(Dispatchers.IO) {           // ① 切到IO线程池，取一个worker
        println(Thread.currentThread().name) // worker-1

        withContext(Dispatchers.Main) {     // ② 切回主线程
            println(Thread.currentThread().name) // main
            // worker-1 被释放回线程池！

            withContext(Dispatchers.IO) {   // ③ 再切到IO线程池
                println(Thread.currentThread().name) // worker-1 或 worker-2
                // 不一定是同一个worker！
            }
            // 切回主线程
        }
        // 切回IO线程（可能是新的worker）
        println(Thread.currentThread().name) // worker-2（可能变了）
    }
    // 切回主线程
}

// IO(IO) → 只用一个IO线程（同调度器优化，不切换）
// IO(MAIN(IO)) → 用了两个不同的IO线程（中间经过主线程，原IO线程被释放）

// 自定义线程池大小
val limitedIO = Dispatchers.IO.limitedParallelism(4) // 最多4个线程
launch(limitedIO) { ... }
```
## async
async 也返回一个协程，不同点在于实现了 Deferred 接口，是一种延迟执行的方法，需要调用 .await（也是一个 suspend 方法）获取结果，一般用于同时开启多个任务，获取到所有返回结果再继续执行后面的任务。  
默认启动，可以通过 start 手动开启，懒加载，在 launch 中传入CoroutineStart.LAZY，可以在 job.invokeOnCompletion 回调之后执行一些操作，正常和异常执行完毕都会走。  

```
suspend fun compare() {

    // ① withContext：挂起当前协程，等结果，串行
    val result = withContext(Dispatchers.IO) {
        "IO结果"  // 必须等这里完成
    }
    println(result) // 一定能拿到值，顺序执行

    // ② launch：新建子协程，不等待，无返回值，并行
    val job = launch(Dispatchers.IO) {
        delay(1000)
        println("launch完成") // 不知道何时执行
    }
    println("launch后立即执行") // 不等launch

    // ③ async：新建子协程，不等待，有返回值(Deferred)，并行
    val deferred = async(Dispatchers.IO) {
        "async结果"
    }
    println("async后立即执行") // 不等async
    val value = deferred.await() // 这里才挂起等待
    println(value)
}
```
## 启动模式
DEFAULT：立即开始调度  
LAZY：start/join/await 开始调度  
ATOMMIC：立即开始调度，第一个挂起点前无法取消（涉及cancel才有意义）  
UNDISPATCHED：立即开始调度，直到第一个挂起点（之后由调度器（Dispatcher基于线程池实现）处理）  
### start、join、await
start：  
非挂起，返回 Boolean 值，使用对象 Job / Deferred   
启动一个懒加载协程，返回值：true=成功启动，false=已经启动过或已完成    
用于预热协程，条件触发  
```
// 默认模式：launch 创建后立即启动，不需要 start()
val job1 = launch { println("立即启动") } // 已自动启动

// 懒加载模式：创建后不启动，需要手动 start() 或 join()/await()
val job2 = launch(start = CoroutineStart.LAZY) {
    println("懒加载，等待start()")
}

println("job2还没启动")
job2.start() // 现在才启动，非挂起，立即返回
println("start()返回，但job2可能还没执行完")

// 应用场景：
// 1. 预创建协程，但延迟执行
// 2. 根据条件决定是否启动
if (userIsLoggedIn) {
    job2.start()
}
```

join：  
挂起 ，返回 Unit，使用对象 Job  
挂起当前协程，等待目标Job完成，不关心返回值，只关心"完不完成"，不会抛出子协程的异常（异常由父协程的异常处理器处理）    
用于有序的初始化步骤  
```
suspend fun joinExample() {
    val job = launch(Dispatchers.IO) {
        delay(1000)
        println("子任务完成")
        // 没有返回值
    }

    println("等待子任务...")
    job.join() // 挂起，等job完成
    println("子任务已完成，继续执行") // join后一定执行

    // 应用场景1：顺序执行多个无返回值任务
    val step1 = launch { initDatabase() }
    step1.join() // 等数据库初始化完
    val step2 = launch { loadUserData() } // 再加载数据
    step2.join()

    // 应用场景2：等待一组任务全部完成
    val jobs = List(10) { index ->
        launch(Dispatchers.IO) {
            processItem(index)
        }
    }
    jobs.forEach { it.join() } // 等所有任务完成
    println("10个任务全部完成")
}
```

await：  
挂起 ，返回 T（结果），使用对象 Deferred<T>  
挂起当前协程，等待Deferred完成并返回结果，Deferred = 有返回值的Job（async创建），会抛出异常，可以捕获    
用于并行请求合并结果  
```
suspend fun awaitExample() {
    // 单个await
    val deferred = async(Dispatchers.IO) {
        delay(1000)
        "计算结果" // 有返回值
    }
    val result = deferred.await() // 挂起等待，拿到结果
    println(result) // "计算结果"

    // 并行多个任务，等所有结果（最常用场景）
    val userDeferred = async(Dispatchers.IO) {
        api.getUser() // 并行请求1
    }
    val orderDeferred = async(Dispatchers.IO) {
        api.getOrders() // 并行请求2
    }
    val bannerDeferred = async(Dispatchers.IO) {
        api.getBanner() // 并行请求3
    }

    // 三个请求并行执行
    // 总耗时 = max(三个请求耗时) 而非 sum
    val user = userDeferred.await()
    val orders = orderDeferred.await()
    val banner = bannerDeferred.await()

    updateUI(user, orders, banner)
}

// awaitAll 简化写法
suspend fun awaitAllExample() {
    val results = awaitAll(
        async { api.getUser() },
        async { api.getOrders() },
        async { api.getBanner() }
    )
    // results[0] = user, results[1] = orders, results[2] = banner
}
```
## Dispatcher
定义了 Coroutine 执行的线程。CoroutineDispatcher 可以限定 Coroutine 在某一个线程执行、也可以分配到一个线程池来执行、也可以不限制其执行的线程，CoroutineDispatcher 是一个抽象类，所有 dispatcher 都应该继承这个类来实现对应的功能，Dispatchers 本质是一个线程池 + 任务队列。  
Dispatchers.Default：    
默认的调度器，适合处理后台计算，是一个CPU密集型任务调度器，创建 Coroutine 的时候没有指定 dispatcher，则一般默认使用这个作为默认值，defaultDispatcher 使用一个共享的后台线程池来运行里面的任务。注意它和IO共享线程池，只不过限制了最大并发数不同。    
Dispatchers.IO：    
顾名思义这是用来执行阻塞 IO 操作的，是和Default共用一个共享的线程池来执行里面的任务。根据同时运行的任务数量，在需要的时候会创建额外的线程，当任务执行完毕后会释放不需要的线程。    
Dispatchers.Unconfined：    
由于Dispatchers.Unconfined未定义线程池，所以执行的时候默认在启动线程。遇到第一个挂起点，之后由调用resume的线程决定恢复协程的线程。    
Dispatchers.Main：    
指定执行的线程是主线程，在Android上就是UI线程，由于子Coroutine 会继承父Coroutine 的 context，所以为了方便使用，我们一般会在 父Coroutine 上设定一个 Dispatcher，然后所有 子Coroutine 自动使用这个 Dispatcher。    
## Suspend
简单来说：  
1.suspend 函数 = 状态机  
2.每个挂起点 = 一个状态  
3.挂起 = 保存当前状态，释放线程  
4.恢复 = 从保存的状态继续执行，可以在任意线程    
线程执行到 suspend 方法时，不再执行剩余的协程代码，跳出代码块，invokeSusped 返回	COROUTINE_SUSPENDED 时执行协程的 return，去执行别的任务，而协程 post 到指定线程继续执行，完成后会切回原线程，即协程 post 一个 Runnable，让剩余代码回到原线程执行：withcontext 调用	startCoroutineCancellable，传入 coroutine 回调，DispatchedCoroutine，持有外部要恢复的协程，协程 return 会走 resumeWith，重写 afterCompletion，判断是否挂起，通过 resumeCancellableWith 恢复外部协程，再次触发 invoke。  
```
// 你写的代码：
suspend fun fetchUser(): User {
    delay(1000)
    return User("张三")
}

// 编译器实际生成的代码（CPS变换）：
// suspend 函数被编译为带 Continuation 参数的普通函数
fun fetchUser(continuation: Continuation<User>): Any {
    // Continuation 就是"回调"，将传入的 continuation 包装为状态机
    // 第一次调用时，continuation 是外部传入的
    // 恢复时，continuation 是上次保存的，包含 恢复执行所需的上下文 + 结果处理逻辑
    val sm = continuation as? FetchUserStateMachine
        ?: FetchUserStateMachine(continuation)
    
    when (sm.label) {
        0 -> {
            sm.label = 1
            // 注册延迟回调，1000ms后恢复，传入状态机作为 continuation
            val  result = delay(1000, continuation) // 挂起点

            // delay 返回 COROUTINE_SUSPENDED
            if (result == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED // 告诉调用方：挂起了
            }
        }
        1 -> {
            // 检查是否有异常
            val result = sm.result
            if (result is Failure) throw result.exception

            // delay 完成，继续执行 return User("张三")
            val user = User("张三")
            
            // 通知外部调用者：我完成了，结果是 user
            sm.completion.resume(user)
            return user
        }
    }
}

// 状态机类（保存协程的所有状态）
class FetchUserStateMachine(
    val completion: Continuation<User> // 外部的"回调"
) : Continuation<Any?> {
    
    var label: Int = 0        // 当前状态
    var result: Any? = null   // 上一步的结果
    
    // 当 delay 完成时，框架调用这个方法恢复协程
    override fun resumeWith(result: Result<Any?>) {
        this.result = result.getOrNull()
        
        // 重新调用 fetchUser，但这次 label=1
        // 相当于从状态1继续执行
        val outcome = fetchUser(this)
        
        if (outcome != COROUTINE_SUSPENDED) {
            // 如果没有再次挂起，说明函数执行完了
            completion.resumeWith(Result.success(outcome as User))
        }
    }
    
    override val context: CoroutineContext
        get() = completion.context
```

delay底层实现：  
```
// delay 的简化实现
suspend fun delay(timeMillis: Long): Unit {
    if (timeMillis <= 0) return
    
    // suspendCancellableCoroutine 是最底层的挂起原语
    return suspendCancellableCoroutine { continuation ->
        // continuation 就是上面的 FetchUserStateMachine
        
        // 根据当前调度器，注册延迟回调
        val dispatcher = continuation.context[ContinuationInterceptor]
        
        when (dispatcher) {
            is HandlerContext -> {
                // 主线程：用 Handler 延迟
                val handler = dispatcher.handler // 主线程 Handler
                val runnable = Runnable {
                    // 1000ms 后执行这个 Runnable
                    // 恢复协程
                    continuation.resume(Unit)
                    // 这会触发 FetchUserStateMachine.resumeWith()
                    // 协程从 label=1 继续执行
                }
                handler.postDelayed(runnable, timeMillis)
                
                // 支持取消：协程取消时，移除这个 Runnable
                continuation.invokeOnCancellation {
                    handler.removeCallbacks(runnable)
                }
            }
            
            is ScheduledExecutorService -> {
                // IO/Default 线程：用 ScheduledExecutor
                val future = scheduledExecutor.schedule(
                    { continuation.resume(Unit) },
                    timeMillis,
                    TimeUnit.MILLISECONDS
                )
                continuation.invokeOnCancellation {
                    future.cancel(false)
                }
            }
        }
        
        // 函数返回后：
        // 1. continuation 已经被注册到定时器
        // 2. 返回 COROUTINE_SUSPENDED
        // 3. 当前线程被释放，去做其他事
    }
}
```

完整流程：  
调用 fetchUser(outerContinuation)：  

① 创建 FetchUserStateMachine(completion=outerContinuation)  
   sm.label = 0  

② 进入 when(0)：  
   调用 delay(1000, sm)  
   
③ delay 内部：  
   向 Handler 注册 postDelayed(sm.resume, 1000ms)  
   返回 COROUTINE_SUSPENDED  
   
④ fetchUser 返回 COROUTINE_SUSPENDED  
   当前线程释放！去做其他事  

⑤ 1000ms 后：  
   Handler 触发回调  
   调用 sm.resumeWith(Result.success(Unit))  
   
⑥ sm.resumeWith 内部：  
   sm.label 已经是 1  
   重新调用 fetchUser(sm)  
   
⑦ 进入 when(1)：  
   val user = User("张三")  
   sm.completion.resume(user)  ← 通知外部调用者  
   return user  
   
⑧ 外部调用者（outerContinuation）被恢复  
   拿到 User("张三")  
## 作用域
涉及到代码作用范围、协程 lifecycle 等    
lifecycle：创建协程返回 job 对象，以此获取当前协程状态，有 isActive/isCompleted/isCancelled  
GlobalScope：单例，全局作用域，不会自动结束执行（进程级别），所以一般不用，如果用了需要及时注销，避免 OOM     
MainScope：主线程作用域，也是全局范围  
lifecycleScope：lifecycle 范围，用于 Activity 等 lifecycle 组件，destroy 时自动结束（绑定生命周期）  
viewmodelScope：ViewModel 自带的协程作用域，表示 viewmodel 范围，被回收时自动结束（每个 ViewModel 都有一个 viewModelScope，ViewModel 销毁时，viewModelScope 自动取消，所有在里面启动的协程全部取消（其实就是：父协程取消 → 所有子协程取消），不会内存泄漏，默认主线程启动）  
CoroutineScope（coroutineContext）：传入 Dispatcher.IO/Main/Default 指定作用域（也可以自定义作用域），activity 可以实现这个接口  
Coroutinecontext：以 Key（接口，元素实现CoroutineContex）为索引数据集合接口，左向链表存储数据（节点为具体实现类 CombinedContext），可获取Job、Dispatcher，Job 的子协程发生异常被取消会同时取消 Job（返回 JobImpl 对象，继承 JobSupport，childCancelled 调用 cancellImpl 取消父 Job 和父 Job 所有子 Job）的其它子协程，而 SupervisorJob（返回 SuperviosrJobImpl 对象，是 JobImpl 子类，重写了 childCancelled 方法，返回 false）不会。  
flow：流式处理，lifecycle与协程绑定  
flow {}.onStart{}.onCompletiom{}.collect（suspend，默认的 flow 是冷流，即当执行 collect 时，上游才会被触发执行）{it -> ()}，不能自主取消，依赖协程取消，flowOn 切换线程，需要指明 context，launchIn 为接收操作符，指明此 Flow 运行的 scope。  
## Warning
async：async 作为根协程时，会在 await 的时候抛出异常（源码中有 root coroutine 的标注），作为子协程时会立即抛出异常。  
可以通过加 coroutineExceptionHandler来保底（在 launch 中传入 {coroutinContext, throwable ->}），作用域中未被捕获的异常会最终交给 coroutineEceptionHandler 来处理。  
此外，作用域不同，结果也不同，如在子协程 launch 中传入该 handler，也不会处理（协程的异常会向父协程委托处理，最终会走到根协程中，所以需要在根协程中传入这个 handler），coroutineScope 作用域设置也不会拦截子协程的异常，而 supervisor 作用域可以（被视为根协程），源码注释不建议使用这个作为常规手段（执行到这里时，协程已经完成并出现了相应异常，无法从该 handler 中的异常恢复）
## Flow
建立在协程之上，顺序发出多个同一类型值，不只返回单个值的挂起函数，通过挂起函数异步生成和使用值  
1.提供方异步生成添加到数据流中的数据（流式发射中间状态）  
2.（可选）中介修改发送到数据流的值，或修正数据流本身  
3.使用方使用数据流中的值    
4.需要重发数据，选择 SharedFlow。需要重发最新的数据，选择 StateFlow，StateFlow 不会发送重复的数据  
5.有背压处理，支持取消，可组合多个数据流  
6.对于 flow 而言，可以看做是 RxJava 的简化版，但功能相同，可以处理复杂场景，简单的场景还是可以使用 liveData 的。
### 背压
背压（Backpressure）= 生产速度 > 消费速度 时的处理策略  

场景：  
生产者：每10ms 发射一个事件（100个/秒）  
消费者：每100ms 处理一个事件（10个/秒）  

问题：  
1.全部缓存 → 内存溢出  
2.丢弃→ 数据丢失  
3.阻塞生产者→ 可能死锁  

处理：  
```
// Flow 是冷流，天然支持背压
// 生产者和消费者在同一协程链上
// 消费者处理完才会触发下一个生产

flow {
    repeat(100) { i ->
        emit(i)           // 发射
        // emit 是挂起函数！
        // 消费者没处理完，emit 会挂起等待
        // 生产者自动被"限速"
    }
}.collect { value ->
    delay(100)            // 模拟慢消费
    println(value)        // 处理完才会触发下一个 emit
}
// 这就是 Flow 的天然背压：消费者控制生产速度
```

背压操作符：  
```
val fastFlow = flow {
    repeat(1000) { i ->
        emit(i)
        delay(10) // 每10ms发一个
    }
}

// ① buffer：缓冲区，生产者不等消费者
fastFlow
    .buffer(capacity = 64) // 最多缓冲64个
    .collect { value ->
        delay(100) // 慢消费
        println(value)
    }
// 生产者可以提前生产，放入buffer
// buffer满了才挂起等待

// ② conflate：只保留最新值，丢弃中间值
fastFlow
    .conflate() // 消费者忙时，只保留最新的
    .collect { value ->
        delay(100)
        println(value) // 只会看到最新的值，中间的被丢弃
    }

// ③ collectLatest：新值来了，取消旧的处理
fastFlow
    .collectLatest { value ->
        delay(100) // 如果100ms内来了新值，这里会被取消
        println("处理完: $value") // 只有最后一个值能打印出来
    }

// ④ debounce：防抖，静默一段时间后才处理
fastFlow
    .debounce(500) // 500ms内没有新值才处理
    .collect { value ->
        println(value) // 只处理"稳定"的值
    }
```
### collect
收集 Flow 发出的每个值：  
Flow   = 水管  
emit   = 往水管里放水  
collect = 在水管末端接水  
### Tips
1.flow 是冷流，当观察者开始订阅时，才触发 emit（每次 collect 都重新执行生产逻辑，没有 collect 就不生产），rxJava 是一旦数据产生就会推送给所有观察者。  
2.可以使用 buffer 操作符添加缓存区，配合 onEach 进行流的中间处理，此外还可使用 conflate 保证只传递最新数据，来尽量避免背压。  
3.StateFlow 是单一值状态 Flow，主要用于处理单一状态的场景，如 ViewModel 中 UI 状态。而 SharedFlow 允许有多个订阅者，并能缓存一定数量的最新元素，适用于多个订阅者需要获取历史元素的场景。如果只关心最新状态，StateFlow 更合适，如果需要获取历史元素，或者存在多个订阅者，可使用 SharedFlow。  
4.StateFlow 是热流（sharedFlow 的特化版本，始终有一个当前值，只有值变化时才通知（去重）），本身并没有对线程的调度进行限制，因此在多线程环境中，需要在合适的协程上下文中使用 StateFlow。通常建议在主线程上更新 StateFlow，以确保 UI 的线程安全性。在不同协程中更新 StateFlow 可能会导致竞态条件，因此需要确保在更新 StateFlow 时使用适当的同步机制，如 Mutex，确保在不同协程中更新 StateFlow 时的同步性，可以有效避免竞态条件（Mutex.withLock）。    
5.SharedFlow 是热流，在订阅者加入后开始产生事件，（不管有没有 collect，都可以发射，多个 collector 共享同一个流），因此可能存在热启动问题，即在订阅前产生的事件会被忽略。为了解决这个问题，可以使用 stateIn 操作符来创建一个 StateFlow，并在需要时将其转换为 SharedFlow，通过使用 stateIn 的 SharingStarted.Eagerly 参数，可以确保在订阅者加入前就开始产生事件，避免热启动问题。
## Cover
1. Kotlin-JVM 中所谓的协程是假协程，本质上还是一套基于原 生Java Thread API 的封装。和 Go 中的协程完全不是一个东西，不要混淆,更谈不上什么性能更好。  
2.Kotlin-JVM 中所谓的协程挂起，就是开启了一个子线程去执行任务（不会阻塞原先 Thread 的执行，要理解对于 CPU 来说，在宏观上每个线程得到执行的概率都是相等的），仅此而已，没有什么其他高深的东西。  
3.Kotlin-JVM 中的协程最大的价值是写起来比 RxJava 的线程切换还要方便。几乎就是用阻塞的写法来完成非阻塞的任务。  
4.对于 Java 来说，不管你用什么方法，只要你没有魔改 JVM，那么最终你代码里 start 几个线程，操作系统就会创建几个线程，是 1 比 1 的关系。
## VS Thread
线程：由操作系统调度，抢占式，开发者无法控制"下一个执行什么"  
协程：由协程框架调度，协作式，挂起点明确，框架决定下一个任务  
### Thread
操作系统线程调度（抢占式）：  

开发者能做的：  
1.创建线程、指定任务  
2.设置线程优先级，影响调度概率  
3.使用同步原语（等待、通知）  

不能做的：  
1.决定线程何时获得时间片  
2.决定线程运行多久被中断   
3.决定多个就绪线程的执行顺序  

CPU时间片：  
线程A: ████░░░░████░░░░████  
线程B: ░░░░████░░░░████░░░░  
线程C: ░░░░░░░░░░░░░░░░░░░░  ← 被阻塞，不参与调度  

时间片分配算法 CFS：  
```
让每个线程获得"公平"的CPU时间

关键概念：vruntime（虚拟运行时间）
├── 每个线程维护一个 vruntime 值
├── 线程运行时，vruntime 增加
├── 线程等待时，vruntime 不增加
└── 调度器总是选择 vruntime 最小的线程运行
    → 运行时间少的线程优先获得CPU
    → 实现"公平"

CFS 调度过程：

就绪队列（红黑树，按vruntime排序）：
        [线程C vruntime=10]
       /                   \
[线程A vruntime=5]    [线程D vruntime=20]
       \
    [线程B vruntime=8]

调度器：取 vruntime 最小的 → 线程A 获得CPU

线程A 运行一个时间片（约4ms）：
线程A.vruntime += 4ms × weight_factor

重新插入红黑树：
        [线程B vruntime=8]
       /                   \
[线程C vruntime=10]    [线程D vruntime=20]
       /
[线程A vruntime=9]  ← A的vruntime增加了，位置变了

下次调度：取 vruntime 最小的 → 线程B
```

时间片大小计算：  
```
动态计算：
时间片 = 调度周期 / 就绪线程数

调度周期（sched_latency）= 6ms ~ 48ms（根据线程数调整）
就绪线程数 = 4
→ 每个线程时间片 = 12ms / 4 = 3ms

线程优先级（nice值）影响 weight_factor：
├── 高优先级线程：weight大，vruntime增长慢
│   → 相同时间内，vruntime增加少
│   → 更快轮到它执行
└── 低优先级线程：weight小，vruntime增长快
    → 更少机会获得CPU
```

调度触发：  
```
① 时间片用完
   当前线程的 vruntime 超过下一个线程太多
   → 抢占，切换到 vruntime 最小的线程

② 线程主动让出
   Thread.sleep() / wait() / IO等待
   → 线程进入 SLEEPING/WAITING 状态
   → 从就绪队列移除
   → 立即调度其他线程

③ 高优先级线程就绪
   某个等待的高优先级线程被唤醒
   → 如果它的 vruntime 比当前线程小很多
   → 抢占当前线程

④ 系统调用返回
   线程从内核态返回用户态时检查是否需要调度
```

Tips：  
1.同一线程内部，代码按顺序执行  
2.不同线程之间，执行顺序有OS决定  
3.thread.start 之后，主线程立即继续，子线程可能在此之前或之后  

特点：  
1.OS 随时可以中断任何线程（时间片用完）  
2.线程切换由 OS 决定，开发者无法干预  
4.线程间没有"代码顺序"关系  
5.线程切换有上下文切换开销（寄存器保存/恢复，约1~10μs）  
### Coroutine
协程调度（协作式）：  

主线程消息队列：  
[协程A任务1] → [协程B任务1] → [协程A任务2] → [点击事件] → [协程B任务2]  
      ↑               ↑               ↑
  A执行到挂起点    B执行到挂起点    A恢复执行  

特点：  
1.协程主动让出执行权（在挂起点）  
2.框架决定下一个执行哪个协程的哪段代码  
3.挂起 ≠ 线程阻塞（线程可以去做其他事）  
4.同一线程上的协程，某一时刻只有一个在执行  

协程框架：  
```
// 协程框架 = 调度器（Dispatcher）+ 协程上下文

// 主线程调度器（HandlerContext）
// 本质：Android 主线程的 Handler + MessageQueue

// 当你写：
launch(Dispatchers.Main) {
    println("任务1")
    delay(1000)        // 挂起点
    println("任务2")
}

// 框架做的事：
// 1. 将"执行到delay之前的代码"包装成 Runnable
//    → 投递到主线程 MessageQueue
// 2. 主线程从 MessageQueue 取出，执行到 delay
// 3. delay 注册1000ms后的回调，返回 SUSPENDED
// 4. 主线程继续取 MessageQueue 下一个任务（可能是其他协程/点击事件）
// 5. 1000ms后，delay回调触发
//    → 将"从delay之后继续执行"包装成 Runnable
//    → 投递到主线程 MessageQueue
// 6. 主线程取出，执行 println("任务2")
```

挂起恢复：  
```
// 完整场景演示
fun main() {
    val scope = CoroutineScope(Dispatchers.Main)

    // 协程A
    scope.launch {
        println("A1")
        delay(1000)    // A 挂起
        println("A2")  // 1000ms后恢复
    }

    // 协程B
    scope.launch {
        println("B1")
        delay(500)     // B 挂起
        println("B2")  // 500ms后恢复
    }

    println("主线程其他代码")
}

// 主线程 MessageQueue 执行过程：
//
// 时刻 0ms：
// 队列：[启动A的任务] [启动B的任务] [主线程其他代码]
//
// 取出"启动A"：
//   执行 println("A1")
//   执行 delay(1000)
//     → 向Handler注册：1000ms后投递"恢复A"
//     → 返回 SUSPENDED
//   A挂起，当前任务结束
//
// 队列：[启动B的任务] [主线程其他代码]
//
// 取出"启动B"：
//   执行 println("B1")
//   执行 delay(500)
//     → 向Handler注册：500ms后投递"恢复B"
//     → 返回 SUSPENDED
//   B挂起
//
// 队列：[主线程其他代码]
//
// 取出"主线程其他代码"：
//   执行 println("主线程其他代码")
//
// 队列：[] （空，主线程空闲等待）
//
// 时刻 500ms：
// Handler触发，投递"恢复B"
// 队列：[恢复B]
//
// 取出"恢复B"：
//   从 delay 之后继续执行
//   执行 println("B2")
//   B结束
//
// 时刻 1000ms：
// Handler触发，投递"恢复A"
// 取出"恢复A"：
//   执行 println("A2")
//   A结束
```
### 阻塞 vs 挂起
```
// 线程阻塞（Thread.sleep）
thread {
    println("线程开始")
    Thread.sleep(1000) // 线程被阻塞！
    // 这1秒内，这个线程什么都不能做
    // 如果是主线程 → ANR！
    println("线程结束")
}

// 协程挂起（delay）
launch(Dispatchers.Main) {
    println("协程开始")# 
    delay(1000) // 协程挂起，但主线程没有阻塞！
    // 这1秒内，主线程可以处理点击事件、刷新UI等
    println("协程结束")
}

// 本质区别：
// Thread.sleep → 系统调用，线程进入 WAITING 状态，让出CPU
// delay → 向调度器注册回调，协程状态机保存，线程继续跑其他任务
```
