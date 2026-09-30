# 专题 E：Mantle 现状全面盘点（截至 2026-09-06）

> **文档定位**：本文是 Launchpad-on-Mantle 研究项目的"现状基线"专题。目标是把 Mantle 的链、生态、产品、历史、用户、基建、代币七个维度的**事实**钉死，供后续专题（与 Solana/BSC/Base/Robinhood Chain 对比、以及最终的 Mantle Launchpad 设计）直接引用。
>
> **数据口径与可信度标注**：
> - `【实测】` = 本文作者在 2026-09-06 直接读取 Mantle 主网 RPC / Ethereum L1 合约 / DefiLlama & CoinGecko API 得到，可复现，置信度最高。
> - `【一手】` = 官方文档、官方公告、L2Beat、PR 原文。
> - `【二手】` = 聚合器、媒体、研究机构转述，未经一手源交叉验证。
> - `【存疑】` = 存在源冲突或无法证实，已在第 9 节汇总。
>
> 除特别注明外，所有快照时间为 **2026-09-06**。价格基准：MNT = $0.6070，ETH = $2,500.64【实测·CoinGecko】。

---

## 0. 执行摘要：七条最重要的结论

1. **技术上 Mantle 已经"补完作业"，但这不是它的瓶颈。** 2025-09 上 OP Succinct（SP1）ZK 有效性证明，提现从 7 天砍到 **12 小时**；2026-04 Arsia 升级砍掉 EigenDA，DA 全量迁到 Ethereum blobs，L2Beat 据此把 Mantle 从 Validium 重新归类为 **ZK Rollup**。技术栈现在跟 Base 属于同一梯队甚至部分更激进。

2. **但链是空的。** 【实测】出块 2.000 秒、区块 gas limit 6000 万、平均每块 **1.45 笔交易**、区块填充率 **0.173%**、base fee 长期钉死在 50 gwei 下限（EIP-1559 从未被触发）。日交易约 6.3 万笔，L2Beat 口径 UOPS = 0.13。**Mantle 的问题从来不是性能不够，而是没有需求。**

3. **DeFi TVL 在 2026 年 4 月经历了一次 -86% 的崩塌，且那是"租来的 TVL"。** ATH $704.59M（2026-04-16）→ 4 天内跌到 $196M → 现在 **$97.84M**。峰值来自 2026 年 2–3 月 Aave 部署带来的激励性存款，撤离后原形毕露。

4. **日 DEX 交易量约 90 万美元** —— 比 Base 小 **约 600 倍**，比 Solana 小 **约 2,000 倍**。这不是"差一个数量级"，是差 2.5–3 个数量级。

5. **"mStocks" 这个产品不存在。** 经全面检索，Mantle/Bybit/Backed 的任何一手材料中都**没有** "mStocks" 这个品牌。真实存在的是 **xStocks**（Backed Finance 发行，2025-11-07 官宣上 Mantle，Bybit 提供 CEX 出入金，Fluxion 做链上流动性）。用户"类比 Binance bStocks"的直觉是对的 —— bStocks 确实存在（2026-06-11 上线 BNB Chain，7 周内 AUM 破 $500M），**但 Mantle 侧的对应物叫 xStocks，不叫 mStocks**。后续所有文档需统一改名。

6. **Mantle 试过 meme，而且是自上而下砸钱试的，两次都死了。** Funny Money（2025-02 上线，100 万 MNT 奖池，明确写着"harness the virality of meme culture"）→ **2026-08-16 关停**；Printr（Bybit Venture Studio + Mantle EcoFund 背书，融资 $4.5M，2025-10-21 上线）→ **2026-08-18 关停**。两个都活了 ~10–18 个月。

7. **最致命的是分发断层：交易机器人生态对 Mantle 是零覆盖。** GMGN、Photon、BullX、Axiom、Trojan、Banana Gun、Maestro —— **全部不支持 Mantle**。只有 DexScreener / GeckoTerminal / Birdeye 这类"看盘"工具支持（已实测）。同时 Bybit 自己的 DEX **Byreal 建在 Solana 上，不是 Mantle**。meme 交易的"最后一公里"在 Mantle 上是断的。

---

## 1. 链技术栈现状（2026-09）

### 1.1 升级时间线

| 升级 | 时间 | 内容 | 置信度 |
|---|---|---|---|
| Mantle v1 上线 | 2023-07-14 | OP Stack 分叉 + MantleDA（EigenDA），属 Validium | 【一手】 |
| **Tectonic** | 2024-03 | 迁移到 OP Stack **Bedrock**；`startingTimestamp = 1710474204`（2024-03-15）【实测】 | 【一手】 |
| Everest / Euboea / Skadi | 2024–2025 | fork 名称序列可确认，具体内容与日期未经一手源验证 | 【存疑】 |
| **OP Succinct 上线** | **2025-09-16** | 引入 SP1 ZK 有效性证明；提现窗口 7 天 → 12 小时（2025-09-23 生效） | 【一手】L2Beat milestone + docs |
| **Limb** | Sepolia 2025-12-03 / 主网 2026-01-14 | 兼容 Ethereum **Fusaka** 硬分叉 | 【二手】日期未经一手确认 |
| **Arsia** | **2026-04-16**（L2Beat）/ 2026-04-22（部分二手源） | 新费用模型 + OP Stack 对齐 + **DA 迁至 Ethereum blobs、EigenDA 代码路径移除** | 【一手】L2Beat milestone，附 L1 交易 `0xa9f65671…e55a` |

> 官方 fork 序列（docs.mantle.xyz）：`BaseFee → Tectonic → Everest → Euboea → Skadi → Limb → Arsia`。
> **Arsia 日期存在 6 天分歧**（L2Beat 4/16 vs 二手源 4/22），见第 9 节。

### 1.2 Arsia 升级的具体内容【一手】

1. **压缩感知的 L1 data fee**：用 FastLZ 估算压缩后体积 → 线性回归拟合 Brotli 等效大小；用 `baseFeeScalar` / `blobBaseFeeScalar` 双标量取代旧的 `overhead + scalar`。
2. **新增 Operator Fee**：`operatorFeeConstant + operatorFeeScalar × gasUsed × 100`，走新的 `OperatorFeeVault` 合约 —— 这是给 sequencer 新开的一条收入。
3. **动态 EIP-1559 参数**：base fee 不再固定在 0.02 gwei，sequencer 可逐块通过 `extraData` 设定 denominator / elasticity / min base fee。
4. **DA 足迹感知的区块大小控制**（新增 gas scalar）。
5. **一次性对齐 OP Stack 全部分叉**：Canyon、Delta、Ecotone、Fjord、Granite、Holocene、Isthmus、Jovian 等价功能同时激活。
6. **节点 rebase 到 op-node v1.16.3**；从 Mantle 自定义的多 blob RLP 编码切到标准 OP Stack「一 blob 一 frame」格式。
7. 新增 RPC `eth_estimateTotalFee`；废弃 `eth_getBlockRange`。

> **战略解读**：Arsia 的本质是「放弃技术差异化，换取生态兼容性」。Mantle 过去三年的技术特色（EigenDA、自定义编码、独特费用模型）全部被删除，代价是它现在跟其他 OP Stack 链几乎无差别 —— 好处是工具链/索引器/钱包适配成本骤降，坏处是**再也没有"为什么选 Mantle 而不是 Base"的技术答案**。

### 1.3 OP Succinct / SP1 状态

- **已在主网运行**。L2Beat 徽章："Built on the OP Succinct stack"，prover 为 **SP1 Hypercube**。
- 取代原有的 optimistic 欺诈证明。`SuccinctL2OutputOracle` 保留一个 **optimistic mode 回退开关**（不需要证明、靠 challenger 挑战）—— L2Beat 明确把这列为风险："若开启 optimistic 模式且无人挑战，资金可被盗"。
- **链上验证器**：Ethereum 上的 `SP1Verifier`（`0xc3c6dDDAc8829b233Dc6536Ec024775a57b0AF2A`、`0x8a0fd5e825D14368d90Fe68F31fceAe3E17AFc5C`），Plonk/Gnark **需要可信设置（trusted setup）**。
- **程序版本**：`github.com/mantle-xyz/op-succinct` tag `v3.8.1-mainnet-mantle-arsia.1`。verification key 于 2026-08-10 轮换，L2Beat 标注为 **"not verified"**（因复现需要一个私有依赖）。
- 同一套 SP1 验证器也被 Base、Celo、Morph、X Layer 等 19 条链使用。
- **证明成本**：未找到公开数据。【存疑】

### 1.4 DA：EigenDA 退役

- **迁移时间**：随 Arsia，2026-04-16【一手·L2Beat】。
- **官方给出的三条理由**【一手·docs.mantle.xyz】：
  1. **安全升级** —— 从 Validium 升到真 Rollup，获得完整以太坊经济安全性；
  2. **经济窗口** —— Ethereum Fusaka + BPO2 把 blob 容量从 Target=6/Max=9 提到 **Target=14/Max=21**，blob 变便宜，迁移窗口打开；
  3. **工程对齐** —— 删掉 op-batcher/op-node 里的自定义 EigenDA 代码，回归上游 OP Stack。
- **EigenDA 已完全退役**，L2Beat 的 DA 徽章只剩 "Ethereum with blobs"。
- **DA 吞吐【一手·L2Beat，过去 1 年】**：累计 118.30 GiB，日均 **331.89 MiB**，每个 UOP 平均 **8.51 KiB**。

> **注**：EigenDA 退役对 Mantle 是"止损"而非"胜利"。EigenDA 原本是 Mantle 唯一的架构叙事（modular / DA 分离），现在这条叙事没了，而 EigenLayer 生态本身在 2025–2026 也在收缩（对照 Mantle 自家 cmETH 的停摆，见 3.2）。

### 1.5 出块时间与 1 秒计划

- **【实测】出块时间 = 2.000 秒**（取 5,000 个区块跨度，误差 <0.001s；43,200 块 = 恰好 24 小时）。L1 合约 `l2BlockTime()` **返回 2**【实测】。
- **是否有降到 1s 的计划：未找到任何官方路线图、博客或论坛承诺。**【存疑 / 倾向"没有"】
- **判断**：即使降到 1s 也没有意义 —— 当前区块填充率 0.173%，瓶颈完全不在出块速度。这一点对后续 Launchpad 设计很关键：**不要把"Mantle 不够快"当作失败原因，那是错的诊断。**

### 1.6 Finality（实测 L1 合约参数）

| 参数 | 值 | 含义 |
|---|---|---|
| `finalizationPeriodSeconds()` | **43,200 秒 = 12 小时** | 提现最终性窗口 |
| `submissionInterval()` | **1,800 个 L2 区块 = 1 小时** | state root 提交间隔 |
| `l2BlockTime()` | 2 | 出块时间 |
| `challenger()` | `0x2F44BD2a54aC3fB20cd7783cF94334069641daC9` | = **MantleEngineeringMultisig**（Mantle 自己的 Gnosis Safe） |
| `owner()` | `0x4e59e778a0fb77fbb305637435c62faed9aed40f` | |
| `proposer()` | `0x0000…0000` | legacy 槽为空；L2Beat 判定 proposer 为**白名单许可制** |

合约：`OPSuccinctL2OutputOracle` @ Ethereum `0x31d543e7BE1dA6eFDc2206Ef7822879045B9f481`【实测】

- **软确认**：单一中心化 sequencer，出块即确认，**无正式 preconfirmation SLA**。
- **L1 交易数据提交间隔**：平均 **~7 分钟**【一手·L2Beat】。
- **state root 提交间隔**：平均 **~1 小时**，与 `submissionInterval` 吻合【实测+L2Beat互证】。
- **提现到 L1**：**12 小时**（2025-09-23 起）。官方文档明确说明 12h 是**主动留的安全缓冲**（给事故响应时间），不是技术下限。
- ⚠️ 广泛流传的"Mantle 提现 ~1 小时"是**错的**，与官方文档矛盾。

### 1.7 Gas 定价与费用模型（实测参数）

**原生 gas token = MNT**（L2Beat 确认，Chain ID 5000）。

`TotalFee = L2ExecutionFee + L1DataFee + OperatorFee`

**GasPriceOracle @ `0x420000000000000000000000000000000000000F`（version 1.1.0）实测值：**

| 参数 | 实测值 | 说明 |
|---|---|---|
| **`tokenRatio()`** | **4,143** | **ETH→MNT 换算系数** |
| `baseFeeScalar()` | 169,019 | |
| `blobBaseFeeScalar()` | **0** | 当前不对 blob 费单独计价 |
| `l1BaseFee()` | 40,734,246 wei (≈0.0407 gwei) | |
| `blobBaseFee()` | 1,102,167,443,889,514 wei | |
| `operatorFeeScalar()` | 100,000,000 | |
| `operatorFeeConstant()` | 0 | |
| `overhead()` / `scalar()` / `decimals()` | 188 / 10,000 / 6 | legacy 字段，Arsia 后已弃用 |

> **`tokenRatio = 4143` 的交叉验证**：ETH $2,500.64 ÷ MNT $0.6070 = **4,119**。实测值 4,143 与真实汇率偏差仅 0.6% —— 证实 `tokenRatio` 就是 **ETH:MNT 现价比**，由治理/预言机定期更新，用于把以 ETH 计价的 L1 成本换算成 MNT 收取。
>
> **L1 data fee 用 MNT 支付**（先按 ETH 算，再乘 tokenRatio），且 Arsia 后**不再占用 L2 gas limit**。

**关键实测发现：base fee 已经"死"了**

| 区块 | baseFee (gwei MNT) | gasUsed |
|---|---|---|
| 100,278,282 | 50 | 46,299 |
| 100,277,882 | 50 | 57,487 |
| 100,276,282 | 50 | 392,656 |
| …（12 个采样点，跨 ~80 分钟）| **全部 = 50** | |

**distinct baseFee 值只有一个：50 gwei。** 即使遇到 39 万 gas 的大块也不动。这说明 EIP-1559 常年贴着 **min base fee 下限**运行，**Mantle 事实上没有费用市场**。

> **对 Launchpad 的直接含义**：
> - 好处：费用极度可预测，不会出现 Solana 那种 meme 冲刺时 gas 飙升。
> - 坏处：**没有拥堵定价 = 没有优先级市场**。抢开盘只能靠 priority fee（在空块环境下几乎无效）和延迟。Solana 的 local fee market / 本地热点隔离在这里是"无病呻吟"—— Mantle 根本没到需要隔离热点的地步。

### 1.8 实际单笔成本【实测】

抓取近期真实用户交易（非系统交易）逐笔解算：

| 交易 | gasUsed | L2 费 | L1 费 | 合计 |
|---|---|---|---|---|
| `0xaa92bd18…` | 116,219 | 0.013946 MNT | 0.00028826 MNT | **$0.00864** |
| `0xf584db25…` | 117,594 | 0.006468 MNT | 0.00025538 MNT | **$0.00408** |

- **一笔 DEX swap ≈ $0.004 – $0.009（约合 0.4–0.9 美分）**。
- L1 data fee 只占总成本 **2–4%** —— 迁到 blob 后 DA 成本已基本可忽略。
- 参考：L2Beat 统计 Mantle 全年 L1 运营成本合计仅 **$4.89K**，日均 **$13.40**，每个 UOP **$0.000336**。

> **结论：成本绝对不是 Mantle 的问题。** 单笔 <1 美分，与 Base 同档甚至更便宜。

### 1.9 区块 gas limit / TPS 上限

- **【实测】区块 gas limit = 60,000,000**（一手确认，此前只有二手源）。
- **【实测】平均区块 gasUsed = 104,016 → 填充率 0.173%**。
- **【实测】平均每块 1.45 笔交易 → 约 62,589 笔/日**。
- **理论 TPS 上限**：按 6000 万 gas / 2 秒、一笔简单转账 21,000 gas 计 ≈ **1,428 TPS**；按一笔 swap ~120,000 gas 计 ≈ **250 TPS**。官方未公布口径数字。
- **历史观测峰值**：L2Beat 记录 max UOPS = **25.47**，发生在 **2023-12-27**（即上线半年后的空投农耕期，此后再未接近）。
- 当前 UOPS = **0.13**，即**当前活跃度是历史峰值的 0.5%**。

### 1.10 Sequencer 中心化与强制包含

- **单一中心化 sequencer**，由 Mantle 团队运营；未找到更具体的法律实体信息。
- **L2Beat Stage 分级：Stage 0**（Stage 1 尚差 3 项，Stage 2 尚差 2 项）。
- **风险玫瑰图【一手·L2Beat】**：
  - Sequencer failure → **Self sequence**（可通过 L1 强制包含，最长 12 小时延迟）
  - State validation → **Validity proofs (ST, SN)**，SNARK 需可信设置
  - Data availability → **Onchain**
  - **Exit window → NONE** ⚠️ ——「合约可即时升级，用户没有退出窗口」
  - Proposer failure → **Cannot withdraw**（只有白名单 proposer 能提交 state root）
- **L2Beat 明列的 CRITICAL 风险**：
  1. 「合约收到恶意代码升级即可盗取资金，**代码升级无延迟**（CRITICAL）」
  2. 「中心化 validator 宕机则资金冻结，用户无法自行出块（CRITICAL）」
  3. 「运营方可利用中心化地位**抢跑用户交易**（MEV）」
- **强制包含**：用户可直接向 L1 的 `OptimismPortal` 调用 `depositTransaction` 绕过 sequencer。**延迟窗口存在源冲突**：Mantle 文档写 `sequencer_window = 24 小时`，L2Beat 写 "up to 12h"。【存疑】
- **去中心化路线图**：仅找到一篇 "fair sequencing" 研究博客（提出基于 VRF 的排序），**无主网时间表**。【二手·低置信】

### 1.11 Preconfirmation / Flashblocks

**未找到任何证据表明 Mantle 有 preconfirmation、sub-block streaming、Flashblocks 或 rollup-boost 方案。**

- 对照：Base 已上线 **Flashblocks**（200ms 预确认），Unichain 用 **rollup-boost**。
- Mantle 在这条赛道上**完全缺席**。
- 【一手源为"未找到"，二手源一致认为没有；无 Mantle 工程侧明确表态】

### 1.12 MEV / 私有 mempool

| 项目 | 状态 |
|---|---|
| 公开 mempool | 未明确说明；单 sequencer 的 OP Stack 链**默认是 sequencer-only 私有 mempool**（无公开 P2P 交易池） |
| 保护型 RPC / MEV-Share | **未找到任何 Mantle 特定实现** |
| Priority fee 拍卖 / builder 市场 | **不存在** |
| Searcher / bot 活动 | **无数据**；结合 1.7（base fee 常年触底）与第 6 节（bot 零覆盖），可合理推断**几乎没有** |

> **对 Launchpad 的含义**：这是把双刃剑。
> - **正面**：没有公开 mempool = 天然抗三明治夹子，狙击手看不到 pending 交易。对"公平发射"是结构性优势 —— 这是 Mantle 相对 BSC/Ethereum 少见的真实优势点。
> - **负面**：没有 bot、没有 searcher = **没有做市深度、没有套利者维持跨池价格、没有开盘流动性**。meme 市场恰恰依赖机器人提供即时流动性。**Mantle 消灭了 MEV，也消灭了做市。**

---

## 2. 生态数据

### 2.1 TVL：一次教科书式的"租赁 TVL"崩塌

**当前链上 TVL = $97,840,541**【实测·DefiLlama API】

**历史序列【实测·`api.llama.fi/v2/historicalChainTvl/Mantle`】：**

| 日期 | TVL |
|---|---|
| 2025-08 | $234.7M |
| 2025-12 | $173.4M |
| **2026-02** | **$137.8M** ← 低点 |
| **2026-03** | **$516.8M** ← Aave 部署，暴涨 3.75× |
| 2026-04-16 | **$704,585,156** ← **历史最高** |
| 2026-04-17 | $689.3M |
| 2026-04-18 | $697.0M |
| **2026-04-19** | **$492.6M** ← 崩塌开始 |
| 2026-04-20 | $332.6M |
| 2026-04-21 | $255.6M |
| **2026-04-22** | **$196.3M**（4 天 -72%） |
| 2026-07 | $143.3M |
| **2026-08** | **$62.1M** ← 周期最低 |
| 2026-09-06 | **$97.8M** |

**从峰值回撤 -86.1%。**

**成因分析（本文推断，标注推理链）：**
- 上涨端**可确认**：2026 年 2–3 月 **Aave V3 部署到 Mantle**，官方 PR 称「Mantle 与 Aave 在三周内突破 $1B 总市场规模，DeFi TVL 创历史新高」。
- 下跌端**存疑**：EcoMetrics 研究员将 4/19 的崩塌归因于 **Kelp DAO rsETH 漏洞/脱锚事件**（波及 20+ 网络），并指出 Mantle DAO 随后通过 **MIP-34** 向 Aave DAO 提供 30,000 ETH 紧急贷款以兜底坏账。时间高度吻合，但**未找到 Mantle 官方事故复盘确认因果**。
- **注意一个时间巧合**：Arsia 升级也发生在 2026-04-16，正是 TVL 峰值当天。目前没有证据表明 Arsia 导致资金外流（DA 迁移不影响资产），但两个事件的重叠使得单一归因不可靠。

> **对本项目最重要的结论**：Mantle 的 TVL 高点是**用激励租来的**（Aave 部署 + 补贴），资金是 mercenary capital，一遇风吹草动即刻撤离，且撤离后**回不到起点**（$137M → 峰值 $704M → 现在 $97.8M，比激励前还低）。这直接说明：**在 Mantle 上砸钱买 TVL/交易量是无效的**，Funny Money 和 Printr 的失败是同一个病根。

### 2.2 DEX 交易量

| 指标 | 值 |
|---|---|
| 24h | **$871K – $940K**（快照时点不同） |
| 7d | **$16.20M**（环比 **-10.3%**） |
| 30d | **$67.16M** |
| Perps 24h | **$0**（唯一被跟踪的 IntentX 已标记 Deprecated） |

### 2.3 主要 DEX 逐个点名【实测·DefiLlama】

| DEX | 24h 量 | 30d 量 | TVL | 状态与说明 |
|---|---|---|---|---|
| **Agni Finance** | $441,406 | $30.27M | **$16.56M** | **Mantle 第一大 DEX**，量与 TVL 双第一。Uniswap V3 式集中流动性 |
| **Fluxion Network** | $268,038 | $23.13M | $7.27M | **2025-12-18 主网上线**。混合 AMM(v2+v3) + **RFQ/订单簿**；**xStocks 的主要链上场所**（2026-04 集成，2026-05 上线 "xChange" 原子 RFQ）。上线 9 个月冲到第二 |
| **Merchant Moe (Liquidity Book)** | $148,869 | $13.05M | **$12.76M** | Trader Joe LB 式 AMM。MOE 代币价 ≈$0.012，市值仅 ≈$2.2M |
| Merchant Moe (V2 DEX) | $7,750 | $283,614 | — | 旧版 |
| **Uniswap V3** | $4,754 | $389,695 | 极小 | 部署了但基本没人用 |
| iZiSwap | $175 | $15,686 | — | 边缘 |
| Butter.xyz | $101 | $8,068 | — | 边缘 |
| **Printr** | **$0** | **$6,276** | — | **已关停**（见 4.3） |
| Swapsicle V2 | $396 | $6,131 | — | 边缘 |
| **FusionX V2 / V3** | $70 / $0 | $3,454 / $0 | $93K | **仍存活但已归零**。⚠️ 勿与 2025-10 接管 WOO X 的 PE 基金 "FusionX Digital" 混淆，二者无关 |
| Cleopatra Legacy / CL | $7 / $0 | $575 / $0 | — | 事实死亡 |
| Curve / Pendle V2 / Clipper / WOOFi / Native / Swaap / Reax / FCON / Archly / Napier / Vertex / KTX / Mach / Skate / Tristero | **全部 $0** | **$0** | — | **已部署但完全休眠（15 个）** |

> **28 个被跟踪的 DEX 中，只有 3 个月交易量超过 $1M。** 前三名（Agni / Fluxion / Merchant Moe）合计占 30 天总量的 **99.0%**。
>
> **未在 Mantle 上找到**：PancakeSwap、Ambient Finance。Ondo 只是 RWA 资产发行方，不是 DEX。

### 2.4 稳定币供应【实测·DefiLlama】

**总计 $576,123,790**（7 日 **-2.59%**，30 日 +0.37%）

| 稳定币 | 市值 | 占比 |
|---|---|---|
| **USDT** | $458.59M | **79.6%** |
| Ethena **USDe** | $59.98M | 10.4% |
| Ondo **USDY** | $28.64M | 5.0% |
| **USDC** | $23.76M | 4.1% |
| Agora **AUSD** | $5.15M | 0.9% |
| GRAI | $12,498 | ≈0（且脱锚 -26.6%，报价 $0.73） |

- 历史峰值 ≈ **$980M**（2026-03，与 TVL 峰值同期）。
- **未找到 Mantle 原生的 "mUSD" 稳定币**。
- **重要反差**：稳定币存量 $576M **是链上 DeFi TVL（$97.8M）的 5.9 倍**。说明大量美元躺在链上**不干活** —— 这些是 Bybit 出入金和 RWA 相关的沉淀资金，不是活跃 DeFi 资本。

### 2.5 活跃度

| 指标 | DefiLlama 口径 | growthepie 口径 | 本文实测 |
|---|---|---|---|
| 日活跃地址 | 2,306 | **991** | — |
| 日新增地址 | 221 | — | — |
| 日交易笔数 | 54,144 | **10.9K** | **~62,589** |
| 日链上手续费 | $220 | $214 | — |
| L2 排名（交易数） | — | **第 21 / 27** | — |
| L2 排名（稳定币供应）| — | **第 6 / 27** | — |

> 三个口径差异说明：growthepie 只算用户发起交易与唯一 from 地址；DefiLlama 含合约/系统交易；本文实测是**区块内原始交易总数**（含 sequencer 系统交易，每块必有 1 笔 `0x4200…` 系统交易，故实测值偏高）。
>
> **无论哪个口径，结论一致：日活跃地址在 1,000–2,300 之间。这是一条"千人级"的链。**
>
> ⚠️ 网上流传的「Mantle Q1 2025 日活 65 万 / 日交易 3000 万」比所有可信源高 2–3 个数量级，**判定为不实**。

### 2.6 与 Base / BSC / Solana 的量级对比【实测·同一时点】

| 指标 | **Mantle** | Base | BSC | Solana |
|---|---|---|---|---|
| 链上 TVL | **$97.84M** | $5.669B | $5.792B | $5.925B |
| 稳定币市值 | **$576.12M** | $4.912B | $13.311B | $16.337B |
| **DEX 24h 量** | **$0.94M** | **$567.2M** | **$1,637.4M** | **$1,960.6M** |
| DEX 30d 量 | $67.2M | $24,277.5M | $33,136.5M | $65,028.3M |
| 日活跃地址 | 2,306 / 991 | 238,687 | 1.96M | 2.03M |
| 日交易笔数 | ~54K / 10.9K | 8.59M | 19.13M | 77.57M |
| Perps 24h | **$0** | $146.1M | $18.8M | $551.1M |
| 跨链桥入 TVL | $1.007B | $14.076B | $49.797B | $28.82B |
| 原生代币市值 | MNT $2.00B | —（用 ETH） | BNB $100.6B | SOL $61.6B |

**差距倍数（Mantle 相对）：**

| 指标 | vs Base | vs BSC | vs Solana |
|---|---|---|---|
| TVL | **-58×** | -59× | -61× |
| **DEX 24h 量** | **-604×** | **-1,743×** | **-2,087×** |
| 日交易笔数（DefiLlama 口径） | -159× | -353× | -1,433× |
| 日活跃地址（DefiLlama 口径） | -104× | -850× | -880× |
| **稳定币供应** | **-8.5×** | -23× | -28× |

> **最关键的一行是"稳定币供应"**：这是 Mantle 唯一只落后一个数量级的指标。
>
> **诊断**：Mantle **有钱，但没有交易行为**。$576M 稳定币 + $2.5B 国库 + Bybit 8000 万用户，对应的却是每天 90 万美元 DEX 量和 1,000 个活跃地址。
> **这不是资本问题，是产品与分发问题。** 这一条应作为后续 Launchpad 设计的核心前提。

### 2.7 手续费收入【实测】

| 指标 | 值 |
|---|---|
| 应用层手续费 24h | $33,546 |
| 应用层手续费 30d | $1,052,676 |
| 应用层手续费 1y | $17,126,770 |
| 链手续费（DefiLlama chain 口径）24h | $220 |
| L1 运营成本（L2Beat）日均 | $13.40 |

> 粗算 sequencer 毛利：日收入 ~$220（链口径）vs 日 L1 成本 $13.40 → 毛利率高但**绝对额可忽略（日利润约 $200）**。Mantle 的链本身在财务上是个**不重要的业务**，真正的资产负债表在国库（第 7 节）。

---

## 3. 产品矩阵

### 3.1 总览

| 产品 | 是什么 | 上线 | 规模（2026-09） | 状态 |
|---|---|---|---|---|
| **mETH** | ETH 流动性质押代币（LST） | 2023 | **TVL $590.58M**；LST 排名第 12（Lido $24.3B 第一）；30d 费用 $949K | ✅ 活跃，核心产品 |
| **cmETH** | 再质押代币（LRT），接 EigenLayer/Symbiotic/Karak | 2024 | 峰值 ≈$620M | ⚠️ **停摆中**：2026-05-07 停止新铸造；2026-06 中完成最后一次 EigenLayer 奖励分发；未领奖励 2026-11-07 后作废 |
| **FBTC (ƒBTC)** | 1:1 BTC 抵押的全链合成资产，MPC/TSS 托管，安全委员会含 Antalpha/Mantle/Cobo | 2024 | 约 $100M（2025 末数据，2026-09 精确值未获取） | ✅ 活跃 |
| **Function** | 运营 FBTC 的协议/公司（原 Ignition） | 2024 | — | ✅ 活跃 |
| **UR** | 链上新银行：瑞士 IBAN（EUR/CHF/USD/RMB）、Mastercard 借记卡、SWIFT/SEPA/SIC、NFT 身份 | 2025-06-19 早期访问 | 目标 40+ 国家（亚洲优先）；**无公开用户数** | ✅ 推广中 |
| **Mantle Index Four (MI4)** | 机构级加密指数基金（BTC/ETH/SOL/美元收益），BVI LP，Securitize 代币化，Fireblocks 托管，KPMG 审计 | 2025-04 | 国库锚定投资至 **$400M**；AUM 2025-08 破 **$200M**；DefiLlama 现示 RWA TVL **$154.4M** | ✅ 活跃 |
| **xStocks on Mantle** | Backed Finance 发行的代币化美股（NVDAx/AAPLx/MSTRx/TSLAx/METAx/SPCXX…） | **2025-11-07 官宣** | 未单独披露 | ✅ 活跃，**美国用户禁入** |
| **Fluxion** | Mantle 原生 RWA/股票现货 DEX，混合 AMM + 原子 RFQ | **2025-12-18 主网** | TVL $7.27M；24h $268K；**Mantle 第二大 DEX** | ✅ 活跃 |
| **Mantle Vault** | 稳定币收益产品，接 Aave / Grove / CIAN / Fluxion | 2026-03 | 宣称把 **$1.25B DeFi 深度**接入 Bybit 8000 万 CeFi 用户 | ✅ 活跃 |
| **mStocks** | — | — | — | ❌ **不存在**（见 3.3） |

### 3.2 mETH / cmETH 的关键细节

- **mETH 的 TVL 不计入 Mantle 链上 TVL**。DefiLlama 把 mETH Protocol 归在 **Ethereum** 链下（抵押品托管在 L1）。所以：
  - Mantle 链上 TVL $97.8M
  - mETH Protocol TVL $590.6M（记在 Ethereum）
  - 官方 PR 说的「Mantle DeFi TVL 破 $1B」是**把 mETH(L1) + Mantle 链上 + Aave 存借规模相加**得到的营销口径，**不等于**任何单一 DefiLlama 数字。引用时务必注明口径。
- **cmETH 正在退场**是一个被低估的信号：这是 Mantle 2024 年最大的增长叙事（Methamorphosis 活动、COOK 代币），现在整条产品线关闭。**COOK 代币较 2024-11-08 的 ATH $0.045 下跌约 95%，现价 $0.0022。**

### 3.3 ⚠️ 关于 "mStocks"：必须纠正的命名错误

**结论：Mantle / Bybit / Backed 的任何一手材料中都不存在 "mStocks" 这一产品名。**

检索范围：mantle.xyz 博客与新闻稿、Bybit 公告中心、Backed Finance 官网、PRNewswire / Chainwire / EQS 新闻稿、The Block / CoinDesk / Cointelegraph / Bankless、DefiLlama、Nansen。**零命中。**

（唯一同名实体是印度 Mirae Asset 旗下的散户券商 "m.Stock"，与加密无关。）

**真实存在的对应物是 xStocks：**

| 维度 | 内容 |
|---|---|
| 品牌 | **xStocks**（后缀 x：NVDAx、AAPLx、MSTRx、TSLAx、METAx、SPCXX） |
| 发行方 | **Backed Finance**（瑞士，2021 成立），非 Mantle 非 Bybit |
| 官宣 | **2025-11-07**，Mantle + Bybit + Backed 三方 |
| 托管 | 受监管独立托管方，1:1 足额背书 |
| 代币标准 | **ERC-20（Mantle/Ethereum）+ SPL（Solana），可自由转账** |
| CEX 通道 | Bybit 支持经 **Mantle 网络**充提 xStocks |
| 链上场所 | **Fluxion**（主）、Merchant Moe（部分） |
| 公司行为处理 | "Multiplier" 乘数机制处理拆股/分红 |
| 地域 | **美国公民/美国境内禁止** |
| 2026 进展 | 2026-04 集成 Fluxion；2026-05 上线 xChange 原子 RFQ；**2026-06 上线 SpaceX 代币 SPCXX**；2026-07 开放周末交易；Bybit 已将部分 xStocks 接入 Dual Asset 结构化产品与统一账户保证金抵押 |

**用户直觉的验证 —— bStocks 确实存在，类比成立：**

| 维度 | **xStocks**（Bybit / Mantle） | **bStocks**（Binance / BNB Chain） |
|---|---|---|
| 发行方 | Backed Finance（瑞士第三方） | **BTech Holdings Ltd.（币安关联方，自营）** |
| 监管框架 | Backed 合规代币化框架 | **ADGM/FSRA 批准的招股说明书** |
| 代币标准 | ERC-20 / SPL，**自由转账** | BEP-20，可自托管，但为"权益凭证"性质 |
| 上线 | **2025-11-07** | **2026-06-11** |
| 规模 | 未单独披露 | **2026-07-28 AUM 破 $500M**（首日 $5.6M → 7 周） |
| 标的数 | NVDAx/AAPLx/MSTRx/TSLAx/METAx/SPCXX 等 | 首发 5 个 → **7 周内 46+** |
| 链上场所 | **Fluxion（专用场所）** | PancakeSwap 等通用 BSC DeFi |
| 市场份额 | — | **BNB Chain 到 2026-08/09 占代币化股票市场约 45–50%** |

> **战略含义（对最终 Launchpad 设计极其重要）**：
> 1. **Mantle 早了 7 个月，却输了。** xStocks 2025-11 上线，bStocks 2026-06 上线晚 7 个月，却在 7 周内做到 $500M AUM 和 46 个标的，市场份额 45–50%。Mantle 有先发优势但没转化。
> 2. **差异在自营 vs 代销。** 币安自己发（BTech + 自有招股书），因此能快速扩标的、深度绑定自家分发；Mantle/Bybit 依赖第三方 Backed，扩展速度受制于人。
> 3. **差异在分发。** bStocks 直接进币安主站与 BNB Chain 通用 DeFi；xStocks 需要用户从 Bybit 提到 Mantle 再去 Fluxion —— 多了两跳。
> 4. **但 Mantle 有一个 bStocks 没有的东西：Fluxion 这个"专用场所" + 原子 RFQ（可直接向发行方 mint/redeem）。** 这是做 stock-meme 混合产品时唯一的结构性资产，后续设计应围绕它展开。

### 3.4 其他动作

- **Anchorage** 机构级 MNT 托管集成
- **Moomoo 上线 MNT**，触达美国散户
- **Tokenization-as-a-Service (TaaS)**：面向机构的合规 RWA 代币化全流程
- **RWA 黑客松与奖学金**
- 合作方：Ethena USDe、Ondo USDY、OP-Succinct、EigenLayer、Aave、Grove、CIAN
- 官方自我定位（2025-11 新闻稿原文）：**"the premier distribution layer and gateway for institutions and TradFi to connect with onchain liquidity"**，宣称 **"$4B+ in community-owned assets"**

---

## 4. Meme / Launchpad 历史与结局

### 4.1 Funny Money —— 官方钦定的"meme 引擎"，已关停

| 项 | 内容 |
|---|---|
| 定位 | **AI agent launchpad + DeFAI 交易终端**（非纯 meme 发射台）：无代码发币、永续合约 DEX（100+ 资产，50x 杠杆）、"programmatic prompt trading" AI 交易代理、开发者 API |
| 官宣 | **2025-02-18/19**，Chainwire 新闻稿，Mantle DeFi 增长负责人 Gabriel Foo 站台，宣传语强调背靠 Mantle **~$4B 国库** |
| 激励 | **100 万 $MNT 奖池**交易赛，公开表述为 *"harness the virality of meme culture"*；前 2 名团队获 Bybit 独家合作机会 + 联合营销/KOL 资源 |
| 集成 | MetaMask + **Bybit Web3 Wallet** |
| 再启动 | 2025-06-10 有一次"正式上线"里程碑 |
| **结局** | **2026-08-16 官网公告关停**："funny.money is sunsetting its trading platform" |
| 数据 | 未被 DefiLlama/DappRadar 单独收录，**无 TVL/交易量可查**【存疑】 |
| 备注 | 二手源称创始人有 pump.fun 背景，**未获一手证实**；原 mantle.xyz 公告链接现已 302 跳回首页 |

**存活时长：约 18 个月。**

### 4.2 Mantle 上的 memecoin

| 代币 | 类型 | 峰值 | 现状 |
|---|---|---|---|
| **MINU (Mantle Inu)** | 唯一有持续曝光的原生 meme，合约 `0x51cfe5b1e764dc253f4c8c1f19a081ff4c3517ed`，总量 420.69M | 无可查 ATH 市值 | **事实死亡**：CMC/Coinbase/Crypto.com 显示 $0/NaN；小聚合器显示价格 $0.000052、市值 **≈$21,970**、24h 量 **≈$28** |
| $PILL | Funny Money 社区活动奖励代币 | 无数据 | 随平台关停而死 |
| Printr 发行的各代币 | 跨链 bonding curve 发行 | 无逐币数据 | 平台已关停 |
| $MOE / $SKYDROME | **不是 meme**，是 DEX 治理/工具代币 | — | MOE 市值仅 ≈$2.2M |

**【实测·DexScreener API】当前 Mantle 上排名靠前的交易对：**

| 交易对 | DEX | 24h 量 | 流动性 |
|---|---|---|---|
| WMNT/USDT0 | Agni | $324,067 | $3,497,960 |
| WETH/WMNT | Merchant Moe | $12,185 | $22,495 |
| CATI/WMNT | Merchant Moe | $4,150 | $30,067 |
| ELSA/WMNT | (自定义池) | $3,818 | $44,174 |
| BILLI/WMNT | oku | $3,393 | $53,815 |
| MOE/WMNT | Merchant Moe | $1,223 | $245,524 |

> **量最大的 meme 类交易对日成交约 $3,000–4,000。** 对比 Solana 单个热门 pump.fun 币开盘几分钟就能做到六位数美元。**Mantle 上不存在 meme 市场。**

### 4.3 Launchpad 产品全表

| 产品 | 类型 | 结局 |
|---|---|---|
| **Funny Money** | AI agent + 无代码发币 + 永续 DEX | ☠️ **2026-08-16 关停**（存活 ~18 个月） |
| **Printr** | 跨链（"every chain"）bonding curve 发射台，支持 Solana/Mantle/Ethereum/Base/BNB。**投资方：Bybit Venture Studio、Mantle EcoFund、Axelar Foundation、Sui Foundation、Mirana Ventures、L1 Digital，融资 $4.5M**（$2.5M pre-seed + $2M seed 扩展） | ☠️ **2025-10-21 上线 → 2026-08-18 关停**（存活 ~10 个月）。DefiLlama 上 30 日量残值 $6,276 |
| Merchant Moe | DEX（Liquidity Book） | ✅ 存活，但**明确没有 bonding curve 发币功能**，新代币需自带流动性并人工上架 |
| Skydrome | DEX | 存活（低置信） |
| pump.fun 白标脚本（Maticz / Fenizo 等） | 通用 EVM 模板 | 无任何团队在 Mantle 上做起来 |

> **两次尝试，两次死亡，都在 2026 年 8 月的同一周内关停。** 且两次都不是"缺钱"或"缺背书"——Printr 有 Bybit + Mantle EcoFund + $4.5M；Funny Money 有 100 万 MNT 奖池 + Bybit 联合营销。

### 4.4 官方激励计划全表

| 计划 | 时间 | 预算/规模 | 机制 | 结果 |
|---|---|---|---|---|
| **Mantle Journey — Season Alpha** | 2023-08 → 2024-01 | **2,500 万 MNT**（1,500 万给用户"MJ Miles"，1,000 万给协议排行榜） | 链上/链下活动积分 | 季内 TVL/活跃度冲高，季后典型空投农民流失（无独立留存数据） |
| **Methamorphosis S1** | 2024-07-01 → 2024-10-08 | Powder 积分 → $COOK | 持有/质押 mETH | 完成 COOK 分发；面向 LST 收益用户 |
| **Methamorphosis S2** | 2024-10-30 起 110 天 | Powder → $COOK | 持有/质押 **cmETH** | 把 mETH 用户迁到 cmETH 再质押 |
| **Mantle Rewards Station**（锁 MNT） | 持续多季（如 "MNT Reward Booster S3"） | 逐季不同（曾有 100 万 UXLINK 赠送季） | 锁 MNT → 得 "MNT Power" → 分配到奖池 | MNT 持有者留存工具，与 meme 无关 |
| **COOK Feast** | 持续 | mETH 协议手续费收入注资 | 锁 COOK 赚收益 | 常规治理代币质押 |
| **Mantle EcoFund** | 长期 | **$2 亿**；MIP-26 授权最多 1.2 亿 MNT + 6,000 万 USDx + 30,000 ETH 用于流动性支持（单项目上限 ~2,000 万 MNT） | 战略投资 + 流动性支持 | 投了 Printr、Veda、Agora/AUSD、Infinex、L3E7、Lombard、PumpBTC 等 |
| **Mantle Scouts** | 持续 | **$100 万** MNT 社区赠款 | 社区 Scout 提名 | 小额草根赠款 |
| **公开 Grants** | 持续 | 单项目最高 **$2 万** MNT | 早期项目资助 | 小额开发者支持 |
| **Funny Money 启动奖池** | 2025-02 | **100 万 MNT** | meme 主题交易赛 | 平台已关停 |

**国库历史空投分发记录（Rewards Station）：**
- Ethena (ENA)：25,292 人，400 万 ENA，2024-12-06 → 2025-03-06
- EigenLayer (EIGEN)：22,732 人，57 万 EIGEN，2024-12-11 → 2025-03-11
- mShards/Ethena：25,292 人，4,296,230 ENA
- Methamorphosis (COOK)：**30,141 人**，2 亿 COOK，2024-07-16 → 2024-10-09

> **注意参与人数量级：所有活动的参与者都在 2–3 万人区间。** 这是 Mantle 真实的可动员用户盘子 —— 相对 Bybit 的 8,000 万注册用户，转化率约 **0.03%**。

### 4.5 为什么没跑起来 —— 归因

**A. 定位冲突（最根本）**
Mantle 官方定位是「RWA 分发与流动性层」「链上银行」，主推 xStocks、Openstock 售前 IPO 金库、Franklin Templeton/Ondo/Ethena 合作、$2.5B 国库。**这与 bonding curve 赌场文化在品牌、合规、用户画像上全面冲突。** 一个要 KYC 和招股书的链，很难同时做匿名土狗。

**B. Bybit 把散户投机"包"在 CeFi 里，而不是导到链上**（结构性，最关键）
Bybit Mantle Vault、Bybit Alpha、MNT 作为交易所权益代币 —— 这些设计都让用户**在交易所 UI 内完成投机**，而不是去链上跟合约交互。**Mantle 最大的用户漏斗，恰恰是它链上活跃度的最大抑制器。**
对照：four.meme 深度绑定 Binance Wallet、pump.fun 绑定 Phantom/Solana Mobile、Base 绑定 Coinbase Wallet —— 三者都是**把 CEX/钱包用户直接推进 bonding curve**。

**C. 双代币摩擦（"two-token problem"）**
上链需要：L1 有 ETH → 跨桥 → 再拿到 MNT 付 gas。对比 Solana（单一 SOL）、Base（gas 就是 ETH，与 Coinbase 原生资产一致）。这是被反复提及的 UX 瓶颈。出块 2 秒不是问题，**代币与跨桥摩擦才是**。

**D. 没有任何一个 bonding curve 发射台达到逃逸速度**
Funny Money 和 Printr **都是自上而下用机构/VC/Bybit 资金堆出来的，不是从 degen 社区自下而上长出来的**，且都在 ~1–1.5 年内关停。这指向 PMF 错配，而不是"没人试过"。

**E. 社区构成本身就是收益农民与机构 DeFi 参与者**
Nansen Q2 2026 报告与 OakResearch 均描述 Mantle 活跃社区在玩 mETH/cmETH、MI4、RWA，**不在猎 meme**。自我强化的负循环。

**F. 激励预算的方向全错**
最大的几笔钱（EcoFund $2 亿、Methamorphosis、Mantle Journey 2,500 万 MNT）全部结构化为**质押/再质押/锁仓**，即"奖励不动的钱"。**Mantle 从来没有建立过奖励高频交易的赌博飞轮。**

**G. 机器化交易生态零覆盖**（详见第 6 节）
没有 GMGN、没有 Photon、没有 BullX、没有 Telegram bot。meme 交易的执行层在 Mantle 上不存在。

> **注**：未找到一篇权威的、专门复盘"为什么 meme 在 Mantle 失败"的一手长文（Mirror/论坛/研究报告）。以上归因由多份局部分析 + 本文实测数据综合得出。

### 4.6 与 four.meme / pump.fun 的差距（简述，详见专题 A/B/C）

four.meme（BSC）与 pump.fun（Solana）成功靠三件事：
1. **零启动资金的 bonding curve** —— 任何人即刻发币交易，无需自筹流动性池；
2. **自动"毕业"机制** —— 达阈值后自动建 PancakeSwap/Raydium 池，提供内建叙事弧线与终点；
3. **交易所/钱包级分发** —— Binance Wallet 直连 four.meme、Phantom/Solana Mobile 直连 pump.fun、Coinbase Wallet 直连 Base 发射台，把海量散户直接灌进 bonding curve。

**Mantle 三样全缺**：无持久的原生 bonding curve + 毕业产品；主漏斗 Bybit 把投机包在 CEX 内；执行层机器人生态为零。

---

## 5. 用户构成与分发

### 5.1 Bybit

| 指标 | 值 |
|---|---|
| 注册用户 | **~8,000 万**（Bybit 2025 年度回顾新闻稿，2026 全年沿用） |
| 增长 | 2025 年初 ~5,000 万 → 年末 ~8,000 万（**净增 3,000 万**，即便经历 2 月被盗） |
| 全球排名 | **交易量第 2**（仅次于币安） |
| 现货份额 | CoinGecko 2026 Q2 报告：中心化现货成交额 **~10.0%** |
| Bybit Card 用户 | **300 万+**（2026-04） |
| DAU/MAU | **未披露**【存疑】 |

**2025-02-21 被盗事件**：约 **$14–15 亿** ETH 及相关代币，史上最大加密盗窃。手法为供应链攻击 —— 攻破 Safe{Wallet} 开发者机器，注入恶意 JS 伪造多签 UI，诱使签名者批准冷钱包转账。FBI 归因朝鲜 Lazarus/TraderTraitor。事发数小时内 **35 万+ 提现请求**，Bybit 全程未停提现，以桥接贷款覆盖约 80% 缺口。**用户数未受长期损害。**

### 5.2 Bybit Web3 Wallet 与 Mantle 的集成度

| 项 | 状态 |
|---|---|
| 支持 Mantle 网络 | ✅ 支持，但**只是众多 EVM 链之一，非默认网络**，未被特殊对待 |
| 内置 Swap & Bridge | ❌ **2025 年年中已下线**（Web3 产品线整合） |
| CEX 直接充提 Mantle | ✅ 支持 MNT 经 Mantle 网络充提；充值免费，提现固定链上费 |
| MNT 交易所权益 | 现货手续费约 **-25%**，部分合约 **-10%**（需在账户设置激活） |
| 专属 Mantle 空投中心 / dApp 浏览器 | ❌ 未找到 |

> **结论：集成度是"支持"级别，不是"原生绑定"级别。** 对照 Coinbase Wallet 之于 Base（默认链、一键入金、App 内 Base 生态入口），Bybit 之于 Mantle 的耦合度**明显更浅**。

### 5.3 Bybit 有没有 Binance Alpha 的对应物？—— 有，但没导向 Mantle

| 产品 | 说明 | 是否导流 Mantle |
|---|---|---|
| **Bybit Alpha** | **2025-10 由 "Bybit Web3" 更名而来**。基于统一交易账户的 CeDeFi 入口，用户**不用管助记词和 gas** 就能交易链上资产、RWA、早期/热门代币；含 "Alpha Radar" 热门代币筛选。**这是 Binance Alpha 的功能对应物。** | ⚠️ **反向作用** —— 它的设计目标就是让用户**不必上链** |
| **Bybit Launchpool** | 质押 MNT/USDT/USDC 挖新项目代币。**MNT 是支持的质押资产** | ✅ 有一定 MNT 需求捕获 |
| **Bybit MegaDrop** | 类似币安 MegaDrop 的早期分发 | 资料有限【存疑】 |
| **Byreal** | ⚠️ **Bybit 自己孵化的 DEX，建在 Solana 上，不是 Mantle。** 混合 CEX/DEX 模型（RFQ + CLMM），接入 Raydium/Orca/Meteora 流动性，含 bbSOL（Bybit 的 Solana LST）金库。2025-06 官宣，2025-06-30 测试网，2025 年内主网 | ❌ **完全绕过 Mantle** |

> **Byreal 是本节最重要的事实。**
> Bybit 想做链上 DEX 时，**选择了 Solana 而不是自家生态的 Mantle**。未找到 Bybit 官方解释。合理推断（**非官方证实**）：
> 1. Solana 的散户/meme 交易量与既有 DEX 流动性远超 Mantle，冷启动容易得多；
> 2. Bybit 已在 Solana 有 bbSOL 质押产品，可复用；
> 3. Solana 的低延迟更适合 CEX 级 RFQ；
> 4. 分工定位 —— Mantle 已被定为 Bybit 的 "CeDeFi/RWA 链"，Byreal 去 Solana 抓另一批散户交易人群。
>
> **无论原因如何，结论是硬的：Bybit 用真金白银投票，认为散户交易应该发生在 Solana，而不是 Mantle。** 这是任何 Mantle Launchpad 提案都必须正面回应的问题。

### 5.4 Bybit ↔ Mantle 的关系

- **出身**：Mantle 源自 **BitDAO**（Bybit 早期重度支持）。2023 年两个社区投票合并品牌：BitDAO → Mantle Governance，BIT → MNT。
- **治理**：名义上由 MNT 持有者 DAO 治理；Bybit 非直接控股，但通过**持仓集中度 + 顾问席位 + 流动性/分发支持**拥有巨大影响力。
- **顾问席**：Bybit 联席 CEO **Helen Liu**、现货交易主管 **Emily Bao** 担任 Mantle 顾问委员会 Key Advisor。（注：3.3 引用的 xStocks 新闻稿中，Emily Bao 同时以 "Head of Spot at Bybit" 和 "Key Advisor at Mantle" 两个身份发言。）
- **2026 进展**：Mantle Vault on Bybit（2026-03，宣称接入 $1.25B DeFi 深度给 8,000 万 CeFi 用户）；xStocks 合作；Anchorage 托管；Moomoo 上线。

### 5.5 Mantle 自身用户数

- 官方口径："**超过 100 万活跃用户**"（mantle.xyz）—— 口径不明，与日活 1,000–2,300 的观测值差距巨大，应理解为**累计**而非活跃。
- **未找到**精确的"创世以来唯一地址总数"一手数字。
- **可动员盘子的实测代理指标**：历次官方激励活动参与人数稳定在 **2–3 万人**（见 4.4）。

---

## 6. 开发者与交易基础设施

### 6.1 RPC 提供商

| 提供商 | Mantle 主网 | Archive | Trace/Debug | 备注 |
|---|---|---|---|---|
| **公共 RPC** | ✅ `https://rpc.mantle.xyz`（Chain ID **5000**）【实测可用】 | — | — | 本文所有实测均经此端点 |
| Alchemy | ✅ | ✅（私有端点） | 未确认 | 有联合市场推广 |
| Infura | ✅ | 未确认 | 未确认 | |
| **QuickNode** | ✅ | ✅ 明确 "no pruning" | ✅ `debug_traceTransaction` | 覆盖最完整 |
| Ankr | ✅ | ✅ | ✅ | 公共 + 付费层 |
| Chainstack | ✅ | ✅（专用节点） | 可能 | |
| dRPC | ✅ | ✅（需按 key 确认） | 未确认 | |
| Blockdaemon / Tenderly | ❓ 未核实 | — | — | 【存疑】 |

- **各家免费/公共层的具体 req/s 限额均未找到。**【存疑】
- **mempool streaming**：OP Stack 单 sequencer 链**没有公开 P2P 交易池**，标准的 mempool 订阅产品在此不适用。这对 meme 抢跑 bot 是根本性障碍。

### 6.2 索引器

| 索引器 | Mantle 支持 |
|---|---|
| **The Graph** | ✅ 可部署可查询（有 0xgraph 合作） |
| **Goldsky** | ✅ 完整（Subgraphs + Mirror 实时流） |
| **Covalent / GoldRush** | ✅ 已列 Mantle 主网 |
| **Dune** | ✅ 支持链，有社区看板 |
| Envio | ⚠️ 泛 EVM 支持推断，未见 Mantle 专门声明 |
| Ponder / Alchemy Subgraphs / SubQuery / Flipside / Allium | ❓ 均未确认【存疑】 |

### 6.3 钱包

| 钱包 | 默认内置 Mantle |
|---|---|
| **Rabby** | ✅ |
| **OKX Wallet** | ✅（原生 MNT + 内置 DEX 聚合） |
| **Trust Wallet** | ✅ |
| **Bybit Web3 Wallet** | ✅（可选，非默认） |
| Rainbow | ✅（泛 EVM，确认强度较弱） |
| **MetaMask** | ❌ **需手动添加网络**（或经 Chainlist） |
| **Phantom** | ❌ **不支持 Mantle**（其 EVM 仅覆盖 Ethereum/Base 等） |
| Coinbase Wallet / Backpack | ❓ 未核实 |

> **Phantom 与 MetaMask 两项最致命**：Phantom 是 meme 交易者的默认钱包（完全不支持）；MetaMask 是最大 EVM 钱包（需手动加网络，对小白是硬门槛）。

### 6.4 ⚠️ 交易 bot / 终端覆盖 —— 本报告最关键的表

| 工具 | 支持 Mantle | 依据 |
|---|---|---|
| **DexScreener** | ✅ **已实测确认** | 【实测】API 返回 `chainId: "mantle"`，13 个交易对 |
| **GeckoTerminal** | ✅ **已实测确认** | 【实测】`/networks` 返回 network id `mantle` |
| **Birdeye** | ✅ | 为 Bybit Alpha 提供实时数据 |
| **Ave.ai** | ✅ | 聚合 130–190+ 链 |
| DEXTools | ✅ | 【二手】 |
| **OKX DEX 聚合器** | ✅ | 官方集成公告 |
| **KyberSwap** | ✅ | 多链聚合器 |
| **LI.FI / Jumper** | ✅ | swap + bridge 均支持 |
| 1inch | ⚠️ **证据冲突** | 两次独立检索给出相反结果，**需人工到 app.1inch.io 确认**【存疑】 |
| Odos | ➖ **不适用** —— 运营主体 **2026-07-30 关停全部交易/API 服务** | |
| **GMGN** | ❌ **不支持** | 官方链列表：Solana、BSC、Base、Ethereum、Tron、Monad、HyperEVM、MegaETH、X Layer、Robinhood —— **无 Mantle** |
| **Photon** | ❌ 不支持 | Solana 专用 |
| **BullX** | ❌ 不支持 | Neo = Solana + TRON；Turbo 覆盖 ETH/BNB/Base/Blast/Arbitrum，**无 Mantle** |
| **Axiom** | ❌ 不支持 | Solana 专用 |
| **Trojan** | ❌ 不支持 | Solana 专用 |
| **Banana Gun** | ❌ 不支持 | 支持 ETH/Solana/BNB/Base/MegaETH，**无 Mantle** |
| **Maestro** | ❌ 不支持 | 支持 14+ 链，**无 Mantle** |
| **Jupiter** | ❌ | Solana 专用（设计如此） |

> **结论 —— 分发链条在最后一环断裂：**
>
> | 环节 | Mantle 状态 |
> |---|---|
> | 看盘（charting） | ✅ DexScreener / GeckoTerminal / Birdeye 都有 |
> | 聚合路由（routing） | ✅ OKX / KyberSwap / LI.FI 都有 |
> | **执行（bot / 终端 / 一键狙击）** | ❌ **全部缺席** |
>
> 用户**能看到** Mantle 上的币，也**能换**，但**没有任何一个 meme 交易者习惯使用的执行工具支持 Mantle**。GMGN 的链列表里甚至包含了 Monad、MegaETH、X Layer、Robinhood Chain 这些更新更小的链，**唯独没有 Mantle** —— 这说明不是"太新没来得及适配"，而是**需求信号不足以让它们适配**。
>
> 这是先有鸡还是先有蛋的死锁：**没有量 → bot 不适配 → 没有执行工具 → 更没有量。** 任何 Mantle Launchpad 方案必须自带执行层（自建终端/bot/移动端），不能指望第三方来接。

### 6.5 跨链桥

| 桥 | 机制 | 耗时 | 成本（$100 转账） |
|---|---|---|---|
| **官方 Mantle Bridge** | 规范桥 | 存入：分钟级；**提现到 L1：12 小时**（实测 `finalizationPeriodSeconds=43200`，非旧文所称 7 天） | 仅 gas（L2 侧 MNT + L1 领取侧 ETH） |
| Stargate (LayerZero) | 流动性池 | 分钟级 | ~0.05–0.06% + gas；⚠️ Stargate V1 将于 2026-12 退役 |
| Orbiter | maker-sender | 秒级 ~1 分钟 | ~0.01–0.03% + gas |
| Owlto | 流动性 | <30 秒 | 按比例服务费 |
| Relay | solver | 近即时 | 约 $0.02 固定费 + gas |
| Across | solver + UMA 预言机 | 1–3 分钟 | ~0.05–0.12% + gas |

- **Solana → Mantle 无原生桥路径**。需走 CEX 中转（卖 SOL，从 Bybit/OKX 提 MNT/USDT 到 Mantle），或用支持 Solana 侧的聚合器。**这对承接 Solana meme 用户是重大摩擦。**【存疑：未直接验证 LI.FI 的 Solana→Mantle 路由】

### 6.6 账户抽象 / 无 gas / 嵌入式钱包

- **ERC-4337**：Mantle EVM 等价，标准 EntryPoint/UserOperation 可用。
  - **Particle Network** ✅ 明确公告支持 Mantle（bundler + paymaster）
  - **Etherspot** ✅ 有历史合作
  - **Pimlico / Biconomy / ZeroDev** ⚠️ 标准 4337 供应商，理论可用，但**未见 Mantle 专门公告**
- **嵌入式钱包**：Privy ✅ / Dynamic ✅ / Turnkey ✅（均为泛 EVM）
- **Mantle 自有**：**Mantle Passport**（由 **Para** 提供 MPC 能力）—— 面向 Mantle 生态的无助记词通用钱包。

> AA/无 gas 这块是 Mantle 少数**基础扎实**的领域，且 Mantle Passport 提供了自建入口的可能。**这是 Launchpad 设计中可以依赖的资产。**

---

## 7. MNT 代币经济

### 7.1 供应与市场数据【实测·CoinGecko】

| 指标 | 值 |
|---|---|
| 价格 | **$0.6070** |
| 市值 | **$2,004,351,912** |
| FDV | **$3,774,860,162** |
| 排名 | **#46**（CoinGecko）/ #40（CMC） |
| 流通量 | **3,302,294,383 MNT** |
| 总量 = 最大供应 | **6,219,316,795 MNT** |
| **流通率** | **53.1%** |
| ATH | **$2.86**（**2025-10-08**） |
| 距 ATH | **-78.8%** |
| 近 1 年 | **-48.2%** |
| 近 60 天 | **+47.2%** |
| 24h 成交额 | **$19,500,330** |

> **24h 成交额 $19.5M vs 市值 $2.0B → 换手率仅 0.97%。** 流动性偏薄。

### 7.2 供应历史与解锁

- **BitDAO → Mantle**：BIP-21 + MIP-22（2023-06）确定 **1 BIT : 1 MNT**；2023-07-17 开放迁移（L1 单向）。
- **唯一的"销毁"事件**：**MIP-23** —— 国库持有的 **30 亿 BIT 不参与转换**，使 FDV 供应从 ~92.19 亿降到 **62.19 亿**。这是一次性动作，非持续机制。
- **无常规团队/投资人归属计划**。供应大致分为：~51–53% 流通 + **~47–49% 国库**（DAO 治理释放）。
- **未发现 2025–2026 年任何具体的解锁悬崖事件。** 国库 MNT 的释放需逐笔 MIP + Snapshot 投票，**非日历式解锁**。
- ⚠️ 这意味着 **MNT 没有"解锁抛压日历"，但有"治理随时可放 47% 供应"的悬顶之剑**。

### 7.3 回购 / 销毁

**未找到任何经治理批准的、收入驱动的 MNT 回购或销毁机制。**

- 无任何 MIP / Snapshot 提案 / 官方公告设立回购。
- Tokenomist.ai 的 MNT 页面 Burn 和 Buyback 字段均为 `--`。
- 社区讨论的 "Bybit Flywheel"（MNT 在 Bybit 的手续费折扣、Earn、Card、Launchpool 权益）**是需求侧效用，不是回购销毁**。
- 该结论经多次定向检索确认，**判定为"确实不存在"，而非"未研究到"**。

### 7.4 国库【一手·group.mantle.xyz/treasury，2026-09-06 09:18 UTC】

**总规模：$2,514,355,366**

| 资产 | 美元价值 | 占比 |
|---|---|---|
| **MNT** | **$1,851,738,756** | **73.64%** |
| BTC | $234,245,512 | 9.31% |
| ETH | $222,849,869 | 8.86% |
| 稳定币 | $119,532,123 | 4.75% |
| mETH & cmETH | $67,092,729 | 2.66% |
| bbSOL | $15,415,849 | 0.61% |
| COOK | $3,480,525 | 0.13% |

> ⚠️ **国库的 73.64% 是自己发的币。** 剔除 MNT 后，**真实的"外部资产"仅约 $6.63 亿**（BTC + ETH + 稳定币 + mETH + bbSOL）。
> 官方营销常引用的 "$4B+ community-owned assets" / "$2.4B+ treasury" 应按此口径理解 —— **可动用的硬通货约 6.6 亿美元，其中真正的流动稳定币只有 $1.2 亿。**
> 这对 Launchpad 提案的直接含义：**能拿出的现金激励远小于"$25 亿国库"给人的印象**，且大额动用 MNT 会砸自己的盘。

**另有**：Mantle EcoFund **$2 亿**"催化资本池"（已投 Veda、Agora/AUSD、Infinex、L3E7、Lombard、PumpBTC、Printr 等）。
**MI4**：国库锚定投资**最高 $4 亿**（独立于上表的基金结构）。

### 7.5 MNT 的效用与价值捕获

| 用途 | 状态 |
|---|---|
| Gas 代币 | ✅ 原生 gas |
| 治理 | ✅ forum.mantle.xyz 讨论 + Snapshot（space 仍为 `bitdao.eth`）投票 MIP |
| Bybit 交易所权益 | ✅ 手续费折扣、Earn、Card、Launchpool 质押资产 |
| **原生质押收益** | ❌ **不存在**。真实收益走 mETH（ETH 质押）/ cmETH（再质押）/ MI4，**不流向 MNT** |
| **Sequencer 收入分配** | ❌ **不存在**。L2 手续费减 L1 成本后流入国库，**无自动销毁、分红、回购或质押奖励** |
| **Fee switch** | ❌ 未找到任何在提案或已上线的 fee switch |

> **MNT 是纯粹的 gas + 治理 + 交易所权益代币，没有现金流捕获。** 价值累积路径是间接的（治理 $25 亿国库 + 生态增长），不是直接的。

### 7.6 治理

- 结构：MNT 持有者 → forum.mantle.xyz 讨论 → Snapshot（`bitdao.eth`）投票 MIP → Mantle Core 执行。
- 近期重要提案：
  - **MIP-34「Strategic Credit Facility」**（约 2026-05 通过）：国库向 **Aave DAO 提供 30,000 ETH 贷款**，帮助 Aave 应对 Kelp DAO 漏洞相关的约 $92M 坏账。**Bybit CEO Ben Zhou 公开表态支持。**
  - MIP-33 第三预算周期（2025-08 提交讨论）
  - MIP-32 Mantle Index Fund 治理批准
  - MIP-31 Mantle Core 第二预算周期
  - MIP-30「Exploring the Next Phase of mETH」
  - MIP-26 国库资产支持应用（EcoFund 授权）
- **实际控制权**：形式上 MNT 持有者投票；实际上 **Mantle Core 设定议程 + Bybit 通过持仓集中度与公开背书施加超额影响**。未找到 Bybit 持有链上单方否决/管理员权限的证据。
- ⚠️ **未找到当前 MNT 投票权集中度的链上明细。**【存疑】

---

## 8. 对 Meme / Launchpad 设计的结构化启示

> 本节是把前七节的事实翻译成设计约束，供最终方案专题使用。

### 8.1 必须停止的错误诊断

| 常见说法 | 实测结论 |
|---|---|
| "Mantle 太慢" | ❌ 错。2.000s 出块，区块填充率 0.173%，从未拥堵 |
| "Mantle gas 太贵" | ❌ 错。单笔 swap $0.004–0.009，比 Base 便宜 |
| "Mantle 提现要 7 天" | ❌ 过时。**12 小时**（2025-09-23 起） |
| "Mantle 技术落后" | ❌ 错。ZK Rollup + Ethereum blobs + SP1，L2Beat 归类与 Base 同级 |
| "Mantle 没钱" | ❌ 部分错。国库 $25.1 亿，但**非 MNT 的硬资产仅 $6.6 亿，流动稳定币仅 $1.2 亿** |
| "Mantle 没试过 meme" | ❌ 错。**试过两次，砸了 100 万 MNT + EcoFund，都在 2026-08 关停** |

### 8.2 真正的约束（按严重度排序）

1. **执行层缺失（最致命）** —— GMGN/Photon/BullX/Axiom/Trojan/Banana/Maestro **零覆盖**，Phantom 不支持。**必须自建执行入口**，不能等第三方。
2. **分发被 CeFi 截流** —— Bybit Alpha 的设计目标就是让用户不上链；Byreal 去了 Solana。**必须争取到 Bybit 主站级别的入口，否则重复 Funny Money 的命运。**
3. **无费用市场 = 无优先级机制** —— base fee 常年钉在 50 gwei 下限，priority fee 在空块中无区分度。**开盘公平性需要协议层机制（批量拍卖 / 承诺-揭示 / 时间锁），不能靠 gas 竞价。**
4. **双代币摩擦** —— 需要 ETH 跨桥再换 MNT。**必须做 gas 抽象（paymaster 代付，用 USDT/xStocks 付费），Mantle Passport + Particle 已具备条件。**
5. **社区构成是收益农民** —— 2–3 万人的可动员盘，且习惯质押而非交易。**要么改造他们，要么从 Bybit 8,000 万里重新拉人。**
6. **激励结构错配** —— 历史激励全在奖励"锁仓不动"。**新方案必须奖励换手率与创作者，而非 TVL。**

### 8.3 Mantle 独有的、可利用的结构性优势

1. **无公开 mempool → 天然抗夹子/抗狙击。** 这是 Mantle 相对 BSC/Ethereum 真实且稀缺的优势，是"公平发射"叙事的硬底座。
2. **Fluxion 的原子 RFQ**（可直接向发行方 mint/redeem）—— 这是 bStocks 在 BSC 上没有的专用基础设施，是做 **xStocks 衍生玩法**的独门武器。
3. **xStocks 比 bStocks 早 7 个月**，且是**可自由转账的 ERC-20**（bStocks 是受限的权益凭证），**可组合性更强**。
4. **成本与容量彻底不是瓶颈** —— 6000 万 gas、0.17% 占用率，可以设计非常"重"的链上机制（复杂 bonding curve、链上撮合、频繁 rebalance）而不担心成本。
5. **AA / 嵌入式钱包基础扎实** —— Mantle Passport (Para) + Particle，可做完全无助记词、无 gas 的移动端体验。
6. **稳定币存量 $576M 是链上 DeFi TVL 的 5.9 倍** —— 有大量待激活的闲置美元。

### 8.4 对 outline 中"链 infra 需求"问题的回答

针对 outline 第 9–17 行提出的假设，用本报告数据校验：

| outline 假设 | 实测校验 |
|---|---|
| 瞬时执行容量 | **非瓶颈**。60M gas / 2s，占用率 0.17%，冗余超过 500 倍 |
| 热点争用隔离 / LFM in EVM | **当前无意义**。Mantle 从未拥堵，base fee 从未离开下限。这是"Solana 的问题"，不是 Mantle 的问题。**但若 Launchpad 成功，会立刻变成真问题** —— 应作为 v2 预留设计，非 v1 前提 |
| 弹性可调节区块空间 | **Arsia 已具备**：sequencer 可逐块通过 `extraData` 设定 EIP-1559 denominator/elasticity/min base fee，并有 DA 足迹感知的区块大小控制。**能力已就位，只是没被使用** |
| 发行与流动性组合（bonding curve/AMM） | Mantle **没有**任何存活的 bonding curve 产品。Fluxion 的 AMM+RFQ 混合架构是最接近的可复用底座 |
| 毕业后锁定流动性 | 无先例。Merchant Moe/Agni 可作毕业目标池，但两者 TVL 合计仅 $29M，**深度不足以承接毕业**——这是必须解决的设计约束 |
| 机器化交易生态（RPC/bot/MEV/钱包） | **最大短板，见 8.2 第 1 条** |

---

## 9. 存疑 / 未证实清单

**技术**
1. **Arsia 主网激活确切日期**：L2Beat 记 2026-04-16（附 L1 交易哈希），部分二手源记 2026-04-22 07:00 UTC。**6 天分歧未解**。本文以 L2Beat 为准（有链上交易佐证）。
2. Everest / Euboea / Skadi 三次升级的一手内容与日期。
3. Limb 升级的确切 Sepolia/主网激活日期（仅二手）。
4. OP Succinct / SP1 的证明运行成本（$/proof 或 $/block）。
5. **强制包含窗口冲突**：Mantle 文档 `sequencer_window = 24 小时` vs L2Beat "up to 12h"。**未解**。（注：本文实测的 43,200 秒是 `finalizationPeriodSeconds`（提现），与 sequencer window 是不同参数，不能互证。）
6. Arsia 后费用模型文档页自带 "currently valid solely on Mantle Sepolia" 横幅 —— **但本文实测证明主网 GasPriceOracle v1.1.0 已启用 Arsia 参数**（`tokenRatio`、`operatorFeeScalar`、`baseFeeScalar` 全部可读），故判定文档横幅为**未更新的陈旧提示**。
7. 是否有降至 1 秒出块的官方计划（倾向"没有"）。
8. 官方公布的理论 TPS 上限口径。
9. Mantle mempool 公开/私有的官方明确表态；任何 MEV 保护 RPC。
10. sequencer 去中心化的主网时间表（"fair sequencing" 博客仅二手引用）。
11. sequencer 运营方的法律实体细节。

**生态**
12. **2026-04-19 TVL 崩塌的确切成因** —— Kelp DAO rsETH 事件为**时间吻合的推断**，无 Mantle 官方复盘证实；且与 Arsia 升级日期重叠，单一归因不可靠。
13. Mantle 历史 DEX 交易量峰值的确切数值与日期。
14. FusionX Finance、Cleopatra 的独立 TVL。
15. Mantle 创世以来唯一地址总数（未直接抓 mantlescan）。
16. Funny Money 的 TVL/交易量峰值（未被任何分析平台单独收录；原公告 URL 已 302）。
17. MINU 的历史 ATH 市值。

**产品**
18. FBTC 在 2026-09 的精确 TVL/供应量（仅有 2025 年末 ~$100M）。
19. UR 新银行的用户数/存款额（**从未公开**）。
20. MI4 的投资人数量。
21. xStocks 在 Bybit CEX 与 Fluxion 链上的成交量分配比例。
22. 是否存在独立于 UR 品牌的 "Mantle Banking" / "Mantle Card"（判定为同一产品线）。

**用户与基建**
23. Bybit 的 DAU/MAU（从未披露）。
24. Bybit 选择 Solana 而非 Mantle 建 Byreal 的官方理由（本文推断均未经证实）。
25. **1inch 是否支持 Mantle —— 两次独立检索结果矛盾，需人工到 app.1inch.io 确认。**
26. Blockdaemon、Tenderly 对 Mantle 的支持状态。
27. Ponder / Alchemy Subgraphs / SubQuery / Flipside / Allium 的 Mantle 支持。
28. Coinbase Wallet、Backpack 的 Mantle 默认支持。
29. 各 RPC 提供商的免费层限速。
30. Solana → Mantle 的直接桥路径。
31. Bybit MegaDrop 针对 MNT/Mantle 的具体活动细节。

**代币**
32. 逐日期的 MNT 解锁时间表（Tokenomist granular 表在付费墙后）。
33. 2025–2026 是否存在具体解锁悬崖（**倾向"不存在"**）。
34. Bybit / Mantle Core 当前的链上治理投票权占比。
35. Mantle 官方（非 DefiLlama 推导）的 sequencer 收入数字。
36. DefiLlama 的 "chain fees $220/24h" 与 "app fees $33.5K/24h" 差距过大，疑似 Mantle 适配器不完整，**该链级数字应谨慎使用**。

---

## 10. 可复现的实测方法附录

本报告中标注【实测】的数据可通过以下方式复现：

```javascript
// 链参数
const U = "https://rpc.mantle.xyz";              // Chain ID 5000
rpc("eth_getBlockByNumber", ["latest", false])   // gasLimit 60,000,000; baseFeePerGas 50 gwei
// 出块时间：取 latest 与 latest-5000 的 timestamp 差 / 5000 → 2.000s

// GasPriceOracle 预编译 @ 0x420000000000000000000000000000000000000F  (version 1.1.0)
//   tokenRatio()          selector 0x06f837d3 → 4143
//   operatorFeeScalar()   selector 0x4d5d9a2a → 100000000
//   operatorFeeConstant() selector 0x16d3bc7f → 0
//   baseFeeScalar()       selector 0xc5985918 → 169019
//   blobBaseFeeScalar()   selector 0x68d5dca6 → 0
//   l1BaseFee()           selector 0x519b4bd3 → 40734246
// 注：tokenRatio 的 selector 需经 4byte.directory 反查，
//     多数常见猜测 selector 会 revert。

// L1 上的 OPSuccinctL2OutputOracle @ 0x31d543e7BE1dA6eFDc2206Ef7822879045B9f481
//   finalizationPeriodSeconds() 0xf4daa291 → 43200  (12h)
//   submissionInterval()        0x529933df → 1800   (1h)
//   l2BlockTime()               0x002134cc → 2
//   challenger()                0x534db0e2 → 0x2F44BD2a54aC3fB20cd7783cF94334069641daC9

// 数据 API
// https://api.llama.fi/v2/chains
// https://api.llama.fi/v2/historicalChainTvl/Mantle
// https://api.llama.fi/overview/dexs/mantle
// https://stablecoins.llama.fi/stablecoinchains
// https://api.dexscreener.com/latest/dex/search?q=mantle   → chainId "mantle"
// https://api.geckoterminal.com/api/v2/networks            → network id "mantle"
```

---

## 11. 核心信息源

**一手**
- L2Beat Mantle：https://l2beat.com/scaling/projects/mantle
- Mantle 文档：https://docs-v2.mantle.xyz/ ；Arsia 升级说明、Fee Model Handbook、DA 迁移说明、Forced Transaction Inclusion
- Mantle 国库看板：https://group.mantle.xyz/treasury
- Mantle 治理论坛：https://forum.mantle.xyz/ ；Snapshot：https://snapshot.org/#/bitdao.eth
- op-succinct（Mantle 分支）：https://github.com/mantle-xyz/op-succinct/tree/v3.8.1-mainnet-mantle-arsia.1
- xStocks 官宣（2025-11-07）：https://www.prnewswire.com/apac/news-releases/mantle-collaborates-with-bybit-and-backed-to-bring-us-equities-onchain-pioneering-next-trillion-dollar-wave-of-tokenized-assets-302608747.html
- Fluxion 主网（2025-12-18）：https://chainwire.org/2025/12/18/fluxion-mainnet-goes-live-on-mantle-advancing-native-spot-liquidity-for-defi-and-rwas/
- cmETH 退场时间表：https://methprotocol.xyz/blog/announcements/cmeth-wind-down-timeline
- Printr 融资与上线：https://www.prnewswire.com/news-releases/printr-raises-4-5m-partners-with-bybit-mantle--byreal-and-officially-launches-the-first-every-chain-token-launchpad-302589422.html
- MI4：https://www.businesswire.com/news/home/20250424178524/en/ ；https://app.rwa.xyz/assets/MI4
- MIP-23（供应优化）：https://forum.mantle.xyz/t/passed-mip-23-mnt-supply-optimization-in-preparation-for-launch/7148
- Binance bStocks：https://www.binance.com/en/bstocks-landing

**数据**
- DefiLlama：https://defillama.com/chain/mantle ；/dexs/chain/mantle ；/stablecoins/mantle ；/protocol/meth-protocol
- growthepie：https://www.growthepie.com/chains/mantle
- CoinGecko：https://www.coingecko.com/en/coins/mantle
- DexScreener：https://dexscreener.com/mantle ；GeckoTerminal：https://www.geckoterminal.com/mantle/pools

**分析**
- Nansen Mantle Q2 2026 报告：https://nansen.ai/post/mantle-q2-2026-report
- OakResearch Mantle 综述：https://oakresearch.io/en/reports/protocols/mantle-mnt-comprehensive-overview-full-stack-on-chain-banking-infrastructure
- Blockworks Mantle 财务看板：https://blockworks.com/analytics/mantle/mantle-financials

---

*文档生成：2026-09-06 | 专题 E | 下一步衔接：专题 A/B/C/D 的对比结论 + 最终 Launchpad 设计方案*
