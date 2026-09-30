# 第二部分：第二代 meme Launchpad —— BSC 的 four.meme 与 Base 的创作者经济

> 数据截止 **2026-09-06**。本部分的 four.meme 曲线参数、毕业 LP 构成、BSC 出块时间跃变点，均为**链上直接实测**。
> 详细底稿见 [`research/B-bsc-fourmeme.md`](../research/B-bsc-fourmeme.md)（673 行）与 [`research/C-base.md`](../research/C-base.md)（1,498 行）。

---

## 2.0 什么叫「第二代」

第一代（pump.fun）解决的是**"任何人都能零成本发币并立即交易"**。
第二代解决的是四件第一代没解决的事：

| 维度 | 第一代（pump.fun） | 第二代 |
|---|---|---|
| **计价资产** | 只能是 SOL | **可插拔**：BNB / 稳定币 / 老 meme / **代币化美股** |
| **发行形态** | 只有一种公平发射 | **可插拔**：公平发射 / 反狙击动态费 / 税费代币 / 订阅式 IDO / 固定价格窗口 / 游戏门禁 |
| **协议层** | 封闭 app | **可插拔**：OpenFour（BSC）、Flaunch RevenueManager（Base）、Doppler（Base）—— **变成别人构建的底座** |
| **分发** | KOL + 终端自然流量 | **被交易所/钱包内置**：Binance Wallet 直接复用 four.meme 的发射引擎；Base App 把 Zora 接进社交流 |

**并且第二代在技术上收敛到了同一个底座：Uniswap v4 hooks（Base）与其等价物（PancakeSwap Infinity Hook，BSC）。**

---

## 2.1 BSC / four.meme：把 launchpad 变成交易所的白标基础设施

### 2.1.1 主体与关键参数（链上实测）

| 项 | 值 | 来源 |
|---|---|---|
| **创建费** | **0**（只付约 0.005 BNB 的 gas） | 官方 GitBook + 生产 API `deployCost=0` |
| 曲线形式 | **带虚拟储备的恒定乘积**（与 pump.fun 同构） | Helper3 `calcInitialPrice` + API 字段 |
| **曲线可售（maxOffers）** | **800,000,000（80%）** | 链上 `eth_call` Helper3.getTokenInfo |
| **毕业阈值（BNB 池）** | **18 BNB** | 链上实测 `maxFunds=18` |
| **买入/卖出费** | **1% / 1%**（`tradingFeeRate=100`） | 链上实测 |
| 最低交易费 | GitBook 写 0.001 BNB，**链上实测 `minTradingFee = 0`** | 二者冲突，以链上为准 |
| 第三方路由分佣 | 集成方可自设，**上限 5%** | `trade-guide.md` |

> ⚠️ **曲线数学内部实现不开源**：官方明确声明 *"bonding-curve math internals and proprietary libraries"* 不在开源范围内。

**由链上初始价格反推的完整曲线刻度**（以 BNB 计，与 BNB 价格无关）：

| 刻度 | 数值 |
|---|---|
| 起始价格 | `5.7398e-9` BNB/token（链上实测） |
| **起始 FDV** | **≈ 5.74 BNB** |
| 虚拟报价储备 x₀ | 4.5918 BNB（= p₀ × y₀），k = 3.6735e9 |
| 毕业时售出 | ≈ **637.4M 枚（63.7%）** |
| 毕业价格 | ≈ `1.389e-7` BNB/token |
| **毕业 FDV** | **≈ 139 BNB** → **内盘全程约 24.2 倍价格空间** |

> 对照 pump.fun 的 **14.70 倍**：four.meme 的内盘涨幅空间更大（24.2x），**因为它只卖 63.7% 就毕业（pump.fun 卖 79.31%）**。
> **这是一个刻意的取舍：留更多筹码给毕业后的 AMM，让二级市场有余量。**

### 2.1.2 毕业时的 LP 真实构成 —— 一条官方没写的 2% 抽成

链上过滤 `LiquidityAdded` 事件，6 笔样本**全部一致**：

| 毕业代币 | 计价资产 | 入池代币数 | 入池计价资产 | 阈值 | 比例 |
|---|---|---|---|---|---|
| `0xcfc2…4444` | BNB | 200,000,000 | **17.64 BNB** | 18 | **98%** |
| `0x94a4…ffff` 等 4 笔 | **QQQb** | 200,000,000 | **15.68 QQQb** | 16 | **98%** |
| `0x4fad…ffff` | **GMEb** | 200,000,000 | **637 GMEb** | 650 | **98%** |

> ### ⭐ 三条链上发现（官方 GitBook 均未写）
> ① **入池代币恒为总量的 20%（200M 枚）**，而曲线卖出约 637M 枚 —— **剩余约 163M 枚未售代币不进池**（去向未说明，最可能是销毁，待验证）。
> ② **入池计价资产恒为毕业阈值的 98% —— 协议在毕业环节额外留存 2%**。这条 "seeding fee" 是隐性收入。
> ③ **6 笔毕业里 5 笔是 bStocks 计价（QQQb / GMEb），只有 1 笔是 BNB 计价。**

**第 ③ 条是本报告最重要的单条发现之一，见 2.1.4。**

### 2.1.3 多计价资产：毕业线锚定的是「美元门槛」而非「代币数量」

生产 API 实时快照（2026-09-06），`status=PUBLISH` 即当前开放：

| 计价资产 | b0Amount（虚拟储备） | totalBAmount（毕业阈值） |
|---|---|---|
| **BNB** | 8 | **18** |
| **USDT / USD1 / UUSD** | 4,000 | **12,000** |
| **NVDAb**（英伟达） | 8 | **60** |
| **QQQb**（纳指 ETF） | 8 | **16** |
| **HOODb**（Robinhood） | 8 | **80** |
| **SPCXb**（SpaceX） | 8 | **90** |
| **GMEb**（GameStop） | 8 | **650** |
| **DJTb**（Trump Media） | 8 | **1,000** |
| **MRNAb**（Moderna） | 8 | **65** |
| **FLNCb**（Fluence） | 8 | **1,000** |
| CAKE / LisUSD / THE / SHELL / FORM / ASTER / USDC / BNX / BabyDoge / Koge / **WHY / binancedog / 币安人生 / Broccoli714** | 各异 | 各异（INIT，已配置未开放） |

> ### ⭐ 两条关键观察
>
> **① 所有 bStocks 底池的毕业阈值都被校准到 ≈ 1 万美元等值**（60 NVDAb、16 QQQb、80 HOODb…），BNB 池 18 BNB、稳定币池 12,000 USD。
> **即 four.meme 把毕业线统一锚定到「约 1.2–1.8 万美元」这个心理门槛，而不是某个固定代币数量。**
> 这才是 2025 年 "24 BNB" 到 2026 年 "18 BNB" 演进的真实动因 —— **BNB 涨价 → 降 BNB 计数以维持美元门槛恒定**。
> **这与 pump.fun 的"纯代币数量阈值 + 美元值随汇率漂移"是相反的哲学。**（见第一部分 1.1.8）
>
> **② `binancedog`、`Broccoli714`、`WHY` 等 meme 币本身也被配置成计价资产。**
> 即 four.meme 支持"**用一个老 meme 当另一个新 meme 的底池**"，把老 meme 的持币者变成新 meme 的流动性来源。
> **这是 Solana 系 launchpad 没有系统化做的一层。**

### 2.1.4 ⭐⭐ four.meme 已经把「meme × 代币化美股」跑通成产品

这是本报告最重要的发现之一，且**在中文与英文媒体中几乎无人报道**。

**证据链**：
1. 生产 API 中 **8 个 bStocks 代币已 `PUBLISH`**（NVDAb / QQQb / HOODb / SPCXb / GMEb / DJTb / MRNAb / FLNCb）
2. **链上实测的 6 笔毕业中，5 笔是 bStocks 计价**
3. four.meme 有 **"Stock Meme" 专属产品线 + bStocks 专属做市流动性支持**
4. OpenFour 的 **Royalty 框架已支持 NVDAb 作为版税结算资产**

> ### 对 Mantle 的直接含义
> **「用代币化美股做 meme 的 quote 资产」这件事，在 2026 年已经有两个独立的生产实现：**
> **Robinhood Chain 的 PAIR / LONG（用 RHJ 股票代币），以及 BNB Chain 的 four.meme Stock Meme（用 bStocks）。**
>
> **→ 这不再是一个需要论证可行性的创新，而是一个需要论证差异化的既有赛道。**
> **→ Mantle 的方案必须回答："凭什么用户要在 Mantle 上做，而不是在已有深度的 RH Chain 或 BSC 上做？"**
> 我的答案见第六部分 6.1 与 6.4。

### 2.1.5 发行形态的可插拔性

**X Mode（原 Fair Mode，2025-10-30 升级）**：
- 开盘后**逐区块递减的交易费**（第三方报道称 Block 0 最高可至 100%）
- **自动回购销毁**：代币毕业迁往 PancakeSwap 后，**所有高于 1% 的那部分启动费被立即用于买入该代币并销毁**；未毕业的项目该部分转入社区地址
- 集成字段 `feePlan: true`；额外费 `extraFee` **不计入** `TokenPurchase` 事件的 `fee` 字段（做数据分析时是个坑）

**Tax Token（税费代币）**：
- 税率四选一：**1% / 3% / 5% / 10%**；仅 Free Mode 支持
- **内盘阶段与毕业后都持续收税**（这点与绝大多数 launchpad 不同 —— 多数只在曲线期收平台费）
- 税收四路分配，比例之和须 = 100%：国库 / **持币分红** / 销毁 / 加流动性
- 分红机制：累积到**等值 1,000 美元**后，**每日 UTC+8 00:00 自动分发**；可设最低持仓门槛防灰尘地址
- 链上按 `creatorType` 区分三种模板，其中 type 9 额外含 `giggleCharityRate`、`binanceCharityRate` 两个**慈善分成**字段

**Universal Subscription（订阅式发行）—— 一个被严重低估的机制**：

> ⚠️ 网上大量二手文章仍在说"four.meme 是纯公平发射，没有预售和白名单"。**这在 2026 年已经过时。**

| 能力 | 细节 |
|---|---|
| 分配方式 | **不是先到先得**，而是订阅期结束后按认购额占比 **pro-rata 分配** |
| 超募 | 支持；用户**只支付最终分配到的部分**，其余退回 |
| 定时发射 | 内建"订阅期"时间窗口 |
| **白名单** | 支持限定已批准钱包参与 |
| **反狙击** | 内建 bot 抢跑、同区块狙击防护 |
| 税费 | 支持 tax 设置、白名单、到期时间 |
| **创建费** | **0.2 BNB**（远高于普通发射的 0） |
| **平台费** | **仅对项目方保留（未进 LP）的那部分募集资金收 10%；若 100% 进 LP，则不收平台费** |

> ### ⭐ 这个费率结构是可以直接抄的
> **"钱进 LP 就免平台费、钱进项目方口袋就收 10%"** —— 在经济上直接把项目方推向"多留流动性"。
> 它把 **IDO 的公平性**与 **bonding curve 的即时可交易性**解耦。
> 详见第六部分 6.3.6。

**反狙击的四层实现**：
1. **同交易预买**（`preSale`）：创作者在 `createToken` 同一笔 tx 内买入，狙击者不可能拿第一笔
2. **X Mode 动态费**：开盘若干区块内费率极高并逐块衰减
3. **Free Mode Anti-Sniping 开关**
4. **Token Name Protection（2025-10-18）**：代币在曲线期持币人数达 **100+** 时，其名称/ticker 被**锁定 72 小时**，期间禁止创建同名/近似名代币。特意设计成"几乎同时创建的两个可以都成功"，以防机器人批量抢注名字
   → **这是针对"蹭热点同名盘"这一 BSC 特有毒瘤的产品级解法，Mantle 应直接抄。**

### 2.1.6 OpenFour：把 launchpad 变成协议

（GitHub `four-meme-community/openfour-docs`，README 最近更新 **2026-09-03**，仍在高频迭代）

OpenFour 把 Token / Vault / Curve / Trade / CustomData / Migrate 拆成**可替换模块**，第三方开发者通过安全审核后可部署自己的"发行模式"，并**分享该模式产生的交易激励**。

已上线的四个第三方模式：

| 模式 | 提供方 | 机制要点 |
|---|---|---|
| **GoPlus Creator Incentives** | GoPlus Security | 创作者费**随市值从 0.02% 递增，上限 1%**，自动到账，无需 claim |
| **GoPlus Skill Royalty → Royalty** | GoPlus × SafuSkill | 每个 Skill Coin 一个独立链上金库；**费用不经平台国库**；**70% 验证创作者 / 15% 经验证真实用户 / 15% 生态**；**只有通过 GitHub 验证的 Skill 所有者才能事后绑定钱包领版税** —— "先发币、后验证身份认领版税"专门防抢跑冒领 |
| **Likwid Dex** | Likwid | 对新毕业代币的**无预言机杠杆交易**：直接用池内储备定价，现货与杠杆共用一个流动性源；**风险定价费** —— 价格冲击 ≤10% 收基础费，>10% 费用随冲击**三次方**增长，接近 70%+ 时封顶 **99%** |
| **CubePeg** | Cubus × **PancakeSwap Infinity** | 用 Infinity Hook 挂在毕业后的 v4 池上，每笔 swap 刷新链上随机种子；持仓每 10 万枚自动 mint 1 枚 Art 徽章，跌破档位自动 burn |

**Royalty 框架**现支持五种模式：`Code Royalty`、`X Verified Royalty`（认证 X 账号绑定 EVM 地址，绑定后**不可更改**，未领收益 60 天后按规则处理）、**`Ecosystem Buyback Royalty`（A 币的交易费自动去买 B 币，再销毁/分红/生态激励）**、`Unverified Royalty`（版税模板市场：模板作者拿每次分配的 **6%**）、`Custom Royalty`。

> ### ⭐ 战略解读
> **当发行本身彻底商品化、费率被打到 1% 以下时，唯一的护城河是成为别人构建的底座。**
> Mantle 若要做 launchpad，这是必须提前想清楚的终局：**是做一个 app，还是做一个 registry + 模块市场？**
>
> 注意 `Ecosystem Buyback Royalty`（A 币的费用自动买 B 币）—— **这正是第六部分 6.3.7 建议的"meme 手续费反哺 mStocks 流动性"的现成模板。**

### 2.1.7 与 Binance 的真实耦合关系（一条被普遍误解的事实）

**(a) Binance Wallet Bonding Curve TGE**：认购期内代币**不可外部转让**但可在钱包内沿 bonding curve 买卖；达里程碑后自动迁往 PancakeSwap。

**(b) Meme Rush（2025-09/10，Binance Wallet Keyless 用户专享）**：三阶段 `New` → `Finalizing` → `Migrated`（达 $1M FDV 后迁 DEX）。

> ### ⭐⭐ 官方公告的原文措辞是 **"integrates four.meme's launch technology"**
> **即 four.meme 是白标基础设施供应商，Binance Wallet 是前台品牌。**
> **four.meme 不是"被 Binance 导流"，而是"成为了 Binance 钱包的发行引擎"。**
> **这是整个 BSC 飞轮里最重要的一条结构性事实，也是获客成本趋零的唯一可复制齿轮。**

**(c) Binance Alpha**：嵌在 Binance Wallet / 交易所 App 内的**预上市发现层**，官方定义路径 **Alpha → Futures → Spot**。进 Alpha **不保证**未来上现货。

**(d) TGE / Booster 参与机制**：需备份 Keyless 钱包 + KYC + **Alpha 积分门槛**（因活动而异）；按 **pro-rata** 分配、未使用资金退回、代币可能有锁仓。

**(e) 股权关系**：**无证据**表明 Binance / YZi Labs 持有 four.meme 股权。定位是"紧密生态合作 + 技术供应"。

### 2.1.8 ⚠️ 「four.meme → Alpha → 现货」不是通道，是一次性运营活动

**可确认的完整闭环案例仅 4 例**：

| 代币 | 发行 | 峰值市值 | 上现货 | 现状(2026-09) |
|---|---|---|---|---|
| **$TST** | 2025-02-05，BNB Chain 官方**发币教学视频**里的占位代币被扒出合约地址 | ≈ **$5 亿**（口径分歧） | **2025-02-09 Binance 现货** | ≈ $17–18M |
| **$Mubarak** | 2025-03-14 | > **$2.7 亿** | 2025-04-22 现货 | 未证实 |
| **$Broccoli714**（CZ 的狗） | 2025-02-13 CZ 发帖后 | 数亿美元 | 2025-04-22 现货 | 未证实 |
| **$BANANAS31** | 2024-11-16 | ATH $0.058–0.074 | 2025-04-22 现货 | ≈ **$83–85M** |

平均耗时：TST **< 1 个月**；其余三个从 2025-03 社区投票到 2025-04-22 上现货约 **6 周**。

> ### ⚠️ 关键限定
> **这 4 例全部集中在 2025 Q1–Q2 这一个特定窗口，由"CZ/何一发帖 + Binance 社区投票上币"这两个非常规事件驱动。2025 下半年至 2026 年未见规模化复制。**
> **BUBB、Palu、Koma 均未进入 Alpha 或现货。**
>
> **正确的表述是：这不是一条"通道"，而是一次"运营活动"。**
> **没有任何证据表明 four.meme 与 Binance 之间存在协议化的自动上币机制。**
>
> **→ 对 Mantle 的含义（重要的期望管理）**：不要把"Bybit 上币直通车"当作方案的核心假设。
> 真正可复制的齿轮是 **(b) 的白标关系** —— 让 Bybit Alpha 的前台复用 Mantle launchpad 的发行引擎，而不是承诺上币。

### 2.1.9 数据：一个残酷的教训

| 指标 | 数值 |
|---|---|
| **当前新建代币速率**（链上实测） | **≈ 13,700 个/天** |
| 2025-10 高峰期日均 | ≈ 16,000 个/天 |
| **当前毕业速率**（链上实测） | ≈ **10 次/天** |
| **当前毕业率** | **≈ 0.07% 数量级** |
| 2025-10 毕业率 | ≈ **1.34%** |
| 累计 Fees（DefiLlama） | **$98.05M** |
| 累计 Revenue | **$96.63M** |
| **24h Fees** | **$9,578** |
| **峰值单日 Revenue** | **$1.43M（2025-10-08）**，当天短暂超过 pump.fun 的 $1.14M |

> ### ⭐⭐⭐ 全报告最重要的一条教训
> **2026-09 的日发币量（1.37 万）几乎没比 2025-10 的高峰（1.6 万）掉多少（-14%），但收入掉了 99%（$1.43M → $9,578）。**
>
> **说明发币这个动作已经彻底沦为零成本噪音（创建费 0 + gas 0.005 BNB），真正枯竭的是曲线上的买盘。**
>
> **毕业率在一年内下降了一个数量级以上（1.34% → 0.07%）。**
>
> **→ 任何 launchpad 的 KPI 必须是「毕业数 × 毕业后 7 天存活率」，绝不能是「日发币量」。**

**必须纠正两个流行叙事**：
1. **"2025-10-08 four.meme 单日收入超越 pump.fun" 是真实但一次性的事件。** 累计口径上 pump.fun 是 four.meme 的 **21 倍**；30 天口径上是 **510 倍**。**BSC 从未在趋势层面赢过 Solana 的发行赛道。**
2. **2026 年 BSC launchpad 赛道的收入王座已经不是 four.meme，而是 Flap** —— Flap 24h Fees **$2.88M**，约占当日 BSC 全链 app fees（$5.35M）的 **54%**，而 four.meme 只占 **0.18%**。

### 2.1.10 安全事件：毕业迁移是攻击面最集中的一步

| 时间 | 事件 | 损失 | 根因 |
|---|---|---|---|
| 2025-02-11 | PancakeSwap **V3** 迁移价格校验漏洞 | ≈ **$183,000** | 迁移时若目标 pair 已存在，合约**不校验其 `sqrtPriceX96`** 直接复用。攻击者预建极端价格假池（约为正确值的 3.68e14 倍），诱导 four.meme 把 200,000,000 枚代币 + 约 **24 WBNB** 打进被污染的池 |
| 2025-03-18 | 抢跑毕业 / 绕过转账限制 | ≈ **200 BNB（$12–13 万）** | 攻击者在官方上线前低价买入，转入**尚未创建但地址可预计算**的 PancakeSwap Pair，自建该 pair 加流动性绕过 `MODE_TRANSFER_RESTRICTED` |

官方响应促成 **2025-03-31 的架构调整**：迁移目标 **V3 → V2**、**LP 强制销毁**、地址统一 `4444` 结尾。

> ### ⭐ 设计教训（对 Mantle 直接适用）
> **"毕业迁移"是 launchpad 攻击面最集中的一步。**
> **V3/CLMM 迁移需要传入价格参数 → 天然引入价格校验漏洞；V2 恒定乘积迁移不需要外部价格输入 → 攻击面小得多。**
> **four.meme 用真金白银买到的结论是：为了安全，宁可退回 V2。**
>
> 对照 Pons V2 的做法（第三部分 3.3.3）：**曲线用未来池的 quote 资产计价 → 毕业时零滑点、零预言机、零 MEV 窗口** —— 这是比"退回 V2"更优雅的解法，Mantle 应采用后者。

### 2.1.11 BNB Chain 的基础设施升级

| 升级 | 主网激活 | 关键变化 |
|---|---|---|
| Bohr | 2024-09-26 | BEP-341 连续出块、**BEP-322 Builder API**（PBS 落地） |
| Pascal | 2025-03-20 | BEP-441（EIP-7702 账户抽象） |
| **Lorentz** | 2025-04-29 | **BEP-520**：出块 **3s → 1.5s**；Fast Finality 7.5s → 3.75s |
| **Maxwell** | 2025 | 出块 **1.5s → 0.75s** |
| **Fermi / BEP-619** | **2026-01-14 02:30 UTC**（链上二分法定位 block ≈ 75,140,600） | 出块 **0.75s → 0.45s** |
| **Pasteur** | **2026-08-25** | **BEP-675**：Builder 提交**已执行完毕**的区块，消除 builder–validator 双重执行；测试网吞吐 **1237 → 2324 TPS（+88%）**，执行开销 **125ms → 15ms** |

**当前链上实测**：`gasLimit = 55,000,000`、`eth_gasPrice = 0.05 gwei`、**`baseFee = 0`**、平均 128.5 tx/block → **≈ 285 TPS**。

> ### ⚠️ 三条事实核查（纠正广泛流传的错误）
> 1. **"BEP-7928" 不存在**（BEP 仓库编号最高约 706）。真实对应的是以太坊 **EIP-7928（Block Access Lists）**；BSC 当前落地的是 **BEP-592 非共识 BAL**（随 Fermi 生效，+12%）。
> 2. **"Superblocks" 一词未见于任何官方文档**，判定为误传。
> 3. **BSC 的 `baseFee` 恒为 0 —— 它没有真正的费用市场，也没有本地费用市场、没有弹性区块空间、没有 preconfirmation。**

### 2.1.12 BSC 的 MEV 生态：集中化的双刃剑

- **结构**：没有 relay 的**许可制 builder 市场**（BEP-322 Builder API）
- **集中度**：**48Club + BlockRazor 两家控制 >87% 的区块** —— 双寡头
- **Good Will Alliance（善意联盟）**：正因为集中，**"只需说服两家即可消灭 95% 的夹子"**成为可能

> ### ⭐ 对 Mantle 的直接启示
> **Mantle 的中心化 sequencer 拥有同等甚至更强的杠杆 —— 它是唯一的区块生产者。**
> **BSC 需要说服两家 builder 才能做到的事，Mantle 一个决定就能做到。**
> **应把"协议级抗夹"作为对 meme 交易者的明确承诺。** 详见第五部分 5.7 与第六部分 6.6。

### 2.1.13 为什么 meme 流量曾流向 BSC：四个齿轮

```
① 交易所品牌力 + 名人效应（CZ / 何一发帖直接造币）
        ↓
② 交易所内置分发（Binance Wallet 白标复用 four.meme 引擎，获客成本 → 0）
        ↓
③ 极低成本（创建费 0、gas 0.005 BNB、baseFee 0）+ 持续的基础设施升级（3s → 0.45s）
        ↓
④ 上币预期（Alpha → Futures → Spot 的可见阶梯，即使只兑现了 4 次）
        ↓
    + BNB Chain 基金会的真金白银（2024-04 Meme Innovation Campaign 奖池 $1M；
      2025-02 $4.4M 流动性支持 Round 1，直接注入 BNB 并永久留存；
      2025-10 与 four.meme/PancakeSwap/Trust Wallet 联合的 $45M "Reload Airdrop"）
```

**为什么在 2026 年失速**：
- CZ 效应无法持续（4 个案例全在 2025 Q1–Q2）
- Alpha 积分体系被大规模 wash trading 污染（Binance 于 **2025-06-17** 停止把 Alpha 代币互相交易计入积分）
- 毕业率崩塌到 0.07%
- 王座易主给 Flap

---

## 2.2 Base：创作者经济路线的完整兴衰

### 2.2.1 三条主线对照

| | **Clanker** | **Zora** | **Flaunch** |
|---|---|---|---|
| 定位 | Farcaster 原生「@ 一下就发币」 | 「每个 post 即币」的创作者经济 | **Uniswap v4 hook 金融工程实验室** |
| 上线 | 2024-11-08 | Coins 协议 2025 初，Creator Coins 2025-06-20 | 2025-01 |
| AMM | v0–v3.1 用 v3；v4.0/v4.1 用 **v4 hook** | **v4**（`ZoraV4CoinHook`），Doppler 提供初始多曲线流动性 | 全程 **v4**（`PositionManager` 即 hook） |
| **有无 bonding curve** | **无**。100B 全供应一次性投成单边 LP | **无**。1B 供应直接进 v4 池（多曲线） | **无**。固定价格 Fair Launch → 全区间 AMM 两段式 |
| 核心创新 | 单边流动性 + 永久锁 LP + **模块化 MEV 模块** | **货币层级**（内容币↔创作者币↔ZORA）+ 99%→1%/10s 狙击税 | **Progressive Bid Wall** + **收益权 NFT 化** |
| 累计手续费 | **$90.82M** | **$10.43M**（Coins, Base） | **$3.59M** |
| 协议净收入 | $15.15M | $7.70M | $3.07M（主要来自 flETH 的 Aave 收益，**非抽成**） |
| 累计发币数 | **737,210**（Base 711,358 = 96.5%） | 约 160 万枚（2025-07 口径） | 151,995 |
| 近 30 天手续费 | $283,465 | $15,067 | **$389.64** |
| **峰值 vs 现在** | 2026-02 月费 $22.39M → 2026-08 $0.24M（**−98.9%**） | 2025-08 $2.51M → 2026-08 $0.015M（**−99.4%**） | 2025-01 $1.12M → 2026-08 $283（**−99.97%**） |

### 2.2.2 ⚠️⚠️ Base 官方已在 2026 年公开否定这条路线

**这是本部分最重要的新事实，也是对整个「社交图谱做冷启动」叙事的直接反证。**

2026-07-15/16，**Jesse Pollak 把 Base App 交给 Cobie（Jordan Fish）**，并写道：

> *"the entire social side of the market that many of us had been building towards - **farcaster, zora, miniapps, and yes, creator coins - disintegrated completely**… **i was definitively wrong**."*
>
> *"the collateral damage was pretty bad… and this year has been an exercise in eating shit."*

- 2026 年三大优先级改为 **"winning trading, payments, and agents"**
- **Brian Armstrong 同期表态 content coins "didn't work"**，Coinbase「今年年初就转向了」
- **Base App 的 Creator Rewards 项目已于 2026-02-18 关停** —— 7 个月共发出约 **$450,000 给约 17,000 名创作者，平均每人约 $26**

**数据侧的印证**：
- Base App 于 2025-07-16 上线当天把 Zora 接进社交流，Zora 日发币量从 <5,000 跳到**单日约 54,000（2025-07-27）**，日活交易钱包从 <5,000 到 >50,000，月协议收入从约 $600K 跳到约 $2M
- **一年后（2026-09-05→06）Zora 的 24h 新建币只有 422 枚，相比峰值 −99.2%**；月成交额从 2025-08 的 $113M 掉到 2026-08 的 **$0.5M（−99.6%）**

> ### ⭐⭐ 结论：分发能买到冷启动，买不到留存
> 这是全行业最干净的一次"分发即冷启动"实验，**结论是负面的**。
>
> **→ 对 Mantle 的含义（极其重要的期望管理）**：
> 任何"把 Bybit 的 8000 万用户导进来就能成"的假设，都必须先回答 Base 已经用 $450K 和 13 个月证伪过的那个问题：
> **流量进来之后，靠什么留下？**
> 我的答案是：**不能靠"创作者经济"这种需要持续内容生产的叙事，要靠"外生的、日历化的 catalyst"**（见第六部分 6.4）。

### 2.2.3 但第二波跑出来了 —— 而且是 AI agent，不是人

**Clanker 逐月手续费**（一手数据）：

| 月份 | 发币数 | 手续费 (USD) | **每枚币产生的手续费** |
|---|---|---|---|
| 2024-11 | 2,407 | $3,282,423 | $1,364 |
| **2024-12** | 10,039 | **$13,761,579** | **$1,371** ← 第一波峰值 |
| 2025-03 | 75,180 | $3,359,458 | $45 |
| 2025-10 | 27,687 | $6,048,779 | $218 |
| 2026-01 | 39,083 | $11,286,528 | $289 |
| **2026-02** | **261,047** | **$22,394,092** | **$86** ← **第二波，历史最高** |
| 2026-06 | 20,307 | $507,253 | $25 |
| 2026-08 | 4,614 | $242,379 | **$53** |

**2026-02 单月部署 261,047 枚（日均 9,323 枚），手续费占当月整个 Base 应用层手续费的 19.7%（$22.39M / $113.40M）。**

**诱因**：**Moltbook**（只允许 AI agent 互动的社交网络）+ Clawstr 带来的 **agent ↔ memecoin 反馈回路**。

> ### ⭐ 结论要改写为
> **Base 的 launchpad 冷启动来源是「任何高频、低摩擦、带身份的社交图谱」—— 人类的（Farcaster / Base App）和机器的（Moltbook）都算，而机器那一波甚至更猛。**

### 2.2.4 单币价值捕获塌陷比总量塌陷更本质

**Clanker「每枚币产生的手续费」：$1,371（2024-12）→ $86（2026-02）→ $53（2026-08），−96%。**

> **发币量能靠机器人 / agent 刷回来，单币价值捕获回不来。**
> **这是所有「无门槛发射」模式的共同宿命。**

### 2.2.5 价值在接口层，不在协议层

**近 30 天：Bankr（自然语言交易 agent 接口）$1.17M > Clanker + Zora + Flaunch 三个协议之和 $0.30M 的 3.9 倍。**

这与第一部分 1.0 的观察（GMGN 吃掉 RH Chain 40% DEX 量）完全一致。

> ### ⭐⭐ 跨链一致的结构性规律
> **在 meme 赛道，价值捕获的排序是：接口层 > 协议层 > 链层。**
> **→ Mantle 若只做协议层而不做接口层，会把最大的一块价值拱手让人 —— 而且现在根本没人来接（GMGN 不支持 Mantle）。**

---

## 2.3 四个必须理解的机制创新（Mantle 可直接抄）

### 2.3.1 Clanker：单边流动性 + 永久锁 LP —— 零成本发射的最简组合

1. 总供应固定 **100B**，一次铸造，不可增发
2. 可选 vault 预留（v3.1 最多 30% 最短 30 天；v4 最短 7 天，extensions 合计 ≤90%）
3. 池子用「代币 + 配对代币（默认 WETH）」初始化，把**扣除 vault/airdrop/devbuy 后的全部剩余供应**作为**单边流动性**存入 —— **池子开局是 100% 代币 / 0% 配对代币**
4. **因为 100% 供应已在池子里、且价格由 tick 定死，所以不需要 bonding curve、不需要「毕业」**
5. v4 泛化了这一点：`LockerConfig` 允许把供应拆到**最多 7 组** `(tickLower, tickUpper, positionBps)`，区间可不连续可重叠，但**全部必须 ≥ 起始 tick** —— 形态上等于一道**阶梯式卖墙**
6. LP 仓位 NFT 交给 locker，**locker 没有 withdraw 函数** → LP 永久锁死

**费用分成（推翻「60/40」「80/20」传闻）**：官方唯一的协议级数字是 **协议费 = 创作者 LP 费 × 20%，加在上面（不是从里面切）**：

| 创作者 LP 费 | 协议费 | 交易者实付 | 协议占比 |
|---|---|---|---|
| 1% | 0.2% | **1.2%** | 16.67% |

创作者那一侧由 `LockerConfig.rewardBps` 拆给**最多 7 个**接收人，**部署后不可改**；每个接收人可独立选择收 WETH / 收 meme 币 / 两者都收（`FeeIn` 枚举）—— **创作者可以选择只收 WETH 避免自己砸盘**。

> **给 Mantle 的启示**：单边流动性 + 永久锁 LP 是「零成本发射 + 反 rug」的最简组合，**不需要写 bonding curve，也不需要毕业逻辑**，把复杂度从"两套定价系统 + 迁移"降到"一次 tick 计算"。
> **代价是没有「毕业」这个天然的注意力事件**，也没有曲线阶段的抗夹优势。

### 2.3.2 三种抗狙击范式的取舍（Mantle 的核心选择题）

| 模块 | 机制 | 取舍 |
|---|---|---|
| **硬延迟**（Clanker `MevModule2BlockDelay`） | 新池前 2 个区块完全不可交易 | 最简单，但**把 MEV 价值直接烧掉了，谁都没拿到** |
| **拍卖**（`ClankerSniperAuctionV0/V2`） | 最多 5 轮、每 2 区块 1 轮，狙击者用 gas price 竞价"下一笔 swap 的执行权"；**拍卖收入按 80/20 分给代币奖励接收人 / Clanker** | 把 MEV 价值**回流给创作者**，但**依赖链的 priority ordering**，需精确落块能力（官方自己吐槽 Base 上没有 bundle 支持） |
| **衰减费**（`ClankerMevDescendingFees` / Zora 狙击税 / Pons 衰减税） | Clanker：起始费 ≤80%，**抛物线**衰减 `fee = endingFee + feeRange × (timeDecay/timeToDecay)²`，官方推荐 **80% 起、30 秒衰减到 5%**；Zora：**99% → 1%，10 秒线性衰减** | **最通用、不依赖链的排序语义**，把 MEV 价值转成 LP 费分给所有受益人 |

Clanker 文档里的一个算术很说明问题：**默认起始市值 $40k + 80% 起始费 = 狙击者的实际入场市值 $200k。**

> ### ⭐ 结论
> **衰减费的移植性最好，Mantle 应优先选这条。**
> **但（关键约束）：Mantle 的 2 秒出块会让衰减曲线粒度过粗**（见第三部分 3.3.4）。
> → 第六部分 6.3.5 给出了针对 2 秒出块的替代方案。

### 2.3.3 Flaunch 的 Progressive Bid Wall —— 本报告最值得抄的单个机制

合约 docstring：
> *"a single sided liquidity position (**Plunge Protection**) that is placed **1 tick below spot price**, using the ETH fees accumulated. After each deposit into the BidWall **the position is rebalanced to ensure it remains 1 tick below spot**."*

**接线方式**（BidWall 本身不是 hook）：
- `PositionManager` 才是真正的 v4 hook；`BidWall` 是卫星合约
- `PositionManager` 在 **`afterSwap`** 的费用分配路径里调 `BidWall.deposit(...)`，传入的是**社区侧（非创作者）那部分 1% 手续费，且已折算为 ETH（flETH）**
- 在 **`beforeSwap`** 里调 `checkStalePosition(...)`，若 **`staleTimeWindow`（默认 7 天）**内无 BidWall 交易，强制重定位

**触发阈值**：`_swapFeeThreshold = 0.1 ether` —— *"A new PBW is created for every 0.1 ETH of trading fees it receives."*

**挂单位置计算**：
```
TICK_SPACING = 60
baseTick     = nativeIsZero ? currentTick + 1 : currentTick - 1   // 现价外 1 个 raw tick，ETH 买侧
newTickLower = validTick(baseTick)                                 // 对齐到 60 的倍数
newTickUpper = newTickLower + 60                                   // 恰好一个 tickSpacing 宽
liquidity    = getLiquidityForAmount0/1(...)                       // 只用 ETH/flETH 计量
```
→ **这是真正的单边限价买单，不是对称 LP 仓位。**

**「棘轮上移」的实现（精髓）**：每次 `_reposition()`：
1. **先完全移除**旧 BidWall 仓位，收回其中的 ETH **和/或** memecoin
2. **再重建**一个新仓位，规模 = `ethWithdrawn + newFees`，位置按**当时的** currentTick 重新算

因为价格涨过之后旧仓位的 ETH 已被吃成 memecoin，收回的 ETH + 新手续费会被部署到**新的、更高的**现价下方 → **买墙价位单调不降**。

**被吃单之后的钱**：`memecoinWithdrawn` → **直接转给该代币的 `MemecoinTreasury`**（不烧、不卖）；`ethWithdrawn` → 循环进新仓位。

> ### ⭐⭐ 为什么这比「用手续费市价回购」更优
> 官方论点：*"PBWs have the effect of supporting price, **without risk of loss to MEV or bots via market buys**, achieving a more effective result for memecoin holders."*
>
> **市价回购是一笔可预测的大买单，必然被三明治夹；挂成限价单则是被动成交，MEV 无从下手**，且提供了真实的挂单深度。
>
> **这一点对 Mantle 尤其重要 —— 在流动性薄的链上，被动挂单比主动市价买入的滑点损耗小一个数量级。**

### 2.3.4 Flaunch 的另外四个可抄机制

**(a) Fixed-Price Fair Launch（固定价格公平发射窗口）**
- **默认 30 分钟**，把总供应的一个百分比放在**单一 tick** 上 → **窗口期内所有人同价**，degen、bot、KOL 一视同仁
- **只能买不能卖**；但窗口结束后可按同价卖出（扣手续费）→ **价格风险为零**
- **窗口结束时的两步（设计上最漂亮的一步）**：
  1. Fair Launch 募到的**全部 ETH → 立刻做成一道位于现价下方的买墙（PBW）**，保证 Fair Launch 买家能按入场价原价退出
  2. **未卖出的额度 + 剩余供应 → 全部投成从当前现价起的全区间仓位**，价格发现正式开始
- **与 bonding curve 的本质区别**：价格在整个窗口内**完全平坦**，然后**跳变**到全区间 AMM —— 是**离散两段式，不是平滑曲线**

**(b) 创作者收益 0–100% 可调 + Memestream 收益权 NFT 化**
- swap 费固定 **1%**，买卖双向；**池子的 v4 原生 `fee` 参数被设为 0**，所有真实收费在 hook 内由 `FeeDistributor._captureSwapFees` 执行
- **创作者份额发币时一次选定 0%–100%，发币后不可改**；默认 UI 分法 **创作者 80% / 社区回购（PBW）20%**
- **创作者没拿的部分自动全进 PBW**
- **Royalty NFT / "Memestream"**：发币时 mint 一枚 ERC-721 给创作者，代表**该代币创作者手续费流的独占权利**，可自由转让 → 解锁：**卖掉未来现金流 / 以未来收入做抵押借贷 / 碎片化 / 利率互换 / 真正的 CTO（社区接管）市场**（创作者跑路后社区可在二级市场买下收入流 + 管理权）

**(c) flETH：把 AMM 里躺着的 quote 资产做成生息资产**
- flETH 是 1:1 ETH 背书的 ERC-20 包装
- **闲置在 Flaunch 池里的 flETH 背书资产会被扫进 `AaveV3Strategy` 吃 Aave v3 借贷收益**（FAQ 称 "ultra low risk 2% yield"）
- 这份收益归 Flayer Foundation —— **这就是 DefiLlama 上 Flaunch "Revenue" 那一行的来源，而不是手续费抽成**

> ### ⭐⭐ 这是第二个最值得抄的机制，且对 Mantle 是天然契合
> **Mantle 有 mETH / cmETH 这套生息 ETH 基础设施。**
> **「launchpad 的 quote 资产默认是生息资产」在 Mantle 上是结构性优势。**
> 详见第六部分 6.3.4。

**(d) `InternalSwapPool`：用真实用户的 swap 对冲协议手续费，而不是去砸盘**
- docstring：*"**Frontruns Uniswap** to sell undesired token amounts from protocol fees into desired tokens ahead of fee distribution, acting as a **partial orderbook** that removes impact against the pool."*
- 当手续费收到的是非原生币时，不去公开 AMM 砸盘，而是**用真实用户的 swap 在内部对冲掉**
- 定价用 **`oracle.twapTick()`（绝不是 spot）** → **原子内操纵不可行**

**(e) Game Mode（2026 新增，目前在 Robinhood Chain 上线）**

问题定义说得非常准：
> *"A fair launch is meant to give everyone the same shot at a coin. In practice **the first block goes to whoever has the best infrastructure**… Technically nothing stopped you from buying; practically the good price was gone before you saw the coin."*

机制：把「抢跑竞赛」换成「窗口」—— 窗口开着时，**唯一上曲线的方式是玩游戏赚额度**。
1. 代币带窗口发射。**窗口打开前，池子里每一笔 swap 都 revert —— 这是池子自己的规则，不靠游戏服务器执行**
2. 所有人共享一个窗口、一个排行榜；**计分在服务器端，服务器根据玩家输入重放每一步动作**（不信任浏览器）
3. **分数换成 ETH 花费额度**，上限是对所有人一致的 per-wallet cap
4. 领取额度产生一个**签名授权**，池子在每一笔 swap 上校验签名与金额
5. 窗口关闭，门禁解除

**池子（而非服务器）保证的四件事**：窗口前不能买 / 不能买超过赚到的 / 不能买超过份额（**累计**强制）/ **代币一定会按时进入公开市场**（门禁带写进池子的 expiry，过期后自行停止执行，**不需要任何人发交易**）。

**一次真实轮次的分发数据**：

| 时点 | 已售供应 | 持有钱包数 |
|---|---|---|
| 10 秒 | 0.72% | **2** |
| 60 秒 | — | 138 |
| 窗口关闭 | **35.55%** | **194** |

**对照组**：一个 15 钱包的 bundler 集群，**窗口内买入 0.00%**，窗口结束后在公开市场买了 9.27%，**价格是玩家们已经定好的**。
官方总结：***"For the first time, the players front-ran the bundlers."***

> ### ⭐⭐⭐ 这是抗狙击范式的第四条路，也是对 Mantle 最有价值的一条
> **Game Mode 的本质是「用链下可验证的努力换取链上额度」的通用模板。**
> 把 `spend-gate`（trusted signer 签名 + hook 内强制 max-spend / per-wallet cap / expiry）抽出来，
> **游戏可以换成任何东西：Mantle 生态任务、mETH 持仓时长、Bybit KYC 等级、mStocks 的持仓证明。**
>
> **这比 CAPTCHA 或白名单强得多，因为执行在 hook 里，不依赖服务器活着。**
> **并且它完美绕开了「Mantle 2 秒出块导致衰减税粒度过粗」这个硬约束 —— 因为它根本不依赖出块时间。**
> 详见第六部分 6.3.5。

### 2.3.5 Flaunch 的一个反面教材（Mantle 必须避开）

**第一个 `PositionManager` 合约的 `protocolFeeRecipient` 被写死为 Flayer Foundation 多签，不可更改。** 文档写明截至撰写时该合约上有 **4,755 枚**已 flaunch 的代币，并说明基金会**在法律上没有义务**把这些费用用于回购。

> **教训：把「收款人」写成不可变的会永久绑定一个可能过时的治理主体。设计时务必留可治理的 recipient 指针。**

### 2.3.6 Zora 的货币层级 —— 一个 Mantle 可以复用的结构

Zora 三类币，全部固定 **1B 供应**：

| | **Creator Coin** | **Content Coin** | **Trend Coin**（2026 新增） |
|---|---|---|---|
| 粒度 | 一个 profile 一枚 | 一个 post 一枚 | 一个话题/梗一枚 |
| 进池供应 | 500M（50%） | 990M（99%） | **1,000M（100%）** |
| 创作者分配 | 500M，**5 年线性解锁** | 10M，即时到账 | **0** |
| 交易费 | 1% | 1% | **0.01%** |
| 狙击税 | 99% → 1% / 10 秒 | 同 | 99% → 0.01% / 10 秒 |
| **配对资产** | **ZORA** | **该创作者的 Creator Coin** | ZORA |
| ticker 唯一性 | 否 | 否 | **是（链上强制）** |

**Trend Coin 的精巧点**：ticker 的 hash **就是 CREATE2 salt** → 地址完全由 ticker 决定，可提前预测；重复 ticker `revert TickerAlreadyUsed`。池子用**预配置的 Doppler 多曲线**：3 条曲线，供应占比 5%（宽发现区间）/ 12.5%（中）/ 20%（窄）。

> ### ⭐ 「货币层级」的含义
> **Content Coin 的 quote 是该创作者的 Creator Coin，Creator Coin 的 quote 是 ZORA。**
> 于是每一笔内容币交易都会**多跳换汇**，形成对上层货币的**结构性买盘**（注意：这是"结构性买盘"，不是"回购"）。
>
> **→ 对 Mantle 的启示**：可以设计 **meme 币 → mStocks → MNT** 的三层货币层级，
> 让每一笔 meme 交易都自动产生对 mStocks 和 MNT 的买盘。详见第六部分 6.3.7。

### 2.3.7 Uniswap v4 hooks：**Base 的技术优势不可持续，可以被完整移植**

Clanker / Zora / Flaunch / Doppler 四家的差异化 **100% 建立在 hook 之上**：
- **动态费**（狙击税）—— `OVERRIDE_FEE_FLAG`
- **LP 硬锁** —— `beforeAddLiquidity` / `beforeRemoveLiquidity` revert
- **单边流动性**
- **自定义曲线** —— `beforeSwapReturnDelta` / `BeforeSwapDelta`
- **手续费自动路由** —— `afterSwap`

> ### ⭐⭐ 结论
> **任何有 Uniswap v4 的 EVM 链（含 Mantle）都能复制这套设计。**
> **说明 Base 的技术优势不可持续，其真实优势始终是 Coinbase 分发 —— 而分发这件事，Base 自己已经承认没转化成留存。**
>
> **→ 这对 Mantle 是好消息：技术门槛不存在，需要解决的是分发与留存。**

---

## 2.4 本部分对 Mantle 的可迁移结论（12 条）

| # | 结论 | 出处 |
|---|---|---|
| 1 | **「meme × 代币化股票」已有两个生产实现**（RH Chain 的 PAIR/LONG，BSC 的 four.meme Stock Meme）。这不是可行性问题，是差异化问题 | 2.1.4 |
| 2 | **毕业阈值应锚定美元门槛而非代币数量**（four.meme 的做法），因为 quote 资产会波动 | 2.1.3 |
| 3 | **「钱进 LP 免平台费、钱进项目方口袋收 10%」的费率结构**可直接抄 | 2.1.5 |
| 4 | **Token Name Protection**（持币人达 100+ 时锁定 ticker 72 小时）是产品级解法 | 2.1.5 |
| 5 | **成为白标发行引擎 > 求上币直通车**。four.meme 的真正杠杆是 Binance Wallet 复用它的技术 | 2.1.7 / 2.1.8 |
| 6 | **KPI 必须是「毕业数 × 毕业后 7 天存活率」，绝不是日发币量** | 2.1.9 |
| 7 | **毕业迁移是攻击面最集中的一步**。最优解是 Pons V2 的"曲线用未来池的 quote 计价" | 2.1.10 |
| 8 | **中心化 sequencer 是抗夹的强杠杆**，应作为对交易者的明确承诺 | 2.1.12 |
| 9 | **分发能买冷启动，买不到留存** —— Base 已用 $450K 和 13 个月证伪 | 2.2.2 |
| 10 | **价值捕获排序：接口层 > 协议层 > 链层** —— Mantle 的接口层为零 | 2.2.5 |
| 11 | **抗狙击优先选衰减费；但 2 秒出块下应改用 Game Mode 式的额度门禁** | 2.3.2 / 2.3.4 |
| 12 | **quote 资产默认生息**（Flaunch flETH → Aave）是 Mantle 的天然优势（mETH/cmETH） | 2.3.4 |
