# ConcurrentHashMap等并发集合【面试超高频考点】

## 1. 并发容器概览

ConcurrentHashMap：线程安全的HashMap

CopyOnWriteArrayList: 线程安全的List

BlockingQueue: 这是一个接口，表示阻塞队列，非常适合用于作为数据共享的通道。

ConcurrentLinkedQueue：高效的非阻塞并发队列，使用链表实现。可以看做一个线程安全的LinkedList。

ConcurrentSkipListMap: 是一个Map，使用跳表的数据结构进行快速查找。

## 2. 趣说集合类的历史——古老和过时的同步容器

### Vector和Hashtable



#### ArrayList和HashMap

##### 虽然这两个类不是线程安全的，但是可以用Collections.synchronizedList(new ArrayList<E>())和Collections.synchronizedMap(new HashMap<K, V>())使之变成线程安全的

#### ConcurrentHashMap和CopyOnWriteArrayList

##### 绝大多数并发情况下，ConcurrentHashMap和CopyOnWriteArrayList的性能都优于同步的HashMap和同步的ArrayList，唯一例外的是一个List经常被修改，那么同步的ArrayList性能会优于CopyOnWriteArrayList， CopyOnWriteArrayList更适合读多写少的场景，因为它每次写入都需要完整复制，比较消耗资源。

## 3. ConcurrentHashMap（重点、面试常考）

#### 磨刀不误砍柴工：Map简介

##### HashMap

##### Hashtable

##### LinkedHashMap

##### TreeMap

##### 常用方法

#### 为什么需要ConcurrentHashMap？

##### 为什么不用Collections.synchronizedMap()

###### Collections.synchronizedMap()是线程安全的，但是它是通过使用一个全局的锁来同步不同线程间的并发访问，因此会带来较大的性能问题。

##### 为什么HashMap是线程不安全的？

###### 同时put碰撞导致数据丢失

###### 同时put扩容导致数据丢失

###### 死循环造成的CPU100%

###### 彩蛋：调试技巧——如何修改JDK版本，从8到7

####### File-Project Structure，下载JDK7然后安装，然后添加JDK进来，然后选择。新建module选择JDK7，可以不影响到当前JDK8的代码。

###### 彩蛋：调式技巧——多线程配合，模拟真实场景

####### 如果不会这个调试技巧，永远都无法在本地模拟出线上bug的情况的，因为出现的几率太低了。

###### HashMap在高并发下的死循环（仅在JDK7及以前存在）

####### 代码演示

## 4. 九层之台，起于累土、罗马不是一天建成的：HashMap分析

##### 结构图

##### 红黑树介绍

###### 红黑树是每个节点都带有颜色属性的二叉查找树，本质是对二叉查找树BST的一种平衡策略，颜色为红色或黑色。

###### 我们理解为是一种平衡二叉查找树就可以，查找效率高，会自动平衡，防止极端不平衡从而影响查找效率的情况发生。

##### HashMap关于并发的特点

#### JDK1.7的ConcurrentHashMap实现和分析

##### 整体概念

###### Java 7中的ConcurrentHashMap最外层是多个segment，每个segment的底层数据结构与HashMap类似，仍然是数组和链表组成的拉链法。

###### 每个segment独立上ReentrantLock锁，每个segment之间互不影响，提高了并发效率。

###### ConcurrentHashMap 默认有 16 个 Segments，所以最多可以同时支持 16 个线程并发写（操作分别分布在不同的 Segment 上）。这个默认值可以在初始化的时候设置为其他值，但是一旦初始化以后，是不可以扩容的。

##### Segment图解

#### JDK1.8的ConcurrentHashMap实现和源码分析

##### 简介

##### 结构

##### 源码分析（1.8）

#### 对比JDK1.7和1.8的优缺点（为什么要把1.7的结构改成1.8的结构）

#### 组合操作：ConcurrentHashMap也不是线程安全的？

#### 实际生产案例

## 5. CopyOnWriteArrayList

#### 诞生的历史和原因

##### Vector和SynchronizedList的锁的粒度太大，并发效率相对比较低，并且迭代时无法编辑

#### 适用场景

##### 读操作可以尽可能地快，而写即使慢一些也没有太大关系

##### 读多写少

###### 黑名单，每日更新

###### 监听器：迭代操作远多余修改操作

#### 读写规则

##### 回顾读写锁

###### 读读共享、其他都互斥（写写互斥、读写互斥、写读互斥）

##### 读写锁规则的升级

###### 读取是完全不用加锁的，并且更厉害的是：写入也不会阻塞读取操作。只有写入和写入之间需要进行同步等待

#### 代码演示

#### 实现原理

##### CopyOnWrite的含义

##### 创建新副本、读写分离

##### “不可变”原理

##### 迭代的时候

#### 缺点

##### 内存占用问题

###### 因为CopyOnWrite的写是复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存。

##### 数据一致性问题

###### CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

#### 源码分析

##### 数据结构

###### 底层是数组

##### get

###### get操作都没有加锁，保证了读取操作的高速。

##### add操作

###### 在添加的时候就上锁，并复制一个新数组，增加操作在新数组上完成，将array指向到新数组中，最后解锁。

##### 迭代器

###### 执行迭代操作的时候，操作的都是原数组，而原数组不会被修改（修改都会去修改副本数组），所以执行迭代操作不需要加锁，也不会抛异常。

## 6. 并发队列Queue（阻塞、非阻塞队列）

#### 为什么要使用队列

##### 用队列可以安全地在线程间传递数据：生产者消费者模式、银行转账

##### 考虑锁等线程安全问题的重任从“你”转移到了“队列”上

#### 并发队列简介

##### Queue

##### BlockingQueue

#### 各并发队列关系图

#### 彩蛋：画漂亮的UML图

#### ArrayBlockingQueue

##### 有界

##### 指定容量

##### 公平

#### LinkedBlockingQueue

##### 无界

###### 容量Integer.MAX_VALUE

#### PriorityBlockingQueue

##### 支持优先级

##### 自然顺序（而不是先进先出）

##### 无界队列

##### PriorityQueue 的线程安全版本

#### SynchronousQueue

##### 功能

###### SynchronousQueue首先是一个阻塞队列，然后不同之处在于，它的容量为0 ，所以没有一个地方来暂存元素，导致每次取数据都要先阻塞，直到有数据被放入；同理，每次放数据的时候也会阻塞，直到有消费者来取。

###### 需要注意的是，SynchronousQueue的容量不是1而是0，因为SynchronousQueue不需要去持有元素，它所做的就是直接传递（direct handoff）。

###### 每当需要传递的时候，SynchronousQueue会把元素直接从生产者传给消费者，在此期间并不需要做存储，所以效率很高

##### 注意点

###### 1.	SynchronousQueue没有peek等函数，因为peek的含义是取出头结点，但是SynchronousQueue的容量是0，所以连头结点都没有，也就没有peek方法。

###### 2.	同理，没有iterate相关方法。

###### 3.	是一个极好的用来直接传递的并发数据结构。

###### 4.	SynchronousQueue是线程池Executors.newCachedThreadPool()使用的阻塞队列。

#### 非阻塞队列ConcurrentLinkedQueue

##### 并发包中的非阻塞队列只有ConcurrentLinkedQueue这一种，顾名思义ConcurrentLinkedQueue是使用链表作为其数据结构的，使用 CAS 非阻塞算法来实现线程安全（不具备阻塞功能），适合用在对性能要求较高的并发场景。用的相对比较少一些。

#### 如何选择适合自己的队列？

##### 边界

##### 空间

##### 吞吐量

### 各并发容器总结

#### java.util.concurrent包提供的容器，分为3类：Concurrent*、CopyOnWrite*、Blocking*