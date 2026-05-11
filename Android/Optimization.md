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
