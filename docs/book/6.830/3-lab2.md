# lab2

阅读 lab2.md

完善 buffer pool ，增加页面置换算法。

这个实验中不需要考虑实现事务或锁定。

实现 Filter 和 Join ，已经提供了 Project 和 OrderBy 的实现。

实现 StringAggregator 和 IntegerAggregator 。编写计算一个特定字段在输入 tuple 序列的多个组中的聚合。

其中 IntegerAggregator 使用整数除法来计算平均数，因为 SimpleDB 只支持整数。StringAggegator 只需要支持 COUNT 聚合，因为其他操作对字符串没有意义。

实现`Aggregate`操作符。和其他运算符一样，聚合运算符实现了 `OpIterator` 接口，这样它们就可以放在 SimpleDB 查询计划中。注意`Aggregate`运算符的输出是每次调用`next()`时整个组的聚合值，并且聚合构造器需要聚合和分组字段。

实现`插入`和`删除`操作符。像所有的操作符一样，`Insert`和`Delete`实现了`OpIterator`，接受一个要插入或删除的 tuple 流，并输出一个带有整数字段的单一 tuple ，表示插入或删除的 tuple 数量。这些操作者将需要调用`BufferPool`中的适当方法，这些方法实际上是修改磁盘上的页面。检查插入和删除 tuple 的测试是否正常工作。

## Exercise 1

实现 `execution/Predicate.java`,`execution/JoinPredicate.java`,`execution/Filter.java` 和 `execution/Join.java` 并通过 PredicateTest、JoinPredicateTest、FilterTest 和 JoinTest 中的单元测试。此外，还需通过系统测试 FilterTest 和 JoinTest。

`Predicate.java` 比较表内的字段和提供的数据，三个参数分别是待比较的字段序号、比较符和待比较的数。其中 `filter()` 方法输入一个 Tuple ，然后比较 Tuple 的 Field 是否符合预期。

`JoinPredicate.java` 和 `Predicate.java` 类似，只是实现两个 Tuple 的比较。

`Filter.java` 在构造函数中实例化 Predicate 和 OpIterator。其中 `fetchNext()` 方法逐个读取 OpIterator 中的 Tuple ，然后让他们与 Predicate 中 Field 进行比较，如果为真则返回该 Tuple。

`Join.java` 就是对 `JoinPredicate.java` 的使用，通过构造函数实例化 `JoinPredicate` 和两个OpIterator 。实现一系类get方法和open、close等迭代器的函数。最后完成fetchNext函数找到两个迭代器中可以jion的字段进行join。

fetchNext 中由两个 while 循环进行遍历，直到最外层迭代器遍历完成，每次遍历 child1 取出一个 Tuple ，与 child2 中的所有 Tuple 做 filter 比较，直到有符合要求的，创建新的 TupleDesc ，并且将 child1 和 child2 的字段（field），加入newTuple中，然后返回newTuple，同时将 child2 重置到最开始。


## Exercise 2

> 这个练习写起来挺困难的。🙄

实现下面几个方法并通过 IntegerAggregatorTest 、StringAggregatorTest 和 AggregateTest 单元测试。此外还需要通过 AggregateTest 的系统测试。

* src/java/simpledb/execution/IntegerAggregator.java
* src/java/simpledb/execution/StringAggregator.java
* src/java/simpledb/execution/Aggregate.java

只需要实现单个字段的聚合（aggregation）和单个字段的分组（group by）即可。聚合其实就对一组数据进行操作（加减乘除，最值等）。具体可参考：[SQL GROUP BY 语句](https://www.runoob.com/sql/sql-groupby.html)。

`IntegerAggregator(0, Type.INT_TYPE, 1, Aggregator.Op.SUM)` 是生成一个整数聚合的对象。

其中 0 表示分组(Group By)字段位置，也就是根据第零列来聚合。可以为 NO_GROUPING，表示不进行聚合。

Type.INT_TYPE 表示这一列的数据类型，目前只有整数和字符串。1 表示待聚合的字段，Aggregator.Op.SUM 表示执行加法操作。

需要看懂 IntegerAggregatorTest 测试类。其中 scan1 是一张基础表，sum/min/max/avg 是四张经过聚合操作后的表，用于验证 scan1 经过聚合后的结果是否符合预期。

`mergeTupleIntoGroup()` 根据 gbField 字段先判断是否需要进行 group by 。如果需要，那么根据 gbField 从 tup 中提取待聚合的字段，再判断是否是初次填入，然后根据对应 Op 执行对应逻辑。如果不需要 group by 直接累加即可，不需要映射，

StringAggregator 和 IntegerAggregator 逻辑类似，并且仅支持 COUNT 。

Aggregate 是将前两个整合一下。

## Exercise 3.

增加 tuple 或删除 tuple

1. 编写 `HeapPage.java` 并通过 `HeapPageWriteTest` 。

首先根据要删除 tuple 的 RecordId 判断是否被使用，如果已经被使用就比较当前的 tuple 和待删除的 tuple 对象，一致就删除并标记。如果没有被使用那么 tuple slot 就是空。

* markDirty() 用一个队列来记录脏页的 tid，如果是脏页就加入队列中，如果不是就从队列中删除。
* isDirty() 返回队列中最后一个脏页，如果没有脏页就返回 null。
* insertTuple() 首先判断当前页面 td 和待插入 tuple 的 TupleDesc 是否匹配。然后遍历空余的 slot，寻找插入位置找到后插入并设置 RecordId 。最后标记该位置已经被插入。
* markSlotUsed() 修改 head 表示 tuple 被使用。
* deleteTuple() 依旧是判断当前页面 td 和待删除 tuple 的 TupleDesc 是否匹配。然后根据待删除的 tuple 找到 RecordId 判断是否存在，最后根据索引判断 slot 是否被使用，如果使用就删除。

2. 编写 HeapFile.java 并通过 `HeapFileWriteTest` 

* `insertTuple()` 如果当前没有页面就调用 writePage 在磁盘中创建空页。然后去 BufferPool 取页，接下来判断取到的页中是否含有空 slot ，然后插入 tuple 。
* `deleteTuple()` 从 BufferPool 中取出 page 然后删除 tuple 。

3. 编写 BufferPool.java 中的 insertTuple() 和 deleteTuple() 并通过 `BufferPoolWriteTest`。

## Exercise 4.

实现 `execution/Insert.java` 和 `execution/Delete.java` 并通过 InsertTest 和 InsertTest，DeleteTest system tests

## Exercise 5.

实现 BufferPool.java 中的 flushPage() 方法，

通过 EvictionTest system test

discardPage() 方法是直接从缓冲池中删除不写回磁盘。

用 LRU 来实现！通过这道题可以学会 LRU ，[Leetcode 146. LRU 缓存](https://leetcode-cn.com/problems/lru-cache/)，这个[视频](https://www.bilibili.com/video/BV1hp4y1x7MH)讲的很好！ 

# Lab 3: Query Optimization

* [数据库内核杂谈（七）：数据库优化器（上）](https://www.infoq.cn/article/GhhQlV10HWLFQjTTxRtA)

这个 lab 大致要实现的东西：

1. 实现 TableStats 类中的方法，使其能够使用直方图（IntHistogram类提供的骨架）或你设计的其他形式的统计数据来估计过滤器的选择性和扫描的成本。
2. 实现JoinOptimizer类中的方法，使其能够估计 join 的成本和选择性。
3. 编写JoinOptimizer中的orderJoins方法。这个方法必须为一系列的连接产生一个最佳的顺序（可能使用Selinger算法），给定前两个步骤中计算的统计数据。

基于开销优化器的主要思想：

* 根据 table 的统计数据来估计不同查询计划的开销。通常一个计划的成本与 intermediate joins 和 tuple 数量，以及 selectivity of filter 和 join predicates 的选择性有关。
* 根据统计数据以最佳方式排列连接和选择，并从几个备选方案中选择连接算法的最佳实现。

优化器将会被 `simpledb/Parser.java` 调用，写实验之前回顾 [lab 2 parser exercise](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab2.md#27-query-parser)

## Exercise 1: IntHistogram.java

实现 IntHistogram 并通过 IntHistogramTest。

针对一个字段构建一张直方图，横坐标代表属性对应范围，纵坐标代表对应范围内 tuple 的数量。

占比计算 `(h / w) / ntups`  ntups 是纵坐标的累加和，也就是 tuple 的总数。

部分区间的占比 ：`b_part =（b_right - const）/ w_b` `b_f = h_b / ntups` `b_f x b_part`

* `addValue(int v)`

根据输入数据构建直方图的分部，计算出对应桶序号累加即可。

* `estimateSelectivity(Predicate.Op op, int v)` 估计

这个类用来计算占比。具体的计算规则是根据运算符 op 判断(大于，小于，等于...)，v 就是 const ，遍历。例如 op 是大于， v 是 3 ，那么就是计算横坐标大于 3 所有 tuple 个数除以总 tuple 个数(ntuple)。也就是大于 3 tuple 占总 tuple 的百分比。

## Exercise 2: TableStats.java

实现 TableStats 并通过 TableStatsTest。

* 实现TableStats构造函数

为 table 的每一个 field 构建一张直方图。

根据 tableid 拿到 table，然后遍历 table 的每个字段(field)构建直方图。注意 field 分为整数和字符串两种类型，分别用 map 来存。

首先获取每一列对应的内容，放入 list 中。然后获取所有列的内容，一列就是一个 field ，一列生成一个直方图。

* estimateScanCost() 

IO 成本是页数乘上单页 IO 的开销。

* estimateTableCardinality()

tuple 总数乘上系数 (selectivityFactor)。

* estimateSelectivity()

根据输入的参数来估计 Selectivity ，三个参数分别是待估计的字段，比较符号，const。区分field 的int 和 string 分别调用 estimateSelectivity() 即可。

## Exercise 3: Join Cost Estimation

编写 JoinOptimizer 并通过 JoinOptimizerTest 中的 estimateJoinCostTest 和 estimateJoinCardinality 即可。

* 实现 `estimateJoinCost()` 方法，估计 join 的成本。

计算公式：

  joincost(t1 join t2) = scancost(t1) + ntups(t1) x scancost(t2) //IO cost
                      + ntups(t1) x ntups(t2)  //CPU cost

> Nested-loop (NL) join是所有join算法中最naive的一种。假设有两张表R和S，NL join会用二重循环的方法扫描每个(r, s)对，如果行r和行s满足join的条件，就输出之。显然，其I/O复杂度为O(|R||S|)。随着参与join的表个数增加，循环嵌套的层数就越多，时间复杂度也越高。因此虽然它的实现很简单，但效率也较低。

总结：成本 (cost) 分为 I/O 成本和 CPU 成本。I/O 成本是扫描表时和磁盘交互所产生的，而 CPU 成本是判断数据是否符合条件所产生的。其中 cost1 是扫描 t1 的 I/O 成本，cost2 同理。因为是 NL join 所以总的 I/O 开销就是 `cost1 + card1 * cost2` 。而 CPU 开销则是 `card1 * card2` 。总成本相加即可。

* estimateJoinCardinality 估计 join 后产生的 tuple 数。

lab3.md 中 2.2.4 Join Cardinality 部分有详细解释。

Cardinality 表示一列数据中数据的重复程度，如果等于 1 那么数据没有重复的，如果等于 0 那么全部都重复，其他情况加载 [0 , 1] 之间。具体可参考：[What is cardinality in Databases?](https://stackoverflow.com/questions/10621077/what-is-cardinality-in-databases) 。

对于等价连接() 其中一个属性是主键时，由连接产生的 tuples 的数量不能大于非主键属性的cardinality。只要保证这一点成立即可，所以其中一个是主键的话就选择一个小的，两个都是主键的话选择小的，两个都不是主键的话选择大的。这块的实现很灵活。

对于非等价连接文档给了公式 `card1 * card2 * 0.3` 。

## Exercise 4: Join Ordering

实现 JoinOptimizer.java 中的 orderJoins 方法并通过 JoinOptimizerTest 和系统测试 QueryTest 。

ex3 实现了开销估计和基数个数的估计。这个练习则是在多表连接的情况下根据开销分析选择最优的连接顺序。直接枚举的话复杂度是 O(n!) 。此处选择了一种 DP 的方法将复杂度降低到了 O(2^n)。

首先要理解什么是 left-deep-tree 可参考这篇[文章](https://www.infoq.cn/article/JCJyMrGDQHl8osMFQ7ZR)，写的很好！

然后阅读 [Exercise 4: Join Ordering](https://blog.csdn.net/weixin_45834777/article/details/120788433?spm=1001.2014.3001.5501) 部分。

JoinOptimizer 中的 join 属性是一个队列，其中存的都是 LogicalJoinNode 对象。

PlanCache 类，用来缓存 Selinger 实现中所考虑的连接子集的最佳顺序，接下来的任务就是找到最佳顺序。

`enumerateSubsets(joins, i);` 其中 i 表示子集中的子集的元素个数。例如 a,b,c 三张表，当 i=1 时，返回数据大致形态 set(set(ab) , set(ac), set(bc)) ， 注意 ab 是一个 LogicalJoinNode 所以尺寸是 1 。如果 i=2 ，那么返回的数据类似 set(set(ab, c) , set(ac, b), set(bc, a)) 。可以优化为回溯，避免创建大量对象。

> 这块内容建议阅读帆船书《Database System Concepts》第七版的 16.4.1 Cost-Based Join-Order Selection 部分

# Lab 4: SimpleDB Transactions

Transactions 被翻译为事物，其实就是一个原子级的操作，重点是操作不能被中断。用锁来实现原子级别的操作，但是单纯用锁的话存在串行化的问题，于是引入了 2PL 从而保证了串行化。

在 2PL 下，事物分为增长阶段 (growing phase) 和收缩阶段 (shrinking phase) ，区别在于前者只能不断加锁，而后者只能不断减锁，一旦开始减锁就意味着从增长阶段转为收缩阶段。

> 2PL 建议看这个视频 [16-两阶段锁](https://www.bilibili.com/video/BV1AZ4y1Q7vx/?spm_id_from=333.788) 或者阅读 《Database System Concepts》 18.1.3 The Two-Phase Locking Protocol 这篇文章也不错：https://zhuanlan.zhihu.com/p/59535337

## Exercise 1 and 2.

这两个练习是编写 BufferPool  最终通过 LockingTest 。

为 BufferPool 添加获取锁和释放锁的功能，修改 getPage() 实现 unsafeReleasePage(), holdsLock() 实现下一个练习才能通过 LockingTest 。

具体思路，实现一个 Lock 类和 LockManager 类。LockManager 类实现三个功能申请锁、释放锁、查看指定数据页的指定事务是否有锁。

## Exercise 3.

之前没有区分是否是脏页就直接写回了，不能将脏页直接淘汰。

修改 evictPage() 方法，倒着遍历，删除一个非脏页即可。

## Exercise 4.

实现 `transactionComplete()` 

通过 TransactionTest 单元测试和 AbortEvictionTest 系统测试

如果 commit 那么就把 tid 对应的所有页面持久化，也就是写入磁盘否则把该事物相关的页面加载进缓存中。

## Exercise 5.

检测死锁，然后通过 DeadlockTest 和 TransactionTest 系统测试。

设置一个区间，如果超时就说明发生死锁了。

# lab 5

了解 B+ 树。

BTreeFile 由四种不同的页面组成，

* BTreeInternalPage.java 内部页
* BTreeLeafPage.java 叶子页
* BTreePage.java 包含了叶子页和内部页的共同代码
* BTreeHeaderPage.java 跟踪文件中哪些页正在使用

## Exercise 1: BTreeFile.findLeafPage()

在 BTreeFile.java 中实现 findLeafPage() 方法，功能是给定一个特定的键值的情况下找到合适的叶子页。

具体流程如下图：根节点是 6 是一个内部页，两个指针分别指向了叶子页。如果输入 1 那么 findLeafPage() 应当返回第一个叶子页。如果输入 8 那么应当返回第二个叶子页。如果输入 6 此时左右叶子页都含有 6 ，函数应当返回第一个叶子页，也就是左边的叶子页。

![](image/index/1644485406419.png)

findLeafPage() 递归搜索节点，节点内部的数据可以通过 BTreeInternalPage.iterator() 访问。

当 key value 为空的时候，应当递归做左边的子页进而找到最左边的叶子页。BTreePageId.java中的pgcateg() 函数检查页面的类型。可以假设只有叶子页和内部页会被传递给这个函数。

BTreeFile.getPage() 和 BufferPool.getPage() 原理一样但需要一个额外的参数来跟踪脏页。

findLeafPage() 访问的每一个内部（非叶子）页面都应该以 READ_ONLY 权限获取，除了返回的叶子页面，它应该以作为函数参数提供的权限获取。这些权限在本实验中不重要但是后续实验中很重要。

> 这个练习很简单，上面的内容本来是文档的总结，后来发现几乎就是代码的文字版。。。

通过 BTreeFileReadTest.java 中的所有单元测试和 BTreeScanTest.java 中的系统测试。

## Exercise 2: Splitting Pages

在 BTreeFile.java 中实现 splitLeafPage() 和 splitInternalPage() 并通过 BTreeFileInsertTest.java 中的单元测试和 systemtest/BTreeFileInsertTest.java 中的系统测试。

通过 findLeafPage() 可以找到应该插入 tuple 的正确叶子页，但是页满的情况下插入 tuple 可能会导致页分裂，进而导致父节点分裂也就是递归分裂。

如果被分割的页面是根页面，你将需要创建一个新的内部节点来成为新的根页面，并更新 BTreeRootPtrPage
否则，需要以 READ_WRITE 权限获取父页，进行递归分割，并添加一个 entry。getParentWithEmptySlots()对于处理这些不同的情况非常有用。

在 splitLeafPage() 中将键“复制”到父页上，页节点中保留一份。而在 splitInternalPage() 中，你应该将键“推”到父页上，内部节点不保留。

当内部节点被分割时，需要更新所有被移动的子节点的父指针。updateParentPointers() 很有用。

每当创建一个新的页面时，无论是因为拆分一个页面还是创建一个新的根页面，都要调用 getEmptyPage() 来获取新的页面。这是一个抽象函数，它将允许我们重新使用因合并而被删除的页面（在下一节涉及）。

BTreeLeafPage.iterator() 和 BTreeInternalPage.iterator() 实现了叶子页和内部页进行交互，除此之外还提供了反向迭代器 BTreeLeafPage.reverseIterator() 和 BTreeInternalPage.reverseIterator() 。

BTreeEntry.java 中有一个 key 和两个 child pointers ，除此之外还有一个 recordId 用于识别底层页面上键和子指针的位置。

## Exercise 3: Redistributing pages

实现 BTreeFile.stealFromLeafPage(), BTreeFile.stealFromLeftInternalPage(), BTreeFile.stealFromRightInternalPage() 并通过 BTreeFileDeleteTest.java 中的一些单元测试（如testStealFromLeftLeafPage和testStealFromRightLeafPage）

删除存在两种情况，如果兄弟节点数据比较多可以从兄弟节点借，反之数据较少可以和兄弟节点合并。

stealFromLeafPage() 两个页面 tuple 加一起然后除二，平均分成两个 leaf page 。

## Exercise 4: Merging pages

实现 BTreeFile.mergeLeafPages() 和 BTreeFile.mergeInternalPages() 。

现在应该能够通过 BTreeFileDeleteTest.java 中的所有单元测试和 systemtest/BTreeFileDeleteTest.java 中的系统测试。

# Lab 6: Rollback and Recovery

根据日志内容实现 rollback 和 recovery 。

当读取 page 时，代码会记住 page 中的原始内容作为 before-image 。 当事务更新 page 时，修改后的 page 作为 after-image 。使用 before-image 在 aborts 进行 rollback 并在 recovery 期间撤销失败的事务。

## Exercise 1: LogFile.rollback()

实现LogFile.java中的rollback()函数

通过LogTest系统测试的TestAbort和TestAbortCommitInterleaved子测试。

rollback() 回滚指定事务，已经提交了的事务上不能执行该方法。将上一个版本的数据写回磁盘。

当一个事务中止时，在该事务释放其锁之前，这个函数被调用。它的工作是解除事务可能对数据库做出的任何改变。


## Exercise 2: LogFile.recover()

实现 Implement LogFile.recover().

重启数据库时会率先调用 LogFile.recover() 

对于未提交的事务：使用before-image对其进行恢复，对于已提交的事务：使用after-image对其进行恢复。