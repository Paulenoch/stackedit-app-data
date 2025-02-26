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
      private char ch;  
      private boolean isKey;   
      private DataIndexedCharMap next;

      private Node(char c, boolean blue, int R) {
         ch = c; 
         isKey = blue;
         next = new DataIndexedCharMap<Node>(R);
      }
   }
}
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgxOTQzODA0NywtNTUxMzUwOTY2XX0=
-->