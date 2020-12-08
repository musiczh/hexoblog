---
title: 	Java		#标题
date: 2020/8/1 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - 
					
categories:	#分类
 - 

updated: 					#更新日期
author: 一只修仙的猿 #作者
toc: true	#是否显示toc
mathjax:  #数学公式

keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字

katex:
aplayer:
highlight_shrink: true       #代码框是否打开
---









```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许插入空值或空键
    if (key == null || value == null) throw new NullPointerException();
    // 高低位异或扰动
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组为空则进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 计算下标并判断是否为空
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 重点：采用CAS进行插入
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                break;
        }
        // 数组正在扩容，帮忙转发节点
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 同步加锁
            synchronized (f) {
                // 重复检查一下刚刚获取的对象有没有发生变化
                if (tabAt(tab, i) == f) {
                    // 链表处理情况
                    if (fh >= 0) {
                        binCount = 1;
                        // 循环链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 找到相同的则记录旧值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // 判断是否需要更新数值
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 若未找到则插在链表尾
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树处理情况
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            // 判断是否需要转化为红黑树，和返回旧数值
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 总数+1；这是一个非常硬核的设计
    addCount(1L, binCount);
    return null;
}
```







```java

private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 这里的循环是采用自旋的方式而不是上锁来初始化
    while ((tab = table) == null || tab.length == 0) {
        // size是一个非常关键的变量；
        // 默认为0，-1表示正在初始化，<-1表示有多少个线程正在帮助扩容，>0表示阈值
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // 让出cpu执行执行，自旋
        // 通过CAS设置sc为-1，其他线程则无法进入初始化，进行等待
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 重复检查是否为空
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 设置sc为阈值，n>>>2表示1/4*n
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    // 最后返回tab数组
    return tab;
}
```



```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果数组不为空 或者 数组为空且直接更新basecount失败
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        
        CounterCell a; long v; int m;
        // 表示没发生竞争
        boolean uncontended = true;
        // 这里有以下情况会进入fullAddCount方法：
        // 1. 数组为null且直接修改basecount失败
        // 2. hash后的数组下标CounterCell对象为null
        // 3. CAS修改CounterCell对象失败
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 该方法保证完成更新，重点方法！！
            fullAddCount(x, uncontended);
            return;
            
        }
        
        // 如果长度<=1不需要扩容（说实话我觉得这里有点奇怪）
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        // 扩容相关
    }
}
```



```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 如果当前线程随机数为0，强制初始化一个线程随机数
    // 这个随机数的作用就类似于hashcode，不过他不需要查找所以不需要唯一
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    
    // 直译为碰撞，如果他为true，则表示需要进行扩容
    boolean collide = false;      
    
    // 下面分为三种大的情况：
    // 1. 数组不为null，对应的情况为CAS更新CounterCell失败或者countCell为null
    // 2. 数组为null，表示之前CAS更新baseCount失败
    // 3. 第二步获取不到锁，尝试CAS更新baseCount
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        
        // 第一种情况：数组不为null
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 对应下标的CounterCell为null的情况
            if ((a = as[(n - 1) & h]) == null) {
                // 判断当前锁是否被占用
                if (cellsBusy == 0) {            
                    CounterCell r = new CounterCell(x); 
                    // 尝试获取锁来添加一个新的CounterCell对象
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               
                            CounterCell[] rs; int m, j;
                            // recheck一次是否为null
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            // 释放锁
                            cellsBusy = 0;
                        }
                        // 创建成功也就是+1成功，直接返回
                        if (created)
                            break;
                        // 拿到锁后发现已经有别的线程插入数据了
                        continue;          
                    }
                }
                // 想创建一个对象，但是拿不到锁
                collide = false;
            }
            // 之前直接CAS改变CounterCell失败，重新hash，再循环一次
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 尝试对CounterCell进行CAS
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // 如果发生过扩容或者长度已经达到虚拟机最大可以核心数，直接认为无碰撞
            // 因为已经无法再扩容了
            // 所以并发线程数的理论最高值就是NCPU
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            // 如果上面都是false，那么认为需要进行扩容
            else if (!collide)
                collide = true;
            // 这一步获取锁，并进行扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        // 扩大数组为原来的2倍
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    // 释放锁
                    cellsBusy = 0;
                }
                collide = false;
                // 继续循环
                continue;                   
            }
            // 这一步是重新hash，找下一个CounterCell对象
            h = ThreadLocalRandom.advanceProbe(h);
        }
        
        // 第二种情况：数组为null，尝试获取锁
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {
                // recheck判断数组是否为null
                if (counterCells == as) {
                    // 初始化数组
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                // 释放锁
                cellsBusy = 0;
            }
            // 如果初始化完成，直接跳出循环，
            // 因为初始化过程中也包括了新建CounterCell对象
            if (init)
                break;
        }
        
        // 第三种情况：数组为null，但是拿不到锁，意味着别的线程在新建数组，尝试直接更新baseCount
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            // 更新成功直接返回
            break;                         
    }
}
```









```java
private final void addCount(long x, int check) {
    ... // 总数+1逻辑
        
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
       	// 当长度达到阈值且长度并未达到最大值时进行下一步扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 这个数配合后续的sizeCtr计算
            // 他的格式是第16位肯定为1,低15位表示n前面连续的0个数
            int rs = resizeStamp(n);
            // 小于0表示正在扩容或者正在初始化,下一步判断是否需要帮忙迁移
            if (sc < 0) {
                // 如果正在迁移右移16位后一定等于rs
                // ( sc == rs + 1 ||sc == rs + MAX_RESIZERS)这两个条件我认为不可能为true
                // 有兴趣可以点击下方网站查看
                // https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427
                // nextTable==null说明下个数组还未创建
                // transferIndex<=0说明迁移已经够完成了
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 帮忙迁移,sc+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 抢占锁进行扩容
            // rs第16位是1，左移16位之后变成一个负数
            // 这个数字表示当前当前正在迁移的线程数
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            // 更新节点总数，继续循环
            s = sumCount();
        }
    }
}
```





```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // stride表示每次前进的步伐，最低是16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 如果新的数组还未创建，则创建新数组
    // 只有一个线程能进行创建数组
    if (nextTab == null) {            
        try {
            @SuppressWarnings("unchecked")
            // 扩展为原数组的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      
            // 扩容失败出现OOM，直接把阈值改成最大值
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 更改concurrentHashMap的内部变量nextTable
        nextTable = nextTab;
        // 迁移的起始值为数组长度
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 标志节点，每个迁移完成的数组下标都会设置为这个节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // advance表示当前线程是否要前进
    // finish表示迁移是否结束
    // 官方的注释表示在赋值为true之前，必须再重新扫描一次确保迁移完成，后面会讲到
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    // i表示当前线程迁移数据的下标，bound表示下限，从后往前迁移
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        
        // 这个循环主要是判断是否需要前进，如果需要则CAS更改下个bound和i
        while (advance) {
            int nextIndex, nextBound;
            // 如果还未到达下限或者已经结束了，advance=false
            if (--i >= bound || finishing)
                advance = false;
            // 每一轮循环更新transferIndex的下标
            // 如果下一个下标是0，表示已经无需继续前进
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 利用CAS更高bound和i继续前进迁移数据
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        
        // i已经达到边界，说明当前线程的任务已经完成，无需继续前进
        // 如果是第一个线程需要更新table引用
        // 协助线程需要将sizeCtl减一再退出
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 如果已经更新完成，则更新table引用
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // 同时更新sizeCtl为阈值
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 完成自己的迁移任务，将sizeCtl减一
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 检验sc与初始值是否相等，如果是说明其他线程都已经迁移完成，可以直接返回
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // finish设置为true表示已经完成
                // 这里把i设置为n，重新把整个数组扫描一次
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果当前节点为null，表示迁移完成，设置为标志节点
        else if ((f = tabAt(tab, i)) == null)
            // 这里的设置有可能会失败，所以不能直接设置advance为true，需要再循环
            advance = casTabAt(tab, i, null, fwd);
        // 当前节点是ForwardingNode，表示迁移完成，继续前进
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 给头节点加锁，进行迁移
            synchronized (f) {
                // 上锁之后再判断一次看该节点是否还是原来那个节点
                // 如果不是则重新循环
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // hash值大于等于0表示该节点是普通链表节点
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 这一步循环找到尾部都是同个迁移位置整体迁移
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 判断尾部整体迁移到哪个位置
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 这个node节点是改造过的
                            // 相当于使用头插法插入到链表中
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 链表构造完成，把链表赋值给数组
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 设置标志对象，表示迁移完成
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 树节点的处理，和链表思路相同，不过他没有lastRun，直接分为两个链表，采用尾插法
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```





```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 判断当前节点为ForwardingNode，且已经创建新的数组
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // sizeCtl<0表示还在扩容
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 校验是否已经扩容完成或者已经推进到0，则不需要帮忙扩容
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 让sc+1并帮忙扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        // 返回扩容之后的数组
        return nextTab;
    }
    // 若数组尚未初始化或节点非ForwardingNode,返回原数组
    return table;
}
```

#### LinkHashMap

HashMap是无法记住插入顺序的，在一些需要记住插入顺序的场景下，HashMap就显得无能为力，所以LinkHashMap就应运而生。LinkHashMap内部新建一个内部节点类LinkedHashMapEntry继承自HashMap的Node，增加了前后指针。每个插入的节点，都会使用前后指针联系起来，形成一个链表，这样就可以记住插入的顺序，如下图：

![](https://i.loli.net/2020/12/03/fzuy3hJ7B2pd4qN.png)

图中的红色线表示双向链表的引用。遍历时从head出发可以按照插入顺序遍历所有节点。

#### TreeMap

HashMap的key排列是无序的，hash函数把每个key都随机散列到数组中，而如果想要保持key有序，则可以使用TreeMap。TreeMap的继承结构如下：

![](https://i.loli.net/2020/12/03/Y5dN8H7m63WJznh.png)

他继承自Map体系，实现了Map的接口，同时还实现了NavigationMap接口，该接口拓展了非常多的方便查找key的接口，如最大的key、最小的key等。

TreeMap虽然拥有映射表的功能，但是他底层并不是一个映射表，而是一个红黑树。他可以将key进行排序，但同时也失去了HashMap在常数时间复杂度下找到数据的有点。所以若不是有排序的需求，常规情况下还是使用HashMap。

#### 小结

#### 

这几个类弥补了HashMap在一些特殊的情况下的能力，如线程安全、记住插入顺序、给key排序。一般情况下如若没有特殊要求，使用HashMap即可。

> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

