# Serialize
序列化：对象将其状态写入临时（传输时使用）或持久性（持久化处理）存储区域  
反序列化：重新创建该对象，通过 FileOutPutStream/FileInPutStream 以及 ObjectOutPutStream/ObjectInPutStream 生成和读取二进制文件
## Serializable
编译器会根据类字段自动生成，也可以手动重写来生成 seriaVersionUID，这样即使类结构变化仍可以序列化  
通过 FileOutPutStream 创建 ObjectOutPutStream，创建 ObjectStramClass 并写入对象信息，类名以及 seriaVersionUID，最后写入对象需要反射解析要序列化的对象来生成 ObjectStreamClass，所以性能不太好，但是写起来方便点    
加上 transition 关键字的属性不会序列化
## Parcelable
重写 writeToParcel（用来序列化，将需要序列化的字段写入一个 parcel 对象） 并创建一个 Creator 重写 createFromParcel（反序列化，从 parcel 对象读取数据）  
Intent 使用的就是 Parcelable，如果传入的数据采用 Serializable 则会进行二次序列化（先序列化为字节数组再写入），所以尽量先统一一下
# Binder
## Foundation
只需要拷贝一次数据，需要内核支持，采用 Linux 内存映射（mmap，常用在文件等对象的操作，用内存读写代替 IO，在物理介质和用户空间建立映射）方式，将用户空间的内存区域映射到内核空间，两个空间变化都会反应到另一空间，这样可以减少拷贝次数。  
Binder 作为独立模块，通过内存映射，运行时被链接到内核作为内核的一部分运行（Linux 动态内核可加载模块）
## Connection
1.Binder 驱动在内核空间开辟出一个缓存区，建立缓存区和用户空间地址的映射关系  
2.发送进程将数据 copy 到内核中的缓存区，映射的存在相当于把数据发送到了接收进程的用户空间
## Space
1.C & S 为用户空间，S 创建 Binder 并命名，将 Binder 通过驱动传递给 ServiceManager，让 manager 注册该 binder。同时，驱动为该 binder 在内核创建一个对象以及对 manager 和这个对象的引用，将 binder 名和引用传递给 manager  
2.ServiceManage 也为用户空间，提供对 binder 的解析，将 binder 字符名转换为引用，使 C 可以通过 binder 名获取到对题诗的引用。与其他进程的通信也是通过 binder，自身维护一个 binder，没有名也无需注册（注册为 manager 时，驱动自动创建 binder 对象），其他进程通过该 binder 引用来注册。
3.Driver 属于内核空间，负责进程间 binder 通信的建立，binder 的传递，binder 引用的管理等。进程获取另一进程的 binder 对象获取到的其实时一个代理，本身不具有原对象的属性和方法，是通过驱动调用原进程对象的方法和获取属性，返回给该进程。（驱动会维护一个表，对象和代理对象一对一）  
4.Binder Pool 会将多个 binder 请求统一发送，避免 S 端重复创建 service（通过 queryBinder 返回不同 C 端的 binder 对象）
# AIDL
## Foundation
底层为 Binder，支持一对多和实时通信，可以处理并发场景，需要处理好线程安全问题
## Server
创建 service 并创建 AIDL（需要定义参数 in/out/inout，代表输入/输出/输入输出数据，指定数据流向，如 in 是 C 到 S） 和它的实现类，C 端获取 IBinder 转换给 AIDL 来调用方法
## Client
绑定 service，重写 onServiceConnection，创建 binder 对象（AIDL.Stub.asInterface）
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
