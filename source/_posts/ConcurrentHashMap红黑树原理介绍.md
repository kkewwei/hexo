---
title: ConcurrentHashMap红黑树原理介绍
date: 2017-11-06 17:48:34
tags:
toc: true
---
为了加快ConcurrentHashMap查找数据的速度, 若链表长度过长, 会将链表转化为红黑树, 本文主要讲解相关知识。
# 红黑树介绍
一棵二叉查找树如果满足下面的红黑性质, 则为一棵红黑树:
1）每个节点或者是黑的，或者是红的。
2）根节点是黑的。
3）每个叶子节点（NIL）是黑的。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
4）如果一个节点是红的，则它的两个儿子都是黑的。
5）对每个节点, 从该节点到其子孙节点的所有路径上包含相同数目的黑节点。
红黑树示例如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap3.png" height="300" width="350"/>
## 链表转化为红黑树
```
        TreeBin(TreeNode<K,V> b) {
            super(TREEBIN, null, null, null);
            this.first = b;
            //设置前置节点
            TreeNode<K,V> r = null;
            for (TreeNode<K,V> x = b, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                //初始化节点
                if (r == null) {
                    x.parent = null;
                    x.red = false;
                    r = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    //从根节点开始找到合适的地方存放
                    for (TreeNode<K,V> p = r;;) {
                        int dir, ph;
                        K pk = p.key;
                        //左边
                        if ((ph = p.hash) > h)
                            dir = -1;
                        //右边
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null && //继续比较，类名接口名、hash值等方面来比较
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);
                            TreeNode<K,V> xp = p;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) { //判断是否找到头了，
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            //开始调整
                            r = balanceInsertion(r, x);
                            break;
                        }
                    }
                }
            }
            this.root = r;
            assert checkInvariants(root);
        }
```
主要做了如下事情:
+ 初始化TreeBin节点, 设置其hash=TREEBIN(-2)
+ 设置first结点, 这个节点就是链表的头, 根据这个头, 可以遍历整个红黑树。
+ 根据first开始遍历红黑树, 针对每个节点hash值, 插入红黑树中合适的位置, 然后调整该二叉树以满足红黑色的要求(balanceInsertion)。
## 红黑树插入后平衡balanceInsertion
插入的节点为红色, 而可能导致红黑树出现红色的节点子节点还是红色, 违反4)条, 以下是调整过程:
```
        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
            x.red = true;//首先赋值头为red
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) { //如果插入的是头结点
                    x.red = false; //那么可以直接退出
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null) //xp是插入的x的par
                    //只有两层，x是红色不改变黑节点高度，只有par是red才需要调整（两个red不能相连）
                    return root;
                if (xp == (xppl = xpp.left)) { //父节点是左孩子
                    if ((xppr = xpp.right) != null && xppr.red) { //叔叔节点是红色的
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp; //x节点上移到祖父节点
                    }
                    else {
                        //叔叔节点是黑色节点，x是右节点，那么移动到左边
                        if (x == xp.right) {
                            //x变成xp的位置，结构没变，只是x指向变了
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) { //叔叔是黑色节点，x是左节点
                            xp.red = false;
                            if (xpp != null) { //那么右旋就行了
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                   ...
                }
            }
        }
```
首先设置插入的节点为红色, 若插入节点的父节点是黑色的,那么没有打破红黑树的原则, 可以不用调整直接退出, 当且仅当父节点是红色时候才需要调整, 这里主要讨论插入节点的父节点是祖父节点左孩子(又孩子一样)的情况, 分三种情况进行调整:
case1. 叔叔C是红色的
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap5.png" height="250" width="400"/>
调整: 父节点B与叔叔节点C变黑, 祖父节点A变红, 调整节点上移到祖父节点A
case2. 叔叔C是黑色的, 插入节点X是右孩子
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap6.png" height="250" width="400"/>
调整: 父节点B右旋, 将调整节点指向曾经的自己D, 变成case3的情况
case3. 叔叔C是黑色的, 插入节点X是左孩子
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap7.png" height="250" width="400"/>
调整: 祖父节点A右旋, 父节点B变黑, 祖父节A点变红, 调整完成, 退出函数。

## 红黑树删除后平衡balanceInsertion
红黑树删除分为两步:
+ 二叉树正常删除数据。红黑树在删除节点时, 若该节点做孩子或者右孩子最多只存在一个, 那么删除的就是本身节点; 若左右孩子同时存在, 那么结构上实际删除的节点将是`该节点的中序遍历的后继节点`, 然后再将实际删除的那个节点数据来置换理论上被删除的那个节点的数据, 所以:实际被删除的那个节点两个孩子不可能同时存在。
+ 删除节点后的调整。 删除后, 若删除的是黑色节点, 导致从该节点到其子孙节点的所有路径上包含相同不同数目的黑节点, 将违反第5)条。 以下是调整过程:
```
        //x点代表了一次删除的那个黑色
        static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                if (x == null || x == root)
                    return root;
                else if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                 //红色的，那么直接将红色变成黑色
                else if (x.red) {
                    x.red = false;
                    return root;
                }
                //开始分两种情况，都是一样的， x是左兄弟
                else if ((xpl = xp.left) == x) {
                 //第一种情况：有兄弟是红色的，先将兄弟变成红色的
                    if ((xpr = xp.right) != null && xpr.red) {
                        xpr.red = false;
                        xp.red = true;
                        root = rotateLeft(root, xp);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    if (xpr == null)//若有兄弟不存在，可以向上移动一个节点
                        x = xp;
                    else { //那么右兄弟为黑色的
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                         //右兄弟的两个孩子若存在，那么是黑色的
                            (sl == null || !sl.red)) {
                            //右兄弟将提取一个黑色，和本节点一起组成一个复黑色给父亲节点
                            xpr.red = true;
                            x = xp;
                        }
                        else { //右兄弟为黑色
                            //并且右兄弟的右孩子为若存在，是黑色。反过来说，左孩子一定存在，并且为红色
                            if (sr == null || !sr.red) {
                                if (sl != null)//这里判断没有意义，一定是red
                                    sl.red = false;
                                xpr.red = true;
                                root = rotateRight(root, xpr);//右旋
                                xpr = (xp = x.parent) == null ? //更新指向xpr和xp的指针
                                    null : xp.right;
                            }
                            //右兄弟为黑色，右兄弟的右孩子一定为red
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red;
                                if ((sr = xpr.right) != null)
                                    sr.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateLeft(root, xp);
                            }
                            x = root;//那么旋转结束
                        }
                    }
                }
                else { // symmetric
                   ...
                }
            }
        }
```
若删除了红节点, 那么对二叉树没有任何影响, 调整的都是删除节点是黑色的情况; 若删除节点的父节点是红色的,直接变成黑的就over, 这里仅仅讨论删除节点的父节点是左孩子(右孩子类似)的情况.
这里的x需要看成双重黑色, 平衡的目的是将这个双重黑色分摊到别的节点, 变成单黑色, 调整节点迁移到父节点
case1: 兄弟是红色的
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap12.png" height="350" width="400"/>
调整: 情况1: 若无右兄弟, 则将调整节点上移到父节点A; 若有右兄弟C, 对父节点A左旋, 父节点A变成红色, 兄弟节点C变成黑色
case2: 兄弟是黑色的, 兄弟的两个孩子(存在2个、1个、或者0个)没有一个是红色的
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap13.png" height="250" width="400"/>
调整: 将兄弟节点C变成红色, 调整节点上移到父节点C。
case3: 兄弟是黑色的, 兄弟的左孩子是红色。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap14.png" height="250" width="400"/>
调整: 右旋兄弟节点C, 兄弟节点左孩子变成黑色, 兄弟节点C变成红色, 变成case4。
case3: 兄弟是黑色的, 兄弟的右孩子是红色。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ConcurrentHashMap15.png" height="250" width="400"/>
调整: 左旋父节点A, 将父节点A的颜色给兄弟节点, 父节点A变成黑色, 调整完成, 退出函数。
# 总结
红黑树重点是定义, 每次插入节点或者删除节点都会破坏红黑树的性能, 然后再通过调整修复这些性质就好了。