# 《Deeply Intents》核心深度解读与技术备忘录
## "Tilting at PropAMMs" —— Markus Schmitt (Propeller Heads 创始人)

---

### 一、节目基本信息
- **播客来源**：《Deeply Intents》（专注以太坊交易微观结构、DEX 路由求解与意图基础设施的前沿访谈）
- **本期嘉宾**：Markus Schmitt（Propeller Heads 创始人，旗下打造了 Tycho、Fynd 与 Turbine 等求解器底层技术产品）
- **音频时长**：66 分 09 秒
- **对应音频文件**：`tilting-at-propamms-markus.mp3`
- **中英双语转录**：
  - [分段转录 Part 1 (00:00 - 20:00)](bilingual_transcript_part1.md)
  - [分段转录 Part 2 (20:00 - 40:00)](bilingual_transcript_part2.md)
  - [分段转录 Part 3 (40:00 - 01:00:00)](bilingual_transcript_part3.md)
  - [分段转录 Part 4 (01:00:00 - 01:06:09)](bilingual_transcript_part4.md)
  - [全集完整双语对照](bilingual_transcript_full.md)

---

### 二、核心背景：求解器（Solver）与路由视角下的流动性革命

在以太坊生态中，做市商（Makers）负责提供流动性，交易者（Takers）发起兑换，而连接二者的中枢是 **DEX 聚合器与求解器（Solvers / Routers，如 CoW Swap、1inch、UniswapX）**。

Markus Schmitt 创立的 Propeller Heads 一直致力于解决链上最极端的数学与工程难题：**如何在毫秒级时间内从以太坊成千上万个流动性碎片中，为用户找到成本最低、滑点最小、零失败率的全局最优路径？**

当行业大肆宣扬“PropAMM（专有做市 AMM）”时，求解器开发者眼中的现实却更加硬核与复杂。本期播客以极具思辨性的视角（"Tilting at PropAMMs"，唐吉诃德式的冲锋与审视），拆解了 PropAMM 在不同链上的工程妥协，并公布了支撑下一代链上高效交易的 Propeller 三件套（Tycho、Fynd、Turbine）。

---

### 三、核心论点与微观结构技术全景

#### 1. PropAMM 的本质定义与技术基因
- **继承两大传统体系的双重特性**：
  - **AMM 的外壳**：它是一个部署在链上的智能合约，持有真实资产流动性储备，任何外部地址或合约均可直接向其发起原生调用进行结算。
  - **RFQ 的内核**：传统 AMM 的参数（中心价格、费率、曲线斜率）只能通过治理慢速修改或由交易行为被动推移；而 PropAMM 赋予了**特定链下实体（专业做市商）唯一的专属权限（Proprietary Rights）**，允许做市商根据外部市场行情随时主动修改合约定价与买卖价差。
- **Solana 与以太坊（EVM）的根本分流**：
  - **Solana 模式**：出块时间快（约 400ms），单笔交易成本比以太坊便宜 2 到 4 个数量级。做市商可以直接在链上发起超高频参数重置交易，每个区块多次刷新价格，被动 LP 几无生存空间。
  - **以太坊模式**：12 秒的漫长区块间隔加上高昂的 Gas 开销，使得高频链上主动更新（On-chain parameter updates）在经济上完全不可行。因此以太坊上的 PropAMM 必须演进出全新的协调模式——**与区块构建者（Block Builders）链下协同，在区块顶部（Top-of-Block）以预言机流式报价一次性校准**。

#### 2. 求解器（Solver）路由 PropAMM 的巨大工程挑战
- **确定性（Deterministic Execution）与交易失败惩罚**：
  - 在 CoW Swap 等批次拍卖中，求解器若提交了包含 PropAMM 的方案，而该 PropAMM 在区块打包前一刻撤销了报价或修改了参数，会导致整笔批次交易回滚（Revert），求解器将面临严重的押金扣罚。
  - 因此，求解器对 PropAMM 提出了严苛的要求：**流动性不仅要价格好，更必须具备确定的可成交性（Execution Certainty）**。
- **状态同步的毫秒级博弈**：求解器不能等区块上链后再去查询合约储备，必须在交易打包前的微秒级时间窗口内，实时在本地内存中精准模拟所有相关合约的状态演化。

#### 3. Propeller Heads 的全栈破局利器：Tycho、Fynd 与 Turbine
针对上述瓶颈，Markus 团队打造了模块化的求解基础设施：
- **Tycho（超高速链上状态索引引擎）**：
  - 传统的 RPC（如 `eth_call`）速度太慢，无法支撑复杂的实时图遍历搜索。
  - Tycho 能够以亚毫秒级延迟从底层节点提取状态变更，并在内存中构建高保真、零延迟的本地全网流动性状态镜像。
- **Fynd（全局流动性路由与求解引擎）**：
  - 运用先进图论算法与凸优化技术，能够同时跨越传统 CFMM、集中流动性池、PropAMM 以及链下 RFQ 机制，瞬间计算出拆单比例与多跳兑换路径。
- **Turbine（高性能执行与 MEV 保护中枢）**：
  - 负责将求解结果与交易捆绑（Bundle），通过专有私密信道直通顶级区块构建者，确保交易享受顶部优先打包且完全规避夹子套利（Sandwich Attacks）。

#### 4. 以太坊去中心化交易的三足鼎立终局
Markus 对链上流动性架构的终局演化给出了清晰判断：
1. **传统 CFMM（Uniswap 等）**：仍将在**长尾代币（Long-tail Assets）、初创项目无许可冷启动**方面占据绝对主导，因为专业做市商不愿为流动性差、波动不可预测的长尾币承担做市风险。
2. **链下 RFQ（0x, Bebop RFQ）**：在主流资产（ETH、WBTC、稳定币）的**超大额点对点交易（Whale trades）**中占据一席之地，具备零链上滑点和链下风控的优势。
3. **专有做市 AMM（PropAMM）**：将成为**主流资产链上现货兑换与多协议可组合（DeFi Composability）的核心主力**。它兼备了 AMM 的智能合约原子可组合性，又享受了专业做市商主动定价消除 LVR 带来的亚基点价差。

---

### 四、核心术语速查表

| 术语 | 英文全称 | 概念定义与微观机制 |
| :--- | :--- | :--- |
| **PropAMM** | Proprietary AMM | 专有做市 AMM，由特定做市商独家拥有合约参数动态更新权的主动流动性池。 |
| **Tycho** | Tycho Indexer | Propeller Heads 开源的亚毫秒级区块链本地状态极速索引与同步引擎。 |
| **Fynd** | Fynd Router | 针对链上异构流动性池进行多跳全局最优路径搜索的专业求解算法引擎。 |
| **Turbine** | Turbine Execution | 具备防 MEV 夹资与零回滚保证的高性能链上交易打包与执行中间件。 |
| **Execution Certainty** | 执行确定性 | 求解器在规划兑换路径时，确保做市商流动性不会因突发改价或撤单而导致交易失败的可靠性度量。 |
| **State Contention** | 状态冲突 | 多笔交易试图读写同一流动性合约存储槽时引发的执行阻塞与依赖问题。 |
| **Tilting at Windmills**| 堂吉诃德战风车 | 播客标题双关语，既指 Propeller（螺旋桨/风车）团队直面挑战，也喻示对 PropAMM 狂热叙事的理性审视。 |
