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
到目前为止，我们假设`n`的值是固定的，并且$n = |G_i| = |G_j|$。  
现在我们取消这一限制，并尝试在两个方向上尽可能地扩展任意一个种子 SD，以确保能够找到“真实” SD 的边界。

这种扩展可以通过逐步增加`n`和`m`的值来实现（每步花费$O(log\ s)$的时间），其本质是不断扩展集合$W(X_i)$ 和 $W(X_j)$：  
对于任何出现在位置`i + n + 1`和`j + m + 1`的 minimizer，都将其加入有序集合`L`中。

这里我们使用了与前面步骤（见第 3.4 节）相同的数据结构，并持续扩展`SD`区域，直到 **winnowed MinHash** 估计值低于阈值$\tau$。

---

如果`n` 和 `m` 都变得过大，我们也会终止扩展（我们在 `SEDEF` 中限制 `SD` 最多为 `1 Mbp` 长度，参考 `WGAC`）。

注意以下几点：

- 项 $|S(W(X_i) \cup W(X_j))|$ 会不断增长；
- 而 $|S(W(X_i) \cap W(X_j))|$ 会在两个区域不再相似时保持不变；
- 这会逐步降低 **Jaccard 相似度估计值**。

---

我们还会在字符串 $G_i$ 和 $G_j$ 开始发生重叠时中止扩展。  
值得注意的是，我们也可以**反向执行**该扩展过程，即通过逐步减小`i`和`j`的值，并应用相同的技术进行处理。

最终，我们报告所有 **MinHash 估计值大于** $\tau$ 的最大$G_i$和$G_j$。

同时考虑利用`q gram`优化这个过程：对于任意两个基因组片段$G_i$和$G_j$，若其编辑错误率低于$\delta$且满足`SD`的错误模型，则它们至少共享$n(1-\delta_{G}-q\delta_{M})-(np_G+1) \cdot (q+1)$个`q gram`，其中$p_{G}$是1个base pair的期望对应gap数，这个修改使得我们针对给定的$p_G$值，无损地排除所有不满足`SD`误差模型的子串对$(G_i, G_j)$。

通过上述算法对整个人类基因组进行分析时，由于基因组中存在大量小重复序列，会产生超过5亿个潜在SD区域。为缓解这一问题，我们在种子SD检测过程中仅使用包含至少一个非重复屏蔽核苷酸 的k-mer。同时，为了允许进化过程中重复序列插入SD的情况，SD扩展步骤会利用所有可用k-mer进行种子扩展。最后，我们通过为每个潜在SD区域填充预定义数量的碱基（与潜在区域大小相关的函数）来进一步提高在目标区域内定位大型SD的概率。

## SD chaining
在确定潜在的SD区域后，我们枚举所有满足SD标准的局部比对（长度为1000）。为了高效完成这一任务，SEDEF采用了一种两级局部链算法，类似于Abouelhoda和Ohlebusch（2003）及Myers和Miller（1995）提出的方法。第一阶段，我们使用种子扩展法构建匹配种子列表（种子长度≥11），并通过Abouelhoda和Ohlebusch（2003）及Myers和Miller（1995）描述的O(nlog n)稀疏动态规划算法，寻找由这些种子形成的最长链。在此步骤中，我们将种子间的最大间隙限制为l·δ_G，以尽可能紧密地聚类种子。找到这些初始链（可能跨度<1000bp）后，我们通过允许更大的间隙进一步优化它们，形成较大的最终链。为了与WGAC兼容（其允许SD内的任意大间隙且不惩罚间隙延伸），我们在构建SD链时采用仿射缺口罚分；然而，为避免低质量比对，我们将间隙限制为不超过10000bp。链式比对与全局比对结合进行，后者通过KSW2库实现，该库利用单指令多数据流（SIMD）并行化技术（通过流式SIMD扩展指令集SSE）加速全局序列比对（Li, 2017）。重要的是，我们以标准BEDPE格式报告所有比对结果，并附带对应的CIGAR格式编辑字符串（Li et al., 2009），以及其他类似WGAC的有用指标，如Kimura双参数遗传距离（Kimura和Ohta, 1972）和Jukes-Cantor距离（Jukes和Cantor, 1969）。   

在实验中，我们为种子阶段和链式阶段分别设置k=12和k=11（注：此参数可由用户配置）。虽然较小的k值可能提升灵敏度，但我们发现这种改进微乎其微，不值得增加运行时间。另一方面，较大的k值虽能缩短运行时间，但会降低灵敏度。 


# 程序
这个程序写的比较有意思，把几个步骤通过命令行传到可执行程序里，然后用脚本串起来流程。

## samtools 
会创建一个有关`.fa`文件的元数据

执行
```bash
samtools faidx test2.fa 
```

生成了这样的格式

```bash
ref|NC_000004.12|:190063426-190092984	29559	94	70	71
```

生成的 test2.fa.fai 是一个 ​5 列的文本文件，各列含义如下：

​1. 序列名称（如染色体名称 chr1）。

2. 序列总长度（单位为 bp）。

​3. 序列在 FASTA 文件中的起始偏移量（以字节为单位，包括换行符）。

​4. 列：每行的碱基数（最后一行除外）。

​5. 列：每行的总长度（包括换行符，如 Linux 换行符为 \n，占 1 字节）。

# Seed
使用以下命令行，
```bash
for j in `seq 0 $((numchrs - 1))`; do # reference
	for i in `seq $j $((numchrs - 1))`; do # query; query < reference
		for m in n y ; do
			[ "$m" == "y" ] && rc="-r" || rc="";
			echo "${TIME} -f'TIMING: %e %M' sedef search -k 12 -w 16 ${rc} ${input} -t $i $j >${output}/seeds/${i}_${j}_${m}.bed 2>${output}/log/seeds/${i}_${j}_${m}.log"
		done
	done
done | tee "${output}/seeds.comm" | ${TIME} -f'Seeding time: %E' parallel --will-cite -j ${jobs} --bar --joblog "${output}/seeds.joblog"
```

# chaining
chaining的过程是，首先需要生成一些anchors，生成anchors后对这些anchors进行dp连接

## dp
首先把区间q的端点`[q, q+l-1]`存进xs里，然后按坐标排序，建一颗线段树。

排好序后开始dp，从小到大枚举区间r的端点`r+l-1`，假设当前枚举的坐标为i，则查询$[i-gap, i-1]$区间的最大值
。

假设查询的最大值的位置为j，那么考虑更新dp值：
$$dp_i=dp_j+w-gap$$
其中$w=SCORE \cdot a.has_u +
              \frac{SCORE}{2} * (a.l - a.has_u)$

$gap = a_q - (p_q + p_l) + a_r - (p_r + p_l)$

求出所有dp值后，对dp值进行排序，然后backtrace一下，标记路径上访问的节点，不能重复访问。
这样收集了一个path，把所有backtrace获得的path都收集起来。

然后开始对这些路径进行一些筛选：求出最大的区域$span=max(rhi - rlo, qhi - qlo)$，如果$span < 750$，或者所有区域都被masked了，则跳过。
否则将匹配收集起来：`Hit a{query_ptr, qlo, qhi, ref_ptr, rlo, rhi, up}`。

收集好后，开始进行refine，对收集的hit进行再次挑选
- 首先看两个区间的相交区域： `qo = max(0, min(orig.query_start + qhi, orig.ref_start + rhi) - max(orig.query_start + qlo, orig.ref_start + rlo))`
- 然后计算两个区间的非重叠长度`rhi - rlo - qo`和`qhi - qlo - qo`，如果两个非重叠区域的大小都小于500，那么跳过
- 然后枚举所有前驱anchor，然后先比较一下两个anchor的端点，如果前驱锚点结束位置超出当前锚点结束位置，或者前驱锚点起始位置在当前锚点之后，就跳过
- 然后过滤gap，如果gap过大也过滤掉，如果两个anchor的gap>=10000，也跳过
- 如果两个anchor的gap有重叠，也过滤掉
- 然后计算一下连接的得分，大概是`dp[i]=dp[j]+score-miss-gap`
- dp之后，再像之前一样进行一次backtrace
- 然后开始合并alignment的结果，扫描backtrace获取的所有path的锚点，枚举相邻两个，如果相邻两个锚点相交，那么把他们的alignment结果进行合并，使用merge
  - merge的大概流程是，假设后面的anchor为b，前面的anchor为a，首先修剪b的结尾和a的开头
  - 然后考虑两个anchor之间的gap，如果gap<=1000，那么把两个gap进行alignment，然后把alignment的结果换成CIGAR放在最后，如果gap>1000，则直接考虑两种情况，一种是把gap填成gap，另一种是填成mismatch，然后把这两种情况分别做alignment，再比较错误率，选择错误率较小的方法填进去
  - 最后再合并字符串，把a、gap、b加起来
  - 最后再重新求一遍align_a和align_b和error