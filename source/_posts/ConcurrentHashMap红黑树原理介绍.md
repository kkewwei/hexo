---
title: ConcurrentHashMap红黑树原理介绍
date: 2017-11-06 17:48:34
tags:
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
<img src="http://owsl7963b.bkt.clouddn.com/ConcurrentHashMap3.png" height="400" width="450"/>
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
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```
首先设置插入的节点为红色, 若插入节点的父节点是黑色的,那么没有打破红黑树的原则, 可以不用调整直接退出, 当且仅当父节点是红色时候才需要调整, 这里主要讨论插入节点的父节点是祖父节点左孩子(又孩子一样)的情况, 分三种情况进行调整:
1. 叔叔是红色的
<img src="http://owsl7963b.bkt.clouddn.com/ConcurrentHashMap5.png" height="400" width="450"/>
调整:
父节点与叔叔节点变黑, 祖父节点变红, 调整节点上移到祖父节点
2. 叔叔是黑色的, 插入节点是右孩子
<img src="http://owsl7963b.bkt.clouddn.com/ConcurrentHashMap5.png" height="400" width="450"/>
调整:
父节点右旋, 调整节点指向曾经的父节点。
3. 叔叔是黑色的, 插入节点是左孩子
<img src="http://owsl7963b.bkt.clouddn.com/ConcurrentHashMap6.png" height="400" width="450"/>
调整:
## 红黑树删除后平衡balanceInsertion

