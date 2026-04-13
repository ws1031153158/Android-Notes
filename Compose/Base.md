# @Composable 函数
对应一个XML的Layout。  
与kotlin中的suspend关键字类似，表明一个函数类型，同样，也只能调用/被调用@Composable标识的函数，也需要一个贯穿所有的上下文调用对象。  
## 执行模式
传递的调用上下文是一个Composer，实现包含了一个与 Gap Buffer (间隙缓冲区) 密切相关的数据结构。  
Gap Buffer内存中使用扁平数组 (flat array) 实现，该数据比原数据集合大，未使用的空间则为间隙。   
例如：  
<img width="1164" height="1478" alt="image" src="https://github.com/user-attachments/assets/08ad7c11-e4fc-4883-abe4-4f5025ff0782" />

一个正在执行的 Composable 的层级结构可以使用这个数据结构，而且我们可以在其中插入一些东西。    
例如：  
<img width="1162" height="1470" alt="image" src="https://github.com/user-attachments/assets/be93574f-3b7f-499e-a874-518844814e2f" />

完成层级结构的执行后，可以将游标重置回数组的顶部并再次遍历执行来重新组合，可以选择仅查看数据，或更新数据的值。  
例如：  
<img width="1214" height="1452" alt="image" src="https://github.com/user-attachments/assets/9f677317-0f1a-4fac-a45d-eaaa678ed52a" />

改变 UI 结构，并进行一次插入操作时，把间隙移动至当前位置。  
例如：  
<img width="1166" height="1482" alt="image" src="https://github.com/user-attachments/assets/10f851a0-f581-45f3-8a57-f3eb9ec2026a" />  

<img width="1162" height="1448" alt="image" src="https://github.com/user-attachments/assets/1adac1d3-2173-4a3f-9ae2-1a91d9492164" />

### 编译
当编译器看到 Composable 注解时，它会在函数体中插入额外的参数和调用。  
1.添加一个 composer.start 方法的调用，并向其传递一个编译时生成的整数 key（以key为基准，判断UI结构是否变动）。  
2.将 composer 对象传递到函数体里的所有 composable 调用中。  
3.在函数末尾声明一个composer.end。    
例如：  
@Composable  
fun Counter() {  
 var count by remember { mutableStateOf(0) }                    
 Button(  
   text="Count: $count",  
   onPress={ count += 1 }  
 )  
}  
  
↓    
  
fun Counter($composer: Composer) {  
 $composer.start(123)  
 var count by remember { mutableStateOf(0) }  
 Button(     
   text="Count: $count",  
   onPress={ count += 1 }  
 )  
 $composer.end()  
}  

内存如下（数据结构持有了来自组合的所有对象，整个树的节点也已经按照深度优先遍历的执行顺序排列）：  
<img width="1400" height="1117" alt="image" src="https://github.com/user-attachments/assets/f8ceaeba-e60e-4812-94ad-ed3a163817ab" />  

这些组对象已经占据了很多空间，用来管理动态 UI 可能发生的移动和插入，编译器知道哪些代码会改变 UI 的结构，所以它可以有条件地插入这些分组。  
### 存储参数
Compose 将 Composable 函数的参数存储在插槽表中，添加 static 参数来消除非上层函数入参的其他参数多次存储的冗余。   
例如：  
@Composable fun Google(number: Int) {  
 Address(  
   number=number,  
   street="Amphitheatre Pkwy",  
   city="Mountain View",  
   state="CA"  
   zip="94043"  
 )  
}  
   
@Composable fun Address(  
 number: Int,  
 street: String,  
 city: String,  
 state: String,  
 zip: String  
) {  
 Text("$number $street")  
 Text(city)  
 Text(", ")  
 Text(state)  
 Text(" ")  
 Text(zip)  
}  

↓  

fun Google(  
 $composer: Composer,  
 $static: Int,  
 number: Int  
) {  
 Address(  
   $composer,  
   0b11110 or ($static and 0b1),  
   number=number,  
   street="Amphitheatre Pkwy",  
   city="Mountain View",  
   state="CA"  
   zip="94043"  
 )  
}  

  fun Address(  
  $composer: Composer,  
  $static: Int,  
  number: Int, street: String,   
  city: String, state: String, zip: String  
) {  
  Text($composer, ($static and 0b11) and (($static and 0b10) shr 1), "$number $street")  
  Text($composer, ($static and 0b100) shr 2, city)  
  Text($composer, 0b1, ", ")  
  Text($composer, ($static and 0b1000) shr 3, state)  
  Text($composer, 0b1, " ")  
  Text($composer, ($static and 0b10000) shr 4, zip)  
}  
# 重组
编译器为 Counter 函数生成的代码含有一个 composer.start 和一个 compose.end。每当 Counter 执行时，运行时就会理解：当它调用 count.value 时，它会读取一个 appmodel 实例的属性。在运行时，每当我们调用 compose.end，我们都可以选择返回一个值。   
例如：  
$composer.end()?.updateScope { nextComposer ->  
 Counter(nextComposer)  
}  
接下来，我们可以在该返回值上使用 lambda 来调用 updateScope 方法，从而告诉运行时在有需要时如何重启当前的 Composable。  
  
PS：appmodel是指在Jetpack Compose运行时中，用于管理和存储UI状态及相关属性的实例，编译器生成的代码通过它读取UI元素的属性。具体来说，它是Compose中承载UI状态（如count.value）的核心组件，负责在重组过程中维护和传递这些状态信息。
# Lazycolumn
类似 RecyclerView，只渲染屏幕上可见的item，滚动时复用，使用一个key帮助 LazyColumn 识别item身份。  
1.避免不必要的重新渲染    
2.让动画正确播放  
3.用唯一稳定的字段作为key  
4.避免使用index或随机值
# StateFlow
设计为单一数据源，如一个 data class 管理所有UI状态。  
1.用update{}原子更新  
2.对外暴露只读的 asStateFlow()  
3.UI只订阅一个流  
4.状态不会出现矛盾
# LaunchedEffect
在 Composable 里启动协程的方式。  
Composable 函数会反复执行（重组），如果直接在里面启动协程，每次重组都会启动新协程，造成协程泄漏。  
LaunchedEffect 只在 key 变化时重新启动协程（取消旧协程），Composable重组完成，自动取消协程。  
LaunchedEffect 跟随 Composable 生命周期，ViewModelScope 跟随 ViewModel 生命周期（更长）。
# remember
Composable 函数的特点是反复执行（重组）：状态变化、父组件重组、影响UI的变化（数据等），而每次重组重新执行时，局部变量会重置，remember 则会保持值不变，重组后使用缓存值。  
## 用法
1.保存状态：var XX by remember {mutableStateOf(X)}  
2.保存对象：val XX = remember {XX()}  
3.带key（key变化重新初始化）：var result by remember(id) {multableStateOf(XX(id))}  
4.rememberSaveable（屏幕旋转后保留）：var text by rememberSaveable{multableStateOf(X)}  
5.rememberXX系列：官方封装好的 remember，把常用的对象创建封装起来
