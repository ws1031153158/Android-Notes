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
