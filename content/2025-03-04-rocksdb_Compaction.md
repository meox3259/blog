---
title: RocksDB - Compaction
draft: true
tags:
  - C++
  - 分布式存储
---

本文主要尝试介绍Rocksdb中Compaction的大致流程。

# DBImpl::BackgroundCompaction

