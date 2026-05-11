# Crash
## 核心指标
```
核心指标：

Crash Rate = Crash 次数 / DAU
目标：< 0.01%（优秀）< 0.05%（良好）

ANR Rate = ANR 次数 / DAU
目标：< 0.05%

UV Crash Rate = 发生 Crash 的用户数 / DAU
（比 Crash Rate 更能反映用户体验）

Crash Free Users = 1 - UV Crash Rate
目标：> 99.9%
```
## Java Crash
```
Java Crash 触发流程：

① 抛出未捕获异常（RuntimeException/Error）
② JVM 查找当前线程的 UncaughtExceptionHandler
③ 找不到 → 查找线程组的 Handler
④ 找不到 → 使用默认 Handler（打印堆栈，终止进程）

默认 Handler：
Thread.getDefaultUncaughtExceptionHandler()
→ RuntimeInit.KillApplicationHandler
→ 调用 Process.killProcess(Process.myPid())
→ 进程终止

我们的机会：
在默认 Handler 执行之前
设置自定义 UncaughtExceptionHandler
拦截异常，收集信息，上报
```
### 全局 UncaughtExceptionHandler
```
// ================================
// 完整的 Java Crash 捕获方案
// ================================

class JavaCrashHandler private constructor(
    private val context: Context
) : Thread.UncaughtExceptionHandler {

    // 保存系统默认 Handler，处理完后交给它
    private val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()

    companion object {
        fun install(context: Context) {
            val handler = JavaCrashHandler(context.applicationContext)
            Thread.setDefaultUncaughtExceptionHandler(handler)
        }
    }

    override fun uncaughtException(thread: Thread, throwable: Throwable) {
        try {
            // 1. 收集崩溃信息
            val crashInfo = collectCrashInfo(thread, throwable)

            // 2. 保存到本地（网络可能不可用）
            saveCrashToLocal(crashInfo)

            // 3. 尝试上报（可能失败，下次启动补传）
            trySyncReport(crashInfo)

            // 4. 可选：弹出友好提示（不推荐，影响体验）
            // showCrashDialog()

        } catch (e: Exception) {
            // 崩溃处理本身不能崩溃！
            Log.e("CrashHandler", "处理崩溃时发生异常", e)
        } finally {
            // 5. 交给默认 Handler（系统处理，生成 tombstone）
            defaultHandler?.uncaughtException(thread, throwable)
        }
    }

    private fun collectCrashInfo(
        thread: Thread,
        throwable: Throwable
    ): CrashInfo {
        return CrashInfo(
            // 异常信息
            exceptionType = throwable.javaClass.name,
            exceptionMessage = throwable.message ?: "",
            stackTrace = getStackTrace(throwable),
            cause = throwable.cause?.let { getStackTrace(it) },

            // 线程信息
            threadName = thread.name,
            threadId = thread.id,
            isMainThread = thread == Looper.getMainLooper().thread,

            // 设备信息
            deviceModel = Build.MODEL,
            osVersion = Build.VERSION.RELEASE,
            sdkVersion = Build.VERSION.SDK_INT,
            appVersion = getAppVersion(),

            // 内存信息
            javaHeapUsedMB = getJavaHeapUsed(),
            nativeHeapMB = Debug.getNativeHeapAllocatedSize() / 1024 / 1024,
            availRamMB = getAvailRam(),

            // 运行时信息
            foregroundActivity = ActivityStack.getTopActivity(),
            processName = getProcessName(),
            timestamp = System.currentTimeMillis(),

            // 自定义上下文（业务埋点）
            customContext = CrashContext.getAll()
        )
    }

    private fun getStackTrace(throwable: Throwable): String {
        val sw = StringWriter()
        throwable.printStackTrace(PrintWriter(sw))
        return sw.toString()
    }

    private fun saveCrashToLocal(crashInfo: CrashInfo) {
        try {
            val file = File(
                context.getExternalFilesDir("crash"),
                "crash_${crashInfo.timestamp}.json"
            )
            file.writeText(crashInfo.toJson())
        } catch (e: Exception) { /* 忽略 */ }
    }

    private fun trySyncReport(crashInfo: CrashInfo) {
        // 同步上报（崩溃时网络可能不稳定）
        // 超时时间要短，避免阻塞太久
        try {
            CrashReporter.reportSync(crashInfo, timeoutMs = 3000)
        } catch (e: Exception) {
            // 上报失败，下次启动时补传
        }
    }
}

// ================================
// 崩溃上下文（业务埋点）
// ================================
object CrashContext {
    private val context = ConcurrentHashMap<String, String>()

    // 在关键业务节点埋点，崩溃时自动附带
    fun put(key: String, value: String) {
        context[key] = value
        // 限制大小，防止内存占用过多
        if (context.size > 50) {
            context.remove(context.keys.first())
        }
    }

    fun getAll(): Map<String, String> = context.toMap()
}

// 使用示例
class CheckoutActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        CrashContext.put("page", "checkout")
        CrashContext.put("user_id", UserSession.userId)
    }

    fun onPayClick() {
        CrashContext.put("pay_step", "click_pay")
        CrashContext.put("order_id", currentOrderId)
        startPayment()
    }
}
```
### 崩溃聚合与归因
```
// 崩溃聚合：将相同根因的崩溃归为一组
// 避免同一个 bug 产生大量重复告警

object CrashAggregator {

    // 生成崩溃指纹（相同根因的崩溃应该有相同指纹）
    fun generateFingerprint(crashInfo: CrashInfo): String {
        // 策略：取堆栈的前 N 帧（去掉动态变化的部分）
        val keyFrames = crashInfo.stackTrace
            .lines()
            .filter { it.trimStart().startsWith("at ") }
            .filter { line ->
                // 过滤掉框架层代码，只保留业务代码
                !line.contains("java.lang.reflect") &&
                !line.contains("android.os.Handler") &&
                line.contains("com.xxx") // 只保留业务包名
            }
            .take(5) // 取前5帧
            .joinToString("|")

        return "${crashInfo.exceptionType}|$keyFrames"
            .md5() // 生成 MD5 作为指纹
    }

    // 崩溃优先级评估
    fun evaluatePriority(crashInfo: CrashInfo): CrashPriority {
        return when {
            // 主线程崩溃 + 高频 = P0
            crashInfo.isMainThread &&
            getCrashCount(crashInfo.fingerprint) > 100 ->
                CrashPriority.P0

            // 主线程崩溃 = P1
            crashInfo.isMainThread -> CrashPriority.P1

            // 子线程崩溃 + 高频 = P2
            getCrashCount(crashInfo.fingerprint) > 50 ->
                CrashPriority.P2

            else -> CrashPriority.P3
        }
    }
}
```
## Native Crash
```
Native Crash 触发流程：

程序运行在用户态，系统调用、中断或异常时进入内核态，信号涉及两种状态转换。  
接收：接收信号由内核代理，收到信号会放到对应进程信号队列中，向进程发送一个中断，使其进入内核态。（此时信号只在队列中，对进程来说暂时不知道信号到来）。  
检测：进程进入内核态有两种场景对信号检测：    
1.从内核态返回用户态前进行检测。  
2.在内核态中，从睡眠状态被唤醒时进行检测。    
有新信号时进入信号处理：处理运行在用户态，处理前，内核将当前栈内数据拷贝到用户栈，修改指令寄存器（eip）并指向信号处理函数，之后返回用户态，执行对应处理函数，处理完成后返回内核态，检查是否有信号未处理。所有信号处理完成，恢复内核栈（用户栈拷贝回来），恢复指令寄存器（eip）并指向中断前的运行位置，最后回到用户态继续执行进程。

① C/C++ 代码发生异常
   ├── 空指针解引用（SIGSEGV）
   ├── 非法指令（SIGILL）
   ├── 算术错误（SIGFPE，如除以0）
   ├── 堆栈溢出（SIGSEGV，特殊地址）
   └── abort() 调用（SIGABRT）

② Linux 内核发送信号（软中断信号）给进程

③ 进程的信号处理函数被调用
   默认：终止进程 + 生成 tombstone

④ debuggerd（系统守护进程）
   捕获崩溃，生成 /data/tombstones/tombstone_XX

tombstone 包含：
├── 信号信息（SIGSEGV at 0x00000000）
├── 寄存器状态（PC/SP/LR 等）
├── 调用栈（Native 符号）
├── 内存映射（/proc/pid/maps）
└── 所有线程的状态
```
### 捕获
```
// Native Crash 捕获需要在 C++ 层注册信号处理函数

// Kotlin 层接口
object NativeCrashHandler {

    external fun install(logDir: String)
    external fun setAppVersion(version: String)
    external fun setCustomInfo(key: String, value: String)

    fun install(context: Context) {
        val logDir = File(context.filesDir, "native_crash").also {
            it.mkdirs()
        }
        install(logDir.absolutePath)
        setAppVersion(BuildConfig.VERSION_NAME)
    }

    init {
        System.loadLibrary("native_crash_handler")
    }
}

// C++ 层实现（native_crash_handler.cpp）
#include <signal.h>
#include <unistd.h>
#include <pthread.h>
#include <android/log.h>
#include <unwind.h>
#include <dlfcn.h>

// 需要处理的信号列表
static const int SIGNALS[] = {
    SIGSEGV,  // 段错误（空指针/越界）
    SIGABRT,  // abort() 调用
    SIGFPE,   // 浮点异常
    SIGILL,   // 非法指令
    SIGBUS,   // 总线错误
    SIGSTKFLT // 栈错误
};

// 保存旧的信号处理函数
static struct sigaction gOldHandlers[6];

// 备用信号栈（防止栈溢出时无法处理信号）
static uint8_t gAltStack[SIGSTKSZ * 2];

// 信号处理函数
static void signalHandler(int sig, siginfo_t* info, void* context) {
    // 1. 收集崩溃信息
    CrashInfo crashInfo;
    crashInfo.signal = sig;
    crashInfo.faultAddr = info->si_addr;

    // 2. 采集调用栈
    captureBacktrace(crashInfo.backtrace, 32);

    // 3. 写入文件（异步IO可能不安全，使用 write 系统调用）
    writeCrashToFile(crashInfo);

    // 4. 调用旧的信号处理函数（让系统生成 tombstone）
    sigaction(sig, &gOldHandlers[getSignalIndex(sig)], nullptr);
    raise(sig);
}

// 采集调用栈（使用 _Unwind_Backtrace）
struct BacktraceState {
    void** current;
    void** end;
};

static _Unwind_Reason_Code unwindCallback(
    struct _Unwind_Context* context, void* arg) {

    BacktraceState* state = static_cast<BacktraceState*>(arg);
    uintptr_t pc = _Unwind_GetIP(context);

    if (pc) {
        if (state->current == state->end) {
            return _URC_END_OF_STACK;
        }
        *state->current++ = reinterpret_cast<void*>(pc);
    }
    return _URC_NO_REASON;
}

static size_t captureBacktrace(void** buffer, size_t max) {
    BacktraceState state = {buffer, buffer + max};
    _Unwind_Backtrace(unwindCallback, &state);
    return state.current - buffer;
}

// 符号解析（将地址转换为函数名）
static void resolveSymbol(void* addr, char* output, size_t size) {
    Dl_info info;
    if (dladdr(addr, &info)) {
        snprintf(output, size, "%s(%s+%p) [%p]",
            info.dli_fname,           // so 文件名
            info.dli_sname ?: "???",  // 函数名
            (void*)((char*)addr - (char*)info.dli_saddr), // 偏移
            addr
        );
    } else {
        snprintf(output, size, "??? [%p]", addr);
    }
}

// 安装信号处理函数
extern "C" JNIEXPORT void JNICALL
Java_com_xxx_NativeCrashHandler_install(
    JNIEnv* env, jobject, jstring logDir) {

    // 设置备用信号栈（防止栈溢出时无法处理）
    stack_t ss;
    ss.ss_sp = gAltStack;
    ss.ss_size = sizeof(gAltStack);
    ss.ss_flags = 0;
    sigaltstack(&ss, nullptr);

    // 注册信号处理函数
    struct sigaction sa;
    sa.sa_sigaction = signalHandler;
    sa.sa_flags = SA_SIGINFO | SA_ONSTACK; // SA_ONSTACK：使用备用栈
    sigemptyset(&sa.sa_mask);

    for (int i = 0; i < 6; i++) {
        sigaction(SIGNALS[i], &sa, &gOldHandlers[i]);
    }
}
```
### Breakpad
```
Breakpad：Google 开源的跨平台崩溃捕获库
├── 支持：Linux/Android/Windows/macOS/iOS
├── 核心产物：minidump 文件（.dmp）
│   └── 比 tombstone 更小，更标准化
└── 工具链：
    ├── dump_syms：从 so 提取符号信息
    ├── minidump_stackwalk：解析 minidump
    └── minidump-2-core：转换为 core dump

与直接信号处理的对比：
┌──────────────┬──────────────────┬────────────────────┐
│              │ 自定义信号处理    │ Breakpad           │
├──────────────┼──────────────────┼────────────────────┤
│ 跨平台       │ 否               │ 是                 │
│ 符号化       │ 需要自己实现      │ 完整工具链          │
│ minidump     │ 否               │ 是（标准格式）      │
│ 稳定性       │ 一般             │ 高（Google验证）    │
│ 接入复杂度   │ 低               │ 中                 │
└──────────────┴──────────────────┴────────────────────┘

Breakpad 崩溃捕获流程：

① 安装异常处理器
   ├── Linux/Android：注册信号处理函数
   │   SIGSEGV/SIGABRT/SIGFPE/SIGILL/SIGBUS
   └── 使用备用信号栈（SA_ONSTACK）

② 崩溃发生时：
   ├── 信号处理函数被调用
   ├── 创建 minidump 文件
   │   ├── 线程列表 + 每个线程的寄存器状态
   │   ├── 内存映射（/proc/pid/maps）
   │   ├── 异常信息（信号/地址）
   │   └── 系统信息（OS版本/CPU架构）
   └── 写入磁盘

③ 下次启动时上传 minidump

④ 服务端符号化：
   minidump + 符号文件 → 可读的崩溃堆栈

minidump 文件结构：
┌─────────────────────────────────┐
│  Header（签名/版本/流数量）       │
├─────────────────────────────────┤
│  StreamDirectory（流目录）        │
├─────────────────────────────────┤
│  ThreadListStream（线程列表）     │
│  ├── Thread 0（主线程）          │
│  │   ├── ThreadId               │
│  │   ├── SuspendCount           │
│  │   ├── Context（寄存器状态）   │
│  │   └── Stack Memory           │
│  └── Thread N...                │
├─────────────────────────────────┤
│  ModuleListStream（模块列表）     │
│  ├── so文件路径                  │
│  ├── 基地址                     │
│  └── Build ID                   │
├─────────────────────────────────┤
│  ExceptionStream（异常信息）      │
│  ├── 信号类型                   │
│  ├── 崩溃地址                   │
│  └── 崩溃线程ID                 │
└─────────────────────────────────┘
```
### LLVM libunwind
```
libunwind：LLVM 项目的栈展开库
Android NDK 默认使用 libunwind 进行栈展开

栈展开（Stack Unwinding）：
从当前栈帧出发，逐帧向上追溯调用链
最终得到完整的调用栈

两种栈展开方式：
┌──────────────────┬──────────────────┬──────────────────┐
│                  │ Frame Pointer    │ DWARF/EH         │
│                  │ 基于帧指针        │ 基于调试信息      │
├──────────────────┼──────────────────┼──────────────────┤
│ 速度             │ 快               │ 慢               │
│ 准确性           │ 一般             │ 高               │
│ 需要编译选项     │ -fno-omit-frame- │ 默认支持          │
│                  │ pointer          │                  │
│ Android默认      │ 否（优化掉了）    │ 是               │
└──────────────────┴──────────────────┴──────────────────┘

DWARF 栈展开原理：

每个 so 文件包含 .eh_frame 段：
记录了每个 PC（程序计数器）地址对应的
CFA（Canonical Frame Address）计算规则

展开步骤：
① 获取当前 PC（程序计数器）
② 在 .eh_frame 中查找 PC 对应的 FDE（Frame Description Entry）
③ 根据 FDE 中的 CFA 规则计算上一帧的 SP（栈指针）
④ 根据 CFA 恢复上一帧的寄存器（包括 PC）
⑤ 重复 ①~④，直到无法继续展开

ARM64 寄存器：
├── PC（x30/LR）：程序计数器，当前执行地址
├── SP（x31）：栈指针
├── FP（x29）：帧指针（如果开启）
└── LR（x30）：链接寄存器，保存返回地址
```
### tombstone 分析
```
# 获取 tombstone 文件
adb pull /data/tombstones/tombstone_00

# tombstone 内容解析：
# *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
# Build fingerprint: 'google/sdk_gphone64_x86_64/...'
# pid: 12345, tid: 12345, name: com.xxx.app
# signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
#                                                          ↑ 空指针
# backtrace:
#   #00 pc 00034abc  /data/app/com.xxx/lib/arm64/libnative.so
#   #01 pc 00012345  /data/app/com.xxx/lib/arm64/libnative.so
#   ...

# 符号化（将地址转换为函数名+行号）
# 需要带符号的 so 文件（不是 strip 后的）
ndk-stack -sym $PROJECT/obj/local/arm64-v8a -dump tombstone_00

# 或使用 addr2line
arm-linux-androideabi-addr2line \
    -f -e libxxx.so \
    0x00034abc
```
### addr2line 符号还原
Native Crash 符号还原流程：   
```
线上 so（strip 后）：
├── 没有调试信息（.debug_info/.debug_line）
├── 没有符号表（.symtab）
└── 只有最基本的运行时信息

崩溃日志中的地址：
#00 pc 00034abc  /data/app/com.xxx/lib/arm64/libnative.so

需要还原为：
#00 pc 00034abc  /data/app/.../libnative.so
    com::xxx::DataManager::loadData(std::string const&)
    DataManager.cpp:45

还原步骤：
① 保留带符号的 so 文件（构建时生成两份）
   ├── libnative.so（strip后，用于发布）
   └── libnative_unstripped.so（带符号，用于符号化）

② 确认 so 的 Build ID 匹配
   （确保符号文件和崩溃时的 so 是同一个版本）

③ 使用 addr2line 还原
```

addr2line使用:  
```
# 基本用法
# -f：显示函数名
# -e：指定带符号的 so 文件
# -C：demangle C++ 符号
# -p：pretty print
# addr：崩溃地址（相对地址）

# 单个地址还原
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/\
llvm-addr2line \
    -f -C -e obj/local/arm64-v8a/libnative.so \
    0x00034abc

# 输出：
# DataManager::loadData(std::string const&)
# /src/DataManager.cpp:45

# 批量还原（从文件读取地址）
$NDK/llvm-addr2line -f -C \
    -e libnative_unstripped.so \
    $(cat crash_addrs.txt)

# 注意：使用相对地址（relPc），不是绝对地址
# 绝对地址 = so基地址 + 相对地址
# addr2line 需要相对地址

#!/usr/bin/env python3
# symbolize.py：自动化符号还原脚本

import subprocess
import re
import os
import sys

# NDK 路径
NDK_PATH = os.environ.get('ANDROID_NDK', '/path/to/ndk')
ADDR2LINE = f"{NDK_PATH}/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-addr2line"

# 符号文件目录（按版本/ABI组织）
SYMBOL_DIR = "./symbols"

def get_symbol_file(so_name: str, build_id: str, abi: str) -> str:
    """根据 so 名称和 Build ID 找到对应的符号文件"""
    # 目录结构：symbols/{abi}/{build_id}/{so_name}
    symbol_path = os.path.join(SYMBOL_DIR, abi, build_id, so_name)
    if os.path.exists(symbol_path):
        return symbol_path

    # 降级：只按 so 名称查找（不验证 Build ID）
    fallback = os.path.join(SYMBOL_DIR, abi, so_name)
    return fallback if os.path.exists(fallback) else None

def addr2line(symbol_file: str, address: str) -> tuple:
    """调用 addr2line 还原单个地址"""
    try:
        result = subprocess.run(
            [ADDR2LINE, '-f', '-C', '-p', '-e', symbol_file, address],
            capture_output=True, text=True, timeout=5
        )
        output = result.stdout.strip()
        # 解析输出：函数名 at 文件名:行号
        if ' at ' in output:
            func, location = output.split(' at ', 1)
            return func.strip(), location.strip()
        return output, ''
    except Exception as e:
        return f'addr2line failed: {e}', ''

def symbolize_crash_log(crash_log: str, abi: str = 'arm64-v8a') -> str:
    """符号化完整崩溃日志"""

    # 匹配崩溃堆栈行
    # 格式：#00 pc 00034abc  /path/to/libnative.so
    frame_pattern = re.compile(
        r'(#\d+)\s+pc\s+([0-9a-f]+)\s+(/[^\s]+\.so)(?:\s+$([^)]+)$)?'
    )

    lines = crash_log.split('\n')
    result_lines = []

    for line in lines:
        match = frame_pattern.search(line)
        if match:
            frame_num = match.group(1)
            address = match.group(2)
            so_path = match.group(3)
            so_name = os.path.basename(so_path)

            # 查找符号文件
            symbol_file = get_symbol_file(so_name, '', abi)

            if symbol_file:
                func_name, location = addr2line(symbol_file, f'0x{address}')
                # 替换原始行，添加符号信息
                symbolized = f"{line}\n      {func_name}"
                if location and '?' not in location:
                    symbolized += f"\n      {location}"
                result_lines.append(symbolized)
            else:
                result_lines.append(f"{line}  [no symbol file]")
        else:
            result_lines.append(line)

    return '\n'.join(result_lines)

# 使用示例
if __name__ == '__main__':
    crash_log = sys.stdin.read()
    print(symbolize_crash_log(crash_log))


# 获取 so 文件的 Build ID
# 确保符号文件和崩溃时的 so 版本匹配

# 方法一：readelf
readelf -n libnative.so | grep "Build ID"
# 输出：Build ID: a1b2c3d4e5f6...

# 方法二：objdump
objdump -s -j .note.gnu.build-id libnative.so

# 方法三：file 命令
file libnative.so
# 输出包含 BuildID

# 在崩溃日志中也包含 Build ID
# tombstone 中：
# build_id: 'a1b2c3d4e5f6...'

# 符号文件目录按 Build ID 组织：
# symbols/
# ├── arm64-v8a/
# │   ├── a1b2c3d4/
# │   │   └── libnative.so  ← 带符号的 so
# │   └── b2c3d4e5/
# │       └── libnative.so
# └── armeabi-v7a/
#     └── ...
```
# ANR
```
ANR 类型与超时时间：

┌──────────────────────┬──────────┬──────────────────────┐
│ ANR 类型             │ 超时时间 │ 触发条件              │
├──────────────────────┼──────────┼──────────────────────┤
│ Input ANR            │ 5秒      │ 触摸/按键事件未响应   │
│ Service ANR          │ 20秒(前台)│ Service.onCreate/    │
│                      │ 200秒(后台)│ onStartCommand 超时 │
│ Broadcast ANR        │ 10秒(前台)│ onReceive 超时       │
│                      │ 60秒(后台)│                      │
│ ContentProvider ANR  │ 10秒     │ publish 超时          │
└──────────────────────┴──────────┴──────────────────────┘

Input ANR 详细流程：
① 用户触摸屏幕
② InputDispatcher 分发事件给 App
③ App 主线程处理事件
④ 5秒内未处理完 → InputDispatcher 触发 ANR
⑤ AMS 收到 ANR 通知
⑥ dump 所有进程的堆栈（traces.txt）
⑦ 弹出 ANR 对话框
```
## 应用侧归因
1.死锁  
2.主线程调用 thread 的 join()方法、sleep()方法、wait()方法或者等待线程锁的时候  
3.主线程阻塞在 nSyncDraw  
4.主线程耗时操作，如复杂的 layout，庞大的 for 循环，IO 等  
5.主线程被子线程同步锁 block  
6.主线程等待子线程超时  
7.主线程 Activity 生命周期函数执行超时  
8.主线程 Service 生命周期函数执行超时  
9.主线程 Broadcast.onReceive 函数执行超时（即使调用了 goAsync ）  
10.渲染线程耗时  
11.耗时的网络访问  
12.大量的数据读写  
13.数据库操作  
14.硬件操作（比如 Camera)  
15.service binder 的数量达到上限  
16.其它线程终止或崩溃导致主线程一直等待  
17.Dump 内存操作  
18.大量 SharedPerference 同时读写  
## 系统侧归因
1.与 SystemServer 进行 Binder 通信，SystemServer 执行耗时：方法本身执行耗时导致超时，或者 SystemServer Binder 锁竞争太多，导致等锁超时  
2.等待其他进程返回超时，比如从其他进程的 ContentProvider 中获取数据超时（10s，但是这个场景很少见）  
3.Window 错乱导致 Input 超时（5s）  
4.ContentProvider 对端的进程频繁崩溃，也会杀掉当前进程  
5.整机低内存  
6.整机 CPU 占用高：查看是总使用率是否过高，同时需要关注缺页次数（访问的页面不在内存时，会产生一次缺页中断，需要调入主存，一次调入次数加一），xxx minor 表示高速缓存中的缺页次数，可以理解为进程在做内存访问，xxx major 表示内存的缺页次数，可以理解为进程在做 IO 操作   
7.整机 IO 使用率高  
8.SurfaceFlinger 超时  
9.系统冻结功能出现 Bug：应用是否处于 D 状态（TASK_UNINTERRUPTIBLE，不可中断的睡眠状态，通常是由于它正在执行一个不能中断的 I/O 操作，如磁盘读写、网络传输等）    
10.System Server 中 WatchDog 出现 ANR  
11.整机触发温控限制频率  
12.Service 前台 20 s，后台 200 s 超时（system server 在 service 流程开始前会发生 delay 消息到 main handler，若 app 端在 delay 消息内未通知 system server 移除消息，则超时）  
1.Broadcast 前台 10 s，后台 60 s 超时（同样，也是发生 delay 消息，等待 receiver 去移除，contentProvider 一样的机制）
## ANR 监控
### ANR WatchDog
```
// ================================
// WatchDog 检测 ANR（前面已讲）
// 补充：ANR 级别的堆栈采集
// ================================
class ANRWatchDog(private val anrThreshold: Long = 5000L) {

    private val mainHandler = Handler(Looper.getMainLooper())
    private val stackSampler = StackSampler()

    @Volatile private var tickCount = 0L
    @Volatile private var lastTickCount = -1L
    @Volatile private var blockStart = 0L

    fun start() {
        mainHandler.post { tickCount++ }

        thread(name = "anr-watchdog", isDaemon = true) {
            while (true) {
                Thread.sleep(1000)

                if (tickCount == lastTickCount) {
                    val now = SystemClock.elapsedRealtime()
                    if (blockStart == 0L) {
                        blockStart = now
                        // 开始密集采集堆栈
                        stackSampler.startSampling(
                            Looper.getMainLooper().thread,
                            intervalMs = 100 // 每100ms采集一次
                        )
                    }

                    val blocked = now - blockStart
                    if (blocked >= anrThreshold) {
                        // ANR 级别！采集完整信息
                        reportANR(blocked)
                    }
                } else {
                    blockStart = 0L
                    lastTickCount = tickCount
                    stackSampler.stopSampling()
                    mainHandler.post { tickCount++ }
                }
            }
        }
    }

    private fun reportANR(blockedMs: Long) {
        val mainStack = stackSampler.getCollectedStacks()
        val allThreadStacks = getAllThreadStacks()
        val cpuInfo = getCpuInfo()
        val memInfo = getMemInfo()

        ANRReporter.report(
            ANRInfo(
                blockedMs = blockedMs,
                mainThreadStacks = mainStack,
                allThreadStacks = allThreadStacks,
                cpuUsage = cpuInfo,
                memoryInfo = memInfo,
                messageHistory = MessageMonitor.getRecentMessages()
            )
        )
    }

    private fun getAllThreadStacks(): Map<String, String> {
        return Thread.getAllStackTraces()
            .entries
            .associate { (thread, stack) ->
                thread.name to stack.joinToString("\n") { "\tat $it" }
            }
    }

    private fun getCpuInfo(): String {
        return try {
            File("/proc/stat").readLines().firstOrNull() ?: ""
        } catch (e: Exception) { "" }
    }
}
```
### FileObserver
```
// ================================
// FileObserver 监控 traces.txt
// ================================
class ANRFileObserver(
    private val onANRDetected: (String) -> Unit
) : FileObserver("/data/anr/", CLOSE_WRITE) {

    override fun onEvent(event: Int, path: String?) {
        if (path?.startsWith("traces") == true) {
            // traces.txt 被写入 = 发生了 ANR
            val tracesFile = File("/data/anr/$path")
            if (tracesFile.canRead()) {
                val content = tracesFile.readText()
                // 过滤出当前进程的堆栈
                val myTrace = extractMyTrace(content)
                onANRDetected(myTrace)
            }
        }
    }

    private fun extractMyTrace(content: String): String {
        val pid = Process.myPid()
        // 找到当前进程的堆栈段
        val startMarker = "pid: $pid"
        val startIndex = content.indexOf(startMarker)
        if (startIndex == -1) return content

        val endIndex = content.indexOf("\n----- pid", startIndex + 1)
        return if (endIndex == -1) {
            content.substring(startIndex)
        } else {
            content.substring(startIndex, endIndex)
        }
    }
}
```
### traces.txt
结构：  
```
----- pid 12345 at 2024-01-01 12:00:00 -----
Cmd line: com.xxx.app
...

DALVIK THREADS (23):                    ← 线程总数
"main" prio=5 tid=1 Blocked             ← 主线程状态！
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x... self=0x...
  | sysTid=12345 nice=0 cgrp=default sched=0/0 handle=0x...
  | state=S schedstat=( 0 0 0 ) utm=100 stm=50 core=0 HZ=100
  | stack=0x... stackSize=8192KB
  | held mutexes=
  at com.xxx.DataManager.loadData(DataManager.kt:45)  ← 卡在这里！
  - waiting to lock <0x12345678> (a java.lang.Object)  ← 等待这把锁
  - locked <0x87654321> (a java.lang.Object)           ← 持有这把锁
  at com.xxx.MainActivity.onClick(MainActivity.kt:100)
  at android.view.View.performClick(View.java:7125)
  ...

"background-thread" prio=5 tid=5 Blocked
  at com.xxx.DataManager.saveData(DataManager.kt:80)
  - waiting to lock <0x87654321>  ← 等待主线程持有的锁！
  - locked <0x12345678>           ← 持有主线程等待的锁！
  ...
  ↑ 死锁！主线程和后台线程互相等待对方的锁

关键信息解读：
├── "main" ... Blocked → 主线程被阻塞
├── waiting to lock → 等待哪把锁
├── locked → 持有哪把锁
├── 两个线程互相等待 → 死锁
└── Native 调用（如 binder） → Binder 阻塞
```

拆解器：  
```
// traces.txt 自动解析器
object TracesAnalyzer {

    data class ANRAnalysis(
        val mainThreadState: String,
        val blockReason: BlockReason,
        val blockDetail: String,
        val relatedThreads: List<String>,
        val isDeadlock: Boolean
    )

    enum class BlockReason {
        DEADLOCK,           // 死锁
        LOCK_CONTENTION,    // 锁竞争
        BINDER_CALL,        // Binder 调用
        IO_WAIT,            // IO 等待
        SLEEP,              // 主动 sleep
        CPU_BUSY,           // CPU 繁忙
        UNKNOWN
    }

    fun analyze(tracesContent: String): ANRAnalysis {
        val mainThreadBlock = extractMainThreadBlock(tracesContent)

        val blockReason = when {
            mainThreadBlock.contains("waiting to lock") &&
            hasDeadlock(tracesContent) ->
                BlockReason.DEADLOCK

            mainThreadBlock.contains("waiting to lock") ->
                BlockReason.LOCK_CONTENTION

            mainThreadBlock.contains("android.os.BinderProxy") ||
            mainThreadBlock.contains("binder") ->
                BlockReason.BINDER_CALL

            mainThreadBlock.contains("java.io") ||
            mainThreadBlock.contains("FileInputStream") ->
                BlockReason.IO_WAIT

            mainThreadBlock.contains("Thread.sleep") ->
                BlockReason.SLEEP

            else -> BlockReason.UNKNOWN
        }

        return ANRAnalysis(
            mainThreadState = extractThreadState(mainThreadBlock),
            blockReason = blockReason,
            blockDetail = extractBlockDetail(mainThreadBlock),
            relatedThreads = findRelatedThreads(tracesContent, mainThreadBlock),
            isDeadlock = blockReason == BlockReason.DEADLOCK
        )
    }

    private fun hasDeadlock(content: String): Boolean {
        // 检测循环等待：A等B，B等A
        val lockPattern = Regex("waiting to lock <(0x[0-9a-f]+)>")
        val heldPattern = Regex("locked <(0x[0-9a-f]+)>")

        val threadBlocks = content.split(Regex("\"[^\"]+\" prio="))

        // 构建等待图
        val waitGraph = mutableMapOf<String, Set<String>>()
        threadBlocks.forEach { block ->
            val waiting = lockPattern.find(block)?.groupValues?.get(1)
            val held = heldPattern.findAll(block).map { it.groupValues[1] }.toSet()
            if (waiting != null) {
                waitGraph[waiting] = held
            }
        }

        // 检测环路（DFS）
        return detectCycle(waitGraph)
    }

    private fun detectCycle(graph: Map<String, Set<String>>): Boolean {
        val visited = mutableSetOf<String>()
        val inStack = mutableSetOf<String>()

        fun dfs(node: String): Boolean {
            visited.add(node)
            inStack.add(node)
            graph[node]?.forEach { neighbor ->
                if (!visited.contains(neighbor) && dfs(neighbor)) return true
                if (inStack.contains(neighbor)) return true
            }
            inStack.remove(node)
            return false
        }

        return graph.keys.any { !visited.contains(it) && dfs(it) }
    }
}
```
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
