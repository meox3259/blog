---
title: RocksDB - WriteBatch
draft: false
tags:
  - C++
  - 分布式存储
---

# Memtable的写入

对于`Rocksdb`中后台`flush`到`l0`的过程，后台线程会`PickUp`合适的`ImmutableMemtables`刷到`l0`，然后删除对应的`WAL`。

为什么说是挑选？因为`Rocksdb-3.0`中引入了`cf(Column Family)`，每个`cf`有自己的`memtable`，但是所有`cf`共享一个`WAL`，因此显然当我们把一个`cf`刷到磁盘后，不能将`WAL`删除，因此需要考虑`WAL`的生命周期的问题，只有所有`memtable`都被删除后，对应的`WAL`才能被刷盘。

所以这里可以发现，`cf`的个数不能很多，因为显然的，如果`cf`的个数过多，会导致`WAL`很难被刷到磁盘里，会占用大量的资源，因此可以考虑使用多个`Rocksdb`解决该问题。

# WriteBatch

为什么总结这篇文章是因为看到了一个问题：`WriteBatch`如何保证跨`cf`的原子性写入？

原子性写入的意义是，一次批量的写入，要么全部成功要么全部失败，显然这是满足`WriteBatch`的定义。由于所有`cf`是共享一个`WAL`，所以问题转化为将一个`WriteBatch`的结构体写入一个磁盘文件`WAL`，是否是原子的。

这里存在不原子的风险即将一块数据写入文件系统，这里有两种想法

1. 一个`WriteBatch`大小和扇区大小一致，利用文件系统本身的原子性实现原子性。
2. 加一个`Checksum`，如果数据损坏则`Checksum`不一致

还有一个问题是，如果不开启`WAL`，怎么保证`WriteBatch`跨`cf`的原子性写入？这里提到的方法是

1. 原子性写入`MANIFEST`，对于`MANIFEST`开启`O_DIRECT`和`fsync`强制刷盘，即使得元数据是原子性写入的

本文主要抄自[此处](https://mp.weixin.qq.com/s?__biz=MzkyMjIxMzIxNA==&mid=2247488161&idx=1&sn=0ab3c67458c5adcfcd1f659eb098a07e&scene=21#wechat_redirect)。

## WriteBatch准备工作

首先看下`WriteBatch`内部存储的格式

```cpp
|sequnce|count|kv_1|kv_2|...|kv_count|
```

前`12`个字节存储了一个`WriteBatch`的元信息

* `sequnce:`8个字节，记录了当前`WriteBatch`是`Rocksdb`创建以来第几个`WriteBatch`。
* `count:`4个字节，记录了当前`WriteBatch`有几个`kv`对。

然后看`kv`对是如何存储的，这里序列化成`Record`，分为有无`cf`

```cpp
default cf:
  |KTypeValue|
  |key_size|key_bytes|
  |value_length|value_bytes|

specify cf:    
  |kTypeColumnFamilyValue|column_family_id|
  |key_size|key_bytes|
  |value_length|value_bytes|
```

* `TypeValue:`1个字节，表示是`put`还是`delete`，以及是否指定了`cf`。
* `column family id:`4个字节，如果是指定了`cf`才会有这个字段。

`WriteBatch`有大小限制，如果当前写入超过了`WriteBatch`写入大小限制，那么新建一个`WriteBatch`。

### WriteThread::Writer

`WriteBatch`实际是封装到`WriteThread::Writer`里

```cpp
  struct WriteThread::Writer {
    WriteBatch* batch;
    //.. too many other fields ..

    std::atomic<uint8_t> state;
    std::aligned_storage<sizeof(std::mutex)>::type state_mutex_bytes;
    std::aligned_storage<sizeof(std::condition_variable)>::type state_cv_bytes;
    Writer* link_older;  // this 之前之前写入的 writers
    Writer* link_newer;  // this 之后写入的 writers

    //... methods
  };
```

可以发现`Writer`实际上是一个双向链表，多个`Writer`用链表连接了起来。

### WriteThread::WriteGroup

显然一次只能写入一个`Writer`是很慢的，这样所有`Writer`串行写入，那么其他`Writer`需要阻塞，显然效率低。于是使用了一个`WriteGroup`统一写入。

```cpp
  struct WriteThread::WriteGroup {
    Writer* leader = nullptr;
    Writer* last_writer = nullptr;
    //... other fields

    struct Iterator {
      Writer* writer;
      Writer* last_writer;

      explicit Iterator(Writer* w, Writer* last)
          : writer(w), last_writer(last) {}

      Writer* operator*() const { return writer; }

      Iterator& operator++() {
        assert(writer != nullptr);
        if (writer == last_writer) {
          writer = nullptr;
        } else {
          writer = writer->link_newer;
        }
        return *this;
      }

      bool operator!=(const Iterator& other) const {
        return writer != other.writer;
      }
    };

    Iterator begin() const { return Iterator(leader, last_writer); }
    Iterator end() const { return Iterator(nullptr, nullptr); }
  };
```

可以发现`WriteGroup`对多个`Writer`进行了一个封装

* 记录了`WriteGroup`对应`Writer`链表的`head`和`tail`，且链表是按照时间顺序排序的。
* 对链表封装了`Iterator`。

### WriteThread::State

自然对于`Writer`需要维护当前`Writer`的状态

```cpp
  enum WriteThread::State : uint8_t {
    STATE_INIT = 1,
    STATE_GROUP_LEADER = 2,
    STATE_MEMTABLE_WRITER_LEADER = 4,
    STATE_PARALLEL_MEMTABLE_WRITER = 8,
    STATE_COMPLETED = 16,
    STATE_LOCKED_WAITING = 32,
  };
```

`Writer`创立后先`INIT`，然后经过`WriteThread::JoinBatch`

1. 要么成为本次写入流程的`leader`，即`State::STATE_GROUP_LEADER`状态，然后组建自己的`writer_group`，代替`writer_group`中所有 `writers`完成写入，所有的`writers`状态都变成`State::STATE_COMPLETED`。
2. 要么加入一个已经选出`leader`但是尚未执行的`writer_group`成为`follower`，让该`leader`代替自己执行完本次写入，完成后自己状态即`State::STATE_COMPLETED`。
3. 要么阻塞等待之前的`WriteGroup`写入完成。

## JoinGroup

写入的流程为`JoinGroup->选出GroupLeader->WAL->Memtable->Exit`。

这里`JoinGroup`相当于一个锁，只有`leader`才能执行写入`WAL`和`Memtable`操作，其他`Writer`阻塞等待。

现在考虑如何`JoinGroup`，其实很简单，主要考虑`WriteThread`中`newest_writer_`，即最新的`Write`

1. 新插入一个`Writer`对象`w`后，尝试让`newest_writer_`指向`w`，如果当前`WriteStall`了则等待后再指向
2. 如果`newest_writer_==NULL`则直接进入后续流程
3. 否则认为当前已经存在`leader`，当前`writer`所需做的就是等待`leader`写完所有的`writer`，即等待`w->state`变成`STATE_GROUP_LEADER`或`STATE_COMPLETED`，即当前`writer`变成了`leader`或者被`leader`做完了

具体如下

```cpp
void WriteThread::JoinBatchGroup(Writer* w) {
  assert(w->batch != nullptr);
  // 1. 让 nnewest_writer_ 指向 w
  bool linked_as_leader = LinkOne(w, &newest_writer_);

  if (linked_as_leader) {
    SetState(w, STATE_GROUP_LEADER);
  }
  if (!linked_as_leader) {
     // 2. 阻塞等待 w->state & mask != 0
    AwaitState(w,
               STATE_GROUP_LEADER | STATE_MEMTABLE_WRITER_LEADER |
                   STATE_PARALLEL_MEMTABLE_WRITER | STATE_COMPLETED,
               &jbg_ctx);
  }
}
```

这里`WriteStall`还有一种额外配置

* `Writer::no_slowdown == false`，通过条件变量`stall_cv_`等待`WriteStall`结束被唤醒
* `Writer::no_slowdown == true`，直接返回`Status::Incomplete`，类似非阻塞行为

```cpp
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  assert(newest_writer != nullptr);
  assert(w->state == STATE_INIT);
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    assert(writers != w);
    if (writers == &write_stall_dummy_) {
      if (w->no_slowdown) {
        w->status = Status::Incomplete("Write stall");
        SetState(w, STATE_COMPLETED);
        return false;
      }
      {
        MutexLock lock(&stall_mu_);
        writers = newest_writer->load(std::memory_order_relaxed);
        if (writers == &write_stall_dummy_) {
          stall_cv_.Wait();
          // Load newest_writers_ again since it may have changed
          writers = newest_writer->load(std::memory_order_relaxed);
          continue;
        }
      }
    }
    w->link_older = writers;
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);
    }
  }
}
```

## EnterAsBatchGroupLeader

然后看`leader`是如何处理写的，可以发现传入`leader`后，不断向后枚举其他`Writer`，直到`WriteBatch`达到大小上限

```cpp
size_t WriteThread::EnterAsBatchGroupLeader(Writer* leader,
                                            WriteGroup* write_group) {
  assert(leader->link_older == nullptr);
  assert(leader->batch != nullptr);
  assert(write_group != nullptr);

  size_t size = WriteBatchInternal::ByteSize(leader->batch);

  // 1. write_group 大小限制
  size_t max_size = max_write_batch_group_size_bytes;
  const uint64_t min_batch_size_bytes = max_write_batch_group_size_bytes / 8;
  if (size <= min_batch_size_bytes) {
    max_size = size + min_batch_size_bytes;
  }

  // 2. 初始化 write_group
  leader->write_group = write_group;
  write_group->leader = leader;
  write_group->last_writer = leader;
  write_group->size = 1;
  Writer* newest_writer = newest_writer_.load(std::memory_order_acquire);
 
  // 3. 补全 [leader-writer, newest_writer] 丢失的 prev 指针
  CreateMissingNewerLinks(newest_writer);

  // 4. 封装到 write_group
  Writer* w = leader;
  while (w != newest_writer) {
    assert(w->link_newer);
    w = w->link_newer;

    // ... other break conditions

    if (size + WriteBatchInternal::ByteSize(w->batch) > max_size) {
      break;
    }

    w->write_group = write_group; // 设置 w 属于当前 write_group
    size += batch_size;           // 这个 write_group 的数据量大小
    write_group->last_writer = w; // 更新 last_writer
    write_group->size++;          // writer_group 中 writers 的个数
  }
  return size;
}
```

## ExitAsBatchGroupLeader

接着看`leader`完成后如何退出。首先需要选出下一个`leader`，方法是看`newest_writer_`是否出现了新的`writer`，如果出现了新的则将当前`WriteBatch`下一个`Writer`变成`leader`。

然后将当前完成的`Writer`修改，所有状态变成`STATE_COMPLETED`。

```cpp
void WriteThread::ExitAsBatchGroupLeader(WriteGroup& write_group,
                                         Status& status) {
  Writer* leader = write_group.leader;
  Writer* last_writer = write_group.last_writer;
  assert(leader->link_older == nullptr);

  if (enable_pipelined_write_) {
    // ...
  } else {
    Writer* head = newest_writer_.load(std::memory_order_acquire);
    if (head != last_writer ||
        !newest_writer_.compare_exchange_strong(head, nullptr)) {
      // 存在新插入的 writer
      assert(head != last_writer);

	// 1. 补全 [last_writer, head] 的 prev指针
      CreateMissingNewerLinks(head);
      assert(last_writer->link_newer != nullptr);
      assert(last_writer->link_newer->link_older == last_writer);
      // 2. 断开链表
      last_writer->link_newer->link_older = nullptr; 

      // 3. 设置新的 leader-writer
      SetState(last_writer->link_newer, STATE_GROUP_LEADER);
    }

    // 4. 将 [leader, last_writer] 区间的 writers 状态设置为 STATE_COMPLETED
    //    即解除阻塞，让 follower-writer 返回应用层
    while (last_writer != leader) {
      assert(last_writer);
      last_writer->status = status;
      // 需要先获取 next指针，再更改状态为 STATE_COMPLETED
      // 因为先更改 STATE_COMPLETED 很可能导致 last_writer 就被正在阻塞的线程销毁了
      // 再获取 next 指针就会触发 coredump
      auto next = last_writer->link_older;
      SetState(last_writer, STATE_COMPLETED); // 解除阻塞

      last_writer = next;
    }
  }
}
```

# Pipeline优化

优化的地方主要是，原来所有`WriteGroup`都是串行的，即`WAL1->MEM1->WAL2->MEM2...`。显然这样有浪费，因为如果一个`WriteGroup`已经写入了`WAL`，那么可以认为数据已经持久化了，因此写`MEM`的时候可以和下一个`WriteGroup`写`WAL`并行。

## Memtable并发写

目前只有`skiplist`支持并发写。

## 流程

1. 完成当前`write_group0` 的`WAL`写入流程
2. 通知`write_group1`开启`WAL`写入流程，即`write_group1`无需等待`write_group0` 完成`MemTable`写入流程才开启自己的`WAL`写入流程
3. `write_group0`的`WAL`写入流程完成后，需要启动`write_group0`的并发写 `MemTable`流程

由于后两个任务都需要等待第一个任务完成，因此三个任务的分界点就可以设置在 `WrriteThread::ExitAsBatchGroupLeader` 函数中: T1 在写 WAL 期间需要整个 RocksDB 只有一个`leader-writer`，在 T1 任务结束后就可以不再担任`leader`角色。此时有两件事需要做

1. 挑选出下一个`WriteGroup`中的`leader-writer`，让 T2 任务可以`pipeline`执行
2. 开启当前`WriteGroup`并发写入`MemTable`流程