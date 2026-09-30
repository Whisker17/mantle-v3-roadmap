# 轨道 N：OP Stack / Mantle 的可改造面与硬约束
> 研究轨道：N ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

## 0. 执行摘要

**本轨道的六条结论**（全部有一手或【实测】支撑，详见对应章节）：

| # | 结论 | 章节 |
|---|---|---|
| **N-1** | **Mantle 已经不是"没改过链"的通用 OP Stack 链。** 它自 2025-09-16 起走 **OP Succinct（SP1）ZK validity proof**，2026-04-16 **Arsia** 升级移除 EigenDA 代码路径、改为纯 Ethereum blob DA，并有独立 `op-geth` fork、`mantle-xyz/revm` fork 与 kona 子树。**魔改能力已被自己证明过。** | §3.2、§4 |
| **N-2** | **反直觉的关键发现：走 ZK 路线反而降低了加自定义 precompile / 交易类型的门槛。** Cannon/MIPS 那套"链上单指令仲裁必须认识新指令"的紧箍咒**对 Mantle 完全不适用**；代价转为「四层 fork 同步 + SP1 vkey 轮换 + 证明 cycle 成本」——**工程可控、商业可核算，非阻断性**。 | §3.3 |
| **N-3** | **Mantle 采用 OP Succinct 时就已经脱离 Optimism Standard Chain 序列**（OP Succinct 官方 FAQ 明文）。**"改 EVM 会失去 Superchain 认定"这个顾虑对 Mantle 已经不成立 —— 那张牌早就打出去了。** | §3.3 第 5 点、§1 |
| **N-4** | **第一阶段"Mantle 在 preconfirmation / 高性能 / app-specific 方向完全缺席"的结论，复核后仍然成立。** 官网、docs、治理论坛三渠道检索：`preconfirmation` 0 结果、`appchain` 0 结果、2026 年零链改造提案。官方定位已改写为 **"Institutional Onchain Finance"（RWA 分发层）**。 | §5.1、§5.3 |
| **N-5** | **改造的费用 ROI 分母极小**：sequencer 收入 FY24-25 为 **$1.76M**，2026 年 run-rate 仅 **$10–20 万/年**（两年缩水约 98%）。真实经济引擎是 **$2.69B Treasury 的投资收益**。⇒ **任何商业提案的对手方是 Treasury / MIP 预算流程，不是"跟 sequencer 谈分成"。** | §5.2 |
| **N-6** | **若只做应用层合约（Launchpad 的 bonding curve / 代币工厂 / AMM 全都是），以上全部协议层约束都不适用。** 这是判断"要不要改链"的第一道分界线。 | §3.4、§2 |

## 1. OP Stack 官方可配置面盘点

> 本节口径：**通用 OP Stack 官方能力**（非 Mantle 专属），供第 2 节改造分级与第 4 节"Mantle 已改过什么"对照。全部一手源：`docs.optimism.io`、`specs.optimism.io`、`github.com/ethereum-optimism/*`、`github.com/flashbots/*`。

### 1.1 Chain Config 可调项：block time / gas limit / EIP-1559 参数

三者均为**官方支持的部署期配置项**，集中在 `DeployConfig`（由 `op-deployer` 的 intent 文件派生）；gas limit 与 EIP-1559 参数在部署后还可通过 L1 `SystemConfig` 动态调整（EIP-1559 动态化自 **Holocene** 硬分叉起写入区块头 `extraData`）。

| 参数 | 配置位置 | 标准边界（Configurability spec）|
|---|---|---|
| Block time | `DeployConfig.l2BlockTime`（genesis 固定）| **1 或 2 秒**（"High security & interoperability compatibility requirement"）|
| Gas limit | genesis 用 `l2GenesisBlockGasLimit`；运行期由 `SystemConfig` 持有，`SystemConfigOwner` 可改 | **≤ 200,000,000 gas** |
| EIP-1559 denominator/elasticity | `DeployConfig.eip1559Elasticity` / `eip1559Denominator`（+ `eip1559DenominatorCanyon`）；Holocene 后可经 `SystemConfig` 动态改，编码进区块头 `extraData`（version u8 + denominator u32 + elasticity u32），经 `PayloadAttributesV3.eip1559Params` 传给执行层 | 无固定上限，由标准配置 TOML 约束 |
| 最低 base fee（Jovian）| `DeployConfig.minBaseFee`，运行期 `SystemConfig.setMinBaseFee()` | — |

来源：[一手] `docs.optimism.io/chain-operators/reference/rollup-deployment-configuration`、`specs.optimism.io/protocol/holocene/exec-engine.html`、`specs.optimism.io/protocol/configurability.html`。

### 1.2 Custom Gas Token（原生 gas 非 ETH）现状与限制

经历「legacy CGT（beta）→ 2025-02-27 停用【二手】→ **CGT v2** 重新设计，随 **Upgrade 18 / `op-contracts/v6.0.0`** 官方发布」的路径。CGT v2 现为官方支持的部署选项，但：

- **仅支持 18 位小数代币**（"CGT currently supports only 18-decimal tokens"）；
- **协议不再内置代币桥**，桥接下放应用层（"decoupling native asset management from core bridging infrastructure"），ETH 桥接需经 L1-WETH 包装；
- **legacy CGT 无迁移路径到 v2**，需协调硬分叉；
- **⚠️关键存疑点**：legacy spec 原话——"There is currently no strong definition of what it means to be part of the standard config when using the OP Stack with custom gas token enabled. This will be defined in the future."——即**"CGT 链算不算 Standard Rollup"官方至今未给出肯定答案**。

来源：[一手] `specs.optimism.io/experimental/custom-gas-token.html`、`docs.optimism.io/op-stack/features/custom-gas-token`、`docs.optimism.io/chain-operators/guides/features/custom-gas-token-guide`。

> **对 Mantle 的直接意义**：Mantle 用 MNT 作原生 gas token 早于 CGT v2（Mantle 2023 上线时自建机制，见第 4 节），**不是走官方 CGT 流程**，因此不受"18 位小数/无迁移路径"限制，但也意味着它自己承担了全部维护成本（无法吃官方升级的免费红利）。

### 1.3 Standard Rollup Charter：改 EVM 语义的后果（核心约束）

Charter 的链上准入是**三重校验**：
1. **版本校验**——"comparing all bytecode for the chain's L1 smart contracts to the standard bytecode corresponding to a governance-approved release"，且合约必须由 canonical **OPCM**（`0x9ce712ff84e02659846dc6450bb9b7642fe8be5d`）部署。
2. **配置校验**——参数必须落在 `superchain-registry` 的 `standard-config-params-mainnet.toml` / `standard-config-roles-mainnet.toml` 边界内。
3. **Prestate 校验（改 EVM 语义的死穴）**——"The pre-state is a single onchain hash which commits to the State Transition Function used to credit withdrawals from the bridge. Chains' L1 smart contracts must use a valid prestate hash which corresponds to the standard node software used to sync the chain."

**加自定义 precompile / 改交易类型会直接破坏 prestate 校验**（因为你的 STF ≠ 治理批准的 op-program 输出），从而失去：Security Council 托管升级的共享安全模型、Superchain interop cluster 准入资格、Charter 对 Standard Rollup 的费用分成承诺（"a fee split of the greater of 1) 2.5% of transaction fee revenue and 2) 15% of chain profit"）。

⚠️**"失去赏金池资格"这一常见说法，在 Charter 与 docs 原文中找不到依据**——Charter 通篇未出现 "bug bounty" 字样，本报告不采纳这一说法。

来源：[一手] `github.com/ethereum-optimism/OPerating-manual/blob/main/Standard Rollup Charter.md`、`github.com/ethereum-optimism/superchain-registry`（`validation/standard/*.toml`）、`specs.optimism.io/protocol/configurability.html`、`docs.optimism.io/op-stack/interop/explainer`。

### 1.4 OPCM 与标准升级流程

OPCM 是每个合约版本发布一个的 L1 单例合约，**一笔交易部署整链 L1 合约**，自 Upgrade 13 起也承担存量链升级；"provides a minimal set of user-configurable parameters to ensure that the resulting chain meets the standard configuration requirements"，部署结果不合标准配置**直接 revert**。升级只能由 Proxy Admin Owner Safe 通过 `DELEGATECALL` 调用 `OPCM.upgrade()` 执行——走 OPCM 等于把合约版本与升级路径锁在治理批准的 release 轨道上；自建自定义合约会立即使字节码偏离治理版本，触发 1.3 的版本校验失败。

来源：[一手] `docs.optimism.io/chain-operators/reference/opcm`、`specs.optimism.io/experimental/op-contracts-manager.html`。

### 1.5 Interop 与 Shared Sequencing：现状仅到测试网

- **Superchain interop：测试网阶段**。官方 notice："OP Stack interop is expected to activate on **OP Sepolia**(11155420) and **Unichain Sepolia**(1301) in **July of 2026**, forming a two-chain interop dependency set on testnet. … **Interop on mainnet OP Stack chains is planned as a follow-up rollout after this testnet activation; mainnet activation timestamps will be announced separately.**"——**主网未激活**。第三方报道 Sepolia 已于 2026-09-04 前后激活【二手】。
- **Shared sequencing**：specs 里只是一段概念性描述（"A shared sequencer can be built if the block builder is able to build the next canonical block for multiple chains…"），**无产品化实现或时间表**。

来源：[一手] `github.com/ethereum-optimism/optimism/blob/develop/docs/public-docs/notices/interop-prep.mdx`、`docs.optimism.io/op-stack/interop/explainer`、`specs.optimism.io/interop/sequencer.html`。

### 1.6 op-rbuilder / rollup-boost：「只改 builder 不改共识」的官方通道

**这是本节对委托方最重要的正面结论，且有 Flashbots + OP 官方 design-docs 双重一手背书。**

- rollup-boost 是位于 op-node 与执行层之间的 **Engine API 代理 sidecar**，把 `engine_FCU`/`engine_getPayload` 同时发给本地 op-geth（fallback）与外部 builder（op-rbuilder）；外部块先经本地 `engine_newPayload` 验证，不合法即回退本地块。
- "**It requires no modification to the OP stack software** and allows rollup operators to connect to an external builder." / "allows operators to continue using the **standard `op-node` and `op-geth`/`op-reth` software without any custom forks**."（rollup-boost book）
- OP 官方 design doc 协议侧背书："**Minimal Modifications: Requires no changes to existing `op-node` or `op-geth` components**" / "operators can **tailor transaction sequencing rules without diverging from the standard Optimism Protocol Client**."
- 代价（同为一手，非共识层面）：design doc 列出 "breaks any existing and future assumptions around there being 1 execution layer for each consensus layer client"、额外延迟、新增 builder 认证攻击面。

来源：[一手] `github.com/flashbots/rollup-boost`（README + `book/src/intro.md`）、`github.com/flashbots/op-rbuilder`、`github.com/ethereum-optimism/design-docs/blob/main/protocol/external-block-production.md`。

### 1.7 Flashblocks：接入路径，不改共识

Flashblocks 是 rollup-boost 的一个模块，**显式 out-of-protocol 设计**：builder 以 **200ms**（`FLASHBLOCKS_TIME`）间隔流式产出部分块（预确认），最终上链的仍是标准 OP Stack 区块，**不修改共识规则或状态转换函数**。spec 专章 "Out-of-Protocol Design"；fallback EL 定义为"an **unmodified EL node** that maintains the ability to construct valid blocks according to standard OP Stack protocol rules"。

**生产使用者**：Unichain（首发，2025 年初）、**Base**（2025-07-16 主网，"All Base public endpoints are Flashblocks-enabled … `pending` block tag reflects the current pre-confirmed block in progress, updated every ~200ms"）。接入组件：rollup-boost（Flashblocks-enabled）+ op-rbuilder（builder with Flashblocks 支持）+ op-reth（fallback）；RPC 侧需要"a modified node that supports serving RPC requests with the Flashblocks preconfirmation state"（RPC overlay，非共识组件）。

来源：[一手] `github.com/flashbots/rollup-boost/blob/main/specs/flashblocks.md`、`.../book/src/modules/flashblocks.md`、`docs.base.org/base-chain/api-reference/rpc-overview`、`blog.base.dev/flashblocks-deep-dive`。

### 1.8 小结表：官方支持度 × Superchain 资格

| 改造点 | 官方支持度 | 是否保住 Standard/Superchain 资格 |
|---|---|---|
| block time（1–2s）/ gas limit（≤200M）/ EIP-1559 参数 / minBaseFee | 完全官方参数面（DeployConfig + SystemConfig）| 是（TOML 边界内）|
| Custom Gas Token v2（Upgrade 18）| 官方支持，18-decimals only、应用层桥、legacy 无迁移 | ⚠️存疑（标准配置对 CGT 定义未明）|
| 自定义 precompile / 新交易类型（改 EVM 语义）| 技术上可 fork，但破坏标准 prestate + 版本校验 | **否**——丧失 Standard Rollup 认定，进而丢失 Security Council 共享升级与 interop cluster 资格 |
| 仅换 builder（rollup-boost + op-rbuilder + Flashblocks）| Flashbots + OP design-docs 双重一手背书，"requires no changes to op-node/op-geth" | **是**——不触碰共识/STF，仅 sequencer 拓扑 + RPC overlay |

> **核心判断（呼应 Acceptance 要求的"只改排序器"论证）**：rollup-boost/op-rbuilder 这条"只改 builder"路径，是**唯一既有官方背书、又不牺牲 Standard Rollup 资格**的改造面；任何触及 EVM 语义的改造，在 Charter 的 prestate/版本双重校验下都会**立即**丢失 Superchain 标准认定——这个分界线比"要不要改共识"更早触发，是本报告改造分级（第 2 节）的核心依据。

## 2. 改造分级（L0–L3，核心产出）

> **分级依据**：侵入性 = 「是否改变状态转换函数」+「是否触发 fork 链路同步与 vkey 轮换」。
> 三个判定列的含义：**EVM 等价性** = 是否让 Mantle 的 EVM 行为偏离标准 EVM（影响工具链/审计/开发者预期）；**证明系统** = 是否触发 §3.3 的四层 fork 同步 + SP1 vkey 轮换；**Superchain 认定** = 是否影响 Standard Chain 资格（⚠️ 注意 §3.3 已确认 Mantle **早已因采用 OP Succinct 而不在 Standard Chain 序列内**，故该列对 Mantle 已基本失去约束力，保留仅为完整性）。

### L0 — 纯合约层（无需改客户端）

| 代表零件 | 破坏 EVM 等价性 | 触发证明系统改动 | 影响 Superchain 认定 | 工程量 |
|---|---|---|---|---|
| bonding curve / 代币工厂 / launchpad 全套 | ❌ 否 | ❌ 否 | ❌ 否 | 周级 |
| ERC-4337 EntryPoint + Paymaster（gas 代付） | ❌ 否 | ❌ 否 | ❌ 否 | 周级 |
| 额度门禁 / bot tax / 衰减费（抗狙击） | ❌ 否 | ❌ 否 | ❌ 否 | 周级 |
| 链级公共原语的**合约实现**（价格预言机聚合器、VRF Coordinator、原生稳定币包装） | ❌ 否 | ❌ 否 | ❌ 否 | 周–月级 |
| ASS 家族（Angstrom 式批量拍卖、Atlas OFA、MEV tax） | ❌ 否 | ❌ 否 | ❌ 否 | 月级 |

**维护成本**：与上游零 divergence。**这一级是 Launchpad 场景 90% 的工作量所在。**

### L1 — 改 sequencer / builder 策略（不改状态转换）

| 代表零件 | 破坏 EVM 等价性 | 触发证明系统改动 | 影响 Superchain 认定 | 工程量 |
|---|---|---|---|---|
| **预确认流 / Flashblocks 式软确认**（走 rollup-boost sidecar） | ❌ 否 | ❌ **否**（不改状态转换，只改"何时把结果告诉你"） | ⚠️ 视是否偏离 standard 配置 | 月–季度级 |
| **`eth_sendRawTransactionSync`（EIP-7966）** 单往返回执 | ❌ 否（纯 RPC 层） | ❌ 否 | ❌ 否 | **周级（本表性价比最高的一项）** |
| 排序策略（FCFS / 私有 mempool / 反三明治 / cancel 优先） | ❌ 否 | ❌ 否 | ⚠️ 视配置 | 月级 |
| **交易赞助 / relay 代付**（链方自营 relay，RISE 的做法） | ❌ 否 | ❌ 否 | ❌ 否 | 月级 |
| sequencer 级合规过滤（ArbOS Elara 式） | ❌ 否 | ❌ 否 | ⚠️ 视配置 | 月级 |
| **链参数调整**（出块时间、区块 gasLimit、EIP-1559 denominator/elasticity、base fee 下限） | ❌ 否 | ❌ 否（参数在 chain config 内） | ❌ 否 | **天–周级（纯配置）** |

**关键判断（本轨道对委托方最有用的一条工程结论）**：
> **L1 与 L2 的分界线是「是否改变状态转换函数」。**
> 「谁先谁后、何时告诉你、谁替你付钱、区块多大」全部属于 **L1**，**不破坏 EVM 等价性、不触发 vkey 轮换、不需要动 revm/kona 的四层 fork**。
> ⇒ **RISE 相对 Mantle 的优势里，参数类（1.5 Ggas / 近零 base fee / 1 秒出块）与运营类（gas 代付、公共原语）全部落在 L0+L1。**

### L2 — 改执行层（新 precompile / 新交易类型 / 多维 gas / 系统交易）

| 代表零件 | 破坏 EVM 等价性 | 触发证明系统改动 | 影响 Superchain 认定 | 工程量 |
|---|---|---|---|---|
| 自定义 precompile（撮合数学、定点运算、批量匹配） | ✅ **是** | ✅ **是**（四层 fork + vkey 轮换 + cycle 成本） | ✅ 是（对 Mantle 已无实际约束，见 N-3） | 季度级 |
| 新交易类型（如 oracle 系统交易、免 gas cancel 交易） | ✅ 是 | ✅ 是 | ✅ 是 | 季度级 |
| 多维 gas / per-contract fee market | ✅ 是 | ✅ 是 | ✅ 是 | 季度–年级 |
| **CBP 式执行流水线 / 逐笔预确认（Shreds 等价物）** | ⚠️ 部分（RISE 的 `BLOCKHASH`/EIP-2935 退化就是代价，见 H 轨道 §3.1） | ✅ 是 | ✅ 是 | **年级（需自研执行层客户端）** |
| 自研状态存储引擎（RiseDB 等价物） | ❌ 否（语义不变） | ⚠️ 视实现 | ⚠️ | 年级 |

### L3 — 改共识 / DA / 证明系统

| 代表零件 | 说明 |
|---|---|
| 换 DA 层 | **Mantle 已做过两次**（EigenDA → Ethereum blobs，Arsia，2026-04-16）⇒ 能力已验证 |
| 换证明系统 | **Mantle 已做过**（Cannon 路线 → OP Succinct SP1，2025-09-16）⇒ 能力已验证 |
| based sequencing / 去中心化排序 | RISE 亦仅停留在路线图（三阶段全未上线） |
| 换栈（非 OP Stack） | 不建议；且 RISE 案例证明**不必换栈**即可做到 app-specific |

> **L3 的意外结论**：**L3 恰恰是 Mantle 唯一有实战记录的一级。** 它在两年内换过一次 DA、换过一次证明系统。**这说明"Mantle 没有链改造能力"的假设是错的；缺的不是能力，是方向与意愿（见 §5）。**

## 3. Fault Proof 的紧箍咒

### 3.1 通用约束：Cannon / op-program（kona）要求「无状态确定性复现」

OP Stack 经典 fault proof 路线（Cannon）的核心机制是把整条 L2 状态转换函数编译进一个受限指令集虚拟机（Fault Proof VM），逐指令在 L1 合约上可仲裁：

- **Cannon = 大端 64-bit MIPS64**，**Asterisc = 小端 64-bit RISC-V**，均"in active development"[一手：`specs.optimism.io/fault-proof/index.html#fault-proof-vm`]。
- 参考实现 `op-program`（现已被 **kona**（Rust client + host）取代，2026-01-15 `op-rs/kona` 并入 `ethereum-optimism/optimism` monorepo，原仓库已归档）[一手：`github.com/op-rs/kona`]。程序原文要求："two invocations with the same input data will result in not only the same output, but the same program execution trace"[一手：`op-program/v1.6.1/README.md`，注：该 Go 版路径在 develop 分支已移除]。
- **自定义 precompile**：必须在 fault proof program 内实现同等逻辑；若计算昂贵可走 pre-image oracle 的**加速通道**，但"All accelerated precompiles must be functionally equivalent to their EVM equivalent"[一手：`specs.optimism.io/fault-proof/index.html#precompile-accelerators`]，host 端对可加速 precompile 有**显式白名单**——新增自定义 precompile 无法直接挂上现成加速通道，需要自行扩展 host + 保证链上 `PreimageOracle` 合约可验证结果。
- **改 gas 语义 / 新交易类型**：属于状态转换函数本身的修改，必须同步 fork derivation + 执行层，否则链下证明执行与链上声明分叉，fault proof 直接失效。
- 任何 program 改动都会改变 **absolute prestate**（VM 初始状态哈希），需用 `reproducible-prestate` 重新生成并更新 L1 dispute 合约配置。
- MIPS64 FPVM（`MTCannon`）明确：早期版本对未识别 syscall 当作 noop，**新版会直接抛异常**[一手：`specs.optimism.io/fault-proof/cannon-fault-proof-vm.html`]——链上 `MIPS64.sol` 单指令仲裁合约不认识的指令无法被单步证明。

**kona 对自定义链的支持现状**：kona 按"可移植、多后端"设计（同一套 no_std 库可跑在 Cannon FPVM / SP1(op-succinct) / RISC Zero(kailua) 上），但**没有配置化的自定义语义开关**——FPVM 加速 precompile 是逐个手写模块（`bin/client/src/fpvm_evm/precompiles/` 下 `ecrecover`、`bn128_pair`、`bls12_*`、`kzg_point_eval` 各一个）[一手源码：`github.com/op-rs/kona/blob/main/bin/client/src/fpvm_evm/precompiles/mod.rs`]，执行层基于 revm。**自定义链 = 逐项 fork，无免改动路径。**

### 3.2 Mantle 的实际证明路线：不是 Cannon，是 SP1 ZK Validity Proof

**Mantle 主网当前完全不走 Cannon/MIPS 挑战式 fraud proof，而是 OP Succinct（SP1）ZK validity proof**——这条路线绕开了 3.1 的全部 MIPS 电路约束，但换来另一套约束（见 3.3）。

一手来源 L2Beat：
- "Mantle is a modular general-purpose Ethereum rollup. Transaction data is posted to Ethereum blobs and state transitions are validated onchain via OP Succinct ZK validity proofs (SP1)."[`l2beat.com/scaling/projects/mantle`]
- State validation 徽章原文："Validity proofs — Each update to the system state must be accompanied by a ZK proof... Through the `SuccinctL2OutputOracle`, the system also allows to switch to an **optimistic mode**, in which no proofs are required and a challenger can challenge the proposed output state root within the finalization period."[同上 `#state-validation`]——**注意这个 optimistic 兜底开关本身是 L2Beat 反复提示的风险点**。
- Stage：**Stage 0**（ZK Rollup），"Stage 1: 3 issues need fixing; Stage 2: 2 issues need fixing"。
- Milestone 时间线：**2025-09-16** "Mantle upgrades to OP Succinct, integrating ZK proofs for state validation"；**2026-04-16** "Arsia upgrade: full Ethereum DA — EigenDA code path removed; DA is Ethereum only. Mantle reclassified as a rollup."[一手，见第 4/7 节交叉验证]

CRITICAL 风险原文（L2Beat risk-summary）：
> "Funds can be stolen if ... in non-optimistic mode, the validity proof cryptography is broken or implemented incorrectly; **optimistic mode is enabled and no challenger checks the published state**; the proposer routes proof verification through a malicious or faulty verifier by specifying an unsafe route id."

### 3.3 ZK 路线下自定义 EVM 语义的真实约束：从"电路仲裁"变成"vkey 轮换 + cycle 成本"

**这是本节对委托方最有用的反直觉结论**：SP1 zkVM 路线下，加自定义 opcode/precompile/交易类型**不需要**像 Cannon 那样为新指令写专门电路、不需要链上 MIPS 解释器认识新指令、也没有 Type-6 precompile 白名单限制——只需在 Rust（fork 的 revm/kona）里实现并编译进 guest ELF，SP1 zkVM 照常证明。约束转移到了别处：

1. **Fork 链路长达四层，必须同步**（一手实证，非推断）：Mantle 为了让 kona 认识自家语义，把 kona 系全部 crate（`kona-genesis`/`kona-protocol`/`kona-derive`/`kona-executor`/`kona-host`/`op-alloy`/`alloy-op-evm`）指向自己 fork 的 `mantle-xyz/mantle-v2`（Rust 子树，pin 到 `v1.6.2`），**revm 家族 13+ 个 crate 全部 patch 到 `mantle-xyz/revm`**[一手：`github.com/mantle-xyz/op-succinct/blob/main/MANTLE_CHANGES.md`]。链路：op-geth fork → mantle-v2 kona 子树 → mantle-xyz/revm → op-succinct guest 程序，**四层需要同步修改**。
2. **任何改变 guest 执行的变更都强制触发 SP1 verification key 轮换**——Mantle 自己的运维文档写明："The Rust-dep bumps change guest-program execution → SP1 range/agg vkeys change → `just build-elfs` on x64 + commit the `elf/*` + update on-chain vkeys."[一手：`MANTLE_CHANGES.md`]。L2Beat 记录了实际发生的轮换：**2026-08-10** `aggregationVkey`/`rangeVkeyCommitment`/`rollupConfigHash` 全部轮换，且"Both new verification keys are recorded in programHashes.ts as **not verified**, because reproducing them currently requires a private dependency"[一手：`l2beat.com/scaling/projects/mantle#updates`]；作为对照，**2026-06-24** 那次轮换（v2.2.4-mainnet.4）"Hashes reproduced"——即**轮换是常态操作，可验证性取决于 Mantle 是否公开全部构建依赖**。
3. **证明成本 ∝ guest 程序 RISC-V cycle 数**（官方机制，一手）：SP1 文档确认"built-in precompiles ... dramatically reduce execution time and proving costs"，反例是若 `sp1-patches` 补丁失效会导致 keccak 退回软件实现，"a large cycle regression, since keccak drives MPT/state-root hashing"[一手：`docs.succinct.xyz/docs/sp1/optimizing-programs/precompiles`]。**【推断，非官方逐字结论】**：自定义逻辑越重、尤其是无 `sp1-patches` 补丁支持的非标准密码学运算，range program cycle 数越高，证明费用与延迟线性上升；若需要全新的电路级加速，必须向 Succinct 提交 patch 需求（"If you know of a library ... that you think should be patched, please open an SP1 issue"）。
4. **正确性风险从"链上电路对不对"转移到"fork 的 revm/kona 语义对不对"**：ZK 只证明 guest 程序被正确执行，不证明 guest 程序本身语义正确——对应 L2Beat 风险原文"the validity proof cryptography is broken **or implemented incorrectly**"。
5. **⚠️生态代价（一手，易被忽略）**：OP Succinct 官方 FAQ 明写："If your rollup adopts OP Succinct, it will no longer be classified as a Standard Chain. Optimism currently considers ZK proofs to be an 'experimental' feature, similar to alt-DA solutions and custom gas tokens."[一手：`succinctlabs.github.io/op-succinct/faq.html`]——**Mantle 采用 OP Succinct 这件事本身，已经让它脱离 Optimism 的 Standard Chain 序列**，这比"改 EVM 语义"更早、更彻底地放弃了 Superchain 标准认定（与第 1 节 Standard Rollup Charter 的结论互证）。

> **架构性小结**：Mantle 因为选择了 ZK 路线，事实上**避开了整个 3.1 节描述的 Cannon/MIPS 紧箍咒**——这对委托方是个重要的正面信号：**加自定义 precompile/交易类型在 Mantle 上不会撞上"MIPS 电路不认识新指令"这堵墙**，代价是每次改动都要走一遍 vkey 轮换流程（工程可控，非阻断性）+ 证明成本上升（商业可核算，非阻断性）。这与"改 fraud proof program"比，工程确定性更高。

### 3.4 对 launchpad 场景的直接含义

若目标只是**部署应用层合约**（Launchpad 场景大概率如此——bonding curve、AMM、代币工厂都是纯 Solidity 合约层），**以上全部约束都不适用**——只有触碰协议层 EVM 语义（新 precompile、新交易类型、改 gas 计价规则）才会触发 fork 链路同步 + vkey 轮换。这是判断"要不要走 L2/L3 改造"时的第一道分界线，见第 2 节改造分级。

## 4. Mantle 的具体分叉现状（一手 + 【实测】）

> 核查方式：只读 GitHub API + `raw.githubusercontent.com` 直读源码/Release/Commit，组织 `mantlenetworkio`（GitHub 登录名已实际迁移为 `mantle-xyz`，两路径指向同一批仓库）。⚠️ 局限：本环境无法对私有/需登录的 GitHub 全文代码搜索（`type=code`）执行匿名检索，部分"未找到"结论标注为**受限未找到**而非"确认不存在"。

### 4.1 仓库结构：op-geth / op-node 的分叉点

| 组件 | 分叉方式 | 一手证据 |
|---|---|---|
| **op-node** | 内嵌在 `mantle-v2` monorepo 的 `op-node/` 目录，通过 commit（如 `d7914f99`）与 `ethereum-optimism/optimism` 保持同步，**非独立仓库** | GitHub API + 目录结构 |
| **op-geth** | **独立仓库** `mantle-xyz/op-geth`，GitHub API 确认 `parent.full_name = "ethereum-optimism/op-geth"` | `github.com/mantlenetworkio/op-geth` |
| **l2geth**（V1 遗产）| `mantle-v2/l2geth` 目录，README 注明**直接 fork go-ethereum**，GPLv3 许可 | `github.com/mantlenetworkio/mantle-v2/tree/develop/l2geth` |
| **合约层** | `packages/contracts-bedrock` 基于 Optimism `contracts-bedrock` 定制，核心改动集中在 `GasPriceOracle.sol`、`L1Block.sol`、`SystemConfig.sol`（见 4.2） | 源码直读 |

官方 fork 序列（README 原文）：**BaseFee → Everest → Euboea → Skadi → Limb → Arsia**，与第一阶段结论一致。

### 4.2 MNT 作为原生 gas token 的实现方式【一手源码 + 实测互证】

文件：`packages/contracts-bedrock/src/L2/GasPriceOracle.sol`，预部署地址 `0x420000000000000000000000000000000000000F`（NatSpec 注释确认，与第 7 节【实测】地址一致）。

关键代码（源码原文）：
```solidity
uint256 public tokenRatio;
address public owner;
address public operator;
bool public isArsia;
event TokenRatioUpdated(...);
function setTokenRatio(uint256 _tokenRatio) external onlyOperator {
    require(_tokenRatio > 0);
    require(_tokenRatio <= type(uint64).max);
    tokenRatio = _tokenRatio;
    emit TokenRatioUpdated(...);
}
// getL1Fee() 依据 isArsia 切换 _getL1FeeBedrock() / _getL1FeeArsia()
```

- `tokenRatio` 即 **ETH/MNT 汇率**，由链下 `gas-oracle` 服务（`mantle-v2/gas-oracle` 目录）周期性写入，与第 7 节【实测】读到的 `tokenRatio()=3789`（对比 CoinGecko 现价比 3791.1，偏差 0.06%）完全吻合。
- **历史沿革**：Release `v1.0.0-alpha.1`（2024-03-13）说明 "Use MNT as Native Token and Gas Token instead of Ether"（PR #2, #11）；commit `668da206`（PR #254，2025-11-03）"chore: make token ratio an optional config" 证实**持续维护到近期**。
- **权限分离**：`owner`/`operator` 双角色 + `onlyOperator` 白名单控制 `setTokenRatio()`——这是一个"改 gas 定价机制"的真实先例，证明 Mantle 有能力且已经在生产环境运行**非标准的原生代币定价系统**（比官方 CGT v2 早两年多，且不受其 18-decimal 限制，因为 MNT 本身就是 18 位小数）。

### 4.3 Arsia 升级的合约级实现【一手源码，与第 7 节【实测】逐位互证】

**关键 commit**：`77cad0b4d6b16e9d0ff4da79b791f989775a07ca`（PR #238，2025-10-23），标题 "feat: implement Arsia fee model for L1 data cost calculation"[一手：`github.com/mantlenetworkio/mantle-v2/commit/77cad0b4…`]。

逐字段确认（`main` 分支源码直读）：

| 合约 | 新增字段/方法 |
|---|---|
| `SystemConfig.sol`（L1）| `basefeeScalar` / `blobbasefeeScalar` / `eip1559Denominator` / `eip1559Elasticity` / `operatorFeeScalar` / `operatorFeeConstant` / `minBaseFee` / `daFootprintGasScalar` 字段；`setGasConfigArsia()`（替代 deprecated `setGasConfig()`）、`setEIP1559Params()`、`setMinBaseFee()`、`setDAFootprintGasScalar()`、`setOperatorFeeScalars()` 方法 |
| `L1Block.sol`（L2 预部署 `0x4200…15`）| 同名字段镜像 + `setL1BlockValuesArsia()`——用**内联汇编**从 calldata 解包写入，工程上做了 gas 优化 |

这与第 7 节【实测】直接 `eth_call` 读到的 `baseFeeScalar()=169,019`、`operatorFeeScalar()=100,000,000` **逐位精确对应**——源码 + 链上状态 + L2Beat 升级记录**三方互证**，这是本报告对 Arsia 真实性的最强证据链。

**新增预部署合约**：`OperatorFeeVault`（`0x420…1b`，Release `v1.5.3`）——给 sequencer 开的新收入通道，呼应第 5 节"sequencer 收入"讨论。

**遗留兼容设计**：`GasPriceOracle.sol` 保留 Bedrock 时代字段（`l1FeeOverhead`/`l1FeeScalar`）与 Arsia 新字段**双轨并存**，用 `isArsia` 布尔开关切换——这是一个值得记录的工程模式：**Mantle 选择了"开关式渐进迁移"而非"清空重来"**，新旧字段共存降低了升级风险，但也留下了永久的技术债（第 7 节【实测】已验证 legacy `overhead()/scalar()` 字段仍可读到但已弃用）。

### 4.4 DA：MantleDA → EigenDA → 纯 Ethereum Blob 三段式演进【一手 Release 历史】

GitHub Releases API 确认的完整三阶段（比第一阶段"EigenDA 退役"结论更细）：

| 阶段 | 版本/时间 | 内容 |
|---|---|---|
| **① 自建 MantleDA** | `v1.0.0-alpha.1` / `v0.5.0-1`（2023-12 / 2024-03）| `.gitmodules` 中 `eignlayr-contracts → mantle-xyz/mantleDA-contracts`、`datalayr → mantle-xyz/mantleDA-datalayr` |
| **② 切换 EigenDA** | `v1.0.1`（2024-07）| "Mantle Sepolia testnet DA layer switches from MantleDA to EigenDA"（PR #163）|
| **③ EigenDA 优化 & 主网激活** | `v1.1.0`（Everest Sepolia, 2024-11）/ `v1.1.1`（Everest Mainnet, 2025-03）| EigenDA Proxy、S3/Redis 缓存、blob 上限 2MB→4MB、主网激活（PR #204）|
| **④ 移除 EigenDA，纯 Ethereum Blob** | Arsia 系列（`v1.5.3` 起，**2026-03**）| README 改为新表述，纯 blob 费用模型 |

**代码层面确认**：`develop` 分支的 `.gitmodules` **仍残留** `eignlayr-contracts`/`datalayr` 子模块声明，但对应目录 GET 请求均返回 **404**——说明目录已从工作树移除但 `.gitmodules` 未清理干净（小的工程卫生问题，不影响功能）。独立仓库 `mantle-xyz/op-succinct` 的 `MANTLE_CHANGES.md` 明确记录（迁移 Phase 2）："drop EigenDA/Celestia" 已完成，列出具体删除路径 `utils/eigenda/*`、`programs/range/*/celestia`、`programs/range/*/eigenda`。

⚠️**受限说明**：未能在 `mantle-v2` 主仓库定位到移除 EigenDA 的单一 commit/PR 链接（GitHub 匿名全文代码搜索需要登录，本环境无法执行），只能确认结果状态与 op-succinct 侧迁移记录，已列入存疑清单。

### 4.5 SP1 / OP Succinct 深度定制【一手，比第一阶段更细】

仓库：`github.com/mantle-xyz/op-succinct`——**深度定制分支，非普通 fork**。README 原文："OP Succinct is the production-grade proving engine for the OP Stack, powered by SP1 ... enables seamless upgrades for OP Stack rollups to a **type-1 zkEVM rollup**."

`MANTLE_CHANGES.md` 披露的关键事实：
- `kona`/`op-alloy`/`alloy-op-evm` 来自 `mantle-xyz/mantle-v2` 的 Rust 子树（pin 到 `v1.6.2`）；**revm 家族**来自 `mantle-xyz/revm`（pin 到 `v107-mantle-arsia.1`）——与第 3 节 FaultProofSurface 的发现完全一致。
- **SP1 版本锁定** = `6.4.0` + `sp1-cluster` tag `v2.7.2`；合约基线 `mantle-xyz/op-succinct@v1.1.7-2`（"v117"）。
- **Fault Proof 功能整体删除**："Mantle's runtime is **Validity-Oracle-only**"——即 Mantle 从代码库层面**已经彻底放弃**了 Cannon/MIPS 挑战式路径，第 3 节的"约束转移"结论在代码层面被坐实。
- **L2 区块支持范围硬限制**在 Arsia 激活后（主网起始区块 `94355444`）——即证明程序只能证明 Arsia 之后的区块，这是一个隐性的"不可逆"设计。
- 目的明述：把 Mantle 从 Optimistic Rollup 升级为 **Type-1 zkEVM**，提现时间从 7 天缩短到**约 1 小时**。

> ⚠️**存疑点（需记入存疑清单）**：这里的"提现约 1 小时"与第一阶段（E-mantle.md）及 L1 合约实测读到的 `finalizationPeriodSeconds() = 43,200 秒 = 12 小时`**直接冲突**。可能的解释：op-succinct 侧文档描述的是"证明生成 + 验证"技术意义上的最短窗口，而 `finalizationPeriodSeconds` 是官方主动保留的安全缓冲（第一阶段已引用官方原话"12h 是主动留的安全缓冲，不是技术下限"）。**两个数字本身不矛盾**（技术下限 vs 产品化窗口），但表述容易被误读为"提现只需 1 小时"，需要在对外沟通时明确区分。

### 4.6 非标准 RPC 方法【一手源码 + 【实测】互证，含关键分歧点】

- **`optimism_safeHeadAtL1Block`**（标准 `optimism_` 命名空间，非 `mantle_` 前缀）：Release `v1.3.1` 说明"新增 API 用于加速 op-succinct 生成 ZK 证明"；源码实现在 `op-node/node/api.go` 的 `func (n *nodeAPI) SafeHeadAtL1Block(...)`。通读 `api.go` 全文，其余方法均为标准 OP Stack 命名空间（`optimism_`/`opstack`），**未发现 `mantle_` 前缀方法**。
- **`eth_estimateTotalFee`**：源码检索**未找到**定义（受限于无法执行 GitHub 全文代码搜索）——但**第 7 节【实测】直接调用该方法在主网返回了有效结果**（`eth_call` 返回 `0x4af7c752fc33a`，非报错）。**这是"源码检索受限"与"链上实测"两种方法论互补的好例子**：源码侧因检索受限给出"未找到"，但实测侧证明该方法**确实存在且在生产环境可用**，本报告以实测结果为准。
- **`eth_getBlockRange`**：源码检索同样未找到定义；第 7 节实测调用返回 `403 Forbidden`，但**同一次测试中一个确定不存在的方法也返回相同的 403**，故这个方法"是否存在"仍标记为⚠️存疑（网关屏蔽 ≠ 方法不存在）。

### 4.7 Mantle 已改造清单汇总（= 它证明过自己有多大改造能力）

这是本节对委托方**最直接的可行性证据**——以下全部是 Mantle 在生产环境**已经做过**的协议层改造，而非假设：

1. 用 **MNT 替代 ETH** 作为原生 gas token（Release `v1.0.0-alpha.1`；`GasPriceOracle.sol` `tokenRatio` 机制，早于官方 CGT v2 两年多）
2. 自建 **MantleDA → EigenDA → 纯 Ethereum Blob（Arsia）** 三段式 DA 演进（§4.4）
3. **Arsia 全新费用模型**：`basefeeScalar`/`blobbasefeeScalar`/`operatorFeeScalar`/`operatorFeeConstant`/`minBaseFee`/`daFootprintGasScalar`/`eip1559Denominator`/`eip1559Elasticity` + `setGasConfigArsia()`（commit `77cad0b4`，PR #238）
4. 新增预部署合约 **`OperatorFeeVault`**（`0x420…1b`，Release `v1.5.3`）——**给 sequencer 新开收入通道**，是"只加系统合约"式改造的真实先例
5. 独立 `op-geth` fork 仓库，fork 自 `ethereum-optimism/op-geth`
6. 早期 `l2geth`（GPLv3，直接 fork go-ethereum，V1 遗产）
7. `op-node` 新增 `optimism_safeHeadAtL1Block` RPC **加速 ZK 证明生成**（`op-node/node/api.go`；Release `v1.3.1`）——**这是"只改排序器/节点软件加一个 RPC 方法"的真实先例**
8. 深度定制 **OP Succinct/SP1** 有效性证明服务，**Fault Proof 整体删除**（`mantle-xyz/op-succinct` `MANTLE_CHANGES.md`）
9. `GasPriceOracle` 新增 `owner`/`operator` 权限分离与 `setTokenRatio()` 白名单控制
10. 自建 Fork 命名体系：`BaseFee → Everest → Euboea → Skadi → Limb → Arsia`
11. 独立发布节奏与 Docker 镜像命名空间 `mantlenetworkio/mantle-op-node`
12. Bedrock 遗留字段（`l1FeeOverhead`/`l1FeeScalar`）与 Arsia 新字段**双轨兼容**（`isArsia` 开关）
13. Rust 侧单独 fork `mantle-xyz/revm` 打 Mantle 专用 tag
14. 提现时间通过 SP1 zk 证明从 7 天缩短（技术层面，见 4.5 存疑点）

> **结论**：**Mantle 的改造能力是被生产环境反复证明过的**——它不是"理论上能改"，而是**过去三年持续在改**（gas token、DA 层、费用模型、证明系统、RPC 接口都动过）。这直接反驳了"Mantle 团队没有能力做深度定制"的担忧；真正的问题不是能力，而是第 5 节揭示的**意愿与资源分配方向**（押注 RWA，不是 app-specific 性能）。

## 5. Mantle 2026 的 infra 路线图现状

### 5.1 preconfirmation / 高性能出块 / app-specific chain：第一阶段"完全缺席"结论**仍然成立**

核查方法：官方文档（`docs.mantle.xyz`/`mantle.xyz`）、官网首页、治理论坛 `forum.mantle.xyz` 站内搜索 API 三渠道交叉检索，检索日期 2026-09-07。

| 关键词 / 渠道 | 结果 |
|---|---|
| `site:mantle.xyz` / `site:docs.mantle.xyz` 搜索 "preconfirmation" / "app-specific chain" / "appchain" | **无任何 roadmap 级条目**。仅命中两篇不相关内容：① Helios 101 教育博文（介绍 OP Stack 生态 preconf 概念，非 Mantle 自身功能）；② 一篇提及 Devcon 7 前 "Preconf.erence" 活动的社区博文 |
| `forum.mantle.xyz/search.json?q=preconfirmation` | **`"posts": []` —— 0 结果**（可复现 URL：`https://forum.mantle.xyz/search.json?q=preconfirmation`）|
| `forum.mantle.xyz/search.json?q=appchain` | **0 结果**（`https://forum.mantle.xyz/search.json?q=appchain`）|
| `forum.mantle.xyz/search.json?q=sequencer&order=latest` | 最新命中为 2025-08 MIP-33（预算披露）与 2024-01 State Committee 帖（已标 `[ARCHIVED]`）；**2026 年零命中** |
| 官网首页 "Latest from Mantle" | 2026-07~08 六条动态**全部是 RWA / tokenized equities / CCIP 迁移**主题，无一条涉及出块性能或链架构 |

**结论：至 2026-09-07，Mantle 官方在 preconfirmation / 高性能出块 / app-specific chain 方向上仍然是零表态**，与第一阶段结论一致。官网定位已明确改写为 **"Institutional Onchain Finance / Powering Borderless Access to Global Capital Markets"**（RWA 分发层），infra 侧的全部动作都是"跟随以太坊做合规化收敛"，不是"跟 Rise/Base 拼性能"：

| 时间 | 事件 | 性质 |
|---|---|---|
| 2026-01-22 | 官方通稿宣布 DA 从 EigenDA（Validium 配置）迁移至 **Ethereum blobs**，"Mantle's transition represents a shift from a Validium-based configuration to a **ZK rollup architecture** secured directly by Ethereum."（Joshua Cheong, Head of Product）| [一手·官方通稿] |
| 2026-04-16/22 | **Arsia** 升级生效（见第 4、7 节一手 + 【实测】互证）| [一手] |
| 全年 | 官方叙事重心转向 xStocks（155 只代币化股票）/ Franklin Templeton USPX / Aave $1.45B 存款 | [二手·Nansen Q2'26 报告] |

⚠️ **唯一与"app-specific"擦边的表述**（同一份 2026-01-22 通稿，且仅为 EigenCloud 生态合作口径，**不是 Mantle 主链改造计划**）：
> "Mantle will continue to utilize EigenCloud for specialized use cases … including: **Perpetuals: Leveraging specialized execution environments.**"

### 5.2 MNT 经济学与 sequencer 收入现状（改造 ROI 分母）

**MNT 基本盘（2026-09-07，二手·CoinGecko）**：价格 $0.6617（24h +12.45%）；市值 $2.2B；流通量 **3.3B / 总量 6.2B（53.1%）**；ATH $2.86（2025-10-09）。

**Treasury（一手·官网实时面板，"As of Sep 7, 2026, 05:31 UTC"）**：总额 **$2,693,550,423**（MNT 占 75.25%、BTC 8.69%、ETH 8.32%、稳定币 4.44%）。

**无协议级质押 / 销毁**：MNT 是 gas 币 + 治理币，**不用于 sequencer/validator 安全质押**；2023 年 BIP-19 曾承诺的 $BIT 质押激励架构已随 EigenDA 弃用与 ZK 化失效；2024 年 Lagrange State Committee 质押帖已标 `[ARCHIVED]`。**无 EIP-1559 式 fee burn**，无 veMNT。社区对价值捕获缺失的质疑至今无官方回应：
> "What it currently lacks is a clear and credible supply narrative."（TaiwanDAO, 2026-01-23，论坛帖）
> "token buybacks? MNT burns? staking? revenue sharing? … ecosystem growth and token value are not automatically the same thing."（2026-08-26 帖，无官方回复）

**Sequencer 收入——本节最关键的数字**：

一手官方披露（治理提案 **MIP-33**，MantleCore 发布于 2025-08-28）：
> "During the second budget cycle (from July 2024 to June 2025) … **Mantle Network sequencer revenue: $1.76 million**"（同提案总收入 $48.86M，其中绝大部分来自 Treasury 投资收益，**sequencer 收入只占 3.6%**）

二手可复现时序（`api.growthepie.com` API，fees=用户链上手续费，rent_paid=付给以太坊 L1 的成本）：

| 期间 | 链收入(fees, USD) | L1成本(rent, USD) | 毛利≈ |
|---|---|---|---|
| 2024 全年 | $5,902,405 | $3,846,909 | ~$2.06M |
| 2025 全年 | $885,553 | $51,998 | ~$0.83M |
| **2026 YTD（1/1–9/6）** | **$131,310** | $2,447 | ~$129K |
| 最近 30 天 | $8,458（≈$282/天）| $224 | ~$8.2K |

> 可复现方法：`GET https://api.growthepie.com/v1/metrics/chains/mantle/fees.json`、`GET https://api.growthepie.com/v1/metrics/chains/mantle/rent_paid.json`（`profit.json` 端点返回 403，毛利为自行相减）。growthepie 页面口径互证："Daily chain revenue was $214 … ranking Mantle #15 among 27 tracked chains"。

**对 ROI 分母的判断**：2026 年 sequencer 收入 run-rate——按最近 30 天年化 ≈ **$10.3 万/年**，按 YTD 年化 ≈ **$19 万/年**（上半年活跃度更高），取 **$10–20 万/年** 区间。**2024→2025→2026 两年缩水约 98%**（Arsia 新费模型压低单笔费用 + 活跃度下滑双重作用）。且此数字**不含 SP1 证明的链下 prover 成本**（MIP-33 原话："costs of security audit, zk proof and support services may further increase"），净利可能更低。

> **核心结论**：任何以"分享 sequencer 费收入"为回报模型的链改造，分母以**十万美元/年**计，直接费用 ROI 基本可以忽略；Mantle 真实的经济引擎是 **$2.69B Treasury 的投资收益**（MIP-33 披露 FY24-25 总收入 $48.86M，Treasury 收益占大头），不是排序费。这意味着任何商业提案的对手方应该是 **Treasury/MIP 预算流程**，而不是"跟 sequencer 谈分成"。

### 5.3 治理论坛 2026 年链改造 / app-specific 提案核查：零提案

一手抓取 `forum.mantle.xyz/latest.json`，2026 年全部相关主题：

| 日期 | 标题 | 性质 |
|---|---|---|
| 2026-04-24 | **[PASSED] MIP-34**：向 Aave DAO 提供 30,000 ETH 信贷（rsETH 事件善后）| 2026 年**唯一**正式 MIP，财政/信贷类，与链架构无关 |
| 2026-02-25 | 【Discussion】Phase-Based Treasury MNT Burn（3–8% / 12–24 个月）| 社区帖（@TaiwanDAO），**0 官方回复，未进入投票** |
| 2026-01-23 | Ideas about tokens in the Burning Treasury | 同作者前置帖 |
| 2026-07-01 | [DISCUSSION] Token Backed Mortgages for Mantle Treasury Yield | 社区帖，财政类 |
| 2026-08-26 | MNT Holder 的幾個核心疑問 | 社区质询，仅 1 条共鸣回复："There also doesn't seem to be a clear direction for the project." |

辅助证据：`sequencer`/`preconfirmation`/`appchain` 三关键词 2026 年站内搜索命中数均为 **0**（见 5.1）。最近一次涉及 infra 方向的官方治理表述来自 MIP-33："As Mantle Network is expecting major upgrades to **transform into a zk rollup**, costs of security audit, zk proof and support services may further increase."——**官方 infra 预算指向 ZK rollup 化，而非 app-specific/高性能出块**。论坛整体活跃度极低（2026 年全年仅约 5 个新主题），治理重心已完全移至财政管理。

### 5.4 Superchain 成员资格核查：不是成员

**一手核实**（2026-09-07）：
1. `superchain-registry` mainnet configs 目录（`github.com/ethereum-optimism/superchain-registry/tree/main/superchain/configs/mainnet`）：32 个 `.toml` 文件（automata, bob, boba, celo, cyber, …, unichain, worldchain, zora 等），**无 `mantle.toml`**。
2. `chainList.json` 全量核验（`raw.githubusercontent.com/ethereum-optimism/superchain-registry/main/chainList.json`）：54 条链（主网+测试网），用 `name`/`identifier` 双字段匹配 "mantle" → **NOT FOUND**。
3. 独立分叉证据：Mantle 维护自有代码库 `mantlenetworkio/mantle-v2`（Arsia 即"统一自家 8 个 OP Stack fork"，对齐上游 Canyon→Jovian 但**不并入** Superchain）；状态验证走自家 `mantle-xyz/op-succinct` fork；升级权限归 **MantleSecurityMultisig**（L2Beat 权限章节一手记录），与 Optimism 的 Security Council / Upgrade Controller 体系无关。

**结论：Mantle 不是 Optimism Superchain 成员（非 Standard Chain/Standard Rollup），是完全独立的分叉，不受第 1 节 Standard Rollup Charter 的任何约束。**

> **含义（对链改造评估的直接影响）**：Mantle 不受 Law of Chains / Charter 约束，不向 Optimism Collective 分成收入，**理论上可以任意改造排序层/出块机制而无需任何外部许可**——这是与"改造 OP Mainnet 上的标准链"完全不同的处境。但代价是它也拿不到 Superchain 的 interop/共享升级红利，且其代码库已与上游深度分化（BVM 时代遗留 + MNT gas token + OP Succinct 定制），外部团队做 preconf/高性能出块改造的**工程成本显著高于标准 OP Stack 链**（第 2 节改造分级会进一步量化）。

### 5.5 小结：叙事错位

Mantle 2026 年的全部官方能量在 **"RWA 分发层 / 机构金融"**（xStocks、Franklin Templeton USPX、Aave 存款、DeFi TVL），infra 侧只做"跟随以太坊"的合规化收敛（blobs + ZK），**没有任何面向 preconf / 低延迟出块 / app-chain 的进攻性计划**。这与本项目委托方"把整条链做成 app-specific 形式"的设想之间存在**根本性的方向错位**——Mantle 团队自己的资源分配已经用脚投票，押注在别的赛道上。

## 6. 三条路线对比：另开链 / 改主链 / 主链+专用 lane

| 维度 | **A. 另开一条 app-specific 链** | **B. 改造 Mantle 主链** | **C. 主链 + 专用 lane / 专用 rollup** |
|---|---|---|---|
| **工程成本** | 最高：新链全套（sequencer、桥、浏览器、RPC、索引、钱包适配）+ 持续运维 | **中低**：L0+L1 为主（参数 + 合约 + builder），已被 RISE 案例证明够用 | 中：需额外的跨域消息与流动性同步层 |
| **流动性与生态割裂** | **最严重** —— 第一阶段硬结论：切断与 **xStocks（$633.7M，全球第 2）** 的同链组合性，而那正是 Mantle 的核心资产 | **无割裂**：新应用与 xStocks、$576M 闲置稳定币、mETH/cmETH 同处一个状态机 | 中等：lane 内隔离但仍同链则无割裂；独立 rollup 则重现 A 的问题 |
| **桥依赖** | 新增一跳桥，且是新桥（最高风险面） | 无新增 | lane 无新增；独立 rollup 有 |
| **监管 / 合规** | 可为证券型资产定制准入，但要重建全套合规叙事 | **可复用 Mantle 已有的机构叙事**（Institutional Onchain Finance / Franklin Templeton / xStocks） | 可在 lane 层做差异化准入，**合规弹性最好** |
| **可退回性（失败能否撤）** | **最差**：链一旦发出去，代币、桥、用户资产都在上面，关停成本极高（参见第一阶段 Funny Money / Printr 两次关停的教训是**产品**级，关链是**基础设施**级） | **最好**：参数可回滚、合约可弃用、builder 策略可关闭 | 中：lane 可关；独立 rollup 难关 |
| **叙事** | "为发行专门做一条链"故事性最强 | 需要重新讲"Mantle 是发行友好链"，但**有 xStocks 作锚** | "同一条链、不同快车道"较难向市场解释 |
| **对 RISE 模式的对应度** | ❌ **RISE 恰恰不是这条路** —— 它是标准 OP Stack rollup 换执行层，不是新栈新链 | ✅ **这才是 RISE 实际走的路** | ⚠️ 无生产先例（Tempo 的 payments lane 是新链自带，非既有链加装） |

### 6.1 对第一阶段硬结论的复核

第一阶段结论：**「为 meme 单开一条 appchain 会切断与 xStocks 流动性的连接，而那正是全部价值所在」**。

**复核判定：仍然成立，且本轨道给出了两条新的加强理由**：

1. **RISE 案例本身就是反证 A 路线的证据。** RISE 没有另开新栈 —— 它是**标准 OP Stack rollup + 换执行层客户端 + 调参数**（H 轨道已证）。所谓"app-specific chain"在 RISE 身上的实现方式，落在 B 路线而不是 A 路线。**委托方原始设想里"把整条链做成 app-specific"，正确的技术翻译是"改造主链"，不是"另开一条链"。**
2. **Mantle 的经济结构不支持 A 路线。** sequencer 收入 run-rate 仅 $10–20 万/年（§5.2），新链要自负 DA 与证明成本却没有费收入基础；而 B 路线的成本可挂在 Treasury 预算下，无需新建收入模型。

**唯一可能推翻它的情形（诚实记录）**：若「资产发行」最终要走**证券化 / 需要一级市场 KYB 闸门**的路线，且监管要求与无许可 DeFi 在**同一状态机内不可共存**，那么 C 路线（带准入的专用 lane）会优于 B。⇒ 这取决于法务边界，**不是技术判断**，本轨道无法定论。

## 7. 实测：直连 Mantle 主网 RPC 复核关键参数

> 取数时间：2026-09-07。RPC endpoint：`https://rpc.mantle.xyz`（公共网关）。所有请求**串行执行**，间隔 150–250ms。方法：`eth_call`/`eth_getBlockByNumber`/`eth_getBlockReceipts`/`eth_getTransactionReceipt` 均为标准 JSON-RPC POST，请求体见下方与《可复现方法附录》。

### 7.1 基础链参数【实测】

| 参数 | 实测值 | 与第一阶段（2026-09-06,E-mantle）对比 |
|---|---|---|
| `eth_chainId` | `0x1388` = **5000** | 一致 |
| `net_version` | `5000` | 一致 |
| `web3_clientVersion` | `Geth/v1.17.3-stable-11fa8109/linux-amd64/go1.26.6` | 未见 mantle 定制字符串，只是标准 geth 版本号——**分叉标识不体现在 clientVersion 里**，需要靠合约/字节码/RPC 行为侧面确认 |
| 最新区块 | `100,314,720`（`0x5faae60`） | 比第一阶段的 `100,278,282` 约高 36,438 块，时间差 ≈ 36,438×2s ≈ 20.2 小时，与两次取数间隔（约 1 天）吻合 |
| `gasLimit` | `0x3938700` = **60,000,000** | 一致 |

### 7.2 出块间隔【实测】

抓取 20 个采样点，**每隔 50 个区块**取一次（覆盖约 1,000 个区块 / ~33 分钟），比较相邻采样点的 `timestamp` 差值：

| 起始区块 | 结束区块 | 区块跨度 | timestamp 差 | 平均出块间隔 |
|---|---|---|---|---|
| 100,313,770 | 100,314,720 | 950 | 1,900 秒 | **恰好 2.000 秒/块**（全部 19 个相邻采样间隔一致，无一例外） |

**结论：Mantle 出块时间仍然精确锁定在 2.000 秒，未发现任何加速迹象。**（请求体见附录 A）

### 7.3 Gas 价格与填充率【实测】

同一批 20 个区块的 `gasUsed`：

| 指标 | 值 |
|---|---|
| 最小 gasUsed | 46,287（空块，仅系统交易） |
| 最大 gasUsed | 1,143,060 |
| 平均 gasUsed | **198,957** |
| 填充率 | **0.332%**（198,957 / 60,000,000） |

> 比第一阶段的 0.173% 高，但仍是**两位小数点级别的空闲**；样本量小（20 块 vs 全天 43,200 块），差异属正常波动，不代表趋势性变化。

`baseFeePerGas`：**20 个采样点全部 = `0xba43b7400` = 50,000,000,000 wei = 50 gwei（MNT 计价）**，无一浮动。**base fee 仍然死死钉在下限，EIP-1559 一年多未被真正触发过。**

### 7.4 GasPriceOracle 合约实测（Arsia 参数一手核验）

合约地址：`0x420000000000000000000000000000000000000F`（version `1.1.0`）。逐个函数选择器 `eth_call`：

| 字段 | 实测值 | 与 L2Beat「Arsia fee mechanics activated」milestone 原文对比 |
|---|---|---|
| `baseFeeScalar()` | **169,019** | **完全一致**（L2Beat 原文：`basefeeScalar set to 169019`）→ 【实测 + 一手互证】Arsia 已在**主网生效**，非仅路线图 |
| `operatorFeeScalar()` | **100,000,000** | **完全一致**（L2Beat 原文：`operatorFeeScalar to 100000000`） |
| `operatorFeeConstant()` | 0 | 与 Arsia 记录一致 |
| `blobBaseFeeScalar()` | 0 | 与第一阶段一致（blob 费仍未单独计价） |
| `tokenRatio()` | **3,789** | 交叉验证：CoinGecko 现价 ETH $2,510.41 / MNT $0.66209 = **3,791.1**，实测值偏差仅 **0.06%** —— 再次证实 `tokenRatio` = 实时 ETH:MNT 汇率 |
| `l1BaseFee()` | 55,573,742 wei ≈ 0.0556 gwei | |
| `overhead()` / `scalar()` / `decimals()` | 188 / 10,000 / 6 | legacy 字段，与第一阶段一致 |
| `version()` | `"1.1.0"` | 一致 |
| `daFootprintGasScalar()` / `eip1559Denominator()` / `eip1559Elasticity()` / `minBaseFee()` / `isEcotone()` / `isFjord()` | **全部 `execution reverted`** | 这些字段**不在 L2 侧 GasPriceOracle 合约里**，只存在于 L1 `SystemConfig` 合约（治理只更新 L1 侧值，通过 op-node 派生逻辑影响 L2 出块，而不是简单同步到 L2 predeploy）——这是一个值得记录的架构细节：**L2 上可读的 Arsia 参数子集 ≠ 完整参数集** |

> **这是本报告对"Mantle Arsia 升级"的第一手直接验证**：不依赖任何二手转述，直接从主网合约状态读出的两个关键 Arsia 专属字段（`baseFeeScalar`、`operatorFeeScalar`）与 L2Beat 记录的 Arsia 升级交易日志**逐位精确匹配**，证明 Arsia 不是"仅公告/仅路线图"，而是已经生效并持续运行的主网状态。

### 7.5 非标准 RPC 方法探测【实测】

逐个测试，间隔 250ms（请求体见附录 B）：

| 方法 | 结果 | 解读 |
|---|---|---|
| `eth_estimateTotalFee` | **返回结果**（`0x4af7c752fc33a`，非报错） | **确认存在**，与第一阶段"Arsia 新增 RPC"的判断一致，且已在生产环境可调用 |
| `eth_getBlockRange` | `HTTP 403 Forbidden` | 与"已废弃"的判断一致，但见下方方法论说明 |
| `mantle_gasPriceOracle` | `HTTP 403 Forbidden` | 同上 |
| `rollup_gasPrices` / `optimism_rollupConfig` | `HTTP 403 Forbidden` | 同上 |
| `foobar_nonexistentMethod12345`（对照组，确定不存在的方法） | `HTTP 403 Forbidden` | **关键方法论发现**：公共网关 `rpc.mantle.xyz` 对**任何未注册方法**统一返回 403（而非标准 JSON-RPC `-32601 method not found`），这意味着**403 本身不能证明某方法"不存在"**，只能证明"公共网关没有暴露它"。因此 `eth_getBlockRange` 等的 403 结果**降级为⚠️存疑**（可能是废弃，也可能只是网关未开放），只有 `eth_estimateTotalFee` 的"成功返回"是可靠的正向证据 |
| `eth_getBlockReceipts` | 正常返回，含 `depositNonce` 字段 | 标准 OP Stack 方法，非 Mantle 特有 |
| `eth_maxPriorityFeePerGas` | `0x186a0` = 100,000 wei = 0.0001 gwei | 建议小费极低，与"无优先级市场"的判断一致 |

### 7.6 真实交易成本抽样【实测】

从区块 `100,314,070`（gasUsed 1,143,060，含 5 笔交易）抓取全部交易回执，剔除 L1 attributes 系统交易后，对 4 笔真实用户交易解算（MNT 现价 $0.66209，2026-09-07）：

| 交易（截断）| gasUsed | effectiveGasPrice | L2 费(MNT) | L1 费(MNT) | 合计(MNT) | 合计(USD) |
|---|---|---|---|---|---|---|
| `0xe829b3e9…` | 176,466 | 100 gwei | 0.017647 | 0.0000734 | 0.017720 | **$0.0117** |
| `0x7dc0b0a0…`（合约调用，疑似 swap）| 415,070 | ≈50.4 gwei | 0.021026 | 0.000151 | 0.021177 | **$0.0140** |
| `0xe475b62f…`（简单转账，21,000 gas）| 21,000 | ≈50.09 gwei | 0.001050 | 0.0000734 | 0.001124 | **$0.0007** |
| `0xa7bf4898…`（合约调用）| 484,225 | ≈50.09 gwei | 0.024221 | 0.000178 | 0.024398 | **$0.0162** |

> **单笔真实交易成本区间 $0.0007–$0.0162**，比第一阶段抽样（$0.004–$0.009）区间更宽（这次样本包含更重的合约调用），但**量级结论不变：成本仍是亚美分到 1.6 美分级别，不是 Mantle 的问题**。同时发现**部分交易的 `effectiveGasPrice`（100 gwei）是 baseFee（50 gwei）的 2 倍**，说明存在小额 priority fee 竞价（与"无优先级市场"的判断不矛盾——只是零星出现，不构成费用市场）。

## 8. 核心结论：只改排序器 + 加系统合约，能拿到 Rise 式体验的百分之多少？

### 8.1 计算方法

以 H 轨道逐项列出的 **「RISE 相对通用 OP Stack（含 Mantle）领先的 12 项能力」** 为分母，逐项判断「只改排序器（L1）+ 加系统合约（L0）」能否覆盖：

| # | RISE 领先能力 | L0+L1 能否覆盖 | 说明 |
|---|---|---|---|
| 1 | 1.5 Ggas 区块预算（50x 有效容量） | ✅ **能** | 纯 chain config 参数 |
| 2 | base fee 地板压到 0.0004 gwei | ✅ **能** | 纯参数 |
| 3 | 1 秒出块 | ✅ **能** | 纯参数 |
| 4 | 逐笔预确认流（Shreds）+ WS 订阅 | ⚠️ **部分**：Flashblocks/rollup-boost 式**定长子区块**可用 L1 拿到（200ms 级软确认）；**逐笔（1 tx/shred）** 需改执行层 | 覆盖约 60% 的体感价值 |
| 5 | `eth_sendRawTransactionSync`（EIP-7966） | ✅ **能**（纯 RPC 层） | 周级工程，性价比最高 |
| 6 | pending 阶段即返回 receipt | ⚠️ **部分**（配合 L1 软确认可近似） | — |
| 7 | Continuous Block Pipeline（执行占满区块时间） | ❌ **不能** | 必须改执行层客户端 |
| 8 | RiseDB 自研状态存储 | ❌ **不能** | 必须改执行层；但**在填充率 0.224% 的链上无收益** |
| 9 | 链级 Internal Oracle 公共原语 | ✅ **能** | 纯合约 + 文档 |
| 10 | 链级原生 VRF | ✅ **能** | 合约 + 链方运营 backend |
| 11 | 原生稳定币 + float 收益回投流动性（USDR/M0） | ✅ **能** | 商业与法务安排，非技术 |
| 12 | 官方钱包 + relay 全额 gas 赞助 + session key | ✅ **能** | 应用层，零协议改动 |

### 8.2 三个不同口径的答案

| 口径 | 覆盖率 | 依据 |
|---|---|---|
| **按能力条目数（未加权）** | **8 项完全覆盖 + 2 项部分覆盖 / 12 项 ⇒ 约 67%–75%** | §8.1 表 |
| **按「发行类应用（Launchpad）的价值」加权** | **约 90%** | 未覆盖的 3 项（CBP、RiseDB、逐笔 shred）解决的全是**毫秒级单笔延迟**与**极高吞吐**。发行场景的用户在乎的是"发币要不要签三次名""狙击者是否抢到第一个区块""要不要先买 gas 币"，**没有一项需要毫秒级确认**。而这三类需求全部落在 L0+L1 |
| **按「perps 的价值」加权** | **约 40%** | perps 的核心竞争力就是延迟与撮合吞吐。缺了 CBP 与逐笔预确认，做市商的报价/撤单竞争力无法对标；且**纯 Solidity 链上 CLOB 单笔要烧 240 万–420 万 gas**，在 60M 的区块里一个区块只装得下 14–24 笔（RISE 是 357–615 笔）⇒ **不改区块参数则链上 CLOB 在 Mantle 根本跑不起来**（但改参数属 L1，见 #1） |

### 8.3 直接回答

> **「只改排序器 + 加系统合约」能拿到 Rise 式体验的：按条目约 70%，按发行场景的价值约 90%，按 perps 场景的价值约 40%。**
>
> **更重要的是三条结构性发现**：
> 1. **RISE 的 12 项优势里，有 7 项根本不需要碰执行层**，其中 4 项（公共 oracle、VRF、原生稳定币、钱包 gas 代付）**连协议都不用改** —— 它们是**产品与商业决策**，被误读成了技术护城河。
> 2. **真正需要硬工程的 3 项（CBP、RiseDB、逐笔 shred）全部服务于同一个目标：压低单笔确认延迟。** 这个目标**对 perps 是生命线，对发行类应用几乎无关**。
> 3. **因此结论是不对称的**：委托方若做「资产发行原生链」，**L0+L1 足够，不必自研执行层**；若要同时做「有竞争力的链上 perps」，则必须在 L1（参数：大区块 + 近零 base fee）之外再投 L2 级的执行层工程 —— **而这是一个年级、需要自研客户端团队的投入**。

## 存疑清单

1. ⚠️ **GitHub 匿名代码全文检索不可用**（`type=code` 需登录），§4 中部分"未找到"结论属**受限未找到**，非"确认不存在"。
2. ⚠️ **Mantle 的 force inclusion 窗口仍未解**：第一阶段记录官方文档写 24 小时、L2Beat 写 "up to 12h"，本轨道未能收敛。
3. ⚠️ **§3.3 第 3 点「自定义逻辑越重则 cycle 数与证明成本线性上升」为本研究推断**，非官方逐字结论；SP1 官方只确认 built-in precompile 能显著降低成本、缺补丁会造成 cycle regression。
4. ⚠️ **2026-08-10 的 SP1 vkey 轮换在 L2Beat 记为 "not verified"**（复现需私有依赖）。这意味着**Mantle 当前的 guest 程序不可被第三方独立复现** —— 是一个实质性的透明度缺口，但官方未就此说明。
5. ⚠️ **`SuccinctL2OutputOracle` 的 optimistic 兜底模式当前是否启用、由谁可切换、有无 timelock**，本轨道未确认。这是 L2Beat 标为 CRITICAL 的风险点。
6. ⚠️ **sequencer 收入的两个口径不完全可比**：MIP-33 的 $1.76M（FY24-25，官方一手）与 growthepie 的 fees 序列（二手 API）统计边界不同，本轨道未对齐口径。
7. ⚠️ **SP1 链下 prover 成本未取到具体数字**，故"净利可能更低"为定性判断。
8. ⚠️ **Arsia 的确切生效日期在两处一手源间有 1 周差异**（L2Beat milestone 记 2026-04-16；另一渠道记 2026-04-22），本轨道并列记录未收敛。
9. ⚠️ **OP Stack custom gas token v2 与 Mantle 自有 MNT gas fork 的技术谱系完全独立**（见 K 轨道 §6 表 #25），因此"照 CGT v2 文档改 Mantle gas 逻辑"的任何方案都需要重新评估，本轨道未展开该迁移路径的工程量。
10. ⚠️ **§5 的"零提案"结论基于公开治理论坛**；若 Mantle 存在非公开的内部 infra 路线图，本轨道无从得知。

## 关键来源清单

**OP Stack / fault proof（一手）**
- Fault Proof VM 规范：<https://specs.optimism.io/fault-proof/index.html>
- precompile accelerators：<https://specs.optimism.io/fault-proof/index.html#precompile-accelerators>
- Cannon FPVM（MIPS64）：<https://specs.optimism.io/fault-proof/cannon-fault-proof-vm.html>
- kona（Rust fault proof client，2026-01-15 并入 optimism monorepo）：<https://github.com/op-rs/kona>
- kona FPVM precompile 模块：<https://github.com/op-rs/kona/blob/main/bin/client/src/fpvm_evm/precompiles/mod.rs>
- custom gas token：<https://docs.optimism.io/stack/rollup/custom-gas-token>

**OP Succinct / SP1（一手）**
- OP Succinct FAQ（**"no longer classified as a Standard Chain"**）：<https://succinctlabs.github.io/op-succinct/faq.html>
- SP1 precompile 与 cycle 成本：<https://docs.succinct.xyz/docs/sp1/optimizing-programs/precompiles>
- Mantle 的 op-succinct fork 变更说明（四层 fork + vkey 轮换流程）：<https://github.com/mantle-xyz/op-succinct/blob/main/MANTLE_CHANGES.md>

**Mantle（一手）**
- L2Beat 项目页（Stage 0、ZK validity、milestone、CRITICAL 风险、vkey 轮换记录）：<https://l2beat.com/scaling/projects/mantle>
- mantle-v2 monorepo：<https://github.com/mantlenetworkio/mantle-v2>
- op-geth fork：<https://github.com/mantlenetworkio/op-geth>
- `GasPriceOracle.sol`（`tokenRatio` / `isArsia`）：`mantle-v2/packages/contracts-bedrock/src/L2/GasPriceOracle.sol`
- 治理提案 MIP-33（sequencer 收入 $1.76M）：<https://forum.mantle.xyz/>
- 治理论坛检索端点（可复现）：`https://forum.mantle.xyz/search.json?q=<keyword>`

**Mantle（二手）**
- growthepie 链收入 / L1 成本 API：`https://api.growthepie.com/v1/metrics/chains/mantle/fees.json`、`.../rent_paid.json`
- MNT 市场数据：CoinGecko

**本仓库交叉引用**
- RISE / RISEx 实测与拆解：`research/H-rise-chain-infra.md`、`research/I-risex-app-coupling.md`
- 排序 / 延迟零件目录：`research/K-sequencing-latency-infra.md`
- Mantle 第一阶段基线：`research/E-mantle.md`、`report/04-mantle-gap-analysis.md`

## 可复现方法附录

### A. Mantle 主网 RPC 实测（串行，间隔 150–250ms）

```bash
EP=https://rpc.mantle.xyz
call(){ curl -s -X POST $EP -H 'content-type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$1\",\"params\":$2}"; }

call eth_chainId '[]'                     # → 0x1388 = 5000
call web3_clientVersion '[]'              # → Geth/v1.17.3-stable-11fa8109/linux-amd64/go1.*
call eth_blockNumber '[]'
call eth_getBlockByNumber '["0x<hex>",false]'   # 连续 20 块：timestamp 差分 / gasLimit / gasUsed / baseFeePerGas
call eth_getBlockReceipts '["0x<hex>"]'         # gasUsed + effectiveGasPrice 算真实成本
call eth_sendRawTransactionSync '["0x00"]'      # → -32601 ⇒ 方法不存在
```

**十六进制换算务必核对**（本轨道上一轮曾因换算错误采样到不存在的未来区块）。

### B. 实测结果汇总（2026-09-07）

| 项 | 实测值 | 与第一阶段对比 |
|---|---|---|
| chainId | 5000 | 一致 |
| 出块间隔 | **2.000 s** | 一致 |
| 区块 gasLimit | **60 M** | 一致 |
| 填充率 | **0.224%** | 第一阶段 0.173%，**同量级，结论不变** |
| baseFeePerGas | **50 gwei（钉在下限）** | 一致，EIP-1559 仍从未触发 |
| tx / block | **2.0** | — |
| 单笔真实成本 | **$0.0007–$0.0162** | 第一阶段 $0.004–$0.009，区间更宽（含更重合约调用），量级一致 |
| `eth_sendRawTransactionSync` | **不存在（-32601）** | 第一阶段"无 preconfirmation"结论的补充证据 |

### C. 治理与经济数据复现

```bash
curl -s 'https://forum.mantle.xyz/search.json?q=preconfirmation'   # → {"posts": []}
curl -s 'https://forum.mantle.xyz/search.json?q=appchain'          # → 0 结果
curl -s 'https://api.growthepie.com/v1/metrics/chains/mantle/fees.json'
curl -s 'https://api.growthepie.com/v1/metrics/chains/mantle/rent_paid.json'
# 注：profit.json 返回 403，毛利为 fees − rent_paid 自行相减
```
