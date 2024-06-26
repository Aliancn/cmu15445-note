# 讲座#07:哈希表
15-445/645数据库系统(春季2023)https://15445.courses.cs.cmu.edu/spring2023/卡内基梅隆大学查理加罗德
## 1 数据结构
数据库管理系统在系统内部的许多不同部分使用不同的数据结构。一些例子包括:
- 内部元数据:这是跟踪数据库和系统状态信息的数据。（页表，页字典）
- 核心数据存储:数据结构被用作数据库中元组的基础存储。
- 临时数据结构:DBMS可以在处理查询时动态构建数据结构以加快执行速度(例如，用于连接的哈希表)。
- 表索引:辅助数据结构可以用来更容易地找到特定的元组。
- 
在为DBMS实现数据结构时，有两个主要的设计决策需要考虑:
1. 数据组织:为了支持有效的访问，我们需要弄清楚如何布局内存以及在数据结构中存储哪些信息。
2. 并发性:我们还需要考虑如何允许多个线程访问数据结构而不会产生问题。
## 2 哈希表
哈希表实现了将键映射到值的关联数组抽象数据类型。它提供平均O(1)的操作复杂度(最坏情况下为O (n))和O (n)的存储复杂度。请注意，即使平均操作复杂度为0(1)，在现实世界中也需要考虑常数因子优化。

哈希表的实现由两部分组成:
- 哈希函数:它告诉我们如何将一个大的键空间映射到一个较小的域。它用于计算桶或槽数组的索引。我们需要考虑快速执行和碰撞率之间的权衡。在一种极端情况下，我们有一个总是返回一个常量的哈希函数(非常快，但一切都是碰撞)。在另一种极端情况下，我们有一个“完美的”哈希函数，其中没有碰撞，但计算时间非常长。理想的设计是介于两者之间。
- 哈希方案:这告诉我们如何处理哈希后的键冲突。在这里，我们需要考虑分配大型哈希表以减少冲突和在发生冲突时必须执行额外指令之间的权衡。

## 3 哈希函数
哈希函数接受任何键作为其输入。然后它返回该键的整数表示(即“哈希”)。该函数的输出是确定性的(即，相同的键应该总是生成相同的哈希输出)。

DBMS不需要使用加密安全哈希函数(例如SHA-256)，因为我们不需要担心保护密钥的内容。这些散列函数主要在DBMS内部使用，因此信息不会泄露到系统外部。一般来说，我们只关心哈希函数的速度和碰撞率。

目前最先进的哈希函数是Facebook的XXHash3。

## 4 静态哈希方案
静态哈希方案是指哈希表的大小是固定的。这意味着，如果DBMS耗尽了哈希表中的存储空间，那么它必须从头开始重建一个更大的哈希表，这是非常昂贵的。通常，新哈希表的大小是原哈希表的两倍。

为了减少浪费的比较次数，避免散列键的冲突是很重要的。通常，我们使用两倍的槽数作为期望的元素数。

以下假设通常在现实中是站不住脚的:
1. 元素的数量是提前知道的。
2. 密钥是唯一的。
3. 存在一个完美的哈希函数。

因此，我们需要适当地选择哈希函数和哈希模式。

### 4.1 线性探测哈希
这是最基本的哈希方案。它通常也是最快的。它使用数组槽的循环缓冲区。
哈希函数将键映射到槽。当发生碰撞时，我们线性搜索相邻的槽，直到找到一个开放的槽。对于查找，我们可以检查键散列指向的槽，然后线性搜索，直到找到所需的条目。如果我们到达一个空槽，或者我们遍历了哈希表中的每个槽，那么键不在表中。注意，这意味着我们必须将键和值同时存储在槽中，以便检查条目是否为所需条目。删除则更加棘手。我们必须小心地从槽中删除条目，因为这可能会阻止以后的查找找到已经放在现在空的槽下面的条目。对于这个问题有两种解决方法
- 最常见的方法是使用“墓碑”。我们没有删除这个条目，而是用一个“墓碑”条目来代替它，这个条目告诉以后的查找继续扫描。
- 另一种选择是在删除条目后移动相邻的数据来填充现在的空槽。然而，我们必须小心地只移动最初被移动的项。这在实践中很少实现。

*非唯一键*:在同一个键可能与多个不同的值或元组关联的情况下，有两种方法。
- 独立链表:我们不是用键来存储值，而是存储一个指向单独存储区域的指针，该存储区域包含所有值的链表。
- 冗余键:更常见的方法是简单地在表中多次存储相同的键。即使我们这样做，所有线性探测仍然有效。
### 4.2 罗宾汉哈希
这是线性探测哈希的扩展，旨在减少每个键在哈希表中的最佳位置(即它们被哈希到的原始槽)的最大距离。这种策略从“富”键中窃取插槽，并将其提供给“穷”键。

在这种变体中，每个条目还记录了它们与最佳位置的“距离”。然后，在每次插入时，如果被插入的键离当前槽的最佳位置比当前表项的距离更远，我们替换当前表项并继续尝试将旧表项插入表中更远的位置

### 4.3 布谷鸟哈希
这种方法不使用单个哈希表，而是使用不同的哈希函数维护多个哈希表。哈希函数是相同的算法(例如，XXHash, CityHash);它们通过使用不同的种子值为同一键生成不同的哈希值。

当我们插入时，我们检查每个表并选择一个有空闲槽的表(如果多个表有一个空闲槽，我们可以比较诸如负载因子之类的东西，或者更常见的是，只是随机选择一个表)。如果没有表有空闲槽，我们选择(通常是随机的)并驱逐旧的表项。然后将旧条目重新散列到另一个表中。在极少数情况下，我们可能会陷入一个循环。如果发生这种情况，我们可以使用新的哈希函数种子(不太常见)重新构建所有哈希表，或者使用更大的表(更常见)重新构建哈希表。

布谷鸟哈希保证0(1)次查找和删除，但插入可能更昂贵。

教授注:布谷鸟哈希的本质是多个哈希函数将一个键映射到不同的槽。
在实践中，布谷鸟哈希是通过多个哈希函数实现的，这些哈希函数将一个键映射到单个哈希表中的不同槽。此外，由于哈希可能并不总是O(1)，因此布谷哈希查找和删除的成本可能超过O(1)。

## 5 动态哈希方案
静态散列方案要求DBMS知道它想要存储的元素的数量。否则，如果需要增加/缩小大小，则必须重新构建表。

动态哈希方案能够根据需要调整哈希表的大小，而无需重新构建整个表。这些方案以不同的方式执行这种大小调整，可以最大化读或写。
### 5.1 链式哈希
这是最常见的动态散列方案。DBMS为哈希表中的每个槽维护一个桶的链表。散列到同一槽的键被简单地插入到该槽的链表中。
### 5.2 可扩展哈希
链式散列的改进变体，拆分桶而不是让链永远增长。这种方法允许哈希表中的多个槽位置指向相同的桶链。

重新平衡哈希表背后的核心思想是在拆分时移动桶项，并增加检查以查找哈希表中的项的位数。这意味着DBMS只需要在分割链的桶内移动数据;所有其他桶都保持不变。
- DBMS维护全局和本地深度位计数，确定在槽数组中查找桶所需的位数。
- 当一个桶已满时，DBMS拆分桶并重新洗牌它的元素。如果分裂桶的本地深度小于全局深度，则新桶仅添加到现有槽阵列中。否则，DBMS将槽数组的大小加倍以容纳新桶，并增加全局深度计数器。
### 5.3 线性哈希
该方案不是在桶溢出时立即拆分桶，而是维护一个拆分指针，跟踪下一个要拆分的桶。无论这个指针是否指向溢出的存储桶，DBMS总是分裂。溢出标准留给实现。
- 当任何桶溢出时，通过添加新的槽项在指针位置拆分桶，并创建新的哈希函数。
- 如果哈希函数映射到先前被指针指向的槽位，则应用新的哈希函数。
- 当指针到达最后一个槽位时，删除原来的哈希函数，用新的哈希函数代替。
