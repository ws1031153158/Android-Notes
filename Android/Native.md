# Memory
## ELF
ELF（Executable and Linkable Format）文件是类Unix系统（包括Android）中可执行文件、共享库、目标代码的标准格式。  
<img width="437" alt="986da92b-4fb7-44e3-8a95-504b6f6f1cd7" src="https://github.com/user-attachments/assets/0966f0db-c773-4e54-92e5-15bee23c02b3" />   
## malloc
标准内存分配函数,仅保证基本对齐（通常8/16字节）,void* malloc(size_t size);堆内存
## memalign
对齐内存分配函数,可指定任意对齐值（需是2的幂次）,void* memalign(size_t alignment, size_t size);堆内存
# so
## armeabi & arm64-v8
armeabi是较旧的32位ARM架构，主要用于旧的设备，而arm64-v8a是64位ARM架构，支持更现代的处理器   
 <img width="470" alt="bb1235c7-e694-4546-82bf-8d6d3df730d6" src="https://github.com/user-attachments/assets/06dcb03a-6bff-4eac-b190-d4de7650961b" />
