---
title: Fast characterization of segmental duplications in genome assemblies
tags:
  - 生物信息
---

# Segmental duplications
片段重复（Segmental duplications, SDs）是指基因组中长度超过1千碱基对（Kbp）的片段，这些片段以串联或散布形式重复出现，且与其他重复区域的序列一致性至少达到90%。

# 限制
假设一个基因序列
$$G=g_1g_2g_3...g_{|G|}$$
其中$g_i \in {A,C,G,T}$
假设$G_{i:i+n}=g_i,...,g_{i+n-1}$是$G$的起始自$i$的一个子串。

更多的，设$X_i$为$G_{i:i+n}$中所有`k-mer`的集合，这里`k`是预先固定好的。

设`l`为$G_i$和$G_j$的alignment的长度，显然$l \geq max(n,m)$

然后定义
$$err(G_i,G_j) = \frac{ed(G_i,G_j)}{l}$$

我们定义$G_i$和$G_j$是一个长度为$l$且错误率为$\delta$
- $alignment(G_i,G_j)=l$
- $err(G_i,G_j) \leq \delta$

同样我们可以得出$G_i$和$G_j$最多相交$\delta n$

## 编辑距离模型
该文章的优势在于，对于错误率较高的情况(25%)，同样可以进行识别，相比之前对于SD的定义为10%的错误率。

该文章核心的概念在于，在模拟人类基因组中的此类事件时，将生殖系突变率（称为小突变）与新发结构变异（SV）率分开考虑。总结下来，结论是总的编辑距离不会超过10%。

总结：
错误模型分离 ：  
- 小突变（dM ≤ 0.15） ：种系突变（如单碱基替换、小Indels）
- 大结构变异（dG = d - dM） ：如转座子插入
- 泊松错误模型 ：通过k-mer保留概率$Pr(无突变)=e^{−kdM}$​估计SD相似度

假设所有SD均满足以上错误模型。

## Jaccard similarity
采用对于子串的`k-mer`集合的`Jaccard similarity`进行计算，即
$$J(X_i,X_j)=\frac{|X_i \cap X_j|}{|X_i \cup X_j|}$$

这里`Jaccard similarity`通过`MinHash`进行计算。

此处可变化为
$$\frac{|S(X_i \cup X_j) \cap S(X_i) \cap S(X_j)|}{|S(X_i \cup X_j)|}$$
其中$S(X_i)$表示通过哈希函数`h`选择最小的`s`个元素组成的子集。由于实际应用中`s`远小于$|X_i|$，因此效率较高。

如Jain等（2018）所示，MinHash技术在处理长字符串时可通过以下方法进一步优化性能。相较于直接计算集合$S(X_i​)$，可转而计算$S(W(X_i​))$，其中$W(X_i​)$是字符串$G_i$​对应的滑动窗口指纹。具体而言，$W(X_i​)$的生成方法是：以长度为w的滑动窗口遍历$G_i$​，并在每个窗口中选取哈希值最小的`k-mer`（若存在哈希值相同的情况，则选择最右侧的`k-mer`）。对于随机序列`A`，其滑动窗口指纹`W(A)`的期望大小为$\frac{2|A|}{w+1}$。

# 方法
SD Detection问题可以被视为以下
- 找到所有$(i,j)$，且$l \geq 1000$，使得$G_i$和$G_j$之间的编辑距离$\leq \delta$，并且$alignment(G_i, G_j) \geq l$，且该alignment不被一个更大且更好的alignment包含。

最简单的方法是对G进行一个自己的local alignment，但是显然这样太慢了，需要$|G|^2$的时间和空间。

## SD seeding
目的是找到所有$(G_i, G_j)$，称为seed，且长度都大于$1000 \cdot |1-\delta|=750$，且$G_i$和$G_j$都满足$SD$标准。

方法是枚举所有i，然后通过哈希查所有满足的j，使得winnowed MinHash的期望$\leq \tau$。

为验证字符串$G_i$和$G_j$在SD误差模型下的编辑错误是否$\leq d$，我们需计算其对应`k-mer`集合$X_i$和$X_j$的$Jaccard$相似度并验证其是否$\leq \delta$。为便于解释，假设两字符串等长（即$n=|G_i|=|G_j|$，非等长情况可类推）。设c为$X_i$与$X_j$的共有`k-mer`数量，t=n−k+1为集合$X_i$和$X_j$的大小（即k-mer总数），则两集合的Jaccard相似度可表示为：

$$J(X_i, X_j) = \frac{c}{2t-c}$$

同时这里假设$k-mer$各不相同。

单纯使用$\delta$去计算$\tau$的期望下界不可行，因为$\tau$的取值较大，且重复区间的差异并非完全随机分布，因此将$\delta$分解为$\delta_{M}+\delta_{G}$。$\delta_{M}$为小突变误差率，$\delta_{G}$为大片段插入缺失的误差率。

若存在跨越$G_i$和$G_j$的有效SD区域，集合$X_j$可以被认为是两个不相交的集合$X_{J}^{G}$和$X_{J}^{M}$

$X_{M}^{J}$代表最初的`k-mer`集合，并且后面会经过小突变，$X_{G}^{J}$代表了所有新的`k-mer`，在经过巨大的突变之后。通过这种方式可以将小突变和大突变分开。理想情况下，$X_{G}^{J}$和$X_i$应该没有交集。

同样的，我们定义$t=t_M+t_G$，其中$t_M=|X_{J}^{M}|$，$t_G=|X_{G}^{M}|$（$|X_i|=|X_j|$推断出$|X_{i}^{G}| \approx |X_{j}^{G}|$，由于$|X_{i}^{M}|$和$|X_{j}^{M}|$很相近）

设$\frac{c}{t_M}$是$X_{i}^{M}$和$X_{j}^{M}$中没有改变的部分，期望值可以得出如下
$$E(\frac{c}{t_M})=e^{-k \delta_M}$$

然后推导最终的相似度
$$J(X_i,X_j) \geq \frac{1-\delta_{G}}{1+\delta_{G}}J(X_{i}^{M},X_{j}^{M})$$

由于$J(X_{i}^{M},X_{j}^{M})=\frac{c}{2t_{M}-c}$

因此最终得出Jaccard similarity的期望值$\tau$
$$\tau = E[J(X_i,X_j)] \geq \frac{1-2\delta_{G}}{1+2\delta_{G}} \cdot \frac{1}{2e^{k\delta_M} - 1}$$




## SD extension
这里松弛限制，假设$|G_i|=|G_j|=n$。这里尝试向左右扩展$G_i$和$G_j$，直到winnowed MinHash的期望$\leq \tau$。停止条件是达到了SD大小上限，或者显著的相交了。

## SD chaining
最后对之前获取的潜在的SD区域，进行local chain和是sparse dynamic programming进行确定最终区域。

