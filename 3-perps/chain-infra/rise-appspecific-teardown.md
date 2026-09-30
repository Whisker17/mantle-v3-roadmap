# 第七部分：Rise 为 RiseX 做了什么 —— app-specific 链的工程真相

> **研究日期**：2026-09-07
> **目的**：拆解 RISE Chain 为其旗舰 perps 应用 RISEx 做的链 infra 原生适配，对照 OP Stack / Mantle 这类通用链模式，判断「把整条链做成 app-specific」这一模式的真实成本与真实杠杆。
> **底稿**：`research/H-rise-chain-infra.md`（链级）、`research/I-risex-app-coupling.md`（应用耦合）、`research/K-sequencing-latency-infra.md`（零件目录）、`research/N-opstack-mantle-surface.md`（OP Stack/Mantle 可改造面）
> **数据约定**：`[一手]` 官方文档/源码 ｜ `[二手]` 媒体转述 ｜ `【实测】` 本研究直接调用 RPC/合约/API ｜ `⚠️存疑`

---

## 一页纸结论

### 最重要的一句话

> **RISE 不是"为 RiseX 改造了 EVM"。它是一条标准 OP Stack rollup，把「区块 gas 预算放大 25 倍 + base fee 压到近零 + 换掉执行层客户端」这三件事做完，然后把链的公共原语、代币激励、钱包、稳定币全部对齐到一个应用上。**
>
> **技术门槛远低于直觉。组织与经济门槛远高于直觉。**

### 七条核心结论

**① RISE 是标准 OP Stack rollup，不是新栈。** [一手]+【实测】
L1 侧是完整的 `OptimismPortal` / `DisputeGameFactory` / `AnchorStateRegistry` / `L1StandardBridge` / `SystemConfig` 套件；L2 侧是完整的 `0x4200...` predeploy 套件（含 `GasPriceOracle`、`SequencerFeeVault`、`L1Block`、`GovernanceToken`）；官方 DA 文档直接称 `op-batcher`；CBP 文档承认在用 OP 的 Engine API + payload attributes 模型。**差异化 100% 集中在被替换掉的执行层客户端上**（【实测】`web3_clientVersion` = `rise-replica/sha-80d9780/linux`）。

**② 链级"适配"的实质是三个参数 + 一个客户端，EVM 语义几乎没改。** 【实测】

| 项 | RISE | Mantle | 倍数 |
|---|---|---|---|
| 出块间隔 | **1.000 s** | 2.000 s | 2× |
| 区块 gasLimit | **1,500 M（1.5 Ggas）** | 60 M | **25×** |
| **有效容量** | **1,500 Mgas/s** | 30 Mgas/s | **50×** |
| baseFeePerGas | **0.0004153 gwei** | 50 gwei（钉在下限） | — |
| 填充率 | 8.4–10.4% | **0.224%** | 40× |
| tx/block | 83–100 | **2.0** | 45× |
| `eth_sendRawTransactionSync` | **有** | **无** | — |

**全站检索未发现任何自定义 opcode、任何自定义 precompile、任何 EVM 语义特权。**

**③ 「全链上订单簿」不是算法奇迹，是 gas 预算的暴力解。** 【实测】
纯 Solidity 的链上 CLOB 撮合，**单笔烧 240 万–420 万 gas**。

- 在 Mantle 的 60M 区块里：**一个区块只装得下 14–24 笔**
- 在 RISE 的 1.5 Ggas 区块里：**装得下 357–615 笔**

**⇒ RISE 必须把区块放大 25 倍，否则 RISEx 在物理上跑不起来。这是整条链最重要的一个"适配"，而它是一行配置。**

**④ 流量层面，"整条链在为一个 app 服务"是可量化的事实。** 【实测】
连续 12 个区块 1,139 笔交易中：

| 目标 | 占比 |
|---|---|
| RISExUniversalRouter | **82.5%** |
| RISEx FundingRate | **10.5%** |
| RISEx OperatorHub + OrdersManager | 1.1% |
| **RISEx 直属合约合计** | **≈ 94.1%** |
| + 为 RISEx 服务的 Stork oracle 推送 | **≈ 96.2%** |

**但同一窗口内只有 37 个不同发送地址**，最活跃单地址 12 秒发 268 笔。**「链很忙」≠「链有很多用户」。**

**⑤ 官方是书面、反复、明确地自我定位为"为 RISEx 服务的链"。** [一手]
> *"**RISE and RISEx are one team, one vision, one token.**"*
> *"**100% of RISE points are allocated to RISEx users.** ... Why? **RISE is an exchange chain and RISEx is the core product.**"*

这不是外界解读，是官方 FAQ 与积分文档原话。

**⑥ 但"链级技术特权"这一层，取证结果是：没有。**
15 项候选链级适配的取证结果：**8 项有证据、2 项有反证、5 项无证据**。而那 8 项里：

- **真正的链级改造：只有 1 项**（Shreds —— 且它是任何应用都能用的通用能力）
- **链级公共品（合约/运营层，零协议改动）：3 项**（USDR、VRF、Internal Oracle）
- **应用/钱包层实现：4 项**（gas 代付、session key、cancel 优先、子账户）

**⑦ RISEx 的"链原生体验"几乎全部由中心化 operator 合成。** [一手]
用户**不发交易** —— 用户签一个 EIP-712 消息，POST 给 REST API（前置 Cloudflare，限流 200 请求/10 秒），由 RISEx 自营 operator 打包上链**并代付 gas**。

「no off-chain components」的正确表述是：**链上撮合引擎 + 中心化订单网关**。

---

## 1. RISE 的血缘：一条被换了发动机的 OP Stack 车

### 1.1 证据链

| 证据 | 内容 |
|---|---|
| **L1 系统合约** | `OptimismPortalProxy` `0xad92Fa18…db4C`、`DisputeGameFactoryProxy` `0x6A413981…1aA3`、`AnchorStateRegistryProxy`、`Guardian`、`Challenger`、`Proposer`、`BatchSubmitter`、`UnsafeBlockSigner` —— **标准 OP Stack 全套** |
| **L2 predeploy** | `0x42…0006` WETH、`0x42…000F` GasPriceOracle、`0x42…0011` SequencerFeeVault、`0x42…0015` L1Block、`0x42…0019` BaseFeeVault、`0x42…0042` GovernanceToken —— **标准 OP predeploy 全套** |
| **batcher** | 官方 DA 文档原文：*"Ethereum fallback is triggered whenever the **`op-batcher`** receives an error from EigenDA"* |
| **CL/EL 模型** | CBP 文档：*"we extend CL to send an extra set of payload attributes"*、*"replace Engine API with straightforward intra-process communication"* |
| **执行层血统** | *"we **override Reth's default implementation** of `eth_getTransactionReceipt`"*；pevm 基于 `revm` |
| **客户端指纹** | 【实测】`rise-replica/sha-80d9780/linux` |
| **交易类型枚举** | shred payload 定义含 `deposit` 类型（带 `sourceHash`/`mint`/`isSystemTransaction`）—— OP Stack 存款交易 |

### 1.2 这条结论为什么最重要

**委托方原始设想是「把整条链做成 app-specific 的形式」。RISE 给出的答案是：不需要换栈，不需要新链，在 OP Stack 内部就能做到。**

**⇒ 「app-specific chain」的正确技术翻译是「改造主链」，而不是「另开一条链」。**（→ 见 §6 三条路线对比）

### 1.3 安全模型：比营销表述弱

| 维度 | RISE 实况 |
|---|---|
| **DA** | **EigenDA 为主**（100 MB/s），Ethereum blob 仅 fallback（触发条件包括**batcher 余额不足付 EigenDA 费用**）⇒ 按 L2Beat 分类学属 **Optimium**，非 rollup。「secured by Ethereum」只覆盖结算，不覆盖 DA |
| **证明系统** | **OP Succinct Lite** 单轮 ZK fraud proof；L1 合约名为 `PermissionedDisputeGame` ⇒ **挑战是许可制** |
| **最终性** | **259,200 区块 ≈ 3 天** [一手] |
| **sequencer** | 官方 FAQ 承认：*"Execution is managed by **centralised sequencers**"* |
| **公共 RPC 节点类型** | 【实测】`rise-replica` ⇒ 是 **Replica** 节点（应用 state-diff 不重执行，官方自评"安全性低，依赖 fraud proof"） |
| **未上线的去中心化承诺** | based sequencing 三阶段（The Taste / The Aligning / The Basedening）**全部为路线图** |

---

## 2. 真正上线的三件事，与没上线的两件事

### 2.1 已上线：Continuous Block Pipeline（CBP）

**官方对 OP Stack 的诊断**（一手，值得完整引用）：
> *"we measured in early 2024 that **OP-Reth only spent 12%-36% of a second executing mempool transactions**... If a user transaction enters the mempool at an unlucky time, it may have to wait **640ms-880ms** before processing, even without congestion."*

**做法**：三线程流水线 —— ①pre-execution 线程持续提前执行 mempool 交易 ②state root 专用线程并行算根 ③主驱动线程用预执行结果封 payload。

**官方声称效果**：*"executing transactions **close to 100% of the available block time**"*、**"3x-8x the chain throughput"**。

**付出的 EVM 语义代价（官方承认）**：
- **`BLOCKHASH` / EIP-2935 会失效**（预执行拿不到父块 state root）。官方处理：检测到此类交易时"**最多每 12 秒**调度一个 in-time 执行的区块"，并公开劝开发者改用 VRF。
- ⇒ **这是本研究能找到的、RISE 唯一确凿的 EVM 语义偏离。**

### 2.2 已上线：Shreds —— 逐笔预确认

| 维度 | 实况 |
|---|---|
| 定义 [一手] | *"mini-blocks without a state root"*，含 **ChangeSetRoot**（只承诺本 shred 的状态变更） |
| **【实测】每 shred 交易数** | **均值 1.00，最大 1 ⇒ 恒为 1 笔** |
| **【实测】每 1 秒区块的 shred 数** | 均值 **87.7**，单块最高观测 **419** |
| 签名 | **sequencer 单签**（*"Shreds with invalid signatures are discarded"*） |
| 信任假设 | **等价于 OP Stack unsafe head，保证强度为 0**。L1 经济担保属 based sequencing Phase 1，**未上线** |
| 用户接口 | 【实测】`wss://rpc.risechain.com/ws` + `eth_subscribe(["shreds"])` 主网可用 |

**与同名/近似机制的辨析**：

| | RISE Shreds | Base Flashblocks | Unichain rollup-boost | Solana shreds |
|---|---|---|---|---|
| 本质 | **每笔一个 mini-block** | 定长 200ms 子区块 | 定长子区块 + TEE builder | **区块的纠删码传输分片** |
| 触发 | **事件驱动** | 定时器 | 定时器 | 出块过程 |
| 可验证性 | sequencer 承诺 | sequencer 承诺 | **TEE attestation** | N/A |
| 目的 | 降低单笔确认延迟 | 降低感知延迟 | 延迟 + 可验证排序 | **提高传播带宽利用率** |

> **Solana 的 "shred" 与 RISE 的 "Shred" 除名字外没有关系。** 二手材料常混淆。
> **与 Flashblocks 的真差别**：Flashblocks 把 2 秒切 10 份；RISE 每来一笔发一份。**逐笔反馈型 UX（下单/撤单/抢购）前者不如后者；批量吞吐后者不如前者。**

### 2.3 已上线：`eth_sendRawTransactionSync`（EIP-7966）

单次 RPC 往返直接返回完整 receipt。官方称配合 shreds 可在 **"ping + 1~3ms"** 返回。【实测】主网方法存在。

**这是整个 RISE 技术栈里性价比最高的一项** —— 纯 RPC 层改动，不碰状态转换，不影响证明系统。**Mantle 【实测】不存在此方法。**

### 2.4 **未上线：PEVM（并行 EVM）** — 关键否定发现

官方文档页首第一句 [一手]：
> *"**Note the PEVM is not currently live on RISE & is a future improvement.** We have already published the PEVM implementation in GitHub for reference. With our current architecture still able to achieve 1 GGas/s with sub 3 millisecond-execution."*

- legacy pevm 官方自评为 **pre-alpha**
- 官方自己指出：**CLOB 是写冲突极高的负载，并行 EVM 对它帮助有限** —— *"dApps do need to innovate new parallel designs... like Sharded AMM and RISE's **upcoming** novel CLOB"*
- **⇒ 与第一阶段结论一致**：「launchpad 的写冲突率接近 100%，乐观并行的回滚重执行是纯开销」

### 2.5 未上线：local fee market / 拥塞隔离

仅存在于 pevm 路线图的 "Mempool Preprocessing" 构想里（官方称其效果 *"relatively similar to the local fee market on Solana"*）。

**今天 RISE 的费用市场实况**【实测】：**4,004 个连续区块内 base fee 恒为 415,300 wei，distinct 值 = 1。EIP-1559 从未触发。**

> **结构上与 Mantle 完全同类**（Mantle 是 50 gwei 地板），只是地板高度不同。
> **代价**：一旦真实需求填满 1.5 Ggas，全局 base fee 同时惩罚所有应用，且无任何隔离机制 —— 这正是第一阶段记录的 Robinhood Chain 事故（meme 让 base fee 11 天涨 82 倍）的同一结构性风险。

---

## 3. 逐项适配取证：15 项候选，只有 1 项是真链级改造

| # | 候选链级适配 | 判定 | 实质 |
|---|---|---|---|
| 1 | 下单/撮合吃 Shred 级预确认 | ✅ 有证据 | **链级，但是通用能力**（任何 RISE 应用都能用） |
| 2 | gasless 下单/撤单 | ✅ 有证据 | **应用层**：RISE Wallet Relay 代付（*"RISEx trading is fully sponsored"*），**非协议级 paymaster** |
| 3 | session key / 一次签名多次下单 | ✅ 有证据 | **应用层 + 钱包层**：`registerSigner`（7 天有效、权限枚举 All/Perps/Spot/MoveFund）+ Porto/EIP-7702 |
| 4 | sequencer 级排序特权 | ⭕ **无证据** | 全站无 priority lane / reserved blockspace 表述；【实测】无 `rise_*` RPC、无 txpool 可见性 |
| 5 | oracle 先于撮合的确定性顺序 | ⭕ **无证据** | oracle 是普通 EOA 发的普通交易；顺序保证只能来自 operator 自己按序提交 |
| 6 | cancel 优先于 fill | ✅ 有证据 | **应用层「人为延迟」**（见 §3.1），链层无此语义 |
| 7 | 撮合专用 precompile | ❌ **有反证** | 【实测】单笔 240–420 万 gas ⇒ 纯 Solidity。若有 precompile，gas 会低 1–2 个数量级 |
| 8 | 区块空间预留 / 专用 lane | ⭕ **无证据** | 且 2026-08-23 拥塞时的应对是**在应用层加延迟**，而非动用预留空间 |
| 9 | 原生 sub-account / 权限委托 | ✅ 有证据 | **合约层，且 RIP-1 状态 = Draft，未上线** |
| 10 | revert protection / 失败单不收费 | ⭕ 无证据（链级） | `sendRawTransactionSync` 只是**更快返回** `status:0x0`；但因 operator 代付，用户侧等效免费 |
| 11 | 链级风控（pre-trade risk check 进状态转换） | ❌ **有反证** | 风控在 `CollateralManager`/clearinghouse **合约**内 |
| 12 | native USD / 统一保证金资产 | ✅ 有证据 | **链级**：USDR（M0/T-bill 背书）。但 RISEx 当前抵押资产是 USDC，二者尚未统一 |
| 13 | 快速提现通道 | ⭕ 无证据 | 提现最终性 **≈3 天**（标准 OP Stack 挑战期） |
| 14 | 链级原生 VRF | ✅ 有证据 | **链级公共品**；测试网 Coordinator，主网状态未确认，官方警示当前实现不适合生产 |
| 15 | 链级 Internal Oracle | ✅ 有证据 | **链级公共品 = RISEx 自己的 oracle**（见 §4.2） |

### 3.1 Latency Bumps —— 本研究最有价值的发现之一

RISEx 用**人为注入延迟**在应用层实现了做市商公平性 [一手]：

| 账户档 | Taker 延迟 | Maker 延迟 | **Cancel 延迟** |
|---|---|---|---|
| **API Trader** | **100 ms**（2026-08-23 起临时改为 **200 ms**） | 10 ms | **0 ms** |
| **Click Trader**（默认） | **300 ms** | 200 ms | **0 ms** |

官方动机原文：
> *"When prices move, takers with faster infra race to pick off stale quotes before makers can cancel. Makers eat the loss, widen spreads, books thin out. **By delaying takers, makers are able to pull their orders before being adversely selected.**"*

**三层含义**：
1. **这就是「cancel 优先于 fill」的排序语义** —— 但实现方式是在 operator 里给不同动作插入不同长度的 sleep，**不是链级排序规则**。
2. **它是纯中心化裁量权** —— 谁算 API Trader、谁的流量"toxic"、延迟设多少，全由 RISEx 单方面决定并可随时撤销（官方明写"fast-lane access may be revoked"）。
3. **2026-08-23 的拥塞公告是一记警钟**：一条 1.5 Ggas/s、填充率仅 10% 的链，其旗舰应用仍会因 "congestion" 把 taker 延迟翻倍。**⇒ 瓶颈不在区块空间，在 operator 的单点处理能力。**

### 3.2 TX 额度门禁 —— 与第一阶段结论的独立收敛

因为 operator 替用户付了 gas，RISEx 失去了 gas 这个天然反垃圾闸门，于是重新发明了一个 [一手]：

| 机制 | 参数 |
|---|---|
| 新地址初始额度 | **免费 10,000 TX** |
| 额度赚取 | **每 $5 终身成交量 → +1 TX，永不重置** |
| 下单 / TP-SL 单 | 各扣 1 TX |
| **撤单** | **永久免费**（*"Cancels are intentionally free"*） |
| 额度耗尽 | 软惩罚 1 请求/10 秒，不封号 |
| 边缘限流 | Cloudflare **200 请求 / 10 秒** |

> **第一阶段独立得出的结论**是：Mantle 2 秒出块使「衰减税」退化为阶梯函数，**抗狙击必须用额度门禁**（Flaunch Game Mode 式 spend-gate）。
> **RISEx 从完全不同的问题（gas 代付后的反垃圾）出发，收敛到同一个机制。**
> **⇒ 两条独立路径的交叉验证，大幅提高了「额度门禁」作为 issuance-native 原语的可信度。**（→ 进入第八部分设计方案）

---

## 4. 真正的杠杆：三个「链级公共品」与一个价值捕获模型

**这是本研究的核心归纳：RISE 的 app-specific 化不体现在 EVM 特权，而体现在「链把本该由应用自建的东西，做成了链级公共品」。**

### 4.1 官方钱包 + Relay：用产品合成"链原生免 gas"

| 组件 | 内容 |
|---|---|
| 智能账户 | **Porto Smart Accounts** + **EIP-7702** 原生 AA（无需部署合约钱包） |
| 密钥 | P256 / secp256k1 / **WebAuthn passkey**（FaceID/TouchID，设备安全区） |
| Relay | **Gas Sponsorship** + **原子批量** + **Circuit Breakers** |
| 赞助规则 | 按用户 tier / 日限额 / 合约白名单 / 函数白名单校验 |
| 赞助范围 [一手] | *"New users receive daily gas budget"*、*"Core protocol interactions (swaps, mints) are sponsored"*、**"RISEx trading is fully sponsored"** |
| Session Key | 有效 session key 时**完全跳过钱包弹窗** |

> **判定**：**这是应用/基础设施层，不是协议级 paymaster。** 但因为 RISE 官方同时是链方 + 钱包方 + 交易所方，**用户感知上等同于"链原生免 gas"**。
> **⇒ 委托方最该学的一条：「链原生体验」可以由「链方自营钱包 + relay 代付」合成，零协议改动。**

### 4.2 链级 Internal Oracle —— 最锋利的一手证据

`/docs/builders/mainnet-internal-oracles` 是面向**所有开发者**的"内部预言机"页。它的地址表只有两行：

| Contract | Address |
|---|---|
| **RISExOracle** | `0x8fC4D0Cf74cdF595254cB763d4C05D38Df0e9503` |
| **RISExStork** | `0x76A559C716c5B93b9d743e08D9E9f23f96a4f975` |

接口是 `getIndexPrice(uint16 marketId)` / `getMarkPrice(uint16 marketId)`，**marketId 直接沿用 RISEx 的市场编号**。

> **链提供给全体开发者的公共预言机原语，就是交易所自己的预言机，连命名空间都共用。**
> 对照：**测试网**版本是独立的 ETH/USDC/USDT/BTC 四个 `latestAnswer()` 预言机 ⇒ **主网是刻意收敛到 RISEx oracle 的**。

**oracle 路径的反差**：

| 链 | oracle 实现 | gas 成本 |
|---|---|---|
| **Hyperliquid** | validator 集体喂价，**共识层职责** | 无 |
| **dYdX v4** | `x/prices` **链的原生模块** | 无 |
| **RISE** | **普通合约 + 普通 EOA 每 500ms 推送** | 【实测】**每次推送烧 134 万 gas，占全链交易 2.1%** |

> **RISE 在 oracle 这一维度上恰恰是三者中最"不原生"的。** 它靠的是 gas 便宜到可以每 500ms 烧 134 万 gas。

### 4.3 USDR：app-specific 链真正的商业模式

| 维度 | 内容 [一手] |
|---|---|
| 定位 | *"the **native stablecoin** of the RISE ecosystem... the **base unit of account** across RISE"*，用于 AMM（Icarus）、货币市场（Spine）、RISEx 的 AutoYield vault |
| backing | **非算法**。是 **M0 协议 `$M`** 的全额抵押包装，1:1 由短久期美国国债支撑 |
| 铸造 | canonical 路径：RISE 与 M0 有直接铸造关系，Ethereum USDC 1:1 铸 USDR，M0 solver 网络负责转换；另有 Bungee 桥 |
| **收入模型** | *"**All yield generated by the reserves backing USDR is retained by RISE as protocol revenue.** These revenues are redirected to **deepen USDR liquidity**: funding **LP incentives**, tightening spreads on USDR trading pairs..."* |

**⇒ 这是 RISE 商业模式的真正核心，比 Shreds 重要得多：**

1. **gas 币是 ETH，不是自有币** ⇒ **主动放弃 gas 费价值捕获**
2. 改为捕获**稳定币储备的国债 float 收益**
3. 该收益**不分红，而是回投为 LP 激励与点差补贴**
4. ⇒ 形成 **「TVL ↑ → float 收益 ↑ → 流动性激励 ↑ → 交易体验 ↑ → TVL ↑」** 的自循环

**对照 Mantle 的经济现实**（底稿 N，一手）：

| 项 | 数字 |
|---|---|
| Mantle sequencer 收入 FY24-25（MIP-33 官方披露） | **$1.76M**（占总收入 $48.86M 的 **3.6%**） |
| Mantle 2026 sequencer 收入 run-rate | **$10–20 万/年**（两年缩水约 **98%**） |
| Mantle Treasury | **$2.69B** |

> **两条链得出了同一个结论：排序费不是生意，float 与 treasury 才是。**
> **⇒ 任何"靠分 sequencer 手续费回本"的链改造提案，分母以十万美元/年计，直接 ROI 可忽略。**

### 4.4 代币激励 100% 对齐

> *"**100% of RISE points are allocated to RISEx users**, including traders, LPs, and builder-code integrators. Why? **RISE is an exchange chain and RISEx is the core product.**"* [一手]

Season 1 "Ignite" 自 **2026-07-20** 起，每周 **200k 点**，预计不晚于 **2027 Q2** 结束。计分维度包含 **"总成本（手续费 + 滑点 + 负 markout）"** —— 即**激励驱动交易量**的典型设计。

---

## 5. 单位经济学：为什么"全链上撮合 + 代付 gas"能跑通

| 项 | 数值 |
|---|---|
| 单笔下单类交易 gasUsed【实测】 | **2,437,216**（最高频 selector） |
| baseFee【实测】 | 0.0004153 gwei |
| **单笔链上 gas 成本** | ≈ **$0.0041**（假设 ETH $4,000）⚠️ 价格为假设值 |
| RISEx Tier-1 taker 费率 [一手] | **3.00 bps** |
| 一笔 $1,000 名义成交的手续费 | **$0.30** |
| **手续费 / gas 成本** | **≈ 75×** |

> **这就是「operator 替所有人付 gas」在商业上成立的原因。** 它依赖两个条件同时成立：
> **(a) base fee 近零**（靠 25 倍区块 + 极低地板）、**(b) 交易有 bps 级手续费**。
>
> **对发行类应用的直接推论**：launchpad 的手续费率（pump.fun 级 **1%**）比 3 bps 高 **33 倍**，**gas 代付的经济空间更大得多**。→ 第八部分设计方案的财务前提。

---

## 6. 与 OP Stack / Mantle 的正面对照

### 6.1 RISE 领先的 12 项，按改造侵入性分类

| # | 能力 | Mantle | 侵入性 |
|---|---|---|---|
| 1 | 1.5 Ggas 区块预算（50× 有效容量） | ❌ 60M | **纯参数** |
| 2 | base fee 地板 0.0004 gwei | ❌ 50 gwei | **纯参数** |
| 3 | 1 秒出块 | ❌ 2 秒 | **纯参数** |
| 4 | 逐笔预确认流 + WS 订阅 | ❌ 完全缺席 | 需改执行层 |
| 5 | `eth_sendRawTransactionSync` | ❌ | **需改 RPC 层（最低成本）** |
| 6 | pending 阶段即返回 receipt | ❌ | 需改执行层 |
| 7 | Continuous Block Pipeline | ❌ | **需改执行层（最硬）** |
| 8 | RiseDB 自研状态存储 | ❌ | 需改执行层 |
| 9 | 链级 Internal Oracle 公共原语 | ❌ | **纯合约 + 文档** |
| 10 | 链级原生 VRF | ❌ | **合约 + 运营** |
| 11 | 原生稳定币 + float 回投流动性 | ❌ | **商业/法务，非技术** |
| 12 | 官方钱包 + relay 全额 gas 赞助 | ❌ | **应用层** |
| — | 并行 EVM | ❌ | **双方都没有** |
| — | local fee market | ❌ | **双方都没有** |
| — | 自定义 precompile/opcode | ❌ | **双方都没有** |

**分类汇总**：

| 侵入性 | 项数 | 占比 |
|---|---|---|
| 纯参数 / 配置 | **3** | 25% |
| 纯合约 / 运营 / 商业（零协议改动） | **4** | 33% |
| 需改 RPC 层 | 1 | 8% |
| 需改执行层客户端 | 4 | 33% |
| **需改 EVM 语义** | **0** | **0%** |

> **RISE 相对 Mantle 的 12 项优势里，7 项（58%）不需要碰执行层客户端**，其中 4 项连协议都不用改。
> **需要真正硬工程的只有 4 项，且全部服务于同一个目标：压低单笔确认延迟。**

### 6.2 关键的不对称性

**那 4 项硬工程解决的问题，对 perps 是生命线，对发行类应用几乎无关。**

| 场景 | 用户真正在乎的 | 是否需要毫秒级确认 |
|---|---|---|
| **perps 做市/高频** | 报价能否在被打穿前撤掉、下单延迟 | **是，生命线** |
| **发行 / Launchpad** | 发币要不要签三次名、狙击者是否抢到第一个区块、要不要先买 gas 币 | **否，全部无关** |

**⇒ 「只改排序器 + 加系统合约」能拿到 Rise 式体验的：**

| 口径 | 覆盖率 |
|---|---|
| 按能力条目数 | **约 70%** |
| **按发行类应用的价值加权** | **约 90%** |
| 按 perps 的价值加权 | **约 40%** |

### 6.3 一个反直觉的加分项：Mantle 的魔改门槛比想象中低

底稿 N 的一手核查给出三条对委托方极重要的发现：

1. **Mantle 已经不是"没改过链"的通用链。** 它自 **2025-09-16** 起走 **OP Succinct（SP1）ZK validity proof**；**2026-04-16 Arsia 升级**移除 EigenDA 代码路径改为纯 Ethereum blob DA；有独立 `op-geth` fork、`mantle-xyz/revm` fork 与 kona 子树。**换 DA、换证明系统这两件 L3 级的事，它两年内各做过一次。**
2. **走 ZK 路线反而降低了加自定义 precompile / 交易类型的门槛。** Cannon/MIPS 那套「链上单指令仲裁必须认识新指令」的紧箍咒对 Mantle **完全不适用**；代价转为「四层 fork 同步 + SP1 vkey 轮换 + 证明 cycle 成本」——**工程可控、商业可核算，非阻断性**。
3. **"改 EVM 会失去 Superchain 认定"这个顾虑对 Mantle 已经不成立。** OP Succinct 官方 FAQ 明文：*"If your rollup adopts OP Succinct, it will **no longer be classified as a Standard Chain**."* **那张牌早就打出去了。**

> **⇒ Mantle 缺的不是链改造能力，是方向与意愿。**
> 复核确认：至 2026-09-07，官网 / docs / 治理论坛三渠道检索 `preconfirmation` **0 结果**、`appchain` **0 结果**、2026 年**零链改造提案**；官方定位已改写为 **"Institutional Onchain Finance"**（RWA 分发层）。

### 6.4 三条路线对比

| 维度 | **A. 另开 app-specific 链** | **B. 改造主链** | **C. 主链 + 专用 lane** |
|---|---|---|---|
| 工程成本 | 最高（新链全套 + 持续运维） | **中低**（L0+L1 为主） | 中 |
| 流动性割裂 | **最严重** —— 切断与 **xStocks（$633.7M，全球第 2）** 的同链组合性 | **无割裂** | 中等 |
| 桥依赖 | 新增一跳新桥（最高风险面） | 无新增 | lane 无 |
| 可退回性 | **最差**（关链是基础设施级成本） | **最好**（参数可回滚、合约可弃用） | 中 |
| **对 RISE 模式的对应度** | ❌ **RISE 恰恰不是这条路** | ✅ **这才是 RISE 实际走的路** | ⚠️ 无生产先例 |

> **第一阶段硬结论「为 meme 单开 appchain 会切断与 xStocks 流动性的连接」复核后仍然成立**，且本研究补上两条新理由：
> ① **RISE 案例本身就是 A 路线的反证** —— 它是标准 OP Stack rollup 换执行层 + 调参数，落在 B 路线。
> ② **Mantle 的经济结构不支持 A 路线** —— 新链要自负 DA 与证明成本却无费收入基础（run-rate $10–20 万/年）。
>
> **唯一可能推翻的情形（诚实记录）**：若「资产发行」最终必须走证券化 / 一级市场 KYB 闸门，且监管要求与无许可 DeFi **不可共处同一状态机**，则 C 路线优于 B。**这取决于法务边界，不是技术判断。**

---

## 7. RISEx 的真实规模与未偿债务

### 7.1 规模【实测，2026-09-07】

| 指标 | 值 |
|---|---|
| 市场数 | **30** |
| 24h 总成交额 | **$73,556,104** |
| 总未平仓（OI） | **$57,279,216** |
| 状态 | **仍为 gated mainnet（需邀请码）** |
| 准入限制 | 排除美国及另 8 国公民；禁 VPN |
| 审计 | **仅内部审计完成**，第三方审计"in process" |

**RWA 永续已上线并有真实流量**（本研究认为这是对第一阶段判断的重要印证）：

| 市场 | 24h 成交额 | 未平仓 |
|---|---|---|
| XAU（黄金） | $6,766,372 | $3,757,583 |
| XAG（白银） | $2,716,970 | $1,860,002 |
| **SPY**（标普 ETF） | $2,086,278 | **$6,936,965** |
| **QQQ**（纳指 ETF） | $1,214,900 | **$4,910,475** |
| MSTR | $1,605,346 | $130,802 |
| SNDK | $1,245,489 | $261,185 |

**RWA 类合计 24h 成交 ≈ $15.6M（占 21.3%）**，且 **SPY / QQQ 的未平仓显著高于其成交额** ⇒ 是**持仓型/对冲型资金**，不是刷量。

> **与第一阶段「meme × 代币化股票」的赛道判断在需求侧互相印证：用户确实要在链上交易股票风险敞口。**

### 7.2 未上线的功能（必须与已上线严格区分）

| 功能 | 状态 |
|---|---|
| 永续、全仓/逐仓、自成交防护、TP/SL、RLP Vault | **V1 已上线** |
| AutoYield（闲置保证金生息） | **V1.1，未上线** |
| Modular Sub-accounts（RIP-1） | **Draft，未上线** |
| Permissionless Portfolio Margin（RIP-2，依赖 Morpho Lite） | **"Coming soon"** |
| Spot Markets | **未上线** |
| Builder Codes | **未上线** |

**⇒ RISEx 当前抵押资产只有 USDC 一种；"Leverage anything"是路线图。**

### 7.3 app-specific 模式最大的未偿债务

**RISEx 的优势归因中有 7 项建立在中心化之上**：latency bump、清算内部化（自营风控引擎发 IoC 单，无 keeper MEV 竞赛）、额度发放、公共原语准入、USDR 收益归属、上币权、gated 准入。

**而 RISE 官方承诺 progressive decentralization（based sequencing 三阶段）。**

> **一旦 based sequencing 落地，latency bump、清算内部化、operator 代付这些机制全部要重新设计。**
> **这是 app-specific 模式最大的结构性风险：它的绝大部分优势来自"同一个团队握有全部控制点"，而去中心化恰恰是要交出这些控制点。**

---

## 8. 给委托方的六条判断

**① 「把整条链做成 app-specific」的正确技术翻译是「改造主链」，不是「另开一条链」。** RISE 自己就是标准 OP Stack rollup。

**② app-specific 的真正杠杆不在技术，在六个控制点的对齐**：链、旗舰应用、钱包、oracle、稳定币、代币激励。技术只是其中最小的一块（12 项优势里 0 项动了 EVM 语义）。

**③ 若目标是资产发行，不需要自研执行层。** L0（合约）+ L1（参数与排序器）能覆盖约 90% 的价值。那 4 项硬工程服务的是 perps 的毫秒延迟，发行场景无关。

**④ 若同时要做有竞争力的链上 perps，必须先改区块参数。** 【实测】纯 Solidity CLOB 单笔烧 240–420 万 gas，Mantle 60M 区块只装 14–24 笔。**不改参数则链上 CLOB 在 Mantle 物理上跑不起来** —— 但这属 L1，是配置而非工程。

**⑤ 商业模式要照抄的是 USDR，不是 Shreds。** 放弃 gas 费捕获、改捕获稳定币 float 收益、且不分红而回投流动性激励。Mantle 有 **$576M 闲置稳定币** 与 **$2.69B Treasury**，这条路的基础比 RISE 更好。

**⑥ 最该立刻抄的三个低成本项**：
- **`eth_sendRawTransactionSync`（EIP-7966）** —— 纯 RPC 层，周级工程
- **官方钱包 + relay gas 代付 + session key** —— 应用层，零协议改动
- **链级公共原语（oracle 聚合器 / VRF / 原生稳定币包装）** —— 纯合约 + 文档

---

## 附：本部分引用的实测数据索引

| 数据 | 底稿位置 |
|---|---|
| RISE / Mantle / Base / Arbitrum 四链参数对照 | `research/H-rise-chain-infra.md` §8.1 |
| base fee 4,004 区块恒定性 | 同上 §8.2 |
| Shred WebSocket 实测（1 tx/shred、87.7 shred/块） | 同上 §8.3 |
| 非标准 RPC 方法探测 | 同上 §8.4 |
| 单笔 gas 成本与美元折算 | 同上 §8.5 |
| 链上流量归属（94.1% / 96.2% / 37 地址） | 同上 §8.6 |
| RISEx 30 市场成交额与 OI | `research/I-risex-app-coupling.md` §5 |
| Mantle 复核（2.000s / 0.224% / 50 gwei / 单笔 $0.0007–0.0162） | `research/N-opstack-mantle-surface.md` §7 |
| Mantle sequencer 收入序列 | 同上 §5.2 |

**所有实测均附可复现请求体，见各底稿「可复现方法附录」。**
