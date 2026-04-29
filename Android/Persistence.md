# SharedPreference
1.以键值对形式存储数据（存为 xml 文件），一般用来存放各种参数（如 boolean 值、字符串等），记录某种状态或参数数值  
2.apply 没有返回值，提交到内存，然后异步提交到硬盘，并覆盖上一次内存的值  
3.commit 返回值标识是否成功，同步提交到硬盘
## ANR
sp 只适合轻量级数据的存储，适合少量数据的持久化，存在 I/O 瓶颈，每次写入都是整个文件重新写入，不是增量写入，如果读写操作慢，就有可能导致 ANR   
1.在 UI 线程中调用 getXXX 或 edit() 方法 （第一次调用 getSharedPreferences() 后）   
2.用 apply 方法提交修改（内部是用一个单线程的线程池实现），当 Activity 的 onPause/onStop 等方法被调用时    
3.在 UI 线程调用 commit 方法（写操作是在调用线程中执行）  
## DataStore
基于 Kotlin 协程 + Flow，替代 SharedPreferences。  
### Preferences DataStore
键值对存储，不需要定义 Schema  

创建：  
```
// 推荐在顶层创建，避免重复实例化
val Context.dataStore: DataStore<Preferences> 
    by preferencesDataStore(name = "settings")
```

定义Key：  
```
object DataStoreKeys {
    val TOKEN = stringPreferencesKey("token")
    val USER_ID = intPreferencesKey("user_id")
    val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
    val TOTAL_ASSETS = doublePreferencesKey("total_assets")
}

// Key 类型对应：
// stringPreferencesKey    → String
// intPreferencesKey       → Int
// booleanPreferencesKey   → Boolean
// floatPreferencesKey     → Float
// doublePreferencesKey    → Double
// longPreferencesKey      → Long
// stringSetPreferencesKey → Set<String>
```

写入：  
```
// 必须在协程里调用
suspend fun saveToken(context: Context, token: String) {
    context.dataStore.edit { preferences ->
        preferences[DataStoreKeys.TOKEN] = token
    }
}
```

读取：  
```
// 返回 Flow，自动感知变化
fun getToken(context: Context): Flow<String?> {
    return context.dataStore.data.map { preferences ->
        preferences[DataStoreKeys.TOKEN]
    }
}

// 只读取一次（不需要监听变化）
suspend fun getTokenOnce(context: Context): String? {
    return context.dataStore.data.first()[DataStoreKeys.TOKEN]
}
```

删除：  
```
// 删除单个 key
suspend fun removeToken(context: Context) {
    context.dataStore.edit { preferences ->
        preferences.remove(DataStoreKeys.TOKEN)
    }
}

// 清空所有数据
suspend fun clearAll(context: Context) {
    context.dataStore.edit { preferences ->
        preferences.clear()
    }
}
```
#### 对比
```
SharedPreferences 的问题：
// 同步写入，阻塞主线程
sp.edit().putString("token", token).commit()

// 异步写入，但异常无法捕获
sp.edit().putString("token", token).apply()

// 不支持协程
// 不是类型安全（任何 key 都是 String）
// 多进程不安全


DataStore 的优势：
// 完全异步，基于协程的 Mutex 互斥锁
suspend fun save() {
    dataStore.edit { it[KEY] = value }
}

// 基于 Flow，自动感知变化
val data: Flow<String> = dataStore.data.map { it[KEY] }

// 异常可以捕获
dataStore.data
    .catch { e ->
        if (e is IOException) emit(emptyPreferences())
        else throw e
    }
    .map { it[KEY] }

// 类型安全（Key 有类型）
val KEY = stringPreferencesKey("token") // 只能存 String
```
#### 劣势：
只适合用户设置/Token/简单配置  
1.大量数据/复杂查询用 ROOM：   
存储结构：  
底层是一个文件，所有键值对存在同一个 Preferences 文件里。  

读取机制：  
每次读取都是读整个文件，反序列化整个 Preferences 对象，然后再取你要的那个 key（没有查询能力，只能过滤 map）。  

数据量大 → 每次读取都要读取整个文件，反序列化所有数据，内存占用大，速度慢。  

2.需要频繁高速读写用 MMKV：  
写入机制：  
每次 edit { } 都会：  
    1. 加锁（Mutex）  
    2. 读取当前文件  
    3. 修改数据  
    4. 序列化  
    5. 写入文件  
    6. 释放锁  
每次写入都是完整的文件 IO 操作
### Prot DataStore
强类型存储，需要定义 .proto 文件，类型安全  
## MMKV
腾讯开源的高性能键值存储框架，基于 mmap 内存映射实现。  
### 读写
```
普通文件读写：
App → 系统调用 → 内核缓冲区 → 磁盘
App ← 系统调用 ← 内核缓冲区 ← 磁盘
每次都要经过内核，有切换开销

mmap 内存映射：
把磁盘文件直接映射到内存地址空间
App 操作内存 = 操作文件
省去了系统调用的开销
由操作系统负责内存和磁盘同步
操作系统保证内存和磁盘同步，即使 App crash，操作系统也会完成同步
额外增加了 CRC 校验，检测到数据损坏时自动恢复
不支持直接存复杂对象（需要序列化）

类比：
普通读写：你要一本书
    去图书馆（内核）
    借出来（拷贝到用户空间）
    看完还回去

mmap：图书馆把书架
    直接搬到你家
    你直接在书架上看/写
    图书馆自动同步
```
### 基本用法
```
// 获取默认实例
val mmkv = MMKV.defaultMMKV()

// 写入
mmkv.encode("token", "abc123")
mmkv.encode("user_id", 123)
mmkv.encode("is_logged_in", true)
mmkv.encode("total_assets", 100000.0)

// 读取（第二个参数是默认值）
val token = mmkv.decodeString("token", "")
val userId = mmkv.decodeInt("user_id", -1)
val isLoggedIn = mmkv.decodeBool("is_logged_in", false)
val totalAssets = mmkv.decodeDouble("total_assets", 0.0)

// 删除
mmkv.removeValueForKey("token")
mmkv.removeValuesForKeys(arrayOf("token", "user_id"))

// 清空
mmkv.clearAll()

// 判断是否存在
mmkv.containsKey("token")

// 多实例
// 不同业务用不同实例，数据隔离
val userMMKV = MMKV.mmkvWithID("user_data")
val settingsMMKV = MMKV.mmkvWithID("settings")
val cacheMMKV = MMKV.mmkvWithID("cache")

userMMKV.encode("token", "abc123")
settingsMMKV.encode("dark_mode", true)

//多进程支持
// 多进程场景（如有 Service 在独立进程）
// 通过文件锁保证安全，性能差
val mmkv = MMKV.mmkvWithID(
    "shared_data",
    MMKV.MULTI_PROCESS_MODE
)

// 加密
// 支持 AES 加密存储
val mmkv = MMKV.mmkvWithID(
    "secure_data",
    MMKV.SINGLE_PROCESS_MODE,
    "your_encrypt_key"  // 加密密钥
)

// 封装
object MMKVManager {

    private val userMMKV by lazy {
        MMKV.mmkvWithID("user")
    }

    private val settingsMMKV by lazy {
        MMKV.mmkvWithID("settings")
    }

    // Token 相关
    var token: String
        get() = userMMKV.decodeString("token", "") ?: ""
        set(value) = userMMKV.encode("token", value).let {}

    // 用户 ID
    var userId: Int
        get() = userMMKV.decodeInt("user_id", -1)
        set(value) { userMMKV.encode("user_id", value) }

    // 深色模式
    var darkMode: Boolean
        get() = settingsMMKV.decodeBool("dark_mode", false)
        set(value) { settingsMMKV.encode("dark_mode", value) }

    // 清除用户数据（退出登录）
    fun clearUser() {
        userMMKV.clearAll()
    }
}

// 使用
MMKVManager.token = "abc123"
val token = MMKVManager.token

// 从 sp 迁移
// MMKV 提供了一键迁移方法
val oldSP = context.getSharedPreferences(
    "old_data", Context.MODE_PRIVATE
)
val mmkv = MMKV.mmkvWithID("new_data")

// 一行代码迁移所有数据
mmkv.importFromSharedPreferences(oldSP)

// 迁移完清除旧数据
oldSP.edit().clear().apply()
```
# SQLite
原生Android内置，关系型数据库，数据拆分为相关的数据表。  
直接写SQL，一般通过 contentProvider 进行增删改查，重写 onUpgrade 进行数据库升降级操作。
## ContentValues
ContentValues是一个用于存储一组值的类，ContentResolver 可以处理这些值。  
在数据库操作中，ContentValues 常用于添加数据，其中 Key 是字段名称，Value 是字段的值。类似于一个 Map（构造函数创建一个 map，用于存取数据），但只能存储基本类型的数据，如 string、int 等，不能存储对象。  
# File
涉及到 IO，效率低，此外由于权限问题安全性也较差。一般用来存储音频或图片等占用空间较大文件。
# Cache
## LruCache
### Class
LruCache 针对内存缓存  
DisLruCache 充当存储设备缓存，将缓存写入 File，通过一个 editor 对象完成添加操作
NetWork 从后端请求数据
# ROOM
