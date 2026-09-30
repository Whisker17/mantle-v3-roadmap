# 专题 D：Robinhood Chain 深度拆解

> 数据截止：**2026-09-06**。
> 标注约定：
> - **[一手]** = Robinhood / Arbitrum / Uniswap / Chainlink / SEC 官方文档或链上直读
> - **[链上]** = 本报告作者通过 `https://rpc.mainnet.chain.robinhood.com` 直接 `eth_call` 读取，或 DefiLlama API 直取
> - **[二手]** = 媒体（The Defiant / Business Insider / CryptoSlate / crypto.news 等）
> - **[未证实]** = 仅有单一来源或社区说法，未能与一手来源对齐

---

## 0. 一页纸速览（Executive Summary）

Robinhood Chain 是 2026 年最反直觉的一个案例：**一家美国上市券商（NASDAQ: HOOD）为「代币化美股」造了一条 L2，结果这条链 90% 以上的经济活动来自 meme 币投机，而 meme 币的报价资产（quote asset）正是它自己发行的美股代币。**

三个数字定义了这条链：

| 维度 | 数值 | 来源 |
|---|---|---|
| 主网上线 | 2026-07-01（伦敦 "The World is Flat" 发布会） | [一手] Robinhood Newsroom / Arbitrum Blog |
| 应用层累计手续费（2 个月） | **$386.79M** | [链上] DefiLlama |
| 其中 Pons 一家 | **$72.48M**（V1 $25.00M + V2 $47.48M） | [链上] DefiLlama |
| 对比 pump.fun 生涯累计手续费 | $1.21B（约 2 年） | [链上] DefiLlama |
| Pons 单日手续费（2026-09-04） | **$8.75M** vs pump.fun 同期 $0.68M | [链上] DefiLlama |
| 链上代币化股票市值 | $133.2M（全球代币化股票 $2.91B 的 4.6%） | [二手] rwa.xyz |
| 链 24h DEX 交易量 | $1.53–1.61B（全链第 2，仅次于 Solana） | [链上] DefiLlama |

**核心张力**：Robinhood 想要的是 RWA 结算层，得到的是一个 meme 赌场；但恰恰是 meme 赌场给美股代币带来了它原本不可能拥有的**链上流动性和持有需求** —— 因为 meme 币需要一个 quote 资产，而在 Robinhood Chain 上最有叙事张力的 quote 资产就是 TSLA / NVDA / AMC。这是**「用投机需求给 RWA 引流」**的第一个大规模实验，也是本报告要给 Mantle mStocks 提炼的核心可移植结构。

---

## 1. 链本身（Robinhood Chain）

### 1.1 技术栈与身份

| 项 | 值 | 来源 |
|---|---|---|
| 架构 | Arbitrum Orbit / Nitro（2026 年官方品牌为 **"Arbitrum Dedicated Blockchains" / Arbitrum Platform**） | [一手] docs.robinhood.com/chain |
| 结算层 | **Ethereum L1**（不是 Arbitrum One，因此触发 AEP 许可条款） | [一手] Arbitrum DAO Factsheet |
| 核心技术方 | **Offchain Labs**（非 Caldera / Conduit；社区流传的 RaaS 中间商说法未获证实） | [一手] 治理页 + [未证实] 反证 |
| chainID | **4663**（测试网 46630） | [一手] docs.robinhood.com/chain/connecting |
| 原生 gas token | **ETH**（无自有 gas 代币） | [一手] |
| 浏览器 | robinhoodchain.blockscout.com（Blockscout） | [一手] |
| 公共 RPC | `https://rpc.mainnet.chain.robinhood.com`；推荐商 **Alchemy**（`robinhood-mainnet.g.alchemy.com`），另有 QuickNode / Blockdaemon / dRPC / Validation Cloud | [一手] |
| 测试网 | 2026-02-10 上线，主网前累计处理 **>2 亿笔**交易 | [一手] Arbitrum DAO Factsheet |
| 主网 | **2026-07-01** | [一手] |

**"launch-and-migrate" 范式**：Arbitrum 官方博客把 Robinhood 树为样板 —— 2025-06 先在 **Arbitrum One** 上线第一代 Classic Stock Tokens 验证需求，一年后再迁到专属链。这对 Mantle 是直接可比的：*先在共享链上验证资产需求，再决定是否要专属执行环境。*

### 1.2 ~100ms 区块与 preconfirmation

- Arbitrum 官方表述：**"Configurable block times and preconfirmations to achieve 100ms latency, delivering consumer-grade responsiveness while maintaining settlement-grade finality."** [一手]
- **独立验证（本报告）**：2026-09-06 读取 `eth_blockNumber` = `0x3547092` = **55,865,490**。自 2026-07-01 主网上线起约 67 天 ≈ 5,788,800 秒 → **平均 103.6 ms/block**。[链上]
- The Defiant 在 2026-09-04 事故取证中测得 3 小时内产出 106,756 个区块 = **101 ms/block**。[二手]
- **结论：~100ms 是真实的、可持续的，不是营销数字。** 但 preconfirmation 的具体协议（是否为 Arbitrum 的 Bolt 类方案、preconf 承诺由谁签名、违约罚没如何设计）**没有任何公开技术规格** —— 目前只有营销层描述。[未证实]

> **对 Mantle 的含义**：100ms 区块在 EVM Rollup 上已被工程验证。Mantle 若要做 meme launchpad，2 秒出块是结构性劣势 —— 不是"慢一点"，而是**狙击/抢跑的博弈完全不同**（见 §3.4 anti-snipe 设计与出块时间的耦合）。

### 1.3 Rollup vs AnyTrust、DA 方案

- **标准 Rollup，不是 AnyTrust。** 官方原文：*"Robinhood Chain is an Arbitrum Layer-2 Chain built on Ethereum, **using Ethereum blobs for data availability** and ETH as the native gas token."* [一手]
- 因此**不存在 DAC（Data Availability Committee）**，"委员会成员是谁"这个问题在此链上无意义。
- **代价**：Robinhood Chain 据报是**全体 L2 中最大的以太坊 blob 消费者** —— 平静期占全网 blob 的 **45%**，2026-09-04 拥堵期占 **28%**，单链超过 Base + Arbitrum One 之和。[二手] The Defiant/Blobscan
- **2026-09-04 事故**：批次提交地址 `0xDaa5…87F4` 两次静默，合计 **14 分钟**（8m36s + 5m24s）没有向 L1 sequencer inbox 提交 blob。**链本身没有停机**（区块仍以 101ms 产出），停的是 L1 数据可用性与提款能力。Arbitrum 官方归因为"blob 市场拥堵"，但第一段 8m36s 发生时以太坊尚有 263 个空闲 blob slot 且价格低 —— 官方解释与数据不符，**Robinhood 至今未发布技术复盘**。[二手 + 未证实归因]

> **对 Mantle 的含义**：选 Rollup + blob 的成本是真金白银的，而且在高频 meme 场景下会把整条链变成 blob 市场的价格接受者。Mantle 已有自己的 DA 方案（EigenDA），这在成本上是**优势**，但在"Ethereum-aligned 安全叙事"上是劣势 —— 这是个需要显式取舍的点。

### 1.4 Gas 与费用模型

- Gas token = **ETH**，两段式费用：L2 执行 gas + L1 calldata/blob 费；通过 `ArbGasInfo` 预编译查询。[一手]
- Arbitrum 侧称之为 **"Dynamic Pricing"**，卖点是"可预测的单位经济"。[一手]
- **实际表现（费用是被 meme 打爆的）**：
  - 2026-08-22：链上 gas 费 **~$54,254/日**
  - 2026-09-02：**~$4.45M/日**，11 天涨 **82 倍**，单日超过 Ethereum($304K) + Solana($613K) + Tron($874K) + BNB Chain($480K) **之和**
  - 普通用户单笔成本从 <$0.01 涨到 **~$0.32**（base fee floor 0.02 gwei）
  [二手] The Defiant，基于 DefiLlama
- **Gas 补贴**：Robinhood 在自家 Robinhood Wallet 内对 >$0.50 的 swap **全额代付 gas，无上限，至 2026-09-29 23:59 EST 截止**。[一手] Robinhood 支持页
  → **当前所有链上活跃度数据都不是稳态**。补贴退出是这条链 2026 年 Q4 最大的单一变量。

### 1.5 Sequencer 归属、排序规则与 Timeboost

这是 Robinhood Chain **最具设计意图**的地方，也是最值得 Mantle 抄的一点。

**FCFS，没有优先费，没有 Timeboost。** 官方原文（`differences-from-ethereum` 页）：

> *"On Ethereum, miners or validators order transactions based on priority fees... **Robinhood Chain employs a first-come, first-served model based on sequencer arrival time. Priority gas auctions do not exist here; consequently, increasing your fee will not shift your transaction ahead of others already in the queue.**"* [一手]

- **Arbitrum Timeboost / express lane：未启用。** 官方定位为 *"Predictable Transaction Ordering... no transaction can bypass others by paying higher fees."* [一手]
- 因此**不存在 express-lane 拍卖收入需要分配**，MEV 收入模型退化为纯粹的"到达时间竞速"（延迟战争，而非出价战争）。
- Sequencer 由 **Robinhood 独家运营**；L2Beat 对 "Sequencer failure" 一项标注 **"No mechanism"**。[二手] L2Beat

**⚠️ 关键合规钩子：Transaction Screening（交易筛查）**

官方文档明文承认存在 sequencer 级别的筛查：

> *"Robinhood Chain maintains compliance standards through **sequencer-level screening**. While rare, this mechanism can influence smart-contract execution; for instance, **any transaction associated with a sanctioned address will be excluded from inclusion**... Since a blocked transfer is never processed, it simply appears as though the event never occurred, ensuring indexers remain synchronized with the actual state."* [一手]

链接指向 Arbitrum 的 **Advanced Compliance Filtering** 文档。L2Beat 的 Discovery 数据进一步指出，ArbOS 61 引入了 `ArbFilteredTransactionsManager` 预编译（`0x…0074`），授权的 "filterer" 角色可以登记任意交易哈希并使其在状态转换函数中失败 —— **包括从 L1 强制包含（force-inclusion）进来的交易**，且没有延迟窗口。[二手] L2Beat
（Arbitrum 在 2026-08 发布的 **ArbOS "Elara"** 版本标题即为 *"Compliance Filtering, Priority Fee Support"* —— 说明这是被产品化的能力。[一手] Arbitrum Blog）

> **这意味着**："permissionless" 这个词在 Robinhood Chain 上只适用于**合约部署与调用**，不适用于**抗审查**。合约层是开放的，交易层是可被过滤的。这是"合规链"的真实形态，也是它能同时容纳美股代币和 meme 币的制度前提。

### 1.6 治理、验证者与 L2Beat 评级

**Security Council：8 席多签**（[一手] docs.robinhood.com/chain/governance）

| 席位 | 参与方 |
|---|---|
| 2 | **Robinhood** |
| 1 | BitGo, Inc. |
| 1 | Chainlink Labs |
| 1 | Fireblocks Trust Company |
| 1 | Offchain Labs |
| 1 | Paxos |
| 1 | Talos |

- 常规操作：**6/8** 批准 + **7 天链上 timelock**
- 紧急操作：**7/8** 批准，**绕过 timelock**
- → Robinhood 单方**无法**修改协议参数，但这仍是一个高度中心化的许可结构。

**验证者**：使用 **BoLD**（Bounded Liquidity Delay）争议解决，但只有 **2 个许可制验证者：Offchain Labs 与 Alchemy**。[一手]
L2Beat 据此标注 "less than 5 external actors that can submit challenges"。

**L2Beat 分级：Stage 0**（Stage 1 与 Stage 2 各有 3 项待修）。TVS ~$2.96B（2026-09-06）。[二手] L2Beat

> **一句话**：*链的使用是无许可的；链的验证与审查是许可的。* 这两句必须分开说，媒体几乎全部混为一谈。

### 1.7 AEP 10% 收入分成

- **确认适用**。Robinhood Chain 直接结算到 Ethereum（而非 Arbitrum One），因此落入 Arbitrum Expansion Program 许可范围。[一手] Arbitrum DAO Factsheet
- **分成结构：净协议收入的 10%** →
  - **8% → Arbitrum DAO 国库**
  - **2% → Arbitrum Developer Guild**
  - 通过 **AEP fee router** 流入，计入 DAO 常规财报。[一手]
- Robinhood CFO **Shiv Verma** 在 2026 Q2 财报电话会上称 Robinhood 每笔交易赚 "a few basis points"，**约一半分给 Arbitrum** —— 但未给出精确费率、交易笔数或 GAAP 对账。[二手] CryptoSlate 引财报会
- 有二手报道称 Robinhood Chain 的许可费占 Arbitrum DAO 2026 年 7 月总收入的 **~35%**。[未证实，仅第三方聚合]

> **对 Mantle 的含义**：Mantle 自有 L2，不存在 AEP 抽成 —— 这是 **10% 的纯毛利优势**。但反过来，Arbitrum 生态给 Robinhood 提供了 Offchain Labs 的工程支持、BoLD、compliance filtering 这些开箱即用件。Mantle 需要自己补齐（尤其是**合规过滤能力** —— 这是 mStocks 上链的前置条件）。

### 1.8 生态基础设施

官方 ecosystem 表（[一手] docs.robinhood.com/chain）：

| 类别 | 合作方 | 作用 |
|---|---|---|
| RPC & AA | **Alchemy** | 推荐 RPC、Data API、Gasless Transaction Infra（ERC-4337 一等公民支持） |
| 预言机 | **Chainlink** | Data Feeds（股票代币）、Data Streams、CCIP；亦持 1 个 Security Council 席位 |
| 跨链 | **LayerZero** | 全链消息与资产桥；另有 canonical Arbitrum bridge |
| 机构托管 | **Fireblocks**、**BitGo** | 各持 1 个 Security Council 席位 |
| 稳定币 | **Paxos（USDG / Global Dollar）** | 链上主力稳定币，持 1 席 |
| 分析 | **Entropy Advisors**（arbdata.com/ecosystems/robinhood）、**Allium**、**Zerion**、**CoinGecko** | |
| 合规风控 | **TRM Labs** | |
| DEX | **Uniswap**（专属 AMM 部署，v2/v3/v4/UniswapX 全量）、**Rialto**（PropAMM）、**Pleiades**（自营 AMM） | |
| 借贷 | **Morpho** | Robinhood Earn 的底层 |
| 永续 | **Lighter**、**Arcus** | Lighter 承诺 1,100 万 $LIT 给 Robinhood 社区 |

- **Alchemy 同时是推荐 RPC + 2 个验证者之一** —— 基础设施集中度很高。
- **未证实项**：Chainlink Proof of Reserve 是否用于股票代币储备证明（官方文档只提 Data Feeds）；Goldsky/Ponder 等 indexer 未在官方名单。

### 1.9 上线以来的数据（时间序列）

**应用层手续费（DefiLlama `overview/fees/Robinhood Chain`，即链上所有 dApp 收取的费用，非 gas 费）**：[链上]

| 日期 | 应用层日手续费 | 备注 |
|---|---|---|
| 2026-06-19 | $5 | 上线前零星 |
| 2026-07-01 | $2,146 | **主网日** |
| 2026-07-14 | ~$160K | **Pons V1 首日** |
| 2026-07-29 | $3.25M | V1 高峰期 |
| 2026-08-22 | ~$2.20M | meme 大潮前夜 |
| 2026-08-29 | $8.72M | |
| 2026-09-01 | $16.98M | |
| **2026-09-04** | **$24.51M** | **历史峰值** |
| 2026-09-06 | $9.62M | 周末回落 |

**汇总（2026-09-06）**：[链上]

| 指标 | 24h | 7d | 30d | 累计 |
|---|---|---|---|---|
| 应用层手续费 | **$18.62M** | $113.12M | $188.16M | **$386.79M** |
| 应用层收入（协议留存） | $4.21M | $25.50M | $42.86M | $111.65M |
| 链 gas 费（Robinhood 自身收入基数） | ~$2.9M | $24.95M | $27.97M | — |
| DEX 交易量 | **$1.53–1.61B** | $11.30B | $24.12B | **$45.96B** |
| TVL | $908.65M | — | — | 7 日 +29.7%（$701M→$909M） |

**其他关键指标**：
- 累计交易数 **576M+**、地址数 **12.3M**（上线两月，Robinhood 官方口径；地址数含大量 bot/合约，非真人）[二手]
- 单日交易峰值 **5.5M 笔**（2026-08-30）[二手]
- 链上稳定币市值 **$964.56M**（USDG 占 66.4%）[二手] DefiLlama
- 链上 RWA 活跃市值 **$251.58M**；其中 Robinhood 股票代币 **$133.2M** [二手] DefiLlama / rwa.xyz
- 2026-07 下旬超越 Solana 的代币化股票交易量；约上线 3 周超越 Base 的 DAU；2026-08-29 单日应用收入超越 Ethereum [二手] The Defiant

**RH Chain 24h DEX 交易量按协议拆分（2026-09-06）**：[链上]

| 协议 | 24h 量 |
|---|---|
| **Uniswap V4** | **$1,029.7M** |
| GMGN（交易 bot/终端） | $641.7M |
| **Pons V2** | $161.4M |
| Ramses CL V2 | $77.1M |
| up v3 | $76.5M |
| GIGA V3 | $41.5M |
| Metric V1 | $36.4M |
| Fables | $35.5M |
| Uniswap V2 | $34.2M |
| Orvex | $19.9M |

> Uniswap v4 承接了绝大部分流量 —— 因为 **Pons / PAIR 毕业后的池子全部是 v4**。这一条对 Mantle 有直接施工意义：**launchpad 的毕业目的地决定了链上 DEX 格局**。

---

## 2. Robinhood Stock Tokens（美股代币）

### 2.1 两代产品，法律结构完全不同（媒体几乎全部混淆）

| | **Classic Stock Tokens**（旧，EU app 内） | **Stock Tokens**（新，Robinhood Chain 上） |
|---|---|---|
| 发行/对手方 | **Robinhood Europe, UAB**（立陶宛，公司代码 306377915） | **Robinhood Assets (Jersey) Limited（"RHJ"）** |
| 法律形态 | **OTC 衍生品合约**，MiFID II 金融工具，PRIIP（KID 风险等级 **7/7**） | **代币化债务证券（tokenised debt securities）**，Base Prospectus + Final Terms 结构化票据 |
| 代币角色 | 权利来自与 RHEU 的双边合同，代币只是"表示" | **ERC-20 代币本身就是该证券** |
| 转账 | **不支持转出到外部钱包** —— 事实上是封闭账本 | **自托管、可自由转账的标准 ERC-20** |
| 上线 | 2025-06，Arbitrum One | 2026-07-01，Robinhood Chain |
| 现状 | 继续在 Robinhood Europe app 提供 | 主推产品 |

**→ 只有第二代才具备 DeFi 可组合性。第一代不能做 quote 资产，第二代才能。** 这是 Pons/PAIR 得以存在的前提。

### 2.2 发行主体与法律结构 [全部一手，来自 docs.robinhood.com/rhj]

- **发行人**：Robinhood Assets (Jersey) Limited
- **注册地**：泽西岛私人有限公司，First Floor, La Chasse Chambers, Ten La Chasse, St. Helier, JE2 4UE
- **注册号**：**162428**；**LEI：984500ADFHQZ9D6B9A29**
- **监管状态（官方原文，极其重要）**：

  > *"**The Issuer is not regulated.** However, in connection with the issuance of Stock Tokens, the Issuer has obtained certain consents in Jersey. Such consents do not constitute prudential supervision of the Issuer or an endorsement of its products. The relevant Jersey authorities do not assume any responsibility for the financial soundness of the Issuer..."*

- **招股书审批**：**列支敦士登金融市场管理局（FMA Liechtenstein）** 作为 EU Prospectus Regulation 下的主管机构批准了 Base Prospectus。[一手 /rhj/product]
  （注：有二手来源称经挪威 Finanstilsynet 通行护照 —— 属于 EEA passporting 的下游动作，主管机构是 FMA。）
- **权利限定（反复出现的原文）**：*"provide economic exposure to underlying securities but **do not grant investors any legal or beneficial rights in, or against the issuer of, those underlying securities**."* → **无投票权、无股东权利、无对标的公司的任何请求权。**

### 2.3 服务商全景（这是最完整的一张图，官方 /rhj/service-providers）[一手]

| 角色 | 实体 | 地点 |
|---|---|---|
| **Authorised Participant（唯一 AP）** | **Bitstamp Global Ltd** | 英属维尔京群岛，Road Town, Tortola |
| **Broker & Custodian（券商兼托管）** | **Alpaca Securities LLC** | 纽约，12 E 49th St |
| **Paying Account Provider** | **JPMorgan Chase Bank, N.A., London Branch** | 伦敦金丝雀码头 |
| **Tokenizer** | Robinhood Assets (Jersey) Limited c/o Cavendish Fiduciary (Jersey) Limited | 泽西岛 |
| **Security Agent & Verification Agent** | **Security Agent Services AG** | 瑞士楚格 |

**关键解读**：
1. **唯一 AP = Bitstamp（Robinhood 2024 年收购的交易所）**，链上文档中简写为 **"BBVI"**（Bitstamp BVI）。原文：*"Only Authorised Participants (at issuance, **the only Authorised Participant is BBVI**) may subscribe for Stock Tokens directly from RHJ after KYB onboarding (the primary market)."* [一手]
2. **托管方 = Alpaca Securities LLC**，一家美国注册券商 —— 这解答了此前"未披露托管方"的疑问。（注：Alpaca 同时也是 Dinari dShares 的托管方，即两个竞争性代币化股票模型共用同一家美国券商基础设施。）
3. **Security Agent Services AG (Zug)** 的存在，说明这是一个**有担保的票据程序（secured note programme）** —— 破产时由独立担保代理人变卖标的股票、以现金分配给代币持有人。

### 2.4 1:1 背书与赎回

- **官方 FAQ 原文**：*"Yes. **Every single Stock Token in circulation is backed 1:1 by the corresponding underlying equity.** The underlying shares are held securely by our US-based custody partner."* [一手]
- **破产处置**：*"In the unlikely event of the Issuer's insolvency, **an independent security agent will sell the underlying shares, and arrange for the cash proceeds to be paid to token holders**."* → **现金清偿，不交付实股。** [一手]
- **赎回**：可在二级市场卖出；也可在无 AP 的情况下**直接向发行人赎回**，须完成 RHJ 的 KYC/AML。[一手]
- **投资者费率**：[一手 /rhj/product]
  - 申购/铸造：**0.00%**
  - 赎回：**发行后 90 天内 0.00%**；**90 天后 0.05%**
  - 发行人可在 Final Terms 限制内调整
- **法定转让确认数**：**1 个区块**（"Block confirmations for a legal transfer: 1"）—— 即链上 1 个 100ms 区块即构成法律上的证券过户。这是极为激进的法律设计。[一手]
- **储备证明**：**未找到任何独立第三方审计/attestation 或 Chainlink Proof of Reserve feed**。Chainlink 关于 Robinhood 的文档只覆盖价格喂价。[未证实/疑似缺失]

### 2.5 代币技术形态：ERC-20 + **ERC-8056 multiplier**（本节最重要的技术创新）

这是 Robinhood 处理"股票代币无法 rebase"这个老大难问题的解法，**极其值得 Mantle mStocks 直接照抄**。

**问题**：股票有分红、拆股、并股。如果代币做 rebase（改 `balanceOf`），所有 AMM 池、借贷协议、会计系统都会炸。如果不处理，代币就会永久偏离标的。

**Robinhood 的解法：ERC-8056（Scaled UI Amount Extension）**[一手 docs.robinhood.com/chain/building-with-stock-tokens]

```solidity
interface IScaledUIAmount {
  function uiMultiplier() external view returns (uint256);   // 18 位定点，1e18 = 1.0
  event UIMultiplierUpdated(uint256 oldMultiplier, uint256 newMultiplier, uint256 effectiveAtTimestamp);
  event TransferWithScaledUI(address indexed from, address indexed to, uint256 value, uint256 uiValue);
}
interface IScaledUIAmountNewUIMultiplier {
  function newUIMultiplier() external view returns (uint256); // 预定生效的新乘数
  function effectiveAt()     external view returns (uint256); // 生效时间戳
}
interface IScaledUIAmountBalances {
  function balanceOfUI(address account) external view returns (uint256); // 折算成"股数"
  function totalSupplyUI() external view returns (uint256);
}
```

核心性质：
- **`balanceOf()` 与 `totalSupply()` 永远不变** —— **股票代币不是 rebasing token**。[一手明确声明]
- 每代币代表的标的股数 = `raw amount × uiMultiplier / 1e18`
- 上线时 `uiMultiplier = 1e18`（1 代币 = 1 股）
- **分红处理**：不派现金。公司派息 → 自动再投资买入更多股票 → **`uiMultiplier` 上调**。原文：*"Instead of receiving a cash payout, your token's multiplier increases. This means your token dynamically represents more than one share of stock over time."* [一手]
- **拆股处理**：10:1 拆股 → multiplier 1.0 → 10.0
- **AMM 不受影响**：*"Onchain swaps remain unaffected"* —— 因为 raw balance 不动，池子里的数量不变。
- **预言机自动吸收 multiplier**：Chainlink feed 返回的是 **每代币价格 = 标的股价 × multiplier**，集成方**不要自己再乘一次**。

**含义**：股票代币价格会**持续高于**标的股价（因为分红被再投资），跟踪的是**总回报（total return）**而非价格回报。官方举例：$100 股票，分红后 multiplier 升至 1.05，则 feed 价格 = $105。

> **对 Mantle mStocks 的直接建议**：ERC-8056 是目前唯一被大规模生产验证的「让证券型代币在 AMM 里正常工作」的方案。若 mStocks 采用 rebase 或"发新代币"处理公司行动，任何 launchpad 的永久锁仓 LP 都会在第一次分红时被套利抽干。**这是设计的第一性约束。**

### 2.6 Chainlink 喂价与休市处理

[一手 docs.robinhood.com/chain/oracles-and-price-feeds + docs.chain.link]

- 用的是 **Chainlink Data Feeds**（推送式，`AggregatorV3Interface` / `latestRoundData()`），**不是** Data Streams。Chainlink 单独开了一个品类叫 **"Tokenized Equity Feeds"**。
- 定价公式：**Token Price = Underlying Equity Market Price × Multiplier**，标的价来自 Chainlink 的 **24/5 股票喂价**（聚合盘前、盘中、盘后、overnight 时段）。
- **休市行为（关键）**：官方原文 —— *"Stock feeds update **24/5**, following market hours."*；Chainlink 侧 —— *"When underlying equity markets are closed (weekends, holidays, thin overnight windows), **the feed may hold the last published price**... **These feeds do not have heartbeats during off-hours**."*
- **公司行动期间暂停喂价**：代币暴露 `oraclePaused()`。RHJ 流程为 `pauseOracle()` → `updateMultiplier(new, effectiveAt)` → 确认对齐 → `unpauseOracle()`。
  ⚠️ 官方提醒：*"The flag is **advisory and not enforced on-chain**, so a paused oracle may still return a value — keep your staleness check (`updatedAt` vs. heartbeat) as the primary guard."*
- **Chainlink 明确免责**：*"Chainlink does not provide corporate-action calendar data or automated pause triggers; pause timing and multiplier updates are coordinated by Robinhood."* → **所有公司行动判断权 100% 中心化在 Robinhood。**
- 另提供 **L2 Sequencer Uptime Feed**，官方建议读价前先验 sequencer 活性。

### 2.7 24/7 交易 vs 美股休市：真实机制与已发生的事故

**这是整个 Robinhood Chain 最脆弱的地方，也是 Mantle 必须提前设计的地方。**

结构性事实：
1. 美股每周交易 ~32.5 小时；链上交易 168 小时。**每周有 135+ 小时没有新的股票侧参考价、没有股票做市商套利。**
2. 链上价格的**唯一再锚定机制**是 **AP（Bitstamp/BBVI）的 mint/redeem** —— 这是**自由裁量的、非自动化的**，与"连续套利的包装资产"完全不同。
3. Chainlink feed 在休市时**冻结**，AMM 却继续成交 → 报价与预言机可以任意背离。

**已发生的真实事故：**

| 事件 | 日期 | 详情 |
|---|---|---|
| **HIMS / BONER 逼空** | 2026-08-29 ~ 09-01 周末 | 一个叫 **BONER** 的 meme 币在单个 RH Chain 池子里**囤积了 31,198 / 58,714 = 53% 的代币化 Hims&Hers 全部流通量**。NYSE 休市期间，该池把 HIMS 打到 **$132.64**，而周五真实收盘价是 **$28.84**（>4.6x 背离）。最终由唯一 AP **BBVI 增发约 4,000 枚 HIMS 代币**才把价格拉回 **$30.10**。[二手] The Defiant |
| **AMC 池 35x** | 2026-08-30 周末 | PAIR 上线 AMC 作为首个"仙股"配对资产后，一个 AMC 配对代币在无股票做市商的周末冲到最后参考价的 **~35 倍**。[二手] |
| **Farmmi / JINQIAN 反向传导** | 2026-09-02 | 一个从 Farmmi 自己的 SEC 20-F 文件里抠出来的词（"金钱菇"）做成 meme 币，配对到一个**假冒的、无发行人的"代币化 Farmmi"合约**（匿名钱包一次性铸造、自留 38%、自写 `PoolRepricer`）。假币盘中冲到 $1.83，**而真实纳斯达克上市的 FAMI 股票当日盘中暴涨 321%**（$0.1187 → $0.50）—— 链上投机疑似反向传导到真实市场。30 分钟内出现 10+ 个跟风 meme。[二手] The Defiant |

**Robinhood 官方的处理工具**（[一手 /rhj/corporate-actions]）：
- 公司行动期间**暂停交易**：通常从生效日凌晨 ~2AM CET/CEST 起停止新订单，到美股开盘前 ~3:30PM CET/CEST 恢复
- 部分公司行动会**取消未成交订单**
- 退市/清算等情形下，**可能只允许卖出或赎回**
- 另设 **Price Deviations 页面**，公示"持续溢价/折价"的代币 [一手 /rhj/price-deviations]

> **注意：以上工具全部作用于 Robinhood 自己的产品面（app / RFQ / 一级市场），对第三方 AMM 池毫无约束力。** Uniswap v4 上一个 AMC/meme 池不会因为 AMC 停牌而停摆。这是**监管套利和系统性风险的交汇点**。

### 2.8 可用地区与限制

[一手 /rhj/restricted-jurisdictions + newsroom]

- 宣称覆盖 **120+ 国家**，通过 Robinhood Wallet 提供；**190+ 只股票代币与 ETF**
- **明确排除**：**美国（及所有 U.S. Persons，Reg S 定义）、加拿大、英国、瑞士**
- **Prohibited Investors（制裁类，可变更）**：古巴、白俄罗斯、伊朗、朝鲜、俄罗斯、叙利亚、乌克兰、南苏丹、苏丹、缅甸、委内瑞拉
- **英国特殊措辞**：网站援引 FSMA 2000 第 21 条豁免，*"directed only at persons who are resident or incorporated outside of the United Kingdom"*
- **技术层面**：合约是**标准 ERC-20，无 ERC-3643 式转账钩子，未发现链上白/黑名单函数** → 限制发生在 **KYC（app 层）+ 一级市场 KYB（仅 Bitstamp 可 mint/redeem）**，而非代币层。二级市场转账是完全开放的。
  ⚠️ **未证实**：ERC-20 字节码内是否存在未公开的 pausable/blacklist 后门（本报告未做字节码审计）。
- Chainlink 文档承认这一开放性：*"Primary mint and redeem... can be permissioned, but **once the tokens are onchain, anyone can hold them and anyone can liquidate positions that use the feed**."*

### 2.9 与 xStocks / bStocks / Ondo / Dinari 的结构对比

**SEC 2026-01-28 Corp Fin 声明**把代币化股票分为三类，这是当下最权威的分类框架 [一手 sec.gov]：

1. **Issuer-sponsored（发行人自发）** —— 代币就是证券本身，发行人/过户代理把股东名册上链（Securitize SECZ、Superstate Opening Bell）
2. **Third-party custodial（第三方托管）** —— 代表持有人通过 security entitlement 对标的证券的间接权益（Dinari dShares、Ondo Global Markets）
3. **"Linked securities"（挂钩证券）** —— 由第三方自行发行、提供**合成敞口**、**不是标的证券发行人的义务、不赋予任何来自标的发行人的权利**
   → **Robinhood Stock Tokens 属于第三类。** SEC 特别警示：持有人"可能面临对该第三方的风险（如破产），而持有标的证券本不会面临此类风险"。

| 维度 | **Robinhood（RHJ）** | **xStocks（Backed）** | **bStocks（Binance/BTech）** | **Ondo Global Markets** | **Dinari dShares** |
|---|---|---|---|---|---|
| SEC 分类 | ③ Linked securities | ③ Linked（瑞士 DLT 法 tracker certificate） | ③ Linked | ② Custodial（离岸版偏③） | ② Custodial |
| 法律包装 | 泽西岛**代币化债务证券**，Base Prospectus（FMA Liechtenstein 批准） | 瑞士 DLT 法 *Registerwertrechte*，Liechtenstein FMA 招股书 | BEP-20 代币化证券，BTech Holdings | 离岸 BVI 实体 + 美国境内经 SEC 注册过户代理 | 由 SEC 注册、FINRA 会员券商 Dinari Securities LLC 持实股 |
| 发行人受监管？ | **"The Issuer is not regulated"**（官方原文） | 受瑞士/列支敦士登框架 | [未证实] | 部分 | 是 |
| 股东权利 | **无**（分红经 multiplier 再投资） | 无 | 无 | 离岸无；**美国境内版有投票权** | **有**（投票、分红、公司行动） |
| 托管 | **Alpaca Securities LLC**（美国券商） | InCore Bank / Maerki Baumann（瑞士银行） | 未具名 "regulated custodian" | 美国注册券商 | Dinari Securities LLC |
| 可转让性 | **标准 ERC-20，完全自由转账**；一级市场仅 Bitstamp | 多链、可转让 | BNB Chain，可自托管 | 多链 | 多链，但更偏合规托管 |
| DeFi 可组合性 | **极强**（Morpho 抵押、Uniswap v4 池、launchpad quote 资产） | 强 | 中 | 中（Euler 抵押） | 弱 |
| 公司行动 | **ERC-8056 multiplier**（业界唯一大规模生产方案） | 通常调整代币数量或价格 | [未证实] | [未证实] | 直通实股 |
| 赎回 | 现金赎回（0.05%，90 天后）；不交付实股 | 现金 | 经 Binance 转换 | 2026 起 24/7 即时 mint/redeem | 销毁 dShare → 券商卖出 → USDC |
| 代币化股票市值（2026-09-05） | **$133.2M** | $633.7M | $659.4M | **$869.6M** | $11.2M |

**全球代币化股票总规模 $2.91B（2026-09-05，rwa.xyz）**，30 天 +14.4%，267 万持有人。

> **关键洞察**：Robinhood 的代币化股票**市值只排第 6**，但**链上交易量与 DeFi 组合度第 1**。它赢在**开放的二级市场 + ERC-8056 + 自有链**，输在**监管包装最薄**（发行人"不受监管"）。
>
> **Mantle mStocks（对标 Binance bStocks，实为 Bybit + Mantle）的定位取舍**：
> - 如果走 bStocks 路线（第三类 linked、封闭生态），**无法成为 launchpad quote 资产**，就复制不了 Robinhood 的飞轮。
> - 想复制飞轮，必须做到三件事：**① 标准 ERC-20 自由转账；② ERC-8056 式 multiplier 处理公司行动；③ 有 AP 能在偏离时增发/赎回来做锚定。** 第③点是 Robinhood 在 HIMS 事件中唯一的救命机制。

---

## 3. Pons 完整机制（本章为工程级拆解，可直接照此重实现）

### 3.1 项目概览

| 项 | 值 |
|---|---|
| 官网 | **ponsfamily.com**（X: @ponsdotfamily）；源码 **github.com/ponsdotdev/ponsfamily**（MIT） |
| 运营方 | **Pons Labs, LLC**（匿名团队，社区称 "MEADGod"），**无披露融资** |
| 与 Robinhood 关系 | **完全无关**。Robinhood 官方声明第三方应用"不构成背书、合作或担保" |
| V1 factory | `0xA5aAb3F0c6EeadF30Ef1D3Eb997108E976351feB`（`PonsLaunchFactory`） |
| V2 factory | `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e`（`PonsV2LaunchFactory`） |
| $PONS 代币 | `0x39dBED3a2bd333467115dE45665cC57F813C4571` |
| V1 首笔手续费 | **2026-07-14** |
| V2 首笔手续费 | **2026-08-04**（架构据报 2026-08-03 上线） |
| Solidity | V1 `^0.8.30`，V2 `^0.8.26`；OZ `Ownable2Step` / `ReentrancyGuard` / `SafeERC20` |

**两代同时在线**（README: *"Both generations are live source and both factories are verified on chain"*），V1 仍在产生手续费。

### 3.2 V1：直接进 Uniswap V3（无 bonding curve）[一手 GitHub]

**单笔交易 `launchToken()` 完成全部步骤：**

1. **CREATE2** 部署固定供应量 ERC-20（`PonsLauncherToken`），**全部供应量铸给 factory**
   - 地址可预测（`predictTokenAddress()`），且有 **vanity 后缀 `...bbbb`**
   - 链上元数据：logo / description / socials
2. 初始化 Uniswap **V3** 池（`DexConfig`：V3 factory / positionManager / swapRouter / fee tier / tickSpacing，均由 owner 配置）
3. 铸造**单边（one-sided）集中流动性头寸** —— **只放 meme 币，不放任何 quote 资产**，从 owner 配置的 `initialTick` 起始
   → 这是 V1 的精髓：**创建者零成本注入流动性**，价格从最低 tick 单向上行，本质是"用集中流动性模拟 bonding curve"
4. 头寸 **NFT 锁进 locker**（V1 的 locker 是 *"configurable"* —— 比 V2 弱的保证）
5. 可选 **atomic dev-buy**：用 `msg.value` 余额在同一笔交易内买入

**其他 V1 参数**（`LaunchConfig`）：`pairToken`（quote 资产，非硬编码 ETH）、`graduationThreshold`、`initialTick`、`supply`、`maxWalletBps`、`maxTxBps`、`restrictionBlocks`

**V1 的 anti-snipe**：只有硬上限 —— 同区块买入封锁、单钱包上限、累计买入上限、限制区块窗口。**无衰减税。**

**V1 的"毕业"**：`graduationStatus()` 只是一个 **view 函数**，比较锁仓头寸本金与阈值，**不触发任何迁移**。

### 3.3 V2：bonding curve → Uniswap V4 [一手 GitHub]

**核心设计哲学（README 原文）**：*"The curve **trades in the same quote asset its future V4 pool will use**... Because the curve collects the eventual pool asset from the very first trade, **graduation seeds the pool directly — no router, no swap, and no price oracle anywhere in the system.**"*

> 这是对 pump.fun 模型的一个真正的工程改进：**毕业时零滑点、零预言机依赖、零 MEV 窗口**。pump.fun 系的迁移往往需要 swap/router，会产生可被抢跑的窗口。

**曲线数学**（`PonsV2BondingCurveMath.sol`）：**恒定乘积（x·y=k），输入端收费**

```
amountOut = amountIn·(10000 − feeBps)·reserveOut
            ─────────────────────────────────────────────
            reserveIn·10000 + amountIn·(10000 − feeBps)
```

改编自已审计的 `BootstrapPool.sol`（code-423n4/2025-01-iq-ai）。**属 pump.fun 的虚拟储备恒定乘积家族，不是指数/线性/Bancor 曲线。**

**关键状态量**：
- `phantomQuote` = **虚拟 quote 储备**，决定开盘价（每次 launch 由 owner 配置）
- `graduationThreshold` = 真实 quote 储备目标
- **预留代币公式（决定曲线卖完点与毕业点重合）**：
  ```
  reservedTokens = supply · phantomQuote / (phantomQuote + graduationThreshold)
  ```
  在曲线初始化时固定 → **相同配置的每次 launch 都有确定性的毕业种子，永远不会有"剩余代币"。**

**毕业阈值 4.2 ETH**：
- 二手来源**一致地**引用 **4.2 ETH** 作为原生 quote 的默认阈值（稳定币 quote 据报为 **~8,090 USDG**）
- **但在 factory 源码中未找到硬编码常量** —— 它是 `LaunchConfig` 里的 owner 可配置字段
- **结论**：4.2 ETH 是**运营默认值（policy），不是协议常量**。[二手 + 一手反证]

**两阶段、无许可的毕业流程**：

1. `curve.graduate()` —— 停止曲线交易、清扫手续费、把 **100% 储备**交给 factory
   - **在跨越阈值的那笔买单内部自动触发，包在 try/catch 里** → 毕业失败绝不会让用户的买单回滚
   - **强制转入（force-sent）的捐赠被排除在种子之外**，防止有人靠直接打款扭曲毕业价格
2. `factory.createGraduatedPool()` —— 直接铸造 **full-range Uniswap V4 头寸**，NFT 送进 locker
   - **可重试**（retryable），储备永远不会被卡死
   - `PonsV2GraduationGuard`：**无状态预检**，镜像 V4 真实的拒绝路径，确保曲线绝不会被排干进一个无法 seed 的头寸
   - **7 天 rescue delay** 兜底真正无法 seed 的 launch

**永久锁仓（V4）**：
- `PonsV2LaunchLocker` **不暴露 `collectFees`、不暴露提取、不暴露任意调用（arbitrary call）** —— **代码级永久锁定**
- 注意：NFT **不是**被打进黑洞地址，而是被放进一个**没有出口的合约**。效果等价，但可审计性更好。
- 池子的 **V4 核心 LP 费必须设为 0**，全部费用经济学走 hook。

**Hook 架构**（`hooks/PonsV2MemeHook.sol`）：
- **Singleton（单例）** hook 服务**所有**毕业池，用 `PoolId` 索引注册表
- 只启用 **`afterSwap`**（+ `beforeInitialize` 做注册门禁）
- 每笔 swap 从 V4 的 flash-accounting 中直接抽成
- 若抽到的是 meme 币，会在**受价格冲击上限约束**（`maxInternalPriceImpactBps` 默认 **300 = 3%**）的前提下，批量用池子自身流动性换回 quote 资产 → **协议/创建者收入永远是 quote 计价，不会变成没人要的 meme 灰尘**

### 3.4 Anti-snipe 衰减税（部分证实，需注意矛盾）

**Factory 层已证实的常量**：[一手 GitHub]

| 参数 | 默认值 | 上限 |
|---|---|---|
| `snipeTaxStartBps` | **9,900 bps = 99%** | 硬顶 9,900 bps |
| `snipeTaxSeconds` | **15 秒** | 可调 1–60 秒 |
| 豁免名单 | ≤ **32 个地址**，创建者 + 费用接收方自动豁免 | |

- 参数在 launch 创建时**快照固化**，事后调整 factory 默认值**不影响已有 launch**
- 若非零，`snipeTaxStartBps` 必须超过 2,000 bps 的常规费用上限

**⚠️ 未能证实的部分**（重要，重实现前必须验字节码）：
- **衰减曲线形状** —— 二手来源普遍称"指数衰减到 0"，但称衰减窗口为 **5 秒**，与 factory 的 15 秒默认值**冲突**
- **是否只对买单生效**（二手称卖单不收）
- **税款去向** —— 二手称"折回该 launch 自己的曲线流动性"，非销毁、非国库、非创建者

**设计意图解读**：99% 起始税意味着**第 0 秒抢跑者的收益被完全没收**。这在 100ms 出块的链上是必要的 —— 15 秒 = **150 个区块**的博弈窗口，足以让人类用户与 bot 站到同一起跑线。

> **对 Mantle 的含义**：anti-snipe 税的窗口必须以**秒**而非区块计价，因为出块时间是变量。若 Mantle 出块 2 秒，15 秒只有 7-8 个区块，衰减曲线的粒度会粗糙到无法用 —— **这是出块时间对 launchpad 机制设计的隐性约束。**

### 3.5 费用结构与 70/30 分成（有一个被媒体普遍误报的关键点）

**Hook 默认参数**：[一手 GitHub]

| 参数 | 默认值 | 含义 |
|---|---|---|
| `hookFeeBps` | **100 = 1%** | 交易费（owner 可上调至 10%） |
| `protocolFeeShareBps` | **3,000 = 30%** | 协议分成 |
| `buybackBurnBps` | **5,000 = 50%** | 从创建者那 70% 里再切 50% |
| `creatorTaxBps` | 创建者自设，协议上限 **1,000 = 10%** | 叠加在上述之外，**全额归创建者** |
| 总交易费硬顶 | 曲线费 ≤10%，创建者税 ≤10%，**合计 ≤20%** | |

**因此 1% 基础费的真实拆解是三段而非两段：**

```
1% 交易费
├── 30%  → 协议（Pons Labs）
├── 35%  → 创建者（即时可领）
└── 35%  → 回购金库（buyback vault）
```

**⚠️ 最重要的一条更正：Pons 的回购是「锁仓」，不是「销毁」。**

README 原文：*"**Buybacks are locked, not burned**: `PonsV2BuybackVault` vests bought-back supply linearly over five years with a weighted-average vesting clock."* 且设计笔记第 7 条：*"locks are permanent... **buybacks vest, they don't burn.**"* [一手]

→ 这 35% 被用来买回**该 launch 的 meme 币**，存入 `PonsV2BuybackVault`，**5 年线性解锁**，解锁时再按记录的份额在协议/创建者间分配。

**其他费用**：
- **创建费（launch fee）**：**0.0005 ETH / 每次发币**（DefiLlama 方法论明文，V1 与 PAIR 同为此数）[链上]
- **毕业费**：无单独条目
- **推荐/返佣**：合约中未发现（可能存在于前端层）
- 手续费通过 `IPonsV2FeeEscrow` 的 claim-based 账本结算，支持 ETH 或 ERC-20
- `CREATOR_FEE_RECIPIENT_TIMELOCK = 3 天`（改收款地址要等 3 天）

### 3.6 $PONS 代币经济与回购销毁（**链上直读验证**）

**⚠️ 这里存在一个被所有二手来源混淆的关键区分：**
- **`PonsV2BuybackVault`（合约层）** 回购并锁仓的是**每个 launch 的 meme 币**，5 年归属，**不销毁**。
- **$PONS 代币本身**另有一套**协议层的回购销毁政策**，走的是打入黑洞地址。

**本报告的链上直读结果（2026-09-06，`eth_call` @ RPC mainnet）**：[链上，一手证据]

| 项 | 值 |
|---|---|
| `totalSupply()` | **1,000,000,000 PONS** |
| `balanceOf(0x…dEaD)` | **296,993,295.46 PONS** |
| `balanceOf(0x0)` | 0 |
| **已销毁比例** | **29.70%** |

（二手来源普遍称 "~29.3% burned" —— **与链上实测吻合**，说明黑洞地址口径正确。）

**CoinGecko（2026-09-06）**：价格 **$0.9289**，市值 **$661.26M**，24h 交易量 **$216.62M**，市值排名 **#93**，流通 712.10M（略滞后于链上）。

**回购销毁政策（二手，非合约强制）**：
- 据报**协议费用份额的 80%** 用于从公开市场回购 PONS 并永久销毁
- 通过 **TWAP** 自动执行以降低市场冲击
- **⚠️ 分析师明确指出：这个 80% 是 Pons Labs, LLC 设定的「政策」，不是不可变的智能合约条款，团队随时可以调整。** [二手] fintrender / apex.exchange
- **本报告立场**：销毁量（29.70%）是链上硬事实；80% 的比例与 TWAP 执行是**运营承诺**，未在已公开源码中找到对应合约。**Mantle 若照抄，应把这条写进合约而非博客。**

**Uniswap Labs 于 2026-09-03/04 买入 PONS**（据报 100 万枚）"for long-term alignment" —— 发生在 Uniswap 自己的 pools.trade 与 Pons 竞争之后。[二手] The Defiant / Dealroom

### 3.7 Pons 的运营数据（DefiLlama，链上直取，2026-09-06）

**手续费与收入：**

| | 24h | 7d | 30d | **累计** |
|---|---|---|---|---|
| **Pons V1 手续费** | $296,944 | $2.59M | $7.94M | **$25.00M** |
| Pons V1 协议收入 | $59,970 | $546,057 | $1.80M | $6.39M |
| **Pons V2 手续费** | **$8,750,574** | $33.85M | $47.36M | **$47.48M** |
| Pons V2 协议收入 | $1,788,459 | $6.62M | $8.99M | $9.02M |
| **Pons 合计手续费** | **~$9.05M** | ~$36.44M | ~$55.30M | **$72.48M** |

**日手续费时间序列（美元）：**

*V1（2026-07-14 起）*：
```
07-14  159,750  ← 首日
07-15  683,026
07-21  1,535,814  ← V1 峰值
07-27  1,074,846
08-04    384,871
08-13    177,675  ← V2 分流后衰减
08-20     98,262  ← 谷底
09-05    296,944
```

*V2（2026-08-04 起）—— 这条曲线是整个报告的核心图景*：
```
08-04     31,868  ← 首日
08-14    517,025
08-24    349,655
08-25    983,188
08-27  2,089,382
08-29  3,460,512
08-30  4,704,070
09-01  4,221,588
09-02  5,702,128
09-03  6,085,253
09-04  8,750,574  ← 峰值，31 天从 $31.8K 涨到 $8.75M（275x）
```

**方法论说明（DefiLlama 官方口径）**：[链上]
- V1 Fees = *"1% swap fees paid on all token swaps of tokens launched on the platform (only pools with at least $200 in TVL are included) and **0.0005 $ETH per token launched**"*
- V2 Fees = *"launch fees, curve swap fees and swap fees post graduation (uniswap v4: realised through fee swept events)"*
- V2 SupplySideRevenue = *"swap fees to creators, taxes (optional) to creators and **buybacks (if enabled by creators) of meme tokens**"* → 印证 §3.5 的三段拆分，且**回购由创建者可选开启**

**其他指标**：
- 发币数：**>167,000 个代币**，**>52,000 个持币地址**（~2026-08-30）[二手]
- **毕业数与毕业率：未找到可靠数据源。⚠️ 这是本节最大的数据缺口** —— DefiLlama 不追踪，未找到公开 Dune 看板。
- 2026-08-31 单日占**全加密行业 launchpad 总手续费的 63.9%**（$4.89M vs pump.fun $1.72M）[二手] The Defiant

### 3.8 Pons 是否支持 stock token 作为 quote？

**合约层：技术上支持。** V2 的 `PairTokenEconomics` 是一个**通用机制**，任何 owner 批准的 ERC-20（最低 6 位小数）都可以做 `pairToken`，每种资产配自己的 `phantomQuote` 与 `graduationThreshold`。**它不是为股票代币专门设计的，只是资产无关（asset-agnostic）。** [一手]

**运营层：未证实 Pons 主打股票配对。** 报道中最清晰的股票配对案例（Artificial Inu/NVDA、BONER/HIMS、SPACEHOOD/SPCX）均归属于**另一个叫 LONG（long.xyz）的竞品**，而非 Pons。

> **结论：Pons ≈ Robinhood Chain 的 pump.fun（ETH 计价的通用 meme 工厂）；PAIR 与 LONG 才是「股票代币做 quote」这个赛道的专业玩家。** 这个区分极其重要，媒体常混为一谈。

---

## 4. PAIR / pair.fund（股票代币作 quote 的专业化 launchpad）

### 4.1 概览

| 项 | 值 |
|---|---|
| 官网 | **pair.fund**（X: @pairdotfund / @pairecosystem；GitHub: pairdotfund） |
| 定位标语 | **"Stop launching against ETH"** —— 直接对标 Pons 的 ETH 计价模型 |
| 创始人 | Tugg（@0xTugg），"PAIR Labs by Luxington" |
| 与 Pons 关系 | **不是 fork，不是同一团队**，是直接竞品 |
| 首次上线 | 2026-07-21（单配对，Uniswap v3） |
| **V5 multipool 上线** | **2026-08-26** |
| 公开发布 + AWS 基础设施合作 | 2026-08-31 |
| Launchpad 合约（proxy） | `0x8660A7F019C7943b0b0A91B8E39AFf3b6DB6Ae62`（`PairLaunchpadV5Upgradeable`）[二手，未做字节码验证] |
| $PAIR 代币 | `0x6b1d42927b1a84ec28fa88d4fc6fa7af404966be`（**已链上验证**） |

### 4.2 核心机制：multipool 原子发射

**一笔原子交易（`PairLaunchpadV5Upgradeable`）内完成：**

1. 部署新 ERC-20，**固定供应 1,000,000,000**，**无 mint 函数、无后续增发**
   - ✅ **本报告链上验证**：`totalSupply()` = 1,000,000,000 [链上]
2. 创建 **1–5 个 Uniswap v4 池**，每个池对应创建者从**白名单 24 只 Robinhood 股票代币**中选一只作 quote
   - 白名单：`AAPL, AMC, AMD, AMZN, BABA, BE, CRCL, CRWV, GOOGL, INTC, META, MSFT, MU, NVDA, ORCL, PLTR, QQQ, SGOV, SLV, SNDK, SPCX, SPY, TSLA, USAR`
   - **创建者自选权重，必须精确加总为 100%** —— 这是固定供应在多池间的分配方式
   - 例：权重 33/33/34 分给 AAPL/MSFT/NVDA，该代币行为上就是一个**三资产指数**
3. 每个池以**预言机导出的开盘价**注入**单边集中流动性**，**在发射交易内直接出资**（不经预售曲线）
4. 可选**付费 dev buy**：在选定的一个池内市价买入，代币直接给创建者钱包
5. **每个 LP 头寸永久锁进 `PairV4Locker`** —— 无创建者提取路径，永远
6. 代币注册上链，出现在 PAIR explorer/API

**"原子发射（atomic emission）"的准确含义**：代币 + N 个池 + 锁仓 + 配对元数据**要么全部落在同一个区块，要么整笔交易回滚**。**不存在"代币已存在但池未注资"或"池存在但头寸未锁"的中间态。**

- **发射费**：**0.0005 ETH**（与 Pons 相同），不到 1 分钟完成

### 4.3 无 bonding curve、无毕业迁移

- **完全没有 bonding curve。** 每个代币从第一笔 swap 起就在**真实的、已注资的 Uniswap v4 池**里交易 —— 即"即时永久流动性"模型（结构上接近 Pons **V1** 的思路，但注资方式与锁仓强度不同）。
- **"毕业"只是一个信息性链上标志位**：任何人可无许可触发，条件是锁仓本金超过 **4.2 ETH × ETH/USD**（**注意：PAIR 直接沿用了 Pons 的 4.2 ETH 作为文化符号**）。**翻转这个标志位不移动任何流动性。**
- **流动性连续投放到 Uniswap 极限可用 tick —— 没有价格天花板。**

### 4.4 Anti-snipe（与 Pons 的税模型截然不同）

**PAIR 用硬上限而非衰减税：**

| 参数 | 值 |
|---|---|
| 生效窗口 | 发射区块 + 之后 **5 个区块** |
| 单笔买入上限 | **5.5%** 供应量 |
| 单钱包持仓上限 | **5.0%** 供应量 |
| 卖出 | **永不受限** |
| 解除 | 自动，无需管理员操作 |
| 转账税 / 运营 bot | **无** |

> 5 个区块 @ 100ms = **0.5 秒**窗口。相比 Pons 的 15 秒 99% 衰减税，PAIR 的保护弱得多 —— 这是"追求上线即真实池"必须付出的代价。

### 4.5 多池套利与聚合路由

**`PairV5MultiPoolAggregator`** —— 无许可路由合约，AUTO 模式下：
- 为买/卖报价**每一条池腿**
- **拒绝实时价格冲击 >15% 的路由**
- **只在 gas 调整后的执行改善足够时**才拆单到多池
- 应用**每腿 + 总量**双重最小输出（滑点）保护
- 钱包签名前**立即重新模拟**
- 用 **USDG** 作为各腿共同的输入/输出资产

交易者也可绕过 AUTO，直接交易某一个具名 ticker 池（此时直接花费/收到该股票代币本身）。

**⚠️ 关键设计缺陷**：**聚合器不做池间再平衡，它只为单个交易者优化执行。** 篮子内各池的价格对齐**完全依赖外部套利资本**。而按 PAIR 自己的承认，**在股票市场休市期间，这种纠正可能长时间缺席。**

### 4.6 $PAIR 代币经济（**链上直读验证**）

| 项 | 值 | 来源 |
|---|---|---|
| 合约 | `0x6b1d42927b1a84ec28fa88d4fc6fa7af404966be` | [链上] |
| `totalSupply()` | **1,000,000,000** | [链上] |
| `balanceOf(0x…dEaD)` | **102,177,191.72** | [链上] |
| **已销毁比例** | **10.22%**（上线 8 天） | [链上] |
| 上线 | 2026-08-29，**在自家平台发射，配对 SPY** | [二手] |
| 价格 / 市值 | **$0.041914 / $37.70M**，24h 量 **$40.05M** | CoinGecko 2026-09-06 |
| 回购销毁政策 | **协议费用的 90% 回购销毁 $PAIR**；剩余 10% 用于创建者拓展、市场、基础设施 | [二手] |

**销毁记录（二手）**：上线首 24h 销毁 >$15,000；随后单笔 $80,000 销毁；再 $10,000。
→ 与链上 10.22% 的数字比对，说明销毁节奏比公告披露的快得多（链上为准）。

### 4.7 费用与运营数据（DefiLlama，2026-09-06）[链上]

**费用结构（DefiLlama 官方方法论原文）**：
> *"**1% pool swap fee on every trade**, accruing to permanently locked Uniswap V4 positions and periodically swept to creators and the protocol, plus a **0.0005 ETH launch fee** paid at each token deployment."*
> *"Revenue: **30% of collected swap fees** credited to the treasury claimable balance, plus the full 0.0005 ETH launch fee."*
> *"SupplySideRevenue: **70% of collected swap fees**..."*

→ **PAIR 与 Pons 的 1% / 70-30 完全一致**，但 PAIR 没有 buyback vault 那一层，创建者拿满 70%。

| | 24h | 7d | **累计** |
|---|---|---|---|
| 手续费 | $52,112 | $389,921 | **$433,275** |
| 协议收入 | $15,680 | $118,431 | **$131,643** |

**日手续费序列**：
```
08-27    3,416  ← 首日
08-28    9,689
08-29   30,249
08-30   75,811  ← AMC 上线
08-31   42,444
09-02   15,418
09-04  160,945  ← 峰值
09-05   52,112
```

**其他（2026-08-31 官方新闻稿口径，二手）**：
- 累计交易量：08-29 破 $6M → 08-30 破 $15M → 08-31 破 $26M
- **>160,000 笔交易**，**>1,200 个代币**（含所有版本）
- 创建者奖励累计发放 **>$180,000**
- 已"毕业"（信息性）10 个，代表作：**ABSOLUTE CINEMA**（AMC）、**Chips Party Pack**（NVDA+AMD+INTC+MU）、**PEAR**（AAPL+MSFT+NVDA）、**X Holdings**（SPCX+TSLA）

> **规模判读**：PAIR 累计手续费 $433K vs Pons $72.48M —— **PAIR 是 Pons 的 0.6%**。PAIR 的价值不在规模，在于它是**「股票代币作 quote」这个机制的最完整实现**，是 Mantle 最直接的抄袭对象。

### 4.8 实际被用作 quote 的股票代币

**链级数据（CoinGecko 统计口径，2026-06-29 ~ 07-27）**：[二手]

| 股票代币 | 配对 meme | 交易量 |
|---|---|---|
| **NVDA** | AI / CHIPS / JACKET / REAL / LONGSHOT / SWOGE | **$34.3M+**，占据前 25 大配对中的 8 席 |
| **GME** | GME / WSB / AMC 系 meme | $26.8M + $1.9M + $1.7M |
| **SPCX**（SpaceX） | MARSCOIN / SPACEHOOD / ASTEROID / MOON | $7.8M + $4.5M + $1.4M + $0.8M |
| **AAPL** | AP / AAPLCAT | $4.4M + $2.8M |
| **MSFT** | CLIPPY | $2.2M |
| **TSLA** | USEDTESLA | $1.6M |

**最成功的股票配对 meme：Artificial Inu (AI)，配对 NVDA**（在 **Long.xyz** 而非 PAIR 上）：市值从 ~$1.5M（08-01）→ ~$135M（08-30）→ 09-02 报 ~$275M，单池流动性 ~$26M，24h 量 >$11M。**其 NVDA 配对池的流动性（~$3.3M）是其 WETH 池的 3 倍以上。** [二手]

> **这条数据是整个报告最有说服力的一条**：交易者**主动选择**把流动性放在股票配对池而非 WETH 池，深度是 3 倍。**「股票代币作 quote」不是噱头，是被市场投票认可的产品形态。**

### 4.9 股票代币作 quote 的风险剖析

| 风险 | 机制 | 是否已实际发生 |
|---|---|---|
| **休市陈旧价（staleness）** | 每周 135+ 小时无股票侧参考价、无股票做市商 | ✅ HIMS 4.6x、AMC 35x |
| **周一跳空回补** | 开盘后链上价格向真实价暴力收敛，周末溢价买家爆亏，**协议层无任何熔断** | ✅ 隐含于上述事件 |
| **逼空/囤积浮筹** | 单个 meme 池可囤积某股票代币的大部分链上浮筹 | ✅ BONER 囤了 53% 的 HIMS |
| **公司行动脱钩** | 股票代币层用 `uiMultiplier` 处理，**但 PAIR 池层没有任何对应的再平衡逻辑** —— 拆股/大额分红可能让池价与每股经济脱节 | ⚠️ 未发生，**未解决的开放风险** |
| **发行人停摆** | 只有 BBVI 能 mint/redeem。若增发被暂停（AMC 争议期间几乎发生），股票代币将**完全脱钩且不可套利** | ⚠️ 未发生 |
| **无常损失** | meme 币与股票是**不相关资产**，两者背离时 LP（PAIR 自己是唯一锁仓 LP）承受剧烈 IL | ✅ 结构性存在 |
| **假冒标的** | Farmmi 案中 meme 配对的是一个**完全伪造的"代币化 FAMI"** —— 无发行人、无赎回、匿名铸造 | ✅ 已发生 |

**PAIR 已宣布但未完全上线的缓解措施（2026-09-01 新闻稿）**：
- **"peg guard"** —— 当某股票代币链上价偏离最后收盘价过大时，暂停对其新发射
- 链上风险标签：交易前显示实时溢价/折价
- 招募链上做市商提供周末覆盖；据报 Robinhood Crypto 追加周末流动性
- **⚠️ 目前不存在任何链上预言机偏离熔断器或自动停牌。** 官方策略是**「篮子分散 + 信息披露」，不是硬编码的安全机制。**

**PAIR 自己给出的多池设计理由**：单一仙股配对的代币"生死取决于那一只股票的流动性"；篮子里"没有任何单一配对能定其地板"，AMC 这类薄池的剧烈波动会被 NVDA/TSLA 这类深池"缓冲"。—— **注意：这是分散化，不是锚定机制。**

---

## 5. Robinhood Chain 上的其他应用

### 5.1 Noxa 的兴与亡（2026-06-30 → 07-16，**16 天**）

Robinhood Chain 上第一个统治级 launchpad，也是最快的崩塌案例：

| 日期 | 事件 |
|---|---|
| 2026-06-30 | 随主网上线（比 Robinhood 官宣早 1 天） |
| 06-30 ~ 07-11 | 拿下链上 **65.8% 的发币份额（~60,000 个代币）**；**连续 5 天日手续费超过 Solana 的 pump.fun** |
| 2026-07-11 | **暂停发币**，理由是 bot 刷量与山寨代币压垮基础设施 |
| 2026-07-13 | **官网下线**，归咎于 Cloudflare |
| 2026-07-14 | 重新出现，留言 *"the cat has been liberated"*，承诺未来 **100% 手续费归创建者** |
| 2026-07-16 | 域名注册商**查封并转卖其域名**，只剩一个 ENS 托管页面 |

- 旗舰代币 **CASHCAT**（取自 Robinhood 创始人最初的公司名）事后跌 >30%，**但 CASHCAT 本身活了下来** —— 2026-08-06 被 **Robinhood 主 App 直接上架现货交易**，CEO Vlad Tenev 关注了它的账号。
- Noxa 的残余业务（NOXA Fun）转战 Monad / MegaETH / Merlin / Stable / Intuition 等链，**累计手续费 $21.70M，24h 仍有 $96,904**。[链上 DefiLlama]

> **教训**：**先发优势在 launchpad 赛道价值极低。** Noxa 拿下 65.8% 份额后 11 天内归零，原因是**基础设施承压 + 团队运营失能**，而非机制不好。**能扛住 bot 洪水的工程能力比机制创新更重要。**

### 5.2 Uniswap：既是基础设施，也是竞争者

- **Uniswap v2/v3/v4/UniswapX 主网首日全量部署**，合计承接链上约 **99% 的 DEX 流动性**；股票代币中约 **73% 在 v4 / 26% 在 v3**。[二手]
- 2026-09-06 单日 **Uniswap V4 在 RH Chain 的交易量 $1,029.7M**（占全链 DEX 量的 ~67%）。[链上]
- **Uniswap Labs 自己下场做 launchpad：`pools.trade`，2026-08-05 上线** —— **发币零费用、每笔交易 0.25%**（低于 Pons/PAIR 的 1%）。**首日发币数就超过 Pons**，并使 Pons 当周下跌约 49%，随后 Pons 反弹。[二手]
- 2026-09-03/04：**Uniswap Labs 转而买入 PONS 代币**"for long-term alignment"。[二手] The Defiant

> 这条线索很有意思：**DEX 自己做 launchpad 的尝试没有压死专业 launchpad，最后选择了资本层面的结盟。** 说明 launchpad 的护城河不在 AMM，在**发行体验 + 社区 + 费用分配设计**。

### 5.3 借贷、永续与其他

| 类别 | 项目 | 状态 |
|---|---|---|
| **借贷** | **Morpho**（官方合作） | Robinhood Earn 的底层：美国用户可将 USDG 通过自托管钱包出借，**估算 7% APY**，经 **Lloyd's of London + RELM** 承保网络/合约攻击损失；风险策展方 Steakhouse、Ethena、Spark、Maple。[一手] RH Chain 上 Morpho Blue 累计手续费 $1.75M [链上] |
| **永续** | **Lighter**（`robinhoodchain.lighter.xyz`） | 已在 Robinhood Wallet 内原生集成；**Lighter 承诺 1,100 万 $LIT 给 Robinhood 社区**，通过 RH Wallet 交易得 2x 积分。TVL $64.66M [链上] |
| **永续** | **Arcus** | 官方生态名单 |
| **PropAMM** | **Rialto** | 做市商背书的链上流动性，专为股票代币薄流动性场景设计 —— 与 RFQ 的区别是**它在链上，因此可组合** |
| **自营 AMM** | **Pleiades** | 主网首日合作方，定位"primary prop trading venue" |
| **稳定币** | **USDG（Paxos）** | 链上稳定币市值 $964.56M，USDG 占 **66.4%** [链上] |
| **交易终端/bot** | **GMGN** | **RH Chain 上 24h 手续费 $1.65M、DEX 量 $641.7M** —— 单个 bot 前端就吃掉全链约 40% 的 DEX 量 [链上] |
| 其他 launchpad | **Long.xyz**（单股票配对专业户）、**o1 Exchange**、**LetsCash**、**Bags**（$635K 累计）、**Flap.sh**、**clanker** | [链上] |
| 其他 DeFi | **Delta**（流动性引导，类 Meteora）、**UP**（ve(3,3) 排放，类 Aerodrome）、**NetNet**（OHM 式债券，8 月市值一度 >$117M） | [二手] |

**股票代币作抵押品**：官方文档明确列为用例（*"Deposit NVDA as collateral on a lending market and borrow USDG"*），但**未找到一个以股票代币为主要抵押品的旗舰协议**。目前 DeFi 侧股票代币存款约 **$72.7M**。[未证实，单一二手来源]

### 5.4 代表性 stock-paired meme 案例：AMC 事件

见 §6.1。此处仅列产品事实：
- PAIR 于 **2026-08-30** 上线 AMC 作为首个"仙股"配对资产
- 代表代币 **ABSOLUTE CINEMA**（AMC 配对）
- **不存在"官方 AMC meme 币"** —— 争议对象是 (a) Robinhood 自己的代币化 AMC 股票代币，与 (b) 第三方 launchpad 上与之配对的、完全无关联的 meme 币

---

## 6. 监管与风险

### 6.1 AMC / Adam Aron 事件（2026-09-04 ~ 05）

**时间线与原话** [二手 Business Insider / crypto.news / The Defiant，均有 X 原帖链接]：

**AMC CEO Adam Aron（@ceoadam）公开长帖**：
> *"The list of concerns is almost existential."*
>
> *"In good conscience, how can Robinhood as a U.S. company set up an operation in **far offshore Jersey, an island 3000 miles away**, and market **a security sort of posing as AMC** in some shape or fashion, and not comply with U.S. securities laws. **That is shocking and shameful.**"*
>
> *"These are but a few of my concerns about your actions. **I hereby call on you and Robinhood to voluntarily CEASE AND DECIST the trading of AMC stock tokens.** If you don't, our high priced securities counsel has been asked to see whether we can force you to stop."*

他另称这些代币"contemptible"、"vile"，是一个 **"fictitious synthetic equity market"**，核心担忧是：
1. 无所有权、无投票权、剥夺股东保护
2. 未注册证券，规避 AMC 自己必须承担的美国合规成本
3. 脱钩的合成市场可能**损害 AMC 的融资能力**

**Robinhood 的回应 —— 极其强硬：**
- **首席法务官 Dan Gallagher（2011–2015 年 SEC 委员）** 在 X 上直接回击：
  > *"**We know a little something about the U.S. securities laws and will not 'DECIST.' Send your lawyers and we'll educate them.**"*
- 公司发言人：*"We stand firmly behind our Stock Tokens and their ability to provide international exposure to US equities, modernize the financial system and expand opportunities for ownership globally."*
- CEO Vlad Tenev：*"We stand behind Stock Tokens."*

**结果（截至 2026-09-06）**：
- **AMC 股价当日盘中涨 21%**（争议本身成了催化剂）
- AMC 已聘请外部证券律师，声称要向 SEC 提出，但**尚未向 SEC 提交任何文件**
- **无任何 SEC/FINRA 执法行动**
- 争议未解决，持续进行中

### 6.2 由此引发的"代币化模型之争"

AMC 事件把整个行业拖进了一场关于"哪种代币化股票模型才正当"的公开辩论 [二手 The Defiant 2026-09-05]：

| 发言人 | 立场 |
|---|---|
| **Gabriel Otte**（Dinari 联创） | *"...it's not about legality, it's just that **synthetic tokens like @RobinhoodApp stock tokens and @Ondo are just indisputably worse for the end investors than even common stocks**."* |
| **Anna Wroblewska**（Dinari CBO） | *"The main problem here isn't tokenization. It's the **marketing of a discretionary debt instrument, which functions essentially as an onchain CFD, as an investment in the US stock market**."* |
| **Hayden Adams**（Uniswap 创始人） | *"They're not worse if you want programmability, or to trade at night/weekend/holidays, live outside the US, don't have a bank account, want to use them in DeFi apps... **tokenized stocks are pretty similar to early stablecoins**."* |
| **Carlos Domingo**（Securitize CEO） | *"I would also not want people creating offshore derivatives of our stock that trade all over the place. This is why we tokenized our own stock natively and in the US, in a fully compliant way."* |
| **Ariel Givner**（金融科技/IP 律师） | *"**You bought economic exposure from an offshore affiliate that slapped someone else's ticker on a derivative. That is NOT tokenization.**"* |
| **Brian Huang**（Glider 联创，前 XTX 股票交易员） | *"**AMMs do not guarantee best execution for consumers.** In the US... you are guaranteed to get best execution [via NBBO rules]. Now, that is not true via AMMs."*<br>*"When you put in a meme coin with the stock, they're not really correlated assets. **You're exposing people to a lot of impermanent loss.**"* |
| **Binji Pande**（Ethlabs） | 称"六美元的代币化 AMC 报价"是 "bad market structure"；发行人"lost control over how their assets are used" |

**监管侧的书面输入**：
- **Continental Stock Transfer & Trust**（2026-07-21 提交 SEC Crypto Task Force）：第三方合成代币 *"do not establish a legal relationship between the token holder and the issuer"*，会 *"confuse investors, impair issuer governance, create disclosure and market-integrity risks, and **bypass the shareholder-record and corporate-action infrastructure**."*
- **Computershare**（2026-07-28）：立场相反，主张对各种账簿形态**中性对待**。

### 6.3 「股票代币作 quote」的合规争议

**美国法**：
- SEC 2026-01-28 Corp Fin 声明把 Robinhood 归入第三类 **"linked securities"**，并警示破产/交易对手风险。
- **AMM 池交易证券型代币是否构成未注册的证券交易所（Reg ATS）？**
  → **没有任何针对 Robinhood Chain / Pons / PAIR 的 SEC 执法、无异议函或法院裁决。** ⚠️
  → 律所（Morgan Lewis、Skadden、Fenwick、Chapman 等 2026 Q1 客户简报）的普遍看法：SEC 的"实质重于形式"原则意味着交易证券类代币的 AMM 池**可能触发 Reg ATS / 未注册交易所风险**；而提供"纯经济敞口而无所有权"的产品可能被定性为 **security-based swap**，在美国对零售分销有极重限制。
- Robinhood 的防线：**代币不向美国人发售**（Reg S），发行人在泽西岛，池子在无许可链上 —— 即**主体地理隔离 + 无许可基础设施**的组合。

**欧盟法**：
- **MiCA 明确排除**符合"金融工具"定义的代币 → 这类代币仍归 **MiFID II** 与既有证券法管辖，而非 MiCA。
- Robinhood 的股票代币由泽西岛（非欧盟成员国）发行、面向 EEA 用户销售 → **具体适用哪个交易场所授权制度（MiFID II 交易场所规则 vs 泽西岛制度）在公开来源中无定论**。⚠️
- 欧盟委员会正在评估是否扩大 MiCA 范围以更好覆盖代币化。[二手] The Block 2026-07-08

**⚠️ 明确的知识空白**：**没有任何具名监管机构就「Pons/PAIR 的 AMM 池交易 Robinhood 股票代币」适用何种制度作出过表态。** 这是整个赛道最大的悬空风险。

### 6.4 Robinhood 官方态度与可用的控制手段

**态度演变**：
- 2026-07-02 Tenev 对 CNBC：RWA 是加密的 *"durable direction"*
- 2026-07-08 Tenev 发帖：链 *"works great for memes too"* —— **CASHCAT 统治链上后的态度软化**
- Johann Kerbrat（Crypto SVP）内部框架：**"two wolves"** —— 受监管的 RWA 产品 vs 投机性 meme 活动 [二手 Decrypt]
- 官方文档免责：*"Inclusion on this page does not constitute an endorsement, partnership, affiliation, sponsorship, or warranty by Robinhood. Robinhood makes no representations regarding the safety, legitimacy, or suitability of any featured protocol."* [一手]

**Robinhood 实际拥有的控制杠杆（按强度排序）**：

| 杠杆 | 强度 | 说明 |
|---|---|---|
| **一级市场垄断（Bitstamp/BBVI 是唯一 AP）** | ★★★★★ | 可增发/赎回来纠偏（HIMS 事件中实际使用），也可**停止增发**使代币彻底脱钩 |
| **Sequencer 级 transaction screening** | ★★★★☆ | 官方承认存在，可排除制裁地址交易；ArbOS filtering 甚至可作废 L1 强制包含的交易 |
| **`pauseOracle()` / `updateMultiplier()`** | ★★★★☆ | 完全中心化的公司行动裁量权；喂价一停，所有依赖预言机的协议（借贷清算等）失灵 |
| **Security Council 8 席多签** | ★★★☆☆ | 可改协议参数，但需 6/8（常规，+7 天 timelock）或 7/8（紧急）—— **Robinhood 单方做不到** |
| **代币层黑白名单** | ☆ | **未发现**。二级市场转账是完全开放的 |

> **结论：Robinhood 有能力也有意愿在「资产层」干预（AP 增发、暂停喂价、停牌），但在「应用层」（第三方 AMM 池）事实上放任。** 这是一个**刻意的责任切割**：合规风险留在自己的产品面，投机风险外包给"无许可的第三方"。

### 6.5 系统性与运营风险清单

| 风险 | 现状 |
|---|---|
| **Gas 补贴悬崖** | Robinhood Wallet 内 gas 全额代付**至 2026-09-29 23:59 EST 到期**。当前所有活跃度数据都不是稳态 |
| **Blob 成本与依赖** | 单链占以太坊 blob 空间 28–45%；2026-09-04 出现 14 分钟批次提交中断，**Robinhood 未发布技术复盘** |
| **L2Beat Stage 0** | 2 个许可制验证者、无 sequencer 故障兜底、exit window "None" |
| **收入集中度** | 应用层手续费高度依赖单一协议（Pons 占 ~50%）与单一叙事（meme） |
| **Robinhood 自身经济学不透明** | CFO 仅称"每笔几个基点、约一半给 Arbitrum"，**无精确费率、无 GAAP 对账**。首次可能对账的窗口是 **2026-11-04 的 Q3 财报** |
| **发行人破产路径未经检验** | Security Agent Services AG 变卖股票 → 现金分配，**从未实战验证** |
| **无储备证明** | 未找到独立第三方 attestation 或 Chainlink PoR |

---

## 7. 数据总表与横向对比

### 7.1 Launchpad 手续费横向对比（DefiLlama，2026-09-06 链上直取）

| 协议 | 链 | 24h 手续费 | 7d | 30d | **累计手续费** | 累计协议收入 |
|---|---|---|---|---|---|---|
| **Pons V2** | Robinhood | **$8,750,574** | $33.85M | $47.36M | **$47.48M** | $9.02M |
| **Pons V1** | Robinhood | $296,944 | $2.59M | $7.94M | $25.00M | $6.39M |
| **Pons 合计** | Robinhood | **~$9.05M** | ~$36.4M | ~$55.3M | **$72.48M** | ~$15.40M |
| **Flap.sh** | BSC / X Layer / Monad / **Robinhood** | $2,884,424 | $11.52M | $20.02M | $34.88M | — |
| **pump.fun**（launchpad） | Solana | $679,806 | $9.45M | $47.04M | **$1,210.68M** | — |
| **Pump**（含 PumpSwap 全体系） | Solana 等 4 链 | — | — | — | — | **$1,286.93M** |
| **Bags** | Solana + **Robinhood** | $30,603 | $117,324 | $1.81M | $64.00M | $32.0M |
| **NOXA Fun** | Monad / MegaETH / **Robinhood** 等 6 链 | $96,904 | $1.12M | $3.62M | $21.70M | — |
| **four.meme** | BSC | $9,578 | $53,108 | $290,434 | $98.05M | $96.63M |
| **LaunchLab / LetsBonk** | Solana | $4,921 | $10,484 | $64,831 | $15.93M | — |
| **PAIR** | Robinhood | $52,112 | $389,921 | $433,275 | **$433,275** | $131,643 |
| **clanker** | Base 等（RH 链上仅 $2,202） | $2,202 (RH) | — | — | $90.82M（全链） | $15.15M |

**⚠️ 数据缺口**：Zora、Believe 的 DefiLlama slug 未能确认。

### 7.2 与 pump.fun 的核心对比（本报告最重要的一张表）

| 维度 | **Pons** | **pump.fun** |
|---|---|---|
| 上线 | 2026-07-14（V1）/ 2026-08-04（V2） | 2024-01 |
| 运行时长（至 2026-09-06） | **~54 天** | **~32 个月** |
| **累计手续费** | **$72.48M** | $1,210.68M |
| **日均手续费（生涯）** | **~$1.34M/天** | ~$1.24M/天 |
| **当前日手续费** | **$9.05M** | $0.68M |
| **当前倍数** | **13.3x pump.fun** | — |
| 曲线 | 恒定乘积 + phantom quote 虚拟储备 | 恒定乘积 + 虚拟储备 |
| 毕业目的地 | **Uniswap v4 + singleton hook，永久锁定（locker 无出口）** | PumpSwap |
| 毕业滑点/预言机 | **零**（曲线用未来池的 quote 资产计价） | 需要迁移 swap |
| 交易费 | **1%** | 1%（历史）→ 分级 |
| 费用分配 | **30% 协议 / 35% 创建者 / 35% 回购金库(5年归属)** | 协议为主，后加创建者分成 |
| 代币回购 | **$PONS：政策性 80% 收入 TWAP 回购销毁；链上已销毁 29.70%** | PUMP 回购销毁 |
| 出块时间 | **~100ms** | ~400ms（Solana slot） |
| Anti-snipe | **99% 起始税，15 秒衰减窗口，≤32 豁免地址** | 无原生衰减税 |
| 发币数 | >167,000（~08-30） | 数百万 |

> **判读**：Pons 在 54 天里做到了 pump.fun 生涯累计的 **6%**，但**当前日流水是它的 13 倍**。这既证明了「新链 + 100ms + 股票叙事」的爆发力，也提示了**极端的不可持续性** —— 这条曲线 31 天涨了 275 倍，且建立在**将于 2026-09-29 到期的 gas 补贴**之上。**任何基于当前数字的战略推演都必须做补贴退出后的压力测试。**

### 7.3 链级横向对比（2026-09-06）

| 链 | 24h DEX 量 | 30d DEX 量 | 24h 链 gas 费 |
|---|---|---|---|
| Solana | $1.915B | $62.08B | ~$613K（09-02） |
| **Robinhood Chain** | **$1.53–1.61B** | $24.12B | **~$2.9M**（峰值 $4.45M @09-02） |
| Base | $589.22M | $24.16B | — |
| BSC | — | — | $882,890 |
| Ethereum | ~$1.3B（09-01） | — | ~$304K（09-02） |

- Robinhood Chain 在 DEX 交易量上**全链第 2**，仅次于 Solana。
- 2026-09-02 单日 gas 费 **超过 Ethereum + Solana + Tron + BNB Chain 之和**。

### 7.4 Robinhood Chain 关键指标汇总（2026-09-06）

| 指标 | 值 |
|---|---|
| TVL | **$908.65M**（7 日 +29.7%） |
| 稳定币市值 | $964.56M（USDG 66.4%） |
| RWA 活跃市值 | $251.58M |
| 其中 Robinhood 股票代币 | **$133.2M**（全球代币化股票 $2.91B 的 4.6%，排名第 6） |
| 应用层 24h 手续费 / 收入 | $18.62M / $4.21M |
| 链 24h gas 费 | ~$2.9M |
| 24h DEX 量 / 7d | $1.53–1.61B / $11.30B |
| 24h 永续量 / 7d | $230.93M / $2.381B |
| 累计交易数（2 个月） | 576M+ |
| 地址数 | 12.3M（含大量 bot） |
| 平均出块 | **101–104 ms**（实测） |
| L2Beat TVS / Stage | $2.96B / **Stage 0** |

---

## 8. 提炼给 Mantle / mStocks 的结构性结论

> 本节为 §D 的收口，为后续「Mantle 可行方案」章节提供可直接施工的约束条件。

### 8.1 Robinhood 飞轮的真实因果链

```
自有 L2（100ms + FCFS + 合规过滤）
  → 可自由转账的 ERC-20 股票代币（ERC-8056 处理公司行动）
  → 股票代币成为 launchpad 的 quote 资产（NVDA 池深度 = WETH 池的 3 倍）
  → meme 投机带来爆炸性交易量与手续费
  → 交易量给股票代币带来它本不可能拥有的链上流动性与持有需求
  → 链的 gas 收入 + Robinhood 的 AP/交易收入
```

**关键点：Robinhood 并没有"设计"这个飞轮 —— 它设计了前两环，第三环是第三方（Pons/PAIR/LONG）自发涌现的。** 但正是第三环让这条链在 2 个月内做到全链 DEX 量第 2。

### 8.2 五条硬约束（Mantle 若要复制，缺一不可）

1. **mStocks 必须是可自由转账的标准 ERC-20，且一级市场可 KYB 准入、二级市场开放。**
   → 若走 bStocks 式封闭生态，第三环永远不会出现。
2. **必须用 ERC-8056 式的 `uiMultiplier` 处理分红/拆股，绝不能 rebase。**
   → 否则第一次分红就会把所有永久锁仓 LP 套利抽干。
3. **必须有一个能在偏离时增发/赎回的 AP。**
   → HIMS 事件中，唯一救回 4.6x 脱钩的机制就是 AP 增发 4,000 枚。这是**唯一有效的锚定手段**，比任何"peg guard"都重要。
4. **必须有 sequencer 级的合规过滤能力。**
   → 这是券商/发行方肯把证券放上链的前置条件。Arbitrum 已把它产品化（ArbOS Elara），Mantle 需自建。
5. **出块时间决定 anti-snipe 机制的可用形态。**
   → 100ms 下 15 秒 = 150 个区块，衰减税粒度细腻；2 秒出块下只有 7-8 个区块，只能退化成 PAIR 式的硬上限（保护弱得多）。

### 8.3 Robinhood 没解决、Mantle 可以做出差异化的三个空白

1. **周末/休市的价格纪律** —— 目前**零协议级熔断**。PAIR 的"peg guard"只是暂停新发射，不管存量池。
   → 可做：**预言机偏离熔断的 v4 hook** —— 当池价与最后收盘价偏离超过阈值时，自动收窄可交易区间或对该方向加征惩罚性费用。这是一个**尚无人实现的、可申请专利级的机制创新**。
2. **公司行动的池层适配** —— 股票代币层有 multiplier，但**池层没有任何再平衡逻辑**。拆股会让永久锁仓 LP 与真实经济脱节。
   → 可做：v4 hook 监听 `UIMultiplierUpdated` 事件，在 `effectiveAt` 时自动调整池的 tick 参照系。
3. **储备证明** —— Robinhood **没有**任何独立 attestation 或 Chainlink PoR。
   → Mantle/Bybit 若提供**链上可验证的 PoR**，是对 Robinhood "The Issuer is not regulated" 的直接降维打击。

### 8.4 必须避开的三个坑

1. **不要指望先发优势** —— Noxa 拿下 65.8% 份额后 16 天归零。**抗 bot 洪水的工程能力 > 机制创新。**
2. **不要用补贴堆数据** —— Robinhood Chain 全部指标都建立在 2026-09-29 到期的 gas 补贴上，稳态未知。
3. **不要低估「发行人愤怒」的政治风险** —— AMC 事件里 Robinhood 敢硬刚，是因为它有前 SEC 委员当 CLO、代币不向美国人发售、发行人在泽西岛。**Mantle/Bybit 若没有同等的法律隔离结构，第一次被上市公司 CEO 点名就会很被动。**

---

## 附录 A：本报告链上直读的原始数据（可复现）

RPC: `https://rpc.mainnet.chain.robinhood.com`，读取时间 2026-09-06

```
eth_blockNumber                              → 0x3547092 = 55,865,490
   ⇒ 自 2026-07-01 起 67 天 ⇒ 平均 103.6 ms/block

PONS  0x39dBED3a2bd333467115dE45665cC57F813C4571
  totalSupply()                              → 1,000,000,000.000
  balanceOf(0x000...dEaD)                    →   296,993,295.464   (29.70%)
  balanceOf(0x000...0000)                    →             0

PAIR  0x6b1d42927b1a84ec28fa88d4fc6fa7af404966be
  totalSupply()                              → 1,000,000,000.000
  balanceOf(0x000...dEaD)                    →   102,177,191.722   (10.22%)
  balanceOf(0x000...0000)                    →             0
```

官方合约地址（docs.robinhood.com/chain/contracts）：
```
WETH  0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73
USDG  0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168
```
（各股票代币地址由链上 asset registry 动态生成，官方警示：*"a token with a matching name/ticker but a different contract address is not a Robinhood Stock Token"*）

Launchpad 合约：
```
Pons V1 factory  0xA5aAb3F0c6EeadF30Ef1D3Eb997108E976351feB   [一手 GitHub]
Pons V2 factory  0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e   [一手 GitHub]
PAIR launchpad   0x8660A7F019C7943b0b0A91B8E39AFf3b6DB6Ae62   [二手，未验证字节码]
```

---

## 附录 B：主要信源

**一手**
- https://docs.robinhood.com/chain/ （概览、生态表）
- https://docs.robinhood.com/chain/connecting/ （网络配置、blob DA、RPC）
- https://docs.robinhood.com/chain/differences-from-ethereum/ （**transaction screening、FCFS 排序**）
- https://docs.robinhood.com/chain/governance/ （**Security Council 8 席、BoLD 2 验证者**）
- https://docs.robinhood.com/chain/stock-tokens/ ; /building-with-stock-tokens/ ; /oracles-and-price-feeds/ ; /contracts/
- https://docs.robinhood.com/rhj/ ; /product/ ; /service-providers/ ; /faq/ ; /corporate-actions/ ; /restricted-jurisdictions/
- https://robinhood.com/us/en/newsroom/robinhood-accelerates-global-expansion-robinhood-chain-mainnet-stock-tokens-agentic-trading/
- https://blog.arbitrum.io/robinhood-chain-mainnet/
- https://forum.arbitrum.foundation/t/arbitrumdao-factsheet-robinhood-chain-mainnet-launch/31041 （**AEP 8%+2%**）
- https://github.com/ponsdotdev/ponsfamily （**Pons V1/V2 全部源码与设计笔记**）
- https://eips.ethereum.org/EIPS/eip-8056 （Scaled UI Amount Extension）
- https://www.sec.gov/newsroom/speeches-statements/corp-fin-statement-tokenized-securities-012826-statement-tokenized-securities （**SEC 三分类框架**）
- DefiLlama API（`api.llama.fi/summary/fees/*`、`overview/fees/Robinhood Chain`、`overview/dexs/Robinhood Chain`）

**二手**
- The Defiant：Pons vs pump.fun、gas 费 82 倍、blob 中断取证、BONER/HIMS、Money Mushroom/Farmmi、AMC 模型之争、Uniswap 买入 PONS
- Business Insider（2026-09-04，Max Adams）：Adam Aron 原话
- crypto.news（2026-09-05）：AMC 股价 +21%
- CryptoSlate：Robinhood 实际收入不透明、2026-09-04 事故与企业反弹
- TrustSwap（2026-07-15）：CASHCAT 接管全链
- Decrypt：Kerbrat "two wolves"
- rwa.xyz/stocks、L2Beat、CoinGecko、arbdata.com/ecosystems/robinhood（Entropy Advisors）

**已明确标注的数据缺口**
1. Pons 的毕业数与毕业率 —— 无可靠来源
2. Pons anti-snipe 衰减曲线形状、是否仅对买单、税款去向 —— 源码中未定位，二手说法（5 秒）与合约默认值（15 秒）冲突
3. RHJ 股票代币 ERC-20 字节码中是否存在未公开的 freeze/blacklist 函数
4. Robinhood 从链上获得的 GAAP 口径收入 —— 公司未披露，最早 2026-11-04 Q3 财报
5. 独立储备证明 / Chainlink PoR —— 疑似不存在
6. 具名监管机构对「AMM 池交易股票代币」适用制度的表态 —— 不存在
