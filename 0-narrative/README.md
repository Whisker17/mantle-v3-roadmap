# Mantle v3 核心叙事与产品路由母篇：CeDeFi Capital Markets

> **战略地位**：全仓库 High-Level 主叙事总纲与产品路由母目录  
> **核心命题**：**CeDeFi Capital Markets（链上资本市场）**  
> **战略 Tagline**：**Where Assets Go Public（万物上市）**  
> **核心破局点**：承认历史，完成闭环——将 Mantle 从沉淀了 $576M 资金却无处交易的「链上银行」，重构为集**一级发行、做市撮合、现货交易、全天候衍生品、杠杆借贷与最终清算**于一体的完整「链上资本市场」。

> **架构选型阅读说明**：当前按三条路线比较：① 保持现有 Mantle 执行层；② 同一执行层引入 MantleCore；③ Mantle L2 结算、App-specific L3 执行。先读 [07 路线比较](07-three-architectural-routes-tradeoff-analysis.md)与 [09 L3 完整方案](09-native-l3-architecture-and-errata.md)。本页下方及 00、06、08、10 中的既有叙事保留作早期材料，不表示已确定选择双 L3；技术边界和工作量以本轮 07、09 为准。

---

## 1. 宏观叙事重构：从「链上银行」到「链上资本市场」

### 1.1 过去三年：建成了买方，却缺少市场
过去三年，Mantle 投入数亿美元生态激励（EcoFund $2 亿、Methamorphosis、Journey 2,500 万 MNT），成果显著却极度偏科：
- **资金入口与吸储**：UR 新银行入口；
- **资管与生息产品**：MI4（AUM 破 $200M）、mETH / cmETH 生息资产、Mantle Vault；
- **全球领先的资产库**：沉淀了全球第 2 大代币化股票库（155 个 xStocks 标的，AUM 达 **$633.7M**，规模为 Robinhood 的 4.8 倍）；
- **链上资金沉淀**：拥有 **$576M 闲置稳定币**（为链上 DeFi TVL 的 5.9 倍）。

**历史失败归因**：
Mantle 过去的激励机制全部结构化为质押与锁仓，**「奖励不动的钱」，从未建立过「奖励交易的飞轮」**。结果是链上聚集了庞大的买方资金，却**没有交易场所与卖方供给**——日均 DEX 交易量长期徘徊在 **$0.94M**，活跃地址仅千余个。

### 1.2 Mantle v3 的战略飞轮：一核双星架构
> **「银行聚集了钱，双 L3 极速资本市场让钱动起来。」**

新叙事不否定过去，而是完成商业闭环：
- **过去建买方**：储蓄、理财、沉淀资产在 **Mantle L2 资产信贷总行**；
- **现在建卖方与场内**：解耦为两条高性能专有执行链——**Launchpad L3 发行链** 与 **Perps L3 衍生品链**，消灭单链拥堵与状态爆炸。

```
     ┌─────────────────────────────────────────────────────────────┐
     │ Mantle L2 中央资产与信贷总行：$576M 稳定币 + 155 个 xStocks 标的 │
     │ Mantle Lending 融资融券中心 ｜ Unified Margin 统一授信中心   │
     └──────────────────────────────┬──────────────────────────────┘
                                    │ 远程授信 / 意图闪电充值通道
                                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│              专属执行层：双 L3 极速交易大厅 (The Twin Trading Floors)           │
│                                                                               │
│  【L3-A 发行专属链 (Tape)】             【L3-B 衍生品专属链 (Equity Perps)】   │
│  • 50ms 极速 Bonding Curve 抢打新        • 24/7 股票永续合约 ｜ 极速清算通道    │
│  • 0 Gas 准入 ｜ 状态物理剪枝            • 接受 L2 远程保证金授信免资金跨链     │
│  • ★ 内嵌毕业 AMM (原地相变零断层)       • ★ 内嵌 RFS 流式撮合 (做市商紧点差)   │
│             │                                                   │             │
│             └─────────────────────┬─────────────────────────────┘             │
│                                   ▼                                           │
│                       【外部性反哺与价值回流】                                │
│                       25% 反哺 RFS 深度池 + 20% 合约级强制市价回购 MNT        │
└───────────────────────────────────┬───────────────────────────────────────────┘
                                    │ 外部动力与分发接入
                                    ▼
     ┌─────────────────────────────────────────────────────────────┐
     │ Bybit CeDeFi Sidecar：W1 经纪商 + W2 做市商 + W3/W4 深度赋能│
     └─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心产品矩阵介绍与目录路由导引

资本市场的五个支柱不是孤立的功能清单，而是**完整复刻传统金融市场的资本分层与流转机制**。每个产品均有独立的顶层研究目录，并在各自目录下配置了**针对该产品定制的 Chain Infra 优化方案**：

### 🚀 支柱 1：发行层 —— Tape meme Launchpad
- **产品定位**：**代币化股票的情绪一级市场（The Primary Sentiment Market）**，资本市场的点火器。
- **核心机制**：股票 Quote 专注（NVDAx、TSLAx 计价），寄生于美股财报季与 FOMC 外生日历；相变式 Bonding Curve（毕业零迁移、零预言机）；Spend-Gate 额度门禁抗狙击；全行业首创休市熔断 Hook 与公司行动 Hook。
- **反哺闭环**：每一笔 meme 打新与交易，均直接为主业资产（xStocks）创造买盘，20% 协议费写入合约自动回购 MNT。
- **👉 详细产品研究与规范路由**：[`../meme-launchpad/`](../meme-launchpad/)
  - 专属链优化：[`../meme-launchpad/chain-infra/`](../meme-launchpad/chain-infra/)（原生代币扩展、转账钩子与合规过滤）
  - 竞品全面对标：[`../meme-launchpad/benchmarks/`](../meme-launchpad/benchmarks/)（Solana / BSC / Base / Robinhood）

### 💱 支柱 2：现货交易层 —— RFS (Request for Stream) & Prop AMM
- **产品定位**：**mStocks 链上唯一的二级市场交易闸口与做市商流式报价场所（Streaming Prop Venue）**。
- **核心机制**：做市商以毫秒级 EIP-712 流式签名提供 0 滑点、极窄点差报价；**底层 AMM DEX 被封装在内部，仅充当价格发现参考与极端断流兜底（严格不鼓励散户绕过 RFS 在 AMM Swap）**；外部仅允许 Bybit 现货充值、Solana 跨链桥接与 Lending 借出三大源头 Inflow；做市商享有 Preconfs 撤单优先保障，并由 Omni-Vault 金库分担库存风险。
- **👉 详细产品研究与规范路由**：[`../rfq-propamm/`](../rfq-propamm/)
  - 系统架构规格书：[`../rfq-propamm/fluxion-propamm-spec.md`](../rfq-propamm/fluxion-propamm-spec.md)（EIP-712 签名协议与开闭市双模状态机）
  - 专属链优化：[`../rfq-propamm/chain-infra/`](../rfq-propamm/chain-infra/)（无公开 mempool 抗夹承诺、Cancel Priority 做市商撤单优先通道、Flashblocks 预确认流）
### 📈 支柱 3：衍生品层 —— 股票 Perps 与链原生支持
- **产品定位**：**代币化股票 24/7 全天候杠杆与对冲市场（Equity Perps）**。
- **核心机制**：避开与 Hyperliquid 在通用加密 Perps 的正面死亡竞争，主打股票 Perps；依赖 Mantle 独有的现货 xStocks + Fluxion RFQ 提供真实基差腿，打造无风险资金费率套利飞轮；将周末与夜间休市定价做成特色产品而非流动性事故。
- **👉 详细产品研究与规范路由**：[`../perps/`](../perps/)
  - 专属链优化与三大机制空白：[`../perps/chain-infra/`](../perps/chain-infra/)（清算专用 Lane、模拟免拒单 Revert Protection、预言机原子绑定）
  - 对标案例工程真相：[`../perps/chain-infra/rise-appspecific-teardown.md`](../perps/chain-infra/rise-appspecific-teardown.md)（RISE 1.5 Ggas 暴力解 CLOB 拆解与经验借鉴）

### 🏦 支柱 4：融资层 —— Margin Lending（融资融券）
- **产品定位**：**证券抵押信贷与杠杆资金池（Securities Margin Facility）**。
- **核心机制**：坚决不做同质化 Aave Fork（杜绝雇佣兵资本大起大落）；专注「持有 xStocks / mETH 抵押 ──► 借出稳定币 ──► 加杠杆打新/交易」真实融资融券业务，成为承接并盘活 $576M 闲置稳定币的唯一真实需求出口。
- **👉 详细产品研究与规范路由**：[`../lending/`](../lending/)
  - 专属链优化：[`../lending/chain-infra/`](../lending/chain-infra/)（市场日历预编译、周末动态折价 Haircut 与预言机防陈旧保护）

### 💻 支柱 5：接口与分发层 —— Tape Terminal & Bybit Alpha
- **产品定位**：**资本市场执行中枢与超级流量漏斗（Interface & Distribution Hub）**。
- **核心机制**：直面价值捕获最高层（接口层 > 协议层 > 链层），打破 GMGN 等工具对 Mantle 零覆盖的死锁；自建专业原生 Tape Terminal（战壕看板、老鼠仓审计、Spend-Gate 额度看板）；结合 Bybit Alpha 白标发行引擎，让 8,000 万 CEX 托管用户在免助记词、免 gas 前提下享受链上发行红利。
- **👉 详细产品研究与规范路由**：[`../terminal/`](../terminal/)
  - 专属链优化：[`../terminal/chain-infra/`](../terminal/chain-infra/)（Mantle Passport 生物识别 Passkey 登录 + ERC-4337 Paymaster 全免 gas）

---

## 3. 基础设施哲学：拒绝抽象空转，链优化必须为产品服务

Mantle v3 彻底摒弃「为了改链而改链」的传统工程师思维：
- **不另立抽象的独立 Infra 目录**：基础设施没有孤立的价值，其唯一合法性来自其所支撑的具体交易产品。
- **针对性产品赋能**：
  - **Launchpad 需要**：合规过滤、转账钩子、出块时间解耦；
  - **RFQ 现货需要**：做市商撤单优先队列、无公开 Mempool 抗夹、毫秒级预确认；
  - **Perps 需要**：清算专用保留车道、模拟-免费拒单、预言机与清算原子化；
  - **Lending 需要**：市场日历预编译与预言机陈旧度守卫；
  - **Terminal 需要**：Passkey 原生抽象与 Gas 代付。
- **Mantle 的底层改造底气**：Mantle 早已转向 **OP Succinct（SP1 ZK proof）**，摆脱了 Optimism 传统 MIPS 单指令仲裁的强约束，魔改执行层（op-geth/revm）的工程难度与治理风险可控（详见技术底稿 `04-mantle-opstack-surface.md`）。

---

## 4. 外部动力引擎：Bybit CeDeFi Sidecar 挂斗

如果说 Mantle 是链上资本市场的**交易所与清算所**，那么 Bybit 就是这个体系不可替代的**外部动力系统（Sidecar）**：
- **W1 经纪商（Broker）**：Bybit Alpha 白标前台，8000 万用户直达；
- **W2 做市商（Market Maker）**：自营库存为 Fluxion RFQ 与股票 Perps 注入点差极低的种子流动性；
- **W3 发行方（Issuer）**：bStocks 路线原生自营股票发行，取得 ADGM/FSRA 合规招股书，实现高频上新；
- **W4 主经纪商（Prime Broker）**：统一交易账户（UTA）保证金打通，链上资产无缝充当交易所合约保证金。
- **👉 详细 Sidecar 整合架构与工作流**：[`../sidecar/`](../sidecar/)

---

## 5. MNT 价值捕获与经济飞轮

Mantle v3 彻底终结 MNT「无价值捕获、无收益回流」的历史：
- **25% 协议手续费**：直接注入 Fluxion 的 xStocks 深度池，做大主业资产流动性；
- **20% 协议手续费**：**直接写入不可篡改的智能合约**，在二级市场自动市价回购 MNT；
- **45% 创作者激励**：通过永久 Fee Key NFT 绑定优质资产发行方，构建可持续激励；
- **打新准入赋能**：Spend-Gate 额度与用户的 mETH/cmETH、xStocks 持仓及 Bybit VIP 等级深度绑定，直接拉动生态资产锁仓需求。

---

## 6. 本目录底稿与技术支撑索引

| 文件 | 类型 | 核心内容 |
|---|---|---|
| [`00-executive-narrative-and-ecosystem-architecture.md`](00-executive-narrative-and-ecosystem-architecture.md) | **【汇报总纲】** | **高层汇报主文档**：基于「L2 金库总行 + 双 L3 极速交易」一核双星架构、以 mStocks 为核心资产、内嵌 AMM/RFS 设计、Bybit/国库 Sidecar 与全景 Mermaid 拓扑图 |
| [`01-repositioning-and-roadmap.md`](01-repositioning-and-roadmap.md) | **【决策定稿】** | 四大锁定决策全景推导、一核双星业务闭环、四轨路线图（P0–P4）与未决开放问题 |
| [`02-gap-analysis.md`](02-gap-analysis.md) | **【现状诊断】** | 链空事实、执行层零覆盖、两次失败复盘、七条归因与 Mantle 六张独有底牌 |
| [`03-mantle-baseline.md`](03-mantle-baseline.md) | **【链上实测】** | RPC 数据实测、TVL 崩塌序列分析、28 个 DEX 活跃度与资金分布全景 |
| [`04-mantle-opstack-surface.md`](04-mantle-opstack-surface.md) | **【底层可改造面】** | OP Succinct（SP1 ZK proof）架构优势、EVM 改造边界与 Treasury 预算治理机制 |
| [`05-infra-requirements-framework.md`](05-infra-requirements-framework.md) | **【需求框架】** | 14 维链级基础设施需求框架、LFM in EVM 阶梯与弹性区块空间推导 |
| [`06-cross-layer-interop-and-composability.md`](06-cross-layer-interop-and-composability.md) | **【早期跨层草案】** | 远程授信、毕业 AMM、RFS、意图通道与限流设想；签名结算、风险隔离和限流示例的修正见 09 |
| [`07-three-architectural-routes-tradeoff-analysis.md`](07-three-architectural-routes-tradeoff-analysis.md) | **【路线比较入口】** | 保持现有执行层 vs MantleCore vs App-specific L3：统一基线、能力与风险矩阵、同产品工作量、排期假设及选型证据 |
| [`08-route-2-hypercore-dual-engine-deep-dive.md`](08-route-2-hypercore-dual-engine-deep-dive.md) | **【早期双引擎推演】** | MantleCore / EVM 交互草图；原子性、SP1 集成与成本的比较口径见 07 |
| [`09-native-l3-architecture-and-errata.md`](09-native-l3-architecture-and-errata.md) | **【L3 完整路线方案】** | 以 Lighter 为主要参照：Overview、执行与结算、充提和重组、证明与 DA、退出和升级、可选扩展、优劣势与开发工作量 |
| [`10-base-l3-and-appchain-research.md`](10-base-l3-and-appchain-research.md) | **【早期外部案例材料】** | Base Appchain 与 op-enclave 的参考设想；TEE、altDA 和安全继承边界见 09，案例不作为 Mantle 必选 L3 的依据 |
| [`11-lighter-reference-for-mantle-l3.md`](11-lighter-reference-for-mantle-l3.md) | **【Lighter 参考设计】** | 白皮书与固定版本源码核验；Ethereum / Lighter 到 Mantle L2 / L3 的逐层映射、优先队列、批次结算、账户差分、Blob 适配和异常退出 |
