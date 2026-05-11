
# 电量优化	
## WakeLock	
### 非必要 WakeLock 释放
### 超时保护
## 后台任务	
### JobScheduler/WorkManager 合并任务
### Doze 模式适配
## 定位优化	
### 按需请求定位
### 降低定位频率
### Geofencing 替代持续定位
## 传感器	
### 及时注销传感器监听
### 降低采样率
## 网络电量	
### 减少轮询
### Push 替代 Pull
### 批量网络请求
## 渲染电量	
### 降低帧率（非交互场景）
### 深色模式（OLED 省电）
## Battery Historian	
### 电量分析工具使用
### 耗电归因
# 存储优化	
## SharedPreferences	
sharedPreferences 是一个 xml 的读取和存储操作，在使用前都会调用 getSharedPreferences 方法，这时它会去异步加载文件当中的配置文件，load 到内存当中，调用 get 或 put 属性时，如果 load 内存的操作没有执行完成，那么就会一直阻塞进行等待，都是拿同一把锁，它既然是 IO 操作，如果这文件存在很久，这个时间就会很长,如果项目比较大，有几十个类使用 SharedPreferences 文件，里面的文件也非常多   
1.在 Application 中 MultiDex 之前加载 SharedPreferences（如果其他类在 Multidex 之前加载进行操作，会因为一些类不在主 dex 当中，导致崩溃，Sharedpreferences 是系统类，不会报错）   
2.创建 SharedPreferences 并且保存到 Map 中，那么需要的时候可以在 SP_MAP 中直接获取
### ANR 风险
### MMKV/DataStore 替代方案
## 数据库优化	
### SQLite 索引
### 事务批量操作
### Room 查询优化
### WAL 模式
## 文件 IO	
### 异步 IO
### BufferedStream
### 内存映射文件（MappedByteBuffer）
## 序列化优化	
### Protobuf/FlatBuffers 替代 Java 序列化
## 磁盘缓存	
### 缓存分级
### 缓存淘汰策略
### 缓存大小控制
# 并发优化
## 线程池管理	
### 统一线程池
### 避免线程无限创建
### 核心线程数配置
## 协程优化	
### Kotlin Coroutines 调度器选择
### 结构化并发、Flow 背压
## 线程泄漏	
### 未终止线程检测
### 线程生命周期管理
## 死锁检测	
### 锁顺序规范
### DeadlockDetector
### synchronized 优化
### 锁粒度控制
## 异步框架	
### RxJava 线程切换
### LiveData/Flow 线程安全
# 错误处理
## 错误分类
1.网络错误:    
无网络连接  
请求超时  
DNS 解析失败  

2.HTTP错误:  
401 未登录/Token 过期  
403 无权限  
404 资源不存在  
500 服务器错误  


3.业务错误:  
success: false  
服务器返回的业务异常  

4.本地错误:  
数据解析失败  
本地数据库异常
## 统一处理
统一错误模型：  
```
sealed class AppError {
    // 网络错误
    data object NoNetwork : AppError()
    data object Timeout : AppError()

    // HTTP 错误
    data object Unauthorized : AppError()   // 401
    data object Forbidden : AppError()      // 403
    data class HttpError(
        val code: Int,
        val message: String
    ) : AppError()

    // 业务错误
    data class BusinessError(
        val message: String
    ) : AppError()

    // 未知错误
    data class Unknown(
        val message: String
    ) : AppError()
}
```

1.OkHttp 拦截器捕获网络层错误  
```
class ErrorInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = try {
            chain.proceed(chain.request())
        } catch (e: Exception) {
            // 网络层异常转换
            throw when (e) {
                is UnknownHostException -> NoNetworkException()
                is SocketTimeoutException -> TimeoutException()
                else -> e
            }
        }

        // HTTP 错误码处理
        when (response.code) {
            401 -> throw UnauthorizedException()
            403 -> throw ForbiddenException()
            500 -> throw ServerException()
        }

        return response
    }
}
```

2.Repository 层统一转换为 AppError  
```
suspend fun getWatchlist(): Result<List<WatchlistItem>> {
    return runCatching {
        api.getWatchlist(bearerToken())
    }.mapError { e ->
        // 统一转换为 AppError
        when (e) {
            is NoNetworkException -> AppError.NoNetwork
            is TimeoutException -> AppError.Timeout
            is UnauthorizedException -> AppError.Unauthorized
            else -> AppError.Unknown(e.message ?: "未知错误")
        }
    }
}
```

3.ViewModel 层根据错误类型差异化处理  
```
fun loadWatchlist() {
    viewModelScope.launch {
        repository.getWatchlist()
            .onSuccess { items ->
                _uiState.update {
                    it.copy(items = items, isLoading = false)
                }
            }
            .onFailure { error ->
                when (error) {
                    is AppError.NoNetwork -> {
                        // 加载本地缓存
                        loadFromLocal()
                        _uiState.update {
                            it.copy(
                                errorMessage = "网络不可用，显示缓存数据",
                                isOffline = true
                            )
                        }
                    }
                    is AppError.Unauthorized -> {
                        // Token 过期，跳转登录
                        _uiState.update {
                            it.copy(navigateToLogin = true)
                        }
                    }
                    is AppError.Timeout -> {
                        // 超时重试
                        retryLoad()
                    }
                    else -> {
                        _uiState.update {
                            it.copy(errorMessage = error.message)
                        }
                    }
                }
            }
    }
}
```
## 用户体验
1.无网络 → 显示缓存数据 + 离线提示  
```
// 不同错误展示不同 UI
when {
    uiState.isOffline -> {
        // 顶部显示离线提示条
        OfflineBanner()
    }
    uiState.errorMessage != null -> {
        // Snackbar 提示
        LaunchedEffect(uiState.errorMessage) {
            snackbarHostState.showSnackbar(
                message = uiState.errorMessage,
                actionLabel = "重试"
            )
        }
    }
    uiState.items.isEmpty() && !uiState.isLoading -> {
        // 空状态页
        EmptyState(onRetry = vm::loadWatchlist)
    }
}
```

2.Token过期 → 自动跳转登录   
3.超时 → 自动重试  
```
// Retrofit 配置重试
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(RetryInterceptor(maxRetry = 3))
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(10, TimeUnit.SECONDS)
    .build()

class RetryInterceptor(private val maxRetry: Int) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var retryCount = 0
        var response: Response
        do {
            response = chain.proceed(chain.request())
            retryCount++
        } while (!response.isSuccessful && retryCount < maxRetry)
        return response
    }
}
```

4.业务错误 → Snackbar 提示具体原因
## 监控运维
1.客户端上报错误日志  
2.结合后端监控告警  
3.形成完整的错误闭环

# Pending
```
# 网络优化	
## 请求优化	
### 请求合并
### 批量上报
### 接口聚合（BFF）
## 协议优化	
### HTTP/2 多路复用
### QUIC/HTTP3
### Protobuf 替代 JSON
## 缓存策略	
### 强缓存/协商缓存
### 离线缓存、预请求
## 连接优化	
### 连接池复用
### DNS 预解析、IP 直连
## 弱网优化	
### 超时重试策略
### 断点续传
### 网络质量感知
## 安全优化	
### HTTPS 证书校验
### SSL Pinning
### 数据加密
## 网络监控	
### 请求耗时
### 成功率
### 流量统计
### OkHttp Interceptor
```
