# Java 集合框架

Java的所有集合框架可以大致分为**Collection（集合，存储一个元素集合）**与**Map（图，存储键值对映射）**两种。
前者又可分为List、Set、Queue三大部分。比较常见的几种有ArrayList、LinkedList、HashSet、LinkedHashSet、HashMap、HashTable等。

## Collection接口
上述的所有集合类都直接或间接实现了Collection接口，而collection接口提供了一组用于**集合操作的方法**。

## List接口
由Collection接口扩展来，允许一个**有重复值的有序集合**。增加了**面向位置的操作**，还提供了**双向集合迭代器**。

常用实现类：
**ArrayList**：使用**单纯的动态增长数组**来存储集合数据。由于数组使用的是一片连续的内存空间，所有其可以很快的进行查询查找，但其插入与删除操作在发生在数组内部时，效率是很低的。一般使用数组需要预留空间，而且数组不利于扩展。
**LinkedList**：采用**链表的形式来完成**集合数据的存储。链表中的元素可存在与内存的任何地方，不要求连续。由于链表的上下顺序关系只在父子间传递，所有其查找效率较数组要低，但其插入和删除效率较高。不用指定大小，利于扩展。

## Set接口
也扩展自Collection接口。与List不同的是，Set提供的是一个不允许重复值的元素集合。
常见实现类：
**HashSet**：使用HashMap来提供了一个允许空元素的，不允许重复元素的无序集合。其具体实现是利用了HashMap。将add的元素当成所插入HashMap的Key，使用一个默认定义好的对象作为插入HashMap的Value。这样当插入两个相同的对象时，对于HashMap来说，其hash值一样，插入的Key一样，所以会返回此Entry的oldValue。此时HashSet就可以根据HashMap的put方法的返回值是否为null来判断是否插入了重复值。
**LinkedHashSet**：存储过程和HashSet类似，但HashSet借助的是HashMap，LinkedHashSet借助的是LinkedHashMap。而LinkedHashMap与HashMap的区别就在于LinkedHashMap除了由和HashMap一样的存储结构外，还引入了一个具有一定顺序的链表，使LinkedHashSet既可以按照**插入顺序**也可以按照**访问顺序**来访问节点。这里的LinkedHashSet就是借助了LinkedHashMap的插入顺序的访问。
LinkedHashMap复写了HashMap的一系列方法，如newNode()、afterNodeAccess()等方法，就是为了在创建、访问、修改、替换节点的时候，能将处理过的节点重新在链表中排序。
**TreeSet**：底层借助的是TreeMap。其存储过程本质上和上面两个没有啥区别，也是借助一个虚拟的value来讲需要插入的值当成key存储进去，所以插入的值都是不允许重复值的，而实际的存储结构确截然不同。TreeMap是一个由红黑树构成的存储结构，他的一切操作都遵循红黑树的处理。TreeSet和TreeMap一样，可以在构造函数中传入一个自定义的比较器来取代插入元素（Key）的缺省比较。TreeSet还提供了一系列方法：lower()、greater()、floor()、ceiling()用来快速查找比给定元素大的最小或相等元素、比给定元素小的最大或相等元素。

## Queue接口
Queue接口提供了一个先进先出的数据结构集合，数据在队尾添加，在队首删除。
Queue提供了一系列方法：
add/offer：两者都是向队列中添加一个元素，不同的是如果队列的长度有限制，则在队列满队的时候add方法将抛出一个unchecked异常，而offer会返回一个false。
poll/remove：两种都是从队首移除一个元素。不同的是，当对列为空时，poll将返回null，而remove将抛出一个异常。
peek/element：查询队首元素。不同的时，当队列为空时，peek将返回null，而element将抛出异常。

Deque接口：继承自Queue接口，提供了一个可以在队首队尾两端进行插入和删除的双向队列。
常见实现类：
**LinkedList**：实现了Deque接口，所以可以用LinkedList来自定义一个基本队列。
**PriorityQueue**：java提供的优先队列。继承与AsbtractQueue。其构成就是一个小顶堆，对堆中任意节点，其左右子树上的节点恒比其大。对于堆中的层序遍历序号的特殊性，我们可以采用数组来对对进行一个存储。对于插入操作，其采用向上调整方法，将新插入的节点丢在队尾，然后与其父节点进行大小比较和位置调整（当其比父节点小时，将父节点下移，然后比较节点替换为当前父节点的父节点）。对于移除队首操作采用的是向下调整，首先将队尾元素取出来替换队首元素，然后将新的队首元素向下比较和位置调整（首先取其两个孩子节点中较小的节点min进行比较，若其比min节点大，则将min节点上移，新的队首节点继续向原min节点的子树上进行比较）。这里可以看出，经过每一次插入和删除操作，PriorityQueue队列都无法对数组中的元素顺序进行保证。PriorityQueue可以自定义比较器，来判断优先级的高低。
**PriorityBlockingQueue**：针对PriorityQueue的线程不安全，引入了ReentrantLock进行了一个加锁操作。其他此类的操作都与PriorityQueue相似。

## Map接口
提供了一种存储键值对映射的容器类，Map中键可以为任意类型的对象，但不允许重复的键（允许键值为null），每一个键对应一个值，存储的结构是键值与其他信息组成的条目对象。
常见实现类：
**HashMap**：基于哈希表的Map接口的实现类，属于非线程安全类。内部使用了数组+链表的结构（在Java8引入了红黑树结构来替换链表）来进行存储。其内部的数组table可以看成一系列的链表头构成的数组（或者红黑树树根）。HashMap进行存储时，首先会计算Key的hash值，这里的hash值并不是普通的直接拿到Key对象的hashCode，而是会对其进行一个特殊处理--进行一个高低位的与运算，这样做的原因是能够将高位也加入之后的hash散列运算中（避免低位相同，高位不同的对象被错误的归类到同一个hash链表中）。同时，为了能够均匀的进行哈希分布，一般都是取hash值对table数组的长度就像一个取模操作。但由于取模运算的代价太高，所以HashMap对table数组做了一个限制，使其长度必为2的次方倍数，这样对table.length - 1进行按位与操作得到的结果就与取模操作等价了。HashMap也有一个存储阈值，其是根据装载因子和最大容量来相乘获得的。在put新的键值对进入HashMap时，HashMap会对其容量进行一个监测，如果大于阈值，则会进行一个扩容。默认情况下会将存储阈值、最大容量（table数组长度）都进行一个翻倍操作。然后按照新的table数组进行一个resize操作。
对与java7的resize操作，首先会对table数组进行一个新的扩容，然后再将原table数组中的链表中存储的Entry进行rehash操作，放入新的table数组所管理的链表中，并且再rehash时发生了hash冲突时，会将新加入的Entry放到对应链表的链首。
在Java8中对这一部分进行了优化：在进行resize操作时，由于是对table数组进行一个翻倍操作，原数组上的链表中对应的Entry在新数组上的位置其实时由据可循的。由于length的长度增加了一倍，在进行与运算时，相当于增加了一位可进行计算的位，这位可能的值只能为0或1，所以得到的新的table index只可能时原index或者原index + 原table.length，所以我们可以根据这个依据来减少掉Java7的rehash操作。这里在Java8中根据原Entry的hash值与原length进行与操作，若等于0，则此Entry在新table数组上依旧是原index，当不为0，则是原index+原table.length。这里就将分为两个链表lo和hi来对这两种情况进行处理。同时，这里发生hash冲突的情况也不是和Java7一致，将新插入的Entry插入链首，而是插入链尾。
其次，Java8对HashMap的插入时也做了一个优化，当某个链表的节点长度大于8时，将会将此链表转化为一棵红黑树，已优化访问速度。
对于HashMap的线程安全性来说，不安全的。因为在多线程操作下，可能会有两个或多个线程在对HashMap进行插入操作，此时可能两个线程都触发了resize操作。而在某种特殊情况下会形成Entry链表的死循环。

**LinkedHashMap**：使用哈希表以及链表来的Map接口实现类。其与HashMap不同的地方是在于比HashMap多提供了一条双向链表来链接其中所有的Entry，这个链表的访问顺序也是迭代器访问的顺序（在访问顺序下使用keyset迭代器访问会报错）。LinkedHashMap是如何将所有Entry链入这个双向链表中的（或者从这个链表中移除的）？LinkedHashMap通过重写了HashMap的一些列函数，如newNode()、afterNodeRemoval()等方法，这些方法会在插入新节点或者删除节点时，将修改的节点放入链表的尾部、或将删除的节点从链表中移除。除此之外，LinkedHashMap还可以有两种访问模式：插入顺序和访问顺序。
对于插入顺序，LinkedHashmap对于上述的双向链表的Entry顺序的改变只发生在上述的newNode()和afterNodeRemoval()方法中，插入同一个key或者replace值都是不会对链表中的Entry顺序进行修改的。而当是Access-Order（访问模式）时，此时LinkedHashMap的accessOrder为true，此时对上述的链表中的Entry的操作除了发生在刚刚说的两种情况下，也会发生在访问任意Entry的时候，如给一个已有的key设置新的value，replace操作或者简单的get操作都会将当前访问的Entry放到链尾，这种情形下，LinkedHashMap就相当于一个LRU（最近最久未使用链表）。在引入了双向链表后，LinkedHashMap的访问顺序就和HashMap的访问时乱序不同，这个为某些对顺序有要求的情况提供了代替HashMap的工具，不过由于引入了这个链表，所有LinkedHashMap的开销与内存占用也高于HashMap。

**TreeMap**：其构成相当于一颗红黑树。红黑树即是二叉排序树的一种变体，满足二叉排序树的树中任意节点的左子树上所有节点都恒比根节点小，右子树上的节点恒比根节点大。此外还有五大特性：
1、所有节点都是由颜色的（红色或黑色）。
2、根节点是黑色的，新插入的节点再调整之前是红色的。
3、所有叶子节点都为黑色的NIL节点。此外所有节点都必须由有两个子节点，其中可能有一个以上的NIL节点。NIl节点作为子树路径结束的标志。
4、在任意子树路径上，不会存在两个连续的红色节点。也就是红色节点的子节点必须为黑色。
5、任意一个节点到其NIL叶子节点的所有路径经过的黑色节点个数都是相同的。

这里的第四条也从侧面说明了某个节点到其NIL节点的最长路径和最短路径相差最多为两倍。这样就可以完全避免二叉排序树在插入一组有序数据时产生的单向子树情形（最差情况）。

**ConcurrentHashMap**：
