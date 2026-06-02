# Linux
```
Linux进程隔离：
每个进程有独立的虚拟地址空间
进程A的地址 0x1000 和 进程B的地址 0x1000
指向的是完全不同的物理内存

进程A无法直接读写进程B的内存
这是操作系统的安全基石

所以跨进程通信必须借助内核
```

内存模型：  
```
物理内存（RAM）
┌─────────────────────────────────────┐
│  物理页帧0  │  物理页帧1  │  ...    │
└─────────────────────────────────────┘
       ↑              ↑
       │映射           │映射
┌──────────────┐  ┌──────────────┐
│  进程A       │  │  进程B       │
│  虚拟地址空间 │  │  虚拟地址空间 │
│              │  │              │
│  0x0000      │  │  0x0000      │
│  ...         │  │  ...         │
│  用户空间    │  │  用户空间    │
│  (0~3GB)     │  │  (0~3GB)     │
├──────────────┤  ├──────────────┤
│  内核空间    │  │  内核空间    │
│  (3~4GB)     │  │  (3~4GB)     │
└──────────────┘  └──────────────┘
        ↑                ↑
        └────────────────┘
         内核空间在所有进程
         中映射同一块物理内存
```

虚拟地址/物理地址转换：  
```
虚拟地址
    │
    ▼
MMU（内存管理单元）
    │
    ▼  查询页表（Page Table）
    │  页表存储：虚拟页号 → 物理页帧号
    │
    ▼
物理地址

页表是每个进程独有的
所以同一个虚拟地址在不同进程
映射到不同的物理地址
```

mmap:  
```
// 让用户空间的虚拟地址 直接映射到 内核的物理页帧

步骤1：调用 mmap()
  └── 内核在用户虚拟地址空间
      分配一段虚拟地址区间（VMA）
  └── 此时只是建立映射关系
      并没有真正分配物理内存

步骤2：首次访问该虚拟地址
  └── 触发缺页异常（Page Fault）
  └── 内核处理缺页异常：
      分配物理页帧
      建立 虚拟地址→物理地址 的页表项
  └── 之后访问该地址
      直接读写物理内存，无需拷贝

步骤3：用户空间读写
  └── 就像操作普通内存一样
  └── 实际上直接操作的是
      和内核共享的物理页帧

用户虚拟地址空间          物理内存
┌──────────────┐         ┌──────────────┐
│              │         │              │
│  mmap区域    │────────▶│  物理页帧    │
│  0xA000      │  页表   │  (实际数据)  │
│              │  映射   │              │
└──────────────┘         └──────────────┘
                                ↑
                         内核虚拟地址空间
                         ┌──────────────┐
                         │  内核缓冲区  │────┘
                         │  也映射到    │  同一块
                         │  同一物理页  │  物理内存
                         └──────────────┘

结果：
用户空间写入 0xA000
= 直接修改了物理页帧
= 内核缓冲区也看到了变化
零拷贝！
```
# Serialize
序列化：对象将其状态写入临时（传输时使用）或持久性（持久化处理）存储区域  
反序列化：重新创建该对象，通过 FileOutPutStream/FileInPutStream 以及 ObjectOutPutStream/ObjectInPutStream 生成和读取二进制文件
## Serializable
1.编译器会根据类字段自动生成，也可以手动重写来生成 seriaVersionUID，这样即使类结构变化仍可以序列化  
2.通过 FileOutPutStream 创建 ObjectOutPutStream，创建 ObjectStreamClass 并写入对象信息，类名以及 seriaVersionUID，最后写入对象需要反射解析要序列化的对象来生成 ObjectStreamClass，所以性能不太好，但是写起来方便点    
3.加上 transition 关键字的属性不会序列化  
4.产生大量临时对象（GC压力）  
5.适合：持久化存储（写文件/数据库）
## Parcelable
1.重写 writeToParcel（用来序列化，将需要序列化的字段写入一个 parcel 对象） 并创建一个 Creator 重写 createFromParcel（反序列化，从 parcel 对象读取数据）  
2.Intent 使用的就是 Parcelable，如果传入的数据采用 Serializable 则会进行二次序列化（先序列化为字节数组再写入），所以尽量先统一一下  
3.不产生额外对象  
4.适合：内存中的IPC传输
## Parcel
1.Binder使用的就是这个，直接操作内存（native层），性能好    
2.只有这个支持Binder对象传递（跨进程传递Binder引用）  
3.生命周期需要手动管理（recycle()）  
4.
# Binder
## Foundation
只需要拷贝一次数据，需要内核支持，采用 Linux 内存映射（mmap，常用在文件等对象的操作，用内存读写代替 IO，在物理介质和用户空间建立映射）方式，将用户空间的内存区域映射到内核空间，两个空间变化都会反应到另一空间，这样可以减少拷贝次数。    

```
映射完成后的内存布局：

Server进程虚拟空间        物理内存          内核虚拟空间
┌──────────────┐         ┌──────────────┐  ┌──────────────┐
│              │         │              │  │              │
│  mmap区域   │────────▶│  Binder      │◀─│  内核        │
│  (用户空间) │  页表A  │  物理缓冲区  │  │  缓冲区      │
│  只读映射   │         │              │  │  (可读写)    │
│             │         │              │  │              │
└──────────────┘         └──────────────┘  └──────────────┘

Server用户空间 和 内核空间
指向同一块物理内存！
```

Binder 作为独立模块，通过内存映射，运行时被链接到内核作为内核的一部分运行（Linux 动态内核可加载模块）
## 拷贝
```
普通 IPC（如 Socket、管道）
两次拷贝：
发送进程 → 内核缓冲区（用户态 → 内核态）
内核缓冲区 → 接收进程（内核态 → 用户态）

Binder 优化：
Binder 驱动在内核中为 接收进程 预先分配一块连续的内存（Binder Buffer）
发送数据时：
发送进程将数据 一次性拷贝 到这块内核缓冲区。
接收进程通过内存映射（mmap）直接访问这块缓冲区，不需要再拷贝到用户空间。
所以是 一次拷贝：发送进程用户空间 → 接收进程的内核缓冲区

无法0拷贝：
1.安全隔离
Linux 进程地址空间隔离，用户态不能直接访问其他进程的内存
如果让发送进程直接映射接收进程的缓冲区：
发送方可以在任何时间修改接收方的数据（数据篡改风险）
无法保证数据一致性（接收方还没读完，发送方就改了）

2.生命周期管理
Binder Buffer 属于接收进程，由 Binder 驱动分配和回收
如果发送进程直接写这块内存，驱动无法精确控制数据何时写完、何时可读、何时释放
可能出现：
发送方写到一半崩溃 → 接收方读到脏数据
接收方释放缓冲区 → 发送方还在写（悬空指针）

3. 数据一致性与同步
0 拷贝需要复杂的同步机制（锁、信号量）来保证写入和读取的时序
Binder 的设计目标是“消息传递 + 内存安全”，不是共享内存
如果要 0 拷贝，应该用 Ashmem / SharedMemory 这种共享内存机制，而不是 Binder
```
## Connection
1.Binder 驱动在内核空间开辟出一个缓存区，建立缓存区和用户空间地址的映射关系  
2.发送进程将数据 copy 到内核中的缓存区，映射的存在相当于把数据发送到了接收进程的用户空间
## Space
1.C & S 为用户空间，S 创建 Binder 并命名，将 Binder 通过驱动传递给 ServiceManager，让 manager 注册该 binder。同时，驱动为该 binder 在内核创建一个对象以及对 manager 和这个对象的引用，将 binder 名和引用传递给 manager  
2.ServiceManage 也为用户空间，提供对 binder 的解析，将 binder 字符名转换为引用，使 C 可以通过 binder 名获取到对题诗的引用。与其他进程的通信也是通过 binder，自身维护一个 binder，没有名也无需注册（注册为 manager 时，驱动自动创建 binder 对象），其他进程通过该 binder 引用来注册。
3.Driver 属于内核空间，负责进程间 binder 通信的建立，binder 的传递，binder 引用的管理等。进程获取另一进程的 binder 对象获取到的其实时一个代理，本身不具有原对象的属性和方法，是通过驱动调用原进程对象的方法和获取属性，返回给该进程。（驱动会维护一个表，对象和代理对象一对一）  
4.Binder Pool 会将多个 binder 请求统一发送，避免 S 端重复创建 service（通过 queryBinder 返回不同 C 端的 binder 对象）
## Binder Pool
```
Server端不是单线程处理：
Binder线程本质上就是普通的Linux线程
只不过它：

1. 阻塞在 ioctl(BINDER_WRITE_READ) 上等待请求
2. 有请求来了被驱动唤醒
3. 处理完请求后重新阻塞等待
4. 由Binder驱动统一调度管理

由 ProcessState / IPCThreadState 管理

Server进程
┌─────────────────────────────────┐
│  Binder线程池（默认最多16线程） │
│                                 │
│  Thread-1  等待请求中...        │
│  Thread-2  处理Client-A的请求  │
│  Thread-3  处理Client-B的请求  │
│  ...                            │
└─────────────────────────────────┘

线程管理：
├── 主线程调用 startThreadPool()
├── 驱动根据负载动态创建线程
├── 最大线程数：15+1=16
└── 空闲线程阻塞在 ioctl 等待

Client调用时：
├── 同步调用：Client线程阻塞等待
├── 异步调用（oneway）：Client立即返回
└── 死亡通知：linkToDeath 监听Server死亡
```

流程：  
```
系统状态：
MediaPlayerService 已启动
线程池初始状态：主线程 + Binder-Thread-1 共2个线程

时间线：

t=0ms
  App-A（音乐播放器）调用 mediaPlayer.prepare()
  └── 这是同步Binder调用
  └── App-A主线程发起请求，挂起等待

  Binder驱动：
  └── 有空闲线程（Binder-Thread-1）
  └── 唤醒 Binder-Thread-1 处理请求

t=1ms
  Binder-Thread-1 开始处理 App-A 的 prepare()
  └── 需要解析音频文件，耗时50ms
  └── 此时线程池：主线程空闲，Thread-1忙碌

t=5ms
  App-B（视频播放器）调用 mediaPlayer.prepare()
  └── Binder驱动：主线程空闲
  └── 唤醒主线程处理 App-B 的请求

t=6ms
  主线程 开始处理 App-B 的 prepare()
  └── 也需要50ms
  └── 此时线程池：主线程忙碌，Thread-1忙碌
  └── 空闲线程数 = 0！

t=10ms
  App-C（直播App）调用 mediaPlayer.setDataSource()
  └── Binder驱动：没有空闲线程！
  └── 发送 BR_SPAWN_LOOPER 给 MediaPlayerService
  └── MediaPlayerService 创建 Binder-Thread-2

  Binder-Thread-2 创建完成
  └── 调用 joinThreadPool() 加入线程池
  └── 立即被分配处理 App-C 的请求

t=51ms
  Binder-Thread-1 处理完 App-A 的请求
  └── 返回结果给 App-A
  └── App-A 主线程被唤醒，prepare() 返回
  └── Thread-1 重新进入等待状态

最终线程池状态：
主线程 + Thread-1 + Thread-2 = 3个线程
根据后续负载决定是否继续创建
```

等待唤醒：  
```
而是基于 Linux 等待队列（wait_queue）：

Binder线程
    │
    ▼
ioctl(BINDER_WRITE_READ)
    │
    ▼  陷入内核态
binder_thread_read()
    │
    ▼  检查是否有待处理请求
    │  没有请求
    ▼
wait_event_interruptible(thread->wait, ...)
    │
    ▼  线程进入 TASK_INTERRUPTIBLE 状态
       CPU不再调度此线程
       完全不占用CPU资源！

有新请求到来时：
    │
    ▼
binder_transaction()
    │
    ▼
wake_up_interruptible(&thread->wait)
    │
    ▼  线程状态变为 TASK_RUNNING
    ▼  等待CPU调度
    ▼  被调度后从 wait_event 返回
    ▼  继续处理请求
```

oneway串行异步：  
```
同一个Client对同一个Server的oneway调用
是串行的！不会并发！

Client连续发3个oneway请求：
    请求1 ──▶ 进入Server的异步队列
    请求2 ──▶ 进入Server的异步队列
    请求3 ──▶ 进入Server的异步队列

Server处理：
    处理请求1 → 处理请求2 → 处理请求3
    严格按顺序，不并发
```
## oneway
AIDL 接口方法声明前的关键字，表示这个方法是 单向异步调用  
调用方（Client）不会等待 服务端（Server）执行完成，立即返回  
非阻塞，对应的是 Binder 的 ASYNC_TRANSACTION 模式  

注意：  
不能有返回值（必须是 void 方法）  
不能抛异常（AIDL 语法限制）  
参数会立即拷贝到 Binder 驱动缓冲区，然后方法返回  
服务端方法会在 Binder 线程池中异步执行  

```
// 定义
interface IMyService {
    oneway void sendLog(String log);
}

// 编译生成
public interface IMyService extends android.os.IInterface {
    public void sendLog(String log) throws android.os.RemoteException;
}

// 调用方
myService.sendLog("User clicked button"); // 立即返回

// 服务端
@Override
public void sendLog(String log) {
    // 异步执行
    Log.d("IMyService", "Log: " + log);
}
```

不适合：  
1.需要返回结果  
2.需要保证顺序（异步可能乱序）

原理：  
Binder 默认是同步调用（Client 线程阻塞，等待 Server 处理完成并返回结果）  
异步队列：oneway 事务进入 async_todo，普通事务进入 todo  
线程池：Server 进程的 Binder 线程池会同时处理同步和异步事务，但异步事务不会阻塞调用方  
## 完整流程
1.准备阶段（Server启动时）：  
```
Server进程
    │
    ├── open("/dev/binder")  打开驱动
    │
    ├── mmap(1MB)  建立内核缓冲区映射
    │   └── Server用户空间虚拟地址
    │       映射到 Binder内核缓冲区
    │       的同一块物理内存
    │
    └── ioctl(BINDER_WRITE_READ) 进入循环
        等待Client请求
```

2.Client发起调用:  
```
Client进程（用户空间）
    │
    │  proxy.someMethod(arg)
    │
    ▼
Parcel data
    │  把参数序列化写入 Parcel
    │  data.writeString("hello")
    │  data.writeInt(123)
    │
    ▼
ioctl(fd, BINDER_WRITE_READ, &bwr)
    │  bwr包含：
    │  write_buffer → 指向Parcel数据
    │  write_size   → 数据大小
    │
    │  系统调用，陷入内核态
    ▼
```

3.Binder驱动处理（内核态）:  
```
Binder驱动
    │
    ├── 1. 接收Client的数据
    │      从Client用户空间
    │      拷贝数据到内核缓冲区
    │      ← 这是唯一的一次copy_from_user！
    │
    ├── 2. 找到目标Server
    │      根据Binder句柄找到Server进程
    │      找到Server进程的mmap缓冲区
    │
    ├── 3. 建立映射（零拷贝关键）
    │      在Server的页表中
    │      添加新的映射条目：
    │      Server用户虚拟地址 → 刚才的内核物理页
    │      
    │      ┌─────────────────────────────────┐
    │      │  内核物理缓冲区（已有Client数据）│
    │      └─────────────────────────────────┘
    │              ↑                ↑
    │         内核虚拟地址      Server用户虚拟地址
    │         （已映射）         （新建映射）
    │
    ├── 4. 唤醒Server进程
    │      通过等待队列唤醒
    │      Server的阻塞线程
    │
    └── 5. Client进入等待
           当前线程挂起
           等待Server处理完成
```

4.Server处理:  
```
Server进程被唤醒
    │
    ├── 从mmap区域读取数据
    │   └── 直接读取虚拟地址
    │       实际访问的是
    │       和内核共享的物理页
    │       无需任何拷贝！
    │
    ├── Stub.onTransact()
    │   └── 反序列化参数
    │   └── 调用实际方法
    │   └── 得到返回值
    │
    └── 把返回值写回
        ioctl(BINDER_WRITE_READ)
        把reply数据传回驱动
```

5.返回结果:  
```
Binder驱动
    │
    ├── 收到Server的reply
    │
    ├── copy_from_user（第二次拷贝，reply数据）
    │   把reply从Server用户空间拷贝到内核
    │
    ├── copy_to_user（拷贝到Client）
    │   把reply从内核拷贝到Client用户空间
    │
    └── 唤醒Client线程

Client进程
    └── 读取reply Parcel
    └── 反序列化返回值
    └── 方法调用返回
```
# AIDL
## Foundation
底层为 Binder，支持一对多和实时通信，可以处理并发场景，需要处理好线程安全问题
## Server
创建 service 并创建 AIDL（需要定义参数 in/out/inout，代表输入/输出/输入输出数据，指定数据流向，如 in 是 C 到 S） 和它的实现类，C 端获取 IBinder 转换给 AIDL 来调用方法
## Client
绑定 service，重写 onServiceConnection，创建 binder 对象（AIDL.Stub.asInterface）  
```
    private var downloadService: IDownloadService? = null
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            // 获取代理对象
            downloadService = IDownloadService.Stub.asInterface(service)
            //  asInterface 判断：
            //  同进程 → 直接返回Stub对象（无Binder开销）
            //  跨进程 → 返回Proxy对象（走Binder）
        }
        override fun onServiceDisconnected(name: ComponentName) {
            downloadService = null
        }
    }
```
## Tips
C 和 S 持有不同的对象，多次 IPC 过程 C 传给 S 同一个对象，binder 会生成不同的多个对象，但底层的 bidner 对象都是一个
# Messenger
实现了 Parcelabel 接口，通过自定义 service 传递 target（handler，重写 handleMessage）实现 IPC，支持一对多和实时通信，是对 AIDL 的封装，通过 Message 传输所以只支持 Bundle 支持的类型，不适用于高并发场景（一次只处理一个请求）  
C 端 bindService，S 端返回 IBinder（handler 创建的 messenger 的 binder 对象），通过 messenger 发送 message
# Bundle
实现了 Parcelabel 接口，包含一些数据，传入到 intent 中，内部采用 HashMap 存储数据，只支持基本数据类型以及实现了 Serializable 或 parcalizable 的类型
# File
文件共享，安全性很差，涉及到 IO 操作，效率很低
# Socket
主要用于本机进程间和跨网络的低速通信，可以跨端通信，性能稍高，但需要考虑网络延迟和带宽问题，需要网络权限，存在安全问题  
也是一种 C/S 机制，S 端创建 socket 并绑定到指定网络地址和端口，待 C 端连接，C 端创建 socket 指定端口后发起连接请求  
S 端和 C 端为一对多，每个 C 端分配独立的 socket，UDP/TCP 都可采用
# ContentProvider
参考四大组件的 ContentProvider
# Message Queue & Pip
采用存储-转发的方式，数据从发送发缓存区拷贝到内核开辟的缓存区，再拷贝到接收方缓存区，至少两次拷贝过程
# SharedMemory
无需拷贝，但对于共享内存的访问控制需要考虑更多问题
# Tips
## 单应用多进程
1.偷内存，内存按照进程分配，通过进程统计  
2.互不影响，新进程崩溃不导致主应用受影响  
3.保活，主应用推出，新进程可以存活（如推送机制）
4.application 多次初始化，每次进程创建，走一遍初始化流程，appilcation 会多次创建  
5.变量无法共享，不同进程中是单独存在的
