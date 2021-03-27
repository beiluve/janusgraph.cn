# 图分区 Graph Partitioning

When JanusGraph is deployed on a cluster of multiple storage backend instances, the graph is partitioned across those machines. Since JanusGraph stores the graph in an adjacency list representation the assignment of vertices to machines determines the partitioning. By default, JanusGraph uses a random partitioning strategy that randomly assigns vertices to machines. Random partitioning is very efficient, requires no configuration, and results in balanced partitions. Currently explicit partitioning is not supported.

当JanusGraph部署在具有多个存储后端实例的群集上时，该图将在这些计算机之间分区。由于JanusGraph将图形存储在邻接列表表示中，因此将顶点分配给计算机将确定分区。默认情况下，JanusGraph使用随机分区策略，该策略随机将顶点分配给计算机。随机分区非常有效，不需要配置，因此可以实现均衡分区。当前不支持显式分区。
```
cluster.max-partitions = 32
id.placement = simple
```
The configuration option `max-partitions` controls how many virtual partitions JanusGraph creates. This number should be roughly twice the number of storage backend instances. If the cluster of storage backend instances is expected to grow, estimate the size of the cluster in the foreseeable future and take this number as the baseline. Setting this number too large will unnecessarily fragment the cluster which can lead to poor performance. This number should be larger than the maximum expected number of nodes in the JanusGraph graph. It must be greater than 1 and a power of 2.

There are two aspects to graph partitioning which can be individually controlled: edge cuts and vertex cuts.

配置选项max-partitions控制JanusGraph创建多少个虚拟分区。该数目应该大约是存储后端实例数目的两倍。如果存储后端实例的群集预计会增长，请在可预见的将来估计群集的大小，并以该数字为基准。将此数字设置得太大会不必要地使群集破碎，从而导致性能下降。此数字应大于JanusGraph图中的最大预期节点数。它必须大于1且为2的幂。

图形分区有两个方面可以单独控制：边切割和顶点切割。
<a name="edge-cut"></a>

## Edge Cut 边切割
In assigning vertices to partitions one strives to optimize the assignment such that frequently co-traversed vertices are hosted on the same machine. Assume vertex A is assigned to machine 1 and vertex B is assigned to machine 2. An edge between the vertices is called a **cut edge** because its end points are hosted on separate machines. Traversing this edge as part of a graph query requires communication between the machines which slows down query processing. Hence, it is desirable to reduce the edge cut for frequently traversed edges. That, in turn, requires placing the adjacent vertices of frequently traversed edges in the same partition.

Vertices are placed in a partition by way of the assigned vertex id. A partition is essentially a sequential range of vertex ids. To place a vertex in a particular partition, JanusGraph chooses an id from the partition’s range of vertex ids. JanusGraph controls the vertex-to-partition assignment through the configured placement strategy. By default, vertices created in the same transaction are assigned to the same partition. This strategy is easy to reason about and works well in situations where frequently co-traversed vertices are created in the same transaction - either by optimizing the loading strategy to that effect or because vertices are naturally added to the graph that way. However, the strategy is limited, leads to imbalanced partitions when data is loaded in large transactions and not the optimal strategy for many use cases. The user can provide a use case specific vertex placement strategy by implementing the `IDPlacementStrategy` interface and registering it in the configuration through the `ids.placement` option.

When implementing `IDPlacementStrategy`, note that partitions are identified by an integer id in the range from 0 to the number of configured virtual partitions minus 1. For our example configuration, there are partitions 0, 1, 2, 3, ..31. Partition ids are not the same as vertex ids. Edge cuts are more meaningful when the JanusGraph servers are on the same hosts as the storage backend. If you have to make a network call to a different host on each hop of a traversal, the benefit of edge cuts and custom placement strategies can be largely nullified.

在将顶点分配给分区时，人们努力优化分配，以使频繁共遍历的顶点托管在同一台计算机上。假定将顶点A分配给机器1，将顶点B分配给机器2。由于两个顶点之间的端点位于单独的机器上，因此它们之间的边称为切边。作为图查询的一部分遍历此边缘需要机器之间的通信，这会减慢查询处理。因此，期望减少频繁移动的边缘的边缘切割。这进而需要将频繁遍历的边缘的相邻顶点放置在同一分区中。

顶点通过分配的顶点ID放置在分区中。分区本质上是顶点ID的顺序范围。要将顶点放置在特定分区中，JanusGraph从分区的顶点ID范围中选择一个ID。 JanusGraph通过配置的放置策略控制顶点到分区的分配。默认情况下，在同一事务中创建的顶点将分配给同一分区。这种策略很容易推论，并且在同一事务中频繁创建同遍顶点的情况下效果很好-可以通过优化加载策略来达到这种效果，或者因为顶点自然会以这种方式添加到图形中。但是，该策略是有限的，在大事务中加载数据时会导致分区不平衡，而不是许多用例的最佳策略。用户可以通过实现IDPlacementStrategy接口并通过ids.placement选项在配置中注册来提供特定于用例的顶点放置策略。

实施 IDPlacementStrategy 时，请注意，分区由整数ID标识，范围为0到配置的虚拟分区数减1。在我们的示例配置中，分区为0、1、2、3，.. 31。分区ID与顶点ID不同。当JanusGraph服务器与存储后端位于同一主机上时，边缘削减更有意义。如果必须在遍历的每个跃点上与另一台主机进行网络调用，则可以大幅减少边沿切割和自定义放置策略的优势。

## Vertex Cut 点切割
While edge cut optimization aims to reduce the cross communication and thereby improve query execution, vertex cuts address the hotspot issue caused by vertices with a large number of incident edges. While [vertex-centric indexes](https://docs.janusgraph.org/index-management/index-performance/#vertex-centric-indexes) effectively address query performance for large degree vertices, vertex cuts are needed to address the hot spot issue on very large graphs.

Cutting a vertex means storing a subset of that vertex’s adjacency list on each partition in the graph. In other words, the vertex and its adjacency list is partitioned thereby effectively distributing the load on that single vertex across all of the instances in the cluster and removing the hot spot.

JanusGraph cuts vertices by label. A vertex label can be defined as _partitioned_ which means that all vertices of that label will be partitioned across the cluster in the manner described above.

边沿切割优化旨在减少交叉通信并由此改善查询执行，而边沿切割解决了由具有大量入射边的顶点引起的热点问题。尽管以顶点为中心的索引有效地解决了大角度顶点的查询性能，但仍需要进行顶点切割以解决非常大的图上的热点问题。

切割顶点意味着将顶点邻接列表的子集存储在图中的每个分区上。换句话说，对顶点及其邻接列表进行了分区，从而有效地将单个顶点上的负载分布在群集中的所有实例上，并删除了热点。

JanusGraph按标签切割顶点。顶点标签可以定义为分区的，这意味着该标签的所有顶点将以上述方式在整个群集中分区。
```
mgmt = graph.openManagement()
mgmt.makeVertexLabel('user').make()
mgmt.makeVertexLabel('product').partition().make()
mgmt.commit()
```
In the example above, `product` is defined as a partitioned vertex label whereas `user` is a normal label. This configuration is beneficial for situations where there are thousands of products but millions of users and one records transactions between users and products. In that case, the product vertices will have a very high degree and the popular products turns into hot spots if they are not partitioned.

在上面的示例中，产品定义为分区的顶点标签，而用户是普通标签。此配置对于存在数千个产品但数百万个用户且一个记录用户和产品之间的交易的情况非常有用。在这种情况下，产品顶点将具有很高的度，并且如果不进行分区，受欢迎的产品将成为热点。

## Graph Partitioning FAQ
### Random vs. Explicit Partitioning 随机分区与显式分区
When the graph is small or accommodated by a few storage instances, it is best to use random partitioning for its simplicity. As a rule of thumb, one should strongly consider enabling explicit graph partitioning and configure a suitable partitioning heuristic when the graph grows into the 10s of billions of edges.

当图形较小或由几个存储实例容纳时，为简单起见，最好使用随机分区。 根据经验，当图增长到数十亿个边时，应该强烈考虑启用显式图分区并配置合适的分区启发式方法。
