# 轨道 J：交易类 app-specific chain 横向对标

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/J-appchain-comparables.md`

主题：**交易类 app-specific chain 的横向对标** —— 重点是「链级原生模块」到底长什么样、被证明有效还是无效。

## Change

对以下每个案例做**统一模板**拆解：**链级原生做了什么 / 为什么必须在链级 / 效果数据 / 代价与失败教训**。

### 1. Hyperliquid（最重要，最详细）
- **HyperCore 作为 L1 原生状态机**：原生订单簿、原生 margin、validator 集体喂价 oracle；HyperBFT 共识
- **HIP-1 原生代币标准 + 荷兰式拍卖上币额度**、**HIP-2 原生自动做市（Hyperliquidity）**、**HIP-3 无许可永续市场部署** —— 这三条对本研究的「资产发行原生」命题**极关键**，要拆到**参数级**：拍卖周期、起拍价、成交价历史区间、部署保证金门槛、额度稀缺性设计
- **HyperEVM dual-block 架构**（small block / big block）、**CoreWriter 与 read precompiles**（HyperCore ↔ HyperEVM 的原子性与非原子性边界 —— 这是"原生模块如何暴露给通用 EVM"的唯一生产级样本，务必拆细）
- gas 与手续费模型、HYPE 的价值捕获路径
- 数据表现（量 / OI / 收入 / 份额，标日期）与近期事故（JELLY 事件的机制教训、清算 / ADL 争议、oracle 操纵）

### 2. dYdX v4
- Cosmos 自建链的 app-specific 模块清单（clob / prices-oracle / subaccounts / vault-megavault）
- **订单簿在验证者内存中、下单撤单零 gas** 的设计与其代价（订单簿非共识状态意味着什么）
- 从 StarkEx 迁移的动因**原话**
- 数据表现与市场份额变化（从龙头到今天，标日期）

### 3. Injective
- exchange module（链级原生 CLOB + **Frequent Batch Auction 抗 MEV**）、oracle module
- **tokenfactory / permissions module / RWA module**（对 L 轨道也有价值，标 `→ 交给 L`）
- 原生 gas 补偿机制、MultiVM / EVM 兼容（inEVM / native EVM）
- 效果数据 + **「链级原生撮合但生态没起来」的归因** —— 这个负面归因对委托方极重要

### 4. 通用栈做 perps 会遇到什么墙
- **Aevo（OP Stack rollup 做 perps）最有对照价值** —— 它就是"用通用栈做交易应用链"的实验，结局如何？
- **Lighter**（zk perp appchain 的 prover / latency 声明与实际）
- **Paradex**（Starknet appchain）
- **Vertex / Edge**
- Aster / edgeX / GRVT 中选择性覆盖
- 回答：用通用 rollup 栈做 perps 会遇到什么墙

### 5. 反面与横向
- Robinhood Chain：本仓库 `report/03-robinhood-chain.md` 与 `research/D-robinhood.md` **已详细覆盖，只做一句话交叉引用，禁止重复研究**
- **app-specific 但失败 / 退化的案例**（专用 appchain 后来被弃用、迁回通用链的），提炼「**app-specific 的失败模式**」

### 6. 归纳章节（必写）
- **表 A**：链级原生模块清单 × 各链是否具备（原生订单簿 / 原生 oracle / 原生代币发行 / 原生做市 / 原生保证金 / 无 gas 下单 / MEV 抗性机制 / 原生上币额度拍卖 / 原生保险基金 …）
- **表 B**：「必须链级」vs「应用层足够」的判定，每条给论证
- **表 C**：app-specific 的**代价清单**（生态组合性损失、桥依赖、开发者稀缺、审计面、去中心化倒退、单应用风险、流动性割裂）
- **结论**：app-specific chain 在什么条件下成立、什么条件下是自杀。给出**可检验的判据**（如 TVL / 量的临界规模、单应用占比阈值、团队工程能力门槛）

## Acceptance

- 文件已创建，五个案例组 + 归纳章节齐全，**Hyperliquid 章节最详细**（含 HIP-1/2/3 参数级描述与一手 docs URL）
- 表 A / 表 B / 表 C 三张归纳表齐全
- 每个案例都有效果数据（量 / TVL / 份额）并标注日期与来源
- 「存疑清单」不少于 **8** 条
- **显式回答**：除 Hyperliquid 外，**链级原生撮合有第二个成功案例吗**？如果没有，这对「app-specific 模式可行性」意味着什么
