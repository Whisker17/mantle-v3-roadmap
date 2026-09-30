# 轨道 N：OP Stack / Mantle 的可改造面与硬约束

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/N-opstack-mantle-surface.md`

主题：**OP Stack / Mantle 的「可改造面」与「硬约束」** —— 最终方案的**工程现实性检查**。回答：其他轨道提出的链级零件，在 OP Stack（尤其是 Mantle 的具体分叉状态）上到底能不能做、要付什么代价。

## Change

### 1. OP Stack 官方可配置面盘点
一手源：`docs.optimism.io`、`specs.optimism.io`、GitHub `ethereum-optimism/optimism`
- chain config 可调项：block time、gas limit、EIP-1559 参数（denominator / elasticity）、**custom gas token 的现状与限制**
- **Standard Rollup Charter 与 Superchain 合规要求对魔改的约束** —— 关键问题：一旦改 EVM 语义，是否失去 Superchain / standard 认定与共享安全 / 赏金池
- OPCM / 升级流程
- interop 与 shared sequencing 现状
- **op-rbuilder / rollup-boost 的模块化出块 sidecar 可插拔性**（这是"只改 builder 不改共识"的官方通道）
- Flashblocks 的接入路径，以及它**是否要求改共识**

### 2. 改造分级（核心产出）
把改造按侵入性分 4 级，每级给代表零件、工程量、审计面、与上游 divergence 的维护成本：
- **L0 纯合约层**（无需改客户端）
- **L1 改 sequencer / builder 策略**（排序、lane、赞助、preconf —— 不改状态转换）
- **L2 改执行层**（新 precompile / 新交易类型 / 多维 gas / 系统交易）
- **L3 改共识 / DA / 证明系统**（新 fraud proof program、换栈）

**重点论证 L1 与 L2 的分界**：哪些能力可以「**只改排序器就拿到**」而不破坏 EVM 等价性与 fault proof —— 这是本研究给委托方**最有用的一条工程结论**。

### 3. fault proof 的紧箍咒
- 加自定义 precompile / 改 gas 语义会如何影响 op-program / Cannon / Kona 的可证明性
- **Mantle 当前的 proof 状态**（是否已上 fault proof、L2Beat stage 评级）与它给魔改留下的空间
- 若 Mantle 尚无 fault proof，说明这反而**降低**了魔改成本 —— 务必核实并给证据（这是个反直觉但很重要的结论）

### 4. Mantle 的具体分叉现状（一手 + 【实测】）
- op-geth / op-node 的分叉点与已有自定义
- **MNT 作为原生 gas token 的实现方式**（token ratio / gas oracle 机制、`GasPriceOracle` 合约的 Mantle 改动）
- EigenDA 集成方式
- **Mantle v2 Tectonic 的改动清单**
- 是否有 SP1 / Succinct zk 相关工作
- 已有的非标准 RPC / EVM 行为
- **Mantle 已经改过哪些东西 = 它证明过自己有多大改造能力**，这是可行性的直接证据。尽量读 GitHub `mantlenetworkio/*` 的实际 diff 或 fork 说明

### 5. Mantle 2026 的 infra 路线图现状
- 官方是否已宣布任何 preconfirmation / 高性能 / app-specific 方向的计划（第一阶段结论是"完全缺席"，**请核实至 2026-09 是否仍然成立**）
- MNT 经济学与 sequencer 收入现状（改造的 ROI 分母 —— 如果 sequencer 年收入只有几十万美元，那改造的商业理由必须来自别处）

### 6. 三条路线对比
- **另开一条 app-specific 链**
- **改造主链**
- **主链 + 专用 rollup / 专用 lane**

每条给：工程成本 / 生态割裂代价 / 流动性与桥的影响 / 监管合规影响 / **可退回性（失败了能不能撤）**

第一阶段已给出一条硬结论 —— 「为 meme 单开 appchain 会切断与 xStocks 流动性的连接，而那正是全部价值所在」。请在此基础上做**更细的论证**，并**检验它是否仍然成立**（若你找到反驳理由，直接写出来）。

### 7. 实测
直连 Mantle 主网 RPC 复核关键参数，标【实测】并附可复现请求体：
- chainId、连续采样出块间隔、gasLimit、gasUsed / 填充率
- baseFee 是否仍钉在下限
- `web3_clientVersion` 暴露的分叉版本
- 是否存在任何非标准 RPC 方法（试 `mantle_*`、`eth_getBlockReceipts` 等）

第一阶段实测值需要复核是否仍成立：**出块 2.000s、填充率 0.173%、base fee 钉在 50 gwei 下限、单笔 swap $0.004–0.009**。

## Acceptance

- 文件已创建，含 7 节
- **改造分级表（L0–L3）**齐全，每级至少 4 个代表零件，并标注「是否破坏 EVM 等价性 / fault proof / Superchain 认定」
- Mantle 已有自定义清单有一手来源或源码级证据
- 三条路线对比表齐全
- 实测复核数据齐全并附请求体
- 「存疑清单」不少于 **8** 条
- **显式回答**：**「只改排序器 + 加系统合约」能拿到 Rise 式体验的百分之多少？** 给出有依据的估计与依据链条
