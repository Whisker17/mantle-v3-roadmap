# 交易类 app-specific chain 横向对标
> 研究轨道：J ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

## 0. 概述与方法论

**本轨道要回答的问题**：交易类 app-specific chain 里，"链级原生模块"到底指什么、是否真的比"应用层/应用合约"更优、优在哪、代价是什么。统一模板（每个案例尽量对齐）：

1. **链级原生做了什么**——哪些功能被下沉到共识层/状态机层，而不是作为一个部署在通用 VM 上的智能合约；
2. **为什么必须在链级**——如果换成应用层合约会损失什么（性能、原子性、抗 MEV、gas 语义等），官方或研究者是否给出过论证；
3. **效果数据**——量、TVL、OI、收入、市场份额，标日期；
4. **代价与失败教训**——中心化倒退、审计面、生态可组合性损失、单点故障、迁移/弃用历史。

**范围声明**：本轨道只覆盖"交易类"（CLOB / perps / 撮合）app-specific chain。资产发行原生原语（tokenfactory、RWA module 的资产铸造侧）详见 → 交给 L 轨道（`research/L-issuance-native-primitives.md`）；为 perps 新增链原生支持的正向设计空间 → 交给 M 轨道；OP Stack / Mantle 的可改造面 → 交给 N 轨道。本文只做"外部对标"的事实拼图，不重复设计。

**证据密度声明**：Hyperliquid 是本文証据密度最高的案例（官方文档最完整、数据最公开），其余案例证据密度依次递减——这本身也是一个观察结果，写入 §6 结论。


## 1. Hyperliquid（最重要，最详细）

### 1.1 HyperCore：作为 L1 原生状态机

<!-- TODO -->

### 1.2 HIP-1 原生代币标准 + 荷兰式拍卖上币额度

<!-- TODO -->

### 1.3 HIP-2 原生自动做市（Hyperliquidity）

<!-- TODO -->

### 1.4 HIP-3 无许可永续市场部署

<!-- TODO -->

### 1.5 HyperEVM：dual-block 架构、CoreWriter 与 read precompiles

**HyperEVM 上线时间**：主网 **2025-02-18** 上线（官方公告 https://app.hyperliquid.xyz/announcement/wctu4xz6fze [一手]；报道 https://tokeninsight.com/en/news/hyperliquid-launches-hyperevm-on-mainnet-to-bring-general-purpose-programmability [二手]）。Chain ID = 999，RPC `https://rpc.hyperliquid.xyz/evm`。CoreWriter 合约 **2025-07-05** 上线（[二手] Oak Research https://oakresearch.io/en/reports/protocols/hyperliquid-hype-s1-2025-activity-report ；The Defiant https://thedefiant.io/news/defi/hype-rallies-ahead-of-corewriter-launch ）。

**Dual-block 架构参数**（[一手] https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/dual-block-architecture ，取数 2026-09-07）：

| 参数 | Small block（fast） | Big block（slow） |
|---|---|---|
| 出块间隔 | **1 秒** | **1 分钟** |
| Gas limit | **3M gas** | **30M gas** |
| 用途 | 低确认延迟的常规交易 | 大体积交易（复杂合约部署等） |

官方原文：*"The primary motivation behind the dual-block architecture is to **decouple block speed and block size** when allocating throughput improvements. Users want faster blocks for lower time to confirmation. Builders want larger blocks to include larger transactions such as more complex contract deployments."*

机制细节（原文）：两类区块共享**递增且唯一**的 EVM 区块号序列；HyperEVM "mempool" 本身是链上状态，拆成两个独立 mempool；每地址仅接受下 8 个 nonce；mempool 中超过 1 天的交易被清理。用户可提交 `{"type": "evmUserModify", "usingBigBlocks": true}` 把自己的交易定向到 big block。

**CoreWriter 合约与 read precompiles**（[一手] https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore ）：

- **Read precompiles**：地址从 `0x...0800` 起，可查询 perps 仓位、spot 余额、金库权益、质押委托、oracle 价格、L1 区块号；官方保证 *"The values are guaranteed to match the latest HyperCore state at the time the EVM block is constructed."* Gas 成本 `2000 + 65 * (input_len + output_len)`，输入非法则消耗全部 gas。
- **CoreWriter 写合约**：地址 `0x3333333333333333333333333333333333333333`，调用 `sendRawAction(data)`；`data = [version(1B)][actionId(3B)][ABI-encoded fields]`，共 16 种 action（限价单、金库转账、质押委托/存取、spot 转账、批准 builder fee 等）。**烧 ~25,000 gas 后发出一条 log，由 HyperCore 异步处理**，实际基础调用 gas 消耗约 47,000。
- **反延迟套利设计**（原文）：*"To prevent any potential latency advantages for using HyperEVM to bypass the L1 mempool, **order actions and vault transfers sent from CoreWriter are delayed onchain for a few seconds**."*
- **官方 GitHub**（`hyperliquid-dex/contracts`，https://github.com/hyperliquid-dex/contracts ）**只公开了桥合约 `Bridge2.sol`**；`CoreWriter.sol` / `L1Read.sol` 仅以文档附件形式发布，并未在官方仓库开源——这是"生产级样本"里一个**未开源的关键组件**，值得存疑标注。

**原子性边界（核心结论）**：同一 L1 块内执行顺序为 ①L1 块构建 → ②EVM 块构建 → ③EVM→Core 转账处理 → ④CoreWriter action 处理。
- **原子/同步**：一笔 EVM 交易内所有 precompile 读取对应同一份"EVM 块构建时刻"的 HyperCore 状态快照；EVM 交易本身按标准 EVM 语义原子执行。
- **非原子/异步**：CoreWriter 写操作是 fire-and-forget——EVM 交易只烧 gas、发 log，**HyperCore 在 EVM 块构建完成之后才处理该 action**，合约读不到写操作的执行结果，**Core 侧失败不会回滚 EVM 交易**；订单类与金库转账 action 还额外有秒级延迟；Core→EVM 方向转账排队到下一个 HyperEVM 块才处理；且 CoreWriter 调用发起账户**必须在 EVM 块构建前已存在于 HyperCore**，同块初始化会导致 action 被拒绝。
- **一句话**：读是同步一致的（同块快照），写是异步单向的（跨执行域消息、无返回值、不可回滚、订单类还有秒级延迟）。这是目前"原生模块如何暴露给通用 EVM"唯一的生产级样本，其代价是**合约开发者必须自行处理跨域竞态与静默失败**，不能假设 CoreWriter 调用等价于原子函数调用。
[一手] https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interaction-timings （取数 2026-09-07）

### 1.6 Gas / 手续费模型与 HYPE 价值捕获

**HyperEVM gas**：[一手] *"HYPE is the native gas token on the HyperEVM. … The HyperEVM uses the Cancun hardfork without blobs. EIP-1559 is enabled. Base fees are burned as usual … **Unlike most other EVM chains, priority fees are also burned** because the HyperEVM uses HyperBFT consensus."*（base fee + priority fee 全部销毁，验证人不拿小费）。

**HyperCore 交易不收 gas**：下单/撤单/转账等 L1 action 不逐笔收 gas，靠**地址级限速**防滥用——*"1 request per 1 USDC traded cumulatively since address inception … initial buffer of 10000 requests."* 例外：新账户激活费（首笔转入需 1 枚 quote token，如 1 USDC）；可选优先费（做市商可付费降低 gossip/下单延迟，以 HYPE 计价且全部销毁）。[一手] https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits 、.../priority-fees 、.../activation-gas-fee

**Maker/Taker 费率**（[一手] https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees ，2026-09-07，按 14 天滚动加权量分层）：

| 层级 | Perps Taker/Maker | Spot Taker/Maker |
|---|---|---|
| Tier 0（基准） | 0.045% / 0.015% | 0.070% / 0.040% |
| Tier 3（>$100M） | 0.030% / 0.004% | 0.040% / 0.010% |
| Tier 6（>$7B） | 0.024% / 0.000% | 0.025% / 0.000% |

质押 HYPE 可再打折（Wood>10 HYPE 5% → Diamond>500k HYPE **40%**）；HIP-3 growth mode 下 taker 费率可降到 0.0045%–0.009%（基准的 1/5–1/10），HIP-3 部署者可抽成 0–300%（自留最多 50%）。

**价值捕获路径（官方原文）**：*"On most other protocols, the team or insiders are the main beneficiaries of fees. **On Hyperliquid, fees are entirely directed to the community (HLP, the assistance fund, and deployers).**"* 四条销毁/回流通道：
1. **Assistance Fund（AF）自动回购并销毁**：系统地址 `0xfefe...fefe`，*"converts trading fees to HYPE in a fully automated manner as part of the L1 execution… HYPE in the assistance fund is burned, removing the tokens permanently from the circulating and total supply."* 约 **99% 手续费**流向 AF 回购（[二手] DefiLlama 方法论：2025-08-30 起 HLP 分成从 3%→1%）。
2. **AF 持仓正式确认为销毁**：Hyper Foundation 于 **2025-12-24** 经验证人投票（约 85% 质押权重赞成）将 AF 累计 ~3,700 万枚 HYPE 从流通/总量中永久移除。[二手] https://thedefiant.io/news/tokens/hyperliquid-proposes-burning-13-percent-of-circulating-token-supply
3. **HyperEVM base+priority fee 全额销毁**、**HyperCore 优先费/拍卖费销毁**（同上）、**HIP-1 拍卖所付 HYPE 销毁**。
4. **质押奖励来自 future emissions reserve**（非手续费），奖励率与 √(总质押量) 成反比，400M 总质押时年化 ≈2.37%；质押同时打交易费折扣，形成"持币→质押→降费→交易"飞轮。[一手]

**实时数据（2026-09-07，【实测】官方 info API 直读）**：AF 余额 **47,040,579 HYPE**，累计买入成本 $1.2806B（均价≈$27.2），按现价 $86.73 计市值≈$4.08B；质押约 **440.53M HYPE / 34 个验证人**。HYPE 现价 $86.73，流通市值 $19.29B，流通量 222.4M，总量 955.3M，ATH $89.60（2026-09-06，即取数前一日）。[一手 CoinGecko + 官方 API]

### 1.7 数据表现与近期事故

**协议手续费月度序列**（[二手] DefiLlama https://api.llama.fi/summary/fees/hyperliquid ，取数 2026-09-07，单位 $M；不完整月标注 *）：

| 月份 | 费用 | 月份 | 费用 | 月份 | 费用 |
|---|---|---|---|---|---|
| 2024-12* | 11.1 | 2025-08 | **145.0**（峰值） | 2026-04 | 58.7 |
| 2025-01 | 52.2 | 2025-09 | 115.3 | 2026-05 | 62.2 |
| 2025-02 | 46.6 | 2025-10 | 124.8 | 2026-06 | 81.0 |
| 2025-03 | 40.9 | 2025-11 | 101.4 | 2026-07 | 55.1 |
| 2025-04 | 44.2 | 2025-12 | 68.7 | 2026-08 | 67.3 |
| 2025-05 | 73.0 | 2026-01 | 77.3 | 2026-09* | 12.3（至9/4） |
| 2025-06 | 63.1 | 2026-02 | 70.7 | | |
| 2025-07 | 96.4 | 2026-03 | 69.5 | | |

**累计手续费 $1.5369B；近一年 $945.9M；近 30 天 $71.7M。** 单日峰值：2025-10-10 $10.74M（10·10 ADL 大清算日）、2025-08-14 $9.54M、2025-08-22 $7.97M。

**交易量与规模**：perps 24h 成交 **$3.737B**（【实测】官方 API `metaAndAssetCtxs` 加总，233 个永续市场，2026-09-07）；现货 24h $78.5M、30d $4.35B、累计 $165.1B（[二手] DefiLlama）。历史峰值：**2025-05 月度永续成交 $248B，约占同期 Binance 合约流量 10.5%**（[二手] The Block，2025-06-06，https://www.theblock.co/news/defi/2025-06-06-hyperliquid-hits-record-248-billion-perp-volume-in-may-capturing-over-10-of-binance-flow-356654 ）；2025-07 约 $320B（[二手] The Block，2025-08-05）。当前 OI ≈ **$11.05B**（【实测】官方 API，2026-09-07）。

**TVL / TVS / 市场份额**：DefiLlama 链 TVL 从 2025-03-01 $0.66B 涨至 2025-09-01 峰值区 $2.41B，回落至 2026-09-06 **$1.54B**（[二手]）；L2Beat 将 Hyperliquid 列为"App-chain（主桥在 Arbitrum）"，**TVS $6.70B**（2026-09-07，[二手] https://l2beat.com/scaling/projects/hyperliquid ），并标注风险：*"Critical contracts can be upgraded by an EOA which could result in the loss of all funds."*——**桥合约存在 EOA 可升级的中心化风险**。市场份额：2025 年多数时间占链上永续成交 60%+，但 **2025Q4 起 Aster / Lighter / edgeX 崛起侵蚀份额**，Aster 曾单日成交超越 Hyperliquid（[二手] BlockEden 2026-01-29、CoinMarketCap Academy）；精确月度份额序列需付费数据源，**未找到**免费权威序列，⚠️存疑。吞吐/延迟官方声明：约 **200k orders/sec**，端到端延迟中位数 0.2 秒、99 分位 0.9 秒。[一手]

**近期事故（案例级教训）**：

| 事件 | 日期 | 机制 | 损失/后果 | 教训 |
|---|---|---|---|---|
| **JELLY 事件** | 2025-03-26 | 攻击者对薄流动性 JELLY 开空单同时现货拉盘逼多方补仓失败，backstop 清算把仓位甩给 HLP；验证人投票下架合约并以非市价强制结算 | HLP 一度浮亏$10–12M（最终转为+$70万）；用户由 Hyper Foundation 补偿 | backstop 机制让 HLP 成为可被狙击的最终对手方；预言机可被薄现货操纵；验证人可投票覆盖结算价，损害去中心化叙事 |
| **3·12 巨鲸"自我清算"** | 2025-03-12 | 50x ETH 多单通过提取浮盈保证金主动触发清算，把滑点转嫁 HLP | HLP 亏≈$4M，交易者带走≈$1.8M | 清算可被当作退出流动性工具；促成引入 margin tiers、BTC/ETH 最大杠杆下调至 40x/25x |
| **XPL 逼空** | 2025-08-26/27 | hyperp（无外部现货预言机的预上线永续）自引用定价，鲸鱼 5 分钟内拉盘 200%+ | 空头连环清算，估计 $17M–$60M+ | 自引用 mark price 即 oracle 输入，是可拉的"内部预言机"；官方补丁：mark price 封顶 8h EMA 的 3 倍 |
| **10·10 大崩盘 + ADL 争议** | 2025-10-10/11 | 宏观消息触发全市场$19B清算，HL占$10.3B；触发跨保证金 ADL，约12分钟强平$2.1B盈利仓位 | 零坏账，但"赢家被课税"争议；学界测算 ADL 队列设计过度使用 | ADL 是偿付能力最后防线，但按盈利/杠杆排序会惩罚正确方向的对冲者 |
| **POPCAT 操纵** | 2025-11-12 | 分散钱包开杠杆多单+挂假买墙制造深度假象，随后砸盘引爆自身仓位，订单簿无力承接 | HLP 承接坏账≈$4.9M | JELLY 后的 OI 上限措施未根除"薄流动性资产+HLP最终对手方"结构性攻击面 |

来源：Halborn 安全复盘（JELLY: https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025 ；POPCAT: https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-november-2025 ）[二手]；The Defiant（JELLY 补偿 https://thedefiant.io/news/defi/hyperliquid-to-compensate-jellyjelly-traders-and-strengthen-risk-protocols ；3·12 https://thedefiant.io/news/defi/whale-s-nine-figure-eth-liquidation-costs-hyperliquid-usd4-million ）[二手]；hyperps 机制 [一手] https://hyperliquid.gitbook.io/hyperliquid-docs/trading/hyperps ；ADL 规则 [一手] https://hyperliquid.gitbook.io/hyperliquid-docs/trading/auto-deleveraging ；ADL 学术分析 [二手] arXiv 2512.01112 https://arxiv.org/html/2512.01112v2 ；Forbes "who pays for a crash" [二手] https://www.forbes.com/sites/boazsobrado/2026/08/03/never-punished-for-winning-cryptos-fight-over-who-pays-for-a-crash/ 。


## 2. dYdX v4

### 2.1 Cosmos 自建链的 app-specific 模块清单

<!-- TODO -->

### 2.2 订单簿在验证者内存中、零 gas 下单撤单

<!-- TODO -->

### 2.3 从 StarkEx 迁移的动因（原话）

<!-- TODO -->

### 2.4 数据表现与市场份额变化

<!-- TODO -->

## 3. Injective

### 3.1 exchange module（链级原生 CLOB + Frequent Batch Auction）与 oracle module

<!-- TODO -->

### 3.2 tokenfactory / permissions module / RWA module

<!-- TODO -->

### 3.3 原生 gas 补偿机制、MultiVM / EVM 兼容

<!-- TODO -->

### 3.4 效果数据 + 「链级原生撮合但生态没起来」归因

<!-- TODO -->

## 4. 通用栈做 perps 会遇到什么墙

### 4.1 Aevo（OP Stack rollup 做 perps）

<!-- TODO -->

### 4.2 Lighter（zk perp appchain）

<!-- TODO -->

### 4.3 Paradex（Starknet appchain）

<!-- TODO -->

### 4.4 Vertex / Edge

<!-- TODO -->

### 4.5 Aster / edgeX / GRVT（选择性覆盖）

<!-- TODO -->

### 4.6 小结：通用栈做 perps 的墙

<!-- TODO -->

## 5. 反面与横向

### 5.1 Robinhood Chain（交叉引用，不重复研究）

**已在本仓库详细覆盖，本节只做一句话交叉引用，不重复研究**（详见 `report/03-robinhood-chain.md` 与 `research/D-robinhood.md`，取数日期 2026-09-06）[一手/链上]。

一句话结论：Robinhood Chain 是"合规发行方为 RWA（代币化美股）自建的 app-specific chain"而非"交易类原生撮合"案例——它**没有链级原生 CLOB**，DEX 撮合全部走 Uniswap v4 / Pons 等应用层 AMM 合约（见 D 轨道 §1.9 协议拆分表），链级原生模块只体现在**排序规则（FCFS 无优先费拍卖）**与**sequencer 级合规过滤（ArbFilteredTransactionsManager 预编译）**两点上，这两点都不是"撮合"意义上的原生化，而是"排序公平性 + 合规钩子"的原生化。因此 Robinhood Chain 对本轨道（交易类横向对标）的直接参考价值有限，但对 §5.2 的"失败/代价清单"及委托方"是否要为发行做专属链"的判断仍有旁证价值：**它证明了"专属链 + 通用应用层合约"这种轻量组合本身也能成立**（不是所有 app-specific chain 都需要把撮合下沉到链级），→ 交给 L 轨道进一步讨论"资产发行原生"与"交易原生"两种 app-specific 诉求是否需要同一张答卷。


### 5.2 app-specific 但失败 / 退化的案例

<!-- TODO -->

## 6. 归纳章节

### 表 A：链级原生模块清单 × 各链具备情况

<!-- TODO -->

### 表 B：「必须链级」vs「应用层足够」判定

<!-- TODO -->

### 表 C：app-specific 的代价清单

<!-- TODO -->

### 结论：app-specific chain 的成立条件与可检验判据

<!-- TODO -->

### 显式回答：除 Hyperliquid 外，链级原生撮合有第二个成功案例吗？

<!-- TODO -->

## 存疑清单

<!-- TODO -->

## 关键来源清单

<!-- TODO -->

## 可复现方法附录

<!-- TODO -->
