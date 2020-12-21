HashMap:

数组+链表
put时，放最后：p.next = newNode
到达阈值则将链表转为红黑树treeifyBin

resize：2的n次方 有利于将取模变为按位与操作，提升性能。

到达阈值扩容，重新哈希