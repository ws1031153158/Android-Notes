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
# HashSet
内部无序，不允许重复元素，存的是一个对象，所以值也不同，也就只能有一个空对象，HashMap 允许值相同但 key 不能相同，HashTable 都不能为空。
# SparseArray
稀疏数组，类似 HashMap，某些场景性能更好，底层为两个数组分别存放 key 和 value，key 以升序排列，key 只能为 int，避免基本数据类型装箱操作，不使用额外结构体，节省空间 ，冲突直接覆盖原值，不返回。
一切操作以二分查找为基础，若二分查找该位置 value 标记为 delete（无冲突 key），则直接覆盖，若数组已满，则执行 gc，删除 delete，若无法 gc （无 delete）则扩容，remove 删除 value，gc 才删除 key，这样可以保持索引结果，且不会频繁更新数组结构。