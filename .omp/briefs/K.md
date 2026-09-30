# 轨道 K：排序 / 延迟 / 区块空间 / MEV 的链级零件目录

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/K-sequencing-latency-infra.md`

主题：**排序 / 延迟 / 区块空间 / MEV 的链级技术货架** —— 为最终设计方案提供「可选零件目录」，每个零件都要有「**谁在生产环境用了、代价是什么**」。

## Change

### 1. pre-confirmation 家族
- **Base Flashblocks**（200ms；含 rollup-boost / Flashtestations TEE；**Denim 提案**的最新现状）
- **Unichain**（rollup-boost + TEE block builder + **Verifiable Priority Ordering**，原理与实测效果）
- **MegaETH**（mini-blocks 10ms、单 sequencer + 专用 prover / full-node 分工、是否已主网）
- **Solana** 的 shreds / Turbine 与 **Alpenglow** 现状
- RISE Shreds：**只做交叉引用**，深挖交给 H 轨道

做**统一维度对照表**：软确认延迟 / 谁签名 / 可否回滚 / 用户可感知语义 / 是否需要改客户端 / 生产状态与上线日期

### 2. 排序规则货架
- **FCFS**（Arbitrum、Robinhood Chain 实况）
- **priority gas auction**
- **Arbitrum Timeboost**（express lane 拍卖的**实测中心化数据**与 2026 停用提案的最新现状）
- **verifiable priority ordering**（Unichain）
- **批量拍卖 / frequent batch auction**（Injective、CoW Protocol、Penumbra）
- **latency-fair ordering 学术方案**（Themis / Aequitas / order-fairness 的不可能性结论）
- **私有 mempool / encrypted mempool**（Shutter、SUAVE 的现状 —— SUAVE 是否已死？要核实）

每条给：机制 / 生产状态 / 已知实测后果

### 3. application-specific sequencing（ASS）—— 关键概念货架
- **Sorella / Angstrom**（Uniswap v4 hook 内做批量拍卖，是否已主网）
- **Fastlane / Atlas**（Monad 上的 app 级拍卖）
- **MEV tax / "Priority Is All You Need"（Robinson & White）** 的机制与**前提条件**（要求 priority ordering 且 sequencer 诚实 —— 这个前提在 OP Stack 上成立吗？）
- 「app 自己做排序而不需要自己开链」的可行边界

**本节要直接回答**：委托方想要的效果，**有多少可以不开新链、只靠 app 级排序拿到**？

### 4. 区块空间与拥塞隔离
- **Solana local fee market**（SIMD-0286 后的 CU 参数、per-writable-account 上限的确切数字）
- EIP-1559 全局 base fee 的失效场景
- **多维 gas / EIP-7706** 现状
- per-contract fee market 的现有提案与实现（若有）
- **预留 lane / 分区区块空间的生产案例**：**Tempo（Stripe/Paradigm 的支付链）的 payments lane**、**Plasma 的 zero-fee USDT 转账路径** 是重点 —— **务必查清是协议级内建还是 paymaster / relayer 级**
- rate limiting / 抗 bot 洪水的链级手段

### 5. gas 抽象货架
- 协议级 paymaster（Plasma / Tempo 的 fee-token 抽象是否协议内建）
- ERC-4337 vs EIP-7702 vs 原生 AA（zkSync / Starknet）
- **custom gas token**（OP Stack 的 custom gas token 现状与限制、Mantle 的 MNT gas）
- 「用应用代币付 gas」的生产实现

### 6. 总表（必写）
**零件目录表** —— 列：
| 零件 | 解决什么问题 | 生产案例（+上线日期） | 实测效果 | 已知代价 | 在 OP Stack 上的实现难度 |

实现难度用四档：`无需改客户端` / `需改 op-node 或 builder` / `需改 EVM 执行层` / `需换栈`。

**这张表是最终设计方案的直接输入，是本轨道最重要的产出。**

## Acceptance

- 文件已创建，含 6 节，零件目录表覆盖**不少于 20 个零件**
- 每个「生产案例」都必须给出**上线日期**与一手来源；**仅提案的必须明确标注为提案**
- ASS 一节必须给出「不开链也能拿到的部分 vs 必须改链的部分」的**明确切分**
- 「存疑清单」不少于 **8** 条
- **显式回答**：若目标是 launchpad / 发行类应用的 UX，排序与延迟这条线的**投入产出比排序**应该是什么？
  - 约束条件：第一阶段实测 **Mantle 区块填充率仅 0.173%、base fee 常年钉在 50 gwei 下限、EIP-1559 从未触发** —— 吞吐**不是**瓶颈。请在这个约束下作答，不要给"提升 TPS"这种无效建议。
