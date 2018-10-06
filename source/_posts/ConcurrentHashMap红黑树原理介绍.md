---
title: ConcurrentHashMap红黑树原理介绍
date: 2017-11-06 17:48:34
tags:
---
# 红黑树介绍
一棵二叉查找树如果满足下面的红黑性质, 则为一棵红黑树:
1）每个节点或者是黑的，或者是红的。
2）根节点是黑的。
3）每个叶子节点（NIL）是黑的。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
4）如果一个节点是红的，则它的两个儿子都是黑的。
5）对每个节点, 从该节点到其子孙节点的所有路径上包含相同数目的黑节点。
## 链表转化为红黑树
```
        TreeBin(TreeNode<K,V> b) {
            super(TREEBIN, null, null, null);
            this.first = b;
            //根节点
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
## 红黑树插入putTreeVal
## 红黑树插入后平衡balanceInsertion
## 红黑树删除
## 红黑树删除后平衡balanceInsertion

