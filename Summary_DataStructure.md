# Data Structure in JDK

## HashMap(JDK8)

HashMap由数组表示slot，每个slot存放一根链表或者红黑树。key和value都可以为null。  

* 初始化  
  capacity默认为16，当构造方法中设置了initCapcity时，在第一次put时(table为null)会以设置的initCapcity作为长度。  

* put
    1. 根据key的hashcode计算出hash值((h ^ (h >>> 16))，高16位不变，低16位与高16位亦或(手动扰动))。  
    2. 如果数组为空，则先进行resize初始化。  
    3. hash值与数组长度-1做与相当于对长度取模，拿到slot中的第一个Node。如果这个位置为空，直接放入Node就可以了。  
    4. 如果不为空且是红黑树将节点插入红黑树中；如果是链表，遍历整根链表用key进行比较，如果有相同的根据onlyIfAbsent进行覆盖，没有相同的插入链表末尾，如果插入后链表长度大于等于8个，则会将链表转为红黑树，并且如果插入后size大于阈值，则会进行resize。  

* resize扩容
    1. 将原数组长度和阈值都扩大一倍，创建新的数组。  
    2. 遍历原数组的所有slot，由于新数组长度×2，因此rehash后一个Node要么在原来位置，要么往后原长个slot。若当前slot非空，且为链表，则遍历整根链表，将它拆成2根新链表，并且保留原来的先后顺序。  

* get
    1. 计算key的hash值，找到对应的slot。
    2. 如果该位置就是要的直接返回，如果是红黑树调用红黑树的查找方法，如果是链表则遍历找到相等的key，没有就返回null。

## ConcurrentHashMap(JDK8)

  ConcurrentHashMap也是数组为slot，每个slot中存放一根链表或者红黑树。key和value都不能为null(null说明可能存在错误，需要额外处理)。

* 初始化  
  默认构造函数中什么也没做，当设置了initCapacity时，实际设置的是sizeCtl(minPow2(1.5*init+1))。  

* put  
  put中使用putVal，onlyIfAbsent为false。  
  1. 首先根据key的hashcode计算出hash值((h ^ (h >>> 16)) & HASH_BITS，高16位不变，低16位与高16位亦或(手动扰动)，然后取低31位，也就是强制为正数，因为负数用来表征其他情况(-1为forwardNode，-2为TreeBinNode))。接下来所有的操作在大for中。  
  2. 接着如果数组为空，则对数组进行初始化。  
  3. hash值与数组长度-1做与相当于对长度取模，拿到slot中的第一个Node。如果该位置为空，对key/value创建一个新节点，使用一次CAS将它放入该位置，成功就返回，失败说明存在并发操作等情况，再次进入循环。  
  4. 如果第一个节点的hash值为MOVED，说明是forWordNode，这个节点已经被搬迁了，整个Map正在扩容中，使用helpTransfer帮助搬迁，之后再次进入for循环。  
  5. 上边情况都不满足说明头结点不为空，则拿到头结点的监视器锁，如果头结点的hash值>=0说明是链表，遍历这个单向链表，如果发现有相等的key，覆盖这个node的value，没有相等的key，就把他放到链表的最后边。在遍历的同时，会统计插入后node的节点数，如果大于临界值8，但是数组大小小于64就数组扩容，数组大小大于等于64就将这个链表转为红黑树。如果头结点是红黑树节点，则插入红黑树中。成功后跳出大for循环。  

  如果put某个slot时，这个slot尚未resize，那么就能拿到监视器锁再原map上修改；如果正在resize，那么就拿不到监视器锁，等到拿到了会在判断头是否修改了if (tabAt(tab, i) == f)，修改了说明被搬迁了，重新走for循环；如果已经resize这个节点就是forwardNode，会helpTransfer。

* 初始化数组  
  在数组为空的时候put会使用initTable初始化数组，ConcurrentHashMap中有个int的sizeCtl字段，经常在不同的方法中作为状态控制。这里通过CAS给这个字段赋值-1，成功的话说明可以给数组赋值，数组默认长度为16，构造方法中如果给sizeCtl赋值了，那就用sizeCtl的值。  

* 扩容tryPreSize  
  扩容每次增大一倍。通过sizeCtl字段标示还没有扩容或是扩容中，如果大于等于0，说明还没有扩容，通过CAS将sizeCtl设置为一个负数的值，成功说明第一次迁移，调用transfer方法，第一个参数是原数组，第二个参数是要搬迁的数组，这里是第一次所以第二个参数为null。一次扩容会涉及到一个transfer(null)和多次transfer(新数组)。  

* 搬迁transfer  
  ConcurrentHashMap有nextTable字段，扩容过程是在nextTable中创建新的数组并搬迁，全部完成后再替换原来的table。迁移过程可以被多个线程执行，put方法中如果put的slot的首节点是forwardNode说明map正在扩容中，也会使用transfer帮助搬迁。假如原数组长度为n，会以stride为步长(n>>>3/nCPU)最小为16，从数组尾巴开始，每次搬迁stride个slot。全局的transferIndex字段表明了当前需要搬迁的位置，初始从尾部开始。  
  前边说到调用transfer会传入原数组和新数组，第一次新数组为null。这时会将nextTable赋值为一个新的数组，大小为原来的2倍，同时将transferIndex设置为原长。如果transferIndex<=0说明所有的搬迁任务都已经分配给不同线程了，如果当前线程是最后一个执行完任务的线程，那就将finish标志置为true，将新数组赋值给老数组字段，并且将sizeCtl设置为新数组长度的0.75倍；如果当前线程不是最后一个执行完的，那么就直接返回。(其他线程使用transfer方法前会给sizeCtl+1，transferIndex<=0时会用CAS减去1，如果是sizeCtl恢复为原值了，那就说明是最后一个线程)。如果transferIndex足够，使用CAS给transferIndex减去stride，成功的话，就拿到了本次的任务。如果搬迁的index的首节点是空的，那么就CAS放入一个forwardNode。如果这个位置已经是一个forwardNode，就跳过。是有效节点，就拿到它的监视器锁(旧table上的首节点)，如果是红黑树就进行红黑树的搬迁，如果是链表那么和HashMap一样要么是原位置，要么是原位置+n。根据每个node的hash值与原长相&，是0说明在原位置，1为新位置，克隆每个节点，创建两条新链表(应该是每次插头)，放到对应的index中，然后首节点设为forwardNode。(还有一个优化，会先遍历一遍链表，找到某个位置，这个位置之后的节点hash与原长&的结果相同，也就是都是原地址或者新地址，省去克隆过程)。  

* get
  1. 通过key的hashcode计算出hash值，拿到对应的首节点。  
  2. 如果节点为null直接返回null，如果刚好是我们要的直接返回value。  
  3. 如果该节点hash值小于0说明要么在扩容(forWardNode，-1)要么是红黑树节点(TreeBin，-2)。直接调用node的find方法，分别被ForwardNode和TreeNode覆盖，如果是forwardNode，则会在还没有完全搬迁好的nextTable中查找(但是这个位置已经搬迁好了)。  
  4. 如果是链表节点，往后遍历就好了。  

### Note

1. volatile修饰了数组不能保证数组元素的可见性。从JDK9开始，拿到节点tabAt只用了getAcquire(我觉得可能有可见性问题，但是不会影响put，因为锁外CAS最多浪费一次，锁内保证可见性)，在锁内设置数组值用setRelease，在锁外用的CAS。JDK8用的是getVolatile，setVolatile和CAS。  

## MappedByteBuffer & DirectByteBuffer  

DirectByteBuffer(堆外，不受GC限制)  
MappedByteBuffer(mmap)  
HeapByteBuffer(不是页对齐的，也就是说如果要使用JNI的方式调用native代码时，JVM会先将它拷贝到页对齐的缓冲空间。受到GC限制，可能被GC线程搬移了位置，因此往往真正写到Channel时会先复制到临时的DirectBuffer中)  

直接内存区域由-maxDirectMemorySize设置(主要给DirectByteBuffer用的)，默认约等于-Xmx的JVM堆最大值。理论上Native内存只受OS虚拟内存和实际物理内存限制，但是这个参数是JVM层面做的限制，也就是JVM会统计使用的大小，如果达到限制了，就不给再开了(实际OS可能还可以再开)。  

DirectByteBuffer继承自抽象类MappedByteBuffer，但是并不意味着DirectByteBuffer或是MappedByteBuffer使用了mmap。常用的ByteBuffer.allocateDirect()实际只是使用了DirectByteBuffer(int cap)这个构造函数，里边malloc来开辟一块空间，没有做什么mmap操作，对应的映射fd为null。  
(Java中的MappedByteBuffer是否只支持FileChannel?)在FileChannel(Impl)的map()方法中会产生一个mmap的MappedByteBuffer(DirectByteBuffer)，具体而言map()使用了mmap，拿到映射addr，然后通过反射调用protected DirectByteBuffer(int cap, long addr, FileDescriptor fd, Runnable unmapper)这个构造方法。也就是说DirectByteBuffer的创建本身和mmap没有任何关系，重点是传入的addr(如果有)是否是mmap的。  

而不管有没有使用mmap，Direct/Mapped都需要释放对应空间，这个是不受GC控制的(Direct/Mapped对象本身是在堆上的)，因此一般会将释放操作做成一个runnable然后用Cleaner(对象)包装一下，然后DirectByteBuffer持有这个Cleaner。Cleaner继承自虚引用，引用DirectByteBuffer(跟踪DirectByteBuffer)(注意，虚引用不论何时get返回的都是null，和ReferenceQueue一起使用，跟踪引用对象的存活状态)，当DirectByteBuffer被GC时，Cleaner执行其Clean方法(好像不会进ReferenceQueue)，从而释放Native内存。在Reference类中有个ReferenceHandler继承自Thread在里边会处理所有的Reference，如果是Cleaner类型会执行其clean方法，其它类型就进各自的queue，ReferenceHandler在Reference的static{}代码块中被启动。  

### [mmap & zero-copy](<https://www.linuxjournal.com/article/6345>)  

通过FileChannel的map()方法使用mmap来产生一个MappedByteBuffer。使用FileChannel.transferTo()方法来使用sendFile()方法(可能使用到了zero-copy)。  
zero-copy是针对CPU是否参与而言的，不考虑单纯的DMA读写。硬件与内核空间的交互由DMA传输，内核到内核，内核与用户空间需要CPU参与。  

1. 一读一写传统4次copy，加上内核态用户态切换  
  硬件 --DMA copy--> 内核buffer --CPU copy--> 用户buffer --CPU copy--> 硬件缓冲区如socket buffer --DMA copy--> 硬件  

2. mmap，也需要内核态用户态切换  
  硬件 --DMA copy--> 内核buffer(map共享 用户buffer) --CPU copy--> 硬件缓冲区如socket buffer --DMA copy--> 硬件  

3. sendfile，不需要切换  
  硬件 --DMA copy--> 内核buffer --CPU copy--> 硬件缓冲区如socket buffer --DMA copy--> 硬件  

4. sendfile，zero-copy(硬件需要支持gather，可以从多个内存区域组装数据)  
  硬件 --DMA copy--> 内核buffer --不需要赋值，添加相关描述符到socketbuffer即可--> 硬件缓冲区如socket buffer --DMA copy--> 硬件  

## ScheduledExecutorService  

JDK的定时任务也是一个线程池ScheduledThreadPoolExecutor，和普通的线程池不同在于其使用的是DelayedWorkQueue，这是一个配了Lock和Condition的优先级队列。take元素时，检查最近的任务，如果时间到了就返回，否则就等在Condition上，超时时间为最近任务还剩余的时间。在task的run()方法中(用户的runnable会再经过包装)，如果是周期性的任务，那么运行之后，根据策略设置好下次的超时时间，然后会再将自己放入Queue中。  
和Netty相比，大体上类似，都是使用堆来管理。  
scheduleAtFixedRate()方法按照理想时间为initialDelay, initialDelay + period, initialDelay + 2 * period...
scheduleWithFixedDelay()方法则是保证两次实际运行期间的间隔为Delay。  

## PriorityBlockingQueue & DelayQueue & DelayedWorkQueue  

PriorityBlockingQueue就是加了Lock的PriorityQueue，逻辑和PriorityQueue基本上一样，除了扩容的时候，先解锁在CAS标志位创建新数组，然后加锁复制。创建新数组时不包括在lock()中(复制数据时肯定得加lock)，猜测是因为new一个数组可能比较费时间，先释放掉Lock，期待其它线程可以在这个期间poll()一些数据，poll()之后就有空位了其他线程的offer()也可以执行，增加并行度。  

DelayQueue内部持有一把Lock来保护一个PriorityQueue其中的元素都是Delayed类型。和一般的PriorityQueue不同在于，在take()时，不仅要有数据，还需要满足第一个元素的延时要求。内部有一个Thread类型的leader字段，表示第一个等在第一个元素上的Thread，如果take时发现不能时间还没到，并且leader为null，就把leader置为自己，然后将延时作为await的超时时间；如果发现已经有人等在第一个元素上了，就把自己无超时的await()。当等在第一个元素上的Thread取完之后会唤醒一个等待的Thread。  

DelayedWorkQueue也是堆结构，类似上边两种，但是特殊一些。内部的Queue只能持有RunnableScheduledFuture<?>类型的元素，一般的ScheduledExecutorService在submit一个task时会先将其包装成ScheduledFutureTask(实现了RunnableScheduledFuture接口)，其中很重要的一个特点是它保存了自己在heap中的索引下标，这样在remove时原本需要线性遍历删除再调整o(n+logn)，现在直接能定位只需要o(logn)。由于每次siftUp或者siftDown都需要修改对应的元素内部保存的下标字段，所以不是很容易复用DelayQueue或是PriorityBlockingQueue。  

## Random & ThreadLocalRandom  
  
Random本身就是线程安全的，但是多线程下有竞争，有性能瓶颈。Thread类存储了ThreadLocalRandom需要的变量，这样避免了同步操作。  

## FutureTask  

FutureTask实现了RunnableFuture接口。内部有一个Callable字段，真正需要运行的任务就是这个Callable。内部还有Thread字段，表示这个运行这个Callbale的线程。outcome字段表示运行的结果。  
它的run()方法为，首先通过CAS将Thread字段设为当前Thread(猜测为了线程安全吧，防止错误使用)，然后运行callable拿到result，如果成功就把outcome字段置为result，失败将outcome置为exception。然后unpark()所有等在这个task上的线程。  
get()方法首先判断state是不是完成了，完成了返回outcome，没完成粗略的来说就会把自己包装成一个Node(链表形式)，然后park()自己。  
cancel(mayInterrupted)方法会尝试interrupt执行这个task的线程。  

## CompletableFuture  

CompletableFuture实现的功能很多，其中CompletableFuture#whenComplete(BiConsumer<? super T, ? super Throwable> action)如cf.whenCompete((value, throwable) -> {})方法是由complete这个cf的线程运行的，也可以使用whenCompleteAsync()改成异步的。  
