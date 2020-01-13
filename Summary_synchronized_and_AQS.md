# Concurrent

## Synchronized

### 标志位  

偏向锁锁101 无锁001 轻量级00 重量级10  
注意OOP对象头中两个指针_mark和_meta，其中_mark是无法压缩的，64位下就为64bit。JVM中8B对齐(栈上的对象是否也是这样？如LockRecord)，也就是最后3bit都是0，因此_mark代表ObjectMonitor或LockRecord时最后3bit都是0，和是否启用压缩指针无关，因此可以放心的设置。  

### 偏向锁

1. 加锁  
和轻量级共享逻辑，先在当前栈帧中拿到一个LockRecord，将其object设为lockObject，判断低3位是否101(偏向模式)。如果MarkWord的中的线程是当前线程并且epoch相等就什么都不做。如果epoch过期通过CAS重偏向。构造匿名偏向MarkWord，以它作为期待值进行CAS修改MarkWord，也就是如果MarkWord是匿名偏向就能修改成功，从而获得锁，否则执行MonitorEnter进行偏向锁的撤销和升级。如果当前锁已偏向其他线程||epoch值过期||偏向模式关闭||获取偏向锁的过程中存在并发冲突，都会进入到InterpreterRuntime::monitorenter方法， 在该方法中会对偏向锁撤销和升级。  
2. 撤销  
InterpreterRuntime::monitorenter中如果使用偏向锁，则进入fastEnter先进行撤销或重偏向，再进行slowEnter进行升级。一般的，如果要撤销的锁是当前线程则直接调用revoke_bias方法撤销，否则压入到VM_Thread等待所有Java线程进入safepoint时由VM_Thread执行(这个压入似乎是阻塞的)。~~revoke_bias方法中，首先判断偏向线程是否存活，如果不存活，根据是否允许重偏向将MarkWord设置为匿名偏向或无锁状态；如果存活，遍历线程栈的LockRecord，如果找到对应的说明还在同步块中，将对应的LockRecord修改为轻量级的模式，如果不在同步块中根据是否允许重偏向设置为匿名偏向或无锁状态。~~ 撤销了之后会在slowenter中再次锁升级(获取)。一般的，如果是A已经获得偏向锁，B尝试获得锁的情况下，是不允许重偏向的，所以若A已经死亡，或者出了同步块会撤销到无锁状态(001不再允许重偏向，并且这个时候A中已经没有关于这个Object的LockRecord了)，之后B进行轻量级锁的获取；如果A还存活且还在同步块中，就会将A中LockRecord修改为轻量级状态，之后B进行重量级锁的获取。  
3. 解锁  
从低到高在栈中找到第一个关联锁对象的LockRecord，将LockRecord的objectReference置为NULL。如果是偏向锁就到这里了，如果是轻量级且没有重入还会CAS替换MarkWord。

### 轻量级锁

1. 加锁  
当偏向锁获取失败后，变为轻量级锁。首先在当前栈帧中拿到一个LockRecord(ObjectReference为NULL具体怎么分配的暂不明确)，将LockRecord的objectReference置为Object(OOP)，按照Object的MarkWord构建一个无锁状态的副本(将其末尾置01，DisplacedMarkWord)，并将其保存在LockRecord中。对Object的MarkWord字段进行CAS，修改为LockRecord的地址，如果成功说明拿到了轻量级锁，失败了再检查当前线程是否已经拿过这个锁，拿过了就将本次LockRecord的MarkWord副本置位NULL代表重入，否则说明需要重量级锁。注意：由于LockRecord复制本地副本的时候将末尾置01了(lockee->mark()->set_unlocked()不确定，这各个unlock是否指的是将末尾置01)，因此如果Object不是无锁状态01下是一定无法CAS成功的。  
2. 解锁  
首先，LockRecord的objectReference会置成NULL表明释放了这个LockRecord(无论偏向还是轻量级)，接着如果MarkWord副本(DisplacedMarkWord)不为NULL(说明不是重入)，需要将对应Object的MarkWord通过CAS恢复(成功恢复之后应该又是无锁状态了)，失败了则要进入重量级流程。

### 重量级锁

1. 膨胀  
轻量级锁在多线程竞争下膨胀为重量级锁，线程在膨胀过程中会分配一个ObjectMonitor对象并初始化，设置monitor的相关字段，并将锁对象OOP的markword置为重量级状态10 ~~(?似乎Monitor对象本身的地址就能保证最后两位为10)~~ ，指向刚分配的monitor对象。  
2. 抢锁enter  
Monitor对象主要还包括cxq(ContentionList)，EntryList ，WaitSet，OnDeck以及owner。先自旋尝试获得锁，如果锁被占用，则将线程包装为ObjectWaiter对象通过CAS插入到ContentionList的队首，接着是个无限大for，其中先尝试拿到锁break，否则调用park将当前线程挂起。当抢到锁之后将包装的node从队列中移除。  
3. 解锁exit  
设置owner为null，这个操作等于释放了锁，其他线程可以抢锁。如果当前没有等待的线程或者已经有假定继承人了，那当前线程不需要唤醒其他线程直接返回。然后当前线程通过CAS尝试获取锁，如果不能获取到说明已经有其他线程拿到锁不需要唤醒了，直接返回。如果拿到锁了，则根据QMode执行不同的唤醒策略，默认如果EntryList非空，则唤醒其头结点，立即返回；如果EntryList为空，将cxq的所有元素按原序放入EntryList中，再唤醒其头结点。由于cxq入队列是入队头，唤醒时唤醒队头，因此多线程下会有竞争，所以设置了EntryList一下把cxq全部拉进EntryList减少竞争。  
4. 等待wait  
如果一个线程在同步块中调用了Object.wait方法，会将该线程对应的ObjectWaiter从EntryList移除并加入到WaitSet中，然后释放锁。当wait的线程被notify之后，会将对应的ObjectWaiter从WaitSet移动到EntryList中。  

## AQS

AQS整体类似一个双向队列结构，有一个head一个tail以及int类型的state。独占模式下state为0表示锁无人占用，大于0表示被重入了几次；共享模式下state能表示准入数或加锁次数等。内部静态类Node表示链表元素，内部有Thread字段一个Thread对应一个Node，waitStatus表示该Thread的状态，默认0，-1为signal表示当前Node的后继Node需要被当前Node唤醒，-2为condition，1为cancel。  

1. 同步acquire  
acquire()方法中主要是tryAcquire()和acquiredQueued()两个方法。tryAcquire()一般需要子类重载，用来尝试一下获取独占锁。ReentrantLock中公平模式下会先拿到AQS的state如果是0说明没有线程占着锁或者刚被释放还没有唤醒下一个Node，如果当前线程之前没有其他线程在等待并且CAS这个state的值成功了，将AQS的持有线程设为自己，就可以返回了，如果state不为0，但是持有线程是自己，说明重入了，给state加上本次想要占据的值然后返回；如果是非公平模式， ~~在tryAcquire()前就会先尝试CAS这个state的值，~~(似乎在JDK12中已经删去，lock方法统一为sync.acquire(1)) ，在tryAcquire()中state为0时，公平会用hasQueuedPredecessors()先判断当前线程之前没有其他线程在等待才能CAS，非公平下直接CAS。  
tryAcquire()失败说明得进同步队列阻塞，首先将自己包装成Node，并无限CAS添加到同步队列尾部。然后执行acquiredQueued()，线程park和park之后都在这个函数中。整个函数是个无限for，首先拿到自己的前驱节点，若前驱节点是head说明自己有可能拿到锁，尝试tryAcquire()，如果拿到锁就返回，没拿到锁或者前驱不是head说明自己得阻塞，同步队列中阻塞的Node是通过前驱节点唤醒的，因此前驱节点的waitstaus必须是-1，通过CAS将前驱节点置为-1，如果大于0说明前驱节点取消了阻塞，则会一直往前找到waitstatus<=0的节点作为前驱。前驱节点设置完了就park自己。  
2. 解锁release  
release时将state减去对应想release的值，如果减完为0说明真的释放了锁，找到head后第一个waitstatus<=0的节点，unpark对应的线程。  
3. 条件wait  
AQS中有内部类ConditionObject也是链表结构，一个ConditionObject就是一个条件队列。当调用condition.await()方法时，线程会将自己包装成Node放入对应的条件队列中，并且完全释放锁，然后park自己，直到自己被其他线程从条件队列移动到同步队列并被唤醒，之后执行前边获取锁时候说的acquiredQueued()，再次park自己，等待被前驱节点唤醒。notify()时，线程会将对应条件队列的第一个Node取出，放入同步队列尾部，同时唤醒它，如果Node取消等待了，就顺延唤醒下一个。  

## Question

* synchronized与ReentrantLock的区别？  

  1. synchronized是JVM层次的实现，ReentranLock是JDK层次的实现。底层阻塞线程使用的是同一种park方式。  
  2. synchronized无法在代码中判断锁状态，ReentrantLock可以通过ReentrantLock.isLocked判断。  
  3. synchronized是非公平的，并且已经在同步队列中的线程是后来的线程先获得锁，而ReentrantLock既有公平模式也有非公平模式，已经在同步队列中的线程一定是先来的线程先获得锁。  
  4. synchronized是不可以被中断的，而ReentrantLock.lockInterruptibly方法是可以被中断的。  
  5. 在发生异常时synchronized会自动释放锁（由javac编译时自动实现），而ReentrantLock需要开发者在finally块中显示释放。  
  6. ReentrantLock更灵活，有各种tryLock，带超时的lock，以及一个lock对应多个condition；synchronize使用起来更方便，任何object都可以搭配synchronize。  

* 乐观锁和悲观锁的区别？  

* 如何实现一个乐观锁？  

* AQS是如何唤醒下一个线程的？  
  AQS中一个线程对应一个Node，线程是由前驱节点唤醒的。节点在park自己时会给自己找一个有效的前驱节点，将其waitstatus置为-1。当线程释放锁时，会从head开始找到第一个waitstatus<=0唤醒它，被唤醒的线程会尝试获取锁。  

* ReentrantLock如何实现公平和非公平锁是如何实现？  
  acquire()方法中主要是tryAcquire()和acquiredQueued()两个方法。tryAcquire()一般需要子类重载，用来尝试一下获取独占锁。ReentrantLock中公平模式下会先拿到AQS的state如果是0说明没有线程占着锁或者刚被释放还没有唤醒下一个Node，如果当前线程之前没有其他线程在等待并且CAS这个state的值成功了，将AQS的持有线程设为自己，就可以返回了，如果state不为0，但是持有线程是自己，说明重入了，给state加上本次想要占据的值然后返回；如果是非公平模式， ~~在tryAcquire()前就会先尝试CAS这个state的值，~~(似乎在JDK12中已经删去，lock方法统一为sync.acquire(1)) ，在tryAcquire()中state为0时，公平会用hasQueuedPredecessors()先判断当前线程之前没有其他线程在等待才能CAS，非公平下直接CAS。  

* CountDownLatch和CyclicBarrier的区别？各自适用于什么场景？

  1. CountDownLatch有内部类Sync继承自AQS。state表示初始设置的N，所有调用await的线程都park在AQS的同步队列中，如果state已经为0那么就直接返回。调用countdown的线程将state-1(如果state已经为0，就直接返回false)若减完正好为0，就用doReleaseShared()方法唤醒第一个阻塞的Node，被唤醒的Node将自己置为队头，然后也调用doReleaseShared()。执行唤醒操作的线程最后会判断head是否有变化，如果被唤醒的线程还没有来得及将自己置为head，那么执行唤醒操作的线程就结束循环，否则循环唤醒下一个。  
  2. CyclicBarrier依托于一把ReentrantLock和一个Condition。count字段表明现在还有多少个线程没有就位。传入的Runnable由最后一个就位的线程执行。每次await时先拿到Lock，给count-1。如果减完为0，就执行Runnable，同时唤醒所有park在condition上的线程；如果减完大于0，就await在condition上。什么时候栅栏会被打破，总结如下：中断，如果某个等待的线程发生了中断，那么会打破栅栏，同时抛出 InterruptedException 异常； 超时，打破栅栏，同时抛出 TimeoutException 异常； 指定执行的操作抛出了异常。  
  3. CountDownLatch适合一次性的，多个线程同步阻塞，由其他线程决定这些线程什么时候执行。CyclicBarrier适合多次周期性的，多个线程需要同步开始。  

* 使用ThreadLocal时要注意什么？比如说内存泄漏?  

* 说一说往线程池里提交一个任务会发生什么？  

  1. 线程池中的工作线程叫Worker，实现了Runnable接口，会不断从workQueue中取任务执行。  
  2. 如果是返回一个Future的submit方法先将Runable包装成一个FutureTask，再execute。execute一个Runnable的时候，如果当前线程数小于核心线程数，会直接以coreSize为界通过addWorker方法，通过CAS给Worker数加1，新创建一个Worker，并在构造方法中将提交的这个Runnable作为这个Worker的第一个task，启动成功就直接返回。  
  3. 如果当前线程数大于等于核心线程数，或者addWorker失败(并发创建导致超过核心线程数)，如果线程池处于Running状态，将提交的Runnable添加到workQueue中。如果队列满了，添加失败，则尝试以maxSize为界限，通过addWorker启动新的线程，如果启动失败执行拒绝策略。  

``` java
// execute一个Runnable时
if (workerCountOf(c) < corePoolSize) { // 步骤2
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
if (isRunning(c) && workQueue.offer(command)) { // 步骤3.1
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false)) // 步骤3.2
    reject(command);
```

* 线程池什么时候执行拒绝策略？  

  1. workers数量大于corePoolSize，将任务添加到workQueue，但是队列已经满了，添加失败；同时线程数已经达到maxSize，无法新建Worker，则会执行拒绝策略。  
  2. 还有一个少见的情况。如果当前线程数大于等于核心线程数，线程池正在运行需要将任务加入Queue中。此时若成功将任务加入workQueue中后，会在检查一下线程池的状态，如果这个时候恰好不为Running状态，会将刚刚添加的任务删去，并执行拒绝操作。  

* 关闭线程池会发生什么？

  1. shutdown时会拿到整个线程池的锁，状态由Running变为ShutDown，此时不接受新任务的提交，但是会继续workQueue中的任务。  
  2. shutdownnow时，状态变为STOP，不仅不接受新任务，也不处理workQueue，**尝试中断正在执行的任务的Worker**。  
  3. TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()
  4. TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个

* 线程池的几个参数如何设置，尤其是keepalive？
* 线程池的非核心线程什么时候会被释放？  
  在getTask中，如果当前线程数大于maxSize(通过setMaxSize突然调小)或者当前线程数大于核心数并且从workQueue中阻塞拉取任务(最多keepalive的时长)没拿到task，会返回NULL。在Worker的run方法中，如果拿到task是NULL会结束循环，从而结束run方法。  

* 如何排查死锁？  
