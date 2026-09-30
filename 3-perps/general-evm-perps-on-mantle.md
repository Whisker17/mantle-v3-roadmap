# General EVM 上落地 Perps：Mantle-as-is 决策研报

> **研究归属**：Mantle v3 Roadmap · 支柱 3  
> **取数日期**：2026-09-15  
> **定位**：决策级。读者读完即可在「Mantle 保持现有 OP Stack / general EVM + 少量 sequencer/txpool/RPC 适配」约束下做架构选型。不是四范式百科。  
> **可信度**：[一手] 官方文档 / 源码 / 合约 / 官方 blog / L2Beat　[二手] 媒体或聚合器　【实测】仓库既有 RPC 基线　⚠️存疑  
> **对照基线**：`0-narrative/03-mantle-baseline.md`（2026-09-06 实测：2.000s / 60M gasLimit / 单 sequencer / 无 preconfs）  
> **对照上一篇**：`3-perps/onchain-perps-landscape-and-opstack-feasibility.md`（下称「景观文」）

---

## 0. 与景观文的显式冲突（先读这段）

景观文把 **Variational** 和 Aevo/Vertex 捆成「范式三：链下 CLOB + 链上结算」，并把 **Ostium** 捆成「范式四：GMX 式点对池」。一手文档不支持这两条归类。

| 景观文说法 | 一手事实 | 对本决策的影响 |
|---|---|---|
| Variational = Aevo 式 hybrid CLOB | Variational 官方：「*Variational is a request-for-quote (RFQ) protocol, and does not utilize an orderbook.*」[一手] https://docs.variational.io/variational-protocol/key-concepts/trading-via-rfq | **不得再按 CLOB 评估 Variational。** 它是 RFQ + 双边 settlement pool + last look。 |
| Ostium = GMX 式交易者 vs LP 池 | Ostium 官方 FAQ：「*Does Ostium run an order book? No. Ostium uses request-for-quote (RFQ)*」；OLP 是 senior working capital，**不是**方向性对手盘；净 delta 链下对冲 [一手] https://docs.ostium.com/protocol/how-ostium-works.md | Ostium 更接近「RFQ + 链上金库结算 + 链下对冲」，不是 GMX。 |
| Oracle-based 是 Mantle 唯一主干 | GMX 的两步 keeper 模型、Variational RFQ、Ostium RFQ+hedge **都已经跑在 Arbitrum 这种 general EVM L2 上，零链改** | Oracle-pool 仍可行，但不是唯一。D2（股票 Perps + Oracle-based 起步）本报告**不锁定**。 |
| Hybrid CLOB 是「高频做市商扩展路径」 | Vertex 曾把 off-chain sequencer + on-chain settlement 部署在包括 Mantle 在内的多条 general EVM 上，**2025-07-08 起停止现有 EVM 部署** [一手] https://www.prnewswire.com/news-releases/vertex-protocol-team-to-join-ink-foundation-build-new-defi-primitives-on-ink-layer-2-302499517.html ；Aevo 选择的是 **专用 OP Stack L2**，不是共享通用链 [一手] https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/exchange-structure/layer-2-architecture.md | Hybrid CLOB 在「共享 general L2」上有失败先例；在「专用 rollup」上可跑，但那已经不是 Mantle-as-is。 |

景观文里「纯链上 Solidity CLOB 在 60M/2s 下物理不可行」这一条**仍然成立**，本报告不重做。Out：Hyperliquid / dYdX L1、RiseX 暴力 gasLimit、Lighter 专用 ZK。

---

## 1. 一页纸结论

**Mantle 今天就能跑 perps。卡点不是 2s 出块或 60M gas，是对手盘、预言机、清算执行与信任假设。** 填充率 【实测】0.173%、每块 1.45 笔（`03-mantle-baseline.md`）。没有需求，谈不上挤出。

**推荐双路径（都是 general EVM 合约，不改共识 / 不改 EVM / 不拉 gasLimit）：**

1. **默认路径 A — RFQ 结算（Variational / Ostium 类）**  
   链下要价、链上托管与清算。适合多标的、长尾、RWA/股票：不需要为每个市场养订单簿。选哪种子类取决于流动性从哪来：  
   - 自营 MM + 外部对冲、用户资金永不离开链上托管 → Variational / OLP  
   - 机构 dealer 报价 + 链下对冲账本、金库只做日内结算缓冲 → Ostium  
2. **并列路径 B — Oracle-priced pool（GMX v2 类）**  
   无撮合。用户 `createOrder`，keeper 带 Chainlink Data Streams 签名价 `executeOrder`。适合已有链上库存、愿意让 LP 承担方向性风险的市场。

**明确不作为 v1：** 共享 Mantle 上的 hybrid CLOB（Aevo/Vertex 类）。Vertex 已经在这条路上死过（含多链 Edge）；Aevo 靠专用 L2 + 交易所代付结算 gas 才把 CLOB UX 做出来。若未来只为 BTC/ETH 要 CEX 盘口，再单独立项，不要当默认。

**Infra 排序（都落在 sequencer / txpool / RPC，不改共识）：**

| 优先级 | 项 | 对推荐路径的真实收益 | 现在要不要做 |
|---|---|---|---|
| P1 | Revert protection（打包前模拟，必 revert 则丢弃不收费） | GMX/Ostium keeper 与清算的运维门槛 | **值得做**。Mantle 空链上也便宜，而且是协议强制才有差异化（Flashbots Protect 只是自愿 RPC） |
| P2 | Flashblocks / preconfs（~200ms 软确认） | 缩短两步执行窗口、RFQ last look 的 UX | **值得做，但不 unblock 产品**。产品在 2s 下已经能跑 |
| P3 | 清算专用 lane | 极端行情清算不被挤出 | **预留设计，不要空转**。0.173% 填充率下今天没有挤出 |
| — | Cancel-before-fill 排序 | 只服务 CLOB 做市商 | **v1 不做**（推荐路径没有 resting book） |

---

## 2. 四条 general-EVM 生产系统（决策粒度）

分类轴是 **撮合发生在哪、对手盘是谁、结算写不写进共享 EVM 状态**。不是「去不去中心化」口号。

### 2.1 对照总表

| 维度 | Variational / Omni | Ostium | GMX v2 | Aevo | Vertex（历史） |
|---|---|---|---|---|---|
| **宿主** | Arbitrum（共享 L2）[一手] 合约在 Arbiscan | Arbitrum（共享 L2）[一手] | Arbitrum / Avalanche / Botanix / MegaETH [一手] https://docs.gmx.io/docs/intro | **专用** OP Stack L2（Conduit）[一手] | 曾在 Arbitrum 等 EVM；**已停** [一手] PR Newswire 2025-07-08 |
| **撮合位置** | 链下 RFQ；**无订单簿** [一手] | 链下 RFQ；**无订单簿** [一手] | **无撮合**；预言机定价 [一手] | 链下 CLOB + 风控引擎 [一手] | 链下 sequencer CLOB（~10–30ms）[二手] Elixir 集成文档 |
| **对手盘** | 唯一 MM：OLP；每用户一条 User<>OLP pool [一手] | 机构伙伴对冲净 delta；OLP **不是**方向对手盘 [一手] | GM Pool / GLV（LP 承担方向）[一手] | 订单簿匿名对手方 | 订单簿 + 链上 AMM slow-mode |
| **结算** | 链上 isolated escrow（settlement pool）[一手] | 链上金库即时结算 USDC；每日与链下对冲账对账 [一手] | 链上 OrderVault + DataStore；两步执行 [一手] | 资金与仓位始终在 L2 合约 [一手] | Endpoint / Clearinghouse 批量上链 [二手] |
| **保证金** | 池级 IM/MM；Omni 平台统一；cross / isolated [一手] https://docs.variational.io/omni/trading/margin.md | 25% collateral backstop；杠杆越高阈值越近 [一手] https://docs.ostium.com/traders/trading/liquidation.md | 市场 `minCollateralFactor` 0.25%–1% notional [一手] https://docs.gmx.io/docs/trading/liquidations | 全账户 cross：`AB+UP-OO-MM>0` [一手] | 统一 cross-margin 子账户 [二手] |
| **清算** | MM≥100% 触发部分强平；罚 0.5% 报价偏移；双边同时平 [一手] https://docs.variational.io/omni/trading/liquidation.md | Gelato keeper；阈值跌破则全额没收剩余抵押 [一手] | keeper 带 min/max 预言机价强平；先尝试 SL [一手] | 链下清算引擎接管账户；订单簿→保险基金→ADL [一手] | 链上 risk engine + 预言机 [二手] |
| **预言机** | 自研 Variational Oracle，多交易所加权 [一手] https://docs.variational.io/variational-protocol/key-concepts/variational-oracle.md | 自研 consensus oracle；开仓 on-demand；sub-second [一手] | Chainlink Data Streams pull；minPrice/maxPrice [一手] https://docs.gmx.io/docs/api/contracts/architecture | 自研 index / mark [一手] 文档目录 | Vertex 运营聚合 Pyth/Stork/Chainlink [二手] |
| **Keeper / 执行** | 官方 Transactor 代提交、代付 gas；Watcher 确认 [一手] https://docs.variational.io/variational-protocol/overview.md | 官方称 keeper permissionless；清算走 Gelato Functions [一手] | 有角色的 Oracle keeper + Order keeper [一手] | 交易所自己撮合、代付结算 gas [一手] | 中心化 sequencer；slow-mode 回退 AMM [二手] |
| **信任假设** | 报价/last look/预言机/Transactor 中心化；**用户资金隔离在 pool，OLP 对冲只用自有资金** [一手] | 预言机 + 链下对冲伙伴 + 每日 USDC 进出金库；界面可替换 [一手] | Chainlink DON + 许可 keeper；LP 方向性风险 | 撮合/风控/清算全链下；专用 sequencer | 撮合单点；2025-07 整栈停运本身就是最大实证 |
| **ADL** | OLP 被清算 = 对手方 ADL；用户拿罚金 [一手] https://docs.variational.io/omni/trading/automatic-deleveraging-counterparty-liquidation.md | 不走订单簿 ADL；亏损先打 junior buffer [一手] | `AdlHandler`；池 PnL 超阈值减盈利仓 [一手] 架构页 | 保险基金不够才 ADL；BTC/ETH/SOL options+perps 当前不在 ADL [一手] | 有协议级清算瀑布 [二手] |

### 2.2 Variational：RFQ + P2P settlement pool（Arbitrum）

**机制（官方流程）** [一手] https://docs.variational.io/variational-protocol/key-concepts/trading-via-rfq.md

1. Taker 发 RFQ（结构、方向、规模）。  
2. Maker 回 quote：价格 + 将使用的 settlement pool 参数（保证金、清算罚金）。  
3. Taker 接受并预授权转抵押，**资金此时不动**。  
4. Maker **last look** 通过后，才创建/使用 pool、划转抵押、记账。

Omni 把上述收成「用户 vs 唯一 maker OLP」：参数预置，OLP last look 查 risk limit，用户 last look 查滑点。官方写全程「数秒」。

**结算**：settlement pool 是链上合约，装仓位、USDC、保证金规则。Omni 上每用户一条 User<>OLP 双边池，池与池隔离，无传染 [一手] https://docs.variational.io/variational-protocol/key-concepts/settlement-pools.md https://docs.variational.io/variational-protocol/key-concepts/p2p-trading-protocol-vs-dex.md

**坏账**：穿仓时赢家无法向破产对手方追偿——这是 P2P 的定义风险，官方直写。OLP 破产则用户后续 PnL 变成 bad debt。用户本金「从未被 OLP 转到 CEX」[一手] https://docs.variational.io/omni/the-omni-liquidity-provider-olp.md

**主网合约（Arbitrum）** [一手] https://docs.variational.io/technical-documentation/mainnet-contracts.md

- OLP Vault `0x74bbbb0e7f0bad6938509dd4b556a39a4db1f2cd`
- Settlement Pool Factory `0x0F820B9afC270d658a9fD7D16B1Bdc45b70f074C`
- Oracle `0x84BE56470d45b7f6629A66A219a38681F6BA6172`
- Treasury `0x5e91b40467fb8902c46a7b6cb90482363188d645`（文档写抽 OLP 价差的 20%，并标明仍在试验）

**对 Mantle 的含义**：这是「general EVM 上实现 perps」的存在证明，而且**故意不做订单簿**。复制它需要：报价引擎、last look 服务、链上 factory/pool、自研或可验证预言机、代付 gas 的 transactor。不需要改 Mantle 客户端。

### 2.3 Ostium：RFQ + 链上金库 + 链下对冲（Arbitrum）

**两层并行** [一手] https://docs.ostium.com/protocol/how-ostium-works.md （更新 2026-07-22）

- **链上结算**：USDC 在 Arbitrum 合约；开平仓、清算、PnL 即时结算。  
- **链下对冲**：Jump 等机构按净 delta 在底层市场对冲。  
- **每日 settlement run**：赢的交易者由金库垫付，当日再从链下对冲账补回 buffer；输的方向相反。官方原话：日内变的是 USDC *location*，协议净敞口在两本账之间是平的。

OLP = senior tranche 工作资本，junior buffer（Ostium 关联方/战略伙伴）先吃交易亏损。OLP 收益只来自开仓费的 30%，**不再吃交易者 PnL**（文档明确这是升级后模型）。

**执行**：用户提交后「keeper 拉当前预言机价、扣费、链上开仓，通常 1–2 秒」[一手] https://docs.ostium.com/traders/trading/opening-a-trade.md 。清算：Gelato Functions，协议付 gas，剩余抵押归协议 [一手] https://docs.ostium.com/traders/trading/liquidation.md

**价格**：自研 consensus oracle；多 publisher 签名后上链；开仓 on-demand。Crypto 用交易所订单簿，RWA 用多家行情商，官方称 sub-second。动态 price-impact 对 crypto/股票生效（单向流加点差）[一手] order-types 页。

**Ostium 为什么强调 Arbitrum**：官方列四条——sub-second 终局、约 $0.01 gas、以太坊安全、原生 USDC。其中 **sub-second 是 Mantle 没有的**（Mantle 2.000s）。1–2 秒开仓在 Mantle 上会变成「至少跨一个 2s 区块」，见 §4。

**对 Mantle 的含义**：同样零链改。真正的准入门槛是 **对冲柜台 + 自研/可信 RWA 预言机 + 金库分层**，不是 sequencer。股票 Perps 若走这条，xStocks/Fluxion 可以当对冲腿，但不自动等于 Ostium 的 Jump 网络。

### 2.4 GMX v2：无撮合、两步请求-执行（Arbitrum 等）

**执行模型** [一手] https://docs.gmx.io/docs/api/contracts/architecture

```
用户 createOrder（不含预言机价）→ 请求上链
Oracle keeper 签 Data Streams 价
Order keeper executeOrder(key, oraclePrices)
合约校验价格并原子成交；失败则 market 单取消、limit 单冻结
```

官方写明两步是为了防抢跑：意图先上链，执行价后绑。延迟「通常数秒」，靠 `acceptablePrice` 保护。

**定价**：Chainlink Data Streams 的 min/max（bid/ask）。开多付 maxPrice，平多用 minPrice，做空相反 [一手] https://docs.gmx.io/docs/trading/order-types 。Limit 不是 resting CLOB：预言机价碰到触发价，keeper 才执行；快速行情可能跳过触发价。

**对手盘**：每个市场一个 GM 池（long token + short token），GLV 再包多层 GM。交易/清算/借款费流入池，LP 吃方向 [一手] https://docs.gmx.io/docs/providing-liquidity

**清算**：剩余抵押（扣未实现亏损、费用、封顶负价格冲击）< 市场最低抵押（0.25%–1% 仓位）。keeper 先尝试关联 SL / 条件补保证金，失败再强平。剩余抵押退钱包（与 Ostium「清算后归零」不同）[一手] https://docs.gmx.io/docs/trading/liquidations

**Keeper 不是完全无许可**：`RoleStore` 给 handler / keeper / 治理分角色。用户付 execution fee（原生 gas token），多余退回。

**对 Mantle 的含义**：合约可原样部署。代价是 (1) 每个市场要有 GM 库存，股票/长尾很难靠 LP 铺；(2) 两步延迟叠 2s 出块比 Arbitrum 更钝；(3) 清算依赖 keeper 抢块，空链上没问题，满链才要 lane。

### 2.5 Aevo：专用 OP Stack 上的 hybrid CLOB

[一手] https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/exchange-structure/off-chain-orderbook-and-risk-engine.md

- 挂单/撮合/开仓前风控 **全部链下**。匹配成功才打进 L2 合约。  
- 资金与仓位始终在链上合约 [一手] on-chain-settlement。  
- **创建/取消订单 0 gas**；结算 gas 由交易所付；存取用户付 [一手] layer-2-architecture。  
- Conduit sequencer **每 1 小时** 向以太坊交 batch；争议期 **2 小时**；提现 2–3 小时。存款走 Optimism Standard Bridge，约 10 分钟。  
- 清算：链下引擎接管 → 取消挂单 → 每 2s 在订单簿上增量平仓最多 30s → 保险基金 → ADL [一手] liquidations。

**对 Mantle 的含义**：Aevo 证明「OP Stack **可以**承载 hybrid CLOB」，但证明的是 **专用链 + 运营方代付结算 gas + 链下订单簿**，不是「在一条空的共享 general L2 上再部署一套 Aevo」。若 Mantle 把整条链让给一个交易所，那是改产品形态，不是 Mantle-as-is。

### 2.6 Vertex：共享 EVM 上的 hybrid CLOB，已停

架构（非官网、作历史机制）：off-chain sequencer 维护订单簿，匹配后批量打 Endpoint/Clearinghouse；sequencer 挂了走链上 AMM slow-mode [二手] https://github.com/ElixirProtocol/vertex-contracts/blob/main/docs/docs.md

**停运是一手的**：Ink Foundation 2025-07-08 新闻稿，Vertex 停止现有 EVM 部署，团队与撮合栈迁 Ink，VRTX sunset [一手] PR Newswire。这不是「机制不能跑」——它跑过——而是 **共享 general L2 上的 hybrid CLOB 没能活成独立产品**。Mantle 曾出现在第三方「Edge 多链」叙述里 [二手]；本报告不把「Vertex 在 Mantle 的定量成交」当事实（见存疑）。

---

## 3. 对 vanilla EVM L2 的硬要求

数字能核对的写数字；不能的标 ⚠️。不要把 Arbitrum 的体感直接抄成 Mantle 的物理极限。

| 架构 | 链上 gas / 费用 | 延迟敏感点 | 预言机 | Keeper | 信任 / 运营 |
|---|---|---|---|---|---|
| **Variational RFQ** | 用户开仓的链上 tx 由 Transactor 代付；用户感知 0 gas。单笔 gasUsed **本轮未实测** ⚠️ | last look 窗口（市价漂移）；官方「数秒」[一手] | 自研流式加权；上链 oracle 合约存在 | 官方 Transactor + Watcher，中心化 | 必须有能对外对冲的 MM 库存；OLP 资金风险 ≠ 用户托管风险 |
| **Ostium RFQ+hedge** | 官方 Arbitrum「约 $0.01 / 笔」[一手]；另收 $0.10 oracle fee / 次询价 [一手] | 开仓 1–2s（Arb）；清算「数秒内」[一手] | 自研 sub-second，开仓 pull | Gelato；清算协议付 gas | 对冲伙伴 + 每日跨链下账本调拨 USDC |
| **GMX v2** | 至少两笔 L2 tx（create + execute）+ execution fee。景观文 15–30 万 gas **本轮未复测** ⚠️ | 两步窗口「数秒」[一手]；Arb 出块 ~0.25s vs Mantle 2s | Chainlink Data Streams；要有对应 feed 才能上市场 | 许可 keeper；gas 暴涨可能推迟执行 | GM 库存；LP 方向性；Data Streams 覆盖面 |
| **Aevo CLOB** | 挂撤 0；结算运营方付。gasUsed ⚠️ | 撮合 <10ms 级（官网宣传，本轮未独立测）⚠️；链上结算跟 L2 出块 | 自研 index | 无公共 keeper，运营方即撮合方 | **专用链**；1h batch / 2h 挑战 |
| **Vertex CLOB** | 结算批量上链；挂单 EIP-712 链下 [二手] | 撮合 10–30ms [二手] | 自营聚合 | sequencer 单点 | 已停运 |

**共同硬约束（与是否 CLOB 无关）：**

1. **清算必须在下一个或至多数个区块内发出。** 高杠杆下 2–3 个 Mantle 块 = 4–6s。Ostium 200x 官方示例：0.4% 价格移动即清算 [一手]。这是 keeper/sequencer 问题，不是 EVM 指令集问题。  
2. **预言机必须是 pull 或等价的「执行时才绑定价格」。** GMX 两步、Ostium 开仓 fetch、Variational firm quote 都在解决同一件事：用户不能对已知未来价下单。  
3. **状态争用只打 CLOB。** RFQ/oracle-pool 的热点是「单用户账户」或「单市场 OI」，不是全市场订单簿红黑树。并行 EVM 对推荐路径几乎无增益——景观文这条对 CLOB 仍对，对 RFQ/GMX **不要当成否决票**。

---

## 4. 映射到 Mantle-as-is（零 infra 改动）

基线（与 `03-mantle-baseline.md` 对齐，2026-09-06 【实测】除非注明）：

| 参数 | 值 |
|---|---|
| 出块 | **2.000 s** |
| gasLimit | **60,000,000** |
| 填充率 / 每块交易 | **0.173% / 1.45 tx** |
| Sequencer | 单 sequencer；出块即软确认；**无 preconfirmation SLA** |
| 证明 | OP Succinct / SP1，提现 ~12h |
| `eth_sendRawTransactionSync` | 不存在（-32601）【实测，见 `04-mantle-opstack-surface.md`】 |
| Flashblocks / rollup-boost | **无** |

### 4.1 零改动能否跑？

| 架构 | 零改动 | 卡住的不是链参数，是 |
|---|---|---|
| Variational RFQ | **能跑** | OLP/报价/对冲团队；自研预言机；Transactor 密钥与代付；P2P 坏账政策 |
| Ostium RFQ+hedge | **能跑** | 对冲柜台；RWA 预言机；Gelato 或自建 keeper；2s 出块让官方「1–2s 开仓」变成「跨块」 |
| GMX v2 | **能跑** | Chainlink Data Streams 是否覆盖目标资产；GM 池冷启动；两步最少 2s（请求块）+ 2s（执行块）≈ 4s 体感 |
| Aevo 式 CLOB 放在 **共享** Mantle | 机制能部署 | 挂撤若要 0 gas 必须运营方代付；做市商被 2s 逆向选择；与链上其他应用抢块（今天空，成功后才痛）；Vertex 同构已停 |
| Aevo 式 **专用 L2** | 能跑，但 **越界** | 那是再发一条 Conduit 链，不是 Mantle-as-is |
| 纯链上 CLOB | **不能** | 景观文 2.4–4.2M gas/笔、14–24 笔/块。本报告不重复推导 |

### 4.2 2 秒出块分别打在哪

- **RFQ**：firm quote 的时效。Variational 已有 last look + 用户滑点；2s 只是把「数秒」拉长，不破坏模型。Preconfs 改善的是 UX，不是可行性。  
- **Ostium**：官方把 Arbitrum sub-second 写成选型理由。迁 Mantle 后开仓/清算的链上腿变慢，**对冲腿（链下）不受影响**。高杠杆股票开盘缺口更危险。  
- **GMX**：两步窗口变宽 → 预言机价更容易偏离 `acceptablePrice` → 市价单取消率上升。Keeper 若按「每块扫一次」，频率已经是 Arb 的 ~1/8。  
- **CLOB**：2s 撤单确认是做市商逆向选择的教科书条件。这是 hybrid CLOB 不该作为 v1 的链级原因（另一条是 Vertex 停运）。

### 4.3 60M gas / 空链

今天任何推荐路径都不会打满区块。景观文「单块 200–400 笔 GMX 开平仓」即使 gas 数字 ⚠️，数量级仍远小于 60M 在空链上的余量。**不要为尚未存在的 perps 去改 gasLimit。**

---

## 5. Infra delta（只评估允许的四项）

约束：不改共识、不改 EVM 指令集、不暴力拉 gasLimit。改动面限于 sequencer / txpool / RPC。

### 5.1 Preconfs / Flashblocks

| | |
|---|---|
| **收益** | ~200ms 软确认。GMX 两步的「请求已进入 pending 状态」可被 keeper 更早看见；RFQ 前端不必等 2s 才显示成交；`eth_sendRawTransactionSync` 同类 UX。 |
| **改动面** | OP Stack **out-of-protocol** sidecar：rollup-boost + builder + RPC overlay。最终密封块仍是 2s 标准块。官方 spec：[一手] https://specs.optimism.io/protocol/flashblocks.html ；OP 文档：[一手] https://docs.optimism.io/op-stack/features/flashblocks.md 。Base 主网 200ms：[一手] https://docs.base.org/base-chain/flashblocks/faq |
| **先例** | Base（2025-07 主网）、Unichain、OP Mainnet 250ms。Mantle：**无**（`04-mantle-opstack-surface.md` N-4）。 |
| **攻击面** | Preconf 是 sequencer 承诺，不是 L1 终局。Sequencer 可在密封前改序/丢交易；应用若按 pending 放款/成交，存在预确认与密封块不一致。Flashblocks 在极端情况下会关停并回退 2s 块 [一手] Base FAQ。OP Succinct 下欺诈/有效性证明覆盖的是密封块，**不自动证明每一条 preconf**。 |
| **决策** | **P2，做。** 不改架构叙事，且是 Base 已验证的 OP Stack 插件。不要宣传成「200ms 终局」。 |

### 5.2 清算专用 lane

| | |
|---|---|
| **收益** | 为清算/ADL tx 保留不可借用的 gas 配额，避免与投机流量竞价。对 GMX/Ostium 这类 **清算必须上链** 的模型有意义。Variational 清算由 Transactor 提交，同样受益。CLOB 链下清算（Aevo）几乎不需要。 |
| **改动面** | op-geth txpool + 出块器：交易类型标签 + 预验证（「该 tx 确实对应已越阈仓位」）。配额空置也不能借给普通 tx，否则退化为优先费。 |
| **先例** | 检索仍无协议层「不可挤占清算配额」生产先例（与 `native-chain-support.md` 空白①一致）。Solana priority fee / Flashbots 私有通道是加钱插队，不是保留配额。 |
| **攻击面** | 标签伪造 = 免费优先出块。预验证必须保守。全市场级联时 lane 内部仍会堵，需要按风险排序的二级规则。空链上收益为 0，只有 perps 真的打满才显现。 |
| **决策** | **P3，设计预留、不要空转。** 与交易类型标签基础设施一起做，等填充率不再是 0.173%。 |

### 5.3 Revert protection（模拟后免费拒绝）

| | |
|---|---|
| **收益** | GMX keeper、Ostium Gelato、清算机器人今天都要为 revert 付 gas。协议强制「模拟必失败则不打包」，降低 keeper 门槛，减少垃圾块。对 RFQ last look 失败单同样有用。 |
| **改动面** | 单 sequencer 本来就能在打包前执行。制度化：对白名单选择器（清算、executeOrder、条件单）失败则丢弃且不收费。必须按选择器开关——全链一刀切会降低狙击/垃圾 tx 的试错成本（发行场景的 bot tax 会失效，见既有 native-chain-support 提醒）。 |
| **先例** | Flashbots Protect = **自愿 RPC**，不是协议。[一手概念对照] 用户仍可直连 sequencer。NEAR 按用量退款 ≠ 执行前拒绝。协议强制层仍无生产先例（空白②）。 |
| **攻击面** | 模拟与执行状态不一致（同块前序 tx 改变仓位）→ 误丢真清算或误放假清算。免费拒绝 = 免费 CPU DoS，必须限模拟 gas、限地址频率。隐藏 revert 让链上可观测性变差，要另打日志。 |
| **决策** | **P1，做。** 成本最低、与推荐路径直接相关、且 Mantle 单 sequencer 没有「多 builder 模拟不一致」问题。 |

### 5.4 排序策略（cancel-before-fill 等）

| | |
|---|---|
| **收益** | 只在 **resting CLOB** 上保护做市商撤单。RFQ 没有挂单队列；GMX limit 不是队列优先。 |
| **改动面** | txpool 排序键。Gas=0 撤单必须加频率配额，否则 DoS。 |
| **先例** | Hyperliquid HyperCore（非 EVM L1）[一手，景观文已引]。OP Stack 共享链无生产先例。 |
| **攻击面** | 伪造 cancel 占满优先队列；与 fee 排序的公开承诺冲突。 |
| **决策** | **v1 明确不做。** 若未来上 hybrid CLOB 再开。 |

### 5.5 不做清单（infra）

- 改共识 / 新 VM / precompile 撮合 / 热点账户串行 lane  
- 把 gasLimit 拉到 RiseX 级 1.5 Ggas  
- Oracle 系统交易插区块首位（越出本次允许的 sequencer/txpool/RPC 上限；且 RFQ/GMX 已在应用层绑价）  
- 为尚未存在的 CLOB 做 cancel 优先  

---

## 6. 推荐架构（双路径）与「明确不做」

### 6.1 推荐

**v1 在 Mantle 现架构上部署 RFQ 结算协议（路径 A），Oracle-pool（路径 B）作为可并行的第二市场类型，而不是互相替代的宗教。**

| 若你的约束是… | 选 |
|---|---|
| 要上几十到上百个股票/商品，没有为每个市场养 LP 或订单簿的预算 | **A：Variational 类**（自营 OLP + 外部对冲，用户资金隔离）或 **Ostium 类**（dealer RFQ + 链下对冲 + 金库分层） |
| 已有 USDC/ETH 库存、标的有 Chainlink Data Streams、能接受 LP 方向性 | **B：GMX v2 类** |
| 要 CEX 深度图、高频 maker、免费挂撤 | **不要当 v1**。那是 Aevo 专用链或已死的 Vertex 路径 |

股票 Perps **不是前提**。若产品恰好是股票：路径 A 更省事（Ostium 已在 Arb 上做股票；Variational 以「有可靠价格源就能上市场」为官方卖点）。路径 B 需要 Data Streams 覆盖该股票，GMX 已开始上部分 TradFi 市场 [一手] Chainlink 生态页 2026-04 黄金/白银、WTI 公告，但覆盖面仍是预言机说了算。

**与 D2 的关系**：D2 把 Oracle-based 当成唯一起步，在「不要上链上 CLOB」这一点上仍然对；在「因此只能做 GMX/Ostium 点对池」这一点上 **过窄**——Ostium 自己已经不是点对池，Variational 也不是。本报告把 D2 降级为「路径 B 可选」，不当作产品锁。

### 6.2 明确不做

1. 纯 Solidity 链上 CLOB  
2. 为 perps 再发一条专用 OP Stack（Aevo 复制品）——那是放弃 Mantle 共享生态  
3. 把 Variational 当 hybrid CLOB 抄一套撮合引擎  
4. 把共享 Mantle 改造成 Vertex Edge 2.0 当 v1  
5. 靠拉 gasLimit / 并行 EVM / 撮合 precompile 让 CLOB「变得能跑」  
6. 在空链上先做清算 lane / cancel 优先「等流量」  
7. 把 preconfs 写成终局性或写成产品上线的门禁  

### 6.3 最小落地组合（应用层 + 允许的 infra）

```
应用层（必须有，链改 0）
  ├─ 链上：托管 / 保证金 / 清算 / 记账（pool 或 vault 或 GM）
  ├─ 链下：报价或预言机绑定（RFQ last look 或 Data Streams）
  └─ 执行：Transactor 或 keeper（Gelato 可先用）

链层（按优先级）
  ├─ P1 revert protection（选择器白名单）
  ├─ P2 Flashblocks sidecar + pending RPC（可选 eth_sendRawTransactionSync）
  └─ P3 清算 lane 的交易类型标签（先打地基，配额可先为 0）
```

---

## 7. 存疑清单

1. ⚠️ **GMX 单笔 15–30 万 gas、hybrid 结算 10–18 万 gas**：来自景观文，本轮未在 Arbiscan 复测。不影响「远小于 60M」的方向判断，**不影响选型**。  
2. ⚠️ **Variational / Ostium 开仓的实际 gasUsed**：官方只给「代付」或「~$0.01」，没有公开 gas 表。  
3. ⚠️ **Ostium 预言机与 Stork 的关系**：第三方审计笔记写 Stork 基础设施 [二手]；Ostium 官方 how-ostium-works 只写 in-house consensus oracle，未点名 Stork。以官方为准，Stork 标存疑。  
4. ⚠️ **Vertex 是否正式部署过 Mantle Edge 主网、成交量多少**：第三方协议卡提到 Mantle [二手]；PR Newswire 只说「现有 EVM 部署」不停点名。不把「Mantle 上 Vertex 失败」写成已核实的量。失败事实本身（协议停运）是一手的。  
5. ⚠️ **Aevo 官网 <10ms / >5000 tps**：未独立测；当作营销数字。机制（链下撮合、交易所付结算 gas）不依赖这两个数。  
6. ⚠️ **Variational 抽成 20% 价差进 treasury**：官方写「仍在试验、可改」。  
7. ⚠️ **Flashblocks 在 OP Succinct（SP1）下的证明覆盖**：Base/OP 先例多为 optimistic 或混合；Mantle 已是 validity proof。sidecar 不改 STF 则证明范围仍是密封块，但工程集成（op-rbuilder vs mantle 的 geth 分叉 / SP1 vkey）本轮未做客户端 diff。  
8. ⚠️ **Gelato Functions 在 Mantle 是否生产可用**：Ostium 用它是因为在 Arbitrum。Mantle 上的 keeper 网络覆盖未核实；可退回自建 keeper。  
9. ⚠️ **景观文「Oracle-Liquidation Atomicity」**：越出本次允许的 infra 上限（接近系统交易/预编译），本报告不推荐为 v1。

---

## 8. 关键来源清单

### Variational [一手]

- https://docs.variational.io/variational-protocol/overview.md  
- https://docs.variational.io/variational-protocol/key-concepts/trading-via-rfq.md  
- https://docs.variational.io/variational-protocol/key-concepts/p2p-trading-protocol-vs-dex.md  
- https://docs.variational.io/variational-protocol/key-concepts/settlement-pools.md  
- https://docs.variational.io/variational-protocol/key-concepts/variational-oracle.md  
- https://docs.variational.io/omni/about-omni.md  
- https://docs.variational.io/omni/the-omni-liquidity-provider-olp.md  
- https://docs.variational.io/omni/trading/liquidation.md  
- https://docs.variational.io/omni/trading/margin.md  
- https://docs.variational.io/omni/trading/automatic-deleveraging-counterparty-liquidation.md  
- https://docs.variational.io/omni/trading/quoted-index-and-mark-prices.md  
- https://docs.variational.io/technical-documentation/mainnet-contracts.md  

### Ostium [一手]

- https://docs.ostium.com/protocol/how-ostium-works.md  
- https://docs.ostium.com/vault/overview.md  
- https://docs.ostium.com/traders/trading/opening-a-trade.md  
- https://docs.ostium.com/traders/trading/liquidation.md  
- https://docs.ostium.com/traders/trading/order-types.md  
- https://docs.ostium.com/traders/reference/fees.md  

### GMX [一手]

- https://docs.gmx.io/docs/intro  
- https://docs.gmx.io/docs/api/contracts/architecture  
- https://docs.gmx.io/docs/api/contracts/exchange-router  
- https://docs.gmx.io/docs/trading/order-types  
- https://docs.gmx.io/docs/trading/liquidations  
- https://docs.gmx.io/docs/providing-liquidity  
- https://docs.chain.link/data-streams  

### Aevo [一手]

- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/exchange-structure/off-chain-orderbook-and-risk-engine.md  
- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/exchange-structure/on-chain-settlement.md  
- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/exchange-structure/layer-2-architecture.md  
- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/liquidations.md  
- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/auto-deleveraging-adl.md  
- https://www.aevo.xyz/docs/aevo-products/aevo-exchange/technical-architecture/margin-framework.md  

### Vertex [一手停运 / 二手机制]

- https://www.prnewswire.com/news-releases/vertex-protocol-team-to-join-ink-foundation-build-new-defi-primitives-on-ink-layer-2-302499517.html  
- https://github.com/vertex-protocol/vertex-contracts  
- https://github.com/ElixirProtocol/vertex-contracts/blob/main/docs/docs.md  

### Mantle 基线与 preconfs

- `0-narrative/03-mantle-baseline.md` 【实测 2026-09-06】  
- `0-narrative/04-mantle-opstack-surface.md`（无 preconfs；Flashblocks 接入路径）  
- https://specs.optimism.io/protocol/flashblocks.html  
- https://docs.optimism.io/op-stack/features/flashblocks.md  
- https://docs.base.org/base-chain/flashblocks/faq  

### 本仓库对照（不重写）

- `3-perps/onchain-perps-landscape-and-opstack-feasibility.md`  
- `3-perps/chain-infra/native-chain-support.md`  
- `3-perps/README.md`（既有 D2，已被本报告降级为可选路径 B）
