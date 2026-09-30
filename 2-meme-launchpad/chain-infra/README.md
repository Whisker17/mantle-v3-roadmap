# 资产发行层专属链优化（Launchpad Dedicated Chain Infra）

> **归属产品**：支柱 1 · 发行层（Tape meme Launchpad & 代币化资产发行）  
> **核心命题**：为资产发行、打新防狙击、合规过滤与无感相变提供专属链级原语与执行层优化。  
> **分期**：先做发射白名单上的 Sequencer 政策，再做多应用调度与 VM 原语。定稿见 [`short-long-goals.md`](short-long-goals.md)。
---

## 1. 为什么发行层需要专属链优化？

普通通用 EVM 在资产发行场景存在三大致命短板：
1. **代币标准能力匮乏**：ERC-20 缺乏原生生命周期钩子，任何转账税、黑名单、防狙击都必须侵入式写进应用层合约，极易引入漏洞。
2. **出块时间与竞价博弈失效**：Mantle 固定的 2.0 秒出块间隔以及缺乏优先竞价费用市场的特点，导致传统 Solana/EVM 的「衰减税（Decay Tax）」机制退化为离散阶梯函数，无法阻挡脚本 Sniper 抢跑。
3. **证券型代币的合规与公司行动断层**：当以 xStocks（代币化美股）作为计价货币或标的时，链级缺乏对拆股（Split）、分红（Dividend）以及周末休市的统一感知。

---

## 2. 专为 Tape Launchpad 打造的链级优化方案

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                   Launchpad 专属链级优化矩阵                          │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 1. 排序与交易池调度 (Sequencer & Mempool Scheduling)                  │
 │    - 热点合约分桶与配额硬顶：单合约 Gas ≤ 25%，为关键结算保留独立空间  │
 │    - 开盘微批次拍卖 (FBA)：同批次统一清算价，协议底层杜绝抢跑夹子     │
 │    - 模拟-免费拒绝 (Revert Protection)：试跑失败直接 Drop，不上链不扣费 │
 │    - 毫秒级增量预确认流：100~200ms 软确认，消除反复刷单与盲盒滑点      │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 2. 链级代币原语增强 (Token Primitives)                                 │
 │    - 兼容 ERC-20 的安全 Transfer Hook：VM 级防重入保护与动态衰减税     │
 │    - 原生费用路由 (Fee Routing)：免去应用层复杂计账，自动分账           │
 │    - 归零代币状态修剪与租金回收 (Rent Pruning)：防止全节点存储腐化     │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 3. DEX 与零迁移相变 Hook (Uniswap v4 Phase Transition)                 │
 │    - 曲线达到毕业阈值时，在 Uniswap v4 Hook 内部直接相变，无资金跨池迁移│
 │    - 永久锁定 Locker + 费用钥匙 (Fee Key NFT)：防 Rug 与永续分成兼得   │
 │    - 底层必须完整支持 Cancun EIP-1153 (TSTORE/TLOAD) 瞬态存储           │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 4. 共享流动性与跨链加速 (Shared Liquidity & Cross-Chain)               │
 │    - 1inch Aqua 式统一 Quote 水库：散户稳定币一键打新，消除 155 标的碎片化│
 │    - Ethereum FCR 快速确认集成：单 Slot 12~13 秒入金，加速跨链做市垫资 │
 │    - 拒绝外部共享排序 (Espresso)：坚守自营 Sequencer 的调度特化权      │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 5. 协议级合规过滤 (Compliance Filtering)                               │
 │    - 参照 ArbOS Elara (ArbFilteredTransactionsManager)，STF 级交易拦截  │
 │    - 美股代币/RWA 必须做；纯 Crypto-Native 改用 TIP-403 资产级策略     │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 6. 账户与交互体验层 (Native AA & Session Key)                         │
 │    - 协议级 Session Key：TG Bot 1-Click 极速打新，免频繁弹窗签名        │
 │    - 原生 Paymaster：任意代币/稳定币免 Gas 支付，Web2 级低门槛体验      │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 本目录底稿索引

- [`short-long-goals.md`](short-long-goals.md)：**【Short / Long 分期定稿】** 第一期只服务 Launch Registry（S1 热点配额、S4 失败 Drop、S5 发射 RPC；S3 FBA / S6 AA 复用作第二包）；**S2「瞬时热点吞吐」因无链上实例降为 L0**（D-infra-7）；ASS、LFM、VM Hook、rent 进后期。机制细节见下一份 spec，对外叙事版见 [`../outputs/06-chain-infra-plan.md`](../outputs/06-chain-infra-plan.md)。
- [`launchpad-chain-spec.md`](launchpad-chain-spec.md)：**【Meme Launchpad Specific Chain 架构规范】** 调度层（分桶/FBA/Revert Protection）、执行层（ERC-20 兼容 Hook/费用路由）、Uni v4 毕业相变、合规过滤与 Native AA。
- [`shared-liquidity-and-quote.md`](shared-liquidity-and-quote.md)：**【mStocks 计价冷启动与共享流动性架构】** 剖析 155 个股票代币四大冷启动死锁，借鉴 1inch Aqua 打造统一做市水库与 JIT 结算，横向评估 Ethereum FCR 与 Espresso 跨链机制。
- [`issuance-primitives.md`](issuance-primitives.md)：**【资产发行原生链级原语深度盘点】** 全面梳理 Solana Token Extensions（Token-2022）、Cosmos 模块化发行、批量拍卖机制与 EVM 缺失能力清单。
