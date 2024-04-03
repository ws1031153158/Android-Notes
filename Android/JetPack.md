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
lifecycle：只在需要销毁时执行 onCleard。  
ViewModelProvider：管理 ViewModel 实例的创建（一般通过默认工厂，并传入 ViewModelStore(viewModelStoreowner 接口通过getViewModelStore获取，为空会新建)）和获取(.get(XX::Class))，根据 key 从 store 获取，store 没有则重新创建.  
ViewModelStore：存储 ViewModel 的容器，用于存储与某个特定的生命周期相关联的 ViewModel（全局容器，底层是 HashMap），并指定唯一 key 标识，全局保存（脱离 Activity 或 Fragment，只是移除引用，根据业务来设计单例和确定回收时机），会被 GC 回收或执行 clear 方法。另外，通过 onRetaionNonConfigurationInstance 保存 ViewModelStore，调用 getLastNonConfigurationInstance 恢复数据，若获取为空则重新创建。  
activity/fragment 提供 viewModelStore，通过viewModelProvider(activity).get 创建时传入自身，activity/fragment 销毁时调用 viewModel.clear，正常 back 事件 finish 掉 activity 时 viewModel 会销毁。
## ViewBinding
一个 layou t对应一个 viewBinding 对象，会在指定目录下系统生成一个对应 xml 资源的 binding 类，实现默认方法。（编译时通过一个 gradle task 扫描 layout 文件生成）通过 viewBinding.inflate 获取 binding（最终使用的还是findViewById（通过参数获取 rootView）），binding.root 就是 layout。  
子 view 通过 viewBinding.bind（parentBinding）与父布局进行绑定，之后就可以通过 binding 获取到 xml 中的各个控件（子 view）。  
Null 安全：视图绑定会创建对视图的直接引用，不存在视图 ID 无效引发 Null 指针异常。此外，如果视图仅出现在布局的某些配置中（比如横竖屏布局内容差异），则绑定类中包含其引用的字段会使用 @Nullable 标记。  
类型安全：绑定类中字段具有与它们在 XML 文件中引用的视图相匹配的类型，布局和代码的不兼容会在编译时（而非运行时）失败。
### DataBinding
viewBinding：  
1.只支持视图绑定  
2.效率高，避免了数据绑定的开销  
3.避免了空指针异常（相对 kotlin 扩展）  
dataBinding：    
1.包含了 viewBinding 功能  
2.支持 data 和 view 双向绑定  
3.效率低  
4.需要添加 layout属性（viewBinding 不用）  
xml 多了一个 <data/> 标签，填写布局文件需要引用的类或变量。  
DataBidingUtil.setContentView：首先是 activity.setContentView，activity 获取 window，window 再获取 decorView，最后通过 decorView 获取到 viewGroup，最终执行 bindToAddedViews（遍历获取 view 数组对象，通过数据绑定 library 生成对应 binding 类）。  
bind：在父类（viewBinding）创建回调或 handler，通过 mapBinding 遍历布局获取 includes、ID Views 的数组对象并赋给对应 view 最终创建 runnnable 执行动态绑定。  
设置变量：bean 对象通过一个 mDirtyFlags 标识变量，通过 notifyPropertyChanged 通知更新（调用到设置的回调，通知对象属性变化，转给生成的 binding 类处理（判断是否需要重新绑定并执行）），调用 requestRebind 重新绑定，将数据更新到 view，observable 对象注册一个对象的监听（保存在 local 数组中，addOnPropertyChangedCallback 将监听绑定到对象上（会创建通知 obervable 重新绑定更新的回调并添加到回调列表）） ，后续和 bean 对象相同，此外，observableFields 对象的监听在 executeBindings（上述 runnable 中的最后一步（真正绑定的方法））注册。  
事件：生成 Binding 类中实现 View 事件监听，在构造时实例化监听，在绑定时将事件监听对象赋值给对应 View。  
viewStub：利用 ViewStubProxy 延迟绑定（使用 layout 中 ViewStub 实例化 ViewStubProxy 对象赋 viewStub 变量，并与 Bingding 关联，在 infate 时执行 mProxyListener，生成 ViewStub的Binding，强制执行主 Binding 重绑）。
# WorkManager
AndroidX 库所有，用于替代后台 Service（特定时间执行后台任务或大型操作以及重复性任务），拥有更加严谨的监管机制，更好的性能，重写 doWork 方法执行任务，有返回值。
# Compose
避免布局嵌套多次测量问题，引入固有特性测量（Intrinsic），允许父对子测量前，先测子的固有尺寸（先对整个 viewTree 进行固有特性测量，在对整体进行正常测量），将命令式编程变为声明式编程。  
# Room

## Foundation
## Update
## Compatible
# Navigation
