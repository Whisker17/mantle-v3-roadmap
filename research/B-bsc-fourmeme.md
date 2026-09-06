# 专题 B：BSC / four.meme 深度拆解

> 研究日期：2026-09-06。
> 方法说明：本文优先使用**一手来源**——four.meme 官方 GitBook、four.meme 生产环境公开 API（`https://four.meme/meme-api/v1/public/config`）、four.meme 官方集成文档仓库（GitHub `four-meme-community/fourmeme-docs`、`openfour-docs`）、BNB Chain 官方博客与 GitHub BEP 原文——并辅以**作者本人通过 BSC 公共 RPC 节点直接读取链上状态与事件日志**得到的实测数据（下文标注「链上实测」）。二手转述数字一律标注来源；无法证实者标注 **UNVERIFIED**。
> 全文口径提示：`BNB` 与 `USD` 混用处均单独标注；涉及 Binance Alpha 的"交易量"数字须与链上真实量区分（见 §3.6）。

---

## 0. 一页速览（Executive Summary）

| 维度 | 结论 |
|---|---|
| 定位 | four.meme = BSC 上事实标准的 meme 发射台；不只是 pump.fun 的复刻，而是演化成了**"发行操作系统"**：多计价资产内盘 + 税费代币 + 订阅式发行 + 模块化协议层（OpenFour）+ 美股代币（bStocks）交易底池 |
| 内盘核心参数（链上实测 2026-09-06） | 总量固定 1,000,000,000；内盘可售 `maxOffers = 800,000,000`（80%）；BNB 计价毕业阈值 `maxFunds = 18 BNB`；买卖费率 `tradingFeeRate = 100/10000 = 1%`；`minTradingFee = 0`；创建费 0 BNB（仅 gas ≈0.005 BNB） |
| 毕业阈值演进 | 24 BNB（2024 至 2025 Q1，Verichains 攻击报告实证）→ **18 BNB（现值，链上实测）**；同时演化出"每个计价资产各有自己的阈值"（USDT/USD1/UUSD 12,000；NVDAb 60；QQQb 16；SPCXb 90 …） |
| 迁移 | 早期 PancakeSwap **V3** → 2025-03-31 起改为 **PancakeSwap V2**，LP token **销毁**；**未**迁往 Pancake Infinity（v4），Infinity 仅出现在 OpenFour 的 CubePeg 玩法模块中 |
| 迁移时的"抽成" | **链上实测**：`LiquidityAdded` 事件显示入池永远是 **200,000,000 枚代币（恰好 20%）+ 募集额 ×98%**（18 BNB → 17.64 BNB；16 QQQb → 15.68；650 GMEb → 637）。即**毕业时协议额外留存 2% 募集资金**，这是官方文档未明写的一条真实费率 |
| 当前热度（链上实测） | four.meme 新建代币 ≈ **13,700 个/天**；毕业（`LiquidityAdded`）≈ **10 个/天**量级 → **毕业率 ≈ 0.07%** 数量级，远低于 2025 年高峰期的 1.3% |
| 收入（DefiLlama，2026-09-06 快照） | 累计 Fees **$98.05M** / Revenue **$96.63M**；但 24h Fees 仅 **$9,578**，30d **$290K** —— 相对 2025-10-08 峰值单日 **$1.43M** 已回落约 99% |
| 链 infra | BSC 出块时间 3s→1.5s（Lorentz, 2025-04）→0.75s（Maxwell, 2025-06）→**0.45s（Fermi, 2026-01-14 02:30 UTC，链上实测确认）**；当前 gasLimit **55,000,000**、gasPrice **0.05 gwei**、baseFee 恒为 **0** |
| MEV | 无 relay 的许可制 builder 市场（BEP-322）；48Club(Puissant) + BlockRazor 双寡头占 >87% 区块；Good Will Alliance（2025-03）用"善意 builder 白名单"把恶意三明治压降约 95%（官方口径） |
| 流量为何来 BSC | 不是技术优势，而是**交易所背书 → 分发（Binance Wallet 内置 Meme Rush / Alpha）→ 上币预期（Alpha→合约→现货）**这条飞轮；但飞轮在 2025 Q2 后未能规模化复制 |
| 对 Mantle 的直接启示 | four.meme 已经把"meme × 美股代币（bStocks）"跑通成产品（Stock Meme，NVDAb/QQQb/HOODb/SPCXb/GMEb/DJTb/MRNAb/FLNCb 共 8 个美股代币可作内盘底池）。这正是 mStocks 应该对标的形态——详见 §8 |

---

## 1. four.meme 完整机制拆解

### 1.1 主体、时间线与合约

| 项 | 内容 | 来源 |
|---|---|---|
| 平台上线 | 官方 Points 文档写明 **Four.Meme Platform Launch: 2024-07-03**（DWF Labs 研报称 2024-01，属误记或指内测） | gitbook `guide/four.meme-points-summer-season-1` |
| 母体 | 与 BinaryX 关系密切；BinaryX 于 **2025-02-21 更名 "Four"**，$BNX 1:1 换 **$FORM**（2025-03 完成）。注意三者区别：**four.meme（平台）/ Four（原 BinaryX，代币 $FORM）/ $4·$FOUR（无关的社区 meme 币）** | Binance Square 2025-02-21 |
| TokenManager (V1) | `0xEC4549caDcE5DA21Df6E6422d448034B5233bFbC` — 仅供交易 **2024-09-05 之前**创建的代币 | fourmeme-docs `integration-guide.md` |
| TokenManager2 (V2) | `0x5c952063c7fc8610FFDB798152D69F0B9550762b` — **2024-09-05 起**全部新代币的创建与内盘交易入口，支持 BNB 与 BEP20 计价 | 同上 |
| TokenManagerHelper3 | `0xF251F83e40a78868FcfA3FA4599Dad6494E46034`（BSC）；**Arbitrum One** `0x02287dc3CcA964a025DAaB1111135A46C10D3A57`；**Base** `0x1172FABbAc4Fe05f5a5Cebd8EBBC593A76c42399` | 同上 |
| 合约地址后缀规范 | 2025-03-31 起标准代币地址统一以 **`4444`** 结尾；OpenFour Royalty 系税费代币以 **`ffff`** 结尾（官方声明：仅是生态标识，**不代表审计/安全背书**） | Product Update #3；`openfour/royalty-en` |

> ⚠️ four.meme 已经**不只在 BSC**：Helper3 在 **Arbitrum One 与 Base 均有部署**。这说明 four.meme 自己已经把"发行协议"当作可移植的中间件在做多链扩张——这是判断"BSC 特殊性"时必须打的折扣。

### 1.2 创建费与代币参数

- **创建费 = 0**。官方 GitBook：*"Launching your project is free of charge on Four.meme, the only fee you will pay is the transaction fee which is ~0.005 BNB"*；集成文档 `create-guide.md` 亦写 *"Latest documented creation fee reference: 0.00 BNB"*；生产 API 中所有计价资产的 `deployCost` 字段均为 `"0"`。历史上是否曾收过非零创建费：**UNVERIFIED**。
- **总量固定 1,000,000,000**，第三方**不可**自定义（`create-guide.md` §3.2 "Fixed / non-customizable"）。
- **内盘可售 80%（`saleRate = 0.8` → 800,000,000 枚）**，**留给 LP 20%（200,000,000 枚）**。
- `lpTradingFee` **固定 0.0025**（即毕业后 Pancake 池的 0.25% 费率档），第三方不可改。
- 创作者可在**同一笔创建交易内预买**（`preSale` 字段，单位为计价资产数量），官方定位是"防狙击"而非"内部分配"。**创作者预买上限**：UNVERIFIED（API 未见硬上限字段）。
- 可选 `label` 分类：Meme / AI / Defi / Games / Infra / De-Sci / Social / Depin / **Charity** / Others。

### 1.3 Bonding curve：形式与实测参数

**形式**：带虚拟储备的**恒定乘积**（x·y=k with virtual reserves），非线性/指数曲线。证据有三：
1. Helper3 暴露 `calcInitialPrice(uint256 maxRaising, uint256 totalSupply, uint256 offers, uint256 reserves)`；
2. 公开配置里每个计价资产都带 `b0Amount`（虚拟基础储备）与 `totalBAmount`（募集目标）两个字段，与 pump.fun 同构；
3. 官方明确声明 *"bonding-curve math internals and proprietary libraries"* **不在开源范围内**（`integration-guide.md` §8）。

**链上实测（作者 2026-09-06 直接 `eth_call` Helper3.getTokenInfo）**，取三只当天新建代币：

```
maxOffers          = 800,000,000   (= 80% × 1e9)
maxFunds (BNB 计价) = 18 BNB
tradingFeeRate     = 100  → 1%      minTradingFee = 0
初始 lastPrice      = 5.7398e-9 BNB / token
```

由初始价格反推曲线：虚拟报价储备 `x0 = p0 × y0 = 4.5918 BNB`，`k = x0·y0 = 3.6735e9`。据此可以给出**四组关键估值刻度（以 BNB 计，与 BNB 价格无关）**：

| 刻度 | 数值 | 说明 |
|---|---|---|
| 起始价格 | 5.7398e-9 BNB/token | 链上实测 |
| **起始 FDV** | **≈ 5.74 BNB** | 1e9 × 起始价 |
| 毕业时曲线剩余 offers | ≈ 162.6M 枚 | k/(x0+18) |
| 毕业时售出 | ≈ 637.4M 枚（占总量 63.7%） | 800M − 162.6M |
| 毕业价格 | ≈ 1.389e-7 BNB/token | (x0+18)/162.6M |
| **毕业 FDV** | **≈ 139 BNB** | 内盘全程约 **24.2 倍**价格空间 |

> 推论（**UNVERIFIED**）：毕业时售出 637.4M + 入池 200M = 837.4M，剩余约 162.6M 枚曲线未售代币去向官方未说明，最可能是销毁。建议后续用一笔毕业 tx 的 Transfer 事件做终局验证。

**多计价资产（Multi-Token Trading）—— 每个资产一套曲线参数。** 以下为生产 API 实时快照（2026-09-06），`status=PUBLISH` 即当前对外开放：

| 计价资产 | 合约 | b0Amount | totalBAmount（毕业阈值） | 状态 |
|---|---|---|---|---|
| **BNB** | WBNB `0xbb4c…c095c` | 8 | **18** | PUBLISH |
| **USDT** | `0x55d3…7955` | 4,000 | **12,000** | PUBLISH |
| **USD1** | `0x8d0d…8b0d` | 4,000 | **12,000** | PUBLISH |
| **UUSD** | `0x61a1…0000` | 4,000 | **12,000** | PUBLISH |
| **NVDAb**（英伟达） | `0x02fc…7436` | 8 | **60** | PUBLISH |
| **QQQb**（纳指 ETF） | `0x2058…efc7` | 8 | **16** | PUBLISH |
| **HOODb**（Robinhood） | `0xa394…7c65` | 8 | **80** | PUBLISH |
| **SPCXb**（SpaceX） | `0xbe9d…03e1` | 8 | **90** | PUBLISH |
| **GMEb**（GameStop） | `0x46ce…b15c` | 8 | **650** | PUBLISH |
| **DJTb**（Trump Media） | `0xf2ec…fb6b` | 8 | **1,000** | PUBLISH |
| **MRNAb**（Moderna） | `0x5fd8…6503` | 8 | **65** | PUBLISH |
| **FLNCb**（Fluence） | `0x4af1…14ac` | 8 | **1,000** | PUBLISH |
| CAKE / LisUSD / THE / SHELL / FORM / ASTER / USDC / BNX / BabyDoge / Koge / WHY / binancedog / 币安人生 / Broccoli714 | — | 各异 | 各异 | INIT（已配置未开放） |

关键观察：
1. **所有 bStocks 底池的毕业阈值都被校准到 ≈ 1 万美元等值**（60 NVDAb、16 QQQb、80 HOODb…），而 BNB 池 18 BNB、稳定币池 12,000 USD——即 four.meme 在把**毕业线统一锚定到"约 1.2–1.8 万美元"**这个心理门槛上，而不是锚定某个固定 BNB 数量。这是 2025 年"24 BNB"到 2026 年"18 BNB"演进的真实动因（BNB 涨价 → 降 BNB 计数以维持美元门槛恒定）。
2. `binancedog`、`Broccoli714`、`WHY` 等**meme 币本身也被配置成计价资产**——即 four.meme 支持"用一个 meme 当另一个 meme 的底池"，把老 meme 的持币者变成新 meme 的流动性来源。这是 Solana 系 launchpad 没有系统化做的一层。

### 1.4 内盘费率与费用分配

| 项目 | 数值 | 来源 |
|---|---|---|
| 买入费 / 卖出费 | **1% / 1%**（`buyFee=sellFee=0.01`，链上 `tradingFeeRate=100`） | API + 链上实测 |
| 最低交易费 | GitBook 写 0.001 BNB；**链上实测当前 `minTradingFee = 0`** | 二者冲突，以链上为准 |
| 第三方路由分佣 | 集成方可在 `sellToken(origin, token, amount, minFunds, feeRate, feeRecipient)` 中自设 `feeRate`，**上限 5%**（`100`=1%，`10`=0.1%），以计价 ERC20 计收 | `trade-guide.md` |
| 迁移/建池抽成 | **2%**（链上实测：18 BNB 目标 → 入池 17.64 BNB） | 链上实测，官方文档未列 |
| 官方统一邀请返佣 | **UNVERIFIED**（未见一手条款；市面"70/30 分成""0.5% 费率"说法均无一手支撑） | — |

**X Mode 动态费（原 Fair Mode，2025-10-30 升级并更名，UTC 8:00 上线）**：
- 开盘后**逐区块递减的交易费**（第三方报道称 Block 0 最高可至 100%），目的是让狙击机器人的抢跑无利可图；参数官方声明"可能随后续发射调整"。
- **自动回购销毁**：代币从 X Mode 毕业并迁往 PancakeSwap 后，**所有高于 1% 的那部分启动费被立即用于买入该代币并销毁**；未成功毕业的项目，这部分费用转入社区地址，用于后续回购销毁或流动性支持。
- 集成层对应字段：`create-guide.md` 的 `feePlan: true` 即 AntiSniperFeeMode；额外费 `extraFee` 记录在 `_tokenInfoEx1s`，**不计入** `TokenPurchase` 事件的 `fee` 字段（做数据分析时是个坑）。

### 1.5 creator 分成：标准模式没有，OpenFour 里才有

- **标准 Fair Launch 模式下，four.meme 没有 pump.fun 式的"创作者交易费分成"**。创作者的收益来源只有两条：`preSale` 自购的浮盈，以及（若选择税费代币）自设的 tax 收款钱包。
- **OpenFour 引入了真正的 creator 分成**（见 §2.6），其中 `GoPlus Creator Incentives` 是最接近 pump.fun creator fee 的产品：创作者地址在发射时写入链上分配合约，**交易费按市值动态分给创作者：早期 0.02% 起步，随市值上升，上限 1%**，无需手动 claim。

### 1.6 Tax Token（税费代币）

（GitBook《Introducing Tax Tokens on Four.Meme》，含 2026-02-13 的新规，说明文档现役）

- 税率四选一：**1% / 3% / 5% / 10%**；**仅 Free Mode 支持，X Mode 暂不支持**。
- **内盘阶段与毕业后都持续收税**（这点与绝大多数 launchpad 不同——多数只在曲线期收平台费）。
- 税收分配四路，比例之和须 = 100%：`Funds Recipient Wallet`（国库）/ `Divide to Holders`（持币分红）/ `Burn` / `Add to Liquidity`。
- 分红机制：累积到**等值 1,000 美元**后，**每日 UTC+8 00:00 自动分发**一次；可设最低持仓门槛（`minSharing`，形如 d×10ⁿ，n≥5）以防灰尘地址。**2026-02-13 之前发行的代币仍需手动领取**。
- 链上按 `creatorType = (template >> 10) & 0x3F` 区分：`TaxToken`(type 5，基点计) / `TaxToken8`(type 8，买卖税分离，百分比计，上限 10%) / `TaxToken9`(type 9，额外含 `giggleCharityRate`、`binanceCharityRate` 两个**慈善分成**字段)。
- Free Mode 下可单独开启 **Anti-Sniping**：开盘前几个区块施加高额费用。

---

## 2. 差异化创新（vs pump.fun）

### 2.1 一句话结论

four.meme 的差异化**不在 bonding curve 本身**（那部分与 pump.fun 同构），而在四个方向：
1. **计价资产可插拔**（BNB / 稳定币 / 老 meme / **美股代币**）；
2. **发行形态可插拔**（Free Mode 公平发射 / X Mode 反狙击 / Tax Token / **Universal Subscription 订阅式发行**）；
3. **协议层可插拔**（OpenFour：第三方开发者写自己的曲线/交易/分成/迁移模块）；
4. **分发渠道被交易所内置**（Binance Wallet 的 Bonding Curve TGE 与 Meme Rush 直接复用 four.meme 的发射技术）。

### 2.2 预售 / 白名单 / 定时发射 / 最大买入 —— 现状澄清

> ⚠️ 网上大量二手文章（含 2025 年的评测）仍在说"four.meme 是纯公平发射，没有预售和白名单"。**这在 2026 年已经过时。**

**Universal Subscription（通用订阅）** 是 four.meme 的订阅式发行机制，官方文档明确列出的能力：

| 能力 | 细节 |
|---|---|
| 分配方式 | **不是先到先得**，而是订阅期结束后**按认购额占全池比例 pro-rata 分配** |
| 超募 | 支持超募；用户**只支付最终分配到的那部分**，其余按合约结算规则退回 |
| 定时发射 | 内建"订阅期"（时间窗口），本质就是定时发射 |
| **白名单** | "Whitelist Support — Projects can choose to restrict participation to approved wallets" |
| **反狙击** | "Anti-Sniping Protection — 降低 bot 抢跑、同区块狙击" |
| 税费 | 支持 tax 设置、tax 白名单、tax 到期时间 |
| 参数自定义 | 代币总量、售卖数量、流动性分配、募资参数均可配 |
| **创建费** | **0.2 BNB**（远高于普通发射的 0） |
| **平台费** | 仅对**项目方保留（未进 LP）的那部分募集资金收 10%**；若募集资金 100% 进 LP，则不收平台费 |

官方给的示例：计划卖 100 BNB 的代币、收到 1,000 BNB 认购，则认购 10 BNB 者拿约 1% 的售卖份额、实际只花约 1 BNB，其余退回。

**这是一个被严重低估的设计**：它把"IDO 的公平性"与"bonding curve 的即时可交易性"解耦，且用**"钱进 LP 就免平台费、钱进项目方口袋就收 10%"**的费率结构，在经济上直接把项目方推向"多留流动性"。这是可以直接抄给 Mantle 的机制（见 §8）。

**单钱包最大买入限额**：作为 Universal Subscription 的"发行参数"存在（官方列了"灵活配置发行参数"但未公开上限字段名），普通 Free/X Mode 下 **UNVERIFIED**。

### 2.3 反狙击的三层实现

1. **同交易预买**（`preSale`）：创作者在 `createToken` 同一笔 tx 内买入，让狙击者不可能拿到第一笔。
2. **X Mode 动态费**：开盘若干区块内费率极高并逐块衰减，把狙击的期望收益打成负数；超额费用毕业后回购销毁。
3. **Free Mode Anti-Sniping 开关**：开盘前几个区块高额费。
4. **Token Name Protection（2025-10-18 上线）**：代币在曲线期持币人数达 **100+** 时，其名称/ticker 被**锁定 72 小时**，期间禁止创建同名/近似名的 Fair Mode 代币（跨 Fair/Free 双模式查重；Free Mode 代币不受此限）。特意设计成"几乎同时创建的两个可以都成功"，以防机器人批量抢注名字。这是针对"蹭热点同名盘"这一 BSC 特有毒瘤的产品级解法。

### 2.4 加速器（Accelerator Program）

官方页面存在，分三档：
- **Basic Support**：站内流量位、空投活动；
- **Platform Support**：徽章、官方社媒推广、社区支持；
- **Ecosystem Support**：**BNB Chain 官方对接**（推荐位与资源），且明确写着"需 BNB Chain 团队完成尽调后才提供"，LP 支持条件参照 BNB Chain 官方博客《BNB Chain dedicates 900K liquidity pool to support and develop meme coin ecosystem》。
- 官方注明评估标准"灵活"，市值/交易量阈值不是唯一因素（二手来源 smithii.io 称里程碑为 $44K / $1M / $5M / $10M，**UNVERIFIED**）。

配套的 BNB Chain 官方资金面动作（一手 bnbchain.org）：
- 2024-04：**Meme Innovation Campaign**，奖池最高 **$1,000,000**；
- 2025-02：**$4.4M 流动性支持计划 Round 1**，直接从 BNB Chain 基金会钱包注入 BNB 并**永久留存**；合格发射平台名单明确包含 **four.meme、Burve、Gra.Fun、PinkSale、Flap、TokenFi、Beeper、HoloworldAI、Myshell**；
- 2025-10：与 four.meme / PancakeSwap / Trust Wallet 联合的 **$45M "Reload Airdrop"**，面向 16 万+ 此前交易过 meme 的用户做补偿（多信源交叉，一手公告链接待补）。

### 2.5 AI 工具

- **原生 AI 生成 logo/名称/叙事：不存在**（多个教程均要求用户自行用第三方工具做图后上传；`token/upload` 接口也强制图片必须上传到 four.meme 自己的存储，不接受外链）。
- **AI 相关的真正落点是 OpenFour 的 "GoPlus Skill Royalty / Code Royalty"**：给 **AI Skill Coin**（SafuSkill 发行）配一个链上版税金库，分成写死在合约里：**验证创作者 70% / 经验证的真实用户（下载+使用可追踪）15% / 生态 15%**；且只有**通过 GitHub 验证的 Skill 所有者**才能事后绑定钱包领取版税——这个"先发币、后验证身份认领版税"的设计专门用来防止发射瞬间被抢跑冒领。后续升级为更通用的 **Code Royalty**（"代码贡献 → 链上资产 → 持续版税"）。
- **Telegram Bot**：官方 `t.me/Four_memeBot` + MiniApp（买入提醒、新币推送）。X(Twitter) Bot：UNVERIFIED。

### 2.6 OpenFour：把 launchpad 变成协议

（GitHub `four-meme-community/openfour-docs`，README 最近更新 **2026-09-03**，可见仍在高频迭代）

OpenFour 是 four.meme 的**模块化发行引擎**：把 Token / Vault / Curve / Trade / CustomData / Migrate 拆成可替换模块，第三方开发者通过安全审核后可以部署自己的"发行模式"，并**分享该模式产生的交易激励**。合约接口已开源（`IOpenFourCore`、`IOpenFourCurveModule`、`IOpenFourTradeModule`、`IOpenFourMigrateModule`、`IPancakeV2MigrationAdapter`、`ITaxStrategy`、`ITaxVault` 等）。

已上线的四个第三方模式：

| 模式 | 提供方 | 机制要点 |
|---|---|---|
| **GoPlus Creator Incentives** | GoPlus Security | 创作者地址写入链上费用分配合约；创作者费**随市值从 0.02% 递增，上限 1%**，自动到账 |
| **GoPlus Skill Royalty → Royalty** | GoPlus × SafuSkill | 每个 Skill Coin 一个独立 `SkillRoyalty` 链上金库；**费用不经平台国库**，无多签托管；70/15/15 分成 |
| **Likwid Dex** | Likwid | 对新毕业代币的**无预言机杠杆交易**：直接用池内储备定价，现货与杠杆共用一个流动性源；借出资产用"内部镜像储备"表示，不制造虚假价格；**风险定价费**——价格冲击 ≤10% 收基础费，>10% 费用随冲击**三次方**增长，接近 70%+ 时费率封顶 **99%** |
| **CubePeg** | Cubus × **PancakeSwap Infinity** | 用 Infinity Hook 挂在毕业后的 v4 池上，每笔 swap 刷新链上随机种子；持仓每 100,000 枚自动 mint 1 枚 Art 徽章，跌破档位自动 burn，单笔最多 mint 50 枚 |

**Royalty 框架**（Skill Royalty 的通用化）现支持五种模式：`Code Royalty`（代码仓库领版税）、`X Verified Royalty`（认证 X 账号绑定 EVM 地址领版税，绑定后**不可更改**，未领收益 60 天后按规则处理）、`Ecosystem Buyback Royalty`（A 币的交易费自动去买 B 币，再销毁/分红/生态激励）、`Unverified Royalty`（开放版税模板市场：模板作者拿每次税费分配的 **6%**，94% 进代币的 Royalty Vault；作者地址若是零地址/黑洞/金库本身则 100% 归金库）、`Custom Royalty`。Royalty 已支持 **NVDAb** 作为版税结算资产。

> 战略解读：OpenFour 是 four.meme 对 pump.fun 生态位竞争的答案——**当发行本身彻底商品化、费率被打到 1% 以下时，唯一的护城河是成为别人构建的底座**。Mantle 若要做 launchpad，这是必须提前想清楚的终局：是做一个 app，还是做一个 registry + 模块市场。

### 2.7 与 Binance Web3 Wallet / Binance Alpha 的耦合

**(a) Binance Wallet Bonding Curve TGE**（Binance 官方公告）：认购期内代币**不可外部转让**但可在钱包内沿 bonding curve 买卖；达到流动性里程碑后自动迁往 PancakeSwap。

**(b) Meme Rush**（2025-09/10，Binance Wallet Keyless 用户专享）：三阶段 `New`（初始 bonding curve）→ `Finalizing`（接近迁移）→ `Migrated`（达到里程碑如 $1M FDV 后迁 DEX）。官方公告原文措辞为 **"integrates four.meme's launch technology"** —— 即 **four.meme 是白标基础设施供应商，Binance Wallet 是前台品牌**。这是整个专题最重要的一条结构性事实：**four.meme 不是"被 Binance 导流"，而是"成为了 Binance 钱包的发行引擎"**。

**(c) Binance Alpha**：嵌在 Binance Wallet / 交易所 App 内的**预上市发现层**，官方定义的三段式路径为 **Alpha → Futures → Spot**。Alpha 阶段评估用户接受度、基本面、风险；进 Alpha **不保证**未来上现货。

**(d) TGE / Booster 参与机制**（Binance 官方 FAQ）：需备份 Keyless 钱包 + KYC + **Alpha 积分门槛（因活动而异，非固定值，曾见 2 分与 61 分两种极端）**；路径 Wallet → Discover → Booster；TGE 按 **pro-rata** 分配、窗口结束后领取、未使用资金退回、代币可能有锁仓。

**(e) 官方持股关系**：**无证据**表明 Binance / YZi Labs（原 Binance Labs）持有 four.meme 股权（UNVERIFIED）。定位是"紧密生态合作 + 技术供应"，而非自营。

---

## 3. 2025–2026 BSC meme 周期数据

### 3.1 发币量（链上实测优先）

| 指标 | 数值 | 口径/来源 |
|---|---|---|
| **当前新建代币速率** | **≈ 13,700 个/天** | **链上实测**：抓 TokenManager2 的 `TokenCreate` 事件，2026-09-06 采样 800 个区块得 57 次，按 192,000 块/天外推 |
| 2025-10 高峰期日均 | ≈ 16,000 个/天 | Binance Square 转述，**UNVERIFIED** |
| 2025-03 累计 | > 77,000 个 | WuBlockchain/defioasis 引 dune.com/four_meme/fourmeme |
| 2025-10 累计 | ≈ 384,000 个 | Gate.com 引行业数据，**UNVERIFIED 二手** |
| 对照：pump.fun 2024 全年 | ≈ 1,300 万个（日均 36,000+） | dune.com/adam_tehc/pumpfun |

> 注意一个反直觉事实：**2026 年 9 月 four.meme 的日发币量（1.37 万）几乎没比 2025 年 10 月的高峰（1.6 万）掉多少，但收入掉了 99%。** 说明发币这个动作已经彻底沦为零成本噪音（创建费 0 + gas 0.005 BNB），真正枯竭的是**曲线上的买盘**。这是所有 launchpad 设计者必须记住的教训：**发币数是虚荣指标，毕业数和曲线成交额才是真指标。**

### 3.2 毕业率（链上实测）

- **链上实测**：过滤 `LiquidityAdded` 事件（topic `0xc18aa711…c44b0`），在最近 ~24h 的区间内（因公共 RPC 限流，实际覆盖约 15/24 个分片）捕获 **6 次毕业**，外推 **≈ 10 次/天**量级。
- 对比日发币 13,700 → **当前毕业率约 0.07% 数量级**。
- 对照 2025-10 的二手数据：累计毕业 ≈ 5,150 个、毕业率 **≈ 1.34%**（Gate.com 引 Dune，UNVERIFIED）。
- **即毕业率在一年内下降了一个数量级以上。** 与行业整体趋势一致（有二手/学术来源称全行业毕业率到 2026 年中降至 ~0.26%，UNVERIFIED）。

**毕业时 LP 的真实构成（链上实测，6 笔样本全部一致）**：

| 毕业代币 | 计价资产 | 入池代币数 | 入池计价资产 | 阈值 | 实际入池比例 |
|---|---|---|---|---|---|
| `0xcfc2…4444` | BNB | 200,000,000 | **17.64 BNB** | 18 | **98%** |
| `0x94a4…ffff` / `0xfec0…ffff` / `0xc828…ffff` / `0x8a12…ffff` | QQQb | 200,000,000 | **15.68 QQQb** | 16 | **98%** |
| `0x4fad…ffff` | GMEb | 200,000,000 | **637 GMEb** | 650 | **98%** |

→ **结论：入池代币恒为总量的 20%（200M 枚），入池计价资产恒为毕业阈值的 98%，协议在毕业环节额外留存 2%。** 这条 2% 的"seeding fee"在官方 GitBook 中没有写。同时也验证了 §1.3 的一个推论：入池只用 200M 枚，而曲线卖出约 637M 枚，剩余约 163M 枚未售代币不进池。

另注：6 笔毕业里 **5 笔是 bStocks 计价（QQQb/GMEb）**，只有 1 笔是 BNB 计价。样本极小不能外推，但足以说明 **Stock Meme 不是 PPT 功能，它已经承担了 four.meme 当前相当比例的实际毕业量**。

### 3.3 协议收入（DefiLlama 口径，USD）

| 指标 | 数值 | 快照 |
|---|---|---|
| 累计 Fees | **$98.05M** | 2026-09-06 |
| 累计 Revenue | **$96.63M** | 2026-09-06 |
| 30d Fees / Revenue | $290,434 / $286,548 | 滚动 30 天 |
| 7d Fees / Revenue | $53,108 / $52,041 | 滚动 7 天 |
| **24h Fees / Revenue** | **$9,578 / $9,340** | 2026-09-06 |
| **峰值单日 Revenue** | **$1.43M（2025-10-08）**，当天短暂超过 pump.fun 的 $1.14M | CryptoTimes 2025-10-08，据 DefiLlama 排行榜 |
| 2026-08 回落水平 | ≈ $17,000/天 | The Defiant 转述 |

对照组（同一 2026-09-06 快照）：

| 平台 | 24h Fees | 30d Fees | 累计 Fees |
|---|---|---|---|
| pump.fun（Solana 为主） | $3.55M | $147.9M | **$2.07B** |
| four.meme（BSC） | $9,578 | $290K | $98.05M |
| Flap.sh（BSC） | **$2.88M** | — | — |

> **必须纠正一个流行叙事**："2025-10-08 four.meme 单日收入超越 pump.fun"是**真实但一次性**的事件。累计口径上 pump.fun 是 four.meme 的 **21 倍**；30 天口径上是 **510 倍**。BSC 从未在**趋势层面**赢过 Solana 的发行赛道。
>
> **另一个必须纠正的**：2026 年 BSC launchpad 赛道的收入王座已经不是 four.meme 而是 **Flap**（24h Fees $2.88M，约占当日 BSC 全链 app fees $5.35M 的 54%，而 four.meme 只占 0.18%）。见 §7。

### 3.4 BSC 链上活动

| 指标 | 数值 | 时点/来源 |
|---|---|---|
| DEX 日交易量峰值 | ≈ **$6B**（2025-10-08 前后） | CryptoTimes 配图（DefiLlama） |
| 其他高点 | ≈ $2.13B（2025-03）；≈ $4.26B（2025-09-22，报道称超 Solana） | 二手，**UNVERIFIED 精确值** |
| **当前 DEX 日量** | **$1.555B**（7d $8.254B，周环比 +21%） | DefiLlama 2026-09-06 |
| 当前日活地址 / 日交易笔数 | 1.96M / **19.13M** | DefiLlama 2026-09-06 |
| 当前链 Fees | $882,890/24h | DefiLlama 2026-09-06 |
| PancakeSwap | TVL $2.266B（占 BSC DeFi TVL $5.792B 的 ~39%）；24h fees $1.07M | DefiLlama 2026-09-06 |
| **链上实测 TPS** | **≈ 285 TPS**（平均 128.5 tx/block ÷ 0.45s） | 作者链上实测 2026-09-06 |

→ 相对 2025-10 的 $6B 峰值，2026-09 的 $1.55B **回落约 74%**。

### 3.5 代表性代币路径

| 代币 | 发行 | 平台 | 峰值市值 | 进 Alpha | 上现货 | 现状(2026-09) |
|---|---|---|---|---|---|---|
| **$TST** | 2025-02-05，BNB Chain 官方**发币教学视频**里的占位代币被扒出合约地址 | four.meme | ≈ **$5亿**（一说 $5,000万，口径分歧） | 病毒传播后进入 | **2025-02-09 Binance 现货**（公告 8aa1b661…） | ≈ $17–18M |
| **$Broccoli**（CZ 的狗） | 2025-02-13 CZ 发帖透露狗名后，多链涌现同名币；**714 与 F3B 两个版本**互为竞品 | 多平台 | 数亿美元 | 2025-03 社区投票 | 2025-04-22 现货（714 版） | UNVERIFIED |
| **$Mubarak** | 2025-03-14 | four.meme | > **$2.7亿** | 2025-03 社区投票 | 2025-03-17/18 合约；2025-04-22 现货 | UNVERIFIED |
| **$BANANAS31** | **2024-11-16** | four.meme | ATH $0.058–0.074（2025-07） | 2025-03 社区投票 | 2025-04-22 现货 | ≈ **$83–85M** |
| **$Palu** | 2025-03-13/14，源自何一发的组织架构图"神秘第八人"梗 | four.meme | UNVERIFIED | 未见 | 未上 | UNVERIFIED |
| **$BUBB / $Koma** | UNVERIFIED | four.meme(?) | UNVERIFIED | 未见 | 未上 | UNVERIFIED |

**"four.meme → Alpha → 现货"通道的真实成色**：
- 可确认的完整闭环案例 **仅 4 例**：TST、MUBARAK、BROCCOLI(714)、BANANAS31。
- 平均耗时：TST 从发行到上现货 **< 1 个月**；其余三个从 2025-03 社区投票公告到 2025-04-22 上现货约 **6 周**。综合 **1.5–2 个月**。
- **关键限定**：这 4 例全部集中在 **2025 Q1–Q2** 这一个特定窗口，由"CZ/何一发帖 + Binance 社区投票上币"这两个非常规事件驱动。**2025 下半年至 2026 年未见规模化复制。** BUBB、Palu、Koma 均未进入 Alpha 或现货。
- 因此正确的表述是：**这不是一条"通道"，而是一次"运营活动"。** 任何把 four.meme 的成功归因于"存在制度化上币直通车"的分析都是错的——**没有任何证据表明 four.meme 与 Binance 之间存在协议化的自动上币机制**。

### 3.6 数据卫生：Binance Alpha 刷量的干扰

- Binance Alpha 积分体系被证实存在大规模 wash trading：用户为刷积分（换空投/TGE 资格）反复对倒。
- Binance 于 **2025-06-17** 修改规则，**停止把 Alpha 代币之间的互相交易计入积分**，直接针对刷量；并多次公开封禁使用脚本刷分的账户。
- **口径隔离说明**：本文 §3.3 的 DefiLlama Fees/Revenue 与 §3.2 的链上事件计数，都基于 **BSC 链上 bonding curve / DEX 的真实事件**，与 Binance 中心化交易所内部的 Alpha 积分刷量是两套独立数据，不存在混淆。但链上量本身仍可能含普通 wash trading，**four.meme 链上刷量占比无公开研究，UNVERIFIED**。
- 引用任何"Alpha 交易量/热度排名"数字时必须打折。

### 3.7 存活率

- 缺乏权威量化研究。可用的最强 proxy 就是**毕业率本身**：2025-10 约 1.34% → 2026-09 约 0.07% 数量级（链上实测），即 **99.9% 的 four.meme 代币在曲线阶段就死亡**。
- 二手且方法论不透明的数字（**均 UNVERIFIED**）：2025 年发行的 memecoin 超 62% 在 30 天内被标记疑似 rug（coinlaw.io）；BNB Chain 自 2022 年以来约 2.5% 的代币有明确 rug 行为；97% 的 meme 交易者亏损。

### 3.8 安全事件

| 时间 | 事件 | 损失 | 根因 |
|---|---|---|---|
| 2025-02-11 | PancakeSwap **V3** 迁移价格校验漏洞 | ≈ **$183,000** | 迁移时若目标 pair 已存在，合约**不校验其 `sqrtPriceX96`** 直接复用。攻击者预建极端价格假池（约为正确值的 3.68e14 倍），诱导 four.meme 把 200,000,000 枚代币 + 约 **24 WBNB** 打进被污染的池，再用极少代币提走全部 WBNB。（Verichains 2025-02-21 技术复盘——**这也是"当时阈值 = 24 BNB"的一手实证**） |
| 2025-03-18 | 抢跑毕业 / 绕过转账限制 | ≈ **200 BNB（$12–13万）** | 攻击者在官方上线前低价买入少量代币，转入**尚未创建但地址可预计算**的 PancakeSwap Pair，自建该 pair 加流动性绕过 `MODE_TRANSFER_RESTRICTED`，在非预期价格建池后反复抽干。资金经 FixedFloat 洗出。（SlowMist 预警 / QuillAudits 复盘） |

官方响应：暂停 launch 功能排查，承诺一周内核实并补偿。两起事件共同促成 **2025-03-31 的架构调整**：迁移目标 V3 → **V2**、LP **强制销毁**、地址统一 `4444` 结尾。旧的 V3 池代币不会自动迁移，引发争议（如 muppets 市值从 $20M 跌至 $4M）。

> 设计教训（对 Mantle 直接适用）：**"毕业迁移"是 launchpad 攻击面最集中的一步**。V3/CLMM 迁移需要传入价格参数 → 天然引入价格校验漏洞；V2 恒定乘积迁移不需要外部价格输入 → 攻击面小得多。four.meme 用真金白银买到的结论是：**为了安全，宁可退回 V2。**

---

## 4. BNB Chain 基础设施升级

### 4.1 硬分叉时间线

| 升级 | 主网激活 | 核心 BEP | 关键变化 |
|---|---|---|---|
| Tycho / Haber | 2024-06-20 | BEP-336（EIP-4844） | — |
| Bohr | 2024-09-26 | BEP-341（连续出块）、**BEP-322（Builder API）**、BEP-402/404 | PBS 落地 |
| Pascal | 2025-03-20 | BEP-439(BLS12-381)、BEP-440(EIP-2935)、**BEP-441(EIP-7702)**、BEP-466 | 账户抽象 |
| **Lorentz** | 2025-04-29 | **BEP-520**（Short Block Interval Phase One） | 出块 **3s → 1.5s**；Fast Finality **7.5s → 3.75s**；Epoch 200→500；TurnLength 4→8 |
| **Maxwell** | 2025-06-30 | **BEP-524**（Phase Two）、BEP-563（Enhanced Validator Network）、BEP-564（`bsc/2` 新区块获取消息）、BEP-593（增量快照） | 出块 **1.5s → 0.75s**；Fast Finality **3.75s → 1.875s** |
| **Fermi** | **2026-01-14 02:30 UTC**（客户端 v1.6.4） | **BEP-619**（Phase Three）、BEP-590（Extended Voting Rules）、**BEP-592**（非共识 BAL）、BEP-593、**BEP-610**（EVM Super Instruction） | 出块 **0.75s → 0.45s** |
| Osaka / Mendel | **2026-04-28 02:30 UTC** | 9 个 BEP（对齐 6 个以太坊 EIP：7823/7825/7883/7939/7934/7951）+ BEP-648（Fast Finality）+ BEP-656（`eth_config` RPC）；元文档 BEP-658 | 执行一致性、gas 估算稳定性 |
| **Pasteur** | **2026-08-25 02:30 UTC**（客户端 v1.7.7） | **BEP-675**（Builder 提交**已执行完毕**的区块）、BEP-682（跨链桥签名验证加固）、BEP-695（验证人密钥/质押治理加固） | 消除 builder–validator 双重执行；测试网吞吐 **1237 → 2324 TPS（+88%）**，执行开销 **125ms → 15ms** |

**链上实测验证（作者，2026-09-06）**：
- 用二分法定位出块间隔跃变点：**区块 ≈ 75,140,600，时间 2026-01-14 02:30 UTC 前后，0.75s → 0.45s**，与 Fermi 官方激活时间**完全吻合**（官方博文未给出精确高度，此处为独立链上证据）。
- 抽样 2026 年各月：2026-02 至 2026-09 全程 0.45s，gasLimit 全程 55,000,000；2025-11 至 2026-01-12 为 0.75s、gasLimit 100,000,000。

### 4.2 Gas 与区块空间

| 项 | 数值 | 来源 |
|---|---|---|
| **区块 gasLimit 演进** | 140M(3s) → **100M(1.5s)** → **75M(0.75s)** → **55M(0.45s)** | BEP-619 原文表格；**链上实测确认 100M 与 55M 两档** |
| 每秒 gas 供给 | 100M/0.75s ≈ 133 MGas/s → 55M/0.45s ≈ **122 MGas/s** | 由上表推算 |
| **当前 gasPrice** | **0.05 gwei** | **链上实测** `eth_gasPrice` |
| **baseFee** | **恒为 0** | **链上实测**；BEP-226 明确 EIP-1559 base fee 不随拥堵调整；BEP-658 拒绝 EIP-7918 时原文再述 *"base fee in BSC is always 0"* |
| gasPrice 下调路径 | 3 → 1 → 0.1 → 0.05 gwei | 第三方媒体，**官方公告原文 UNVERIFIED** |
| 当前实际负载 | 平均 **128.5 tx/block**、gasUsed ≈ 34.5M/55M（约 63%）、**≈ 285 TPS** | **链上实测** |

> ⚠️ **benchmark vs 真实负载**：官方 H2 2026 路线图称 H1 2026 基准吞吐 **~5,200 TPS（~400 MGas/s）**，网络带宽 133 MGas/s，峰值日处理约 5 万亿 gas。这是**压力测试数字**；真实主网当前约 **285 TPS**。两者不可混用。

### 4.3 并行执行 / "BEP-7928" / "Superblocks" 的事实核查

- **BEP-7928 不存在。** bnb-chain/BEPs 仓库编号目前最高约 706。传闻中的 7928 实为**以太坊 EIP-7928（Block-Level Access Lists, BAL）**。
- BSC 当前落地的是 **BEP-592**（非共识层 BAL），随 **Fermi** 在客户端生效（GitHub 状态字段仍显示 Candidate，属文档滞后）。官方博客《Boosting BNB Smart Chain Performance with Block Access List》(2025-12-11) 的 165 MGas 基准：无 BAL **583 MGas/s** → BEP-592 **653.71 MGas/s（+12%）** → EIP-7928 测试版 **691.48 MGas/s（+18.6%）**。
- **完整的 EIP-7928（把 BAL 哈希写入区块头、变成共识字段）仍在 benchmark 阶段，尚未上主网。**
- **"Superblocks" 一词未见于任何 BNB Chain 官方文档**，判定为误传/社区俗称。官方术语是 *conflict-less parallel execution* / *BAL-based parallel execution*。
- 真正把"并行"思路做到共识层的是 **BEP-675（Pasteur）**：让 builder 提交**已执行完毕**的区块，验证人只做共识规则校验 + 签名，省掉重复执行。这更接近"执行外包"而非"多核并行"。
- **BEP-610（EVM Super Instruction）** 随 Fermi 上线，是指令级优化。

### 4.4 2026 路线图（官方 H2 2026 Tech Roadmap）

- H1 2026 已达成：出块 **450ms**，内存内终局 **650ms**，基准吞吐 ~5,200 TPS。
- H2 2026 三大承诺：**吞吐再翻倍（长期 10x）**、**资源隔离以降低应用间干扰**、**按行业垂直做精细化 gas 费调整**。
- 下一代 L1（研发中）：**100K+ TPS、sub-50ms 预确认、sub-1s 终局**；测试网预计 2026 年末，主网目标 2027 年初；终极愿景 100 万 TPS。

### 4.5 对 meme launchpad 至关重要的能力：BSC 缺什么

| 能力 | BSC 现状 | 影响 |
|---|---|---|
| **本地费用市场（local fee market）** | **没有**。全局单一 gas 拍卖 | 单个爆火 meme 的抢购会把全链 gas 抬高，热点争用无法隔离。BSC 靠"gas 绝对便宜（0.05 gwei）+ 区块极快（0.45s）"来**掩盖**这个问题，而不是解决它 |
| **EIP-1559 动态基础费** | 已启用但 **baseFee 硬编码为 0** | 拥堵信号完全靠 priority fee 表达，等价于纯 first-price 拍卖 |
| **弹性区块空间** | 几乎没有。只有 `GasLimitBoundDivisor` 允许验证人小幅调节，非自动机制 | 瞬时容量上限是硬的（55M gas/块） |
| **公开 mempool** | 公开 P2P + BEP-322 私有 builder 通道并存 | 夹子天然可行，靠 §5 的社会性方案压制 |
| **Preconfirmation** | **未上线**，仅在下一代 L1 远期路线图（sub-50ms 目标） | 用户体验上"确认"仍等于"出块"，靠 0.45s 出块硬扛 |

> **给 Mantle 的推论**：BSC 的 meme 承载力**主要来自"暴力提速 + 极低 gas"，而不是任何精巧的费用市场设计**。这意味着 Mantle 若想在 infra 层做出差异化，**"EVM 上的本地费用市场（LFM）+ 弹性区块空间"是一个 BSC 明确没有、Solana 有、且对 meme 场景极度对症的空位**——而不是去卷 TPS 数字。

---

## 5. BSC 的 MEV 生态

### 5.1 结构：没有 relay 的许可制 builder 市场

**BEP-322《Builder API Specification for BNB Smart Chain》**（Status: Enabled，Created 2023-11-15）是 BSC 版的 MEV-Boost，但有两个决定性差异：

1. **没有 relay 角色。** 角色只有 Builder、Proposer/Validator、以及可选的 **Mev-Sentry**（跑在验证人前面的代理，负责 bid 通信、隐藏验证人真实 IP、抗 DDoS、代付 builder 费用）。官方理由：BSC 验证人数量少（提案时约 40 个、2,000 万 BNB 质押），本身就是高声誉实体，不需要以太坊那种可信中介。
2. **builder 是白名单/许可制的**，不是以太坊那种 permissionless。

交互模式：One-Round（builder 连交易一起提交 bid，验证人可直接签）与 Two-Round（先只提交 bid，选中后再要交易，更安全更慢）。关键 API：`mev_retrieveTransactions`、`mev_reportIssue`。

### 5.2 市场集中度：双寡头

- **48Club（品牌 Puissant）+ BlockRazor** 两家合计产出 **> 87% 的区块、> 90% 的 MEV 利润**（arXiv:2602.15395《MEV in Binance Builder》，Qin Wang 等，观察期 2025-04-01 至 2026-02-28）。BlockSec 的口径为 80–96% 区块 / 92% MEV 利润，量级一致。
- 次级 builder：**bloXroute、NodeReal、Blocksmith** 合计约 19%（二手，**UNVERIFIED 精确拆分**）。其他出现在 Good Will Alliance 材料里的名字：txboost/BlockRoute、Jetbldr（JetBuilder）、NFA。
- 论文归因的三个集中化成因：**(1) 白名单准入壁垒；(2) 区块间隔被压到 0.75s→0.45s 使"可竞争窗口"极窄，放大头部的延迟优势；(3) 部分 validator 与 builder 存在纵向整合**，加剧中心化与审查脆弱性。

> **这是 BSC 高速化的隐性代价**：把出块压到 450ms 的同时，也把区块构建市场压成了双寡头。**对 Mantle 的启示：如果照抄"极短出块间隔"，要预判 builder 集中化，并提前设计准入与反审查机制。**

### 5.3 Good Will Alliance（善意联盟）

（一手：BNB Chain 官方博客，2025-03-18）

- **性质**：社区 + builder 联合发起的**链下协调机制**，不是共识层强制。起于 BNB Chain 论坛"Call for proposal: Addressing malicious MEV attacks"与配套 Snapshot 提案（二手称 79% 赞成通过，**UNVERIFIED**）。
- **首批参与方**：**BlockRazor** 与 **48 Club**，两家率先在各自区块构建流程中部署"三明治攻击过滤器"。后续 JetBuilder 等加入。
- **规则三条**：
  1. builder 在构建流程中部署三明治过滤器，主动拒绝打包夹子 bundle；
  2. 联盟在 GitHub `bnb-chain/good-will-alliance` 的 `mev-info/bsc-mainnet/builders` 目录维护"善意 builder"名单；
  3. **号召所有验证人只接受名单内 builder 的 bid** —— 这是核心惩罚机制，本质是**社会性/声誉性的白名单排他**，**没有链上强制惩罚代码**。
- **效果**：官方博客《How the Goodwill Alliance Slashed Sandwich Attacks by 95%》称截至 2025-07 恶意三明治攻击下降约 **95%**（官方口径；统计口径为笔数还是金额未逐字核实，**建议二次确认**）。

> **这是一个极其重要的治理范式**：BSC 没有在协议层做任何反 MEV（没有加密 mempool、没有强制 inclusion list），而是靠"**builder 市场高度集中 → 只需说服两家 → 白名单排他**"这条捷径解决了夹子问题。**集中化在这里反而成了治理杠杆。** 这对 Mantle（同样是少量排序者/中心化 sequencer 的架构）是可直接复用的思路。

### 5.4 用户侧抗夹方案

| 方案 | 提供方 | 原理 | 入口 |
|---|---|---|---|
| **PancakeSwap MEV Guard** | PancakeSwap（底层 48Club 支撑） | 交易走私有 relay / 受信 builder，绕开公开 mempool | `https://bscrpc.pancakeswap.finance` |
| **48 Club Privacy RPC**（原 Puissant） | 48 Club | 私有通道直连验证人，交易对外不可见 | `https://rpc-bsc.48.club` |
| **bloXroute Front-Running Protection** | bloXroute | `bsc_private_tx` Cloud API，交易直发 builder | 需 API 集成 |
| **BackRunMe** | bloXroute | 私有交易可选允许 searcher 做 backrun 套利并**分润给用户** | 同上 |
| 交易所钱包默认 RPC | Binance / Trust Wallet | 官方称集成 swap 时走私有通道 | 钱包内置（是否**默认**开启因版本而异，**部分 UNVERIFIED**） |
| four.meme 自带 | four.meme | 2024-09-19 Product Update #1 即上线 MEV Protection（当时仅 MetaMask 用户） | 站内 |

**bloXroute 在 BSC 的角色**：BDN（全球低延迟传播网络）+ 私有交易 + MEV relay + 次级 builder。定价已从固定订阅制转为**按链/按服务/按并发的模块化计费**，有免费档，**具体费率未公开（UNVERIFIED）**。

**协议层反 MEV**：**BSC 没有**。组合拳是 `BEP-322（builder 市场标准化）+ Good Will Alliance（社会协调）+ 私有 RPC（用户自选规避）`。BEP-675 提升的是吞吐而非反 MEV。

### 5.5 交易机器人生态

- 主流：**GMGN、Maestro、Banana Gun、Sigma、BullX、DBot、Pepeboost**，普遍费率 **1%**（Banana Gun 手动 0.5% / 狙击 1%），推荐码可减 10–30%，另加优先费。多数支持 BSC 与 four.meme 代币。
- four.meme 官方也有 Telegram Bot + MiniApp。
- **反制现实**：X Mode 的动态费能抬高狙击成本，但**钱包创建成本近乎为零，Sybil 抵抗有限**，第三方狙击 bot 依然活跃。

---

## 6. 为什么 meme 流量流向了 BSC：飞轮拆解与证据

### 6.1 飞轮的四个齿轮

```
① 交易所背书（CZ/何一发帖 → 叙事凭空产生）
        ↓
② 零摩擦发行（four.meme：0 创建费、0.005 BNB gas、0.45s 出块、0.05 gwei）
        ↓
③ 交易所内置分发（Binance Wallet 的 Bonding Curve TGE / Meme Rush 直接内嵌 four.meme 的发射引擎）
        ↓
④ 上币预期（Alpha → 合约 → 现货，且有 4 个真实成功案例做样本）
        ↓
   回到 ①：赚钱效应 → 更多人盯着 CZ 的每一条推文
```

### 6.2 逐个齿轮的证据

**① 交易所背书 —— 事件驱动的叙事供给**

| 时间 | 事件 | 结果 |
|---|---|---|
| 2025-02-05 | BNB Chain 团队发**发币教学视频**，占位代币 $TST 合约地址意外暴露 | 数小时内市值从 0 冲到数千万美元；CZ 澄清"非官方"反而助推 |
| 2025-02-13 | CZ 发帖提到爱犬名叫 Broccoli | 多链涌现同名币，一度数亿美元市值 |
| 2025-03-13/14 | 何一发组织架构图，社区发现"神秘第八人"，官方号戏称 Palu | $PALU 诞生 |
| 2025-03-14 | $Mubarak 发射 | 峰值 > $2.7 亿 |

**这是 Solana 结构性不具备的东西**：Solana 没有一个"其言论能瞬间创造数亿美元叙事、且与该链利益完全绑定"的人格化 IP。这是 BSC 唯一真正不可复制的资产。

**② 零摩擦 —— 但这不是差异化**
0 创建费、~$0.01 的一笔 swap 成本、0.45s 出块。Solana 同样便宜快速。**成本从来不是 BSC 赢的原因。**

**③ 内置分发 —— 这才是最硬的护城河**
Binance Wallet 的 Meme Rush 官方措辞 *"integrates four.meme's launch technology"*：**four.meme 的曲线被塞进了全球最大交易所的钱包首页**。pump.fun 没有任何等价物——它必须自己获客，而 four.meme 的一部分获客是 Binance 代做的。这是**分发权的降维打击**。

**④ 上币预期 —— 真实但被夸大**
4 个成功案例（TST / Mubarak / Broccoli714 / BANANAS31），平均 1.5–2 个月。但如 §3.5 所述，全部集中在 2025 Q1–Q2，靠非常规运营事件驱动，此后未复制。**这个齿轮在 2025 下半年就已经卡住了。**

### 6.3 飞轮为什么在 2026 年失速：反证

这是本专题最有价值的部分——**BSC 的 meme 周期已经证伪了"交易所流量能长期替代产品力"这个假设**：

| 指标 | 2025-10 高峰 | 2026-09 | 变化 |
|---|---|---|---|
| four.meme 日收入 | $1.43M | **$9,578** | **−99.3%** |
| four.meme 日发币 | ~16,000 | ~13,700 | −14% |
| **毕业率** | ~1.34% | **~0.07% 量级** | **−95%** |
| BSC DEX 日量 | ~$6B | $1.555B | −74% |
| BSC launchpad 收入第一 | four.meme | **Flap（$2.88M/24h）** | 易主 |

**结论：日发币量几乎没跌，收入跌了 99%。** 说明交易所导流带来的是**创建行为**而不是**持续买盘**。当 ④ 上币预期这个齿轮停转，整个飞轮的动能就只剩 ①（偶发的 CZ 推文），而 ① 的边际效应在 2025-10 CZ 公开澄清"社媒发帖不代表财务背书"之后急剧衰减。

### 6.4 给 Mantle 的直接结论

1. **不要试图复制 ①**。Mantle/Bybit 有交易所，但没有 CZ 那种"人格化叙事发生器"。硬追这一条会失败。
2. **必须复制 ③**。"把发行曲线内嵌进 Bybit App / Mantle 官方钱包首页"是唯一可复制且高价值的齿轮。four.meme 的实证是：**当发行技术成为交易所前台的后端，获客成本趋近于零。**
3. **④ 必须制度化，而不是运营化**。BSC 的失败正在于上币是"活动"而非"规则"。如果 Mantle 能公开一条**可验证、有明确量化门槛的"链上毕业 → Bybit 上币评估"规则**，其可信度会强于 BSC 那 4 个孤例。
4. **② 只是入场券**，不是卖点。

---

## 7. 其他 BSC launchpad 与竞争格局

### 7.1 Flap（flap.sh）—— 2026 年的赢家

| 项 | 内容 |
|---|---|
| 机制 | permissionless bonding curve，恒定乘积式；无需创作者预注资 |
| 毕业阈值 | ≈ **16 BNB**（二手，**UNVERIFIED**，需核 docs.flap.sh） |
| 创建费 | ≈ 0.001 BNB（**UNVERIFIED**） |
| **核心差异化** | 支持 `TOKEN_TAXED_V3`：**买卖税可非对称配置（1/3/5/10%）**，含分红/佣金接收者机制。（注：four.meme 后来也补上了 Tax Token，此差异已被追平） |
| FLAPSHARE | 交易收入分享机制，创作者/持有者可分润 |
| 多链 | 已扩展 X Layer、Monad、Morph |
| **2026-09-06 收入** | **24h Fees $2.88M**，约占当日 BSC 全链 app fees（$5.35M）的 **54%**；DefiLlama TVL $1.65M |
| 平台币 | 未找到官方平台代币发行确认（市面 "FLAP" 多为同名社区 meme 币），**UNVERIFIED/倾向未发** |

> **这是本专题最重要的"意外发现"之一**：所有关于"BSC meme = four.meme"的既有认知，到 2026 年已经不成立。**Flap 在收入口径上以约 300 倍的差距碾压 four.meme**，甚至有报道称 Flap 日收入超过 pump.fun（The Defiant：*"Flap overtakes Pump.fun in daily revenue with $1.18 million"*）。做 Mantle 的竞品分析时，**必须把 Flap 单独拉出来做一次深度拆解**（本次因分工边界未展开，列为后续工作项）。

### 7.2 PancakeSwap SpringBoard

| 项 | 内容 |
|---|---|
| 上线 | **2024-12-04** |
| 形态 | 不是独立品牌，而是**集成进 PancakeSwap DEX 内**的 launchpad 模块 |
| 毕业阈值 | 约 **24 BNB**（曲线 100%） |
| 迁移 | 自动迁到 **PancakeSwap V3**，流动性由 GoPlusLabs 锁定 |
| 费率 | 创建 **0**；曲线交易 **1%**（最低 0.001 BNB）；**毕业时 2% 做市费，50% 返创作者 / 50% 归 PancakeSwap** |
| 差异化 | 背靠 Pancake 现有流量与 TVL（占 BSC DeFi TVL ~39%）；"Pancake Picks" 同时展示 four.meme 与自家毕业代币 —— **共存而非替代** |

> 注意 SpringBoard 的**毕业 2% 做市费一半返给创作者**——这是 four.meme 标准模式**没有**的 creator 分成，也解释了为什么 four.meme 后来必须靠 OpenFour 的 GoPlus Creator Incentives 来补这一课。

### 7.3 GraFun

- 2024-09 上线，DWF Labs 2024-10 研报时曾在**发币数（~13,000）、用户数（~31,000）上超过 four.meme（~6,800 币 / ~19,200 用户）与 Flap（~250 币 / ~5,600 用户）**。
- 毕业阈值 **38.75 BNB**（高门槛路线）；创建费 0.0005 BNB；交易费 1%；曲线费一部分进 DAO Treasury，毕业后持币人通过 DeXe Protocol 获得该 Treasury 的治理权。
- 获 Floki、DWF Labs、DeXe 背书。
- **2026 现状 UNVERIFIED**——从收入榜消失，很可能已边缘化。这本身就是一个教训：**2024 年靠"发币数第一"领先的平台，两年后已不在牌桌上。**

### 7.4 份额小结与数据源

| 平台 | 2024-10（DWF 研报） | 2025-10 | 2026-09（DefiLlama） |
|---|---|---|---|
| four.meme | 6,800 币 / 19,200 用户 | **收入行业第一**，单日 $1.43M 短暂超 pump.fun | 24h Fees **$9,578**（BSC app fees 的 0.18%） |
| GraFun | **13,000 币 / 31,000 用户**（当时第一） | — | 未见 |
| Flap | 250 币 / 5,600 用户（当时最小） | — | **24h Fees $2.88M（BSC app fees 的 54%）** |
| PancakeSwap SpringBoard | 未上线 | 与 four.meme 共存 | Pancake 整体 24h fees $1.07M |

看板：`dune.com/four_meme/fourmeme`、`dune.com/mwuhjyf/memecoin-launch-on-fourmeme`、`defillama.com/protocol/four.meme`、`defillama.com/protocol/flap-sh`、`defillama.com/chain/bsc`。**没有统一整合四家的实时看板**，DWF Labs 研报是唯一系统性横向对比但已过时（正文写于 2024-10）。

### 7.5 其他

- **Bee.fun / Meme.cooking BSC 版 / Binance Wallet 自带发币入口**：机制细节与份额 **UNVERIFIED**。
- 注意：**"Mubarak 生态"不是 launchpad**，MUBARAK 是 2025-03 经 four.meme 发行的具体 meme 币（后被社区 CTO 接管）；网上以此为名的"发射站"需警惕假冒。

---

## 8. 对 Mantle / mStocks 的直接启示（本专题的落点）

> 这一节是把 §1–§7 的事实转成 Mantle 可执行的判断，供后续"Mantle launchpad 设计"专题引用。

### 8.1 最重要的发现：four.meme 已经把 "meme × 代币化美股" 跑通了

**Stock Meme** 不是概念：
- 创建代币时可选 **bStocks 作为内盘交易底池**，已上线 **8 个**（NVDAb / QQQb / HOODb / SPCXb / GMEb / DJTb / MRNAb / FLNCb），全部 `status = PUBLISH`；
- 每个 bStock 底池有**独立的毕业阈值**（NVDAb 60、QQQb 16、HOODb 80、SPCXb 90、GMEb 650、DJTb 1000、MRNAb 65、FLNCb 1000），全部校准到约 1 万美元等值；
- bStocks 方为符合条件的 Stock Trading Pool **提供专属做市流动性支持**（"dedicated liquidity support…from launch"）；
- OpenFour 的 **Royalty 框架已支持用 NVDAb 结算版税**；
- **链上实测的 6 笔毕业里有 5 笔是 bStocks 计价**——这条线已经在承担实际毕业量。

**mStocks 与 bStocks 是同构的**（Bybit↔Binance、Mantle↔BNB Chain）。因此 Mantle **不需要发明新叙事，只需要在 four.meme 已验证的产品形态上做得更好**。

### 8.2 可直接抄的四个机制

1. **多计价资产内盘（Multi-Token Trading）**
   把 mStocks 当作 launchpad 的底池资产，而不是当作一个独立的交易品类。价值在于：**给 mStocks 持有者一个"不卖出股票就能参与投机"的场景**，把静态持仓变成流动性。同时用"每个底池独立阈值、统一锚定 ~1 万美元"的方式做参数管理。

2. **Universal Subscription（订阅式发行 + 按比例分配 + 超募退款）**
   费率结构是精髓：**创建费 0.2 BNB（等价物）+ 仅对"项目方留存"部分收 10%，全额进 LP 则免费**。这在经济上直接把项目方推向"多留流动性"，对 RWA/股票类资产尤其重要（这类资产最怕薄流动性）。且天然自带白名单、定时、反狙击、税费配置——**是"合规敏感资产"最合适的发行形态**，远优于纯 FCFS 曲线。

3. **毕业迁移只用恒定乘积池 + LP 销毁**
   four.meme 用两次被黑（$183K + 200 BNB）买到的教训：**CLMM/V3 迁移必须传价格参数 → 天生有价格校验攻击面；V2 迁移不需要外部价格输入。** Mantle 的 launchpad 毕业迁移**应默认走恒定乘积池并销毁 LP**，把 CLMM 留给毕业后由市场自行开池。

4. **OpenFour 式的模块化协议层**
   当发行费率被卷到 1% 以下时，唯一的护城河是成为别人的底座。Mantle 若做 launchpad，应从第一天就把 **Curve / Trade / Migrate / Vault / Royalty** 拆成可注册模块，并公开 registry 与审核流程。

### 8.3 必须避开的三个坑

1. **不要把"日发币量"当 KPI。** four.meme 的实证：日发币仅跌 14%，收入跌 99%。KPI 应该是 **毕业数 × 毕业后 30 天存活率**。
2. **不要把上币做成"运营活动"。** BSC 的 4 个 Alpha→现货案例全部集中在两个月内、由推文驱动，之后归零。Mantle 应把"链上毕业 → Bybit 评估"写成**公开、可验证、有量化门槛的规则**，其长期可信度远高于零星的运营惊喜。
3. **不要照抄"极短出块间隔"而不管 builder 集中化。** BSC 把出块压到 0.45s 的代价是 builder 市场变成双寡头（>87% 区块）。好消息是：**集中化反而让 Good Will Alliance 这种"只需说服两家 builder 就能消灭 95% 夹子"的治理捷径成为可能**——Mantle 的中心化 sequencer 天然具备同等甚至更强的杠杆，应当把"协议级抗夹"作为对 meme 交易者的明确承诺来卖。

### 8.4 Mantle 在 infra 层的真正机会

BSC 的 meme 承载力来自"暴力提速 + 极低 gas"，而**不是**任何精巧的费用市场设计。BSC 明确**没有**：本地费用市场、动态 baseFee（恒为 0）、弹性区块空间、preconfirmation。

→ **"EVM 上的本地费用市场（LFM）+ 弹性区块空间 + preconfirmation"是 BSC 与大多数 EVM 链的公共空位，且对 meme 场景高度对症**（单个爆火资产的抢购不应污染全链 gas）。这比去卷 TPS 数字更有辨识度。

---

## 9. 未证实 / 待补清单（UNVERIFIED）

**four.meme 机制**
1. 毕业阈值 24 → 18 BNB 的**确切生效日期与官方公告**（Product Update #1–#6 均未提及；推测在 2025-02/03 安全事件后到 2025-03-31 架构调整之间）。
2. 迁移时 2% 抽成的**官方条款出处**（本文为链上实证，文档未写）。
3. 曲线毕业后剩余 ~163M 枚未售代币的**去向**（推测销毁，未验证）。
4. 是否存在官方统一的**邀请返佣比例**。
5. 创作者 `preSale` 预买是否有**硬性上限**。
6. X Mode 动态费的**逐区块具体费率表**（官方图片未 OCR）。
7. four.meme 平台**精确上线日期**（官方 Points 页写 2024-07-03，DWF 研报写 2024-01）。
8. TokenManager V1/V2 在 BscScan 的**精确部署时间戳**（bscscan.com 返回 403）。
9. 是否有过**无限铸造类漏洞**（倾向"无"）。

**数据**
10. four.meme **分年度**收入拆分（需下载 DefiLlama CSV）。
11. four.meme **每日发币/毕业的完整时间序列**（需直接跑 Dune SQL）。
12. 本文毕业率实测受公共 RPC 限流影响，仅覆盖约 15/24 个时间分片，**需用归档节点重跑全天**。
13. Broccoli **714 vs F3B** 两个版本的准确区分与各自数据。
14. BUBB / Palu / Koma 的发行日期与峰值市值。
15. BSC 三明治攻击的 **2025 年逐日美元规模**（未定位到 EigenPhi 的 BSC 专项）。
16. 行业"毕业率 0.26%"与"火山式喷发"定性描述的**原始出处**。

**基础设施**
17. Lorentz / Maxwell / Pascal / Bohr / Tycho 的**精确激活区块高度**（Fermi 的 ≈75,140,600 为本文链上实测，官方博文未给出）。
18. gasPrice 3→1→0.1→0.05 gwei 各次下调的**官方公告原文**。
19. Pasteur 官方博文**全文**（本次仅得 web_search 摘要）。
20. BEP-610 / BEP-592 在 GitHub 上仍显示 Candidate 与"已随 Fermi 生效"的**状态字段滞后**需官方确认。

**竞品**
21. **Flap 的完整机制拆解**（毕业阈值精确值、费率、FLAPSHARE 分成比例、为何在 2026 年反超）——**优先级最高的后续工作项**。
22. GraFun 的 2026 现状。
23. Bee.fun / Meme.cooking BSC 版 / Binance Wallet 自带发币入口的机制与份额。
24. bloXroute / NodeReal / Blocksmith 的精确 builder 份额拆分。

---

## 10. 主要来源

**一手（官方文档 / API / 链上）**
- four.meme GitBook：`https://four-meme.gitbook.io/four.meme/`（`guide/how-it-works`、`guide/introducing-tax-tokens-on-four.meme`、`openfour/about-openfour-en`、`openfour/royalty-en`、`openfour/universal-subscription-en`、`stock-meme/about-stock-meme-en`、`brand/accelerator-program`、`product-update/1`–`/6`）
- four.meme 生产配置 API：`https://four.meme/meme-api/v1/public/config`
- GitHub `four-meme-community/fourmeme-docs`（integration/trade/create/tax guides + lite ABI）
- GitHub `four-meme-community/openfour-docs`（README 更新 2026-09-03；模块接口与 FairLaunch 样例）
- **BSC 公共 RPC 直读**（本文所有"链上实测"：`eth_blockNumber` / `eth_getBlockByNumber` / `eth_getLogs` / `eth_call` on TokenManagerHelper3 `0xF251F83e40a78868FcfA3FA4599Dad6494E46034`）
- BEP-322：`https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP322.md`；BEP-520 / BEP-524 / BEP-619 / BEP-226 / BEP-592 / BEP-658
- BNB Chain 官方博客：Fermi(`/fermi-hard-fork-accelerates-bsc-to-0-45-second-block-times`)、Osaka/Mendel、Pasteur、BEP-675、Good Will Alliance(`/bnb-good-will-alliance`)、`/how-the-goodwill-alliance-slashed-sandwich-attacks-by-95`、Block Access List、$4.4M 流动性支持、Meme Innovation Campaign
- GitHub `bnb-chain/good-will-alliance`
- Binance 官方：Bonding Curve TGE 公告 `29d8942bfcd0472890c00d30d6b2676c`、Meme Rush 公告、Alpha 学院页、Booster/TGE FAQ `5ee2a485f6434f13b1a3581a2d4d7129`、TST 上币公告 `8aa1b6610a534fcb95b46956f2ed4391`
- PancakeSwap：`pancakeswap.finance/mev`、MEV Guard FAQ；48 Club `docs.48.club/privacy-rpc`；bloXroute `docs.bloxroute.com/bsc/submit-transactions/bsc-private-transactions`

**二手 / 研究**
- DefiLlama：`/protocol/four.meme`、`/protocol/pump`、`/protocol/flap-sh`、`/chain/bsc`、`/fees`
- Dune：`dune.com/four_meme/fourmeme`、`dune.com/mwuhjyf/memecoin-launch-on-fourmeme`、`dune.com/adam_tehc/pumpfun`
- arXiv:2602.15395《MEV in Binance Builder》(Qin Wang et al., 2026-02-17)
- Verichains《four.meme hack analysis》(2025-02-21)；QuillAudits(2025-03-18)；SlowMist
- DWF Labs《Comparison of BNB Chain Memecoin Launchpads: GraFun vs Four.Meme vs Flap》(2024-10-18 发布)
- CryptoTimes(2025-10-08)；WuBlockchain/defioasis(2025-03-24)；The Defiant；ChainCatcher(2025-04-01)；BlockSec
