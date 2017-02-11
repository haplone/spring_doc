# java 集合小结

java集合主要有3方面：不能重复的set、线性存储的列表、key-value结构的map。

其中set、list、queue都继承Collection接口。

## set

set的元素不能重复，也不保障顺序，Null只能有一个。
所有Set几乎都是内部用一个Map来实现, 因为Map里的KeySet就是一个Set，而value是假值，全部使用同一个Object即可。
Set的特征也继承了那些内部的Map实现的特征。


set的有这么几个典型类：HashSet、TreeSet、ConcurrentSkipListSet、CopyOnWriteArraySet、LinkedHashSet。

### HashSet
主要用于快速查找，平时会用于数据去重。存入其中的对象必须定义hashCode()api。比较对象是否是重复是先比较hash code，如果一致才使用equals。
HashSet：内部是HashMap。

### TreeSet
可以实现排序，底层实现是树结构。可以使用提供的Comparator进行排序。
TreeSet：内部是TreeMap的SortedSet。

### LinkedHashSet
使用链表实现，查询速度与HashSet相同。遍历顺序是插入顺序。
内部是LinkedHashMap。


### ConcurrentSkipListSet
内部是ConcurrentSkipListMap的并发优化的SortedSet。

### CopyOnWriteArraySet：内部是CopyOnWriteArrayList的并发优化的Set，利用其addIfAbsent（）方法实现元素去重，如前所述该方法的性能很一般。

### ConcurrentHashSet，本来也该有一个内部用ConcurrentHashMap的简单实现，但JDK偏偏没提供。Jetty就自己简单封了一个，Guava则直接用java.util.Collections.newSetFromMap（new ConcurrentHashMap（）） 实现。


## List

list主要是随机存储的ArrayList和链表实现的LinkedList。另外还Vector和Stack。
在Collection接口iterator基础上添加了双向遍历的ListIterator，并且遍历时可修改。在迭代时，如果有对元素进行修改，会fast-fail，其实现原理是通过对modCount校验实现。

### ArrayList
以数组实现。节约空间，但数组有容量限制。超出限制时会增加50%容量，用System.arraycopy（）复制到新的数组。因此最好能给出数组大小的预估值。默认第一次插入元素时创建大小为10的数组。
使用ensureCapacity保障容量
Collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类


### LinkedList
以双向链表实现。链表无容量限制，但双向链表本身使用了更多空间，每插入一个元素都要构造一个额外的Node对象，也需要额外的链表指针操作。
插入、删除元素时修改前后节点的指针即可，不再需要复制移动。但还是要部分遍历链表的指针才能移动到下标所指的位置。
实现Deque接口

Apache Commons 有个TreeNodeList，里面是棵二叉树，可以快速移动指针到位。

### CopyOnWriteArrayList
并发优化的ArrayList。基于不可变对象策略，在修改时先复制出一个数组快照来修改，改好了，再让内部指针指向新数组。

因为对快照的修改对读操作来说不可见，所以读读之间不互斥，读写之间也不互斥，只有写写之间要加锁互斥。但复制快照的成本昂贵，典型的适合读多写少的场景。比如listeners/observers集合。

虽然增加了addIfAbsent（e）方法，会遍历数组来检查元素是否已存在，性能可想像的不会太好。






线程安全

AbstractList

AbstractSequentialList


## Map

Map的实现类主要有HashMap、TreeMap、LinkedHashMap、WeakHashMap、IdentityHashMap。

### HashMap
以Entry[]数组实现的哈希桶数组，用Key的哈希值取模桶数组的大小可得到数组下标。

插入元素时，如果两条Key落在同一个桶（比如哈希值1和17取模16后都属于第一个哈希桶），我们称之为哈希冲突。

JDK的做法是链表法，Entry用一个next属性实现多个Entry以单向链表存放。查找哈希值为17的key时，先定位到哈希桶，然后链表遍历桶里所有元素，逐个比较其Hash值然后key值。

在JDK8里，新增默认为8的阈值，当一个桶里的Entry超过閥值，就不以单向链表而以红黑树来存放以加快Key的查找速度。

当然，最好还是桶里只有一个元素，不用去比较。所以默认当Entry数量达到桶数量的75%时，哈希冲突已比较严重，就会成倍扩容桶数组，并重新分配所有原来的Entry。扩容成本不低，所以也最好有个预估值。

取模用与操作（hash & （arrayLength-1））会比较快，所以数组的大小永远是2的N次方， 你随便给一个初始值比如17会转为32。默认第一次放入元素时的初始值是16。

iterator（）时顺着哈希桶数组来遍历，看起来是个乱序。



### LinkedHashMap
类似于HashMap，但是迭代遍历它时，取得“键值对”的顺序是其插入次序，或者是最近最少使用(LRU)的次序。只比HashMap慢一点。而在迭代访问时发而更快，因为它使用链表维护内部次序。
扩展HashMap，每个Entry增加双向链表，号称是最占内存的数据结构。

支持iterator（）时按Entry的插入顺序来排序（如果设置accessOrder属性为true，则所有读写访问都排序）。

插入时，Entry把自己加到Header Entry的前面去。如果所有读写访问都要排序，还要把前后Entry的before/after拼接起来以在链表中删除掉自己，所以此时读操作也是线程不安全的了。

### TreeMap
 基于红黑树数据结构的实现。查看“键”或“键值对”时，它们会被排序(次序由Comparabel或Comparator决定)。TreeMap的特点在于，你得到的结果是经过排序的。TreeMap是唯一的带有subMap()方法的Map，它可以返回一个子树。

以红黑树实现，红黑树又叫自平衡二叉树：

对于任一节点而言，其到叶节点的每一条路径都包含相同数目的黑结点。
上面的规定，使得树的层数不会差的太远，使得所有操作的复杂度不超过 O（lgn），但也使得插入，修改时要复杂的左旋右旋来保持树的平衡。

支持iterator（）时按Key值排序，可按实现了Comparable接口的Key的升序排序，或由传入的Comparator控制。可想象的，在树上插入/删除元素的代价一定比HashMap的大。

支持SortedMap接口，如firstKey（），lastKey（）取得最大最小的key，或sub（fromKey, toKey）, tailMap（fromKey）剪取Map的某一段。

### EnumMap
枚举类型作为键值的Map。因为键的数量相对固定，所以在内部用一个数组储存对应值。通常来说，效率要高于HashMap。
EnumMap的原理是，在构造函数里要传入枚举类，那它就构建一个与枚举的所有值等大的数组，按Enum. ordinal（）下标来访问数组。性能与内存占用俱佳。

美中不足的是，因为要实现Map接口，而 V get（Object key）中key是Object而不是泛型K，所以安全起见，EnumMap每次访问都要先对Key进行类型判断，在JMC里录得不低的采样命中频率。



### WeakHashMap
弱键(weak key)Map，Map中使用的对象也被允许释放: 这是为解决特殊问题设计的。如果没有map之外的引用指向某个“键”，则此“键”可以被垃圾收集器回收。
WeakHashMap：这种Map通常用在数据缓存中。它将键存储在WeakReference中，就是说，如果没有强引用指向键对象的话，这些键就可以被垃圾回收线程回收。值被保存在强引用中。因此，你要确保没有引用从值指向键或者将值也保存在弱引用中m.put(key, new WeakReference(value))。


### IdentifyHashMap
使用==代替equals()对“键”作比较的hash map。专为解决特殊问题而设计。
IdentityHashMap：这是一个特殊的Map版本，它违背了一般Map的规则：它使用 “==” 来比较引用而不是调用Object.equals来判断相等。这个特性使得此集合在遍历图表的算法中非常实用——可以方便地在IdentityHashMap中存储处理过的节点以及相关的数据。

### ConcurrentHashMap
get操作全并发访问，put操作可配置并发操作的哈希表。并发的级别可以通过构造函数中concurrencyLevel参数设置（默认级别16）。该参数会在Map内部划分一些分区。在put操作的时候只有只有更新的分区是锁住的。这种Map不是代替HashMap的线程安全版本——任何 get-then-put的操作都需要在外部进行同步。
ConcurrentSkipListMap：基于跳跃列表（Skip List）的ConcurrentNavigableMap实现。本质上这种集合可以当做一种TreeMap的线程安全版本来使用。

并发优化的HashMap。

在JDK5里的经典设计，默认16把写锁（可以设置更多），有效分散了阻塞的概率。数据结构为Segment[]，每个Segment一把锁。Segment里面才是哈希桶数组。Key先算出它在哪个Segment里，再去算它在哪个哈希桶里。

也没有读锁，因为put/remove动作是个原子动作（比如put的整个过程是一个对数组元素/Entry 指针的赋值操作），读操作不会看到一个更新动作的中间状态。

但在JDK8里，Segment[]的设计被抛弃了，改为精心设计的，只在需要锁的时候加锁。

支持ConcurrentMap接口，如putIfAbsent（key，value）与相反的replace（key，value）与以及实现CAS的replace（key, oldValue, newValue）。


### ConcurrentSkipListMap
JDK6新增的并发优化的SortedMap，以SkipList结构实现。Concurrent包选用它是因为它支持基于CAS的无锁算法，而红黑树则没有好的无锁算法。

原理上，可以想象为多个链表组成的N层楼，其中的元素从稀疏到密集，每个元素有往右与往下的指针。从第一层楼开始遍历，如果右端的值比期望的大，那就往下走一层，继续往前走。

 典型的空间换时间。每次插入，都要决定在哪几层插入，同时，要决定要不要多盖一层楼。

 它的size（）同样不能随便调，会遍历来统计。

## Queue

Queue是在两端出入的List，所以也可以用数组或链表来实现。


队列是一种数据结构．它有两个基本操作：在队列尾部加人一个元素，和从队列头部移除一个元素就是说，队列以一种先进先出的方式管理数据，如果你试图向一个 已经满了的阻塞队列中添加一个元素或者是从一个空的阻塞队列中移除一个元索，将导致线程阻塞．在多线程进行合作时，阻塞队列是很有用的工具。工作者线程可 以定期地把中间结果存到阻塞队列中而其他工作者线线程把中间结果取出并在将来修改它们。队列会自动平衡负载。如果第一个线程集运行得比第二个慢，则第二个 线程集在等待结果时就会阻塞。如果第一个线程集运行得快，那么它将等待第二个线程集赶上来。

### 普通队列

#### LinkedList
是的，以双向链表实现的LinkedList既是List，也是Queue。

#### ArrayDeque
以循环数组实现的双向Queue。大小是2的倍数，默认是16。

为了支持FIFO，即从数组尾压入元素（快），从数组头取出元素（超慢），就不能再使用普通ArrayList的实现了，改为使用循环数组。

有队头队尾两个下标：弹出元素时，队头下标递增；加入元素时，队尾下标递增。如果加入元素时已到数组空间的末尾，则将元素赋值到数组[0]，同时队尾下标指向0，再插入下一个元素则赋值到数组[1]，队尾下标指向1。如果队尾的下标追上队头，说明数组所有空间已用完，进行双倍的数组扩容。

#### PriorityQueue
用平衡二叉最小堆实现的优先级队列，不再是FIFO，而是按元素实现的Comparable接口或传入Comparator的比较结果来出队，数值越小，优先级越高，越先出队。但是注意其iterator（）的返回不会排序。

平衡最小二叉堆，用一个简单的数组即可表达，可以快速寻址，没有指针什么的。最小的在queue[0] ，比如queue[4]的两个孩子，会在queue[2*4+1] 和 queue[2*（4+1）]，即queue[9]和queue[10]。

入队时，插入queue[size]，然后二叉地往上比较调整堆。

出队时，弹出queue[0]，然后把queque[size]拿出来二叉地往下比较调整堆。

初始大小为11，空间不够时自动50%扩容。

### 线程安全的队列

#### ConcurrentLinkedQueue/Deque
无界的并发优化的Queue，基于链表，实现了依赖于CAS的无锁算法。

ConcurrentLinkedQueue的结构是单向链表和head/tail两个指针，因为入队时需要修改队尾元素的next指针，以及修改tail指向新入队的元素两个CAS动作无法原子，所以需要的特殊的算法。

### 线程安全的阻塞队列
BlockingQueue，一来如果队列已空不用重复的查看是否有新数据而会阻塞在那里，二来队列的长度受限，用以保证生产者与消费者的速度不会相差太远。当入队时队列已满，或出队时队列已空，不同函数的效果见下表：

#### ArrayBlockingQueue
定长的并发优化的BlockingQueue，也是基于循环数组实现。有一把公共的锁与notFull、notEmpty两个Condition管理队列满或空时的阻塞状态。

#### LinkedBlockingQueue/Deque
可选定长的并发优化的BlockingQueue，基于链表实现，所以可以把长度设为Integer.MAX_VALUE成为无界无等待的。

利用链表的特征，分离了takeLock与putLock两把锁，继续用notEmpty、notFull管理队列满或空时的阻塞状态。

#### PriorityBlockingQueue
无界的PriorityQueue，也是基于数组存储的二叉堆（见前）。一把公共的锁实现线程安全。因为无界，空间不够时会自动扩容，所以入列时不会锁，出列为空时才会锁。

#### DelayQueue
 内部包含一个PriorityQueue，同样是无界的，同样是出列时才会锁。一把公共的锁实现线程安全。元素需实现Delayed接口，每次调用时需返回当前离触发时间还有多久，小于0表示该触发了。

 pull（）时会用peek（）查看队头的元素，检查是否到达触发时间。ScheduledThreadPoolExecutor用了类似的结构。

### 同步队列

#### SynchronousQueue
同步队列本身无容量，放入元素时，比如等待元素被另一条线程的消费者取走再返回。JDK线程池里用它。


ArrayBlockingQueue在构造时需要指定容量，之后不能修改。 并可以选择是否需要公平性，如果公平参数被设置true，等待时间最长的线程会优先得到处理（其实就是通过将ReentrantLock设置为true来 达到这种公平性的：即等待时间最长的线程会先操作）。通常，公平性会使你在性能上付出代价，只有在的确非常需要的时候再使用它。它是基于数组的阻塞循环队 列，此队列按 FIFO（先进先出）原则对元素进行排序。


ConcurrentLinkedDeque / ConcurrentLinkedQueue：基于链表实现的无界队列，添加元素不会堵塞。但是这就要求这个集合的消费者工作速度至少要和生产这一样快，不然内存就会耗尽。严重依赖于CAS(compare-and-set)操作。

DelayQueue：无界的保存Delayed元素的集合。元素只有在延时已经过期的时候才能被取出。队列的第一个元素延期最小（包含负值——延时已经过期）。当你要实现一个延期任务的队列的时候使用（不要自己手动实现——使用ScheduledThreadPoolExecutor）。
DelayQueue（基于PriorityQueue来实现的）是一个存放Delayed 元素的无界阻塞队列，只有在延迟期满时才能从中提取元素。该队列的头部是延迟期满后保存时间最长的 Delayed 元素。如果延迟都还没有期满，则队列没有头部，并且poll将返回null。当一个元素的 getDelay(TimeUnit.NANOSECONDS) 方法返回一个小于或等于零的值时，则出现期满，poll就以移除这个元素了。此队列不允许使用 null 元素。 

LinkedBlockingDeque / LinkedBlockingQueue：可选择有界或者无界基于链表的实现。在队列为空或者满的情况下使用ReentrantLock-s。此队列按 FIFO（先进先出）排序元素。

LinkedTransferQueue：基于链表的无界队列。除了通常的队列操作，它还有一系列的transfer方法，可以让生产者直接给等待的消费者传递信息，这样就不用将元素存储到队列中了。这是一个基于CAS操作的无锁集合。

PriorityBlockingQueue：PriorityQueue的无界的版本。
PriorityBlockingQueue是一个带优先级的 队列，而不是先进先出队列。元素按优先级顺序被移除，该队列也没有上限（看了一下源码，PriorityBlockingQueue是对 PriorityQueue的再次包装，是基于堆数据结构的，而PriorityQueue是没有容量限制的，与ArrayList一样，所以在优先阻塞 队列上put时是不会受阻的。虽然此队列逻辑上是无界的，但是由于资源被耗尽，所以试图执行添加操作可能会导致 OutOfMemoryError），但是如果队列为空，那么取元素的操作take就会阻塞，所以它的检索操作take是受阻的。另外，往入该队列中的元 素要具有比较能力。

SynchronousQueue：一个有界队列，其中没有任何内存容量。这就意味着任何插入操作必须等到响应的取出操作才能执行，反之亦反。如果不需要Queue接口的话，通过Exchanger类也能完成响应的功能。


java.ulil.concurrent包提供了阻塞队列的4个变种。默认情况下，






