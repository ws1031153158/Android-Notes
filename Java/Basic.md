# 函数式编程
只有一个入口，同时也只有一个出口，通过将过程原子化，来实现结果的一致性 —— 有怎样输入，就有且只有怎样输出，关注点只有这两处，因 “集中管理” 而从根本上杜绝不可预期结果。
# Enum
最终实现原理还是类，编译完成后，为每一种类型生成静态对象，对象需要的内存空间大于静态常量，对象中默认有一个字符数组空间的申请。  
可通过 ProGuard 将简单枚举优化为一个整型（int），也有限制，以下无法优化：  
1.枚举实现了自定义接口，并被调用  
2.发生类型转换（使用了不同签名存储枚举、使用 instanceOf 指令判断、对枚举强转）  
3.使用枚举加锁操作  
4.调用静态方法 valueOf  
5.定义可以外部访问的方法  
# Annotation
将数据、信息和类、方法、变量进行关联。  
元注解：  
@Target：使用目标，由 ElementType 枚举  
@Retention：使用级别，由 RetentionPolicy 枚举  
@Documented：javadoc 文档化,输出中显示注解  
@Inherited：允许子类继承父类注解  
自定义注解：  
public @interface XXX {  
    T xx() default x  
}  
注解处理器：  
编译时：  
APT(Annotation Processot Tool)实现，编译器扫描处理注解信息，以 .java 或编译后的 .class 作为输入，输出一般为 .java，和其他源文件一起被编译，继承 AbstractProcessor 实现自定义处理器，注册并打包 jar。  
运行时：  
反射实现，AnnotationProcessor.init 反射获取类构造函数或其他方法、元素，接着通过构造函数（method、元素）.getAnnotation 获取注解，最终初始化元素，通过 AnnottedElement 对象来获取 Annotation 信息，包含各种getAnnotation方法。
# 泛型
## Class
真泛型：泛型中的类型真实存在，伪泛型：仅在编译时类型检查，运行时擦除类型。    
在编译期间类型检查提高类型安全，减少运行时对象类型不匹配异常，只在编译期约束，运行时无法控制。    
可通过反射在规定类型的集合中添加不同类型的元素：  
1、创建 Integer 类型集合  
2、对象名.getClass 获取 Class 对象  
3、getMethod 获取指定 Method  
4、invoke 将不同数据类型数据添加到集合  
## 类型擦除
为了兼容 API，老版本不支持泛型，运行时 JVM 不知道泛型，编译后泛型版本替换为原始无泛型的版本，泛型会被转化为 Object，指定上限后会替换为上限类型。
## Problem
方法重载时与要重载的方法参数类型不一致会有问题，可以通过桥接方法解决，生成两个方法，一个实现 Object 的一个实现重载参数的，用接口方法调用额外添加的方法。
# 内存溢出-OOM/内存泄露
## Class
内存溢出是当前所需内存大于可用内存。  
内存泄漏是有异常代码造成的内存不足，可能导致内存溢出，内存未使用但无法被 GC 回收，本质为长生命周期对象持有短生命周期对象的引用。  
未收回引用包括未关闭 Bitmap、Cursor、Stream、网络请求、广播、异步任务等、静态变量（如单例持有 Activity）、内部类对象隐式持有外部类对象引用、WebView。    
非静态内部类对象隐式持有外部类对象引用，编译后生成各自 class 文件，内部类通过 this 访问外部类的成员，编译器自动为内部类添加成员变量，类型和外部类类型相同， 指向外部类对象 (this) 引用 ，接着编译器自动为内部类构造方法添加参数， 类型是外部类类型， 构造方法内部使用参数为内部类添加的成员变量赋值，再调用内部类的构造函数初始化内部类对象时，默认传入外部类的引用。如 Handler 匿名内部类持有 Activity 引用，Activity销毁时无法回收。    
WebView，与 Chromium 内核版本有关，新版 Chromium 内核内存泄漏问题已解决， Android 5.0开始将 Chromium WebView 迁移到了一个独立的 APP -- Android System WebView，低版本 Android 搭载的 Chromium 内核一般来说也不会太旧，所以出现内存泄漏的概率应该是比较小的。如果仍需要兼容这很小的一部分机型，可以先移除 WebView 组件，确保先调用到 onDetachedFromWindow 方法解注册，然后再通过 WebView.destroy 方法处理其它销毁逻辑。
## Resolve
使用软引用/弱引用；及时关闭资源请求入口；及时注销监听、广播，及时取消异步任务；单例持有的 context 使用 applicationcontext，和应用 lifecycle 相同；内部类定义为 static，静态变量引用内部类实例或将实例化操作放在外部类静态方法中，内部类对外部类的引用使用 WeakReference。
# 字节码
一组可以由 Java 虚拟机(JVM)执行的高度优化的指令，被记录在 Class 文件中，在虚拟机加载 Class 文件时执行，存储在Class文件中的方法表中，它以Code属性的形式存在
# Tips
## == / equals
==：对于基本数据类型（如 int、double），用于比较它们的值是否相等； 对于引用类型，比较的是对象的引用是否相同，即它们是否指向同一个内存地址（其实还是比较值，这里的值就是内存地址）     
equals：是 Object 类中的方法，底层还是通过 == 实现的，被许多类（包括基本数据类型的包装类如 Integer、Double）重写，对于基本数据类型的包装类，比较的是对象的值是否相等，如 new Integer(5).equals(new Integer(5)) 返回 true，还要注意在使用 equals 之前，需要确保对象不是 null，此外，针对自己实现的类，需要重写 equals 逻辑
## 深/浅拷贝
浅拷贝：创建一个新对象，这个对象有着原始对象属性值的一份拷贝，如果属性是基本类型，拷贝的就是值，如果属性是引用类型，拷贝的就是内存地址 ，如果其中一个对象改变了这个地址，就会影响到另一个对象     
深拷贝：将一个对象从内存中完整的拷贝一份出来，从堆内存中开辟一个新的区域存放新对象，修改新对象不会影响原对象  
深拷贝的实现可以考虑：  
1.实现 Cloneable 接口，重写 Object 类中 clone 方法，实现层层克隆的方法（对象的 clone 方法默认是浅拷贝）    
2.通过序列化(Serializable)的方法，将对象写到流里，然后再从流中读取出来。虽然这种方法效率很低，但是这种方法才是真正意义上的深度克隆  
# 动态代理
只针对接口，在使用时，通过反射，生成调用对象的代理对象 -> Proxy.newProxyInstance(classLoader, Interfaces, InvocationHandler)，也可以通过传入的 handler 实现自定义 invoke 操作，这个也是核心，传入的 ，method 就是需要代理的方法，最终会调用，所以需要我们传入原始类的接口
# String
## “+” or StringBuilder
1.远古版本的 + 会创建多个字符串对象，造成回收资源浪费，高版本JDK + 会被优化为 new StringBuilder.append  
2.循环场景使用StringBuidler性能更好，因为 + 会创建多个StringBuilder对象  
3.非循环场景使用 + 即可，可读性强
# 递归
函数定义中调用自身、将复杂问题层层分解为同类小问题。  
## 三要素
1.基准条件 (Base Case)：明确的递归出口，当问题简化到一定程度时停止递归，返回特定值。  
2.递归前进段 (Recursive Step)：不断将复杂问题分解为规模更小的同类问题。  
3.递归返回段 (Return Path)：将子问题处理的结果层层上传到上一层，最终组合出整体解。  
## 栈
通过“栈”来实现递归。每进入一层函数调用，系统会为该层调用开辟一块内存（栈帧）来存储局部变量和返回地址（由于栈空间有限，过深的递归会耗尽栈空间，导致“栈溢出”错误；此外，存在函数调用开销，运行效率会低一些）
# 反射
## Class.forName
```
// Class.forName("com.xxx.PaymentProcessor") 内部：

① 字符串解析
   "com.xxx.PaymentProcessor" → 解析包名和类名

② 类查找
   遍历 ClassLoader 链：
   BootClassLoader → PathClassLoader → ...
   在每个 ClassLoader 的 dex 文件中查找类名
   
   // 底层：在 dex 文件的类型索引中二分查找
   // dex 文件的 type_ids 是有序的，支持二分查找
   // 但仍然比直接访问慢很多

③ 类加载（如果还没加载过）
   ├── 读取 dex 中的类定义
   ├── 分配 Class 对象内存
   ├── 初始化静态字段
   ├── 执行 static {} 块
   └── 向 ClassLoader 注册

④ 安全检查
   检查调用方是否有权限访问该类

// 有缓存：同一个类第二次 forName 会快很多
// 但第一次：可能需要 1ms ~ 10ms
```
## getDeclaredMethod
```
// method = clazz.getDeclaredMethod("processPayment", Double.class)

① 获取类的所有方法列表
   从 Class 对象中读取方法数组
   
② 遍历匹配
   for (Method m : declaredMethods) {
       if (m.getName().equals("processPayment") &&
           Arrays.equals(m.getParameterTypes(), paramTypes)) {
           return m;
       }
   }
   // 线性遍历！方法越多越慢

③ 参数类型匹配
   Double.class == double.class ? 不等！
   需要处理装箱/拆箱

④ 创建 Method 对象
   每次调用都可能创建新的 Method 对象（有缓存但有限）

⑤ 安全检查
   检查访问权限

// 没有高效缓存！每次调用都要遍历
// 可以手动缓存 Method 对象来优化
private static Method cachedMethod;
static {
    cachedMethod = MyClass.class.getDeclaredMethod("myMethod");
    cachedMethod.setAccessible(true); // 关闭安全检查
}
```
## method.invoke
```
// result = method.invoke(instance, amount)

① 参数装箱
   double amount = 99.9;
   // 必须装箱为 Double 才能放入 Object[] args
   Object[] args = new Object[]{ Double.valueOf(amount) }; // 堆分配！

② 安全检查（每次都检查，除非 setAccessible(true)）
   checkAccess(caller, clazz, obj, modifiers)
   // 涉及类加载器检查、模块系统检查等

③ 方法分派
   // 反射调用不能被 JIT 内联优化
   // 普通调用：JIT 可以内联，消除函数调用开销
   // 反射调用：必须通过 JNI 调用到 native 层执行

④ native 层执行
   // 通过 JNI 调用 ART 的 art::Reflection::InvokeMethod
   // 参数从 Java 对象转换为 ART 内部格式
   // 执行方法
   // 结果转换回 Java 对象

⑤ 返回值装箱
   // boolean/int/double 等基本类型
   // 必须装箱为 Boolean/Integer/Double 返回

// 综合开销：比直接调用慢 10~50 倍
// 主要来自：装箱、安全检查、无法JIT优化、JNI开销
```
