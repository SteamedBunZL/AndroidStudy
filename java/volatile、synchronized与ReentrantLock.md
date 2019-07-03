## volatile、synchronized与ReentrantLock





### synchronized

#### synchronized实现原理

synchronized 锁特性由 JVM 负责实现。在 JDK 的不断优化迭代中， synchronized 锁的性能得到极大提升，特别是偏向锁的实现，使得 synchronized 已经不是昔日那个 低性能且笨重的锁了。 JVM底层是通过监视锁来实现 synchronized 同步的。**监视锁即 monitor， 是每个对象与生俱来的一个隐藏字段。**使用 synchronized 时， JVM会根 据 synchronized 的当前使用环境，找到对应对象的 monitor，再根据 monitor 的状态进 行加、解锁的判断。例如，线程在进入同步方法或代码块时，会获取该方法或代码块 所属对象的 monitor， 进行加锁判断。如果成功加锁就成为该 monitor 的唯一持有者。monitor 在被释放前，不能再被其他线程获取。

```
public void testSynchronized();
	descriptor:()V
	flags: ACC_PUBLIC ACC_SYNCHRONIZED
	Code:
		stack=2,locals=2,args_size=1;
		0:getstatic
		...
		monitorenter
		...
		monitorexit
		return
```

方法元信息中会使用 ACC_SYNCHRONIZED 标识该方法是一个同步方法。同步代码块中会使用 monitorenter及 monitorexit两个字节码指令获取和释放 monitor。 如果使用 monitorenter进入时 monitor 为 0，表示该线程可以持有 monitor 后续代码， 并将 monitor 加 1，如果当前线程已经持有了 monitor， 那么 monitor 继续加 1 ;如果 monitor 非 0， 其他线程就会进入阻塞状态。 



#### synchronized优化

JVM对synchronized的优化主要在于对monitor的加锁、解锁上。

自旋锁与自适应自旋

锁消除

锁粗化

偏向锁

轻量级锁

重量级锁





