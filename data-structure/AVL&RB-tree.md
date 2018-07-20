# 数据结构-AVL树和红黑树的对比
---------------------------------------------------------------

1.红黑树并不追求“**完全的平衡**”，它只要求达到部分的平衡，降低了对旋转的要求，从而提高了性能。

2.红黑树能够以**O(log2 n)** 的时间复杂度进行搜索、插入、删除操作，由于它的设计，任何不平衡都会**在三次旋转之内**解决。

3.还有一些更好的，但实现起来更复杂的数据结构能够做到一步旋转之内达到平衡，但红黑树能够给我们一个比较“便宜”的解决方案。**红黑树的算法时间复杂度和AVL相同，但统计性能比AVL树更高**

4.红黑树的**插入/删除效率更高**

5.**红黑树的平衡要求不如AVL树严格，理论上search要慢些，实际也如此，不过差距并不大**



# 总结：
---------------------------------------------------------------

#### 查找比较

显然，**avl树要比红黑树更平衡，因此avl树的查找效率更高。**

#### 插入比较

如果插入一个node引起了树的不平衡，AVL和RB-Tree都是最多只需要2次旋转操作，即两者都是O(1)

#### 删除比较

在删除node引起树的不平衡时，最坏情况下，**AVL需要维护从被删node到root这条路径上所有node的平衡性，因此需要旋转的量级O(logN)，而RB-Tree最多只需3次旋转，只需要O(1)的复杂度。**

**AVL的结构相较RB-Tree来说更为平衡，在插入和删除node更容易引起Tree的unbalance，因此在大量数据需要插入或者删除时，AVL需要rebalance的频率会更高。因此，RB-Tree在需要大量插入和删除node的场景下，效率更高。自然，由于AVL高度平衡，因此AVL的search效率更高。**

**红黑树的查询性能略微逊色于AVL树，因为他比avl树会稍微不平衡最多一层，也就是说红黑树的查询性能只比相同内容的avl树最多多一次比较，但是，红黑树在插入和删除上完爆avl树，avl树每次插入删除会进行大量的平衡度计算，而红黑树为了维持红黑性质所做的红黑变换和旋转的开销，相较于avl树为了维持平衡的开销要小得多**