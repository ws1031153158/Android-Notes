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
# Dagger2
# LeakCanary
## Activity
## Fragment
## ViewModel
