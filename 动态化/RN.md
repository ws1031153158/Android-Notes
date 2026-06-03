# RN 原理
## 架构概览
```
React Native（RN）是一个跨端框架：

上层：JavaScript 编写业务逻辑和 UI 描述（React 组件）
中层：Bridge（桥）负责 JS 与 Native 的通信
下层：Native 端负责实际 UI 渲染和设备功能调用（Android/iOS）

JS 层（React） → JS 引擎（JSCore/V8）
                 ↓
              Bridge（消息队列）
                 ↓
Native 层（Java/Kotlin + Android UI）
```
## 线程模型
```
RN 在 Android 上有三个主要线程：

UI Thread（主线程）
负责 Android 原生 UI 渲染和事件分发
所有 View 的创建、布局、绘制都在这里

JS Thread
运行 JS 代码（React 组件、业务逻辑）
由 JS 引擎（JSCore/V8）驱动

Shadow Thread（Layout）
负责计算布局（Yoga 引擎）
将布局结果传给 UI Thread
```
## 通信机制
```
RN 的 JS 与 Native 通信是异步的：

JS → Native：
JS 调用 Native 模块的方法，生成一条消息（JSON 格式）
通过 Bridge 放入消息队列
Native 线程读取消息，执行对应方法（比如创建 View）

Native → JS：
Native 事件（点击、网络回调）通过 Bridge 发送到 JS Thread
JS 执行对应回调函数

Bridge 是批量传输的：
避免频繁调用导致性能下降
但批量传输会造成 JS 与 UI 渲染之间有延迟
```
## 渲染流程
```
JS 层：
React 组件描述 UI（<View><Image/><Text/></View>）
运行在 JS Thread

布局计算：
React 组件树转成 Shadow Tree（Yoga 布局引擎）
Shadow Thread 计算布局

UI 更新：
布局结果通过 Bridge 发送到 UI Thread
UI Thread 创建 Native View（ImageView、TextView）

绘制：
Native View 在主线程绘制到屏幕
```
## 性能优化关键点
```
首屏加载优化：
预加载 JS Bundle（提前加载到内存）
缓存模板和资源

减少 Bridge 调用：
合并批量更新
避免频繁 JS ↔ Native 交互

图片/视频优化：
使用原生组件（FastImage、原生视频播放器）

布局优化：
减少复杂嵌套，避免 Yoga 布局计算过多
```
## 运转流程
```
服务端下发 Bundle，分两部分：
1.模板（UI 结构）
2.数据

客户端解析：
1.网络拉取 Bundle（JSON）
2.映射为 React 组件（模板 JSON 转为 React 元素）

绑定事件：
如点击事件

渲染：
1.JS Thread 运行 React 组件代码
2.Shadow Thread（Yoga）计算布局
3.Bridge 将布局结果和创建 View 的指令传给 UI Thread
4.UI Thread 创建 ImageView、TextView 并绘制

事件处理：
1.UI Thread 捕获点击事件
2.通过 Bridge 发送到 JS Thread
3.JS Thread 执行点击回调（上报事件到服务端）
```
