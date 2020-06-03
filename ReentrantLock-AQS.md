### 实现AQS以创造一个锁
1. 需要实现 #tryAcquire、 #tryRelease、#tryAcquireShared、#tryReleaseShared、#isHeldExclusively，这四个方法
2. 在ReentrantLock中，使用Sync抽象内部类进行实现，NonfairSync和FairSync继承Sync完成非公平锁与公平锁的实现
3. AQS中构造了Node head和Node tail，用于保存等待的线程，其中head表示获得锁的线程。

### 加锁过程总结：
1. 如果是第一个线程tf，那么和队列无关，线程直接持有锁。并且也不会初始化队列，如果接下来的线程都是交替执行，那么永远和AQS队列无关，都是直接线程持有锁，
2. 如果发生了竞争，比如tf持有锁的过程中T2来lock，那么这个时候就会初始化AQS，初始化AQS的时候会在队列的头部虚拟一个Thread为NULL的Node，因为队列当中的head永远是持有锁的那个node（除了第一次会虚拟一个，其他时候都是持有锁的那个线程锁封装的node），
3. 现在第一次的时候持有锁的是tf而tf不在队列当中所以虚拟了一个node节点，队列当中的除了head之外的所有的node都在park，
4. 当tf释放锁之后unpark某个（基本是队列当中的第二个，为什么是第二个呢？前面说过head永远是持有锁的那个node，当有时候也不会是第二个，比如第二个被cancel之后，至于为什么会被cancel，不在我们讨论范围之内，cancel的条件很苛刻，基本不会发生）node之后，node被唤醒，假设node是t2，那么这个时候会首先把t2变成head（sethead），
5. 在sethead方法里面会把t2代表的node设置为head，并且把node的Thread设置为null，为什么需要设置null？其实原因很简单，现在t2已经拿到锁了，node就不要排队了，那么node对Thread的引用就没有意义了。所以队列的head里面的Thread永远为null
6. [JUC AQS ReentrantLock源码分析（一）](https://blog.csdn.net/java_lyvee/article/details/98966684)

### 解锁过程
1. 调用unlock()，会调用到release()，进而调用到tryRelease(arg)
2. tryRelease()由Sync进行实现，其内部对state进行减一，完成一次释放锁，如果锁还有其它层未释放，返回false，然后不做其它操作
3. 如果锁已经完全释放，即state=0，则将队列中的head节点的waitStatusCAS地设置为0，然后将第二个节点设置为head节点，并将其代表的Thread，unpark唤醒。该线程获得锁
4. 如果第二个节点被取消，则从tail向前获得一个为被取消的node进行Thread唤醒。