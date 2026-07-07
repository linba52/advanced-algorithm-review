# 技术笔记

本文档记录对 *Tiny Pointers* 的技术理解如何逐步形成。

最终 review 需要解释：

- 论文解决了什么问题；
- 这个问题为什么重要；
- 论文引入了什么模型；
- 论文使用了什么技术；
- 核心直觉是什么；
- 构造如何工作；
- 为什么构造是正确的；
- 为什么这些界很强，甚至是最优的。

## 初始技术地图

### 模型

有一个包含 `n` 个槽位的数组或 store。每个活跃 key `k` 同一时刻最多拥有一个槽位。
dereference table 维护 key 到槽位的分配。关键点是：`p` 不是绝对槽位编号；
`Dereference(k, p)` 使用 owner key 和短 pointer 一起计算槽位。

### 固定长度上界

定理 1 给出了固定长度 tiny pointers：在负载率 `1 - delta` 下，pointer 大小为
`O(log log log n + log delta^{-1})` 位。

构造包含两个部分：

1. 主 load-balancing table 存储绝大多数分配。它把 store 分成大小为
   `Theta(delta^{-2} log delta^{-1})` 的 bucket。key `k` 被哈希到一个 bucket，
   `p` 记录 bucket 内的 slot。该表允许少量分配失败。
2. 稀疏副表存储主表 overflow。副表使用 power-of-two-choices 风格构造，bucket 大小为
   `Theta(log log n)`，因此局部 slot 可用 `O(log log log n)` 位描述。

固定长度 pointer 需要足够位数来说明它指向主表还是副表，并记录对应局部信息。`log
delta^{-1}` 项来自高负载所需的 bucket 大小；`log log log n` 项来自稀疏
power-of-two-choices 副表。

### 可变长度上界

定理 2 通过允许 pointer 长度变化，去掉了固定长度中的 `log log log n` 障碍。构造先将
高负载情形归约到常数负载情形，然后构造一个常数负载表，使其 pointer 长度有常数期望
并具有双指数尾界。

常数负载构造把 key 哈希到期望大小为 `Theta(log n)` 的 containers。每个 container
内部有多个 levels。第 `i` 层的 bucket 数大约是上一层的一半，bucket 容量是常数。若在
某一层分配失败，就尝试下一层。overflow arrays 用来隔离坏层，防止局部坏事件级联破坏
整个 container。

期望之所以小，是因为到达第 `i` 层需要连续失败，而每层只有常数比例 bucket 是满的。
尾界之所以是双指数，是因为较晚的 overflow 对应非常小的 level capacity，而这样的坏
事件概率关于 level size 指数级下降。

### 下界

定理 3 证明可变长度 tiny pointers 需要期望 `Omega(log delta^{-1})` 位。证明使用
potential 思想：有些 slot 对很多 key “有用”，但这样的 slot 不可能太多；反复随机插入
会迫使很多分配使用不那么有用的 slot，从而需要更长 pointer。

定理 4 证明固定长度 tiny pointers 需要
`Omega(log log log n + log delta^{-1})` 位。在已有 `Omega(log delta^{-1})` 的基础上，
新的部分来自 balls-and-bins 下界：如果每个 key 只有 `S = 2^s` 个 pointer 值，那么它
只有 `S` 个预定候选 bin。经典 sequential balls-into-bins 下界说明选择太少会导致某个
bin 过满。为了避免这种情况，必须有 `S = Omega(log log n)`，因此
`s = Omega(log log log n)`。

## 详细技术理解

### 为什么通常的 `log n` 下界不适用

普通指针必须独立标识 `n` 个 slot 中的一个，因此需要 `log n` 位。tiny pointer 刻意
不是自解释的。槽位由下面的式子确定：

```text
slot = Dereference(k, p)
```

其中 `k` 已经由调用方知道。因此用于定位的信息被拆分成：

- owner 的 key 或上下文 `k`；
- 返回的短 pointer `p`；
- 表中的公开或随机元数据。

所以 standalone address 的下界不是正确下界。正确问题应是：在 `k` 已知后，`p` 还
必须贡献多少位？

### 与 probe sequences 的等价视角

对每个 key `k`，可以想象一个无限 probe sequence：

```text
h_1(k), h_2(k), h_3(k), ...
```

如果 `Allocate(k)` 把 `k` 放在 `h_i(k)`，那么 tiny pointer 只需要编码 `i`，或编码
足以重建该局部选择的信息。`Dereference(k, p)` 的值就是被选中的 `h_i(k)`。

在这个视角下：

- keys 是 balls；
- slots 是 bins；
- 活跃分配必须无冲突；
- pointer 长度大致是 `log i`；
- 高负载意味着表中空 bin 很少；
- 困难情形是任意删除和重新插入。

“同一个 key 重新出现”很重要。第一次插入 key `k` 时，它的 hash/probe sequence 与表
状态独立；但 `k` 删除后再插入时，当前表状态可能已经包含它过去放置造成的影响。这种
历史依赖正是简单 probing schemes 难以动态分析的原因。

### 固定长度构造的操作级细节

目标：在负载率 `1 - delta` 下，让每个 tiny pointer 都有相同最大长度：

```text
O(log log log n + log delta^{-1})
```

构造组合两个表。

#### 主表

主表是一个 load-balancing table。

参数：

- 表大小 `m = (1 - delta/2)n`；
- 目标失败松弛大约为 `delta^2`；
- bucket 大小 `b = Theta(delta^{-2} log delta^{-1})`。

分配：

1. 将 key `k` 哈希到 bucket `h(k)`。
2. 如果 bucket 中有空 slot，就把 value 放入。
3. 返回局部 slot index `j` 作为 tiny pointer。
4. 如果 bucket 已满，则主表失败，将该分配交给副表。

解引用：

1. 重新计算 `h(k)`。
2. 将 `p` 解释为该 bucket 内的局部 slot。

释放：

1. 重新计算 `h(k)`。
2. 将 `p` 指示的局部 slot 标为空闲。

为什么短：

```text
local pointer size = log b
                   = O(log(delta^{-2} log delta^{-1}))
                   = O(log delta^{-1})
```

单独的主表还不够，因为它会让一小部分分配失败。它的作用是让 overflow set 足够小。

#### 副表

副表处理主表 overflow。

参数：

- 大小 `n' = delta n / 2`；
- 稀疏负载大约为 `Theta(1 / log log n')`；
- bucket 大小 `Theta(log log n')`。

分配：

1. 将 `k` 哈希到两个 bucket。
2. 选择空 slot 更多的 bucket 放入。
3. 返回一位信息说明使用了两个 bucket 中的哪一个，再加上局部 slot。

pointer 大小：

```text
1 + log(Theta(log log n')) = O(log log log n)
```

主表保证只有大约 `delta^2 n` 个活跃分配 overflow。副表有 `Theta(delta n)` 个 slot，
因此副表负载足够低，可以使用 power-of-two-choices 分析。

#### 组合后的 pointer 格式

固定长度 pointer 存储：

```text
table tag | local pointer
```

tag 表示主表或副表。局部 pointer 被 padding 到两种格式中的较大长度。

因此：

```text
size = O(1 + log delta^{-1} + log log log n)
```

#### 正确性检查清单

验证一次分配是否正确：

1. 如果主表成功，bucket free bits 或 free lists 保证只有一个 key 得到该主表 slot。
2. 如果主表失败，该分配进入副表。
3. 副表也只分配空 slot。
4. pointer 包含足够信息，可以从 `k` 重新计算相同位置。
5. `Free(k, p)` 使用同一路径解码，并释放准确的 occupied slot。

验证高概率成功：

1. Lemma 1 说明活跃主表失败数以高概率为 `O(delta^2 n)`。
2. 副表有 `Theta(delta n)` 个 slot，因此 overflow load 远低于副表容量阈值。
3. Lemma 2 说明副表分配以高概率成功。
4. 对各组件使用 union bound 得到整体高概率成功。

### 可变长度构造的操作级细节

目标：在负载率 `1 - delta` 下，tiny pointer 期望大小为：

```text
O(1 + log delta^{-1})
```

高负载情形通过复用主表/副表思想归约到常数负载情形：主表吸收几乎所有分配，副表只需
在常数负载下存储少量 overflow。因此新的核心问题是：

```text
构造一个常数负载 dereference table，使 pointer 期望长度为 O(1)。
```

#### Containers

构造将 key 哈希到大约 `n / log n` 个 containers。每个 container 最多允许：

```text
s = c log n
```

个 item。由 Chernoff bounds 可知，选择足够大的 `c` 后，每个 container 都会以高概率
低于容量。

每个 container 独立工作，使用 `O(log n)` 个 slot，因此所有 containers 合计使用
`O(n)` 个 slot。

#### Container 内部的 levels

在一个 container 内有 levels：

```text
i = 0, 1, ..., log_2 s - 1
```

第 `i` 层有：

```text
s_i = s / 2^i
```

个 buckets，每个 bucket 容量为常数 `b`。

预期行为是几何式的：

- 先尝试 level 0；
- 如果选中的 bucket 满了，尝试 level 1；
- 如果还失败，尝试 level 2；
- 依此类推。

由于每层 bucket 容量为常数且有效负载较低，进入下一层的概率至多是某个常数，例如
`1/b`。因此到达 level `i` 的概率大约至多为 `b^{-i}`。

#### 为什么需要 overflow arrays

朴素的 level cascade 可能在几个小 level 连续运气不好时崩坏。小 level 单独没有
高概率保证；一个坏 level 可能把过多工作推给所有后续 level。

修复方法是给每个 level `i` 一个大小为 `s_i` 的 overflow array。算法维护计数器 `L_i`：
到达过 level `i` 的分配数量，包括后来继续被送到更深层的分配。若下一层会被要求处理
过多对象，就把该分配转入当前 level 的 overflow array。

关键不变量是：

```text
L_i <= s_i
```

因为 level `i` 的 overflow array 也有大小 `s_i`，所以它不会耗尽。这样可以隔离坏运气：
如果某层表现异常差，代价在该层 overflow array 中局部支付，而不会传播到 container
后面的所有层。

#### 分配过程

对 key `k`：

1. 将 `k` 哈希到某个 container。
2. 如果 container 已经有 `s` 个 item，则失败。
3. 对每个 level `i`：
   - 增加 `L_i`；
   - 尝试将 `k` 放入 level `i` 的 load-balancing table；
   - 若成功，返回一个 pointer，编码：
     - level `i`；
     - “普通 level table”；
     - 局部 bucket slot；
   - 若下一层会超过限制，则把 `k` 放到 level `i` 的 overflow array，并返回一个
     pointer，编码：
     - 从后往前数的 level 编号；
     - “overflow”；
     - 局部 overflow slot。

overflow 情况下“从后往前数”的编码很微妙也很重要。如果 overflow 发生得很晚，`s_i`
很小，因此局部 slot index 很短。编码 `log s - 1 - i` 而不是 `i`，正好让这些晚期
overflow 的层号本身也短。

#### Pointer 大小界

如果分配在 level `i` 的 load-balancing table 成功，pointer 大小约为：

```text
O(log i) + O(1)
```

如果分配进入 level `i` 的 overflow array，pointer 大小约为：

```text
O(log s_i)
```

普通 level `i` 的概率：

```text
Pr[B_i] <= 1 / b^i
```

因为要到达 level `i`，前面所有选中的 buckets 都必须满。

level `i` overflow 的概率：

```text
Pr[O_i] <= exp(-Omega(s_i))
```

这是由 Lemma 1 推出，因为 level `i` 只有在关于其 size 指数小概率的坏事件下才会危险
过载。

因此期望 pointer 长度为：

```text
sum_i Pr[B_i] O(log i) + sum_i Pr[O_i] O(log s_i) = O(1)
```

尾界更强：pointer 长度超过 `ell` 的概率关于 `ell` 双指数小。直观地说，长 pointer
意味着要么到达指数级靠后的普通 level，要么在某个 capacity scale 下发生指数小概率的
overflow。

### 为什么可变长度优于固定长度

固定长度 pointer 必须为高概率可能出现的最坏 live pointer 预留足够位数。这个最坏
pointer 需要处理罕见但可能的深层 placement，因此产生 `log log log n` 项。

可变长度 pointer 只在罕见深层 placement 真的发生时付费。由于深层 placement 的概率
快速下降，期望成本仍为常数，再加上高负载的 `log delta^{-1}` 项。

这是论文最漂亮的概念点之一：

```text
fixed-size model: 每个 pointer 都为罕见事件付费
variable-size model: 只有罕见事件发生时才付费
```

### 可变长度 pointer 的下界

定理 3 证明期望大小为 `Omega(log delta^{-1})`。

证明直觉：

1. 令 `ell = Theta(delta^{-1})`。
2. 如果 pointer value `i` 表示 slot `h_i(k)`，那么短 pointer 只能使用前 `ell` 个候选。
3. 定义 slot `j` 的 potential：

```text
phi(j) = 映射到 j 的 key/candidate pairs (k, i <= ell) 所占比例
```

4. 高 potential slots 对很多 key 有用，但这种 slots 不可能太多，因为所有 slot 的总
   potential 只有 `ell`。
5. 低 potential 的空 slots 虽然多，但随机新 key 的前 `ell` 个候选不太可能命中它们。
6. 在高负载下的随机交替插入/删除 workload 中，常数比例的插入不能使用前 `ell` 个候选
   完成。
7. 这些插入需要 pointer value 大于 `ell`，因此至少需要
   `Omega(log ell) = Omega(log delta^{-1})` 位。

这个下界不是说每个 pointer 都必须很大，而是说期望长度不能太小，因为常数比例的随机
插入会被迫离开短候选区域。

### 固定长度 pointer 的下界

定理 4 为固定长度 pointer 增加了 `Omega(log log log n)`。

如果固定 pointer 大小是 `s`，那么每个 key 只有：

```text
S = 2^s
```

种可能 tiny-pointer values。因此每个 key 只有 `S` 个 candidate bins。

下界归约如下：

1. 假设存在固定 pointer 大小 `s = o(log log log n)` 的 dereference table。
2. 则 `S = 2^s = o(log log n)`。
3. 用该 dereference table 定义一个 sequential balls-into-bins 过程：每个 ball/key
   带有由 `Dereference(k, 1), ..., Dereference(k, S)` 导出的 `S` 个候选 bins。
4. 经典 balls-and-bins 下界说明，任何只有 `S` 个选择的 sequential process 最大负载
   至少为：

```text
Omega((log log m) / S)
```

5. 因为 `S = o(log log m)`，某个 bin 会得到 `omega(1)` 个 balls。
6. 但 dereference table 的成功分配在原始 `n` 个 slots 中无冲突；将 `n` 个 slots 分组
   成 `m = Theta(n)` 个 bins 后，每个 grouped bin 只能包含 `O(1)` 个 balls。
7. 产生矛盾。

因此 `S` 至少是 `Omega(log log n)`，所以：

```text
s >= Omega(log log log n)
```

结合可变长度下界，固定长度 pointer 得到：

```text
s = Omega(log log log n + log delta^{-1})
```

### 技术上最厉害的地方

1. 抽象很干净：它把“pointer 表示什么”和“pointer 自身存多少位”分离开。
2. 固定长度和可变长度模型中都有紧的上下界。
3. 构造能处理任意动态更新，包括同一个 key 的删除和重新插入；很多简单 probing schemes
   在这种情况下缺乏清晰分析。
4. 可变长度构造中的 overflow arrays 不是临时补丁，而是用于局部化坏事件并保持强尾界
   的核心机制。
5. 论文把同一工具转化为多个黑盒数据结构变换，说明 pointer overhead 是反复出现的
   元数据瓶颈，而不是某个孤立问题。

