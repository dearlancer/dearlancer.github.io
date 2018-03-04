---
layout: post
title: HashMap和ConcurrentHashMap的实现原理
excerpt: ""
categories:
- java
- 源码
comments: true
---
## 线程不安全的HashMap
常用的底层数据结构主要有数组和链表。数组存储区间连续，占用内存较多，寻址容易，插入和删除困难。链表存储区间离散，占用内存较少，寻址困难，插入和删除容易。
HashMap要实现的是哈希表的效果，尽量实现O(1)级别的增删改查。它的具体实现则是同时使用了数组和链表，可以认为最外层是一个数组，数组的每个元素是一个链表的表头
### 构造
* int initialCapacity 数组长度，假如未指定长度，默认构造长度为16的table数组，否则构造指定长度的两倍的长度的数组。
* 负载因子 当数据达到容量*负载因子时，hashMap会主动扩容至原来的两倍。默认为0.75
### 寻址方式
对于新插入的数据或者待读取的数据，HashMap将Key的哈希值对数组长度取模，结果作为该Node在数组中的index。并忘对应index的链表中插入数据。
### 优化的存储方式
当链表的长度超过8时，为了寻址和插入等操作的方便，HashMap会把对应index下的链表转换为红黑树进行存储。
```ruby
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```
### 线程不安全的主要原因：resize死循环&Fast-fail
1. resize死循环
当HashMap的size超过Capacity*loadFactor时，需要对HashMap进行扩容。具体方法是，创建一个新的，长度为原来Capacity两倍的数组，保证新的Capacity仍为2的N次方，从而保证上述寻址方式仍适用。同时需要通过如下transfer方法将原来的所有数据全部重新插入（rehash）到新的数组中，如下可见，该HashMap的扩容其实是对链表做一个逆序操作的。当两个线程同时进行resize的时候，很有可能出现链表的死循环问题。
```ruby
void transfer(Entry[] newTable, boolean rehash) {
  int newCapacity = newTable.length;
  for (Entry<K,V> e : table) {
    while(null != e) {
      Entry<K,V> next = e.next;
      if (rehash) {
        e.hash = null == e.key ? 0 : hash(e.key);
      }
      int i = indexFor(e.hash, newCapacity);
      e.next = newTable[i];
      newTable[i] = e;
      e = next;
    }
  }
}
```
2.  在使用迭代器的过程中如果HashMap被修改，那么ConcurrentModificationException将被抛出，也即Fast-fail策略。
当HashMap的iterator()方法被调用时，会构造并返回一个新的EntryIterator对象，并将EntryIterator的expectedModCount设置为HashMap的modCount（该变量记录了HashMap被修改的次数）,
在通过该Iterator的next方法访问下一个Entry时，它会先检查自己的expectedModCount与HashMap的modCount是否相等，如果不相等，说明HashMap被修改，直接抛出ConcurrentModificationException。该Iterator的remove方法也会做类似的检查。该异常的为hashMap主动抛出的，意在提醒用户及早意识到线程安全问题
```ruby
HashIterator() {
  expectedModCount = modCount;
  if (size > 0) { // advance to first entry
  Entry[] t = table;
  while (index < t.length && (next = t[index++]) == null)
    ;
  }
}
```

# ConcurrentHashMap
ConcurrentHashMap的迭代中主要经历了两种方式的实现：
1. Java 7基于分段锁的ConcurrentHashMap 
2. Java 8基于CAS的ConcurrentHashMap
##  Java 7基于分段锁的ConcurrentHashMap 
Java 7中的ConcurrentHashMap的底层数据结构仍然是数组和链表。与HashMap不同的是，ConcurrentHashMap最外层不是一个大的数组，而是一个Segment的数组。每个Segment包含一个与HashMap数据结构差不多的链表数组。
![jdk1.7_hashMap](/img/jdk1.7_hashMap.png)
### 寻址方式
在读写某个Key时，先取该Key的哈希值。并将哈希值的高N位对Segment个数取模从而得到该Key应该属于哪个Segment，接着如同操作HashMap一样操作这个Segment。为了保证不同的值均匀分布到不同的Segment，需要通过如下方法计算哈希值。
```ruby
private int hash(Object k) {
  int h = hashSeed;
  if ((0 != h) && (k instanceof String)) {
    return sun.misc.Hashing.stringHash32((String) k);
  }
  h ^= k.hashCode();
  h += (h <<  15) ^ 0xffffcd7d;
  h ^= (h >>> 10);
  h += (h <<   3);
  h ^= (h >>>  6);
  h += (h <<   2) + (h << 14);
  return h ^ (h >>> 16);
}
```
### 同步方式
Segment继承自ReentrantLock，所以我们可以很方便的对每一个Segment上锁。
对于读操作，获取Key所在的Segment时，需要保证可见性。ConcurrentHashMap使用如下方法保证可见性，取得最新的Segment。
```ruby
Segment<K,V> s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)
```
获取Segment中的Node时也使用了类似方法
```ruby
Node<K,V> e = (Node<K,V>) UNSAFE.getObjectVolatile
  (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE)
```
对于写操作，并不要求同时获取所有Segment的锁，因为那样相当于锁住了整个Map。它会先获取该Key-Value对所在的Segment的锁，获取成功后就可以像操作一个普通的HashMap一样操作该Segment，并保证该Segment的安全性。
同时由于其它Segment的锁并未被获取，因此理论上可支持concurrencyLevel（等于Segment的个数）个线程安全的并发读写
### size操作
put、remove和get操作只需要关心一个Segment，而size操作需要遍历所有的Segment才能算出整个Map的大小。为更好支持并发操作，ConcurrentHashMap会在不上锁的前提逐个Segment计算3次size，如果某相邻两次计算获取的所有Segment的更新次数（每个Segment都与HashMap一样通过modCount跟踪自己的修改次数，Segment每修改一次其modCount加一）相等，说明这两次计算过程中无更新操作，则这两次计算出的总size相等，可直接作为最终结果返回。如果这三次计算过程中Map有更新，则对所有Segment加锁重新计算Size.
```ruby
public int size() {
  final Segment<K,V>[] segments = this.segments;
  int size;
  boolean overflow; // true if size overflows 32 bits
  long sum;         // sum of modCounts
  long last = 0L;   // previous sum
  int retries = -1; // first iteration isn't retry
  try {
    for (;;) {
      if (retries++ == RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
          ensureSegment(j).lock(); // force creation
      }
      sum = 0L;
      size = 0;
      overflow = false;
      for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j);
        if (seg != null) {
          sum += seg.modCount;
          int c = seg.count;
          if (c < 0 || (size += c) < 0)
            overflow = true;
        }
      }
      if (sum == last)
        break;
      last = sum;
    }
  } finally {
    if (retries > RETRIES_BEFORE_LOCK) {
      for (int j = 0; j < segments.length; ++j)
        segmentAt(segments, j).unlock();
    }
  }
  return overflow ? Integer.MAX_VALUE : size;
}
```
### HashMap与jdk1.7下ConcurrentHashMap的区别
ConcurrentHashMap与HashMap相比，有以下不同点
1. ConcurrentHashMap线程安全，而HashMap非线程安全
2. HashMap允许Key和Value为null，而ConcurrentHashMap不允许
3. HashMap不允许通过Iterator遍历的同时通过HashMap修改（会抛出异常），而ConcurrentHashMap允许该行为，并且该更新对后续的遍历可见
## Java 8基于CAS的ConcurrentHashMap
### 实现
ConcurrentHashMap在构造上采用了更优秀的CAS操作来保证操作的原子性。对通过对数组和Node中引用使用volatile关键字来保证内存可见性。以及对put操作的
![jdk1.8_hashMap](/img/jdk1.8_hashMap.png)
### 寻址方式
Java 8的ConcurrentHashMap同样是通过Key的哈希值与数组长度取模确定该Key在数组中的索引。
### 同步方式
Java 8的ConcurrentHashMap对数组使用volatile保证了内存可见性。并且对Node中的next引用使用同样的方式保证可见行。
对于put操作，如果Key对应的数组元素为null，则通过CAS操作将其设置为当前值。如果Key对应的数组元素（也即链表表头或者树的根元素）不为null，则对该元素使用synchronized关键字申请锁，然后进行操作。如果该put操作使得当前链表长度超过一定阈值，则将该链表转换为树，从而提高寻址效率。
对于读操作，由于数组被volatile关键字修饰，因此不用担心数组的可见性问题。同时每个元素是一个Node实例（Java 7中每个元素是一个HashEntry），它的Key值和hash值都由final修饰，不可变更，无须关心它们被修改后的可见性问题。而其Value及对下一个元素的引用由volatile修饰，可见性也有保障。
```ruby
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  volatile V val;
  volatile Node<K,V> next;
}
```
### size操作
#### put方法和remove方法都会通过addCount方法维护Map的size。
##### putVal的逻辑，使用了cas操作来进行插入
1. 检查key/value是否为空，如果为空，则抛异常，否则进行2
2. 进入for死循环，进行3
3. 检查table是否初始化了，如果没有，则调用initTable()进行初始化然后进行 2，否则进行4
4. 根据key的hash值计算出其应该在table中储存的位置i，取出table[i]的节点用f表示。
    根据f的不同有如下三种情况：1）如果table[i]==null(即该位置的节点为空，没有发生碰撞)，
                                则利用CAS操作直接存储在该位置，如果CAS操作成功则退出死循环。
                                2）如果table[i]!=null(即该位置已经有其它节点，发生碰撞)，碰撞处理也有两种情况
                                    2.1）检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
                                    2.2）说明table[i]的节点的hash值不等于MOVED，如果table[i]为链表节点，则将此节点插入链表中即可
                                        如果table[i]为树节点，则将此节点插入树中即可。插入成功后，进行 5
5. 如果table[i]的节点是链表节点，则检查table的第i个位置的链表是否需要转化为数，如果需要则调用treeifyBin函数进行转化
##### remove方法的逻辑如下
1. 先根据key的hash值计算书其在table的位置 i。
2. 检查table[i]是否为空，如果为空，则返回null，否则进行3
3. 在table[i]存储的链表(或树)中开始遍历比对寻找，如果找到节点符合key的，则判断value是否为null来决定是否是更新oldValue还是删除该节点。

size方法通过sumCount获取由addCount方法维护的Map的size。
```ruby
 public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
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

### Last 关于hashMap时间复杂度的问题
Hashmap中，get操作的时间复杂度应分为数组查找+链表（树）查询两步
理论最优化的复杂度为o(1),此时时间复杂度应为o(1)+o(1),也就是默认hash不冲突，实际复杂度应为O（1）+O（n）
当链表长度超过8时，链表会自动转化为红黑树，此时的时间复杂度为O（1）+O（log n）