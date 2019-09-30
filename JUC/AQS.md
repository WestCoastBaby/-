# 								AQS



##### AQS是做什么的？

​		这个类被设计出来的目的，是为了定义一套实现同步器的框架。不仅仅是定义了接口，因为同步器是有很大的共性的，AQS把这些共性已经做好了，在我们实现的时候直接调用就行了。[知乎](https://zhuanlan.zhihu.com/p/49326706)



##### 同步器的共性有哪些？

​		所有的同步器至少要有信号量state。那么一系列同步操作这个信号量的方法，就是AQS需要提供的功能之一。AQS有以下几大功能：`同步操作state`、`自动阻塞/唤醒等待的线程`、`排队/取消排队`、`条件等待`、`共享/排他`。AQS的作者把他觉得开发者可能会在实现同步器时使用的所有方法/小工具都写出来了，以至于这个类中有57个方法，但是就理解AQS来说，只需要掌握几个核心的方法就够了。在使用AQS时一般都将其作为一个non-public内部工具类，参考ReentrantLock。



##### 🦐AQS如何实现这些功能的？

​		AQS的核心是一个[CLH队列](https://www.iteye.com/blog/googi-1736570)。以FIFO的方式排队。显然原版的CLH模型简单，无法支持很丰富的功能的，性能也有一些缺陷。所以AQS对它做了一些改造：

- 等待锁时使用挂起/唤醒而不是自旋

- 增加一个Thread字段，用来表示当前线程

- 增加了next指针（CLH中只有prev）

- 节点中为了满足某些功能增加了几个控制信息例如标识共享/排他的字段

  


​        当线程申请锁被阻塞时就会创建一个Node，去阻塞队列尾部排队，等待其前继节点释放资源。当占用资源的节点（head节点）释放资源时，会唤醒它的后继节点。



​        同步操作state：同步器的状态，节点能不能申请到锁都是通过操作一个int state实现的。值得一提的是AQS并不关心state具体值的含义，比如state=0，用户在实现自己的同步器的时候可以完全自由的赋予这一状态具体的含义（空闲/被占用）。但是对state的操作一定要保证是同步的，AQS提供了三个具有原子性的方法getState,setState,compareAndSetState。用户操作state字段必须通过这三个方法。其原子性语义的实现原理是利用[UNSAFE类](#UNSAFE类的威力)。CAS系列API完成。

​		阻塞/唤醒等待的线程：申请不到锁的线程需要进入WAITING，释放CPU资源。  依然是使用UNSAFE类。UNSAFE.unpark唤醒，UNSAFE.park阻塞。

​        条件等待：提供和Object中的wait/notify一样的功能。用来挂起和唤醒线程。Java中每个对象都有自己的锁，叫对象锁，同时也有一个条件变量。如果线程不满足条件变量，就处于等待状态不会去申请锁。对象本身的条件变量只有一个，被多个线程共享。因此在某些情况下会出现不友好的问题（简书）。Condition提供了7个接口，见名知意。AQS中的内部类ConditionObject对Condition进行了实现。一般情况直接使用ConditionObject就可以。相比对象中的条件变量，它的优势是可以有多个条件，更加灵活。实现原理是利用[LockSupport](#LockSupport)

​        共享/排他：共享和排他使用的都是同一个FIFO队列，只是Node节点的属性不同。AQS提供了对两种不同节点的唤醒处理机制。①如果对于排他锁被阻塞然后唤醒的情况，只需要唤醒一个Node。



​		AQS功能已经介绍完了，深入理解它还需要通过看代码了解一些细节。可以通过acquire，release，acquireShared，releaseShared四个方法了解到加锁解锁的整个过程。代码难度不大不需要一步步讲解，在这里提一下其中几个不好理解的点和自己看到的一些细节。

1.遇到LockSupport.park(this);方法时应该意识到此时当前线程已经阻塞了，后面的代码在线程被唤醒之前都不会执行，因此要去看一下唤醒的时机。

2.每个头结点释放锁的时候都会唤醒自己的后继节点，但是移除自己的操作是下一个成功变成head的节点完成的。如果共享节点成为了head节点，它会立即尝试唤醒后继节点。这个逻辑下，如果后继节点也是共享节点，就会立即获得锁。

3.除了CLH阻塞队列，还有个条件队列。当线程条件阻塞的时候会进入条件队列，满足条件之后会从条件队列到阻塞队列的尾部排队。

4.Node节点的waitState字段表示等待状态。当值为SIGNAL时表示自己释放时需要唤醒后继节点，这个状态由后继节点在排队时设置。

5.ReentrantReadWriteLock支持了读写两种锁，利用state高16位表示读锁，低16位表示写锁。

6.ReentrantLock  支持锁降级, 锁升级（读锁之后加写锁） 是不支持的。读写互斥，当存在写锁的时候可以保证没有别的任何线程持有任何锁 所以当前线程可以放心的加读锁 （支持写后加读锁、降级）
但是升级的时候，必须保证只有只有只有当前线程持有读锁，这一条件的判定是很费力的  所以干脆就不支持。

7.ReentrantLock对于公平和非公平的实现实际上是不完备的。因为AQS的核心是一个FIFO的队列，基因中就是公平的，想让它非公平代价很大，ReentrantLock的非公平对性能做了让步，导致大多数情况下依然是公平的。

8.Semaphore信号量是非阻塞的，如果没有资源会立即返回false。

9.CyclicBarrier利用ReentrantLock。可以想一下为什么没有直接使用AQS完成，毕竟它就比CountDownLatch多了个循环使用的特性。









##### UNSAFE类的威力

​		它可以直接修改某地址处的值，例如：

​		unsafe.compareAndSwapInt(this, stateOffset, expect, update);

​		如果this对象的偏移量为stateOffset的值等于expect，则修改为update返回true，否则返回false，这个操作是不需要考虑同步的。Java本来是个安全语言，安全语言有很多美好的特性，例如，JIT可以精确的知道任何时刻栈信息甚至寄存器中的内容。这一前提条件就是内存分配是JVM完成而不是程序员。但是UNSAFE类可以破坏这一前提。好处就是可以做到很多Java本身不支持的操作。除了上面的例子之外AQS中还利用UNSAFE改变wait状态下的线程的信息。唤醒和休眠线程。以volatile方式读非volatile值。另外，上述CAS是在Java中的API，硬件层面，现在大多数64位CPU都有一个CAS指令，由于是单条指令所以天然的具有原子性且性能高。



Java中线程的状态在java.lang.Thread.State这个枚举中，一共以下几类：

- NEW 创建Thread对象，未调用start()方法。
- RUNNABLE 虚拟机正在执行的线程，实际上可能并未执行，可能等待一些资源例如操作系统分配CPU。
- BLOCKED 被锁阻塞，尝试获取锁资源。例如被synchronized阻塞。
- WAITING  在调用Object#wait()#join()LockSupport#park() 后，线程处于无限期等待状态。需要满足指定条件比如其他线程对其调用notify，unpack等等。满足条件之后进入BLOCKED状态。
- TIMED_WAITING 和上面一样，多了超时的机制，时间条件达成后自动进入BLOCKED状态。
- TERMINATED 终止。



##### Synchronized

​		可重入锁，可以修饰方法，对象，代码块。在 Java 中，每个对象都会有一个 monitor 对象，这个对象其实就是 Java 对象的锁，通常会被称为“内置锁”或“对象锁”。Synchronized锁的就是对象锁，阻塞也是在monitor队列中排队（这也是为什么修饰代码块时候也要指定一个对象的原因）。不同对象锁之间没有互斥关系。

- 当修饰代码块时，获得的是填入的对象的锁。

- 当修饰静态方法时获得的是方法所在类的Class对象锁。

- 当修饰非静态方法时获得的是实例对象锁。

  由于Synchronized是虚拟机实现的，所以会保证无论如何都会释放锁。从字节码看，使用monitorenter获取锁，monitorexit释放锁。

使用synchronized时需要注意：

- JVM String有字符串常量池，等值字符串背后的对象可能就是同一个。锁的行为会不确定。不建议使用String作为常量池。
- 潜在的生成新对象的对象作为锁。例如Integer自增操作会生成新的对象，锁的行为可能不是期望的那样。



##### LockSupport

​		一个用来阻塞线程的工具，用来替换Object.wait()方法。核心方法就是java.util.concurrent.locks.LockSupport#park()调用后当前线程会进入WAITING状态

java.util.concurrent.locks.LockSupport#unpark(Thread)调用后，将处于WAITING状态的线程唤醒，新的状态是BLOCKED。

​		LockSupport中的方法大多数都重载了，增加了一个Object blocker参数。这个参数的目的是为了方便分析Dump定位问题（Java dump简介：[掘金](https://juejin.im/post/5b31b510e51d4558a426f7e9) ）。synchronized因为每次都会锁对象，dump时跟踪这个对象的状态可以方便定位问题，他们的dump信息差异如下：

- 带blocker（值为new Person()）的park方法：java.lang.Thread.State: WAITING (parking)- parking to wait for  <0x00000000d6662e10> (a com.example.model.Person)
- synchronized(new Person)中的wait()： java.lang.Thread.State: WAITING (on object monitor)- waiting on <0x00000000d635af88> (a com.example.model.Person)
- 不带blocker的park方法：java.lang.Thread.State: WAITING (parking) （少了-waiting on...这部分信息）

​	

##### Condition

​		条件变量，提供和Object中的wait/notify一样的功能。用来挂起和唤醒线程。Java中每个对象都有自己的锁，叫对象锁，同时对象还有一个条件变量。如果线程不满足条件变量，就处于等待状态不会去申请锁。对象本身的条件变量只有一个，被多个线程共享。因此在某些情况下会出现不友好的问题（[简书](https://www.jianshu.com/p/82bdd60ff816)）。因此AQS中条件变量是可以有多个的（new Condition()就行了），每个条件都会有单独的条件等待队列，

​		Condition提供了7个接口，见名知意。AQS中的内部类ConditionObject对Condition进行了实现。一般情况直接使用ConditionObject就可以，不需要别的实现了。





##### 一些文档上的注释

​		核心是一个FIFO的队列，提供一个框架解决大部分的同步和阻塞控制问题。依赖于一个int值表示状态（state字段）。子类必须定义protected方法去改变status，这些方法（可以去看一下AQS中有哪些protected方法）定义了status表示该对象被加锁或被释放锁的具体含义。鉴于此，AQS中的其他方法完成所有的排队和阻塞机制。子类可以维护其他的状态字段，但是更新值必须具备原子性，使用getStatus,setStatus和compareAndSetState操作status的值，因为这三个是具有原子性的。

> 这三个方法都是 protected final的，里面的实现由AQS的作者完成，大佬来保证其原子性语义。

子类应该被定义成non-public内部辅助类，用于实现外部类的同步功能。AQS没有实现任何同步接口，而是定义类像acquireInterruptibly这样的方法，这些方法可以被具体的锁和同步器适当的调用，以实现一些接口中的方法。

> 例如java.util.concurrent.locks.Lock 接口的一个实现 java.util.concurrent.locks.ReentrantLock，ReentrantLock实现Lock接口中的lockInterruptibly方法，就用到了acquireInterruptibly。这里可以逐渐感受到AQS的功能了，我们实现同步手段，都是直接调用AQS中的方法就行了，不需要也不能（AQS中大多数方法都是final的）自己去做同步操作。

AQS支持共享和排他两种模式。在排他模式下，其他线程继续申请排他锁就会失败。共享模式可以被多个线程申请，但是也不一定成功。AQS并不理解共享与排他两种模式的差异，它只会机械的运行：当一个共享模式申请成功后，评估下一个等待线程（如果有）是否可以申请。不同模式下的线程使用同一个FIFO队列等待。通常子类只会实现一种模式，但是也能同时实现两种模式例如ReadWriteLock。

> 上面关于共享锁多次申请不一定成功，有一些细节性的判断例如锁有没有达到$$2^{16}$$ 个等。按照常规的共享和排他语义理解就可以了。



跳过两段实在翻译不出来不知道这鬼佬英文为啥能一句写这么长！

序列化此类，只存储底层的原子int值。所以反序列化对象的线程队列是空的。典型的，子类声明可序列化需要定义readObject方法。



使用

​		AQS是用于定义同步器的基础，定义以下方法，使用getState，setState ，compareAndSetState检查或修改同步状态。

{@link #tryAcquire} {@link #tryRelease} {@link #tryAcquireShared}  {@link #tryReleaseShared}

 {@link #isHeldExclusively}

​		这些方法的默认实现是抛出UnsupportedOperationException异常。实现这些类必须保证线程安全，应该非阻塞耗时短。定义这些方法是唯一使用AQS的手段。其他所有方法都被声明为final。

​		AQS继承了AbstractOwnableSynchronizer，目的是为了使用AOS中的3个小方法。它们是跟踪哪个线程目前持有锁的小工具。

​		虽然AQS内部基于一个FIFO队列，它并不能自动的执行FIFO策略。排他锁大体会采用以下形式（共享锁类似稍有区别）：

```java
//Acquire:   
while (!tryAcquire(arg)) {
	//如果没有在队列中，进入队列
    //可能要阻塞当前线程
}
//Release:
if (tryRelease(arg)){
    //解锁队列中第一个节点
}
```

​		





Node(等待队列节点)：

​		等待队列是CLH锁队列的一个变种。CLH通常使用自旋，这里使用阻塞同步。但是使用的控制同步的手段都是一样的，一个线程维护了其前一个节点的控制信息。Node中的status字段，表示一个线程是否应该被阻塞。线程会在其前一个节点释放锁的时候收到一个通知信号。Each node of the queue otherwise serves as a specific-notification-style monitor holding a single waiting thread. status字段并不控制线程获得锁等。线程会尝试自己处于队列的第一个（意味着持有锁），但是并不一定会成功。

​		一个节点进入CLH队列，会自动的拼接到队尾。插入CLH队列只需要在尾部进行一个原子操作，所以，有一个简单的原子指针划分入队和未入队。类似的，出队只需要更新‘head’。但是出队需要更多步骤来确认谁成功了，比如解决超时或者中断情况时节点取消排队。

​		"prev"链（原始CLH中没这个），主要用来处理取消排队。如果一个节点被取消了，它的接班人就会被重新链接到前面一个没有取消的节点。为了在自旋中解释类似的机制，看下面这篇文章：***

​		“next”链用来实现阻塞机制，每个节点都保存了它当前线程的id，所以前一个节点唤醒下一个节点就是通过这个链。判定下一个节点，必须避免与新来的节点竞争。

