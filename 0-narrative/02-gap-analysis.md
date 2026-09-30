# 第四部分：Mantle 的现状与 meme Launchpad 失败的真实原因

> 数据截止 **2026-09-06**。本部分标 `【实测】` 的数据为直接读取 Mantle 主网 RPC / Ethereum L1 合约 / DefiLlama & CoinGecko API 得到，可复现。
> 详细底稿见 [`research/E-mantle.md`](../research/E-mantle.md)（954 行，含可复现实测附录）。

---

## 4.0 先纠正术语：「mStocks」不存在

**经全面检索（mantle.xyz 博客与新闻稿、Bybit 公告中心、Backed Finance 官网、PRNewswire / Chainwire / EQS、The Block / CoinDesk / Cointelegraph、DefiLlama、Nansen），Mantle / Bybit / Backed 的任何一手材料中都不存在 "mStocks" 这一产品名。零命中。**
（唯一同名实体是印度 Mirae Asset 旗下的散户券商 "m.Stock"，与加密无关。）

**真实存在的对应物是 xStocks**：

| 维度 | 内容 |
|---|---|
| 品牌 | **xStocks**（后缀 x：NVDAx、AAPLx、MSTRx、TSLAx、METAx、SPCXx） |
| 发行方 | **Backed Finance**（瑞士，2021 成立），**非 Mantle 非 Bybit** |
| 官宣上 Mantle | **2025-11-07**，Mantle + Bybit + Backed 三方 |
| 托管 | 受监管独立托管方，1:1 足额背书 |
| 代币标准 | **ERC-20（Mantle/Ethereum）+ SPL（Solana），可自由转账** |
| CEX 通道 | Bybit 支持经 **Mantle 网络**充提 xStocks |
| 链上场所 | **Fluxion**（主）、Merchant Moe（部分） |
| 公司行为 | "Multiplier" 乘数机制处理拆股/分红 |
| 地域 | **美国公民/美国境内禁止** |
| 2026 进展 | 2026-04 集成 Fluxion；2026-05 上线 xChange 原子 RFQ；**2026-06 上线 SpaceX 代币 SPCXx**；**2026-07 开放周末交易**；Bybit 已将部分 xStocks 接入 Dual Asset 与统一账户保证金抵押 |

**用户"类比 Binance bStocks"的直觉是对的，但对照结果很刺痛：**

| 维度 | **xStocks（Bybit / Mantle）** | **bStocks（Binance / BNB Chain）** |
|---|---|---|
| 发行方 | Backed Finance（**瑞士第三方**） | **BTech Holdings Ltd.（币安关联方，自营）** |
| 监管框架 | Backed 合规代币化框架 | **ADGM/FSRA 批准的招股说明书** |
| 上线 | **2025-11-07** | **2026-06-11**（晚 7 个月） |
| 规模 | 未单独披露 | **2026-07-28 AUM 破 $500M**（首日 $5.6M → 7 周） |
| 标的数 | 初期 10 个 → Q2 末 155 个 | 首发 5 个 → **7 周内 46+** |
| 链上场所 | **Fluxion（专用场所）** | PancakeSwap 等**通用 BSC DeFi** |
| **市场份额** | 全球代币化股票市值 $633.7M（**第 2**） | **BNB Chain 到 2026-08/09 占代币化股票市场约 45–50%** |

> ### ⭐ 四条战略含义
> 1. **Mantle 早了 7 个月，却在增速上输了。** bStocks 7 周做到 $500M AUM 和 46 个标的。
> 2. **差异在自营 vs 代销。** 币安自己发（BTech + 自有招股书），扩标的不受制于人；Mantle/Bybit 依赖第三方 Backed。
> 3. **差异在分发路径长度。** bStocks 直接进币安主站 + BNB Chain 通用 DeFi；xStocks 需要用户「从 Bybit 提到 Mantle 再去 Fluxion」—— **多了两跳**。
> 4. **但 Mantle 有一个 bStocks 没有的东西：Fluxion 这个「专用场所」+ 原子 RFQ（可直接向发行方 mint/redeem）。**
>    **这是做 stock-meme 混合产品时唯一的结构性资产，后续设计应围绕它展开。**

**本报告后续统一使用 xStocks，但为尊重用户语境，在设计章节中沿用「mStocks 类资产」指代 Mantle 上的代币化股票。**

---

## 4.1 停止错误诊断：Mantle 的问题不是「慢」也不是「贵」

这是本部分最重要的一节。几乎所有关于"为什么 meme 在 Mantle 跑不起来"的讨论都建立在错误前提上。

| 常见说法 | 实测结论 |
|---|---|
| "Mantle 太慢" | ❌ **错**。出块 **2.000 秒**【实测】，**区块填充率 0.173%**【实测】，**从未拥堵** |
| "Mantle gas 太贵" | ❌ **错**。**单笔 DEX swap $0.004–0.009**【实测】，比 Base 便宜 |
| "Mantle 提现要 7 天" | ❌ **过时**。**12 小时**（2025-09-23 起，`finalizationPeriodSeconds = 43,200`【实测】） |
| "Mantle 技术落后" | ❌ **错**。ZK Rollup（OP Succinct SP1）+ Ethereum blobs，**L2Beat 归类与 Base 同级** |
| "Mantle 没钱" | ❌ **部分错**。国库 $25.1 亿，但**非 MNT 的硬资产仅 $6.6 亿，流动稳定币仅 $1.2 亿** |
| "Mantle 没试过 meme" | ❌ **错**。**试过两次，砸了 100 万 MNT + EcoFund，都在 2026-08 关停** |

### 4.1.1 链的实测状态

| 参数 | 实测值 |
|---|---|
| 出块时间 | **2.000 秒**（取 5,000 个区块跨度，误差 <0.001s；43,200 块 = 恰好 24 小时）；L1 合约 `l2BlockTime()` 返回 2 |
| **区块 gas limit** | **60,000,000** |
| **平均区块 gasUsed** | **104,016 → 填充率 0.173%** |
| 平均每块交易数 | **1.45 笔 → 约 62,589 笔/日** |
| **base fee** | **恒为 50 gwei（min base fee 下限）** —— 12 个采样点跨 ~80 分钟，即使遇到 39 万 gas 的大块也不动 |
| 理论 TPS 上限 | 按 21,000 gas 简单转账 ≈ **1,428 TPS**；按 120,000 gas swap ≈ **250 TPS** |
| 历史观测峰值 | L2Beat 记录 max UOPS = **25.47**，发生在 **2023-12-27**（上线半年后的空投农耕期，此后再未接近） |
| **当前 UOPS** | **0.13** —— 即**当前活跃度是历史峰值的 0.5%** |

> ### ⭐⭐⭐ 最关键的一条实测发现
> **Mantle 的 EIP-1559 从未被触发。base fee 常年贴着下限运行 —— Mantle 事实上没有费用市场。**
>
> **对 launchpad 的双面含义**：
> - **好处**：费用极度可预测，不会出现 Solana / RH Chain 那种 meme 冲刺时 gas 飙升
> - **坏处**：**没有拥堵定价 = 没有优先级市场。** 抢开盘只能靠 priority fee（在空块环境下几乎无区分度）和网络延迟
>
> **→ 这直接推翻了 outline 里"热点争用隔离 / LFM in EVM"作为 v1 前提的假设。**
> **Mantle 根本没到需要隔离热点的地步 —— 那是 Solana 和 Robinhood Chain 的问题，不是 Mantle 的问题。**
> **但若 Launchpad 成功，它会在几周内变成真问题。** 详见第五部分 5.2 的重新定位。

### 4.1.2 技术栈其实已经"补完作业"

| 升级 | 时间 | 内容 |
|---|---|---|
| **OP Succinct 上线** | **2025-09-16** | 引入 SP1 ZK 有效性证明；提现窗口 **7 天 → 12 小时**（2025-09-23 生效） |
| **Arsia** | **2026-04-16** | 新费用模型 + OP Stack 全分叉对齐 + **DA 迁至 Ethereum blobs、EigenDA 代码路径移除** |

**Arsia 的七项具体内容**：
1. **压缩感知的 L1 data fee**：FastLZ 估算压缩体积 → 线性回归拟合 Brotli 等效大小；`baseFeeScalar` / `blobBaseFeeScalar` 双标量取代旧的 `overhead + scalar`
2. **新增 Operator Fee**：`operatorFeeConstant + operatorFeeScalar × gasUsed × 100`，走新的 `OperatorFeeVault` —— **给 sequencer 新开的一条收入**
3. **⭐ 动态 EIP-1559 参数**：base fee 不再固定，**sequencer 可逐块通过 `extraData` 设定 denominator / elasticity / min base fee**
4. **⭐ DA 足迹感知的区块大小控制**（新增 gas scalar）
5. 一次性对齐 OP Stack 全部分叉（Canyon / Delta / Ecotone / Fjord / Granite / Holocene / Isthmus / Jovian）
6. 节点 rebase 到 op-node v1.16.3；切到标准 OP Stack「一 blob 一 frame」格式
7. 新增 RPC `eth_estimateTotalFee`

> ### ⭐ 第 3、4 条被严重低估
> **outline 问的「弹性可调节区块空间是否可实现」—— 答案是：Arsia 已经把能力做出来了，只是没被使用。**
> sequencer 可以逐块调 EIP-1559 参数 + DA 足迹感知的区块大小控制，**这已经是"可编程区块空间"的雏形。**
> 详见第五部分 5.2.4。

**战略解读**：Arsia 的本质是**「放弃技术差异化，换取生态兼容性」**。Mantle 过去三年的技术特色（EigenDA、自定义编码、独特费用模型）全部被删除。
- **好处**：工具链 / 索引器 / 钱包适配成本骤降
- **坏处**：**再也没有"为什么选 Mantle 而不是 Base"的技术答案**

### 4.1.3 gas 定价的实测参数

`GasPriceOracle @ 0x420000000000000000000000000000000000000F`（version 1.1.0）：

| 参数 | 实测值 | 说明 |
|---|---|---|
| **`tokenRatio()`** | **4,143** | **ETH→MNT 换算系数** |
| `baseFeeScalar()` | 169,019 | |
| `blobBaseFeeScalar()` | **0** | 当前不对 blob 费单独计价 |
| `l1BaseFee()` | 40,734,246 wei | |
| `operatorFeeScalar()` | 100,000,000 | |

**`tokenRatio = 4143` 的交叉验证**：ETH $2,500.64 ÷ MNT $0.6070 = **4,119**。实测值与真实汇率偏差仅 **0.6%** → 证实 `tokenRatio` 就是 **ETH:MNT 现价比**，由治理/预言机定期更新。

**实际单笔成本**（抓真实用户交易逐笔解算）：

| 交易 | gasUsed | 合计 |
|---|---|---|
| `0xaa92bd18…` | 116,219 | **$0.00864** |
| `0xf584db25…` | 117,594 | **$0.00408** |

- **一笔 DEX swap ≈ $0.004 – $0.009**
- L1 data fee 只占总成本 **2–4%** —— 迁到 blob 后 DA 成本已基本可忽略
- L2Beat 统计 Mantle **全年 L1 运营成本合计仅 $4.89K，日均 $13.40**

---

## 4.2 真正的问题：链是空的

### 4.2.1 与 Base / BSC / Solana 的量级对比【实测·同一时点】

| 指标 | **Mantle** | Base | BSC | Solana |
|---|---|---|---|---|
| 链上 TVL | **$97.84M** | $5.669B | $5.792B | $5.925B |
| 稳定币市值 | **$576.12M** | $4.912B | $13.311B | $16.337B |
| **DEX 24h 量** | **$0.94M** | **$567.2M** | **$1,637.4M** | **$1,960.6M** |
| DEX 30d 量 | $67.2M | $24,277.5M | $33,136.5M | $65,028.3M |
| 日活跃地址 | 2,306 / 991 | 238,687 | 1.96M | 2.03M |
| 日交易笔数 | ~54K / 10.9K | 8.59M | 19.13M | 77.57M |
| Perps 24h | **$0** | $146.1M | $18.8M | $551.1M |

**差距倍数（Mantle 相对）**：

| 指标 | vs Base | vs BSC | vs Solana |
|---|---|---|---|
| TVL | **-58×** | -59× | -61× |
| **DEX 24h 量** | **-604×** | **-1,743×** | **-2,087×** |
| 日交易笔数 | -159× | -353× | -1,433× |
| 日活跃地址 | -104× | -850× | -880× |
| **稳定币供应** | **-8.5×** | -23× | -28× |

> ### ⭐⭐⭐ 最关键的一行是「稳定币供应」
> **这是 Mantle 唯一只落后一个数量级的指标。**
>
> **诊断：Mantle 有钱，但没有交易行为。**
> **$576M 稳定币 + $2.5B 国库 + Bybit 8,000 万用户，对应的却是每天 90 万美元 DEX 量和 1,000 个活跃地址。**
>
> **这不是资本问题，是产品与分发问题。**

补充数据：**稳定币存量 $576M 是链上 DeFi TVL（$97.8M）的 5.9 倍。** 大量美元躺在链上**不干活** —— 这些是 Bybit 出入金和 RWA 相关的沉淀资金，不是活跃 DeFi 资本。

### 4.2.2 TVL 的一次教科书式「租赁资本」崩塌

| 日期 | TVL |
|---|---|
| 2026-02 | $137.8M ← 低点 |
| 2026-03 | $516.8M ← **Aave 部署，暴涨 3.75×** |
| **2026-04-16** | **$704.59M ← 历史最高** |
| 2026-04-19 | $492.6M ← 崩塌开始 |
| **2026-04-22** | **$196.3M（4 天 -72%）** |
| **2026-08** | **$62.1M ← 周期最低** |
| 2026-09-06 | **$97.8M** |

**从峰值回撤 −86.1%。**

- **上涨端可确认**：2026 年 2–3 月 **Aave V3 部署到 Mantle**，官方 PR 称「三周内突破 $1B 总市场规模」
- **下跌端存疑**：研究者归因于 **Kelp DAO rsETH 漏洞/脱锚事件**（波及 20+ 网络），Mantle DAO 随后通过 **MIP-34** 向 Aave DAO 提供 **30,000 ETH** 紧急贷款兜底坏账。时间高度吻合但**无官方复盘确认**
- ⚠️ 注意时间巧合：**Arsia 升级也发生在 2026-04-16，正是 TVL 峰值当天**。目前无证据表明 Arsia 导致资金外流，但两事件重叠使单一归因不可靠

> ### ⭐ 对本项目最重要的结论
> **Mantle 的 TVL 高点是用激励租来的**（Aave 部署 + 补贴），资金是 mercenary capital，一遇风吹草动即刻撤离，**且撤离后回不到起点**（$137M → 峰值 $704M → 现在 $97.8M，**比激励前还低**）。
>
> **→ 这直接说明：在 Mantle 上砸钱买 TVL / 交易量是无效的。**
> **Funny Money 和 Printr 的失败是同一个病根。**

### 4.2.3 DEX 格局：三家占 99%

| DEX | 24h 量 | 30d 量 | TVL | 说明 |
|---|---|---|---|---|
| **Agni Finance** | $441,406 | $30.27M | **$16.56M** | **第一大**，Uniswap V3 式集中流动性 |
| **Fluxion Network** | $268,038 | $23.13M | $7.27M | **2025-12-18 主网**。混合 AMM(v2+v3) + **RFQ/订单簿**；**xStocks 的主要链上场所** |
| **Merchant Moe (LB)** | $148,869 | $13.05M | **$12.76M** | Trader Joe LB 式 AMM。MOE 市值仅 ≈$2.2M |
| Uniswap V3 | $4,754 | $389,695 | 极小 | **部署了但基本没人用** |
| **Printr** | **$0** | $6,276 | — | **已关停** |
| Curve / Pendle V2 / Clipper / WOOFi / Native / Swaap 等 **15 个** | **全部 $0** | $0 | — | **已部署但完全休眠** |

**28 个被跟踪的 DEX 中，只有 3 个月交易量超过 $1M。前三名合计占 30 天总量的 99.0%。**

**Mantle 上的 meme 交易对现状**【实测·DexScreener API】：

| 交易对 | 24h 量 |
|---|---|
| WMNT/USDT0 (Agni) | $324,067 |
| WETH/WMNT | $12,185 |
| CATI/WMNT | $4,150 |
| ELSA/WMNT | $3,818 |
| BILLI/WMNT | $3,393 |

> **量最大的 meme 类交易对日成交约 $3,000–4,000。**
> 对比：Solana 单个热门 pump.fun 币开盘几分钟就能做到六位数美元。
> **Mantle 上不存在 meme 市场。**

---

## 4.3 ⭐⭐⭐ 最致命的一条：执行层零覆盖

这是本报告对"为什么 Mantle 跑不起来"的**核心诊断**。

| 工具 | 支持 Mantle | 依据 |
|---|---|---|
| **DexScreener** | ✅ **已实测确认** | API 返回 `chainId: "mantle"`，13 个交易对 |
| **GeckoTerminal** | ✅ **已实测确认** | `/networks` 返回 network id `mantle` |
| **Birdeye** | ✅ | 为 Bybit Alpha 提供实时数据 |
| DEXTools / Ave.ai | ✅ | |
| **OKX DEX 聚合器 / KyberSwap / LI.FI** | ✅ | 官方集成 |
| 1inch | ⚠️ **证据冲突**，需人工确认 | |
| **GMGN** | ❌ **不支持** | 官方链列表：Solana、BSC、Base、Ethereum、Tron、**Monad、HyperEVM、MegaETH、X Layer、Robinhood** —— **无 Mantle** |
| **Photon** | ❌ | Solana 专用 |
| **BullX** | ❌ | Neo = Solana + TRON；Turbo 覆盖 ETH/BNB/Base/Blast/Arbitrum，**无 Mantle** |
| **Axiom** | ❌ | Solana 专用 |
| **Trojan** | ❌ | Solana 专用 |
| **Banana Gun** | ❌ | 支持 ETH/Solana/BNB/Base/**MegaETH**，**无 Mantle** |
| **Maestro** | ❌ | 支持 14+ 链，**无 Mantle** |

**钱包侧同样致命**：

| 钱包 | 默认内置 Mantle |
|---|---|
| Rabby / OKX Wallet / Trust Wallet | ✅ |
| Bybit Web3 Wallet | ✅（**可选，非默认**） |
| **MetaMask** | ❌ **需手动添加网络** |
| **Phantom** | ❌ **不支持 Mantle** |

> ### ⭐⭐⭐ 分发链条在最后一环断裂
>
> | 环节 | Mantle 状态 |
> |---|---|
> | 看盘（charting） | ✅ DexScreener / GeckoTerminal / Birdeye 都有 |
> | 聚合路由（routing） | ✅ OKX / KyberSwap / LI.FI 都有 |
> | **执行（bot / 终端 / 一键狙击）** | ❌ **全部缺席** |
>
> 用户**能看到** Mantle 上的币，也**能换**，但**没有任何一个 meme 交易者习惯使用的执行工具支持 Mantle**。
>
> **GMGN 的链列表里甚至包含了 Monad、MegaETH、X Layer、Robinhood Chain 这些更新更小的链，唯独没有 Mantle。**
> **这说明不是"太新没来得及适配"，而是需求信号不足以让它们适配。**
>
> **这是一个死锁：没有量 → bot 不适配 → 没有执行工具 → 更没有量。**
>
> **→ 任何 Mantle Launchpad 方案必须自带执行层（自建终端 / bot / 移动端），不能指望第三方来接。**
> 这一条与第一部分 1.0（GMGN 吃掉 RH Chain 40% DEX 量）和第二部分 2.2.5（Bankr > 三协议之和 3.9 倍）的观察完全一致：**价值捕获排序是「接口层 > 协议层 > 链层」，而 Mantle 的接口层是零。**

**其他基础设施状态**（相对健康，不是瓶颈）：
- **RPC**：QuickNode（覆盖最完整，含 archive + `debug_traceTransaction`）、Alchemy、Infura、Ankr、Chainstack、dRPC 均支持
- **索引器**：The Graph ✅、Goldsky ✅（Subgraphs + Mirror 实时流）、Covalent ✅、Dune ✅
- **账户抽象**：ERC-4337 可用；**Particle Network** 明确支持（bundler + paymaster）；嵌入式钱包 Privy / Dynamic / Turnkey 均支持；**Mantle 自有的 Mantle Passport（由 Para 提供 MPC）**
- **跨链桥**：官方桥（提现 12 小时）、Stargate、Orbiter（秒级）、Owlto、Relay、Across
- ⚠️ **Solana → Mantle 无原生桥路径**，需走 CEX 中转。**这对承接 Solana meme 用户是重大摩擦**

> **AA / 无 gas 这块是 Mantle 少数基础扎实的领域，且 Mantle Passport 提供了自建入口的可能。这是 Launchpad 设计中可以依赖的资产。**

---

## 4.4 Mantle 的 MEV 现状：一把双刃剑

| 项目 | 状态 |
|---|---|
| 公开 mempool | 单 sequencer 的 OP Stack 链**默认是 sequencer-only 私有 mempool**（无公开 P2P 交易池） |
| 保护型 RPC / MEV-Share | **未找到任何 Mantle 特定实现** |
| Priority fee 拍卖 / builder 市场 | **不存在** |
| Searcher / bot 活动 | **无数据**；结合 base fee 常年触底与 bot 零覆盖，可合理推断**几乎没有** |

> ### ⭐ 这是把双刃剑
> **正面**：没有公开 mempool = **天然抗三明治夹子，狙击手看不到 pending 交易**。对"公平发射"是结构性优势 —— **这是 Mantle 相对 BSC/Ethereum 少见的真实优势点，且从未被宣传过。**
>
> **负面**：没有 bot、没有 searcher = **没有做市深度、没有套利者维持跨池价格、没有开盘流动性**。
> **meme 市场恰恰依赖机器人提供即时流动性。**
>
> **Mantle 消灭了 MEV，也消灭了做市。**
>
> **→ 设计含义**：不能指望第三方 searcher 自然出现来做 launchpad 的开盘流动性与跨池套利，
> **协议必须自带做市/再平衡逻辑**（见第六部分 6.4.3 的 Progressive Bid Wall 与 6.4.1 的 Fluxion RFQ 自动收敛）。

**L2Beat 明列的 CRITICAL 风险**（做设计时必须知情）：
1. 合约收到恶意代码升级即可盗取资金，**代码升级无延迟**
2. 中心化 validator 宕机则资金冻结
3. 运营方可利用中心化地位**抢跑用户交易**（MEV）
- **Exit window → NONE**：合约可即时升级，用户没有退出窗口
- **Stage 0**；proposer 为白名单许可制
- **强制包含**：可直接向 L1 `OptimismPortal` 调 `depositTransaction`。⚠️ 延迟窗口存在源冲突：Mantle 文档写 24 小时，L2Beat 写 "up to 12h"

**Preconfirmation / Flashblocks**：**未找到任何证据表明 Mantle 有 preconfirmation、sub-block streaming、Flashblocks 或 rollup-boost 方案。**
- 对照：Base 已上线 Flashblocks（200ms），Unichain 用 rollup-boost，**Mantle 在这条赛道上完全缺席**

---

## 4.5 Mantle 试过 meme —— 而且是砸钱试的，两次都死了

### 4.5.1 Funny Money（存活约 18 个月）

| 项 | 内容 |
|---|---|
| 定位 | **AI agent launchpad + DeFAI 交易终端**：无代码发币、永续合约 DEX（100+ 资产，50x 杠杆）、"programmatic prompt trading" AI 交易代理、开发者 API |
| 官宣 | **2025-02-18/19**，Chainwire 新闻稿，Mantle DeFi 增长负责人站台，宣传语强调背靠 Mantle **~$4B 国库** |
| **激励** | **100 万 $MNT 奖池**交易赛，公开表述为 *"harness the virality of meme culture"*；前 2 名团队获 **Bybit 独家合作机会 + 联合营销/KOL 资源** |
| 集成 | MetaMask + **Bybit Web3 Wallet** |
| **结局** | **2026-08-16 官网公告关停**："funny.money is sunsetting its trading platform" |
| 数据 | 未被 DefiLlama/DappRadar 单独收录，**无 TVL/交易量可查**；原 mantle.xyz 公告链接现已 302 跳回首页 |

### 4.5.2 Printr（存活约 10 个月）

| 项 | 内容 |
|---|---|
| 定位 | 跨链（"every chain"）bonding curve 发射台，支持 Solana / Mantle / Ethereum / Base / BNB |
| **投资方** | **Bybit Venture Studio、Mantle EcoFund、Axelar Foundation、Sui Foundation、Mirana Ventures、L1 Digital**，融资 **$4.5M**（$2.5M pre-seed + $2M seed 扩展） |
| 上线 | 2025-10-21 |
| **结局** | **2026-08-18 关停**。DefiLlama 上 30 日量残值 $6,276 |

> ### ⭐⭐ 两次尝试，两次死亡，都在 2026 年 8 月的同一周内关停
> **且两次都不是"缺钱"或"缺背书"** —— Printr 有 Bybit + Mantle EcoFund + $4.5M；Funny Money 有 100 万 MNT 奖池 + Bybit 联合营销。
>
> **→ 所以"为什么没跑起来"不能写成"没人试过"。必须给出机制层面的解释。**

### 4.5.3 Mantle 上的 memecoin 现状

| 代币 | 现状 |
|---|---|
| **MINU（Mantle Inu）** | 唯一有持续曝光的原生 meme，总量 420.69M。**事实死亡**：CMC/Coinbase 显示 $0/NaN；小聚合器显示市值 **≈$21,970**、24h 量 **≈$28** |
| $PILL | Funny Money 社区活动奖励代币，随平台关停而死 |
| Printr 发行的各代币 | 平台已关停 |

### 4.5.4 官方激励计划全表 —— 钱全花在了错误的地方

| 计划 | 时间 | 预算 | 机制 |
|---|---|---|---|
| **Mantle Journey — Season Alpha** | 2023-08 → 2024-01 | **2,500 万 MNT** | 链上/链下活动积分 |
| **Methamorphosis S1** | 2024-07 → 2024-10 | Powder → $COOK | **持有/质押 mETH** |
| **Methamorphosis S2** | 2024-10 起 110 天 | Powder → $COOK | **持有/质押 cmETH** |
| **Mantle Rewards Station** | 持续多季 | 逐季不同 | **锁 MNT** → 得 "MNT Power" |
| **COOK Feast** | 持续 | mETH 协议费收入注资 | **锁 COOK** |
| **Mantle EcoFund** | 长期 | **$2 亿**；MIP-26 授权最多 1.2 亿 MNT + 6,000 万 USDx + 30,000 ETH | 战略投资 + 流动性支持 |
| **Mantle Scouts** | 持续 | **$100 万** MNT | 社区提名赠款 |
| **Funny Money 启动奖池** | 2025-02 | **100 万 MNT** | meme 主题交易赛 |

**历史空投分发参与人数**：Ethena (ENA) 25,292 人；EigenLayer (EIGEN) 22,732 人；Methamorphosis (COOK) **30,141 人**。

> ### ⭐⭐ 两条推论
> **① 所有活动的参与者都在 2–3 万人区间。这是 Mantle 真实的可动员用户盘子 —— 相对 Bybit 的 8,000 万注册用户，转化率约 0.03%。**
>
> **② 最大的几笔钱（EcoFund $2 亿、Methamorphosis、Mantle Journey 2,500 万 MNT）全部结构化为「质押/再质押/锁仓」，即"奖励不动的钱"。**
> **Mantle 从来没有建立过奖励高频交易的赌博飞轮。**

### 4.5.5 七条归因（这是本部分的核心结论）

**A. 定位冲突（最根本）**
Mantle 官方定位是「RWA 分发与流动性层」「链上银行」，主推 xStocks、Franklin Templeton/Ondo/Ethena 合作、$2.5B 国库。**这与 bonding curve 赌场文化在品牌、合规、用户画像上全面冲突。一个要 KYC 和招股书的链，很难同时做匿名土狗。**

**B. Bybit 把散户投机"包"在 CeFi 里，而不是导到链上（结构性，最关键）**
Bybit Mantle Vault、**Bybit Alpha**、MNT 作为交易所权益代币 —— 这些设计都让用户**在交易所 UI 内完成投机**，而不是去链上跟合约交互。
> **Mantle 最大的用户漏斗，恰恰是它链上活跃度的最大抑制器。**

对照：four.meme 深度绑定 Binance Wallet（**白标发行引擎**）、pump.fun 绑定 Phantom/Solana Mobile、Base 绑定 Coinbase Wallet —— **三者都是把 CEX/钱包用户直接推进 bonding curve**。

**C. 双代币摩擦（"two-token problem"）**
上链需要：L1 有 ETH → 跨桥 → 再拿到 MNT 付 gas。对比 Solana（单一 SOL）、Base（gas 就是 ETH，与 Coinbase 原生资产一致）。
> **出块 2 秒不是问题，代币与跨桥摩擦才是。**

**D. 没有任何一个 bonding curve 发射台达到逃逸速度**
Funny Money 和 Printr **都是自上而下用机构/VC/Bybit 资金堆出来的，不是从 degen 社区自下而上长出来的**，且都在 ~1–1.5 年内关停。**这指向 PMF 错配，而不是"没人试过"。**

**E. 社区构成本身就是收益农民与机构 DeFi 参与者**
Nansen Q2 2026 报告与 OakResearch 均描述 Mantle 活跃社区在玩 mETH/cmETH、MI4、RWA，**不在猎 meme**。自我强化的负循环。

**F. 激励预算的方向全错**
见 4.5.4 第 ② 条。

**G. 机器化交易生态零覆盖**
见 4.3。

---

## 4.6 与 Robinhood Chain / four.meme 的三点差距对照

| 成功要素 | pump.fun (Solana) | four.meme (BSC) | Pons/PAIR (RH Chain) | **Mantle** |
|---|---|---|---|---|
| **零启动资金的 bonding curve** | ✅ | ✅ | ✅ | ❌ **两个都关停了** |
| **自动毕业 + 锁池，提供叙事弧线** | ✅ LP burn | ✅ V2 + LP burn | ✅ v4 永久锁 | ❌ **无先例；且 Merchant Moe/Agni 合计 TVL 仅 $29M，深度不足以承接毕业** |
| **交易所/钱包级分发直连曲线** | ✅ Phantom / Solana Mobile | ✅ **Binance Wallet 白标复用其引擎** | ✅ Robinhood Wallet 原生 + gas 补贴 | ❌ **Bybit Alpha 反而把用户留在 CeFi** |
| **执行层（bot/终端）** | ✅ 全覆盖 | ✅ 全覆盖 | ✅ GMGN 吃掉 40% DEX 量 | ❌ **零覆盖** |
| **代币化股票作 quote** | — | ✅ **8 个 bStocks 已 PUBLISH** | ✅ **24 只股票白名单** | ❌ **有资产（155 个 xStocks），无 launchpad** |

---

## 4.7 但 Mantle 手里有六张别人没有的牌

这是第六部分设计方案的全部立足点。

1. **无公开 mempool → 天然抗夹子/抗狙击。**
   这是 Mantle 相对 BSC/Ethereum 真实且稀缺的优势，是"公平发射"叙事的硬底座。**从未被宣传过。**

2. **Fluxion 的原子 RFQ（可直接向发行方 mint/redeem）。**
   **开市时段以底层证券实时市场价为锚，提供近乎无滑点执行；休市时段切换到 AMM 执行层维持 24/7 流动性。**
   → **这是 bStocks 在 BSC 上没有、Robinhood Chain 上也没有对位物的专用基础设施。**
   → **它恰好解决了 RWA×meme 最难的问题：周末跳空与 LP 逆向选择。**（对照第三部分 3.4.9：PAIR 至今没有任何链上熔断机制）

3. **xStocks 比 bStocks 早 7 个月，且是可自由转账的 ERC-20**（bStocks 是受限的权益凭证）→ **可组合性更强**。

4. **成本与容量彻底不是瓶颈** —— 6000 万 gas、0.17% 占用率。
   **可以设计非常"重"的链上机制**（复杂 bonding curve、链上撮合、频繁 rebalance、每笔 swap 跑预言机校验）而不担心成本。**这是 Solana 和 BSC 做不到的。**

5. **AA / 嵌入式钱包基础扎实** —— Mantle Passport (Para) + Particle Network，可做完全无助记词、无 gas 的移动端体验。

6. **稳定币存量 $576M 是链上 DeFi TVL 的 5.9 倍** —— 有大量待激活的闲置美元，且它们已经在链上，不需要跨桥。

---

## 4.8 资源约束：国库没有想象中那么多钱

**总规模：$2,514,355,366**【一手·group.mantle.xyz/treasury，2026-09-06】

| 资产 | 美元价值 | 占比 |
|---|---|---|
| **MNT** | **$1,851,738,756** | **73.64%** |
| BTC | $234,245,512 | 9.31% |
| ETH | $222,849,869 | 8.86% |
| **稳定币** | **$119,532,123** | **4.75%** |
| mETH & cmETH | $67,092,729 | 2.66% |
| bbSOL | $15,415,849 | 0.61% |
| COOK | $3,480,525 | 0.13% |

> ### ⚠️ 国库的 73.64% 是自己发的币
> 剔除 MNT 后，**真实的"外部资产"仅约 $6.63 亿**（BTC + ETH + 稳定币 + mETH + bbSOL）。
> **可动用的硬通货约 6.6 亿美元，其中真正的流动稳定币只有 $1.2 亿。**
>
> **→ 对 Launchpad 提案的直接含义**：**能拿出的现金激励远小于"$25 亿国库"给人的印象**，且大额动用 MNT 会砸自己的盘。
> **这也是为什么第六部分的方案必须是"机制驱动"而非"补贴驱动"。**

**MNT 代币经济的另外三条约束**：
- **无常规团队/投资人归属计划**；供应 ~51–53% 流通 + **~47–49% 国库**（DAO 治理释放）
- **MNT 没有"解锁抛压日历"，但有"治理随时可放 47% 供应"的悬顶之剑**
- **未找到任何经治理批准的、收入驱动的 MNT 回购或销毁机制**（经多次定向检索确认为"确实不存在"）。**MNT 是纯粹的 gas + 治理 + 交易所权益代币，没有现金流捕获。**
  → **这反而是一个机会：launchpad 可以成为 MNT 的第一个真实现金流来源。** 见第六部分 6.3.7。

---

## 4.9 一个必须正面回应的事实：Bybit 自己用真金白银投了 Solana

**Byreal 是 Bybit 自己孵化的 DEX，建在 Solana 上，不是 Mantle。**
混合 CEX/DEX 模型（RFQ + CLMM），接入 Raydium/Orca/Meteora 流动性，含 bbSOL（Bybit 的 Solana LST）金库。2025-06 官宣。

**未找到 Bybit 官方解释。** 合理推断（非官方证实）：
1. Solana 的散户/meme 交易量与既有 DEX 流动性远超 Mantle，冷启动容易得多
2. Bybit 已在 Solana 有 bbSOL 质押产品，可复用
3. Solana 的低延迟更适合 CEX 级 RFQ
4. 分工定位 —— Mantle 已被定为 Bybit 的 "CeDeFi/RWA 链"，Byreal 去 Solana 抓另一批散户

> ### ⚠️ 无论原因如何，结论是硬的
> **Bybit 用真金白银投票，认为散户交易应该发生在 Solana，而不是 Mantle。**
> **这是任何 Mantle Launchpad 提案都必须正面回应的问题。**
>
> 我的回应见第六部分 6.0 前提三与 6.5：
> **Mantle 不应该、也不可能去抢 Byreal 的散户现货交易场景。**
> **它应该去做一个 Solana 结构上做不了的东西：以合规代币化股票为 quote 资产的发行与投机层。**

---

## 4.10 本部分小结：Mantle 的 meme launchpad 失败原因（分两部分回答 outline）

### 4.10.1 产品缺陷

| # | 缺陷 | 证据 |
|---|---|---|
| 1 | **两个 launchpad 都是自上而下用机构资金堆出来的**，不是从 degen 社区自下而上长出来的 | Funny Money（100 万 MNT 奖池）、Printr（$4.5M VC + Bybit/EcoFund 背书），均在 2026-08 同周关停 |
| 2 | **Funny Money 定位混乱**：AI agent launchpad + DeFAI 终端 + 50x 永续，什么都做等于什么都不做 | 官方描述 |
| 3 | **Printr 做的是"跨链通用 launchpad"**，在 Mantle 上没有任何 Mantle 特有的东西 | 支持 Solana/Mantle/ETH/Base/BNB 五链 |
| 4 | **没有毕业目的地的深度** —— Merchant Moe + Agni 合计 TVL 仅 **$29M** | 实测 |
| 5 | **没有把 Mantle 的独特资产（xStocks / mETH / Fluxion RFQ）用进去** | 两个平台都没做 stock-quote |
| 6 | **没有自建执行层**，指望第三方 bot 来接 | GMGN 等零覆盖 |
| 7 | **没有可见的"阶梯"** —— 毕业之后是虚无，没有 Alpha → 现货式的晋级路径 | 对照 four.meme 的四个案例 |

### 4.10.2 链 Infra 缺陷

⚠️ **注意：这一节的答案与直觉相反 —— Mantle 的链 infra 缺陷不在性能，而在"缺少 meme 所需的特定能力"。**

| # | 缺陷 | 严重度 | 说明 |
|---|---|---|---|
| 1 | **执行层生态零覆盖**（GMGN/Photon/BullX/Axiom/Trojan/Banana/Maestro 全部不支持；Phantom 不支持） | **致命** | 这不是链参数问题，是网络效应问题 |
| 2 | **无费用市场 = 无优先级机制**（base fee 常年钉在 50 gwei 下限） | **高** | 开盘公平性无法靠 gas 竞价解决，**必须协议层机制**（批量拍卖/承诺-揭示/额度门禁） |
| 3 | **无 preconfirmation / 软确认流** | **高** | 2 秒出块 + 无 200ms 预确认 = 反馈环太长；且 **2 秒粒度让 Pons 式衰减税失效**（15 秒只有 7-8 个区块） |
| 4 | **双代币摩擦**（ETH 跨桥 → MNT 付 gas） | **高** | 需 paymaster 费用抽象解决 |
| 5 | **Solana → Mantle 无原生桥** | 中高 | 承接 Solana meme 用户的重大摩擦 |
| 6 | **无 searcher/做市 bot** | 中高 | 私有 mempool 的代价：消灭了 MEV，也消灭了做市 |
| 7 | **无 sequencer 级合规过滤能力** | 中 | 这是把证券型代币放上无许可 launchpad 的前置条件（Robinhood 有，Arbitrum 已产品化为 ArbOS Elara） |
| 8 | **热点争用隔离 / LFM** | **当前无关，未来必需** | 填充率 0.173%，**现在做是无病呻吟；但若 launchpad 成功会在数周内变成真问题**，应作为 v2 预留设计 |
| 9 | Stage 0 + Exit window NONE + 合约可即时升级 | 中 | 对机构资产是治理风险 |

> ### ⭐ 对 outline 第 9–17 行假设的逐条校验
>
> | outline 假设 | 实测校验 |
> |---|---|
> | **瞬时执行容量** | **非瓶颈**。60M gas / 2s，占用率 0.17%，**冗余超过 500 倍** |
> | **热点争用隔离 / LFM in EVM** | **当前无意义**（从未拥堵，base fee 从未离开下限）。**但若成功会立刻变成真问题** —— 应作为 **v2 预留设计，非 v1 前提** |
> | **弹性可调节区块空间** | **Arsia 已具备能力**：sequencer 可逐块通过 `extraData` 设定 EIP-1559 denominator/elasticity/min base fee，且有 DA 足迹感知的区块大小控制。**能力已就位，只是没被使用** |
> | **发行与流动性组合** | Mantle **没有**任何存活的 bonding curve 产品。**Fluxion 的 AMM+RFQ 混合架构是最接近的可复用底座** |
> | **毕业后锁定流动性** | 无先例。**Merchant Moe/Agni 合计 TVL $29M，深度不足以承接毕业** —— 这是必须解决的设计约束 |
> | **机器化交易生态** | **最大短板** |
