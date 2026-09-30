# 支柱 1：发行层 —— Tape meme Launchpad（资本市场的点火器）

> **产品定位**：代币化股票的情绪一级市场（The Primary Sentiment Market for Tokenized Equities）  
> **核心使命**：作为 Mantle 资本市场的「点火器」，用情绪资产与注意力引爆链上活跃度，并将全部注意力与现金流反哺至 xStocks 深度与 MNT 价值捕获。  
> **决策依据**：锁定决策 D3（股票 quote 专注，彻底摒弃已死两次的通用 meme 路线）。

---

## 1. 为什么是「股票 Quote 专注」？

Mantle 曾两度尝试通过补贴做通用 meme Launchpad，均于 2026 年 8 月中旬同一周宣告关停：
- **Funny Money**（2025-02 上线，100 万 MNT 奖池）→ **2026-08-16 关停**
- **Printr**（Bybit Venture Studio + Mantle EcoFund 投资 $4.5M）→ **2026-08-18 关停**

**根本原因**：通用 meme 是纯内生注意力经济，在缺乏原生零售流动性与执行层工具零覆盖的 Mantle 链上，根本无法凭空制造病毒传播。

**Tape 的破局点**：**寄生在美股天然存在的注意力周期上**。
- **外部日历供给**：美股每个季度的财报季（Earnings Season）、FOMC 利率决议、巨头 IPO、指数重组，天然带来永不停歇、可预期的市场焦点与交易情绪。
- **目标用户匹配**：面向 Bybit 8,000 万存量用户中关注美股的群体，以及全球持有 xStocks 的真实用户。
- **反哺主业资产**：每一笔 meme 交易必须使用对应的代币化股票（如 NVDAx、TSLAx）作为计价货币（Quote Token），直接拉动 xStocks 的现货买盘与链上留存。

---

## 2. Tape 核心机制设计要点

```
   用户使用 NVDAx 购买
         │
         ▼
 ┌───────────────────────────────────────────────┐
 │ 1. Spend-Gate 额度门禁抗狙击                  │
 │    门额 = f(mETH持仓, xStocks持仓, Bybit等级) │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │ 2. xStocks Bonding Curve (相变式曲线)         │
 │    1B 总量 / 75% 曲线可售 / $12K 美元锚定毕业 │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │ 3. 达到毕业阈值：零迁移 Uniswap v4 相变       │
 │    同一 quote 计价 / 永久锁仓 + Fee Key NFT   │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │ 4. 双创新 Hook 安全保护                       │
 │    - 休市熔断 Hook (预言机偏离触发)           │
 │    - 公司行动 Hook (拆股/分红等价重铸)        │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │ 5. 1% 手续费路由 (写入智能合约)               │
 │    - 45% 创作者 (Fee Key NFT 永续分成)        │
 │    - 25% 反哺 Fluxion xStocks 深度池          │
 │    - 20% 强制回购 MNT                         │
 │    - 10% 协议金库 (Treasury)                  │
 └───────────────────────────────────────────────┘
```

1. **相变式 Bonding Curve**：
   曲线直接以未来 Uniswap v4 池相同的 xStocks 计价。毕业时刻在 Hook 内部完成状态相变，**零资金迁移、零预言机依赖、零 MEV 攻击窗口**，规避 four.meme 因迁移漏洞被盗两次的工程风险。
2. **Spend-Gate 额度门禁**：
   针对 Mantle 2 秒出块与无竞价费率特点，摒弃失效的衰减税，采用额度门禁。打新门额与用户的 **mETH / cmETH 持仓、xStocks 持仓、Bybit VIP 账户等级** 挂钩。参与打新即为主业资产制造锁仓与买盘。
3. **两大行业首创 Hook**：
   - **休市熔断 Hook**：监测美股休市期间链上情绪价格与现货指数的偏离，与 Fluxion RFQ 联动实现自动收敛。
   - **公司行动池层适配 Hook**：解决代币化股票拆股（Split）与分红（Dividend）时，流动性池乘数（Multiplier）脱节导致 LP 被套利的行业痛点。
4. **合约级 MNT 价值捕获**：
   20% 的协议手续费写入不可篡改的合约逻辑，直接在二级市场自动回购 MNT，为 MNT 代币建立起第一个由交易量驱动的真实现金流。
5. **毕业通道**：
   曲线毕业代币 -> Uniswap v4 永久锁池 -> 接入 Fluxion RFQ 主路由 -> 接入 Bybit Alpha 专区（W1 白标） -> 满足公开数据指标后晋级 Bybit 现货。

---

## 3. 北极星 KPI 与反目标

- **北极星指标**：`每周毕业代币数 × 毕业后 7 天存活率`（拒绝欺骗性的日发币量指标）。
- **第一阶段验收红线**：毕业 ≥ 3 个/周，7 天存活率 ≥ 30%，曲线日成交额 ≥ $200K。
- **反目标（坚决不做）**：
  - ❌ 不做通用 meme 曲线（通用路线在 Mantle 必死）。
  - ❌ 不为 meme 开设独立 Appchain（防止割裂流动性）。
  - ❌ 不搞无阶梯退出的高额度流动性补贴。

---

## 4. 本目录文档索引

### 终版报告交付产出（Outputs）
| 文档 | 描述 |
|---|---|
| [`outputs/01-sales-points.md`](outputs/01-sales-points.md) | **【第一部分：产品核心卖点与市场破局方案】** 共享水库破冷启动、外生日历破空气、Uni v4 破迁移夹子、Bot 原生 1-Click、直通 Bybit 上市阶梯（含逐节 TL;DR） |
| [`outputs/02-meme-launchpad-fundamentals.md`](outputs/02-meme-launchpad-fundamentals.md) | **【第二部分：Meme Launchpad 底层原理与架构实现全解】** 面向 Devs 和 CTO 的由浅入深解析：虚拟储备数学推导、状态机全景、权限丢弃、Pull 费用路由、v4 零迁移相变与避坑清单（含逐节 TL;DR） |
| [`outputs/03-launchpad-evolution-and-pitfalls.md`](outputs/03-launchpad-evolution-and-pitfalls.md) | **【产品代际迭代与工程避坑全景】** pump.fun ➔ Base/BSC ➔ Robinhood 踩坑史与五大公理（含逐节 TL;DR） |
| [`outputs/04-defects-and-optimizations.md`](outputs/04-defects-and-optimizations.md) | **【第三章：缺陷抽象与优化方向】** 结构性缺陷图谱；Pons/LONG/PAIR 残留问题；近两周 RH 生态新品原理解析；Mantle P0/P1/P2 优化清单 |
| [`outputs/05-compliance-and-permissioning.md`](outputs/05-compliance-and-permissioning.md) | **【3.7 补遗：合规 D7】** ArbOS Elara / `ArbFilteredTransactionsManager` 与 Uniswap v4 Permissioned Pools 对 Launchpad 的层放错问题；Mantle 制裁放链、股票腿放 Hook 的切法 |
| [`outputs/06-chain-infra-plan.md`](outputs/06-chain-infra-plan.md) | **【第四部分：Mantle 发行链的链级优化方案】** 立项四道闸门；八堵瓶颈墙的实锤盘点；Short（S1 配额 / S4 失败 Drop / S5 发射 RPC，S3 FBA + S6 账户作第二包）与 Long（L0~L7）；S2「瞬时热点吞吐」因无实例被降级为 L0 的完整证据链；分层归属与 D-infra-1~7 决策台账 |

### 核心机制与设计输入
| 文档 | 描述 |
|---|---|
| [`tape-design.md`](tape-design.md) | **【Tape 完整机制设计方案】** 详细数学推导、Spend-Gate 参数、双 Hook 源码级设计、费用流向与分阶段排期 |
| [`xstocks-inputs.md`](xstocks-inputs.md) | **【xStocks 资产约束与代币化 IPO 输入】** Backed ERC-8056 合规边界、Multiplier 特性、代币化 IPO 曲线与可组合性硬约束 |

### 专属链级优化（Dedicated Chain-Infra）
| 文档 | 描述 |
|---|---|
| [`chain-infra/README.md`](chain-infra/README.md) | **【发行层链优化总览】** 为什么 Launchpad 需要专属链优化、代币原语、出块时间免疫与相变 Hook |
| [`chain-infra/short-long-goals.md`](chain-infra/short-long-goals.md) | **【Short / Long 分期定稿】** 第一期只打 Launch Registry（热点配额、失败 Drop、发射 RPC）；S2 瞬时吞吐因无实例降为 L0；ASS / LFM / VM Hook 进后期 |
| [`chain-infra/launchpad-chain-spec.md`](chain-infra/launchpad-chain-spec.md) | **【Meme Launchpad Specific Chain 架构规范】** 调度层（分桶/FBA/Revert Protection）、执行层（ERC-20 兼容 Hook/费用路由）、Uni v4 毕业相变、协议级合规过滤与 Native AA |
| [`chain-infra/shared-liquidity-and-quote.md`](chain-infra/shared-liquidity-and-quote.md) | **【mStocks 计价冷启动与共享流动性架构】** 剖析 155 个股票代币四大冷启动死锁，借鉴 1inch Aqua 打造统一做市水库与 JIT 结算，横向评估 Ethereum FCR 与 Espresso 跨链机制 |
| [`chain-infra/issuance-primitives.md`](chain-infra/issuance-primitives.md) | **【资产发行原生链级原语盘点】** Solana Token Extensions、Cosmos 模块化发行、批量拍卖、转账钩子与 EVM 缺失能力对比 |
### 深度全景复盘（References）
| 文档 | 描述 |
|---|---|
| [`references/launchpad-evolution-analysis.md`](references/launchpad-evolution-analysis.md) | **【Meme Launchpad 四代演进全景复盘】** pump.fun ➔ four.meme ➔ Base (Clanker/Flaunch) ➔ Robinhood (Pons/LONG) 逐代拆解“解决什么痛点”与“遗留什么死锁”，附全景指标对比表与五大公理 |

### 行业基准与竞品调研（Benchmarks）
| 文档 | 调研对象与核心结论 |
|---|---|
| [`benchmarks/solana-baseline.md`](benchmarks/solana-baseline.md) | Solana meme 交易四层栈、pump.fun 精确数学、Ascend 费率阶梯与 MEV 结构 |
| [`benchmarks/solana-pumpfun.md`](benchmarks/solana-pumpfun.md) | pump.fun 链上底层实测、MigrateV2 指令反编译、常数解析与存疑清单 |
| [`benchmarks/gen2-bsc-base.md`](benchmarks/gen2-bsc-base.md) | BSC four.meme（已落地 bStocks 代币化美股 quote）、Binance 白标经验、Base 创作者经济反思 |
| [`benchmarks/bsc-fourmeme.md`](benchmarks/bsc-fourmeme.md) | four.meme 生产 API 实测、Helper3 合约解析、两次被黑安全事故复盘 |
| [`benchmarks/base-ecosystem.md`](benchmarks/base-ecosystem.md) | Base Clanker / Zora / Flaunch 合约拆解与补贴退出暴跌教训 |
| [`benchmarks/robinhood-chain.md`](benchmarks/robinhood-chain.md) | Robinhood Chain 架构、Pons 协议拆解、PAIR / LONG 股票代币 quote 机制与周末脱锚事故 |
| [`benchmarks/robinhood-notes.md`](benchmarks/robinhood-notes.md) | Pons 源码级逆向工程、代币销毁量直读、ERC-8056 乘数机制与监管争议 |
