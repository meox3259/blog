---
title: RocksDB - Table Cache
tags:
  - C++
  - 分布式存储
  - rocksdb
---

本文记录一些 RocksDB 中基本的数据结构、文件类型，以便之后阅读源代码时进行查阅以及理解。

# 接口

table cache有以下接口：
- GetTableReader
- FindTable
- NewIterator
- Get
- GetRangeTombstoneIterator
- MultiGetFilter

## GetTableReader

GetReader函数是TableCache中的核心函数，负责创建TableReader对象来读取SST文件。

### 参数

```cpp
Status GetTableReader(
    const ReadOptions& ro,                       // 读取选项
    const FileOptions& file_options,             // 文件选项
    const InternalKeyComparator& internal_comparator, // 内部键比较器
    const FileMetaData& file_meta,               // 文件元数据
    bool sequential_mode,                        // 是否顺序读取模式
    HistogramImpl* file_read_hist,               // 文件读取直方图统计
    std::unique_ptr<TableReader>* table_reader,  // 输出参数：表读取器
    const MutableCFOptions& mutable_cf_options,  // 可变列族选项
    bool skip_filters,                           // 是否跳过过滤器
    int level,                                   // SST文件所在层级
    bool prefetch_index_and_filter_in_cache,     // 是否预取索引和过滤器到缓存
    size_t max_file_size_for_l0_meta_pin,        // L0文件元数据固定的最大文件大小
    Temperature file_temperature)                // 文件温度（热、冷数据区分）
```

### 流程

首先获取当前SST的名字，名字的格式是{ColumnFamily_Path}/{number}_{sst}。
```cpp
std::string fname = TableFileName(
    ioptions_.cf_paths, file_meta.fd.GetNumber(), file_meta.fd.GetPathId());
```

然后读入文件I/O配置（哪些配置？）

```cpp
FileOptions fopts = file_options;
fopts.temperature = file_temperature;
Status s = PrepareIOFromReadOptions(ro, ioptions_.clock, fopts.io_options);
```

再下来尝试打开文件，如果以上命名方式方式无法查到，则换另一种方式打开。打开后记录文件统计信息
```cpp
if (s.ok()) {
  s = ioptions_.fs->NewRandomAccessFile(fname, fopts, &file, nullptr);
}
if (s.ok()) {
  RecordTick(ioptions_.stats, NO_FILE_OPENS);
} else if (s.IsPathNotFound()) {
  // 尝试使用另一种命名格式查找文件
  fname = Rocks2LevelTableFileName(fname);
  // ... 重试打开文件的代码 ...
}
```

如果成功打开文件，则进行对于文件读取的设置，目的在于增强文件读取的性能

```cpp
if (s.ok()) {
  if (!sequential_mode && ioptions_.advise_random_on_open) {
    file->Hint(FSRandomAccessFile::kRandom);
  }
  if (ioptions_.default_temperature != Temperature::kUnknown &&
      file_temperature == Temperature::kUnknown) {
    file_temperature = ioptions_.default_temperature;
  }
```
以上代码的Hint很有意思，其实底层调用了一个Linux的系统调用posix_fadvise，用处是调整预读策略或清理缓存。

rocksdb中一共配置了以下几种:POSIX_FADV_NORMAL，POSIX_FADV_RANDOM，POSIX_FADV_SEQUENTIAL，POSIX_FADV_WILLNEED，POSIX_FADV_DONTNEED。

Table Cache中用到了Random类型，用处是随机访问模式，禁止了prefetch，避免缓存无用数据。如果没有设置sequential_mode顺序读且设置了随机访问的advice，则开启RANDOM。

然后是创建文件读取器和性能监控，记录打开文件所需要的微秒数，同时具有统计、跟踪功能
```cpp
StopWatch sw(ioptions_.clock, ioptions_.stats, TABLE_OPEN_IO_MICROS);
std::unique_ptr<RandomAccessFileReader> file_reader(
    new RandomAccessFileReader(std::move(file), fname, ioptions_.clock,
                               io_tracer_, ioptions_.stats, SST_READ_MICROS,
                               file_read_hist, ioptions_.rate_limiter.get(),
                               ioptions_.listeners, file_temperature,
                               level == ioptions_.num_levels - 1));
```

然后将file_reader传入工厂类，开始构建
```cpp
s = mutable_cf_options.table_factory->NewTableReader(
    ro,
    TableReaderOptions(
        // 大量参数设置...
    ),
    std::move(file_reader), file_meta.fd.GetFileSize(), table_reader,
    prefetch_index_and_filter_in_cache);
```

### Table Factory
在TableCache中，Status TableCache::GetTableReader函数需要获取一个TableReader。这里使用了一个工厂模式的Table Factory实现。

调用最上层的基类为class TableFactory，继承自public Customizable，里面的方法都是纯虚函数。可以发现一共有三种创建方式：BlockBasedTable，PlainTable，CuckooTable。三种创建方式由class AdaptiveTableFactory这个类中的函数选定，也是继承自TableFactory，代码如下：
```cpp
Status AdaptiveTableFactory::NewTableReader(
    const ReadOptions& ro, const TableReaderOptions& table_reader_options,
    std::unique_ptr<RandomAccessFileReader>&& file, uint64_t file_size,
    std::unique_ptr<TableReader>* table,
    bool prefetch_index_and_filter_in_cache) const {
  Footer footer;
  IOOptions opts;
  auto s =
      ReadFooterFromFile(opts, file.get(), *table_reader_options.ioptions.fs,
                         nullptr /* prefetch_buffer */, file_size, &footer);
  if (!s.ok()) {
    return s;
  }
  if (footer.table_magic_number() == kPlainTableMagicNumber ||
      footer.table_magic_number() == kLegacyPlainTableMagicNumber) {
    return plain_table_factory_->NewTableReader(
        table_reader_options, std::move(file), file_size, table);
  } else if (footer.table_magic_number() == kBlockBasedTableMagicNumber ||
             footer.table_magic_number() == kLegacyBlockBasedTableMagicNumber) {
    return block_based_table_factory_->NewTableReader(
        ro, table_reader_options, std::move(file), file_size, table,
        prefetch_index_and_filter_in_cache);
  } else if (footer.table_magic_number() == kCuckooTableMagicNumber) {
    return cuckoo_table_factory_->NewTableReader(
        table_reader_options, std::move(file), file_size, table);
  } else {
    return Status::NotSupported("Unidentified table format");
  }
}
```
可以发现，这里的判断是基于footer.table_magic_number的。

然后是构建过程，这里直接new一个PlainTableBuilder进行构建
构造函数接收大量参数，主要包括：
- 配置选项：ioptions和moptions（不可变和可变的列族选项）
- 文件信息：file（可写文件）、file_number（文件编号）
- 编码相关：user_key_len（用户键长度）、encoding_type（编码类型）
- 索引相关：index_sparseness（索引稀疏度）、store_index_in_file（是否将索引存储在文件中）
- 布隆过滤器相关：bloom_bits_per_key（每个键的位数）、num_probes（探测次数）
- 哈希表相关：hash_table_ratio（哈希表比率）、huge_page_tlb_size（大页TLB大小）
- 标识信息：column_family_id、column_family_name、db_id、db_session_id

## Table Builder
以下翻译自Table Builder上所加的注释，可以大致知道Table Builder使用的位置：
```
返回一个表构建器（table builder），用于将此表类型的数据写入文件。该函数在以下场景被调用：
(1) 当将内存表（memtable）刷新（flush）到 Level-0 层的输出文件时，
通过调用 BuildTable() 创建表构建器（见 DBImpl::WriteLevel0Table()）。
(2) 在压缩（compaction）过程中，通过 DBImpl::OpenCompactionOutputFile() 
 获取构建器以写入压缩输出文件。
(3) 从事务日志（transaction logs）恢复数据时，通过调用 BuildTable() 
创建表构建器以写入 Level-0 层的输出文件（见 DBImpl::WriteLevel0TableForRecovery）。
(4) 在运行修复工具（Repairer）时，通过调用 BuildTable() 创建表构建器，将日志转换为 SST 文件（见 Repairer::ConvertLogToTable()）。
可通过此函数访问多项配置参数，包括但不限于压缩选项（compression options）。
参数 `file` 是一个可写文件（writable file）的句柄。
调用方需负责保持文件处于打开状态，并在表构建器关闭后关闭该文件。
参数 `compression_type` 表示此表中使用的压缩类型。
```

Table Builder的逻辑仅仅是调用一下::Open函数，这里以BlockBasedTable::Open作为例子。

## Open

```cpp
const bool prefetch_all = prefetch_index_and_filter_in_cache || level == 0;
const bool preload_all = !table_options.cache_index_and_filter_blocks;

if (!ioptions.allow_mmap_reads && !env_options.use_mmap_reads) {
  s = PrefetchTail(/* 参数... */);
  // 检查错误
} else {
  // 内存映射模式不需要预取
  prefetch_buffer.reset(new FilePrefetchBuffer(/* 参数... */));
}
```

首先判断是否需要prefetch所有索引和过滤器，结果丢到下面的PrefetchTail使用。
- 根据传入的参数决定是否要prefetch，如果是L0层的文件则必须prefetch。

然后是有关mmap的设置，众所周知mmap是用来直接映射内存的，这里对于是否使用mmap对预取有不同的影响：
- 如果不用mmap，则PrefetchTail。
- 如果用了，则建立一个Prefetch buffer。

接下来是读取Table里的内容，注释中给出了顺序：
```
1. Footer
2. [metaindex block]
3. [meta block: properties]
4. [meta block: range deletion tombstone]
5. [meta block: compression dictionary]
6. [meta block: index]
7. [meta block: filter]
```

由于Footer是索引的索引，因此显然我们需要先取出Footer，再取出二级索引metaindex，进而取出meta block。