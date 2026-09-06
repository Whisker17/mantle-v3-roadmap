# Meme Launchpad 深度研究：从 Solana 到 Robinhood Chain，以及 Mantle 该怎么做

> **研究日期**：2026-09-06
> **目的**：为 Mantle 生态提供关于 meme Launchpad 的完整分析与可执行设计建议，重点结合 Mantle 的代币化股票资产（用户称 "mStocks"，实为 **xStocks**）。
> **方法**：6 条并行研究轨道 + 大量链上直接读取。标 `【实测】`/`[链上]` 的数据为直接调用 RPC / 合约 / API 得到，可复现。

---

## 📖 报告结构

| 章节 | 内容 | 篇幅 |
|---|---|---|
| **[第一部分：Solana 基线](report/01-solana-baseline.md)** | 从 GMGN 看懂 meme 交易四层栈；pump.fun bonding curve 的精确数学（链上解码）；完整生命周期；费用体系演进；MEV；Solana infra 为何适配 meme | 16K |
| **[第二部分：第二代 —— BSC 与 Base](report/02-gen2-bsc-base.md)** | four.meme 的可插拔计价资产（**已跑通 meme × 代币化美股**）、OpenFour、Binance 白标关系；Base 的创作者经济兴衰（**官方已公开否定**）；Clanker/Zora/Flaunch 的可抄机制 | 26K |
| **[第三部分：Robinhood Chain](report/03-robinhood-chain.md)** | 链架构与 FCFS 排序；Stock Tokens 的 ERC-8056 multiplier；**Pons 工程级拆解**；**PAIR 的股票代币 quote 机制**；三起真实休市事故；AMC 监管风暴；给 Mantle 的五条硬约束 | 37K |
| **[第四部分：Mantle 现状与失败归因](report/04-mantle-gap-analysis.md)** | **术语澄清（mStocks 不存在）**；停止错误诊断；链是空的；**执行层零覆盖**；两次 meme 尝试的死亡；七条归因；六张独有底牌 | 21K |
| **[第五部分：链 Infra 需求框架](report/05-infra-requirements.md)** | 14 维需求框架；**LFM in EVM 的四层可实现性阶梯**；弹性区块空间的三个子问题；抗狙击四范式；RWA quote 的四项额外要求 | 22K |
| **[第六部分：Mantle 设计方案](report/06-mantle-design-proposal.md)** | **Tape 协议**：五个设计前提、三张底牌、完整机制设计、**两个无人实现的机制创新**、分阶段路线图、风险登记表 | 28K |

**研究底稿**（每一份都比上面的章节更详细，含更多原始数据与来源）：
| 文件 | 内容 | 规模 |
|---|---|---|
| [`research/A-solana-pumpfun.md`](research/A-solana-pumpfun.md) | Solana 全栈，含链上解码的曲线常数、25 档 Ascend 费率阶梯、MigrateV2 交易逐指令分解 | 964 行 |
| [`research/B-bsc-fourmeme.md`](research/B-bsc-fourmeme.md) | BSC/four.meme，含生产 API 快照、Helper3 链上读取、LiquidityAdded 事件实测 | 673 行 |
| [`research/C-base.md`](research/C-base.md) | Base 生态，含 Clanker/Zora/Flaunch 完整合约级拆解、逐月数据序列、v4 hooks 14-flag 表 | 1,498 行 |
| [`research/D-robinhood.md`](research/D-robinhood.md) | Robinhood Chain，含 Pons/PAIR 源码级拆解、链上代币销毁量直读、监管争议原话 | 1,116 行 |
| [`research/E-mantle.md`](research/E-mantle.md) | Mantle 全面盘点，含 RPC 实测参数、TVL 崩塌序列、28 个 DEX 全表、36 项存疑清单 | 954 行 |
| [`research/F-infra-and-mechanism-notes.md`](research/F-infra-and-mechanism-notes.md) | 链 infra 与曲线机制的独立采集 | — |
| [`research/G-mstocks-design-inputs.md`](research/G-mstocks-design-inputs.md) | xStocks 可组合性约束、Fluxion RFQ、代币化 IPO | — |

---

## 🎯 一页纸执行摘要

### 全报告最重要的六条结论

**① 「meme × 代币化股票」已经不是空白市场，而是有两个生产实现的既有赛道。**
- **Robinhood Chain**：PAIR（24 只股票白名单 quote）与 LONG（单股票配对）。**NVDA 配对池的流动性是同一代币 WETH 池的 3 倍以上** —— 交易者用真金白银投票认可这个形态。
- **BNB Chain**：four.meme 的 **8 个 bStocks 已 PUBLISH**；**链上实测的 6 笔毕业里 5 笔是 bStocks 计价**。
- **→ 问题不是"可不可行"，而是"凭什么在 Mantle 做"。**

**② Mantle 的失败原因不是"慢"也不是"贵"，是执行层零覆盖 + 分发被 CeFi 截流。**
- 【实测】出块 **2.000s**、区块填充率 **0.173%**、base fee 常年钉在 **50 gwei 下限**（EIP-1559 从未触发）、单笔 swap **$0.004–0.009**
- **GMGN / Photon / BullX / Axiom / Trojan / Banana Gun / Maestro 全部不支持 Mantle**；**Phantom 不支持**
- GMGN 的链列表里有 Monad、MegaETH、X Layer、Robinhood Chain —— **唯独没有 Mantle**
- **Bybit Alpha 的设计目标就是让用户不必上链** —— Mantle 最大的用户漏斗，恰恰是它链上活跃度的最大抑制器

**③ Mantle 试过两次 meme，都是砸钱试的，都在 2026 年 8 月同一周关停。**
- **Funny Money**（2025-02 上线，**100 万 MNT 奖池**，明写 "harness the virality of meme culture"）→ **2026-08-16 关停**
- **Printr**（Bybit Venture Studio + Mantle EcoFund 背书，融资 **$4.5M**）→ **2026-08-18 关停**
- **→ "为什么没跑起来"不能写成"没人试过"**

**④ 但 Mantle 手里有三张别人没有的牌。**
- **Fluxion 的 Atomic RFQ**：开市锚定实时价、近乎无滑点；休市切 AMM 维持 24/7 —— **RH Chain 与 BSC 都没有对位物，而它恰好解决了 RWA×meme 最难的问题**
- **xStocks 规模全球第 2（$633.7M，是 Robinhood 的 4.8 倍）**、155 个标的、可自由转账 ERC-20
- **mETH/cmETH 生息基础设施** + 链上 **$576M 闲置稳定币**（是链上 DeFi TVL 的 5.9 倍）

**⑤ 有两个机制空白全行业无人填补，Mantle 可以独占。**
- **休市/周末的价格纪律**：RH Chain 已发生 HIMS **4.6 倍**、AMC **35 倍**的周末背离事故，**至今零协议级熔断**
- **公司行动的池层适配**：股票代币层有 multiplier，**但池层没有任何再平衡逻辑**（Robinhood 自己标注为"未解决的开放风险"）

**⑥ 「热点争用隔离 / LFM in EVM」对 Mantle 是"成功后的问题"，不是"现在的问题"。**
- Mantle 填充率 0.173%，**现在做 LFM 是无病呻吟**
- 但 Robinhood Chain 的反面教材极其鲜明：meme 让 base fee **11 天涨 82 倍**，Yakovenko 称其模型 "brain-dead"
- **→ v1 不做，但 v1 架构必须为 v2 留好接口**

---

## 📊 关键数字速查

### Launchpad 手续费横向对比（DefiLlama，2026-09-06 链上直取）

| 协议 | 链 | 24h | 30d | **累计** |
|---|---|---|---|---|
| **pump.fun** | Solana | $679,806 | $47.04M | **$1,210.68M** |
| **Pons（V1+V2）** | Robinhood | **~$9.05M** | ~$55.3M | **$72.48M** |
| four.meme | BSC | $9,578 | $290K | $98.05M |
| clanker | Base 等 | $8,965 | $283K | $90.82M |
| Bags | Solana + RH | $30,603 | $1.81M | $64.00M |
| **Flap.sh** | BSC 等 | **$2,884,424** | $20.02M | $34.88M |
| NOXA Fun | 多链 | $96,904 | $3.62M | $21.70M |
| LetsBonk | Solana | $4,921 | $64,831 | $15.93M |
| Zora Coins | Base | — | $15,067 | $10.43M |
| Flaunch | Base | $28.93 | $389.64 | $3.59M |
| **PAIR** | Robinhood | $52,112 | $433,275 | **$433,275** |

**Pons V2 的 31 天曲线**：$31,868（08-04 首日）→ **$8,750,574（09-04 峰值）= 275 倍**
⚠️ **但建立在 2026-09-29 到期的 gas 补贴之上，任何推演都必须做补贴退出压力测试。**

### 链级对比（2026-09-06）

| 链 | 24h DEX 量 | 30d DEX 量 | 日活跃地址 | TVL |
|---|---|---|---|---|
| Solana | $1.915B | $62.08B | 2.03M | $5.925B |
| **Robinhood Chain** | **$1.53–1.61B** | $24.12B | — | $908.65M |
| Base | $589.22M | $24.16B | 238,687 | $5.669B |
| BSC | $1.555B | $33.14B | 1.96M | $5.792B |
| **Mantle** | **$0.94M** | **$67.2M** | **991–2,306** | **$97.84M** |

> **Mantle 的 24h DEX 量是 Robinhood Chain 的 1/1,630。**
> **但稳定币供应只落后 Base 8.5 倍 —— Mantle 有钱，没有交易行为。这不是资本问题，是产品与分发问题。**

### 代币化股票市场（rwa.xyz，2026-09-05，全球总规模 $2.91B）

| 发行方 | 市值 | 特点 |
|---|---|---|
| Ondo Global Markets | **$869.6M** | 第 1 |
| **bStocks（Binance/BTech）** | **$659.4M** | **自营发行；2026-06-11 上线，7 周 AUM 破 $500M、46+ 标的、市场份额 45–50%** |
| **xStocks（Backed，Mantle/Bybit）** | **$633.7M** | **2025-11-07 上 Mantle，早 7 个月；155 个标的** |
| Robinhood（RHJ） | $133.2M | **市值第 6，但链上交易量与 DeFi 组合度第 1** |
| Dinari dShares | $11.2M | 唯一有股东权利 |

---

## 🔑 20 条可执行结论

### 关于机制设计

1. **bonding curve 的三个自由度一旦选定，其余全部锁死。** pump.fun 的 `30 SOL / 1.073B / 793.1M` 唯一确定了 `85.005359 SOL` 与 `410.88 SOL`。
2. **曲线是极度前置倾斜的 —— 前 30 SOL（35% 募资）卖掉 53.65% 供应。这是 sniper 军备竞赛的数学根源，不是治理问题。**
3. **毕业不应带来用户可感知的成本跳变**（pump.fun Ascend tier 0 上界 420 SOL 恰好高于毕业市值 410.88 SOL）。
4. **毕业阈值应锚定美元门槛而非代币数量**（four.meme 把 24 BNB 降到 18 BNB 以维持 ~$1.2–1.8 万门槛恒定）。
5. **"毕业迁移"是攻击面最集中的一步。** four.meme 用 $183K + 200 BNB 两次被黑买到教训，最终退回 V2。**最优解是 Pons V2：曲线用未来池的 quote 资产计价 → 毕业时零滑点、零预言机、零 MEV 窗口。**
6. **流动性锁定应用「永久锁 + Fee Key NFT」**，而非 LP burn（burn 会孤儿化费用收益）。Pons 的 locker **不暴露 collectFees / 提取 / 任意调用**，可审计性优于黑洞地址。
7. **抗狙击有四种范式**：硬延迟（烧掉 MEV）/ 拍卖（回流创作者但依赖排序语义）/ 衰减费（最通用）/ **额度门禁（不依赖出块时间）**。
8. **Mantle 的 2 秒出块让衰减税失效**（15 秒只有 7–8 个区块 → 退化成阶梯函数）→ **必须用额度门禁（Flaunch Game Mode 式 spend-gate）**。
9. **Progressive Bid Wall 优于市价回购** —— 市价回购必被夹，限价挂单 MEV 无从下手。**在薄流动性链上滑点损耗小一个数量级。**
10. **quote 资产默认生息**（Flaunch flETH → Aave）是 Mantle 的天然优势（mETH/cmETH）。

### 关于 RWA × meme

11. **公司行动必须用 ERC-8056 式 multiplier，绝不能 rebase** —— 否则第一次分红就会把永久锁仓 LP 套利抽干。**这是第一性约束。**
12. **AP 的 mint/redeem 是唯一有效的锚定手段**（HIMS 事件中靠 Bitstamp 增发 4,000 枚救回 4.6 倍脱钩），**比任何 "peg guard" 都重要**。
13. **一级市场 KYB 闸门 + 二级市场完全开放** 是整个飞轮的技术前提。若走封闭生态（bStocks 路线），第三方 launchpad 永远不会出现。
14. **曲线阶段应尽量不依赖预言机** —— 休市时喂价冻结（Chainlink 股票 feed **24/5，休市无 heartbeat**），任何实时喂价依赖都会被陈旧价格套利。
15. **必须有 sequencer 级合规过滤能力** —— 这是把证券型代币放上无许可 launchpad 的前置条件（Arbitrum 已产品化为 ArbOS Elara）。

### 关于分发与生态

16. **价值捕获排序是「接口层 > 协议层 > 链层」。** 三条独立证据：GMGN 占 RH Chain 40% DEX 量；Bankr 是 Base 三大协议之和的 3.9 倍；GMGN/Photon 收入与 pump.fun 同量级。**→ Mantle 必须自建执行层。**
17. **成为白标发行引擎 > 求上币直通车。** four.meme 的真正杠杆是 Binance Wallet **"integrates four.meme's launch technology"**；而 "four.meme → Alpha → 现货" 只是 **4 个一次性运营案例**，不是制度化通道。
18. **分发能买冷启动，买不到留存。** Base 用 $450K 和 13 个月证伪（Zora 日发币 54,000 → 422，**−99.2%**）；Jesse Pollak 原话 *"i was definitively wrong"*。
19. **KPI 必须是「毕业数 × 毕业后 7 天存活率」，绝不是日发币量。** four.meme 日发币量只跌 14%，日收入跌 99%；Clanker 每枚币手续费从 $1,371 跌到 $53。
20. **先发优势在 launchpad 赛道价值极低。** Noxa 拿下 RH Chain 65.8% 份额后 **16 天归零**，死因是基础设施承压 + 运营失能。**抗 bot 洪水的工程能力 > 机制创新。**

---

## ⚠️ 必须核实的事项（在启动任何开发之前）

这三条不通过，第六部分的方案需要重做：

1. **Mantle 侧 xStocks 的具体 mint/合约配置是否启用了 transfer hook 或黑白名单**
   （Solana 侧使用 Token Extensions 的可编程合规能力；EVM 侧需单独确认）
2. **Backed 的 "Multiplier" 机制是否等价于 ERC-8056（非 rebase）**
   若是 rebase，整个永久锁仓 LP 模型会在第一次分红时崩溃
3. **Backed / Bybit 是否接受 xStocks 被用作第三方 permissionless launchpad 的 quote 资产**（法务边界）

**其他重要的存疑项**（详见各研究底稿的存疑清单）：
- pump.fun 2026 年的毕业阈值已不再统一（抽样 3 笔迁移注入 SOL 相差 205 倍），疑与 2026-07 的 BOOST 机制有关，**未获一手确认**
- Pons 的 anti-snipe 衰减曲线形状与税款去向，源码中未定位；二手称 5 秒与 factory 默认 15 秒**冲突**
- Pons 的毕业数与毕业率 —— **无可靠数据源**
- Mantle 2026-04-19 TVL 崩塌（-72%/4 天）的确切成因 —— Kelp DAO rsETH 事件为时间吻合的推断，**无官方复盘**
- Mantle 强制包含窗口：官方文档写 24 小时，L2Beat 写 "up to 12h"，**未解**
- **没有任何具名监管机构就「AMM 池交易股票代币」适用何种制度表过态** —— 这是整个赛道最大的悬空风险

---

## 📌 术语澄清

| 用户用词 | 实际对应 | 说明 |
|---|---|---|
| **mStocks** | **xStocks** | 经全面检索，Mantle/Bybit/Backed 一手材料中**不存在 "mStocks" 这一产品名**。真实存在的是 Backed Finance 发行的 **xStocks**，2025-11-07 官宣上 Mantle。用户"类比 Binance bStocks"的直觉正确 —— **bStocks 确实存在**（BTech Holdings 自营，2026-06-11 上 BNB Chain）。 |
| **Pair** | **PAIR / pair.fund** | Robinhood Chain 上的 multipool RWA launchpad，用 1–5 个股票代币作 quote。**注意**：真正专攻"股票代币作 quote"的是 PAIR 与 **LONG（long.xyz）**；**Pons 是 ETH 计价的通用 meme 工厂（RH Chain 的 pump.fun）**，媒体常混为一谈。 |
| **Robinhood Chain 上的 Pons** | **Pons Family / ponsfamily.com** | Pons Labs, LLC（匿名团队），**与 Robinhood 完全无关**，Robinhood 官方声明第三方应用"不构成背书"。 |

---

## 数据可信度约定

| 标记 | 含义 |
|---|---|
| `【实测】` / `[链上]` | 本研究直接调用 RPC / 合约 / 官方 API 得到，可复现（各底稿附有可复现方法附录） |
| `[一手]` | 官方文档、官方公告、源码、L2Beat、PR 原文 |
| `[二手]` | 聚合器、媒体、研究机构转述，未经一手源交叉验证 |
| `⚠️ 未证实 / 存疑` | 存在源冲突或无法证实，已在各底稿的存疑清单中汇总 |

**本报告的原则是：宁可标注"不确定"，也不编造收敛。**
各研究底稿共标注了 **100+ 项**存疑与数据缺口，包括对本研究自己发现的三条互相矛盾的链上证据（pump.fun 2026 毕业阈值异常）不做猜测性收敛。
