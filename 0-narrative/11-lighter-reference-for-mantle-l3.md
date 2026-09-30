# 以 Lighter 为参照设计 Mantle App-specific L3

> **用途**：为 [09 L3 路线方案](09-native-l3-architecture-and-errata.md)提供具体的执行、状态、证明与父链结算参考，支撑 [07 三路线比较](07-three-architectural-routes-tradeoff-analysis.md)。  
> **核心映射**：Ethereum 承担的父链角色移至 Mantle L2；Lighter 承担的专用执行角色移至 Mantle App-specific L3。  
> **证据类型**：官方白皮书、官方架构文档、固定版本公开代码；审计报告与 L2BEAT 作为交叉检查。  
> **范围限制**：本轮为设计与源码核验，未部署、未运行证明基准，未证明所读代码与当前生产部署逐字一致。Mantle 方案是设计推导，不是 Lighter 官方支持的部署方案。

## 1. 判断：可以下移一层，角色比层级编号更重要

Lighter 是具体可参考的应用专用 Rollup 架构：交易在独立执行环境内完成，父链保存资产与状态承诺，验证批次并处理提现。这个分工不要求父链必须是 Ethereum L1；具备适当执行、验证、数据可用性和退出能力的 Mantle L2，也可以承接父链角色。[L1][L2]

因此，Mantle L3 可以按以下模式设计：

> 在 Mantle L2 上部署交易应用的托管与结算协议，在独立的专用执行环境中运行订单簿、仓位、保证金和清算，再以有效性证明将结果结算回 Mantle L2。

这使路线三有了明确参照。它不要求把每笔交易拆成 L2 与 L3 之间的远程调用，也不要求先建设跨 L3 统一保证金。复杂性主要集中在资金进入、批次结算、退出与异常恢复的边界。

Lighter 能支持该分工的可行性，但不能直接证明 Mantle 版本的延迟、费用、开发工期或安全性与其相同。迁移后增加了 Mantle 父层依赖，也需要适配 DA 和证明验证路径。

## 2. 核验版本与证据边界

| 材料 | 本轮使用的版本 | 能说明什么 |
|---|---|---|
| Lighter 白皮书 | PDF 标注 October 2025，23 页 | 设计动机、状态树、执行周期、账户差分、证明聚合与 Escape Hatch |
| 官方 Lighter Core 架构页 | 本轮抓取的在线内容 | 组件分工和用户状态恢复原则；网页会更新 |
| `elliottech/lighter-prover` | `38bea38e59be67a5381665e9562ec0c7508e8bee`，提交时间 2026-09-07 | 所读电路、证明包装、构建、基准与退出工具的实现 |
| `elliottech/lighter-contracts` | `75c2a73a5c25c9a6a36b66ba73aaf71267232b42`，提交时间 2026-09-07 | 充值队列、批次状态、提现余额、Blob 绑定和异常退出接口 |
| zkSecurity 第三次报告 | 2025 年审计范围 | Account Delta 与聚合层的机制和历史问题；不是以上提交的全面审计 |
| L2BEAT Lighter | 本轮读取的部署观察 | 安全与运行配置的交叉检查，不覆盖我们拟建的 L3 |

白皮书与后续代码属于不同时间点。本文不把旧审计中的树高、固定交易类型数量或旧配置当成现版本常量；涉及具体实现时给出代码版本。[L1][L3][L4][L10][L11]

## 3. Lighter 实际把什么放在了父链

### 3.1 父链托管与执行层账本

白皮书 §2、§2.5 说明：Ethereum 智能合约持有存入资产并维护规范状态，Sequencer 在独立执行环境中处理交易，Prover 验证其状态转换。挂单、撮合、资金费和清算无需逐笔在 Ethereum 上执行。[L1][L2]

父链保留资产的物理托管，不意味着资产可同时在父链被自由使用。存入交易系统的资金已经对应其内部账本权益，退出必须满足该账本的扣账与验证规则。这正是 Mantle L3 可以采用的预存保证金模式。

### 3.2 两条输入路径

- **日常交易路径**：用户通过 API 提交交易，Sequencer 执行并向 Indexer、API 和证明服务发布执行数据。
- **父链优先路径**：关键请求进入父链队列，协议跟踪处理进度和超时；运营方未按要求执行时允许进入退出模式。

所读合约中，`registerDeposit` 将充值加入优先队列。`addPriorityRequest` 计算队列前缀哈希、设置过期时间并发出事件；`commitBatch` 检查批次声称处理的优先请求数量与前缀哈希。[L4a]

这提供了一个具体的父链消息认证模式：电路不能随意虚构充值，父链也不只是相信 Sequencer 发来的一句“已经处理”。队列约束需与证明内的输入消费规则共同验证。

### 3.3 Commit、Verify、Execute 与实际领取分别记录

白皮书将状态提交与验证概括为一条流程；所读合约进一步区分了以下状态：[L4b]

| 阶段 | 所读实现 | 语义 |
|---|---|---|
| Commit | `commitBatch` | 校验批次连续性、优先队列承诺和 Blob 数据绑定，保存批次承诺 |
| Verify | `verifyBatch` | 按顺序验证已提交批次的证明，更新验证进度 |
| Execute | `executeBatches` / `_executeOneBatch` | 执行已验证批次的父链操作，检查操作哈希并累加可提现余额 |
| Claim | `withdrawPendingBalance` | 用户或调用方触发向资产所有者付款，扣减实际支付对应的余额 |

没有待处理父链操作时，代码存在执行进度的延迟更新优化，不能机械要求每批都发四笔交易。关键是**提交、证明通过、父链副作用执行、用户已收到资金具有不同语义**。

Lighter 的这条提现路径采用批次操作哈希与父链待领余额，不能笼统描述为“每个用户都必须提供独立的提现 Merkle 证明”。Mantle 的参考设计可以直接借鉴该账务划分。

## 4. 专用执行环境最值得借鉴的部分

### 4.1 执行与证明一起设计数据结构

白皮书 §3.7 的 Order Book Tree 将价格与同价订单的优先序编码到叶子索引，内部节点保存子树订单数量或报价聚合信息。证明器可以利用认证路径检验当前撮合是否选择了正确优先级的订单。[L1]

它的价值在于减少“为了证明业务正确而重复做的工作”。如果运行时使用快速数据结构，却需要在证明中重建另一套昂贵索引，执行快不一定带来端到端效率。

Mantle Perps L3 应据此一起设计订单索引、账户仓位、风险快照和证明见证；不直接复用旧报告中“每笔只有几千约束”之类没有同口径测量支持的数字。

### 4.2 本地完整的交易和风险闭环

白皮书的 Block Pre-Execution 处理市场级工作，如价格、资金费和溢价更新；后续执行周期验证签名、Nonce、状态路径及保证金，再修改业务状态。[L1，§4]

一笔用户交易可能展开成多个内部执行周期。比如一个 taker 吃掉多个 maker，需要持续推进匹配指令，不能把“一笔订单”“一个成交”“一个电路执行周期”当作同一个计量单位。

映射到 Mantle 时，撮合、仓位、保证金、资金费和清算留在 L3。L2 对整批结果验证，不参与每次开仓的同步授权。

### 4.3 公共退出状态与完整执行状态分开

白皮书 §3.1–§3.4 区分了账户执行状态、Public Account Tree、Account Delta Tree 和公开市场数据：[L1]

- 执行状态包含订单、API Key、仓位和业务元数据等。
- Public Account Tree 保留计算账户资产价值与归属所需的公共信息。
- Account Delta Tree 是批次内的聚合变更，在每个批次开始时初始化，并汇总对公共账户状态的修改。
- 公开市场数据包含计算持仓价值、资金费等所需的信息。

这为 Mantle 提供一个重要设计选择：**为了让用户安全退出，需要发布哪些数据；为了让任何人接管并继续运行完整订单簿，又需要哪些额外数据？** 两者的成本与保障不同，必须分别写进协议目标。

## 5. 数据发布与证明系统：哪些不能混为一谈

### 5.1 Lighter 的公开 DA 目标是可恢复退出权益

白皮书 §2.1 明确使用 hybrid data availability 一词：高频订单操作不全部发布到 Ethereum，发布聚合后的账户变更与市场信息，以支持用户恢复账户价值和 Escape Hatch。[L1]

不能据此推断“每次挂撤单都没有任何系统成本”，也不能把“恢复公共账户状态”写成“任何人可从父链恢复所有订单历史和完整实时订单簿”。后者需要另行检查完整运行数据的可用性。

[L2BEAT 将 Lighter 分类为 Appchain ZK Rollup](https://l2beat.com/scaling/projects/lighter)，使用 onchain state diffs 描述其 DA。[L11] 本文不因白皮书的 hybrid 用词直接重分类整个系统，而是明确每种数据保障覆盖什么。

### 5.2 证明后端并非 SP1

固定版本的 `types/config.rs` 使用 `GoldilocksField` 和 `Poseidon2GoldilocksConfig`；工作空间包含 Plonky2 和 Plonky3 Schnorr 相关组件。最终包装使用 gnark，在 BN254 上构建验证电路；白皮书 §5 说明最终提交的是 PLONK 包装证明。[L1][L3a][L3b]

因此有两条不同实现路径：

| 路径 | 借鉴内容 | 额外工作 |
|---|---|---|
| 按 Lighter 架构自建，使用 SP1 等 zkVM | 业务状态模型、公共差分、批次协议和退出方式 | 将业务程序编译进 zkVM，验证同等不变量，测量成本；不是 Lighter 专用电路的直接移植 |
| 基于可获得授权的 Lighter 实现适配 | 可研究或复用专用电路、合约和工具 | 确认代码许可与完整交付范围，重做父链 / DA 适配、密钥、部署和安全验证 |

Mantle L2 使用什么证明系统，不强制所有 L3 使用同一个系统。L2 能正确执行选定的 L3 Verifier 才是直接条件；Verifier 在 L2 上的 Gas、相关 precompile 的证明实现，以及对 Mantle 自身证明负载的影响仍须测试。

固定代码中部分电路配置 `zero_knowledge: false`，白皮书也说明当前主要用途是有效性验证，不是隐私 Rollup。[L1，§2.4 注释][L3a] Private L3 不能因复用该证明框架就宣称订单隐私已经具备。

### 5.3 不能将基准工具当成生产证明时延

仓库提供 block proving benchmark，但本轮未运行。所读 `bench.rs` 的汇总计时由 pre-execution、交易证明、链递归和最终 block proof 的部分计时相加；单独生成签名批证明的调用不在该求和项内。该入口也不覆盖最终批次 Blob / Delta 包装、gnark 包装和父链提交的完整端到端流程。[L3c]

README、构建脚本和 CLI 的 light chunk 默认值也并非所有位置都一致。因此比较时应显式记录参数、见证、硬件与计时边界，不引用无对应配置的“秒级”或“快 100 倍”。

## 6. Ethereum / Lighter 到 Mantle L2 / L3 的对应关系

| Lighter 现有角色 | Mantle 参考设计 | 可保留的设计 | 必须适配或验证 |
|---|---|---|---|
| Ethereum 上资产托管 | Mantle L2 上的每 L3 Escrow | 父链托管、子链账本权益 | Token 地址、精度、冻结 / 转账行为和原生 Gas 资产 |
| 父链优先请求队列 | Mantle L2 Priority Inbox | 充值、强制退出请求、消费进度与超时 | 父链事件跟踪、确认策略、重组恢复、超时预算 |
| Lighter Sequencer / 交易状态机 | App-specific Perps L3 | 大量交易在本地风险域内执行 | 市场规则、Oracle、部署与恢复服务 |
| 完整状态与公共退出状态 | L3 执行状态 + 退出状态承诺 | 按用途划分数据，证明两者一致 | 数据充分性、对账、恢复与资产守恒 |
| Blob 账户差分与市场数据 | Mantle L2 上的批次数据发布 | 账户差分聚合与数据绑定 | 不能假设 Mantle 提供 Ethereum 式原生 Blob 交易 |
| Commit / Verify / Execute | L2 批次结算状态机 | 分离数据提交、验证与父链副作用 | 合约命名空间、Gas、程序版本与父链最终性 |
| 待领余额和领取接口 | L2 Withdrawal Ledger | 已验证扣账才生成可领取权益 | 防重复兑付、转账失败、手续费和限流 |
| Desert / Escape Hatch | L2 托管合约中的冻结与退出模式 | 无需运营方协作的权益退出 | 未平仓头寸估值、未处理充值退款、父链可用性 |

对应关系是一条**交易应用 Rollup 协议**的迁移。多条 L3 可以使用共同平台组件，但每条链必须有独立的状态、消息命名空间、托管权益和损失承担范围。

## 7. Mantle 版本的参考拓扑与操作流

### 7.1 L2 在上，L3 在下

```text
Mantle L2: parent settlement
  +-- Escrow + Priority Inbox
  +-- Batch commitments + data commitment
  +-- Proof verification + execution progress
  +-- Pending withdrawals + claims
  +-- Forced requests + frozen-state exit
                |                         ^
       deposits / priority requests       | data / proofs / effects
                v                         |
Mantle App-specific L3: trading execution
  +-- API -> Sequencer -> Orders / Margin / Liquidation
  +-- Full execution state -> Public account / market state
  +-- Batch account deltas -> Data publisher
  +-- Witness -> Execution proofs -> Aggregation / wrapper
```

日常下单与撮合不经过上方的 L2 合约。用户分配资金后，L3 在其余额和风险限制内持续执行。

### 7.2 原生充值

用户在 Mantle L2 存入抵押资产；Priority Inbox 记录充值；L3 认证并消费消息，增加交易余额。L2 接受后续批次时检查所消费的请求确实来自父链队列，证明约束充值与状态变化一致。

L3 可按明确策略给出暂定入账，但必须关联 Mantle 区块历史。若父层输入被重组，L3 对依赖状态恢复；不允许将未经约束的暂定权益绕过原生结算变成不可撤回支付。这个确认策略需要测量和验证，不能只由“单 Sequencer”推导绝对最终性。

不要求充值也经过一套通用 Solver 网络。只有原生路径不能满足具体产品体验时，再评估 ERC-7683 或其他垫资实现。

### 7.3 本地交易与批次结算

交易改变 L3 账户、仓位和市场状态。执行器同时维护可证明的公共退出状态，批次聚合账户差分和市场数据。父链记录该批次的数据承诺、优先请求消费进度、前后状态承诺和待执行操作承诺，再验证证明并推进执行进度。

状态、证明和 DA 的一致性必须贯穿全部阶段：只验证“业务算得对”，却没有证明公开差分就是该业务的输出，用户退出仍然可能失败。

### 7.4 原生提现

L3 先检查可提金额并扣账，生成批次提现操作。L2 验证批次后执行该操作，在本链记录用户待领余额；领取时支付资产并减少该余额。暂未执行的批次、已验证但未领取的余额、以及已实际付款需要分别查询。

这是 Mantle 基础设计中的推荐参考方式。通用 Outbox 仍可作为抽象名称，但不必额外要求“每位用户都领取一个 Merkle 消息”，也不需要仅凭 Sequencer 签名释放托管资金。

## 8. 迁移中最主要的三个适配点

### 8.1 DA：Ethereum Blob 不能直接当作 Mantle L2 的本地 Blob

固定版本的 Lighter `commitBatch` 只接受 `PubDataMode.Blob`；`_processBlobs` 读取 `blobhash`，并使用 point-evaluation precompile 绑定实际发布的 Blob。[L4b][L4c] 构建脚本出现 `PUBDATA_MODE` 变量，不足以证明端到端 calldata 模式已经可用。

标准 OP Stack 在 Ecotone 规范中明确禁用 L2 Blob 交易。[L8] Mantle 将自己的批次发布到 Ethereum Blob，与“允许 L3 在 Mantle 上发送 Blob 交易”是两件事；本文没有完成 Mantle 当前部署对该能力的专项验证，不能把它视为已有接口。

**参考适配**：把账户差分与必要市场数据作为 Mantle L2 交易 calldata 发布，在结算合约中计算数据承诺，证明公开输入绑定同一承诺，并约束解码结果与公共状态更新一致。超过单笔数据限制时，分块数据按批次号、块数、顺序和内容承诺绑定，只有完整发布后才能验证结算。

这项改动涉及发布器、合约、证明包装与恢复工具。若采用 Lighter 的 Blob 电路，不能只替换 RPC URL；若自建 SP1 程序，则可以直接实现选定的 calldata 承诺规则。calldata 方案是否更便宜仍需测量，不能因为父链是 L2 就预设总费用下降。

### 8.2 父链确认、重组和退出依赖

迁移后，L3 的父链是 Mantle，Mantle 又依赖 Ethereum。需要分别记录 L3 批次处理进度、Mantle 区块确认程度以及父层最终性。Proof 已通过并不使包含验证调用的父链区块不可重组。[L9]

Lighter 合约配置中的优先请求过期时间为 14 天；这是所读版本的策略值，不应原样作为 Mantle 参数。[L4d] 更短期限会改善退出体验，但也需能容忍父层中断、证明积压和数据获取延迟。

L3 退出到 Mantle L2 不等于在 Mantle 停机时直接退出 Ethereum L1。后者需要额外的父层恢复或退出能力，不能由下移 Lighter 合约自动得到。

### 8.3 身份、资产与验证环境

需要替换并测试父链及子链身份、签名域、合约地址、账户归属、Gas 资产、资产精度、Oracle 与市场配置。Solidity 和 EVM 兼容有利于复用父链合约思路，但不能代替 Verifier 的 precompile、Gas、返回值与证明后端兼容性测试。

代码中为 Ethereum 原生资产设计的路径，也不能按名字原样假定适用于 Mantle。先用一种明确支持的抵押资产贯通闭环，再扩多资产与股票抵押。

## 9. Escape Hatch 应如何具体借鉴

白皮书 §6 说明：优先请求超时后冻结协议状态，用户基于公共账户及市场信息，证明资产、持仓盈亏和池份额价值并退出。白皮书的未平仓头寸退出说明使用最新 mark price；固定版本退出电路也读取公共市场的 `mark_price` 和资金费信息。[L1][L3d]

所读合约还包含：每账户每资产的退出去重、由退出证明增加待领余额，以及对未处理充值的退款处理。[L4e] 这比“停机后凭余额退出”更完整。

Mantle 版本应继承三项设计要求：

1. **退出状态可重建**：退出者不需要运营方提供私有数据库才能计算权益；数据保留和可验证快照需要长期维护。
2. **持仓有终止规则**：冻结状态、价格、资金费和保险 / 坏账规则必须事前确定。股票休市与过期价格需要专门规则，不能自动照搬。
3. **未消费充值也能取回**：用户已向 L2 存款但 L3 尚未处理时，不应因进入退出模式而失去退款路径；退款不能与已消费充值双重兑付。

同时区分“故障后用户可退出”和“第三方能无缝接管原订单簿继续交易”。Lighter 的公共差分模式为前者提供具体参考，后者需要额外的完整执行数据和接管协议。

## 10. 代码复用与开发工作量的影响

### 10.1 架构参考、代码适配与完整产品交付不同

公开 prover 仓库包含电路、证明包装、基准和退出工具；公开 contracts 仓库包含父链合约。它们是很有价值的研究对象，但本轮没有取得一套经确认完整可运行的生产 Sequencer、Indexer、Witness 调度和部署运维包。[L3][L4]

两个固定版本的根 LICENSE 均为 **Business Source License 1.1**，`Additional Use Grant: None`，列出的 Change Date 为 `2029-01-01`，Change License 为 GPL v2.0 or later；具体转换还受许可证条款约束。[L5][L6] 因此不能将公开可读等同于当前可无条件生产部署。直接代码复用需要先确认有效授权范围；本轮只做架构研究与只读核验。

### 10.2 对现有 ROM 的处理

09 中 42–68 直接人月的参考估计是“使用通用库和证明框架、自建首条 L3 闭环”的场景。本轮增加了具体设计依据，但没有取得足以重新报价的完整可复用资产，因此不直接把工期打折。

| 工作包 | Lighter 参考带来的帮助 | 仍要完成的工作 |
|---|---|---|
| 协议和资产模型 | 可借鉴优先队列、批次阶段和待领余额 | Mantle 父链确认、隔离额度、消息域和升级 |
| Perps 业务状态机 | 可借鉴订单树、风险检查及内部执行周期 | 产品规则、Oracle、实现、测试及审计 |
| 证明 | 已有专用电路和包装可研究 | 后端选择、许可、性能、公开输入和版本适配 |
| DA | 已有公共状态与账户差分模型 | calldata 或选定 DA 的新绑定与恢复验证 |
| 退出 | 已有持仓价值、池份额、未处理充值路径 | Mantle 超时、股票风险、用户工具和故障演练 |
| 运行服务 | 组件职责明确 | 完整运行栈的代码盘点、部署和持续运维 |

如后续获得生产授权和完整参考实现，应把已有代码与 09 工作包逐项匹配，分别估算复用、改造、重测和审计量，再重估预算。授权费、基础设施、证明成本与实际运维责任另列。

## 11. 纳入 Mantle L3 方案的设计结论

本轮将 Lighter 作为 Perps L3 的主要架构参照，采用以下设计基线：

- 父链托管资产，交易者预存资金；高频撮合、仓位和保证金在 L3 本地闭环。
- 父链优先队列负责充值和关键强制请求，证明与队列消费记录一起约束输入。
- 执行状态、公共退出状态和批次账户差分分别建模，证明它们的一致性。
- 结算区分 Commit、Verify、Execute 和 Claim，用户 API 显示各自进度。
- 原生提现使用批次父链操作与待领余额作为具体参考；Solver 与跨层授信保持可选。
- 优先研究向 Mantle L2 发布 calldata 差分的适配；不预设本地 Blob 能力。
- Escape Hatch 覆盖持仓估值和未处理充值，不仅覆盖余额证明。
- 证明后端保留选择：架构参考 Lighter，不把 Lighter 实现误写成 SP1，也不要求 L3 与 Mantle 使用同一证明体系。

这仍属于三路线比较中的路线三细化，不表示路线一和 MantleCore 已被排除。

## 12. 来源与复核入口

### 一手资料

- **[L1] Lighter 白皮书，October 2025**：重点 §2–§6，§2.1 的 DA 目标、§3.2 的公共账户差分、§5 的 Blob 绑定、§6 的退出估值。<https://assets.lighter.xyz/whitepaper.pdf>
- **[L2] 官方架构文档**：<https://docs.lighter.xyz/about-lighter/technical-architecture-lighter-core>
- **[L3] Prover 固定版本**：<https://github.com/elliottech/lighter-prover/tree/38bea38e59be67a5381665e9562ec0c7508e8bee>
- **[L3a] 证明配置**：<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/circuit/src/types/config.rs>
- **[L3b] 包装及构建**：<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/snark/builder/build_circuit.go>；<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/build_circuits.sh>
- **[L3c] 基准计时入口**：<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/bench/src/bin/bench.rs>
- **[L3d] 退出电路**：<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/desertexit/circuits/src/inner_circuit.rs>
- **[L4] Contracts 固定版本**：<https://github.com/elliottech/lighter-contracts/tree/75c2a73a5c25c9a6a36b66ba73aaf71267232b42>
- **[L4a] 充值和优先队列**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/AdditionalZkLighter.sol#L607>
- **[L4b] 批次状态机**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/ZkLighter.sol#L438>；待领余额 <https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/ZkLighter.sol#L720>
- **[L4c] Blob 绑定**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/ZkLighter.sol#L864>
- **[L4d] 超时配置**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/Config.sol#L75>
- **[L4e] 退出和未处理充值退款**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/contracts/ZkLighter.sol#L335>
- **[L5] Prover LICENSE**：<https://github.com/elliottech/lighter-prover/blob/38bea38e59be67a5381665e9562ec0c7508e8bee/LICENSE>
- **[L6] Contracts LICENSE**：<https://github.com/elliottech/lighter-contracts/blob/75c2a73a5c25c9a6a36b66ba73aaf71267232b42/LICENSE>
- **[L8] OP Stack Execution Engine**：Ecotone 禁用 L2 Blob 交易。<https://specs.optimism.io/protocol/exec-engine.html#ecotone-disable-blob-transactions>
- **[L9] OP Stack Derivation**：父链推导、确认级别和重组。<https://specs.optimism.io/protocol/derivation.html>

### 交叉检查

- **[L10] zkSecurity 第三次审计**：<https://reports.zksecurity.xyz/reports/lighter-3/>
- **[L11] L2BEAT Lighter**：<https://l2beat.com/scaling/projects/lighter>

此前的 [Lighter 技术报告](../references/lighter_zk_rollup_technical_research.md)中，固定加速倍数、单笔美元成本和跨系统约束数比较缺少可比基准，不用于本方案的性能或预算依据。
