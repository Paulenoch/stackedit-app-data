# 1. 各集合框架底层实现
ArrayList：Object[]数组，
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
eyJoaXN0b3J5IjpbMTEyNDI4Mzk4OF19
-->