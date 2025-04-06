---
title: MinHash
tags:
  - 生物信息
---

最小哈希是一种快速估计两个集合相似度的算法。本文来自[Wiki](https://en.wikipedia.org/wiki/MinHash)。

# Jaccard similarity
Jaccard similarity通常用来表示两个集合的相似度，定义U是一个集合，A和B是U的子集，那么Jaccard similarity为
$$J(A,B)=\frac{|A \cap B|}{|A \cup B|}$$
显然Jaccard similarity系数越大表示集合越相似，反之越小。

# 哈希
定义h是将U中的元素映射到一些不相交整数的哈希函数，perm是集合U中元素的排列，对于任意集合S，定义$h_{min}{(S)}$为S中具有最小h(x)函数值的x元素x。

假设没有哈希碰撞，那么有以下结论
$$Pr[h_{min}{(A)}=h_{min}{(B)}]=J(A,B)$$

因此可以将Jaccard similarity的计算转化为通过哈希进行计算，即两个集合的最小哈希值相等的概率等于它们的Jaccard相似度。

# 方法
- 将所有哈希函数的最小值签名按行排列，形成**签名矩阵**：  
  $$
  \begin{bmatrix}
  h_1(A) & h_1(B) \\
  h_2(A) & h_2(B) \\
  \vdots & \vdots \\
  h_k(A) & h_k(B)
  \end{bmatrix}
  $$
  矩阵的每一行对应一个哈希函数的签名，列对应不同集合

- **相似度计算**：统计签名矩阵中两个集合对应行的哈希值相等的比例：  
  $$\text{Similarity} = \frac{\text{相等的哈希值行数}}{k}$$
  该比例即为Jaccard相似度$J(A,B) = \frac{|A \cap B|}{|A \cup B|}$的无偏估计

