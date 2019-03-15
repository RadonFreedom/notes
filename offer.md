# offer

## 漫谈

1. 说项目，说平常怎么学习java，说除了项目有自己开发过什么。

   现在正在写一个基于Spring Cloud微服务架构的后台管理系统，实现简单的CURD操作，最重要的是感受微服务架构。主要参考的是github上的另外一个微服务项目。

2. 什么是微服务？

   微服务是基于分布式系统构建的，但是它对于服务的要求更独立更原子，不像SOA更多注重的是代码和功能复用。



## JVM

### 运行时内存



### 对象的内存布局

![分代收集](images\offer\对象内存.png)

其中对象头32位还是64位与虚拟机是多少位有关。

![img](https://img-blog.csdn.net/20140619204318875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxuanVscA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Mark Word

Mark Word是一个复用的数据结构。在不同的对象状态下，其存储的数据也不尽相同。

![img](https://img-blog.csdn.net/20140619210443906?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxuanVscA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### JVM ERROR

### 垃圾收集

#### 可达性分析算法

通过一系列成为GC roots的对象作为起始点，从这些节点开始向下搜索，搜索过的路径成为引用链，当一个对象没有任何引用链相连，证明这个对象不可用。

#### 垃圾收集算法

| 算法 | 标记清除                                                     | 复制算法                                                     | 标记整理                                             |
| ---- | :----------------------------------------------------------- | :----------------------------------------------------------- | ---------------------------------------------------- |
| 描述 | 将不用的对象标记，然后清除其所在空间。                       | 将内存按容量划分为大小相等的两块，每次使用一块。内存区满就把存活的对象复制到另外一块。 | 每次GC让存活对象向一端移动，然后清理掉边界外的内存。 |
| 优点 | 简单                                                         | 没有碎片问题，实现简单                                       |                                                      |
| 缺点 | 效率低，产生大量的内存碎片，内存碎片多了就无法存储大对象，不得不提前GC | 内存代价高                                                   |                                                      |

#### 分代回收算法

1. 对象优先在Eden中分配

2. 大对象直接进入老年代

3. 长期存活对象直接进入老年代

   对象动态年龄判定，在经历15次MinorGC后进入老年代

![分代收集](images\offer\分代收集.png)

#### 垃圾收集器

| 收集器                | 串行、并行or并发 | 新生代/老年代 | 算法               | 目标         | 适用场景                                  |
| --------------------- | ---------------- | ------------- | ------------------ | ------------ | ----------------------------------------- |
| **Serial**            | 串行             | 新生代        | 复制算法           | 响应速度优先 | 单CPU环境下的Client模式                   |
| **Serial Old**        | 串行             | 老年代        | 标记-整理          | 响应速度优先 | 单CPU环境下的Client模式、CMS的后备预案    |
| **ParNew**            | 并行             | 新生代        | 复制算法           | 响应速度优先 | 多CPU环境时在Server模式下与CMS配合        |
| **Parallel Scavenge** | 并行             | 新生代        | 复制算法           | 吞吐量优先   | 在后台运算而不需要太多交互的任务          |
| **Parallel Old**      | 并行             | 老年代        | 标记-整理          | 吞吐量优先   | 在后台运算而不需要太多交互的任务          |
| **CMS**               | 并发             | 老年代        | 标记-清除          | 响应速度优先 | 集中在互联网站或B/S系统服务端上的Java应用 |
| **G1**                | 并发             | both          | 标记-整理+复制算法 | 响应速度优先 | 面向服务端应用，将来替换CMS               |

##### CMS收集器

**CMS（Concurrent Mark Sweep）**收集器是一种以**获取最短回收停顿时间**为目标的收集器，它非常符合那些集中在互联网站或者B/S系统的服务端上的Java应用，这些应用都非常重视服务的响应速度。从名字上（“Mark Sweep”）就可以看出它是基于**“标记-清除”**算法实现的。

CMS收集器工作的整个流程分为以下4个步骤：

- **初始标记（CMS initial mark）**：仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，需要“Stop The World”。
- **并发标记（CMS concurrent mark）**：进行**GC Roots Tracing**的过程，在整个过程中耗时最长。
- **重新标记（CMS remark）**：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。此阶段也需要“Stop The World”。
- **并发清除（CMS concurrent sweep）**

由于整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作，所以，从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。通过下图可以比较清楚地看到CMS收集器的运作步骤中并发和需要停顿的时间：

![img](images\offer\f60599b2.png)

**优点**

**并发收集**、**低停顿**：标记和清理都是并发的，STW的初始标记和重新标记时间很短。

**缺点**

- **并发GC，因而对CPU资源非常敏感** 
- **标记-清除算法导致的空间碎片**

##### G1垃圾收集器

G1垃圾收集算法主要应用在多CPU大内存的服务中，在**满足高吞吐量的同时，尽可能的满足垃圾回收时的暂停时间**，该设计主要针对如下应用场景：

- 垃圾收集线程和应用线程并发执行，和CMS一样
- 不希望牺牲太多的吞吐性能
- 可预测的GC暂停时间，在一个长度M ms的时间片中，消耗在GC上的时间不得超过N ms

###### 堆内存结构

1、以往的垃圾回收算法，如CMS，使用的堆内存结构如下：



![img](images\offer\2184951-f6a73e5ef608cfa8.png)



- 新生代：eden space + 2个survivor
- 老年代：old space
- 持久代：1.8之前的perm space
- 元空间：1.8之后的metaspace

这些space必须是地址连续的空间。

2、**在G1算法中，采用了另外一种完全不同的方式组织堆内存**，堆内存被划分为多个大小相等的内存块（Region），每个Region是逻辑连续的一段内存，结构如下：

![img](images\offer\2184951-715388c6f6799bd9-1552460099982.png)



每个Region被标记了E、S、O和H，说明每个Region在运行时都充当了一种角色，其中H是以往算法中没有的，它代表Humongous，这表示这些Region存储的是巨型对象（humongous object，H-obj），当新建对象大小超过Region大小一半时，直接在新的一个或多个连续Region中分配，并标记为H。

**Region**

堆内存中一个Region的大小可以通过`-XX:G1HeapRegionSize`参数指定，大小区间只能是1M、2M、4M、8M、16M和32M，总之是2的幂次方，如果G1HeapRegionSize为默认值，则在堆初始化时计算Region的实际大小，具体实现如下：

![img](images\offer\2184951-c6194652e3232be2-1552460118628.png)

默认把堆内存按照2048份均分，最后得到一个合理的大小。

###### GC模式

G1中提供了三种模式垃圾回收模式，young gc、mixed gc 和 full gc，在不同的条件下被触发。

**young gc**

发生在年轻代的GC算法，一般对象（除了巨型对象）都是在eden region中分配内存，当所有eden region被耗尽无法申请内存时，就会触发一次young gc，这种触发机制和之前的young gc差不多，执行完一次young gc，活跃对象会被拷贝到survivor region或者晋升到old region中，空闲的region会被放入空闲列表中，等待下次被使用。

| 参数                    | 含义                                |
| ----------------------- | ----------------------------------- |
| -XX:MaxGCPauseMillis    | 设置G1收集过程目标时间，默认值200ms |
| -XX:G1NewSizePercent    | 新生代最小值，默认值5%              |
| -XX:G1MaxNewSizePercent | 新生代最大值，默认值60%             |

**mixed gc**

当越来越多的对象晋升到老年代old region时，为了避免堆内存被耗尽，虚拟机会触发一个混合的垃圾收集器，即mixed gc，该算法并不是一个old gc，除了回收整个young region，还会回收一部分的old region，这里需要注意：是一部分老年代，而不是全部老年代，可以选择哪些old region进行收集，从而可以对垃圾回收的耗时时间进行控制。

那么mixed gc什么时候被触发？

先回顾一下cms的触发机制，如果添加了以下参数：

```
`-XX:CMSInitiatingOccupancyFraction=``80` `-XX:+UseCMSInitiatingOccupancyOnly`
```

当老年代的使用率达到80%时，就会触发一次cms gc。相对的，mixed gc中也有一个阈值参数 `-XX:InitiatingHeapOccupancyPercent`，当老年代大小占整个堆大小百分比达到该阈值时，会触发一次mixed gc.

mixed gc的执行过程有点类似cms，主要分为以下几个步骤：

1. initial mark: 初始标记过程，整个过程STW，标记了从GC Root可达的对象
2. concurrent marking: 并发标记过程，整个过程gc collector线程与应用线程可以并行执行，标记出GC Root可达对象衍生出去的存活对象，并收集各个Region的存活对象信息
3. remark: 最终标记过程，整个过程STW，标记出那些在并发标记过程中遗漏的，或者内部引用发生变化的对象
4. clean up: 垃圾清除过程，如果发现一个Region中没有存活对象，则把该Region加入到空闲列表中

**full gc**

如果对象内存分配速度过快，mixed gc来不及回收，导致老年代被填满，就会触发一次full gc，G1的full gc算法就是单线程执行的serial old gc，会导致异常长时间的暂停时间，需要进行不断的调优，尽可能的避免full gc.

#### Metaspace

如果Metaspace的空间占用达到了设定的最大值，那么就会触发GC来收集死亡对象和类的加载器。G1和CMS都会很好地收集Metaspace区（一般都伴随着Full GC）。

## 并发

### 关键字

#### 乐观锁和悲观锁

##### 悲观锁

假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。**共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**。

- `synchronized`和`ReentrantLock`等独占锁
- 行锁，表锁，读锁，写锁，间隙锁

##### 乐观锁

假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据。**乐观锁适用于多读的应用类型，这样可以提高吞吐量**。

应用：

- CAS算法：Atomic类
- 版本号机制：Mysql的快照读
- tryLock 和 自旋锁

#### synchronized 和 volatile

|      | synchronized                                                 | volatile                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理 | 编译为class文件之后会形成monitorenter和monitorexit字节码指令，锁定传入的对象引用，以其作为锁（this，其他的引用，或this.getClass()） | CPU指令                                                      |
| 作用 | 保证当前线程访问代码的独占，确保被操作状态的原子性和可见性   | 将缓存及时刷新到内存和禁止指令重排序的作用，因此只能确保可见性 |

#### BIO 和 NIO

- BIO是阻塞IO，进行IO的线程会一直阻塞到数据IO完毕才结束，一个线程只能处理一个流的I/O事件。

- NIO是非阻塞IO，原理是轮询机制，Linux下底层实现用的是epoll。
  - poll/select：只能无差别轮询所有流，需要找出能读出数据，或者写入数据的流，对他们进行操作。
  - epoll：轮询得到的流都是可处理的流，没有可处理的流时将会阻塞。

### JUC包

#### ConcurrentHashMap

1. hash计算是依赖于put(key, val)中的key的，毕竟key值决定了这个键值对的存储位置和查找方式。如果传入的key值是null，hash为0，否则调用key.hashCode()并将低16位与高16位异或（保证高位的特征被利用上了）。
2. 

#### hashmap线程不安全的表现

- put get 操作不是线程安全的：是非原子操作。
- 扩容不是线程安全的：可能有多个线程同时扩容。

#### ReentrantLock

与内置锁相比的功能点：显式

- 可中断阻塞
- 可尝试获取锁
- 可实现公平锁
- 可以绑定多个Condition

#### CAS

1. CAS底层在`Hotspot`源码的`Unsafe.cpp`中实现，Linux的X86下主要是通过cmpxchg这个指令在CPU级完成CAS操作的。
2. 简述：若内存位置与预期原值匹配则处理器将该位置更新为新值。否则不做操作。
3. Atomic类全都用到了CAS
4. 优点：非阻塞的同步机制，性能好，可伸缩性好（当增加计算资源CPU内存磁盘存储容量和IO带宽时，程序的处理能力相应增加），活跃性好（不会出现饥饿和死锁）
5. 缺点：
   - **ABA问题**：在CAS操作时，带上版本号，每修改一次，版本号+1，之后比较原值的时候还要比较版本号
   -   **循环时间长开销大**：自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销
   - **只能保证一个共享变量的原子操作**：使用锁或者利用`AtomicReference类`把多个共享变量合并成一个共享变量来操作。

#### AQS

AQS是ReentrantLock的实现。

1. 锁状态。state = 0，未被获取；state > 1，被获取。
2. 线程的阻塞和解除阻塞。AQS 中采用了 LockSupport.park(thread) 来挂起线程，用 unpark 来唤醒线程。
3. 阻塞队列。AQS 用的是一个 FIFO 的队列，就是一个链表，每个 node 都持有后继节点的引用。

首先，第一个线程调用 reentrantLock.lock()，翻到最前面可以发现，tryAcquire(1) 直接就返回 true 了，结束。只是设置了 state=1，连 head 都没有初始化，更谈不上什么阻塞队列了。要是线程 1 调用 unlock() 了，才有线程 2 来，那世界就太太太平了，完全没有交集嘛，那我还要 AQS 干嘛。

如果线程 1 没有调用 unlock() 之前，线程 2 调用了 lock(), 想想会发生什么？

线程 2 会初始化 head（new Node()），同时线程 2 也会插入到阻塞队列并挂起 (注意看这里是一个 for 循环，而且设置 head 和 tail 的部分是不 return 的，只有入队成功才会跳出循环)

首先，是线程 2 初始化 head 节点，此时 head==tail, waitStatus==0

![aqs-1](images\offer\aqs-1.png)

然后线程 2 入队：

![aqs-2](images\offer\aqs-2.png)

同时我们也要看此时节点的 waitStatus，我们知道 head 节点是线程 2 初始化的，此时的 waitStatus 没有设置， java 默认会设置为 0，但是到 shouldParkAfterFailedAcquire 这个方法的时候，线程 2 会把前驱节点，也就是 head 的waitStatus设置为 -1。

那线程 2 节点此时的 waitStatus 是多少呢，由于没有设置，所以是 0；

如果线程 3 此时再进来，直接插到线程 2 的后面就可以了，此时线程 3 的 waitStatus 是 0，到 shouldParkAfterFailedAcquire 方法的时候把前驱节点线程 2 的 waitStatus 设置为 -1。

![aqs-3](images\offer\aqs-3.png)

这里可以简单说下 waitStatus 中 SIGNAL(-1) 状态的意思，Doug Lea 注释的是：代表后继节点需要被唤醒。也就是说这个 waitStatus 其实代表的不是自己的状态，而是后继节点的状态，我们知道，每个 node 在入队的时候，都会把前驱节点的状态改为 SIGNAL，然后阻塞，等待被前驱唤醒。这里涉及的是两个问题：有线程取消了排队、唤醒操作。

### 锁的分类和优化

#### 重量级锁

JVM使用操作系统的互斥量mutex实现的锁

#### 自旋锁

JDK1.6引入了自适应的自旋锁。

自旋等待避免了线程切换的开销，但当且仅当自旋时间小于线程切换时间才有效果。

自适应是指按照该锁之前的自旋时间和线程执行时间来判断自旋次数。

#### 锁消除

JVM对一些代码上要求同步，但是被检测到不存在共享状态竞争的锁进行消除。

实现：JIT即时编译器逃逸分析技术，判断当前线程栈上SLOT数据都不会逃逸而被其他线程访问到，就可以认为他们是线程私有的，无需加锁。

#### 锁粗化

一段代码如要重复加上和释放同一个锁，JVM会扩展锁的锁定区域（如果不违背代码语义）来减少锁定开销。

#### 轻量级锁

- 目的：在同一时刻没有线程竞争锁时，减少传统重量级锁（JVM使用操作系统的互斥量mutex实现的锁）的使用。

- 使用依据：“对于绝大部分锁，在整个同步代码块内都不存在竞” 的经验法则。

- 实现：**使用CAS操作消除重量级锁同步使用的互斥量。**

  （1）在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝。

  （2）拷贝对象头中的Mark Word复制到锁记录中。

  （3）拷贝成功后，虚拟机将使用**CAS操作**尝试将对象的Mark Word更新为指向Lock Record的指针。如果更新成功，则执行步骤（3），否则执行步骤（4）。

  （4）CAS成功。该线程就拥有该对象的锁，将对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态。

  （5）CAS失败。检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程重入该对象的锁，可以直接进入同步块继续执行。

  （6）否则，说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，当前线程使用自旋来获取锁，后面等待锁的线程也要进入阻塞状态。

- 优点和缺点：满足经验法则的前提下提升性能；多个线程同时竞争锁时降低性能。

#### 偏向锁

目的：数据在无竞争时，消除代码中的同步需求。

实现：

​	（1）确认为可偏向状态。访问Mark Word中偏向锁的标识是否设置成1 && 锁标志位是否为01。

　　（2）如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）。

　　（3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置	为当前线程ID，然后执行（5）；如果竞争失败，执行（4）。

　　（4）如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂	起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

　　（5）执行同步代码。

缺点：如果某个对象的锁会被很多不同线程访问，偏向模式就是多余的。

### 线程池

#### 线程池参数

- `corePoolSize`：核心线程数

  - 核心线程会**一直存活**，即便没有任务需要执行

  - **当线程数小于核心线程数时，即使有线程空闲，线程池也会优先创建新线程处理**
  - 设置allowCoreThreadTimeout=true（默认false）时，核心线程会超时关闭

- `maximumPoolSize` ：最大线程数

  - 当线程数>=corePoolSize，且任务队列已满时，线程池会创建新线程来处理任务

  - 当线程数=maxPoolSize，且任务队列已满时，线程池会拒绝处理任务而抛出异常

- `keepAliveTime`：线程空闲时间

  当线程空闲时间达到keepAliveTime时，线程会退出，直到线程数量=corePoolSize

- `rejectedExecutionHandler`：任务拒绝处理器

  两种情况会拒绝处理任务：

  - 队列已满，线程数已经达到maxPoolSize，会拒绝新任务
  - 当线程池被调用shutdown()后，会等待线程池里的任务执行完毕，再shutdown。如果在调用shutdown()和线程池真正shutdown之间提交任务，会拒绝新任务

  线程池会调用rejectedExecutionHandler来处理这个任务。如果没有设置默认是AbortPolicy，会抛出异常。ThreadPoolExecutor类有几个内部实现类来处理这类情况：

  - AbortPolicy 丢弃任务，抛运行时异常

  - CallerRunsPolicy 执行任务
  - DiscardPolicy 忽视，什么都不会发生
  - DiscardOldestPolicy 从队列中踢出最先进入队列（最后一个执行）的任务

#### **原理**

线程池按以下行为执行任务：

1. 当线程数小于核心线程数corePoolSize时，创建线程。
2. 当线程数大于等于核心线程数，且任务队列未满时，将任务放入任务队列。
3. 当线程数大于等于核心线程数，且任务队列已满
   1. 若线程数小于最大线程数，创建线程
   2. 若线程数等于最大线程数，抛出异常，拒绝任务

#### SynchronousQueue

当一个线程往队列中写入一个元素时，写入操作不会立即返回，需要等待另一个线程来将这个元素拿走；同理，当一个读线程做读操作的时候，同样需要一个相匹配的写线程的写操作。

因为SynchronousQueue没有存储功能，因此put和take会阻塞，直到有另一个线程已经准备好参与到交付过程中。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付的工作时，才适合使用同步队列。

### 线程安全机制

| 线程安全机制 | 互斥（阻塞）同步                                             | 非阻塞同步                   | 线程封闭                |
| ------------ | ------------------------------------------------------------ | ---------------------------- | ----------------------- |
| 技术         | synchronized，volatile，Lock，Semaphore，CyclicBarrier，CountDownLatch | CAS，Atomic类                | ThreadLocal，无状态的类 |
| 优点         |                                                              |                              |                         |
| 缺点         | 阻塞或者唤醒一个线程，都需要OS从用户态切换到内核态，切换上下文的开销可能比一般的setter getter开销更大 | 只能是一个状态变量的原子操作 | 无法真正意义的并发      |



### 线程

原理：Java线程1：1映射到操作系统原生线程（内核线程、轻量级进程）之上。因此阻塞或者唤醒一个线程，都需要OS从用户态切换到内核态，切换上下文的开销可能比一般的setter getter开销更大。

### TODO:死锁的原因，如何避免



## 计算机网络



## 框架和中间件





## 命令行



## 算法题

1. 手写排序



## 数据库

### 数据库引擎及比较

| Feature                 | InnoDB | MyISAM |
| ----------------------- | ------ | ------ |
| **Clustered indexes**   | Yes    | No     |
| **Data caches**         | Yes    | No     |
| **Foreign key support** | Yes    | No     |
| **Locking granularity** | Row    | Table  |
| **MVCC**                | Yes    | No     |
| **Transactions**        | Yes    | No     |
| **Storage limits**      | 64TB   | 256TB  |

### 索引原理和实现

操作系统从磁盘读取数据到内存时是以磁盘块（block）为基本单位的，位于同一个磁盘块中的数据会被一次性读取出来。

InnoDB存储引擎中有页（Page）的概念，页是其磁盘管理的最小单位。InnoDB默认每个页的大小为16KB：

```
mysql> show variables like 'innodb_page_size';
```

B-Tree：每个节点中不仅包含数据的key值，还有data值。而每一个页的存储空间是有限的，如果data较大时将会导致每个节点（即一个页）能存储的key的数量小，B-Tree的深度大，增大查询时的磁盘I/O次数。

![20160202204827368](images\offer\20160202204827368)

在B+Tree中，所有数据记录节点都是按照键值大小顺序存放在同一层的叶子节点上，而非叶子节点上只存储key值信息，这样可以大大加大每个节点存储的key值数量，降低B+Tree的高度。
![1552547005558](images\offer\20160202205105560)

B+Tree的高度一般都在2~4层。InnoDB存储引擎**根节点常驻内存**，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作。

数据库中的B+Tree索引可以分为聚集索引（clustered index）和辅助索引（secondary index）。

- 聚集索引的B+Tree中的非叶子结点存放的是主键上的索引，叶子节点存放的是整张表的行记录数据。
- 辅助索引的叶子节点存储相应行数据的聚集索引键，即主键。
- 辅助索引来查询数据：先遍历辅助索引找到主键，然后再通过主键在聚集索引中找到完整的行记录数据。

### 事务

#### 什么是数据库事务

CURD复合原子操作。

#### 数据库事务的性质ACID

原子性(atomicity)
一个事务必须被视为一个不可分割的最小工作单元，整个事务中的所有操作要么全部提交成功，要么全部失败回滚。
一致性(consistency)
数据库总是从一个一致性的状态转换到另外一个一致性的状态，满足业务的一致性约束。
隔离性(isolation)
通常来说，一个事务所做的修改在最终提交以前，对其他事务是不可见的。
持久性(durability)
一且事务提交，则其所傲的修改就会永久保存到数据库中。此时即使系统崩溃，修改的数据也不会丢失。

#### 隔离级别

![1552548959252](images\offer\捕获.JPG)

- 脏读：事务可以读取其他事务尚未提交的数据。破坏了AI
- 不可重复读：在一个事务内两次SELECT查询相同的数据结果不同。
- 幻读：一个事务插入了新行，当另一个事务的DML操作涉及到该范围的数据行时，会失败，因为幻行被加锁。

#### MVCC：Snapshot Read vs Current Read

- **MVCC的快照读利用版本号解决了不可重复读的问题，但并没有解决幻读。**
- **MVCC的当前读解决了幻读问题，解决方法是GAP锁。**

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议MVCC (Multi-Version Concurrency Control) (注：与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。MVCC最大的好处：读不加锁，读写不冲突。在读多写少的OLTP (online transaction processing)应用中，极大的增加了系统的并发性能。

在MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。

**快照读：**读取的是记录的可见版本 (历史版本)，不用加锁。简单的select操作，属于快照读，不加锁。

- select * from table where ?;

**当前读：**读取的是记录的最新版本，当前读返回的记录都会加上锁，保证其他事务不会再并发修改这条记录。特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。

- select * from table where ? lock in share mode;
- select * from table where ? for update;
- insert into table values (…);
- update table set ? where ?;
- delete from table where ?;

所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁 (排它锁)。

![update](images/offer/medish.jpg)

从图中，可以看到，一个Update操作的具体流程。当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁 (current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。同理，Delete操作也一样。Insert操作会稍微有些不同，简单来说，就是Insert操作可能会触发Unique Key的冲突检查，也会进行一个当前读。

**注**：根据上图的交互，针对一条当前读的SQL语句，InnoDB与MySQL Server的交互，是一条一条进行的，因此，加锁也是一条一条进行的。先对一条满足条件的记录加锁，返回给MySQL Server，做一些DML操作；然后在读取下一条加锁，直至读取完毕。

#### 数据库如何实现串行化

```MYSQL
mysql> set global transaction isolation level read committed; //全局的

mysql> set session transaction isolation level read committed; //当前会话
```

### 锁

#### **事务中的加锁是每执行一条语句就加一层锁，直到事务结束释放所有锁。**

#### 判断事务中DML操作的加锁：

- **前提一：**id列是不是主键？

- **前提二：**当前系统的隔离级别是什么？

- **前提三：**id列如果不是主键，那么id列上有索引吗？

- **前提四：**id列上如果有二级索引，那么这个索引是唯一索引吗？

- **前提五：**SQL的执行计划是什么？索引扫描？全表扫描？

`delete from t1 where id = 10;  `  在不同组合下的加锁：

- **组合五：**id列是主键，RR隔离级别

![idä¸"é®+rc](http://pic.yupoo.com/hedengcheng/DnJ6RtaP/medish.jpg)

- **组合六：**id列是二级唯一索引，RR隔离级别

![id unique+rc](http://pic.yupoo.com/hedengcheng/DnJ6PDep/medish.jpg)

- **组合七：**id列是二级非唯一索引，RR隔离级别

![id 非唯一索引 + rr](images\offer\medish1.jpg)

为了保证两次当前读返回一致的记录，那就需要在第一次当前读与第二次当前读之间，其他的事务不会插入新的满足条件的记录并提交。GAP锁不会出现幻读的关键。GAP锁锁住的位置，也不是记录本身，而是两条记录之间的GAP。所谓幻读，就是同一个事务，连续做两次当前读 (例如：select * from t1 where id = 10 for update;)，那么这两次当前读返回的是完全相同的记录 (记录数量一致，记录本身也一致)，第二次的当前读，不会比第一次返回更多的记录 。

- **组合八：**id列上没有索引，RR隔离级别

  ![id æ ç´¢å¼+rr](http://pic.yupoo.com/hedengcheng/DnJ6Rf3q/medish.jpg)

- **组合九：**Serializable隔离级别

  Serializable隔离级别，影响的是`select * from t1 where id = 10;` 这条SQL，在RC，RR隔离级别下，都是快照读，不加锁。但是在Serializable隔离级别，SQL1会加读锁，也就是说快照读不复存在，MVCC并发控制降级为Lock-Based CC。

- **一条复杂的SQL**

  ![å¤æSQL](http://pic.yupoo.com/hedengcheng/DnJ6S3ta/medish.jpg)

  - **Index key：**pubtime > 1 and puptime < 20。此条件，用于确定SQL在idx_t1_pu索引上的查询范围。
  - **Index Filter：**userid = ‘hdc’ 。此条件，可以在idx_t1_pu索引上进行过滤，但不属于Index Key。

  - **Table Filter：**comment is not NULL。此条件，在idx_t1_pu索引上无法过滤，只能在聚簇索引上过滤。

  ![SQLå é](http://pic.yupoo.com/hedengcheng/DnJ6S1s7/medish.jpg)

  在Repeatable Read隔离级别下，针对一个复杂的SQL，首先需要提取其where条件。Index Key确定的范围，需要加上GAP锁；Index Filter过滤条件，在5.6后支持了Index Condition Pushdown，则在index上过滤，不满足Index Filter的记录不用加X锁(图中，用红色箭头标出的X锁，是否要加，视是否支持ICP而定)；Table Filter过滤条件，无论是否满足，都需要加X锁。

### 缓存

#### 页面置换算法Least Recently Used

内存不够的场景下，淘汰掉最不经常使用的数据。

##### LRU原理

假设内存只能容纳3个页大小，按照 7 0 1 2 0 3 0 4 的次序访问页。**假设内存按照栈的方式来描述访问时间，在上面的，是最近访问的，在下面的是，最远时间访问的，LRU就是这样工作的。**

![img](https://pic1.zhimg.com/80/v2-584ed398c35ba76250cfb2f01b20ec0c_hd.jpg)

但是如果让我们自己设计一个基于 LRU 的缓存，这样设计可能问题很多，这段内存按照访问时间进行了排序，会有大量的内存拷贝操作，所以性能肯定是不能接受的。

那么如何设计一个LRU缓存，使得放入和移除都是 O(1) 的，我们需要把访问次序维护起来，但是不能通过内存中的真实排序来反应，有一种方案就是使用双向链表。

##### 基于 HashMap 和 双向链表实现 LRU

整体的设计思路是，可以使用 HashMap 存储 key，这样可以做到 save 和 get key的时间都是 O(1)，而 HashMap 的 Value 指向双向链表实现的 LRU 的 Node 节点，如图所示。

![img](https://pic4.zhimg.com/80/v2-09f037608b1b2de70b52d1312ef3b307_hd.jpg)

LRU 存储是基于双向链表实现的，下面的图演示了它的原理。其中 head 代表双向链表的表头，tail 代表尾部。首先预先设置 LRU 的容量，如果存储满了，可以通过 O(1) 的时间淘汰掉双向链表的尾部，每次新增和访问数据，都可以通过 O(1)的效率把新的节点增加到对头，或者把已经存在的节点移动到队头。

下面展示了，预设大小是 3 的，LRU存储的在存储和访问过程中的变化。为了简化图复杂度，图中没有展示 HashMap部分的变化，仅仅演示了上图 LRU 双向链表的变化。我们对这个LRU缓存的操作序列如下：

save("key1", 7)

save("key2", 0)

save("key3", 1)

save("key4", 2)

get("key2")

save("key5", 3)

get("key2")

save("key6", 4)

相应的 LRU 双向链表部分变化如下：

![img](https://pic3.zhimg.com/80/v2-e9a42fee5cdbf9e4e4d23015112fad4e_hd.jpg)s = save, g = get

总结一下核心操作的步骤:

1. save(key, value)，首先在 HashMap 找到 Key 对应的节点，如果节点存在，更新节点的值，并把这个节点移动队头。如果不存在，需要构造新的节点，并且尝试把节点塞到队头，如果LRU空间不足，则通过 tail 淘汰掉队尾的节点，同时在 HashMap 中移除 Key。
2. get(key)，通过 HashMap 找到 LRU 链表节点，因为根据LRU 原理，这个节点是最新访问的，所以要把节点插入到队头，然后返回缓存的值。

完整基于 Java 的代码参考如下

```java
class DLinkedNode {
	String key;
	int value;
	DLinkedNode pre;
	DLinkedNode post;
}
```

LRU Cache

```java
public class LRUCache {
   
    private Hashtable<Integer, DLinkedNode>
            cache = new Hashtable<Integer, DLinkedNode>();
    private int count;
    private int capacity;
    private DLinkedNode head, tail;

    public LRUCache(int capacity) {
        this.count = 0;
        this.capacity = capacity;

        head = new DLinkedNode();
        head.pre = null;

        tail = new DLinkedNode();
        tail.post = null;

        head.post = tail;
        tail.pre = head;
    }

    public int get(String key) {

        DLinkedNode node = cache.get(key);
        if(node == null){
            return -1; // should raise exception here.
        }

        // move the accessed node to the head;
        this.moveToHead(node);

        return node.value;
    }


    public void set(String key, int value) {
        DLinkedNode node = cache.get(key);

        if(node == null){

            DLinkedNode newNode = new DLinkedNode();
            newNode.key = key;
            newNode.value = value;

            this.cache.put(key, newNode);
            this.addNode(newNode);

            ++count;

            if(count > capacity){
                // pop the tail
                DLinkedNode tail = this.popTail();
                this.cache.remove(tail.key);
                --count;
            }
        }else{
            // update the value.
            node.value = value;
            this.moveToHead(node);
        }
    }
    /**
     * Always add the new node right after head;
     */
    private void addNode(DLinkedNode node){
        node.pre = head;
        node.post = head.post;

        head.post.pre = node;
        head.post = node;
    }

    /**
     * Remove an existing node from the linked list.
     */
    private void removeNode(DLinkedNode node){
        DLinkedNode pre = node.pre;
        DLinkedNode post = node.post;

        pre.post = post;
        post.pre = pre;
    }

    /**
     * Move certain node in between to the head.
     */
    private void moveToHead(DLinkedNode node){
        this.removeNode(node);
        this.addNode(node);
    }

    // pop the current tail.
    private DLinkedNode popTail(){
        DLinkedNode res = tail.pre;
        this.removeNode(res);
        return res;
    }
}
```

那么问题的后半部分，是 Redis 如何实现，这个问题这么问肯定是有坑的，那就是redis肯定不是这样实现的。

##### Redis的LRU实现

如果按照HashMap和双向链表实现，需要额外的存储存放 next 和 prev 指针，牺牲比较大的存储空间，显然是不划算的。所以Redis采用了一个近似的做法，就是随机取出若干个key，然后按照访问时间排序后，淘汰掉最不经常使用的，具体分析如下：

为了支持LRU，Redis 2.8.19中使用了一个全局的LRU时钟，`server.lruclock`，定义如下，

```cpp
#define REDIS_LRU_BITS 24
unsigned lruclock:REDIS_LRU_BITS; /* Clock for LRU eviction */
```

默认的LRU时钟的分辨率是1秒，可以通过改变`REDIS_LRU_CLOCK_RESOLUTION`宏的值来改变，Redis会在`serverCron()`中调用`updateLRUClock`定期的更新LRU时钟，更新的频率和hz参数有关，默认为`100ms`一次

Redis支持和LRU相关淘汰策略包括，

- `volatile-lru` 设置了过期时间的key参与近似的lru淘汰策略
- `allkeys-lru` 所有的key均参与近似的lru淘汰策略

Redis会基于`server.maxmemory_samples`配置选取固定数目的key，然后比较它们的lru访问时间，然后淘汰最近最久没有访问的key，maxmemory_samples的值越大，Redis的近似LRU算法就越接近于严格LRU算法，但是相应消耗也变高，对性能有一定影响，样本值默认为5。

## 分布式

### 分布式事务

#### 分布式系统的一致性

一致性就是数据保持一致，在分布式系统中，可以理解为集群中多个节点中数据的值是一模一样的，也可以理解为不同节点的数据是满足某种一致性约束的。

而一致性又可以分为强一致性与弱一致性。

- 强一致性可以理解为在任意时刻，所有节点中的数据是一致的。同一时间点，你在节点A中获取到key1的值与在节点B中获取到key1的值应该都是一致的。
- 弱一致性包含很多种不同的实现，目前分布式系统中广泛实现的是最终一致性。

所谓最终一致性，就是不保证在任意时刻任意节点上的同一份数据都是相同的，但是随着时间的迁移，不同节点上的相关（一样的或者有一致性约束的）数据总是在向趋同的方向变化。也可以简单的理解为在一段时间后，节点间的数据会最终达到一致状态。

CDN的最终一致性。

#### 分布式锁，如何添加，放在什么位置

- 分布式与单机情况下最大的不同在于其不是多线程而是**多进程**。
- 多线程由于可以共享堆内存，因此可以简单的采取内存作为标记存储位置。而进程之间甚至可能都不在同一台物理机上，因此需要将标记存储在一个所有进程都能看到的地方。

##### 什么是分布式锁？

- 当在分布式模型下，数据只有一份（或有限制），此时需要利用锁的技术控制某一时刻修改数据的进程数。
- 与单机模式下的锁不仅需要保证进程可见，还需要考虑进程与锁之间的网络问题。（我觉得分布式情况下之所以问题变得复杂，主要就是需要考虑到**网络的延时和不可靠**。。。一个大坑）
- 分布式锁还是可以将标记存在内存，只是该内存不是某个进程分配的内存而是公共内存如 Redis、Memcache。至于利用数据库、文件等做锁与单机的实现是一样的，只要保证标记能互斥就行。