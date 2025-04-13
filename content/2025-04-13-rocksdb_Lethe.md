---
title: RocksDB - Lethe & Rocksdb中针对delete的优化
draft: false
tags:
  - C++
  - 分布式存储
  - rocksdb
---

`lsm tree`是一个针对写优化的数据结构，所有写都是追加的形式，因此`delete`也是`out-of-place-update`的，显然这样会导致`delete`存在一定的性能问题。

# 性能问题

首先对`lsm tree`中的删除方式进行总结
- `Point delete`
  - 插入`tombstone`
  - 写放大（所有`tombstone`到最后一层才会被删除）
  - 空间放大（`tombstone`会占据额外的空间，直到一段时间后才会被删除）
- `Range delete`
  - 全量插入所有的`tombstone`
  - 大量的`tombstone`导致性能变差
- `Secondary Delete`
  - 按非主键进行删除
  - 由于`lsm tree`中元素按照主键排序，因此对于一个范围的`Secondary Key`，不连续地分布在很多`SST`上，因此需要进行全量`compaction`。显然单次巨大的`compaction`导致集中出现大量`I/O`，导致出现毛刺，影响可用性

综上所述，可以发现`delete`存在以下问题
- `delete tombstone`带来的读/写/空间放大
- `range delete`每删除一个`kv`，都需要一个`delete tombstone`，导致空间放大
- `Secondary delete`需要进行全量`compaction`

# Rocksdb优化
## Point delete
**方法1**：在到达最后一层之前，提前清理`delete tombstone`
- 在`compaction`期间，处理`delete tombstone`时，如果发现更高层的`SST`里`key range`和`delete tombstone`没有重叠，则说明这些`delete tombstone`没有用了，直接删除。

**方法2**：优先`compaction`含`tombstone`率高的文件
- 一般情况下，`Rocksdb`会选择一个和下层重叠较少的`SST`进行`compaction`来减小写放大。对于`delete heavy`的场景，考虑对这个策略进行修改，优先对`tombstone`多的`SST`进行`compaction`。由于`SST`的元数据中存储了`tombstone`的数量，所以可以不通过额外代价计算出来。

**方法3**：该方法基于特定无修改的场景，如果对于一个`Key`插入后不再修改，那么可以发现`delete`可以和已有的`Key`直接抵消。

## Range delete
显然将`Range delete`中所有都一个一个加入`tombstone`是一个不好的做法。[Rocksdb](https://rocksdb.org/blog/2018/11/21/delete-range.html)中提到了当前对于`Range delete`的实现及优化。

原本的`Range delete`一般有以下几种方法
- `scan-and-delete`：`scan`一遍要删除的键，挨个删除，这样显然较慢，因为磁盘`I/O`不友好，同时产生大量`tombstone`。
- `compaction filter`：通过设置`compaction filter`在`compaction`的时候异步地删除。问题是仍然在非最下层会存在`tombstone`，同时也无法保证`readers`一定无法看见在`delete range`中的`key`。
- `compact range`：是一个`rocksdb`中的`api`，会手动触发`compact`，删除一定范围内的`key`。这样可能会导致毛刺，同时阻塞其他`compaction`任务。

在上文的官方文档中提到的方法，大致方法是同样地在`SST`中的元数据区域保存删除标记，在进行`compaction`的时候，对这些删除的区间进行合并，使得这些删除的区间形成和`SST`中键类似的区间，且不相交。由于还存在快照，因此还需要考虑`seqnum`的存储，因此一个删除标记应该保存成一个三元组的形式`(start_key, end_key, seqnum)`，因此可以视为一个二维平面上的长方形。为了对保存的数据大小尽量压缩，因此分割的方法实际上是矩形并，将这些长方形进行求并，然后将端点存储下来。具体图示如下
![图1](https://im.gurl.eu.org/file/AgACAgEAAxkDAAJCT2f7pLcK7dQMOMzW7atj3gxsHZeFAALUrTEbanfZR-FQSNZFBtqiAQADAgADeQADNgQ.png)

![图2](https://im.gurl.eu.org/file/AgACAgEAAxkDAAJCUGf7pSI6dWS4lFAW3mbXtQatvtD4AALVrTEbanfZRzdsmip3quLNAQADAgADeQADNgQ.png)

观察以上图片，可以发现存储的方式只用存储图2中标记的红点。具体查询的时候，只需要查询一个`(key, seq)`是否在这些长方形形成的图形内部。

## Secondary delete
可以发现，以上两种优化并没有解决`Secondary delete`的问题。`Lethe`对该场景进行了优化，例如需要按时间戳而非`Key`删除一定范围内的值。

### FADE
`FADE`实际上是一种`compaction`策略，针对`tombstone`进行优化
- 对每个`tombstone`设立一个`TTL`，其中从上到下，每层的`TTL`是上一层的`T`倍。
- 计算每层超过`TTL`的`tombstone`数量，并计算一个总和，每次`compaction`选择超过`TTL`最多的层，把其中超时的`tombstone`全部`compaction`到下一层。

### KiWi
`KiWi`是一种针对`SST`结构的更改。原本`SST`的结构是`SST(file)->Block(page)`，文件内部按照主键排序，`KiWi`修改了这个结构，改成了`SST(file)->delete tile->Block(page)`的三层结构，其中`delete tile`是按照`delete key`进行排序的，而`Block`内部和`delete tile`之间则还是主键。

这样的好处比较显然，即可以让`delete key`变得有序，而不至于像之前一样是完全无序的。有序的程度可以自己进行配置，即`h`，表示一个`delete tile`内部包含`h`个`Block(page)`。

论文中给了一个例子如何计算`h`
- 有一个大小为`400G`的数据库，`page`大小为`4KB`，在两个`range delete`之间进行`50M`次点查，`10K`个短的区间查询，且`bloom filter`的假阳性率为`0.02`，且`level`之前的大小比例`T=10`
- `h`的计算方法为
$$h \leq \frac{400GB / 4KB}{5 \cdot 10^7  \cdot 2 \cdot 10^{-2} + 10^{4} \cdot log_{10}{(\frac{400GB}{4KB})}} \approx 10^2$$

考虑删除的过程，按顺序取出`delete tile`，由于`delete tile`内部`Block`之间是按`delete key`有序的，因此对于`Key`的范围在`delete range`内的`block`，直接`drop`。

这样存在的坏处是查询时，相比原来的`Iterator`能一次直接定位到对应的`block`，由于`delete tile`内部不是按照主键排序，因此需要访问多个`block`。同样由于`block`之间是无序的，因此`KiWi`需要每个`block`有一个`bloom filter`，而`rocksdb`中一个`SST`才有一个`bloom filter`。不过由于`bloom filter`的存在，如果假阳性率不搞，且由于`bloom filter`是内存操作，因此性能下降的并不厉害。

![图3](https://im.gurl.eu.org/file/AgACAgEAAxkDAAJCU2f7r_9_6W15w43KVyHa4MCtfaOHAALZrTEbanfZR0ImTxRFWa2FAQADAgADeAADNgQ.png)

可以从上图发现，对于`bloom filter`性能下降了，但是磁盘`I/O`的性能显著提升了，因此总而言之性能有明显提升。