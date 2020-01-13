# JMM-Volatile

## JMM与硬件架构

* 多缓存与JMM  
    为了匹配CPU运算与内存读写速度，现代计算机结构都会设计多级缓存，例如L1和L2缓存由各个CPU核心独立拥有，L3由各个CPU共享。CPU一般不直接访问memory而是访问cache。  
    JMM是JVM定义的抽象内存模型，规定每个线程都拥有自己的工作内存，工作内存保存主内存的变量副本，线程对变量的修改只能在自己的工作内存中进行，不能直接访问主内存，并且线程拥有的工作内存不能被其他线程访问，共有lock, unlock, read, load, use, assign, store, write 8种操作。线程A与线程B通信需要A从主存中将共享变量读入工作内存，并访问工作内存修改变量值，再回写至Memory；线程B从Memory中读入共享变量从而完成通信。  
    JMM的设计围绕者并发中的原子性，有序性，可见性。  
    其中可见性主要由: 多缓存导致，由于实际硬件的多缓存或者抽象在JMM中的独占工作内存存在，使得多个CPU或线程的缓存中可能存在数据不一致问题。  
    其中有序性问题主要由于:  
    1. 编译器对程序的重排序优化
    2. 处理器CPU对微指令的重排序优化
    3. 内存系统上多缓存的存在，使得读取和写入不一致，从而整体表现为乱序，因此也有说法将可见性包括在有序性中。  

    在CPU的Cache设计中一般使用类MESI协议保证了缓存一致性。  
    MESI中规定Cache中的line可能存在4种状态M(已修改Modified，该Cache Line有效，数据已经被修改了和内存不一致，数据只存在在该Cache中)，E(排他Exclusive，该Cache Line有效，数据和内存一致，数据只存在在该Cache中)，S(共享Shared，该Cache Line有效和内存一致，数据存在于多个Cache中)，I(无效Invalid)。  
    规定仅当本Cache Line处于E或M状态时才能写入数据。举例来说假如x=0被Cache0, Cache1, Cache2共享，当CPU0想向Cache0中写x变量时，需要将Cache0的对应Line变为E状态，同时等待Cache1, Cache2将对应Line变为I状态，才能写入，写完后Cache0变为M状态；此时若Cache1想要加载x的值，必须等待Cache0将值刷入Memory中，并将状态变为S状态，才能从Memory读x并将自己变为S状态。  
    总之MESI本身就应该保证了Cache一致性。所有Cache都会在总线上嗅探，当一个Cache对数据做出了修改，其他Cache都能感知到，将自己的对应Line无效化。但是大量的同步等待工作会降低CPU的性能，因此引入了StoreBuffer和InvalidQueue。CPU在写入值时不会直接写入Cache(因为需要和其他Cache同步)，而是写入StoreBuffer，立即返回。同时通知其他Cache，其他Cache将对应需要无效的操作放入InvalidQueue中，立即返回ACk。在合适的时候StoreBuffer的内容写入Cache，刷新InvalidQueue无效某些Line。当引入了StoreBuffer和InvalidQueue将同步变为了异步，使得一致性有可能不能得到保证，看起来就有了可见性问题。  

* Memory Barrier  
    为了保证一定的内存顺序，可以施加Memory Barrier，从Load和Store来看最基本的Barrier可以有LoadLoad，LoadStore，StoreStore以及StoreLoad4种，其中StoreLoad最强具有其他所有的功能，LoadStore一般都是CPU直接保证的。  
    1. 从可见性上可定义LoadBarrier和StoreBarrier。  
        其中LoadBarrier可以保证刷新InvalidQueue，使得每次读取对应变量都会从内存中拿到最新值；  
        StoreBarrier能保证刷新StoreBuffer，使得对Cache的修改被刷新到Memory中。一般由StoreLoadBarrier来同时实现这两个功能。  
    2. 从有序性上可定义Acquire和Release两个Barrier。
        其中Acquire相当于LoadLoad屏障与LoadStore屏障，在读操作后插入，禁止该读操作与其后的任何读写操作发生重排序；  
        Release相当于LoadStore屏障与StoreStore屏障的组合。在一个写操作之前插入，禁止该写操作与其前面的任何读写操作发生重排序。  
        这个定义和JDK中的内存序acquire/release定义是一致的，比如VarHandle中的acquireFense()方法和releaseFense()方法。  

## Volatile

* Volatile写  
    Volatile写相当于：Release Barrier -> 变量写 -> StoreLoadBarrier。其中ReleaseBarrier保证了Volatile变量的写操作不会与之前的读写操作重排序，这个Barrier ~~相当于C中的volatile关键字作用，(待考证，基本没问题，C中volatile所谓的从内存中取只是虚指，可能是Cache)~~ 能够禁止编译器和处理器重排序以及优化(如将变量保存到寄存器中，下次直接读取寄存器)。StoreLoadBarrier实际是Lock指令 + addl $0,0(%%esp)指令实现的。Lock指令显式地锁定了对应内存区域在Cache中的缓存段(老的CPU架构是锁总线)，强制的其他Cache的相关段的已修改部分会写入Memory并应用InvalidQueue无效化其CacheLine，同时应用当前StoreBuffer将当前Cache写入Memory，相当于刷新了StoreBuffer和InvalidQueue。同时由于锁定了对应CacheLine，在其解锁(S)前其他CPU无法修改对应内存地址。  
* Volatile读  
    Volatile读相当于：(LoadBarrier -> )变量读 -> AcquireBarrier。由于Volatile写无效化了其他所有Cache，当其他Cache读时需要从内存中拿取最新值，从而保证可见性。同时在读之后施加AcquireBarrier保证读不会与之后的读写重排 ~~，相当于C中的Volatile(待考证)~~ 。  
* 但是实际在x86是强内存模型不会对loadload，LoadStore以及StoreStore进行排序，因此只需要施加StoreLoadBarrier。考虑到一般是读多写少，因此StoreLoadBarrier一般施加在写变量之后而不是读变量之前。  

因此，Volatile可以保证可见性与有序性。同时Volatile读没有太大的性能损失，Volatile写要写入Memory同时维护多个Cache一致性会有一定的性能消耗。  

## CAS  

CAS底层在多核处理器下使用lock + cmpxchg指令，lock指令锁定了对应的cacheline，并进行cmpxchg，在当前指令结束前(当前CacheLine恢复成S状态)，其他CAS指令无法进行修改。但是每次的lock都会使得其他CPU的对应CacheLine无效，需要重新从Memory中load数据。  
CAS不会记录中间状态可能存在ABA问题，可以增加一个stamp字段来记录状态，比较时不仅比较value还要比较stamp，如AtomicStampedReference类。  

假设两个core都持有相同地址对应cacheline,且各自cacheline 状态为S, 这时如果要想成功修改,就首先需要把S转为E或者M, 则需要向其它core invalidate 这个地址的cacheline,则两个core都会向ring bus发出 invalidate这个操作, 那么在ringbus上就会根据特定的设计协议仲裁是core0,还是core1能赢得这个invalidate,胜者完成操作, 失败者需要接受结果, invalidate自己对应的cacheline,再读取胜者修改后的值, 回到起点。
