# 第三部分：Robinhood Chain —— 股票代币做 quote 资产的第一次大规模实验

> 数据截止 **2026-09-06**。本部分是全报告的核心参照系，因为它是目前唯一一个「代币化股票 × meme launchpad」的规模化生产案例。
> 详细底稿见 [`research/D-robinhood.md`](../research/D-robinhood.md)（1,116 行，含链上直读原始数据）。

---

## 3.0 为什么这一节最重要

Robinhood Chain 是 2026 年最反直觉的案例：

> **一家美国上市券商（NASDAQ: HOOD）为「代币化美股」造了一条 L2，结果这条链 90% 以上的经济活动来自 meme 币投机 —— 而这些 meme 币的报价资产（quote asset），正是它自己发行的美股代币。**

三个月内发生的事：

| 维度 | 数值 |
|---|---|
| 主网上线 | 2026-07-01（伦敦 "The World is Flat" 发布会） |
| 应用层累计手续费（2 个月） | **$386.79M** |
| 24h DEX 交易量 | **$1.53–1.61B（全链第 2，仅次于 Solana）** |
| 链上代币化股票市值 | $133.2M（全球代币化股票 $2.91B 的 4.6%，**仅排第 6**） |
| Pons 单日手续费峰值（2026-09-04） | **$8.75M**（同期 pump.fun $0.68M，**13.3 倍**） |

**关键张力**：Robinhood 想要 RWA 结算层，得到的是 meme 赌场；但恰恰是 meme 赌场给美股代币带来了它本不可能拥有的**链上流动性与持有需求**。

这就是要移植给 Mantle 的那个结构。

---

## 3.1 链本身：一条「使用无许可、验证与审查有许可」的链

### 3.1.1 技术栈

| 项 | 值 | 来源 |
|---|---|---|
| 架构 | Arbitrum Orbit / Nitro（2026 官方品牌 "Arbitrum Dedicated Blockchains"） | 一手 docs.robinhood.com/chain |
| 结算层 | **Ethereum L1**（不是 Arbitrum One，故触发 AEP 许可条款） | 一手 Arbitrum DAO Factsheet |
| 核心技术方 | **Offchain Labs** | 一手 |
| chainID | **4663**（测试网 46630） | 一手 |
| 原生 gas token | **ETH**（无自有 gas 代币） | 一手 |
| DA | **Ethereum blobs**（标准 Rollup，非 AnyTrust，**无 DAC**） | 一手 |
| 测试网 | 2026-02-10 上线，主网前累计处理 **>2 亿笔**交易 | 一手 |

**"launch-and-migrate" 范式**：Arbitrum 官方把 Robinhood 树为样板 —— 2025-06 先在 **Arbitrum One** 上线第一代 Classic Stock Tokens 验证需求，一年后才迁到专属链。

> **这一条对 Mantle 有直接可比性**：先在共享链上验证资产需求，再决定是否需要专属执行环境。Mantle 已经跳过了这一步（自己就是 L2），但对应的问题变成了：**Mantle 是否需要为 launchpad 做一个隔离的执行域？**（见第五部分 5.2）

### 3.1.2 ~100ms 出块是真实的

- Arbitrum 官方表述："Configurable block times and preconfirmations to achieve 100ms latency."
- **独立验证**：2026-09-06 读取 `eth_blockNumber` = `0x3547092` = **55,865,490**。自 2026-07-01 起约 67 天 ≈ 5,788,800 秒 → **平均 103.6 ms/block**。
- The Defiant 在 2026-09-04 的事故取证中测得 3 小时内 106,756 个区块 = **101 ms/block**。

**结论：100ms 不是营销数字。** 但 preconfirmation 的具体协议规格（谁签承诺、违约如何罚没）**没有任何公开技术文档** —— 目前只有营销层描述。

> **对 Mantle 的含义（关键）**：100ms 出块在 EVM Rollup 上已被工程验证。Mantle 的 **2 秒出块不是"慢一点"的问题，而是让某一整类机制设计失效**。具体见 3.3.4 的 anti-snipe 分析。

### 3.1.3 blob 依赖是真金白银的代价

- Robinhood Chain 据报是**全体 L2 中最大的以太坊 blob 消费者**：平静期占全网 blob 的 **45%**，2026-09-04 拥堵期占 **28%**，单链超过 Base + Arbitrum One 之和。
- **2026-09-04 事故**：批次提交地址两次静默共 **14 分钟**没有向 L1 sequencer inbox 提交 blob。链本身没停（区块仍以 101ms 产出），停的是 **L1 数据可用性与提款能力**。Arbitrum 归因"blob 市场拥堵"，但第一段 8m36s 发生时以太坊尚有 263 个空闲 blob slot 且价格低 —— **官方解释与数据不符，至今无技术复盘**。

### 3.1.4 费用：被 meme 打爆的教科书案例

| 日期 | 链上 gas 费/日 |
|---|---|
| 2026-08-22 | **~$54,254** |
| 2026-09-02 | **~$4.45M** |

**11 天涨 82 倍。** 2026-09-02 单日超过 Ethereum($304K) + Solana($613K) + Tron($874K) + BNB Chain($480K) **之和**。普通用户单笔成本从 <$0.01 涨到 **~$0.32–0.40**。

**这引发了 2026 年最重要的一场链设计辩论**（详见第五部分 5.2.1）：
- **Yakovenko（Solana）**：这个费用模型 "brain-dead" —— 链不该靠拥堵赚钱。
- **Goldfeder（Offchain Labs）**：Orbit 让应用方从"租户"变"房东"，保留约 90% gas 收入；建在 Solana 上则费用全给验证者。

**Gas 补贴**：Robinhood 在自家 Wallet 内对 >$0.50 的 swap **全额代付 gas，无上限，至 2026-09-29 23:59 EST 截止**。
→ ⚠️ **当前所有链上活跃度数据都不是稳态。补贴退出是这条链 2026 Q4 最大的单一变量。**

### 3.1.5 排序规则：FCFS，没有优先费，没有 Timeboost —— 这是最值得抄的一点

官方 `differences-from-ethereum` 页原文：

> *"Robinhood Chain employs a **first-come, first-served model based on sequencer arrival time. Priority gas auctions do not exist here**; consequently, increasing your fee will not shift your transaction ahead of others already in the queue."*

- **Arbitrum Timeboost / express lane：未启用。**
- 因此不存在 express-lane 拍卖收入需要分配；MEV 模型退化为纯粹的**到达时间竞速**（延迟战争，而非出价战争）。
- Sequencer 由 Robinhood 独家运营；L2Beat 对 "Sequencer failure" 标注 **"No mechanism"**。

**⚠️ 关键合规钩子：sequencer 级 Transaction Screening**

官方文档明文承认：

> *"Robinhood Chain maintains compliance standards through **sequencer-level screening**... **any transaction associated with a sanctioned address will be excluded from inclusion**... Since a blocked transfer is never processed, it simply appears as though the event never occurred, ensuring indexers remain synchronized with the actual state."*

L2Beat 的 Discovery 进一步指出：ArbOS 61 引入 `ArbFilteredTransactionsManager` 预编译（`0x…0074`），授权的 "filterer" 角色可登记任意交易哈希使其在状态转换中失败 —— **包括从 L1 强制包含进来的交易，且无延迟窗口**。Arbitrum 2026-08 的 ArbOS "Elara" 版本标题即为 *"Compliance Filtering, Priority Fee Support"*。

> **这意味着**："permissionless" 在 Robinhood Chain 上只适用于**合约部署与调用**，不适用于**抗审查**。
> **合约层是开放的，交易层是可被过滤的。**
> 这是"合规链"的真实形态，也是它能同时容纳美股代币和 meme 币的**制度前提**。
> **→ Mantle 若要把 mStocks 放上无许可 launchpad，必须先自建这个能力。这是第五部分列出的 P0 项之一。**

### 3.1.6 治理与验证

**Security Council：8 席多签** —— Robinhood 2 席，BitGo / Chainlink Labs / Fireblocks / Offchain Labs / Paxos / Talos 各 1 席。
- 常规：**6/8** + **7 天链上 timelock**
- 紧急：**7/8**，绕过 timelock
- → Robinhood 单方**无法**修改协议参数。

**验证者**：用 **BoLD** 争议解决，但只有 **2 个许可制验证者**（Offchain Labs、Alchemy）。

**L2Beat：Stage 0**，TVS ~$2.96B。

> 一句话：**链的使用是无许可的；链的验证与审查是许可的。** 媒体几乎全部把这两句混为一谈。

### 3.1.7 AEP 10% 分成

- 净协议收入的 **10%** → **8% Arbitrum DAO 国库 + 2% Arbitrum Developer Guild**，经 AEP fee router 流入。
- Robinhood CFO Shiv Verma 在 2026 Q2 财报会称每笔交易赚 "a few basis points"，**约一半分给 Arbitrum** —— 无精确费率、无 GAAP 对账。

> **对 Mantle 的含义**：Mantle 自有 L2，**无 AEP 抽成 —— 这是 10% 的纯毛利优势**。
> 但代价是 Arbitrum 生态提供的 Offchain Labs 工程支持、BoLD、compliance filtering 这些开箱即用件，Mantle 得自己补齐。

### 3.1.8 上线以来的数据

**应用层日手续费**（DefiLlama）：

| 日期 | 日手续费 | 备注 |
|---|---|---|
| 2026-07-01 | $2,146 | 主网日 |
| 2026-07-14 | ~$160K | Pons V1 首日 |
| 2026-07-29 | $3.25M | V1 高峰 |
| 2026-08-22 | ~$2.20M | meme 大潮前夜 |
| 2026-09-01 | $16.98M | |
| **2026-09-04** | **$24.51M** | **历史峰值** |
| 2026-09-06 | $9.62M | 周末回落 |

**汇总（2026-09-06）**：应用层手续费 24h **$18.62M** / 累计 **$386.79M**；DEX 量 24h **$1.53–1.61B** / 累计 **$45.96B**；TVL **$908.65M**（7 日 +29.7%）。

**24h DEX 量按协议拆分** —— 这张表对 Mantle 有直接施工意义：

| 协议 | 24h 量 |
|---|---|
| **Uniswap V4** | **$1,029.7M** |
| **GMGN（交易 bot/终端）** | **$641.7M** |
| Pons V2 | $161.4M |
| Ramses CL V2 | $77.1M |
| up v3 | $76.5M |

> 两条推论：
> ① **launchpad 的毕业目的地决定了链上 DEX 格局** —— Pons/PAIR 毕业后全进 v4，于是 v4 吃掉 67%。
> ② **单个交易 bot 前端（GMGN）吃掉全链约 40% 的 DEX 量。** 这是"执行层基础设施"价值的最硬证据 —— 也正是 Mantle 完全缺失的那一层（见第四部分 4.3）。

---

## 3.2 Robinhood Stock Tokens：可组合性是怎么被设计出来的

### 3.2.1 两代产品，法律结构完全不同（媒体几乎全部混淆）

| | **Classic Stock Tokens**（旧） | **Stock Tokens**（新，RH Chain 上） |
|---|---|---|
| 发行/对手方 | Robinhood Europe, UAB（立陶宛） | **Robinhood Assets (Jersey) Limited（"RHJ"）** |
| 法律形态 | **OTC 衍生品合约**，MiFID II 金融工具，PRIIP KID 风险等级 **7/7** | **代币化债务证券**，Base Prospectus + Final Terms 结构化票据 |
| 代币角色 | 权利来自与 RHEU 的双边合同，代币只是"表示" | **ERC-20 代币本身就是该证券** |
| 转账 | **不支持转出外部钱包** —— 事实上是封闭账本 | **自托管、可自由转账的标准 ERC-20** |
| 上线 | 2025-06，Arbitrum One | 2026-07-01，Robinhood Chain |

> **→ 只有第二代才具备 DeFi 可组合性。第一代不能做 quote 资产，第二代才能。**
> **这是 Pons / PAIR / LONG 得以存在的唯一前提。**

### 3.2.2 发行结构：一个刻意设计的责任切割

| 角色 | 实体 | 地点 |
|---|---|---|
| **发行人** | Robinhood Assets (Jersey) Limited（注册号 162428，LEI 984500ADFHQZ9D6B9A29） | 泽西岛 |
| **唯一 Authorised Participant** | **Bitstamp Global Ltd（"BBVI"）** | 英属维尔京群岛 |
| **Broker & Custodian** | **Alpaca Securities LLC** | 纽约 |
| **Paying Account Provider** | JPMorgan Chase Bank, N.A., London Branch | 伦敦 |
| **Security Agent & Verification Agent** | Security Agent Services AG | 瑞士楚格 |

**官方原文（极其重要）**：

> *"**The Issuer is not regulated.** However, in connection with the issuance of Stock Tokens, the Issuer has obtained certain consents in Jersey. Such consents do not constitute prudential supervision of the Issuer or an endorsement of its products."*

- 招股书由**列支敦士登金融市场管理局（FMA）**批准（EU Prospectus Regulation 主管机构）。
- 权利限定：*"provide economic exposure... but **do not grant investors any legal or beneficial rights in, or against the issuer of, those underlying securities**."* → **无投票权、无股东权利、无对标的公司的任何请求权。**
- **1:1 背书**：*"Every single Stock Token in circulation is backed 1:1 by the corresponding underlying equity."*
- **破产处置**：独立担保代理人变卖标的股票，**现金清偿，不交付实股**。
- **投资者费率**：申购 0.00%；赎回 **90 天内 0.00%，90 天后 0.05%**。
- **法定转让确认数：1 个区块** —— 即链上 1 个 100ms 区块即构成法律上的证券过户。**这是极为激进的法律设计。**
- **储备证明**：**未找到任何独立第三方 attestation 或 Chainlink Proof of Reserve**。⚠️

### 3.2.3 ERC-8056 multiplier —— 本节最重要的技术创新，Mantle 必须照抄

**问题**：股票有分红、拆股、并股。代币若做 rebase（改 `balanceOf`），所有 AMM 池、借贷协议、会计系统都会炸。若不处理，代币会永久偏离标的。

**Robinhood 的解法：ERC-8056（Scaled UI Amount Extension）**

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
1. **`balanceOf()` 与 `totalSupply()` 永远不变** —— 股票代币**不是** rebasing token。
2. 每代币代表的标的股数 = `raw amount × uiMultiplier / 1e18`；上线时 `uiMultiplier = 1e18`。
3. **分红处理**：不派现金。公司派息 → 自动再投资买更多股票 → **`uiMultiplier` 上调**。
4. **拆股处理**：10:1 拆股 → multiplier 1.0 → 10.0。
5. **AMM 完全不受影响**：*"Onchain swaps remain unaffected"* —— raw balance 不动，池子里的数量不变。
6. **预言机自动吸收 multiplier**：Chainlink feed 返回**每代币价格 = 标的股价 × multiplier**，集成方不要自己再乘一次。

**推论**：股票代币价格会**持续高于**标的股价（分红被再投资），跟踪的是**总回报（total return）**而非价格回报。

> ### ⭐ 对 Mantle mStocks 的第一性约束
> ERC-8056 是目前**唯一被大规模生产验证的**「让证券型代币在 AMM 里正常工作」的方案。
> **若 mStocks 采用 rebase 或"发新代币"处理公司行动，任何 launchpad 的永久锁仓 LP 都会在第一次分红时被套利抽干。**
> 这不是优化项，是先决条件。

### 3.2.4 Chainlink 喂价与休市：结构性的脆弱点

- 用的是 **Chainlink Data Feeds**（推送式 `latestRoundData()`），**不是** Data Streams。Chainlink 单开了一个品类叫 **"Tokenized Equity Feeds"**。
- 定价公式：**Token Price = Underlying Equity Market Price × Multiplier**，标的价来自 Chainlink 的 **24/5** 股票喂价。
- **休市行为**：官方 —— *"Stock feeds update **24/5**, following market hours."*；Chainlink 侧 —— *"When underlying equity markets are closed... **the feed may hold the last published price**... **These feeds do not have heartbeats during off-hours**."*
- **公司行动期间暂停喂价**：代币暴露 `oraclePaused()`。RHJ 流程 `pauseOracle()` → `updateMultiplier()` → `unpauseOracle()`。
  ⚠️ 官方提醒：*"The flag is **advisory and not enforced on-chain**, so a paused oracle may still return a value — keep your staleness check as the primary guard."*
- **Chainlink 明确免责**：*"Chainlink does not provide corporate-action calendar data or automated pause triggers; pause timing and multiplier updates are coordinated by Robinhood."* → **所有公司行动判断权 100% 中心化在 Robinhood。**

### 3.2.5 24/7 vs 休市：三起已经发生的真实事故

结构性事实：
1. 美股每周交易 ~32.5 小时；链上交易 168 小时。**每周有 135+ 小时没有股票侧参考价、没有股票做市商套利。**
2. 链上价格的**唯一再锚定机制**是 AP（Bitstamp/BBVI）的 mint/redeem —— **自由裁量的、非自动化的**。
3. Chainlink feed 在休市时**冻结**，AMM 却继续成交 → 报价与预言机可任意背离。

| 事件 | 日期 | 详情 |
|---|---|---|
| **HIMS / BONER 逼空** | 2026-08-29 ~ 09-01 周末 | meme 币 **BONER** 在单个池里囤积了 **53% 的代币化 HIMS 全部流通量**（31,198 / 58,714）。NYSE 休市期间该池把 HIMS 打到 **$132.64**，周五真实收盘 **$28.84**（**>4.6 倍背离**）。最终靠唯一 AP **BBVI 增发约 4,000 枚 HIMS 代币**才拉回 $30.10 |
| **AMC 池 35x** | 2026-08-30 周末 | PAIR 上线 AMC 作为首个"仙股"配对资产后，一个 AMC 配对代币在无股票做市商的周末冲到最后参考价的 **~35 倍** |
| **Farmmi / JINQIAN 反向传导** | 2026-09-02 | 从 Farmmi 自己的 SEC 20-F 里抠出的词做成 meme 币，配对到一个**假冒的、无发行人的"代币化 Farmmi"合约**（匿名钱包一次性铸造、自留 38%、自写 `PoolRepricer`）。假币冲到 $1.83，**而真实纳斯达克上市的 FAMI 当日盘中暴涨 321%**（$0.1187 → $0.50）—— **链上投机疑似反向传导到真实市场** |

**Robinhood 的处理工具**：公司行动期间暂停交易（生效日 ~2AM CET 停到美股开盘前 ~3:30PM CET）、取消未成交订单、退市时只允许卖出/赎回、Price Deviations 公示页。

> ⚠️ **但以上工具全部作用于 Robinhood 自己的产品面（app / RFQ / 一级市场），对第三方 AMM 池毫无约束力。**
> Uniswap v4 上一个 AMC/meme 池不会因为 AMC 停牌而停摆。
> **这是监管套利与系统性风险的交汇点，也是 Mantle 最大的差异化机会（见第六部分 6.4）。**

### 3.2.6 可用地区与技术限制

- 覆盖 **120+ 国家**，**190+ 只股票代币与 ETF**。
- **明确排除**：**美国（及所有 U.S. Persons，Reg S 定义）、加拿大、英国、瑞士** + 11 个制裁法域。
- **技术层面（关键）**：合约是**标准 ERC-20，无 ERC-3643 式转账钩子，未发现链上白/黑名单函数** → **限制发生在 KYC（app 层）+ 一级市场 KYB（仅 Bitstamp 可 mint/redeem），而非代币层。二级市场转账完全开放。**
- Chainlink 文档承认这一开放性：*"Primary mint and redeem can be permissioned, but **once the tokens are onchain, anyone can hold them and anyone can liquidate positions that use the feed**."*

> ### ⭐ 这条决定了 Mantle 版方案的可行性
> **"一级市场 KYB 闸门 + 二级市场完全开放"是整个飞轮的技术前提。**
> 好消息：Mantle 上的 xStocks 同样是**可自由转账的标准 ERC-20**（Backed 发行，ERC-20 on Mantle/Ethereum + SPL on Solana）。
> **→ 结构上，Mantle 版 PAIR 是可行的，不需要额外的合规包装层。**
> （⚠️ 仍需与 Backed 核验 Mantle 侧具体 mint 配置是否启用了 transfer hook。）

### 3.2.7 与 xStocks / bStocks / Ondo / Dinari 的结构对比

**SEC 2026-01-28 Corp Fin 声明**把代币化股票分为三类（当下最权威的分类框架）：
1. **Issuer-sponsored** —— 代币就是证券本身，发行人/过户代理把股东名册上链（Securitize SECZ、Superstate Opening Bell）
2. **Third-party custodial** —— 通过 security entitlement 对标的证券的间接权益（Dinari dShares、Ondo Global Markets）
3. **"Linked securities"** —— 第三方自行发行、提供**合成敞口**、**不是标的发行人的义务、不赋予任何来自标的发行人的权利**
   → **Robinhood Stock Tokens、xStocks、bStocks 全属第三类。**

| 维度 | **Robinhood（RHJ）** | **xStocks（Backed）** | **bStocks（Binance/BTech）** | **Ondo GM** | **Dinari dShares** |
|---|---|---|---|---|---|
| 法律包装 | 泽西岛代币化债务证券，FMA Liechtenstein 批准 | 瑞士 DLT 法 tracker certificate | BEP-20，ADGM/FSRA 批准招股书 | 离岸 BVI + 美国注册过户代理 | SEC 注册、FINRA 会员券商持实股 |
| 发行人受监管？ | **"The Issuer is not regulated"** | 受瑞士/列支敦士登框架 | BTech（币安关联方，**自营**） | 部分 | 是 |
| 股东权利 | 无（分红经 multiplier 再投资） | 无 | 无 | 离岸无 / 美国版有投票权 | **有** |
| 托管 | Alpaca Securities LLC | InCore Bank / Maerki Baumann | 未具名 | 美国注册券商 | Dinari Securities LLC |
| 可转让性 | **标准 ERC-20，完全自由** | **可自由转账，多链** | BNB Chain，可自托管 | 多链 | 多链，偏合规托管 |
| DeFi 可组合性 | **极强** | 强 | 中 | 中 | 弱 |
| 公司行动 | **ERC-8056 multiplier** | 通常调整数量或价格 | 未证实 | 未证实 | 直通实股 |
| **代币化股票市值**（2026-09-05） | **$133.2M** | **$633.7M** | **$659.4M** | **$869.6M** | $11.2M |

全球代币化股票总规模 **$2.91B**（2026-09-05，rwa.xyz），30 天 +14.4%，267 万持有人。

> ### ⭐ 三条最关键的推论
>
> **① Robinhood 的代币化股票市值只排第 6，但链上交易量与 DeFi 组合度第 1。**
> 它赢在**开放的二级市场 + ERC-8056 + 自有链**，输在**监管包装最薄**。
>
> **② xStocks（$633.7M）实际比 Robinhood（$133.2M）大 4.8 倍 —— Mantle 手里的资产盘子并不小。**
> 缺的不是资产，是**围绕资产的投机层**。
>
> **③ bStocks 的对照最刺痛：**
> xStocks 2025-11 上线 Mantle，bStocks 2026-06-11 才上线 BNB Chain，**晚了 7 个月**，却在 **7 周内做到 $500M AUM、46+ 标的、代币化股票市场份额 45–50%**。
> 差异在两点：**(a) 币安自营发行（BTech + 自有招股书），扩标的不受制于人；(b) 直接进币安主站 + BNB Chain 通用 DeFi，而 xStocks 需要用户从 Bybit 提到 Mantle 再去 Fluxion —— 多了两跳。**

---

## 3.3 Pons：工程级拆解（这是全报告最值得逐行抄的一节）

### 3.3.1 概览

| 项 | 值 |
|---|---|
| 官网 / 源码 | ponsfamily.com；**github.com/ponsdotdev/ponsfamily（MIT，开源）** |
| 运营方 | **Pons Labs, LLC**（匿名团队），**无披露融资** |
| 与 Robinhood 关系 | **完全无关**。Robinhood 官方声明第三方应用"不构成背书、合作或担保" |
| V1 factory | `0xA5aAb3F0c6EeadF30Ef1D3Eb997108E976351feB` |
| V2 factory | `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e` |
| $PONS | `0x39dBED3a2bd333467115dE45665cC57F813C4571` |
| V1 首笔手续费 | **2026-07-14** |
| V2 首笔手续费 | **2026-08-04** |

**两代同时在线**，V1 仍在产生手续费。

### 3.3.2 V1：单边集中流动性模拟 bonding curve（无曲线）

单笔交易 `launchToken()` 完成全部步骤：

1. **CREATE2** 部署固定供应 ERC-20，**全部供应铸给 factory**（地址可预测，带 vanity 后缀 `...bbbb`）
2. 初始化 Uniswap **V3** 池
3. 铸造**单边集中流动性头寸** —— **只放 meme 币，不放任何 quote 资产**，从配置的 `initialTick` 起始
   → **这是 V1 的精髓：创建者零成本注入流动性**，价格从最低 tick 单向上行
4. 头寸 NFT 锁进 locker（V1 的 locker 是 *configurable* —— 比 V2 弱）
5. 可选 **atomic dev-buy**

**V1 的 anti-snipe**：只有硬上限（同区块买入封锁、单钱包上限、累计买入上限、限制区块窗口），**无衰减税**。
**V1 的"毕业"**：`graduationStatus()` 只是 **view 函数**，**不触发任何迁移**。

### 3.3.3 V2：bonding curve → Uniswap V4，**毕业时零滑点、零预言机、零 MEV 窗口**

**核心设计哲学（README 原文）**：

> *"The curve **trades in the same quote asset its future V4 pool will use**... Because the curve collects the eventual pool asset from the very first trade, **graduation seeds the pool directly — no router, no swap, and no price oracle anywhere in the system.**"*

> ⭐ **这是对 pump.fun 模型的一个真正的工程改进。** pump.fun 系的迁移需要 swap/router，会产生可被抢跑的窗口；Pons V2 消除了它。

**曲线数学**（`PonsV2BondingCurveMath.sol`）：**恒定乘积（x·y=k），输入端收费**

```
amountOut = amountIn·(10000 − feeBps)·reserveOut
            ─────────────────────────────────────────
            reserveIn·10000 + amountIn·(10000 − feeBps)
```

改编自已审计的 `BootstrapPool.sol`（code-423n4/2025-01-iq-ai）。**属 pump.fun 的虚拟储备恒定乘积家族，不是指数/线性/Bancor。**

**关键状态量**：
- `phantomQuote` = **虚拟 quote 储备**，决定开盘价（每次 launch 可配置）
- `graduationThreshold` = 真实 quote 储备目标
- **预留代币公式（保证"曲线卖完点"与"毕业点"重合）**：
  ```
  reservedTokens = supply · phantomQuote / (phantomQuote + graduationThreshold)
  ```
  在曲线初始化时固定 → **相同配置的每次 launch 都有确定性的毕业种子，永远不会有"剩余代币"。**

**⚠️ 关于 4.2 ETH 毕业阈值的更正**：
- 二手来源一致引用 **4.2 ETH**（稳定币 quote 据报 ~8,090 USDG）
- **但 factory 源码中没有硬编码常量** —— 它是 `LaunchConfig` 里 owner 可配置的字段
- **结论：4.2 ETH 是运营默认值（policy），不是协议常量。**

**两阶段、无许可的毕业流程**：

1. `curve.graduate()` —— 停止曲线交易、清扫手续费、把 **100% 储备**交给 factory
   - **在跨越阈值的那笔买单内部自动触发，包在 try/catch 里** → **毕业失败绝不会让用户的买单回滚**
   - **强制转入（force-sent）的捐赠被排除在种子之外**，防止有人靠直接打款扭曲毕业价格
2. `factory.createGraduatedPool()` —— 直接铸造 **full-range Uniswap V4 头寸**，NFT 送进 locker
   - **可重试**，储备永远不会被卡死
   - `PonsV2GraduationGuard`：**无状态预检**，镜像 V4 真实的拒绝路径
   - **7 天 rescue delay** 兜底

**永久锁仓（V4）**：
- `PonsV2LaunchLocker` **不暴露 `collectFees`、不暴露提取、不暴露任意调用** —— **代码级永久锁定**
- NFT **不是**打进黑洞地址，而是放进一个**没有出口的合约**。效果等价，**可审计性更好**
- 池子的 **V4 核心 LP 费必须设为 0**，全部费用经济学走 hook

**Hook 架构**（`hooks/PonsV2MemeHook.sol`）：
- **Singleton（单例）** hook 服务**所有**毕业池，用 `PoolId` 索引注册表
- 只启用 **`afterSwap`**（+ `beforeInitialize` 做注册门禁）
- 每笔 swap 从 V4 的 flash-accounting 中直接抽成
- 若抽到的是 meme 币，会在**价格冲击上限约束**（`maxInternalPriceImpactBps` 默认 **300 = 3%**）下批量换回 quote 资产
  → **协议/创建者收入永远是 quote 计价，不会变成没人要的 meme 灰尘**

### 3.3.4 Anti-snipe 衰减税：**为什么它对 Mantle 是个坏消息**

**Factory 层已证实的常量**：

| 参数 | 默认值 | 上限 |
|---|---|---|
| `snipeTaxStartBps` | **9,900 bps = 99%** | 硬顶 9,900 |
| `snipeTaxSeconds` | **15 秒** | 可调 1–60 秒 |
| 豁免名单 | ≤ **32 个地址**（创建者 + 费用接收方自动豁免） | |

- 参数在 launch 创建时**快照固化**，事后调整 factory 默认值**不影响已有 launch**
- 若非零，`snipeTaxStartBps` 必须超过 2,000 bps 的常规费用上限

**未能证实的部分**（重实现前必须验字节码）：衰减曲线形状（二手称"指数衰减"且窗口 **5 秒**，与 factory 的 15 秒默认值**冲突**）、是否只对买单生效、税款去向。

**设计意图**：99% 起始税意味着**第 0 秒抢跑者的收益被完全没收**。在 100ms 出块的链上，**15 秒 = 150 个区块**的博弈窗口，足以让人类用户与 bot 站到同一起跑线。

> ### ⚠️ 这条对 Mantle 是一个硬约束
> **anti-snipe 衰减税的窗口必须以「秒」计价，但它的有效粒度取决于出块时间。**
> - RH Chain：15 秒 = **150 个区块** → 衰减曲线细腻，每个区块税率都不同，抢跑者无法找到"最优区块"
> - Mantle：15 秒 = **7–8 个区块** → 衰减曲线粗糙到**退化成一个阶梯函数**，抢跑者只需算出"第几个区块的税率低于我的预期利润"，然后在那个区块集中开火
>
> **→ Mantle 若不改出块时间，就不能用 Pons 式的衰减税，只能退化成 PAIR 式的硬上限（保护弱得多），或者引入完全不同的机制（见第六部分 6.3.5 的批量拍卖方案）。**

### 3.3.5 费用结构：一个被媒体普遍误报的关键点

**Hook 默认参数**：

| 参数 | 默认值 | 含义 |
|---|---|---|
| `hookFeeBps` | **100 = 1%** | 交易费（owner 可上调至 10%） |
| `protocolFeeShareBps` | **3,000 = 30%** | 协议分成 |
| `buybackBurnBps` | **5,000 = 50%** | 从创建者那 70% 里再切 50% |
| `creatorTaxBps` | 创建者自设，上限 **1,000 = 10%** | 叠加在上述之外，全额归创建者 |
| 总交易费硬顶 | 曲线费 ≤10%，创建者税 ≤10%，**合计 ≤20%** | |

**所以 1% 基础费的真实拆解是三段而非两段：**

```
1% 交易费
├── 30%  → 协议（Pons Labs）
├── 35%  → 创建者（即时可领）
└── 35%  → 回购金库（buyback vault）
```

**⚠️ 最重要的一条更正：Pons 的回购是「锁仓」，不是「销毁」。**

README 原文：*"**Buybacks are locked, not burned**: `PonsV2BuybackVault` vests bought-back supply linearly over five years with a weighted-average vesting clock."*

→ 这 35% 用来买回**该 launch 的 meme 币**，存入 `PonsV2BuybackVault`，**5 年线性解锁**，解锁时再按份额在协议/创建者间分配。

**其他费用**：
- **创建费**：**0.0005 ETH / 每次发币**
- **毕业费**：无单独条目
- 手续费通过 `IPonsV2FeeEscrow` 的 claim-based 账本结算
- `CREATOR_FEE_RECIPIENT_TIMELOCK = 3 天`

### 3.3.6 $PONS 代币经济（链上直读验证）

**⚠️ 一个被所有二手来源混淆的关键区分：**
- **`PonsV2BuybackVault`（合约层）** 回购并锁仓的是**每个 launch 的 meme 币**，5 年归属，**不销毁**
- **$PONS 代币本身**另有一套**协议层的回购销毁政策**，走黑洞地址

**链上直读结果（2026-09-06）**：

| 项 | 值 |
|---|---|
| `totalSupply()` | **1,000,000,000 PONS** |
| `balanceOf(0x…dEaD)` | **296,993,295.46 PONS** |
| **已销毁比例** | **29.70%** |

CoinGecko（2026-09-06）：价格 **$0.9289**，市值 **$661.26M**，24h 量 **$216.62M**，市值排名 **#93**。

**回购销毁政策（二手，非合约强制）**：据报**协议费用份额的 80%** 用于公开市场回购 PONS 并永久销毁，通过 **TWAP** 执行。
⚠️ **分析师明确指出：这个 80% 是 Pons Labs, LLC 设定的「政策」，不是不可变的智能合约条款，团队随时可以调整。**

> **→ Mantle 若照抄，应把这条写进合约而非博客。** 这是一个几乎零成本、但能形成真实信任差异化的动作。

**Uniswap Labs 于 2026-09-03/04 买入 PONS**（据报 100 万枚）"for long-term alignment" —— 发生在 **Uniswap 自己的 pools.trade 与 Pons 竞争之后**。

### 3.3.7 运营数据

| | 24h | 7d | 30d | **累计** |
|---|---|---|---|---|
| **Pons V1 手续费** | $296,944 | $2.59M | $7.94M | **$25.00M** |
| **Pons V2 手续费** | **$8,750,574** | $33.85M | $47.36M | **$47.48M** |
| **合计** | **~$9.05M** | ~$36.44M | ~$55.30M | **$72.48M** |

**V2 日手续费序列 —— 这条曲线是整个 2026 年 launchpad 赛道最重要的一张图：**

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
09-04  8,750,574  ← 峰值，31 天从 $31.8K 涨到 $8.75M（275 倍）
```

**其他**：发币数 **>167,000**，**>52,000** 个持币地址（~2026-08-30）。2026-08-31 单日占**全加密行业 launchpad 总手续费的 63.9%**。
⚠️ **毕业数与毕业率：未找到可靠数据源。这是本节最大的数据缺口。**

### 3.3.8 Pons 是否支持股票代币作 quote？

- **合约层：技术上支持。** V2 的 `PairTokenEconomics` 是**通用机制**，任何 owner 批准的 ERC-20（最低 6 位小数）都可做 `pairToken`，每种资产配自己的 `phantomQuote` 与 `graduationThreshold`。**它是资产无关（asset-agnostic）的，不是为股票代币专门设计。**
- **运营层：未证实 Pons 主打股票配对。** 报道中最清晰的股票配对案例（Artificial Inu/NVDA、BONER/HIMS、SPACEHOOD/SPCX）均归属于**另一个叫 LONG（long.xyz）的竞品**。

> **结论（媒体常混为一谈，务必区分）**：
> **Pons ≈ Robinhood Chain 的 pump.fun（ETH 计价的通用 meme 工厂）；PAIR 与 LONG 才是「股票代币做 quote」这个赛道的专业玩家。**

---

## 3.4 PAIR / pair.fund：「Stop launching against ETH」

### 3.4.1 概览

| 项 | 值 |
|---|---|
| 官网 | **pair.fund** |
| 定位标语 | **"Stop launching against ETH"** —— 直接对标 Pons 的 ETH 计价模型 |
| 创始人 | Tugg（@0xTugg），"PAIR Labs by Luxington" |
| 与 Pons 关系 | **不是 fork，不是同一团队**，是直接竞品 |
| 首次上线 | 2026-07-21（单配对，Uniswap v3） |
| **V5 multipool 上线** | **2026-08-26** |
| 公开发布 + AWS 基础设施合作 | 2026-08-31 |
| $PAIR | `0x6b1d42927b1a84ec28fa88d4fc6fa7af404966be` |

### 3.4.2 核心机制：multipool 原子发射

**一笔原子交易内完成：**

1. 部署新 ERC-20，**固定供应 1,000,000,000**，**无 mint 函数、无后续增发**（本报告链上验证 `totalSupply()` = 1B）
2. 创建 **1–5 个 Uniswap v4 池**，每个池对应创建者从**白名单 24 只 Robinhood 股票代币**中选一只作 quote
   - 白名单：`AAPL, AMC, AMD, AMZN, BABA, BE, CRCL, CRWV, GOOGL, INTC, META, MSFT, MU, NVDA, ORCL, PLTR, QQQ, SGOV, SLV, SNDK, SPCX, SPY, TSLA, USAR`
   - **创建者自选权重，必须精确加总为 100%**
   - 例：权重 33/33/34 分给 AAPL/MSFT/NVDA，该代币行为上就是一个**三资产指数**
3. 每个池以**预言机导出的开盘价**注入**单边集中流动性**，**在发射交易内直接出资**（不经预售曲线）
4. 可选**付费 dev buy**
5. **每个 LP 头寸永久锁进 `PairV4Locker`** —— 无创建者提取路径，永远
6. 代币注册上链，出现在 PAIR explorer/API

**"原子发射"的准确含义**：代币 + N 个池 + 锁仓 + 配对元数据**要么全部落在同一个区块，要么整笔交易回滚**。**不存在"代币已存在但池未注资"或"池存在但头寸未锁"的中间态。**

- **发射费**：**0.0005 ETH**（与 Pons 相同），不到 1 分钟完成

### 3.4.3 无 bonding curve、无毕业迁移

- **完全没有 bonding curve。** 每个代币从第一笔 swap 起就在**真实的、已注资的 Uniswap v4 池**里交易 —— 即"即时永久流动性"模型。
- **"毕业"只是一个信息性链上标志位**：任何人可无许可触发，条件是锁仓本金超过 **4.2 ETH × ETH/USD**（**PAIR 直接沿用了 Pons 的 4.2 ETH 作为文化符号**）。**翻转这个标志位不移动任何流动性。**
- **流动性连续投放到 Uniswap 极限可用 tick —— 没有价格天花板。**

### 3.4.4 Anti-snipe：硬上限而非衰减税

| 参数 | 值 |
|---|---|
| 生效窗口 | 发射区块 + 之后 **5 个区块** |
| 单笔买入上限 | **5.5%** 供应量 |
| 单钱包持仓上限 | **5.0%** 供应量 |
| 卖出 | **永不受限** |
| 解除 | 自动，无需管理员操作 |
| 转账税 / 运营 bot | **无** |

> 5 个区块 @ 100ms = **0.5 秒**窗口。相比 Pons 的 15 秒 99% 衰减税，PAIR 的保护弱得多 —— **这是"追求上线即真实池"必须付出的代价。**

### 3.4.5 多池套利与聚合路由 —— 以及一个关键设计缺陷

**`PairV5MultiPoolAggregator`**（无许可路由合约，AUTO 模式）：
- 为买/卖报价**每一条池腿**
- **拒绝实时价格冲击 >15% 的路由**
- **只在 gas 调整后的执行改善足够时**才拆单到多池
- 应用**每腿 + 总量**双重滑点保护
- 钱包签名前**立即重新模拟**
- 用 **USDG** 作为各腿共同的输入/输出资产

> ### ⚠️ 关键设计缺陷（Mantle 必须解决的第一个空白）
> **聚合器不做池间再平衡，它只为单个交易者优化执行。**
> 篮子内各池的价格对齐**完全依赖外部套利资本**。
> 而按 PAIR 自己的承认，**在股票市场休市期间，这种纠正可能长时间缺席。**

### 3.4.6 $PAIR 代币经济（链上直读）

| 项 | 值 |
|---|---|
| `totalSupply()` | **1,000,000,000** |
| `balanceOf(0x…dEaD)` | **102,177,191.72** |
| **已销毁比例** | **10.22%**（上线仅 8 天） |
| 上线 | 2026-08-29，**在自家平台发射，配对 SPY** |
| 价格 / 市值 | **$0.041914 / $37.70M**，24h 量 **$40.05M** |
| 回购政策 | **协议费用的 90% 回购销毁 $PAIR**；剩余 10% 用于创建者拓展、市场、基础设施 |

### 3.4.7 费用与数据

**DefiLlama 官方方法论原文**：
> *"**1% pool swap fee on every trade**, accruing to permanently locked Uniswap V4 positions and periodically swept to creators and the protocol, plus a **0.0005 ETH launch fee**."*
> *"Revenue: **30% of collected swap fees** + the full launch fee."* / *"SupplySideRevenue: **70% of collected swap fees**."*

→ **PAIR 与 Pons 的 1% / 70-30 完全一致**，但 PAIR 没有 buyback vault 那一层，**创建者拿满 70%**。

| | 24h | 7d | **累计** |
|---|---|---|---|
| 手续费 | $52,112 | $389,921 | **$433,275** |
| 协议收入 | $15,680 | $118,431 | **$131,643** |

**其他（官方口径）**：累计交易量 08-29 破 $6M → 08-30 破 $15M → 08-31 破 $26M；**>160,000 笔交易**，**>1,200 个代币**；创建者奖励累计 **>$180,000**；已"毕业"（信息性）10 个。
代表作：**ABSOLUTE CINEMA**（AMC）、**Chips Party Pack**（NVDA+AMD+INTC+MU）、**PEAR**（AAPL+MSFT+NVDA）、**X Holdings**（SPCX+TSLA）。

> **规模判读**：PAIR 累计手续费 $433K vs Pons $72.48M —— **PAIR 是 Pons 的 0.6%**。
> **PAIR 的价值不在规模，在于它是「股票代币作 quote」这个机制的最完整实现，是 Mantle 最直接的抄袭对象。**

### 3.4.8 市场投票：这不是噱头

**链级数据（CoinGecko 口径，2026-06-29 ~ 07-27）**：

| 股票代币 | 配对 meme | 交易量 |
|---|---|---|
| **NVDA** | AI / CHIPS / JACKET / REAL / LONGSHOT / SWOGE | **$34.3M+**，占据前 25 大配对中的 8 席 |
| **GME** | GME / WSB / AMC 系 | $26.8M + $1.9M + $1.7M |
| **SPCX**（SpaceX） | MARSCOIN / SPACEHOOD / ASTEROID / MOON | $7.8M + $4.5M + $1.4M + $0.8M |
| **AAPL** | AP / AAPLCAT | $4.4M + $2.8M |
| **MSFT** | CLIPPY | $2.2M |
| **TSLA** | USEDTESLA | $1.6M |

**最成功的股票配对 meme：Artificial Inu (AI)，配对 NVDA**（在 **Long.xyz** 上）：
市值从 ~$1.5M（08-01）→ ~$135M（08-30）→ 09-02 报 **~$275M**，单池流动性 ~$26M，24h 量 >$11M。

> ### ⭐⭐ 这条数据是整个报告最有说服力的一条
> **其 NVDA 配对池的流动性（~$3.3M）是其 WETH 池的 3 倍以上。**
> 交易者**主动选择**把流动性放在股票配对池而非 WETH 池。
> **「股票代币作 quote」不是噱头，是被市场用真金白银投票认可的产品形态。**

另有链级统计：2026-09-02，meme × 股票代币交易对成交额 >**$217M**，**一度超过底层股票代币自身的成交额**。

### 3.4.9 股票代币作 quote 的七类风险（Mantle 的设计输入）

| 风险 | 机制 | 是否已实际发生 |
|---|---|---|
| **休市陈旧价** | 每周 135+ 小时无股票侧参考价、无股票做市商 | ✅ HIMS 4.6x、AMC 35x |
| **周一跳空回补** | 开盘后链上价格向真实价暴力收敛，**协议层无任何熔断** | ✅ 隐含于上述事件 |
| **逼空/囤积浮筹** | 单个 meme 池可囤积某股票代币的大部分链上浮筹 | ✅ BONER 囤了 53% 的 HIMS |
| **公司行动脱钩** | 代币层有 `uiMultiplier`，**但池层没有任何对应的再平衡逻辑** | ⚠️ 未发生，**未解决的开放风险** |
| **发行人停摆** | 只有 BBVI 能 mint/redeem。若增发被暂停，股票代币将**完全脱钩且不可套利** | ⚠️ 未发生（AMC 争议期间几乎发生） |
| **无常损失** | meme 与股票是**不相关资产**，背离时唯一锁仓 LP 承受剧烈 IL | ✅ 结构性存在 |
| **假冒标的** | Farmmi 案中 meme 配对的是一个**完全伪造的"代币化 FAMI"** | ✅ 已发生 |

**PAIR 已宣布但未完全上线的缓解措施**（2026-09-01 新闻稿）：
- **"peg guard"** —— 当某股票代币链上价偏离最后收盘价过大时，**暂停对其新发射**
- 链上风险标签：交易前显示实时溢价/折价
- 招募链上做市商提供周末覆盖

> ⚠️ **目前不存在任何链上预言机偏离熔断器或自动停牌。**
> 官方策略是**「篮子分散 + 信息披露」，不是硬编码的安全机制。**
> **→ 这是 Mantle 最大的、可申请专利级的差异化机会（见第六部分 6.4）。**

---

## 3.5 生态其他玩家：三个教训

### 3.5.1 Noxa 的兴与亡（2026-06-30 → 07-16，**16 天**）

| 日期 | 事件 |
|---|---|
| 2026-06-30 | 随主网上线（比 Robinhood 官宣早 1 天） |
| 06-30 ~ 07-11 | 拿下链上 **65.8% 的发币份额（~60,000 个代币）**；**连续 5 天日手续费超过 pump.fun** |
| 2026-07-11 | **暂停发币**，理由是 bot 刷量与山寨代币压垮基础设施 |
| 2026-07-13 | **官网下线**，归咎于 Cloudflare |
| 2026-07-14 | 重现，留言 *"the cat has been liberated"*，承诺未来 100% 手续费归创建者 |
| 2026-07-16 | 域名注册商**查封并转卖其域名**，只剩 ENS 托管页面 |

- 旗舰代币 **CASHCAT** 事后跌 >30%，**但活了下来** —— 2026-08-06 被 **Robinhood 主 App 直接上架现货交易**，CEO Vlad Tenev 关注了它的账号。
- Noxa 残余业务转战 Monad / MegaETH / Merlin 等链，累计手续费 $21.70M。

> ### 教训一：先发优势在 launchpad 赛道价值极低
> Noxa 拿下 65.8% 份额后 16 天内归零，原因是**基础设施承压 + 团队运营失能**，而非机制不好。
> **能扛住 bot 洪水的工程能力 > 机制创新。**

### 3.5.2 Uniswap：既是基础设施，也是竞争者

- **Uniswap v2/v3/v4/UniswapX 主网首日全量部署**，合计承接链上约 **99% 的 DEX 流动性**；股票代币中约 **73% 在 v4 / 26% 在 v3**。
- 2026-09-06 单日 **Uniswap V4 在 RH Chain 交易量 $1,029.7M**（占全链 DEX 量 ~67%）。
- **Uniswap Labs 自己下场做 launchpad：`pools.trade`，2026-08-05 上线** —— **发币零费用、每笔交易 0.25%**（低于 Pons/PAIR 的 1%）。**首日发币数就超过 Pons**，并使 Pons 当周下跌约 49%，随后 Pons 反弹。
- 2026-09-03/04：**Uniswap Labs 转而买入 PONS 代币**"for long-term alignment"。

> ### 教训二：launchpad 的护城河不在 AMM
> DEX 自己下场做 launchpad（零发币费 + 更低交易费）**没有压死专业 launchpad**，最后选择资本层面结盟。
> **护城河在「发行体验 + 社区 + 费用分配设计」，不在流动性层。**
> → 这对 Mantle 是好消息：**不需要先赢 DEX 战争才能做 launchpad。**

### 3.5.3 完整生态名单

| 类别 | 项目 |
|---|---|
| **借贷** | **Morpho**（Robinhood Earn 底层，USDG 出借 ~7% APY，经 **Lloyd's of London + RELM** 承保；累计手续费 $1.75M） |
| **永续** | **Lighter**（RH Wallet 内原生集成，承诺 1,100 万 $LIT 给 RH 社区，TVL $64.66M）、**Arcus** |
| **PropAMM** | **Rialto**（做市商背书的链上流动性，专为股票代币薄流动性场景设计；与 RFQ 的区别是**它在链上，因此可组合**） |
| **自营 AMM** | **Pleiades** |
| **稳定币** | **USDG（Paxos）**，链上稳定币市值 $964.56M，占 **66.4%** |
| **交易终端/bot** | **GMGN** —— RH Chain 上 24h 手续费 **$1.65M**、DEX 量 **$641.7M** |
| **其他 launchpad** | **Long.xyz**（单股票配对专业户）、**pools.trade**（Uniswap）、o1 Exchange、LetsCash、**Bags**、**Flap.sh**、**clanker**、**Flaunch Game Mode** |
| 其他 DeFi | **Delta**（流动性引导）、**UP**（ve(3,3)）、**NetNet**（OHM 式债券，8 月市值一度 >$117M） |

**股票代币作抵押品**：官方文档明确列为用例（*"Deposit NVDA as collateral on a lending market and borrow USDG"*），但**未找到一个以股票代币为主要抵押品的旗舰协议**。目前 DeFi 侧股票代币存款约 **$72.7M**。

---

## 3.6 监管与政治：AMC 事件

### 3.6.1 时间线与原话（2026-09-04 ~ 05）

**AMC CEO Adam Aron 公开长帖**：

> *"The list of concerns is almost existential."*
>
> *"In good conscience, how can Robinhood as a U.S. company set up an operation in **far offshore Jersey, an island 3000 miles away**, and market **a security sort of posing as AMC** in some shape or fashion, and not comply with U.S. securities laws. **That is shocking and shameful.**"*
>
> *"I hereby call on you and Robinhood to voluntarily **CEASE AND DECIST** the trading of AMC stock tokens. If you don't, our high priced securities counsel has been asked to see whether we can force you to stop."*

他另称这些代币 "contemptible"、"vile"，是一个 **"fictitious synthetic equity market"**。三条核心担忧：无所有权/投票权；未注册证券且规避 AMC 自己必须承担的合规成本；脱钩的合成市场可能**损害 AMC 的融资能力**。

**Robinhood 的回应 —— 极其强硬：**
- **首席法务官 Dan Gallagher（2011–2015 年 SEC 委员）**：
  > *"**We know a little something about the U.S. securities laws and will not 'DECIST.' Send your lawyers and we'll educate them.**"*
- CEO Vlad Tenev：*"We stand behind Stock Tokens."*

**结果（截至 2026-09-06）**：AMC 股价当日盘中 **+21%**（争议本身成了催化剂）；AMC 已聘外部证券律师但**尚未向 SEC 提交任何文件**；**无任何 SEC/FINRA 执法行动**；争议持续中。

### 3.6.2 由此引发的「代币化模型之争」

| 发言人 | 立场 |
|---|---|
| **Gabriel Otte**（Dinari 联创） | *"synthetic tokens like Robinhood stock tokens and Ondo are just **indisputably worse for the end investors** than even common stocks."* |
| **Anna Wroblewska**（Dinari CBO） | *"The main problem isn't tokenization. It's the **marketing of a discretionary debt instrument, which functions essentially as an onchain CFD**, as an investment in the US stock market."* |
| **Hayden Adams**（Uniswap 创始人） | *"They're not worse if you want programmability, or to trade at night/weekend/holidays, live outside the US... **tokenized stocks are pretty similar to early stablecoins**."* |
| **Carlos Domingo**（Securitize CEO） | *"I would also not want people creating offshore derivatives of our stock... This is why we tokenized our own stock natively and in the US, in a fully compliant way."* |
| **Brian Huang**（Glider 联创，前 XTX 股票交易员） | *"**AMMs do not guarantee best execution for consumers.**"* / *"When you put in a meme coin with the stock, they're not really correlated assets. **You're exposing people to a lot of impermanent loss.**"* |

**监管侧的书面输入**：
- **Continental Stock Transfer & Trust**（2026-07-21 提交 SEC Crypto Task Force）：第三方合成代币 *"do not establish a legal relationship between the token holder and the issuer"*，会 *"**bypass the shareholder-record and corporate-action infrastructure**"*。
- **Computershare**（2026-07-28）：立场相反，主张对各种账簿形态**中性对待**。

### 3.6.3 合规现状：一个巨大的悬空

**美国法**：
- SEC 2026-01-28 Corp Fin 声明把 Robinhood 归入第三类 **"linked securities"**，警示破产/交易对手风险。
- **AMM 池交易证券型代币是否构成未注册的证券交易所（Reg ATS）？**
  → **没有任何针对 Robinhood Chain / Pons / PAIR 的 SEC 执法、无异议函或法院裁决。** ⚠️
  → 律所普遍看法：SEC 的"实质重于形式"原则意味着交易证券类代币的 AMM 池**可能触发 Reg ATS 风险**；"纯经济敞口无所有权"的产品可能被定性为 **security-based swap**，在美国对零售分销有极重限制。
- **Robinhood 的防线**：代币不向美国人发售（Reg S）+ 发行人在泽西岛 + 池子在无许可链上 = **主体地理隔离 + 无许可基础设施**的组合。

**欧盟法**：MiCA **明确排除**符合"金融工具"定义的代币 → 归 **MiFID II** 管辖。Robinhood 由泽西岛（非欧盟）发行、面向 EEA 销售 → **具体适用哪个交易场所授权制度在公开来源中无定论**。⚠️

> ⚠️ **明确的知识空白：没有任何具名监管机构就「Pons/PAIR 的 AMM 池交易 Robinhood 股票代币」适用何种制度作出过表态。这是整个赛道最大的悬空风险。**

### 3.6.4 Robinhood 实际拥有的控制杠杆（按强度排序）

| 杠杆 | 强度 | 说明 |
|---|---|---|
| **一级市场垄断（Bitstamp/BBVI 是唯一 AP）** | ★★★★★ | 可增发/赎回纠偏（**HIMS 事件中实际使用**），也可**停止增发**使代币彻底脱钩 |
| **Sequencer 级 transaction screening** | ★★★★☆ | 官方承认存在，可排除制裁地址；ArbOS filtering 甚至可作废 L1 强制包含的交易 |
| **`pauseOracle()` / `updateMultiplier()`** | ★★★★☆ | 完全中心化的公司行动裁量权；喂价一停，所有依赖预言机的协议失灵 |
| **Security Council 8 席多签** | ★★★☆☆ | 需 6/8（+7 天 timelock）或 7/8（紧急）—— **Robinhood 单方做不到** |
| **代币层黑白名单** | ☆ | **未发现**。二级市场转账完全开放 |

> ### ⭐ 结论：一个刻意的责任切割
> **Robinhood 有能力也有意愿在「资产层」干预（AP 增发、暂停喂价、停牌），但在「应用层」（第三方 AMM 池）事实上放任。**
> 合规风险留在自己的产品面，投机风险外包给"无许可的第三方"。
>
> **→ 这是 Mantle/Bybit 必须做出的第一个战略选择：要不要采用同样的切割？**
> 我的建议见第六部分 6.7 —— **不完全照抄，因为 Bybit 的品牌风险敞口与 Robinhood 不同**。

---

## 3.7 数据总表

### 3.7.1 Launchpad 手续费横向对比（DefiLlama，2026-09-06 链上直取）

| 协议 | 链 | 24h | 30d | **累计手续费** | 累计协议收入 |
|---|---|---|---|---|---|
| **Pons V2** | Robinhood | **$8,750,574** | $47.36M | **$47.48M** | $9.02M |
| **Pons V1** | Robinhood | $296,944 | $7.94M | $25.00M | $6.39M |
| **Pons 合计** | Robinhood | **~$9.05M** | ~$55.3M | **$72.48M** | ~$15.40M |
| **Flap.sh** | BSC / X Layer / Monad / RH | $2,884,424 | $20.02M | $34.88M | — |
| **pump.fun** | Solana | $679,806 | $47.04M | **$1,210.68M** | — |
| **Pump（含 PumpSwap 全体系）** | Solana 等 4 链 | — | — | — | **$1,286.93M** |
| **clanker** | Base 等 | $8,965 | $283,465 | $90.82M | $15.15M |
| **four.meme** | BSC | $9,578 | $290,434 | $98.05M | $96.63M |
| **Bags** | Solana + RH | $30,603 | $1.81M | $64.00M | $32.0M |
| **NOXA Fun** | Monad / MegaETH / RH 等 | $96,904 | $3.62M | $21.70M | — |
| **LaunchLab / LetsBonk** | Solana | $4,921 | $64,831 | $15.93M | — |
| **PAIR** | Robinhood | $52,112 | $433,275 | **$433,275** | $131,643 |
| **Zora（Coins, Base）** | Base | — | $15,067 | $10.43M | $7.70M |
| **Flaunch** | Base | $28.93 | $389.64 | $3.59M | $3.07M |

### 3.7.2 Pons vs pump.fun（本节最重要的一张表）

| 维度 | **Pons** | **pump.fun** |
|---|---|---|
| 上线 | 2026-07-14（V1）/ 2026-08-04（V2） | 2024-01 |
| 运行时长 | **~54 天** | **~32 个月** |
| **累计手续费** | **$72.48M** | $1,210.68M |
| **日均手续费（生涯）** | **~$1.34M/天** | ~$1.24M/天 |
| **当前日手续费** | **$9.05M** | $0.68M |
| **当前倍数** | **13.3x pump.fun** | — |
| 曲线 | 恒定乘积 + phantom quote 虚拟储备 | 恒定乘积 + 虚拟储备 |
| 毕业目的地 | **Uniswap v4 + singleton hook，永久锁定** | PumpSwap |
| **毕业滑点/预言机** | **零**（曲线用未来池的 quote 资产计价） | 需要迁移 swap |
| 交易费 | **1%** | 1% → 分级 |
| 费用分配 | **30% 协议 / 35% 创建者 / 35% 回购金库（5 年归属）** | 协议为主，后加创建者分成 |
| 出块时间 | **~100ms** | ~400ms |
| Anti-snipe | **99% 起始税，15 秒衰减窗口** | 无原生衰减税 |
| 发币数 | >167,000 | 数百万 |

> ### ⚠️ 判读（Mantle 做规划时必须内化这一条）
> Pons 在 54 天里做到了 pump.fun 生涯累计的 **6%**，但**当前日流水是它的 13 倍**。
> 这既证明了「新链 + 100ms + 股票叙事」的爆发力，也提示了**极端的不可持续性**：
> **这条曲线 31 天涨了 275 倍，且建立在将于 2026-09-29 到期的 gas 补贴之上。**
> **任何基于当前数字的战略推演都必须做补贴退出后的压力测试。**

### 3.7.3 链级横向对比（2026-09-06）

| 链 | 24h DEX 量 | 30d DEX 量 | 24h 链 gas 费 |
|---|---|---|---|
| Solana | $1.915B | $62.08B | ~$613K（09-02） |
| **Robinhood Chain** | **$1.53–1.61B** | $24.12B | **~$2.9M**（峰值 $4.45M @09-02） |
| Base | $589.22M | $24.16B | — |
| BSC | — | $33.14B | $882,890 |
| Ethereum | ~$1.3B | — | ~$304K（09-02） |
| **Mantle** | **$0.94M** | **$67.2M** | **$220** |

> **Mantle 的 24h DEX 量是 Robinhood Chain 的 1/1,630。** 这个数字应该被贴在任何 Mantle launchpad 提案的第一页。

---

## 3.8 提炼：Robinhood 飞轮的真实因果链

```
自有 L2（100ms + FCFS + sequencer 级合规过滤）
  → 可自由转账的 ERC-20 股票代币（ERC-8056 处理公司行动）
  → 股票代币成为 launchpad 的 quote 资产（NVDA 池深度 = WETH 池的 3 倍）
  → meme 投机带来爆炸性交易量与手续费
  → 交易量给股票代币带来它本不可能拥有的链上流动性与持有需求
  → 链的 gas 收入 + Robinhood 的 AP / 交易收入
```

> ### ⭐ 最重要的一条观察
> **Robinhood 并没有"设计"这个飞轮 —— 它只设计了前两环，第三环是第三方（Pons / PAIR / LONG）自发涌现的。**
> 但正是第三环让这条链在 2 个月内做到全链 DEX 量第 2。
>
> **对 Mantle 的含义：Mantle 已经有了第一环和第二环（自有 L2 + 可自由转账的 xStocks），但第三环没有自发涌现。**
> **因此 Mantle 必须主动构建第三环 —— 这正是第六部分设计方案的全部内容。**

### 3.8.1 五条硬约束（Mantle 若要复制，缺一不可）

1. **mStocks 必须是可自由转账的标准 ERC-20，一级市场可 KYB 准入、二级市场开放。**
   → 若走 bStocks 式封闭生态，第三环永远不会出现。
   → ✅ **Mantle 上的 xStocks 已经满足**（待与 Backed 确认无 transfer hook）。
2. **必须用 ERC-8056 式的 `uiMultiplier` 处理分红/拆股，绝不能 rebase。**
   → 否则第一次分红就会把所有永久锁仓 LP 套利抽干。
   → ⚠️ **需核实 Backed 的 xStocks 在 Mantle 上如何处理公司行动**（E 篇提到 "Multiplier" 乘数机制，需确认是否等价于 ERC-8056）。
3. **必须有一个能在偏离时增发/赎回的 AP。**
   → HIMS 事件中，唯一救回 4.6x 脱钩的机制就是 AP 增发。**这是唯一有效的锚定手段，比任何"peg guard"都重要。**
   → ✅ **Mantle 有更强的版本：Fluxion 的 Atomic RFQ 可直接向发行方 mint/redeem**（见第六部分 6.2）。
4. **必须有 sequencer 级的合规过滤能力。**
   → 这是券商/发行方肯把证券放上链的前置条件。Arbitrum 已产品化（ArbOS Elara），**Mantle 需自建**。
5. **出块时间决定 anti-snipe 机制的可用形态。**
   → 100ms 下 15 秒 = 150 个区块，衰减税粒度细腻；**2 秒出块下只有 7-8 个区块，只能退化成 PAIR 式的硬上限**。

### 3.8.2 Robinhood 没解决、Mantle 可以差异化的三个空白

1. **周末/休市的价格纪律** —— 目前**零协议级熔断**。PAIR 的 "peg guard" 只暂停新发射，不管存量池。
   → **可做：预言机偏离熔断的 v4 hook** —— 池价与最后收盘价偏离超阈值时，自动收窄可交易区间或对该方向加征惩罚性费用。**这是一个尚无人实现的机制创新。**
2. **公司行动的池层适配** —— 股票代币层有 multiplier，但**池层没有任何再平衡逻辑**。拆股会让永久锁仓 LP 与真实经济脱节。
   → **可做：v4 hook 监听 `UIMultiplierUpdated` 事件，在 `effectiveAt` 时自动调整池的 tick 参照系。**
3. **储备证明** —— Robinhood **没有**任何独立 attestation 或 Chainlink PoR。
   → **Mantle/Bybit 若提供链上可验证的 PoR，是对 "The Issuer is not regulated" 的直接降维打击。**

### 3.8.3 必须避开的三个坑

1. **不要指望先发优势** —— Noxa 拿下 65.8% 份额后 16 天归零。**抗 bot 洪水的工程能力 > 机制创新。**
2. **不要用补贴堆数据** —— Robinhood Chain 全部指标建立在 2026-09-29 到期的 gas 补贴上，稳态未知。Mantle 已经在 Funny Money / Printr 上验证过"砸钱无效"（见第四部分）。
3. **不要低估「发行人愤怒」的政治风险** —— AMC 事件里 Robinhood 敢硬刚，是因为它有前 SEC 委员当 CLO、代币不向美国人发售、发行人在泽西岛。**Mantle/Bybit 若没有同等的法律隔离结构，第一次被上市公司 CEO 点名就会很被动。**
