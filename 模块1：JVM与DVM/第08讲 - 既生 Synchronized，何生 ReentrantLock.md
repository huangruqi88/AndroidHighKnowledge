## synchronized

synchronized 可以用来修饰以下 3 个层面
1. **修饰实例方法；**
2. **修饰静态类方法；**
3. **修饰代码块。**

#### synchronized 修饰实例方法

[![synchronized 修饰实例方法1.png](https://z3.ax1x.com/2021/08/03/fPpYVS.png)](https://imgtu.com/i/fPpYVS)

这种情况下的锁对象是当前实例对象，因此只有同一个实例对象调用此方法才会产生互斥效果，不同实例对象之间不会有互斥效果。比如如下代码：

[![synchronized 修饰实例方法2.png](https://z3.ax1x.com/2021/08/03/fPpf2R.png)](https://imgtu.com/i/fPpf2R)

上述代码，在不同的线程中调用的是不同对象的 printLog 方法，因此彼此之间不会有排斥。运行效果如下：

[![synchronized 修饰实例方法3.png](https://z3.ax1x.com/2021/08/03/fPpjxI.png)](https://imgtu.com/i/fPpjxI)

可以看出，两个线程是交互执行的。
如果将代码进行如下修改，两个线程调用同一个对象的 printLog 方法：

[![synchronized 修饰实例方法4.png](https://z3.ax1x.com/2021/08/03/fP9kGj.png)](https://imgtu.com/i/fP9kGj)

则执行效果如下：

[![synchronized 修饰实例方法5.png](https://z3.ax1x.com/2021/08/03/fP93W9.png)](https://imgtu.com/i/fP93W9)

#### 修饰静态类方法

如果 synchronized 修饰的是静态方法，则锁对象是当前类的 Class 对象。因此即使在不同线程中调用不同实例对象，也会有互斥效果。

将 LagouSynchronizedMehtods 中的 printLog 修改为静态方法，如下：

[![修饰静态类方法01.png](https://z3.ax1x.com/2021/08/03/fP97yq.png)](https://imgtu.com/i/fP97yq)

执行后的打印效果如下：

[![修饰静态类方法02.png](https://z3.ax1x.com/2021/08/03/fPCVte.png)](https://imgtu.com/i/fPCVte)

可以看出，两个线程还是依次执行的。

#### synchronized 修饰代码块

除了直接修饰方法之外，synchronized 还可以作用于代码块，如下代码所示：

[![synchronized 修饰代码块01.png](https://z3.ax1x.com/2021/08/03/fPCIAO.png)](https://imgtu.com/i/fPCIAO)

synchronized 作用于代码块时，锁对象就是跟在后面括号中的对象。上图中可以看出任何 Object 对象都可以当作锁对象。

#### 实现细节

synchronized 既可以作用于方法，也可以作用于某一代码块。但在实现上是有区别的。 比如如下代码，使用 synchronized 作用于代码块：

[![实现细节01.png](https://z3.ax1x.com/2021/08/03/fPP0xA.png)](https://imgtu.com/i/fPP0xA)

使用 javap 查看上述 test1 方法的字节码，可以看出，编译而成的字节码中会包含 monitorenter 和 monitorexit 这两个字节码指令。如下所示：

[![实现细节02.png](https://z3.ax1x.com/2021/08/03/fPPXRJ.png)](https://imgtu.com/i/fPPXRJ)

你可能已经发现了，上面字节码中有 1 个 monitorenter 和 2 个 monitorexit。这是因为虚拟机需要保证当异常发生时也能释放锁。因此 2 个 monitorexit 一个是代码正常执行结束后释放锁，一个是在代码执行异常时释放锁。

再看下 synchronized 修饰方法有哪些区别：

[![实现细节03.png](https://z3.ax1x.com/2021/08/03/fPitLq.png)](https://imgtu.com/i/fPitLq)

从图中可以看出，被 synchronized 修饰的方法在被编译为字节码后，在方法的 flags 属性中会被标记为 ACC_SYNCHRONIZED 标志。当虚拟机访问一个被标记为 ACC_SYNCHRONIZED 的方法时，会自动在方法的开始和结束（或异常）位置添加 monitorenter 和 monitorexit 指令。

关于 monitorenter 和 monitorexit，可以理解为一把具体的锁。在这个锁中保存着两个比较重要的属性：计数器和指针。

+ 计数器代表当前线程一共访问了几次这把锁；
+ 指针指向持有这把锁的线程。

用一张图表示如下：

[![实现细节04.png](https://z3.ax1x.com/2021/08/03/fPFt9e.png)](https://imgtu.com/i/fPFt9e)

## ReentrantLock

####ReentrantLock 基本使用

ReentrantLock 的使用同 synchronized 有点不同，它的加锁和解锁操作都需要手动完成，如下所示：

[![ReentrantLock01.png](https://z3.ax1x.com/2021/08/03/fPkbIf.png)](https://imgtu.com/i/fPkbIf)

图中 lock() 和 unlock() 分别是加锁和解锁操作。运行效果如下：

[![ReentrantLock02.png](https://z3.ax1x.com/2021/08/03/fPA1JO.png)](https://imgtu.com/i/fPA1JO)

可以看出，使用 ReentrantLock 也能实现同 synchronized 相同的效果。

你可能已经注意到，在上面 ReentrantLock 的使用中，我将 unlock 操作放在 finally 代码块中。这是因为 ReentrantLock 与 synchronized 不同，当异常发生时 synchronized 会自动释放锁，但是 ReentrantLock 并不会自动释放锁。因此好的方式是将 unlock 操作放在 finally 代码块中，保证任何时候锁都能够被正常释放掉。 

####公平锁实现

ReentrantLock 有一个带参数的构造器，如下：

[![公平锁实现01.png](https://z3.ax1x.com/2021/08/03/fPA4YT.png)](https://imgtu.com/i/fPA4YT)

默认情况下，synchronized 和 ReentrantLock 都是非公平锁。但是 ReentrantLock 可以通过传入 true 来创建一个公平锁。所谓公平锁就是通过同步队列来实现多个线程按照申请锁的顺序获取锁。

公平锁使用实例如下：

[![公平锁实现02.png](https://z3.ax1x.com/2021/08/03/fPEF0I.png)](https://imgtu.com/i/fPEF0I)

运行效果如下：

[![公平锁实现01.png](https://z3.ax1x.com/2021/08/03/fPEt9U.png)](https://imgtu.com/i/fPEt9U)

可以看出，创建的 3 个线程依次按照顺序去修改 sharedNumber 的值。

网上对公平锁有一段例子很经典：假设有一口水井，有管理员看守，管理员有一把锁，只有拿到锁的人才能够打水，打完水要把锁还给管理员。每个过来打水的人都要得到管理员的允许并拿到锁之后才能去打水，如果前面有人正在打水，那么这个想要打水的人就必须排队。管理员会查看下一个要去打水的人是不是队伍里排最前面的人，如果是的话，才会给你锁让你去打水；如果你不是排第一的人，就必须去队尾排队，这就是公平锁。

####读写锁（ReentrantReadWriteLock）

在常见的开发中，我们经常会定义一个线程间共享的用作缓存的数据结构，比如一个较大的 Map。缓存中保存了全部的城市 Id 和城市 name 对应关系。这个大 Map 绝大部分时间提供读服务（根据城市 Id 查询城市名称等）。而写操作占有的时间很少，通常是在服务启动时初始化，然后可以每隔一定时间再刷新缓存的数据。但是写操作开始到结束之间，不能再有其他读操作进来，并且写操作完成之后的更新数据需要对后续的读操作可见。

在没有读写锁支持的时候，如果想要完成上述工作就需要使用 Java 的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并进行通知之后，所有等待的读操作才能继续执行。这样做的目的是使读操作能读取到正确的数据，不会出现脏读。

但是如果使用 concurrent 包中的读写锁（ReentrantReadWriteLock）实现上述功能，就只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续的读写锁都会被阻塞，写锁释放之后，所有操作继续执行，这种编程方式相对于使用等待通知机制的实现方式而言，变得简单明了。

接下来，我们看下读写锁（ReentrantReadWriteLock）如何使用：

+ 创建读写锁对象：

[![读写锁01.png](https://z3.ax1x.com/2021/08/03/fPE4HI.png)](https://imgtu.com/i/fPE4HI)

+ 通过读写锁对象分别获取读锁（ReadLock）和写锁（WriteLock）：

[![读写锁02.png](https://z3.ax1x.com/2021/08/03/fPVp5V.png)](https://imgtu.com/i/fPVp5V)

+ 使用读锁（ReadLock）同步缓存的读操作，使用写锁（WriteLock）同步缓存的写操作：

[![读写锁03.png](https://z3.ax1x.com/2021/08/03/fPVeV1.png)](https://imgtu.com/i/fPVeV1)

具体实现，参考如下代码片段：

[![读写锁04.png](https://z3.ax1x.com/2021/08/03/fPV8rd.png)](https://imgtu.com/i/fPV8rd)

解释说明：

+ 图中的 number 是线程中共享的数据，用来模拟缓存数据；

+ 图中①处，分别创建 2 个 Reader 线程并从缓存中读取数据，和 1 个 Writer 将数据写入缓存中；

+ 图中②处，使用读锁（ReadLock）将读取数据的操作加锁；

+ 图中③处，使用写锁（WriteLock）将写入数据到缓存中的操作加锁。

[![读写锁05.png](https://z3.ax1x.com/2021/08/03/fPMO74.png)](https://imgtu.com/i/fPMO74)

仔细查看日志，可以看出当写入操作在执行时，读取数据的操作会被阻塞。当写入操作执行成功后，读取数据的操作继续执行，并且读取的数据也是最新写入后的实时数据。

## 总结

这课时我们主要学习了 Java 中两个实现同步的方式 synchronized 和 ReentrantLock。其中 synchronized 使用更简单，加锁和释放锁都是由虚拟机自动完成，而 ReentrantLock 需要开发者手动去完成。但是很显然 ReentrantLock 的使用场景更多，公平锁还有读写锁都可以在复杂场景中发挥重要作用。















