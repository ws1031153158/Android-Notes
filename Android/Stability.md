# Crash
## Java Crash
### 全局 UncaughtExceptionHandler
### 崩溃信息采集、堆栈还原
## Native Crash
### Breakpad / LLVM libunwind
### tombstone 分析
### addr2line 符号还原
## OOM Crash
### 内存快照（Hprof）采集
### 大对象追踪
### 内存兜底策略
## 归因分析
### 线上监控
### 版本/机型/系统分布分析
### 聚合策略
### 报警策略
### 相似崩溃合并
## 崩溃恢复
### 进程重启策略
### 数据保护
### 降级方案
## 热修复
### 进程重启策略、数据保护、降级方案
# ANR
## ANR 监控
### 主线程 WatchDog
### Signal 监听（SIGQUIT）
### ANR-WatchDog 库
## Binder 阻塞
### Binder 线程耗尽
### 系统服务响应慢
## Broadcast ANR
### 前台 10s / 后台 60s 超时
### onReceive 耗时
## ContentProvider ANR
### publish 超时
## Service ANR
### 前台 20s / 后台 200s 超时
## Input 超时
### 主线程 5s 无响应
### 触摸事件分发耗时
## 系统资源不足
### CPU 负载
### IO wait
### 内存压力导致的 ANR
