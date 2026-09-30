# 排序 / 延迟 / 区块空间 / MEV 的链级零件目录
> 研究轨道：K ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

## 0. 本文档定位与结论先行

本文档是「排序 / 延迟 / 区块空间 / MEV」这条技术线的**零件目录**——不做设计推荐，只做「谁在生产环境用了、代价是什么」的实证盘点，供最终设计方案（`report/09`）挑选零件。核心读者约束：委托方的目标场景是 **Mantle 上的 meme / 资产发行 Launchpad**，而第一阶段实测数据显示 **Mantle 区块填充率仅 0.173%、base fee 常年钉在 50 gwei 下限、EIP-1559 从未触发**（数据来源：第一阶段 `report/01-06`，本轨道不重复取数，直接引用作为约束条件）。这意味着：

1. **吞吐 / gas 价格竞争不是本场景的瓶颈** —— 任何以"提升 TPS""降低 gas 波动"为目标的零件，投入产出比在此场景下都接近于零，本文档不推荐。
2. 真正稀缺的是**发行时刻的抢跑权**（bot sniping、抢购、公平分配）与**用户可感知的确认延迟**，这两者跟"链的总吞吐"几乎无关，而跟**排序规则、mempool 可见性、pre-confirmation 语义**强相关。
3. 因此本文档对 20+ 个零件的筛选标准始终锚定：**这个零件是否改变"谁能在什么时刻、以什么信息优势，抢到发行/成交的第一手位置"**。

全文结论汇总在 [§6 总表](#6-零件目录总表核心产出) 与 [§7 投入产出比排序](#7-投入产出比排序给-mantle-launchpad-场景的直接回答)，建议直接跳读这两节。

**三条结论先行**（详细论证见 §7）：

| # | 结论 |
|---|---|
| **K-1** | **对发行类应用，这条技术线的最高 ROI 不是"更快"，而是"让抢跑权可被定价或被剥夺"。** 排序规则（§2）与 ASS（§3）的价值远高于 pre-confirmation（§1）。 |
| **K-2** | **委托方想要的效果，绝大部分不需要开新链。** §3 的 ASS 家族（Angstrom / Atlas / MEV tax）与 §6 表中 9 个「无需改客户端」零件，在 Mantle 今天的合约层就能落地；真正必须改链的只有「软确认延迟」与「区块空间隔离」两类。 |
| **K-3** | **最贵的零件（TEE builder、mini-block、local fee market）解决的都是「拥塞下的公平」问题，而 Mantle 没有拥塞。** 在填充率 0.173% 的链上采购这些零件，是为不存在的问题付费。 |
## 1. pre-confirmation 家族

> pre-confirmation（软确认）指的是链在最终不可逆确认（L1 finality / 共识 finality）之前，向用户提供一个"大概率不会变"的中间确认信号，用来把用户可感知延迟从"秒级/区块级"压到"百毫秒甚至十毫秒级"。**关键分野**：这个信号是纯 UX 优化（不改变最终排序权威）还是真正的密码学可验证承诺（TEE attestation / 签名）。

### 1.1 Base Flashblocks —— 200ms 软确认，规则内建于 base-builder

- **机制**【一手：`docs.base.org/specifications/transactions/transaction-ordering`，2026-09-07 读取】：Base 用自研的 `base-builder`（`github.com/base/base/tree/main/crates/builder`）以 **200ms** 为周期做 priority-fee 拍卖，把 2 秒的常规区块切成最多 10 个 Flashblock，每个 Flashblock 的 gas 预算按 1/10、2/10…10/10 递增分配（例如约 40M→80M→…→400M gas，取决于区块 gas 上限）。
- **语义**：一旦某个 Flashblock 被构建并广播，其中的交易排序被**锁定**——即使后续到达更高 priority fee 的交易，也**不能**插进已经广播的 Flashblock。这是一个"部分不可逆"的软确认：Flashblock 内的顺序不再因为后来者而改变，但 Flashblock 本身仍然可以在 L1 层面被重组（理论上，若 sequencer 作恶或故障）。
- **谁签名 / 可否回滚**：Flashblock 由 Base 官方唯一 sequencer（`base-builder`）签发和广播，**没有独立密码学证明**（不是 TEE attestation），本质上是运营方对外的"运营承诺"而非协议强制。因此严格意义上**可以回滚**（sequencer 故障或重组时），但生产中未见公开的大规模 Flashblock 回滚事件。
- **是否需要改客户端**：**不需要**改钱包/合约逻辑；RPC 提供商在 `eth_getBlockByNumber` 等方法中暴露 `pending` tag 即可读到 Flashblock 级别的中间状态，属于 RPC 层扩展。
- **Flashtestations（TEE 方向）与 "Denim" 提案**：⚠️存疑——`web_search` 与二手来源提到 Base 团队在探索基于 TEE 的 block builder 认证方案（代号可能与 "Flashtestations" 相关），但**未能在 `docs.base.org` 或 Base 官方 GitHub 找到名为 "Denim" 的已发布规范或路线图页面**。判断：**按不存在处理**，若委托方有更具体的一手线索（例如具体 PR / RFC 编号）需要另行核实。这是本节最大的一条存疑，需要收录进「存疑清单」。
- **生产状态**：**已上线主网**（Base Mainnet），200ms Flashblocks 为当前生产架构的一部分【一手：`docs.base.org`，读取日期 2026-09-07；文档未标注具体上线的历史日期，只描述当前状态】。

### 1.2 Unichain —— rollup-boost + TEE block builder + Verifiable Priority Ordering

- **机制**【一手：`developers.uniswap.org/docs/unichain`，2026-09-07 读取；二手补充经 web_search 交叉核实】：Unichain（OP Stack、Superchain 一员）同样提供 **200ms Flashblocks** 软确认（1 秒出块，200ms 软确认），但比 Base 更进一步：block builder 运行在 **Intel TDX TEE** 内部（**Unichain 是首个采用 TEE block builder 的 L2**，这一"首个"表述来自二手来源，未见 Unichain 官方明确自我认领"首个"字样，标记⚠️存疑）。
- **Verifiable Priority Ordering (VPO)**：在 TEE 内部执行严格的 priority-fee 排序规则，并生成**公开可验证的执行证明（attestation）**，任何第三方可以验证区块确实是按声明的规则（priority fee 优先）构建的，而非 sequencer 私下插队。这把"信任 sequencer 诚实"降级为"信任 TEE 硬件的完整性（Intel TDX 的安全假设）"。
- **架构基础**：`rollup-boost` 是 Flashbots、Uniswap Labs、OP Labs 三方共同开发的 sidecar 模块，作为 sequencer 的扩展插件运行 TEE block builder，不改变 L2 共识本身，属于"外挂"式增强【二手转述，来自 Flashbots 技术博客与二手分析，未直接读取 rollup-boost 源码确认三方共同开发关系，标记⚠️存疑待核实署名细节】。
- **可否回滚**：TEE attestation 保证的是"排序规则被遵守"，不保证"整个区块不会被重组"——理论上 L1 层面的重组仍可发生，TEE 只是消除了 sequencer 在**排序层面**的作恶空间。
- **是否需要改客户端**：不需要；与 Base 类似，通过 `pending` tag 的 JSON-RPC 扩展获得 Flashblock 数据。
- **生产状态**：**已上线主网**（Unichain Mainnet，Chain ID 130），Flashblocks 与 TEE 排序是当前生产配置的一部分【一手：`developers.uniswap.org/docs/unichain`】。**Verifiable Priority Ordering 的实测延迟数据**（例如"TEE attestation 生成耗时"“端到端确认延迟分布”）**未能找到 Unichain 官方公开的量化实测报告**，标记⚠️存疑。

### 1.3 MegaETH —— mini-blocks 10ms、单 sequencer 全栈重构

- **机制**【一手：`docs.megaeth.com/mini-block`、`www.megaeth.com/blog-news/endgame-how-weve-achieved-10ms-blocks`，经 web_search 交叉确认后于 2026-09-07 核实标题与内容摘要，未逐字通读全文，标记为核实但非逐行复核】：MegaETH 采用双层出块：**EVM Block**（1 秒一次，兼容标准 EVM 工具链/索引器/钱包）与 **Mini-Block**（约 **10ms** 一次，通过专用 Realtime API 提供近实时执行结果，绕开标准 EVM 区块的等待时间）。
- **单 sequencer + 专用分工**：MegaETH 的架构核心假设是"单一高性能 sequencer 做全部执行"，配合专用的 prover 与 full-node 角色分工（sequencer 做执行、专用节点做证明/存档），并使用名为 **SALT**（Small Authentication Large Trie）的技术把关键状态数据保持在内存中以消除磁盘 I/O 瓶颈。
- **软确认语义**：Mini-Block 提供的是**执行结果的早期可见性**，而非独立的共识层承诺——最终仍要落到 1 秒周期的 EVM Block 上做规范化确认，因此其"软确认可回滚性"取决于 10ms 窗口内 sequencer 尚未对外承诺 finality 的部分。
- **是否需要改客户端**：**需要**——若要拿到 10ms 级别的执行反馈，应用需接入专门的 Realtime API，而非标准 `eth_getBlockByNumber`，这与 Base/Unichain 的"零改动"路线不同。
- **生产状态**：**已上线主网**，MegaETH 主网于 **2026-02-09** 上线【一手来源指向 `www.megaeth.com/blog-news/mega-is-live` 及 The Block 报道 `theblock.co/news/ecosystems/2026-02-09-megaeth-debuts-mainnet-389015`，日期经 web_search 多源交叉确认；未逐条打开每个 megaeth.com 页面核对原文，存在轻微⚠️不确定性，但多个独立来源日期一致，可信度较高】。原生代币 $MEGA 的 TGE 于 **2026-04-30** 完成（触发条件：10 个 "MegaMafia" 生态应用上线，达成于 2026-04-23）。

### 1.4 Solana Shreds / Turbine 与 Alpenglow

- **Turbine / Shreds（现状机制，非本轨道深挖对象）**：Solana 把区块切分为 "shreds"（分片数据包），通过 **Turbine**（基于验证者质押权重的树形中继网络）向全网传播，目的是在大规模验证者集合下压缩区块传播延迟。这是 Solana 现有主网机制，不是本文档的深挖重点——**RISE Chain 的 "Shreds" 概念与 Solana Shreds 系出同源但实现不同，深度对比交给 → H 轨道**（`research/H-rise-chain-infra.md`），本轨道只做交叉引用。
- **Alpenglow —— 下一代共识升级，替换 TowerBFT**【一手：`solana.com/upgrades/alpenglow`，2026-09-07 直接读取官方页面】：
  - **当前状态**：官方页面标注为 **"In Development"（开发中）**，**尚未上线主网**。
  - **目标**：把当前 TowerBFT 的 **~12.8 秒**最终确认时间压缩到约 **150ms**（页面原文："targeting roughly 150ms finality"），同时用新投票协议 **Votor** 取代链上投票交易（改为验证者之间直接传递投票 + 聚合证书），可容忍 **20% 恶意质押 + 20% 离线质押**的组合对抗模型。
  - **时间线**：官方页面标注 **Expected Mainnet Activation Date: Q3 2026**（即 2026 年 7-9 月区间，与 2026-09-07 的取数日期非常接近，存在"即将上线"的可能，但**页面本身仍标注 In Development，不能记为已上线**）。相关 SIMD 提案（SIMD-0326 主提案、SIMD-0387 BLS 公钥管理已于 2026-07-08 上线主网、SIMD-0357 验证者准入票据已于 2026-07-22 上线主网）显示准备工作已进入收尾阶段，但**核心共识切换（Votor 本身）尚未激活**。
  - **Phase 2 — Rotor**：计划在 Votor 稳定后，用新的区块传播协议 **Rotor**（从树形传播改为单层中继）替换现有 Turbine，**目前只有路线图描述，无具体上线时间**。
  - **治理**：验证者社区已于 2025 年 9 月以 98.27% 支持率通过该升级方向【二手：多篇报道引用，未直接核实原始投票记录页面，标记⚠️存疑对具体百分比的精确性，但方向性事实（高票通过）可信度较高】。

### 1.5 RISE Shreds —— 仅交叉引用

RISE Chain 的 "Shreds"（无 state root merkleization 的 mini-block，宣称 1ms 延迟）与本节其他方案同属"更细粒度的区块/子区块"范式，但其架构细节、并行 EVM（pevm）配合方式、与 RISEx 的耦合关系属于 **→ H 轨道**（`research/H-rise-chain-infra.md`）与 **→ I 轨道**（`research/I-risex-app-coupling.md`）的深挖范围，本文档不重复取证，仅在 §6 总表中列一行做定位。

### 1.6 pre-confirmation 家族统一对照表

| 方案 | 软确认延迟 | 谁签名/谁承诺 | 可否回滚 | 用户可感知语义 | 是否需要改客户端 | 生产状态与上线日期 |
|---|---|---|---|---|---|---|
| Base Flashblocks | 200ms | 唯一 sequencer（`base-builder`），无独立密码学证明 | 理论可回滚（sequencer故障/重组），生产未见公开大规模回滚 | Flashblock 内排序锁定，后来高费交易不能插队 | 不需要（`pending` tag 读取） | 已上线主网，当前生产配置【一手】 |
| Unichain Flashblocks + VPO | 200ms | TEE（Intel TDX）内执行+attestation，降级为"信任硬件" | 排序层面不可篡改；L1重组风险仍存在 | 排序规则可密码学验证，非"运营承诺" | 不需要（`pending` tag 读取） | 已上线主网，当前生产配置【一手】；VPO 实测延迟数据⚠️未公开 |
| MegaETH mini-blocks | ~10ms | 单一 sequencer，Realtime API 推送执行结果 | 10ms 窗口内为早期执行反馈，非独立finality | 需要新 API（非标准 eth_getBlockByNumber） | 需要（接入 Realtime API） | 已上线主网 2026-02-09【一手/二手交叉确认】 |
| Solana Alpenglow（Votor） | 目标 ~150ms（对比现状 TowerBFT ~12.8s 最终确认，~400ms 乐观确认） | 验证者聚合证书（一/二轮投票，80%/60%阈值） | 目标是"更快的确定性 finality"，非软确认，是共识层升级 | 无需应用改动，是底层协议升级 | 需改共识客户端（Agave） | **开发中，未上线主网**，目标 Q3 2026【一手：官方页面标注 In Development】 |
| **RISE Shreds** | **官方称 3–5ms(p50)/10ms(p99)；【实测】1 tx = 1 shred，约 88 shred / 1s 区块** | **单一 sequencer 签名**（"Shreds with invalid signatures are discarded"）；L1 经济担保属 based sequencing Phase 1，**未上线** | **可回滚**（等价于 OP Stack unsafe head，保证强度为 0） | 每笔交易独立预确认 + `eth_sendRawTransactionSync` 单往返拿回执 | **需要**（自研执行层 `rise-replica` + P2P 改造） | **已上线主网**（chainId 4153）→ 详见 H 轨道 |


**表的读法（三点辨析）**：

1. **「定长子区块」vs「逐笔预确认」是两条不同路线。** Base/Unichain 是把 2 秒切成 10 个 200ms 片段（定时器驱动）；RISE 是每来一笔交易就发一个 shred（事件驱动，【实测】1 tx/shred）。**对逐笔反馈型 UX（下单、撤单、抢购）后者更优；对批量吞吐前者更省带宽。**
2. **除 Unichain 外，全部方案的软确认都只是「运营承诺」。** 只有 Unichain 用 TEE attestation 把排序规则变成可验证的，其余（含 RISE）本质是"相信 sequencer 不反悔"。**这一点在选型时经常被营销话术掩盖。**
3. **软确认延迟与「抢跑公平性」正交。** 把确认从 2s 压到 200ms 不会改变谁抢到第一位；它只改变用户多久知道结果。→ 因此对发行场景，§2/§3 的排序零件优先级高于本节。

## 2. 排序规则货架

> 本节盘点"谁先谁后"的排序规则本身（区别于 §1 的软确认机制、§3 的应用层排序）。每条给机制 / 生产状态 / 已知实测后果。

### 2.1 FCFS（先到先得）—— Arbitrum 历史默认、Robinhood Chain 现行方案

- **机制**：严格按交易到达 sequencer 的时间戳排序，不设公开 mempool 竞价，priority fee 通常不影响排序（甚至被退还）。
- **Arbitrum**：历史上（直至 2025-04 Timeboost 上线前）Arbitrum One/Nova 默认使用 FCFS【一手：`docs.arbitrum.io/how-arbitrum-works/timeboost/gentle-introduction`】。Timeboost 上线后，FCFS 仍是"非快车道"交易的默认排序（叠加 200ms 人为延迟）；若 2026 年的 PGA 提案通过，Arbitrum One 将转向 PGA，FCFS 退化为 K 参数过大时的降级形态。**已知实测后果**：FCFS 催生"延迟竞赛"（latency race）——搜索者为抢占顺序竞相压缩与 sequencer 的物理/网络延迟，造成基础设施军备竞赛式浪费，这正是 Arbitrum 推出 Timeboost 的直接动机。
- **Robinhood Chain**（基于 Arbitrum Orbit 构建的 L2）：官方文档明确"交易顺序严格由到达 sequencer 的时间决定"，定位为反 PGA/反 MEV 设计，单一中心化 sequencer，约 100ms 区块时间【一手：`docs.robinhood.com/chain/`】。**实测后果**：**2026-09-04 发生约 13 分钟的区块生产中断**，暴露单一 sequencer 缺乏热备份/去中心化 fallback 的可用性风险【二手：`cryptorank.io` 报道；`l2beat.com/layer2s/projects/robinhood` 可交叉核实链上状态】。这是"极简排序规则 + 单点可用性风险"的一个鲜活生产反例。

### 2.2 Priority Gas Auction（PGA）—— 机制原理

交易按用户愿付的 priority fee 竞价排序，出价越高越先被打包。在存在公开 mempool 的场景（历史上的以太坊 L1），搜索者 bot 会实时、程序化地不断加价（bidding war）抢占套利/清算等 MEV 机会。**已知历史后果（以太坊 L1）**：①网络拥堵——加价战产生大量 spam 交易，推高普通用户 gas 成本；②效率低下——"暴力法"竞争不保证社会最优结果；③安全风险——公开 mempool 竞价导致三明治攻击等前置交易问题。行业后续演化出 Flashbots 的 PBS（Proposer-Builder Separation）、私有 bundle/relay 机制来替代裸 PGA。**当前进展**：Arbitrum 计划把 PGA **重新引入** L2（见 2.3），但采用私有 mempool + 125ms 短轮次机制，官方声明"由于排序在私有 mempool 中完成后才对外公布，MEV 相关的三明治攻击和前置交易在 Arbitrum One 上过去和现在都不可能发生"。

### 2.3 Arbitrum Timeboost —— 机制、实测中心化数据、2026 年治理最新进展

- **机制**：Express lane 每 60 秒一轮的密封二级价格拍卖（sealed-bid second-price auction），得标者获得该轮"express lane controller"权利，交易被立即排序；非快车道交易被人为延迟（默认 200ms）。**2025-04 正式上线 Arbitrum One（Nova 同步）**【一手：`blog.arbitrum.io/gattaca-titan-timeboost-live-on-arbitrum/`】。
- **实测中心化数据（一手学术论文）**：《The Express Lane to Spam and Centralization: An Empirical Analysis of Arbitrum's Timeboost》，作者 Johnnatan Messias、Christof Ferreira Torres，**arXiv:2509.22143**（2025-09-26 上传，经本轨道直接读取 arXiv 摘要页核实）。分析 **2026年1-4月超 4850 万笔** express lane 交易、**49.4 万次拍卖**，核心发现：①**三个实体赢得 99.74% 的拍卖**（高度中心化）；②快车道交易虽获更早排序，但真正可套利的 MEV 机会集中在区块末尾，限制了优先权的实际价值；③**约 30% 的 time-boosted 交易被 revert**，说明未能有效抑制 spam；④转售 express lane 权利的二级市场因执行可靠性差、经济不可持续而难以为继；⑤拍卖竞争度随时间下降，DAO 收入持续走低。论文原文结论："Timeboost fails to deliver on its stated goals of fairness, decentralization, and spam reduction. Instead, it reinforces collusion and narrows adoption."
- **2026 年治理最新进展（一手：`forum.arbitrum.foundation`，2026-09-07 直接读取原帖）**：Offchain Labs 于 **2026-06-03** 发起 **[Constitutional] AIP: Transition Arbitrum One ordering policy to Priority Gas Auctions (PGA)**（帖子ID 30942）。核心内容：在 Arbitrum One 上禁用 Timeboost、替换为 PGA（125ms 一轮，K=2，250ms 出块，按 priorityFeePerGas 降序排序，配套"反饥饿"虚拟优先级提升机制防止零/低小费交易被无限期搁置）；在 Arbitrum Nova 上直接停用 Timeboost（不引入 PGA，因 Nova 走"极简化"路线）；收益 **97% 归 ArbitrumDAO 国库、3% 归 Developer Guild**，每 6 个月单独 DAO 投票分配。提案原文明确澄清："This AIP...does not introduce new forms of MEV or create MEV...Arbitrum One will continue to have a private mempool at the sequencer...MEV-related sandwich attacks and frontrunning were not and are not possible on Arbitrum One."
  - **治理状态（截至 2026-09-07）**：该提案已于 **2026-08-17** 与"Fast Feed AIP"合并，将作为单一链上宪法投票推进；两者已分别完成 Snapshot temperature check；链上投票（约 14 天，经由 `alt.gov.arbitrum.foundation`）**尚未完成/生效**——**这仍是一个提案，不是既成事实**，必须明确标注。
  - 另有独立窄范围提案《Automate Timeboost Proceeds Split》（帖子 30920）已于 2026-07 被 Offchain Labs 建议放弃（现有系统架构无法支持）。

### 2.4 Unichain verifiable priority ordering —— 补充实测/上线细节

补充 §1.2 未覆盖的细节：Rollup-Boost **于 2025-05-02 起在 Unichain 主网上线**【一手：`theblock.co/news/ecosystems/2025-05-02-unichain-becomes-first-ethereum-layer-2-to-start-processing-transactions-using-flashbots-rollup-boost-tee-tool-353004`，标题直接称"first Ethereum Layer 2"采用该 TEE 工具】。当前"可验证性"主要基于 **TEE 证明模型**（依赖 Intel TDX 硬件可信根），与 ZK 可验证排序存在本质区别；公开 attestation 验证功能的完善程度未见官方给出量化数据，标记⚠️存疑。

### 2.5 批量拍卖 / Frequent Batch Auction（FBA）

| 项目 | 机制 | 生产状态与日期 |
|---|---|---|
| **Injective** | 链上 CLOB 原生采用 FBA 而非连续双向拍卖（CDA）：同一批次时间窗口（对齐约1-2秒区块时间）内的订单被统一收集，匹配引擎计算能实现最大成交量的单一出清价，同批次所有可执行订单以同一价格成交，消除"先到先得"时序优势 | **协议内建**，作为 Injective Chain（Cosmos SDK 应用链）核心交易模块自 **2021-11 主网上线**起长期在线运行【一手：`injective.com/blog/understanding-injective-architecture-and-consensus`】 |
| **CoW Protocol** | 用户以链下签名"意图"提交订单，每约 30 秒批次窗口内独立第三方 solver 竞争寻找最优结算方案（优先 Coincidence of Wants 直接撮合，否则路由至链上流动性），最终以统一批量清算价格结算 | 前身 Gnosis Protocol V2 于 **2021-04-28** 以太坊主网上线（概念验证），2021-08-13 结束 alpha 并品牌重塑为 CoW Protocol，持续在以太坊主网及多链运行至今【一手：`blog.cow.fi/cowswap-turns-1-year-old-b1e7090e47ca`】 |
| **Penumbra** | ZSwap 将同一区块内所有 swap 需求聚合为单一批次，对 CPMM 统一执行；设计目标是通过"流加密"（flow encryption，门限密码学）实现 sealed-bid，批量执行前单笔金额保持加密不可见 | 批量执行（batching）已上线运行；**完整 sealed-bid 隐私（门限解密隐藏单笔金额）在早期主网版本中未完全落地**——⚠️底层共识层（CometBFT）当时缺乏所需 ABCI 2.0 接口，团队将完整 Flow Encryption 列为后续升级目标，具体落地进度未能核实最新状态，标记⚠️存疑 |

### 2.6 Latency-fair ordering 学术方案与不可能性结论

- **Aequitas（奠基论文）**：《Order-Fairness for Byzantine Consensus》，Mahimna Kelkar、Fan Zhang、Steven Goldfeder、Ari Juels，**CRYPTO 2020**【一手：`eprint.iacr.org/2020/269`】。首次形式化定义"交易顺序公平性"（order-fairness）作为拜占庭共识除一致性、活性外的新性质。
- **Themis（改进版）**：《Themis: Fast, Strong Order-Fairness in Byzantine Consensus》，Kelkar、Deb、Long、Juels、Kannan，**ACM CCS 2023**【一手：`eprint.iacr.org/2021/1465`】。指出原始 Aequitas 协议存在活性缺陷，提出"deferred ordering"（延迟排序）技术解决。
- **不可能性结论**（并非单一命名定理，而是 Kelkar 等人在 CRYPTO 2020 论文中同步证明的结果）：由于分布式网络中不同节点因延迟差异形成的"接收顺序偏好"可能构成类似**孔多塞悖论（Condorcet paradox）**的非传递循环（多数节点认为 A 先于 B、多数节点认为 B 先于 C、多数节点认为 C 先于 A），**不存在能对所有交易对同时满足"多数接收顺序"规则的一致全局线性排序**——严格意义上的"完美接收顺序公平"在异步分布式系统中被证明不可能实现。论文因此提出放宽版本 "γ-batch-order-fairness"（对存在循环的交易分组批处理，组内不保证严格顺序，组间保证公平）。
- **对本项目的含义**：这条学术结论解释了为什么工业界（Timeboost、批量拍卖等）普遍采用"放宽版公平性"而非严格排序公平——**任何设计方案如果承诺"完全公平的交易排序"，在理论上就是在承诺一个已被证明不可能的目标**，只能在"批处理粒度"上谈公平。

### 2.7 私有 mempool / 加密 mempool —— Shutter、SUAVE 现状核实

- **Shutter Network**：基于门限密码学的加密 mempool——交易进入 mempool 前被加密，验证者/排序者先承诺排序位置，随后由分布式"Keypers"门限解密并执行，实现"先定序、后解密"以阻止基于 mempool 内容的抢跑。**"Shutterized Gnosis Chain" 已于 2024-07 在 Gnosis Chain 主网正式上线**【一手：`blog.shutter.network/shutterized-gnosis-chain-is-now-live/`】，截至 2026-09 仍持续维护开发。**重要变数**：2026-08 GnosisDAO 通过治理提案 **GIP-153**，决定将 Gnosis Chain 从独立 L1 转型为 ZK-proven 的以太坊经济区（EEZ）rollup，目标创世窗口 2026 年末至 2027 年初，这将影响当前基于 L1 验证者集的 Shutter 集成的长期架构【二手：`crypto.news` 报道，标记⚠️存疑待官方 Gnosis 文档二次确认】。
- **SUAVE（Flashbots）—— 核实结论：作为独立"链"的原始构想事实上已停止推进，⚠️与网络流传的"仍在活跃开发"说法有出入**：GitHub 仓库 `flashbots/suave-geth` 与 `flashbots/suave-specs` 已于 **2025-05-12 被正式归档（archived）并设为只读**，不再接受新提交。Flashbots 未发布"SUAVE 已死"的直白声明，而是通过一系列官方博文披露战略转移：**2024-11 推出 BuilderNet**（基于 TEE 的去中心化区块构建网络），**2024-12 起 Flashbots 停止运营自有的中心化区块构建器**，全面转向 BuilderNet。原 SUAVE 设想的机密计算（TEE）、MEV 去中心化市场等理念被整合进 BuilderNet 及 Rollup-Boost（正是 Unichain 采用的 TEE 排序方案）等产品，而非以独立 SUAVE 链形式落地【一手：`github.com/flashbots/suave-geth` 仓库归档状态、`writings.flashbots.net/migrating-to-buildernet`】。**结论：SUAVE 作为独立链已事实性搁置，K.md 提出的"SUAVE 是否已死"这个疑问，答案是"链的构想已死，技术理念以 BuilderNet/Rollup-Boost 形式续存"**——之前 web_search 合成结果中"SUAVE 仍在活跃开发"的说法主要指 Flashbots Collective 论坛的一般性讨论热度，与"SUAVE 链本身的工程投入"是两回事，本文档以 GitHub 仓库归档这一更硬的一手证据为准。


## 3. application-specific sequencing（ASS）—— 关键概念货架

> ASS 的核心主张：**不开新链，让应用自己在合约/hook 层定义排序与 MEV 分配规则**。本节先给两个生产案例，再给"MEV tax"理论基础，最后**明确切分**"不开链能拿到什么 / 必须改链才能拿到什么"——这是 K.md 明确要求的强制切分。

### 3.1 Sorella Labs — Angstrom（Uniswap v4 hook）

- **机制**【一手：`sorellalabs.xyz/writing/a-new-era-of-defi-with-ass`（2024-10-14）、`docs.angstrom.xyz`】：Angstrom 是构建在 Uniswap v4 hook 上的 ASS DEX，两个核心机制：
  1. **批量拍卖 + 统一出清价**：同一区块内所有经过 Angstrom 池子的 swap 以**同一价格**成交，三明治攻击因此失去意义（无法在批内插队获利）。
  2. **CEX-DEX 套利拍卖**：每个区块开始时，套利者竞价"第一笔无手续费 swap"的权利，中标出价按比例返还给对应价格区间的 LP，用来对冲 LVR（Loss-Versus-Rebalancing）。
  3. **实现方式**：Angstrom 运行一套**独立于以太坊共识的链下验证者网络**，对下一区块的交易集合达成共识，产出"顺序无关"（order-agnostic）的 bundle，只需支付 base fee 即可被任意 builder 塞入区块的任意位置——**这个链下共识网络本身就是一种轻量级的"影子排序层"**，不是纯合约逻辑。
  4. **强制包含**：节点必须广播收到的合法订单，遗漏多数节点已观察到的订单会导致 proposer 被惩罚。
- **主网状态**：**已上线以太坊主网**，时间为 **2025年7月**（经 Cantina 审计），Angstrom 团队为 L1 与 L2（如 Unichain 等有 sequencer 保证 priority ordering 的 OP Stack 链）分别维护两套合约栈——L1 靠链下验证者网络强制排序，L2 直接靠链上 MEV tax 实现同等效果，因为 L2 sequencer 已经提供了可信的 priority ordering【一手：`docs.angstrom.xyz/contracts/`；官方博客未见 Uniswap Labs/Foundation 就 Angstrom 发布联合公告，Angstrom 是第三方团队独立产品】。

### 3.2 Fastlane / Atlas（Monad 与跨链框架）

- **机制**【一手：`github.com/FastLane-Labs/atlas` README】：Atlas 是**链无关**的"执行抽象"智能合约框架，允许任意 dApp/前端/钱包运行自己的订单流拍卖（OFA）：用户签名 UserOperation（意图）→ 前端经 SDK 收集 → 传播给 Solver 竞价提交 SolverOperation → dApp 自定义的 `DAppControl` 合约按 `BidValue`/`AllocateValue` 规则排序执行 → 交易在为每个用户确定性部署的 Execution Environment（EE）内原子执行。**Atlas 本身不改变底层链排序规则**，而是把撮合搬进一次原子交易内完成。
- **Monad 落地形态与通用框架的差异**（重要区分）：在 Monad 生态，Fastlane 实际落地产品是 **shMONAD 流动性质押 + Fastlane MEV sidecar 拍卖**——Monad 验证者运行 Fastlane 的定制化 sidecar 参与区块内 MEV 拍卖，收益经 shMONAD 分配给质押者，这是**验证者侧车方案**，与部署在 Polygon/BSC/Base/Arbitrum/Berachain/Hyperliquid 等链上的通用 Atlas 合约框架（`deployments.json` 中列出的地址）在架构上不同；`deployments.json` **未列出 Monad 的合约地址**，以太坊 L1 字段也为空，说明 Atlas 尚未以标准合约形式部署在以太坊主网。
- **生产状态**：Monad 主网于 **2025-11-24** 上线，shMONAD 及 Fastlane MEV 基础设施同步上线【官方口径，经 scout 交叉核实；`dev.shmonad.xyz` 页面仍有残留"目前在 Monad 测试网上线"字样，判断为文案未及时更新，标记⚠️存疑待人工复核该页面时效性】。2026年1月，Chainlink 收购 Fastlane 的 Atlas，用于扩展其 Smart Value Recapture（SVR）产品线【一手：`prnewswire.com` 官方收购公告】。

### 3.3 MEV tax / "Priority Is All You Need"（Robinson & White，Paradigm，2024-06-04）

- **一手来源**：`paradigm.xyz/writing/priority-is-all-you-need`
- **核心机制**：在遵循"按 priority fee 降序排列"的链上，MEV 竞争会被竞争到只剩 priority fee 一种表现形式。论文提出：应用合约无需理解自己的业务逻辑产生了多少 MEV，只需把自己收取的手续费定义为 priority fee 的递增函数（`applicationFee = y × priorityFeePerGas`），就能"劫持"这个已经存在的竞价机制，把其中绝大部分价值收归应用自己。数值例子：MEV=100、税率 y=99 时，理性搜索者只会把 priority fee 设为 1（付给出块者），另外 99 转付给应用合约——**应用捕获 99% 的 MEV**，这就是 "MEV tax"。
- **论文明确写出的前提条件（原文摘录）**：
  > "MEV taxes only work if block proposers strictly follow the rules of competitive priority ordering, which include sorting transactions by priority fee without censoring, peeking at, or delaying any. If block proposers deviate from those rules, they can evade MEV taxes to capture the value for themselves. **Today, therefore, MEV taxes depend on trusting L2 sequencers, and would likely not work at all on Ethereum L1**, where block building is dominated by a competitive builder auction that maximizes revenue for the proposer."

  论文进一步把 "competitive priority ordering" 拆成四条规则：**(1) priority ordering**（区块内必须按 priorityFeePerGas 降序排列）、**(2) censorship-resistance**（收到的合法交易必须包含）、**(3) pre-transaction privacy**（提议者必须通过私有端点接收交易，且在承诺区块前不得向任何人泄露）、**(4) no last look**（必须设定一个明确的 `blockTime`，之后不得再接受任何人的交易）。论文自己承认：
  > "While the first property would be easy to enforce at the protocol layer, enforcing the other properties trustlessly is an open problem."
- **这个前提在 OP Stack 上成立吗？** ——**部分成立，取决于具体链的 sequencer 实现，不能一概而论**：
  - OP Stack 的标准/vanilla 排序模式确实是"按到达时间 + priority fee"的简单规则，理论上满足 (1)；但 (2)(3)(4) 三条本质是对**单一中心化 sequencer 是否诚实**的信任假设，OP Stack 协议层**没有内建机制强制验证**这三条规则被遵守——这与 Unichain 用 TEE attestation 补强 (1)(3) 的思路（见 §1.2）正是同一个问题的两种应对：**要么信任单一 sequencer 的商誉/合约条款，要么用 TEE 把信任面从"运营方承诺"降级为"硬件完整性"**。
  - Mantle 目前**没有** TEE block builder、没有 VPO，因此若在 Mantle 上部署 MEV tax 机制，其有效性**完全依赖于对 Mantle 唯一 sequencer 诚实性的信任**——这与依赖中心化交易所诚实性本质相同，是一个需要在设计方案中显式写出的风险点。

### 3.4 「app 不开新链，只靠 app 级排序」的可行边界（本节核心回答）

基于上述三个案例及一手文献交叉分析，可行边界如下：

**不开链也能拿到的部分**：
1. **MEV 内部化/再分配**——只要底层 sequencer 诚实执行 priority ordering，应用合约可以用 MEV tax 把本该被 sequencer/搜索者拿走的价值重新分配给协议/用户/LP（Angstrom 对 LP 的 LVR 返还就是范例）。
2. **批量出清价格 / 反三明治**——通过 hook（如 Uniswap v4）在合约层面把同区块内的多笔交易撮合为统一出清价，不需要改变链的排序协议。
3. **应用自定义的 OFA / solver 竞价流程**（Atlas 模式）——把"谁能执行这笔交易"的竞价过程完全放进合约与链下 relay 网络，不依赖底层链的排序器改造。
4. **对 Launchpad 而言的直接映射**：发行合约完全可以自己内置一个"MEV tax + 批量拍卖"机制，对认购/铸造交易做批量出清或对 bot 抢跑收取递增税率——这些都**不需要 Mantle 改一行 op-node 代码**。

**必须改链才能拿到的部分**：
1. **抗审查保证（censorship-resistance）**——纯链上/合约手段无法验证 sequencer 是否偷偷审查了某笔低 tip 交易，这需要协议层机制（如 FOCIL、去中心化排序、TEE attestation）。
2. **预交易隐私（pre-transaction privacy）与"no last look"**——同样需要 sequencer 侧的架构改造（加密 mempool、TEE、或去中心化定序），应用合约层面看不到、也管不了 sequencer 收到交易后到打包前做了什么。
3. **跨应用 composability 时的一致性**——Sorella 自己的分析承认：ASS 要求交易"顺序无关"才能被任意 builder 插入任意位置，一旦与外部非 ASS 合约组合，一笔 revert 可能连带拖垮整个 bundle，目前只有半成熟方案（inclusion preconfirmation、共享 app-specific sequencer）缓解，这些方案本质上又变成了在应用之上再搭一层轻量"专属排序层"，退回到"事实上的应用专属 L2"路径。
4. **全局费用市场下的 inclusion 博弈**——MEV tax 会把赢家的 priority fee 压到接近 0，在一个有其他应用竞争区块空间的全局费用市场里，这类"0 tip"交易可能在拥挤时被挤出；只有本地化/隔离的费用市场（见 §4）才能让这个问题不存在。**但对 Mantle 而言，当前区块填充率仅 0.173%，全局费用市场竞争约等于不存在，这个"必须改链"的理由在 Mantle 现状下反而不成立**——这是本节对"投入产出比"判断最关键的一条依据，呼应 §7。

**一句话结论**：**Launchpad 场景要解决的"发行时刻抢跑/公平分配"问题，绝大部分可以靠 app 级机制（批量拍卖 + MEV tax + 自定义 solver 竞价）解决，且不需要碰 Mantle 的 op-node**；唯一必须依赖链方配合的，是"sequencer 是否诚实执行 priority ordering"这一条最基础的信任假设——而这条假设在**任何**中心化 sequencer 架构上（包括 Mantle 现状）都只能靠信任，无法用合约验证，这正是 Mantle "完全没有 pre-confirmation / TEE / VPO" 这个第一阶段发现在本轨道的延伸推论。


## 4. 区块空间与拥塞隔离


> 本节盘点"区块空间怎么分""拥塞怎么隔离"的机制。**特别提醒**：鉴于 Mantle 当前区块填充率仅 0.173%（吞吐远未饱和），本节大部分"抗拥塞"机制在 Mantle 现状下**没有直接应用场景**，收录目的是做完整的技术货架，具体取舍见 §7。

### 4.1 Solana local fee market（SIMD-0286 后现状）

- **机制**【一手：`github.com/solana-foundation/solana-improvement-documents` SIMD-0286 原文、`solana.com/upgrades/100m-cu-blocks`】：Solana 费用 = 固定 base fee（5000 lamports/签名）+ priority fee（microlamports/CU）。"Local"体现在 CU 上限按**写锁账户粒度**设定，不同账户的拥堵互不影响费率，这是与 EIP-1559 全局 base fee 的本质区别。
- **SIMD-0286 具体参数变化**：标题"Increase Block Limits to 100M CUs"，作者 Lucas Bruder（Jito Labs），创建于 2025-05-20。**只改动了 Max Block Units 这一个参数**：50M → 60M（SIMD-0256，2025-07 主网生效）→ **100M**（SIMD-0286）。**Per-writable-account 的 CU 上限（Max Writable Account Units）保持不变，始终是 12,000,000（12M）CU**——K.md 要求的"确切数字"即为此。Max Vote Units（36M）、Max Block Accounts Data Size Delta（100MB）全程不变。
- **上线状态**：`solana.com/upgrades` 页面标注 **"Live on Mainnet"**，主网激活时间 **2026-07-29（epoch 1009）**，feature gate 为 `P1BCUMpAC7V2GRBRiJCNUgpMyWZhoqt3LKo712ePqsz`；Testnet 先于 epoch 983、Devnet 于 epoch 1100 激活。⚠️注意：GitHub 提案 markdown 文件头元数据仍标注 `status: Review`，与官方页面"Live on Mainnet"不同步，以 `solana.com/upgrades` 页面为准。
- **历史教训**：2023 年底曾有"Local fee markets are a lie"的业内吐槽（Ben Coverston），因早期调度器仅按到达时间排序，直到 2024-05 Agave v1.18 引入 central scheduler（prio-graph）后才显著改善——**这提示"本地费用市场"这个概念本身不是一次性设计就能生效，需要配套的调度器实现**。

### 4.2 EIP-1559 全局 base fee 的失效场景

【一手：`eips.ethereum.org/EIPS/eip-1559`；`ethereum.github.io/abm1559`（EF 官方 agent-based 模拟工具）；`ethresear.ch/t/who-pays-for-congestion-optimal-design-of-protocol-fees/10174`】

1. **弹性区块 + 滞后效应**：目标 15M gas、硬上限 30M gas。单笔巨额 gas 交易可以被塞进同一区块（只要不超硬上限）——EIP-1559 无法阻止单笔巨额交易本身挤占区块，只能**滞后一个区块**抑制未来同类行为，本质是经济外部性传导，不是实时阻断。
2. **Base fee 钉在 floor 值（低拥堵场景失效，对 Mantle 直接相关）**：持续低于 50% 目标利用率时，base fee 每区块最多下降 12.5%，会持续下滑直至协议 floor。此时 **priority fee 成为事实上唯一的拥堵定价信号**——若需求瞬间从极低跳升至突发峰值（NFT/Launchpad mint 秒杀、清算潮），base fee 因滞后一个区块无法即时反应，用户必须依赖 priority tip 竞价，机制退化为类似 EIP-1559 之前的 first-price auction。**这正是 Mantle 现状**（base fee 常年钉在 50 gwei 下限）——K.md 给出的约束条件在这里得到了理论解释：Mantle 目前处于"EIP-1559 数学上必然钉底"的状态，任何期待 base fee 机制自己去调节 Launchpad mint 峰值拥堵的设计假设都不成立。
3. **佐证**：EIP-7623（已在 Pectra 实现，状态 Final）引入 calldata floor price 机制，防止高 calldata/低执行 gas 交易规避费用约束——这从另一个角度印证"单一维度 gas 计价无法覆盖所有拥堵场景"是以太坊核心开发者的共识判断。

### 4.3 多维 gas / EIP-7706 现状

- **一手来源**：`eips.ethereum.org/EIPS/eip-7706`（2026-09-07 直接读取）。标题 "EIP-7706: Separate gas type for calldata"，作者 Vitalik Buterin，创建日期 **2024-05-13**。
- **当前状态**：**🚧 Stagnant**（Standards Track: Core）——不是 Draft，也未进入 Review/Last Call/Final，是**已经停滞的提案**，无测试网实现记录。依赖 EIP-1559、EIP-4844。
- **核心内容**：引入新交易类型，`max_fees_per_gas`/`priority_fees_per_gas` 变为长度 3 的向量（execution / blob / calldata gas），区块头新增 `gas_limits`/`gas_used`/`excess_gas` 向量字段替代原单一标量。
- **为何停滞**：Vitalik 在提出后不久即公开表示该提案过于复杂，不太可能纳入 Pectra，转而支持更简单的 EIP-7623 作为短期方案——**这是一条应显式写出的"预期特性其实不存在"发现**：多维 gas 这个方向在以太坊 L1 层面**事实上已被搁置**，目前没有正式推进的时间表。

### 4.4 Per-contract fee market —— 现有提案与实现核查结论

**结论：目前没有在以太坊主网路线图上被正式采纳或积极推进的"per-contract"费用市场提案**。现有研究整体转向"multidimensional"（按资源类型如计算/存储/calldata，而非按合约地址）定价方向，主线是 Vitalik 的 "Multidimensional EIP-1559"（`ethresear.ch/t/multidimensional-eip-1559/11651`），后续演化出 EIP-7999。**未发现任何 L2 已真正实现"per-contract 独立费用市场"的生产案例**——技术障碍包括多合约原子交互难以拆分定价、gas 需求内生性导致单合约"正确价格"难以孤立计算、增加节点执行复杂度。文献常引用 Solana 的按账户（而非按合约）本地费用市场（见 4.1）作为最接近"per-资源定价"理念的生产系统对照。**按 K.md 的判断标准，这条"检索无果因此按不存在处理"**。

### 4.5 预留 lane / 分区区块空间的生产案例 —— Tempo 与 Plasma（K.md 重点核查项）

> K.md 明确要求"务必查清是协议级内建还是 paymaster/relayer 级"——以下是逐项核实结论。

**Tempo payments lane —— 判定：协议级，直接内建于区块头结构，非应用层/relayer 排队**

- **一手证据**（直接读取协议文档源码级页面 `tempo.xyz/developers/docs/protocol/blockspace/overview`）：Tempo 区块头在标准以太坊区块头基础上**新增字段**：
  ```
  pub struct Header {
      pub general_gas_limit: u64,
      pub shared_gas_limit: u64,
      pub timestamp_millis_part: u64,
      pub inner: Header,
  }
  ```
  官方原文："Tempo blocks extend the Ethereum block format in multiple ways: there are new header fields to account for payment lanes and shared gas accounting..."
- **机制**：`general_gas_limit` 与 `shared_gas_limit` 直接把以太坊原生 `gas_limit` 字段切分为**支付类容量**与**非支付类容量**两个独立预算池，这是**区块头结构体层面的原生字段**，不是链下 relayer 或合约层的排队逻辑。区块体交易排序被协议规定为四段式：①区块起始系统交易 → ②Proposer lane 交易（受 `general_gas_limit` 约束，非支付类）→ ③消耗共享 gas 预算的剩余交易 → ④协议规定的区块结束系统交易。节点验证规则明确写"A valid Tempo block must include..."，说明这是**共识层区块合法性验证规则的一部分**。
- **生产状态**：Tempo 主网（Chain ID 4217）已于 **2026-03-18** 上线【一手：`tempo.xyz/blog/introducing-tempo/`】，定位为 "payments-first Layer 1 blockchain incubated by Stripe and Paradigm"。
- **结论**：**协议级内建**，是本文档中"预留区块空间"最强的生产实证——直接修改区块头格式做资源隔离，而非在应用层排队。

**Plasma zero-fee USDT 转账路径 —— 判定：见 §5.1 详细拆分**（协议原生 paymaster 赞助 USDT 转账，但"预留 payment lane"式的区块空间隔离机制本身，官方文档 `architecture/system-overview` 页面把它列为 **Core Protocol roadmap**（"dedicated payment lanes...being extended"），**尚未上线**，与 Tempo 已经生产落地的区块头级隔离不是同一个成熟度）。**因此在"分区区块空间"这个维度上，Plasma 目前只有 gas 费用赞助（见 §5.1），没有 Tempo 那种协议级 lane 隔离**——这个区分容易被市面文章混淆，本文档明确切割。

### 4.6 Rate limiting / 抗 bot 洪水的链级手段

| 手段 | 机制 | 生产案例 |
|---|---|---|
| Stake-Weighted QoS（SWQoS） | 区块 leader 按验证者质押比例分配交易转发带宽，专业团队通过 staked connections 绕开公共 mempool 直连当前 leader 的 TPU | **Solana**，生产环境长期存在 |
| Reference Gas Price（RGP） | 每个 epoch（约24小时）验证者提交最低报价，协议取按质押加权 2/3 分位数作为全网 RGP，"tallying rule" 惩罚未及时处理交易的验证者 | **Sui**，生产环境 |
| Block-STM 并行执行 | 非直接限流手段，通过并行事务执行（冲突检测+局部重跑）提高吞吐稀释拥堵影响，属架构层缓解而非经济限流 | **Aptos**，生产环境 |
| 应用层 "Bot Tax" | 对失败/无效交易收取小额惩罚性费用，drain mint sniper bot 资金 | **Solana Metaplex Candy Machine**，生产案例（2022年上线），`developers.metaplex.com/candy-machine/guards/bot-tax` |
| 资源质押模式 | 用户质押代币获得带宽（NET）/CPU 资源配额，超出配额交易被拒绝直至追加质押 | **EOS**，协议级账户级资源速率限制 |
| 隐式限流 | EIP-1559 式 base fee 上涨是经济性隐式限流；RPC 网关层 Token Bucket 限流（按账户 key 而非 IP）是基础设施层标配 | 通用实践 |

**⚠️未找到**严格意义上"链级 CAPTCHA-like PoW 共识机制"的生产案例——生产环境更常见经济手段（质押配额、动态 gas price、应用层 bot tax）与基础设施层限流组合，如实标注为未找到而非编造案例。**对 Launchpad 场景最直接可用的是"应用层 Bot Tax"这一条**（Metaplex Candy Machine 范式），因为它不需要改链，只需要在铸造合约里加惩罚性 revert 费用逻辑。


## 5. gas 抽象货架

> 本节的分析路径来自 §3 类似的方法论：先确认"协议级"与"应用/第三方层级"的严格区分，很多被媒体描述为"协议内建"的方案，一手源核查后其实是标准 ERC-4337 + 第三方基础设施。这个区分本身是本节的重要发现。

### 5.1 协议级 paymaster：Tempo 与 Plasma 的真实实现层级（重要发现）

- **Tempo —— 确认为真正的协议原生方案**【一手：`github.com/tempoxyz/tempo` README、`tempo.xyz/developers/docs/quickstart/predeployed-contracts`、`docs.tempo.xyz/protocol/fees/fee-amm`】：Tempo（Stripe/Paradigm 投资的支付链，基于 Paradigm 的 Reth SDK 自建执行客户端）在协议层预部署了固定地址 **`0xfeec...0000`** 的 **Fee AMM** 系统合约，用户直接以 USD 稳定币支付 gas，Fee AMM 自动兑换成验证者偏好的稳定币结算。其账户模型"Tempo Transactions"是原生智能账户，内置 fee sponsorship、批量支付、被动授权、passkey/WebAuthn 认证，**不经过 ERC-4337 EntryPoint**——这是一套独立于以太坊标准的原生 AA 设计，写入协议规范（TIP-20/TIP-403 等）。
- **Plasma —— 需要拆成两个不同功能分别判断，不能一概而论**【一手：`docs.plasma.org/docs/plasma-chain/network-information/network-fees`，2026-09-07 直接读取原文】：
  1. **"USDT 零手续费转账"（已上线，协议原生）**：Plasma 主网自 **2025-09-25** 上线起即支持标准 USDT `transfer()` 零手续费，由 Plasma **协议自身维护**的原生 Paymaster 合约赞助，官方文档原文区分"external paymasters"（第三方、通常收费）与"Plasma's gas token paymaster"（协议维护、不收费）——这一条**判定为协议级原生**，不是第三方 relayer 代付【二手来源交叉：`okx.com/learn/plasma-defi-blockchain-zero-fee-usdt` 等多篇文章确认该功能自主网上线起即生产可用；一手 `docs.plasma.org` 未直接标注具体上线日期，日期来自主网上线日期的合理推断，标记轻微⚠️不确定性】。
  2. **"任意自定义 gas token"（仍是路线图，未上线）**：`network-fees` 页面开篇即写"This page explains how transaction fees work on Plasma network, **including the roadmap for paying gas in custom tokens**"，并用将来时明确写"Plasma **is building** support for custom gas tokens via a native paymaster contract. Developers **will be** able to register whitelisted ERC-20 tokens..."——**这是尚未交付的路线图功能，不能与已上线的 USDT 零手续费功能混为一谈**。
  3. `tools/account-abstraction` 页面另外列出 Gnosis Safe、Gelato Relay、Thirdweb、ZeroDev 等**第三方** AA/paymaster 服务商，用于一般性 dApp gasless 场景（例如某个 DeFi 协议想给自己用户免 gas），这是与上述两条协议原生机制**并行的、独立的第三方可选集成**，不应混淆为"Plasma 的零手续费机制主要靠第三方"——这是本轨道对市面上大量二手文章的一处重要澄清。
  - **结论（回答 K.md 的追问）**：Plasma 的 USDT 零手续费转账 = **协议级原生**（不是 paymaster/relayer 代付的第三方模式）；但"用任意 token 付 gas"这个更通用的自定义 gas token 能力 = **仍是路线图**，当前该场景下应用若想有其他 token 的 gasless 体验，只能依赖第三方 AA 基础设施。两者必须分开陈述，笼统说"Plasma 协议内建 gas 抽象"或"Plasma 全靠第三方 paymaster"都不准确。


### 5.2 ERC-4337 vs EIP-7702 vs 原生 AA（zkSync Era / Starknet）

| 维度 | ERC-4337 | EIP-7702 | zkSync Era 原生 AA | Starknet 原生 AA |
|---|---|---|---|---|
| 机制 | 应用层：`UserOperation` 走独立 mempool，由 Bundler 打包调用单例 `EntryPoint.handleOps()`；不需要以太坊共识层改动 | 协议层：新增交易类型 `0x04`（SET_CODE_TX_TYPE），EOA 可通过授权元组把自己的 code 设为对某合约地址的委托指针（`0xef0100‖address`），持久化直到被覆盖/清除 | 协议/系统合约层：所有账户在设计上都是可编程合约账户，通过 Bootloader + 系统合约实现原生 Paymaster、账户版本、nonce ordering，无需 EntryPoint | 协议层：Starknet 从设计上没有 EOA 概念，每个账户天生是实现了 `__validate__`/`__execute__` 的合约（推荐遵循 SNIP-6） |
| 生产状态与日期 | **Final**。EntryPoint v0.6 于 2023-03 上线以太坊主网并成为事实标准，现行 v0.7/v0.8 广泛部署于主网及各 L2 | **Final**，随 **Pectra 硬分叉于 2025-05-07** 在以太坊主网正式激活 | 自 **2023-03** zkSync Era 主网上线起即为核心架构 | 自 Starknet 主网上线起即为原生设计 |
| 一手来源 | `eips.ethereum.org/EIPS/eip-4337`（状态：Final，Standards Track: ERC；Requires EIP-712, EIP-7702） | `eips.ethereum.org/EIPS/eip-7702`（状态：Final，Standards Track: Core；Created 2024-05-07） | `docs.zksync.io/zksync-protocol/era-vm/account-abstraction` | `docs.starknet.io/learn/protocol/accounts` |

补充：ERC-4337 v0.8 的 `UserOperation.factory` 字段可设为 `0x7702` 标志，表示 sender 是一个通过 EIP-7702 委托了代码的 EOA——二者在最新版本已经互通，并非互斥关系。zkSync/Starknet 的"原生 AA"本质区别于 ERC-4337：不存在 EntryPoint 单例合约与独立 mempool，账户抽象规则被内嵌进协议的交易验证流程本身。

### 5.3 Custom gas token：OP Stack 现状与 Mantle 的真实技术谱系（重要发现）

- **OP Stack CGT 版本历史**【一手：`docs.optimism.io/op-stack/features/custom-gas-token`、`docs.optimism.io/notices/archive/upgrade-18`】：
  - **CGT v1（Legacy/Beta）**：早期实验特性，链可锚定到 L1 上的一个 ERC-20 作为原生 gas 货币，通过系统交易铸造/销毁。**2025年2月左右 OP Labs 宣布停用该 beta 特性**（生态使用率低）。
  - **CGT v2（现行、生产可用）**：随 **Upgrade 18**（`op-contracts/v6.0.0`）重新设计发布，是与 v1 完全独立的新架构，核心是两个新预部署合约 `NativeAssetLiquidity`（持有创世预铸造资产）与 `LiquidityController`（治理控制的铸造/销毁权限）。**关键限制**：桥接/兑换逻辑完全下沉到应用层（"No bridge or token is enshrined in the protocol"）；**仅支持 18 位小数代币**；v1→v2 **暂无迁移路径**，需协调硬分叉。
- **Mantle 的 MNT gas 机制——技术谱系判定（重要发现）**：Mantle 主网上线于 **2023-07-17**，采用 OP Stack Bedrock 分支并做大量自定义 fork；而官方 CGT 特性（无论 v1 beta 还是 v2）都**晚于** Mantle 上线时间才出现或成熟（v2 随 2025 年 Upgrade 18 发布）。因此可判定：**Mantle 对 MNT-as-gas 的支持是一套独立于官方 CGT 框架的定制化 fork实现**，而非复用 OP Labs 官方 `isCustomGasToken`/`NativeAssetLiquidity`/`LiquidityController` 预部署方案【一手：`github.com/mantlenetworkio/mantle-v2` README 原文："Mantle Network introduces a new DA scheme, EigenDA... Another significant enhancement involves the adoption of \$MNT as the native token for Mantle Network, departing from the more common choice of \$ETH in OP Stack implementations."】。**概念上两者殊途同归（用某 ERC20/原生资产替代 ETH 做 gas），但代码实现谱系完全不同**——对委托方的实际含义是：**任何"参考官方 CGT v2 文档去改 Mantle gas 逻辑"的设计假设都是错的，必须读 Mantle 自己的 fork 代码**（`packages/contracts-bedrock`、`gas-oracle`）。

### 5.4 「用应用代币付 gas」的其他生产实现

| 案例 | 类型 | 机制 | 状态 |
|---|---|---|---|
| X Layer（OKX，Polygon CDK） | 协议级自定义 gas token | 部署时配置 `gasTokenAddress` 参数，用 $OKB 作为链原生 gas 资产，是 Polygon CDK 官方支持的部署选项 | 生产环境【一手：CDK 部署配置项】 |
| DFK Chain（Avalanche Subnet） | 协议级 Subnet 原生配置 | JEWEL 代币作为 Subnet 原生 gas | 生产中，但**已计划于 2026-08 停运**，回迁 Avalanche C-Chain |
| Swimmer Network（Avalanche Subnet） | 协议级 Subnet 原生配置 | CRA 系代币作为原生 gas | 生产环境 |
| Ronin（Axie Infinity） | AA + paymaster 赞助 | 原生 gas 仍为 RON，游戏开发者可通过 AA/paymaster 让用户"体验上"用游戏代币付 gas | 生产环境，非协议级替换 |
| Beam（游戏 L1） | AA + paymaster 赞助 | 原生 gas 固定为 BEAM，SDK 提供 meta-transaction/gas sponsorship 免持币游玩 | 生产环境，非协议级替换 |

**结论**：Avalanche Subnet 架构（DFK、Swimmer）与 Polygon CDK（X Layer）代表"协议级自定义 gas token"在生产环境的两条不同技术路线；Ronin/Beam 代表更轻量的"AA+paymaster 赞助"路线，改动面远小于协议级方案，这个区分直接对应 §6 总表的"实现难度"列。


## 6. 零件目录总表（核心产出）

> 实现难度四档：`无需改客户端`（应用/合约层即可）/ `需改 op-node 或 builder`（sequencer/builder 侧扩展，不动 EVM 语义）/ `需改 EVM 执行层`（协议层交易类型、gas 计价、账户模型改动）/ `需换栈`（脱离 OP Stack 才能实现）。**这张表是最终设计方案的直接输入。**

| # | 零件 | 解决什么问题 | 生产案例（+上线日期） | 实测效果 | 已知代价 | 在 OP Stack 上的实现难度 |
|---|---|---|---|---|---|---|
| 1 | Base Flashblocks（200ms 软确认） | 把用户可感知确认延迟从 2s 压到 200ms | Base 主网，当前生产架构【一手】 | 排序在 Flashblock 内锁定，无量化大规模回滚报告 | 无独立密码学证明，纯运营承诺；Denim 提案（升级为原生规范区块）仍处 Planning，⚠️未锁定时间表 | 需改 op-node 或 builder（`base-builder`+rollup-boost sidecar） |
| 2 | Unichain Flashblocks + Verifiable Priority Ordering | 在软确认基础上把排序规则做成密码学可验证 | Unichain 主网，Rollup-Boost 于 2025-05-02 上线【一手】 | "首个 TEE block builder L2"（二手表述）；量化延迟分布未公开 | 依赖 Intel TDX 硬件可信根，非无信任 | 需改 op-node 或 builder（TEE sidecar，Mantle 需自建或接入 rollup-boost） |
| 3 | MegaETH mini-blocks（~10ms） | 把执行结果可见性压到 10ms 级 | MegaETH 主网 2026-02-09 上线【一手/二手交叉】 | 需要专用 Realtime API，非标准 JSON-RPC | 单一中心化 sequencer 是架构前提，非可选项 | 需换栈（SALT 内存架构、双层出块非 OP Stack 语义） |
| 4 | Solana Alpenglow（Votor 共识） | 把共识层最终确认从 12.8s 压到 ~150ms | ⚠️**开发中，未上线主网**，目标 Q3 2026【一手】 | 尚无生产实测数据 | 是 Breaking Change，需索引器/客户端适配 | 需换栈（非 EVM/OP Stack 语境，仅供概念参考） |
| 5 | **RISE Shreds（逐笔预确认）** | 把单笔确认延迟压到毫秒级，并提供单往返回执 | **RISE 主网已上线**（chainId 4153）【实测】 | 【实测】1 tx/shred、~88 shred/1s 区块、`eth_subscribe("shreds")` 与 `eth_sendRawTransactionSync` 主网可用 | 需自研执行层；预确认无经济担保；**并未解决抢跑公平**（RISEx 仍需在应用层加 latency bump 保护做市商） | **需改 EVM 执行层 + P2P**（但**仍在 OP Stack 内**——RISE 是标准 OP Stack rollup 换执行层客户端，非换栈）→ 详见 H 轨道 |
| 6 | FCFS 排序 | 消除 priority fee 竞价，用到达时间定序 | Arbitrum One/Nova 历史默认；Robinhood Chain 现行【一手】 | 催生"延迟竞赛"（infra 军备竞赛）；Robinhood Chain 2026-09-04 发生13分钟中断，暴露单点风险 | 无法体现用户对优先权的付费意愿；单一 sequencer 无热备份时可用性差 | 无需改客户端（Mantle 当前 op-node 默认即近似 FCFS+简单 priority） |
| 7 | Priority Gas Auction（PGA，公开 mempool） | 用出价竞争排序权 | 以太坊 L1 历史模式；Arbitrum 拟于 2026 年重新引入（私有 mempool+125ms 轮次） | 历史上导致 spam、三明治攻击（公开 mempool 场景） | 若配合私有 mempool 可消除三明治攻击（Arbitrum 官方声明） | 需改 op-node 或 builder（私有 mempool + 轮次调度逻辑） |
| 8 | Arbitrum Timeboost（express lane 拍卖） | 内部化 MEV、生成 DAO 收入 | Arbitrum One/Nova，2025-04 上线【一手】 | **3 个实体赢得 99.74% 的拍卖**（arXiv:2509.22143）；约 30% 交易 revert；DAO 收入从约 746 万美元/年降至约 200 万美元/年 | 强中心化倾向；2026-06 已发起 AIP 拟停用（尚未生效，⚠️提案阶段） | 需改 op-node 或 builder（独立拍卖合约+排序器改造） |
| 9 | 批量拍卖 / Frequent Batch Auction | 消除批内时序优势，统一出清价反三明治 | Injective（协议内建，2021-11 主网起）；CoW Protocol（2021-04-28 起）；Penumbra（部分功能） | Injective/CoW 长期稳定生产运行；Penumbra 的 sealed-bid 隐私未完全落地⚠️ | 需要批次窗口，牺牲一定即时性 | 无需改客户端（合约/应用层可实现，见 Angstrom 案例） |
| 10 | Latency-fair ordering（Themis/Aequitas） | 定义并逼近"交易顺序公平性" | 学术方案，CRYPTO 2020 / ACM CCS 2023，⚠️未见主流生产链采用其严格形式 | 证明了"完美接收顺序公平"在异步系统中**不可能**实现 | 只能做"批处理粒度"的放宽版公平性 | 需改 EVM 执行层（若要落地需要新排序原语） |
| 11 | 加密 mempool（Shutter Network） | 阻止基于 mempool 内容的抢跑 | Shutterized Gnosis Chain，2024-07 上线【一手】 | 抗抢跑效果依赖 Keypers 门限解密的活性 | 引入额外的门限密码学信任假设与延迟；Gnosis Chain 本身 2026-08 起转型为 rollup（GIP-153），⚠️架构变数 | 需改 op-node 或 builder（mempool 加密+门限解密基础设施） |
| 12 | SUAVE / BuilderNet（Flashbots） | 去中心化区块构建、MEV 民主化 | ⚠️SUAVE 作为独立链**已事实性搁置**（GitHub 仓库 2025-05-12 归档只读）；理念延续至 BuilderNet（2024-11 上线）与 Rollup-Boost | 无独立 SUAVE 链生产数据；BuilderNet 数据未在本轮核实范围 | 战略转向本身说明"独立 MEV 链"路线遇到现实阻力 | 需换栈（若走 SUAVE 原始构想）/ 需改 op-node（若走 BuilderNet/Rollup-Boost 路线） |
| 13 | Angstrom（Sorella，ASS） | 在 Uniswap v4 hook 内做批量拍卖，反三明治+返还 LVR | 以太坊主网 2025-07 上线【一手】 | 依赖独立链下验证者网络做排序共识，非纯合约方案 | composability 困境：与非 ASS 合约组合时 revert 可能拖垮整个 bundle | 无需改客户端（Launchpad 可直接借鉴合约设计模式） |
| 14 | Fastlane / Atlas（app 级 OFA） | 应用自定义订单流拍卖，MEV 分配给应用而非出块者 | 通用合约框架多链部署；Monad 定制化 sidecar，2025-11-24 随 Monad 主网上线 | Monad 上是验证者侧车方案而非通用合约直接复用 | 比基础设施层 MEV 捕获耗费更多 gas/blockspace（官方 README 自陈） | 无需改客户端（合约框架可直接部署到 Mantle） |
| 15 | MEV tax（"Priority Is All You Need"） | 用递增税率函数把 MEV 从出块者手中夺回给应用 | Paradigm 研究文章，2024-06-04，多链已有实践尝试 | 理论极限捕获 99%+ MEV，前提是 sequencer 诚实执行 priority ordering | 论文自承：抗审查/隐私/no-last-look 三条规则"trustless 强制执行仍是 open problem"，在以太坊 L1 大概率无效 | 无需改客户端（纯合约逻辑，Mantle 现状唯一门槛是信任 sequencer） |
| 16 | Solana local fee market（SIMD-0286） | 按写锁账户隔离拥堵，避免全局费用互相挤压 | Solana 主网，2026-07-29（epoch 1009）上线【一手】 | Max Block Units 提升到 100M，per-account 上限维持 12M CU 不变 | 需要配套调度器实现（历史上"local fee market are a lie"教训） | 需换栈（非 EVM gas 模型） |
| 17 | EIP-7706（多维 gas） | 给 calldata 独立的 gas 维度定价 | ⚠️**Stagnant**，未进入 Draft 以后阶段，无测试网实现【一手】 | 无生产数据 | 提案本身被作者认为过于复杂，已让位给更简单的 EIP-7623 | 需改 EVM 执行层（若未来复活） |
| 18 | Per-contract fee market | 按合约地址定价拥堵 | ⚠️**未发现任何生产实现或积极推进的提案**，检索无果按不存在处理 | 不适用 | 技术障碍：原子交互难拆分定价、需求内生性 | 需改 EVM 执行层（假设性） |
| 19 | Tempo payments lane | 协议级预留支付类区块空间，隔离于其他交易 | Tempo 主网（Chain ID 4217），2026-03-18 上线【一手】 | 区块头层面原生隔离，非应用层排队 | 需要修改区块头格式，属于新链级设计，无法平移到现有链 | 需换栈（区块头结构改动超出 OP Stack 兼容范围） |
| 20 | Plasma 协议原生 USDT 零手续费 paymaster | 让用户免持原生代币即可转账指定稳定币 | Plasma 主网，2025-09-25 上线起生产可用 | 协议自身维护、不收费，区别于第三方 paymaster | 仅覆盖标准 `transfer()`/`transferFrom()`，"自定义任意 gas token"仍是路线图⚠️ | 无需改客户端（可用 ERC-4337 paymaster 合约模式在 Mantle 复刻） |
| 21 | ERC-4337（Account Abstraction） | 应用层账户抽象，不改共识层 | 以太坊主网及各 L2，EntryPoint v0.6 于 2023-03 上线，现行 v0.7/v0.8 广泛部署【一手】 | 生态成熟，Bundler/Paymaster 工具链完善 | 需要独立 mempool 基础设施（Bundler），增加架构复杂度 | 无需改客户端（Mantle 已是 EVM 兼容链，可直接部署标准 EntryPoint） |
| 22 | EIP-7702（EOA 委托） | 让 EOA 临时/持久获得合约账户能力 | 以太坊主网，随 Pectra 于 2025-05-07 激活【一手】 | 与 ERC-4337 v0.8 已实现互通（`factory=0x7702`） | 协议层改动，需要客户端支持新交易类型 `0x04` | 需改 EVM 执行层（Mantle 需随 OP Stack 上游同步该 EIP 支持） |
| 23 | 原生 AA（zkSync Era / Starknet 模式） | 从协议设计上取消 EOA/合约二元区分 | zkSync Era 主网自 2023-03 起；Starknet 主网起即为原生设计【一手】 | 无需 EntryPoint 单例与独立 mempool，账户抽象内嵌交易验证流程 | 需要从零设计执行客户端的交易验证模型，与现有 OP Stack/EVM 语义不兼容 | 需换栈（不是 OP Stack 可以渐进采用的路径） |
| 24 | OP Stack Custom Gas Token v2（CGT v2） | 让 L2 用任意 ERC-20（18位小数）替代 ETH 做 gas | 随 Upgrade 18（`op-contracts/v6.0.0`）发布，官方生产可用特性【一手】；v1 beta 已于 2025-02 停用 | 桥接/兑换逻辑下沉应用层，无统一 API | 仅支持 18 位小数代币；v1→v2 无迁移路径，需硬分叉 | 需改 EVM 执行层（新增预部署合约+ L1/L2 双侧标志位） |
| 25 | Mantle MNT 原生 gas（自有 fork，非 CGT） | Mantle 现状：MNT 作为唯一原生 gas 资产 | Mantle 主网 2023-07-17 上线，早于官方 CGT 特性存在【一手】 | 已稳定运行 3 年以上 | **技术谱系与官方 CGT v1/v2 完全独立**，任何"参考 CGT v2 文档改 Mantle gas 逻辑"的假设都不成立 | 已实现（现状），若要迁移到官方 CGT v2 架构则等价于"需换栈"级别的硬分叉改造 |
| 26 | 协议级 Subnet/CDK 自定义 gas token（X Layer/DFK/Swimmer 模式） | 部署时直接把应用代币设为链原生 gas 资产 | X Layer 用 $OKB（Polygon CDK 官方部署选项，生产环境）；DFK/Swimmer 用生态代币（Avalanche Subnet，DFK 已计划 2026-08 停运） | 生产验证的部署模式，非实验性 | 与 Mantle 现有的自有 fork（#25）是两条不同技术路线，不能直接套用 Polygon CDK 配置项 | 需改 EVM 执行层（若要迁移 Mantle 现有 fork 到该模式） |
| 27 | 应用层 Bot Tax（Metaplex Candy Machine 模式） | 对失败/无效铸造交易收取惩罚性费用，drain sniper bot 资金 | Solana Metaplex Candy Machine，2022年上线，生产案例【一手】 | 有效经济手段，不依赖链级改造 | 仅对"失败重试型"bot 有效，无法阻止"愿意接受惩罚仍要抢跑"的场景 | 无需改客户端（纯合约逻辑，可直接照搬到 Mantle Launchpad 合约） |
| 28 | Robinhood Chain 单点 FCFS 架构 | 用极简 FCFS + 单一 sequencer 换取低延迟 | Robinhood Chain（Arbitrum Orbit），约100ms区块时间 | 2026-09-04 发生约13分钟区块生产中断 | 单一 sequencer 无热备份时的可用性代价，是"简单排序规则"的反面教材 | 无需改客户端（但警示：Mantle 现状同为单一 sequencer，需评估同类风险） |


## 7. 投入产出比排序：给 Mantle Launchpad 场景的直接回答

### 7.1 排序原则

**约束前提**（第一阶段实测，本轨道复核仍成立）：Mantle 出块 2.000s、区块 gasLimit 60M、填充率 **0.224%**、base fee 钉在 **50 gwei 下限**、tx/block **2.0**。⇒ **区块空间与吞吐都不是瓶颈，且短期内不可能成为瓶颈。**

因此本轨道给出的投入产出比排序如下（数字越小优先级越高）：

| 优先级 | 零件类别 | 代表零件（§6 表号） | 为什么这个顺序 | 实现难度 |
|---|---|---|---|---|
| **P0** | **应用层抗抢跑 / MEV 内部化** | #27 Bot Tax、#13 Angstrom 式批量拍卖、#14 Atlas OFA、#15 MEV tax | **直接命中发行场景唯一真实的稀缺资源：发行时刻的抢跑权。** 全部是合约层，Mantle 今天就能做，不需要任何链改造 | **无需改客户端** |
| **P1** | **额度门禁 / 反 bot 洪水** | （§4 rate limiting；另见 I 轨道 RISEx 的 TX 额度制） | 第一阶段已论证：Mantle 2 秒出块使「衰减税」退化为阶梯函数，**必须用额度门禁**。RISEx 独立收敛到同一机制（新地址 10,000 免费额度 + 每 $5 成交量 +1 + 撤单免费）→ 交叉验证 | **无需改客户端** |
| **P2** | **gas 抽象（发币/交易免 gas 感知）** | #20 Plasma 式协议原生 paymaster、#21 ERC-4337、#22 EIP-7702 | 发行类应用的用户是新手，"要先有 gas 币"是最大漏斗损失。可用 relay 代付合成（RISE 就是这么做的，见 I 轨道） | **无需改客户端**（4337 路线） |
| **P3** | **软确认流（pre-confirmation）** | #1 Base Flashblocks、#2 Unichain VPO | **高 ROI 但非 P0**：改善的是"感知"而非"公平"。第一阶段已判定为 P1 级建议，本轨道下调理由是：在填充率 0.2% 的链上，2 秒确认并不构成用户流失的主因；而抢跑不公平构成 | **需改 op-node / builder** |
| **P4** | **可验证排序（TEE / VPO）** | #2 Unichain 的 TEE builder | 只有在"排序公信力"成为对外卖点时才值得投。它把 Mantle 单 sequencer 的信任问题从"承诺"变成"可验证"——这是**叙事资产**而非性能资产 | **需改 op-node / builder** |
| **P5** | **区块空间隔离 / local fee market** | #16 Solana LFM、#19 Tempo lane、#18 per-contract fee market | **当前明确不建议投入。** 第一阶段结论"现在做 LFM 是无病呻吟"在本轨道成立；#18 更是**检索无任何生产实现** | 需改 EVM 执行层 / 需换栈 |
| **P6** | **express lane 拍卖** | #8 Arbitrum Timeboost | **明确不建议。** 实证已证伪：3 个实体赢下 **99.74%** 拍卖、约 30% 交易 revert、DAO 收入反而从 ~$7.46M/年 降到 ~$2M/年，且 2026-06 已有 AIP 提议停用 | 需改 op-node |

### 7.2 直接回答

> **若目标是 launchpad / 发行类应用的 UX，投入产出比排序应该是：**
> **① 应用层抗抢跑与 MEV 内部化（合约层，零链改造）→ ② 额度门禁 → ③ gas 抽象 → ④ 软确认流 → ⑤ 可验证排序 → 最后才是区块空间隔离与拍卖式排序（当前应明确不投）。**
>
> **反直觉但重要的一条**：本轨道盘点的 28 个零件里，**排序前三名全部不需要动链**。这意味着「为了给发行类应用更好的链 UX 而去改链」这个命题，**在排序/延迟这条线上基本不成立**——链改造的价值要到「资产发行原语本身」（L 轨道）与「gas/账户抽象」这两条线上才显现。

## 存疑清单

1. ⚠️ **Unichain VPO 的量化延迟分布与排序验证失败率未公开**，官方仅有定性表述。
2. ⚠️ **BuilderNet 的生产数据未在本轮核实范围内**（仅确认 2024-11 上线）。
3. ⚠️ **Penumbra 的 sealed-bid 批量拍卖隐私是否完全落地**，检索结果与常识表述冲突，未收敛。
4. ⚠️ **Arbitrum 2026 拟重新引入 PGA（私有 mempool + 125ms 轮次）的提案是否已生效**，未确认；Timeboost 停用 AIP 亦仍在提案阶段。
5. ⚠️ **per-contract fee market（§6 #18）检索无任何生产实现或活跃提案**，按"不存在"处理——若判断错误将影响 P5 的结论。
6. ⚠️ **Tempo payments lane 是否为区块头层面的协议原生隔离**，本轨道依据官方文档判定为"是"，但未读到区块头结构规范原文。
7. ⚠️ **Plasma 的零费 USDT 路径是否协议内建 vs paymaster 合约**：本轨道判定为协议自营 paymaster（覆盖标准 `transfer`/`transferFrom`），但"协议级"与"官方运营的 4337 paymaster"的界线未由一手规范厘清。
8. ⚠️ **MegaETH 主网上线日期 2026-02-09** 为一手/二手交叉确认，未见单一权威公告原文。
9. ⚠️ **Solana Alpenglow 目标 Q3 2026** 为官方页面标注"In Development"，本轨道取数日仍未上线；后续可能变动。
10. ⚠️ **RISE 的实际排序规则（FCFS？priority fee？状态感知预排序？）无一手文档** → 见 H 轨道存疑清单第 8 条。这是本对照表关于 RISE 的最大缺口。

## 关键来源清单

**pre-confirmation 家族**
- Base Flashblocks / Denim：<https://docs.base.org/> ｜ <https://blog.base.dev/flashblocks-deep-dive>
- Unichain Rollup-Boost（2025-05-02）与 TEE block builder：<https://blog.uniswap.org/> ｜ Flashbots rollup-boost：<https://github.com/flashbots/rollup-boost>
- MegaETH：<https://docs.megaeth.com/>
- Solana Alpenglow：<https://www.anza.xyz/blog/alpenglow>
- RISE Shreds：<https://docs.risechain.com/docs/rise-evm/shreds>（→ H 轨道）

**排序规则与 MEV**
- Arbitrum Timeboost：<https://docs.arbitrum.io/how-arbitrum-works/timeboost/gentle-introduction>
- Timeboost 中心化实证：arXiv:2509.22143 <https://arxiv.org/abs/2509.22143>
- Order-fairness 不可能性（Aequitas / Themis）：<https://eprint.iacr.org/2020/269>（CRYPTO 2020）
- Shutter Network：<https://shutter.network/>
- SUAVE 归档（2025-05-12 只读）：<https://github.com/flashbots/suave-geth>

**ASS（application-specific sequencing）**
- Angstrom / Sorella：<https://sorellalabs.xyz/>
- Atlas / FastLane：<https://github.com/FastLane-Labs/atlas>
- "Priority Is All You Need"（Robinson & White, 2024-06-04）：<https://www.paradigm.xyz/2024/06/priority-is-all-you-need>

**区块空间与 gas 抽象**
- Solana SIMD-0286：<https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0286-increase-block-limit-to-100m.md>
- EIP-7706（Stagnant）：<https://eips.ethereum.org/EIPS/eip-7706>
- ERC-4337：<https://eips.ethereum.org/EIPS/eip-4337> ｜ EIP-7702：<https://eips.ethereum.org/EIPS/eip-7702>
- OP Stack Custom Gas Token：<https://docs.optimism.io/stack/rollup/custom-gas-token>
- Metaplex Candy Machine bot tax：<https://developers.metaplex.com/candy-machine>

**本仓库交叉引用**
- Mantle 实测基线：`research/E-mantle.md`、`report/05-infra-requirements.md`
- Robinhood Chain FCFS 与合规过滤：`research/D-robinhood.md`、`report/03-robinhood-chain.md`
- RISE / RISEx：`research/H-rise-chain-infra.md`、`research/I-risex-app-coupling.md`

## 可复现方法附录

本轨道以**一手文档研究为主**，实测部分由主 agent 在 H / N 轨道完成，本轨道直接引用，不重复取数。

引用的实测基线（方法见 `research/H-rise-chain-infra.md` 附录）：

| 链 | 出块 | gasLimit | 有效容量 | 填充率 | baseFee | `eth_sendRawTransactionSync` |
|---|---|---|---|---|---|---|
| RISE | 1.000s | 1,500M | 1,500 Mgas/s | 8.4–10.4% | 0.0004153 gwei | ✅ |
| Mantle | 2.000s | 60M | 30 Mgas/s | 0.224% | 50 gwei（下限） | ❌ |
| Base | 2.000s | 400M | 200 Mgas/s | 8.9% | 0.005 gwei | ✅ |

复现方式：对各链公共 RPC 串行采样 15–20 个连续区块的 `eth_getBlockByNumber`，取 `timestamp` 差分、`gasLimit`、`gasUsed`、`baseFeePerGas`；方法存在性用无效参数调用区分 `-32601`（不存在）与 `-32602`（参数错）。
