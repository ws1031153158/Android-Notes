# Hash
Map、Set，不重复的集合作为某种标准，map 类似，值可以重复  
# Array
反转、temp、给定k次，或某个确定值，先看是否需要取余，分治法，对于某一个指定位置或其他阈值，分为左右分别处理  
# ArrayList
new ArrayList<>() 构造时，括号中可以直接传入参数类型匹配的集合，如 map.values
# 最大公约数
（a(x, y) -> y > 0 ? a(y, x % y) : x  
# 裴蜀定理
对任何整数 a、b 和它们的最大公约数 d，	对于任意整数 x，y，ax + by 都一定是 d 的倍数，	一定存在整数 x，y，使 ax + by = d 成立  
# Tree
dfs、bfs，构建时考虑 hashmap，给节点加上编号，记录数量等问题时可以考虑  
# 动态规划
下一步由上一步推出，一般会有公式  
# 贪心
求最优，每一步都采取当前最优解 
# 前缀和
求某一段区间元素之和，或某个属性的和/次数等，可以直接 s[n] = a[1] +...+a[n] 求前缀和，也可以 x = s[n] - s[3] 求具体某一段，具体求法为一种动态规划 s[n] = s[n - 1] + a[n]  
# 字典树
前缀树 trie，存储和检索字符串数据集中的键  
class Trie {  
        Map<Character, Trie> child;//key类型可变  
        public Trie() {  
            child = new HashMap<>();  
        }  
    }  
# 时间复杂度
20 ——2^n  
50——n^4  
200——n^3  
2000——n^2  
20000——nlogn  
50000——n  
没有范围则从 n 开始，一般为 n——nlogn，有可能要累加考虑，如排序 + 双指针为 n*n  
n：数组：差分、前缀和、双指针、排序、单调栈、单调队列（滑动窗口）  
# 树/图
dfs/bfs、拓扑排序、字典树（前缀树 trie）  
# nlogn
最短路径、二叉搜索树、归并排序、快排、堆、最小生成树、ST表、线段树、树状数组  
# logn
二分、并查集、快速幂、最大公因数     
# PriorityQueue
给每个元素都分配一个数字来标记其优先级，设较小数字具有较高优先级，这样就可以在一个集合中访问优先级最高的元素并对其进行查找和删除操作，保证每次出队都是最大或最小元素（取决于对 Comparator.compare 的重写）。此外，不能插入 null，空间不足会自动扩容。     
# 堆
其实就是完全二叉树（某节点值总是不大于或不小于父节点值）进了一些调整，
# 快慢指针
head 不是虚节点，fast 初始化需要是 head.next（如找链表中点）
# 位运算
n & n - 1：表示二进制数去掉最右边的一个 1
n & 1：表示对 2 取余
0 与 任何数 ^ 都等于数本身
任何数与自身 ^ 都等于 0
# 回溯
需要求出解集（全部解），或者要求回答什么解是满足某些约束条件的最优解时，往往要使用回溯法  
if (index == n) {  
  do something  
} else {  
  for () {  
    add(index)  
    backtrack(index + 1)  
    remove(index)  
  }  
}  
## Tips
1.temp 为 List，res 也为 List 时，res.add 需要为 add(new ArrayList<>(temp))  
2.一般 temp 为 int，则 remove 的是 temp.size() - 1，如果为 char，则是 deleteCharAt(index)
# 链表
哑巴节点：dummy 的 next 指向 head，这样无需判断头节点特殊情况（需要 pre 的时候）  
栈：先将节点入栈，当指定位置再出栈  
快慢指针：环、类似栈（找指定位置，n + m = m + n）
# 二分
while 的条件需要是 l <= r
# 字母异位词
一般可以通过各个字母数量相同来判断，也可以直接 toCharArray，排序后相同来判断
