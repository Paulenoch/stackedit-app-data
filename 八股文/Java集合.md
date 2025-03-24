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
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwNTcwOTkyMF19
-->