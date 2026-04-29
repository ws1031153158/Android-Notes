# HashMap
键值对存储，内部无序，重写 equals 需要同时重新 hashcode 方法。底层为数组 + 链表形式，到达一定长度需要扩容，冲突链表长度≥5 链表会改为红黑树。  
红黑树为自平衡二叉查找树，平衡指的是左右子树高度差不大于 1，查找指的是左子树 < 根节点 < 右子树，put 时若节点已经存在就替换旧值，保证 key 唯一。
## LinkedHashMap
与 HashMap 相比，内部是有序的，因为使用了双向链表，新增两个属性 before 和 after，但不是绝对有序，只针对放入顺序。
## HashTable
是线程安全的，使用了 synchronized 保证并发安全，没有 failed-fast 机制，只有一把锁，从头锁到尾，且直接对方法和 Hashtable 对象加锁，比较沉重。
## ConcurrentHashMap
线程安全，且使用了锁分段技术，默认 16 段 segment，继承 reentrantLock 来加锁，存储键值对,维护 segment 数组，类似一个 hashtable 但不会一次锁住整个表，而是将表分为一段一段的数据，对单独一段数据加锁，根据 hash 算法定位到对应 segment，效率更高。  
后续优化为节点锁，没有最大并发数限制，数据 + 链表 + 红黑树为结构，CAS 操作实现同步。
## TreeMap
结构为红黑树，效率慢，不像数组可以直接按 index 索引，且涉及到平衡的过程，优点是内部有序，因为是平衡二叉查找树，所以对 key 进行大小排序，不允许空 key 但可有空 value。
## Expend
1.进行扩容的时候，HashMap 的容量会变为原来的两倍，但是耗性能，若能估算出 map 的大小，初始化时给定数值，避免频繁扩容。  
2.使用带参数的构造函数，initialCapacity = 要存的元素个数/加载因子 + 1，loadFactor = default 为 0.75。暂时无法确认设置为 16（2 的 16 次方为最大数）。  
3.要添加新的元素时，当前容器元素个数 ≥ （数组长度 * 加载因子）时自动扩容，但是不一定，若将添加的元素在 map 中位置没有被占用则不会扩容。  
4.创建新的数组，将数据拷贝过去，重新计算 hashcode，原来的冲突可能就不存在了。  
5.加载因子默认为 0.75，扩容 2 倍是因为性能，容量需要是一个 2 的幂，使得通过 key 的 hashcode 找位置时 h & (length - 1) = h % length（使用 & 是因为效率更高），此外，JDK 1.8 采用尾插法，扩容保持原顺序，不会出现成环的问题。
## Tips
### keySet & entrySet
1.多次哈希查找：keySet() 遍历时，通过键调用 map.get(key) 方法取值，每次获取值时，都需要进行一次哈希查找操作。如果 HashMap 很大，效率就会明显降低。     
2.额外的内存消耗：keySet() 方法会生成一个包含所有键的集合，需要额外的内存开销来维护这个集合的结构，如果 HashMap 很大，这个内存开销也会变得显著。   
3.避免多次哈希查找：在遍历过程中，可以直接返回键和值（entry.getValue()），不需要再次进行哈希查找，提高效率。   
4.减少内存消耗：entrySet() 方法返回 HashMap 内部的一个视图，不需要额外的内存来存储键的集合。
# HashSet
内部无序，不允许重复元素，存的是一个对象，所以值也不同，也就只能有一个空对象，HashMap 允许值相同但 key 不能相同，HashTable 都不能为空。
# SparseArray
稀疏数组，类似 HashMap，某些场景性能更好，底层为两个数组分别存放 key 和 value，key 以升序排列，key 只能为 int，避免基本数据类型装箱操作，不使用额外结构体，节省空间 ，冲突直接覆盖原值，不返回。
一切操作以二分查找为基础，若二分查找该位置 value 标记为 delete（无冲突 key），则直接覆盖，若数组已满，则执行 gc，删除 delete，若无法 gc （无 delete）则扩容，remove 删除 value，gc 才删除 key，这样可以保持索引结果，且不会频繁更新数组结构。
# Stack
线程安全的（加锁），继承自 Vector，每个方法 push() / pop() / peek() 都加了 synchronized 锁 
## Deque
（LinkedList / ArrayDeque），不加锁，纯数组 / 链表操作，速度快。  

单线程没有竞争，但 synchronized 本身有固定开销，不是只有 “抢锁冲突” 才慢：  
获取对象锁：栈帧去拿当前对象的监视器锁   
内存屏障：强制刷新工作内存 & 主内存可见性   
锁升级逻辑：偏向锁→轻量级锁→重量级锁，JVM 要做一堆判断   
释放锁：方法结束，必须主动释放监视器   
这些操作：指令、判断、内存同步，全是额外耗时  
# PriorityQueue
优先级队列，默认：小顶堆。  
<img width="803" height="418" alt="image" src="https://github.com/user-attachments/assets/96f7ec0e-0869-45c3-88a2-5ad0b024c876" />  

## 底层
动态数组实现的 最小二叉堆（完全二叉树）  
插入、删除：O(logn)（堆是二叉树结构，树高 logn，上浮下沉最多遍历树高，所以是对数级复杂度）  
取堆顶：O(1)  

非线程安全，并发场景推荐 PriorityBlockingQueue  
不允许null，会直接空指针，也不支持不可比较的元素（没有实现 Comparable、也没指定比较器的自定义对象）  

大顶堆：PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a,b) -> b - a);  
## 二叉堆
底层数组存储，逻辑是完全二叉树   
添加：上浮；删除堆顶：下沉   
### 父子节点下标公式（核心） 
设当前节点下标：index：  
父节点——parent=(index−1)÷2   
左子节点——left=2×index+1   
右子节点——right=2×index+2   
举例：  
下标 1 的父：(1−1)/2=0   
下标 2 的父：(2−1)/2=0   
下标 1 左孩子：3，右孩子：4  
### 容量
1.初始化  
无参构造：DEFAULT_INITIAL_CAPACITY = 11   
有参构造：可手动指定初始容量   

2. 没有「负载因子」  
ArrayList/HashMap 有负载因子    
PriorityQueue 无负载因子  
只靠 元素个数 == 数组长度 时触发扩容    

3. 扩容源码规则    
容量 < 64：每次 +2   
容量 ≥ 64：扩容 1.5 倍（右移一位 /2 叠加）
<img width="536" height="247" alt="image" src="https://github.com/user-attachments/assets/8c9f4314-1a4e-4065-9c59-c09af4ce5c8f" />  

### 上浮 & 下沉
1.siftUp 上浮（新增元素、offer）：  
新元素加到数组末尾，不断和父节点比较，太小就往上浮，维持小顶堆。  
从最后一个位置开始 ，比父节点小 → 父下来，自己往上 ，直到不小于父 / 到根节点  
<img width="329" height="359" alt="image" src="https://github.com/user-attachments/assets/67aa054b-f34e-4b43-bc75-3f8234fc415b" />  


siftDown 下沉（删除堆顶、poll）：  
删除堆顶后，把末尾元素挪到堆顶，不断和左右子节点比较，和更小的子节点交换，向下沉。  
堆顶换成末尾元素 ，找左右小孩最小值 ，自己比最小值大，就下沉交换 ，直到满足堆规则  
<img width="604" height="447" alt="image" src="https://github.com/user-attachments/assets/dbfe2d40-a047-43cb-ae88-143c7065b410" />  


扩容流程：  
offer 添加元素  
容量满 → 先扩容    
元素放数组末尾   
调用 siftUp 上浮   
poll 删除堆顶  
取出下标 0 元素   
把最后一位元素放到下标 0   
调用 siftDown 下沉重构堆  
### 负载因子
负载因子 = 一个 “提前扩容的警戒线”  
不是等容器满了才扩容，而是用到一定比例就提前扩容，防止性能下降。  

控制空间和效率的平衡：  
负载因子越小 → 越不容易冲突，查询越快，但越占空间   
负载因子越大 → 越省空间，但越容易冲突，变慢   
提前扩容，避免卡顿，不是等满了再扩容，而是提前扩容，保证性能平稳。  
  



