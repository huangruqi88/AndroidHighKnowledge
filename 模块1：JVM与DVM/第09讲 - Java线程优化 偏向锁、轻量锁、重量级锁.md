## 引言

Java 的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统的帮忙，这就要从用户态转换到核心态，就相当于工作从上海分部转换到瑞典总部的操作一样，因此状态转换需要花费很多的处理器时间。

比如如下代码：

[![引言01.png](https://z3.ax1x.com/2021/08/03/fi8zK1.png)](https://imgtu.com/i/fi8zK1)

value++ 因为被关键字 synchronized 修饰，所以会在各个线程间同步执行。但是 value++ 消耗的时间很有可能比线程状态转换消耗的时间还短，所以说 synchronized 是 Java 语言中一个重量级的操作。

## synchronized 实现原理

要了解 synchronized 的原理需要先理清楚两件事情：**对象头**和 **Monitor**。

#### 对象头

在“[大话Java对象在虚拟机中是什么样子](https://mp.weixin.qq.com/s/fyvoraVu9yjgqX-xhn6EHQ)”这篇文章中，提到了 Java 对象在内存中的布局分为 3 部分：**对象头、实例数据、对齐填充**。当我们在 Java 代码中，使用 new 创建一个对象的时候，JVM 会在堆中创建一个 instanceOopDesc 对象，这个对象中包含了对象头以及实例数据。

instanceOopDesc 的基类为 oopDesc 类。它的结构如下：

[![对象头01.png](https://z3.ax1x.com/2021/08/03/fiYWCD.png)](https://imgtu.com/i/fiYWCD)

其中 _mark 和 _metadata 一起组成了对象头。_metadata 主要保存了类元数据，不需要做过多介绍。这里重点看下 _mark 属性，_mark 是 markOop 类型数据，一般称它为标记字段（Mark Word），其中主要存储了对象的 hashCode、分代年龄、锁标志位，是否偏向锁等。

用一张图来表示 32 位 Java 虚拟机的 Mark Word 的默认存储结构如下：

[![对象头02.png](https://z3.ax1x.com/2021/08/03/fiYq58.png)](https://imgtu.com/i/fiYq58)

默认情况下，没有线程进行加锁操作，所以锁对象中的 **Mark Word** 处于无锁状态。但是考虑到 JVM 的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多的有效数据，它会根据对象本身的状态复用自己的存储空间，如 32 位 JVM 下，除了上述列出的 **Mark Word** 默认存储结构外，还有如下可能变化的结构：

[![对象头03.png](https://z3.ax1x.com/2021/08/03/fitE24.png)](https://imgtu.com/i/fitE24)

从图中可以看出，根据"锁标志位”以及"是否为偏向锁"，Java 中的锁可以分为以下几种状态：

[![对象头04.png](https://z3.ax1x.com/2021/08/03/fit3GD.png)](https://imgtu.com/i/fit3GD)

在 Java 6 之前，并没有轻量级锁和偏向锁，只有重量级锁，也就是通常所说 synchronized 的对象锁，锁标志位为 10。从图中的描述可以看出：当锁是重量级锁时，对象头中 Mark Word 会用 30 bit 来指向一个“互斥量”，而这个互斥量就是 **Monitor**。

#### Monitor

Monitor 可以把它理解为一个同步工具，也可以描述为一种同步机制。实际上，它是一个保存在对象头中的一个对象。在 markOop 中有如下代码：

[![Monitor_01.png](https://z3.ax1x.com/2021/08/03/fitxJO.png)](https://imgtu.com/i/fitxJO)

通过 monitor() 方法创建一个 ObjectMonitor 对象，而 ObjectMonitor 就是 Java 虚拟机中的 Monitor 的具体实现。因此 Java 中每个对象都会有一个对应的 ObjectMonitor 对象，这也是 Java 中所有的 Object 都可以作为锁对象的原因。

那 ObjectMonitor 是如何实现同步机制的呢？

首先看下 ObjectMonitor 的结构：

[![Monitor_02.png](https://z3.ax1x.com/2021/08/03/fiNnSg.png)](https://imgtu.com/i/fiNnSg)

其中有几个比较关键的属性：

[![Monitor_03.png](https://z3.ax1x.com/2021/08/03/fiN0mR.png)](https://imgtu.com/i/fiN0mR)

当多个线程同时访问一段同步代码时，首先会进入 _EntryList 队列中，当某个线程通过竞争获取到对象的 monitor 后，monitor 会把 _owner 变量设置为当前线程，同时 monitor 中的计数器 _count 加 1，即获得对象锁。

若持有 monitor 的线程调用 wait() 方法，将释放当前持有的 monitor，_owner 变量恢复为 null， _count 自减 1，同时该线程进入 _WaitSet 集合中等待被唤醒。若当前线程执行完毕也将释放 monitor（锁）并复位变量的值，以便其他线程进入获取 monitor（锁）。

##### 实例演示

比如以下代码通过 3 个线程分别执行以下同步代码块：

[![实例演示01.png](https://z3.ax1x.com/2021/08/03/fiUpNV.png)](https://imgtu.com/i/fiUpNV)

锁对象是 lock 对象，在 JVM 中会有一个 ObjectMonitor 对象与之对应。如下图所示：

[![实例演示02.png](https://z3.ax1x.com/2021/08/03/fiUM9O.png)](https://imgtu.com/i/fiUM9O)

分别使用 3 个线程来执行以上同步代码块。默认情况下，3 个线程都会先进入 ObjectMonitor 中的 EntrySet 队列中，如下所示：

[![实例演示03.png](https://z3.ax1x.com/2021/08/03/fiUcEq.png)](https://imgtu.com/i/fiUcEq)

假设线程 2 首先通过竞争获取到了锁对象，则 ObjectMonitor 中的 Owner 指向线程 2，并将 count 加 1。结果如下：

[![实例演示04.gif](https://z3.ax1x.com/2021/08/03/fidZY6.gif)](https://imgtu.com/i/fidZY6)

然后线程 1 和线程 3 再次通过竞争获取到锁（Monitor）对象，则重新将 Owner 指向成功获取到锁的线程。假设线程 1 获取到锁，如下：

[![实例演示05.png](https://z3.ax1x.com/2021/08/03/fidb9K.png)](https://imgtu.com/i/fidb9K)

如果在线程 1 执行过程中调用 notify 操作将线程 2 唤醒，则当前处于 WaitSet 中的线程 2 会被重新添加到 EntrySet 集合中，并尝试重新获取竞争锁（Monitor）对象。但是 notify 操作并不会是使程 1 释放锁（Monitor）对象。结果如下：

[![实例演示06.gif](https://z3.ax1x.com/2021/08/03/fiwP9f.gif)](https://imgtu.com/i/fiwP9f)

当线程 1 中的代码执行完毕以后，同样会自动释放锁，以便其他线程再次获取锁对象。

**实际上，ObjectMonitor 的同步机制是 JVM 对操作系统级别的 Mutex Lock（互斥锁）的管理过程，其间都会转入操作系统内核态。也就是说 synchronized 实现锁，在“重量级锁”状态下，当多个线程之间切换上下文时，还是一个比较重量级的操作。**

## Java 虚拟机对 synchronized 的优化

从 Java 6 开始，虚拟机对 synchronized 关键字做了多方面的优化，主要目的就是，**避免 ObjectMonitor 的访问，减少“重量级锁”的使用次数，并最终减少线程上下文切换的频率 **。其中主要做了以下几个优化： **锁自旋、轻量级锁、偏向锁**。

#### 锁自旋

线程的阻塞和唤醒需要 CPU 从用户态转为核心态，频繁的阻塞和唤醒对 CPU 来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力，所以 Java 引入了自旋锁的操作。实际上自旋锁在 Java 1.4 就被引入了，默认关闭，但是可以使用参数 -XX:+UseSpinning 将其开启。但是从 Java 6 之后默认开启。

所谓**自旋，就是让该线程等待一段时间，不会被立即挂起，看当前持有锁的线程是否会很快释放锁。而所谓的等待就是执行一段无意义的循环即可（自旋）**。

自旋锁也存在一定的缺陷：**自旋锁要占用 CPU，如果锁竞争的时间比较长，那么自旋通常不能获得锁，白白浪费了自旋占用的 CPU 时间。这通常发生在锁持有时间长，且竞争激烈的场景中，此时应主动禁用自旋锁。**

#### 轻量级锁

有时候 Java 虚拟机中会存在这种情形：**对于一块同步代码，虽然有多个不同线程会去执行，但是这些线程是在不同的时间段交替请求这把锁对象，也就是不存在锁竞争的情况。在这种情况下，锁会保持在轻量级锁的状态，从而避免重量级锁的阻塞和唤醒操作。**

要了解轻量级锁的工作流程，还是需要再次看下对象头中的 Mark Word。上文中已经提到，锁的标志位包含几种情况：00 代表轻量级锁、01 代表无锁（或者偏向锁）、10 代表重量级锁、11 则跟垃圾回收算法的标记有关。

要了解轻量级锁的工作流程，还是需要再次看下对象头中的 Mark Word。上文中已经提到，**锁的标志位包含几种情况：00 代表轻量级锁、01 代表无锁（或者偏向锁）、10 代表重量级锁、11 则跟垃圾回收算法的标记有关**。

当线程执行某同步代码时，Java 虚拟机会在当前线程的栈帧中开辟一块空间（Lock Record）作为该锁的记录，如下图所示：

[![轻量级锁01.png](https://z3.ax1x.com/2021/08/03/figvzn.png)](https://imgtu.com/i/figvzn)

然后 Java 虚拟机会尝试使用 CAS（Compare And Swap）操作，将锁对象的 Mark Word 拷贝到这块空间中，并且将锁记录中的 owner 指向 Mark Word。结果如下：

[![轻量级锁02.png](https://z3.ax1x.com/2021/08/03/fi27lR.png)](https://imgtu.com/i/fi27lR)

当线程再次执行此同步代码块时，判断当前对象的 Mark Word 是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁。

**轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。**

#### 偏向锁

轻量级锁是在没有锁竞争情况下的锁状态，但是在有些时候锁不仅存在多线程的竞争，而且总是由同一个线程获得。因此为了让线程获得锁的代价更低引入了偏向锁的概念。**偏向锁的意思是如果一个线程获得了一个偏向锁，如果在接下来的一段时间中没有其他线程来竞争锁，那么持有偏向锁的线程再次进入或者退出同一个同步代码块，不需要再次进行抢占锁和释放锁的操作**。偏向锁可以通过 **-XX:+UseBiasedLocking 开启或者关闭**。

偏向锁的具体实现就是在锁对象的对象头中有个 ThreadId 字段，默认情况下这个字段是空的，当第一次获取锁的时候，就将自身的 ThreadId 写入锁对象的 Mark Word 中的 ThreadId 字段内，将是否偏向锁的状态置为 01。这样下次获取锁的时候，直接检查 ThreadId 是否和自身线程 Id 一致，如果一致，则认为当前线程已经获取了锁，因此不需再次获取锁，略过了轻量级锁和重量级锁的加锁阶段。提高了效率。

***其实偏向锁并不适合所有应用场景, 因为一旦出现锁竞争，偏向锁会被撤销，并膨胀成轻量级锁，而撤销操作（revoke）是比较重的行为，只有当存在较多不会真正竞争的 synchronized 块时，才能体现出明显改善；因此实践中，还是需要考虑具体业务场景，并测试后，再决定是否开启/关闭偏向锁。***

对于锁的几种状态转换的源码分析，可以参考：[源码分析Java虚拟机中锁膨胀的过程](https://mp.weixin.qq.com/s/WWRbfmY2vVy-usVXgaIdAA)

## 总结

本课时主要介绍了 Java 中锁的几种状态，其中偏向锁和轻量级锁都是通过自旋等技术避免真正的加锁，而重量级锁才是获取锁和释放锁，重量级锁通过对象内部的监视器（ObjectMonitor）实现，其本质是依赖于底层操作系统的 Mutex Lock 实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，成本非常高。实际上Java对锁的优化还有”锁消除“，但是”锁消除“是基于Java对象逃逸分析的，如果对此感兴趣可以查阅 [Java 逃逸分析](https://mp.weixin.qq.com/s/Pub_K7PSCNE82F-96y2v6g) 这篇文章。
