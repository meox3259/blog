---
title: RocksDB - SSTable
draft: true
tags:
  - C++
  - 分布式存储
  - rocksdb
---

# BlockBaseTable结构

BlockBaseTable大致结构如下图所示：
![SST](/pic/SST.jpg)
```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: index block]
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
[meta block 5: stats block]                   (see section: "properties" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

## Footer
Footer是一个固定48个Byte的块，用来进行最上层对于SST的索引，因为是固定大小所以可以直接取出。
Footer里的metaindex_handler指向了metaindex block，用于对metablock进行索引；index handler指向了data block，对data block进行了索引。因此Footer是索引的索引。

具体结构如下图所示
![Footer](/pic/Footer.png)

```
metaindex_handle: char[p];      // Block handle for metaindex
index_handle:     char[q];      // Block handle for index
padding:          char[40-p-q]; // zeroed bytes to make fixed length
                                // (40==2*BlockHandle::kMaxEncodedLength)
magic:            fixed64;      // 0x88e241b785f4cff7 (little-endian)
```
