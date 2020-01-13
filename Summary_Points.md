# 常见知识点  

## JDK

* Integer(4B),Short(2B)默认缓存-128~127。在JVM中boolean没有明确定义大小，一种方式是实际使用int(4B)来读写；boolean[\]数组通过读写byte[]来实现，这里一个boolean相当于一个1B。  
Integer i = 123 应当等于 Integer i = Integer.valueOf(123);

* Object的方法：int hashcode(), boolean equals(Object obj), Object clone(), String toString(), Class<?> getClass(), void finalized(), void notify(), void notifyAll(), void wait(), void wait(long timeout)  

* 方法覆盖：访问权限可以放宽(protected -> public)，返回类型可以收窄

* 任何一个泛型都可以赋值给泛型原型，不是Object泛型。如List = List\<String\>, 泛型一般可以强转不会报错，如List\<String\>赋值给List原型，再强转为List\<Integer\>，但是强转之后受查类型就变成了Integer，这时参数只能给Integer的父类或子类，但是由于本质还是String，这时给Integer可能会错。  
  例子

  ``` java
    static class Service<T> {
        T echo (T a) {
            return a;
        }
    }
    // 这个是不会错的，因为方法是针对所有泛型的，因此使用哪种都可以
    Service<String> service1 = new Service<>();
    @SuppressWarnings("rawtypes") Service service = service1;
    @SuppressWarnings("unchecked") Service<Integer> service2 = (Service<Integer>)service;
    System.out.println(service1.echo("generic to String"));
    System.out.println(service.echo("raw type"));
    try{
        System.out.println(service2.echo(1));
    }catch(Exception e) {
        System.err.println(e.getMessage());
    }
    System.out.println("done");
  ```

  ``` java
  static class StringService implements Service<String> {
      @Override
      String echo(String s) {
          return "String " + s;
      }
  }

  static class IntegerService implements Service<Integer> {
      @Override
      Integer echo(Integer i) {
          return i*i;
      }
  }

  // 这时再执行上边的操作就会说Integer不能转为String，因为本质上还是String的泛型(方法是针对特定String泛型的)
  ```

* String为什么是不可变的，intern实现  
  String类由final修饰，其方法不在vtable(虚拟函数分发表)中，不允许被继承和方法覆盖，同时属于核心类库由BootStrap ClassLoader直接加载。内部拥有一个被final修饰的char[\]数组(jdk8)或byte[\]数组(jdk9)，因此只能初始化时被赋值，不能修改，修改实际是new一个string。因此String不可变。  
  JVM中有个固定长度为60013的hashtable实现的字符串常量池，JDK8后在heap中。String的intern方法的作用是，如果字符串未在 Pool 中，那么就往 Pool 中增加一条记录，然后返回 Pool 中的引用；如果已经在 Pool 中，直接返回 Pool 中的引用。正是由于String是不可变对象，才对其缓存，避免new多个equal的String。(双引号括中的字符串也会进这个Pool中)，Pool中类似弱引用，如果GCRoot不可达，那也会被回收。  
  String(String s)这个构造方法中新的String的value数组会共用s的value数组，不会复制一个(String不可变，没有必要new新的数组)。JDK8开始，String的substring方法会新new一个value数组，不再共用，防止内存泄露。  

* String,StringBuilder,StringBuffer  
  StringBuilder是final类不可继承和方法覆盖，继承自AbstractStringBuilder类，内部维护一个可变长char[16]数组，实现插入删除等操作。(注意，append(object)实际上为append(String.valueOf(object))，会多创建String)。  
  StringBuffer也是final类，同样继承自AbstractStringBuilder类，区别在于，每个对外方法都经过了synchronized关键字修饰。

* ThreadLocal与FastThreadLocal，以及WeakHashMap  
  在每个Thread对象中有一个ThreadLocalMap字段，这个字段是一个hashtable结构，table中每个元素是Entry类型，继承于WeakReference，内部有value字段，每个ThreadLocal对象被Entry弱引用，同时也相当于hashtable的key值，TreadLocal的hash值是全局有一个静态的AtomicInteger，提供每次递增的hash值。  
  ThreadLocalMap使用开放地址法解决冲突问题，对于get，通过ThreadLocal的hashcode得到table中的位置，然后从那个位置开始一直往后找，直到key相等或者数组中值为null；对于set，也是通过hashcode计算出位置，从那个位置开始往后找，如果key相等就修改value，否则在之后第一个null值处设置新的值。  
  ThreadLocalMap中的Entry对ThreadLocal使用弱引用，因此ThreadLocalMap不会影响ThreadLocal的回收，若在set,get,remove时，发现某个Entry引用的ThreadLocal已经被回收，就会将这个Entry置为null，以此尽量降低内存泄露的风险。ThreadLocalMap使用开放地址法的原因可能是：一般ThreadLocalMap中存放的ThreadLocal变量不会太多，没必要开链法；其次，开放地址法在寻找某个Entry时更有可能回收引用已经失效的Entry。  
  由于对ThreadLocal的操作都是在Thread自己的ThreadLocalMap中进行的，因此不存在线程安全问题。  
  
  Netty中自己实现了FastThreadLocal结构，也是每个Thread中有一个InternalThreadLocalMap对象，和ThreadLocalMap最大的不同在于，它是由可变长数组(直接存放value)实现的，不是hashtable结构。每个ThreadLocal通过静态AtomicInteger生成一个唯一int id，然后用这个id作为数组的下标。优点在于：数组肯定比hashtable快。缺点在于：数组可能更浪费空间，因为不能复用数组的空间，只能越用越大，适合数量不多使用频繁的数据。

  ThreadLocalMap中因为担心Entry对ThreadLocal有强引用，导致线程不结束的情况下永远无法回收ThreadLocal变量，所以才设计成弱引用，但是仍旧没解决对应value的回收问题，在某些情况下直到Thread销毁的时候才能回收value。至于Netty的FastThreadLocal，我觉得它只保证了一个runnable运行结束的时候能够清空它使用过的所有ThreadLocal，也就是可以使用FastThreadLocalRunnable包装一下runnable，在runnable最后加上一个清空当前线程所使用的所有FastThreadLocal和InternalThreadLocalMap。这个操作最主要的作用在于线程池，线程池的生命周期很长，这样可以让它每执行完一个runnable，就清空一下使用的FastThreadLocal。  
  此外，WeakHashMap中的Entry也继承了WeakReference，被引用的是key，Entry有额外的hashcode和value字段，同时Entry还和一个ReferenceQueue搭配。这样当某个key没有强引用被GC的时候，对应的Entry会进入ReferenceQueue，然后在每次set,get,remove的时候会将queue中的Entry从Map中摘出来，将对应的value置为null。从内存问题上看，WeakHashMap更好，每次清理Queue都能把无效的value清理干净。(ThreadLocal为什么不这么做?)

* AVL树和红黑树的主要步骤

* TreeMap的Iterator相当于是中序遍历，每次调用，预先找到下一个node存起来(下次调用返回这个)。如何找下一个node：如果Node有right，就找右子树里的最左节点；没有right就返回上层(parent字段)，如果当前node是parent的右子节点，就再返回上层，直到是某个node的左子树，然后进入这个node的右子树。

* ArrayList默认10大小，扩容1.5倍。FailFast：在数组结构发生变化(增加元素和删除元素时)记录modCnt(修改次数)，设置值读取值没关系。在进行迭代等操作时，会比较操作前后的modCount值，不相符抛出ConcurrentModificaitonException。

* 手写堆(已完成，见Heap.java)

* 实现LRU(自己实现LinkedHashMap)

* AQS和synchronized以及BlockingQueue

* Executors:  

    ``` java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
    }
    ```

    >最大线程数设置为与核心线程数相等，此时 keepAliveTime 设置为 0（因为这里它是没用的，即使不为 0，线程池默认也不会回收 corePoolSize 内的线程），任务队列采用 LinkedBlockingQueue，无界队列。
    过程分析：刚开始，每提交一个任务都创建一个 worker，当 worker 的数量达到 nThreads 后，不再创建新的线程，而是把任务提交到 LinkedBlockingQueue 中，而且之后线程数始终为 nThreads。

    ``` java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    ```

    >生成只有一个线程的固定线程池。

    ``` java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
    }
    ```

    >核心线程数为 0，最大线程数为 Integer.MAX_VALUE，keepAliveTime 为 60 秒，任务队列采用 SynchronousQueue。  
    这种线程池对于任务可以比较快速地完成的情况有比较好的性能。如果线程空闲了 60 秒都没有任务，那么将关闭此线程并从线程池中移除。所以如果线程池空闲了很长时间也不会有问题，因为随着所有的线程都会被关闭，整个线程池不会占用任何的系统资源。  
    过程分析：我把 execute 方法的主体黏贴过来，让大家看得明白些。鉴于 corePoolSize 是 0，那么提交任务的时候，直接将任务提交到队列中，由于采用了 SynchronousQueue，所以如果是第一个任务提交的时候，offer 方法肯定会返回 false，因为此时没有任何 worker 对这个任务进行接收，那么将进入到最后一个分支来创建第一个 worker。之后再提交任务的话，取决于是否有空闲下来的线程对任务进行接收，如果有，会进入到第二个 if 语句块中，否则就是和第一个任务一样，进到最后的 else if 分支创建新线程。

* 如果想中断Executors中运行某个task的线程，可以使用submit(Runnable r)拿到返回的Future(实现类为FutureTask)，调用返回Future的cancel方法，里边会interrupt运行这个task的线程。  

* System.currentTimeMillis() 和 System.nanoTime()  
  currentTimeMillis返回当前时间和1970-01-01 00:00:00 UTC的ms时间差值。主要调用gettimeofday()来实现(注意gettimeofday本身提供微秒级别，但是jvm只用到ms)，虽然是ms级别，但是OS的时间分辨率实现的不同，win下有些版本分辨率在10ms以上(For example, many operating systems measure time in units of tens of milliseconds)。  
  nanoTime()本身的值和当前时间没有明显的对应关系，可以计算高精度的ns时间差值，返回值是和某个固定但任意的时间值的差值。JVM保证同一个JVM进程中所有nanoTime()方法的调用都是返回与同一个起始时间的差值。nanoTime的精度不会比currentTimeMillis差。(有说实际走的是clock_gettime这个系统调用)。  
  gettimeofday()在常见平台上不是系统调用，nanoTime不仅是系统调用并且会比currentTimeMillis要更慢。在高并发的调用下性能下降很快，因此讲究的话可以使用时间缓存，如简单实现的话，使用一个定时任务按照一定分辨率更新时间缓存，其他线程只是读这个缓存。  

* 悲观锁/乐观锁 排它锁/共享锁  
  ReentrantReadWriteLock中的writeLock是排它锁，readLock是共享锁。但是个人认为，其readLock使用AQS的共享模式实现，原则上并不贴合乐观锁的概念。StampedLock中的乐观读模式更贴合乐观锁。  
  悲观锁认为每次读写数据都需要同步(可能需要阻塞)。乐观锁下不显式的做同步处理，CAS是一种常见的实现乐观锁的lock-free策略。  
  一般针对写写场景区分悲观和乐观策略，在读写场景下区分排他和共享策略。  

## JVM

* JVM的内存模型，其中哪些线程私有哪些线程共有，哪个不会内存溢出  
  按规范有Java堆，方法区(共享)；程序计数器，虚拟机栈和本地方法栈(私有)。其中程序计数器不会溢出。  
  从JDK8开始方法区由Meta-Space实现，是一块直接向OS申请的内存。从实际上看有Java堆，线程栈，JIT代码缓存区，元空间，直接字节缓冲区，GC算法使用的部分等。  

* 如何判断方法区是否溢出
  JDK8开始用Meta-Space实现方法区，如果Meta-Space溢出会抛出Oom错误并注明meta space。  
  默认meta-space是不限大小直接向OS申请虚拟内存的，但是如果默认的压缩对象指针和压缩类指针生效(堆大小<32GB)那么meta-space中用于Klass的Class-part默认大小为1G，这个部分是可能会溢出的。通过{java -XX:+PrintFlagsFinal | grep 'CompressedClassSpaceSize'}来查看。一般class-part和non-class-part在1 : 6~8左右。  
  排查的话，可以先用-XX:MaxMetaspaceSize将metaspace设置小一些来重现错误，并且开启VisualVM观察。溢出一般是加载的class太多，超过了size限制。主要关注动态生成类的部分代码。  

* 类加载数据存放在哪里(元数据\*\*klass在方法区meta-space，class对象在堆区)  
  一个类在JVM中的对等实体是InstanceKlass，放在**Meta-Space**中(CompressedClassSpace, Class-Space)，相应的类实例(JAVA对象)则是InstaceOop放在堆中，且_meta字段指向InstaceKlass。  
  InstaceKlass中有个_java_mirror字段，这个指向的就是InstanceMirrorKlass，存放在**堆**中，这个就是java.lang.Class对象，也就是平时obj.getClass()得到的。对静态方法使用synchronized或者synchronized(xxx.class)实际上锁的是这个Class对象(InstanceMirrorKlass)。例如Object.class == new Object().getClass() 为true。  
  JDK8后类的静态字段不再放在InstanceKlass中，放在了Mirror中。  
  补充：objA.getClass() != **classA_obj**.getClass().getClass() == **classB_obj**.getClass().getClass().getClass()...  
  首先obj.getClass()拿到InstaceKlass的Mirror(Class实例)，obj.getClass().getClass()拿到 java.lang.Class 类对应的 java.lang.Class (Mirror)实例，之后就是重复了。  

* Meta-space什么时候才会被回收  
  meta-space是按照或者说由ClassLoader来分配的。  
  Node(最接近OS一次性分配，链表串联，只有当一个Node中所有的ClassLoader都被卸载，才会将Node归还给OS)->Meta-Chunk(一个Node由多个ClassLoader共享，每次给ClassLoader一个Chuck，当ClassLoader被卸载的时候，对应的所有Chuck都变为空闲，可以被复用)->Meta-block(一个Chuck也比较大，内部接着指针碰撞来分配不同的blocks)。  
  每一个ClassLoader有一个(主要的)ClassLoaderData引用ClassLoaderMetaSpace，其中记录了这个ClassLoader使用的所有chunk，对于每个匿名类有自己的ClassLoaderData(为了尽量降低匿名类的生命周期)。  
  只有当这个类加载器加载的所有类都没有存活的对象，并且没有到达这些类和类加载器的引用时，ClassLoader被卸载，相应的 Metaspace 空间才会被 GC 释放。
  
  充分利用了Java语言规范：类及相关的元数据的生命周期与类加载器的一致
  每个类加载器都有它的内存区域-元空间
  只进行线性分配
  不会单独回收某个类（除了重定义类 RedefineClasses 或类加载失败）
  没有GC扫描或压缩
  元空间里的对象不会被转移
  如果GC发现某个类加载器不再存活，会对整个元空间进行集体回收

* Class什么时候被卸载(实际应当只有ClassLoader卸载时，释放其对应的Meta-space，不存在单一的回收Class，在不考虑匿名类的情况下)  
  该类所有的Instance都已经被回收  
  加载该类的ClassLoader被卸载  
  该类的Class对象没有被引用  

* 堆和栈中各自存放什么东西
  Java堆中存放Oop对象，如new出来的对象。JDK8后将字符串常量池也放进了堆中，还有JAVA类对应InstanceKlass对应的Mirror对象，也就是Class对象也在堆中，包含类的静态字段等。  
  栈中存放有一系列连续的栈帧，每个栈帧依次包括局部变量表，固定帧和操作数栈。其中局部变量表包括方法入参和局部变量，方法调用方的操作数栈和被调用方的局部变量表有交叠复用。固定帧是一块固定格式的区域，保存有一些当前信息，比如返回地址，methodOop的地址，局部变量表指针，第一条字节码指令的指针，等。操作数栈是用来求值运算的区域。JVM栈中使用的变量类型有double(8B)，word(4B)，short(2B)，byte(1B)和指向堆上对象的引用类型oop(4B/8B)。  

* 如何判定一个对象是否可以回收，哪些可以作为GCRoot
  从GCRoot开始进行可行性分析。栈中局部变量表中的引用，方法区中常量或类静态字段引用的对象。  
  当前各线程执行方法中的局部变量（包括形参）引用的对象  
  已被加载的类的 static 域引用的对象  
  方法区中常量引用的对象  
  JNI 引用  

* 垃圾回收介绍，调优

* 压缩指针  
  分为2种，一个是对象压缩指针，一个是压缩类指针  
  JVM默认对Java对象8B对齐，最后3bit为0，因此32bit可以表示2^35 = 32G空间，这就是Oop的压缩。  
  另外在Oop的头中有_meta字段，指向对应的Klass，这个字段也是可以压缩的，需要在Oop压缩的情况下。然而，Klass是没有做8B对齐的，因此它是在MetaSpace中开了1块固定基地址的连续内存CompressedClassSpace(默认1G)，那么直到基地址后只需要32bit的offset就可以了，从而实现压缩。这块空间存放的应当是InstanceKlass，其余的Klass如MethodKlass或ConstMethodKlass或ConstantPool等不会被压缩指针访问，因此没有必要在这块区域。  
  (注：methodOop存储Java方法的名称，签名，访问标识，解释入口等。constMethodOop存储字节码指令，行号表，异常表等信息)

## 网络  

* NAT网络地址转换。内网多个IP共享一个外网IP，使用不同的外网IP端口port来区分。

### 传输层TCP

* 服务端出现大量Close_Wait的原因是什么

* 如何确定msl

* TCP段长和MTU
  TCP头固定20B，TCP软件可以将多个写操作合并为一个大段，或者将过长的分割。段长受2个因素影响，一个是IP要求有效载荷不超过65515(65535-20)，另一个是在MTU最大传输单元范围内，以太网有效载荷为1500B，现在可以用ICMP错误消息来发现某条路径上的MTU大小。

* TCP3次握手  
  服务端处于Listen状态
  客户端发送连接请求SYN=1(置位)，seq=x(选择一个序列号)，ACK=0(ACK是符号指示位，确认号是ack)，此时处于SYN_SENT状态  
  服务端收到后，发送确认SYN=1，ACK=1，seq=y，ack=x+1，此时服务端处于SYN_RCVD状态，如果没有程序在Listen这个端口就会发送RST拒绝  
  客户端收到后，发送确认ACK=1，seq=x+1，ack=y+1，此时客户端处于ESTABLISHED状态  
  服务端收到后，进入ESTABLISHED状态(发送ACK?)

  使用三次握手的原因在于：防止失效(在网络中滞留)的连接请求到达服务端，使得错误打开连接。当滞留的请求到达并且服务端应答后，客户端可以通过第三次握手来拒绝连接请求。  

* TCP4次挥手  
  一方(通常为客户端)发送释放请求，FIN=1，此时进入FIN_WAIT_1状态  
  另一方收到后立即返回ACK=1，此时位于CLOSE_WAIT状态，这个ACK一般是底层支持的，因此客户端不太会长期处于FIN_WAIT_1状态  
  一方收到ACK，变为FIN_WAIT_2状态，此时处于半关闭状态，即客户端->服务端的已经关闭，但是服务端仍旧可以给客户端发送信息
  另一方当数据都发生完时，发送FIN请求，处于LAST_ACK状态  
  一方收到FIN后发送ACK，进入TIME_WAIT状态，等待2msl后CLOSED，对端收到ACK后进入CLOSED状态  

  使用4次挥手的原因：因为连接是双向的，因此关闭从语义上只是关闭我这个方向的写操作，使得对端如果存在还没有发送的数据还可以接着发送，然后在关闭  

* 为什么Time_Wait状态需要等待2msl  
  被动关闭的一方通常为服务端，在其发送Fin报文后进入last_ack，客户端收到后发送ack同时自己进入Time_Wait状态。等待2msl主要原因有2:
  其一，如果客户端发送的最后一个ack丢失了，对方会重新发送Fin报文，这段时间内客户端还处于Time_Wait状态，因此可以再次发送ack。  
  其二，2msl能保证旧的重复分节在网络中失效，这样旧的报文就不会对新的连接实体造成干扰。  

* TCP可以在上一段的ACK没有到达前，发送下一个段吗  
  使用了Nagle算法(没有TCP_NODELAY)并且大小没有MSS不行
  如果不超过对端期待的窗口大小awnd和拥塞窗口cwnd(包括半关闭状态)，那还可以发送

* TCP使用累计确认，即ACK确认之前的所有包(新的协议中有SACK项，其中可以指示其他确认的区间，方便选择重传)，TCP用的也是选择重传(每次传1个)

* TCP中4个计时器  
  第一个是重传计时器，用来控制是否要超时重传，主要跟平滑往返时间相关  
  第二个是持续计时器，用在流量窗口中，若一端将对端的发送窗口置0，之后更新窗口的包丢了，那对端在持续计时器到期时，会发送探询消息  
  第三个是keepalive计时器，确定连接是否还有效  
  第四个是用在TIME_WAIT状态时的计时器，默认2msl  

* TCP拥塞控制：TCP-Reno版本(在Tahoe版本的基础上增加快速恢复)  
  开始以较小的拥塞窗口启动(慢开始)，进入指数递增阶段，直到阈值进行线性增长。之后分为2种情况：如果发生了超时，则将阈值置为当前窗口的一半，并重新从较低水平开始指数增长，重复之前的行为。由于每个包可能从不同路径传递，因此可能出现顺序错误，但不会太常见，因此TCP认为3次重复ACK发生了丢包，执行快速重传，此时将阈值置为当前窗口的一半，并且从阈值开始线性递增，而不是超时时候的从1开始递增。

* SocketChannel的close和shutdown  
  对SocketChannel来说，close()方法实际使用Net.shutdown(fd, Net.SHUT_WR)，并且如果不是register到一个selector上，就在kill()即nd.close(fd)。shutdownInput()就是Net.shutdown(fd, Net.SHUT_RD)然后唤醒阻塞在上边的读线程，shutdownOutput()就是Net.shutdown(fd, Net.SHUT_WR)然后唤醒写线程。  
  在C中close()和shutdown(fd, how)，分别指关闭本进程的socket id，不可以读写，但链接还是开着的，用这个socket id的其它进程还能用这个链接，能读或写这个socket id，直到所有持有socket的进程都close才发送FIN包。shutdown效果累计，不可逆转，直接作用于Socket实体，以SHUT_WR/SHUT_RDWR方式调用即发送FIN包(RD似乎只是在内核上丢弃数据)  
  JDK中在使用正常的TCP时只使用close就好了，不影响四次挥手，只是内核对数据态度不同。  

### 应用层

* DNS使用53端口，既可以TCP也可以UDP(通常UDP)  
  可以直接ping DNS服务器如114.114.114.114，ICMP回复，DNS是web server
  可以直接ping域名(会先拿到对应IP?)

* FTP需要2个TCP连接  

* DHCP使用UDP  

* 浏览器输入URL的过程  
  首先进行DNS解析域名，主机通过ARP获得网管路由器的MAC地址，向网管路由器发送DNS查询报文。网关路由器根据路由表，进行路由转发。DNS收到查询报文后，查询对应域名记录，回复给主机。  
  获得HTTP的IP地址后，主机通过3次握手进行TCP连接请求。连接建立之后发送HTTP报文，并等待响应，对Web页面内容进行渲染，最后在必要的时候断开TCP连接。  

#### HTTP  

* HTTP方法  
  GET,HEAD(获取报文首部，不返回报文实体),POST(传输数据),PUT(上传文件),PATCH(对资源进行部分修改),DELETE(删除文件),OPTIONS(查询支持的方法),CONNECT(要求与代理服务器通信时建立隧道),TRACE(服务器会将通信路径返回给客户端)

  POST和GET的区别：  
  Get的参数一般在URL中，并且只支持ASCII码，中文需要转码；Post一般在body中，支持标准字符集。但是都是不安全的，Post也可以被抓包。  
  Get方法不会改变服务器状态，且是幂等的，可被缓存；Post一般不是，也不能缓存。  
  Ajax使用XMLHttpRequest方法时，GET的Header和Data一起发送；Post可能会拆分成两部分(火狐不拆)

* HTTP状态码  
  100 Continue，目前正常，客户端可以继续发送请求，忽略这个响应  
  200 OK  
  204 No Content，请求成功处理，不需要返回数据  
  3xx 重定向
  301 Moved Permanently，永久性重定向  
  302 Found，临时性重定向  
  400 Bad Request，报文存在语法错误  
  401 Unauthorized，用户认证失败  
  403 Forbidden，请求被拒绝  
  404 NOT FOUND
  500 Internal Server Error，服务器执行请求时发生错误  
  503 Service Unavailable，服务器超负荷或停机维护，当前无法处理请求  

* Cookie  
  服务端set-cookie迫使客户端保存数据  
  根据生命周期分为会话期Cookie(浏览器关闭后删除)，持久性Cookie(创建时指定有效期)  
  作用域：不指定默认为当前域名不包含子域名，指定Domain一般包含子域名如Domain=mozilla.org则包含developer.mozilla.org  
  HTTP-Only属性可以要求该Cookie不被js使用，一定程度可以避免XSS
  
  Cookie存放在客户端，适合少量的非敏感的字符串。Session可以存放任何数据，但是可能有分布式问题  

* 代理
  缓存，负载均衡，网络访问控制，访问日志记录  
  正向代理，用户能够感知。反向代理，用户不能感知。  

* 网关  
  网关服务器可能将HTTP转化为其他协议进行通信  

* 隧道  
  特指使用SSL，TSL等手段在客户端与服务端间建立安全线路

* HTTPS  
  在TCP和HTTP间加一层SSL，提供加密(防窃听)，认证(数字证书，防伪装)，完整性保护(MD5值，防篡改)

* HTTP2的协商过程，其优势(多路复用，多个流在同一个TCP连接上传输；压缩首部；服务端推送)

## Sql

## 操作系统

### 进程与线程  

* PCB与task_struct
  Linux将thread_info(线程描述符)和内核态线程堆栈放在一起，其中thread_info有task字段指向task_struct进程描述符。每个进程有一个进程控制块PCB，这个概念对应的数据结构实现是task_struct。  

* 调度的简略流程  
  一个调度主要负责两部分，第一部分决定下一个应当运行的进程，第二部分上下文切换。当进程的ideal_time用完(周期性scheduler判断后交给主scheduler)，或主动让出CPU，阻塞等让出CPU以及中断时，会进行调度。  
  对于普通的非实时进程，使用CFS(Completely Fair Scheduler)调度，有SCHED_NORMAL和SCHED_BATCH调度方式。  
  每个CPU拥有一个cfs_rq(run queue)，实时rt_rq，和deadline的dl_rq。这个队列是一个红黑树结构。  
  (优先级和load_weight)每个task_struct有prio, static_prio, normal_prio和rt_prio这些优先级。其中static_prio由Nice值计算出，0-99为实时优先级，100-139为普通线程优先级，nice值-20~19对应100-139的static_prio。normal_prio表示基于进程的静态优先级static_prio和调度策略计算出的动态优先级，由于可能会临时给它bonus，因此还有一个prio。优先级会转换为load_weight，定性关系为，优先级越高，load_weight越小。  
  (红黑树和vruntime)CFS通过每个进程的虚拟运行时间(vruntime)来衡量哪个进程最值得被调度。CFS中的就绪队列是一棵以vruntime为键值的红黑树，虚拟时间越小的进程越靠近整个红黑树的最左端。因此，调度器每次选择位于红黑树最左端的那个进程，该进程的vruntime最小。  
  (vruntime和load_weight)虚拟运行时间是通过进程的实际运行时间和进程的load_weight计算出来的。vruntime是单调增的，每次更新时加上物理时间差(当前时间和上次更新时间的差值)\*(NICE_0的load_weight/当前优先级的load_weight)，可以看到，Nice值越小，load_weight越大，vruntime增长的越慢，因此更有机会被调度。  
  (被分到的时间片)调度器有调度延迟的概念，即保证每个可运行的进程都应该至少运行一次的某个时间间隔，默认20ms。若线程数没有超过阈值(5?)，每次调度的总运行时间sum_runtime(本次调度决定之后的多少ms工作时间?)为20ms否则为4ms*线程数。这段总时间按照每个load_weight比例分配给进程，作为进程被分到的时间。  
  (pick_next)更新参数后将当前进程放入红黑树，接着调度器决定下一个运行的进程时，将其从红黑树取出，并进行上下文切换。  

* 上下文切换context_switch()  
  进程描述符中active_mm指向进程**使用**的内存描述符，mm字段指向进程**拥有**的内存描述符。对于一般进程，这两个字段相等。对于内核线程它没有自己的地址空间并且它的mm字段总是被置为NULL。  
  上下文切换保证，如果next进程是内核线程，它使用前一个的地址空间。  
  上下文切换主要包含2部分：  
  调用switch_mm()，将虚拟内存从一个进程映射到新的进程，也即切换进程虚拟地址空间(内核虚拟地址空间不需要切换)，主要工作是切换CR3控制寄存器(页目录表物理内存的基地址)。  
  调用switch_to()，切换进程堆栈和寄存器。它保存原进程的所有寄存器信息，恢复新进程的寄存器，主要是切换esp(从esp可以找到进程描述符)，切换ip寄存器(硬件上下文)，切换ebp(栈底指针)。  

* Linux进程状态  
  R: 正在运行或可运行(就绪)，位于run-queue中  
  D: 不可中断睡眠(常用于硬件IO，因为它们一旦中断唤醒，可能出现不可预知的问题)  
  S: 可中断睡眠(等待一个事件完成)  
  Z: 僵尸进程(被终止的进程，但是子进程的进程描述符不会被释放，直到父进程通过wait()或waitpid()获得子进程的信息。如果没有被父进程读取信号，那么进程描述符不会被回收，成为zombie。如果想要消灭zombie，可以终止父进程，这样父进程的所有子进程会被init进程收养，从而被回收描述符。)
  T: 终止

* JAVA线程状态  
  New: 新建Thread调用start转为Runnable
  Runnable： 正在运行或可运行  
  Blocked: synchronized或Lock(?)阻塞  
  Waiting: Object.wait()方法，Thread.join()，LockSupport.park()  
  TimeWaiting: sleep()  
  Terminated: 遇到Exception或是正常结束  

### 内存管理  

* 段页式内存  
  一个虚拟地址由段号+页号+页内偏移组成。  
  段号+段表起址得到段号，若该段号>段表最大长，说明非法越界，进入越界中断。否则查询段表拿到对应页表的首地址，与页号相加得到物理块号，如果不存在则引发缺页置换，接着和页内偏移拼接，得到物理地址。  
  页式优势：由于页面大小固定，分割分配简单，容易创建页表，不存在外碎片。  
  段式优势：根据段长可以分配不同大小的区域，更灵活，并且容易做越界检查。  
  分页主要为了实现虚拟内存，用较少的物理空间获得更大的虚拟空间。分段使得不同的程序和数据可以被划分为独立的地址空间，有助于共享和保护。  

## Spring  

### SpringBoot  

@SpringBootApplication相当于@EnableAutoConfiguration，@ComponentScan，@SpringBootConfiguration。  

#### 独立的SpringBoot应用与JarLauncher  

打包好的jar包中：  

* BOOT-INF/classes 存放应用编译后的class文件  
* BOOT-INF/lib 存放应用依赖的Jar包  
* META-INF/ 存放应用相关的元信息，如MANIFEST.MF文件  
* org/ 存放spring boot相关的class文件  

FAT JAR和WAR的执行模块为spring-boot-loader，其中JarLauncher和WarLauncher分别是Jar和War的启动器。  
  java -jar规定main-class必须指定在MANIFEST.MF文件中。  

Jar中MANIFEST.MF中main-class为JarLauncher，start-class才是我们写的主类如quickstart默认的App.class。  
JarLauncher的main()为new JarLauncher().launch(String... args)。JarLauncher的launch()方法在完成准备工作，创建完ClassLoader之后，load我们写的main-class，通过反射拿到它的main()方法，然后invoke这个Method对象。  

## Netty  

Netty实现了NIO中常见的Reactor模型，以服务端为例，一般情况下，会配置一个BossGroup和一个WorkGroup，每个NIOEventLoopGroup中有一个EventLoop数组，在Netty中EventLoop是实际的处理单元，每个NIOEventLoop拥有一个Selector。和JDK对应，Netty分别实现了NIOServerSocketChannel和NIOSocketChannel，内部分别持有ServerSocketChannel和SocketChannel，每个Netty实现的Channel绑定到了一个EventLoop上，也就是每个链接只由一个NIO线程来读写。  

EventLoop也就是NIO线程，内部拥有一个taskQueue，用来执行外部任务。概括的来说，其run方法是个无限for循环，首先通过selector进行select检测IO事件，接着当select返回的时候，processSelectedKey处理感兴趣的事件，最后根据IORatio处理taskQueue中的任务，包括外部任务和定时任务。  

然后大概说一下服务端的启动流程。调用ServerBootstrap的bind方法的时候，首先新建NIOServerSocketChannel并初始化，然后从EventLoopGroup中挑选下一个EventLoop赋值给ServerChannel，并且调用JDK的register方法，将对应的JDK的ServerChannel注册到Selector上，最后会使用JDK的bind方法启动监听。  

值得注意的是，bind操作是异步的，每个Channel只允许其绑定的EventLoop进行读写，调用bind方法的线程一般是外部线程，所以上边的register注册到Selector和JDK的bind方法是分别封装成task，放进EventLoop的taskQueue中的。EventLoop只有当第一次需要执行task的时候才会启动，外部线程将task放进EventLoop的taskQueue中时，如果发现还没有启动，就会启动一个新的线程，这个线程就是EventLoop的NIO线程。  

新启动的EventLoop线程执行taskQueue中的bind操作，开始了监听。当有accept事件时，创建对应的NIOSocketChannel，然后从workGroup中挑选下一个EventLoop，与创建的NIOSocketChannel绑定，将SocketChannel注册到Selector上封装成一个task，放进EventLoop的taskQueue中，从而完成新连接的接入。  

还有几个有意思的地方。  
第一个是NIOEventLoop定时任务的实现，EventLoop除了taskQueue以外，还有一个优先级队列，用来存放定时任务。每次selector进行select的时间由最近一个定时任务的到期时间来决定，select结束并且处理完select到的事件后，会开始执行外部任务，首先将当前所有到期的定时任务从优先级队列取出放到taskQueue中，然后从taskQueue中取任务执行。这里还用IORatio参数控制执行任务的数量，默认策略是每次处理IO事件和taskQueue中任务的时间1:1，具体来讲，每次统计处理select到的事件的处理时间，然后每完成64个taskQueue中的任务后判断一下，执行task的时间是不是比IO事件的处理时间长了。  

第二个是EventLoop的实现。JDK中的Selector的epoll默认level_trigger，Selector的wakeup(从阻塞的epoll中唤醒)使用pipe。Netty中NioEventLoop使用JDK的Selector，每次阻塞select前会计算最邻近的定时任务的时间，然后使用带超时的epoll_wait。  
Netty自己实现的EpollEventLoop使用edge_trigger，wakeup使用eventfd，epoll的超时使用timerfd来实现。  
epoll的管理使用mmap+redblacktree+deque，被注册的fd行为存储在红黑树中，当对应事件触发时通过epoll_call_back添加到rdlist(ready lists)中，epoll实际只需要关注于rdlist是否为空就好了。其中ET仅当事件状态改变时触发，也就只有通过epoll_call_back方式加入到rdlist中，对于读事件为可读部分变多(不可读变为可读 或 可读字节变多)，对于写事件为可写部分变多(不可写变为可写 或 可写字节变多)。对于LT，除了epoll_call_back方式，在拷贝到用户空间后，仍会放入rdlist，在下次epoll_wait时再判断事件是否满足。若感兴趣的是事件对应的数据，那么使用ET必须要每次读到不能读为止，否则除非到有新数据到来，剩余的数据都无法再获得了。LT则要省心的多，不会丢失数据。但是LT需要对一个event进行更多次的poll来判定是否符合要求，效率更低一点(相当于每次的rdlist更长)。  
对timerfd和eventfd而言，我们只是关系是否被触发，至于其返回的数据是什么并不关心，因此使用ET，并且不用去read它们，但如果使用了LT，则必须要read它们，否则会一直epoll返回，因此对于timerfd和eventfd来说，使用ET可以减少系统调用read的次数。  

第三个是EventBus的实现。很多EventLoop的组件还会实现EventBus，比如Vertx，EventBus可以实现publish和subscribe等msg传递的功能。当注册一个msg的Handler时，可以将订阅的address，对应的处理handler以及归属EventLoop包装起来放在EventBus类中。当要publish或send一个msg时，从EventBus中找到所有对应address的handler，包装成一个task，放进归属的EventLoop的taskQueue中就实现了msg的投递和消费。  
