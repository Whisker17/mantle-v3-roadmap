# 专题 G：mStocks / xStocks 侧的可行性约束与设计输入（主 agent 自采）

> 采集时间：2026-09-06。

---

## 1. 术语澄清（必须在报告开头写明）

用户说的 **"Mantle 的 mStocks"**，在公开信息中的官方对应物是 **xStocks on Mantle**：
- 发行方 **Backed Finance**，瑞士 DLT Act 下的**代币化追踪凭证（tracker certificate）**，1:1 由托管的底层证券全额抵押。
- 2026-04-10 通过 **Mantle × Bybit × Backed × Flowdesk** 合作上线 Mantle，首批 10 个标的：TSLAx / NVDAx / AAPLx / METAx / GOOGLx / MSTRx / HOODx / SPYx / QQQx / CRCLx；**Q2 末扩展到 155 个**。
- 来源：https://chainwire.org/2026/04/10/mantle-becomes-one-of-the-first-ethereum-l2s-to-bring-tokenized-equities-to-on-chain-liquidity-with-xstocks-and-bybit/ , https://nansen.ai/post/mantle-q2-2026-report

用户把它类比成 "Binance 的 bStocks"（Bybit + Mantle 版）。这个类比在**分发结构**上成立，但在**法律结构**上要注意与 Robinhood Stock Tokens 的差别：
| | **xStocks（Mantle/Bybit）** | **Robinhood Stock Tokens（RH Chain）** |
|---|---|---|
| 发行方 | Backed Finance（瑞士） | Robinhood Assets (Jersey) Limited |
| 法律形式 | DLT Act 下的 tracker certificate，1:1 全额抵押 | 代币化债务证券 / 经济敞口凭证 |
| 权利 | 无股东权利 | 无股东权利、无投票权、无分红 |
| 可用地区 | 依 Backed/交易所合规区 | 120+ 国，**排除美国** |
| 链上形态 | Solana 上是 SPL + Token Extensions；EVM 侧为 ERC-20 | ERC-20 |
| 链上定价 | Chainlink 等 + Fluxion Atomic RFQ | Chainlink 喂价 |
来源：见上 + https://docs.robinhood.com/chain/stock-tokens/ , https://robinhood.com/rhj/stocktokens/

---

## 2. **最关键的可行性约束：xStocks 的可组合性不是无限的**

- xStocks 在 Solana 上用 **Token Extensions**，具备**可编程合规**能力：transfer hook、暂停控制、metadata pointer 等，**写在合约层**。
- 「permissionless」仅指**链上可在钱包间转移**；**获取/持有**通常经前端 KYC 闸门（瑞士等法域要求）。
- **不能说"可以进任何 DeFi 池"**，受两点约束：
  1. **协议兼容性**：目标协议必须支持该代币标准并主动允许该资产；若池子自身有白名单/合规要求与发行方的转账限制冲突，则不兼容。
  2. **转账限制**：若代币启用了要求收发双方均 KYC 的 transfer hook，则只能在能满足该条件的池子中流通。
- 开发者应先**核验具体 mint/合约配置**（是否启用 TransferHook / TransferFee），并确认发行方的集成指引。
来源：https://solana.com/news/case-study-xstocks , https://blockeden.xyz/blog/2025/09/03/xstocks-on-solana-a-developer-s-field-guide-to-tokenized-equities/ , https://support.kraken.com/articles/xstocks-faq

> ### ⚠️ 这条直接决定 Mantle 版 "PAIR" 能否成立
> Robinhood 的 Stock Token 之所以能被 pair.fund 当 quote 资产随便建池，是因为 **Robinhood 自己发的代币在自己的链上采用了较开放的转账策略**。
> Mantle 上的 xStocks 是 **Backed 发行的第三方合规资产**，Mantle 无权单方面放开其转账限制。
> **→ 因此 Mantle 版设计不能简单照抄 "meme × 股票代币直接组 LP"，必须做一层"包装/隔离"。**（详见报告设计章第 3 方案）
> **[以上为基于两方来源的推理判断，需与 Backed/Mantle BD 确认具体 mint 配置]**

---

## 3. Fluxion 的 Atomic RFQ：Mantle 已有的、别人没有的底牌

**Fluxion** = Mantle 原生全栈 DEX，2025-12-18 主网上线，为 RWA 提供机构级流动性与执行。
三个模块：
1. **AMM V2 池** —— 稳定/波动资产的路由与兑换
2. **AMM V3 集中流动性** —— 提高资本效率、降低滑点
3. **Atomic RFQ** —— 机构级执行层，允许用户**按实时市场报价、直接通过发行方（xStocks）铸造/赎回**代币化资产

**最关键的机制细节（对设计影响巨大）：**
- **开市时段**：Atomic RFQ 以底层证券的**实时市场价**为锚，提供机构级、**近乎无滑点**的执行。
- **休市时段**：协议**切换到 AMM 执行层**，维持 24/7 连续流动性。

来源：https://chainwire.org/2025/12/18/fluxion-mainnet-goes-live-on-mantle-advancing-native-spot-liquidity-for-defi-and-rwas/ , https://www.prnewswire.com/in/news-releases/mantle-bybit-and-fluxion-bring-xstocks-tokenized-equities-to-institutional-standard-with-atomic-rfq-302765661.html , https://www.hackquest.io/zh-cn/projects/Fluxion

> **战略含义**：Robinhood Chain 上的 PAIR 只是"把股票代币当 quote 资产"，本质是**被动的**。
> Mantle 已经有 **开市锚定 RFQ / 休市切 AMM 的双模执行引擎**，这是解决"周末跳空 + LP 逆向选择"这个 RWA-meme 结合最难问题的**现成基础设施**。
> 这是 Mantle 相对 RH Chain 的**真实技术差异化点**，报告的设计方案必须建立在它之上。

激励结构：xPoints（xStocks）+ Fluxion Points（DEX）双积分。
来源：https://nansen.ai/post/mantle-q2-2026-report

---

## 4. Mantle 已握有的稀缺叙事资产：代币化 IPO / Pre-IPO

- **SPCXx（SpaceX）**：2026-06-12 Mantle 宣布上线，借"史上最大 IPO"的窗口。
  - **Fluxion** 用 xStocks Atomic RFQ 提供直接铸造/赎回；
  - **Merchant Moe** 作流动性中心，推出 "Project X" 激励，为 SPCXx/USDT 池投放 **100,000 MNT** 奖励。
  - **市场反馈是负面的**：大量散户 pre-IPO 认购的实际获配远低于预期（有人只拿到约 $600 等值），部分平台因未能拿到足够底层股份而全额退款。
  来源：https://il.tradingview.com/news/chainwire:e1a81e8c8094b:0-mantle-and-xstocks-bring-tokenized-spacex-spcxx-to-fluxion-merchant-moe-as-history-s-largest-ipo-goes-live/
- Mantle 在一个月内上了**第三个代币化股票新股**（"The newest IPOs are landing on Mantle"）。
  来源：https://www.tradingview.com/news/chainwire:3f6f50ff4094b:0-the-newest-ipos-are-landing-on-mantle-bending-spoons-bspx-as-the-network-s-third-tokenized-equity-in-under-a-month/
  - ⚠️ 注意：关于 Bending Spoons，检索到的另一说法是其于 2026-07-01 以 **BSP** 在 Nasdaq 传统 IPO，且"不存在 BSPX pre-IPO 代币"。**两条信息冲突，标记为未证实，写报告时需谨慎表述。**

> **战略含义**：SPCXx 的"认购不足 + 退款"恰恰证明了**需求远超供给**。
> 这正是 bonding curve 最擅长解决的问题：**当一个资产的一级供给受限而需求无限时，用曲线做连续、无配额的价格发现**。
> → 报告设计章可提出"**IPO 情绪币 / pre-IPO 影子市场**"这一 Mantle 独占玩法。

---

## 5. RWA × meme 结合的四个结构性难题（复述自 F 篇，此处给设计约束）

1. **周末/休市跳空**：代币化股票周末成交量比工作日低 **85–92%**，价差走阔；LP 承担逆向选择。反例：2026-06 有代币化股票在周末独立定价出 **6.5%** 的跳空并被周一开盘验证。
2. **预言机陈旧**：休市时喂价停更 → 池子被套利，需要熔断或二级定价。
3. **流动性碎片化**：2026-06 代币化股票链上月成交额约 **$9.22B**，分散在 Base / Solana / Arbitrum 等。
4. **冷启动陷阱**：缺乏成熟链上借贷市场 → 缺自然借贷需求 → 收益靠补贴。
来源：见 F 篇第 6 节

---

## 6. Robinhood 侧的"股票 × meme"实际数据与舆情（对标基准）

- **交易量**：2026-09-02，meme × 股票代币交易对成交额 >**$217M**，一度**超过底层股票代币自身的成交额**。
  来源：https://www.weex.com/news/detail/217-million-meme-coin-stock-token-pair-trading-volume-increases-niao8un0830lzdib62lm9tkq
- **代表资产**：
  - **NVDA 代币** 成为主力 base pair（例：meme "Artificial Inu"）；
  - **AMC 代币** 成为争议焦点：AMC CEO **Adam Aron** 公开抨击其为 "pseudo-fake market"，并威胁向 SEC 投诉；
  - **CASHCAT**：该链早期爆款 meme。
  来源：https://www.investing.com/news/stock-market-news/amc-ceo-threatens-sec-complaint-over-robinhood-tokens-4889693 , https://dmarketforces.com/nvidia-tokenized-stock-gains-3-on-speculative-activity/
- **Robinhood 官方态度**：CEO 公开表态 "We stand behind stock tokens"，力挺这个（当时报道称）**$104M 规模的生态**，尽管有 backlash。
  来源：https://www.tradingview.com/news/u_today:290c23201094b:0-we-stand-behind-stock-tokens-robinhood-ceo-backs-104-million-ecosystem-amid-backlash/
- **叙事机制**：用户把股票代币当作 "narrative collateral"（叙事抵押品），复刻 GameStop 式逼空文化。
  来源：https://www.mk.co.kr/en/stock/12144219

> **对 Mantle 的三条可迁移经验**：
> 1. "股票代币当 quote" 是**真实需求**，不是噱头 —— 成交额能超过底层资产本身。
> 2. **上市公司会反弹**。Mantle/Bybit 作为有牌照野心的机构，需要**预设一条"标的白名单 + 争议标的下架"的治理路径**，而不是等 CEO 发推。
> 3. **发行方立场是护城河也是风险**：RH 敢做是因为自己发的代币；Mantle 用的是 Backed 的资产，必须先谈好合作边界。

---

## 7. Bybit Alpha：Mantle 版飞轮的分发入口（已存在但未被用于 meme）

- Bybit 已将 "Bybit Web3" 演进为 **Bybit Alpha**：账户制链上交易，直接嵌入**统一交易账户 UTA**，**无需助记词/私钥/gas token**，以 USDT / USDC / SOL / bbSOL 结算。
- **Mantle Chain 于 2026-03 正式接入 Bybit Alpha**。
- 现有通道：Launchpool / Launchpad / MegaDrop；MNT 持有者享 VIP 倍率。
来源：https://www.bybit.com/en/learn/bybit-guide/what-is-bybit-alpha , https://www.binance.com/en/square/post/300073721971937 , https://www.bybit.com/en/learn/bybit-mantle-mnt

> **结论**：Mantle 复刻 BSC 飞轮所需的分发端**已经就位**（Bybit Alpha = Binance Alpha 对位物）。
> 缺的是**资产供给侧**：Bybit Alpha 里没有源源不断的 Mantle 原生新资产可供交易。
> meme launchpad 正是那个"资产供给引擎"。
