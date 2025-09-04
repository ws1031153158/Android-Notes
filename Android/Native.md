# Memory
## ELF
ELF（Executable and Linkable Format）文件是类Unix系统（包括Android）中可执行文件、共享库、目标代码的标准格式。  
<img width="437" alt="986da92b-4fb7-44e3-8a95-504b6f6f1cd7" src="https://github.com/user-attachments/assets/0966f0db-c773-4e54-92e5-15bee23c02b3" />   
## malloc
标准内存分配函数,仅保证基本对齐（通常8/16字节）,void* malloc(size_t size);堆内存
## memalign
对齐内存分配函数,可指定任意对齐值（需是2的幂次）,void* memalign(size_t alignment, size_t size);堆内存
## mmap
inux/Unix 系统下的一个系统调用，用于将文件或设备映射到进程的虚拟内存空间。让应用可以像操作普通内存一样操作文件内容（无需 read/write）。  
用于共享内存、内存映射文件、加载动态库等场景  
mmap 映射的长度和起始地址通常需要是页面大小的整数倍（如4KB或16KB），不对齐会导致系统报错
## dlopen
Linux/Unix 下的一个动态库加载函数，属于动态链接库（Dynamic Linking Loader）机制的一部分。  
在运行时加载 .so（共享库，Shared Object）文件，用于插件化架构、按需加载第三方库等场景。  
dlopen 加载库时，底层会用 mmap 将库映射到内存，如果库的 ELF 段对齐不符合当前设备的页面大小（如只对齐4KB，设备却是16KB），dlopen 可能加载失败或报错
# so
## armeabi & arm64-v8
armeabi是较旧的32位ARM架构，主要用于旧的设备，而arm64-v8a是64位ARM架构，支持更现代的处理器   
 <img width="470" alt="bb1235c7-e694-4546-82bf-8d6d3df730d6" src="https://github.com/user-attachments/assets/06dcb03a-6bff-4eac-b190-d4de7650961b" />
# page size
管理内存时的一个基本单位,物理内存和虚拟内存都被划分为一个个“页”（page），每页的大小就是 pagesize。常见值有 4KB（4096字节）和 16KB（16384字节）  
## 内存分配与管理
操作系统分配、回收、映射内存时，最小粒度就是一页。例如，进程申请内存、mmap文件、加载.so库，都以页为单位。
## 虚拟内存与物理内存映射
页表（Page Table）用于记录虚拟地址与物理地址之间的映射关系，每个页表项对应一个页面
## upgrade
1.更大页面意味着管理的页表项更少，降低内核开销  
2.数据库、缓存系统、文件系统等底层组件能更高效地处理大数据  
3.大页面能减少TLB(Translation Lookaside Buffer)表项数量，降低TLB miss率，提高内存访问效率。
