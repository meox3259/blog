---
title: RocksDB - MultiGet
draft: false
tags:
  - C++
  - 分布式存储
  - rocksdb
---

MultiGet的意义是一次读入多个Key，然后返回多个Value，希望相对比一个一个Key进行点查效率提高。

# 流程

## DBImpl::MultiGetImpl
这个函数是执行查询的核心函数。

首先标记剩余的Key的数量，然后循环不断分批去查
```cpp
  size_t keys_left = num_keys;
  Status s;
  uint64_t curr_value_size = 0;
  while (keys_left) {...}
```

循环先检查当前是否超时，如果超时则返回TimeOut
```cpp
if (read_options.deadline.count() &&
    immutable_db_options_.clock->NowMicros() >
        static_cast<uint64_t>(read_options.deadline.count())) {
  s = Status::TimedOut();
  break;
}
```

然后计算当前查询一批Key的批的大小，最大不能超过`MAX_BATCH_SIZE`，同时创建`CONTEXT`和`RANGE`，用于跟踪上下文以及当前累计查询的Value的总大小
```cpp
size_t batch_size = (keys_left > MultiGetContext::MAX_BATCH_SIZE)
                        ? MultiGetContext::MAX_BATCH_SIZE
                        : keys_left;
MultiGetContext ctx(sorted_keys, start_key + num_keys - keys_left,
                    batch_size, snapshot, read_options, GetFileSystem(),
                    stats_);
MultiGetRange range = ctx.GetMultiGetRange();
range.AddValueSize(curr_value_size);
```

然后开始检查查询范围，首先检查是否需要检查`SST`，然后将上下文清空，初始状态设置为`OK`
```cpp
bool lookup_current = true;
keys_left -= batch_size;

for (auto mget_iter = range.begin(); mget_iter != range.end(); ++mget_iter) {
  mget_iter->merge_context.Clear();
  *mget_iter->s = Status::OK();
}
```

首先看`memtable`，如果是仅查询已持久化的数据，且包含未持久化的数据则跳过`memtable`。如果不要跳过，则通过`super_version`调用`memtable`的`MultiGet`查询。

具体逻辑为：
- 先检查`mem`
- 如果有未查询到的则看`imm`
- 如果所有`Key`都看到了则把`lookup_current`设为`false`，跳过查询`SST`
- 如果不能跳过`SST`则记录`mem`中的`miss`数量
```cpp
bool skip_memtable =
    (read_options.read_tier == kPersistedTier &&
     has_unpersisted_data_.load(std::memory_order_relaxed));
if (!skip_memtable) {
  super_version->mem->MultiGet(read_options, &range, callback,
                               false /* immutable_memtable */);
  if (!range.empty()) {
    super_version->imm->MultiGet(read_options, &range, callback);
  }
  if (!range.empty()) {
    uint64_t left = range.KeysLeft();
    RecordTick(stats_, MEMTABLE_MISS, left);
  } else {
    lookup_current = false;
  }
}
```

然后如果不跳过`SST`，则在`SST`里查询
```cpp
if (lookup_current) {
  PERF_TIMER_GUARD(get_from_output_files_time);
  super_version->current->MultiGet(read_options, &range, callback);
}
```

然后检查当前已经收集的`Value`大小，如果超过阈值`read_options.value_size_soft_limit`则返回`Status::Aborted()`

```cpp
curr_value_size = range.GetValueSize();
if (curr_value_size > read_options.value_size_soft_limit) {
  s = Status::Aborted();
  break;
}
```

最后再进行统计，统计当前已经收集到的`Key`的状态
- 如果一个`Key`的合并次数大于`merge_threshold`，那么标记成`Status::OkMergeOperandThresholdExceeded`状态
- 统计所有读取的字节数及读取成功个数

```cpp
for (size_t i = start_key; i < start_key + num_keys - keys_left; ++i) {
    KeyContext* key = (*sorted_keys)[i];
    assert(key);
    assert(key->s);

    if (key->s->ok()) {
      const auto& merge_threshold = read_options.merge_operand_count_threshold;
      if (merge_threshold.has_value() &&
          key->merge_context.GetNumOperands() > merge_threshold) {
        *(key->s) = Status::OkMergeOperandThresholdExceeded();
      }

      if (key->value) {
        bytes_read += key->value->size();
      } else {
        assert(key->columns);
        bytes_read += key->columns->serialized_size();
      }

      num_found++;
    }
  }
```

最后将错误状态传递，如果之前读取异常，则将异常状态传递给所有`Key`

```cpp
if (keys_left) {
  assert(s.IsTimedOut() || s.IsAborted());
  for (size_t i = start_key + num_keys - keys_left; i < start_key + num_keys;
       ++i) {
    KeyContext* key = (*sorted_keys)[i];
    *key->s = s;
  }
}
```

放上一个AI的总结
1. 批处理机制：不是一次处理所有键，而是分批处理，每批最多处理 MAX_BATCH_SIZE 个键。
2. 查询层次：遵循 RocksDB 的存储层次，先查内存表，再查不可变内存表，最后查 SST 文件。
3. 提前退出策略：
- 超过截止时间时退出
- 累积值大小超过软限制时退出
4. 性能监控：详细的性能计时和统计收集，用于监控和优化。
5. 实时限制：通过监控累积值大小，防止返回过大的数据集。
6. 错误处理：保证所有键都有明确的状态，即使是未处理的键。

## MemTable::MultiGet

首先判断是否有范围删除，如果`read_option`没配或`mem`不允许则没有范围删除
```cpp
bool no_range_del = read_options.ignore_range_deletions || is_range_del_table_empty_.load(std::memory_order_relaxed);
```

如果有范围删除，则不能用`bloom filter`。如果`mem`配置了`bloom filter`且没有范围删除，则先用`bloom filter`筛一遍。这里有个小技巧使用了`std::array<>`，由于是分配在栈上，因此性能较好。

`bloom filter`在这里有两种，一种是全量的`Key`，一种是前缀`Key`。然后将所有`Key`丢进`bloom filter`筛一遍，统计命中率，并将未命中的`Key`从`temp_range`里删除，减少枚举次数。
```cpp
MultiGetRange temp_range(*range, range->begin(), range->end());
  if (bloom_filter_ && no_range_del) {
    bool whole_key =
        !prefix_extractor_ || moptions_.memtable_whole_key_filtering;
    std::array<Slice, MultiGetContext::MAX_BATCH_SIZE> bloom_keys;
    std::array<bool, MultiGetContext::MAX_BATCH_SIZE> may_match;
    std::array<size_t, MultiGetContext::MAX_BATCH_SIZE> range_indexes;
    int num_keys = 0;
    for (auto iter = temp_range.begin(); iter != temp_range.end(); ++iter) {
      if (whole_key) {
        bloom_keys[num_keys] = iter->ukey_without_ts;
        range_indexes[num_keys++] = iter.index();
      } else if (prefix_extractor_->InDomain(iter->ukey_without_ts)) {
        bloom_keys[num_keys] =
            prefix_extractor_->Transform(iter->ukey_without_ts);
        range_indexes[num_keys++] = iter.index();
      }
    }
    bloom_filter_->MayContain(num_keys, bloom_keys.data(), may_match.data());
    for (int i = 0; i < num_keys; ++i) {
      if (!may_match[i]) {
        temp_range.SkipIndex(range_indexes[i]);
        PERF_COUNTER_ADD(bloom_memtable_miss_count, 1);
      } else {
        PERF_COUNTER_ADD(bloom_memtable_hit_count, 1);
      }
    }
  }
```

然后考虑删除的情况（待补充）

```cpp
if (!no_range_del) {
  std::unique_ptr<FragmentedRangeTombstoneIterator> range_del_iter(
    NewRangeTombstoneIteratorInternal(
      read_options, GetInternalKeySeqno(iter->lkey->internal_key()),
        immutable_memtable));
  SequenceNumber covering_seq = range_del_iter->MaxCoveringTombstoneSeqnum(iter->lkey->user_key());
  if (covering_seq > iter->max_covering_tombstone_seq) {
    iter->max_covering_tombstone_seq = covering_seq;
    if (iter->timestamp) {
      // Will be overwritten in SaveValue() if there is a point key with
      // a higher seqno.
      iter->timestamp->assign(range_del_iter->timestamp().data(), range_del_iter->timestamp().size());
    }
  }
}
```

从`mem`里获取`Value`
```cpp
GetFromTable(*(iter->lkey), iter->max_covering_tombstone_seq, true,
                 callback, &iter->is_blob_index,
                 iter->value ? iter->value->GetSelf() : nullptr, iter->columns,
                 iter->timestamp, iter->s, &(iter->merge_context), &dummy_seq,
                 &found_final_value, &merge_in_progress);
```

检查获取的`Value`的状态，如果正在合并当中则标记为`Status::MergeInProgress`

```cpp
if (!found_final_value && merge_in_progress) {
  if (iter->s->ok()) {
    *(iter->s) = Status::MergeInProgress();
  } else {
    assert(iter->s->IsMergeInProgress());
  }
}
```

最后对当前的`Value`获取结果进行处理，无非两种情况
- 读取成功，`found_final_value = true`
- 读取失败，`!iter->s->ok() && !iter->s->IsMergeInProgress()`，表示读取不成功且不是因为正在合并

无论以上那种情况，都进行处理，首先将`Value`的大小累加进当前获取`Value`大小总和，然后标记当前`Key`查询完成。如果当前`Value`大小总和超过了`read_options.value_size_soft_limit`，那么这次读取需要终止，于是进行状态传播，对所有`Key`标记成`Status::Aborted()`，然后返回。

```cpp
if (found_final_value || (!iter->s->ok() && !iter->s->IsMergeInProgress())) {
// `found_final_value` should be set if an error/corruption occurs.
// The check on iter->s is just there in case GetFromTable() did not
// set `found_final_value` properly.
  assert(found_final_value);
  if (iter->value) {
    iter->value->PinSelf();
    range->AddValueSize(iter->value->size());
  } else {
    assert(iter->columns);
    range->AddValueSize(iter->columns->serialized_size());
  }

  range->MarkKeyDone(iter);
  RecordTick(moptions_.statistics, MEMTABLE_HIT);
  if (range->GetValueSize() > read_options.value_size_soft_limit) {
    // Set all remaining keys in range to Abort
    for (auto range_iter = range->begin(); range_iter != range->end(); ++range_iter) {
      range->MarkKeyDone(range_iter);
      *(range_iter->s) = Status::Aborted();
    }
    break;
  }
}
```
