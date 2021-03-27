# 最终一致的存储后端 Eventually-Consistent Storage Backends

When running JanusGraph against an eventually consistent storage backend special JanusGraph features must be used to ensure data consistency and special considerations must be made regarding data degradation.

This page summarizes some of the aspects to consider when running JanusGraph on top of an eventually consistent storage backend like Apache Cassandra or Apache HBase.

在最终一致的存储后端上运行 JanusGraph 时，必须使用特殊的JanusGraph功能来确保数据一致性，并且必须对数据降级进行特殊考虑。

该页面总结了在最终一致的存储后端（如 Apache Cassandra 或 Apache HBase ）上运行 JanusGraph 时要考虑的一些方面。

## Data Consistency 数据一致性
On eventually consistent storage backends, JanusGraph must obtain locks in order to ensure consistency because the underlying storage backend does not provide transactional isolation. In the interest of efficiency, JanusGraph does not use locking by default. Hence, the user has to decide for each schema element that defines a consistency constraint whether or not to use locking. Use `JanusGraphManagement.setConsistency(element, ConsistencyModifier.LOCK)` to explicitly enable locking on a schema element as shown in the following examples.

在最终一致的存储后端上，JanusGraph必须获取锁以确保一致性，因为基础存储后端不提供事务隔离。为了提高效率，JanusGraph默认不使用锁定。因此，用户必须为定义一致性约束的每个 schema 元素决定是否使用锁定。使用JanusGraphManagement.setConsistency（element，ConsistencyModifier.LOCK）显式启用对 schema 元素的锁定，如以下示例所示。
```
mgmt = graph.openManagement()
name = mgmt.makePropertyKey('consistentName').dataType(String.class).make()
index = mgmt.buildIndex('byConsistentName', Vertex.class).addKey(name).unique().buildCompositeIndex()
mgmt.setConsistency(name, ConsistencyModifier.LOCK) // Ensures only one name per vertex
mgmt.setConsistency(index, ConsistencyModifier.LOCK) // Ensures name uniqueness in the graph
mgmt.commit()
```
When updating an element that is guarded by a uniqueness constraint, JanusGraph uses the following protocol at the end of a transaction when calling `tx.commit()`:

1. Acquire a lock on all elements that have a consistency constraint
1. Re-read those elements from the storage backend and verify that they match the state of the element in the current transaction prior to modification. If not, the element was concurrently modified and a PermanentLocking exception is thrown.
1. Persist the state of the transaction against the storage backend.
1. Release all locks.

This is a brief description of the locking protocol which leaves out optimizations (e.g. local conflict detection) and detection of failure scenarios (e.g. expired locks).

The actual lock application mechanism is abstracted such that JanusGraph can use multiple implementations of a locking provider. Currently, two locking providers are supported in the JanusGraph distribution:

1. A locking implementation based on key-consistent read and write operations that is agnostic to the underlying storage backend as long as it supports key-consistent operations (which includes Cassandra and HBase). This is the default implementation and uses timestamp based lock applications to determine which transaction holds the lock.
1. A Cassandra specific locking implementation based on the Astyanax locking recipe.

Both locking providers require that clocks are synchronized across all machines in the cluster.

- Warning

The locking implementation is not robust against all failure scenarios. For instance, when a Cassandra cluster drops below quorum, consistency is no longer ensured. Hence, it is suggested to use locking-based consistency constraints sparingly with eventually consistent storage backends. For use cases that require strict and or frequent consistency constraint enforcement, it is suggested to use a storage backend that provides transactional isolation.

在更新受唯一性约束保护的元素时，JanusGraph在调用 tx.commit() 时在事务结束时使用以下协议：

1. 获取所有具有一致性约束的元素的锁
1. 从存储后端重新读取那些元素，并在修改之前验证它们是否与当前事务中的元素状态匹配。如果不是，则元素被同时修改，并抛出 PermanentLocking 异常。
1. 针对存储后端保留事务状态。
1. 释放所有锁。

这是对锁定协议的简要说明，其中省略了优化（例如本地冲突检测）和故障情况的检测（例如过期的锁）。

实际的锁应用程序机制被抽象化，以便JanusGraph可以使用锁提供程序的多种实现。当前，JanusGraph发行版中支持两个锁定提供程序：

1. 一种基于密钥一致的读写操作的锁定实现，只要其支持密钥一致的操作（包括 Cassandra 和 HBase ），该锁定实现与基础存储后端无关。这是默认实现，它使用基于时间戳的锁应用程序来确定哪个事务持有该锁。
1. 基于 Astyanax 锁定配方的 Cassandra 特定锁定实现。

两个锁定提供程序都要求在群集中的所有计算机之间同步时钟。

- 警告
对于所有失败情况，锁定实现都不是可靠的。例如，当 Cassandra 群集下降到法定数量以下时，将不再确保一致性。因此，建议在最终一致的存储后端中谨慎使用基于锁定的一致性约束。对于需要严格和（或）频繁执行一致性约束的用例，建议使用提供事务隔离的存储后端。

### Data Consistency without Locks 无锁的数据一致性
Because of the additional steps required to acquire a lock when committing a modifying transaction, locking is a fairly expensive way to ensure consistency and can lead to deadlock when very many concurrent transactions try to modify the same elements in the graph. Hence, locking should be used in situations where consistency is more important than write latency and the number of conflicting transactions is small.

In other situations, it may be better to allow conflicting transactions to proceed and to resolve inconsistencies at read time. This is a design pattern commonly employed in large scale data systems and most effective when the actual likelihood of conflict is small. Hence, write transactions don’t incur additional overhead and any (unlikely) conflict that does occur is detected and resolved at read time and later cleaned up. JanusGraph makes it easy to use this strategy through the following features.

由于在提交修改事务时需要额外的步骤来获取锁，因此锁定是确保一致性的一种相当昂贵的方法，并且在许多并发事务尝试修改图中的相同元素时会导致死锁。因此，在一致性比写延迟更重要且冲突事务数量少的情况下，应使用锁定。

在其他情况下，允许冲突的事务进行并在读取时解决不一致可能更好。这是在大型数据系统中通常使用的设计模式，并且在实际发生冲突的可能性较小时最有效。因此，写事务不会产生额外的开销，并且会在读取时检测并解决确实发生的任何（不太可能）冲突，然后进行清理。 JanusGraph通过以下功能使使用此策略变得容易。

#### Forking Edges 分叉边
Because edge are stored as single records in the underlying storage backend, concurrently modifying a single edge would lead to conflict. Instead of locking, an edge label can be configured to use `ConsistencyModifier.FORK`. The following example creates a new edge label `related` and defines its consistency to FORK.

由于边缘作为单个记录存储在基础存储后端中，因此同时修改单个边缘将导致冲突。代替锁定，可以将边缘标签配置为使用ConsistencyModifier.FORK。下面的示例创建一个新的相关边缘标签，并将其与FORK定义一致。
```
mgmt = graph.openManagement()
related = mgmt.makeEdgeLabel('related').make()
mgmt.setConsistency(related, ConsistencyModifier.FORK)
mgmt.commit()
```
When modifying an edge whose label is configured to FORK the edge is deleted and the modified edge is added as a new one. Hence, if two concurrent transactions modify the same edge, two modified copies of the edge will exist upon commit which can be resolved during querying traversals if needed.

修改标签配置为“fork”的边时，将删除该边，并将修改后的边作为新边添加。因此，如果两个并发事务修改了同一边缘，则提交时将存在边缘的两个修改后的副本，可以在查询遍历期间根据需要解决这些副本。

- Note
Edge forking only applies to MULTI edges. Edge labels with a multiplicity constraint cannot use this strategy since a constraint is built into the edge label definition that requires an explicit lock or use the conflict resolution mechanism of the underlying storage backend.

边缘分叉仅适用于MULTI边缘。 具有多重性约束的边缘标签无法使用此策略，因为在边缘标签定义中内置了一个约束，该约束需要显式锁定或使用基础存储后端的冲突解决机制。

#### Multi-Properties 多属性
Modifying single valued properties on vertices concurrently can result in a conflict. Similarly to edges, one can allow an arbitrary number of properties on a vertex for a particular property key defined with cardinality LIST and FORK on modification. Hence, instead of conflict one reads multiple properties. Since JanusGraph allows properties on properties, provenance information like `author` can be added to the properties to facilitate resolution at read time.

同时修改顶点上的单值属性可能会导致冲突。与边缘相似，对于修改时用基数LIST和FORK定义的特定属性键，可以在顶点上允许任意数量的属性。因此，读取多个属性而不是冲突。由于JanusGraph允许属性具有属性，因此可以将诸如author之类的出处信息添加到属性中，以促进读取时的解析。

See [multi-properties](https://docs.janusgraph.org/basics/schema/#property-key-cardinality) to learn how to define those.

## Data Inconsistency 数据不一致

### Temporary Inconsistency 暂时不一致
On eventually consistent storage backends, writes may not be immediately visible to the entire cluster causing temporary inconsistencies in the graph. This is an inherent property of eventual consistency, in the sense, that accepted updates must be propagated to other instances in the cluster and no guarantees are made with respect to read atomicity in the interest of performance.

在最终一致的存储后端上，整个集群可能不会立即看到写入，从而导致图中的暂时不一致。从某种意义上说，这是最终一致性的固有属性，必须将接受的更新传播到群集中的其他实例，并且为了性能而不能保证读取原子性。

From JanusGraph’s perspective, eventual consistency might cause the following temporary graph inconsistencies in addition the general inconsistency that some parts of a transaction are visible while others aren’t yet.

从JanusGraph的角度来看，最终的一致性可能会导致以下暂时的图形不一致，此外还存在交易的某些部分可见而其他部分还看不到的普遍不一致。

**Stale Index entries**索引条目

Index entries might point to nonexistent vertices or edges. Similarly, a vertex or edge appears in the graph but is not yet indexed and hence ignored by global graph queries.

索引条目可能指向不存在的顶点或边。同样，顶点或边出现在图形中，但尚未索引，因此被全局图形查询忽略。

**Half-Edges**半边

Only one direction of an edge gets persisted or deleted which might lead to the edge not being or incorrectly being retrieved. 

边缘的仅一个方向被保留或删除，这可能导致边缘未被或错误地检索。

- Note
In order to avoid that write failures result in permanent inconsistencies in the graph it is recommended to use storage backends that support batch write atomicity and to ensure that write atomicity is enabled. To get the benefit of write atomicity, the number modifications made in a single transaction must be smaller than the configured `buffer-size` option documented in [Configuration Reference](https://docs.janusgraph.org/basics/configuration-reference/). The buffer size defines the maximum number of modifications that JanusGraph will persist in a single batch. If a transaction has more modifications, the persistence will be split into multiple batches which are persisted individually which is useful for batch loading but invalidates write atomicity.

为了避免写失败导致图中的永久性不一致，建议使用支持批量写原子性的存储后端，并确保启用写原子性。为了获得写入原子性的好处，在单个事务中进行的修改数量必须小于《配置参考》中记录的配置的缓冲区大小选项。缓冲区大小定义JanusGraph将在单个批处理中保留的最大修改数量。如果事务有更多修改，则持久性将被分为多个批次，这些批次将分别保存，这对于批量加载很有用，但会使写入原子性无效。

### Ghost Vertices 幽灵顶点
A permanent inconsistency that can arise when operating JanusGraph on eventually consistent storage backend is the phenomena of **ghost vertices**. If a vertex gets deleted while it is concurrently being modified, the vertex might re-appear as a _ghost_.

The following strategies can be used to mitigate this issue:
- **Existence checks** 
  - Configure transactions to (double) check for the existence of vertices prior to returning them. Please see [Transaction Configuration](https://docs.janusgraph.org/basics/transactions/#transaction-configuration) for more information and note that this can significantly decrease performance. Note, that this does not fix the inconsistencies but hides some of them from the user.
- **Regular Clean-ups** 
  - Run regular batch-jobs to repair inconsistencies in the graph using [JanusGraph with TinkerPop’s Hadoop-Gremlin](https://docs.janusgraph.org/advanced-topics/hadoop/). This is the only strategy that can address all inconsistencies and effectively repair them. We will provide increasing support for such repairs in future versions of Faunus.
- **Soft Deletes**
  - Instead of deleting vertices, they are marked as deleted which keeps them in the graph for future analysis but hides them from user-facing transactions.

在最终一致的存储后端上操作JanusGraph时，可能会出现永久性的不一致现象，即幻影顶点现象。如果在同时修改顶点时删除了该顶点，则该顶点可能会重新显示为幻影。

可以使用以下策略来缓解此问题：
- **存在检查**
  - 将事务配置为（加倍）在返回顶点之前检查顶点是否存在。请参阅事务配置以获取更多信息，并注意这会大大降低性能。请注意，这不能解决不一致问题，但是会向用户隐藏其中的一些问题。
- **定期清理**
  - 使用JanusGraph和TinkerPop的Hadoop-Gremlin，运行常规的批处理作业以修复图中的不一致之处。这是可以解决所有不一致问题并有效修复它们的唯一策略。我们将在Faunus的未来版本中为此类维修提供越来越多的支持。
- **软删除**
  - 它们不会删除顶点，而是标记为已删除，从而将它们保留在图形中以供将来分析，但将其隐藏在面向用户的事务中。
