---
title: Slatedb - memtable分析
draft: false
tags:
  - Rust
  - 分布式存储
---

这里主要研究一下读取`memtable`的代码流程，作为记录。

# 单点查询

## get
首先从`get`接口进入，可以发现先生成了一个`getters`，里面包含了`sst/mem/l0`，然后调用`get_inner`

```rust
async fn get(&'a self) -> Result<Option<Bytes>, SlateDBError> {  
    let getters: Vec<BoxFuture<'a, Result<Option<RowEntry>, SlateDBError>>> =  
        vec![self.get_memtable(), self.get_l0(), self.get_compacted()];  
  
    self.get_inner(getters).await  
}
```

这里先看`get_memtable`，挨个`mem`进行查询，通过`.table()`暴露出内部的跳表，然后查询。如果查到了则直接返回

```rust
fn get_memtable(&'a self) -> BoxFuture<'a, Result<Option<RowEntry>, SlateDBError>> {  
    async move {  
        if self.include_wal_memtables {  
            let maybe_val = std::iter::once(self.snapshot.wal())  
                .chain(self.snapshot.imm_wal().iter().map(|imm| imm.table()))  
                .find_map(|memtable| memtable.get(self.key, self.max_seq));  
            if let Some(val) = maybe_val {  
                return Ok(Some(val));  
            }  
        }  
  
        if self.include_memtables {  
            let maybe_val = std::iter::once(self.snapshot.memtable())  
                .chain(self.snapshot.imm_memtable().iter().map(|imm| imm.table()))  
                .find_map(|memtable| memtable.get(self.key, self.max_seq));  
            if let Some(val) = maybe_val {  
                return Ok(Some(val));  
            }  
        }  
        Ok(None)  
    }  
    .boxed()  
}
```

# scan

## scan with option

可以发现这里先遍历了所有`mem`，然后通过`.table()`暴露出接口，再通过`MergeIterator::new(memtable_iters)`生成所有`mem`的迭代器`memtable_iters`，最后和`sst/l0`的迭代器一起传入` DbIterator::new()`内，生成一个大的迭代器
```rust
pub(crate) async fn scan_with_options<'a>(  
    &'a self,  
    range: BytesRange,  
    options: &ScanOptions,  
    snapshot: &(dyn ReadSnapshot + Sync),  
) -> Result<DbIterator<'a>, SlateDBError> {  
    let mut memtables = VecDeque::new();  
  
    if self.include_wal_memtables(options.durability_filter) {  
        memtables.push_back(Arc::clone(&snapshot.wal()));  
        for imm_wal in snapshot.imm_wal() {  
            memtables.push_back(imm_wal.table());  
        }  
    }  
  
    if self.include_memtables(options.durability_filter) {  
        memtables.push_back(Arc::clone(&snapshot.memtable()));  
        for memtable in snapshot.imm_memtable() {  
            memtables.push_back(memtable.table());  
        }  
    }  
    let memtable_iters = memtables  
        .iter()  
        .map(|t| t.range_ascending(range.clone()))  
        .collect();  
  
    let mem_iter = MergeIterator::new(memtable_iters).await?;  
  
    // ...其他代码...  
  
    DbIterator::new(range.clone(), mem_iter, l0_iters, sr_iters).await  
}
```

接着简单看一下最后是如何生成一个迭代器，前面先对所有迭代器进行一次`max_seq`的`filter`，如果当前有的`key`的`seq_num`小于`max_seq`，则把这些`key`删除，最后合成一个迭代器数组，传入最后合成一个大的迭代器

```rust
pub(crate) async fn new(
    range: BytesRange,
    mem_iters: impl IntoIterator<Item = MemTableIterator>,
    l0_iters: impl IntoIterator<Item = SstIterator<'a>>,
    sr_iters: impl IntoIterator<Item = SortedRunIterator<'a>>,
    max_seq: Option<u64>,
) -> Result<Self, SlateDBError> {
    let iters: [Box<dyn KeyValueIterator>; 3] = {
        // Apply the max_seq filter to all the iterators. Please note that we should apply this filter BEFORE
        // merging the iterators.
        //
        // For example, if have the following iterators:
        // - Iterator A with entries [(key1, seq=96), (key1, seq=110)]
        // - Iterator B with entries [(key1, seq=95)]
        //
        // If we filter the iterator after merging with max_seq=100, we'll lost the entry with seq=96 from the
        // iterator A. But the element with seq=96 is actually the correct answer for this scan.
        let mem_iters = Self::apply_max_seq_filter(mem_iters, max_seq);
        let l0_iters = Self::apply_max_seq_filter(l0_iters, max_seq);
        let sr_iters = Self::apply_max_seq_filter(sr_iters, max_seq);
        let (mem_iter, l0_iter, sr_iter) = tokio::join!(
            MergeIterator::new(mem_iters),
            MergeIterator::new(l0_iters),
            MergeIterator::new(sr_iters)
        );
        [Box::new(mem_iter?), Box::new(l0_iter?), Box::new(sr_iter?)]
    };

    let iter = MergeIterator::new(iters).await?;
    Ok(DbIterator {
        range,
        iter,
        invalidated_error: None,
        last_key: None,
    })
}
```

## range
回过头来看一下如何生成一个`memtable`的迭代器，通过`range`函数生成
```rust
pub(crate) fn range_ascending<T: RangeBounds<Bytes>>(&self, range: T) -> MemTableIterator {
    self.range(range, IterationOrder::Ascending)
}

pub(crate) fn range<T: RangeBounds<Bytes>>(
    &self,
    range: T,
    ordering: IterationOrder,
) -> MemTableIterator {
    let internal_range = KVTableInternalKeyRange::from(range);
    let mut iterator = MemTableIteratorInnerBuilder {
        map: self.map.clone(),
        inner_builder: |map| map.range(internal_range),
        ordering,
        item: None,
    }
    .build();
    iterator.next_entry_sync();
    iterator
}

#[self_referencing]
pub(crate) struct MemTableIteratorInner<T: RangeBounds<KVTableInternalKey>> {
    map: Arc<SkipMap<KVTableInternalKey, RowEntry>>,
    /// `inner` is the Iterator impl of SkipMap, which is the underlying data structure of MemTable.
    #[borrows(map)]
    #[not_covariant]
    inner: Range<'this, KVTableInternalKey, T, KVTableInternalKey, RowEntry>,
    ordering: IterationOrder,
    item: Option<RowEntry>,
}
```

这里`build`是自引用库的生成方式，类似构造函数，然后调用`next_entry_sync`。重点关注一下`inner`，可以发现实际上调用了`crossbeam_skiplist::map::Range`，功能是根据传入的`internal_range`会返回该范围的迭代器，可以发现每次`next`或者`next_back`实际使用了`crossbeam_skiplist::map`的功能

```rust
impl MemTableIterator {
    pub(crate) fn next_entry_sync(&mut self) -> Option<RowEntry> {
        let ans = self.borrow_item().clone();
        let next_entry = match self.borrow_ordering() {
            IterationOrder::Ascending => self.with_inner_mut(|inner| inner.next()),
            IterationOrder::Descending => self.with_inner_mut(|inner| inner.next_back()),
        };

        let cloned_entry = next_entry.map(|entry| entry.value().clone());
        self.with_item_mut(|item| *item = cloned_entry);

        ans
    }
}
```