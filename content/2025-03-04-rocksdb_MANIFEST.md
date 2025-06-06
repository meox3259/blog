---
title: RocksDB - MANIFEST
draft: false
tags:
  - C++
  - 分布式存储
---

本文记录一些 RocksDB 中基本的数据结构、文件类型，以便之后阅读源代码时进行查阅以及理解。

## MANIFEST 文件详解

`MANIFEST` 文件本质上是一个记录数据库状态变化的日志文件。它存储了数据库从创建到当前状态的所有重要变更记录。这些记录以 `VersionEdit` 对象的形式存在。

### 恢复数据

当 `RocksDB` 启动时，它需要知道所有活跃的 `SST` 文件和其所在的层级。`MANIFEST` 文件提供了这个信息，使数据库能够从上次关闭的状态准确恢复。

当然也需要获取正确的MANIFEST文件，获取正确文件的索引存储在`CURRENT`文件

**数据库恢复过程**：
- 读取 `CURRENT` 文件找到最新的 MANIFEST 文件
- 顺序读取 MANIFEST 中所有 `VersionEdit` 记录
- 从空状态开始，挨个应用每个 `VersionEdit` 对应的状态，最终重建数据库的状态
- 构建完整的 `VersionSet` 和当前的 `Version`

### 状态的持久化记录

每次 RocksDB 的状态发生变化（新文件创立、文件被删除、压缩），这些变化都会通过 `VersionEdit` 记录到 `MANIFEST` 文件中。这种方式确保了即使系统崩溃，数据库也能恢复到崩溃前的一致状态。

### MVCC 支持

记录所有版本为 MVCC（多版本并发控制）提供了基础，使得数据库能够维护多个版本。

一个 MANIFEST 里维护的数据格式大致如下：

```protobuf
VersionEdit {
  LogNumber: 1
  NextFileNumber: 2
  LastSequence: 0
}
VersionEdit {
  NewFile: level=0, file=1, size=100, smallest_key="a", largest_key="z"
  NextFileNumber: 3
  LastSequence: 100
}
VersionEdit {
  NewFile: level=0, file=2, size=120, smallest_key="c", largest_key="y"
  NextFileNumber: 4
  LastSequence: 200
}
VersionEdit {
  NewFile: level=1, file=3, size=210, smallest_key="a", largest_key="z"
  DeletedFile: level=0, file=1
  DeletedFile: level=0, file=2
  NextFileNumber: 5
  LastSequence: 200
}
```

该文件代表了以下操作：
1. 创建数据库
2. 写入数据，生成 SST 文件，`000001.sst` 在 Level-0
3. 继续写入，生成 `000002.sst` 在 Level-0
4. 压缩这两个文件到 Level-1，生成 `000003.sst`
5. 删除 `000001.sst` 和 `000002.sst`

可以发现对于一个新文件（SST），里面记录了若干 `VersionEdit`。一个 `VersionEdit` 需要记录的信息有:

| 字段 | 描述 |
|------|------|
| `level` | 当前所处层级 |
| `file` | 文件序号 |
| `size` | 文件大小 |
| `smallest_key` & `largest_key` | 该文件包含的最小key和最大key |

> **注意**：MANIFEST 文件的正确维护对 RocksDB 的稳定性和数据一致性至关重要。