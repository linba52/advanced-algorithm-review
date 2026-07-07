# 研究日志

本文档记录 *Tiny Pointers* 论文 review 的资料收集、阅读和理解过程。

## 过程目标

- 确认主论文及其稳定来源。
- 记录论文解决的主要问题、模型、结论和技术贡献。
- 收集论文中提到的相关研究。
- 补充论文没有重点讨论但与主题相关的研究，包括论文发表之后的新工作。
- 形成对论文构造、证明思路和意义的独立解释。

## 日志

- 2026-07-07：初始化用于保留写作过程的仓库结构。此时尚未记录技术性结论。

- 2026-07-07：确认主论文信息：
  - 题目：*Tiny Pointers*
  - 作者：Michael A. Bender、Alex Conway、Martin Farach-Colton、
    William Kuszmaul、Guido Tagliavini
  - arXiv：2111.12800，提交日期为 2021-11-24
  - arXiv 链接：https://arxiv.org/abs/2111.12800
  - DOI 链接：https://doi.org/10.48550/arXiv.2111.12800
  - DBLP 记录：https://dblp.org/rec/journals/corr/abs-2111-12800
  - DBLP 将该记录归为 CoRR abs/2111.12800 (2021)，类型为 informal/other
    publication。目前没有找到单独的会议版或期刊版。

## 第一轮阅读笔记

论文提出的问题看似简单：存储一个指针需要多少位？如果一个指针是指向长度为 `n` 的
数组的独立绝对地址，那么它需要 `log n` 位。论文指出，许多数据结构中的指针并不是
独立绝对地址；它们通常由某个 key、节点、用户或上下文拥有。tiny pointer 正是利用
这个上下文。即使 tiny pointer 本身短到无法单独确定位置，`(key, tiny pointer)` 这对
信息仍然可以共同确定位置。

论文的核心对象是 dereference table，它支持：

- `Allocate(k)`：为 key 或用户 `k` 保留一个槽位，并返回 tiny pointer `p`。
- `Dereference(k, p)`：使用 `k` 和 `p` 找回被保留的槽位。
- `Free(k, p)`：释放该槽位。

主要权衡是负载率 `1 - delta` 与 tiny pointer 大小之间的关系。论文证明了紧的界：

- 固定长度 tiny pointers：`Theta(log log log n + log delta^{-1})` 位。
- 可变长度 tiny pointers：期望 `Theta(1 + log delta^{-1})` 位，并且大 pointer
  出现的概率具有双指数尾界。

论文随后将这一抽象应用到五个问题：

- 带 tiny retrievers 的 relaxed data retrieval。
- 简洁动态二叉搜索树，包括基于旋转的树。
- stable dictionaries。
- 支持可变长度 value 的 dictionaries。
- 面向外存数组的 internal-memory stashes。

第一轮阅读中最重要的技术翻译是：

> key 像 ball，数组槽位像 bin，tiny pointer 记录这个 key 在自己的候选 bin 序列中
> 实际用了哪一个候选。

因此，pointer 大小本质上对应 probe index 的对数。整个问题变成：在任意动态插入、
删除和同一个 key 的重新插入下，如何维护一个分配，使每个活跃 key 都尽量使用自己
候选序列中靠前的位置。

- 2026-07-07：收集相关研究方向。论文本身将 tiny pointers 与 retrieval、succinct/
  dynamic trees、stable hashing/dictionaries、variable-size values、internal-memory
  stashes 以及 dynamic balls-and-bins 联系起来。我也搜索了论文之后的工作。暂时没有
  找到明显以 “tiny pointers” 为直接延续主题的后续论文；后续相关性主要出现在相邻
  方向，例如 succinct dictionaries、cell-probe lower bounds、retrieval structures、
  minimal perfect hashing 和 dynamic rank/select。

