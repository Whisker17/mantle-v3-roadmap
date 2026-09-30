# 第七部分：Mantle 重定位 —— CeDeFi Capital Markets 叙事与新路线图

> 本部分是在前六部分研究基础上，经交互式讨论收敛得出的**定位定稿（v1）**。
> 日期：2026-09-07。四项路线决策已锁定（见 7.0），全部机制细节可回溯 [第六部分](06-mantle-design-proposal.md)，全部失败归因可回溯 [第四部分](04-mantle-gap-analysis.md)。
> 与前六部分的关系：第一至五部分是证据，第六部分是发行层的机制设计（Tape），**本部分是把发行层放回全链战略后的完整定位、产品矩阵与路线图**。

---

## 7.0 决策记录（已锁定）

| # | 决策点 | 选定方案 | 核心依据 |
|---|---|---|---|
| D0 | 系统架构模式 | **一核双星架构（Mantle L2 金库总行 + Launchpad L3 发行链 + Perps L3 衍生链）** | 清算与执行彻底解耦：大额资产与借贷留在以太坊安全级的 L2，高频打新与合约分别放入 50ms 独占 L3，彻底杜绝单链性能与状态爆炸的公地悲剧 |
| D1 | 主叙事 | **CeDeFi Capital Markets（链上资本市场）**；tagline：**Where Assets Go Public** | 覆盖全部支柱、机构友好、对 Bybit 承诺不敏感；"交易所链"仅作内部谈判目标不作公开叙事（Byreal 在 Solana 是硬反证，[E-mantle](../research/E-mantle.md)） |
| D2 | Perps 路线 | **股票 perps 优先，专有 L3-B 独立引擎，内嵌 RFS 流式撮合** | 通用 crypto perps 正面撞 Hyperliquid 必败（Mantle perps 24h = $0）；现货 xStocks + RFS 提供真实基差腿；休市定价做成 24/7 产品；享受 L2 远程保证金授信免跨链 |
| D3 | Launchpad 范围 | **股票 quote 专注（Tape 原案），专有 L3-A 独立引擎，内嵌毕业 AMM** | 通用 meme 在 Mantle 已死两次（Funny Money / Printr，2026-08 同周关停）；50ms 极速 Bonding Curve、0 Gas 准入、状态物理修剪；打新达标就地在内嵌 AMM 开盘，零跨链断层 |
| D4 | Bybit 关系深度 | **全深度：白标 + 做市 + 自营发行 + UTA 打通**（假设 Bybit 支持巨大） | 按四条可独立验收的工作流推进（7.8）；roadmap 对 W1/W2 缺席保持稳健，W3/W4 是 upside |

---

## 7.1 定位与叙事

### 7.1.1 一句话定位

> **Mantle 是由 Bybit 支撑的链上资本市场（CeDeFi Capital Markets）——真实世界资产与加密原生资产在这里发行、定价、交易、加杠杆；交易所提供承销、做市与分发，链提供全天候的市场结构与最终清算。**
>
> Tagline：**Where Assets Go Public（万物上市）。**

「资产发行 + 资产交易」= 一级市场 + 二级市场 = **资本市场**。"上市（go public）"一词同时统一了四件事：meme 发射、代币化 IPO、xStocks 上链、毕业进 Bybit。

### 7.1.2 叙事弧线：从链上银行到链上资本市场

新叙事不推翻旧叙事，而是**补全它的另一半**：

> 过去三年，Mantle 建成了链上资本市场的**买方**：银行入口（UR 新银行）、资管产品（MI4，AUM 破 $200M）、收益基础设施（mETH/cmETH/Mantle Vault）、全球第 2 的代币化股票库（xStocks $633.7M、155 个标的）。
> 结果是链上沉淀了 **$576M 稳定币**（DeFi TVL 的 5.9 倍）——**有钱，没有市场**（日 DEX 量 $0.94M，日活跃地址约 1,000）。
> 现在，Mantle 建设**卖方与场内**：发行（Launchpad + 代币化 IPO + 自营股票发行）、做市（prop AMM + Fluxion RFQ）、交易（现货 + 股票 perps）、融资（xStocks/mETH 抵押借贷）。
>
> **银行聚集了钱，资本市场让钱动起来。**

这条弧线的三个好处：
1. **承认而不否定过去**（对内部与社区政治友好）；
2. 把"$576M 闲置稳定币 vs $0.94M 日交易量"这个最尴尬的数据**变成叙事的起点而非污点**；
3. 与 E-mantle 底稿的诊断严丝合缝：Mantle 历史激励预算（EcoFund $2 亿、Methamorphosis、Journey 2,500 万 MNT）全部结构化为质押/锁仓，"奖励不动的钱"，**从未建立过奖励交易的飞轮** —— 本叙事就是那个转折点。

### 7.1.3 与 Solana "Internet Capital Markets" 的三词差异化

"资本市场"叙事正在被争夺（Solana 推 ICM、Ondo 推 Onchain Capital Markets），Mantle 必须占住别人占不住的变体：

| Solana ICM | **Mantle CeDeFi Capital Markets** |
|---|---|
| 无许可、加密原生资产为主 | **合规资产轴心**（155 个 xStocks，全球第 2；sequencer 级合规过滤） |
| 公开竞价流动性（LP + MEV 生态） | **交易所库存做市**（Bybit prop 库存 + 无公开 mempool 天然抗夹） |
| 7×24 加密时间 | **唯一认真处理"股票休市"的链**（RFQ 双模 + 熔断 hook + perp 休市定价） |

### 7.1.4 "Application-specific" 的具体含义：解耦为「一核双星」

过去讨论中曾担忧“单开 appchain 会切断流动性”，那是因为传统 appchain 依赖粗糙的第三方多签桥。
**Mantle v3 锁定的「一核双星」彻底重定义了 App-Specific 的工程形态**：
- **不是在 L2 上带着镣铐跳舞**（L2 必须与以太坊硬分叉对齐、必须保证 25 亿国库绝对安全，无法无底线将出块压缩至 50ms 或随意删除状态）；
- **也不是建立孤立的通用 L3**（若搞出第 3 条通用 DeFi L3，会产生灾难性的跨 L3 碎片化）；
- **而是「L2 中央信贷总行 + 双专有 L3 极速交易大厅」**：

| 专有层级 | 部署与承载模块 | 专属性能与特权 | 解决的核心矛盾 |
|---|---|---|---|
| **Mantle L2**<br>(中央总行) | • mStocks 现货总金库<br>• Mantle Lending (信贷扩张)<br>• Unified Margin Hub (统一授信) | • 严格对齐以太坊 Hardfork<br>• OP Succinct SP1 ZK 证明<br>• 焊死 2% 速率限流熔断断路器 | 确保 25 亿国库与 5.76 亿稳定币底池拥有不可动摇的物理级安全性 |
| **L3-A 发行专属链**<br>(Tape Launchpad) | • 极速 Bonding Curve 抢打新<br>• **内嵌毕业 AMM DEX**<br>• 48h 未达标死币物理剪枝 | • **50ms 独占出块**<br>• 确定性 FIFO 排序（无抢跑夹子）<br>• 0 Gas 准入 (官方代付) | 彻底消除科学家三明治攻击，根除死 Meme 对全网全节点的状态树污染；毕业就地开盘零延迟 |
| **L3-B 衍生品专属链**<br>(Equity Perps) | • 24/7 股票永续合约撮合<br>• **内嵌 RFS 流式做市中枢**<br>• 极速清算配额通道 (Liquidation Lane) | • 毫秒级流式软报价撮合<br>• 做市商 Cancel 绝对 0ms 优先<br>• 接受 L2 远程信用映射 (Remote Margin) | 突破美股周末休市限制；实现日内零敞口基差套利；资产 100% 留在 L2，合约在 L3 肆意开平仓 |

### 7.1.5 资本市场功能映射（叙事的骨架）

五个产品支柱不是并列的功能清单，而是**复刻一个完整资本市场的功能分层**：

| 传统资本市场 | Mantle 对应 |
|---|---|
| IPO / 一级发行 | Launchpad（草根发行）+ xStocks / 代币化 IPO / 自营发行（机构发行） |
| 二级市场撮合 | Prop AMM / RFQ（主流资产最优执行）+ AMM DEX（长尾） |
| 衍生品交易所 | 股票 Perps（24/7 定价 + 杠杆） |
| 融资融券 | Lending（xStocks/mETH 抵押） |
| 承销商 + 做市商 + 经纪商 | **Bybit**（7.8 的四条工作流） |
| 清算结算所 | Mantle L2 本身（ZK validity、12h 提现） |

---

## 7.2 五支柱产品矩阵（分层归属）

| 支柱 | 锚定产品与层级 | 差异化来源 | 北极星 KPI |
|---|---|---|---|
| **发行层** | **Tape（部署于 L3-A 发行专属链）** + 内嵌毕业 AMM | 50ms 极速出块；0 Gas 准入；财报外生日历；状态物理修剪；打新毕业原地相变为二级池 | 毕业数 × 7 天存活率 |
| **衍生层** | **股票 perps（部署于 L3-B 衍生专属链）** + 内嵌 RFS 流式撮合 | 24/7 跨休市定价；内嵌 RFS 流式报价；接受 L2 远程保证金授信；现货基差无风险套利 | OI、基差收敛速度、休市时段成交占比 |
| **融资层** | **Mantle Lending（部署于 Mantle L2 总行）** | 质押 mStocks/mETH 借出沉睡资金；为 L3 签发远程信用额度；激活 $576M 稳定币 | 稳定币链上利用率、远程授信规模 |
| **现货交易层** | **内嵌 RFS (L3-B) + 内嵌毕业 AMM (L3-A) + L2 大金库** | 无公开 mempool 抗夹 + 做市商 Cancel 优先 + 零补贴做市 | 主流资产对 CEX 价差（bps）、有机 DEX 量 |
| **接口层** | **Tape Terminal（自建）+ Bybit Alpha 白标** | 打破"无执行工具"死锁（GMGN 等七大工具零覆盖）；价值捕获排序接口层最高 | Terminal DAU / 白标端交易占比 |

**支柱间跨层飞轮**：

```
Mantle L2 资产总行（Lending 质押 mStocks 借出稳定币 ｜ 签发远程信用额度）
   │ 意图闪电充值通道                               │ 远程保证金映射 (免跨链)
   ▼                                                ▼
L3-A 发行专属链 (Tape 50ms 抢买)            L3-B 衍生专属链 (股票 Perps 24/7 高频撮合)
   ↓ 达成 $12k 阈值就地相变                          ↑ 现货腿对冲与资金费率套利
内嵌毕业 AMM (零延迟直接二级交易) ─────────────── 内嵌 RFS 流式报价深池
   │                                                │
   └────────────────────────┬───────────────────────┘
                            │ 25% 注入做市金库 + 20% 强制市价回购 MNT
                            ▼
           MNT 回购销毁池与国库储备 (价值终极沉淀)
```

**发行层是五支柱中唯一负责"制造需求"的，其余四个都是承接需求的。** 所以 7.3 单独展开。

---

## 7.3 发行层主体：Tape meme Launchpad —— 资本市场的点火器

> 完整机制设计见[第六部分](06-mantle-design-proposal.md)；本节给出它在新定位中的角色、与两次失败的本质区别、与其余支柱的咬合关系。

### 7.3.1 定位：不是赌场，是「代币化股票的情绪一级市场」

> **每一个 xStocks 标的，都拥有一个可交易的、社区驱动的情绪市场；这个市场的所有现金流回流到 xStocks 深度、Fluxion 流动性和 MNT。**

"造富效应吸引用户、meme 是天然的资产发行点"这一判断完全保留，但**造富效应的来源被替换了**。这是与 Funny Money / Printr 两次失败最本质的区别：

| | 失败的两次（通用 meme） | Tape（股票 quote） |
|---|---|---|
| 注意力来源 | 内生——指望凭空制造病毒传播 | **外生——寄生在美股本来就有的注意力周期**（财报、FOMC、IPO、指数调整） |
| catalyst 节奏 | 随机，无法运营 | **日历化，永不停歇**（每个财报季自动供给叙事） |
| 目标用户 | 与 Solana/BSC 抢 meme 原住民（执行层零覆盖，必败） | **美股关注者 + Bybit 8,000 万用户 + xStocks 持有者** |
| 与主业关系 | 无关，甚至污染 RWA 品牌 | **每一笔交易都在给 xStocks 引流和加深度** |

### 7.3.2 核心机制要点（六条，全部可回溯第六部分）

1. **发行形态：xStocks 作 quote 的 bonding curve。** 用 NVDAx 买 "NVDA 情绪币"。曲线用未来 v4 池的同一 quote 资产计价（Pons V2 哲学）→ 毕业零迁移、零预言机、零 MEV 窗口，在 hook 内相变（规避 four.meme 被黑两次的迁移攻击面）。参数：1B 固定供应、75% 曲线可售、毕业阈值 $12K 等值（美元锚定、治理调参）、创建费为零、交易费 1%。
2. **抗狙击：Spend-Gate 额度门禁。** Mantle 2 秒出块让衰减税失效、无费用市场让 gas 竞价失效，spend-gate 对两者免疫。额度来源 = **mETH/cmETH 持仓、对应 xStocks 持仓、Bybit 账户等级** —— **参与 meme 的门票就是持有主业资产**，发射本身在为 xStocks 和 Bybit 创造需求。这是"点火器反哺主业"的第一个具体咬合点。
3. **两个全行业无人实现的安全机制**：**休市熔断 hook**（RH Chain 已发生 HIMS 4.6×、AMC 35× 周末脱锚且零协议级熔断；Tape 用预言机偏离熔断 + Fluxion RFQ 自动收敛 = "链上自动 AP 增发"）；**公司行动池层适配 hook**（拆股/分红对永久锁仓 LP 等价重铸，Robinhood 自标"未解决风险"）。这两条是"凭什么在 Mantle 做"的技术答案。
4. **费用路由 = 飞轮的经济灵魂**：`1% = 45% creator（Fee Key NFT 永续）/ 25% 反哺 Fluxion 的 xStocks 池 / 20% MNT 回购（写进合约而非博客）/ 10% 协议金库`。所有竞品（pump.fun/Pons/PAIR）的协议收入都在回购销毁自家平台币、对底层生态贡献为零；Tape 是唯一把 launchpad 现金流接回主业资产的设计 —— **"引爆生态活跃度"从口号变成会计科目的地方**。
5. **节奏引擎 Sentiment Engine**（按合规风险从低到高）：**Ticker Wars**（财报季 `$NVDA-BULLS` vs `$AMD-BULLS`，纯排行榜无结算，先做）→ **Index Coins**（1–5 个 xStocks 组篮子作 quote，quote 侧本身就是故事）→ **IPO 主题币**（SPCXx 式代币化 IPO 的稀缺性是天然投机题材，仅做纯主题社区代币形态，法务通过后）。
6. **毕业阶梯**：`曲线 → v4 永久锁池 + Fee Key → Fluxion 主路由 → Bybit Alpha 专区（W1 白标）→ Bybit 现货（只给公开数据标准，不承诺）`。BSC meme 季的真正引擎就是这条可见的阶梯；Mantle 每一级都已存在，只是没连起来。

### 7.3.3 与其他四支柱的咬合关系

```
Launchpad 是需求的源头：
├─ 对 prop AMM：用户要买 quote 资产（NVDAx）→ prop AMM 提供最优价入口
├─ 对 perps：毕业代币上 perps（第二交易场景）；财报事件同时点燃现货曲线与 perp OI
├─ 对 lending：抵押 xStocks 借稳定币加仓 → 激活闲置资金
└─ 对 Terminal：launchpad 是 Terminal 的第一个杀手内容（新币流/战壕面板/额度展示）
```

### 7.3.4 KPI 与红线

- **北极星：毕业数 × 毕业后 7 天存活率**，绝不用日发币量（four.meme 发币量只跌 14% 而收入跌 99% 的教训）。P2 验收线：毕业 ≥3/周、存活率 ≥30%、曲线日成交 ≥$200K。
- **红线**：不做通用曲线（第三次死亡没有借口）；标的白名单 + 冻结治理预案（AMC 的 CEO 反弹先例，Bybit 的法律隔离结构不支持 Robinhood 式硬刚）；补贴必须有阶梯退出曲线（不学 Robinhood 的 9-29 悬崖）。

### 7.3.5 诚实的预期管理：日历锁定的引爆上限

选择"股票 quote 专注"后，**launchpad 的活跃度峰值出现在财报周与 IPO 事件，而非 pump.fun 式全天候随机爆发**。这是拿"可运营、可持续、反哺主业"换"最大理论流量"的交易 —— 换得值（Mantle 社区本就是收益农民而非 meme 猎手，通用流量接不住），但形态预期应是**"每季四次的超级碗"，不是"永不打烊的赌场"**。日历间隙期的填充物（指数篮子长线叙事币、Fee Key 的 CTO 市场等常青玩法）列为开放问题 Q4。

---

## 7.4 现货交易层：Prop AMM（做市商私有报价层）

**定义**：做市商私有库存报价（SolFi/Obric/Hashflow 形态）——对 CEX 价格报价、无公开 LP、走聚合器路由。**本质上 Fluxion 的 Atomic RFQ 已经是 xStocks 的 prop 场所，本支柱 = 把它泛化到全资产（BTC/ETH/MNT/稳定币），库存来自 Bybit（W2）。**

Mantle 的四重结构适配：
1. **无公开 mempool** → 做市商报价天然不被三明治（第四部分标注"真实但从未被宣传"的优势，应写成对交易者的可验证承诺）；
2. **零补贴做市** → 完美匹配"机制驱动而非补贴驱动"硬约束（可动用流动稳定币仅 $1.2 亿；Aave 激励 TVL $137M→$704M→$62M 已证伪补贴路线）；
3. **聚合器路由已就绪**（OKX / KyberSwap / LI.FI 均支持 Mantle），报价接入即可用；
4. **链级配套**：预确认流保证报价新鲜度 + 做市商撤单优先防陈旧报价被狙（7.1.4）。

**对现有 DeFi 版图的态度：收敛而非建设。** 28 个 DEX 中 15 个休眠、前三名占 99% 交易量——供给侧堆砌已被证伪。现货收敛到 Fluxion（prop/RFQ 主场）+ Agni（长尾 AMM），不再引入新的通用 DEX。

**KPI**：主流资产对 CEX 价差（bps）、报价在线率、有机 DEX 量（从 $0.94M/日的基线增长，不计激励量）。

---

## 7.5 衍生层：股票 Perps（oracle-based 起步）

**为什么是股票 perps 而不是通用 perps**：
- Mantle perps 24h 量 = $0，正面对撞 Hyperliquid 无胜算；且 HL 的 HIP-3 已能开股票 perp 市场、Ostium 在 Arbitrum 做 RWA perps——赛道不空白，必须打结构差异。
- **Mantle 独有的结构差异：链上有现货 xStocks + Fluxion RFQ 的 mint/redeem 通道** → perps 资金费率套利有一条真实的现货腿。这是有机交易量的来源，不需要补贴。
- **休市定价从事故变产品**：RH Chain 的周末现货脱锚（HIMS 4.6×、AMC 35×）证明现货 AMM 天生不适合休市时段的股票投机；**现金结算的 perp 恰恰是休市定价的正确工具**——资金费率把价格拉回，不存在库存脱锚。别人把休市当事故，Mantle 把休市做成产品。

**形态决策（D2）**：oracle-based perps 起步（Ostium 式，落地最快、适配 RWA），CLOB 留作 v2 选项。Bybit 做种子做市（W2）；Bybit CEX 同标的 perps 与链上 perps 之间的 CEX–DEX 套利是深度的自然来源。

**与发行层的联动**：财报事件同时点燃现货曲线与 perp OI；毕业 meme 代币可上 perps（第二交易场景，P3）。

**KPI**：OI、基差收敛速度、休市时段成交占比（这是"休市定价是功能"叙事的直接量化）。
**开放技术命门**：休市 mark price 与资金费率政策（开放问题 Q1）。

---

## 7.6 融资层：Lending（激活 $576M 的钩子）

不做又一个通用 Aave fork（Aave 来过，TVL 崩塌已演示过 mercenary capital 的结局）。差异化钩子只有一个：**xStocks 与 mETH/cmETH 作为抵押品**。

- 真实需求链条：`持有 NVDAx → 抵押借 USDT → 加仓曲线/perps` —— 这是"融资融券"，不是"存款吃息"；
- $576M 闲置稳定币是出借侧的现成供给（利用率是北极星）；
- 风控依赖与熔断 hook 同一套市场日历 + 预言机基础设施（休市折价、staleness 检查），与 7.3.2 第 3 条共享建设。

---

## 7.7 接口层：Tape Terminal + Bybit Alpha 白标（第五支柱）

**这是初版设想中缺失、但证据上优先级最高的支柱。** 价值捕获排序「接口层 > 协议层 > 链层」（GMGN 占 RH Chain 40% DEX 量；GMGN/Photon 收入与 pump.fun 同量级），而 Mantle 的执行层覆盖为**零**（GMGN/Photon/BullX/Axiom/Trojan/Banana Gun/Maestro 全部不支持；GMGN 连 Monad、MegaETH 都收录了）。死锁只能自建打破。

- **Terminal 最小范围**（第六部分 6.6.3）：发现（新币流/按 quote 分组的战壕面板）、安全（锁池/creator 持仓/老鼠仓）、执行（一键买卖/限价/quote 一键兑换）、额度（spend-gate 展示）为 P0；移动端（Mantle Passport 无助记词 + paymaster 无 gas）、跟单、Telegram bot/开放 API 为 P1。
- **双前端战略**：自建 Terminal（链上原生用户）+ Alpha 白标（W1，托管用户），后端同一套协议。
- 并行 BD：争取 GMGN/DEXTools/Birdeye 深度收录（Birdeye 已为 Bybit Alpha 供数，最易打通）。

---

## 7.8 Bybit 全深度整合：四条工作流（D4）

一句话定位：**Bybit 是这个资本市场的承销商、做市商、经纪商三位一体；Mantle 是交易所与清算所。** 链负责场所和结算，Bybit 负责资产、库存和分发。四条工作流可独立推进、独立验收：

| 工作流 | 内容 | 落地方式 | 验收标准 |
|---|---|---|---|
| **W1 经纪商：Alpha 白标**（最先落地） | 复刻 "Binance Wallet integrates four.meme" 关系：Alpha 前台托管账户、无助记词无 gas，每笔都是 Mantle 链上真实交易 | **不是把 Alpha 用户"导"上链**（与 Alpha 产品逻辑对抗，必败），而是让 Alpha 把 Tape 当后端发行引擎 | 白标端贡献 ≥50% 曲线成交 |
| **W2 做市商：prop AMM 库存 + perps 种子流动性** | Bybit 自营盘/做市伙伴为 prop AMM 供报价库存、为股票 perps 做种子 | Byreal（RFQ+CLMM）恰好证明 Bybit 愿意做链上 prop 式做市——**把同一套库存承诺复制到 Mantle 的股票与主流资产** | 主流资产对 CEX 价差 ≤ 目标 bps、报价在线率 |
| **W3 发行方：自营股票发行**（战略含金量最高，周期最长） | bStocks 路线：BTech 式关联主体 + ADGM/FSRA 级招股书，**原生发在 Mantle** | 解决 xStocks 输给 bStocks 的根因（上新速度受制于 Backed：早 7 个月上线，仍被 7 周 $500M AUM 反超）。**与 Backed 双轨**：存量 155 个 xStocks 继续（Fluxion RFQ 是其独有资产），自营线补上新速度；避免谈判初期变成替换关系刺激 Backed | 首批自营标的在 Mantle 原生发行 |
| **W4 主经纪商：UTA 打通** | Mantle 成为默认零费充提网络；链上头寸（xStocks、LP、Fee Key NFT）计入统一账户保证金 | xStocks 已部分接入 UTA 抵押，是延伸而非从零谈；**这是 Binance 都没做到的深度，是 "CeDeFi" 三个字的实体** | 链上抵押品在 UTA 内的规模 |

**对 Byreal 的公开口径**：分工——Solana = 散户 crypto 现货，Mantle = 资产发行与 RWA 资本市场（E-mantle 显示 Bybit 内部本就把 Mantle 定位为 "CeDeFi/RWA 链"，顺水推舟）。

**稳健性约定**：roadmap 主线只依赖 W1+W2；W3/W4 是 upside，任何一条缺席，方案降级为"中档"而不崩溃（备选：自建移动端 + Mantle Passport 分发）。

---

## 7.9 MNT 价值捕获：叙事发布时的核心承诺

E-mantle 实测结论：**MNT 今天没有任何价值捕获**——无 fee switch、无回购销毁、sequencer 收入进国库不分配、真实收益全部流向 mETH/MI4 而不流向 MNT。

新定位给 MNT 第一条真实现金流：**协议费的 20% 回购 MNT，写进合约（不是博客）**（对照：Pons 的 80% 回购是"政策"团队随时可调；Flaunch 的 recipient 写死多签成为反面教材——Tape 的所有 recipient 是可治理指针，比例是合约条款）。

- 对外：这是"链上资本市场的股权"叙事；
- 对内：这是激励预算从"奖励不动的钱"转向"奖励交易的钱"的转折点；
- 附带：spend-gate 额度来源包含 mETH/cmETH 与 xStocks 持仓 → 为主业资产创造持有需求。

---

## 7.10 路线图（四轨并行推进）

D0/D2/D3 确立「一核双星」架构后，路线图由单链迭代升级为**四轨并行工程**：

| 阶段 | 时间 | 轨道 A：L2 总行与信贷基底 | 轨道 B：L3-A 发行专属链 | 轨道 C：L3-B 衍生专属链 | 轨道 D：Bybit & 跨层互联 |
|---|---|---|---|---|---|
| **P0 核验** | 0–6 周 | mStocks 现货金库审计；Lending 抵押模型确定 | L3-A 架构设计；Bonding Curve 状态机验证 | L3-B 架构设计；Oracle 预言机对接 | Intent Relay 机制 PoC；Bybit 四条工作流意向签字 |
| **P1 架构试车** | 4–16 周 | 统一保证金中心 (Unified Margin) MVP 部署 | **L3-A 测试网上线**；0 Gas Paymaster 跑通 | **L3-B 测试网上线**；**内嵌 RFS 流式做市上线** | W2 做市商大宗现货库存打通；Fast Relayer 上线 |
| **P2 发行点火 ∥ Perps 主网** | 12–26 周 | L2 金库速率熔断断路器实装上线 | **L3-A 主网上线** (首批 NVDAx/TSLAx Meme)；内嵌 AMM 就地开盘 | **L3-B 股票 Perps 主网 Beta** | **W1 Bybit Alpha 白标入口直通** |
| **P3 杠杆扩张** | 24–38 周 | **mStocks 质押 Lending 正式上线**；激活 5.76 亿资金 | 状态物理修剪上线；Fee Key NFT 二级交易市场 | 极速清算通道 (Liquidation Lane) 实装 | 毕业代币保送 Bybit Alpha 专区制度化 |
| **P4 资本市场成熟** | 36–52 周 | L2 跨层全自动扎差清算系统成熟 | Pre-IPO 资产代币化发行 | 24/7 跨休市连续清算常态化运行 | **W3 自营发股** + **W4 UTA 保证金打通** |

**分阶段验收 KPI**（全部沿用研究结论的度量纪律）：

| 阶段 | 验收指标 |
|---|---|
| P1 | 主流资产对 CEX 价差 ≤ 目标 bps；报价在线率；无助记词无 gas 端到端跑通；压测通过 |
| P2 | 毕业 ≥3/周；毕业后 7 天存活率 ≥30%；曲线日成交 ≥$200K；perps 基差收敛速度达标 |
| P3 | 财报周成交 ≥ 平日 3 倍；稳定币链上利用率显著抬升；熔断机制经历一次真实周末跳空并生效 |
| P4 | 单个财报/IPO 事件带来 ≥$5M 曲线成交；Fee Key 出现二级交易；UTA 内链上抵押规模 |

---

## 7.11 KPI 纪律与反目标

**永远不用的指标**：日发币量（four.meme：发币量 −14%，收入 −99%）、补贴买来的 TVL（Aave 教训：$137M→$704M→$62M，比激励前还低）。

**反目标清单（不做的事）**：
- ❌ 通用 meme 曲线（已死两次，第三次没有借口）
- ❌ 通用 crypto perps 正面刚 Hyperliquid
- ❌ 孤立割裂的通用 L3（拒绝资产碎片化；必须走内嵌 AMM 与远程保证金授信的双专有 L3 架构）
- ❌ 把所有高频交易硬塞在 L2 单链（放弃在 2 秒出块的公共链上打无谓的性能补丁）
- ❌ 用补贴堆 TVL / 交易量
- ❌ 承诺 Bybit 上币直通车（four.meme→Binance 是 4 次运营活动不是制度）

---

## 7.12 风险登记表（重定位视角）

| 风险 | 严重性 | 缓解 |
|---|---|---|
| Backed 不同意 xStocks 作 permissionless quote | **致命** | P0 提前谈判；备选：USDC quote + 预言机挂钩叙事，或 Quote Vault 收敛合规关系 |
| xStocks Multiplier 是 rebase 而非 ERC-8056 | **致命** | P0 核验；若是 rebase 先做 qToken 非 rebase 包装 |
| W1/W2 谈不下来（Alpha 逻辑是"不必上链"；Byreal 先例） | **高** | 说服角度＝"新增差异化产品线、无需教育用户上链"；缺席则降级为中档方案（自建 Terminal + Passport 分发），roadmap 主线不崩 |
| W3 自营发行刺激 Backed 关系 | 高 | 双轨叙事 + 商务排序（先锁 quote 授权，再立自营主体；开放问题 Q2） |
| 监管悬空（无任何具名监管机构就"AMM 交易股票代币"表态） | **高** | IPO 曲线与财报币必须法务前置；优先无结算、无财务权利的纯叙事变体；sequencer 合规过滤 + 标的冻结治理 |
| 上市公司公开反对（AMC 先例） | 高 | 白名单 + 冻结预案；严禁商标；不照抄 Robinhood 硬刚姿态 |
| 流量不足（最现实） | **高** | Alpha 白标反向导流；财报日历制造节奏；PBW + 金库收益补贴做市；毕业阈值压到 $12K |
| 休市脱锚 / 预言机陈旧 | 中高 | 曲线阶段零预言机；熔断 hook + staleness 主防线 + Fluxion RFQ 自动收敛 |
| Noxa 式工程失能（16 天崩塌先例） | 中高 | 抗 bot 洪水工程 > 机制创新；P1 压测限流先行 |
| 补贴退出崩塌 | 中 | 预算上限 + 阶梯退出，写进公开文档 |

---

## 7.13 开放问题（下一轮讨论）

1. **Q1 · perps 休市 mark price 政策**：休市时 index 冻结在收盘价，perp 围绕预期交易——资金费率如何设计才能既允许"周末定价"又不被操纵（clamp 参数、开盘收敛机制）？这是"休市定价是功能"叙事的技术兑现点，是 perps 支柱的命门。
2. **Q2 · W3 与 Backed 的商务排序**：先谈 Backed 的 quote 授权（P0 必需），还是先立自营主体（会惊动 Backed）？倾向前者，需推演。
3. **Q3 · 叙事发布节奏**：等 P1 有对 CEX 价差数据后再官宣（用数据说话），还是先官宣拉预期？鉴于两次 meme 失败的公开记录，倾向前者：先让 prop AMM 的价差数据存在，再讲故事。
4. **Q4 · 日历间隙期的常青玩法**：财报周之间的活跃度填充物（指数篮子长线叙事币、Fee Key 的 CTO 市场、跨标的 Ticker Wars 联赛制等）如何设计而不稀释"股票轴心"？

---

## 7.14 最后一句

**Mantle 的问题从来不是"怎么做一个 launchpad"，而是"怎么让已经聚集的钱动起来"。**

买方已经建成：银行（UR）、资管（MI4）、收益（mETH）、资产库（155 个 xStocks）、入口（Bybit 8,000 万用户）、以及 $576M 躺着不动的稳定币。

本部分给出的答案是把缺失的另一半建出来——**发行、做市、交易、融资**——并让每一层的现金流都回到主业资产。银行聚集了钱，资本市场让钱动起来。

**Where Assets Go Public.**
