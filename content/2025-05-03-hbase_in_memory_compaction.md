---
title: HBase - InMemoryCompaction优化
tags:
  - 分布式存储
---

学习一下`InMemoryCompaction`优化。

# 读取流程
- 活跃段`(MutableSegment)`以跳表方式扫描
- `CellArrayImmutableSegment`以二分查找方式扫描
- `CSLMImmutableSegment`以跳表方式扫描

数据的流转过程大概如下
`MutableSegment  →  ImmutableSegment(s)  →  HFile`

# 流程

## getScanner
由于都是类`LSM`架构，因此方法类似，这里定义了一个类似`Iterator`的`Scanner`。
```java
public KeyValueScanner getScanner(Scan scan, final NavigableSet<byte[]> targetCols, long readPt)
    throws IOException {
    storeEngine.readLock();
    try {
      ScanInfo scanInfo;
      if (this.getCoprocessorHost() != null) {
        scanInfo = this.getCoprocessorHost().preStoreScannerOpen(this, scan);
      } else {
        scanInfo = getScanInfo();
      }
      return createScanner(scan, scanInfo, targetCols, readPt);
    } finally {
      storeEngine.readUnlock();
    }
  }
```
传入的参数`scan`和`targetCols`和`readPt`
- `scan`
  - `private byte[] startRow = HConstants.EMPTY_START_ROW;`        // 起始行
  - `private boolean includeStartRow = true;`                      // 是否包含起始行
  - `private byte[] stopRow = HConstants.EMPTY_END_ROW;`           // 结束行
  - `private boolean includeStopRow = false;`                      // 是否包含结束行
  - `private int maxVersions = 1;`                                 // 最大版本数
  - `private int batch = -1;`                                      // 批处理大小
  - `private boolean allowPartialResults = false;`                 // 是否允许部分结果
  - `private int storeLimit = -1;`                                 // 每个列族返回的最大结果数
  - `private int storeOffset = 0;`                                 // 每个列族的行偏移量
  - `private int caching = -1;`                                    // 扫描器缓存大小
  - `private long maxResultSize = -1;`                             // 最大结果大小
  - `private boolean cacheBlocks = true;`                          // 是否缓存块
  - `private boolean reversed = false;`                            // 是否逆序扫描
  - `private TimeRange tr = TimeRange.allTime();`                  // 时间范围
  - `private Map<byte[], NavigableSet<byte[]>> familyMap;`         // 列族和列的映射
  - `private int limit = -1;`                                      // 行数限制
  - `private ReadType readType = ReadType.DEFAULT;`                // 读取类型
  - `private boolean needCursorResult = false;`                    // 是否需要游标结果
- `long readPt：`
  - `MVCC`(多版本并发控制)读取点，是数据一致性的关键参数
首先需要加一个读锁，保证存储引擎被锁住，结构不发生变化。

然后是创建一个新的`scanner`
```java
protected KeyValueScanner createScanner(Scan scan, ScanInfo scanInfo,
    NavigableSet<byte[]> targetCols, long readPt) throws IOException {
    return scan.isReversed()
      ? new ReversedStoreScanner(this, scanInfo, scan, targetCols, readPt)
      : new StoreScanner(this, scanInfo, scan, targetCols, readPt);
    }
```

## HStore
- `StoreScanner`是对于`HStore`的`Scanner`，介于`RegionScanner`和`KeyValueScanner`之上，对应单个`cf`的数据。
- 合并多个`Memstore`和`HFile`的数据，保存`TTL`规则，确定返回一致性视图的数据。

```java
public StoreScanner(HStore store, ScanInfo scanInfo, Scan scan, NavigableSet<byte[]> columns,
    long readPt) throws IOException {
    this(store, scan, scanInfo, columns != null ? columns.size() : 0, readPt, scan.getCacheBlocks(),
      ScanType.USER_SCAN);
    if (columns != null && scan.isRaw()) {
      throw new DoNotRetryIOException("Cannot specify any column for a raw scan");
    }
    matcher = UserScanQueryMatcher.create(scan, scanInfo, columns, oldestUnexpiredTS, now,
      store.getCoprocessorHost());

    store.addChangedReaderObserver(this);

    List<KeyValueScanner> scanners = null;
    try {
      // Pass columns to try to filter out unnecessary StoreFiles.
      scanners = selectScannersFrom(store,
        store.getScanners(cacheBlocks, scanUsePread, false, matcher, scan.getStartRow(),
          scan.includeStartRow(), scan.getStopRow(), scan.includeStopRow(), this.readPt,
          isOnlyLatestVersionScan(scan)));

      // Seek all scanners to the start of the Row (or if the exact matching row
      // key does not exist, then to the start of the next matching Row).
      // Always check bloom filter to optimize the top row seek for delete
      // family marker.
      seekScanners(scanners, matcher.getStartKey(), explicitColumnQuery && lazySeekEnabledGlobally,
        parallelSeekEnabled);

      // set storeLimit
      this.storeLimit = scan.getMaxResultsPerColumnFamily();

      // set rowOffset
      this.storeOffset = scan.getRowOffsetPerColumnFamily();
      addCurrentScanners(scanners);
      // Combine all seeked scanners with a heap
      resetKVHeap(scanners, comparator);
    } catch (IOException e) {
      clearAndClose(scanners);
      // remove us from the HStore#changedReaderObservers here or we'll have no chance to
      // and might cause memory leak
      store.deleteChangedReaderObserver(this);
      throw e;
    }
  }
```

核心在于获取`scanners`的过程，先跳转到`store.getScanners(...)`
```java
public List<KeyValueScanner> getScanners(boolean cacheBlocks, boolean usePread,
    boolean isCompaction, ScanQueryMatcher matcher, byte[] startRow, boolean includeStartRow,
    byte[] stopRow, boolean includeStopRow, long readPt, boolean onlyLatestVersion)
    throws IOException {
    Collection<HStoreFile> storeFilesToScan;
    List<KeyValueScanner> memStoreScanners;
    this.storeEngine.readLock();
    try {
      storeFilesToScan = this.storeEngine.getStoreFileManager().getFilesForScan(startRow,
        includeStartRow, stopRow, includeStopRow, onlyLatestVersion);
      memStoreScanners = this.memstore.getScanners(readPt);
      // NOTE: here we must increase the refCount for storeFiles because we would open the
      // storeFiles and get the StoreFileScanners for them.If we don't increase the refCount here,
      // HStore.closeAndArchiveCompactedFiles called by CompactedHFilesDischarger may archive the
      // storeFiles after a concurrent compaction.Because HStore.requestCompaction is under
      // storeEngine lock, so here we increase the refCount under storeEngine lock. see HBASE-27484
      // for more details.
      HStoreFile.increaseStoreFilesRefeCount(storeFilesToScan);
    } finally {
      this.storeEngine.readUnlock();
    }
    try {
      // First the store file scanners

      // TODO this used to get the store files in descending order,
      // but now we get them in ascending order, which I think is
      // actually more correct, since memstore get put at the end.
      List<StoreFileScanner> sfScanners = StoreFileScanner.getScannersForStoreFiles(
        storeFilesToScan, cacheBlocks, usePread, isCompaction, false, matcher, readPt);
      List<KeyValueScanner> scanners = new ArrayList<>(sfScanners.size() + 1);
      scanners.addAll(sfScanners);
      // Then the memstore scanners
      scanners.addAll(memStoreScanners);
      return scanners;
    } catch (Throwable t) {
      clearAndClose(memStoreScanners);
      throw t instanceof IOException ? (IOException) t : new IOException(t);
    } finally {
      HStoreFile.decreaseStoreFilesRefeCount(storeFilesToScan);
    }
  }
```

可以发现首先获取内存和磁盘的存储
- `storeFilesToScan`：获取`HFile`对应的文件。
- `memStoreScanners`：根据传入的视图获取内存的`scanner`。

先看`memStore`，可以发现有两种获取的方式，一种是`InMemoryCompaction`，这里先看这个
```java
  @Override
  public List<KeyValueScanner> getScanners(long readPt) throws IOException {
    MutableSegment activeTmp = getActive();
    List<? extends Segment> pipelineList = pipeline.getSegments();
    List<? extends Segment> snapshotList = snapshot.getAllSegments();
    long numberOfSegments = 1L + pipelineList.size() + snapshotList.size();
    // The list of elements in pipeline + the active element + the snapshot segment
    List<KeyValueScanner> list = createList((int) numberOfSegments);
    addToScanners(activeTmp, readPt, list);
    addToScanners(pipelineList, readPt, list);
    addToScanners(snapshotList, readPt, list);
    return list;
  }
```

可以发现获取了`3`个`Scanner`，分别是
- 活跃段：类似`memtable`。
- 管道段：压缩管道内的不可变的段，类似`immemtable`。
- 快照段：准备刷到盘的段集合。

### 活跃段
可以发现最终获得的`active`是一个`MutableSegment`，主要包含以下
- `cellSet`：存储实际数据的容器，通常是`ConcurrentSkipListMap`。
- `comparator`：单元格比较器，决定数据排序方式。
- `memStoreLAB`：内存分配缓冲区，优化小对象内存分配。

其实就是一个`memtable`。

### 管道段
这里就是处理合并操作的管道。可以发现实际是返回了`readOnlyCopy`，是一个`LinkedList<ImmutableSegment>`类型。

对于如何处理`readOnlyCopy`先搁置，而讨论`ImmutableSegment`是什么，是怎么读取的。

### Snapshot
快照。

现在来看是怎么从`Segment`这种数据结构里面读取内容的
```java
public static void addToScanners(List<? extends Segment> segments, long readPt,
  List<KeyValueScanner> scanners) {
  for (Segment item : segments) {
    addToScanners(item, readPt, scanners);
  }
}

protected static void addToScanners(Segment segment, long readPt,
  List<KeyValueScanner> scanners) {
  if (!segment.isEmpty()) {
    scanners.add(segment.getScanner(readPt));
  }
}
```

实际上是调用了`addToScanners`把一个`Segment`放到`scanners`里面。
`scanners`是一个`List<KeyValueScanner>`类型。

实际上`KeyValueScanner`也是一个虚基类，因此查看对应的实现，这里先看`SegmentScanner`，由于比较长，就不全部黏贴了，具体可以发现有以下几个`API`
- public ExtendedCell peek()       // 查看当前Cell但不移动指针
- public ExtendedCell next()       // 获取当前Cell并移动到下一个
- public boolean seek(ExtendedCell cell)     // 定位到指定Cell或之后位置
- public boolean reseek(ExtendedCell cell)   // 重新定位，优化的seek版本
- protected Iterator<ExtendedCell> getIterator(ExtendedCell cell)  // 获取从指定Cell开始的迭代器
- protected void updateCurrent()  // 更新当前Cell，跳过MVCC不相关的版本

列举了以下几个`API`，可以发现实际上就是一个`Iterator`，可以`seek`到某个`key`。

先看一下`seek`是怎么实现的
```java
@Override
public boolean seek(ExtendedCell cell) throws IOException {
  if (closed) {
    return false;
  }
  if (cell == null) {
    close();
    return false;
  }
  // restart the iterator from new key
  iter = getIterator(cell);
  // last is going to be reinitialized in the next getNext() call
  last = null;
  updateCurrent();
  return (current != null);
}
```

首先看一下`getIterator(cell)`，显然是新建一个`Iterator`指向这个`Cell`
```java
protected Iterator<ExtendedCell> getIterator(ExtendedCell cell) {
  return segment.tailSet(cell).iterator();
}
```

实际上底层调用了`CompositeImmutableSegment`里的`tailSet`
```java
待补充
```

然后看一下`updateCurrent`
```java
protected void updateCurrent() {
  ExtendedCell next = null;

  try {
    while (iter.hasNext()) {
      next = iter.next();
      if (next.getSequenceId() <= this.readPoint) {
        current = next;
        return;// skip irrelevant versions
      }
      // for backwardSeek() stay in the boundaries of a single row
      if (stopSkippingKVsIfNextRow && segment.compareRows(next, stopSkippingKVsRow) > 0) {
        current = null;
        return;
      }
    } // end of while

    current = null; // nothing found
  } finally {
    if (next != null) {
      // in all cases, remember the last KV we iterated to, needed for reseek()
      last = next;
    }
  }
}
```

首先是判断视图是否合法，如果当前`sequenceId`比`readPoint`小，那么表示是合理的视图，直接返回。

可以发现整个`seek`的流程就是先二分到对应的`Cell`，然后根据`MVCC`给的视图找到合理的第一个`Cell`。

## CellArrayImmutableSegment
这里看一下数组形式的`ImmutableSegment`，对于`InMemoryCompaction`来说，实际上是把将写优化的`CSLMImmutableSegment`（跳表结构）转换为读优化的`CellArrayImmutableSegment`（数组结构）。

# InMemoryCompaction
核心代码如下
```java
void inMemoryCompaction() {
  // setting the inMemoryCompactionInProgress flag again for the case this method is invoked
  // directly (only in tests) in the common path setting from true to true is idempotent
  inMemoryCompactionInProgress.set(true);
  // Used by tests
  if (!allowCompaction.get()) {
    return;
  }
  try {
    // Speculative compaction execution, may be interrupted if flush is forced while
    // compaction is in progress
    if (!compactor.start()) {
      setInMemoryCompactionCompleted();
    }
  } catch (IOException e) {
    LOG.warn("Unable to run in-memory compaction on {}/{}; exception={}",
      getRegionServices().getRegionInfo().getEncodedName(), getFamilyName(), e);
  }
}

public boolean start() throws IOException {
  if (!compactingMemStore.hasImmutableSegments()) { // no compaction on empty pipeline
    return false;
  }

  // get a snapshot of the list of the segments from the pipeline,
  // this local copy of the list is marked with specific version
  versionedList = compactingMemStore.getImmutableSegments();
  LOG.trace("Speculative compaction starting on {}/{}",
    compactingMemStore.getStore().getHRegion().getRegionInfo().getEncodedName(),
    compactingMemStore.getStore().getColumnFamilyName());
  HStore store = compactingMemStore.getStore();
  RegionCoprocessorHost cpHost = store.getCoprocessorHost();
  if (cpHost != null) {
    cpHost.preMemStoreCompaction(store);
  }
  try {
    doCompaction();
  } finally {
    if (cpHost != null) {
      cpHost.postMemStoreCompaction(store);
    }
  }
  return true;
}
```

实际上核心调用了`doCompaction`
```java
private void doCompaction() {
  ImmutableSegment result = null;
  boolean resultSwapped = false;
  MemStoreCompactionStrategy.Action nextStep = strategy.getAction(versionedList);
  boolean merge = (nextStep == MemStoreCompactionStrategy.Action.MERGE
    || nextStep == MemStoreCompactionStrategy.Action.MERGE_COUNT_UNIQUE_KEYS);
  try {
    if (isInterrupted.get()) { // if the entire process is interrupted cancel flattening
      return; // the compaction also doesn't start when interrupted
    }

    if (nextStep == MemStoreCompactionStrategy.Action.NOOP) {
      return;
    }
    if (
      nextStep == MemStoreCompactionStrategy.Action.FLATTEN
        || nextStep == MemStoreCompactionStrategy.Action.FLATTEN_COUNT_UNIQUE_KEYS
    ) {
      // some Segment in the pipeline is with SkipList index, make it flat
      compactingMemStore.flattenOneSegment(versionedList.getVersion(), nextStep);
      return;
    }

    // Create one segment representing all segments in the compaction pipeline,
    // either by compaction or by merge
    if (!isInterrupted.get()) {
      result = createSubstitution(nextStep);
    }

    // Substitute the pipeline with one segment
    if (!isInterrupted.get()) {
      resultSwapped = compactingMemStore.swapCompactedSegments(versionedList, result, merge);
      if (resultSwapped) {
        // update compaction strategy
        strategy.updateStats(result);
        // update the wal so it can be truncated and not get too long
        compactingMemStore.updateLowestUnflushedSequenceIdInWAL(true); // only if greater
      }
    }
  } catch (IOException e) {
    LOG.trace("Interrupting in-memory compaction for store={}",
      compactingMemStore.getFamilyName());
    Thread.currentThread().interrupt();
  } finally {
    // For the MERGE case, if the result was created, but swap didn't happen,
    // we DON'T need to close the result segment (meaning its MSLAB)!
    // Because closing the result segment means closing the chunks of all segments
    // in the compaction pipeline, which still have ongoing scans.
    if (!merge && (result != null) && !resultSwapped) {
      result.close();
    }
    releaseResources();
    compactingMemStore.setInMemoryCompactionCompleted();
  }

}
```

可以发现`Compaction`分为四种
- `NOOP`:啥都不干直接返回。
- `FLATTEN`:把一个基于跳表的`Segment`换成基于数组的`Segment`。
- `COMPACT`:把当前`pipeline`里的数据全部合并。
- `MERGE`:合并多个段但不进行数据处理，只是结构合并。

## FLATTEN
核心函数是`flattenOneSegment`，这里用了一个乐观并发控制，在锁外和锁内分别检查一下`version`，如果不符合要求则直接返回。

然后是枚举`pipeline`中所有的`Segment`，看是否支持`flatten`，如果支持，则根据当前`Segment`创建一个数组类型的`Segment`。实际上只有基于跳表的`Segment`可以被`flatten`。
```java
  public boolean flattenOneSegment(long requesterVersion, CompactingMemStore.IndexType idxType,
    MemStoreCompactionStrategy.Action action) {

    if (requesterVersion != version) {
      LOG.warn("Segment flattening failed, because versions do not match. Requester version: "
        + requesterVersion + ", actual version: " + version);
      return false;
    }

    synchronized (pipeline) {
      if (requesterVersion != version) {
        LOG.warn("Segment flattening failed, because versions do not match");
        return false;
      }
      int i = -1;
      for (ImmutableSegment s : pipeline) {
        i++;
        if (s.canBeFlattened()) {
          s.waitForUpdates(); // to ensure all updates preceding s in-memory flush have completed
          if (s.isEmpty()) {
            // after s.waitForUpdates() is called, there is no updates pending,if no cells in s,
            // we can skip it.
            continue;
          }
          // size to be updated
          MemStoreSizing newMemstoreAccounting = new NonThreadSafeMemStoreSizing();
          ImmutableSegment newS = SegmentFactory.instance().createImmutableSegmentByFlattening(
            (CSLMImmutableSegment) s, idxType, newMemstoreAccounting, action);
          replaceAtIndex(i, newS);
          if (region != null) {
            // Update the global memstore size counter upon flattening there is no change in the
            // data size
            MemStoreSize mss = newMemstoreAccounting.getMemStoreSize();
            region.addMemStoreSize(mss.getDataSize(), mss.getHeapSize(), mss.getOffHeapSize(),
              mss.getCellsCount());
          }
          LOG.debug("Compaction pipeline segment {} flattened", s);
          return true;
        }
      }
    }
    // do not update the global memstore size counter and do not increase the version,
    // because all the cells remain in place
    return false;
  }
```

## COMPACT
可以发现实际上是以下函数实现的，包括`MERGE`。

首先是获得当前所有`Segment`的集合，然后创建`iterator`
```java
  private ImmutableSegment createSubstitution(MemStoreCompactionStrategy.Action action)
    throws IOException {

    ImmutableSegment result = null;
    MemStoreSegmentsIterator iterator = null;
    List<ImmutableSegment> segments = versionedList.getStoreSegments();
    for (ImmutableSegment s : segments) {
      s.waitForUpdates(); // to ensure all updates preceding s in-memory flush have completed.
      // we skip empty segment when create MemStoreSegmentsIterator following.
    }

    switch (action) {
      case COMPACT:
        iterator = new MemStoreCompactorSegmentsIterator(segments,
          compactingMemStore.getComparator(), compactionKVMax, compactingMemStore.getStore());

        result = SegmentFactory.instance().createImmutableSegmentByCompaction(
          compactingMemStore.getConfiguration(), compactingMemStore.getComparator(), iterator,
          versionedList.getNumOfCells(), compactingMemStore.getIndexType(), action);
        iterator.close();
        break;
      case MERGE:
      case MERGE_COUNT_UNIQUE_KEYS:
        iterator = new MemStoreMergerSegmentsIterator(segments, compactingMemStore.getComparator(),
          compactionKVMax);

        result = SegmentFactory.instance().createImmutableSegmentByMerge(
          compactingMemStore.getConfiguration(), compactingMemStore.getComparator(), iterator,
          versionedList.getNumOfCells(), segments, compactingMemStore.getIndexType(), action);
        iterator.close();
        break;
      default:
        throw new RuntimeException("Unknown action " + action); // sanity check
    }

    return result;
  }
```

先看创立`iterator`的过程，可以发现首先搞了一个`scanners`，然后调用`refillKVS`
```java
  public MemStoreCompactorSegmentsIterator(List<ImmutableSegment> segments,
    CellComparator comparator, int compactionKVMax, HStore store) throws IOException {
    super(compactionKVMax);

    List<KeyValueScanner> scanners = new ArrayList<KeyValueScanner>();
    AbstractMemStore.addToScanners(segments, Long.MAX_VALUE, scanners);
    // build the scanner based on Query Matcher
    // reinitialize the compacting scanner for each instance of iterator
    compactingScanner = createScanner(store, scanners);
    refillKVS();
  }
```

然后看`refillKVS`，可以发现实际上的操作是把之前创建好的`Scanner`里的数据全部放到`kvs`里
```java
private boolean refillKVS() {
    // if there is nothing expected next in compactingScanner
    if (!hasMore) {
      return false;
    }
    // clear previous KVS, first initiated in the constructor
    kvs.clear();
    for (;;) {
      try {
        // InternalScanner is for CPs so we do not want to leak ExtendedCell to the interface, but
        // all the server side implementation should only add ExtendedCell to the List, otherwise it
        // will cause serious assertions in our code
        hasMore = compactingScanner.next(kvs, scannerContext);
      } catch (IOException e) {
        // should not happen as all data are in memory
        throw new IllegalStateException(e);
      }
      if (!kvs.isEmpty()) {
        kvsIterator = kvs.iterator();
        return true;
      } else if (!hasMore) {
        return false;
      }
    }
  }
```

填充完之后开始准备`Compaction`，原理实际上很简单，这里封装的很好，实际上是将之前创立好的`iterator`传入`createImmutableSegment`里，新建一个`ImmutableSegment`
```java
  public ImmutableSegment createImmutableSegmentByCompaction(final Configuration conf,
    final CellComparator comparator, MemStoreSegmentsIterator iterator, int numOfCells,
    CompactingMemStore.IndexType idxType, MemStoreCompactionStrategy.Action action)
    throws IOException {

    MemStoreLAB memStoreLAB = MemStoreLAB.newInstance(conf);
    return createImmutableSegment(conf, comparator, iterator, memStoreLAB, numOfCells, action,
      idxType);
  }
```

看如何新建一个`Segment`，这里关注`CellArrayImmutableSegment`是如何实现的
```java
  private ImmutableSegment createImmutableSegment(final Configuration conf,
    final CellComparator comparator, MemStoreSegmentsIterator iterator, MemStoreLAB memStoreLAB,
    int numOfCells, MemStoreCompactionStrategy.Action action,
    CompactingMemStore.IndexType idxType) {

    ImmutableSegment res = null;
    switch (idxType) {
      case CHUNK_MAP:
        res = new CellChunkImmutableSegment(comparator, iterator, memStoreLAB, numOfCells, action);
        break;
      case CSLM_MAP:
        assert false; // non-flat segment can not be created here
        break;
      case ARRAY_MAP:
        res = new CellArrayImmutableSegment(comparator, iterator, memStoreLAB, numOfCells, action);
        break;
    }
    return res;
  }
```

调用构造函数
```java
  protected CellArrayImmutableSegment(CellComparator comparator, MemStoreSegmentsIterator iterator,
    MemStoreLAB memStoreLAB, int numOfCells, MemStoreCompactionStrategy.Action action) {
    super(null, comparator, memStoreLAB); // initiailize the CellSet with NULL
    incMemStoreSize(0, DEEP_OVERHEAD_CAM, 0, 0); // CAM is always on-heap
    // build the new CellSet based on CellArrayMap and update the CellSet of the new Segment
    initializeCellSet(numOfCells, iterator, action);
  }
```

新建完之后，自然希望把这个`ImmutableSegment`替换当前的`ImmutableSegment`，拿着给的`result`进行填充
```java
if (!isInterrupted.get()) {
  result = createSubstitution(nextStep);
}

// Substitute the pipeline with one segment
if (!isInterrupted.get()) {
  resultSwapped = compactingMemStore.swapCompactedSegments(versionedList, result, merge);
  if (resultSwapped) {
    // update compaction strategy
    strategy.updateStats(result);
    // update the wal so it can be truncated and not get too long
    compactingMemStore.updateLowestUnflushedSequenceIdInWAL(true); // only if greater
  }
}
```

将`pipeline`里的`Segment`进行替换
```java
public boolean swapCompactedSegments(VersionedSegmentsList versionedList, ImmutableSegment result,
    boolean merge) {
  // last true stands for updating the region size
  return pipeline.swap(versionedList, result, !merge, true);
}
```

然后是`swap`。

首先还是进行乐观并发控制，即`version`检查，锁，`version`检查。然后获取当前`pipeline`里存储的`Segment`，通过`swapSuffix`进行替换，替换完后`verison++`，同时更新`readOnlyCopy`。
```java
  @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "VO_VOLATILE_INCREMENT",
      justification = "Increment is done under a synchronize block so safe")
  public boolean swap(VersionedSegmentsList versionedList, ImmutableSegment segment,
    boolean closeSuffix, boolean updateRegionSize) {
    if (versionedList.getVersion() != version) {
      return false;
    }
    List<ImmutableSegment> suffix;
    synchronized (pipeline) {
      if (versionedList.getVersion() != version) {
        return false;
      }
      suffix = versionedList.getStoreSegments();
      LOG.debug("Swapping pipeline suffix; before={}, new segment={}",
        versionedList.getStoreSegments().size(), segment);
      swapSuffix(suffix, segment, closeSuffix);
      readOnlyCopy = new LinkedList<>(pipeline);
      version++;
    }
    if (updateRegionSize && region != null) {
      // update the global memstore size counter
      long suffixDataSize = getSegmentsKeySize(suffix);
      long suffixHeapSize = getSegmentsHeapSize(suffix);
      long suffixOffHeapSize = getSegmentsOffHeapSize(suffix);
      int suffixCellsCount = getSegmentsCellsCount(suffix);
      long newDataSize = 0;
      long newHeapSize = 0;
      long newOffHeapSize = 0;
      int newCellsCount = 0;
      if (segment != null) {
        newDataSize = segment.getDataSize();
        newHeapSize = segment.getHeapSize();
        newOffHeapSize = segment.getOffHeapSize();
        newCellsCount = segment.getCellsCount();
      }
      long dataSizeDelta = suffixDataSize - newDataSize;
      long heapSizeDelta = suffixHeapSize - newHeapSize;
      long offHeapSizeDelta = suffixOffHeapSize - newOffHeapSize;
      int cellsCountDelta = suffixCellsCount - newCellsCount;
      region.addMemStoreSize(-dataSizeDelta, -heapSizeDelta, -offHeapSizeDelta, -cellsCountDelta);
      LOG.debug(
        "Suffix data size={}, new segment data size={}, suffix heap size={},new segment heap "
          + "size={} 　suffix off heap size={}, new segment off heap size={}, suffix cells "
          + "count={}, new segment cells count={}",
        suffixDataSize, newDataSize, suffixHeapSize, newHeapSize, suffixOffHeapSize, newOffHeapSize,
        suffixCellsCount, newCellsCount);
    }
    return true;
  }
```

然后看`swapSuffix`，发现实际上是两步，第一步移除`pipeline`里对应的`Segment`，然后把当前`Segment`放入`pipeline`。
```java
  private void swapSuffix(List<? extends Segment> suffix, ImmutableSegment segment,
    boolean closeSegmentsInSuffix) {
    matchAndRemoveSuffixFromPipeline(suffix);
    if (segment != null) {
      pipeline.addLast(segment);
    }
    // During index merge we won't be closing the segments undergoing the merge. Segment#close()
    // will release the MSLAB chunks to pool. But in case of index merge there wont be any data copy
    // from old MSLABs. So the new cells in new segment also refers to same chunks. In case of data
    // compaction, we would have copied the cells data from old MSLAB chunks into a new chunk
    // created for the result segment. So we can release the chunks associated with the compacted
    // segments.
    if (closeSegmentsInSuffix) {
      for (Segment itemInSuffix : suffix) {
        itemInSuffix.close();
      }
    }
  }
```

主要看`matchAndRemoveSuffixFromPipeline`，从链表尾部开始枚举，向前一个一个对比是否和要删除的链表元素相同，如果不同则抛异常，否则最后把链表尾部一个一个摘除。
```java
  private void matchAndRemoveSuffixFromPipeline(List<? extends Segment> suffix) {
    if (suffix.isEmpty()) {
      return;
    }
    if (pipeline.size() < suffix.size()) {
      throw new IllegalStateException(
        "CODE-BUG:pipleine size:[" + pipeline.size() + "],suffix size:[" + suffix.size()
          + "],pipeline size must greater than or equals suffix size");
    }

    ListIterator<? extends Segment> suffixIterator = suffix.listIterator(suffix.size());
    ListIterator<? extends Segment> pipelineIterator = pipeline.listIterator(pipeline.size());
    int count = 0;
    while (suffixIterator.hasPrevious()) {
      Segment suffixSegment = suffixIterator.previous();
      Segment pipelineSegment = pipelineIterator.previous();
      if (suffixSegment != pipelineSegment) {
        throw new IllegalStateException("CODE-BUG:suffix last:[" + count + "]" + suffixSegment
          + " is not pipleline segment:[" + pipelineSegment + "]");
      }
      count++;
    }

    for (int index = 1; index <= count; index++) {
      pipeline.pollLast();
    }

  }
```