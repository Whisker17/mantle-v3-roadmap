# 专题 F/G：链 Infra 需求 + 曲线/流动性机制设计（主 agent 自采）

> 采集时间：2026-09-06。所有条目附来源。标 **[未证实]** 的为单一来源或推断。

---

## 1. Solana 侧：为什么"热点隔离"是 meme 的命脉

### 1.1 Local Fee Market（LFM）的真实机制
- Solana **没有单一全局费用拍卖**；用 **local fee market + prioritization fee** 在验证者队列里排序。
  来源：https://www.helius.dev/blog/solana-local-fee-markets , https://strongholdsol.com/solanas-local-fee-markets/
- **争用定义在"写锁"粒度**：多笔交易同时要写同一个 account/program 才构成 contention；不触碰该状态的交易**完全不受影响**。
  来源：https://fraxcesco.substack.com/p/in-depth-on-the-tech-solanas-local
- 优先费公式：`Prioritization Fee = Compute Unit Limit × Compute Unit Price`（lamports/CU）。
  故"虚高 CU limit"会自我惩罚 → 天然抑制垃圾交易膨胀。
  来源：https://solana.com/docs/core/fees/fee-structure
- **Central Scheduler**（Agave v1.18 起）：构建交易依赖图，按优先费确定性排序，替代此前的随机线程分配（曾造成 jitter 与低效上链）。
  来源：https://www.helius.dev/blog/solana-local-fee-markets
- **swQoS（Stake-Weighted QoS）**：高质押验证者向当前 leader 发送交易时被优先，用于抗垃圾流量。这就是 GMGN/Photon 这类终端要买"staked connection"的原因。
  来源：https://solana.stackexchange.com/questions/17086/

### 1.2 弹性区块空间的真实边界（关键！）
- **SIMD-0286 于 2026-07-29 激活**：区块上限 60M CU → **100M CU**（+66%），**未改动 400ms 出块**。
  来源：https://solana.com/upgrades/100m-cu-blocks , https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0286-raise-block-limits-to-100M.md
- 升级前数据：约 **11.2% 的区块已触顶**（定义为使用 ≥56M CU）。
  来源：同上
- **最关键的一条**：升级**没有提高单账户写上限**，任一 writable account 在单区块内仍最多消耗 **12M CU**。
  → 也就是说 Solana 明确地把"单个热点合约不能吃掉整个区块"写进了协议层。
  来源：https://solanacompass.com/news/solana-raises-mainnet-block-compute-limit-66-to-100m-cus-with-simd-0286-at
- Block accounts data size delta 上限仍为 **100MB**。
- 历史路径：SIMD-0256（2025-07）50M→60M；SIMD-0286（2026-07）60M→100M。

> **对本报告的推论**：所谓"弹性可调节区块空间"在 Solana 上的落地形态不是"按需扩张"，而是
> ①**总量分级抬升**（治理驱动的 SIMD） + ②**单账户写配额硬顶（12M CU）** + ③**按写锁定价的 LFM**。
> 三者缺一不可：只抬总量而不设单账户顶，热点合约会重新吃满全区块；只设顶而无 LFM，热点合约的用户会被无差别挤出。

---

## 2. EVM 侧：局部费用市场到底能不能做？

### 2.1 现状（2026-09）
- **Arbitrum ArbOS "Dia"（2026-01）**：从单一 EIP-1559 base fee 改为 **多 gas target + 多调整窗口**，最终费用由 **6 个 (target, window) 对**聚合得出。设计目标是当"减震器"——短窗口吸收突发需求，长窗口锚定长期稳定，降低 base fee 的"跳变"。
  来源：https://blog.arbitrum.io/dynamic-pricing-update-2026/
  → 这是**时间维度**的平滑，**不是**状态维度的隔离。热点合约仍然会抬高全链 base fee。
- **EIP-8011（2025 年中）**：多维 gas **计量**（metering），按维度在区块层面独立计量，但交易仍付单一费用以保 UX。
  来源：https://eips.ethereum.org/EIPS/eip-8011
- **EIP-7999（2026 主推）**：统一多维费用市场，对 state growth / calldata / execution 分别定价，对用户仍暴露单一 `max_fee`。
  技术难点：**gas introspection** —— 合约用 `CALL` 的 gas 参数转发固定 gas，该参数是标量，无法表达多维；EIP-7999 提出 "Universal overflow" 模式保持兼容。
  来源：https://ethresear.ch/t/gas-overflow-for-multidimensional-fee-markets/24766 , https://www.tradingview.com/news/cointelegraph:214f46a3f094b:0-ethereum-proposes-unified-fee-market-to-simplify-transaction-costs/
- **理论障碍**：多维约束下的区块打包是 **NP-hard（多维背包）**，实践上只能降维 + 启发式。
  来源：https://www.eclipselabs.io/blogs/multidimensional-fee-markets-at-1m-tps
- **per-contract base fee 的固有难题**：EVM 合约互相调用（A call B），给不同合约不同 base fee 会产生循环定价问题与"绕道更便宜合约"的套利面。
  来源：https://arxiv.org/html/2509.17126v1（ZK-rollup 费用错配研究）+ 综述
- **2026 行业实际解法**：不是链内 per-contract 隔离，而是 **架构性模块化**——高需求应用直接开自己的 rollup/appchain 拿到隔离的费用环境。
  来源：https://crypto-economy.com/execution-and-data-availability-the-modular-stack-reshaping-web3-in-2026/

### 2.2 结论（给 Mantle 的可执行判断）
> **"完全的 Solana 式 LFM 在通用 EVM 上短期不可实现"**，但可以做**四层近似**，实现度从高到低：
> 1. **应用层配额**（完全可做，无需改链）：launchpad 合约内置 per-token/per-pool 的区块级配额与自有优先级队列。
> 2. **Sequencer 层策略**（改 sequencer，不改 EVM）：对交易做 access-list 预测 → 按"目标合约"分桶排队 + 每桶 per-block gas 配额；桶内 PGA、桶间公平轮转。**这是最高性价比的一层**。
> 3. **多目标/多窗口 base fee**（Arbitrum Dia 已生产化）：只解决时间平滑，不解决状态隔离。
> 4. **协议级 per-contract base fee**：需要改 EVM gas 计量，兼容性代价大，2026 年无生产实现。

---

## 3. Meme 交易对链的真实需求清单（扩展用户给的 4 点）

用户给的 4 点：瞬时执行容量 / 热点争用隔离 / 发行与流动性组合 / 机器化交易生态。
下面是我基于证据补的**另外 8 点**：

| # | 需求 | 为什么 | 证据 |
|---|---|---|---|
| 5 | **确认延迟 < 1s（软确认即可）** | meme 是 PvP，反馈环长度直接决定用户留存 | Base Flashblocks 2025-07 主网上线，把有效确认从 2s 压到 **200ms**；开发者用 `eth_subscribe(newFlashblockTransactions)` 或 `pending` tag 订阅。来源：https://docs.chainstack.com/docs/flashblocks-on-base , https://chainstack.com/flashblocks-base-rpc/ |
| 6 | **失败交易成本必须足够低** | EVM 中失败交易照样烧 gas；狙击战里失败率极高 | 失败 tx 消耗执行到 revert 点的 gas，是狙击者的"经营成本"；高 gas 链会直接杀死机器人生态 |
| 7 | **私有/半私有 mempool + 抗夹** | 公共 mempool = 三明治屠宰场 | Arbitrum Timeboost 设计中 **mempool 保持私有**以保护用户免受 sandwich；来源：Timeboost 机制说明 |
| 8 | **可预测的排序规则** | FCFS 奖励低延迟基础设施（colocation），PGA 奖励高出价；两者对 bot 生态的形态影响完全不同 | Arbitrum 2026-08 有 AIP 提议**停用 Timeboost express lane 拍卖、改用 PGA**；且 express lane 高度中心化（3 个实体赢下 ~99.7% 拍卖） |
| 9 | **gas token 不应是波动性投机标的** | 若 gas token 本身是投机资产，费用与叙事共振，放大波动 | RH Chain 用 **ETH** 做 gas；Mantle 用 **MNT** |
| 10 | **原生分发入口（钱包/App）** | 冷启动流量来源 | RH Chain 由 Robinhood Wallet 原生支持并**补贴 gas 至 2026-09-29**；Base 由 Base App/Coinbase 分发；BSC 由 Binance Web3 Wallet 分发 |
| 11 | **数据/索引层实时性** | 终端、扫链器、K 线、持仓分析全靠它 | GMGN 依赖自建节点 + gRPC 流；DexScreener/DEXTools 的链覆盖直接决定新链能否被"看见" |
| 12 | **拥堵时费用不外溢到主业务** | RH Chain 的反面教材 | 2026-09 初 RH Chain 因 meme 活动使平均 tx 费涨到 **~$0.40**，base fee 11 天涨 **82 倍**；Yakovenko 称其费用模型 "brain-dead"，Offchain Labs 的 Goldfeder 回击称"荒谬"。来源：https://www.cryptotimes.io/2026/09/05/solanas-yakovenko-calls-robinhood-chain-fee-model-brain-dead-as-gas-hits-0-40/ , https://thedefiant.io/news/blockchains/robinhood-chain-gas-fees-jump-82-fold-in-11-days-to-top-every-other-chain |

### 3.1 那场"landlord vs tenant"辩论的实质（对 Mantle 极其重要）
- **Yakovenko**：链靠拥堵赚钱是脑残设计；前端应用（券商）应当直接、透明地向用户收费，而不是从底层网络拥堵里抽租。他算过：RH 分给 Arbitrum 的那 **10% 净协议收入**（8% DAO + 2% 开发者基金），已足够覆盖同等交易量在 Solana 上的全部成本 → RH 本可以做"完全 gasless"。
- **Goldfeder（Offchain Labs）**：Orbit 模型让应用方从"租户"变"房东"，保留约 **90% gas 收入**；如果建在 Solana 上，费用直接给验证者，券商拿不到任何基础设施收入。
- 来源：https://www.tradingview.com/news/cryptobriefing:715a97a93094b:0-offchain-labs-and-solana-co-founders-spar-over-whether-robinhood-chain-s-fee-model-is-feature-or-bug/ , https://www.weex.com/news/detail/the-debate-on-blockchain-business-models-choosing-to-be-a-tenant-or-a-landlord-nqt9ppj55ffhj6lxdw6xrzm8

> **对 Mantle 的启示**：Mantle 同样是"房东"（自有 sequencer + MNT gas）。但 Mantle 的主业务是 RWA/机构结算，**绝不能让 meme 拥堵把 xStocks 的结算成本抬起来**。这直接推出"必须做热点隔离"的结论，而不是可选项。

---

## 4. 并行 EVM 的对照（Monad / MegaETH）

- **Monad（L1）**：乐观并行执行。假设交易独立，热点合约冲突时检测 → 回滚 → 按正确顺序重执行。适合"冲突存在但不持续"的高吞吐场景。
  来源：https://docs.monad.xyz/monad-arch/execution/parallel-execution
- **MegaETH（L2）**："Streaming EVM" + 单一高性能 sequencer，目标 **sub-10ms 出块**，用集中排序绕开分布式共识的争用开销。
  来源：https://www.gate.com/learn/articles/interpretation-of-megaeth-whitepaper/3453
- **关键洞察**：乐观并行对 **meme launchpad 这种"所有人写同一个 bonding curve 账户"的负载几乎无效**——冲突率接近 100%，回滚重执行反而是纯开销。Solana 的做法（写锁 + LFM + 单账户配额）才是对症的。
  → 这条对"Mantle 要不要上并行 EVM"是一个**反直觉但重要**的判断。**[部分为推理，非直接引用]**

---

## 5. Bonding Curve 机制设计

### 5.1 曲线族与取舍
| 曲线 | 公式 | 特性 | 适用 |
|---|---|---|---|
| 线性 | P(s) = a·s + b | 价格可预测、透明 | 公平分发、DAO 募资 |
| 指数 | P(s) = a·(1+r)^s | 极度奖励早期 | 社区冷启动、NFT |
| 恒定乘积 | x·y = k | 可持续双向交易 | **launchpad 主流** |
| 对数 / Sigmoid | — | 前快后平 / S 型 | 治理、声誉代币 |
来源：综合检索（bonding curve types 概览）

### 5.2 虚拟储备（virtual reserves）为什么必须存在
恒定乘积曲线若从 0 真实储备起步：
- **下渐近线问题**：初始价格≈0，早期买家可以近乎免费抽干代币；
- **上渐近线问题**：供应被买完时价格趋于无穷，代币永远卖不完 → 无法"毕业"。
**虚拟储备 = 初始化时注入的"假流动性"**，用于①定义一个可用的起始价格 ②保证曲线能被完整买完从而触发毕业。
来源：综合检索 + pump.fun docs（https://pump.fun/docs/bonding-curve）

### 5.3 bonding curve 相对"直接开 LP"的六个真实好处
1. **消灭冷启动**：合约本身就是做市商，创建者无需预置任何 quote 资产。
2. **零预售/零团队预留的可信承诺**：所有人在同一条曲线上，规则对称。
3. **价格发现被压缩进一个确定性的函数**：便于前端展示"进度条"，制造可视化的 FOMO（这是产品层面的核心，不是金融层面）。
4. **单边流动性**：项目方不需要出 quote 资产，protocol 用虚拟储备补足。
5. **可编程的费用与反狙击窗口**：曲线阶段是"受控环境"，可以塞进衰减税、限购、冷却期。
6. **毕业是可编程的确定性事件**：达到阈值即原子迁移 + 锁池，把最大的 rug 风险（撤池）从"信任"变成"代码"。

### 5.4 毕业后的流动性锁定：三种模式
| 模式 | 机制 | 优点 | 缺点 |
|---|---|---|---|
| **LP Burn** | LP token 打到 `0x…dead` | 最强信任信号，绝对不可撤 | 费用收益被孤儿化；未来无法迁移到新 DEX |
| **定时 Lock** | LP token 存进第三方锁仓合约（Unicrypt/TeamFinance） | 保留未来迁移能力 | 投资者需盯着到期日；到期即风险 |
| **v4 Hook 永久锁 + 费用钥匙** | 本金永久锁定，同时铸造 **Fee Key NFT**，持有者可持续领取该锁定头寸产生的交易费（类似 Raydium "Burn & Earn"） | 既有 burn 的信任，又保留现金流；可把 creator 激励做成永续 | 依赖 hook 合约安全性 |
来源：综合检索（LP burn vs lock；Uniswap v4 hooks 能力）

> **给 Mantle 的设计取向**：应当选第三种。理由：本报告后文的 mStocks launchpad 需要"creator 长期分成 + 永不撤池"同时成立。

### 5.5 Uniswap v4 Hooks 给 launchpad 的四种能力
1. **动态费率**：按波动率/成交量实时调整（可做"上线初期高费→衰减"的反狙击税）。
2. **自定义曲线**：`beforeSwap` 可完全接管定价，等于把 bonding curve 直接搬进 pool，**免去"迁移"这一步**（同一 pool 内做曲线段 → AMM 段的相变）。
3. **流动性行为约束**：`beforeAddLiquidity`/`beforeRemoveLiquidity` 可强制永久锁定、阻断 JIT 流动性攻击。
4. **`afterSwap` 现金流路由**：把每笔 swap 的费用实时分流给 creator / 协议 / 回购 / bid wall。
来源：综合检索（Uniswap v4 hooks 综述）

### 5.6 反狙击税（decaying sniper tax）的两个生产实例
- **Zora**：新币 sniper tax 从 **99% 起，10 秒内线性衰减到 1%**。
  来源：https://docs.zora.co/coins/contracts/rewards
- **Pons V2（RH Chain）**：anti-snipe tax 从 **~99% 起，几秒内衰减到 0**。
  来源：Pons V2 机制检索
> 设计要点：衰减税把"抢跑收益"在时间上抹平，让机器人抢到也无利可图，而不是试图（不可能地）阻止机器人。

---

## 6. 代币化股票做 quote 资产：必须解决的四个结构性问题

### 6.1 周末/休市流动性缺口
- 代币化股票 24/7 交易，底层股票 24/5。周末成交量相较工作日 **下降 85–92%**，价差显著走阔。
- **LP 的逆向选择风险**：周末出宏观新闻 → 链上价格剧烈反应 → 周一开盘"跳空"，周末提供流动性的 LP 被套利。
- 反向证据：2026-06 有代币化股票在周末**独立定价出 6.5% 的跳空**，周一开盘被传统市场验证。
来源：综合检索（tokenized stock weekend gap 2026）

### 6.2 预言机
- 主流做法是接入实时价格流（如 **Chainlink Data Streams**）锚定 AMM 定价，而非纯靠池内比价（oracle-based AMM / UAMM）。
- **Oracle Lag 风险**：休市期间若喂价停更或变陈旧，池子会被套利，需要二级定价机制或熔断。
来源：综合检索

### 6.3 流动性碎片化
- 2026-06 代币化股票链上月成交额约 **$9.22B**，但分散在 Base / Solana / Arbitrum 等多条链。
来源：综合检索

### 6.4 冷启动陷阱
- 链上股票市场缺乏成熟的借贷市场 → 缺乏自然借贷需求 → 收益主要来自激励而非真实信贷需求。
来源：综合检索

> **推论（给 mStocks launchpad）**：
> - **不要**让新发的 meme 币与股票代币组成"双波动 AMM 池"（LP 承担 meme 波动 × 股票跳空双重风险）。
> - 更稳的结构：**quote 资产用股票代币，但曲线阶段的定价用"股票数量"计价而非美元计价**；或者引入 **oracle-aware 曲线**，把股票代币的美元价格作为外生输入，让曲线只对"meme 的相对价值"定价。
> - 周末应有**明确的降档模式**（提高费率 / 收窄可交易额度 / 切换到纯 oracle 报价），而不是假装 24/7 无差别。

---

## 7. 排序与狙击：EVM L2 的具体动力学

- L2 上 sequencer 是"交通指挥"，多数为中心化，掌握**排序、纳入、审查**三权。
- **FCFS 模型**：赢家是**物理延迟最低**的（与 sequencer 端点 colocation）。
- **优先费模型**：赢家是**出价最高**的，发币瞬间 gas 价格飙升。
- 成熟 bot 用 `eth_call` / state override **预模拟**，避免必然失败的交易上链以省 gas。
- 项目方侧的反狙击手段：单钱包限购、地址冷却、延迟开盘区块、白名单。
来源：综合检索（memecoin sniping on L2 dynamics）

- **Arbitrum Timeboost**：60 秒一轮的**密封出价二价拍卖**，出价在轮次开始前 15 秒截止；赢家获得 express lane，可绕过施加于普通交易的 **200ms 人工延迟**。它只给**时间优势**，不给重排序权，mempool 保持私有。
  - 收入表现：上线前 3 个月约 **$3M** 手续费；2026-01→04 拍卖竞争度下降，DAO 收入持续走低。2026 上半年 Arbitrum DAO 总收入约 **$6.19M**（含 Timeboost + tx fee + AEP）。
  - 问题：express lane 高度中心化（**3 个实体赢下 ~99.7% 拍卖**），且**没有有效消除垃圾交易**。
  - 2026-08 有 AIP 提议在 Arbitrum One/Nova **停用 Timeboost，改用 PGA（优先 gas 拍卖）**。
  来源：Timeboost 机制与治理检索

> **给 Mantle 的启示**：不要照抄 Timeboost。它的经验教训是：①拍卖式 express lane 会中心化；②它不解决垃圾交易；③它把 MEV 收入货币化但代价是延迟惩罚普通用户。更适合 Mantle 的是"**按合约分桶的配额式排序 + 应用层反狙击税**"。

---

## 8. 市场格局速查（2026-09-06）

| 平台 | 链 | 关键数据 |
|---|---|---|
| **Pons** | Robinhood Chain | 2026-09-01 单日费用峰值 **$5.95M**；常占该链 50–80% 活动；自 8-29 起多次单日费用超过 pump.fun；PONS 在 9-04 创 ATH ~**$0.75** |
| **pump.fun** | Solana | 2026 年中成为**首个累计收入破 $1B 的 Solana 应用**；2026-08 周协议费 >$10M；2026-09 初日收入回落到低个位数百万 |
| **four.meme** | BNB Chain | 截至 2026-08 下旬，30 天费用收入**同比下滑 41%** |
| meme×股票代币交易对 | RH Chain | 2026-09-02 成交额 >**$217M**，一度超过底层股票代币自身的成交额 |
| RH Chain | — | 2026-09-02 前后单日费用收入超过 Solana；单日应用收入曾达 **$2.66M**，翻过 Ethereum 与 Hyperliquid |
来源：https://www.kucoin.com/news/flash/pons-surpasses-robinhood-chain-in-daily-fees-with-5-95m-record , https://memefees.com/launchpads/four-meme , https://www.weex.com/news/detail/217-million-meme-coin-stock-token-pair-trading-volume-increases-niao8un0830lzdib62lm9tkq , https://coingape.com/robinhood-chain-hits-2-66m-in-24h-app-revenue-flipping-ethereum-and-hyperliquid/

### 8.1 RH Chain 生态里的 launchpad 名单
Pons（龙头）、**Pools.trade（Uniswap Labs 自营，2026-08 上线）**、NOXA/NOXA Fun（早期，已衰落）、PAIR（多池 RWA launchpad）、Flap、hood.fun、Bankr、Virtuals、Clanker。
来源：https://www.bitrue.com/blog/best-robinhood-launchpads-2026 , https://blog.uniswap.org/pools-trade-a-new-way-to-launch-on-robinhood-chain

### 8.2 Uniswap Labs × Pons
- 2026-09-04 Pons 团队宣布 **Uniswap Labs 购入 PONS 代币份额**（"购入 stake"，非收购），条款未披露，目的为"长期对齐"。
- 微妙之处：Uniswap Labs **自己在 2026-08 就在 RH Chain 上线了竞品 Pools.trade**。
来源：https://thedefiant.io/news/defi/uniswap-labs-bought-pons-token-for-long-term-alignment , https://cryptopotato.com/uniswap-buys-pons-stake-token-jumps-40-to-new-all-time-high/

### 8.3 分发侧：Bybit Alpha（Mantle 的最大隐藏资产）
- Bybit 已把 "Bybit Web3" 升级为 **Bybit Alpha**：账户制链上交易，直接集成进 **统一交易账户 UTA**，**无需助记词/私钥/gas token**，用 USDT/USDC/SOL/bbSOL 结算。
- **Mantle Chain 于 2026-03 正式接入 Bybit Alpha**，可无跨链操作直接交易 Mantle 原生资产。
- Bybit 现有 Launchpool / Launchpad / MegaDrop 通道，MNT 持有者享 VIP 倍率等优待。
来源：https://www.bybit.com/en/learn/bybit-guide/what-is-bybit-alpha , https://www.binance.com/en/square/post/300073721971937 , https://www.bybit.com/en/learn/bybit-mantle-mnt

> **这是本报告最关键的杠杆点之一**：Bybit Alpha ≈ Binance Alpha 的对位物，且 Mantle 已经接进去了。
> 缺的不是通道，是**通道里没有值得交易的原生资产供给**。
