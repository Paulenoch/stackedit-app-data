Retrieval Trees(Trie)
![输入图片说明](/imgs/2025-02-26/tHWvBdCu8fFcgqla.png)
这个Trie包含key：a、awls、sad、sam、sap、same，每个结束节点需要特殊标记

对于contains()：两种情况会导致miss：结束节点不为白色，或者某个字母不在树上

### Implmentation
```java
public class TrieSet {
   private static final int R = 128; // ASCII
   private Node root;    // root of trie

   private static class Node {
      private boolean isKey;   
      private DataIndexedCharMap next;

      private Node(boolean blue, int R) {
         isKey = blue;
         next = new DataIndexedCharMap<Node>(R);
      }
   }
}
```
![输入图片说明](/imgs/2025-02-26/F9MUpts3IN5Hzx4H.png)
每个node中不需要存储element，因为128个指针和128个ASCII字符是对应的
性能：
-   `add`: Θ(1)
-   `contains`: Θ(1)


### 处理空间问题
上面implement的Trie会造成大量的空间被浪费，可以使用hash table存储![输入图片说明](/imgs/2025-02-26/VBAGpijnBeHkPhlX.png)
或者BTS![输入图片说明](/imgs/2025-02-26/dcNiFBymv9e8wBoh.png)

效率对比
-   DataIndexedCharMap
    -   Space: 128 links per node
    -   Runtime: Θ(1)Θ(1)
-   BST
    -   Space: C links per node, where C is the number of children
    -   Runtime: O(logR)O(logR), where R is the size of the alphabet
-   Hash Table
    -   Space: C links per node, where C is the number of children
    -   Runtime: O(R)O(R), where R is the size of the alphabet

Prefix Matching
返回Trie中所有的key
```
collect():
    Create an empty list of results x
    For character c in root.next.keys():
        Call colHelp(c, x, root.next.get(c))
    Return x

colHelp(String s, List<String> x, Node n):
    if n.isKey:
        x.add(s)
    For character c in n.next.keys():
        Call colHelp(s + c, x, n.next.get(c))
```

自动匹配
```
keysWithPrefix(String s):
    Find the end of the prefix, alpha
    Create an empty list x
    For character in alpha.next.keys():
        Call colHelp("sa" + c, x, alpha.next.get(c))
    Return x
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg5NTUwODAxNywtNTUxMzUwOTY2XX0=
-->