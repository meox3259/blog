---
title: RocksDB - MemTable
draft: true
tags:
  - C++
  - 分布式存储
  - rocksdb
---

MemTable是一个存在内存中的数据结构，之后会被刷盘成为SST。MemTable用于数据被刷写到磁盘SST文件前临时存储和处理读写请求。

# 流程
1. MemTable负责接受所有新的写入操作，并作为读操作第一层查询。写入时，数据会先追加写到WAL保证持久性，然后再插入到MemTable里。读完MemTable查找不到后再去SST里查询。

2. MemTable基于高效的有序数据结构实现，目前使用了跳表。跳表中数据按照Key有序排序，支持快速插入和查询操作。

3. 当MemTable达到阈值时（Rocksdb预设为4Mb），会转为Immutable MemTable（只读状态），然后由后台线程刷盘变成L0的SST。

4. 当MemTable刷盘完成后，其中数据变为可靠的，因此可以从WAL中删除对应数据，释放内存空间。此机制平衡了内存和磁盘的负载，通过额外线程异步刷盘避免了写入阻塞，且可以通过WAL恢复MemTable中的数据。

