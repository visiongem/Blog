---
title: "区块链账户模型入门笔记"
date: 2026-05-20T10:00:00+08:00
draft: false
categories: ["Block"]
tags: ["区块链", "博客"]
---

> My 学习笔记，水平有限，难免疏漏，欢迎指正。

## 一、UTXO、账户、对象三种模型

区块链记账主要有两种核心模型：**UTXO 模型** 和 **账户模型**。后来一些新链在这两者之上做了融合或创新，业内一般称为**对象模型（Object-Based）**——核心思路是把资产从"账户里的一个数字"变成"链上一个独立的对象 / 资源"，从而支持更高的并发。

| 模型 | 代表链 | 直觉类比 |
|---|---|---|
| **UTXO 模型** | BTC、LTC、BCH、DOGE、Dash、Zcash、Cardano（eUTXO 变种） | **现金 / 硬币**：你有一堆"纸币"，付钱时挑几张给出去，找零回来变成新"纸币" |
| **账户模型** | ETH、全部 EVM 链（BNB Chain、Polygon、Arbitrum、Optimism、Base、Avalanche C-Chain 等）、TRX | **银行账户**：你有一个余额数字，转账就是一边减一边加 |
| **对象模型** | Sui 等（每条链做法不同，下文简单提一下） | 在前两者基础上做并行 / 资源化等改造 |

> 其他公链（Solana、Aptos、TON、NEAR、XRP、Kaspa、CKB 等）也有各自独特的设计，但细节我没逐条核实过，本文就不展开了。

---

## 二、UTXO 模型（"现金"）

### 核心理念

UTXO 全称 **Unspent Transaction Output**，意为"未花费的交易输出"。**没有"账户"或"余额"的概念**——你的资产是一笔笔面额不同的"零钱"。

### 运作方式

钱包里有 5 张纸币：100 + 50 + 20 + 10 + 5 = 185 元。要付 60 给小明：

- **选纸币**：挑 50 + 20 = 70 给出去（必须挑出足够覆盖的组合）
- **找零**：70 - 60 = 10，找零回来变你钱包里新的"纸币"
- **手续费**：从这 70 里扣 1 元给矿工，实际转 60 给小明 + 9 元找零

签名时要**对每张"输入纸币"分别签**——证明你拥有它。

### 一笔 BTC 交易的核心数据

```kotlin
data class BtcTransaction(
    val inputs: List<UTXO>,        // 我拿出去的"纸币"
    val outputs: List<TxOutput>,   // 给别人的 + 找零给自己的
    val lockTime: Long = 0L,       // 时间锁（一般 0）
)

data class UTXO(
    val prevTxId: ByteArray,       // 这张"纸币"原来从哪个交易来
    val prevIndex: Int,            // 那个交易的第几个输出
    val scriptPubKey: ByteArray,   // 锁定脚本（证明你能花它）
    val value: Long,               // 多少 satoshi (1 BTC = 10^8 sat)
    val sequence: Long,            // 用于 RBF（可替换交易）
)

data class TxOutput(
    val value: Long,
    val recipientScript: ByteArray,  // 收款人的锁定脚本
)
```

### 签名时硬件钱包要拿到什么

每个 input 的 `prevTx + prevIndex + scriptPubKey + value + sequence` + 派生路径。

**为什么这么复杂？** 多账户钱包里每个 UTXO 可能由不同派生地址持有——必须分别签。

### BTC 的派生路径

BTC 主流用 **BIP84（Native SegWit / bech32）**：

```
m/84'/0'/0'/0/0
   │   │  │  │ │
   │   │  │  │ └─ 地址索引（0,1,2...）
   │   │  │  └─── 0=外部链（收款）/ 1=内部链（找零）
   │   │  └────── 账户索引（默认 0）
   │   └────────── 币种代码（BTC=0、LTC=2、DOGE=3，SLIP-44 标准）
   └────────────── purpose（44=Legacy、49=Nested SegWit、84=Native SegWit、86=Taproot）
```

旧的 BTC 钱包用 BIP44（`m/44'/0'/...`），生成的是 P2PKH 地址（1 开头）。

### 特点

- **天然支持高并发**：不同 UTXO 互不依赖，可并行验证
- **隐私性较好**：每次找零都会换新地址
- **不易支持复杂智能合约**：没有全局状态

---

## 三、账户模型（"银行账户"）

### 核心理念

类似传统银行卡系统。**每个账户都有一个固定的地址和对应的余额**，每次交易直接在账户余额上作加减法。系统全局维护一个状态数据库，记录每个地址当前的资产状态。

### 运作方式

账户余额 185。付 60：

- 余额 - 60 = 125
- 对方账户 + 60
- 矿工费另扣

简单粗暴。但**没有"挑哪张纸币"的唯一性**，所以需要 **`nonce`（顺序号）** 防重放：

> 你账户的"第 5 笔交易转 60 给小明"和"第 6 笔交易转 60 给小明"是**两笔不同**交易。

每笔成功上链的交易都让账户的 nonce 加 1。下次发交易必须用最新 nonce。

### 一笔 ETH 交易的核心数据

```kotlin
data class EthTransaction(
    val nonce: Long,            // 这是你账户的第 N 笔交易（防重放）
    val gasPrice: BigInteger,   // 你愿意付多少 gas 单价（wei）
    val gasLimit: BigInteger,   // 这笔交易最多用多少 gas
    val to: String,             // 收款地址（合约地址或钱包地址）
    val value: BigInteger,      // 转多少 wei (1 ETH = 10^18 wei)
    val data: ByteArray,        // 调合约时的入参；纯转账为空
    val chainId: Long,          // 主网 1 / Polygon 137 / Arbitrum 42161
    // EIP-1559（伦敦升级后）替代 gasPrice 的两个字段：
    val maxFeePerGas: BigInteger? = null,
    val maxPriorityFeePerGas: BigInteger? = null,
)
```

### 签名输入

Legacy 交易：6 字段（nonce、gasPrice、gasLimit、to、value、data）RLP 编码后 keccak256 hash；EIP-155 之后签名前补 chainId、0、0 共 9 字段；EIP-1559（type=2）是另一套格式（带 type 前缀 + access list + maxFeePerGas / maxPriorityFeePerGas）。

最后把 hash 传给硬件设备签名 + 派生路径。**比 UTXO 简单很多**——没有 inputs 数组。

### EVM 兼容链

**所有 EVM 兼容链共用同一套交易格式 + 同一套地址格式**——你的 ETH 主网钱包地址和你的 Polygon 地址、Arbitrum 地址、BNB Chain 地址**完全相同**（都是同一个公钥派生出来的）。

切链 = 改 chainId，其他代码 100% 复用。

代表链清单（我用过或核实过的）：

- ETH 主网
- BNB Chain (BSC)
- Polygon（POL，前身 MATIC）
- Arbitrum / Optimism / Base（Layer 2）
- Avalanche C-Chain
- Fantom

### ETH 的派生路径

```
m/44'/60'/0'/0/0
       │       │ └─ 地址索引
       │       └─── 永远是 0（外部链；ETH 没有内部找零）
       └─────────── 60 = ETH（其他 EVM 链都用 60）
```

### TRX（Tron）

Tron 也是账户模型，但有几个细节和 EVM 不同：

- **交易序列化用 Protocol Buffers**（不是 RLP）
- **地址格式独立**：base58 编码、T 开头（不是 0x）
- **TVM 兼容 EVM 字节码**：智能合约可以从 Solidity 编译过来
- **派生路径**：`m/44'/195'/0'/0/0`（195 是 TRX 的 SLIP-44 编号）

### 特点

- **极易开发和运行智能合约**，逻辑简单直观
- **容易发生状态争用**，存在并发瓶颈（同账户的两笔交易必须串行）

---

## 四、对象模型

随着区块链技术的发展，一些新兴公链在 UTXO 和账户模型之上做了融合与创新，业内一般称为**对象模型（Object-Based）**——核心思路是把资产从"账户里的一个数字"变成"链上一个独立的对象 / 资源"，从而支持更高的并发。我比较确定的一个例子是 Sui。

### Sui：对象模型（Object-Based）

Sui 把资产视为**独立对象**，每个对象有自己的 ID、版本号、所有者。一笔交易就是把对象的所有权改给别人 + 版本号 +1。

```kotlin
data class ObjectRef(
    val objectId: String,    // 对象 ID（256-bit hex）
    val version: Long,       // 当前版本号（防过期）
    val digest: String,      // 对象内容 hash（防篡改）
)
```

钱包构造交易时**必须知道每个对象的最新版本号 + digest**——传过期的 ObjectRef 链上会拒绝。

这么设计的好处是**并行执行**：不互相依赖的交易可以同时跑（基于对象所有权 DAG 调度）。代价是钱包的状态管理比 ETH 复杂。

> Cardano、Aptos、Solana、TON 等链也各自做了不同程度的创新，但具体设计细节我没逐一核实，就不展开了。

---

## 五、给 Android 钱包开发者的实战建议

### 1. 别造"统一的 Chain 接口"

不同链的参数差异太大，硬抽就是 30 个 nullable 字段：

```kotlin
// ❌ 反例：interface Chain 让 SignParams 变成大杂烩
data class SignParams(
    val rawTx: ByteArray,
    val prevTx: ByteArray? = null,        // BTC 才需要
    val nonce: Long? = null,              // EVM 才需要
    val objectId: String? = null,         // Sui 才需要
    // ... 30 个 nullable 字段
)
```

```kotlin
// ✅ 正例：sealed class + when 分发
sealed class ChainSignRequest {
    data class Btc(val inputs: List<UTXO>, val outputs: List<TxOutput>) : ChainSignRequest()
    data class Eth(val nonce: Long, val gasPrice: BigInteger, val chainId: Long, /*...*/) : ChainSignRequest()
    data class Sui(val gasObject: ObjectRef, /*...*/) : ChainSignRequest()
}

fun sign(req: ChainSignRequest): ByteArray = when (req) {
    is ChainSignRequest.Btc -> signBtc(req)
    is ChainSignRequest.Eth -> signEth(req)
    is ChainSignRequest.Sui -> signSui(req)
}
```

### 2. 用 [Trust Wallet Core](https://github.com/trustwallet/wallet-core)

业内标准 SDK，C++ 实现 + JNI 暴露给 Kotlin / Swift。封装了：

- 多链地址派生（HDWallet）
- 交易构造 + 签名（TransactionCompiler）
- BIP39 助记词标准
- 各种编码（Bech32 / Base58 / Hex）

```kotlin
// 派生地址
val wallet = HDWallet(mnemonic, passphrase)
val ethAddr = wallet.getAddressForCoin(CoinType.ETHEREUM)
val btcAddr = wallet.getAddressForCoin(CoinType.BITCOIN)

// 签名（preImage → 硬件签 → assembleTx）
val txInput = Bitcoin.SigningInput.newBuilder()
    .setHashType(BitcoinSigHashType.ALL.value())
    .setAmount(100_000_000)
    .build()

val preImageHashes = TransactionCompiler.preImageHashes(CoinType.BITCOIN, txInput.toByteArray())
// → 把 preImageHashes 传给硬件钱包签名
// → 拿回签名后用 TransactionCompiler.compileWithSignatures 组装最终 tx
```

**没有 Trust Wallet Core 不建议自己造轮子**——签名算法的细节坑很多。

### 3. App 端的复杂度其实在哪

不在密码学，在 **UI + 数据传输 + 用户态校验**：

- 用户 UI 上看到"转 0.05 ETH 给某地址，gas 0.001 ETH"——这些数字怎么来？查链 RPC + 估算
- nonce 不能传错，但用户不可能自己看 nonce——自动从 RPC 查
- 多链场景下 UI 怎么呈现（同一个地址在多条链上的余额）
- 软件钱包 → 硬件钱包的数据传输（BLE / USB）
- 用户操作中断的恢复（输到一半切走又回来）

### 4. 助记词存储

用 Trust Wallet Core 的 `StoredKey`：

```kotlin
val storedKey = StoredKey.importHDWallet(mnemonic, name, password.toByteArray(), CoinType.ETHEREUM)
storedKey.store(walletFilePath)   // scrypt + AES 加密落盘

// 验证密码
storedKey.decryptMnemonic(userInput.toByteArray()) ?: error("wrong password")
```

底层是 **scrypt + AES**（Web3 Secret Storage 格式），已经是行业最佳实践。不要自己写。

---

## 六、想深挖时的入口

| 主题 | 资源 |
|---|---|
| BTC 协议 | [Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook)|
| ETH 协议 | [Ethereum.org docs](https://ethereum.org/en/developers/docs/) |
| Trust Wallet Core | [GitHub README + 源码](https://github.com/trustwallet/wallet-core) |
| BIP 标准 | [BIP39 助记词](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) / [BIP44 派生路径](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) |
| Sui Move | [Sui docs](https://docs.sui.io/) |
| EIP（ETH 改进提案） | [EIPs.ethereum.org](https://eips.ethereum.org/)（特别是 EIP-1559 / EIP-712） |

---

## 七、小结

- 区块链记账核心就两种模型：**UTXO（现金式）** 和 **账户（银行式）**
- 钱包 App 主要工作不是密码学，而是**为每条链单独造交易构造器 + 数据查询 + 用户体验**
- 不要硬抽统一接口，**sealed class + when 分发**更合适
- 底层签名交给 Trust Wallet Core，自己写容易翻车

文中如有不准确的地方，欢迎在评论区指正，一起学习。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
