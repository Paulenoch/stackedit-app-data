# 1. 各集合框架底层实现
## List
底层是Object[]数组，支持快速随机访问，适用于频繁的查找工作，线程不安全

可以添加null值，但不建议因为无意义，而且会让代码难以维护

### ArrayList和LinkedList对比
- 二者都线程不安全
- ArrayList是Object[]数组，LinkedList是双向列表
- LinkedList不支持快速随机访问
- `ArrayList` 的空间浪费主要体现在在 list 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间（因为要存放直接后继和直接前驱以及数据）

### ArrayList扩容机制
以无参数构造方法创建 `ArrayList` 时，实际上初始化赋值的是一个空数组。当真正对数组进行添加元素操作时，才真正分配容量。即向数组中添加第一个元素时，数组容量扩为 10。

`int newCapacity = oldCapacity + (oldCapacity >> 1)`,所以 ArrayList 每次扩容之后容量都会变为原来的 1.5 倍左右（oldCapacity 为偶数就是 1.5 倍，否则是 1.5 倍左右）
	
### 集合中的 fail-fast 和 fail-safe 是什么
快速失败的思想即针对可能发生的异常进行提前表明故障并停止运行，通过尽早的发现和停止错误，降低故障系统级联的风险。
















LinkedList：双向列表

HashSet：基于HashMap实现
LinkedHashSet：基于LinkedHashMap
TreeSet：红黑树

PriorityQueue：Object[]数组实现小顶堆
DelayQueue：PriorityQueue
ArrayDeque：可扩容动态双向数组

HashMap：数组+链表/数组+红黑树（数组长度不满64先扩容数组，大于等于64后链表转红黑树）
LinkedHashMap：在HashMap的基础上增加了一条双向链表，可以保持键值对插入的顺序
HashTable：数组+链表
TreeMap：红黑树

# 2. HashMap为什么线程不安全
在 `HashMap` 中，多个键值对可能会被分配到同一个桶（bucket），并以链表或红黑树的形式存储。多个线程对 `HashMap` 的 `put` 操作会导致线程不安全，具体来说会有数据覆盖的风险。

# 3. ConcurrentHashMap和Hashtable的区别
`ConcurrentHashMap` 直接用 `Node` 数组+链表+红黑树的数据结构来实现，并发控制使用 `synchronized` 和 CAS 来操作。
`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，就不会影响其他 Node 的读写，效率大幅提升。

**`Hashtable`(同一把锁)** :使用 `synchronized` 来保证线程安全，效率非常低下。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAwNDg0Mzk1NSwtMTY3MjU5MTIzLDExMj
QyODM5ODhdfQ==
-->