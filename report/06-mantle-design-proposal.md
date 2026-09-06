# 第六部分：Mantle 上可行的 mStocks meme Launchpad 设计方案

> 代号 **Tape**（取自股票报价带 "ticker tape"，也谐音"胶带" —— 把 meme 与股票粘在一起）。
> 本部分是全报告的落点。所有设计决策都可回溯到前五部分的具体证据。

---

## 6.0 设计前提：先承认五件事

任何跳过这一节的方案都会跑偏。

### 前提一：Mantle 打不赢通用 meme launchpad 的战争

pump.fun 累计手续费 **$1.21B**，Pons 54 天做到 **$72.48M** 且当前日流水是 pump.fun 的 13 倍，four.meme 累计 **$98.05M**。
**Mantle 再做一个"更好的 pump.fun"，只会是第 47 个死掉的 clone —— 而且 Mantle 自己已经死过两次（Funny Money、Printr，均在 2026-08 同周关停）。**

> **任何"我们也做个 bonding curve"的方案都应当被直接否决。**

### 前提二：Mantle 唯一能赢的战场是「代币化股票原生的投机」，但这个赛道已经有两个玩家

**必须先破除一个幻觉**：「meme × 代币化股票」不是空白市场。

| 玩家 | 链 | 形态 | 规模 |
|---|---|---|---|
| **PAIR / LONG** | Robinhood Chain | 24 只股票白名单作 quote；multipool 篮子 | PAIR 累计手续费 $433K；LONG 上的 Artificial Inu(NVDA) 市值 ~$275M |
| **four.meme Stock Meme** | BNB Chain | **8 个 bStocks 已 PUBLISH**（NVDAb/QQQb/HOODb/SPCXb/GMEb/DJTb/MRNAb/FLNCb）；**链上实测 6 笔毕业里 5 笔是 bStocks 计价** | 已成为 four.meme 当前毕业量的主力 |

**所以问题不是"可不可行"，而是"凭什么在 Mantle 做"。**

**答案是三条 Mantle 独有的结构性资产**（详见 6.1）：
1. **Fluxion 的 Atomic RFQ**（开市锚定实时价 / 休市切 AMM）—— **RH Chain 与 BSC 都没有对位物**
2. **xStocks 的规模**（$633.7M 市值，全球第 2，是 Robinhood 的 4.8 倍）+ **155 个标的** + **可自由转账的 ERC-20**
3. **mETH / cmETH 的生息基础设施** —— 可让 quote 资产默认生息

### 前提三：不能让 meme 污染 RWA 主业

Robinhood Chain 正在现场演示这个失败：meme 活动让 base fee **11 天涨 82 倍**、平均 tx 费到 **~$0.32–0.40**。

**Mantle 的品牌承诺是机构级 RWA 结算。如果 xStocks 的结算成本被 meme 抬高，这个方案就是负价值的。**

### 前提四：钱不够，必须是机制驱动而非补贴驱动

国库名义 $25.1 亿，但 **73.64% 是 MNT 自己**；非 MNT 硬资产约 **$6.63 亿**，**真正的流动稳定币只有 $1.2 亿**。

**并且 Mantle 已经证明了砸钱无效**：
- TVL 用 Aave 激励从 $137M 拉到 $704M，撤离后跌到 $62M —— **比激励前还低**
- Funny Money 有 100 万 MNT 奖池 → 关停
- Printr 有 $4.5M VC + Bybit + EcoFund → 关停

### 前提五：Bybit 已经用真金白银投票给了 Solana

**Byreal（Bybit 自己孵化的 DEX）建在 Solana 上，不是 Mantle。**

> **所以 Mantle 不应该、也不可能去抢 Byreal 的散户现货交易场景。**
> **它应该去做一个 Solana 结构上做不了的东西：以合规代币化股票为 quote 资产的发行与投机层。**
> （Solana 上虽有 xStocks，但缺少 Fluxion 式的原子 RFQ 与 Bybit 的一级市场通道。）

> ### ⭐ 由此得出的一句话定位
> **不是「Mantle 版 pump.fun」，而是「代币化股票的情绪衍生层」——**
> **让每一个 xStocks 标的都拥有一个可交易的、社区驱动的情绪市场，**
> **并且这个市场的所有现金流最终回流到 xStocks 的流动性、Fluxion 的深度、和 MNT。**

---

## 6.1 Mantle 的三张独有底牌（方案的全部立足点）

### 6.1.1 底牌一：Fluxion 的 Atomic RFQ —— 解决 RWA×meme 最难问题的现成引擎

**Fluxion** = Mantle 原生全栈 DEX，2025-12-18 主网上线。三个模块：AMM V2 池 / AMM V3 集中流动性 / **Atomic RFQ**。

**Atomic RFQ 的关键机制**：
- **允许用户按实时市场报价、直接通过发行方（xStocks）铸造/赎回**
- **开市时段**：以底层证券的**实时市场价**为锚，提供机构级、**近乎无滑点**的执行
- **休市时段**：协议**切换到 AMM 执行层**，维持 24/7 连续流动性

> ### ⭐⭐⭐ 这为什么是决定性的
> 回看第三部分 3.4.9 与 3.2.5：**Robinhood Chain 上「股票代币作 quote」的全部七类风险中，最严重的三类（休市陈旧价、周一跳空、逼空囤积）都源于同一个根因 —— 链上价格与真实价格之间缺少一个连续的、自动的再锚定机制。**
>
> **Robinhood 唯一的解法是"打电话让唯一 AP（Bitstamp）手动增发"**（HIMS 事件中实际发生）。
> **PAIR 至今没有任何链上熔断机制**，官方策略是"篮子分散 + 信息披露"。
>
> **而 Mantle 已经有一个链上的、自动的、双模的再锚定引擎。**
> **这不是"我们也可以做"，这是"别人做不了而我们已经有了"。**

### 6.1.2 底牌二：xStocks 的规模、标的数与可组合性

| 维度 | xStocks（Mantle） | Robinhood Stock Tokens |
|---|---|---|
| 全球代币化股票市值排名 | **第 2（$633.7M）** | 第 6（$133.2M） |
| Mantle 上标的数 | **155**（Q2 2026 末） | 190+（RH Chain） |
| 代币标准 | **可自由转账的 ERC-20** | 可自由转账的 ERC-20 |
| 独家标的 | **SPCXx（SpaceX）等代币化 IPO** | SPCX 亦有 |
| 一级市场 | Backed，经 Fluxion RFQ | Bitstamp（唯一 AP） |

**关键**：xStocks 是**标准 ERC-20，可自由转账**（限制在一级市场 KYB 与前端 KYC 层，不在代币层）。
**→ 结构上，Mantle 版 PAIR 是可行的，不需要额外的合规包装层。**

> ⚠️ **上线前必须核验的三件事**（这是整个方案的技术前置条件）：
> 1. **Mantle 侧 xStocks 的具体 mint/合约配置是否启用了 transfer hook 或黑白名单**
>    （Solana 侧使用 Token Extensions 的可编程合规能力；EVM 侧需单独确认）
> 2. **Backed 的 "Multiplier" 机制是否等价于 ERC-8056** —— 若是 rebase，**整个永久锁仓 LP 模型会在第一次分红时崩溃**
> 3. **Backed 是否接受其资产被用作第三方 permissionless launchpad 的 quote 资产**（法务边界）

### 6.1.3 底牌三：mETH / cmETH —— 让 quote 资产默认生息

**Flaunch 已经验证了这个模式**：flETH（1:1 ETH 背书的包装）闲置在池里的背书资产被扫进 `AaveV3Strategy` 吃 Aave 收益，**这就是 DefiLlama 上 Flaunch "Revenue" 那一行的来源（$3.07M），而不是手续费抽成**。

**Mantle 有更好的原料**：mETH（TVL $590.58M）、cmETH，以及链上 $576M 的闲置稳定币（是链上 DeFi TVL 的 5.9 倍）。

### 6.1.4 另外三张次要底牌

4. **无公开 mempool → 天然抗夹**（结构性优势，从未宣传）
5. **成本与容量彻底不是瓶颈**（60M gas、0.17% 占用率、$0.004–0.009/笔）→ **可以设计非常"重"的链上机制**（每笔 swap 跑预言机校验、频繁 rebalance、链上撮合），**这是 Solana 和 BSC 做不到的**
6. **AA 基础扎实**（Mantle Passport / Para MPC + Particle bundler & paymaster）

---

## 6.2 产品全景

```
┌──────────────────────────────────────────────────────────────────────┐
│  Tape 协议栈                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  ① Ticker Pairs      用 xStocks 作 quote 的发行（对标 PAIR / Stock Meme）│
│  ② Sentiment Engine  财报/事件日历驱动的定期情绪市场（原创）             │
│  ③ Graduation Rail   毕业 → Fluxion 深池 → Bybit Alpha 上架流水线       │
│  ④ Tape Terminal     自建执行层：终端 + bot + 移动端（不可省略）         │
├──────────────────────────────────────────────────────────────────────┤
│  核心机制层                                                            │
│  • Oracle-Deviation Circuit Breaker Hook（休市熔断）★ 无人实现          │
│  • Corporate-Action Rebalance Hook（公司行动池层适配）★ 无人实现        │
│  • Spend-Gate 额度门禁（绕开 2s 出块对衰减税的限制）                    │
│  • Progressive Bid Wall（薄流动性下的价格支撑）                        │
│  • Fee Router：creator / MNT 回购 / **xStocks 流动性反哺**              │
├──────────────────────────────────────────────────────────────────────┤
│  基础设施配套                                                          │
│  • Paymaster 费用抽象（消除双代币摩擦）                                │
│  • Sequencer 合约分桶（v1 埋钩子，v2 启用配额）                        │
│  • Sequencer 级合规过滤（把证券放上无许可 launchpad 的前置条件）        │
│  • 预确认流（Flashblocks 式 200–250ms）                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6.3 模块一：Ticker Pairs —— 核心发行机制

### 6.3.1 整体架构：单 pool 内的相变，不做迁移

采用 **Pons V2 的哲学 + Uniswap v4 hook 的实现**：

> **曲线用它未来 v4 池将要使用的同一个 quote 资产计价。因为曲线从第一笔交易起就在收集最终池所需的资产，毕业时直接 seed 池子 —— 无 router、无 swap、系统中任何地方都没有预言机。**

**具体流程**：

```
1. createTicker(name, symbol, quoteAsset, weights[], gateConfig)
   └─ 单笔原子交易内：
      ├─ CREATE2 部署固定供应 ERC-20（1,000,000,000，无 mint 函数）
      ├─ 部署/注册 curve 状态到 singleton hook
      ├─ 写入 gate 配置（spend-gate 参数、expiry）
      └─ 可选 atomic dev-buy

2. 曲线阶段（纯 x·y=k，零预言机）
   ├─ 用 quoteAsset（qNVDAx 等）计价
   ├─ Spend-Gate 门禁生效（见 6.3.5）
   ├─ 熔断 hook 监控预言机偏离（见 6.4.1）
   └─ 进度条：已募集 X / 毕业线 Y

3. 毕业（跨越阈值那笔买单内部自动触发，包 try/catch）
   ├─ 曲线停止，100% 储备交给 factory
   ├─ 直接铸造 full-range Uniswap v4 头寸
   ├─ NFT 送进无出口的 locker（永久锁定）
   ├─ 铸造 Fee Key NFT 给 creator
   └─ 自动在 Fluxion 建立 quote↔USDT 路由

4. AMM 阶段
   ├─ 同一个 singleton hook 的 afterSwap 继续收费
   ├─ PBW 用社区侧手续费在现价下方挂买单
   └─ 熔断 hook 继续生效
```

**为什么不做迁移**：
- four.meme 用 **$183,000 + 200 BNB** 两次被黑买到的教训是：**"毕业迁移"是攻击面最集中的一步**。V3/CLMM 迁移需要传入价格参数 → 天然引入 `sqrtPriceX96` 校验漏洞
- Pons V2 的方案比 four.meme 的"退回 V2"更优雅：**根本不需要价格参数**

### 6.3.2 曲线参数（含推导）

采用 pump.fun / Pons V2 家族的**带虚拟储备恒定乘积**：

```
amountOut = amountIn·(10000 − feeBps)·reserveOut
            ─────────────────────────────────────────
            reserveIn·10000 + amountIn·(10000 − feeBps)

reservedTokens = supply · phantomQuote / (phantomQuote + graduationThreshold)
```

`reservedTokens` 在初始化时固定 → **保证"曲线卖完点"与"毕业点"重合，永远不会有剩余代币**（Pons V2 的设计，直接抄）。

**建议参数**：

| 参数 | 建议值 | 推导依据 |
|---|---|---|
| 总供应 | **1,000,000,000**，无 mint 函数 | 行业惯例（Pons / PAIR / Zora 均为 1B） |
| 曲线可售 | **75%**（750M） | 介于 pump.fun 79.31% 与 four.meme 63.7% 之间；留 25% 进 LP |
| 毕业时进 LP | **25%**（250M） | 高于 pump.fun 的 20.69%，因为 **Mantle 的毕业目的地深度不足**（Merchant Moe+Agni 仅 $29M），需要自带更厚的池 |
| 起始 FDV | **≈ $5,000 等值** | 与 pump.fun（27.96 SOL ≈ $5k @$180）、four.meme（5.74 BNB）同量级 |
| **毕业阈值** | **$12,000 等值 quote**（治理可调，按标的分档） | **必须低于 pump.fun 的 $14,276**。理由：Mantle 日 DEX 量是 RH Chain 的 1/1,630，**阈值必须低才有毕业率**；同时高到能形成有意义的 LP |
| 内盘涨幅空间 | 由上述参数推出约 **12–15×** | 与 pump.fun 14.70× 同量级 |
| 创建费 | **0**（只付 gas，约 $0.01） | 抄 four.meme。**发币本身不该是收入来源** |
| 交易费 | **1%** | 与 Pons / PAIR / Clanker / Zora 全行业对齐 |

**⚠️ 毕业阈值必须锚定美元而非代币数量**（抄 four.meme，见第五部分 5.3.3）：
- 155 个 xStocks 的单价差几个数量级（NVDAx vs 某仙股）
- 若用纯代币数量阈值，用户无法理解不同池的门槛
- **实现方式：治理定期调参（如每周），而非实时预言机** —— 保证曲线交易本身完全不碰预言机

### 6.3.3 quote 资产：直接用 xStocks，但加一层可选的生息包装

**基础方案（v1）：直接用 xStocks 作 quote。**
因为 xStocks 是可自由转账的标准 ERC-20，技术上无障碍（**待 6.1.2 的三项核验通过**）。

**增强方案（v2）：`qToken` 生息包装。**

```
用户持有 NVDAx
      ↓ deposit
┌────────────────────────────────┐
│  Tape Quote Vault (per asset)  │
│  持有真实 NVDAx                 │
│  闲置部分投入 Fluxion LP / 借贷  │  ← 抄 Flaunch 的 flETH→Aave 模式
└────────────────────────────────┘
      ↓ mint（价格递增型，非 rebase）
   qNVDAx  ← 作为曲线与 v4 池的 quote 资产
```

**为什么用价格递增型而非 rebase**：与 ERC-8056 同理 —— **rebase 会炸掉所有 AMM 池**。

**三个额外好处**：
1. **quote 资产默认生息** → LP 的机会成本降低 → 愿意提供更深的流动性
2. 金库收益可用于**补贴 launchpad 的做市成本**（Mantle 没有自然 searcher，见第四部分 4.4）
3. 若 Backed 对"直接用 xStocks 做 permissionless quote"有顾虑，**金库层可以作为合规收敛点** —— 用户与 Backed 之间的合规关系收敛到一个可审计的合约地址

**⚠️ 但 v1 不应依赖 v2** —— 先用裸 xStocks 验证需求，再上包装层。

### 6.3.4 白名单与标的治理

抄 PAIR（24 只白名单）而非全开放 155 个：

| 层级 | 标的 | 说明 |
|---|---|---|
| **Tier 1（首发）** | NVDAx / TSLAx / AAPLx / MSTRx / **HOODx** / SPYx / QQQx | 深度最好、叙事最强 |
| **Tier 2** | METAx / GOOGLx / CRCLx / 热门个股 | 逐步开放 |
| **Tier 3（特批）** | **SPCXx 等代币化 IPO / pre-IPO** | 需单独风控（见 6.4.4） |
| **黑名单** | 仙股、争议标的、被 CEO 公开反对的标的 | **预设治理流程，不要等被点名** |

**并且必须预设 "delisting 路径"**：
AMC 事件的教训是 —— **上市公司 CEO 会公开反弹**（Adam Aron 称其 "contemptible"、"vile"、"fictitious synthetic equity market"，威胁向 SEC 投诉）。
**Robinhood 敢硬刚是因为它有前 SEC 委员当 CLO、代币不向美国人发售、发行人在泽西岛。Bybit/Mantle 的法律隔离结构不同，不能照抄这个姿态。**

> **建议：在协议层内置"标的冻结"开关**（停止新发射 + 现存池进入只可卖出模式），并把触发条件写进治理文档。

### 6.3.5 ⭐ 抗狙击：Spend-Gate 额度门禁（绕开 2s 出块的硬约束）

**问题回顾**（第五部分 5.3.5）：
Mantle 2 秒出块下，15 秒的衰减税只有 **7–8 个区块** → 衰减曲线退化成阶梯函数 → 抢跑者只需算出"第几个区块的税率低于预期利润"，然后在那个区块集中开火。
**→ Pons 式的 99% 衰减税在 Mantle 上失效。**

**并且 Mantle 还有第二个问题**：base fee 常年钉在下限，**priority fee 在空块环境下几乎没有区分度** → 连"用 gas 竞价"这条路都走不通。

**解法：抄 Flaunch Game Mode 的 `spend-gate`，但把"游戏"替换成 Mantle 生态原生的凭证。**

```
发射时写入 pool 的门禁参数：
  ├─ windowStart / windowEnd（写进池子，非服务器）
  ├─ perWalletCap（累计强制）
  ├─ trustedSigner（额度授权的签名者）
  └─ expiry（★ 过期后门禁自行停止执行，不需要任何人发交易）

门禁期内：
  ├─ 未持有有效授权的 swap → revert（池子自己的规则）
  ├─ 授权带 maxSpend，池子按 swap 的真实 quote 输入量比对
  └─ perWalletCap 累计强制

额度来源（可组合，由 creator 选择）：
  ① mETH / cmETH 持仓时长快照
  ② 对应 xStocks 标的的持仓证明（★ 最契合本方案）
  ③ Bybit 账户等级 / KYC 等级（经 trusted signer 签名）
  ④ Mantle 生态任务完成度
  ⑤ 链下游戏（Flaunch 原版）
```

**为什么这解决了所有问题**：

| 问题 | Spend-Gate 的解 |
|---|---|
| 2 秒出块让衰减税失效 | **完全不依赖出块时间** |
| 无费用市场，无优先级机制 | **完全不依赖 gas 竞价** |
| 无公开 mempool，bot 看不到 pending（本应是优势，但也意味着无法用 mempool 级防护） | **门禁在 hook 里执行，与 mempool 无关** |
| bundler 集群多钱包绕过 | **perWalletCap 累计强制 + 额度来源本身有成本**（mETH 持仓时长 / xStocks 持仓无法凭空造） |
| 依赖服务器 | **expiry 写进池子，服务器挂了代币也会按时进入公开市场** |

**Flaunch 的真实数据佐证**：

| 时点 | 已售供应 | 持有钱包数 |
|---|---|---|
| 10 秒 | 0.72% | **2** |
| 60 秒 | — | 138 |
| 窗口关闭 | **35.55%** | **194** |

对照组：一个 **15 钱包的 bundler 集群，窗口内买入 0.00%**，窗口结束后才在公开市场买 9.27%，**价格是玩家们已经定好的**。
官方总结：***"For the first time, the players front-ran the bundlers."***

> ### ⭐⭐ 这是本方案最重要的机制选择
> **它把 Mantle 的两个劣势（2 秒出块、无费用市场）从"必须修复的缺陷"变成"无关变量"。**
> **并且它天然地把 xStocks 持仓变成了参与 meme 的门票 —— 直接为主业创造需求。**

**补充第二层：抄 four.meme 的 Token Name Protection**
代币在曲线期持币人数达 **100+** 时，其 name/ticker 被**锁定 72 小时**，期间禁止创建同名/近似名代币；设计成"几乎同时创建的两个可以都成功"，防机器人批量抢注。

### 6.3.6 费用结构与路由 —— 全方案的经济灵魂

**基础费率：1%（买卖双向）**，池子的 v4 原生 `fee` 设为 0，全部收费走 hook（抄 Flaunch / Pons）。

**分配（这是与所有竞品最大的差异）**：

```
1% 交易费
├── 45%  → Creator（通过 Fee Key NFT 永续领取，即时可 claim）
├── 25%  → ★ xStocks 流动性反哺金库（回流到 Fluxion 的对应 quote 池）
├── 20%  → MNT 回购（TWAP 执行，写进合约而非博客）
└── 10%  → 协议金库（运营 / 安全审计 / 做市 / 预言机成本）
```

**为什么第 2 条是灵魂**：

| 竞品 | 协议收入去向 | 对底层生态的贡献 |
|---|---|---|
| Pons | 80% 回购销毁 $PONS（**且这是"政策"不是合约条款，团队随时可调**） | **零** |
| PAIR | 90% 回购销毁 $PAIR | **零** |
| pump.fun | 回购 PUMP | 零 |
| **Tape** | **25% 反哺 xStocks 流动性 + 20% 回购 MNT** | **直接加深主业** |

**这形成一个跨业务飞轮 —— 这是 Robinhood Chain 结构上做不到的**（RH 的 meme 与 stock token 是两个互不输血的业务，PAIR 只是把股票代币当燃料烧掉）：

```
meme 交易量 ↑
    ↓
xStocks 在 Fluxion 的深度 ↑  +  MNT 回购 ↑
    ↓
机构愿意在 Mantle 结算 RWA ↑  +  MNT 有了第一个真实现金流来源
    ↓
更多标的、更好深度、更多叙事
    ↓
（回到顶部）
```

**技术实现参考**：OpenFour 的 **`Ecosystem Buyback Royalty`**（A 币的交易费自动去买 B 币，再销毁/分红/生态激励）—— **这正是"meme 手续费反哺 mStocks 流动性"的现成模板。**

**并且抄 four.meme Universal Subscription 的费率哲学**：
> **「钱进 LP 就免平台费，钱进项目方口袋就收 10%」** —— 在经济上直接把项目方推向"多留流动性"。
> 对应到 Tape：**creator 若选择把自己那 45% 的一部分转投 PBW，则该部分免除协议 10% 抽成。**

**⚠️ 一个必须避开的坑（Flaunch 的反面教材）**：
Flaunch 第一个 `PositionManager` 的 `protocolFeeRecipient` **被写死为基金会多签，不可更改**，导致 4,755 枚代币的费用永久绑定一个可能过时的治理主体，且基金会**法律上没有义务**用于回购。
> **→ Tape 必须把所有 recipient 设计成可治理的指针，且把回购比例写进合约而非博客。**

**Fee Key NFT（抄 Flaunch Memestream + Pons locker）**：
- 毕业时铸造，代表该代币 creator 手续费流的独占权利
- **可自由转让** → 解锁：卖掉未来现金流 / 抵押借贷 / 碎片化 / **真正的 CTO（社区接管）市场**（creator 跑路后社区可买下收入流 + 管理权）
- **收入永远以 quote 资产计价**（抄 Pons：hook 在**受价格冲击上限约束**下把 meme 币换回 quote，`maxInternalPriceImpactBps` 默认 300 = 3%）→ **协议/创建者收入不会变成没人要的 meme 灰尘**

### 6.3.7 货币层级（抄 Zora，为 MNT 创造结构性买盘）

```
meme 币  ──quote──>  qNVDAx / xStocks  ──路由──>  USDT / USDC
                          │
                          └──手续费 25%──> Fluxion 的 xStocks 池
                          └──手续费 20%──> MNT 回购
```

**可选的更激进版本（v3）**：让部分 meme 币的 quote 直接是 **MNT**，形成 `meme → MNT → xStocks` 的三层结构。
但这会重新引入 MNT 波动性，**建议先不做**。

---

## 6.4 核心机制层：四个 Mantle 独有的机制创新

这四条是"凭什么在 Mantle 做"的技术答案。**其中前两条在全行业无人实现。**

### 6.4.1 ★ Oracle-Deviation Circuit Breaker Hook（休市熔断）

**要解决的问题**（第三部分 3.2.5 的三起真实事故）：
- **HIMS / BONER**：周末把 HIMS 打到 **4.6 倍**背离，靠 AP 手动增发才救回
- **AMC 池**：周末冲到最后参考价的 **~35 倍**
- **Farmmi / JINQIAN**：链上假币投机疑似**反向传导到真实纳斯达克市场**（FAMI 当日盘中 +321%）

**Robinhood Chain 的现状：零协议级熔断。** PAIR 的 "peg guard" 只暂停新发射，不管存量池。

**Tape 的设计**：

```solidity
// beforeSwap 中执行
function _checkDeviation(PoolKey key, ...) internal view {
    (int256 refPrice, uint256 updatedAt) = oracle.latestRoundData();
    MarketState state = calendar.state();     // OPEN / EXTENDED / CLOSED / HALTED

    uint256 poolPrice = _getPoolPrice(key);
    uint256 dev = _abs(poolPrice - refPrice) * 1e4 / refPrice;

    if (state == CLOSED) {
        // 休市：参考价 = 最后收盘价
        if (dev > closedBandBps)      revert DeviationHalt();   // 硬熔断
        if (dev > closedWarnBps)      feeOverride = penaltyFee; // 惩罚性费率
        // 且只对"扩大偏离的方向"加征，收敛方向正常费率
    } else if (state == OPEN) {
        if (dev > openBandBps)        revert DeviationHalt();
        // 开市时偏离应当很小，因为 Fluxion RFQ 在做锚定
    }
    // staleness 是主要防线（官方明确提示 oraclePaused 是 advisory）
    if (block.timestamp - updatedAt > maxStaleness && state != CLOSED)
        revert StaleOracle();
}
```

**四个关键设计点**：

1. **只对"扩大偏离的方向"加征惩罚，收敛方向正常** —— 这样套利者被激励去修复偏离，而不是被一起惩罚。**这是与简单"暂停交易"最大的区别。**
2. **`oraclePaused()` 只能当参考，不能当依据** —— Robinhood 官方明确警告 *"The flag is advisory and not enforced on-chain, so a paused oracle may still return a value — keep your staleness check as the primary guard."*
3. **休市时的参考价是"最后收盘价"，不是"实时喂价"** —— 因为 Chainlink 股票 feed 在休市时 hold last price **且无 heartbeat**
4. **市场日历必须上链**（`MarketCalendar` 合约），且需处理半日市、假期、临时停牌

**并且与 Fluxion 联动（这是 Mantle 独有的部分）**：
```
若偏离超过 softBand 且 Fluxion RFQ 可用：
  → hook 自动触发一笔 RFQ mint/redeem 来收敛价格
  → 这是"链上的、自动的 AP 增发"，替代 Robinhood 的手动流程
```

> ### ⭐⭐ 这是全报告识别出的最大的、无人占据的机制空白
> **Robinhood 有 $133M 的股票代币和全链第 2 的 DEX 量，但没有任何链上熔断。**
> **Mantle 有 Fluxion RFQ 但没有 launchpad。**
> **把两者接起来，就是一个别人短期内抄不了的护城河。**

### 6.4.2 ★ Corporate-Action Rebalance Hook（公司行动的池层适配）

**要解决的问题**（第三部分 3.8.2 第 2 点）：
> **股票代币层有 `uiMultiplier`（ERC-8056），但池层没有任何再平衡逻辑。拆股会让永久锁仓 LP 与真实经济脱节。**

**这是 Robinhood 也没解决的开放风险**（其风险表中标注"⚠️ 未发生，未解决"）。

**Tape 的设计**：
```
hook 监听 quote 资产的 UIMultiplierUpdated(old, new, effectiveAt) 事件
  ↓
在 effectiveAt 时刻：
  ├─ 暂停该池交易（几个区块）
  ├─ 按 new/old 的比例调整池的 tick 参照系
  ├─ 对永久锁仓的 full-range 头寸做等价重铸
  └─ 恢复交易，发出 PoolRebased 事件供索引器同步
```

**⚠️ 前置条件**：需确认 Backed 的 xStocks "Multiplier" 机制是否会发出可监听的链上事件。若不发事件，需引入受信任的 relayer 触发。

### 6.4.3 Progressive Bid Wall（薄流动性下的价格支撑）

**抄 Flaunch，理由是 Mantle 的流动性极薄**（全链日 DEX 量 $0.94M）。

**核心机制**（合约 docstring）：
> *"a single sided liquidity position (**Plunge Protection**) placed **1 tick below spot price**, using the accumulated fees. After each deposit the position is **rebalanced to remain 1 tick below spot**."*

**"棘轮上移"的实现**：每次 `_reposition()` 先**完全移除**旧仓位（收回其中的 quote **和/或** meme 币），再用 `withdrawn + newFees` 在**当时的** currentTick 下方重建 → **买墙价位单调不降**。

**为什么比"用手续费市价回购"更优**：
> *"PBWs support price **without risk of loss to MEV or bots via market buys**."*
> **市价回购是可预测的大买单，必然被夹；限价挂单是被动成交，MEV 无从下手。**

> ### ⭐ 这一条对 Mantle 尤其重要
> **在流动性薄的链上，被动挂单比主动市价买入的滑点损耗小一个数量级。**
> **并且 Mantle 没有 searcher / 做市 bot（私有 mempool 的代价），PBW 是协议自带做市能力的最直接实现。**

**参数建议**：触发阈值抄 Flaunch 的"每累积 0.1 ETH 等值手续费重建一次买墙"，按 Mantle 的规模下调到**每 $50–100 等值**。

### 6.4.4 代币化 IPO / Pre-IPO 的特殊处理（机会与红线）

**机会来自一次失败**：SPCXx（SpaceX）上线时，**大量散户 pre-IPO 认购的实际获配远低于预期（有人只拿到约 $600 等值），部分平台因未能拿到足够底层股份而全额退款。**

这暴露了代币化 IPO 的结构性矛盾：**一级供给是配额制的、有限的；需求是无限的。**
**而这恰恰是 bonding curve 最擅长的问题形态。**

**但合规红线极高。三种方案，只推荐第三种：**

| 方案 | 描述 | 判断 |
|---|---|---|
| (a) 影子代币可优先认购真实 SPCXx | 发 sSPCX，持有者可用托管的 USDC 优先认购 | ❌ **几乎确定构成未注册证券要约，不可行** |
| (b) sSPCX 与 SPCXx 组 v4 pool | 托管 USDC 转为 LP，形成强绑定 | ⚠️ **仍有被认定为衍生品的风险** |
| (c) **纯主题社区代币** | **sSPCX 从头到尾就是一个以某公司为主题的社区代币，仅在文化上关联，无任何财务权利** | ✅ **推荐** |

**方案 (c) 的执行要求**：
- **不能**把它描述成"SpaceX 股票的代表"或"优先认购权"
- 前端必须有极强的免责声明与"这不是股票"的视觉标识
- **建议 quote 用 USDC 而非 SPCXx**，进一步切断"这是股票衍生品"的联想
- **必须经法务审查后再实施**

> ### ⭐ 为什么这仍然是护城河
> - **Robinhood 不会做**：它自己就是发行方，做影子市场等于自我竞争且监管自杀
> - **pump.fun / four.meme 做不了**：four.meme 有 SPCXb，但没有 Mantle 的一级市场关系
> - **只有"有 RWA 资产但不是发行方"的 Mantle 处在这个独特位置**

---

## 6.5 模块二：Sentiment Engine —— 给 meme 一个外生的日历

### 6.5.1 洞察

**meme 最大的问题是没有 catalyst 节奏 —— 全靠随机的社交传播。**
Base 已经用 $450K 和 13 个月证明了：**分发能买到冷启动，买不到留存。**

**但股票有天然的日历**：财报、FOMC、CPI、产品发布、指数调整、除权除息、IPO 锁定期到期。

> **把这个日历变成 launchpad 的节奏引擎 —— Mantle 不需要制造病毒传播，只需要寄生在美股本来就有的注意力周期上。**

### 6.5.2 三种玩法（按合规风险从低到高排序）

**A. Ticker Wars（代码之战）—— 零合规风险，先做这个**
- 两个标的的支持者各自发币，比拼谁的 meme 市值先到 $X
- **纯社区行为，无结算，无对赌，无财务权利**
- 平台提供的只是"排行榜 + 时间窗口"这一层协调机制
- 天然适配财报季：`$NVDA-BULLS` vs `$AMD-BULLS`

**B. Index Coins（篮子 quote）—— 低风险，抄 PAIR 的 multipool**
- 用 **1–5 个 xStocks 组成的篮子**作 quote，创建者自选权重（必须加总 100%）
- 例："AI 篮子" = NVDAx 40% + MSFTx 30% + GOOGLx 30%
- **让 quote 侧本身就是一个可讲的故事**
- ⚠️ **必须解决 PAIR 的已知缺陷**：*"聚合器不做池间再平衡，篮子内各池的价格对齐完全依赖外部套利资本，而在股票市场休市期间这种纠正可能长时间缺席。"*
  **→ Tape 应在 hook 内实现主动再平衡，用 Fluxion RFQ 作为兜底对手方。**

**C. Earnings Coins（财报币）—— 高风险，需法务审查**
- 财报季为热门标的自动开出 `NVDA-BEAT` 与 `NVDA-MISS`
- ⚠️ **这个结构在法律上非常接近二元期权**
- **更安全的变体**：两个币都不做链上结算，**只在叙事上对赌，赢家凭社区共识获得关注与流量位**。平台不做任何"兑付"

### 6.5.3 为什么这条路优于"创作者经济"

| | 创作者经济（Base 路线） | 事件日历（Tape 路线） |
|---|---|---|
| catalyst 来源 | **内生** —— 需要创作者持续生产内容 | **外生** —— 美股日历自动提供 |
| 可持续性 | 依赖创作者留存（**Base 已证伪**） | **日历永远不会停** |
| 冷启动 | 需要社交图谱 | **需要的是股票关注度，而这已经存在** |
| 与主业关系 | 无关 | **每次都在给 xStocks 引流** |

---

## 6.6 模块三与四：Graduation Rail 与 Tape Terminal

### 6.6.1 毕业阶梯：抄对 four.meme 的那一课

**BSC meme 季的真正引擎不是 bonding curve，而是「four.meme 发币 → Binance Alpha 上架 → Binance 现货」这条可见的阶梯。**

⚠️ **但必须诚实**：第二部分 2.1.8 已证明这**不是一条制度化通道，而是一次运营活动**（仅 4 例，全在 2025 Q1–Q2，由 CZ/何一发帖驱动，此后未复制）。

> **→ 所以不要把"Bybit 上币直通车"当作方案的核心假设。**

**但阶梯本身的产品价值仍然成立** —— 它给投机提供方向。**Mantle 已经有阶梯的每一级，只是没有连起来**：

```
Tape 曲线 → 毕业进 v4 池（永久锁）→ Fluxion 主路由 + 深度 → Bybit Alpha 专区 → Bybit 现货
   ↑                                          ↑                    ↑
 发行层                              Mantle 原生 DEX      （2026-03 Mantle 已接入）
```

**机制建议**：
1. **定义明确、公开、可验证的晋级标准**（写进合约，铸造"成就 NFT"）：
   - 进入 **Fluxion 主路由**：毕业 + 7 天持续交易量 > $X + 持有人 > N + 无安全事件
   - 进入 **Bybit Alpha 候选池**：30 天量 > $Y + 合约已验证 + 通过安全扫描
   - 进入 **Bybit 现货**：走 Bybit 现有上币流程（**不承诺，只提供数据支持**）
2. **MNT 持有者在 Alpha 专区享额外权益**（复用已有的 VIP 倍率机制）

### 6.6.2 ⭐ 与 Bybit Alpha 的正确关系：白标，而非导流

**这是本方案最关键的分发设计。**

**问题**（第四部分 4.5.5 B 条）：
> **Bybit Alpha 是账户制 CeDeFi 入口，用户不用管助记词和 gas 就能交易链上资产 —— 它的设计目标就是让用户不必上链。**
> **Mantle 最大的用户漏斗，恰恰是它链上活跃度的最大抑制器。**

**错误的解法**：想办法把 Bybit Alpha 的用户"导"到链上。—— 这与 Alpha 的产品逻辑对抗，且 Base 已证明导流买不到留存。

**正确的解法：复制 four.meme × Binance Wallet 的白标关系。**

Binance 官方公告的原文措辞是 **"integrates four.meme's launch technology"** ——
> **four.meme 不是"被 Binance 导流"，而是"成为了 Binance 钱包的发行引擎"。**

**对应到 Mantle**：
```
Bybit Alpha（前台品牌，账户制，无助记词、无 gas）
        ↓ 集成
Tape 发行引擎（后端，链上，Mantle）
        ↓
用户在 Bybit Alpha UI 里买卖曲线上的代币，
但每一笔都是 Mantle 链上的真实交易（Bybit 托管签名）
```

**这样同时得到**：
- Bybit 侧：新的、差异化的产品（"用你的 NVDAx 参与新币发行"），无需教育用户上链
- Mantle 侧：**真实的链上交易量、gas 收入、xStocks 需求**
- 用户侧：零摩擦

**并且这解决了双代币摩擦** —— Bybit Alpha 用 USDT/USDC 结算，gas 由 paymaster 代付。

### 6.6.3 ⭐ Tape Terminal：必须自建执行层

**这是不可省略的一块**（第四部分 4.3 的核心诊断）：

> **GMGN / Photon / BullX / Axiom / Trojan / Banana Gun / Maestro 全部不支持 Mantle。**
> **GMGN 的链列表里甚至包含了 Monad、MegaETH、X Layer、Robinhood Chain 这些更新更小的链，唯独没有 Mantle。**
> **这说明不是"太新没来得及适配"，而是需求信号不足以让它们适配。**
> **这是一个死锁：没有量 → bot 不适配 → 没有执行工具 → 更没有量。只能从内部打破。**

**并且价值捕获排序是「接口层 > 协议层 > 链层」**（三条独立证据：GMGN 占 RH Chain 40% DEX 量；Bankr 是 Base 三大协议之和的 3.9 倍；GMGN/Photon 收入与 pump.fun 同量级）。

**Tape Terminal 的最小可行范围**：

| 层 | 功能 | 优先级 |
|---|---|---|
| **发现** | 新币实时流、进度条、按 quote 资产（NVDAx / TSLAx…）分组的"战壕"面板 | P0 |
| **安全** | 是否锁池、creator 持仓、持仓结构、老鼠仓检测 | P0 |
| **执行** | 一键买卖、限价、止盈止损、**quote 资产一键兑换**（用 USDT 买 NVDAx-quoted meme） | P0 |
| **额度** | Spend-Gate 额度的获取与展示（我的 mETH 持仓 → 我能买多少） | P0 |
| **移动端** | Mantle Passport（Para MPC）无助记词 + paymaster 无 gas | P1 |
| **跟单** | Smart money 追踪、KOL 钱包监控 | P1 |
| **API/Bot** | Telegram bot、开放 API（吸引第三方在上面建） | P1 |
| **数据推送** | WebSocket + 预确认流订阅 | P1 |

**同时并行做 BD**：DexScreener（已支持）/ GeckoTerminal（已支持）→ 争取 **GMGN、DEXTools、Birdeye** 的深度收录。
**Birdeye 已为 Bybit Alpha 提供实时数据，这是最容易打通的一条。**

---

## 6.7 基础设施配套（按优先级）

### P0-1：Paymaster 费用抽象（消除双代币摩擦）

- 用户用 **USDC / USDT / xStocks** 支付 gas，paymaster 结算 MNT
- 新用户前 N 笔交易由协议补贴
- **⚠️ 学 Robinhood 的教训**：它的 gas 补贴**到 2026-09-29 悬崖式到期**，导致"当前所有链上活跃度数据都不是稳态"。
  **→ Tape 的补贴必须有明确预算上限与阶梯式退出曲线，写进公开文档。**
- 现成基础：**Particle Network**（bundler + paymaster，明确支持 Mantle）+ **Mantle Passport**（Para MPC）

### P0-2：Sequencer 级合规过滤

**这是把 xStocks 放上无许可 launchpad 的前置条件，否则 Backed / Bybit 的法务不会签字。**

- 参照 Arbitrum **ArbOS "Elara"**（2026-08，"Compliance Filtering, Priority Fee Support"）与 `ArbFilteredTransactionsManager` 预编译
- Robinhood 官方文档明文承认存在 sequencer 级 screening，且"被拦截的交易看起来就像从未发生过，保证索引器与实际状态同步"
- **Mantle 自有 sequencer，实施难度低于任何去中心化链**
- ⚠️ **必须公开披露这个能力的存在与边界** —— "链的使用是无许可的，链的验证与审查是许可的"这两句必须分开说清楚

### P1-1：预确认流（Flashblocks 式）

- 保持 2s 区块共识，增加 **200–250ms 增量预确认流**
- 提供 `eth_subscribe(newPreconfTransactions | pendingLogs)` 与 `pending` block tag
- **不要去缩短出块时间** —— 高风险低回报（影响共识参数、DA 成本、prover 成本、全部下游工具）

### P1-2：Sequencer 合约分桶（v1 埋钩子，v2 启用配额）

**v1 不需要**（填充率 0.173%），但**架构必须预留**：
- v1：在 sequencer 里实现 access-list 预测 + 按目标合约分桶的**统计与日志**（配额设为无限）
- v2（当单合约 gas 占比连续 N 个区块 > 20% 时启用）：
  - 任一单桶每区块 ≤ **25%** 区块 gas
  - **保留 ≥40% 区块空间给"非热点桶"，该区 gas price 独立计算**
  - 桶内 PGA，桶间确定性轮转
  - **规则公开、确定性、无人工干预；保留 L1 forced inclusion 路径**

### P1-3：明确宣传既有优势

- **"Mantle 无公开 mempool，天然抗三明治"** —— 这是真实且稀缺的优势，从未被讲过
- 参照 BSC 的经验：**集中化可以是优势**（Good Will Alliance 只需说服两家 builder 即可消灭 95% 夹子；Mantle 只需一个决定）
- **应把"协议级抗夹"写成对交易者的明确、可验证的承诺**

### 明确不要做的事

| ❌ 不要做 | 理由 |
|---|---|
| **为 meme 单开一条 appchain** | 会切断与 xStocks 流动性的连接，而那正是全部价值所在 |
| **抄 Arbitrum Timeboost** | express lane 拍卖已被证明会中心化到 3 个实体赢 99.7%，不解决垃圾交易，2026-08 已有 AIP 提议停用 |
| **追求"弹性扩容"** | 正确方向是分区保底，不是无限扩张 |
| **上乐观并行 EVM 来解决热点问题** | launchpad 的写冲突率接近 100%，乐观并行的回滚重执行是纯开销 |
| **缩短出块到 1 秒** | 高风险低回报；Spend-Gate 已经绕开了对出块时间的依赖 |
| **用补贴堆 TVL / 交易量** | Mantle 已用 Aave 那一轮证伪（$137M → $704M → $62M） |
| **承诺 Bybit 上币直通车** | four.meme 的 4 个案例是运营活动不是制度，承诺了做不到会反噬 |

---

## 6.8 分阶段路线图与成功指标

| 阶段 | 周期 | 交付 | **成功指标（不是日发币量）** |
|---|---|---|---|
| **Phase 0：可行性核验** | 0–4 周 | ① 与 Backed 确认 xStocks 的 transfer hook / Multiplier 机制 / 法务边界<br>② 与 Bybit 确认 Alpha 白标集成意愿<br>③ 与 Fluxion 确认 RFQ 的可编程接口 | **三方书面确认，任一不通过则方案需重做** |
| **Phase 1：护栏与地基** | 4–12 周 | Paymaster 费用抽象 + sequencer 合规过滤 + sequencer 分桶钩子（不启用配额）+ 预确认流 | 压测：单合约打满时，非热点交易 gas 不上涨；无助记词无 gas 端到端跑通 |
| **Phase 2：MVP** | 12–22 周 | Ticker Pairs（**Tier 1 的 5–7 个标的**）+ v4 hook 一体化曲线 + Spend-Gate + PBW + **Tape Terminal 的发现/安全/执行三层** | **毕业数 ≥ 3/周；毕业后 7 天存活率 ≥ 30%；日曲线成交额 ≥ $200K** |
| **Phase 3：节奏与差异化** | 22–34 周 | **Oracle-Deviation Circuit Breaker** + Ticker Wars + Index Coins + 全部 Tier 2 标的 + Graduation Rail | **财报周成交额 ≥ 平日 3 倍；首批代币进 Bybit Alpha 专区；熔断机制经历一次真实周末跳空并生效** |
| **Phase 4：护城河** | 34–48 周 | **Corporate-Action Rebalance Hook** + qToken 生息包装 + Fee Key 二级市场 + IPO Curve（法务通过后） | **单个财报/IPO 事件带来 ≥ $5M 曲线成交；Fee Key 出现二级交易** |
| **v2（触发式）** | 当单合约 gas 占比连续超 20% | 启用 sequencer 配额隔离 | **xStocks 结算成本在 meme 峰值期不上涨** |

### 6.8.1 KPI 的选择本身就是一个结论

**必须是「毕业数 × 毕业后 7 天存活率」，绝不能是「日发币量」。**

**证据**（第二部分 2.1.9）：
> **four.meme 的日发币量只跌 14%（1.6 万 → 1.37 万），日收入跌 99%（$1.43M → $9,578）。**
> **毕业率在一年内从 1.34% 掉到 0.07%。**

**同样的教训在 Base**：Clanker 每枚币产生的手续费从 **$1,371（2024-12）→ $53（2026-08），−96%**。
> **发币量能靠机器人 / agent 刷回来，单币价值捕获回不来。**

---

## 6.9 风险登记表

| 风险 | 严重性 | 缓解 |
|---|---|---|
| **Backed 不同意 xStocks 被用作第三方 permissionless quote** | **致命** | Phase 0 提前谈判；备选：先用 USDC 做 quote + 预言机挂钩叙事（不碰真实股票代币），或走 Quote Vault 收敛合规关系 |
| **xStocks 的 Multiplier 是 rebase 而非 ERC-8056** | **致命** | Phase 0 核验；若是 rebase，必须先做一层非 rebase 的包装（qToken），否则永久锁仓 LP 会被套利抽干 |
| **Bybit 不愿做白标集成**（Alpha 的产品逻辑是"不必上链"） | **高** | 用"新增差异化产品线、无需教育用户"作为说服角度；备选是自建移动端 + Mantle Passport |
| **上市公司公开反对**（AMC 已开先例） | **高** | 预设标的白名单 + 冻结治理流程；严禁使用公司商标/logo；强制"非官方、非股票"标识；**不要照抄 Robinhood 的硬刚姿态**（法律隔离结构不同） |
| **被认定为未注册证券发行 / Reg ATS** | **高** | ⚠️ 全行业悬空：**没有任何具名监管机构就"AMM 池交易股票代币"表过态**，也无 SEC 执法。IPO Curve 与 Earnings Coins 必须法务审查；优先无结算、无财务权利的纯叙事变体 |
| **流量不足，发币无人交易**（最现实的风险） | **高** | 从 Bybit Alpha 反向导流；财报日历制造节奏；初期由协议做市（PBW + 金库收益补贴）；**降低毕业阈值到 $12K** |
| **周末跳空导致 LP 被套利** | 中高 | 熔断 hook + 休市提高费率 + 暂停毕业判定 + Fluxion RFQ 自动收敛 |
| **预言机陈旧/操纵** | 中高 | **曲线阶段不依赖预言机**；仅毕业判定与熔断用预言机 + staleness 检查作主要防线 |
| **Noxa 式的工程失能**（拿下份额后 16 天崩塌） | 中高 | **抗 bot 洪水的工程能力 > 机制创新**。Phase 1 必须做压测与限流；不要在没有护栏时开放 |
| **补贴退出后活动崩塌** | 中 | 明确预算上限 + 阶梯式退出（**不要学 Robinhood 的 9-29 悬崖**） |
| **国库资源不足** | 中 | 方案设计为**机制驱动**：唯一的持续性支出是做市与预言机成本，且由协议收入的 10% 覆盖 |
| **毕业目的地深度不足**（Merchant Moe+Agni 仅 $29M） | 中 | 毕业进**自建的 v4 池**（自带 25% 供应 + 全部曲线储备），不依赖现有 DEX 深度 |

---

## 6.10 与竞品的最终定位对照

| | pump.fun | four.meme | Pons | PAIR / LONG | **Tape（建议）** |
|---|---|---|---|---|---|
| 链 | Solana | BNB | RH Chain | RH Chain | **Mantle** |
| quote 资产 | SOL | BNB / 稳定币 / **8 个 bStocks** / 老 meme | ETH | **24 只 RH 股票代币** | **155 个 xStocks（可选生息包装）** |
| 曲线→AMM | 迁移到 PumpSwap | 迁移到 PancakeSwap V2 | **v4，曲线用未来池 quote 计价** | **无曲线，直接进 v4** | **v4 hook 内相变，零迁移** |
| 流动性处理 | **LP burn**（链上验证） | LP burn | **永久锁（locker 无出口）** | 永久锁 | **永久锁 + Fee Key NFT** |
| 毕业阈值锚定 | **纯代币数量**（美元值漂移） | **美元门槛**（~$1.2–1.8 万） | ETH 数量（owner 可配） | 信息性标志位 | **美元门槛（治理调参，曲线不碰预言机）** |
| 抗狙击 | 无原生 | X Mode 逐块递减费 + Name Protection | **99% 起，15s 衰减** | 5 区块硬上限 | **Spend-Gate 额度门禁（不依赖出块时间）** |
| 协议收入去向 | PUMP 回购 | 平台 + 2% 毕业抽成 | **80% 回购销毁 PONS（政策，非合约）** | 90% 回购 $PAIR | **25% 反哺 xStocks 流动性 + 20% 回购 MNT（写进合约）** |
| **休市处理** | N/A | N/A | 无 | **无（结构性缺陷）** | **★ 熔断 hook + Fluxion RFQ 自动收敛** |
| **公司行动池层适配** | N/A | N/A | 无 | **无（Robinhood 自己标注的未解风险）** | **★ Rebalance Hook** |
| 薄流动性支撑 | 无 | 无 | 无 | 无（Flaunch 有 PBW 但在 Base） | **★ Progressive Bid Wall** |
| 分发 | Phantom / 终端 | **Binance Wallet 白标** | RH Wallet + gas 补贴 | RH Wallet | **Bybit Alpha 白标 + 自建 Terminal** |
| 执行层 | GMGN 等全覆盖 | 全覆盖 | GMGN 吃 40% DEX 量 | 同左 | **必须自建（Tape Terminal）** |
| 拥堵隔离 | ✅ Solana LFM | ❌ | **❌（82× base fee 事故）** | ❌ | **v2 sequencer 分桶（v1 埋钩子）** |
| 独占玩法 | — | Stock Meme | — | 股票篮子 quote | **财报日历 + Ticker Wars + IPO 主题币** |

---

## 6.11 最后一句

**Mantle 不应该问「我们怎么做一个 meme launchpad」。**

**应该问：「我们已经有 155 个代币化股票、一个开市/休市双模执行引擎（Fluxion RFQ）、一套生息 ETH 基础设施、和一个接了 8000 万用户的 CeDeFi 入口 —— 怎么让它们产生投机性的、病毒式的、高频的交易需求？」**

**答案是：把 launchpad 当作 RWA 的情绪衍生层来建，而不是当作一个独立的赌场来建。**

**并且要记住这三个数字：**
- **Mantle 的日 DEX 量是 Robinhood Chain 的 1/1,630** —— 起点极低，必须做小而深，不能做大而全
- **Mantle 试过两次，都在 2026 年 8 月同一周关停** —— 失败不是因为没试，是因为试错了方向
- **Mantle 可动用的流动稳定币只有 $1.2 亿** —— 这必须是一个机制驱动的方案，不是补贴驱动的方案
