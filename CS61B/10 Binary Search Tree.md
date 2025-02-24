### search()
```java
static BST find(BST T, Key sk) {
   if (T == null)
      return null;
   if (sk.equals(T.key))
      return T;
   else if (sk ≺ T.key)
      return find(T.left, sk);
   else
      return find(T.right, sk);
}
```

### insert()
```java
static BST insert(BST T, Key ik) {
  if (T == null)
    return new BST(ik);
  if (ik ≺ T.key)
    T.left = insert(T.left, ik);
  else if (ik ≻ T.key)
    T.right = insert(T.right, ik);
  return T;
}
```

### delete()
1. No children：直接删除即可
2. One child：删除该节点，将该节点的子节点连接到该节点的父节点上
3. Two children(**Hibbard deletion**)：
![输入图片说明](/imgs/2025-02-24/NiMWXwGQNkClRHFY.png)
若删除dog，首先找到其左边子树最右侧的节点（cat）或者右边子树最左侧的节点（elf），用这个节点替代dog，并删除该节点（不必担心被删除的节点有两个child节点）
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTcwMTcwMjc4N119
-->