# Java并发



## Java锁



### synchronized



`synchronized`是 Java 内置的关键字，它提供了一种独占的加锁方式。`synchronized`的获取和释放锁由JVM实现，用户不需要显示的释放锁，非常方便。

`synchronized`的局限性

- 当线程尝试获取锁的时候，如果获取不到锁会一直阻塞。
- 如果获取锁的线程进入休眠或者阻塞，除非当前线程异常，否则其他线程尝试获取锁必须一直等待。



同步代码块是使用 `monitorenter` 和 `monitorexit` 指令实现的

同步方法（在这看不出来需要看JVM底层实现）依靠的是方法修饰符上的`ACC_SYNCHRONIZED` 实现。

- **同步代码块**：`monitorenter` 指令插入到同步代码块的开始位置，`monitorexit` 指令插入到同步代码块的结束位置，JVM 需要保证每一个 `monitorenter` 都有一个 `monitorexit` 与之相对应。任何对象都有一个 Monitor 与之相关联，当且一个 Monitor 被持有之后，他将处于锁定状态。线程执行到 `monitorenter` 指令时，将会尝试获取对象所对应的 Monitor 所有权，即尝试获取对象的锁。
- **同步方法**：`synchronized` 方法则会被翻译成普通的方法调用和返回指令如：`invokevirtual`、`areturn` 指令，在 JVM 字节码层面并没有任何特别的指令来实现被`synchronized` 修饰的方法，而是在 Class 文件的方法表中将该方法的 `access_flags` 字段中的 `synchronized` 标志位置设置为 1，表示该方法是同步方法，并使用**调用该方法的对象**或**该方法所属的 Class 在 JVM 的内部对象表示 Klass** 作为锁对象。



### volatile 





## Java Lock 接口





## Java 原子操作类



原子操作（Atomic Operation），意为”不可被中断的一个或一系列操作”。

- 处理器使用基于对缓存加锁或总线加锁的方式，来实现多处理器之间的原子操作。
- 在 Java 中，可以通过锁和循环 CAS 的方式来实现原子操作。CAS操作 —— Compare & Set ，或是 Compare & Swap ，现在几乎所有的 CPU 指令都支持 CAS 的原子操作。

原子操作是指一个不受其他操作影响的操作任务单元。原子操作是在多线程环境下避免数据不一致必须的手段。



**CAS 操作有什么缺点？**

**ABA 问题**

比如说一个线程 one 从内存位置 V 中取出 A ，这时候另一个线程 two 也从内存中取出 A ，并且 two 进行了一些操作变成了 B ，然后 two 又将 V 位置的数据变成 A ，这时候线程 one 进行 CAS 操作发现内存中仍然是 A ，然后 one 操作成功。尽管线程 one 的 CAS 操作成功，但可能存在潜藏的问题。

从 Java5 开始 JDK 的 `atomic`包里提供了一个类 AtomicStampedReference 来解决 ABA 问题。

**循环时间长开销大**

对于资源竞争严重（线程冲突严重）的情况，CAS 自旋的概率会比较大，从而浪费更多的 CPU 资源，效率低于 `synchronized` 。

**只能保证一个共享变量的原子操作**

当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。

