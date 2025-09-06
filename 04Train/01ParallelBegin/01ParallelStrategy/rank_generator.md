## 大模型分布式并行基础——并行策略

### 通信域通信优先级图示
依据通信对带宽的要求，划出通信域优先级图示：
![comm_priority](./comm_priority.jpg)
Intra Node 指将通信更多放在机内，对带宽要求比较高；Inter Node 则是将通信放在机间，对带宽要求相对低。
给出该图示，将有助于接下来推导通信域的创建，讲明为何一般会按照 `tp-cp-ep-dp-pp` 的顺序作为创建 order.

### Megatron-LM 建组逻辑

#### rank generator
分为两个类型，稠密部分（tp-cp-dp-pp），MoE 部分（tp-ep-dp-pp）也叫做（etp-ep-edp-pp）与 world_size 的关系为：

$$ world\_size = tp\_count \times dp\_count $$

$$\text{world\_size} = tp \times cp \times dp \times pp = etp \times ep \times edp \times pp$$

若为**稠密模型**，将选用前者建立通信域；若为 **MoE 模型**，可在 ATTN 部分选用前者建立通信域，在 MLP 部分选用后者。这意味着，原本需要 CP8 x EP8 = 64卡，在这种设计下针对 MoE 模型仅需要8卡，这种设计也叫 **MoE Folding Parallel**. 因此将会引出两种 Rank Generator —— decoder_rank_generator, expert_decoder_rank_generator.

本节以双机16卡进行通信域建立的讲解例子，先给出并行策略并画出切分图示，在下一章节讲述为什么会形成这样的通信域划分。
##### · Dense 模型 TP4-PP2-DP2
##### · MoE 模型 ETP1-PP2-EP4-EDP2
![rank_generator](./rank_generator.jpg)

#### 为通信域划分 ranks

核心在于 [rank_generator.py](./rank_generator.py) 中 `generate_masked_orthogonal_groups`.根据通信优先级图示，建立通信组的关键在于**依据 $order$ 生成 `ranks`**.默认建立通信域 $order$ 为 `tp-cp-ep-dp-pp`，两个主要步骤，即求得 group_index 与 global_rank：

先看 TP4-PP2-DP2 求 DP 域，原始 `order(tp-cp-ep-dp-pp)` 确定通信域中的 ranks，没有 cp 域与 ep 域，直接跳过即可
1. 求什么域 mask 什么域，剩余通信域（tp-pp）求 dp_group_index 形式化一下就是：
$\text{dp\_group\_index} = \text{tp\_rank} + \text{pp\_rank} \times \text{tp\_size}$

2. 计算出通信域中存在多少 group 后确定全局 rank 中哪些 rank 分别属于哪个 group_index（以 range 形式遍历），形式化一下：
$$\text{global\_rank} = \text{tp\_rank} + \text{dp\_rank} \times \text{tp\_size} + \text{pp\_rank} \times \text{tp\_size} \times \text{dp\_size} \quad (2)$$

$$\text{dp\_group[dp\_group\_index]} = \text{tp\_rank} + range(0, \text{dp\_size}) \times \text{tp\_size} + \text{pp\_rank} \times \text{tp\_size} \times \text{dp\_size} \quad (3)$$

可以观察到式(1)是将式(2)中要求的**域 mask（对应 rank 置为0，对应 size 置为1）** 而导出的，而式(3)又是由式(1)与式(2)推导出来的。

依据上述并行策略的具体推导例子：
```python
tp4 ==> tp_rank: [0, 1, 2, 3]
pp2 ==> pp_rank: [0, 1]

# 这里求 dp，mask dp，先利用式(1)求总共有少 dp 组及其下标
# 然后再利用式(2)中 dp_rank 进行遍历求出结果
dp_group_index = tp_rank + pp_rank * tp_size
"""
dp_group_index = 0 + 0 * 4 = 0
dp_group_index = 1 + 0 * 4 = 1
dp_group_index = 2 + 0 * 4 = 2
dp_group_index = 3 + 0 * 4 = 3 # pp0 遍历完 tp0, tp1, tp2, tp3
dp_group_index = 0 + 1 * 4 = 4
dp_group_index = 1 + 1 * 4 = 5
dp_group_index = 2 + 1 * 4 = 6
dp_group_index = 3 + 1 * 4 = 7 # pp1 遍历完 tp0, tp1, tp2, tp3
"""
dp_group[dp_group_index] = tp_rank + range(0, dp_size) * tp_size + pp_rank * tp_size * dp_size
"""
dp_rank 以 range(0, dp_size) 形式呈现
dp_group[0] = 0 + range(0, 2) * 4 + 0 * 4 * 2 ==> [0, 4]
dp_group[1] = 1 + range(0, 2) * 4 + 0 * 4 * 2 ==> [1, 5] 
dp_group[2] = 2 + range(0, 2) * 4 + 0 * 4 * 2 ==> [2, 6]
dp_group[3] = 3 + range(0, 2) * 4 + 0 * 4 * 2 ==> [3, 7] # pp0 遍历完 tp0, tp1, tp2, tp3
dp_group[4] = 0 + range(0, 2) * 4 + 1 * 4 * 2 ==> [8, 12]
dp_group[5] = 1 + range(0, 2) * 4 + 1 * 4 * 2 ==> [9, 13]
dp_group[6] = 2 + range(0, 2) * 4 + 1 * 4 * 2 ==> [10, 14]
dp_group[7] = 3 + range(0, 2) * 4 + 1 * 4 * 2 ==> [11, 15] # pp1 遍历完 tp0, tp1, tp2, tp3
"""
```
以上述形式继续推导 tp 域，pp 域：
```python
dp2 ==> dp_rank: [0, 1]
pp2 ==> pp_rank: [0, 1]

# 这里求 tp，mask tp，先利用式(1)求总共有多少 tp 组及其下标
# 然后再利用式(2)中 tp_rank 遍历求出结果
tp_group_index = dp_rank + pp_rank * dp_size
"""
tp_group_index = 0 + 0 * 2 = 0
tp_group_index = 1 + 0 * 2 = 1 # pp0 遍历完 dp0, dp1
tp_group_index = 0 + 1 * 2 = 2
tp_group_index = 1 + 1 * 2 = 3 # pp1 遍历完 dp0, dp1
"""

tp_group[tp_group_index] = range(0, tp_size) + dp_rank * tp_size + pp_rank * tp_size * dp_size
"""
tp_group[0] = range(0, 4) + 0 * 4 + 0 * 4 * 2 ==> [0, 1, 2, 3]
tp_group[1] = range(0, 4) + 1 * 4 + 0 * 4 * 2 ==> [4, 5, 6, 7] # pp0 遍历完 dp0, dp1
tp_group[2] = range(0, 4) + 0 * 4 + 1 * 4 * 2 ==> [8, 9, 10, 11]
tp_group[3] = range(0, 4) + 1 * 4 + 1 * 4 * 2 ==> [12, 13, 14, 15] # pp1 遍历完 dp0, dp1
"""
```

```python
tp4 ==> tp_rank: [0, 1, 2, 3]
dp2 ==> dp_rank: [0, 1]

# 这里求 pp，mask pp，先利用式(1)求总共有多少 pp 组及其下标
# 然后再利用式(2)中 pp_rank 遍历求出结果
pp_group_index = tp_rank + dp_rank * tp_size
"""
pp_group_index = 0 + 0 * 4 = 0
pp_group_index = 1 + 0 * 4 = 1
pp_group_index = 2 + 0 * 4 = 2
pp_group_index = 3 + 0 * 4 = 3 # dp0 遍历完 tp0, tp1, tp2, tp3
pp_group_index = 0 + 1 * 4 = 4
pp_group_index = 1 + 1 * 4 = 5
pp_group_index = 2 + 1 * 4 = 6
pp_group_index = 3 + 1 * 4 = 7 # dp1 遍历完 tp0, tp1, tp2, tp3
"""

pp_group[pp_group_index] = tp_rank + dp_rank * tp_size + range(0, pp_size) * tp_size * dp_size
"""
pp_group[0] = 0 + 0 * 4 + range(0, 2) * 4 * 2 ==> [0, 8]
pp_group[1] = 1 + 0 * 4 + range(0, 2) * 4 * 2 ==> [1, 9]
pp_group[2] = 2 + 0 * 4 + range(0, 2) * 4 * 2 ==> [2, 10]
pp_group[3] = 3 + 0 * 4 + range(0, 2) * 4 * 2 ==> [3, 11] # dp0 遍历完 tp0, tp1, tp2, tp3
pp_group[4] = 0 + 1 * 4 + range(0, 2) * 4 * 2 ==> [4, 12]
pp_group[5] = 1 + 1 * 4 + range(0, 2) * 4 * 2 ==> [5, 13]
pp_group[6] = 2 + 1 * 4 + range(0, 2) * 4 * 2 ==> [6, 14]
pp_group[7] = 3 + 1 * 4 + range(0, 2) * 4 * 2 ==> [7, 15] # dp1 遍历完 tp0, tp1, tp2, tp3
"""
```

回头看一下 rank 划分图的 Dense 部分，这就完成了通信域的划分。接下来推导一个 MoE 场景的例子：

ETP1-PP2-EP4-EDP2 求 EP 域，依旧遵从原始 `order(tp-cp-ep-dp-pp)` 来开始计算：

1. 求什么域 mask 什么域，剩余通信域（etp-edp-pp）求 ep_group_index 形式化一下就是：
$$\text{ep\_group\_index} = \text{etp\_rank} + \text{edp\_rank} \times \text{etp\_size} + \text{pp\_rank} \times \text{etp\_size} \times \text{edp\_size} \quad (1)$$

2. 计算出通信域中存在多少 group 后确定全局 rank 中哪些 rank 分别属于哪个 group_index（以 range 形式遍历），形式化一下：
$$\text{global\_rank} = \text{etp\_rank} + \text{ep\_rank} \times \text{etp\_size} + \text{edp\_rank} \times \text{etp\_size} \times \text{edp\_size} + \text{pp\_rank} \times \text{etp\_size} \times \text{ep\_size} \times \text{edp\_size} \quad (2)$$

$$\text{ep\_group[ep\_group\_index]} = \text{etp\_rank} + range(0, \text{ep\_size}) \times \text{etp\_size} + \text{edp\_rank} \times \text{etp\_size} \times \text{ep\_size} + \text{pp\_rank} \times \text{etp\_size} \times \text{ep\_size} \times \text{edp\_size} \quad (3)$$

依据上述并行策略的具体推导例子：
```python
etp1 ==> etp_rank: [0]
edp2 ==> edp_rank: [0, 1]
pp2 ==> pp_rank: [0, 1]

# 这里求 ep，mask ep，先利用式(1)求总共有少 ep 组及其下标
# 然后再利用式(2)中 ep_rank 进行遍历求出结果
ep_group_index = etp_rank + edp_rank * etp_size + pp_rank * etp_size * edp_size
"""
ep_group_index = 0 + 0 * 1 + 0 * 1 * 2 = 0
ep_group_index = 0 + 1 * 1 + 0 * 1 * 2 = 1 # pp0 遍历完 edp0, edp1
ep_group_index = 0 + 0 * 1 + 1 * 1 * 2 = 2
ep_group_index = 0 + 1 * 1 + 1 * 1 * 2 = 3 # pp1 遍历完 edp0, edp1
"""
ep_group[ep_group_index] = etp_rank + range(0, ep_size) * etp_size + edp_rank * etp_size * ep_size + pp_rank * etp_size * ep_size * edp_size
"""
ep_group[0] = 0 + range(0, 4) * 1 + 0 * 1 * 4 + 0 * 1 * 4 * 2 ==> [0, 1, 2, 3]
ep_group[1] = 0 + range(0, 4) * 1 + 1 * 1 * 4 + 0 * 1 * 4 * 2 ==> [4, 5, 6, 7] # pp0 遍历完 edp0, edp1
ep_group[2] = 0 + range(0, 4) * 1 + 0 * 1 * 4 + 1 * 1 * 4 * 2 ==> [8, 9, 10, 11]
ep_group[3] = 0 + range(0, 4) * 1 * 1 * 1 * 4 + 1 * 1 * 4 * 2 ==> [12, 13, 14, 15] # pp1 遍历完 edp0, edp1
"""
```
以上述形式继续推导 edp 域，pp 域（etp为1，则会默认生成16个 groups）
```python
etp1 ==> etp_rank: [0]
ep4 ==> ep_rank: [0, 1, 2, 3]
pp2 ==> pp_rank: [0, 1]

# 这里求 edp，mask edp，先利用式(1)求总共有少 edp 组及其下标
# 然后再利用式(2)中 edp_rank 进行遍历求出结果
edp_group_index = tp_rank + ep_rank * etp_size + pp_rank * etp_size * ep_size
"""
edp_group_index = 0 + 0 * 1 + 0 * 1 * 4 = 0
edp_group_index = 0 + 1 * 1 + 0 * 1 * 4 = 1
edp_group_index = 0 + 2 * 1 + 0 * 1 * 4 = 2
edp_group_index = 0 + 3 * 1 + 0 * 1 * 4 = 3 # pp0 遍历完 ep0, ep1, ep2, ep3
edp_group_index = 0 + 0 * 1 + 1 * 1 * 4 = 4
edp_group_index = 0 + 1 * 1 + 1 * 1 * 4 = 5
edp_group_index = 0 + 2 * 1 + 1 * 1 * 4 = 6
edp_group_index = 0 + 3 * 1 + 1 * 1 * 4 = 7 # pp1 遍历完 ep0, ep1, ep2, ep3
"""
edp_group[edp_group_index] = etp_rank + ep_rank * etp_size + range(0, 2) * etp_size * ep_size + pp_rank * etp_size * ep_size * edp_size
"""
edp_group[0] = 0 + 0 * 1 + range(0, 2) * 1 * 4 + 0 * 1 * 4 * 2 ==> [0, 4]
edp_group[1] = 0 + 1 * 1 + range(0, 2) * 1 * 4 + 0 * 1 * 4 * 2 ==> [1, 5]
edp_group[2] = 0 + 2 * 1 + range(0, 2) * 1 * 4 + 0 * 1 * 4 * 2 ==> [2, 6]
edp_group[3] = 0 + 3 * 1 + range(0, 2) * 1 * 4 + 0 * 1 * 4 * 2 ==> [3, 7] # pp0 遍历完 ep0, ep1, ep2, ep3
edp_group[4] = 0 + 0 * 1 + range(0, 2) * 1 * 4 + 1 * 1 * 4 * 2 ==> [8, 12]
edp_group[5] = 0 + 1 * 1 + range(0, 2) * 1 * 4 + 1 * 1 * 4 * 2 ==> [9, 13]
edp_group[6] = 0 + 2 * 1 + range(0, 2) * 1 * 4 + 1 * 1 * 4 * 2 ==> [10, 14]
edp_group[7] = 0 + 3 * 1 + range(0, 2) * 1 * 4 + 1 * 1 * 4 * 2 ==> [11, 15] # pp1 遍历完 ep0, ep1, ep2, ep3
"""
```
```python
etp1 ==> etp_rank: [0]
ep4 ==> ep_rank: [0, 1, 2, 3]
edp2 ==> pp_rank: [0, 1]

# 这里求 pp，mask pp，先利用式(1)求总共有少 pp 组及其下标
# 然后再利用式(2)中 pp_rank 进行遍历求出结果
pp_group_index = etp_rank + ep_rank * etp_size + edp_rank * etp_size * ep_size
"""
pp_group_index = 0 + 0 * 1 + 0 * 1 * 4 = 0
pp_group_index = 0 + 1 * 1 + 0 * 1 * 4 = 1
pp_group_index = 0 + 2 * 1 + 0 * 1 * 4 = 2
pp_group_index = 0 + 3 * 1 + 0 * 1 * 4 = 3 # edp0 遍历完 ep0, ep1, ep2, ep3
pp_group_index = 0 + 0 * 1 + 1 * 1 * 4 = 4
pp_group_index = 0 + 1 * 1 + 1 * 1 * 4 = 5
pp_group_index = 0 + 2 * 1 + 1 * 1 * 4 = 6
pp_group_index = 0 + 3 * 1 + 1 * 1 * 4 = 7 # edp1 遍历完 ep0, ep1, ep2, ep3
"""
pp_group[pp_group_index] = etp_rank + ep_rank * etp_size + edp_rank * etp_size * ep_size + range(0, 2) * etp_size * ep_size * edp_size
"""
pp_group[0] = 0 + 0 * 1 + 0 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [0, 8]
pp_group[1] = 0 + 1 * 1 + 0 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [1, 9]
pp_group[2] = 0 + 2 * 1 + 0 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [2, 10]
pp_group[3] = 0 + 3 * 1 + 0 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [3, 11] # edp0 遍历完 ep0, ep1, ep2, ep3
pp_group[4] = 0 + 0 * 1 + 1 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [4, 12]
pp_group[5] = 0 + 1 * 1 + 1 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [5, 13]
pp_group[6] = 0 + 2 * 1 + 1 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [6, 14]
pp_group[7] = 0 + 3 * 1 + 1 * 1 * 4 + range(0, 2) * 1 * 4 * 2 ==> [7, 15] # edp1 遍历完 ep0, ep1, ep2, ep3
"""
```

手工做完了所有的推导，可以返回去看看 ranks 划分图，相信会有更多的理解。在掌握了原理后就不用手工推导了，可以直接使用 [rank_generator](./rank_generator_notebook.ipynb) 进行自动化划分。

本质就是上文手工推导时候用到的大量的 **mask**、**prefix profuct** 两个概念，这一切的前提是建立在反复提及的 `order`，从这里出发，配合两个计算方式理论上就能分出想要的各种划分。
我们最后再综合重复一遍 Dense 和 MoE global_rank 计算，采用的 order 依旧是 `tp-cp-ep-dp-pp`：
##### · Dense
$$\text{global\_rank} = \text{tp\_rank} + \text{cp\_rank} \times \text{tp\_size} + \text{dp\_rank} \times \text{tp\_size} \times \text{cp\_size} + \text{pp\_rank} \times \text{tp\_size} \times \text{cp\_size} \times \text{dp\_size}$$

##### · MoE
$$\text{global\_rank} = \text{etp\_rank} + \text{ep\_rank} \times \text{etp\_size} + \text{edp\_rank} \times \text{etp\_size} \times \text{ep\_size} + \text{pp\_rank} \times \text{etp\_size} \times \text{ep\_size} \times \text{edp\_size}$$
