# PropAMM (Proprietary Automated Market Makers) 播客研究专题库

本专题库系统收录了以太坊交易供应链、MEV 机制设计、市场微观结构与专有做市商（PropAMM / BopAMM）的 **4 期核心技术播客**。全套资料包含精确时间戳的**全集中英逐段双语对照转录**，以及针对专业研究员和开发者的**核心技术深度解读备忘录**。

---

## 4 期播客核心矩阵与全景速览

| 目录与主题 | 嘉宾与机构 | 核心视角与技术标签 | 时长与状态 | 核心产物导航 |
| :--- | :--- | :--- | :--- | :--- |
| **[ep-propamms-won-katia-banina/](ep-propamms-won-katia-banina/)**<br>*PropAMMs Won, You Just Didn't Notice* | **Katia Banina**<br>Bebop 联合创始人兼 CEO | **做市商与 DEX 聚合协议视角**<br>`BopAMM` · `RFS 连续报价流` · `Top-of-Block 注入` · `可交易预言机` · `金库原子可组合性` | 58m 20s<br>✅ 已完成 | 📄 [深度技术解读](ep-propamms-won-katia-banina/podcast_summary_and_key_takeaways.md)<br>📜 [全集中英逐段转录全文](ep-propamms-won-katia-banina/bilingual_transcript_full.md) |
| **[ep-block-building-propamms-ethereum/](ep-block-building-propamms-ethereum/)**<br>*Block Building & PropAMMs on Ethereum* | **Kubi Mensah**<br>Titan Builder / Gattaca CEO | **底层区块构建者视角**<br>`PBS 机制` · `多维背包算法` · `私有订单流 (POF)` · `LVR 块首对冲` · `延迟军备竞赛` | 54m 22s<br>✅ 已完成 | 📄 [深度技术解读](ep-block-building-propamms-ethereum/podcast_summary_and_key_takeaways.md)<br>📜 [全集中英逐段转录全文](ep-block-building-propamms-ethereum/bilingual_transcript_full.md) |
| **[ep-propamms-are-eating-defi-quintus/](ep-propamms-are-eating-defi-quintus/)**<br>*PropAMMs are eating DeFi* | **Quintus**<br>Flashbots 核心研究员 | **交易供应链与机制理论视角**<br>`LVR 损失剖析` · `独占订单流 (EOF)` · `构建者寡头垄断` · `Uniswap v4 钩子终局` · `OFA 价值返还` | 50m 51s<br>✅ 已完成 | 📄 [深度技术解读](ep-propamms-are-eating-defi-quintus/podcast_summary_and_key_takeaways.md)<br>📜 [全集中英逐段转录全文](ep-propamms-are-eating-defi-quintus/bilingual_transcript_full.md) |
| **[ep-tilting-at-propamms-markus/](ep-tilting-at-propamms-markus/)**<br>*Tilting at PropAMMs* | **Markus Schmitt**<br>Propeller Heads 创始人 | **DEX 路由求解器 (Solver) 视角**<br>`Solana vs EVM 架构分流` · `执行确定性` · `Tycho 状态索引` · `Fynd 路由` · `Turbine 结算` | 66m 09s<br>✅ 已完成 | 📄 [深度技术解读](ep-tilting-at-propamms-markus/podcast_summary_and_key_takeaways.md)<br>📜 [全集中英逐段转录全文](ep-tilting-at-propamms-markus/bilingual_transcript_full.md) |

---

## 各期详细内容索引

### 1. [ep-propamms-won-katia-banina/](ep-propamms-won-katia-banina/)
> **"It already happened, propAMMs won, you just didn't notice."**
- **播客来源**：《Deeply Intents》Ep. 47
- **音频时长**：58 分 20 秒
- **核心论点**：传统被动 LP 在以太坊 12 秒出块下必然承受 LVR 剥削。Bebop 联合 Flashbots 与顶级做市商（1010 Trading 等）推出 BopAMM，通过构建者在块首写入真金白银背书的“可交易预言机”（Transactable Oracle），为大额交易提供亚基点（Sub-bps）价差，并将 DeFi 推向单笔交易可组合的“金库级资金乐高”。
- **产物文档**：
  - [核心深度解读与技术备忘录](ep-propamms-won-katia-banina/podcast_summary_and_key_takeaways.md)
  - [全集完整中英逐段转录全文](ep-propamms-won-katia-banina/bilingual_transcript_full.md)

---

### 2. [ep-block-building-propamms-ethereum/](ep-block-building-propamms-ethereum/)
> **"From searching to building: inside the multi-dimensional knapsack of Ethereum."**
- **播客来源**：《Credible Commitments》
- **音频时长**：54 分 22 秒
- **核心论点**：作为以太坊主网出块份额第一梯队的 Titan Builder，Kubi Mensah 从底层剖析了构建者的微薄利润与补贴残酷竞争。构建者如何通过在区块顶部（Top-of-Block）协调 PropAMM 价格同步，消弭外部套利者对陈旧报价的攻击，进而将套利空间转化为可持续的独家私有订单流（POF）。
- **产物文档**：
  - [核心深度解读与技术备忘录](ep-block-building-propamms-ethereum/podcast_summary_and_key_takeaways.md)
  - [全集完整中英逐段转录全文](ep-block-building-propamms-ethereum/bilingual_transcript_full.md)

---

### 3. [ep-propamms-are-eating-defi-quintus/](ep-propamms-are-eating-defi-quintus/)
> **"Passive liquidity is economically dead on 12-second blocks; active quantitative market making is the only sustainable equilibrium."**
- **播客来源**：《Deeply Intents》
- **音频时长**：50 分 51 秒
- **核心论点**：Flashbots 机制研究员 Quintus 复盘了其在 SBC 2022 年关于独占订单流（EOF）导致构建者中心化的警示。他深刻论证了为什么传统 CFMM 的被动做市必然被专业量化机构的 PropAMM 替代，以及 Uniswap v4 自定义钩子（Hooks）在微观经济学上的终局——Uniswap 或将退化为各类专有做市商的底层托管合约框架。
- **产物文档**：
  - [核心深度解读与技术备忘录](ep-propamms-are-eating-defi-quintus/podcast_summary_and_key_takeaways.md)
  - [全集完整中英逐段转录全文](ep-propamms-are-eating-defi-quintus/bilingual_transcript_full.md)

---

### 4. [ep-tilting-at-propamms-markus/](ep-tilting-at-propamms-markus/)
> **"Routing through the fog: why solvers need deterministic execution certainty from PropAMMs."**
- **播客来源**：《Deeply Intents》
- **音频时长**：66 分 09 秒
- **核心论点**：DEX 求解器（Solver）必须在毫秒级时间内跨越 AMM、PropAMM 与 RFQ 进行路径规划。Markus 阐明了 Solana（高频低费直接更新）与以太坊（区块顶部协同更新）的根本差异，揭示了求解器面临的状态冲突与惩罚风险，并介绍了 Propeller Heads 的全栈基础设施（Tycho 毫秒索引、Fynd 图求解、Turbine 防夹执行）。
- **产物文档**：
  - [核心深度解读与技术备忘录](ep-tilting-at-propamms-markus/podcast_summary_and_key_takeaways.md)
  - [全集完整中英逐段转录全文](ep-tilting-at-propamms-markus/bilingual_transcript_full.md)

---

## 终局共识与生态认知模型

综合这 4 期播客的行业领袖观点，以太坊交易微观结构的演进脉络已清晰呈现：

```text
┌────────────────────────────────────────────────────────────────────────┐
│                      以太坊流动性与交易终局生态模型                         │
├──────────────────┬──────────────────┬──────────────────────────────────┤
│   长尾资产 / 新币   │   机构大额点对点   │      主流资产现货与深度可组合         │
│ (Long-tail Assets)│  (Whale RFQs)    │     (Blue-chip DeFi Composability)│
├──────────────────┼──────────────────┼──────────────────────────────────┤
│   传统被动 CFMM    │     链下 RFQ     │      专有做市 AMM (PropAMM / BopAMM)│
│ (Uniswap v2/v3/v4)│ (Bebop / 0x RFQ) │   (Block Builder Top-of-Block)   │
├──────────────────┴──────────────────┴──────────────────────────────────┤
│                     底层中枢：DEX 求解器与路由网络                        │
│                   (CoW Swap, 1inch, Propeller Fynd)                    │
├────────────────────────────────────────────────────────────────────────┤
│                     执行层：区块构建者与 PBS 拍卖                       │
│                     (Titan Builder, Flashbots, MEV-Boost)              │
└────────────────────────────────────────────────────────────────────────┘
```
