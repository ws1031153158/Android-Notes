# Lifecycle
LifecycleObserver 是 Lifecycle 的观察者，viewmodel 默认就实现了 Lifececle 这个接口，LifecycleOwner 是具有生命周期的组件，如 Activity、Fragment，持有一个 Lifecycle 对象，Lifecycle 是 LifecycleOwner 的生命周期管理器，它定义了生命周期状态和转换关系，并负责通知 LifecycleObserver 状态变化的事件。
# LiveData
可感知生命周期，仅更新处于活动状态的组件观察者。  
1.生命周期发生变化时，LiveData 会通知对应组件(Observer)，数据发生变化时会通知更新 UI。  
2.Observer 绑定了 Lifecycle 对象， Lifecycle 对象destory 后， Observer 会自动清理。  
3.Activity 或 Fragment 重新创建（如屏幕旋转）会立即重新接收最新数据（liveData 只会向活跃态的 observer 发通知，可以感知 observer 的 lifecycle，避免内存泄露和空指针）。  
4.生命周期为非活跃状态，则会在非活跃态转为活跃态时接收最新数据（如后台切前台）。  
一般配合 viewModel 使用，将数据存储在 viewModel 中，分担 activity（fragment）压力，并将 activity（fragment）作为该 livedata 的观察者（持有一个观察者列表），自定义 ivedata 可以通过重写 onActive和onInactive 来满足业务需求，支持线程切换，可以在后台线程更新数据，然后在主线程中通知观察者更新UI。  
数据倒灌：  
现象：  
新的观察者开始注册观察时，会把上次发的最后一次的历史数据传递给当前注册的观察者。  
原因：  
LiveData 内部有 mVersion 字段记录版本，初始 mVersion 是-1，调用 setValue 或 postValue，其 mVersion 会 + 1；对于每一个观察者 ObserverWrapper，初始 mLastVersion 也为 -1，新注册的观察者 mLastVersion 为 -1； LiveData 设置 ObserverWrapper 时，如果 LiveData 的 mVersion 大于 ObserverWrapper 的 mLastVersion，LiveData 强制把当前 value 推送给 Observer。  
解决：  
1.改变每个 ObserverWrapper 版本号的值。  
2.保证第一次分发不响应。  
# ViewModel
## ViewBinding
### DataBinding
# WorkManager
# Compose
# Room
## Foundation
## Update
## Compatible
# Navigation
