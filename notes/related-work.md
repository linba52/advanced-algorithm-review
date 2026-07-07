# 相关研究笔记

本文档收集 *Tiny Pointers* review 所需的相关研究。

相关研究按以下几类组织：

- 论文之前、用于引出该论文的问题背景；
- 论文中引用或讨论的相关工作；
- 与论文相关但不是论文核心讨论对象的工作；
- 论文发表之后的新相关工作。

## 1. 论文提到的早期与同期工作

### Retrieval 与 Relaxed Retrieval

论文的第一个应用是 dynamic retrieval problem 的一个放松版本。在静态 retrieval 中，
数据结构为一个 key 集合存储函数值；对集合内 key 必须返回正确 value，对集合外 key
可以返回任意值。论文引用了以下工作：

- Stephen Alstrup、Gerth Brodal、Theis Rauhe，"Optimal static range reporting in
  one dimension"（STOC 2001）。论文把这一脉络用于 retrieval 相关下界背景。
- Erik Demaine、Friedhelm Meyer auf der Heide、Rasmus Pagh、Mihai Patrascu，
  "De dictionariis dynamicis pauco spatio utentibus"（LATIN 2006）。这是 dynamic
  succinct dictionary/retrieval 的背景之一。
- Martin Dietzfelbinger、Rasmus Pagh，"Succinct data structures for retrieval and
  approximate membership"（ICALP 2008）。
- Martin Dietzfelbinger、Stefan Walzer，"Constant-time retrieval with o(log m) extra
  bits"（STACS 2019）。

review 中需要强调的一点是：普通 dynamic retrieval 在论文关心的设置中有
`Omega(n log log n)` 类型的元数据下界。Tiny pointers 并不与该下界矛盾；它通过
改变接口，让用户额外保存一个 tiny retriever/hint，从而绕开原问题的下界前提。

### Succinct Trees 与基于旋转的 BST

论文强调，succinct tree representation 可以用 `O(n)` 或 `2n + o(n)` 位编码一棵
二叉树的形状，但高效支持旋转一直是瓶颈。论文引用了：

- Rajeev Raman、Satti Srinivasa Rao，"Succinct dynamic dictionaries and trees"
  （ICALP 2003）。
- J. Ian Munro、Venkatesh Raman、Adam J. Storm，"Representing dynamic binary trees
  succinctly"（SODA 2001）。
- Gonzalo Navarro、Kunihiko Sadakane，"Fully functional static and dynamic succinct
  trees"（ACM Transactions on Algorithms, 2014）。
- Joshimar Cordova、Gonzalo Navarro，"Simple and efficient fully-functional succinct
  trees"（Theoretical Computer Science, 2016）。

review 应强调 tiny pointers 在这里有用的原因：基于旋转的搜索树天然依赖指针，
普通 parent/child navigation 很容易写，但在 succinct representation 中普通指针太贵。

### Stable Hashing 与 Stable Dictionaries

第三个应用关注稳定性：value 插入后，它的位置不应移动。当外部用户持有指向数据结构
内部 value 的指针或引用时，这一点很重要。论文将这个问题与以下工作联系起来：

- Don Knuth 关于 open addressing 的早期笔记以及 *The Art of Computer Programming*
  第三卷。
- Per-Ake Larson 关于 uniform hashing 与 external hashing 的工作。
- Peter Sanders，"Hashing with Linear Probing and Referential Integrity"
  （arXiv:1808.04602, 2018）。
- Demaine 等人的 dynamic succinct dictionaries 工作，其中 stable hashing 在某些
  方案中有已知的 `Theta(log log n)` 开销。
- 工程中的稳定哈希表，例如 Google Abseil 和 Facebook F14。

review 中应区分 “stable values” 和 “stable table cells”：Tiny Pointers 允许用户保留
指向 value 的稳定 tiny pointer，即使 dictionary 自身的元数据布局发生变化。

### Balls and Bins

论文的主要技术引擎是 dynamic balls-and-bins 视角。早期相关工作包括：

- Azar、Broder、Karlin、Upfal 提出的 "power of two choices" 结果，说明两个随机选择
  相比一个随机选择可以指数级降低最大负载。
- Berthold Voecking 关于 asymmetric balanced allocations 的研究；Tiny Pointers 论文
  在固定长度 tiny pointer 下界中使用了这一类结果。
- Philipp Woelfel 对使用简单哈希函数的 asymmetric balanced allocation 的分析。
- Larson 对随机插入/删除下 uniform hashing 的分析；而一般动态插入/删除序列下的简单
  probing scheme 仍然很难分析。

review 应解释为什么动态版本比静态 balls-into-bins 更难：当同一个 key 被删除后再插入，
当前表状态已经可能依赖它过去的 probe sequence。

### Internal-Memory Stashes 与 Page Tables

第五个应用重新讨论了用于定位外部数组中对象的 stashes。论文追溯到：

- Gonnet 和 Larson 关于 external hashing with limited internal storage 的工作。
- Larson 和 Kajla 关于 one-access retrieval/file organization 的工作。
- 现代地址转换和 page table 的动机。
- Bender 等人的 adaptive filter 工作，论文将其改造为 stash。

review 应把这一应用描述为一个非常具体的 pointer-compression 问题：internal stash 是
元数据，它的任务是指向一个更大的外部内存。

## 2. 论文没有重点展开但相关的工作

### Hash Tables 与 Succinct Dictionaries 的大背景

这篇论文属于一个更大的研究议程：让动态 hashing 和 dictionaries 在保持快速操作的同时，
尽量接近信息论空间下界。相关背景包括：

- Pagh 和 Rodler 的 cuckoo hashing。
- Pagh 和 Pagh 关于 constant-time、optimal-space uniform hashing 的工作。
- Arbitman、Naor、Segev 的 de-amortized/backyard cuckoo hashing。
- Bender、Farach-Colton、John Kuszmaul、William Kuszmaul、Liu 的
  "On the Optimal Time/Space Tradeoff for Hash Tables"（arXiv:2111.00602）。该工作给出
  operation time 与 wasted bits per key 之间的权衡。

这对 review 很重要，因为 tiny pointers 不只是“短地址”；它们是减少 metadata overhead
的机制，而 metadata overhead 正是动态结构难以接近信息论空间的原因之一。

### Minimal Perfect Hashing 与 Monotone Minimal Perfect Hashing

Minimal perfect hashing 与 tiny pointers 不是同一个问题，因为它通常是静态问题，
并将 key 无冲突地映射到一个紧凑范围。但它们相邻，因为两者都在问：给定 key 以后，
还需要多少额外信息才能恢复 index、rank 或位置？相关后续工作包括：

- Sepehr Assadi、Martin Farach-Colton、William Kuszmaul，"Tight Bounds for Monotone
  Minimal Perfect Hashing"（arXiv:2207.10556；SODA 2023）。
- Dmitry Kosolobov，"Simplified Tight Bounds for Monotone Minimal Perfect Hashing"
  （arXiv:2403.07760）。

这些工作可作为 review 中讨论 `log log log` 项的参照：当数据结构只存“刚好够用”的
order/index 信息时，这类迭代对数项经常出现。

## 3. *Tiny Pointers* 之后的相关工作

### Dynamic Succinct Dictionaries 与下界

Li、Liang、Yu、Zhou 的 "Tight Cell-Probe Lower Bounds for Dynamic Succinct Dictionaries"
（arXiv:2306.02253；FOCS 2023）证明了 dynamic succinct dictionaries 中时间/空间
权衡的匹配下界。它不是 tiny pointers 的直接续作，但它澄清了更大的研究图景：
near-optimal space 的 dynamic dictionary 往往伴随不可避免的 update-time tradeoff。

来源：https://arxiv.org/abs/2306.02253

### 实用 Retrieval Structures

Dillinger、Huebschle-Schneider、Sanders、Walzer 的
"Fast Succinct Retrieval and Approximate Membership using Ribbon"（arXiv:2109.01892；
SEA 2022）以及 Becht、Lehmann、Sanders 的
"Brief Announcement: Parallel Construction of Bumped Ribbon Retrieval"
（arXiv:2411.12365）说明 retrieval structures 在实践方向上仍然活跃，重点包括低开销、
局部性和并行构造。

来源：

- https://arxiv.org/abs/2109.01892
- https://arxiv.org/abs/2411.12365

这与 *Tiny Pointers* 相关，因为论文把 retrieval 作为一个应用目标；但 Tiny Pointers
更偏理论和动态模型，它通过用户保存 hint 来改变接口。

### Minimal Perfect Hashing 综述

Lehmann、Mueller、Pagh、Pibiri、Sanders、Vigna、Walzer 的 2025/2026 年综述
"Modern Minimal Perfect Hashing: A Survey"（arXiv:2506.06536）记录了 perfect hashing
近年来的实践进展。它同样不是直接续作：minimal perfect hashing 是静态的，而 tiny
pointers 是动态且 owner-contextual 的。不过二者都关心如何用极少额外信息编码位置或
索引。

来源：https://arxiv.org/abs/2506.06536

### Dynamic Rank/Select 与 Succinct Ordered Dictionaries

Kuszmaul、Liang、Zhou 的 "Succinct Dynamic Rank/Select: Bypassing the Tree-Structure
Bottleneck"（arXiv:2510.19175；SODA 2026）处理了相关的 succinct dynamic representation
瓶颈。它与 review 中关于 succinct trees 的讨论特别相关：在 *Tiny Pointers* 之后，
更大的研究方向仍然在寻找绕过动态结构元数据开销的方法。

来源：https://arxiv.org/abs/2510.19175

## 4. review 的阶段性观点

定位 *Tiny Pointers* 最有力的方式，不是把它看作某一个数据结构的小技巧，而是把它看作
一篇“重新理解元数据”的论文。许多 succinct data-structure 问题困难，是因为 metadata
pointer 太大。论文的关键动作是观察到：metadata 通常带有上下文，例如 owner key、tree
node 或 logical item 已经是已知信息。一旦 dereferencing 被允许依赖这个上下文，普通
standalone pointer 的信息论下界就不再适用。

相关研究也说明了论文的限制：tiny pointers 并不能让任意独立地址变短。它们最强的场景
是周围数据结构能够控制 allocation、access 和 ownership。

