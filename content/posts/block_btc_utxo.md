---
title: "比特币钱包用久了会\"变穷\"？聊聊 UTXO 碎片化"
date: 2026-05-21T10:00:00+08:00
draft: false
categories: ["Block"]
tags: ["区块链", "博客"]
---

# 比特币钱包用久了会"变穷"？聊聊 UTXO 碎片化

> 上一篇笔记聊了区块链的三种账户模型。有读者（其实是我自己）好奇追问：UTXO 模型如果交易次数很多，是不是会堆一大堆零钱？
>
> 答案是：**会，而且这是 UTXO 链一个很现实的痛点**。这篇就来科普一下这件事。同样是学习笔记，水平有限，难免疏漏，欢迎指正。

---

## 一、先快速回忆：UTXO 是什么

UTXO 全称 **Unspent Transaction Output**（未花费的交易输出）。它和"账户模型"最大的区别是：

> **你的资产不是一个余额数字，而是一笔笔面额不同的"零钱"。**

每次交易必须：

- 挑出一堆"输入零钱"（UTXO）
- 拼成一笔新的交易
- 多出来的部分**找零回自己**，变成新的 UTXO

代表链：BTC、LTC、BCH、DOGE、Dash、Zcash、Cardano。

---

## 二、零钱是怎么"越攒越多"的

每一次**收款**和每一次**找零**，都会在你的钱包里**产生一个新的 UTXO**。

举个例子：

| 操作 | 钱包里的 UTXO |
|---|---|
| 初始：朋友转你 1 BTC | `[1.0]` |
| 你转 0.3 给别人，找零 0.7 | `[0.7]` |
| 老板发工资 0.5 BTC | `[0.7, 0.5]` |
| 转 0.2 出去，找零 0.3（用了 0.5）| `[0.7, 0.3]` |
| 朋友又转你 0.001 BTC（小额）| `[0.7, 0.3, 0.001]` |
| ... 半年后 | `[0.7, 0.3, 0.001, 0.05, 0.002, 0.0008, ...]` |

一个**活跃地址**用几个月下来，钱包里堆**几十甚至几百个 UTXO** 是非常正常的。交易所的热钱包甚至能堆到几万个。

---

## 三、为什么这是个问题

### 1. 手续费会暴涨 🔥

BTC 手续费**按交易体积算**——交易越大，手续费越多。每多一个 input 都会让交易体积膨胀：

| 输入类型 | 单个 input 占用 |
|---|---|
| Legacy P2PKH（`1...` 地址） | ~148 vbytes |
| Nested SegWit（`3...` 地址）| ~91 vbytes |
| Native SegWit（`bc1q...` 地址）| ~68 vbytes |
| Taproot（`bc1p...` 地址）| ~57.5 vbytes |

> vbytes = virtual bytes，SegWit 之后引入的计费单位，witness 数据有 4 倍折扣

**极端场景**：

假设你想把 0.1 BTC 全部转出去，但你的余额是 200 个 0.0005 BTC 的小 UTXO（手续费从这 0.1 里扣）。

- 这笔交易要塞 **200 个 input**
- 体积 ≈ 200 × 68 = **13,600 vbytes**
- 在 50 sat/vB 的拥堵期，手续费 ≈ **680,000 satoshi ≈ 0.0068 BTC**
- **占这笔钱的 6.8%**——比 Visa 信用卡手续费还贵

### 2. 粉尘 UTXO 会变成"死钱"🪦

当某个 UTXO 的价值 **< 把它当 input 花掉所需的手续费**，花掉它就是**净亏损**——经济上不值得动，相当于一笔废掉的"死钱"。

注意：**这不是协议禁止你花它**。花一个小额 UTXO 完全合法、交易也能正常广播，只是你为它付的 input 手续费超过了它本身的价值，纯属做亏本买卖。

Bitcoin Core 里还有个相关但**不同**的概念——**dust threshold（粉尘阈值）**：

| 输出类型 | 粉尘阈值（按 dustRelayFee = 3 sat/vB） |
|---|---|
| P2PKH | 546 satoshi |
| P2WPKH | 294 satoshi |

这个阈值管的是**创建输出**这一端：一笔交易如果**创建**了低于阈值的输出，会被判为非标准交易，**默认节点不会中继**。换句话说——**你没法给别人转一笔低于 546 sat 的钱**，但已经在你钱包里的小额 UTXO，你想花还是花得掉的。

链上现在堆积着**大量这种小额、不值得花的 UTXO**。它们仍然占用全节点的 UTXO 集（chainstate），是全节点资源压力的一个来源。

### 3. 隐私会泄露 🕵️

当你一笔交易花掉**多个 UTXO** 时，链上分析公司可以推断："这些 UTXO 属于同一个所有者"——这叫 **common-input ownership heuristic（共同输入启发法）**，是链上分析最基础的技术之一。

你以为换了 10 个地址收钱很安全？只要某天你把它们一起花出去，这 10 个地址就被关联起来了。

### 4. 粉尘攻击（Dust Attack）🎯

攻击者**故意往你地址发一笔极小金额**（通常是几百 satoshi，刚好高于粉尘阈值、能正常广播），等着你某天花钱时不小心把它和其他 UTXO 一起花出去——攻击者就能追踪到你其他的关联地址。

**防御办法**：钱包识别出可疑的小额 UTXO，**永远不花它**（标记为 "do not spend"）。Sparrow、Wasabi 等隐私钱包都有这功能。

---

## 四、钱包怎么应对

### 1. 币选择算法（Coin Selection）

钱包构造交易时**怎么挑 UTXO 组合**是个学问。主流算法：

| 算法 | 思路 | 优缺点 |
|---|---|---|
| **Branch and Bound (BnB)** | 找能**精确凑出**转账金额+手续费的组合 | ✅ 不产生找零，省 fee + 保隐私 / ❌ 找不到精确解就回退到其他算法 |
| **Knapsack** | Bitcoin Core 的传统背包算法 | 找接近目标金额的 UTXO 组合 |
| **Largest-First** | 优先花大额 UTXO | ✅ 减少 input 数量 / ❌ 留下一堆零钱 |
| **Smallest-First** | 优先花小额 UTXO | ✅ 顺便清零钱 / ❌ 手续费高 |
| **SRD（Single Random Draw）**| 随机抽 | 隐私好一些 |
| **FIFO** | 先收到的先花 | 税务友好（部分国家） |

Bitcoin Core 自 **0.17 版本**（2018 年）起引入 **BnB** 算法，这是 Mark Erhardt（Murch）在 2016 年的研究成果。

需要说明的是 fallback 逻辑随版本演进过：

- **0.17 时代**：BnB 找不到精确解 → 回退到 **Knapsack**
- **新版（22.0 / 23.0 之后）**：不再是简单"回退"，而是**同时跑 BnB / SRD / Knapsack**，再按 **waste（浪费）指标**择优选一个

### 2. UTXO 合并（Consolidation）

**主动**发起一笔"自己转给自己"的交易，把一堆小 UTXO 合成一个大 UTXO：

```kotlin
data class ConsolidationTx(
    // 100 个小零钱全部作为 input
    val inputs: List<UTXO>,

    // 只输出 1 个大 UTXO 给自己
    val outputs: List<TxOutput> = listOf(
        TxOutput(
            value = totalInputValue - fee,
            recipientScript = myAddress
        )
    )
)
```

**业内最佳实践**：

- 监控 mempool 拥堵情况
- **在 fee 极低的深夜或周末**（比如 1–3 sat/vB）批量发合并交易
- 交易所（Coinbase / Binance / Kraken）的冷钱包都有自动化合并脚本

### 3. 给用户的"整理零钱"按钮

很多用户钱包（Electrum、Sparrow、BlueWallet 等）UI 里有"Consolidate UTXOs"功能，本质就是上面这种合并交易。普通用户不需要懂原理，点一下按钮就行。

### 4. Coin Control（高级用户）

允许用户**手动指定**这笔交易用哪些 UTXO，而不是让钱包自动选。隐私敏感的用户会用这个功能确保不同来源的 UTXO 不会混在一起花。

---

## 五、账户模型有这个问题吗

**没有**——这是 ETH 那一派账户模型的一个天然优势。

| 维度 | UTXO 模型 | 账户模型 |
|---|---|---|
| 资产结构 | 一堆独立的 UTXO | 一个余额数字 |
| 碎片化问题 | ✅ 存在 | ❌ 不存在 |
| 转账手续费 | 受 input 数量影响（不可预测） | 固定 21000 gas（简单转账） |
| 隐私 | 默认换地址 + 找零，但有共同输入推断风险 | 一个地址一直用，反而更容易被追踪 |
| 并行验证 | ✅ 不同 UTXO 互不依赖 | ❌ 同账户必须串行 |

**鱼与熊掌**：UTXO 换来了**并行性 + 默认换地址的隐私**，代价是**碎片化烦恼**。账户模型简单粗暴没碎片，代价是**全局状态 + 并发瓶颈**。

没有完美方案，每条链都在它的取舍里活着。

---

## 六、给钱包开发者的实战 Tips

1. **算预估手续费时一定要算上 input 数量**——不能只看"转账金额"。`fee ≈ (inputs_count × vbytes_per_input + outputs_count × 31 + 11) × feerate`
2. **UI 上提醒用户"你有 200 个小额 UTXO，建议合并"**——好的钱包会主动提示
3. **粉尘 UTXO 标记**——发现可疑的极小额入账（几百 satoshi），给用户红色警告
4. **Coin Control 给高级用户暴露**——隐私敏感场景下用户会感谢你
5. **合并交易选低 fee 时间窗**——可以接 [mempool.space API](https://mempool.space/docs/api) 实时查 fee

---

## 七、想深挖时的入口

| 主题 | 资源 |
|---|---|
| Branch and Bound 研究 | Mark Erhardt（Murch）的币选研究，主页 [murch.one](https://murch.one) 可查到论文与文章 |
| Bitcoin Core 币选源码 | [src/wallet/coinselection.cpp](https://github.com/bitcoin/bitcoin/blob/master/src/wallet/coinselection.cpp) |
| Dust 阈值定义 | [Bitcoin Core policy/policy.cpp 的 GetDustThreshold](https://github.com/bitcoin/bitcoin/blob/master/src/policy/policy.cpp) |
| mempool fee 实时查询 | [mempool.space](https://mempool.space/) |
| 共同输入启发法 | [Bitcoin Wiki: Privacy](https://en.bitcoin.it/wiki/Privacy#Common-input-ownership_heuristic) |

---

## 八、小结

- UTXO 链用久了会**自然碎片化**，是设计的必然结果
- 它带来 4 个现实问题：**手续费暴涨、粉尘死钱、隐私泄露、粉尘攻击**
- 钱包通过**币选择算法 + 主动合并 + 用户教育**来缓解
- 账户模型没这个问题，但有它自己的烦恼
- **作为钱包开发者**，预估手续费时一定要算上 input 数量，并给用户暴露"整理零钱"的入口

文中如有不准确的地方，欢迎在评论区指正，一起学习。


![](https://files.mdnice.com/user/3602/641e52f9-6b55-496c-85b9-ca645698d943.jpg)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
