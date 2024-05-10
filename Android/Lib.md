# OkHttp
网络访问库，封装 socket，实现连接池减少请求延时，并周期定时清理，缓存响应，减少重复的请求，更加精简，更加高效，且支持 HTTP2.0。只是一个 Java 框架，不专用于 Android，定位是网络基础功能库，不需要考虑业务场景是否嵌套调用、切换线程。此外 HttpURLConnection 底层实现也变为 okHttp了。  
基本上只需要了解 OkHttpClient、Request、Call（原型模式，通过 RealCall 调用 newCall 创建，RealCall 真正执行同步或异步的请求）即可。  
同步执行 execute，异步执行 enqueue 并重写 onFailure 和 onResponse（传 callback），最后通过 getResponseWithInterceptorChain（责任链模式）由五大拦截器一层层分发下去，得到结果再一层层返回上来。  
发起请求时，默认会配置一个 gZip 请求头告知服务器，将响应数据使用 gZip 压缩，传输数据量小，时间短。  
首先创建一个 client，接着创建 call 以及一个 request，通过 call 执行 execute/enqueue，由 dispatcher 来分发调度任务，也就是 call，他也负责创建线程池以及记录 call，最后由拦截器决定来 request 还是 post。
## Execute
newCall 为入口，开启请求过程，Call 接口内部除同步、异步方法外还指定了一个 Factory 接口，newCall 位于接口内部，OkHttpClient 实现该接口。  
一个 call 对象只进行一次 execute，刚进入 execute 会做判断，使用一个布尔值来标识，同步是采用了synchronized 锁住了该值。  
内部通过 dispatcher 来分发请求，将请求加入请求队列，完成的 call 会通过 finished 方法 remove 掉。
## Enqueue
同样先做是否已经执行了enqueue 的判断，也是由 synchronized 锁住 this。  
内部是一个双端队列，判断请求队列数是否大于最大请求数 64，以及每个 host 的连接数是否大于最大数 5，没有则加入请求队列，在线程池中去执行这个 call，否则只是加入请求队列，dispatcher 来分发，其实是让 AsyncCall 来请求，相当于委托，后续也会执行 execute，这里的 Call 是 AsyncCall，Call 内部类，实现了 Runnnable 接口，为了能在线程池（类似 asynchTask，有默认线程池）执行 run 方法。  
finished 方法其实是调用了 promoteAndExecute，以此来再次分发 AsyncCall 对象，上次只添加到请求队列没有执行的，会把这个任务添加到 runningAsyncCalls 列表并执行。
## Interceptor
getResponseWithInterceptorChain 获取真正的 response。  
可以在五个拦截器之前自定义业务相关的拦截，例如应用拦截器 addInterceptor，只会调用一次，通常用于统计客户端的网络请求发起情况，对应的有 addNetworkInterceptor，位于 ConnectInterceptor 和 CallServerInterceptor 之间，一次调用代表一次网络通信。  
RetryAndFollowUpInterceptor：失败重传，若DNS设置了多个IP地址，则会对其他地址请求，进行重定向  
BridgeInterceptor：补全请求头，如 Cookie、Accept-coding、Host 等，构建网络请求，并以此访问网络，最后会从网络响应中构建用户响应  
CacheInterceptor：根据算法缓存返回结果，便于下次直接读取，若已有缓存则返回，否侧继续责任链  
ConnectionInterceptor：找到指定服务器的网络链接并获取对应 socket、管理连接池，存取连接来复用连接，就是维护一个复用连接池 ConnectionPool，最后发起连接  
CallServerInterceptor：与服务器建立连接，进行网络请求，并将结果逐层返回  
ConnectionPool：一个双向队列维护 RealConnect，记录连接失败时的的路线名单，最大连接数默认为 5 个、保活时间为 5 分钟，通过判断流是否已经被关闭，并且已经被限制创建新的流来判断当前的连接是否可以使用，如果当前的连接无法使用，就从连接池中获取一个连接，连接池中也没有发现可用的连接，创建一个新的连接，并进行握手，然后将其放到连接池中
# Retrofit
## 概述
底层封装了 OkHttp，是一个 RESTful 的 http 网络请求框架的封装，因为网络请求工作本质上是由 okHttp 来完成，而 Retrofit 负责网络请求接口的封装。  
App 通过 Retrofit 请求网络，实质上是将 okHttp 请求抽象成 Java 接口，使用 Retrofit 接口层采用注解描述和配置网络请求参数、Header、Url 等信息，用动态代理将该接口的注解翻译成一个 Http 请求，之后由 okHttp 来完成后续的请求工作。服务端返回数据后，okHttp 将原始数据交给 Retrofit，Retrofit 根据用户需求解析。  
原理其实就是，拦截到方法、参数，再根据在方法上的注解，去拼接为一个正常的 OkHttp 请求，然后执行。
## 请求流程
1.调用 ApiService 接口的 list方法时，会调用 InvocationHandler 的 invoke方法  
2.执行 loadServiceMethod 方法并返回一个 HttpServiceMethod 对象并调用它的 invoke方法  
3.执行 OkHttpCall的 enqueue方法，本质执行的是 okhttp3.Call 的 enqueue方法  
4.期间会解析方法上的注解，方法的参数注解，拼成 okhttp3.Call 需要的 okhttp3.Request 对象  
5.通过 Converter 来解析返回的响应数据，并回调 CallBack 接口
## 优点
解耦 ，接口定义、接口参数、接口回调不在耦合在一起。  
可以配置不同的 httpClient 来实现网络请求，如 okHttp、httpClient。  
支持同步、异步、Rxjava。  
可以配置不同反序列化工具类来解析不同的数据，如 json、xml。    
请求速度快，使用方便灵活简洁。  
## 注解
Retrofit 使用大量注解来简化请求，Retrofit 将 okHttp 请求抽象成 Java 接口，使用注解来配置和描述网络请求参数。  
如：  
@HTTP 可以替换所有请求方法注解，它拥有三个属性：method、path、hasBody    
@GET 请求方法注解，get请求，括号内的是请求地址，Url的一部分  
@Query 请求参数注解，用于Get请求中的参数    
@Field 请求参数注解，提交请求的表单字段，必须要添加，而且需要配合 @FormUrlEncoded（Post 请求如果有参数需要在头部添加，表示请求实体是一个 From 表单，每个键值对需要使用 @Field 注解）  
@Body 上传 json 格式数据，直接传入实体会自动转为 json，转化方式是 GsonConverterFactory 定义的（不能用于表单或者支持文件上传的表单的编码，即不能与 @FormUrlEncoded 和 @Multipart 注解同时使用）  
@Path 请求参数注解，用于 Url中 的占位符 {}  
@Url 表示指定请求路径，可以当做参数传入  
@headers 请求头注解，用于添加固定请求头，可以添加多个  
@Streaming 表示响应体的数据用流的方式返回，使用于返回数据比较大  
@Multipart 表示请求实体是一个支持文件上传的表单，需要配合 @Part 和 @PartMap 使用，适用于文件上传  
@Part 用于表单字段，适用于文件上传的情况，@Part 支持三种类型：RequestBody、MultipartBody.Part、任意类型  
@PartMap  用于多文件上传， 与 @FieldMap 和 @QueryMap 的使用类似  
## 动态代理
Retrofit 在运行期，生成了 ApiService 接口的实现类，调用了 InvocationHandler 的 invoke方法。  
Retrofit 的 create 方法会实例化定义的 API，并返回一个 call 对象：  
1.获取 classLoader 对象  
2.ApiService 字节码对象传入 Class 的泛型数组  
3.执行 InvocationHandler 的 invoke 方法  
4.代理类生成默认继承 Object，如果是 Object.class，默认调用 object 的方法（method.invoke），如果是默认方法，就执行 platform 的默认方法，否则会执行 loadServiceMethod.invoke 方法  
LoadServiceMethod：
loadServiceMethod(method).invoke(args) 是最关键的代码。  
1.从 ConcurrentHashMap 取出以恶搞 serviceMethod，若存在则直接返回  
2.通过 serviceMethod.parseAnnotations 创建一个 serviceMethod 对象（返回一个 HttpServiceMethod 对象，读取 retrofit 对象获取一些信息，注解参数的解析是由 requestFactory 真正进行的）  
3.用 map 把创建的 serviceMethod 对象缓存起来，因为请求可能多次调用  
invoke：  
最终调用的是 HttpServiceMethod 的 invoke：  
1.创建 call 对象是一个 OkHttpCall  
2.返回一个 adapt 方法  
call 的 enqueue 方法：  
1.声明一个 okHttp3.Call 对象，并初始化（通过 callFactory 创建，在构造中直接赋值，而 callFactory 是一个 okHttpClient 对象，值是在创建 HttpServiceMethod 时赋值的），用来进行网络请求    
2.调用 okHttp3.Call 的 enqueue 方法，进行真正的请求（此方法中 callback 的 onResponse 回调，通过parseResponse解析请求的响应并返回给回调接口，此外会进一步通过 responseConverter 调用convert方法，将返回结果转换为数据模型）  
3.解析响应  
4.成功以及失败的回调  
# Glide
网络图片加载库，通过一系列的链式调用来加载图片。  
placeholder：加载完成前使用的占位图    
error：失败占位图  
fallback：来源为空占位图  
发起请求：with 将参数 lifecycle 和请求绑定，创建一个空 Fragment 监听 Activity 的 lifecycle，接着调用 RequestManager.load 传 url 或 drawable 作为 model 来创建 Request（监听网络状态，其中 RequestManager 包含网络连接监听，网络状态改变会处理图片加载请求）以及创建请求构造器（RequestBuilder），之后会调用 RequestBuilder.into 传入 imageView，通过 GlideBuilder （包含 BitMapPool 等接口和工厂）创建 Request 对象，最后调用 begin。  
启动任务：启动解码任务 DecodeJob，上一步调用 begin 后 Request 会通过 Target 接口的 getSize 获取目标图片的大小，接着调用 Engine.load 尝试从内存获取缓存图片，这里的内存采用 Lrucache，失败则启动 DecodeJob 从图片来源加载请求，加载前或创建 key（资源标识符）与资源绑定，就是将 model 放入 EngineKey。  
解码图片：上一步通过 key 找不到对应资源则 Engine 调用 DecodeJob.run 启动新的任务，接着从第一步传入的 model 获取图片 data(一般为 InputStream)，并通过 ResourceDecoder.decode 根据target大小对 data 进行缩放后的资源，也就是根据请求宽高对原始宽高缩放，将 data 转为 resource，接下里就是对资源进行转码，若有额外选项则通过 Transformation 处理 Resource，调用 Request.onResourceReady，进一步调用 Target.onResourceReady 并将图片设置到目标 view 当中，最终通过 ResourceEncoder.encode 将图片资源编码到磁盘。
## Cache
engine.load 时会先通过 cache 尝试获取，三级缓存分别为内存 -> 磁盘 -> 来源。  
内存为两级：  
1.ActiveResource，包含一个 HashMap 和一个弱引用队列，正在使用或还处于存活状态的资源，Activity 销毁时清理资源。  
2.MemoryCache，为 LruCache，采用 LinkedHashMap 保存数据。    
磁盘：DiskLruCache，磁盘中的图片文件缓存，也有 LinkedHashMap，key 为 String，value 为 Entry（cleanFiles 保存文件），put 的时候 Editor 获取文件，write 写入本地，commit 提交，和 ResourceEncoder（具体写入文件操作）。    
来源：来源不只是服务器（Remote），在设备上（Local）对应目录不属于 glide 管理范围也算来源。
# ARouter
## Jump
1.编译时：通过 APT 封装成目标类信息类 route$xxx.class，对应 compiler，包括 Route、AutoWire、Interceptor 三个解析器。    
2.运行时：将 APT 生成的类添加到集合中，传入 key 与额外信息，并获取对应目标类的信息，结合后进行跳转，本身的 API，包括 Launcher、Frossard、SDK、Logistic Center 四层。   
3.其中，Frossard 内部有 Service（本质是接口，封装一些功能和组件）、Callback 和 Template（编译期生成一些映射文件会按照此模板）。    
4.SDK 包含了 Ware House（存储运行时加载的配置文件和映射关系）、Thread（提供线程池，存在多个拦截器及跳转过程中都是需要异步执行）、Log、Exception 以及 Class 工具（用于解决不同类型 APK 兼容问题）。     
## Advantage
1.直接解析 URL 路由，解析参数并赋值到对应目标字段的页面中。   
2.支持 InstantRun，减少编译的次数，可以简单地将代码修改即时地同步到 APK 中，大规模降低开发复杂度。   
3.允许自定义拦截器，其实就是 AOP 的实现，可以自定义多个拦截器解决一些面向行为编程上出现的问题。    
4.可以提供 IoC 容器。  
5.映射关系自动注册。  
6.提供很多种降级策略供用户自行选择，而原生的路由方案存在无法灵活降级的问题，StartActivity()一旦失败将会抛出运营级异常。
## Foundation
### 注册
1.Interceptor Processor 扫描注解类文件   
2.按照不同种类源文件进行分类  
3.按照固定的命名格式生成映射文件：ARouter$project.name$modole.name  
4.初始化通过固定包名加载映射文件  
### 加载
1.分组加载，ARouter 允许某一个模块下有多个分组，所有的分组最终会被一个 root 节点管理，每个 root结 点都会管理整个模块中的 group 节点，每个 group 结点则包含了该分组下的所有页面，可以按照一定的业务规则或者命名规范把一部分页面聚合成一个分组，每个分组相当于路径中的一段，而每个模块中都会有一个 Interceptor 结点，此外每个模块还会有控制拦截反转的 provider 结点。   
2.按需加载，ARouter 在初始化的时一次性加载所有 root 结点，不加载 Group 结点，降低初始化时加载结点的数量。当分组下某个页面第一次被访问时，整个分组的全部页面都会被加载。  
### 拦截器
拦截器的实现方式与路径注册方式一样，可以将不同功能的拦截器放在不同功能的模块中，只有模块被打包到整个项目中，因为自动注册机制所以拦截器就会生效，涉及到注册和反注册，且每次跳转可能都会生效，太多的拦截器会影响性能。  
### InstantRun
根据 AndroidSDK 和 GradlePlugin 的版本，支持 SplitAPK，从中可以获取所有 Dex 的位置，进而实现映射关系的加载，不支持 SplitAPK 则需要通过运行时的反射拿到 InstantRun 的 SDK 的一个类的 Path，存在静态方法 getDexFileDirectory 来获取 Dex 真实存放位置。
### 依赖注入
1.在编译期，扫出需要自动装配（自动体现在将字段自动地进行赋值而不需要用户手动干预）的字段并注册在映射文件中，跳转时按照预先的配置从 Url 中提取参数，按照类型放入 Intent 中，解决了如何通过 Url 跳转到 Native 页面。  
2.在运行期会在目标页面进行初始化的时候调用 ARouter.inject(this)，将自身的实例传递进去。ARouter 会查找到编译期为调用方生成的注入辅助类（比方在编译期扫描到一个页面需要进行自动装配，此时就会为页面生成一个注入辅助类，在运行的时候调用注入辅助类的方法对于字段进行赋值，这其实就是模拟用户对于字段进行赋值），在实例化辅助类之后，调用其中的 inject 方法完成对于字段的赋值。
# Dagger2
@Moduel 表明类类型  
@Provides 创建功能对象  
@Component（modules = {XXModule.class}）接收数组（装入容器的 module），注解接口类，类中方法表示容器中功能对象的归属类  
@Inject 注入功能对象，通过 DaggerXXX（定义的 component类（接口））.create.（component 声明的方法）生成功能对象  
编译生成 DaggerXXX：采用建造者模式，重写 inject 方法完成依赖注入（通过生成的 XXX（inject 传入的参数）_MembersInjector 类真正完成依赖注入（对象传递给 XXX），根据标记的 provide 会生成 XXX_YYYFactory 类，调用声明的 provide 方法创建对象），维护module，build创建（赋值）module  
对于复用模块的扩展有好处  
# LeakCanary
定义 contentprovider 进行初始化，onCreate 中调用 AppWatcher.manualInstall(application)  
监听对象为：
Activity -> onDestroy  
Fragment -> onFragmentDestroyed  
Fragment View -> onFragmentViewDestroyed  
ViewModel -> onCleared  
实现：  
1.对象回收时,生成唯一 key，封装进 KeyedWeakReference ，并传入自定义 ReferenceQueue。  
2.将 key 和 KeyedWeakReference 放入一个 map 中。  
3.默认 5 秒后主动触发 GC,将自定义 ReferenceQueue 中的 KeyedWeakReference 全部移除(所引用对象已回收)，同时根据 KeyedWeakReference 的 ke y将 map 中的 KeyedWeakReference 也移除。  
4.若 map 中还有 KeyedWeakReference 剩余没有入队,对应的对象没回收即为产生了内存泄露。
5.分析内存泄露对象引用链,保存数据。
## Activity
Application.registerActivityLifecycleCallbacks 注册生命周期监听，在 onActivityDestroyed 进行 watcher.watch 检测是否泄露
## Fragment
fragment.registerLifecycleCallbacks 注册监听，在 onFragmentDestroyed 以及 onFragmentViewDestroyed 中 检测是否泄露  
策略模式封装三种环境下的流程，主要 FragmentManager 获取不同
Android O以上：activity.getFragmentManager(FragmentDestroyWatcher)  
AndroidX：activity.getSupportFragmentManager(FragmentDestroyWatcher)  
support包：通过 activity.getSupportFragmentManager 获得(AndroidSupportFragmentDestroyWatcher)
获取 FragmentManager 需要 Activity 实例,所以还要监听 Activity 生命周期，onActivityCreated 中拿到 Activity 实例，从而拿到 FragmentManager
## ViewModel
AndroidXFragmentDestroyWatcher 中还会单独监听 onFragmentCreated，在此对 ViewModel 进行监听：ViewModelClearedWatcher.install，通过传入 provider 和 fragment 创建 watcher，一个新的 viewModel，反射获取这个 fragment 里的每个ViewModel ，在 onCleared 里检测是否存在泄漏
