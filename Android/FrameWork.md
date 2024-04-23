# MVC
Model：View相关的数据，部分持久化的数据  
View：主要是视图  
Controller：Activity 和 Fragment 作为 Controller，用于 View 的交互以及部分 Model 数据交互  
没有做到完全解耦，Controller 过于沉重
# MVP
Model：数据  
View：视图  
Present：View 和 Model 的中介者，做到了 View 和 Model 的解耦  
通过获取局部引用进行精准刷新。  
对于复杂 Layout，会有很多个 Present，接口过多，此外，会持有 activity（view） 对象，可能内存泄露或者 npe
# MVVM
Model：数据  
View：视图  
ViewModel：对 View 和 Model 的交互  
1.让状态管理独立于页面，从而实现 “状态管理分治”、“状态管理一致性” 和 “状态共享”。  
2.为状态设置作用域，使状态共享做到作用域可控。    
3.实现单向依赖，避免内存泄漏。    
做到了 View 和 Model 的解耦，配合 ViewBinding/DataBinding 以及 LiveData 使用，关系更加清晰。    
ViewBinding：提供数据绑定视图，数据改动反应到视图  
dataBinding：提供双向绑定，互相影响（需要加 <layout/> 标签，通过 <data/> 标签设置要绑定的数据（类），在视图 xml 代码块中通过 @ 引用（采用 = 才是真正的双向绑定））
## DataBinding
职责：通过 “适配器模式” + “数据驱动” 设计，规避 View 实例 Null 安全一致性问题，且 “数据驱动” 顺带免去 “调用 View 实例” 导致的冗余 “判空处理”和大量 “样板代码”。  
## LiveDta
1.在 Lifecycle 的帮助下，规避订阅者回调的 “生命周期安全问题”。  
2.遵循 “唯一可信源” 理念的约束，也即通过 “决策逻辑内聚” 和 “读写分离” 来规避 “跨域消息同步” 等场景下 高频 存在的可靠一致问题、使团队新手也能不假思索写出低风险代码。  
3.就算不用 DataBinding，也能使 “单向依赖” 成为可能、规避潜在的内存泄漏等问题。  
# MVI
单向数据流动 + 状态集中管理  
Model 主要指 View 的状态（会维护一个 data class ViewState 或者一个 sealed (相当于枚举类的扩展) class ViewEvent），View 指的是任一个 UI 的容器，I 是 Intent，把每个操作封装为 Intent，发送给 Model 处理（触发 State 的改变，View 对 State 监听，变化后自身也会做出调整（通过 Action 与 Model 联系，解耦）  
model 暴露唯一入口，用来接收 view 唯一出口发出的 intent，同时也只有一个唯一出口用来向 view 返回 state，view 的唯一入口和此出口统一。  
intent -> model -> view   
# 插件化
APP 分为多个模块，一个模块一个 apk，分开打包，只发布主 APK，插件 APK 动态下发给主 apk
## Load Class
通过 DexClassLoader，将插件的 dex 文件加载到宿主中（可以通过反射更改 assets.addAssetPath（设置 apk 文件路径））  
DexClassLoader：也是一种双亲委派，首先加载 class，若已加载完毕，则直接返回 class 对象，否则向父加载器发起加载请求（如 Path 发给 Boot），可以找到 class，则父加载器返回 child.findClass，最终返回 class 对象，若父加载器无法成功找到或加载 class，则由 BaseDexClassLoader 去 findClass，使用一个成员列表维护 dex 文件，通过反射获取 dexElements（.dex 文件，包含插件 dexElement 以及 主 APK 的 dexElement，先合并再更新到主 APK 列表中）
## 四大组件
通过反射和动态代理实现 Hook，对 AMS 和 Handler 进行 Hook 来启动插件的四大组件，Hook 点一般为修改 Intent 的地方。  
启动步骤进行到 AMS 进程时将插件 Activity 替换掉代理 Activity，从 AMS 切换为应用进程时将代理 Activity 替换掉插件 Activity（不唯一）。  
instrumentation.execStartActivity：通过反射，将 AM.getService 返回的 IBinder 对象（AMS 对象）替换为动态代理对象（当执行方法为 startActivity 时替换参数 Intent），随后在 handleMessage 时，将 mCallback 作为 Hook 点，其中 msg.obj 为 ActivityClientRecord，包含一个 Intent，去替换这个 Intent。此外，在 R 之后 ATMS 会有屏蔽，无法在外部通过反射获取，可以通过代理 activity，反射调用插件 activity 的生命周期等方法实现
## 加载资源
Resources 也是通过 AssetManager 类访问被编译过的资源，访问前先根据资源 ID获取对应资源文件名，通过实现类 impl.getAssets 获取 AssetManager 来访问 res 资源，也是一种委托机制，AssetManager 既可通过文件名访问已编译资源， 也可访问未编译过资源，访问 assets 资源。  
raw : Android 会自动为 raw 中资源文件生成 ID，在 xml 中可访问，ID 访问速度最快。  
assets : 生成 ID，只通过 AssetManager 访问，xml 中不能访问，访问速度慢。  
## VirtualAPK
宿主：  对插件 APK 解析，并获取信息，初始化时，对系统组件和接口进行替换，从而对 Activity、Service、ContentProvider 的启动和生命周期进行修改和监控，来启动插件 Apk 的对应组件，通过一个 PluginManager.loadPlugin 加载插件 APK，生成 LoadedPlugin 对象（含有插件信息和一个 DexClassLoader）保存在 Map。  
Activity：  
Hook Instrumentaion（替换为自定义 instrumentation） 或主线程 Handler 的 callback（自定义 instrumentation 实现 callback 接口），对 Intent（在 execStartActivity中injectIntent，对原始 intent 解析，目标 Activity 包名和当前进程不同且对应 LoadedPlugin 对象存在，则为已加载插件 APK 的 Activity，对原始 Intent 的目标 Activity 替换（替换为对应占位 Activity））和 Activity（newActivity 创建实例时，通过 catch（notfindname）处理，使用LoadedPlugin 中 DexClassloader 创建 Activity）进行替换。在宿主 APP 预设占位 Activity，不会启动，欺骗 AMS（先注册）。启动插件 APK Activity 时根据启动模式选择对应占位 Activity, AMS 启动时对占位 Activity 处理，创建 Activity 实例时实际创建插件 APK 中要启动的 Activity（创建之后通过 attach 将更改后的信息传递给 AMS）。最后在 Activity 创建完对 context 进行更换（替换为 PluginContext，内部保存 LoadedPlugin 对象来对 context 内部资源进行替换）  
Service：
使用动态代理来代理宿主 APP 中 Service 请求。不是插件 APK 则为宿主 APP的，直接打开，是插件 APK 的判断是否为远端 Service，是则启动 RemoteService，是本地 Service 则启动 LocalService，在 onStartCommand 中根据所代理的生命周期方法进行处理。通过动态代理对 AMS 的实现类 AM 的 Proxy 进行替换，之后调用 AMS 方法会调到替换后的 invoke 方法中，最终在 RemoteService（运行在另一进程，重新对 APK 解析和装载，生成 LoadedPlugin）或 LocalService 的 onStartCommand 中对 Service 进行处理，ContentProvider 相似。    
插件：注意资源文件冲突，gradle 配置，在插件 APK 编译时对资源 ID（32 位的 16 进制整数，前 8 位代表 app, 后 8 位代表 typeId(string、layout、id)，从 01 开始累加，最后四位为资源 id,从 0000 开始累加）进行重写（获取插件 app 和宿主 app 资源集合，寻找冲突资源 id 进行修改，双重循环，外循环从 01 开始对 typeId 遍历，内循环从 0000 开始对 typeId 对应的资源 ID 遍历重写）
# 组件化
app分为多个模块，包含在一个apk中，可分开调试，编译。通信一般采用路由 + 接口的方式，模块实现接口并向路由注册，客户端向路由发送请求，回调模块的方法，也可采用 eventBus，通过发布-订阅的观察者模式，去实现组件间通信，缺点是需要维护多个 event 类。此外，解决资源冲突，如 Manifest，可用 tools:replace 来指定替换掉的属性。
## ARouter
基于路由表（RouterMap）实现跳转。  
RouteProcessor：注解处理器，.parseRoutes 通过 JavaPoet 接口生成 java 代码）生成路由表，LogisticCenter（.init 判断未在编译时插入则运行时反射，ClassUtils 加载 dex 中的 class 信息，反射初始化并保存到 SP 本地和 Warehouse 索引。  
RegisterTransform：在编译时插入，从 jar 包读取路由表信息，继承 Transform（原生 API，用来修改 class 等资源（读取处理 jar、aar、resource），是一个 Gradle 任务）加载路由表。  
首先通过 ARouter.build 创建一个 Postcard 对象，调用 navigation，进一步调用 _ARouter.navigation，这一步进行 preLoad（实现PretreatmentService接口添加@Route注解，在跳转时加一些case在加载路由表前面），随后 LogisticCenter/RegisterTransform 填充 Postcard 信息，未找到对应路由信息或预处理返回 false 则走降级策略，比如报错或跳到别的指定页面（实现 DegradeService 接口，添加 @Route 注解），最后如果走绿色通道则调用 greenChannel 方法，否则需要实现拦截器（责任链模式），按照类型进行跳转。  
Postcard：继承 RouteMeta（路由表内容，包含跳转路线基本信息，如路线类型、终点、路径、标志（生成路由文档）等），包含在跳转时的传参（uri/tag/bundle/flags/timeout/provider/greenchannel/serializationservice）和动画等信息，未获取到路由信息则重新尝试获取，获取到了就填充内容，如果为 ContentProvider 则先初始化然后 greenchannel，是 Fragment 也走 greenchannel 其他就结束走拦截（也得判断是否手动设置了greenchannel）。  
Annotation：RouteProcessor 对 @Route 类解析，先获取路由元素（method/class..），再创建路由元信息（RouteMeta）将四大组件和一个 RouteMeta 关联，并把路由元信息进行分组（放入不同 groupMap，即 @Route 中设置的 group 值），最后生成路由文件（group、provider、root 路由文件，三者组成路由表）并通过 LogisticsCenter 填充 Warehouse（routes 和 providerIndex 等索引来为跳转提供信息）。  
跳转：首先调用 postcard.navigation 进一步调用 _arouter.navigation 这一步加载路由表。此外，withTransition 可以设置转场动画。  
拦截：onContinue 或 onInterrupt 实现拦截器，至少调一个，是 Activity 就正常走启动流程，是其他三个组件就在终点（destination）用反射创建实例，此外 Fragment 还要设置参数返回实例，contentProvider（实现 IProvider 接口自定义服务）还要通过依赖查找发现服务（依赖注入也可以 @Autowired）
## SPI
Service Provider Interface，该方案是为某个接口动态寻找服务的机制，类似IOC的思想。    
通过 ServiceLoader.load(Machine.class) 创建 ServiceLoader，去遍历 ServiceLoader 就可以找到所有有实现类。
### ServiceLoader
1.先获得一个 classloader    
2.然后去加载 META-INF/services/ 下面的文件，获取相关的配置    
3.获取对应的实现类名，即 parse 方法   
4.利用反射，根据类名去创建对应的实例， 即 nextService 方法  
### AutoService
auto-service 库提供注解，对类加上注解，无需手动维护配置表：  
1.@AutoService(Processor.class)：  
允许/支持的注解类型，让注解处理器处理，编译阶段，会执行 AutoServiceProcessor.process，在该方法中会先调用 generateConfigFiles 生成配置文件，最终生成了配置文件中的类便指向了我们自定义的 Processor，当这个模块在使用的时候，便可以通过该配置找到具体的实现类，并完成实例化。    
2.@SupportedAnnotationTypes({Constants.ROUTER_ANNOTATION_TYPES})：  指定JDK编译版本    
3.@SupportedSourceVersion(SourceVersion.RELEASE_7)：  注解处理器接收的参数  
