---
title: 	深入解析HashMap #标题
date: 2020/12/3 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - java
 - HashMap
					
categories:	#分类
 - Java

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





## 前言

很高兴遇见你~

HashMap是一个非常重要的集合，日常使用也非常的频繁，同时也是面试重点。本文并不打算讲解基础的使用api，而是深入HashMap的底层知识，讲解关于HashMap的重点知识。

文章整体分为三个部分：第一部分是概述HashMap的重点知识，第二部分是从源码角度分析HashMap，第三部分讲解与HashMap相关的集合类和一些常见问题。

本文所有源码如若未特殊说明，均为JDK1.8版本。


## 基础解析

#### 概述

HashMap是一个映射表的数据结构，他可以存储键值对数据。我们可以在常数时间复杂度通过键key找到我们的值value，是一个非常常用的Java集合类。HashMap的继承结构如下：

![](https://img-blog.csdnimg.cn/img_convert/58436ea26caf0e20e7f14cb4d2fa8fb7.png)

属于Map集合体系的一部分，同时继承了Serializable接口可以被序列化，继承了Cloneable接口可以被复制。

HashMap的核心在于计算key下标的hash函数、冲突解决方案以及扩容方案。这也是考验一个映射表表现的三大重点。hash函数决定了key分布是否均匀、冲突解决方案决定了在发生大量冲突的情况下是否能保持良好的性能、扩容方案决定了每次扩容是否需要消耗大量的性能。JDK1.8中对HashMap的内部算法进行了全新的升级，让性能得到很大的提升。

HashMap并不是全能的，对于一些特殊的情景下的需求官方拓展了一些其他的类来满足，如线程安全的ConcurrentHashMap、记录插入顺序的LinkHashMap等。

#### 底层结构

jdk1.7及以前采用的是数组+链表的模式，而1.8之后采用的是数组+链表或者数组+红黑树的数据结构：

![](https://i.loli.net/2020/12/03/Lvy6hTMIlbYakuC.png)

红黑树的好处是数据量较大是查询速度比链表快得多，但缺点是在数据量很小时性能优化并不明显甚至倒退，树节点的大小是普通节点的两倍。所以jdk1.8的HashMap做了以下优化：

- 当链表的长度>=8且数组长度>=64时，会把链表转化成红黑树。
- 当链表长度>=8，但数组长度<64时，会优先进行扩容，而不是转化成红黑树。

当数组长度较短时，如16，链表长度达到8已经是占用了最大限度的50%，意味着负载已经快要达到上限，此时如果转化成红黑树，之后的扩容又会再一次把红黑树拆分平均到新的数组中，这样非但没有带来性能的好处，反而会降低性能。所以在数组长度低于64时，优先进行扩容。  

事实上，得益于HashMap的优秀hash算法，链表的长度很少能达到8，在链表较短时，直接采用链表更加快速。而在某些极端的情况下，发生了大量的冲突，那么红黑树则可以发挥的强大作用，抗住这种极端情况。

jdk1.8之后HashMap的数组大小控制为2的整数次幂，每次扩容都会扩展到原来的2倍；而jdk1.7数组的大小为素数，且每次扩容之后的大小都是素数。

#### 装载因子

**转载因子**是一个非常关键的参数。他表示HashMap中的节点数与数组长度之比到达对应阈值后进行数组扩容。例如数组长度是16，装载因子是0.75，那么可容纳的节点数是16*0.75=12。那么当往HashMap插入12个数据之后，就会进行扩容。装载因子越大，数组利用率越高，同时发生哈希冲突的概率也就越高；装载因子越小，数组利用率降低，但发生哈希冲突的概率也降低了。所以**装载因子的大小需要权衡空间与时间之间的关系**。

HashMap中的装载因子的默认大小是0.75，是一个在严密计算和实践证实比较适合的一个值。没有特殊要求的情况下，不建议修改他的值。

#### 线程安全

HashMap并不是线程安全，所以在多线程的情况下不能直接使用HashMap。可以采用`ConcurrentHashMap`类或者`Collections.synchronizeMap()`方法来保证线程安全。如有多线程同步需求，更加建议使用`ConcurrentHashMap`类，他的锁粒度更低，而`Collections.synchronizeMap()`方法默认锁的是整个Map对象，感兴趣的读者可以自行阅读源码。

那使用了`ConcurrentHashMap`类是不是绝对线程安全？先观察下面的代码：

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("abc","123");

Thread1:
if (map.containsKey("abc")){
    String s = map.get("abc");
}

Thread2:
map.remove("abc");
```

当Thread1调用containsKey之后释放锁，Thread2获得锁并把“abc”移除再释放锁，这个时候Thread1读取到的s就是一个null了，也就出现了问题了。所以`ConcurrentHashMap`类或者`Collections.synchronizeMap()`方法都只能在一定的限度上保证线程安全，而无法保证绝对线程安全。

##### fast-fail

当使用HashMap的迭代器遍历HashMap时，如果此时HashMap发生了结构性改变，如插入新数据、移除数据、扩容等，那么Iteractor会抛出fast-fail异常，防止出现并发异常，在一定限度上保证了线程安全。

fast-fail异常只能当做遍历时的一种安全保证，而不能当做多线程并发访问HashMap的手段。若有并发需求，还是需要使用上述的两种方法。

## 源码解析

#### 关键变量的理解

HashMap源码中有很多的内部变量，这些变量会在下面源码分析中经常出现，首先需要理解这些变量的意义。

```java
// 存放数据的数组
transient Node<K,V>[] table;
// 存储的键值对数目
transient int size;
// HashMap结构修改的次数，主要用于判断fast-fail
transient int modCount;
// 最大限度存储键值对的数目(threshodl=table.length*loadFactor)，也称为阈值
int threshold;
// 装载因子，表示可最大容纳数据数量的比例
final float loadFactor;
// 静态内部类，HashMap存储的节点类型；可存储键值对，本身是个链表结构。
static class Node<K,V> implements Map.Entry<K,V> {...}
```



#### 获取索引

HashMap获取索引主要分为三步：

- 获取key的hashcode
- 将hashcode的高16位与低16位进行异或运算，保留原高16位
- 对运算结果进行取模

前两步的源码可以直接看到`hash()`方法：

```java
static final int hash(Object key) {
    int h;
    // 获取到key的hashcode，在高低位异或运算
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

取模运算在jdk1.7之前的采用的常规的求余运算，如`13%11==2`。为了使键的分布更加均匀，jdk1.7控制数组的长度大小为素数。jdk1.8的HashMap数组长度为2的整数次幂，这样的好处是直接对key的hashcode进行求余运算和让hashcode与数组长度-1进行位与运算是相同的效果。可以看下图帮助理解：

![](https://i.loli.net/2020/12/03/S16Rc4fN8ynCPgL.png)

但位与运算的效率却比求余高得多，提升了性能。同时为了使高位也能参与hash散列，把高16位与低16进行异或，使分布更加均匀。

取模运算看到`putVal()`方法，该方法在`put()`方法中被调用：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    ...
	// 与数组长度-1进行位与运算，得到下标
    if ((p = tab[i = (n - 1) & hash]) == null)
        ...
}
```

详细的计算过程可以参考下图：

![](https://i.loli.net/2020/12/03/xN9VvrpT3PcqlC4.png)



#### 添加数值

调用`put()`方法添加键值对，最终会调用`putVal()`来真正实现添加逻辑。代码解析如下：

```java
public V put(K key, V value) {
    // 获取hash值，再调用putVal方法插入数据
    return putVal(hash(key), key, value, false, true);
}

// onlyIfAbsent表示是否覆盖旧值，true表示不覆盖，false表示覆盖，默认为false
// evict和LinkHashMap的回调方法有关，不在本文讨论范围
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    
    // tab是HashMap内部数组，n是数组的长度，i是要插入的下标，p是该下标对应的节点
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 判断数组是否是null或者是否是空，若是，则调用resize()方法进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 使用位与运算代替取模得到下标
    // 判断当前下标是否是null，若是则创建节点直接插入，若不是，进入下面else逻辑
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        
        // e表示和当前key相同的节点，若不存在该节点则为null
        // k是当前数组下标节点的key
        Node<K,V> e; K k;
        
        // 判断当前节点与要插入的key是否相同，是则表示找到了已经存在的key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 判断该节点是否是树节点，如果是调用红黑树的方法进行插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 最后一种情况是直接链表插入
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 长度大于等于8时转化为红黑树
                    // 注意，treeifyBin方法中会进行数组长度判断，
                    // 若小于64，则优先进行数组扩容而不是转化为树
                    if (binCount >= TREEIFY_THRESHOLD - 1) 
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到相同的直接跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 如果找到相同的key节点，则判断onlyIfAbsent和旧值是否为null
        // 执行更新或者不操作，最后返回旧值
        if (e != null) { 
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // 如果不是更新旧值，说明HashMap中键值对数量发生变化
    // modCount数值+1表示结构改变
    ++modCount;
    // 判断长度是否达到最大限度，如果是则进行扩容
    if (++size > threshold)
        resize();
    // 最后返回null（afterNodeInsertion是LinkHashMap的回调）
    afterNodeInsertion(evict);
    return null;
}
```

代码中关于每个步骤有了详细的解释，这里来总结一下：

1. 总体上分为两种情况：找到相同的key和找不到相同的key。找了需要判断是否更新并返回旧value，没找到需要插入新的Node、更新节点数并判断是否需要扩容。
2. 查找分为三种情况：数组、链表、红黑树。数组下标i位置不为空且不等于key，那么就需要判断是否树节点还是链表节点并进行查找。
3. 链表到达一定长度后需要扩展为红黑树，当且仅当链表长度>=8且数组长度>=64。

最后画一张图总体再加深一下整个流程的印象：

![](https://i.loli.net/2020/12/03/dqGmZPIB8ux5hW6.png)



#### 扩容机制

扩容机制的源码一共分为两个部分：

- 确定新的数组大小
- 把旧数组中的元素迁移到新的数组中

扩容机制中的一个重点是：**使用HashMap数组长度是2的整数次幂的特点，以一种更高效率的方式完成数据迁移**。

JDK1.7之前的数据迁移比较简单，就是遍历所有的节点，把所有的节点依次通过hash函数计算下标，再插入到新数组的链表中。这样会有两个缺点：**1、每个节点都需要进行一次求余计算**；**2、插入到新的数组时候采用的是头插入法，因为尾插入法需要另外维护一个数组，而头插入法会打乱原有的顺序**。

jdk1.8之后进行了优化，原因在于他控制数组的长度始终是2的整数次幂，每次扩展数组都是原来的2倍。带来的好处是key在新的数组的hash结果只有两种：在原来的位置，或者在原来位置+原数组长度。具体为什么我们可以看下图：

![](https://i.loli.net/2020/12/03/pljrTk5wWXQx8ed.png)

从图中我们可以看到，在新数组中的hash结果，仅仅取决于高一位的数值。如果高一位是0，那么计算结果就是在原位置，而如果是1，则加上原数组的长度即可。这样我们**只需要判断一个节点的高一位是1 or 0就可以得到他在新数组的位置，而不需要重复hash计算**。HashMap源码中并没有一个个地把节点搬到新数组，而是**把每个链表拆分成两个链表，再分别插入到新的数组中，这样就可以在不需要开辟一个数组维护尾节点的条件下，完成尾插法，保留原来的节点顺序**。

具体源码分析如下：

```java
final Node<K,V>[] resize() {
    // 分别是原数组、原数组大小、原阈值；和新数组大小、新最大限度数据量
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // 如果原数组长度大于0
    if (oldCap > 0) {
        // 如果已经超过了设置的最大长度(1<<30,也就是最大整型正数)
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 直接把阈值设置为最大正数
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 设置为原来的两倍
            newThr = oldThr << 1; 
    }
    
    // 原数组长度为0，但最大限度不是0，把长度设置为阈值
    // 对应的情况就是新建HashMap的时候指定了数组长度
    else if (oldThr > 0) 
        newCap = oldThr;
    // 第一次初始化，默认16和0.75
    // 对应使用默认构造器新建HashMap对象
    else {               
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果原数组长度小于16或者翻倍之后超过了最大限制长度，则重新计算阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    @SuppressWarnings({"rawtypes","unchecked"})
    // 建立新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 循环遍历原数组，并给每个节点计算新的位置
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果他没有后继节点，那么直接使用新的数组长度取模得到新下标
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是红黑树，调用红黑树的拆解方法
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 新的位置只有两种可能：原位置，原位置+老数组长度
                // 把原链表拆成两个链表，然后再分别插入到新数组的两个位置上
                // 不用多次调用put方法
                else { 
                    // 分别是原位置不变的链表和原位置+原数组长度位置的链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 遍历老链表，判断新增判定位是1or0进行分类
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 最后赋值给新的数组
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 返回新数组
    return newTab;
}
```



## 相关类

#### HashTable

Hashtable（注意t是小写）和HashMap的功能基本一样，Hashtable是老一代的java集合类，他的一个重要的特点就是：线程安全。他的所有需要访问数据的方法都被synchronize关键字修饰。但这个类已经被官方锁抛弃不建议使用，原因有下：

- Hashtable继承自Dictionary类而不是AbstractMap，类图如下（jdk1.8）

  ![](https://i.loli.net/2020/12/03/eyHVOdfliQTmW83.png)

  Hashtable诞生的时间是比Map早，属于第一批的java集合类。而为了兼容新的集合在jdk1.2之后也继承了Map接口，Map接口包含了Dictionary中的方法。而Dictionary接口在目前已经完全被Map取代了，所以更加建议使用继承自AbstractMap的HashMap。

- Hashtable的设计已经落后了。Hashtable的底层结构采用的是数据+链表，初始数组大小是11，后续扩容为2n+1（n是原数组大小），保证数组长度是素数，通过直接求余来得到数组下标。 在jdk1.8之后HashMap采用高低位异或+位与代替取模的优秀设计提高哈希散列性能，使用红黑树来提高在极端冲突情况下的抗压能力。而这些在Hashtable中都没有用到，性能表现不如HashMap。

- Hashtable被设计为线程安全，但是锁粒度太大性能较低，相比ConcurrentHashMap后者的锁粒度更低，性能更佳。且在大部分情况下，并不涉及到并发操作，所以频繁加锁解锁是没有必要的。所以正常情况下使用HashMap更佳，而如果有并发需求，使用ConcurrentHashMap是更好的选择。

#### ConcurrentHashMap

HashMap无法保证在多线程并发访问下的安全，而ConcurrentHashMap就是为了解决这个问题。ConcurrentHashMap并不是和Hashtable一样采用直接对整个数组进行上锁，而是对数组上的一个节点上锁，这样如果并发访问的不是同个节点，那么就无需等待释放锁。如下图：

![](https://i.loli.net/2020/12/03/tNxWwMGIv5R8moJ.png)

不同线程之间的访问不同的节点不互相干扰，提高了并发访问的性能。当然，如果访问同个节点，还是需要等待上一个线程释放锁。

这是jdk1.8优化之后的设计结构，jdk1.7之前是分为多个小数组，锁的粒度比Hashtable稍小了一些。如下：

![](https://i.loli.net/2020/12/03/2mVLQEZNFgbdx3I.png)

锁的是Segment，每个Segment对应一个数组。而jdk1.8之后锁的粒度进一步降低，性能也进一步提高了。

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
这几个类弥补了HashMap在一些特殊的情况下的能力，如线程安全、记住插入顺序、给key排序。一般情况下如若没有特殊要求，使用HashMap即可。

## 常见问题

这部分总结一些常见的问题，问题的答案在文章中都有涉及到，也当为全文回顾。

#### 为什么jdk1.7以前控制数组的长度为素数，而jdk1.8之后却采用的是2的整数次幂？

答：素数长度可以有效减少哈希冲突；JDK1.8之后采用2的整数次幂是为了提高求余和扩容的效率，同时结合高低位异或的方法使得哈希散列更加均匀。

为何素数可以减少哈希冲突？若能保证key的hashcode在每个数字之间都是均匀分布，那么无论是素数还是合数都是相同的效果。例如hashcode在1~20均匀分布，那么无论长度是合数4，还是素数5，分布都是均匀的。而如果hashcode之间的间隔都是2，如1,3,5...,那么长度为4的数组，位置2和位置4两个下标无法放入数据，而长度为5的数组则没有这个问题。**长度为合数的数组会使间隔为其因子的hashcode聚集出现，从而使得散列效果降低**。详细的内容可以参考这篇博客：[算法分析：哈希表的大小为何是素数](https://blog.csdn.net/zhishengqianjun/article/details/79087525),这篇博客采用数据分析证实为什么素数可以更好地实现散列。

#### 为什么数组长度是2的整数次幂？HashMap如何保证数组长度一定是2的整数次幂呢？

答：方便取模和扩容，提高性能；默认数组为16长度，每次扩容都是为原来的2倍，若初始化指定非2的整数次幂，会取刚好大于该指定大小的最小2的整数次幂。

第一个问题在前面讲过，这里不再赘述，第二个问题要结合源码来分析，当我们初始化指定一个非2的整数次幂长度时，HashMap会调用`tableSizeFor()`方法来使得长度为2的整数次幂：

```java
public HashMap(int initialCapacity, float loadFactor) {
    ...
    this.loadFactor = loadFactor;
    // 这里调用了tableSizeFor方法
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

`tableSizeFor()`方法使得最高位1后续的所有位都变为1，最后再+1则得到刚好大于initialCapacity的最小2的整数次幂数。可参考下图理解：

![](https://i.loli.net/2020/12/03/6ARdnmVx2BlgDtr.png)

这里使用了8位进行模拟，32位也是同理。所以当我们在构造函数中指定HashMap的长度并不是2的整数次幂时，他会自动变换成2de整数次幂。如输入5，会变成8，输入11会变成16。

#### 为什么装载因子是0.75？可不可以换成其他数值？

 答：0.75是在严谨的数学计算以及权衡时间和空间得出的性价比最高的结果，一般情况下并不建议改动。

一般情况下，我们总是想要装载因子越高越好，可以提高数组的利用率。但装载因子太高带来的问题是发生剧烈的哈希冲突，性能急剧下降，冲突数量的增加在0.75这个拐点之后呈指数爆炸增长，而0.75则刚好是在保证时间复杂度在低水平的情况下尽可能大地利用数组资源。在HashMap中有这样一段注释：

```java
/*...
* 0:    0.60653066
* 1:    0.30326533
* 2:    0.07581633
* 3:    0.01263606
* 4:    0.00157952
* 5:    0.00015795
* 6:    0.00001316
* 7:    0.00000094
* 8:    0.00000006
* more: less than 1 in ten million
*...
*/
```

他表示的是装载因子是0.75的情况下，在理论概率中一个数组节点中链表长度的可能性，可以看到长度为8的可能性已经是非常低了。而现实的数据可能并不符合均匀分布，所以红黑树的存在主要是为了解决极端情况下的数据压力。

那转载因子可不可以换成其他数字？没有特殊的要求不要更换。调低了时间复杂度降低不明显但是却会带来空间复杂度的提升，调高了空间复杂度提升不明显带来的却是时间复杂度的剧烈提升。更加具体的分析可以参考这篇文章：[面试官：为什么 HashMap 的加载因子是0.75？](https://zhuanlan.zhihu.com/p/149687607),作者通过数学分析来说明为什么0.75是最好的。

#### 为什么插入HashMap的数据需要实现hashcode和equals方法？对这两个方法有什么要求？

答：通过hashcode来确定插入下标，通过equals比较来寻找数据；两个相等的key的hashcode必须相等，但拥有相同的hashcode的对象不一定相等。

这里需要区分好他们之间的区别：hashcode就像一个人的名，相同的人名字肯定相等，但是相同的名字不一定是同个人；equals比较内容是否相同，一般由对象覆盖重写；“==”判断的是引用地址是否相同。

HashMap中需要使用hashcode来获取key的下标，如果两个相同的对象的hashcode不同，那么会造成HashMap中存在相同的key；所以equals比较相同的key他们的hashcode一定要相同。HashMap比较两个元素是否相同采用了三种比较方法结合：`p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))` 。关于更加深入的讲解可以参考这篇文章：[Java提高篇——equals()与hashCode()方法详解](https://www.cnblogs.com/qian123/p/5703507.html)，作者非常详细地剖析了这些方法之间的区别。

## 最后

关于HashMap的内容很难在一篇文章讲完，其他发散的知识点读者有兴趣可以深入去了解。HashMap是一种映射表，他的重点在于散列表这种底层的数据结构，装载因子的设计使用了概率论、高等数学等基础学科，掌握好数据结构数学等基本知识非常重要。同时，HashMap的设计权衡了众多因素，提升性能，在jdk1.7提升到1.8可以非常明显看到这点，这些是非常值得我们学习的。

#### 推荐文章&书籍

- [Java 8系列之重新认识HashMap](https://tech.meituan.com/2016/06/24/java-hashmap.html)

  美团博客网站关于JDK8优化HashMap的分析，通俗易懂且深刻，非常值得阅读的文章。

- 《数据结构与算法分析》

  这本书是偏向于散列表数据结构分析，知识较为深入，但并不是很容易理解

- [图解LinkedHashMap原理](https://www.jianshu.com/p/8f4f58b4b8ab)

  这篇文章比较详细地讲解了关于LinkedHashMap的相关知识，且通俗易懂

- [漫画：什么是HashMap？](https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653191907&idx=1&sn=876860c5a9a6710ead5dd8de37403ffc&chksm=8c990c39bbee852f71c9dfc587fd70d10b0eab1cca17123c0a68bf1e16d46d71717712b91509&scene=21#wechat_redirect)

  小灰的漫画算法，非常通俗易懂，适合刚学HashMap入手，也适合老手复习

> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)