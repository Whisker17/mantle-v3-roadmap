# RISE Chain 链级 infra 全面拆解

> 研究轨道：H ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

---

## 0. 本轨道的六条核心结论（先给结论）

| # | 结论 | 证据强度 |
|---|---|---|
| **H-1** | **RISE 是一条标准 OP Stack rollup**，不是新栈。L1 合约是完整的 OptimismPortal / DisputeGameFactory / AnchorStateRegistry 套件，L2 是完整的 `0x42000...` predeploy 套件，batcher 就是 `op-batcher`。它的全部差异化集中在**自研执行层客户端**上。 | [一手] + 【实测】 |
| **H-2** | **RISE 最核心的「链级适配」不是 precompile、不是新 opcode，而是两个参数 + 一个客户端**：区块 gasLimit 拉到 **1.5 Ggas（Mantle 的 25 倍）**、base fee 压到 **0.0004153 gwei 且恒定不动**、执行层换成自研的 `rise-replica`（Reth 血统）。**EVM 语义本身几乎没改。** | 【实测】 |
| **H-3** | **Shreds 是「每笔交易一个 mini-block」**。实测每个 shred 恰好含 1 笔交易、每个 1 秒 L2 区块约含 88 个 shred。它不是 Flashblocks 式的「200ms 定长子区块」，而是**事件驱动的逐笔预确认流**。 | 【实测】 |
| **H-4** | **官方文档明确写明 PEVM 未上线**（"not currently live on RISE & is a future improvement"）。based sequencing 同为路线图。**RISE 今天跑的是单线程顺序执行 + 三线程流水线（CBP）的单一中心化 sequencer。** | [一手] |
| **H-5** | **RISE 的 DA 主路径是 EigenDA，不是 Ethereum**（blob 仅作 fallback）。叠加 OP Succinct Lite 的**单轮 ZK fraud proof** 且 dispute game 为 permissioned。安全模型显著弱于「以太坊安全」的营销表述。 | [一手] |
| **H-6** | **RISE 链上 93% 的交易直达 RISEx 合约，12 秒窗口内只有 37 个不同发送地址。** "整条链在为一个 app 服务"在**流量构成**这个维度上是可量化的事实。 | 【实测】 |

---

## 1. 血缘与定位

### 1.1 它是什么栈

**结论：OP Stack rollup + 自研执行层客户端。**

证据链（全部 [一手]）：

| 证据 | 内容 | 来源 |
|---|---|---|
| L1 系统合约 | `OptimismPortalProxy` `0xad92Fa18EB74E46Db844240623124BF46589db4C`、`DisputeGameFactoryProxy` `0x6A4139810986CF13408330e14C4ac9Daf0511aA3`、`AnchorStateRegistryProxy`、`L1StandardBridgeProxy`、`SystemConfigProxy`、`Guardian`、`Challenger`、`Proposer`、`BatchSubmitter`、`UnsafeBlockSigner` —— **标准 OP Stack 全套** | <https://docs.risechain.com/docs/builders/mainnet-contract-addresses> |
| L2 predeploy | `0x4200...0006` WETH、`...000F` GasPriceOracle、`...0010` L2StandardBridge、`...0011` SequencerFeeVault、`...0015` L1Block、`...0016` L2ToL1MessagePasser、`...0019` BaseFeeVault、`...001A` L1FeeVault、`...0042` GovernanceToken —— **标准 OP predeploy 全套** | 同上 |
| batcher 命名 | 官方 DA 文档直接称 `op-batcher`："Ethereum fallback is triggered whenever the **`op-batcher`** receives an error from EigenDA" | <https://docs.risechain.com/docs/rise-evm/data-availability> |
| CL/EL 分离 | CBP 文档描述 "Consensus (C): Deriving L1 transactions and new block attributes"、"we extend CL to send an extra set of payload attributes"、"replace Engine API with straightforward intra-process communication" —— **用的是 OP 的 Engine API + payload attributes 模型** | <https://docs.risechain.com/docs/rise-evm/cbp> |
| 执行层血统 | CBP 文档："we **override Reth's default implementation** of `eth_getTransactionReceipt`"；pevm 仓库基于 `revm` | 同上 |
| 客户端指纹 | 【实测】`web3_clientVersion` = **`rise-replica/sha-80d9780/linux`**（主网）、`rise-replica/sha-58f9936.arm64/linux`（测试网）。命名为自研 `rise-replica`，非 `op-geth`/`op-reth` 原样 | 见 §8 |

> **判定**：RISE = OP Stack 的 CL/结算/桥/predeploy + **替换掉的 EL**。
> **这条对委托方极其重要**：它证明「做 app-specific 链」**不需要离开 OP Stack**，改造点可以集中在执行层客户端与链参数。→ 交给 N 轨道做改造分级。

### 1.2 网络参数 [一手] + 【实测】

| 项 | 主网 | 测试网 |
|---|---|---|
| 网络名 | RISE Mainnet | RISE Testnet |
| chainId | **4153** | 11155931 |
| RPC | `https://rpc.risechain.com/` | `https://testnet.riselabs.xyz` |
| WSS | `wss://rpc.risechain.com/ws` | `wss://testnet.riselabs.xyz/ws` |
| Explorer | `https://explorer.risechain.com/`（Blockscout） | `https://explorer.testnet.riselabs.xyz` |
| gas 币 | **ETH**（非自有币） | ETH |
| 单笔 gas 上限 | **16M** | 16M |
| nonce 顺序 | Enforced on-chain | 同 |
| Blocks to Finality | **259,200（约 3 天）** | 同 |

来源：<https://docs.risechain.com/docs/builders/mainnet-details>、<https://docs.risechain.com/docs/builders/testnet-details>

**⚠️ 注意 gas 币是 ETH，不是自有代币。** 这与 Mantle（MNT 作 gas）路线相反 —— RISE 放弃了 gas 代币的价值捕获，转而用 **USDR 的储备收益**做价值捕获（见 §7.3）。这是一个值得单独注意的经济学选择。

### 1.3 团队与融资 [一手]

- 核心团队（官方 docs 原文）：**Sam**（CEO/联创，此前创办 RoboVault、Arkiver）、**Hai Nguyen**（CTO/联创，此前创办 MELD，融资 $45M，带 25+ 工程师）、**Sasha**（CGO/联创，前 KyberSwap growth 负责人）；团队其他成员来自 **JUMP、FalconX、Coinbase**，含数名数学奥赛奖牌得主。
  来源：<https://docs.risechain.com/docs/risex/about/overview>
- 投资方（官方 docs 原文）：*"RISE is backed by industry leaders and investors, including **Galaxy Digital, Finality Capital, DACM, Vitalik Buterin** and more."*
  来源：同上、<https://docs.risechain.com/docs/risex>
- ⚠️**存疑**：官方 docs 未给出轮次、金额、日期。本轨道未找到官方一手融资公告。按「金额未知」处理。

### 1.4 官方自我定位的演变 [一手]

- docs 站点首页现标题：**"RISEx — The unified exchange"**；首页大标题：**"The trading chain"**。
- `/docs` 首页原文：*"RISE is an Ethereum Layer 2 blockchain delivering near-instant transactions at unprecedented scale with **sub-3ms latency and 100,000+ TPS capacity**... Built for **CEX-grade performance** and full EVM composability, RISE enables builders, traders, and institutions to **create and connect to global orderbooks** alongside a thriving DeFi ecosystem."*
- 首页宣传语（risechain.com）：*"1ms latency and 50,000+ TPS"*。

> **⚠️ 数字口径不一致（一手源自相矛盾）**：`docs/` 首页写 **sub-3ms / 100,000+ TPS**，`risechain.com` 首页与 RISEx 文档写 **1ms / 50,000+ TPS**，RISEx 架构页写 **"sub-50ms block times"**，而【实测】L2 出块为 **1.000s**。这些数字分别指 shred 延迟、目标吞吐、与不同版本的营销口径，**没有任何一个对应实测的 L2 出块时间**。引用时必须注明口径。

→ docs 站点改版时间点的考证交给 I 轨道（wayback 对比）。

---

## 2. Shreds 的工程级细节

### 2.1 官方定义 [一手]

原文：*"Since merkleization is not required for every L2 block, we can break L2 blocks (12-second long) into **Shreds** (sub-second long), essentially **mini-blocks without a state root**."*
来源：<https://docs.risechain.com/docs/rise-evm/shreds>

> ⚠️**文档与实测冲突**：文档此处假设 L2 block 为 **12 秒**，而【实测】主网 L2 出块为 **1.000 秒**。该文档段落应为早期设计稿，未随主网参数更新。

### 2.2 结构与验证 [一手]

| 要素 | 内容 |
|---|---|
| 标识 | `block_num` + `seq_num` 唯一标识一个 shred；`seq_num` 在新 L2 block 开始时重置 |
| 状态承诺 | 含 **ChangeSetRoot** —— 只承诺本 shred 内的状态变更（ChangeSet），不含全局 state root |
| 为何快 | ChangeSet 条目数远小于 state 总量、稀疏、可放内存，因此 ChangeSetRoot 构建代价低；**完全省掉 state root merkleization** |
| 签名 | **sequencer 签名**。原文：*"Shreds with invalid signatures are discarded."* |
| 传播 | 逐 shred 通过 P2P 广播，不等整个 L2 block |
| 节点处理 | *"As peer nodes receive valid Shreds, they optimistically construct a local block and provide preconfirmations to users."* |
| 免重执行 | sequencer 可在 shred 内附带 ChangeSet，节点验证其与 ChangeSetRoot 一致后**可直接 apply 而不重执行**（原文：*"Nodes trusting the sequencer can apply the ChangeSet immediately to their local state without re-executing transactions."*） |
| merkleization 时点 | *"Merkleization for an L2 block only happens after the last Shred is generated."* |

### 2.3 信任假设（关键）

**Shred 预确认 = 单一中心化 sequencer 的签名承诺，无经济担保、无 L1 强制。**

- 官方对安全性的说法是**相对论式**的：*"This improved latency doesn't sacrifice security, as new L2 blocks can only provide unsafe confirmations anyway."*（意思是：反正 L2 unsafe block 本来也不是最终性，所以提前给你没损失）——这个论证成立，但它同时说明 **shred 的保证强度 = OP Stack 的 unsafe head，即 0**。
- 经济担保（slashing）**属于 based sequencing 路线图 Phase 1**，未上线（见 §6.3）。

### 2.4 与同类机制的逐项辨析

| 维度 | **RISE Shreds** | Base Flashblocks | Unichain rollup-boost | Solana shreds |
|---|---|---|---|---|
| 本质 | **每笔交易一个 mini-block**（实测 1 tx/shred） | 定长 200ms 子区块 | 定长子区块 + TEE builder | 区块的**网络分片**（纠删码传输单元） |
| 是否含状态承诺 | 含 **ChangeSetRoot**（本 shred 状态变更） | 无 state root | 无 state root | 无（纯传输层） |
| 触发方式 | **事件驱动 / interrupt-driven** | 定时器（每 200ms） | 定时器 | 出块过程中持续发出 |
| 谁签名 | sequencer | sequencer / builder | TEE-attested builder | leader |
| 可否回滚 | 可（unsafe head 语义） | 可 | 可 | 可 |
| 是否需改客户端 | **需要**（自研 EL） | 需 sidecar（rollup-boost） | 需 sidecar + TEE | 原生 |
| 用户侧接口 | `eth_subscribe("shreds")`、`eth_sendRawTransactionSync` | `eth_subscribe(newFlashblock…)`、`pending` tag | 同 Flashblocks 家族 | 非用户接口 |
| 目的 | **降低单笔确认延迟** | 降低感知确认延迟 | 降低延迟 + 可验证排序 | **提高区块传播带宽利用率** |

> **重要辨析**：Solana 的 "shred" 与 RISE 的 "Shred" **除了名字之外没有关系** —— 前者是区块传播的纠删码分片，后者是带状态承诺的预确认单元。二手材料经常混淆这两者。
> **与 Flashblocks 的真实差别**：Flashblocks 是「把 2 秒切成 10 份」，RISE 是「每来一笔就发一份」。后者对**逐笔反馈**（下单/撤单）更优，前者对**批量吞吐**更友好。

---

## 3. 执行层

### 3.1 Continuous Block Pipeline（CBP）—— 已上线，是真正的差异化 [一手]

官方对 OP Stack 的诊断（原文，值得完整引用）：
> *"we measured in early 2024 that **OP-Reth only spent 12%-36% of a second executing mempool transactions**... If a user transaction enters the mempool at an unlucky time, it may have to wait **640ms-880ms** before processing, even without congestion."*

RISE 的做法：**三线程流水线**
1. **pre-execution 线程**：持续从 mempool 取交易并**提前执行**，不等 CL 请求新区块
2. **state root 专用线程**：与执行并行算 state root
3. **主驱动线程**：监听 attributes、用预执行结果封装 payload

官方声称的效果：*"The biggest effect is the RISE Block Pipeline executing transactions **close to 100% of the available block time**"*、*"minimizes transaction (receipt) latency and **3x-8x the chain throughput**"*。

来源：<https://docs.risechain.com/docs/rise-evm/cbp>

**CBP 付出的 EVM 语义代价（一手承认，很重要）**：
- **`BLOCKHASH` / EIP-2935 会失效**：预执行时拿不到父块 state root。官方处理：*"We handle this by scheduling an in-time executed block at most every 12 seconds if we notice there's such a transaction."* 并公开劝开发者改用 VRF。
- **L1 origin 切换**：`PREVRANDAO`、deposit 需要正确的 L1 数据，官方靠**扩展 CL 多发一套 payload attributes** 解决。

> **这是本研究能找到的、RISE 唯一确凿的「EVM 语义偏离」**：依赖 `BLOCKHASH` 做链上随机数的合约在 RISE 上行为退化（需要触发特殊的 in-time 区块）。→ 交给 N 轨道评估这类改动对 fault proof 可证明性的影响。

### 3.2 PEVM —— **未上线** [一手，关键否定发现]

官方原文（`/docs/rise-evm/pevm` 页首第一句）：
> *"**Note the PEVM is not currently live on RISE & is a future improvement.** We have already published the PEVM implementation in GitHub for reference. With our current architecture still able to achieve 1 GGas/s with sub 3 millisecond-execution."*

- 实现仓库：<https://github.com/risechain/pevm>（Rust，Block-STM 血统，包装 `revm`）
- 官方自评 legacy pevm 处于 **pre-alpha**
- 早期 benchmark（官方给出）：Uniswap swap 大区块 **22x**；以太坊区块平均约 **2x**；少依赖区块最高约 **4x**；L2 大区块**预期** >5x
- 官方对 Block-STM 前人的批评：Polygon 与 Sei 用 Block-STM "showed limited gains"，原因是缺 EVM 专项优化、Go 语言 GC 停顿、同步开销抵消并行收益
- pevm 的 EVM 专项创新：**lazy updates**（把 beneficiary 账户、原生 ETH/ERC20 转账的余额更新延迟到区块末尾，避免所有交易因 gas 支付而互相依赖）
- **Mempool Preprocessing（路线图）**：*"RISE innovates a new mempool structure... The goal is to pre-order transactions to shared states and maximise parallel execution. This has a relatively similar effect as the **local fee market on Solana**, where congested contracts & states are more expensive regarding gas & latency."*

> **对委托方的关键含义**：
> ① 「并行 EVM」在 RISE 上**是未兑现的路线图**，今天的 1 Ggas/s 是**顺序执行 + 流水线 + RiseDB** 打出来的。
> ② RISE 自己也承认：**CLOB 撮合是写冲突极高的负载，并行 EVM 对它帮助有限** —— pevm 文档写 *"dApps do need to innovate new parallel designs... like Sharded AMM and RISE's upcoming novel CLOB"*，即需要应用侧重新设计才能吃到并行。这与第一阶段「launchpad 写冲突率接近 100%，乐观并行是纯开销」的结论一致。

### 3.3 RiseDB [一手]

- 定位：*"a high-performance verifiable key-value store"*，专为 EVM 链状态设计
- 三个设计点：**统一架构**（把 world state 与 Merkle tree 合并到单一存储层，消除两层架构的上下文切换与数据重复）、**低内存占用**（可跑消费级硬件）、**SSD 优化访问模式**（尽量顺序/追加写，减少随机写）
- 动机：传统 MPT + LSM 后端导致「一次状态读 = 多次磁盘 I/O」，且随状态增长而放大
- ⚠️**存疑**：RiseDB 是否已在主网启用、是否开源、与 Reth 默认 MDBX 的实测对比数据 —— 官方文档均未说明。

来源：<https://docs.risechain.com/docs/rise-evm/rise-db>

### 3.4 非标准 EVM 能力盘点

| 能力 | RISE 状态 | 通用 OP Stack（含 Mantle）是否具备 |
|---|---|---|
| 自定义 opcode | **未发现任何** | — |
| 自定义 precompile（撮合/定点数学） | **未发现任何**。RISEx 的撮合是**纯 Solidity**（因此单笔耗 240 万–420 万 gas，见 §9） | — |
| ERC-4337 EntryPoint v0.6 预部署 | ✅ `0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789` | 多数已有 |
| EIP-7702 | ✅ 支持（shred payload 定义了 `eip7702` 交易类型；RISE Wallet 依赖它） | 视版本 |
| Permit2 / MultiCall3 / CreateX / Safe 全套预装 | ✅ | 部分 |
| EAS + SchemaRegistry predeploy | ✅ `0x4200...0021` / `...0020` | OP Stack 标准可选项 |
| BeaconBlockRoots / HistoryStorage | ✅ 预装 | 视版本 |
| **原生 VRF**（`/docs/builders/vrf`） | ✅ 测试网有 Coordinator，**主网状态未在文档确认**；官方警示"当前实现面向开发测试，主网部署需额外安全措施" | ❌ **无对位物** |
| **链级 Internal Oracle**（面向所有开发者的价格预言机） | ✅ 主网即 `RISExOracle`（见 §7.2） | ❌ 无 |
| **链级原生稳定币 USDR** | ✅（见 §7.3） | ❌ 无 |

> **结论（回答 brief 的核心问题）**：**RISE 没有为 RISEx 做任何 EVM 语义层面的特权改造。** 它做的是①把区块 gas 预算放大 25 倍、②把 base fee 压到近零、③换掉执行层客户端拿到毫秒级逐笔确认、④提供三个"链级公共品"（VRF、Internal Oracle、USDR）。

---

## 4. 交易接口层（对交易类应用最关键）

### 4.1 非标准 / 增强 JSON-RPC 方法表

| 方法 | 状态 | 参数 | 返回 | 说明 | 来源 |
|---|---|---|---|---|---|
| **`eth_sendRawTransactionSync`** | ✅ **主网已上线**（【实测】方法存在） | 已签名交易 hex | **完整 TransactionReceipt** | 基于 **EIP-7966**。单次 RPC 往返即拿回执。官方称配合 shreds 可在 "ping + 1~3ms" 返回 | <https://docs.risechain.com/docs/builders/shreds/api-methods> |
| **`eth_getTransactionReceipt`（重写）** | ✅ 已上线 | 标准 | 标准 | **在 pending block canonicalize 之前就返回回执**（官方原文："override Reth's default implementation... to return receipts before the pending block is canonicalized"） | <https://docs.risechain.com/docs/rise-evm/cbp> |
| **`eth_subscribe("shreds")`** | ✅ **主网已上线**（【实测】可订阅，见 §8.3） | `["shreds"]` 或 `["shreds", true]` | shred 通知对象 | 加 `true` 则填充 `stateChanges` 字段（默认空数组） | <https://docs.risechain.com/docs/builders/shreds/watching-events> |
| `eth_subscribe("logs")`（增强） | ✅ 已上线 | 标准 filter | 标准 log | **log 从 shred 层投递而非 block 层**，官方称 <10ms | 同上 |
| `sendTransactionSync`（viem 侧） | ✅ | 标准 tx 参数 | receipt | 便利方法，内部 3+ 次 RPC（chainId/nonce/gas 估算 + 发送） | 同上 |
| `rise_*` 命名空间 | ❌ **不存在** | — | — | 【实测】`rise_sendRawTransactionSync`、`rise_getShred`、`rise_subscribe`、`rise_shredNumber` 全部返回 `-32601 Method not found` | §8.4 |
| `txpool_*` | ❌ 不存在 | — | — | 【实测】`txpool_status` → `-32601`。**无公开 mempool 可见性** | §8.4 |
| `rollup_getInfo` / `optimism_syncStatus` | ❌ 不存在（EL 端） | — | — | 【实测】`-32601` | §8.4 |

### 4.2 shred 通知的 payload 结构 [一手]

```
{ blockTimestamp, blockNumber, shredIdx, startingLogIndex,
  transactions: [ { transaction: {hash, signer, to, value, type}, receipt: {status, cumulativeGasUsed, logs, type} } ],
  stateChanges: { "0x<addr>": { nonce, balance, storage, newCode } } }
```

交易类型枚举含 `legacy` / `eip1559` / `eip2930` / **`eip7702`** / **`deposit`**（后者带 `sourceHash`、`mint`、`isSystemTransaction` —— 再次印证 OP Stack 血统）。
来源：<https://docs.risechain.com/docs/builders/shreds/watching-events>

### 4.3 官方声称的性能指标 [一手]

| 指标 | 官方数值 | 出处 |
|---|---|---|
| Shred 确认 RTT | **3–5ms (p50)、10ms (p99)** | shreds/api-methods |
| 事件投递 | <10ms（自交易执行起） | 同上 |
| 每 shred 交易数 | **1–100** | 同上（【实测】主网恒为 1） |
| Shreds/秒 | 1000+ | 同上 |
| 总 TPS | 10,000+ | 同上（与首页 50,000/100,000 口径不一致，⚠️见 §1.4） |

### 4.4 revert protection / 失败交易

- **未找到任何 revert protection 机制**（既非链级也非 RPC 级）。
- `eth_sendRawTransactionSync` 的官方说明是：失败交易**立即返回 `status: 0x0`**（"Failed transactions return immediately with status 0x0"）—— 这是**更快地告知失败**，不是**避免失败上链或免收 gas**。
- → 失败交易照常烧 gas。这与第一阶段对 launchpad 的需求（失败交易成本）相关。

---

## 5. 费用市场

| 项 | RISE 实况 |
|---|---|
| gas 币 | **ETH**（`Currency Symbol: ETH`）[一手] |
| EIP-1559 | 有 `baseFeePerGas` 字段，但【实测】**4,004 个连续区块内 base fee 恒为 415,300 wei，零变化** |
| base fee 水平 | **0.0004153 gwei**（约为 Base 的 1/12、Arbitrum 的 1/48） |
| 区块 gasLimit | **1,500,000,000（1.5 Ggas）** |
| 单笔 gas 上限 | **16M** [一手] |
| 实测填充率 | **8.4%–10.4%** |
| 有效容量 | **1,500 Mgas/s**（= 1.5 Ggas ÷ 1.000s） |
| local fee market | ❌ **无**。仅存在于 pevm 路线图的 "Mempool Preprocessing" 构想中 [一手] |
| per-contract / per-account 隔离 | ❌ 无 |
| 公开 mempool | ❌ 【实测】无 `txpool_*`；架构上 RPC 节点经 P2P 转发给单一 sequencer |
| 排序规则 | ⚠️**官方未公开明确规则**。pevm 文档提到「不同于以往 rollup 的 FCFS 或 gas 拍卖」，暗示计划做基于状态冲突的预排序，但**未上线**；今天的实际排序策略**无一手文档** |
| 为 RISEx 的排序 / 费用特权 | ❌ **未找到任何一手证据**表明存在系统交易、oracle 免 gas、cancel 免 gas 或专用 lane 的**链级**机制。RISEx 的「免 gas」是**应用层 relay 代付**实现的（见 §7.1、并交给 I 轨道） |

> **本节最重要的判断**：**RISE 没有做费用市场层面的 app-specific 改造。** 它靠的是「容量极大富余 + base fee 地板极低」把 gas 变成非问题 —— 这是一种**用参数暴力解决拥塞隔离**的思路，而非机制设计。
> **代价**：一旦真实需求填满 1.5 Ggas，全局 base fee 会同时惩罚所有应用，且没有任何隔离机制。这与第一阶段记录的 Robinhood Chain 事故（meme 让 base fee 11 天涨 82 倍）是同一个结构性风险。

---

## 6. DA / 结算 / 安全

### 6.1 DA：EigenDA 为主，Ethereum blob 仅 fallback [一手]

- 主路径：**EigenDA**。官方理由：Ethereum blob 目标吞吐仅 64KB/s（6 blob/block），Fusaka + PeerDAS 后理论 8x 到 512KB/s，*"even with the upgrade, this throughput is insufficient for a high-load rollup"*；EigenDA 主网 **100 MB/s**、平均确认 5s。
- Fallback 触发条件（一手，很具体）：`op-batcher` 收到 EigenDA 错误、收不到 ACK、**或 batcher 余额不足支付 EigenDA 费用**时，回退到 Ethereum blob。
- 来源：<https://docs.risechain.com/docs/rise-evm/data-availability>

> **安全含义**：DA 在 Ethereum 之外 ⇒ 按 L2Beat 分类学属于 **Optimium**（validium 的 optimistic 对偶），而非 rollup。「secured by Ethereum」的营销表述**只覆盖结算，不覆盖 DA**。⚠️ 本轨道未直接读取 L2Beat 页面确认其当前分类与 stage 评级，标为**待核实**。

### 6.2 证明系统：OP Succinct Lite 单轮 ZK fraud proof [一手]

- 实现：**OP Succinct Lite**（<https://github.com/succinctlabs/op-succinct>）。官方理由：单轮解决争议、无需 challenger 与 proposer 交互式二分，且**支持 EigenDA / Celestia 这类 AltDA**（"perfect match for RISE"）。
- L1 合约印证：`PermissionedDisputeGame (OP Succinct)` `0xA9aF0d2efC17ce247c6821D94910cF8f27cC2587`、`Access Manager (OP Succinct)` `0xF90a72FC295DBEf2fD27629Fda4B98Fd3E842d17`。
  → **合约名里的 `Permissioned` 说明 dispute game 目前是许可制**，非无许可挑战。
- 争议状态机（官方给出）：`Unchallenged → Challenged`（带 bond）→ `prove()` 提交 ZK 证明 → `DefenderWins` / `ChallengerWins`（bond 转移）。
- 演进路线（官方明确分期）：**Phase 1 ZK Fraud Proofs（当前）→ Phase 2 Proactive Proving → 最终 ZK Rollup**。
- 提现最终性：**259,200 个区块 ≈ 3 天** [一手]。
- 来源：<https://docs.risechain.com/docs/rise-evm/zk-fraud-proofs>

### 6.3 sequencer 中心化与 based sequencing 路线图 [一手]

RISEx FAQ 原文承认：
> *"L2s inherit Ethereum's security for settlement and data availability. **Execution is managed by centralised sequencers**, however, we are building toward progressive decentralization."*
> 来源：<https://docs.risechain.com/docs/risex/faq>

based sequencing **三阶段路线图（全部未上线）**：

| 阶段 | 名称 | 内容 |
|---|---|---|
| Phase 1 | **The Taste** | 现有 sequencer 增加 gateway 功能；实现 L1 slashing 与 delegation；**发行 L1-secured execution preconf 作为 shreds** |
| Phase 2 | **The Aligning** | 多个**白名单** gateway **round-robin 轮转**，固定 sequencing window（按 L1 区块计）；选择机制由 L1 Registry 合约确定性推导；RISE 自营 default gateway 作 fallback；靠 shred streaming + mempool 同步实现无缝交接 |
| Phase 3 | **The Basedening** | 取消白名单，任何 Ethereum validator 质押抵押品即可成为 gateway |

来源：<https://docs.risechain.com/docs/rise-evm/based-sequencing>

> **关键**：**「shred 带 L1 经济担保」属于 Phase 1，尚未实现。** 今天 shred 预确认的唯一保证是单一 sequencer 的签名。

### 6.4 网络参与者分层 [一手]

RISE 明确设计了**不需要全网重执行**的节点分层（这本身是为高吞吐做的架构取舍）：

| 节点类型 | 同步方式 | 安全性 | 硬件 |
|---|---|---|---|
| **Sequencers** | 自执行 | 高 | 32GB RAM |
| **Replicas** | **应用 state-diff（不重执行）** | **低，依赖 fraud proof** | 8–16GB RAM |
| **Fullnodes** | 带辅助数据重执行 | 高 | 16–32GB RAM |
| **Challengers** | 同 Fullnodes | 高 | 16–32GB RAM |
| **Provers** | 信任 sequencer | N/A | FPGA/GPU，按需启动 |

来源：<https://docs.risechain.com/docs/rise-evm/network-participants>

> 【实测】印证：主网 `web3_clientVersion` 返回 **`rise-replica`** —— 公共 RPC 提供的正是「Replica」类节点，即**以 state-diff 同步、安全性依赖 fraud proof 的节点**。

---

## 7. 三个「链级公共品」——RISE 真正的 app-specific 做法

这一节是本轨道的原创归纳：**RISE 的 app-specific 化不体现在 EVM 特权，而体现在「链把本该由应用自建的东西，做成了链级公共品」。**

### 7.1 账户抽象与 gas 赞助（RISE Wallet Stack）[一手]

| 组件 | 内容 |
|---|---|
| 智能账户 | **Porto Smart Accounts**（<https://porto.sh/>）+ **EIP-7702** 原生 AA，无需部署合约钱包 |
| 密钥类型 | **P256 / secp256k1 / WebAuthn P256**（passkey，FaceID/TouchID/Windows Hello，存于设备安全区） |
| Relay 基础设施 | **Gas Sponsorship**（自动代付）、**Transaction Batching**（原子批量）、**Circuit Breakers**（防滥用/超支） |
| Session Keys | 时限 + 额度受限的临时密钥；有有效 session key 时**完全跳过钱包弹窗**，自动签名 |
| 赞助规则 | Relay 按「用户 tier 与日限额 / 合约白名单 / 允许的函数」校验 |
| 赞助范围（原文） | *"New users receive daily gas budget"*、*"Core protocol interactions (swaps, mints) are sponsored"*、**"RISEx trading is fully sponsored"** |
| 恢复 | guardian recovery、时间锁、多签 |

来源：<https://docs.risechain.com/docs/rise-wallet/how-it-works>

> **判定（重要）**：**gas 赞助是「应用/基础设施层 relay」，不是协议级 paymaster。** 链本身没有 protocol-level fee abstraction。但因为 RISE 官方同时是钱包方 + 链方 + 交易所方，用户感知上等同于「链原生免 gas」。
> **这是委托方最该学的一条**：*「链原生体验」可以由「链方自营钱包 + relay 代付」合成，无需改协议。*

### 7.2 链级 Internal Oracle —— 就是 RISEx 的 oracle [一手，最强证据]

`/docs/builders/mainnet-internal-oracles` 面向**所有开发者**提供的"Internal Oracles"，其地址表只有两个条目：

| Contract | Address |
|---|---|
| **RISExOracle** | `0x8fC4D0Cf74cdF595254cB763d4C05D38Df0e9503` |
| **RISExStork** | `0x76A559C716c5B93b9d743e08D9E9f23f96a4f975` |

调用接口即 `getIndexPrice(uint16 marketId)` / `getMarkPrice(uint16 marketId)`，marketId 直接沿用 **RISEx 的市场编号**。

来源：<https://docs.risechain.com/docs/builders/mainnet-internal-oracles>

> **这是「整条链为一个 app 服务」最锋利的一手证据**：链提供给全体开发者的公共预言机原语，**就是交易所自己的预言机**，连 marketId 命名空间都共用。
> （测试网版本则是独立的 ETH/USDC/USDT/BTC 四个 `latestAnswer()` 预言机 —— 说明主网是**刻意收敛到 RISEx oracle** 的。）

### 7.3 USDR —— 链级原生稳定币 + 链级价值捕获模型 [一手，对本研究极关键]

| 维度 | 内容（官方原文要点） |
|---|---|
| 定位 | *"USDR is the **native stablecoin of the RISE ecosystem**... the **base unit of account** across RISE, used as the main quote asset on AMMs (such as **Icarus**), money markets (such as **Spine**), and vaults (such as the **Autoyield vault on RISEx**)"* |
| backing | **非算法稳定币**。是 **M0 协议 `$M` 代币的全额抵押包装**，1:1 由短久期美国国债（T-bills）支撑 |
| 铸造路径 1 | **M0 Limit Order Protocol**（canonical）：RISE 与 M0 有直接铸造关系，可用 Ethereum 上的 USDC 1:1 铸 USDR；M0 solver 网络负责 `$M` → USDR 的转换，用户无需接触 `$M`。Arbitrum / Base 支持 "coming soon" |
| 铸造路径 2 | **Bungee** 跨链桥聚合器 |
| **收入模型** | *"**All yield generated by the reserves backing USDR is retained by RISE as protocol revenue.** These revenues are redirected to **deepen USDR liquidity** across the ecosystem: funding **LP incentives**, tightening spreads on USDR trading pairs, and strengthening the overall stability and utility of USDR as the primary settlement currency."* |

来源：<https://docs.risechain.com/docs/rise-evm/usdr>

> **这是 RISE 商业模式的真正核心，比 Shreds 重要得多：**
> ① gas 币是 ETH ⇒ 放弃 gas 费价值捕获；
> ② 改为捕获**稳定币储备的国债收益**（float 收益）；
> ③ 该收益**不分红，而是回投为 LP 激励与点差补贴** ⇒ 形成「TVL ↑ → float 收益 ↑ → 流动性激励 ↑ → 交易体验 ↑ → TVL ↑」的自循环。
> **这条对「资产发行原生链」的设计有直接可迁移性** → 交给最终设计方案（report/08）。

---

## 8. 实测（可复现）

**所有实测于 2026-09-07 完成，endpoint `https://rpc.risechain.com`（主网，chainId 4153）。请求体与方法见 §11 附录。**

### 8.1 链参数横向实测对照

| 指标 | **RISE** | **Mantle** | Base | Arbitrum One |
|---|---|---|---|---|
| chainId | **4153** | 5000 | 8453 | 42161 |
| `web3_clientVersion` | **`rise-replica/sha-80d9780/linux`** | `Geth/v1.17.3-stable-11fa8109/linux-amd64/go1.*` | `reth/v2.3.0-9384bc5/x86_64-unknown-linux-gnu` | `nitro/v3.11.4-rc.3-7d5ac27/linux-arm64/go1.25` |
| 出块间隔（均值） | **1.000 s** | **2.000 s** | 2.000 s | 0.286 s |
| 区块 gasLimit | **1,500 M** | **60 M** | 400 M | （名义 1.126e9，不可比） |
| **有效容量 Mgas/s** | **1,500** | **30** | 200 | — |
| 区块填充率 | **8.4%–10.4%** | **0.224%** | 8.9% | ~0 |
| baseFeePerGas | **0.0004153 gwei** | **50 gwei**（钉在下限） | 0.005 gwei | 0.02 gwei |
| tx / block | **83–100** | **2.0** | 165 | 2.8 |
| `eth_sendRawTransactionSync` | **✅ 存在** | **❌ 不存在** | ✅ 存在 | ✅ 存在 |

> **RISE 的有效执行容量是 Mantle 的 50 倍**（1,500 vs 30 Mgas/s）。
> **同时复核了第一阶段的 Mantle 实测结论：出块 2.000s、填充率 ~0.2%、base fee 钉在 50 gwei 下限 —— 全部仍然成立。**

### 8.2 base fee 恒定性实测

对 head 起回溯 **4,004 个连续区块**（≈ 1 小时 07 分）采 `eth_feeHistory`：
- **distinct baseFeePerGas 值的数量 = 1**
- min = max = **415,300 wei**

⇒ **EIP-1559 在 RISE 上从未触发**，base fee 是一个硬地板。结构上与 Mantle 完全同类（Mantle 是 50 gwei 地板），只是地板高度不同。

### 8.3 Shred WebSocket 实测（本研究独有数据）

endpoint `wss://rpc.risechain.com/ws`，`eth_subscribe(["shreds"])`，采样 15 秒：

| 指标 | 实测值 |
|---|---|
| 订阅是否可用 | ✅ 主网可用 |
| 收到 shred 数 | **1,140 个 / 15 秒** |
| **每 shred 交易数** | **均值 1.00，最大 1** ⇒ **恒为 1 笔** |
| 每个 L2 区块的 shred 数 | 13 / 20 / 27 / 28 / 29 / 31 / 58 / 59 / 79 / 79 / 133 / 165 / 419；**均值 87.7** |
| shred 到达间隔 | p50 ≈ 0.0ms、p90 ≈ 0.1ms、max 1,077ms（区块边界） |

**解读**：
- 「1 tx = 1 shred」证实 shred 是**逐笔预确认单元**，官方文档"1–100 transactions per shred"的上限在主网实际负载下未被使用。
- 到达间隔 p50≈0 说明 shred 经 WebSocket **成批投递**，因此本方法**无法验证官方 3–5ms 的端到端 RTT 声明**（需要自己发交易计时，本研究未持有 RISE 主网资金，⚠️未验证）。
- 单个区块最多观测到 **419 个 shred** ⇒ 419 tx/s 的瞬时峰值。

### 8.4 非标准方法探测

| 方法 | 返回 |
|---|---|
| `eth_sendRawTransactionSync` | `-32602 failed to decode signed transaction` ⇒ **方法存在**（仅参数无效） |
| `rise_sendRawTransactionSync` / `rise_getShred` / `rise_subscribe` / `rise_shredNumber` / `eth_getShred` | 全部 `-32601 Method not found` ⇒ **无 `rise_` 命名空间** |
| `txpool_status` | `-32601` ⇒ 无 mempool 可见性 |
| `rollup_getInfo` / `optimism_syncStatus` / `rpc_modules` | `-32601` |
| `eth_maxPriorityFeePerGas` | `0xa010` = 40,976 wei |
| `eth_feeHistory` | ✅ 正常 |

### 8.5 单笔交易的 gas 成本实测

| RISEx 调用（selector） | 平均 gasUsed | 按 baseFee 0.0004153 gwei 计的费用 | 按 $4,000/ETH 折算 |
|---|---|---|---|
| `0x0dd51555`（router，最高频） | **2,437,216** | 1.012e-6 ETH | **≈ $0.0041** |
| `0xf22aeaaa`（router，最重） | **4,199,968** | 1.744e-6 ETH | **≈ $0.0070** |
| `0x4ade0d04`（router） | 650,632 | — | — |
| `0x66b4ec48`（router） | 327,107 | — | — |
| `0x5b4b886a`（FundingRate） | 46,286 | 1.922e-8 ETH | ≈ $0.00008 |

> ⚠️ ETH 价格为**假设值 $4,000**（本研究未取实时价），费用折算仅供数量级参考。
>
> **这组数字是理解 RISE 全部设计的钥匙**：纯 Solidity 的链上 CLOB 撮合单笔要烧 **240 万–420 万 gas**。在 Mantle 的 60M 区块里，**一个区块最多只能装 14–24 笔这样的交易**；在 RISE 的 1.5 Ggas 区块里能装 **357–615 笔**。
> ⇒ **「全链上订单簿」不是靠精妙算法实现的，是靠把区块 gas 预算放大 25 倍 + 把 gas 单价压到近零实现的。**

### 8.6 链上流量构成实测（"整条链为一个 app 服务"的量化）

**采样 A**：连续 12 个区块，共 **1,139 笔**交易

| 目标合约 | 身份 | 交易数 | 占比 |
|---|---|---|---|
| `0xaadde0ce…4a7e` | **RISExUniversalRouter** | 940 | **82.5%** |
| `0x069edf2c…207a` | **RISEx FundingRate** | 120 | **10.5%** |
| `0xacc0a0cf…fd62` | **Stork**（oracle 推送） | 24 | 2.1% |
| `0xca11bde0…ca11` | MultiCall3 | 24 | 2.1% |
| `0x42000000…0015` | L1Block（OP 系统交易） | 12 | 1.1% |
| `0xf665aba9…6734` | **RISEx OperatorHub** | 9 | 0.8% |
| `0xe03c1d50…f2cd` | **RISEx OrdersManager** | 3 | 0.3% |
| 其他 | — | 7 | 0.6% |

- **RISEx 直属合约合计占比 ≈ 94.1%**（router + FundingRate + OperatorHub + OrdersManager）
- 加上为 RISEx 服务的 Stork oracle 推送 ⇒ **≈ 96.2%**
- **12 秒窗口内不同发送地址仅 37 个**；最活跃单地址 12 秒内发了 **268 笔**

**采样 B**：另取 6 个区块 417 笔交易，selector 分布与采样 A 一致（router 的 5 个 selector 占 321 笔 = 77%）。

> **结论 H-6 成立**：无论按交易数、按 gas 消耗还是按合约集中度，**RISE 主网今天几乎就是 RISEx 的专用执行环境**。
> **但要注意反面**：37 个发送地址意味着这些流量绝大部分来自**少数做市 bot / operator**，不是零售用户的广泛活动。「链很忙」与「链有很多用户」是两件事。⚠️ 本轨道未能取到 RISE 的日活地址总数与 DefiLlama TVL（见存疑清单）。

---

## 9. 生态构成

### 9.1 官方文档提及的 RISE 生态应用 [一手]

| 应用 | 类型 | 出处 |
|---|---|---|
| **RISEx** | perps CLOB（旗舰） | 全站 |
| **Icarus** | AMM | USDR 文档："main quote asset on AMMs (such as Icarus)" |
| **Spine** | money market | 同上 |
| **AutoYield vault** | RISEx 内的收益 vault | 同上（RISEx roadmap 标为 **V1.1，未上线**） |
| **Morpho Lite** | 借贷（被 RIP-2 PPM 依赖） | portfolio-margin 文档（标 **coming soon**） |
| **RISE Wallet** | 官方钱包（Porto/7702） | rise-wallet 文档 |
| **LayerZero V1/V2** | 跨链消息 | `/docs/builders/mainnet-layerzero` |

⇒ 官方文档中**具名的第三方 DeFi 应用只有 Icarus、Spine、Morpho 三个**，且都是围绕 USDR / RISEx 保证金构建的配套件，而非独立赛道应用。

### 9.2 ⚠️ 未取到的生态数据

本轨道**未能**取得以下数据（见存疑清单）：RISE 的 DefiLlama chain TVL、日活地址、协议数量、DEX 交易量。前一轮失败的 agent 曾报告 explorer 显示「2.23B 总交易 vs 仅 18,316 个地址」，但**本轨道未独立复现该数字，不予采信，仅记录为线索**。

---

## 10. 显式回答：RISE 有哪些能力是通用 OP Stack 链（含 Mantle）今天做不到或没做的？

| # | 能力 | RISE | Mantle | 差距性质 | 一手依据 |
|---|---|---|---|---|---|
| 1 | **1.5 Ggas 区块 gas 预算**（50x Mantle 的有效容量） | ✅ | ❌ 60M | **纯参数配置** —— Mantle 改 chain config 即可，但需要执行层撑得住 | 【实测】 |
| 2 | **base fee 地板压到 0.0004 gwei** | ✅ | ❌ 50 gwei 地板 | **纯参数配置** | 【实测】 |
| 3 | **1 秒出块** | ✅ | ❌ 2 秒 | 纯参数配置 | 【实测】 |
| 4 | **逐笔预确认流（Shreds）+ WS 订阅** | ✅ | ❌ 完全缺席 | **需改执行层客户端 + P2P**；Base 走 rollup-boost sidecar 路线可平替 | [一手] + 【实测】 |
| 5 | **`eth_sendRawTransactionSync`（EIP-7966）单往返拿回执** | ✅ | ❌ | **需改 RPC 层**（相对独立，成本最低的一项） | [一手] + 【实测】 |
| 6 | **pending 阶段即返回 receipt**（重写 `eth_getTransactionReceipt`） | ✅ | ❌ | 需改执行层 | [一手] |
| 7 | **Continuous Block Pipeline**（执行占满区块时间） | ✅ | ❌ | **需改执行层客户端**（最硬的一项） | [一手] |
| 8 | **RiseDB 自研状态存储** | ✅（⚠️主网启用状态未证实） | ❌ | 需改执行层 | [一手] |
| 9 | **链级 Internal Oracle 公共原语** | ✅ | ❌ | **纯合约层 + 文档层**，零协议改动 | [一手] |
| 10 | **链级原生 VRF（毫秒级）** | ✅ 测试网（主网状态未证实） | ❌ | 合约层 + 链方运营 backend | [一手] |
| 11 | **链级原生稳定币 + float 收益回投流动性** | ✅ USDR/M0 | ❌ | **商业与法务安排，非技术** | [一手] |
| 12 | **官方钱包 + relay 全额 gas 赞助 + session key** | ✅ | ❌ | **应用层**，零协议改动 | [一手] |
| 13 | 并行 EVM | ❌ **未上线** | ❌ | 双方都没有 | [一手] |
| 14 | local fee market / 拥塞隔离 | ❌ 仅路线图 | ❌ | 双方都没有 | [一手] |
| 15 | 自定义 precompile / opcode | ❌ **一个都没有** | ❌ | 双方都没有 | 全站检索 |
| 16 | fault proof 去中心化程度 | ⚠️ Permissioned dispute game + 非以太坊 DA | Mantle 情况交给 N 轨道 | — | [一手] |

### 归纳：RISE 领先的 12 项里，按改造侵入性分类

| 侵入性 | 项目 | 数量 |
|---|---|---|
| **纯参数 / 配置**（改 chain config） | #1 #2 #3 | **3** |
| **纯合约 / 文档 / 运营层**（零协议改动） | #9 #10 #11 #12 | **4** |
| **需改 RPC 层** | #5 | 1 |
| **需改执行层客户端** | #4 #6 #7 #8 | 4 |
| **需改 EVM 语义** | —— | **0** |

> **本轨道最重要的工程结论**：
> **RISE 相对 Mantle 的 12 项优势里，有 7 项（58%）不需要碰执行层客户端**，其中 4 项连协议都不用改。
> **需要真正硬工程（自研执行层）的只有 4 项，且都集中在"降低单笔确认延迟"这一个目标上。**
> ⇒ 「做 app-specific 链」的门槛远低于直觉，但**最贵的那部分（毫秒级逐笔确认）恰好是 perps 最需要、发行类应用最不需要的**。→ 这条直接影响 report/08 的方案取舍。

---

## 存疑清单

1. ⚠️ **融资细节缺失**：官方仅列投资方名单（Galaxy Digital、Finality Capital、DACM、Vitalik Buterin），**未找到任何轮次 / 金额 / 日期的一手公告**。按「金额未知」处理。
2. ⚠️ **主网上线日期未确认**：本轨道未找到 RISE 主网与 RISEx 主网上线的官方日期公告。已知 RISEx 处于 "gated mainnet"（见 I 轨道）。
3. ⚠️ **L2Beat 分类与 stage 评级未独立核实**：依据 DA 架构推断应为 Optimium，但未直接读取 L2Beat 页面确认，也未取得 stage 评级与风险项列表。
4. ⚠️ **force inclusion 窗口未找到**：OptimismPortal 存在，但官方文档未给出强制包含窗口时长。
5. ⚠️ **升级钥匙构成未核实**：`ProxyAdminOwner` `0x9196464e…002c`、`Guardian`/`SystemConfigOwner` 同为 `0x03B85FAa…FB46`（**同一地址兼任 Guardian 与 SystemConfigOwner**，值得注意），但其多签阈值与签名人构成未查。
6. ⚠️ **RiseDB 是否已在主网启用、是否开源、有无实测对比数据** —— 官方文档只讲设计，无部署状态。
7. ⚠️ **官方 1ms / 3ms 延迟声明未被本研究独立验证**：需持有 RISE 主网资金发交易计时才能验证 `eth_sendRawTransactionSync` 的真实 RTT。本研究仅验证了「方法存在」与「shred 订阅可用」。
8. ⚠️ **今天的实际排序规则无一手文档**：FCFS？priority fee？还是 sequencer 自定策略？pevm 文档暗示计划做状态感知预排序但未上线。**这是本轨道最大的信息缺口**，且直接关系到「是否给 RISEx 排序特权」这一核心问题。
9. ⚠️ **VRF 主网状态未确认**：文档只给测试网 Coordinator 地址（`0x9d57aB45…8909`），且明确警示当前实现不适合主网生产。
10. ⚠️ **生态数据未取到**：DefiLlama chain TVL、日活地址、协议数、DEX 量均未取得。前一轮 agent 报告的 "2.23B 总交易 vs 18,316 地址" **未经本轨道复现，不予采信**。
11. ⚠️ **官方数字口径互相矛盾**（见 §1.4）：sub-3ms/100k TPS vs 1ms/50k TPS vs sub-50ms block times vs 实测 1.000s 出块。官方未在任何一处统一口径。
12. ⚠️ **`0xacc0a0cf…fd62`（Stork）与 `0x76A559C7…f975`（RISExStork）的关系未厘清**：前者是链上高频推送目标（实测 2.1% 交易、平均 134 万 gas），后者是文档列出的 "Stork price feed adapter"。→ 交给 I 轨道。
13. ⚠️ **ETH 价格假设**：§8.5 的美元折算基于假设的 $4,000/ETH，未取实时价。
14. ⚠️ **`rise-replica` 是否开源**：GitHub `risechain` org 下已确认存在 `pevm` 仓库，但**本轨道未核实执行层客户端本体是否开源**。

---

## 关键来源清单

**RISE 官方文档（一手）**
- 架构总览：<https://docs.risechain.com/docs/rise-evm>
- Shreds（规范）：<https://docs.risechain.com/docs/rise-evm/shreds>
- Shreds（开发者）：<https://docs.risechain.com/docs/builders/shreds> ｜ API：<https://docs.risechain.com/docs/builders/shreds/api-methods> ｜ 订阅：<https://docs.risechain.com/docs/builders/shreds/watching-events>
- Continuous Block Pipeline：<https://docs.risechain.com/docs/rise-evm/cbp>
- Parallel EVM（**未上线**）：<https://docs.risechain.com/docs/rise-evm/pevm>
- RiseDB：<https://docs.risechain.com/docs/rise-evm/rise-db>
- 交易生命周期：<https://docs.risechain.com/docs/rise-evm/tx-lifecycle>
- Data Availability（EigenDA）：<https://docs.risechain.com/docs/rise-evm/data-availability>
- ZK Fraud Proofs（OP Succinct Lite）：<https://docs.risechain.com/docs/rise-evm/zk-fraud-proofs>
- Based Sequencing（路线图）：<https://docs.risechain.com/docs/rise-evm/based-sequencing>
- Network Participants：<https://docs.risechain.com/docs/rise-evm/network-participants>
- **USDR**：<https://docs.risechain.com/docs/rise-evm/usdr>
- 主网参数：<https://docs.risechain.com/docs/builders/mainnet-details> ｜ 测试网：<https://docs.risechain.com/docs/builders/testnet-details>
- 主网合约全表：<https://docs.risechain.com/docs/builders/mainnet-contract-addresses>
- **Internal Oracles（主网）**：<https://docs.risechain.com/docs/builders/mainnet-internal-oracles>
- Fast VRF：<https://docs.risechain.com/docs/builders/vrf>
- RISE Wallet 架构与 gas 赞助：<https://docs.risechain.com/docs/rise-wallet/how-it-works>
- RISEx FAQ：<https://docs.risechain.com/docs/risex/faq>
- 全量文档（机器可读，21,195 行，本研究已本地留存）：<https://docs.risechain.com/llms-full.txt>

**外部一手**
- EIP-7966（`eth_sendRawTransactionSync`）：<https://eips.ethereum.org/EIPS/eip-7966>
- EIP-7702：<https://eips.ethereum.org/EIPS/eip-7702>
- OP Succinct（Succinct Labs）：<https://github.com/succinctlabs/op-succinct>
- RISE pevm 实现：<https://github.com/risechain/pevm>
- Block-STM 论文：<https://arxiv.org/abs/2203.06871>
- Porto：<https://porto.sh/>
- M0 文档：<https://docs.m0.org/>

---

## 可复现方法附录

所有实测于 **2026-09-07** 完成。

### A. 通用 JSON-RPC 调用形式

```bash
curl -s -X POST https://rpc.risechain.com \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"<METHOD>","params":[<PARAMS>]}'
```

### B. 本研究实际使用的请求

| 目的 | method | params |
|---|---|---|
| chainId | `eth_chainId` | `[]` |
| 客户端指纹 | `web3_clientVersion` | `[]` |
| 链头 | `eth_blockNumber` | `[]` |
| 出块间隔 / gasLimit / 填充率 | `eth_getBlockByNumber` | `["0x<blockNumHex>", false]`，对 head−39…head 连续 40 块采样，取 `timestamp` 差分、`gasLimit`、`gasUsed` |
| 流量构成 | `eth_getBlockByNumber` | `["0x<blockNumHex>", true]`，统计 `to` 分布与 `from` 去重 |
| 实际 gasUsed | `eth_getBlockReceipts` | `["0x<blockNumHex>"]`，按 `transactionHash` 关联 selector |
| base fee 恒定性 | `eth_feeHistory` | `["0x3e8", "0x<blockNumHex>", []]`，以 1000 为步长回溯 4 次覆盖 4,004 块 |
| 合约字节码 | `eth_getCode` | `["0x<addr>", "latest"]`（RISExUniversalRouter 与 FundingRate 均为 **1,074 字节** ⇒ EIP-1967 代理） |
| 方法存在性探测 | 见 §8.4 各方法 | 以无效参数调用，用 `-32601`（不存在）vs `-32602`（参数错）区分 |

### C. Shred WebSocket 实测

```javascript
const ws = new WebSocket("wss://rpc.risechain.com/ws");
ws.onopen = () => ws.send(JSON.stringify({
  jsonrpc: "2.0", id: 1, method: "eth_subscribe", params: ["shreds"]
}));
// 统计：到达时刻 performance.now() 差分；每条通知的
// params.result.blockNumber / shredIdx / transactions.length
```
采样 15 秒 ⇒ 1,140 个 shred，`transactions.length` 恒为 1。

### D. 横向对照使用的 endpoint

| 链 | endpoint |
|---|---|
| RISE | `https://rpc.risechain.com` |
| Mantle | `https://rpc.mantle.xyz` |
| Base | `https://mainnet.base.org` |
| Arbitrum One | `https://arb1.arbitrum.io/rpc` |

对每条链取 15–20 个连续区块，方法同 §B。
