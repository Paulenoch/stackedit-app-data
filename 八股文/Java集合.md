
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

## Set
### 无序性和不可重复性的含义是什么
-   无序性不等于随机性 ，无序性是指存储的数据在底层数组中并非按照数组索引的顺序添加 ，而是根据数据的哈希值决定的
-   不可重复性是指添加的元素按照 `equals()` 判断时 ，返回 false，需要同时重写 `equals()` 方法和 `hashCode()` 方法

### 比较HashSet、LinkedHashSet和TreeSet
-   `HashSet`、`LinkedHashSet` 和 `TreeSet` 都是 `Set` 接口的实现类，都能保证元素唯一，并且都不是线程安全的。
-   `HashSet`、`LinkedHashSet` 和 `TreeSet` 的主要区别在于底层数据结构不同。`HashSet` 的底层数据结构是哈希表（基于 `HashMap` 实现）。`LinkedHashSet` 的底层数据结构是链表和哈希表，元素的插入和取出顺序满足 FIFO（先入先出）。`TreeSet` 底层数据结构是红黑树，元素是有序的，排序的方式有自然排序和定制排序。
-   底层数据结构不同又导致这三者的应用场景不同。`HashSet` 用于不需要保证元素插入和取出顺序的场景，`LinkedHashSet` 用于保证元素的插入和取出顺序满足 FIFO 的场景，`TreeSet` 用于支持对元素自定义排序规则的场景。

## Queue
### ArrayDeque和LinkedList的区别
`ArrayDeque` 和 `LinkedList` 都实现了 `Deque` 接口，两者都具有队列的功能，但两者有什么区别呢？

-   `ArrayDeque` 是基于可变长的数组和双指针来实现，而 `LinkedList` 则通过链表来实现。
-   `ArrayDeque` 不支持存储 `NULL` 数据，但 `LinkedList` 支持。
-   `ArrayDeque` 插入时可能存在扩容过程, 不过均摊后的插入操作依然为 O(1)。虽然 `LinkedList` 不需要扩容，但是每次插入数据时均需要申请新的堆空间，均摊性能相比更慢。
    

从性能的角度上，选用 `ArrayDeque` 来实现队列要比 `LinkedList` 更好。此外，`ArrayDeque` 也可以用于实现栈。

### PriorityQueue
-   `PriorityQueue` 利用了二叉堆的数据结构来实现的，底层使用可变长的数组来存储数据
-   `PriorityQueue` 通过堆元素的上浮和下沉，实现了在 O(logn) 的时间复杂度内插入元素和删除堆顶元素。
-   `PriorityQueue` 是非线程安全的，且不支持存储 `NULL` 和 `non-comparable` 的对象。
-   `PriorityQueue` 默认是小顶堆，但可以接收一个 `Comparator` 作为构造参数，从而来自定义元素优先级的先后。

### BlockingQueue
`BlockingQueue` （阻塞队列）是一个接口，继承自 `Queue`。`BlockingQueue`阻塞的原因是其支持当队列没有元素时一直阻塞，直到有元素；还支持如果队列已满，一直等到队列可以放入新元素时再放入。
`BlockingQueue` 常用于生产者-消费者模型中，生产者线程会向队列中添加数据，而消费者线程会从队列中取出数据进行处理。
![输入图片说明](/imgs/2025-03-28/NHj0lQPFl45VVquQ.png)







# HashMap
## HashSet如何检查是否重复
当把对象加入`HashSet`时，`HashSet` 会先计算对象的`hashcode`值来判断对象加入的位置，同时也会与其他加入的对象的 `hashcode` 值作比较，如果没有相符的 `hashcode`，`HashSet` 会假设对象没有重复出现。但是如果发现有相同 `hashcode` 值的对象，这时会调用`equals()`方法来检查 `hashcode` 相等的对象是否真的相同。如果两者相同，`HashSet` 就不会让加入操作成功。

## HashMap的底层实现
HashMap 通过 key 的 `hashcode` 经过扰动函数处理过后得到 hash 值，然后通过 `(n - 1) & hash` 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

`HashMap` 中的扰动函数（`hash` 方法）是用来优化哈希值的分布。通过对原始的 `hashCode()` 进行额外处理，扰动函数可以减小由于糟糕的 `hashCode()` 实现导致的碰撞，从而提高数据的分布均匀性。

当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树。

### 为什么选择阈值 8 和 64

1.  泊松分布表明，链表长度达到 8 的概率极低（小于千万分之一）。在绝大多数情况下，链表长度都不会超过 8。阈值设置为 8，可以保证性能和空间效率的平衡。
2.  数组长度阈值 64 同样是经过实践验证的经验值。在小数组中扩容成本低，优先扩容可以避免过早引入红黑树。数组大小达到 64 时，冲突概率较高，此时红黑树的性能优势开始显现。

### HashMap的长度为什么是2的幂次方
计算出哈希值之后需要对数组的长度进行取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。若此时数组长度为2的幂次方，取余(%)操作中如果除数是 2 的幂次则等价于与其除数减一的与(&)操作，提高运算速度

长度是 2 的幂次方，可以让 `HashMap` 在扩容的时候更均匀。例如:
当数组从 `oldCap`（旧容量）扩容到 `newCap = 2 * oldCap` 时，每个键的新位置由以下步骤决定：

#### (1) **哈希值的二进制位分析**

假设旧容量为 `oldCap = 16`（二进制 `10000`），扩容后 `newCap = 32`（二进制 `100000`）。  
键的哈希值为 `hash`，其二进制格式如下（以 5 位为例）：


`hash = ... XXXXX （最后 5 位）`

#### (2) **新增的最高有效位**

旧容量的索引计算方式为：  
`oldIndex = (oldCap - 1) & hash`  
此时 `oldCap - 1 = 15`（二进制 `1111`），仅取哈希值的低 4 位。

扩容后，新容量的索引计算为：  
`newIndex = (newCap - 1) & hash`  
此时 `newCap - 1 = 31`（二进制 `11111`），取哈希值的低 5 位。

**关键点**：  
新增的第 5 位（原哈希值的第 5 位，即 `oldCap` 对应的位）决定了键是否需要移动：

-   **如果第 5 位为 `0`**：新索引 = 原索引（留在原位置）。
    
-   **如果第 5 位为 `1`**：新索引 = 原索引 + 旧容量（移动到新位置）。

这一机制使得扩容时的数据迁移时间复杂度为 **O(n)**（遍历所有键值对），但分摊到每次操作仍是 **O(1)**。

扩容完成后，所有查找直接基于新数组的长度重新计算索引，无需额外判断。该机制确保键总能被正确映射到新位置，无论其是否在扩容时移动过。



## 2. HashMap为什么线程不安全
在 `HashMap` 中，多个键值对可能会被分配到同一个桶（bucket），并以链表或红黑树的形式存储。多个线程对 `HashMap` 的 `put` 操作会导致线程不安全，具体来说会有数据覆盖的风险。

## 3. ConcurrentHashMap和Hashtable的区别
`ConcurrentHashMap` 直接用 `Node` 数组+链表+红黑树的数据结构来实现，并发控制使用 `synchronized` 和 CAS 来操作。
`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，就不会影响其他 Node 的读写，效率大幅提升。

**`Hashtable`(同一把锁)** :使用 `synchronized` 来保证线程安全，效率非常低下。

## ConcurrentHashMap为什么Key和Value不能为null
`ConcurrentHashMap` 的 key 和 value 不能为 null 主要是为了避免二义性。null 是一个特殊的值，表示没有对象或没有引用。如果你用 null 作为键，那么你就无法区分这个键是否存在于 `ConcurrentHashMap` 中，还是根本没有这个键。同样，如果你用 null 作为值，那么你就无法区分这个值是否是真正存储在 `ConcurrentHashMap` 中的，还是因为找不到对应的键而返回的。

拿 get 方法取值来说，返回的结果为 null 存在两种情况：

-   值没有在集合中 ；
-   值本身就是 null。

这也就是二义性的由来。

## ConcurrentHashMap可以保证复合操作的原子性吗
不能
```java
// 线程 A
if (!map.containsKey(key)) {
map.put(key, value);
}
// 线程 B
if (!map.containsKey(key)) {
map.put(key, anotherValue);
}
```
如果线程 A 和 B 的执行顺序是这样：

1.  线程 A 判断 map 中不存在 key
2.  线程 B 判断 map 中不存在 key
3.  线程 B 将 (key, anotherValue) 插入 map
4.  线程 A 将 (key, value) 插入 map

那么最终的结果是 (key, value)，而不是预期的 (key, anotherValue)。这就是复合操作的非原子性导致的问题。

`ConcurrentHashMap` 提供了一些原子性的复合操作，如 `putIfAbsent`、`compute`、`computeIfAbsent` 、`computeIfPresent`、`merge`等。这些方法都可以接受一个函数作为参数，根据给定的 key 和 value 来计算一个新的 value，并且将其更新到 map 中。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzIxMTMyODQ5LC0xMTA4MTM5MzAwLDIwMD
Q4NDM5NTUsLTE2NzI1OTEyMywxMTI0MjgzOTg4XX0=
-->