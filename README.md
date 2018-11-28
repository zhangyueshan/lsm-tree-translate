# The Log-Structured Merge-Tree

## 摘要
高性能事务处理系统通常将一条记录插入到历史表去提供快速的查找，并且同时，事务系统会生成日志来用于系统的恢复。这两种类型的生成的信息都可以从高效的索引中受益。一个为人熟知的场景（原文为setting）是TPC-A基准测试应用，支持从活动账号的历史表中查询指定账号的高效查询。这需要一个在快速增长的历史表上的位于account-id列的高效索引。不幸的是，经典的基于磁盘的索引结构比如B-tree，在维护类似这类索引的实时事务中会产生双倍的磁盘IO开销，这使得整个系统的开销升高了50%。显然，一个新的实时生成索引的方法是值得拥有的。The Log-Structured Merge-Tree(LSM-Tree)是一个基于磁盘设计的数据结构，它能为一个文件提供低消耗的索引，体验到在一段持续时间内的高速率的插入和删除操作。LSM树使用一种算法来推迟和批量索引的变更，用一种高效的方式使这些变更从一个基于内存的组件流向到一个或者多个硬盘组件上，这让人想起归并排序。在这期间所有的索引数据可以持续的提供检索（除了一段很短的加锁时间），不论是通过内存中或者是通过硬盘来查询。这个算法相比传统的类似B-Tree数据访问方法能够显著的降低磁盘机械臂的开销，并且同时能够提升在传统方法写入时磁盘机械臂的开销远大于存储介质开销的场景下的性能。LSM-Tree的方法也可推广到除了插入和删除之外的其他操作。当然，在某些情况下，需要快速响应的索引查找会降低I/O性能，所以LSM-Tree通常更适用于那些写入比检索更频繁的系统。这似乎是类似历史记录表，日志表的共同属性。在第六节的结尾比较了混合运用内存和磁盘的LSM-tree方案和通常认为的通过缓存磁盘页的混合方式优化方案。

### 引言
当长时间事务在活动流管理系统中开始商业化的提供，这将会产生更多的对事务日志记录提供索引访问的需求。传统上，事务日志专注于中断和恢复，并且要求系统允许根据相对较短时间内的历史来进行偶尔的事务回滚，此时恢复会通过使用批量顺序读。然而，当系统承担起更复杂的活动的责任时，一个单个长任务的耗时和事件的数量将会提升到某个点，此时某些时候会需要能够实时查看过去的事务步骤来提醒用户哪些已经完成了。并且同时，所有的系统内的活动事件的数量将会提升到某个点，此时我们目前使用的为了跟踪活动日志的驻留在内存中数据结构将不再可用，即使内存的价格如预期持续的下降。回答查询海量的活动日志的需求意味着索引化日志的访问将会越来越重要。
即使是目前的事务系统，在大批量插入的情况下提供索引去支持历史表的查询也是有价值的。网络系统，电子邮件系统和其他的接近事务处理的系统处理海量的日志，这经常不利于它们的主机系统。为了从一个具体的为人熟知的例子开始，我们在接下来的例1和例2中探索了一个修改过的TPC-A基准测试。注意此论文中的示例处理特殊的参数化的数字值，便于展示；总结这些接过是个简单的任务。同时注意即使无论历史表和日志都涉及到时序数据，SLM-Tree中的索引记录并不假定有相同的时间顺序。唯一的假设是能够提升性能的是高频率的插入速率而不是检索速率。
#### 五分钟规则
下面的两个示例都基于五分钟规则。这个基本的结论表明我们那可以通过购买内存缓冲空间来将页面维持在内存中来降低系统的开销。
