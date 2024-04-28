# 扩展函数
## 内联
inline 关键字用于声明内联函数，它的作用是告诉编译器在调用处直接将函数的代码插入，而不是通过常规的函数调用来执行，执行时，程序会按顺序执行内联函数体的代码和 Lambda 表达式的代码。   
可以减少函数调用带来的性能开销，并且可以避免创建匿名对象和类。内联函数通常用于高阶函数、Lambda 表达式等场景，可以提高程序的执行效率。   
let：一般与 .？一起使用，代码块内部是对调用对象的各个方法的访问，it 替代该对象，底层为 inLine 的扩展函数 + lambda 结构，返回值为函数块的最后一行或指定 return 表达式  
with：和 let 类似，内部 this 指代调用对象（可忽略），该结构返回最后一行或指定的 return 表达式。传入两个参数，调用对象和一个 lambda 体，若最后一个参数是函数则可以放在括号外，常用于调用同一个对象的多个方法。（tip：with 不是扩展函数）  
apply：和 run 类似，返回值不同，返回传入对象本身（返回 this），一般用于对象初始化，对属性复制，或动态 inflate 一个 view 给 view 绑定数据  
also：和 let 类似，it 指代，返回当前的这个对象（返回 this），一般可用于多个扩展函数链式调用  
run：可看做 let 和 with 的结合体，只接受一个 lambda 函数作为参数。可以判空，且也是由 this 指代（可省略），以闭包形式返回最后一行代码的值。
### 编译时常量
变量的值固定不变，在编译时我们就可拿到这个变量的值。具体到代码中，就是变量使用 final 修饰，类型只能是字符串或基本类型，在声明时就赋值；在编译时，会直接拿这个变量值去替换变量名，是一种内联优化。
## 自定义
在类的外面定义扩展函数，可在类内部/外部调用，无需修改类本身即可实现功能的扩展，不会作为成员添加到类中，可在接口中定义扩展，可定义成员扩展（不支持引用，且多个隐式访问可读性差）  
Class.function，本质是静态函数，编译器将扩展函数转化为静态函数的调用
## noinline & crossinline
1.函数类型参数本质是一个对象，而 inline 函数会将这个对象的代码展开铺平，消除其对象属性。   
2.noinline 用于标记 inline 函数中的函数类型参数，被标记的函数类型参数不会被内联。   
3.crossinline 是交叉内联，也是作用于内联函数的函数类型参数上，用途是强化函数类型参数的内联优化，使能被间接调用，并且被 crossinline 标记的 Lambda 中不能使用 return。
## Tips
1.只有作为内联函数的参数的 Lambda 表达式中可以直接使用 return  
2.Lambda 表达式中的 return 结束的不是内联函数自身，而是内联函数的调用函数  
# When
等同于 switch，且类型推导让我们无需指定返回值类型也不需要写 break，可以匹配任一对象，没有限制
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
通过编译技术实现，不依赖任何 os/vm，编译时 suspend 函数传入 Continuation 接口，持有 context 和一个 resumeWith 作为参数，其中 resumeWith 会调用 invokeSuspend 协程启动或挂起时恢复，协程运行完成回调。这些代表之后要执行的方法，	编译器将协程体编译成匿名内部类，挂起函数调用位置对应挂起点，	类中实现 create 来创建协程体类实例，使用状态机，维护一个 label，控制 invokeSuspend 方法执行不同条件分支，挂起函数调用分布在不同分支，挂起函数传参为 this，即协程体自身。    
线程会占用内存，且切换开销较大，线程的切换是在内核态下完成的，所以用户态中发起线程的切换会中断，进入内核态，所以需要内核的支持，用户级的线程切换时，才需要内核的支持，用户态和内核态其实就是CPU指令集权限级别的区别，用户态相当于应用程序，内核态相当于操作系统，多个协程可以共用一个线程，切换位于用户态。  
是一种通过挂起机制实现可中断的方法，通过栈实现层级调用
.launch(Dispatchers.IO/Main/Default/Unconfined）{  
}    
//基于线程池实现，四种调度器，父类为	CoroutineScheduler，通过 dispatch 分配运行的线程  
//launch 并不是一个顶层函数，它必须在一个对象中使用，是	CoroutineScope 接口（提供contexat，通过扩展函数管理协程）的扩展函数  
此结构体实现一个协程，通过 withContext 切换运行线程，并在完成后返回当前线程，该方法为一个 suspend 方法，需要在协程或另一个 suspend 方法中调用。  
async 也返回一个协程，不同点在于实现了 Deferred 接口，是一种延迟执行的方法，需要调用 .await（也是一个 suspend 方法）获取结果，一般用于同时开启多个任务，获取到所有返回结果再继续执行后面的任务。  
默认启动，可以通过 start 手动开启，懒加载，在 launch 中传入CoroutineStart.LAZY，可以在 job.invokeOnCompletion 回调之后执行一些操作，正常和异常执行完毕都会走。  
## 启动模式
DEFAULT：立即开始调度  
LAZY：start/join/await 开始调度  
ATOMMIC：立即开始调度，第一个挂起点前无法取消（涉及cancel才有意义）  
UNDISPATCHED：立即开始调度，直到第一个挂起点（之后由调度器（Dispatcher基于线程池实现）处理）  
## Dispatcher
定义了 Coroutine 执行的线程。CoroutineDispatcher 可以限定 Coroutine 在某一个线程执行、也可以分配到一个线程池来执行、也可以不限制其执行的线程，CoroutineDispatcher 是一个抽象类，所有 dispatcher 都应该继承这个类来实现对应的功能，Dispatchers 是一个标准库中帮我们封装了切换线程的帮助类，可以简单理解为一个线程池。  
Dispatchers.Default：    
默认的调度器，适合处理后台计算，是一个CPU密集型任务调度器，创建 Coroutine 的时候没有指定 dispatcher，则一般默认使用这个作为默认值，defaultDispatcher 使用一个共享的后台线程池来运行里面的任务。注意它和IO共享线程池，只不过限制了最大并发数不同。    
Dispatchers.IO：    
顾名思义这是用来执行阻塞 IO 操作的，是和Default共用一个共享的线程池来执行里面的任务。根据同时运行的任务数量，在需要的时候会创建额外的线程，当任务执行完毕后会释放不需要的线程。    
Dispatchers.Unconfined：    
由于Dispatchers.Unconfined未定义线程池，所以执行的时候默认在启动线程。遇到第一个挂起点，之后由调用resume的线程决定恢复协程的线程。    
Dispatchers.Main：    
指定执行的线程是主线程，在Android上就是UI线程，由于子Coroutine 会继承父Coroutine 的 context，所以为了方便使用，我们一般会在 父Coroutine 上设定一个 Dispatcher，然后所有 子Coroutine 自动使用这个 Dispatcher。    
## Suspend
线程执行到 suspend 方法时，不再执行剩余的协程代码，跳出代码块，invokeSusped 返回	COROUTINE_SUSPENDED 时执行协程的 return，去执行别的任务，而协程 post 到指定线程继续执行，完成后会切回原线程，即协程 post 一个 Runnable，让剩余代码回到原线程执行：withcontext 调用	startCoroutineCancellable，传入 coroutine 回调，DispatchedCoroutine，持有外部要恢复的协程，协程 return 会走 resumeWith，重写 afterCompletion，判断是否挂起，通过 resumeCancellableWith 恢复外部协程，再次触发 invoke。  
## 作用域
涉及到代码作用范围、协程 lifecycle 等    
lifecycle：创建协程返回 job 对象，以此获取当前协程状态，有 isActive/isCompleted/isCancelled  
GlobalScope：单例，全局作用域，不会自动结束执行（进程级别）  
MainScope：主线程作用域，也是全局范围  
lifecycleScope：lifecycle 范围，用于 Activity 等 lifecycle 组件，destroy 时自动结束（绑定生命周期）  
viewmodelScope：viewmodel 范围，被回收时自动结束  
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
1.提供方异步生成添加到数据流中的数据  
2.（可选）中介修改发送到数据流的值，或修正数据流本身  
3.使用方使用数据流中的值    
