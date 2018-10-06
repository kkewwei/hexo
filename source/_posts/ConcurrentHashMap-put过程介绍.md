---
title: ConcurrentHashMap Put源码介绍
date: 2017-11-05 11:20:55
tags:
---
在平时项目中, 较多的使用了HashMap容器, 但是它是非线程安全的, 在多线程put的时候, 可能会导致HashMap产生环链而导致死锁。 在并发场景下, 我们就得换成ConcurrentHashMap, 采用分段、红黑树等结构体, 支持多线程同时插入, 又拥有者较高的性能。本文章将围绕put的过程进行详细描述。
首先放张大图, 对ConcurrentHashMap先有大致的了解。
<img src="http://owsl7963b.bkt.clouddn.com/ConcurrentHashMap2.png" height="400" width="450"/>
所有插入的值首先放在table的元素中, 当hash(key)冲突时, 将key-value存放在这个元素的后面, 形成一个链表, 当链表长度达到阈值时, 为减少索引时间, 将链表转变为一个红黑树; 当删除数据时, 红黑树可能会退化为链表; table由于负载高, 也可能会继续扩容。
ConcurrentHashMap系列将分为以下四个方面进行详细描述:
<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/05/ConcurrentHashMap-put%E8%BF%87%E7%A8%8B%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap Put源码介绍</a>
<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/08/ConcurrentHashMap-remove%E8%BF%87%E7%A8%8B%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap Remove源码介绍</a>
<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/14/ConcurrentHashMap%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap扩容源码介绍</a>
<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/06/ConcurrentHashMap%E7%BA%A2%E9%BB%91%E6%A0%91%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap红黑树原理介绍</a>
# 成员变量介绍
ConcurrentHashMap拥有出色的性能, 在真正掌握内部结构时, 先要掌握比较重要的成员:
+ LOAD_FACTOR: 负载因子, 默认75%, 当table使用率达到75%时, 为减少table的hash碰撞, tabel长度将扩容一倍。负载因子计算: 元素总个数%table.lengh
+ TREEIFY_THRESHOLD: 默认8, 当链表长度达到8时, 将结构转变为红黑树
+ UNTREEIFY_THRESHOLD: 默认6, 红黑树转变为链表的阈值
+ MIN_TRANSFER_STRIDE: 默认16, table扩容时, 每个线程最少迁移table的槽位个数。
+ MOVED: 值为-1, 当Node.hash为MOVED时, 代表着table正在扩容
+ TREEBIN, 置为-2, 代表此元素后接红黑树
+ nextTable: table迁移过程临时变量, 在迁移过程中将元素全部迁移到nextTable上
+ sizeCtl: 用来标志table初始化和扩容的,不同的取值代表着不同的含义:
    0: table还没有被初始化
    -1: table正在初始化
    小于-1: 实际值为resizeStamp(n)<<RESIZE_STAMP_SHIFT+2, 表明table正在扩容
    大于0: 初始化完成后, 代表table最大存放元素的个数, 默认为0.75*n
+ transferIndex: table容量从n扩到2n时, 是从索引n->1的元素开始迁移, transferIndex代表当前已经迁移的元素下标
+ ForwardingNode: 一个特殊的Node节点, 其hashcode=MOVED, 代表着此时table正在做扩容操作。扩容期间, 若table某个元素为null, 那么该元素设置为ForwardingNode, 当下个线程向这个元素插入数据时, 检查hashcode=MOVED, 就会帮着扩容。

# putVal插入数据
下面真正开始写入数据:
```
    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 定位到table[]中的i
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //若table为0， 则初始化这个table
            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); //首先初始化
            //根据hash值计算出在table里面的位置
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //如果没有，则向这个位置添加一个节点
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //整个table正在扩容
            else if ((fh = f.hash) == MOVED)
                //帮着一起扩容
                tab = helpTransfer(tab, f);
            //真正插入数据
            else {
                V oldVal = null;
                //那么这个点先禁止别的线程插入数据，若整个table迁移还没有处理到这个元素，此时锁住后，迁移到这里后会被卡主
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                         //普通的数组
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //找到key一样的，则直接替换并退出
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //继续找下一个节点，若数组找到最后都没有找到合适的，那么就进行尾插法加入
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //如果是红合树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            //赋值固定，就不会再去转变了，只要小于TREEIFY_THRESHOLD就行
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        //开始将table中元素为i的数组转变为二叉树
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //链表或者红黑树不应该增加table的负载
        addCount(1L, binCount);
        return null;
    }
```
主要做了如下事情:
+ 根据spread确定key-value的hash值, hash计算过程如下: (h ^ (h >>> 16)) & HASH_BITS, h=key.hashCode(), HASH_BITS=0x7fffffff, 由此可见, 计算出来的hash>0一定成立, 若node.hash<0是, -1(Moved)代表table正在扩容, -2(TREEBIN)代表此元素后接红黑树
+ 检查table是否初始化, 若没有初始化,则开始初始化initTable()。 这里可以看出ConcurrentHashMap使用懒性初始化, 只有在真正插入数据时候才进行扩容。
+ 根据i = (n - 1) & hash))确定需要插入table的位置i:
1. 若table[i]没有元素, 则将key-value存放进去
2. 若table[i].hash为MOVED, 那么说明table正在进行扩容, 则通过helpTransfer()进行扩容(具体参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/14/ConcurrentHashMap%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap扩容源码介绍</a>)
3. 否则开始真正插入数据, 插入数据前, 先将table[i]锁住, 插入数据前, 检查table[i].hash, 若大于0, 说明此元素后接的是链表, 或者是个红黑树。 链表插入采取尾插法, 比较简单; 红黑树的插入详见后续描述。
4. 在链表插入时, 统计当前链表长度, 若长度超过TREEIFY_THRESHOLD(默认值为8), 则需要将链表转变为红黑树结构(具体参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/06/ConcurrentHashMap%E7%BA%A2%E9%BB%91%E6%A0%91%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap红黑树原理介绍</a>)
5. 修改table存放所有元素个数、检查table是否需要扩容等, 详见addCount。 这里感觉代码有些问题, 插入任何一个元素, 无论插在table元素上、还是链表或者红黑树上, 都算对table容量增加了1, 实际上插入链表或者红黑树, 并不会增加table的负载, 这两种情况下, 不应该增加table的负载、而去检查扩容。

## initTable初始化
```
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {//只要没有成功，就一定重新尝试
            if ((sc = sizeCtl) < 0)  //正在初始化，本节点先尝试放弃cpu的使用
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { //这里应该是个原子操作，将sizeCtl设置为-1
                try {
                    if ((tab = table) == null || tab.length == 0) { //
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;//sc 大于零说明容量已经初始化了，否则使用默认容量
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);//  //计算阈值，等效于 n*0.75，就是n-0.25*n
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
初始化table主要做了如下事情:
+ 检查sizeCtl, 若发现<0, 那么说明已经有线程正在初始化, 本线程先放弃cpu使用等待初始化完成
+ 若本线程是第一个初始化table, 那么原子操作, 将sizeCtl设置为-1, 表明有线程正在对table正在初始化。
+  初始化table
+  设置sizeCtl= table.length*0.75, 规定了table最大存放元素的个数
## addCount
addCount主要做两个事情: 并发环境下统计ConcurrentHashMap里属性的个数、检查table是否需要扩容。
```
    private final void addCount(long x, int check) {  //若小于0，则不检查扩容
        CounterCell[] as; long b, s=baseCount+x;
        .......//更新元素个数metrics
        }
        if (check >= 0) { //s是当前table长度
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&//达到容积上限，
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n); //根据目前table长度做一个标识
                if (sc < 0) { //说明table正在扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                } //第一个线程开始扩容，sizeCtl = resizeStamp(n)<<16 + 2
                else if (U.compareAndSwapInt(this, SIZECTL, sc, //sc = rs << 16 + 2
                                             (rs << RESIZE_STAMP_SHIFT) + 2))//rs > 2^15,rs << RESIZE_STAMP_SHIFT) + 2) > 2^31+n,int类型的数就是负数了。
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```
主要介绍检查是否需要扩容, 若ConcurrentHashMap总个数s+x达到了最大元素阈值sizeCtl, 那么开始进行扩容, 这里扩容应该分下场景, 链表和红黑树结构体的插入, 不应该进行扩容的。
+ 若sizeCtl<0, 那么检查是否扩容完成, 若扩容完成了, 退出; 否则将sizeCtl加1, 并进行扩容操作transfer(), 详见<a href="https://kkewwei.github.io/elasticsearch_learning/2017/11/14/ConcurrentHashMap%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B%E4%BB%8B%E7%BB%8D/">ConcurrentHashMap扩容源码介绍</a>
+ 若sizeCtl>0, 那么本线程是第一个开始对table进行扩容的, 将sizeCtl=resizeStamp(n)<<16 + 2, 并进行扩容transfer()

# 总结
ConcurrentHashMap为了加快索引速度和插入并发, 采用由数组+链表+红黑树来存储数据,  数据结构之间能否相互转变。 插入数据时, 会检查table是否初始化, 是否在扩容, 插入的元素位置是否为链表、红黑树, 针对不同的情况采取不同的插入方法。