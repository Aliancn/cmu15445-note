讲座#09:索引并发控制15-445/645数据库系统(春季2023)https://15445.courses.cs.cmu.edu/spring2023/卡内基梅隆大学查理加罗德
1索引并发控制
到目前为止，我们假设所讨论的数据结构是单线程的。但是，大多数dbms需要允许多个线程安全地访问数据结构，以利用额外的CPU内核并隐藏磁盘I/O停顿。
并发控制协议是DBMS用来确保共享对象上并发操作的“正确”结果的方法。
协议的正确性标准可以改变:•逻辑正确性:这意味着线程能够读取它应该读取的值，例如一个线程应该回读它之前写的值。
物理正确性:这意味着对象的内部表示是合理的，例如，数据结构中没有指针会导致线程读取无效的内存位置。
出于本讲座的目的，我们只关心强制物理正确性。我们将在以后的讲座中重新讨论逻辑正确性
2锁与闩锁
在讨论DBMS如何保护其内部元素时，锁和闩锁之间有一个重要的区别。
锁锁是一种高级的逻辑原语，用于保护数据库(如元组、表、数据库)的内容不受其他事务的影响。事务将在其整个持续时间内持有锁。数据库系统可以向用户公开查询运行时持有的锁。锁需要能够回滚更改。
锁存器锁存器是低级保护原语，用于保护DBMS内部数据结构(例如，数据结构，内存区域)的关键部分免受其他线程的影响。锁存仅在操作进行期间保持。锁存器不需要能够回滚更改。锁存器有两种模式:•READ:允许多个线程同时读取同一项。一个线程可以在读模式下获取锁存，即使另一个线程已经在读模式下获取了锁存。
•WRITE:只允许一个线程访问该条目。如果另一个线程以任何模式持有写锁存，则该线程无法获得写锁存。持有写锁存的线程也会阻止其他线程获取读锁存。
3闩锁实现
用于实现闩锁的底层原语是通过现代cpu提供的原子比较与交换(CAS)指令实现的。这样，线程就可以检查内存位置的内容，看看它是否有某个值。如果是，那么CPU将用新值交换旧值。否则，内存位置保持不变。
在DBMS中实现闩锁有几种方法。每种方法在工程复杂性和运行时性能方面都有不同的权衡。这些测试和设置步骤是自动执行的(即，没有其他线程可以在测试和设置步骤之间更新值)。
锁存器的一种可能实现是操作系统内置的互斥锁基础设施。Linux提供了futex(快速用户空间互斥锁)，它由(1)用户空间中的自旋锁存器和(2)操作系统级互斥锁组成。如果DBMS能够获得用户空间锁存，则设置锁存。尽管它包含两个内部锁存器，但它对DBMS显示为单个锁存器。如果DBMS无法获取用户空间锁存器，那么它将进入内核，并尝试获取一个更昂贵的互斥锁。如果DBMS无法获得第二个互斥锁，那么线程通知操作系统它在互斥锁上被阻塞，然后它被重新调度。
操作系统互斥锁在dbms中通常不是一个好主意，因为它是由操作系统管理的，并且开销很大。
优点:使用简单，在DBMS中不需要额外的编码。
缺点:由于操作系统调度，成本高且不可扩展(每次锁/解锁调用约25 ns)。
自旋锁存器(Test-and-Set Spin Latch, TAS)自旋锁存器是操作系统互斥锁的一种更有效的替代方案，因为它由dbms控制。spin latch本质上是线程试图更新的内存位置(例如，将布尔值设置为true)。线程执行CAS以尝试更新内存位置。DBMS可以控制如果无法获得锁存会发生什么。它可以选择再次尝试(例如，使用while循环)或允许操作系统重新安排它。
因此，这种方法给了DBMS比操作系统互斥锁更多的控制权，在操作系统中，无法获得锁存将控制权交给操作系统。
•优点:Latch/unlatch操作效率高(单指令锁定/解锁)。
缺点:不可扩展，也不支持缓存，因为使用多线程时，CAS指令将在不同的线程中多次执行。这些浪费的指令将在高争用环境中堆积起来;对于操作系统来说，线程看起来很忙，即使它们没有做有用的工作。
这将导致缓存一致性问题，因为线程正在轮询其他cpu上的缓存行。
互斥锁和自旋锁不区分读/写(也就是说，它们不支持不同的模式)。DBMS需要一种允许并发读取的方法，所以如果应用程序有大量的读取，它将有更好的性能，因为读取器可以共享资源而不是等待。
读写锁存器允许锁存器保持在读或写模式。它跟踪在每种模式下持有锁存并等待获取锁存的线程的数量。读写锁存器使用前两种锁存器实现中的一种作它们是每种模式下闩锁的队列请求。不同的dbms可以有不同的策略来处理队列。
•示例:std::共享互斥锁•优点:允许并发读。
缺点:DBMS必须管理读/写队列以避免饥饿。由于额外的元数据，比spin latch的存储开销更大。
为原语，并具有额外的逻辑来处理读写队列。
哈希表锁存
由于线程访问数据结构的方式有限，在静态哈希表中支持并发访问很容易。例如，当从一个槽移动到另一个槽时，所有线程都沿着相同的方向移动(即自顶向下)。线程一次也只能访问一个页面/插槽。因此，在这种情况下不可能出现死锁，因为没有两个线程可以竞争另一个线程持有的锁存。当我们需要调整表的大小时，我们可以在整个表上使用全局锁存器来执行操作。
动态散列方案(例如，可扩展)中的锁存是一个更复杂的方案，因为有更多的共享状态需要更新，但一般方法是相同的。
在哈希表中有两种支持锁存的方法，它们在锁存的粒度上有所不同:•页面锁存:每个页面都有自己的Reader-Writer锁存，保护其整个内容。线程在访问页面之前获得读锁存或写锁存。这降低了并行性，因为一次可能只有一个线程可以访问一个页面，但是访问一个页面中的多个插槽对于单个线程来说速度很快，因为它只需要获取一个锁存器。
•槽位闩锁:每个槽位都有自己的闩锁。这增加了并行性，因为两个线程可以访问同一页面上的不同槽。但是它增加了访问表的存储和计算开销，因为线程必须为它们访问的每个槽获取一个锁存器，并且每个槽必须为锁存器存储数据。DBMS可以使用单模闩锁(即自旋闩锁)来减少元数据和计算开销，但要牺牲一些并行性。
也可以直接使用比较与交换(CAS)指令创建无锁存的线性探测哈希表。插入槽可以通过尝试将一个特殊的“null”值与我们希望插入的元组进行比较和交换来实现。如果失败，我们可以探测下一个槽，直到它成功
5个B+树闩
B+Tree锁存的挑战是防止以下两个问题:•线程试图同时修改节点的内容。
•一个线程遍历树，另一个线程拆分/合并节点。
闩锁捕获/耦合是一种允许多个线程同时访问/修改B+Tree的协议。
其基本思想如下。
1. 给父母弄个门闩。
2. 给孩子拿门闩。
3. 如果孩子被认为是“安全的”，为父母释放闩锁。“安全”节点是指在更新时不会拆分、合并或重新分发的节点。换句话说，节点是“安全的”，如果插入:它不是满的。
•用于删除:已满一半以上。
请注意，读锁存器不需要担心“安全”状态。
搜索:从根部开始向下，反复获得孩子的闩锁，然后解开父母的闩锁。
•插入/删除:从根开始向下，根据需要获得X锁存器。一旦孩子被锁住，检查是否安全。如果孩子是安全的，释放所有祖先的锁。
从正确性的角度来看，释放锁存器的顺序并不重要。然而，从性能的角度来看，最好释放树中较高的闩锁，因为它们阻塞了对更大一部分叶节点的访问。
改进的锁存抓取协议:基本锁存抓取算法的问题是，对于每个插入/删除操作，事务总是在根节点上获得独占锁存。这限制了并行性。
相反，可以假设必须调整大小(即拆分/合并节点)是罕见的，因此事务可以获得共享锁存到叶节点。每个事务都假定到达目标叶节点的路径是安全的，并使用READ锁存器和抓取来到达它并进行验证。如果叶节点不安全，那么我们中止并执行之前获得WRITE锁存的算法。
•搜索:算法与之前相同。
•插入/删除:将READ锁存设置为搜索，转到叶子，并将WRITE锁存设置为叶子。如果叶子不安全，释放所有先前的锁存器，并使用先前的插入/删除协议重新启动事务。
叶子节点扫描这些协议中的线程以“自顶向下”的方式获取锁存。这意味着线程只能从低于其当前节点的节点获取锁存。如果所需的锁存器不可用，线程必须等待，直到它可用为止。鉴于此，永远不会有死锁。
然而，叶节点扫描容易受到死锁的影响，因为现在我们有线程试图同时在两个不同的方向上获取独占锁(例如，线程1试图删除，线程2进行叶节点扫描)。索引锁存器不支持死锁检测或避免。
因此，程序员处理这个问题的唯一方法就是通过编码规范。叶节点兄弟闩锁获取协议必须支持“无等待”模式。也就是说，B+树代码必须处理闩锁获取失败的情况。由于锁存的目的是(相对地)短暂地保持，如果一个线程试图获得叶节点上的锁存，但该锁存不可用，那么它应该迅速中止其操作(释放它持有的任何锁存)并重新开始操作。
