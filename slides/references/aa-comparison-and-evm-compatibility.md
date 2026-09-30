# Section 3 补充依据：传统 AA、原生需求与 EVM 兼容性

本稿补充用户提供的三份 [AA 研究材料](aa/03-mantle-aa-coverage-and-fit.md)，用于 Section 3 的比较、机制与路线结论。三份材料已经完整读取；其标注的 2026-09-24 为原始版本背景。本轮进一步核对 EIP/RIP 在线草案、EF 一手文章和固定版本 Bundler 源码；新增口径见 §2、§7–9。草案仍会变化，未运行客户端、账户合约或性能基准。

## 1. 4337 与 7702 不在同一抽象层

| 维度 | ERC-4337 | EIP-7702 |
| :-- | :-- | :-- |
| 改变什么 | 引入 UserOperation、Bundler 与 EntryPoint 这套应用层提交和验证路径 | 允许 EOA 设置代码委托，调用原地址时执行指定代码 |
| 链改动 | 4337 本身不要求新增共识规则 | 链必须支持对应交易类型与委托执行语义 |
| 账户入口 | 由智能账户验证 UserOperation，再执行调用 | 普通 EOA 保留地址与资产，委托代码提供可编程逻辑 |
| 自定义认证 | 账户合约可定义 UserOperation 的验签策略 | 委托设置仍由 EOA 原密钥授权；后续受限操作认证由委托代码实现 |
| 费用 | Bundler 支付外层链上交易费，EntryPoint 按规则向账户或 Paymaster 结算 | 可由其他交易发送者付费，但完整赞助策略与预算需要 Relayer、账户或结合 4337 |
| 并发 | EntryPoint 提供 key/sequence nonce；并发接纳取决于 Bundler 策略 | 本体不将原生交易 nonce 改成多通道；委托逻辑或 4337 可补充应用层操作序号 |
| 部署与迁移 | 可用新智能账户，也可服务已通过 7702 委托的账户 | 用户可保留原 EOA 地址，无需为获得可编程执行而必然迁移资产 |
| 主要代价 | UserOperation 模拟、打包、EntryPoint 包装与费用预留；依赖提交服务的可用性 | 委托代码、初始化与存储布局成为账户安全边界；原密钥及旧交易入口仍需管理 |

不能把 4337 写成“必须换地址”，也不能把 7702 写成“无需任何链改动的完整 Native AA”。当前 4337 规范明确支持 7702 授权路径：可将“保留旧地址”和“标准化 UserOperation 提交/代付”组合使用。

### 两项容易影响产品的失败语义

- 4337 外层交易成功，不代表每个 UserOperation 的业务成功。SDK 要解析对应操作回执与业务状态，而非只看 bundle 的交易状态。
- 7702 中已经处理的代码委托，不会因后续业务执行失败自动回滚。委托可以持续存在，清除/更改委托与撤销某个 Session Key 是不同操作。

安全边界也不同：Bundler 能影响接收、纳入和可用性，但正确签名和验证机制不赋予它任意转走账户资产的权力；7702 委托代码在原账户上下文执行，需严格验证来源、初始化和升级/存储兼容。

### 1.1 真实示例：SimpleAccount 与 Simple7702Account

本次检查 `eth-infinitism/account-abstraction` 的固定提交 **`1c6b669d0eea734e09a87e095ba15e076151718a`**（2025-12-16）。以下是源码级教学示例，没有部署钱包、广播授权或运行资金交易。

**4337 的独立账户示例**：

- Alice 的原 EOA 为 E，另有已经创建的智能账户 A，A 使用 `SimpleAccount` 实现，存储 `owner = E`。
- 100 USDC 归 A 持有，Alice 签署 `sender = A` 的 UserOperation，其 callData 调用 `executeBatch`。
- Bundler 调用 EntryPoint；EntryPoint 调用 `A.validateUserOp`。`SimpleAccount._validateSignature` 检查恢复出的签名者是否等于 `owner`。
- 通过验证后，EntryPoint 调用 A 的批量执行入口，A 再调用 USDC 和 Router。USDC allowance 的授权主体、Router 收到的直接调用者是 A，兑换得到的资产按参数返回 A。
- 这是常见的新智能账户路径；4337 本身也能处理 7702 委托账户，不因此要求所有用户迁移地址。

**7702 的实际委托目标示例**：

- 已部署的 W 是 `Simple7702Account(EntryPoint)` 钱包实现合约，构造参数确定它接受的 EntryPoint。它是专为 7702 批量执行与 4337 接入设计的参考代码，不是 DEX 或 Router。
- Alice 对 E 签署指向 W 的 7702 授权，链在 E 上设置 `0xef0100 || W` 委托标记。之后调用 E 时，EVM 读取 W 的代码，**在 E 的账户上下文执行**。
- 因而钱包执行中的 `address(this) = E`，账户存储仍属于 E，ERC-20 余额仍记为 `balanceOf(E)`；资金不会因为设置委托而转给 W。
- 该实现继承 `BaseAccount.validateUserOp`、`execute`、`executeBatch`；它的 `_checkSignature` 检查 `ECDSA.recover(hash, signature) == address(this)`，在 E 的上下文就是验证 Alice 的原账户签名。
- 它允许来自 EntryPoint 或 E 自身的执行调用。P29 先展示独立的 Owner 自付路径：Alice 用 E 发普通交易调用 E.executeBatch，由 E 自己支付原生费用，钱包的执行入口检查允许这种自调用；此路径不会额外经过 validateUserOp。
- 另可结合 4337：UserOperation 的 sender 设为 E，EntryPoint 调用 E 的验证与批量执行入口，底层运行 W 的代码。E 向 USDC/Router 发起普通 CALL 时，直接调用者仍为 E。这也是 P29A 加入受限 Agent 与 Paymaster 时采用的提交路径。
- 7702 还可以配合其他安全的钱包入口/中继路径，不必永远经 4337；这里把基础自调用与 4337 组合分开，是为了分别说明可编程执行、操作验证和代付的职责。

这里的 E→W 是**代码引用关系**，不是 E 向 W 转账，也不是一次普通外部 CALL。基础授权只决定采用什么代码，并没有顺便设置“Agent K 只能花 100 USDC”。

### 1.2 受限 Agent 的逻辑要放进钱包实现

上述 `SimpleAccount` 与 `Simple7702Account` 都是最小单签示例，不自带完整的 Session key、累计额度或策略权限系统。P29A 使用明确标注的 `W_policy` 教学扩展，不将其说成仓库现有功能。

扩展必须形成完整流程：

1. Owner 选择支持受限策略的 7702 钱包实现，并以受保护的入口配置 K 的授权：目标、函数、资产、接收账户、期限、累计预算。
2. K 签署绑定完整操作、nonce、链和账户域的请求；验证逻辑检查身份与允许的调用。不能仅把“原密钥签名”换成“任意 K 签名都接受”。
3. 执行入口再次按实际状态检查和占用共享预算，核验批次中每个调用、接收人和结果。只在链下预检查额度不足以避免并发超用。
4. Paymaster 独立审核原生手续费的赞助与预算；支付网络费与使用 USDC/mStocks 买入是两本账。
5. Owner 撤权后，后续操作按生效状态拒绝；已发生的交易、现有代币 allowance 与未平仓头寸分别处理。

Owner 的代码委托授权与 Agent 的每次业务签名不是同一份授权。根 EOA 密钥保留管理能力，不交给 Agent。钱包代码更新、初始化和存储布局也必须安全；简单给任意 Router 地址设置委托，不能替代钱包验证入口。

### 1.3 教学买入批次的边界

P28/P29 先假设钱包已准备好，持有 100 USDC；Paymaster 若赞助则另有足够原生费用预算。业务批次为有限授权 Router、兑换 mStocks 并买 Meme，所得发送回实际钱包地址。

`BaseAccount.executeBatch` 在子调用发生 EVM revert 时回滚该次批量调用，但不会自动理解所有 ERC-20 的 false 返回值或所有产品成交条件。生产实现须核对代币返回值、授权残留、最小到手量及实际余额；业务回滚也不退回已消耗的交易费用和操作 nonce。

**固定版本代码**：

- [SimpleAccount.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/accounts/SimpleAccount.sol)：Owner 与 UserOperation 签名验证。
- [Simple7702Account.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/accounts/Simple7702Account.sol)：7702 账户上下文验签和 EntryPoint 接入。
- [BaseAccount.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/core/BaseAccount.sol)：`validateUserOp`、`executeBatch`、执行入口限制与费用预存。
- [SimpleAccountFactory.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/accounts/SimpleAccountFactory.sol)：独立 SimpleAccount 的创建与初始化。
- [EntryPoint.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/core/EntryPoint.sol)：`handleOps`、账户验证、执行调用与费用结算。


## 2. Native AA 的产品动机：降低重复开销，缩短纳入路径

高频 Agent 会反复提交小额买入或链上挂撤改单。每次业务之外的固定开销会累积；额外的提交等待则可能使限价失效、撤单延后。这才是 Mantle 评估 Native AA 的实际动机，不能只用“希望协议原生，所以需要原生”证明必要性。

### 2.1 手续费：有可消除的包装开销，但没有统一节省比例

[RIP-7560 Motivation](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7560.md) 明确将 4337 的额外 Gas 列为动机，原文为：

> Extra gas overhead of ~42k for a basic `UserOperation` compared to ~21k for a basic transaction.

这是该提案对基础操作的历史论据，不是 Mantle 实测，也不能直接换算成当前所有钱包“降低一半费用”或“每笔节省 42k”。账户代码、EntryPoint 版本、是否首次部署、批次分摊与 L2 数据费都会改变结果。

4337 的成本除了必要的业务与认证，还包括外层交易、UserOperation 编解码、EntryPoint 调度及费用记账等。原生化将调度和部分记账移入协议，能去掉其中可消除的合约包装；**必要的验签、重放检查、付款检查与状态处理并不会凭空消失**。不同方案把哪些成本原生化、怎样计价，需要分别测量。

[EF 2026 协议优先级](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026) 将 Native AA 放在 Improve UX，提出智能合约钱包无需 Bundler、Relayer 或额外 Gas overhead 的方向，并提到 8141。这是研究与升级目标，不是已经实现的全链费用承诺。

### 2.2 纳入：Bundler 多一道调度，但 4337 不强制长时间攒批

UserOperation 先经 Bundler 接收、模拟、排队与组包，再成为提交给链的外层交易。其费用门槛、批次大小、发送周期、替换/重试和服务可用性，均可能影响何时纳入；随后仍要经过排序器或区块构建者。Bundler 返回操作 hash 不是入块承诺。

可复查的实际例子是 [Pimlico Alto](https://github.com/pimlicolabs/alto/tree/96529592b67a69be23c013359cbc9990657af64a)，固定提交 `96529592b67a69be23c013359cbc9990657af64a`：

- [options.ts](https://github.com/pimlicolabs/alto/blob/96529592b67a69be23c013359cbc9990657af64a/src/cli/config/options.ts) 将自动组包最小/最大间隔默认设为 **100/1000 毫秒**，并提供 auto/manual 模式。
- [executorManager.ts](https://github.com/pimlicolabs/alto/blob/96529592b67a69be23c013359cbc9990657af64a/src/executor/executorManager.ts) 的 `autoScalingBundling` 先提取并发送批次，再根据近期操作量在上述区间安排下一轮。
- 这些是该实现的调度参数，**不是每笔操作必须等待 100–1000 毫秒，也不是确认延迟的上下界**。它们不能代表所有 Bundler；模拟时间、其他队列与链上等待另算。

Native AA 使账户操作本身成为链可接收的交易，消除对独立 UserOperation→外层交易转换的结构性依赖。RIP-7560 还指出，协议外 UserOperation 不能同等享受针对交易的抗审查机制，并依赖较少的参与节点与非标准 RPC。但原生交易仍会排队、模拟和竞争区块空间；排序器策略、RPC 与赞助服务也可能继续存在，不能承诺立即纳入。

### 2.3 应怎样验证

固定业务、安全策略与付款方式，分别记录：首次设置费、后续单操作总费、失败/重试费、L2 数据费及额外服务收费；时间拆为接收→模拟完成→提交排序器→入块→业务实际生效，报告中位数与 P95/P99、超时率和失败率。对照优化过的 7702/4337 基线，而不是故意设置长时间攒批。

原生有效性与业务成功也要分开：即使账户验证和付款批准进入协议，滑点、余额或业务 Policy 仍可能在执行时失败并产生费用。Session、代付、限权和批处理可先由传统方案交付；原生路线的价值在于同等约束下减少总开销、缩短并稳定纳入路径。

## 3. EVM 兼容是选型前提，非 EVM 案例不能证明必然代价

本轮用户明确的决策口径是：尚未锁定 Native AA 方案，主要顾虑在于它会不会损害 Mantle 已有 EVM 生态。兼容评估需覆盖既有合约字节码与调用语义、钱包/RPC/回执，以及客户端和验证/证明路径；不能只看“仍用 Solidity”。下面两个系统作为背景保留，不再用作 P30C 的主要论据。

| 系统 | 官方资料中的账户机制 | 执行与开发栈 | 对 Mantle 的启示 |
| :-- | :-- | :-- | :-- |
| Aztec | 原生账户抽象，由账户合约定义认证与执行入口 | Noir；私有操作在 PXE 内执行/生成证明，公开操作使用 AVM | 可学习可编程账户入口，但普通 EVM 字节码不是其原生应用路径 |
| Starknet | 账户合约原生参与交易，INVOKE 先 `__validate__` 再 `__execute__` | Cairo / Sierra 与 Starknet 执行体系 | 可学习协议级验证/执行分离，但不能把已有 EVM 应用视为直接原样迁移 |

准确说法是：它们在各自非 EVM 执行栈中采用了原生 AA，不是“为了 Native AA 才丢失原本已有的 EVM 兼容”。Native AA 与 EVM 兼容性是两个设计维度。

Starknet 官方账户文档仍描述协议级顺序 nonce，说明“原生账户”也不自动包含所有并发模型。Aztec 的隐私账户设计同样不能直接当成 EVM 账户规范照搬。

Mantle 要研究的组合是：**保留现有 EVM 合约与资产生态，同时让链原生处理可编程账户交易。** RIP-7560、8130、8141 都在 Ethereum/EVM 体系中提出相应变化；这仍需要客户端、系统合约、工具与验证路径适配，不等于与未修改的 EVM 完全等价。

## 4. RIP-7560 怎样内化 4337 的职责

| 职责 | 4337 | RIP-7560 |
| :-- | :-- | :-- |
| 被链接收的对象 | Bundler 的外层交易；UserOperation 在其内 | 账户作为 sender 的原生 AA 交易 |
| 验证与执行调度 | EntryPoint 合约组织各阶段 | 协议规定并调用部署、账户/Paymaster 验证、执行与后处理 |
| 费用处理 | Bundler 先付外层费用，EntryPoint 结算 | 协议检查付款能力、预扣与结算 |
| 可复用的角色 | Account、可选 Paymaster、部署服务 | 保留相近分工，但需适配回调、交易格式与准入 |

内化的是生命周期和职责，不是原封不动把 EntryPoint.sol 搬进客户端；也不表示协议不再调用钱包代码。其实际次序为：

1. 读取 sender、可选 deployer、可选 Paymaster、验证与执行数据。
2. 如需创建账户，执行部署步骤。
3. 执行账户验证，并通过协议入口的明确回调表示接受。
4. 如有 Paymaster，独立验证付款授权并表示接受。
5. 执行账户的业务调用。
6. 按条件执行 Paymaster 后处理，结算费用并输出分阶段结果。

验证失败使交易无效；业务执行失败不免除已经发生的费用。当前草案中 Paymaster 后处理回滚还会导致相应业务执行回滚，因此不能把后处理概括为完全不影响业务的日志步骤。

非零 nonce key 的多通道能力由伴随的 RIP-7712 定义。不能只看到 7560 信封里有 `nonceKey` 字段，就认为不需配套规范和实现。

7560 的优势是保留较明确的账户/Paymaster 生命周期和已有 4337 经验；代价是协议化后的交易处理、准入、安全与验证适配。没有完整基准时，不用规范中的基础 gas 常量宣称某个固定降费比例。

## 5. 8130 与 8141 在本章中的比较口径

- **8130**：Account 保存资产身份，Actor 是操作者，Authenticator 识别签名身份，Keystore 记录授权。POLICY Manager 门禁只保证入口，业务预算和可移动资金的权限仍需账户/产品实现。原生多通道与 nonce-free 不等于每个 Agent 自动独占通道或隔离余额。
- **8141**：交易由 Frames 组成；VERIFY 检查授权并显式批准执行/付款，SENDER 执行业务，DEFAULT 处理辅助动作。验证器必须检查所有后续受授权动作；原子组失败后，组外动作仍可能继续。
- **8130 Phase**：组内原子，前组成功可保留，当前组失败后跳过后组。
- **8141 Atomic group**：只回滚指定连续组；组外后续 Frames 可继续，条件依赖必须显式表达。
- **并发比较**：8130 内置 key/sequence；8141 本体为标量 nonce，完整比较应加入 EIP-8250。两者都要单独评估准入限制、共享预算和市场状态。
- **EVM 应用兼容**：8141 当前材料指出跨 Frame 清理瞬态存储。此前的 v4 Flash Accounting 不能随意拆到多个 Frames；完整 unlock/callback/settle 应在相容的调用边界内完成并测试。

这些要点来自用户材料，不构成生产实现验证。示例统一使用用户账户 U、Agent K、Sponsor P，以受限预算兑换 mStocks 并买入 Meme；若用代币报销费用，要预留或扣减预算，不让同一笔 100 USDC 被业务和报销重复承诺。

## 6. 来源

### 用户提供的主材料

- [8130 Deep Dive](aa/01-eip-8130-deep-dive.md)：角色、Policy、交易生命周期、nonce、撤权与 Phase。
- [8141 Deep Dive](aa/02-eip-8141-deep-dive.md)：Frame、APPROVE、原子组、逐帧结果、Gas 与 8250。
- [Mantle 能力覆盖与适配评估](aa/03-mantle-aa-coverage-and-fit.md)：需求映射、产品适配、缺口与同条件验证。

### 本次补充核对的官方资料

- [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)：Specification、Nonce、7702 支持、Paymasters、Bundler 行为。
- [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)：授权处理、委托持久性、执行失败不回滚委托、初始化安全。
- [RIP-7560](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7560.md)：验证/执行阶段、Paymaster、后处理、兼容路径。
- [RIP-7712](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7712.md)：7560 的多维 nonce 配套。
- [Aztec Accounts](https://docs.aztec.network/developers/docs/foundational-topics/accounts)：原生合约账户与入口。
- [Aztec Overview](https://docs.aztec.network/developers/overview)：Noir、PXE、AVM。
- [Starknet Accounts](https://docs.starknet.io/learn/protocol/accounts)：验证/执行、费用、nonce。
- [Starknet Protocol Introduction](https://docs.starknet.io/learn/protocol/intro.md)：合约账户与 Cairo 执行语境。

## 7. 8141 的动机、隐私与 CROPS；8130 的两种 profile

### 7.1 能确认什么，不能确认什么

[EIP-8141 Motivation](https://eips.ethereum.org/EIPS/eip-8141) 明确列出后量子签名迁移、账户与 ECDSA 密钥解耦及原生换钥、批处理，以及不依赖中心化第三方 relayer 的替代付费机制。愿景是账户成为带代码的地址，以 EVM 编程实现验证和付款。本次读取的规范正文没有以 privacy 作为独立动机条目；因此**不能确认“8141 最初主要为了隐私”**。可以确认的是，它提供通用交易结构，不专门定义 Session Key、Agent 权限或限额标准。

隐私确有直接的一手证据：[EIP-8250 Motivation](https://eips.ethereum.org/EIPS/eip-8250) 把隐私协议共用 sender 列为首要例子：多用户不使用各自唯一公开 sender，却会争用同一线性 nonce。独立 key 可以从隐私 nullifier 派生，避免无关操作在协议重放规则上互相作废。这里仍需隐私协议本身提供证明与匿名集合；公开 key 并不自动隐藏交易数据。

8141 的验证和独立付款结构可服务这类协议：在合适的验证与准入约束下检查证明，由付款方承担 Gas，减少用户向新地址预充值和专用中继依赖。它不是完整隐私方案，不自动隐藏付款方、金额、调用内容或网络元数据。

### 7.2 与 L1 的 CROPS 方向怎样联系

[EF Mandate](https://ethereum.org/foundation/mandate/) 于 2026-03-13 发布，将 **Censorship Resistance、Open Source and Free（as in Freedom）、Privacy、Security** 合称 CROPS，即抗审查、开源与自由、隐私、安全。

这是 Ethereum 的总体发展原则，不是 8141 的功能认证。可以作出的设计判断是：减少强制中继、开放可编程验证、支持隐私应用付款和密码学迁移，与 L1 的长期方向相符；抗审查纳入仍需要链的纳入机制，安全与隐私仍需要具体实现。不要把四个词画成“采用 8141 即全部满足”的勾选表。

### 7.3 与 8130 的比较须使用当前 profile

当前 [EIP-8130](https://eips.ethereum.org/EIPS/eip-8130) 标题为 **Keystore Accounts**，明确含两种 adoption profile：

- **L1 profile**：计费参数属于规范共识，仅经硬分叉调整；canonical authenticator 固定成本原生验证，也接受在 MAX_AUTHENTICATION_GAS 内完成的其他 authenticator STATICCALL。
- **L2 profile**：计费表可按本链成本模型配置，但必须是本链共识；原生认证路径仅接受 canonical set，避免任意认证代码进入高频验证路径。其他认证器仍可经 EVM/4337 等途径使用。

本次判断：8130 的 L2 profile、标准 Actor/Keystore 和基础授权更直接贴合 Mantle 的交易型 L2 账户需求，值得优先做 PoC；8141 更通用的 Frame 结构有利于评估 L1 式的密码学演进、付款与隐私扩展。**不能写成 8130 只适用于 L2、8141 只适用于 L1**，更不能由定位直接推导性能优胜。

## 8. 8250 怎样沿着 8141 的结构扩展 nonce

8141 提供交易信封、Frames、验证/付款批准、执行与回滚边界。钱包可在验证代码中实现 Session、限权或不同签名算法；改变协议重放规则则需要配套共识扩展。8250 正是后一种，不是给钱包加一个普通合约插件。

1. **交易头**：将 8141 的标量 `nonce` 替换为 `nonce_keys, nonce_seq`，Frame 布局不变。签名摘要覆盖替换后的字段。
2. **协议状态**：`[0]` key 别名到 sender 的旧账户 nonce；非零 key 的序号存在 `NONCE_MANAGER` 系统合约下，以 sender/key 派生槽位。普通调用该合约会 revert，只有协议写入这些槽位。
3. **执行前有效性检查**：每个选中 key 的当前序号都须等于同一个 `nonce_seq`。一笔交易选择多个 key 时，不是每个 key 任意带自己的序号。
4. **接入现有批准点**：在 8141 唯一成功的付款批准 APPROVE 转换中，收取首次 key 状态成本并一次性消耗全部选中 key，与付款批准原子提交。若当前 Frame 回滚，批准与消耗一同回滚；该 Frame 成功后，后续业务 Frame/原子组失败不会恢复已消费 key。无效交易的全部效果须丢弃。
5. **验证器读取**：扩展 TXPARAM 的 nonce 内省字段，验证代码可以核对获授权的域；不能假定任意 Agent 天然只拥有某个 key。

例：同一 sender 的 Agent K1 使用 `[11], seq=0`，K2 使用 `[22], seq=0`。首笔消耗 11 后，22 仍为 0，因此第二笔不因这次 nonce 更新而失效。若两笔共享 key，就仍需遵守该域的顺序。隐私 nullifier 也可作为独立域来源；验证器还须要求首次序号，保证单次消费。

**准入限制必须保留**：当前 8250 明确继承 8141 公共 mempool 每个 sender 只保留一笔待处理 Frame 交易的限制。它移除协议层 nonce 障碍，为未来 keyed-aware mempool 留下空间，没有自动交付同 sender 多笔公共池并发。新 key 还有持久状态增长成本，不等价于短期 nonce-free。

## 9. Mantle 的近期 AA 组件与长期选型

近期以链支持 7702 为前提，交付可供 **Agent Meme Launchpad 与股票 Perps** 共用的钱包实现、SDK 和服务接口；标准提交与代付可组合 4337。7702 负责原地址使用代码，其余功能来自安全的钱包和产品适配，不能从最小 Simple7702Account 直接推定已具备：

| 组件能力 | 首批交付与验收 |
| :-- | :-- |
| 有限授权与撤权 | Session Key；目标、函数、资产、收款人和期限；撤权后拒绝旧授权 |
| 分层权限 | 交易与提现分离；子账户/策略隔离接口，避免 Agent 获得根密钥 |
| 预算 | 本金/累计额度、Gas 赞助、代币报销分账；执行时核对共享预算 |
| 组合执行 | 有限代币授权、兑换与买入；明确失败回滚与残余 allowance |
| 多任务与恢复 | 应用/4337 操作序号、任务去重、重试、回执与状态恢复；不冒充原生多通道 |
| 两产品接入 | 统一 SDK；Launchpad 买卖/退出；Perps 订单签名、撤单与权限适配 |

撤销 Session 不会自动取消所有已签订单、收回已发 allowance 或平掉头寸；这些动作须接入产品自身的生效规则。Perps 风控、保证金、仓位与撮合仍由市场系统负责。

长期用同一组功能验收原生方案：成本与纳入收益、同账户多任务、失败恢复、EVM/RPC/工具兼容及客户端/证明维护成本。8130 L2 profile 可优先验证，8141+8250 与 7560 保留对照；在证据完成前不锁定生产方案。上述为路线与交付要求，没有宣称组件或原生升级已经完成。

