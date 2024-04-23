# Lifecycle
LifecycleObserver 是 Lifecycle 的观察者，viewmodel 默认就实现了 Lifececle 这个接口，LifecycleOwner 是具有生命周期的组件，如 Activity、Fragment，持有一个 Lifecycle 对象，Lifecycle 是 LifecycleOwner 的生命周期管理器，它定义了生命周期状态和转换关系，并负责通知 LifecycleObserver 状态变化的事件。
1.实现 “生命周期组件管理” 代码修改的一致性。   
2.使第三方组件 随时可在自己内部拿到生命周期状态，以便执行 及时叫停 “错过时机” 异步业务 等操作。   
3.使第三方组件调试时，能安全便捷追踪到 “事故所在生命周期源”。
# LiveData
可感知生命周期，仅更新处于活动状态的组件观察者。  
1.生命周期发生变化时，LiveData 会通知对应组件(Observer)，数据发生变化时会通知更新 UI。  
2.Observer 绑定了 Lifecycle 对象， Lifecycle 对象destory 后， Observer 会自动清理。  
3.Activity 或 Fragment 重新创建（如屏幕旋转）会立即重新接收最新数据（liveData 只会向活跃态的 observer 发通知，可以感知 observer 的 lifecycle，避免内存泄露和空指针）。  
4.生命周期为非活跃状态，则会在非活跃态转为活跃态时接收最新数据（如后台切前台）。  
一般配合 viewModel 使用，将数据存储在 viewModel 中，分担 activity（fragment）压力，并将 activity（fragment）作为该 livedata 的观察者（持有一个观察者列表），自定义 ivedata 可以通过重写 onActive和onInactive 来满足业务需求，支持线程切换，可以在后台线程更新数据，然后在主线程中通知观察者更新UI。  
5.它不支持线程切换，其次不支持背压，也就是在一段时间内发送数据的速度 > 接受数据的速度，LiveData 无法正确的处理这些请求。  
6.使用 LiveData 的最大问题是所有数据转换都将在主线程上完成。    
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
4.需要添加 layout属性（viewBinding 不用），xml 多了一个 <data/> 标签，填写布局文件需要引用的类或变量。  
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
可返回 Coroutine、RxJava 等库类型结果。  
Database：数据库入口，继承 RoomDatabase，提供获取 DAO 抽象方法，关联 table 对应 entities，使用注解 @Database(entities,version) 来标识。  
Entity：一个实体代表一个表，属性对应表中 column，必须为 public 或有 get/set 方法，至少一个主键（@PrimaryKey 单个主键），使用注解 @Entity(primaryKeys)来标识，（autoGenerate 自动生成），tableName 指定表名，indices 指定索引，unique 设为唯一索引，注解 @Ignore 标识不持久化进数据库。  
DAO(Data Access Object)：数据库访问者，提供增删改查接口，运行在调用线程，需要做异步处理，注解 @Embeded 标识将属性内嵌到 Entity，相当于包含了多个 Entity 属性。  
一对一：主表（Parent Entity）每条记录与从表（Child Entity）一一对应，此外，foreignkeys 作为注解 @Relation 的属性来定义外键约束。外键只能在从表，从表需要有字段对应到主表主键。通过外键约束（delete/update发出此约束属性），对主表操作受从表影响。（如主表（外键来源表）删除对应记录，先检查该记录是否有对应外键，有则不可删除），注解 @Relation 定义主从关系，parent 为主表主键，entity 为从表外键约束。  
一对多：主表一条记录对应从表中零到多个。    
多对多：主表一条记录对应从表中零到多个，此外，若没有明确的外键约束关系，则需要定义一个 associative entity（交叉连接表），分别建立外键约束，交叉结果为笛卡尔积（两表记录和）。 
## Foundation
编译期通过 kapt 处理 @Dao 和 @Database 注解，生成 DAO 和 Database 实现类(XXX_impl)。  
database_impl：  
1.createOpenHelper：通过 Room.databaseBuilder.build 创建 Database，接着调用 impl.createOpenHelper 创建 SupportSQLiteOpenHelper(创建 DB 以及管理版本)。  
2.createInvalidationTracker ：创建跟踪器，确保 table 记录修改时通知到相关回调方。  
3.clearAllTables：清空 table 实现    
4.xxxDao：创建 xxx_impl   
5.xxxDao_impl：  
1.__db：RoomDatabase 实例  
2.__insertionAdapterOfUser ：EntityInsertionAdapterd 实例，用于数据 insert  
3.__deletionAdapterOfUser：EntityDeletionOrUpdateAdapter 实例，用于数据 update/delete  
Builder：  
1.createFromAsset/createFromFile ：从 SD 卡或 Asset 的 db 文件创建 RoomDatabase 实例  
2.addMigrations() ：添加数据库迁移（migration），数据库升级需要  
3.allowMainThreadQueries() ：允许在UI线程进行数据库查询，默认不允许  
4.fallbackToDestructiveMigration() ：找不到 migration 则重建数据库表（会造成数据丢失）  
5.调用 build 后，创建 xxxDatabase_Impl，并调用 init，内部调用 createOpenHelper  
## Update
1.数据库表结构变化时，需要通过数据库迁移（Migrations）升级表结构，避免数据丢失    
2.迁移需要使用 Migration 类，Migration(int,int)，重写 migrate 方法，执行 db.execSQL    
3.Migration 通过 startVersion 和 endVersion 表明当前是哪个版本间的迁移，运行时按照版本顺序调用各 Migration，迁移到新 Version    
4.迁移找不到对应版 Migration，会抛出 IllegalStateException，添加降级处理，避免 crash（.fallbackToDestructiveMigration）  
fallbackToDestructiveMigration：重建数据库表  
fallbackToDestructiveMigrationFrom：基于某版本重建数据库表  
fallbackToDestructiveMigrationOnDowngrade：数据库表降级到上一个正常版本  
## Compatible
LiveData：DAO 可定义 LiveData 类型结果，Room 内部兼容 LiveData 响应式逻辑，通常 Query 需要命令式获取结果，LiveData 让更新可被观察，DB 数据发生变化时，Room 会更新 LiveData，将查询语句作为局部变量，RoomSQLitQuery.acquire 来获取数据，返调用 db.getInvalidationTracker.createLiveData（接受3个参数：tableNames：被观察的表；inTransaction：查询是否基于事务；computeFunction 记录变化时的回调），重写 call 方法（执行真正的 sql 查询，Observer 首次订阅 LiveData，或表数据发生变化时执行），以及 finalize 方法（进行 release 操作）。  
RxJava：DAO 返回值可以是 RxJava2 的各种类型，注解 @Query 可请求 Flowable/Observable 类型、注解 @Insert/@Update/@Delete 设置属性 Completable/Single/Maybe，使用 fromCallable 创建 Completable 与 Single；RxRoom.createFlowable 创建 Flowable，call 方法执行真正的 sql 操作。    
Coroutine：CURD 方法定义为 suspend，CoroutinesRoom.execute 执行真正的 sql 语句，通过 Continuation 将 callback 变为 Coroutine 的同步调用。
# Navigation
单 Activity 架构成为可能，无需关心具体的 fragment 跳转逻辑，但虽然在 onCreateView 中创建 FrameLayout，真正的容器却是 FrameLayout。创建 Fragment 时内部使用的是 replace，不是 show 和 hide，导致 fragment 每次生命周期都会重新执行，可以和 viewModel 配合使用。此外，需要在跟目录中注册，在各个 fragment 中编写 action，指定跳转路由（destination/popUpTo），在根节点声明 startDestination，fragment 可定义 argument，在跳转时传递参数，指定 name 为 NavHostFragment 的 fragment 为导航容器。defaultNavHost 为 true 时需要在 activity 重写 onSupportNavigateUp（默认 back 事件不会回退 fragment）。  
通过 Navigation.findNavcontroller(fragment).navigate/Up(action) 实现导航或点击逻辑，传统传递 bundle 来在 fragment 之间传递参数，在 navigate 方法中添加 arg 参数，但需要定义多个 name 容易混淆。  
谷歌提供了 safeArgs 方式，为 fragment 提供 directions 文件（用于传参），有 argument 标签的自动生成 Args 文件（用于取参）。
# Compose
