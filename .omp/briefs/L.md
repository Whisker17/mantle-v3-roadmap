# 轨道 L：「资产发行原生（issuance-native）」链级原语盘点

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/L-issuance-native-primitives.md`

主题：**「资产发行原生」的链级原语盘点** —— 本研究最终方案的**核心素材库**。回答：如果一条链要把「资产发行 + 早期流动性 + 交易」做成**链原生能力**，历史上 / 现存有哪些原语可抄。

## Change

### 1. 原生代币标准的能力边界
- **Solana Token-2022 / Token Extensions 逐扩展拆解**：transfer hook、transfer fee、confidential transfer、metadata pointer / metadata、permanent delegate、default account state、interest-bearing、immutable owner、non-transferable、CPI guard、memo required、group/member 等 —— **每一条都要落到"launchpad 能用它做什么"**
- 对照 **EVM 的 ERC-20 + hooks 缺失**：ERC-777 / ERC-1363 / ERC-1155 / ERC-7579 之类的失败史与为什么没被采纳
- **Sui / Aptos** 的 coin / object 模型（Aptos Fungible Asset standard 的 dispatchable hooks 值得专门看）
- **Cosmos tokenfactory**（Osmosis / Injective 的参数：创建费、denom 命名空间、admin 权限、before-send hook）
- **结论要落到**：EVM 缺的到底是哪几项能力，能否用 precompile / 系统合约 / 改 EVM 补上

### 2. 链级原生发行与上币额度
- **Hyperliquid HIP-1 荷兰拍卖上币额度 + HIP-2 原生做市**（与 J 轨道交叉，你从「**发行经济学**」角度拆：拍卖如何充当反垃圾闸门、额度稀缺如何定价、HIP-3 的保证金门槛如何筛选发行人）
- Injective 的 permissions / RWA module
- **任何把 bonding curve 做进链级模块的先例** —— 务必检索 Cosmos SDK module 生态、Substrate pallet 生态、Move 生态、以及 Penumbra 的批量拍卖。若确实没有先例，**明确写"无先例"并说明检索范围** —— 这是重要发现
- Osmosis 的 CosmWasm launchpad 算不算链级？辨析「链级模块」vs「链上合约」的边界

### 3. 原生流动性与激励
- 链级 AMM / DEX 模块：Osmosis 的 GAMM、Injective exchange module
- **Sei 曾有的 DEX module 与它被弃用的原因** —— 这个负面案例**极重要**，务必查清官方归因原话
- **Berachain PoL（把流动性激励写进共识奖励）** 的机制与效果数据（含 PoL 2.0 之类的修订）
- 链级流动性挖矿 / 发行补贴的实现方式

### 4. 发行类应用最痛的链级缺口
结合第一阶段结论（抗狙击、毕业迁移、流动性锁定、费用分配是四大痛点）：
- **抗狙击的链级手段**：发行后 N 区块内的准入控制、per-account rate limit、首块特殊排序、拍卖式首发 —— 有无生产先例？
- **原子发行**：部署 + 建池 + 注入流动性 + 锁仓在一个不可分割步骤内的链级支持
- **费用路由原生化**：每笔交易按规则自动分给创作者 / 协议 / 回购，无需应用层 hook —— 有无链级先例？（对照 Solana transfer fee extension、Token-2022）
- **元数据与索引原生化**：链级 token metadata registry、原生事件索引 / 流式推送（Solana Geyser、Aptos indexer API、Sui 的 checkpoint stream）—— launchpad 前端体验的隐形基础设施，EVM 侧靠什么（Envio / Ponder / eth_subscribe 的局限）
- **反 rug 的链级原语**：LP 锁仓、mint 权限冻结、时间锁的链级表达

### 5. 发行侧的链级合规 / 许可闸门
- **Arbitrum ArbOS Elara**（sequencer 级合规过滤 —— 第一阶段已提及，此处深挖机制与生产状态）
- Injective permissions module
- Canton / Provenance 类许可链的做法
- 回答：「**一级 KYB 闸门 + 二级无许可**」能否在链级表达

### 6. 总表（必写）
**issuance-native 原语目录** —— 列：
| 原语 | 现有实现（链 + 上线日期） | 一手来源 | 若在 EVM L2 实现的落地形态 | 对 launchpad UX 的可感知收益 |

- 「落地形态」四档：`合约层` / `系统合约 + precompile` / `改 EVM 执行层` / `改 sequencer`
- 「可感知收益」必须写成**用户能感知的具体动作**，例如"发币少签一次名"、"狙击者拿不到第一个区块"、"创作者手续费自动到账不用 claim"—— 禁止写"提升用户体验"这种空话

## Acceptance

- 文件已创建，含 6 节 + 总表（**不少于 18 条原语**）
- Token-2022 的扩展**逐项覆盖**且每项都落到 launchpad 用例
- 明确回答「链级 bonding curve / 链级发行模块」**是否有先例**；有则给 URL，无则写明检索范围与"无先例"判定
- 至少覆盖**一个负面案例**（如 Sei DEX module 弃用）并给出归因
- 「存疑清单」不少于 **8** 条
- **显式回答**：**EVM 相对 Solana 在「资产发行」这件事上，缺失的能力清单是什么？哪几项可以用链级改造补齐、哪几项补不了？**
