# java 集合小结

java集合主要有3方面：不能重复的set、线性存储的列表、key-value结构的map。

其中set、list、queue都继承Collection接口。

## set

set的元素不能重复，也不保障顺序，Null只能有一个。

set的有这么几个典型类：HashSet、TreeSet、ConcurrentSkipListSet、CopyOnWriteArraySet、LinkedHashSet。

HashSet 主要用于快速查找，平时会用于数据去重。存入其中的对象必须定义hashCode()api。比较对象是否是重复是先比较hash code，如果一致才使用equals。

TreeSet 可以实现排序，底层实现是树结构。可以使用提供的Comparator进行排序。

LinkedHashSet 使用链表实现，查询速度与HashSet相同。遍历顺序是插入顺序。

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

HashMap就是使用对象的hashCode()进行快速查询的。此方法能够显着提高性能。

HashMap : Map基于散列表的实现。插入和查询“键值对”的开销是固定的。可以通过构造器设置容量capacity和负载因子load factor，以调整容器的性能。
性能问题主要来自碰撞的概率，同时需要平衡空间capacity。

LinkedHashMap : 类似于HashMap，但是迭代遍历它时，取得“键值对”的顺序是其插入次序，或者是最近最少使用(LRU)的次序。只比HashMap慢一点。而在迭代访问时发而更快，因为它使用链表维护内部次序。

TreeMap : 基于红黑树数据结构的实现。查看“键”或“键值对”时，它们会被排序(次序由Comparabel或Comparator决定)。TreeMap的特点在于，你得到的结果是经过排序的。TreeMap是唯一的带有subMap()方法的Map，它可以返回一个子树。

EnumMap：枚举类型作为键值的Map。因为键的数量相对固定，所以在内部用一个数组储存对应值。通常来说，效率要高于HashMap。


WeakHashMap: 弱键(weak key)Map，Map中使用的对象也被允许释放: 这是为解决特殊问题设计的。如果没有map之外的引用指向某个“键”，则此“键”可以被垃圾收集器回收。
WeakHashMap：这种Map通常用在数据缓存中。它将键存储在WeakReference中，就是说，如果没有强引用指向键对象的话，这些键就可以被垃圾回收线程回收。值被保存在强引用中。因此，你要确保没有引用从值指向键或者将值也保存在弱引用中m.put(key, new WeakReference(value))。


IdentifyHashMap : 使用==代替equals()对“键”作比较的hash map。专为解决特殊问题而设计。
IdentityHashMap：这是一个特殊的Map版本，它违背了一般Map的规则：它使用 “==” 来比较引用而不是调用Object.equals来判断相等。这个特性使得此集合在遍历图表的算法中非常实用——可以方便地在IdentityHashMap中存储处理过的节点以及相关的数据。

ConcurrentHashMap：get操作全并发访问，put操作可配置并发操作的哈希表。并发的级别可以通过构造函数中concurrencyLevel参数设置（默认级别16）。该参数会在Map内部划分一些分区。在put操作的时候只有只有更新的分区是锁住的。这种Map不是代替HashMap的线程安全版本——任何 get-then-put的操作都需要在外部进行同步。
ConcurrentSkipListMap：基于跳跃列表（Skip List）的ConcurrentNavigableMap实现。本质上这种集合可以当做一种TreeMap的线程安全版本来使用。


## Queue


队列是一种数据结构．它有两个基本操作：在队列尾部加人一个元素，和从队列头部移除一个元素就是说，队列以一种先进先出的方式管理数据，如果你试图向一个 已经满了的阻塞队列中添加一个元素或者是从一个空的阻塞队列中移除一个元索，将导致线程阻塞．在多线程进行合作时，阻塞队列是很有用的工具。工作者线程可 以定期地把中间结果存到阻塞队列中而其他工作者线线程把中间结果取出并在将来修改它们。队列会自动平衡负载。如果第一个线程集运行得比第二个慢，则第二个 线程集在等待结果时就会阻塞。如果第一个线程集运行得快，那么它将等待第二个线程集赶上来。

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






