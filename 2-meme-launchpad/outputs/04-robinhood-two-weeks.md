# Robinhood 近两周新产品与 X 讨论（2026-08-25 → 2026-09-08）

> 窗口：最近两个星期。数据核验日 **2026-09-08**。
> 口径：Robinhood **官方几乎没有发新应用**；CT 在讨论的是 **Robinhood Chain 上第三方无许可协议**，尤其是「代币化美股做 quote 的 meme launchpad」。
> 标注：`[一手]` 官方/源码/链上 API；`[二手]` The Defiant 等媒体；`[X]` 原帖；`[未证实]` 单一二手或无法对齐。

---

## 0. 两周里大家到底在吵什么

不是「Robinhood App 又上了什么功能」。主线是：

```
可自由转账的 Stock Tokens（Jersey 结构化票据，ERC-20 + ERC-8056）
        ↓
第三方 launchpad 把 NVDA/TSLA/HIMS/AMC 当 quote 资产发 meme
        ↓
休市 135h 无股票侧套利 → 链上股票代币被 meme 池定价
        ↓
唯一 AP（Bitstamp/BBVI）决定要不要增发把价格拉回
```

X 上的热帖几乎全围着这条因果链，而不是 Morpho / Lighter 这类「正经 DeFi」。

| 日期 | X / CT 事件 | 量级 |
|---|---|---|
| 08-29 ~ 09-01 | BONER 囤 53% 代币化 HIMS，周末把 HIMS 打到 $132.64 vs NYSE $28.84 | [二手] The Defiant 2026-08-31 |
| 08-30 | PAIR 上 AMC 仙股配对，周末 ~35x | [一手] PAIR 新闻稿 08-31 |
| 09-02 | 全链终端成交破 $10 亿；GMGN 91% 成交落在 RH Chain | [二手] The Defiant 09-03 |
| 09-02 | 假冒「代币化 FAMI」+ meme「金钱菇」，纳斯达克 FAMI 盘中 +321% | [二手] The Defiant |
| 09-02 | RH Chain gas 11 天涨 82 倍，单日 $4.45M，超 ETH+SOL+TRX | [二手] The Defiant |
| 09-03 | @ponsdotfamily：Uniswap Labs 买入 $PONS "for long-term alignment"（~100 万浏览） | [X] |
| 09-03/04 | @CEOAdam vs @vladtenev vs @DanGallagherDC：「CEASE AND DECIST」/「What's the concern?」/「Send your lawyers」 | [X] 各 200 万+ 浏览 |
| 09-04 | AMC 代币供应 15.8 万 → 150 万；至少 23 个 AMC 谐音 meme | [二手] The Defiant RPC 直读 |
| 09-04 | blob 提交静默 14 分钟（链没停，L1 DA 停） | [二手] The Defiant |
| 09-05 | Pons V2 单日手续费峰值 **$11.06M**（DefiLlama） | [一手] API 2026-09-08 |

**硬约束**：Robinhood Wallet 对 >$0.50 swap **全额代付 gas，至 2026-09-29 23:59 EST**。[一手] 支持页。当前链上数据全部不是稳态。

---

## 1. 窗口内真正「新上线」的产品

### 1.1 PAIR V5 Multipool（pair.fund）—— 两周里机制上最重要的新产品

| | |
|---|---|
| 上线 | V5 2026-08-26；公开发布 + AWS 2026-08-31；$PAIR 2026-08-29（自平台发射，配对 SPY） |
| 运营 | PAIR Labs by Luxington，创始人 Tugg (@0xTugg)。**与 Robinhood 无关** |
| 口号 | **"Stop launching against ETH"** |
| 代理 | `0x8660A7F019C7943b0b0A91B8E39AFf3b6DB6Ae62`（V5 launchpad） |
| $PAIR | `0x6b1d42927b1a84ec28fa88d4fc6fa7af404966be` |
| 数据（DefiLlama 09-08） | 首笔费 08-27；峰值日费 09-04 **$160,945**；近 24h **$14,996** |

**原理（一笔原子交易）**：[一手] GlobeNewswire 2026-08-31

1. 部署固定供应 1B ERC-20（无 mint）
2. 创建者从白名单 **24 只** Robinhood Stock Tokens 里选 **1–5 只** 做 quote，权重必须加总 100%
3. 每个池按预言机开盘价注入**单边集中流动性**（Uniswap v4）
4. 全部 LP 永久锁进 `PairV4Locker`（无提取路径）
5. 可选付费 dev-buy
6. 任何一步失败整笔回滚 —— 不存在「币在、池不在」的中间态

白名单：AAPL, AMC, AMD, AMZN, BABA, BE, CRCL, CRWV, GOOGL, INTC, META, MSFT, MU, NVDA, ORCL, PLTR, QQQ, SGOV, SLV, SNDK, SPCX, SPY, TSLA, USAR。

**没有 bonding curve，没有迁移。** 「毕业」= 锁仓本金超过 `4.2 ETH × ETH/USD` 后任何人可翻的信息性标志位，**不移动流动性**。4.2 ETH 是故意沿用 Pons 的文化符号。

防狙击：发射块 + 随后 5 块（~0.5 秒）单笔买入 ≤5.5% 供应、单钱包 ≤5.0%；卖出永不受限；无转账税。

费用：每笔 swap **1%**，锁仓头寸内累积，70% 创建者 / 30% 协议。发射费 0.0005 ETH。$PAIR：协议费 90% 回购销毁。

聚合器 `PairV5MultiPoolAggregator`：用 USDG 做跨腿共同资产，拒 >15% 冲击路径。**不做池间再平衡** —— 篮子内价格对齐完全依赖外部套利。休市时这种套利可能长时间缺席。这是 PAIR 自己承认的缺陷。

**和之前有什么区别**

| 前代 | PAIR 改了什么 |
|---|---|
| pump.fun / Pons | quote 从 SOL/ETH 换成 **真实美股代币** |
| LONG（单股票配对） | 一笔交易同时配 **1–5 只股票篮子**，自称「社区指数」 |
| Clanker / Flaunch | 同样无迁移 + v4 永久锁，但 Base 没有可组合的股票代币 |
| four.meme bStocks | four.meme 仍要跨合约迁 Pancake；PAIR 从第 0 秒就在最终池里 |

**为什么受关注**

- 08-30 把 AMC 做成首个仙股配对，周末 35x，直接上了 X 热搜。PAIR 的辩护是：「这正是 multipool 存在的理由 —— 单池会死，篮子能垫」。
- 09-01 宣布但**未完全上线**的缓解：peg guard（偏离过大暂停**新发射**）、风险标签、招周末做市商。存量池仍无熔断。
- 代表作：ABSOLUTE CINEMA（AMC）、Chips Party Pack（NVDA+AMD+INTC+MU）、PEAR（AAPL+MSFT+NVDA）、X Holdings（SPCX+TSLA）。
- 规模很小：累计手续费约 $0.43M vs Pons ~$84M。价值在机制，不在流水。

来源：[一手] https://www.globenewswire.com/news-release/2026/08/31/3353221/0/en/pair-launches-the-first-multipool-rwa-launchpad-on-robinhood-chain-pairing-new-tokens-with-baskets-of-tokenized-stocks-partners-with-aws-to-scale-its-infrastructure.html ；DefiLlama `pair`。

---

### 1.2 Pez Family（pez.family）—— 09-06 才出现的零平台费 Pons 仿盘

| | |
|---|---|
| 首笔手续费 | **2026-09-06**（窗口内最新 launchpad） |
| 机制 | bonding curve → Uniswap v4 毕业池 |
| 发射费 | **0**（DefiLlama methodology 原文） |
| 协议收入 | **$0**（零平台费政策） |
| 24h / 累计费 | $45,411 / $315,041（09-08） |
| TVL | $22,060（曲线内 quote 储备） |

DefiLlama 费用拆分字段：Curve Swap Fees、Creator Tax、毕业池 swap 费；去向是创建者与 meme 回购，不是协议金库。

**和之前有什么区别**：把 Pons V2 的曲线+v4 毕业抄过来，用 **协议抽成 = 0** 打价格战。这是 launchpad 战争的经典后手，不是机制创新。

**为什么受关注**：不大。CT 主战场仍是 Pons/PAIR/LONG。Pez 是「费用内卷」信号：Pons 日费千万级之后，立刻有人用零抽成抢流量。先发优势在这条赛道几乎为零（Noxa 16 天归零已验证）。

「Loracle / Fable 5.1 AI 做的」一类来源未与一手对齐，**标 [未证实]**。

来源：[一手] https://defillama.com/protocol/pez-family

---

### 1.3 Pons 开始正式挂股票代币配对（09-03）

Pons 合约层一直是资产无关的（任何 owner 批准的 ERC-20 都可做 `pairToken`），但运营层此前主打 ETH。The Defiant 09-03 报道：Pons **当天新挂一批股票配对**，包括 UPS、SNAP、LULU、PFE、JNJ。

这不是新产品，是 **Pons 从「RH 上的 pump.fun」向「也吃股票 quote」挪半步**。股票配对的爆款（Artificial Inu、BONER）仍主要记在 LONG / Bankr 名下。

---

## 2. 窗口前已存在、这两周被 X 炒爆的产品

### 2.1 Pons V2 —— RH Chain 的 pump.fun，当前全行业 launchpad 收银机

| | |
|---|---|
| V1 | 2026-07-14，单边 Uni v3，无曲线，无迁移 |
| V2 | 2026-08-04，恒定乘积曲线 → 全范围 Uni v4 + singleton hook |
| 源码 | github.com/ponsdotdev/ponsfamily（MIT）[一手] |
| 工厂 | V2 `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e` |
| $PONS | `0x39dBED3a2bd333467115dE45665cC57F813C4571` |
| 09-08 数据 | 24h 费 **$8.41M**；峰值 09-05 **$11.06M**；累计 V2 **$83.6M** |

**原理（V2，README 原文）**：曲线用**未来 v4 池同一 quote 资产**交易。毕业时直接拿储备铸全范围 v4 头寸 —— **无 router、无 swap、无预言机**，消灭 pump.fun `MigrateV2` 的夹子窗口。

费用默认 1%：30% 协议 / 35% 创建者 / 35% 回购金库（**锁 5 年线性归属，不是销毁**）。$PONS 另有一套协议层回购销毁政策（据称费用份额 80% TWAP，**政策不是合约常量**）。链上 `0x…dEaD` 已销毁约 29.7%（09-06 直读）。

Anti-snipe：默认起始税 99%、窗口 15 秒。在 100ms 出块上 = ~150 个区块，粒度细；换 2 秒出块会退化成阶梯。

**和 pump.fun 的关键差**

| | Pons V2 | pump.fun |
|---|---|---|
| 毕业 | 同 quote 资产直铸 v4，零滑点 | 跨程序迁移，可被抢跑 |
| LP | 无出口 locker（可审计） | 销毁 LP mint |
| 链 | 100ms FCFS，无优先费 | ~400ms，优先费市场 |
| 日费（窗口内） | $5–11M | ~$0.68M |

**为什么这两周爆**

1. 08-25 起日费从 $0.35M → 09-05 $11.1M（一个月 275 倍量级）。
2. 09-03 [X] @ponsdotfamily：「Uniswap Labs has purchased $PONS for long-term alignment」。帖文 ~100 万浏览。金额、价格、地址、是市买还是配额 **双方都没披露**。背景：Uniswap 自己的 `pools.trade` 08-05 上线（零发币费、交易费 0.25%），首日发币数超过 Pons，但 08-31 日费 $38k vs Pons V2 $4.89M —— DEX 下场做 launchpad 没打死专业发行层，最后改成资本结盟。
3. Binance Wallet 同期把 PONS 加进 Alpha。
4. Arkham 把 Wintermute 链到约 $2.4M PONS 仓位（The Defiant trending，未在本轮逐笔复核）。

来源：[一手] https://github.com/ponsdotdev/ponsfamily ；[X] https://x.com/ponsdotfamily/status/2095624093944979950 ；[二手] https://thedefiant.io/news/defi/uniswap-labs-bought-pons-token-for-long-term-alignment

---

### 2.2 LONG（long.xyz）+ Bankr —— 「股票做 quote」的专业户，不是 Pons

媒体常把股票配对全算到 Pons 头上。链上最清晰的爆款都在 **LONG / Bankr**：

| 代币 | quote | 发生了什么 |
|---|---|---|
| **Artificial Inu (AI)** | NVDA | 市值 08-01 ~$1.5M → 09-02 ~$275M。NVDA 池深度是其 WETH 池的 **3 倍以上** —— 交易者主动选股票池 |
| **BONER** | HIMS | 08-20 开池。囤 31,198 / 58,714 = **53%** 全部代币化 HIMS。周末 HIMS 打到 $132.64 vs 周五收盘 $28.84。周一 BBVI 约 1 小时增发 4,000 枚才拉回 ~$30 |
| SAYLORMOON | MSTR | 持有约 26% 链上 MSTR 浮筹 |
| SPACEHOOD 等 | SPCX | SpaceX 叙事 |

**原理（七步，The Defiant 把 BONER 拆开了）**

1. RHJ（泽西）买实股、1:1 发代币。代币是债务证券，无投票权。
2. **只有 BBVI 能 mint/burn**。散户无法自己扩供应。
3. 因此链上浮筹极小：HIMS 实股 2.33 亿，链上当时只有 58,714 枚。
4. Launchpad 允许 meme 的交易对是股票代币而不是稳定币。买 meme = 把股票代币打进 AMM。
5. AMM 里付出的 quote 会留在池里。十天把一半 HIMS 锁进 BONER 池。
6. 稳定币池变薄。休市无股票做市商、预言机冻结 → 几千美元就能把包装物打到 4.6 倍。
7. 开盘 + AP 增发，溢价消失。Ondo 的同标的包装（浮筹大 18 倍）整个周末贴着股价，是对照组。

**和之前有什么区别**：pump.fun 把 SOL 锁进曲线；这里锁的是**流通量本来就极小的证券包装物**。meme 不再只是空气，它在给 RWA 定价，并在休市窗口里暂时取代一级市场。

**为什么受关注**：这是 2026 年 8 月底 CT 的主叙事。「用 meme 逼空代币化股票」—— BONER 自己的网站也承认池子相对空头寸是 0.1%，挤不了纽交所，但 X 不在乎算术。

来源：[二手] https://thedefiant.io/news/tokens/a-memecoin-called-boner-has-cornered-half-the-tokenized-hims-and-hers-float

---

### 2.3 GMGN —— 不是协议，是执行层；两周里吃掉 RH 约 40% DEX 量

09-02 全市场交易终端成交 $1.03B（Adam @Adam_Tehc Dune，1 月以来首次破十亿）。GMGN 当天 $479–491M，其中 **90.7% 落在 Robinhood Chain**（$445.5M）。30 天前 RH 只贡献 GMGN $18.2M。GMGN 的 Solana 量几乎没动。

DefiLlama 09-08：GMGN 在 RH 上 24h 费 $1.45M。

**和之前有什么区别**：Solana 时代终端（Axiom/Photon/BullX）绑定 SVM 与 pump.fun。这次是 **EVM 终端把整条新 L2 当成新赌场**，而且分发快过链自己的钱包。

**为什么受关注**：证明「发行层爆了之后，钱从哪个前端进」比「哪个 AMM」更重要。Uniswap v4 吃毕业后的池（09-06 单日 ~$1.03B），GMGN 吃路径。Mantle 缺的就是这一层。

来源：[二手] https://thedefiant.io/news/defi/trading-terminals-post-first-usd1-billion-day-since-january-2025

---

### 2.4 Uniswap `pools.trade` —— DEX 官方 launchpad，打不赢 Pons

08-05 上线：发币 0 费、交易 0.25%（Pons/PAIR 是 1%）。首日发币数超过 Pons，$PONS 当周 −49% 后反弹。08-31 日费仍只有 Pons 的 ~1%。09-03 Uniswap Labs 反过来买 PONS。

教训（已写进既有基准）：**launchpad 护城河在发行体验和费用分配，不在 AMM。**

---

### 2.5 正经 DeFi：Morpho / Lighter / Rialto —— 官方生态，X 几乎不聊

| 产品 | 角色 | 窗口内热度 |
|---|---|---|
| **Morpho**（Robinhood Earn 底层） | USDG 出借，Lloyd's + RELM 承保，~7% APY | 主网上线即有。股票代币作抵押是官方文档用例，**没有旗舰市场** |
| **Lighter Robinhood Perps** | Wallet 内永续，承诺 1,100 万 $LIT | 首笔费 07-21；峰值 08-19 $136k；09-08 24h **$41.6k** —— 相对 Pons 可以忽略 |
| **Arcus** | 另一家 perps | 讨论更少 |
| **Rialto** | PropAMM，做市商链上报价，为薄流动性股票代币设计；可组合（相对 RFQ） | 07-01 即有费；日费仍 ~$2k。CT 不讨论 |

X 的注意力分配极其残酷：**meme × 股票** 吃掉 90% 讨论，借贷/永续/PropAMM 是 Robinhood 想要的图景，不是现在的图景。

---

## 3. 不是产品、但这两周被当成「产品」讨论的三件事

### 3.1 AMC CEO vs Robinhood CLO（09-03/04）—— 监管产品化

[X] @CEOAdam（~210 万浏览）：代币化 AMC「contemptible… vile」，「CEASE AND DECIST」。
[X] @vladtenev：「What's the concern?」（~260 万浏览）
[X] @DanGallagherDC（前 SEC 委员）：「will not 'DECIST.' Send your lawyers and we'll educate them.」（~260 万浏览）
Vlad 转帖：「We stand behind Stock Tokens。」

链上后果比嘴仗更有信息量：Aron 发帖到 Gallagher 回帖的 18 小时，AMC 代币供应 **157,844 → 1,499,255**（RPC 直读）；深夜最深池打到 **$18.04** vs NYSE $2.54；至少 23 个 AMC 谐音 meme，最大的「A Meme Coin」市值一度 $65M，成交超过同期纽交所 AMC。

这是 HIMS 事件的对照实验：HIMS 周末 **没怎么增发**，所以 4.6x；AMC 被点名后 AP **连夜扩了近 10 倍浮筹**，开盘前贴回股价。锚定机制不是预言机，是 **BBVI 的自由裁量 mint**。

### 3.2 假冒代币化股票（Farmmi / JINQIAN，09-02）

匿名钱包一次铸完、自留 38%、自写 `PoolRepricer` 的假 FAMI 合约，配「金钱菇」meme。假币 $1.83 时，真纳斯达克 FAMI 盘中 +321%。这证明：二级市场完全开放 + 无链上白名单 → **任何人都能伪造「代币化某股」**。PAIR/Pons 的白名单挡新发射，挡不住假合约。

### 3.3 链本身被 meme 打成公共品悲剧

- Gas：08-22 ~$54k/日 → 09-02 $4.45M（82x）。Yakovenko 骂费用模型 "brain-dead"；Goldfeder 反驳 Orbit 让应用方当房东、留约 90% gas。
- Blob：平静期占以太坊 blob ~45%；09-04 批次地址静默 14 分钟。区块仍 101ms，停的是 L1 DA 与提款。
- 补贴 09-29 到期。所有「日费千万、DEX 量全链第二」的数字都要按补贴退出重做压力测试。

---

## 4. 一张对照表（只放窗口内有讨论的）

| 产品 | 新/旧 | 核心机制 | vs 前代 | 为何这两周有讨论 | 规模（约） |
|---|---|---|---|---|---|
| **PAIR V5** | 新（08-26） | 1–5 股票篮子、无曲线、v4 永锁 | 相对 LONG：多池；相对 Pons：quote=股票 | AMC 35x、AWS 稿、「Stop launching against ETH」 | 日费 ~$15k |
| **Pez Family** | 新（09-06） | 曲线→v4，**协议抽成 0** | Pons 仿盘打价格战 | 几乎没有，是内卷信号 | 日费 $45k，收入 $0 |
| **Pons V2** | 旧（08-04）爆了 | 同 quote 曲线毕业、99% 衰减税 | 消灭迁移夹子 | Uniswap Labs 买币、日费超 pump.fun 10x+ | 日费 $8.4M |
| **LONG / Bankr** | 旧 | 强制单股票 quote | meme 变成股票衍生品 | BONER/HIMS、AI/NVDA | 不在 Llama 独立项 |
| **GMGN** | 旧，流量迁入 | 终端/bot | Solana 终端搬到 RH | 09-02 十亿日，91% 在 RH | RH 24h 费 $1.45M |
| **pools.trade** | 08-05 | 0 发币费 / 0.25% | Uniswap 官方抢发行 | 打不过后又买 PONS | 日费 ~$52k |
| Morpho / Lighter / Rialto | 07 月 | 借贷 / 永续 / PropAMM | 传统 DeFi 搬到券商 L2 | CT 基本不聊 | Lighter 24h $42k |

---

## 5. X 讨论的结构（不是产品清单能替代的）

1. **「股票 meme」被当成新 meta，而不是「又一个 pump.fun」。** Artificial Inu 的 NVDA 池比 WETH 池深 3 倍，是最硬的市场投票。
2. **休市 = 无锚。** 预言机 24/5 冻结，AP mint 是唯一纠偏。CT 一边玩一边等周一。
3. **发行人愤怒第一次变成娱乐。** Aron 的错别字 DECIST 被做成 23 个 meme 的原材料；Gallagher「send your lawyers」是这两周传播最广的单句。
4. **Uniswap 买竞争对手** 被读成「Labs 在买 RH 的发行层」，回复里反复问是市买还是配额。
5. **几乎没有人认真聊 Morpho 或 Lighter。** 官方想讲 RWA 结算层，CT 在讲赌场。两者同时为真：meme 给了股票代币它本来不会有的链上持有需求。

---

## 6. 明确缺口

- LONG / Bankr 无与 Pons 同级的开源仓库，机制细节依赖 The Defiant 与终端数据。
- Pons 毕业率：公开源没有可靠数。
- Uniswap Labs 买了多少 PONS：未披露。
- Peg guard 是否已上线：PAIR 09-01 宣布，本轮未在合约层核实。
- Pendle 09-04 在 RH 上开了 sNET 市场（到期 09-17），上线时成交 $0；Arcadia 09-07 仍只有聚合站转述，**不采信**。sNET 是 NetNet 包装，不是 RH 原生 gas（gas=ETH）。
- 09-29 gas 补贴退出后的稳态：未知，是 Q4 最大变量。

---

## 7. 来源

- [一手] Pons 源码 https://github.com/ponsdotdev/ponsfamily
- [一手] PAIR 新闻稿 2026-08-31 GlobeNewswire（上文 URL）
- [一手] DefiLlama API `overview/fees/Robinhood Chain`、`summary/fees/{pons-v2,pair,pez-family,gmgn,lighter-robinhood-perps,rialto,pools}` 2026-09-08
- [一手] DefiLlama Pez Family methodology https://defillama.com/protocol/pez-family
- [X] https://x.com/ponsdotfamily/status/2095624093944979950
- [X] https://x.com/CEOAdam/status/2095622531524784212
- [X] https://x.com/DanGallagherDC/status/2095878611852984451
- [X] https://x.com/vladtenev/status/2095711439810159027
- [二手] The Defiant：BONER/HIMS、Uniswap-PONS、AMC、terminals $1B、Robinhood tag 列表
- 既有底稿：`2-meme-launchpad/benchmarks/robinhood-notes.md`、`robinhood-chain.md`（数据截止 09-06）

## 8. 补记（scout 交叉核验）

- Robinhood **新闻稿窗口内零新品**。最近一篇 crypto 类仍是 07-01 主网。唯一官方 X 是 09-02 数据复盘（DEX $34.6B / TVL $1.27B / Lighter 永续累计 $7.29B）：https://x.com/RobinhoodCrypto/status/2095225808595906633
- Pendle 09-04 上线 RH，首个市场 sNET，到期 09-17，当时成交 $0。[二手] Cryptonomist，文末标明 AI 辅助。CT 不讨论。
- 「NET 是 Robinhood Chain 原生代币」是错的。原生 gas = ETH。
