# 数据模型 JanusGraph Data Model

JanusGraph stores graphs in [adjacency list format](https://en.wikipedia.org/wiki/Adjacency_list) which means that a graph is stored as a collection of vertices with their adjacency list. The adjacency list of a vertex contains all of the vertex’s incident edges (and properties).

By storing a graph in adjacency list format JanusGraph ensures that all of a vertex’s incident edges and properties are stored compactly in the storage backend which speeds up traversals. The downside is that each edge has to be stored twice - once for each end vertex of the edge.

In addition, JanusGraph maintains the adjacency list of each vertex in sort order with the order being defined by the sort key and sort order the edge labels. The sort order enables efficient retrievals of subsets of the adjacency list using [vertex centric indices](https://docs.janusgraph.org/advanced-topics/data-model/#vertex-indexes).

JanusGraph stores the adjacency list representation of a graph in any [storage backend](https://docs.janusgraph.org/advanced-topics/data-model/#storage-backends) that supports the Bigtable data model.

JanusGraph以邻接表格式存储图，这意味着一个图以其邻接表存储为一组顶点。顶点的邻接列表包含所有顶点的入射边（和属性）。

通过以邻接表格式存储图形，JanusGraph可确保将所有顶点的入射边和属性紧凑地存储在存储后端中，从而加快遍历速度。不利的一面是每个边缘必须存储两次-边缘的每个顶点都存储一次。

另外，JanusGraph维护每个顶点的邻接列表，并按排序顺序（由排序键定义的顺序和边缘标签的排序顺序）进行维护。排序顺序可以使用顶点中心索引来有效检索邻接表的子集。

JanusGraph在支持Bigtable数据模型的任何存储后端中存储图的邻接列表表示。

## Bigtable 数据模型 Bigtable Data Model
![](https://cdn.nlark.com/yuque/0/2020/png/1209774/1606527970964-230f596f-b5d7-4eb1-9fd0-9b852241d6ad.png#align=left&display=inline&height=229&margin=%5Bobject%20Object%5D&originHeight=229&originWidth=927&size=0&status=done&style=none&width=927)

Under the [Bigtable data model](https://en.wikipedia.org/wiki/Bigtable) each table is a collection of rows. Each row is uniquely identified by a key. Each row is comprised of an arbitrary (large, but limited) number of cells. A cell is composed of a column and value. A cell is uniquely identified by a column within a given row. Rows in the Bigtable model are called "wide rows" because they support a large number of cells and the columns of those cells don’t have to be defined up front as is required in relational databases.

JanusGraph has an additional requirement for the Bigtable data model: The cells must be sorted by their columns and a subset of the cells specified by a column range must be efficiently retrievable (e.g. by using index structures, skip lists, or binary search).

In addition, a particular Bigtable implementation may keep the rows sorted in the order of their key. JanusGraph can exploit such key-order to effectively partition the graph which provides better loading and traversal performance for very large graphs. However, this is not a requirement.

在Bigtable数据模型下，每个表都是行的集合。每行由一个键唯一标识。每行都包含任意（大但数量有限）的单元格。单元格由列和值组成。单元格由给定行中的列唯一标识。 Bigtable模型中的行称为“宽行”，因为它们支持大量的单元格，并且不必像关系数据库中那样预先定义这些单元格的列。

JanusGraph对Bigtable数据模型有一个附加要求：单元必须按其列排序，并且必须有效地检索由列范围指定的单元的子集（例如，通过使用索引结构，跳过列表或二进制搜索）。

另外，一个特定的Bigtable实现可能会使行按其键的顺序排序。 JanusGraph可以利用这种键顺序来有效地划分图形，从而为非常大的图形提供更好的加载和遍历性能。但是，这不是必需的。

## 数据布局 JanusGraph Data Layout 
![](https://cdn.nlark.com/yuque/0/2020/png/1209774/1606527970146-53f0a61a-4418-4f33-906b-04f23c2e0170.png#align=left&display=inline&height=226&margin=%5Bobject%20Object%5D&originHeight=226&originWidth=926&size=0&status=done&style=none&width=926)

JanusGraph stores each adjacency list as a row in the underlying storage backend. The (64 bit) vertex id (which JanusGraph uniquely assigns to every vertex) is the key which points to the row containing the vertex’s adjacency list. Each edge and property is stored as an individual cell in the row which allows for efficient insertions and deletions. The maximum number of cells allowed per row in a particular storage backend is therefore also the maximum degree of a vertex that JanusGraph can support against this backend.

If the storage backend supports key-order, the adjacency lists will be ordered by vertex id, and JanusGraph can assign vertex ids such that the graph is effectively partitioned. Ids are assigned such that vertices which are frequently co-accessed have ids with small absolute difference.

JanusGraph将每个邻接列表在底层存储后端中存储为一行。 （64位）顶点ID（JanusGraph唯一地为其分配给每个顶点）是指向包含顶点邻接表的行的键。每个边缘和属性都作为单独的单元格存储在行中，从而可以高效地进行插入和删除。因此，特定存储后端中每行允许的最大单元数也是JanusGraph可以针对该后端支持的顶点的最大程度。

如果存储后端支持键顺序，则邻接列表将按顶点ID排序，并且JanusGraph可以分配顶点ID，以便有效地对图形进行分区。分配ID，以使经常被同时访问的顶点的ID的绝对差很小。

## 个别边缘布局 Individual Edge Layout 
![](https://cdn.nlark.com/yuque/0/2020/png/1209774/1606527969944-0f0d1f5c-fdfb-45ec-825f-7d90376e618c.png#align=left&display=inline&height=211&margin=%5Bobject%20Object%5D&originHeight=211&originWidth=890&size=0&status=done&style=none&width=890)

每个边缘和属性都作为一个单元格存储在其相邻顶点的行中。它们被序列化，以使列的字节顺序尊重边缘标签的排序键。可变id编码方案和压缩对象序列化用于使每个边缘/单元的存储占用空间尽可能小。

如上图的第一行所示，考虑单个边的存储布局。深蓝色框表示使用可变长度编码方案编码的数字，以减少其消耗的字节数。红色框代表一个或多个属性值（即对象），这些属性值已使用关联的属性键中引用的压缩元数据序列化。灰色框代表未压缩的属性值（即序列化的对象）。

边缘的序列化表示形式以边缘标签的唯一ID（由JanusGraph分配）开头。这通常是一个很小的数字，并使用可变id编码很好地压缩。此ID的最后一位偏移以存储它是传入还是传出边缘。接下来，存储包括排序关键字的属性值。排序键是使用边缘标签定义的，因此可以将排序键对象元数据引用到边缘标签。之后，将存储相邻顶点的id。 JanusGraph不存储实际的顶点ID，而是与拥有该邻接表的顶点ID的差。差异可能比绝对id小，因此压缩效果更好。顶点ID后跟该边的ID。JanusGraph为每个边缘分配了唯一的ID。这样就得出了边缘单元格的列值。边单元的值包含边 signature 签名属性的压缩序列化（由标签的signature签名密钥定义），以及在未压缩序列化中已添加到边缘的任何其他属性。

属性的序列化表示更为简单，并且在列中仅包含属性的键ID。属性ID和属性值存储在值中。但是，如果将属性键定义为list（），则属性ID也存储在该列中。

Each edge and property is stored as one cell in the rows of its adjacent vertices. They are serialized such that the byte order of the column respects the sort key of the edge label. Variable id encoding schemes and compressed object serialization are used to keep the storage footprint of each edge/cell as small as possible.

Consider the storage layout of an individual edge as visualized in the top row of the graphic above. The dark blue boxes represent numbers that are encoded with a variable length encoding scheme to reduce the number of bytes they consume. Red boxes represent one or multiple property values (i.e. objects) that are serialized with compressed meta data referenced in the associated property key. Grey boxes represent uncompressed property values (i.e. serialized objects).

The serialized representation of an edge starts with the edge label’s unique id (as assigned by JanusGraph). This is typically a small number and compressed well with variable id encoding. The last bit of this id is offset to store whether this is an incoming or outgoing edge. Next, the property value comprising the sort key are stored. The sort key is defined with the edge label and hence the sort key objects meta data can be referenced to the edge label. After that, the id of the adjacent vertex is stored. JanusGraph does not store the actual vertex id but the difference to the id of the vertex that owns this adjacency list. It is likely that the difference is a smaller number than the absolute id and hence compresses better. The vertex id is followed by the id of this edge. Each edge is assigned a unique id by JanusGraph. This concludes the column value of the edge’s cell. The value of the edge’s cell contains the compressed serialization of the signature properties of the edge (as defined by the label’s signature key) and any other properties that have been added to the edge in uncompressed serialization.

The serialized representation of a property is simpler and only contains the property’s key id in the column. The property id and the property value are stored in the value. If the property key is defined as `list()`, however, the property id is stored in the column as well.
