# SharedPreference
1.以键值对形式存储数据（存为 xml 文件），一般用来存放各种参数（如 boolean 值、字符串等），记录某种状态或参数数值  
2.apply 没有返回值，提交到内存，然后异步提交到硬盘，并覆盖上一次内存的值  
3.commit 返回值标识是否成功，同步提交到硬盘
## ANR
sp 只适合轻量级数据的存储，适合少量数据的持久化，存在 I/O 瓶颈，每次写入都是整个文件重新写入，不是增量写入，如果读写操作慢，就有可能导致 ANR   
1.在 UI 线程中调用 getXXX 或 edit() 方法 （第一次调用 getSharedPreferences() 后）   
2.用 apply 方法提交修改（内部是用一个单线程的线程池实现），当 Activity 的 onPause/onStop 等方法被调用时    
3.在 UI 线程调用 commit 方法（写操作是在调用线程中执行）   
# SQLite
关系型数据库，数据拆分为相关的数据表。一般通过 contentProvider 进行增删改查，重写 onUpgrade 进行数据库升降级操作。
# File
涉及到 IO，效率低，此外由于权限问题安全性也较差。一般用来存储音频或图片等占用空间较大文件。
# Cache
## LruCache
### Class
LruCache 针对内存缓存  
DisLruCache 充当存储设备缓存，将缓存写入 File，通过一个 editor 对象完成添加操作
