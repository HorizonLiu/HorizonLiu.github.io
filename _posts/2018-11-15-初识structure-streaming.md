#### 什么是structure streaming？

sprak1.x中是基于RDD存储的一个一维、只有行概念的数据集；而spark2.x中，是基于DataSet和DataFrame的二维行+列的数据集，更偏向于一张二维的表，且DataSet和DataFrame支持类似SQL语句的增删改查。

以DataSet/DataFrame的行列表格来表达Structure data，既容易理解，又具有广泛的实用性，比如：

- 可以将java类转换为DataSet/DataFrame
- 多条json对象可以方便的转换为DataSet/DataFrame
- MySql表、行式存储文件、列式存储文件可以很好的转换为DataSet/DataFrame

在DataSet/DataFrame的基础上，就衍生了structure streaming，它和静态的structure data不同，动态的structure data的行列数据表格是一直无限增长的，所以可以把structure streaming看成一个无限增长的表格。

#### structure streaming的组成

structure streaming主要包含三大组件，分别是数据源、计算逻辑和结果数据。可以用下图表示：

![img](https://ws3.sinaimg.cn/large/006tNbRwgy1fy329z06pwj30oe03y74x.jpg)

目前的structure streaming中支持很多的数据源，包括kafka、hdfs等；中间的计算逻辑streamExecution包含了DataSet和DataFrame的一些列变换，而为了保证计算过程中的可靠性和exactly once的特性，streamExcution中还包含了offsetLog，batchCommitLog，currentBatchID等重要的组件，分别用来记录当前执行需要处理的source data的元信息、已经成功处理过的批次和当前执行的批次id等等信息。

#### structure streaming的执行流程及优点

![img](https://ws4.sinaimg.cn/large/006tNbRwgy1fy32azznepj30oe0g5dip.jpg)

 

**执行流程：**

1. StreamExecution 通过 Source.getOffset() 获取最新的 offsets，即最新的数据进度；

2. StreamExecution 将 offsets 等写入到 offsetLog 里这里的 offsetLog 是一个持久化的 WAL (Write-Ahead-Log)，是将来可用作故障恢复用

3. StreamExecution 构造本次执行的 LogicalPlan
  (3a) 将预先定义好的逻辑（即 StreamExecution 里的 logicalPlan 成员变量）制作一个副本出来
  (3b) 给定刚刚取到的 offsets，通过 Source.getBatch(offsets) 获取本执行新收到的数据的Dataset/DataFrame 表示，并替换到 (3a) 中的副本里   经过 (3a), (3b) 两步，构造完成的 LogicalPlan 就是针对本执行新收到的数据的 Dataset/DataFrame 变换（即整个处理逻辑）了

4. 触发对本次执行的 LogicalPlan 的优化，得到 IncrementalExecution逻辑计划的优化：通过 Catalyst 优化器完成物理计划的生成与选择：结果是可以直接用于执行的 RDD DAG逻辑计划、优化的逻辑计划、物理计划、及最后结果 RDD DAG，合并起来就是 IncrementalExecution

5. 将表示计算结果的 Dataset/DataFrame (包含 IncrementalExecution) 交给 Sink，即调用 Sink.add(ds/df)

6. 计算完成后的 commit   
  (6a) 通过 Source.commit() 告知 Source 数据已经完整处理结束；Source 可按需完成数据的 garbage-collection    
  (6b) 将本次执行的批次 id 写入到 batchCommitLog 里



优点，待写......

#### structure streaming的event time

![img](https://ws3.sinaimg.cn/large/006tNbRwgy1fy32bozlnrj30lv0dztb8.jpg)

structure streaming和spark streaming 一样，支持窗口事务的处理，window为窗口大小，slide为滑动步长。以分词统计为例，对于窗口大小为10min，滑动步长为5min的业务，最后将输出三列为结果，分别是时间窗口、单词、单词数量。（对于其他业务类型，可以按照需要定义要输出的结果）

另外，structure streaming对于迟到的数据（late data）有特殊处理，还是以上图为例，若12:06的数据在12:16才到达，structure streaming仍然会根据其真实时间来确定其所属时间窗口并得到正确的结果，对于这种情况，可以用下图表示。

![img](https://ws2.sinaimg.cn/large/006tNbRwgy1fy32cbepsmj30lh0dw40u.jpg)

对于structure streaming的输出，主要分为三种模式，即：complete、apend和update。

**complete模式**：它的输出和state是完全一致的(输出即为上图的State阶段)，即它会保留所有的中间计算状态。这也就是说，结果表将是一张无限增长的表，随着批次和时间的累加会越来越长。

![img](https://ws3.sinaimg.cn/large/006tNbRwgy1fy32csbbioj30lh0ff438.jpg)

**append模式**：该模式下，在确定某个时间窗口内的数据不再会发生变化（被更新）时，就可以将其输出。但注意，一个时间窗口内的结果只能被输出一次，如果在被输出之后，即使这个窗口内的数据发生了变化，这个更新之后的结果仍然不会输出，这就会造成结果错误的现象。针对这个情况，那怎么确定在某个时间窗口内的数据是否会被再次更新呢？在structure streaming中定义了watermark来解决这个问题。

在append模式下，由于在某个时间窗口内的数据不会更新就会被输出，因此，**该模式下StateStore将不会无限增长**。

关于append模式的示例，可以用下面两张图来分别说明。第一张图是输出结果之后，该时间窗口内的结果还会变化(在12:20的时候再次更新了12:00-12:10内和12:05-12:15内cat的个数），所以这个输出结果将是错误的；而第二张图确定了结果不会变化之后再输出才是正确的。

![img](https://ws2.sinaimg.cn/large/006tNbRwgy1fy32d15ysjj30lh0czq6a.jpg)

![img](https://ws3.sinaimg.cn/large/006tNbRwgy1fy32d4byt2j30lh0czjul.jpg)

**update模式**：该模式中，只要结果随着时间发生了变化就会在结果表中显示（不论是新增还是更新），且该模式下StateStore仍然不会无限增长。如：

![img](https://ws3.sinaimg.cn/large/006tNbRwgy1fy32d87ym2j30lh0fcn1o.jpg)

#### 关于watermark

在介绍structure streaming window操作下的输出模式时，我们提到了append模式下需要确定结果什么时候才不会被更新，这个判断就用到了watermark。

关于watermark的说明：

1. 再次强调，(a+) 在对 event time 做 window() + groupBy().aggregation() 即利用状态做跨执行批次的聚合，并且 (b+) 输出模式为 Append 模式或 Update 模式时，才需要 watermark，其它时候不需要；

2. watermark 的本质是要帮助 StateStore 清理状态、不至于使 StateStore 无限增长；同时，维护 Append 正确的语义（即判断在何时某条结果不再改变、从而将其输出）；

3. 目前版本（Spark 2.2）的 watermark 实现，是依靠最大 event time 减去一定 late threshold 得到的，尚未支持 Source 端提供的 watermark；未来可能的改进是，从 Source 端即开始提供关于 watermark 的特殊信息，传递到 StreamExecution 中使用 [2]，这样可以加快 watermark 的进展，从而能更早的得到输出数据

4. Structured Streaming 对于 watermark 的承诺是：    
  (a) watermark 值不后退（包括正常运行和发生故障恢复时）；    
  (b) watermark 值达到后，大多时候会在下一个执行批次输出结果，但也有可能延迟一两个批次（发生故障恢复时），上层应用不应该对此有依赖。



参考链接：<https://github.com/lw-lin/CoolplaySpark/blob/master/Structured%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/1.1%20Structured%20Streaming%20%E5%AE%9E%E7%8E%B0%E6%80%9D%E8%B7%AF%E4%B8%8E%E5%AE%9E%E7%8E%B0%E6%A6%82%E8%BF%B0.md> 