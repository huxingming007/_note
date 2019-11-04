## HashMap

### 概念

> size：存放KV的数量（包括数组和链表）
>
> capacity：容量桶的个数（数组的length），默认为16，扩容一般是2倍，总之是2的幂次方
>
> loadFactor：负载因子，默认0.75
>
> threshold：size大于threshold会resize操作，threshold=capacity*loadFactor

### put

> 1、初始化一个Node<K,V>[] table
>
> 2、实例化一个Node对象，要把这个node对象放到table中，该放到table数组的哪个位置呢？length-1 & key.hashcode决定
>
> 3、假如步骤的算法，算出相同的值（hash碰撞），怎么处理？原来他是以链表的形式来进行存储，如果链表的长度大于8链表就会转变成红黑树（jdk1.8增加的）
>
> 4、假如放入同样的key怎么处理？迭代列表或者红黑树，根据onlyIfAbsent决定是否进行覆盖
>
> ```java
> if (!onlyIfAbsent || oldValue == null)
>     e.value = value;
> ```

### get

> 1、校验table是否为null，长度是否大于0，table是否存在这个key，反之返回null
>
> 2、table中first的key 进行equal，相等的话，直接进行返回
>
> 3、first是否有next，没有的话，直接null返回，有next，如果是树节点，根据树的方式进行返回，如果不是树节点，迭代这个链表

### 负载因子

默认为0.75，当一个map填满了75%的bucket时候，会进行扩容，重新调整map的大小，并且调整node在map中存放的位置，叫做rehashing。

我们可以在创建 HashMap 时根据实际需要适当地调整 load factor 的值，如果程序比较关心空间开销、内存比较紧张，可以适当地增加负载因子；如果程序比较关心时间开销，内存比较宽裕则可以适当的减少负载因子。通常情况下，程序员无需改变负载因子的值。 

### hash & length-1

> 1、位运算的效率快于十进制运算，扩展也是按位扩容
>
> 2、length（2的整数次幂）的特殊性导致了length-1的特殊性（二进制全为1）
>
> 比如：length=16 length-1=15 15的二进制是1111与任何数进行&操作，空间不浪费。假如length=15 length-1=14的二进制是1110与任何数进行&操作，最后一位肯定是0，那么0001、1011、1101、0111等等这几个位置都不用存放元素，**空间浪费**，更糟的是这种情况，数组table可以使用的位置比数组长度小了很多，意味着进一步增加了**hash碰撞**的几率，从而影响查询性能。

### 为什么hashmap的长度是2的幂次方

一个数的hash值是从-21亿到正的21亿，大概有40亿的空间，关键在于一个40亿长度的数组，内存是放不下的，所以这散列值不能直接拿来用。前期的算法是对hash值跟长度进行取模，后期进行了优化，采用了二进制位操作&，相对于%能够提高运算效率，前提是长度必须是2的幂次方，这样计算出来的结果是一样的。

### 线程不安全

1、put数据的时候：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
  // 假如两个线程同时put，并且(n - 1) & hash的值是一样的，那么这两个key会添加到数组的同一个位置，
  // 这样最终只有一个线程是成功，另外一个key就会被覆盖从而丢失
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
      ......
```

2、扩容的时候：一个线程正在扩容，一个线程正在put数据，put的数据很有可能会丢失，扩容会拿到oldtable进行迭代元素，重新计算，放入一个newtable中。

3、《Java并发编程的艺术》一书中是这样说的：HashMap在并发执行put操作时会引起死循环，导致CPU利用率接近100%。因为多线程会导致HashMap的Node链表形成环形数据结构，一旦形成环形数据结构，Node的next节点永远不为空，就会在获取Node时产生死循环。

## ConcurrentHashMap

### put

> 1、先检查table是否为null，为null的情况下，需要初始化，sizeCtl=-1；
>
> 2、位置（计算length-1&hash）上为空，那么直接放入，不需要加锁操作，利用casTabAt操作；
>
> ```java
> else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
>     if (casTabAt(tab, i, null,
>                  new Node<K,V>(hash, key, value, null)))
>         break;                   // no lock when adding to empty bin
> }
> ```
>
> 3、位置存在节点，如果这个节点的hash值为-1（forward节点），帮助扩容。ps：forward节点的作用是来占位，表示原数组中位置i处的节点完成迁移以后，就会在i位置设置一个fwd来告诉其他线程这个位置已经处理过了。
>
> ```java
> // forwardingNode节点，标记作用，表示其他线程正在扩容，并且此节点已经扩容完毕
> else if ((fh = f.hash) == MOVED)
>     tab = helpTransfer(tab, f);
> ```
>
> 4、位置上存在节点，不是forward节点，进行加锁，判断节点类型：
>
> 4.1、链表结构：依次向后遍历，如果key值一样，只需要更新value即可
>
> 4.2、红黑树：直接调用树节点的插入方法进行插入新的值
>
> ```java
> // 对node节点进行加锁
> synchronized (f) {
>     if (tabAt(tab, i) == f) {
>         if (fh >= 0) {
>             binCount = 1;
>             for (Node<K,V> e = f;; ++binCount) {
>                 K ek;
>                 if (e.hash == hash &&
> ```

### sizeCtl

- -1 代表正在初始化
- -N 代表有N-1个线程正在扩容
- 0 代表node数组还没有被初始化
- 正数 代表下一次扩容的大小

### addCount

```java
// put 方法的最后操作，将当前map的元素数量加1，有可能会触发扩容操作，binCount链表长度
....
addCount(1L, binCount);
return null;
```

```java
：private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
   // 如果counterCells为空，通过cas操作BASECOUNT变量（来记录元素个数），说明竞争并不是很激烈
   // 如果counterCells不为空，或者cas失败，说明竞争激烈，不能通过BASECOUNT，而是通过CounterCell来记录元素个数
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
          // 重点：源码分析，本次略。主要是用初始化countercell，来记录元素个数，里面包含扩容，初始化等操作
          fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)// 链表长度小于等于1，不需要考虑扩容
            return;
        s = sumCount();
    }
    if (check >= 0) {
       .....
    }
}
```

```java
// CHM是采用CounterCell数组来记录元素个数的，为什么使用这种方式呢，而不是直接定义一个size的成员变量即可？
// CHM是并发集合，定义一个size成员变量，势必会需要通过加锁或者自旋来实现，如果竞争激烈，冲突大影响性能，所以在CHM采用了分片的方式来记录大小
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

### 扩容阶段

键值对总数basecount>=阈值sizectl时，进行rehash

1、如果当前正在处于扩容阶段，则当前线程会加入并且协助扩容

2、如果当前没有在扩容，则直接出发扩容操作

```java
// 
private final void addCount(long x, int check) {
  ........
if (check >= 0) {
    Node<K,V>[] tab, nt; int n, sc;
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
           (n = tab.length) < MAXIMUM_CAPACITY) {
        int rs = resizeStamp(n);
        if (sc < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                transfer(tab, nt);
        }
        else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                     (rs << RESIZE_STAMP_SHIFT) + 2))
            transfer(tab, null);
        s = sumCount();
    }
}
```

源码太难，看不下去了。。。。。未完待续

#### 扩容过程图解

CHM支持并发扩容，实现方式是，把node数组进行**拆分**，让**每个线程处理自己的区域**，假设table数组总长度是64，默认情况下，那么每个线程可以分到16个bucket。然后每个线程处理的范围，按照倒序来做迁移，通过for自循环处理每个槽位的链表元素。

- 假设总长度是 64 ，每个线程可以分到 16 个桶，各自处理，不会互相影响。

![img](http://ww1.sinaimg.cn/large/006tNc79ly1g61mnzjs3wj313s0c2dgq.jpg)

#### 数据迁移阶段的实现分析

采用了高低位的设计：不需要在每次扩容的时候重新计算hash，极大提升了效率。

![img](http://ww3.sinaimg.cn/large/006tNc79ly1g61pqjnhmlj31760u077n.jpg)

扩容前的状态。

当对 4 号桶或者 10 号桶进行转移的时候，会将链表拆成两份，规则是根据节点的 hash 值取于 length，如果结果是 0，放在低位，否则放在高位。

因此，10 号桶的数据，黑色节点会放在新表的 10 号位置，白色节点会放在新桶的 26 号位置。

下图是循环处理桶中数据的逻辑：

![img](http://ww3.sinaimg.cn/large/006tNc79ly1g61mo1rssaj313c0hojuh.jpg)

处理完之后，新桶的数据是这样的：

![img](http://ww2.sinaimg.cn/large/006tNc79ly1g61mo2p9kkj31h20nqjth.jpg)

#### 高低位的理解

**具体思想：**把一个链表进行高低位的计算，低位的继续留在原先的位置，高位的加扩容的大小（比方说原先桶大小是16，位置是14，扩容成桶大小是32，增加了16，故高位应放在14+16=30的位置），减少了rehash计算，性能提升。

**这么区分高低位：**我们˙知道数组位置的计算方式是(length-1)&hash，假设我们的table长度是16，也就是10000，减去1是15，即01111。计算如下：010101001001特别之处在于右起第五位是0。

~~~java
000000001111                     000000010000 
010101001001 // 结果 9            010101001001 // &运算结果： 0
~~~

当我们扩容之后，16变成32，也就是100000，计算如下：

~~~java
000000011111                    
010101001001 // 结果还是 9
~~~

结论是：扩容之后，元素在数组位置不变。再继续假设：换个hash数010101010001，特别之处右起第五位是1。

~~~java 
000000001111                      000000010000
010101010001 // 结果 1             010101010001 // &运算结果： 1
~~~

当我们扩容之后，16变成32，也就是100000，计算如下：

~~~java
000000011111
010101010001 // 结果变化：10001 == 17
~~~

结论：17比1大16，正好是扩容的大小。现在明白了，为啥0在低位，1在高低位。

#### helpTransfer

```java
// 说明当前节点是forwardingNode，帮助去扩容
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
```

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);// 生成扩容戳
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) { // 说明扩容还未完成的情况下不断循环来尝试将当前线程加入到扩容操作中
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;// 扩容结束
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {// 在低16位上增加扩容线程数
                transfer(tab, nextTab);// 帮助扩容
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

#### 扩容总结

transfer 方法可以说很牛逼，很精华，内部多线程扩容性能很高，通过给每个线程分配桶区间，避免线程间的争用，通过为**每个桶节点加锁**，避免 putVal 方法导致数据不一致。同时，在扩容的时候，也会将链表拆成两份，**A 类的链表保持位置不动，B 类的链表为 14+16(扩容增加 的长度)=30**，高低位的设计，大大增加了扩容的效率，这点和 HashMap 的 resize 方法类似。而如果有新的线程想 put 数据时，也会帮助其扩容。鬼斧神工，令人赞叹。

### JDK1.7与1.8的区别

- 1.7采用分段锁的思想，把一个map分成16个segment，通过每次锁住一个segment来保证每个segment内的操作的线程安全性从而实现全局线程安全。每次操作都会映射到一个segment，理论上可以支持16个线程的并发写入。1.8取消了segment分段设计，直接使用node数组来保存数据，并且采用node数组元素作为锁来实现每一行数据进行加锁来进一步减少并发冲突的概率（***减小锁粒度***）。

![image-20190813163850533](http://ww3.sinaimg.cn/large/006tNc79ly1g61pqu2l2ij310c0r6akt.jpg)

- 将原本数组+单向链表的数据结构变更为数组+单向链表+红黑树。红黑树结构使得查询的时间复杂度降低到O(logN))，可以提升查找的性能。
- 增加了CAS操作来确保node的一些操作原子性，这种方式代替了锁
- 设计了moved状态，会帮助扩容
- 至于为什么JDK8中使用synchronized而不是ReentrantLock，我猜是因为JDK8中对synchronized有了足够的优化吧



## 红黑树

[参考](https://juejin.im/post/5a27c6946fb9a04509096248#comment)

为了解决二叉树多次插入新节点而导致的不平衡！
1、节点是红色或黑色。
2、根节点是黑色。
3、每个叶子节点都是黑色的空节点（NIL节点）。
4、每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
5、从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。
这些规则就是保证了红黑树的平衡，当插入或者删除节点的时候，为了保证红黑树的规则，必须要做出一些调整（变色或者旋转），来继续维持我们的规则。