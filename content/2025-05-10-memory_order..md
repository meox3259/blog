---
title: "C++ - Memory Order"
draft: false
tags:
  - C++
---

由于`CPU`会对代码进行优化，因此顺序会重排，不一定按照代码中的顺序执行。因此在选用同步保证较弱的模型时会出现错误。

`C++`内存模型一共有以下`6`种

```cpp
typedef enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;
```

可以把这`6`种内存模型分为以下`3`类，对于`C++`中的原子操作，包括`load/store/read-modify-write`3种操作：

* 顺序一致`(sequence consistent ordering):memory_order_seq_cst`
* 获取发布`(acquire-release ordering):memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel`
* 松散`(relaxed ordering):memory_order_relaxed`

## 顺序一致模型

`std::memory_order_seq_cst`是`C++`中`std::atomic`默认的内存序。该模型提供了`C++`内存模型中最强的同步保证

* 使用`std::memory_order_seq_cst`的原子操作总会形成一个所有线程都能观测到的**唯一全局总顺序**，这里其实类似一致性模型中的线性一致性。
* 每个线程内部，使用`std::memory_order_seq_cst`的原子操作和代码中书写的顺序一样。
* 当一个线程使用`std::memory_order_seq_cst`完成写操作时，这个写结果会立刻对其他线程可见。

这样的优缺点比较明显

* 优点是**每个线程不存在重排**，以及模型简单，顺序是一致的。
* 缺点是效率较低，为了实现全局的顺序保证，自然需要较大的性能开销。

## 获取发布模型

该模型的同步级别仅次于顺序一致，主要有两种内存序

1. `std::memory_order_acquire`

* 通常用于`read/load`加载操作。
* 能够看到其他线程在该`acquire`之前通过`release`存储写入的所有内容；同样地，该`acquire`之后内存读取操作不会重排到`acquire`之前。

2. `std::memory_order_release`

* 通常用于`store/write`存储操作。
* 在`release`之前的读写操作，不会重排到`release`之后。

对此可以理解为，无论是对于`acquire`还是`release`操作，它们相对于其他的操作相对顺序是不变的。可以进一步总结如下，假设目前有一个场景：

* `A`线程执行了一个`release`的存储操作，`B`线程执行了一个`acquire`的读取操作，这里`B`线程的`acquire`看到了`A`线程的`release`操作，即`A` **happends before** `B`。

  * 由以上针对`release`和`acquire`操作的定义，可以发现这里对线程间进行了同步，即对于`A`中`release`之前的操作，对于`B`中`acquire`操作之后的操作都是可见的。
  * 因此可以视作一种**生产-消费者模型**，即`release`之前生产的信息都被`acquire`之后消费了。

# 宽松模型

该模型是同步级别最宽松的模型

* 仅仅保证一个操作是原子的，对于线程间的同步以及线程内的顺序都不在意。

显然这样的好处是效率最高，在一些场景下比较适合

* 实现一个多线程的计数器，仅仅在最终输出结果，这个时候显然使用宽松模型能够获取最好的效率。