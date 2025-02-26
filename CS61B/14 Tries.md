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
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTY2MjAxMjkxOSwtNTUxMzUwOTY2XX0=
-->