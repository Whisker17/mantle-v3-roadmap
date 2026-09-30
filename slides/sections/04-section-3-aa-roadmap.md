# Section 3｜AA 路线：从可编程钱包走向原生账户

本章共 **22 页**：S3 分隔页、20 页正文、C3 总结页。按原纲 3.1 现有方案、3.2 Native AA 的顺序展开；P28/P29 分别用实际参考钱包解释两种机制，P29A 补充受限 Agent 扩展。新增 P31J 讨论隐私与 L1/L2 定位，P32A 保留长期方案验收；后续章节不重编号。

本章的主张是：近期用 7702 建设两个核心产品共用的 AA 组件；长期通过 Native AA 减少高频操作的额外开销与纳入环节，同时守住 EVM 兼容边界。Session keys、代付和批处理可先交付，原生方案则以同等安全约束下的真实收益决定选型。

| 页面 | 讲述任务 |
| :-- | :-- |
| S3 | 说明账户需求、原生方向与 EVM 兼容目标 |
| P28–P29A | SimpleAccount 买入、7702 委托实际钱包代码、受限 Agent 策略扩展 |
| P30–P30A | 两者的技术优点、限制与组合方式 |
| P30B–P30C | 高频操作的费用与纳入需求，以及暂未锁定方案的兼容性顾虑 |
| P31 | 将 7560 与 4337 对照，解释生命周期怎样进入协议 |
| P31A–P31D | 8130 的身份、交易、并发/撤权和失败处理 |
| P31E–P31H | 8141 的 Frame、批准、原子组，以及 8250 怎样扩展它 |
| P31I–P31J | 两案机制差异，隐私、CROPS 与 L1/L2 定位 |
| P32–P32A | 近期组件功能清单与长期原生选型验收 |
| C3 | 先交付 7702 组件，再以产品收益选择原生路线 |

以用户提供的 [8130](../references/aa/01-eip-8130-deep-dive.md)、[8141](../references/aa/02-eip-8141-deep-dive.md)、[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) 为主要依据，补充 [传统 AA 与 EVM 兼容性核对](../references/aa-comparison-and-evm-compatibility.md)。草案按所引版本理解，不宣称已部署或已测出性能优势。全稿使用 [Mantle 深色规范](../prompts/Mantle-dark-capital-markets.md)。

P28/P29 先用 Owner 签名的最小参考钱包讲基础机制：A 为独立智能账户，E 为原 EOA，W 为委托代码实现；P29A 再加入 Agent 策略。后续 Native AA 教学例统一用用户账户 U、已授权的 Agent K 和付款方 P，在预算内兑换 mStocks 并买 Meme。报销须预留或扣减预算，最小参考钱包不被当作自带 Session 能力。

### S3｜AA 路线：让 Agent 成为链原生的操作主体

**本页要讲什么**

从上一章的账户需求进入实现路径。先看 4337 与 7702 怎样帮助产品交付，再解释高频操作为何需要减少额外开销与提交环节，最后在 EVM 兼容约束下评估原生提案。

**对应原文**

[Section 3：AA 路线](../references/chain-infra.mdx)，3.1 与 3.2；结合本轮用户对 Native AA 需求、兼容性与三种机制的要求。

**需要的图文及原因**

沿用统一章节分隔版式，标题下面保留一句主旨与三个阅读阶段。这里明确方向，不在入口页把 8130 或 8141 标成已选定方案，也不把原生账户等同于放弃 EVM。

**上屏文案**

标题：AA 路线：让 Agent 成为链原生的操作主体

眉标：SECTION 3

主旨：近期用 7702 支持产品；长期让原生账户减少操作开销与提交等待，并保留 EVM 生态。

本章路径：现有 AA → 费用与纳入需求 → 原生方案与交付路线。

## 3.1 现有 AA 方案：机制、优势与限制

### P28｜4337 示例：智能账户怎样用 100 USDC 买入

**本页要讲什么**

用真实参考实现 SimpleAccount 解释 4337。Alice 控制独立智能账户 A，100 USDC 归 A 持有；Alice 签署买入请求，由 Bundler 提交给 EntryPoint。A 的代码验证签名，再批量执行授权、兑换和买入，资产仍归 A。先讲 Owner 签名的最小例子，再在 P29A 引入受限 Agent。

**对应原文**

[3.1 目前已支持的 AA 方案综述](../references/chain-infra.mdx)；[SimpleAccount.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/accounts/SimpleAccount.sol)、[BaseAccount.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/core/BaseAccount.sol)、[EntryPoint.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/core/EntryPoint.sol)。示例核对见 [补充材料 §1.1–1.3](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

需要标注真实方法名的交易流程，明确 Alice 是签名者，A 是持有资金并执行的账户：

1. A 已经创建，采用 SimpleAccount，owner 为 Alice，持有 100 USDC。
2. Alice 签 UserOperation，sender 为 A，callData 指向 executeBatch；不要把 sender 写成 Bundler。
3. Bundler 发外层交易调用 EntryPoint，后者调用 A.validateUserOp；A 检查恢复出的签名者是否等于 owner。
4. 验证通过后，EntryPoint 调用 A.executeBatch，依次有限授权 Router、兑换 mStocks 并买 Meme，收款人指定为 A。
5. 可选 Paymaster 作为独立费用分支，不代替 A 执行业务或持有用户资产。

图要区分签名请求、合约调用与资金归属，不能画成用户把资产交给 Bundler 投资。关键方法名和资金路径均使用下方文案。

**上屏文案**

标题：4337 示例：智能账户怎样用 100 USDC 买入

Alice 的 SimpleAccount 钱包 A 持有 100 USDC。

Alice 签 UserOperation → Bundler 提交 → EntryPoint 调用 A。

A.validateUserOp 核对签名；A.executeBatch 完成授权路由、换入 mStocks、买 Meme，所得回 A。

Paymaster 可代付 Gas。验签与批量执行逻辑来自钱包代码。

**讲述补充（不上屏）**

参考实现的 _validateSignature 检查签名者等于 owner；A 通常由 SimpleAccountFactory 创建代理并初始化 owner，本例省略首次部署步骤。批量执行来自 BaseAccount，子调用发生 EVM revert 时回滚本次批次。生产方案仍要检查代币返回值、授权范围和实际成交条件，不能把最小参考实现当作完整策略钱包。

本例的 A 是独立合约账户，资产须在 A 可支配的范围内；不表示所有 4337 用户都必须迁移地址。下一页用 7702 让原 EOA 成为 4337 sender。Bundler 支付外层费用，再由 EntryPoint 向账户或代付方结算；结果要看 UserOperation 与业务状态。

### P29｜7702 示例：给原地址装上钱包逻辑

**本页要讲什么**

明确委托对象是已经部署的 Simple7702Account 钱包实现 W。Alice 签署代码委托后，原 EOA E 仍持有资产，并可通过一次自调用运行 W 的批量执行逻辑。本页先独立展示 7702 的 Owner 自付路径，下一页再加入 Agent 与 4337 代付；把“代码在哪里、账户是谁、资金归谁”分开讲。

**对应原文**

[3.1 目前已支持的 AA 方案综述](../references/chain-infra.mdx)；[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) 的 Code Delegation；[Simple7702Account.sol](https://github.com/eth-infinitism/account-abstraction/blob/1c6b669d0eea734e09a87e095ba15e076151718a/contracts/accounts/Simple7702Account.sol) 与其继承的 BaseAccount。身份与执行上下文见 [补充材料 §1.1](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

用设置委托和执行买入两段图，继续使用 100 USDC 场景：

- 账户框 E 是 Alice 原 EOA，继续持有 100 USDC；实现框 W 是已部署的 Simple7702Account。E 指向 W 的箭头标“读取钱包代码”，不能标为转账或普通 E→W 外部 CALL。
- Alice 用原密钥签署 7702 授权，链在 E 上记录代码委托。该授权只选择代码，不顺便设置“Agent 最多花 100 USDC”等业务权限。
- 委托设置后，Alice 用 E 发交易调用 E.executeBatch。EVM 读取 W 的代码，但 address(this) 与账户存储身份仍是 E；该实现允许 E 自身进入执行入口。
- E 的一次批量调用完成有限授权 Router、换入 mStocks 与买 Meme。E 向 USDC/Router 发起 CALL 时，直接调用者是 E，所得回 E。本例由 E 支付原生 gas，需另有费用余额。
- 图旁只提示另一条可选路径：该实现也提供 validateUserOp，可让 E 接入 4337。具体 Agent 限权和代付放到下一页，不让两套提交路径挤在同一主流程中。

这样先独立回答委托给什么、怎样获得钱包执行能力、为何无需把资金转入 W。E/W 仅是符号，不填写未经核验的部署地址。

**上屏文案**

标题：7702 示例：给原地址装上钱包逻辑

原 EOA E 持有 100 USDC；委托目标 W 是 Simple7702Account。

Alice 签 E → W 的代码委托。随后 E 调用自己的 executeBatch，一笔完成授权路由、兑换与买入。

EVM 读取 W 代码，但 address(this) = E，资产仍归 E。

本例 E 自付 Gas。该实现还提供 validateUserOp，可供 E 接入 4337。

**讲述补充（不上屏）**

W 是专门支持 7702 上下文的参考钱包，不是任意 DEX 或 Router。代码可复用，但各账户的余额和存储不合并；ERC-20 余额仍记为 Token 合约中的 balanceOf(E)，不会移到 W。

此例的普通交易由 E 的原密钥签署，自调用由钱包的 _requireForExecute 放行，不会额外经过 validateUserOp。另走 4337 时，EntryPoint 才调用 E.validateUserOp，Simple7702Account 以恢复出的签名者是否等于 address(this)=E 进行验证。两条路径都不自带 Session 与累计限额，下一页再讲这种扩展。

委托持续存在，后续业务失败不会自动清除已处理的委托。原 EOA 密钥仍需保护，设置委托不等于把根权限交给 Agent。

### P29A｜Agent 的权限和额度，怎样在委托钱包中实现

**本页要讲什么**

在委托机制上增加受限 Agent：Owner 选择有策略能力的钱包实现并安装授权；Agent 每次签操作，钱包验证身份与静态权限，再在执行时检查累计预算。7702 让原地址执行代码，具体的 AA 策略由这份代码实现。

**对应原文**

本轮用户对“如何实现 AA 逻辑”的要求；[2.2.1 账户模型](../references/chain-infra.mdx)、[3.1 目前已支持的 AA 方案综述](../references/chain-infra.mdx)，以及 [示例扩展核对 §1.2](../references/aa-comparison-and-evm-compatibility.md)。W_policy 是教学策略钱包，不是 Simple7702Account 已有功能或已部署地址。

**需要的图文及原因**

沿用 E→W 的代码关系，将实现替换为明确标注的 W_policy 教学扩展。Owner 预先配置 K：指定 Router、必要的代币授权、资产、累计 100 USDC 预算、有效期和收款人 E。将 Owner 授权与 Agent 每次签名分开。

执行路径为 K 签 UserOperation（sender=E），EntryPoint 调用 E 的验证入口，钱包检查 K 与全部调用，再由受限执行入口按实际状态核对并占用预算。Sponsor 的 Gas 预算单列；不能只是允许 K 验签通过，却仍可调用任意 executeBatch。

**上屏文案**

标题：Agent 的权限和额度，怎样在委托钱包中实现

教学扩展 W_policy：在 E 的账户上下文执行 Session 策略。

Owner 授权 K：限定交易目标、代币授权、100 USDC 总额度、期限与收款人 E。

K 签 UserOperation（sender=E） → 钱包验签和限权 → 执行时检查预算 → 交易并记账。

配合 4337，Paymaster 可另行代付 Gas。

这些策略需钱包实现；最小 Simple7702Account 并不自带。

**讲述补充（不上屏）**

配置、验证、执行三个环节要一起成立。验证器检查签名、期限与完整动作；执行入口用链上实际状态更新共享预算，避免多任务超额。Router 内部的接收人、资产和授权扩张也要检查，不能只匹配 Router 地址。

策略状态需要安全的初始化和存储布局。Owner 可撤销 Session，Agent 不获得原 EOA 根密钥。Paymaster 管费用条件，不替钱包检查交易权限；投入本金、原生手续费与可选代币报销分别记账。


### P30｜4337 与 7702 的优势来自不同层次

**本页要讲什么**

从技术结构比较两者，而不是用“都有代付和批处理”得出等价结论。4337 提供一套标准化的操作提交、验证与费用路径；7702 为原 EOA 增加代码执行入口，降低保留旧地址的迁移负担。二者可以组合，不能简单评选谁全面替代谁。

**对应原文**

[3.1 目前已支持的 AA 方案综述](../references/chain-infra.mdx)；[ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)、[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)，以及 [技术对比依据](../references/aa-comparison-and-evm-compatibility.md) §1。

**需要的图文及原因**

用同维度比较表，保留抽象层、认证、付款、nonce、迁移与运维等差异。上屏突出最影响产品的四项；详细限制由下一页接续。图中的 4337 与 7702 可以汇合到一个组合路径，不画成只能二选一的路线。

**上屏文案**

标题：4337 与 7702 的优势来自不同层次

| 维度 | 4337 | 7702 |
| :-- | :-- | :-- |
| 改动位置 | 合约与提交服务 | EOA 代码委托的协议能力 |
| 主要收益 | 标准操作、验证、代付路径 | 保留原地址与资产 |
| 操作认证 | 智能账户自定义验证 | 委托代码校验操作；设委托仍需原密钥 |
| nonce | EntryPoint 分通道 | 本体不改变原生序号模型 |

可以组合：7702 保留地址，4337 承载提交与代付。

**讲述补充（不上屏）**

4337 不要求自身的共识变更，但需要可用的账户、EntryPoint、Bundler、钱包 SDK 和费用服务。7702 是协议升级能力，不能称为零改链方案；它单独没有完整的 Paymaster 或受限 Actor 标准。委托代码可以实现独立操作序号或结合 4337，因此不能声称所有 Agent 操作都必然被该 EOA 的原生标量 nonce 串行阻塞。

对于更换签名算法，4337 账户可为 UserOperation 定义自定义验证；7702 后续操作也可由委托代码检查不同凭证，但最初设置委托仍依赖原 EOA 授权。这是操作认证与账户根控制权的区别。

### P30A｜看完整交易路径，才能看清成本与限制

**本页要讲什么**

进一步比较工程代价。4337 的提交路径需要模拟、费用预留、打包与入口执行；7702 保留地址，但把委托实现、初始化和旧密钥路径变成重要边界。不能把“少一个 Bundler”直接等同于更安全、更便宜，组合使用也不会自动消除原有成本。

**对应原文**

[3.1 目前已支持的 AA 方案综述](../references/chain-infra.mdx)、[ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) 的 Bundler 行为及 Paymasters、[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) 的初始化与持久委托说明。

**需要的图文及原因**

需要两条完整路径的负担说明：4337 标出模拟、打包、EntryPoint 验证与结算；7702 标出委托设置、受限执行与代付/中继。再给出各自易误判的失败结果：bundle 成功不等于每笔 UserOperation 成功，业务失败不等于委托未生效。使用文字说明，不添加未经测试的费用柱状图。

**上屏文案**

标题：看完整交易路径，才能看清成本与限制

- 4337：多出模拟、打包与 EntryPoint 包装；要维护可用的 Bundler 和代付预留。
- 7702：保留地址，但委托代码、初始化、存储升级和原密钥路径都要保护。
- 两者都能支持 Agent；限权、预算、撤销与失败恢复仍由完整实现负责。

比较同一笔成功业务的总成本与可靠性，不能只比较交易格式。

**讲述补充（不上屏）**

Bundler 影响纳入和服务可用性，不因其存在就获得任意支配用户资产的权限。4337 也不要求固定的长时间攒批，不能无测量断言其一定高延迟。7702 直接调用可能省去一部分 4337 包装，但如果需要同等代付、验证与恢复功能，相应代码和服务仍需存在。

4337 要读 UserOperation 与业务结果，不能只看外层交易 status。7702 授权处理与业务回滚的边界不同，已写入的委托可能在业务失败后继续存在。费用和首次设置开销按实际链与实现测试，不从规范常量推断总体节省比例。

## 3.2 从协议层需求进入 Native AA

### P30B｜Native AA 要减少每笔开销和打包等待

**本页要讲什么**

用高频交易的实际成本解释原生化动机。4337 在业务操作之外增加包装和结算路径，Bundler 又参与模拟、排队与提交决策。Native AA 将这些职责纳入协议，目标是减少可消除的成本与中间环节。对 Agent 连续打新或链上挂撤单，重复支出与尾部延迟都值得优化。

**对应原文**

[3.2 评估 Native AA 的增量价值](../references/chain-infra.mdx)、2.2.1 账户模型；[RIP-7560 Motivation](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7560.md)、[EF 2026 协议优先级](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)；Bundler 实际组包参数、固定源码与比较方法见 [补充依据 §2](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

需要两条结构路径：4337 将 Bundler 的模拟/组包与链的纳入分开，Native AA 则将账户交易直接接入链的验证与纳入流程。用下方两行流程作图中标签，再以一条短句解释高频操作的累积成本。不按未测量的比例缩短时间轴或绘制费用柱状图。

**上屏文案**

标题：Native AA 要减少每笔开销和打包等待

高频打新、链上挂撤单，会反复承担固定开销和提交等待。

4337：操作 → Bundler 模拟、排队、组包 → 外层交易 / EntryPoint → 业务执行。

Native AA：账户交易 → 链直接验证、纳入、执行。

收益目标：减少合约包装与中继环节，降低每次操作的总费和纳入尾延迟。

需实测；原生交易仍会排队，也仍有验证成本。

**讲述补充（不上屏）**

RIP-7560 曾用基础 UserOperation 约 42k 额外 Gas、基础交易约 21k 的口径说明开销；它不是 Mantle 数据，不能据此承诺降费比例。原生化改变部分调度与记账的实现位置，不免除验签、付款或重放检查。费用应包括首次设置、后续操作、失败、数据发布和服务收费。

固定版本 Alto 的自动组包间隔默认配置为 100–1000 毫秒，但这是调度参数，不是所有 4337 请求必须等待的时间或入块保证。Bundler 的筛选、排队、重试和链上拥堵共同影响纳入；4337 并不强制长时间攒批。原生化减少一道外层包装依赖，仍需良好的节点准入、RPC、排序器和费用服务。

尾延迟指最慢一部分请求的等待，用 P95/P99 与超时率观察。混合 Perps 的离线报价不一定逐笔走账户交易；只有实际经过这条路径的操作，才能归因于 AA 优化。

### P30C｜暂不锁定 Native AA，先确认 EVM 兼容边界

**本页要讲什么**

准确表达暂未选择某个原生方案的顾虑：新增账户与交易语义，是否会破坏 Mantle 已有 EVM 应用和接入体系。将兼容性拆成可验证的三层，而非用 Aztec/Starknet 的非 EVM 架构推断原生 AA 必然丧失兼容。

**对应原文**

本轮用户明确的选型顾虑；[2.3 保留两条技术主线，但避免过度简化](../references/chain-infra.mdx)、3.2；[补充依据 §3、§5](../references/aa-comparison-and-evm-compatibility.md)。8130 与 8141 的协议改动分别见 [8130](https://eips.ethereum.org/EIPS/eip-8130)、[8141](https://eips.ethereum.org/EIPS/eip-8141)。

**需要的图文及原因**

需要三层兼容检查图：合约行为、应用接入、链的验证。使用下表作完整图中标签。Aztec/Starknet 不再上屏：它们不是从 EVM 添加 AA 后失去兼容的案例，无法直接支持当前的选型顾虑。

**上屏文案**

标题：暂不锁定 Native AA，先确认 EVM 兼容边界

尚未选定方案，主要顾虑是新增交易语义对现有 EVM 生态的影响。

| 检查层次 | 必须确认什么 |
| :-- | :-- |
| 合约行为 | 调用身份、回滚、瞬态存储是否改变 |
| 应用接入 | 钱包、RPC、签名与回执怎样适配 |
| 链的验证 | 客户端与证明系统怎样支持新规则 |

原生 AA 可以保留 EVM；能用 Solidity，并不等于兼容工作已完成。

**讲述补充（不上屏）**

Aztec 与 Starknet 采用非 EVM 执行栈，这是完整架构选择，不能证明原生 AA 必须付出相同代价。背景仍保存在研究材料中。

8130 声明不需要 EVM 改动，但仍新增交易类型和节点规则；8141 则引入 Frame 上下文、批准与回滚语义。尤其跨 Frame 的瞬态存储边界要核验，不能把依赖该状态的 v4 unlock/settle 随意拆开。既有应用兼容与完全不改客户端是两件事。

### P31｜RIP-7560 将 4337 的调度职责内化到协议

**本页要讲什么**

先讲两者相同的 Account/Paymaster 分工，再对照不同的提交对象与调度位置。4337 由 Bundler 发外层交易、EntryPoint 合约组织操作；7560 让账户请求成为原生交易，由协议安排验证、执行和结算。内化的是职责，钱包代码和付款检查仍然存在。

**对应原文**

[3.1–3.2](../references/chain-infra.mdx)；[ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)、[RIP-7560](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7560.md) 的 Motivation、Validation、Execution 与费用处理；[对应关系 §4](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

用同一条生命周期横跨两条泳道。上方是 Bundler/EntryPoint 调度，下方是原生交易/协议调度，对齐账户验证、付款验证、业务执行与后处理。用相同的账户和 Paymaster 图标强调已有角色可以适配复用，不把迁移画成无需修改钱包接口。

**上屏文案**

标题：RIP-7560 将 4337 的调度职责内化到协议

| 职责 | 4337 | 7560 |
| :-- | :-- | :-- |
| 链接收什么 | Bundler 的外层交易 | 账户的原生 AA 交易 |
| 谁组织流程 | EntryPoint 合约 | 协议规则 |
| 谁结算费用 | EntryPoint | 协议 |

共同分工：账户验证 → 可选代付验证 → 业务执行 → 后处理。

7560 保留账户 / Paymaster 角色；接口仍需适配，多通道需 RIP-7712。

**讲述补充（不上屏）**

生命周期还含可选部署；无 Paymaster 时账户自付。4337 中 Bundler 先付外层费用，再由 EntryPoint 结算；7560 由协议检查、预扣与退还。账户和 Paymaster 仍通过代码及明确回调接受交易，不是把 EntryPoint.sol 原样塞入客户端，也不是取消验证。

验证失败使原生交易无效；业务失败仍会付费。7560 当前草案中 Paymaster 后处理失败还可回滚相应业务，因此后处理不是无关日志。账户、服务、准入与 RPC 均需适配；7560/7712 仍是草案，不能把协议化直接当成已测得的降费与延迟改善。

### P31A｜8130 先分清账户、操作者和授权规则

**本页要讲什么**

8130 的起点是把持有资产的账户与操作它的凭证分开。Authenticator 识别是谁签名，Keystore 提供该操作者在该账户上的授权；协议检查基础 scope 和 Manager 入口，业务 Policy 再检查具体动作与预算。这样能理解“原生身份标准化”的范围。

**对应原文**

[8130 Deep Dive](../references/aa/01-eip-8130-deep-dive.md) §3、§5；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §4、§5.1；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

需要角色关系图：Owner 配置账户 U 的 Keystore，Agent K 通过认证器得到 actorId，协议读取权限并把受限调用导向 Manager，Manager 检查业务 Policy 后操作产品。用 U 作为理解 Policy 的账户入口，保持后续资产调用由 U 发出。所有英文名都有一句中文职责，不画底层存储槽或完整 scope 位域。

**上屏文案**

标题：8130 先分清账户、操作者和授权规则

| 角色 | 负责什么 |
| :-- | :-- |
| Account | 持有资产、保持账户身份 |
| Actor | 被授权的 Owner 或 Agent |
| Authenticator | 验签，识别操作者 |
| Keystore | 保存权限、期限与配置 |
| Manager / Policy | 检查实际操作与预算 |

协议约束受限调用的入口，账户或产品检查入口内能做什么。

**讲述补充（不上屏）**

Actor 是操作身份，不是独立余额或保证金子账户。POLICY-only 授权限制顶层 call 的目标为 Manager；它不会自动检查 Router 内部接收人、资产、累计额度和风险，也不会自动赋予外部 Manager 操作 U 资产的权力。示例令 Manager 为 U 的受限执行入口，不代表可以仅凭 self-call 放行。

scope 是授权能力的组合。给受限 Actor 加上 OPERATOR 会扩大权限，不能把这种改变当作无害兼容修复。认证器负责身份，业务 Policy 负责动作，二者不混为一层。

### P31B｜8130 的一笔交易：Agent 操作，Sponsor 付费

**本页要讲什么**

在角色图基础上，用同一个受限买入场景说明原生交易如何运行。Owner 预先授权后，Agent 签署本次操作，Sponsor 独立认可付款，节点处理认证、nonce 和费用，再由 Manager 检查业务并执行。用户不必为每次交易重新签名，但仍有明确的权限与预算检查。

**对应原文**

[8130 Deep Dive](../references/aa/01-eip-8130-deep-dive.md) §2、§4、§5、§10；[原纲 3.2](../references/chain-infra.mdx)。本页将材料的教学买入改为经路由兑换 mStocks 后买 Meme，不宣称这是标准 ABI。

**需要的图文及原因**

用 Owner 授权与后续交易两段流程。前段只做一次授权配置；后段画 Agent 的操作签名、Sponsor 的付款签名、节点准入与原生处理、Manager 检查、路由业务和结果反馈。资金路径保留 U 账户经 USDC/mStocks/Meme 兑换，不能把“签名主体 K”当成资产所有者。

**上屏文案**

标题：8130 的一笔交易：Agent 操作，Sponsor 付费

Owner 先授权 Agent，并设定 Policy。

Agent 签操作 + Sponsor 签付款 → 节点核对身份、期限、nonce 与费用 → 限制调用到 Manager → 检查预算并执行兑换、买入。

U 始终是用户账户，K 是操作者，P 是付款方。

交易认证、业务权限与实际成交，是三个不同结果。

**讲述补充（不上屏）**

节点按真实入块状态重新验证，不能把准入模拟当作已执行。sender 与 payer 的认证有独立签名域，改费用、付款方或操作时需相应更新签名。payer 可以是外部 Sponsor，也可以是同账户的独立费用密钥；付款权限不自动受交易预算约束。

Manager 门禁位于执行阶段。越过该入口或业务 Policy 拒绝操作，可能成为有效入块交易中的执行失败，nonce 与费用不因此恢复。认证器范围、Sponsor 准入和费用预算仍需配合实现；L2 profile 收窄认证热路径不等于所有自定义算法都能直接原生使用。

### P31C｜8130 用独立请求与权限生命周期支持持续自动化

**本页要讲什么**

解释自动化持续运行的两组规则：交易怎样避免无关序号阻塞，Agent 权限怎样失效和撤销。多通道提供请求组织方式，nonce-free 以短有效期和共识去重处理无序请求；这些不能直接当作 Agent 资金隔离或完整撤单机制。

**对应原文**

[8130 Deep Dive](../references/aa/01-eip-8130-deep-dive.md) §6–7；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §5.3、§5.5；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

图上分开两条有序 key 通道与一组短期独立请求，再加 Owner 撤权连接 Keystore 的路径。将授权配置版本和交易 nonce 分开，指出“撤销当前 Actor”与“退休旧授权材料”是不同动作。不要把两条 nonce 画成两个自动隔离的资金账户。

**上屏文案**

标题：8130 用独立请求与权限生命周期支持持续自动化

- 2D nonce：不同 key 各自编号，减少无关序号等待。
- nonce-free：短有效期配合协议去重，不依赖有序通道。
- 撤权：撤销当前 Actor，并处理仍有效的旧授权材料。

通道不自动归某个 Agent 独占，余额与预算也不会随之隔离。

账户锁定有利于稳定准入，但会限制即时配置变更。

**讲述补充（不上屏）**

有序通道包含各自 sequence；赋予 NONCE scope 不等于只允许该 Actor 使用某一指定 key。共享账户下的通道干扰需另设约束。nonce-free 仍有所有节点一致维护的有界重放记录，满载或有效期不符可被拒绝；换 Sponsor 或有效期还可能改变 replay_id，业务去重仍需 operationId。

只 RevokeActor 可能无法退休之前签出的 JIT 配置材料；本地 epoch 用于让相应旧材料失效，但不自动删除现有 Actor、取消链下订单或关闭仓位。Lock 可减少权限变动，却影响撤权速度；材料指出部分锁定管理入口仍需固定规范版本测试。

### P31D｜8130 的 Phase 决定哪些动作一起回滚

**本页要讲什么**

用报销与买入说明 8130 的原子边界。一个 Phase 内的调用共同成功或失败，先前成功的 Phase 可以保留，当前 Phase 失败则后面的 Phase 跳过。用户看到交易失败时，不能假设授权、费用和所有前序动作都未发生。

**对应原文**

[8130 Deep Dive](../references/aa/01-eip-8130-deep-dive.md) §8–9；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §5.7、§8.4；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

用三个 Phase 的示例：Phase 0 报销费用，Phase 1 在同一组中授权路由并完成兑换/买入，Phase 2 后续业务。沿失败分支注明 Phase 1 失败时各自结果。明确报销只是可选设计，业务预算需扣除或另留报销资产，不让例子重复花费同一份资金。

**上屏文案**

标题：8130 的 Phase 决定哪些动作一起回滚

Phase 是一组共同回滚的调用。

- Phase 0：报销。
- Phase 1：授权路由 + 兑换并买入。
- Phase 2：后续动作。

若 Phase 1 失败：本组回滚，已成功的报销保留，Phase 2 跳过。

nonce 与已发生费用仍保留。交易失败，不等于整笔交易没有留下效果。

**讲述补充（不上屏）**

若 Phase 0 本身失败，后续 Phase 不执行，但 Sponsor 仍可能承担失败尝试的费用。报销能被独立保留不等于必定报销成功；代币返回 false 也必须由业务实现检查。账户配置变更可能在业务失败后仍已生效。

对 Perps，撤单和新单在同一 Phase 时，新单失败会恢复组前状态；放在不同 Phase 时，成功撤单可保留。两种目标不同，应由产品选择。多 Phase 仍不是同步成交的保证，具体业务接口必须能够实际完成所需动作。

### P31E｜8141 用 Frame 编排验证、付款与执行

**本页要讲什么**

8141 从交易结构出发，把一笔交易拆成多个有明确模式的 Frame。它不内置 8130 那样统一的 Actor 表，而是让账户代码通过验证帧决定是否批准后续动作。明确模式与批准状态，才能看懂下一页的 Agent 代付流程。

**对应原文**

[8141 Deep Dive](../references/aa/02-eip-8141-deep-dive.md) §1–2、§4–6、§12；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

用三类 Frame 的职责表，加一条从未批准到执行/付款已批准的状态关系。不要把 APPROVE 画成 USDC.approve，也不把它当成普通业务调用。表后说明无代码 EOA 的默认处理入口，让听众理解基本批处理可以与现有账户共存。

**上屏文案**

标题：8141 用 Frame 编排验证、付款与执行

Frame 是一笔交易中的一个调用步骤。

| 模式 | 做什么 |
| :-- | :-- |
| VERIFY | 检查授权，显式批准执行或付款 |
| SENDER | 以用户账户身份执行顶层业务调用 |
| DEFAULT | 部署、辅助动作或后处理 |

APPROVE 改变交易的批准状态，不是 ERC-20 代币授权。

无代码 EOA 可用默认验证路径；Session 权限仍需账户实现。

**讲述补充（不上屏）**

VERIFY 是受限制的只读验证路径，APPROVE 是允许改变交易批准状态的特例；累计预算的写入通常放在执行阶段。执行批准使后续 SENDER Frames 可使用 sender 身份，付款批准确定 payer；未满足批准条件不能当成有效业务交易。

当前草案还包含协议级 secp256k1/P-256 签名材料预验证，不是所有签名都交给任意 EVM 代码。支持 P-256 也不等于自动完成 WebAuthn/Passkey 的完整认证。DEFAULT/VERIFY 的协议入口上下文与 SENDER 不同，下游普通 CALL 的 msg.sender 仍按调用链变化。

### P31F｜8141 的一笔交易：先批准整份计划，再执行

**本页要讲什么**

把 Frame 落到受限 Agent 买入的具体场景。账户验证器不仅确认谁签名，还要检查整个受授权计划；Sponsor 独立批准付款，之后业务步骤才以用户身份执行。灵活编排的价值与完整计划验证的责任必须一起解释。

**对应原文**

[8141 Deep Dive](../references/aa/02-eip-8141-deep-dive.md) §3、§6–7；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §5.2、§7；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

画出 U 账户验证帧、P 付款验证帧、受限业务帧和可选后处理。沿用 U/K/P 角色，标明批准付款时消费 nonce 并预扣费用。验证框覆盖全部后续 SENDER 动作，不只连接某一笔 buy。常规签名和有效期检查作为前置说明，不能让图形暗示它们可以被省略。

**上屏文案**

标题：8141 的一笔交易：先批准整份计划，再执行

1. VERIFY U：核对 Agent 与全部受授权动作，批准执行。
2. VERIFY P：确认付款；消费 nonce，预扣费用。
3. SENDER：在账户策略内报销、授权路由、兑换并买入。
4. DEFAULT：按需做后处理。

批准的是后续整份执行计划，不能只检查一笔买入就放行其他动作。

**讲述补充（不上屏）**

Owner 预先配置 Session 策略，Agent 签署本次计划；Agent 签了完整交易也不能替代 Owner 的权限限制。验证器需检查目标、内部调用、资产、接收人、value、费用与期限，账户执行入口仍需动态预算检查。

VERIFY 验证失败使交易无效；验证通过后的业务失败按分组规则处理。公共 mempool 还限制验证前缀、可读状态和 Sponsor 接纳，表达得出的 Frame 计划不等于所有节点都会传播。示例可选报销步骤必须处理 ERC-20 的真实支付结果，并在总预算中预留。

### P31G｜8141 原子组失败，组外动作仍可能继续

**本页要讲什么**

讲清与 8130 Phase 最容易混淆的地方：8141 的共同回滚作用于指定连续组，组外后续步骤可以继续。报销、授权和买入怎样分组，决定失败后保留什么；单个 Frame 的执行状态也不能直接当成最终资产状态。

**对应原文**

[8141 Deep Dive](../references/aa/02-eip-8141-deep-dive.md) §3.4、§8；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §5.7、§8.4；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

需要一条已完成验证/付款的 Frame 序列，依次为组外报销、组内授权路由与买入、组外后处理。用明确的业务原子组边界表示买入失败时回滚范围，并将前面成功报销和后面可继续处理分别保留。可以和 P31D 的结果做口头对照，不需要重复完整角色图。

**上屏文案**

标题：8141 原子组失败，组外动作仍可能继续

示例：报销 → [授权路由 + 买入] → 后处理。

若买入失败：组内授权回滚，已成功报销保留，组外后处理可继续；nonce 与费用不退回未提交状态。

与 8130 不同：业务组失败后，组外后续步骤不会默认全部停止。

回执要结合原子组与最终业务状态，不能只看某个 Frame 显示成功。

**讲述补充（不上屏）**

如果报销失败却仍不允许买入，应显式检查依赖或放进共同受控调用；只排列先后不够。组内较早 Frame 的 status 可以记录执行成功，但后续组失败后其状态和日志可能已回滚，SDK 必须重建分组结果。

当前材料规定 Frame 之间会清理瞬态存储（transient storage）。依赖 EIP-1153 的一次完整 v4 unlock/callback/settle 流程应放在兼容的同一 Frame 内执行并测试，不能把未结账务随意留到下一个 Frame。原子组也不能把异步 Perps 请求包装成已经同步成交。

### P31H｜8250 怎样在 8141 的框架上加入 keyed nonce

**本页要讲什么**

8141 提供验证、付款和执行的通用结构，具体 AA 功能可在其上实现。Session 限权可由账户验证代码实现；keyed nonce 改变协议重放规则，需要 8250 这样的配套协议扩展。本页用交易头、状态和批准时点三个具体变化，解释扩展如何接入已有框架。

**对应原文**

[原纲 3.2](../references/chain-infra.mdx)；[8141 材料](../references/aa/02-eip-8141-deep-dive.md) §9–11；[EIP-8250](https://eips.ethereum.org/EIPS/eip-8250) 的 Transaction payload、Nonce state、Stateful validity、Nonce consumption；核对见 [补充依据 §8](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

保留 8141 的 Frame 流程，用三个标注说明：交易头把 nonce 换成 key/序号；协议用 NONCE_MANAGER 存储非零 key 状态；付款批准时消耗选定 key。另画同一 sender 的两个独立域，展示一边消费后另一边仍有效。图中标签仅用下方文案，不把 nonce 状态画成由 Agent 普通调用系统合约随意修改。

**上屏文案**

标题：8250 怎样在 8141 的框架上加入 keyed nonce

8141 提供通用框架，Session / 限权可由验证代码实现；8250 再扩展协议重放规则。

① 交易头：nonce → nonce_keys + nonce_seq。
② 非零 key：序号存入 NONCE_MANAGER。
③ 付款批准：一次消耗所选 key，Frame 结构不变。

同一 sender：key 11 从 0 → 1，key 22 仍为 0。

独立重放域不等于公共池已支持多笔并发。

**讲述补充（不上屏）**

所有选定 key 必须共享同一个 nonce_seq，并在任何 Frame 执行前逐个匹配链上序号；[0] 别名到旧账户 nonce，非零 key 由协议写入 NONCE_MANAGER。签名覆盖新字段，TXPARAM 扩展供验证器核对授权域。该系统合约拒绝普通调用，故这不是单独部署一个钱包插件即可完成的升级。

消耗发生在唯一成功的付款批准 APPROVE 转换中，与首次 key 的状态费用及付款批准原子提交；所在 Frame 失败会回滚该转换。该 Frame 成功后，后续业务或原子组失败不会恢复已消费 nonce；整笔交易无效时则丢弃所有效果。

当前 8250 仍保留公共 mempool 每个 sender 一笔待处理 Frame 交易的限制；未来 keyed-aware 接纳需另外实现。共享预算、余额与市场状态也仍可能冲突。首次 key 有状态增长费用，不等于 8130 的有界短期 nonce-free。8141 的 execution/state gas 则是计量维度，不是 nonce 通道或应用费用隔离。

### P31I｜8130 统一账户身份，8141 统一交易编排

**本页要讲什么**

在实际流程讲完后再比较两条重点路线。8130 的标准化重点是身份、基础授权和 nonce 模型；8141 的重点是可编排验证、付款和执行分组，并用 8250 补多通道。选择应对齐账户模型与产品失败语义，而不是用功能勾选总分决定。

**对应原文**

[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §1、§4–5、§13；[8130 材料](../references/aa/01-eip-8130-deep-dive.md) §5–8；[8141 材料](../references/aa/02-eip-8141-deep-dive.md) §6、§8、§11；[原纲 3.2](../references/chain-infra.mdx)。

**需要的图文及原因**

用身份、权限入口、nonce 与失败行为四个维度的比较表收束。每个格子只写前面已解释的核心差异；表下保留双方共同需要的 Policy、Sponsor 和恢复系统。不要遗漏 8250 后再得出 8130 独有多通道的结论。

**上屏文案**

标题：8130 统一账户身份，8141 统一交易编排

| 维度 | 8130 | 8141 + 8250 |
| :-- | :-- | :-- |
| 身份 | Actor 与认证器原生标准化 | 账户定义授权，Frames 承载验证 |
| 限权 | 受限 Actor 的 Manager 门禁 | 批准前检查全部受授权动作 |
| nonce | 内置 2D 与短期 nonce-free | 8250 提供 keyed nonce |
| 失败 | 当前 Phase 失败，后组跳过 | 指定组回滚，组外可继续 |

双方都需补齐业务 Policy、代付预算、订单权限与恢复。

**讲述补充（不上屏）**

8130 的通道不天然属于某个 Actor，Manager 也不自动获得安全的资产操作权；8141 的验证器要防止未检查的后续 SENDER 动作，公共准入还限制验证前缀。两案的复杂度落在不同位置，不能仅由这张表推导一个已经更快或更安全。

### P31J｜8141 的通用框架，也服务隐私与 L1 长期演进

**本页要讲什么**

说明 8141 不专门定义某套 AA 业务功能，它为不同认证和付款方式提供交易结构。原始动机明确包含后量子迁移、换钥与替代付费，现有证据不足以称其主要为隐私设计；8250 则直接以隐私协议共享 sender 为重要动机。将这些方向与 L1 的 CROPS 原则联系，再对照 8130 的 L2 profile，给出有依据的定位判断。

**对应原文**

本轮关于隐私动机与 L1/L2 定位的研究要求；[EIP-8141 Motivation](https://eips.ethereum.org/EIPS/eip-8141)、[EIP-8250 Motivation](https://eips.ethereum.org/EIPS/eip-8250)、[8130 Adoption Profiles](https://eips.ethereum.org/EIPS/eip-8130)、[EF Mandate](https://ethereum.org/foundation/mandate/) 的 CROPS；原文与判断分开记录于 [补充依据 §7](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

用上下两层：上层从通用验证/付款框架连到后量子与隐私用例，并以 CROPS 为方向说明；下层并列两案设计重心。隐私用例的证明、付款方与共享 sender 不画成 8141 自动附赠的组件。四个 CROPS 词不用全绿勾选，避免暗示单项提案保证全部属性。

**上屏文案**

标题：8141 的通用框架，也服务隐私与 L1 长期演进

原始动机：可编程验证与付费、换钥、后量子迁移。

隐私用例：证明验证 + 独立付款；8250 为共享 sender 提供独立重放域。

贴合 L1 的 CROPS：抗审查、开源与自由、隐私、安全；不代表自动实现。

8141 侧重通用交易编排；8130 更直接标准化 AA，其 L2 profile 更贴近本次需求，但也有 L1 profile。

**讲述补充（不上屏）**

“更贴近”是面向 Mantle 的设计判断，不是两案的排他部署范围。8130 L2 profile 采用 canonical-only 原生认证路径及本链共识计费表；L1 profile 也允许在规定 Gas 上限内运行其他认证器。两种 profile 都存在，不能再把 8130 简化为 L2 专用方案。

8141 的原始动机不能简化成隐私。其验证与付款结构可减少隐私应用预充值地址和专用中继依赖；仍需证明系统、匿名集合与符合准入限制的实现，不会自动隐藏公开调用、金额或网络元数据。8250 将 nullifier 派生域列为用例，公共池限制仍按 P31H 理解。

CROPS 是 EF 对 Ethereum 的总体原则，开放性、安全与抗审查还取决于实现和纳入机制；本页不把该原则冒充 8141 作者的原始设计宣言。

### P32｜近期交付：两个产品共用的 7702 AA 组件

**本页要讲什么**

将本章落到近期可交付范围：基于 7702 钱包实现、SDK 和费用服务，支持 Agent Meme Launchpad 与股票 Perps。明确功能来自实现与产品适配，7702 本身只提供原地址运行代码的基础；标准提交与代付可复用 4337。

**对应原文**

本轮用户要求的近期路线；[2.2.1 账户模型、3.1](../references/chain-infra.mdx)；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §7–8、§10–13；[交付清单 §9](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

需要一个共用组件框，连接上方两个产品，下方列出六项首批交付能力。钱包实现、SDK 和费用服务共同构成组件，不画成只激活 7702 就全部具备。每项能力给出可理解的具体对象，避免只有“安全、并发、Gasless”等抽象标签。

**上屏文案**

标题：近期交付：两个产品共用的 7702 AA 组件

Agent Meme Launchpad、股票 Perps 共用钱包实现、SDK 与费用服务。

- 限权撤权：Session、目标、资产、收款人与期限。
- 分层权限：交易与提现分离，子账户接口。
- 预算代付：本金、Gas 与报销分账，可接 4337。
- 批量执行：授权、兑换、买入及失败回滚。
- 多任务恢复：操作序号、去重、重试与回执。
- 产品适配：买卖退出、订单签名与撤单。

**讲述补充（不上屏）**

以链已支持 7702 为前提，优先评估可审计和复用的实现；最小 Simple7702Account 不能直接代表上述生产能力。近期方案使用钱包或 4337 操作序号，不冒充原生多通道。受限执行入口需在实际状态下检查预算，原 EOA 根密钥不能交给 Agent。

撤销 Session、取消已签订单、收回代币 allowance 和平仓是不同操作，须按产品规则衔接。AA 组件提供订单签名/撤权适配，但不替代 Perps 的仓位、保证金、撮合或风险引擎。交付验收包括正常交易、越权拒绝、超额拒绝、撤权生效和失败恢复。

### P32A｜长期选型：用相同产品任务验证原生收益

**本页要讲什么**

用近期组件建立真实基线，再在同等功能和安全约束下比较原生方案。长期目标是减少总费用和纳入等待，同时满足 EVM 兼容与可维护性；保留原先的工程验证要求，不因近期路线清晰而提前锁定生产提案。

**对应原文**

[3.2 评估 Native AA 的增量价值](../references/chain-infra.mdx) 的四项比较问题；[能力覆盖评估](../references/aa/03-mantle-aa-coverage-and-fit.md) §10–13；[补充依据 §2、§7–9](../references/aa-comparison-and-evm-compatibility.md)。

**需要的图文及原因**

将同一组受限交易任务输入现有组件与三条原生路线，再用四项验收结果比较。不能用完整策略钱包对比裸签名。下方明确 8130 L2 profile 为优先 PoC 候选而非生产选择，8141+8250 与 7560 保留对照。

**上屏文案**

标题：长期选型：用相同产品任务验证原生收益

基线：7702 / 4337，保持相同权限、预算和业务任务。

| 验收 | 比较什么 |
| :-- | :-- |
| 成本 | 成功、失败、数据与服务总费 |
| 纳入 | 提交到生效的 P95/P99、超时率 |
| 多任务 | 准入限制、撤权、去重与恢复 |
| 兼容 | EVM、工具、客户端与证明维护 |

优先验证 8130 L2 profile；以 8141+8250、7560 对照，再决定方案与升级时点。

**讲述补充（不上屏）**

时间拆成接收、模拟、提交排序器、入块与业务生效，不能只测 RPC 返回 hash。费用区分首次设置、稳态业务与失败，并保持相同安全约束与 Sponsor 策略。先优化 Bundler 基线，再判断原生化带来多少额外收益。

Vault 的 request/execute、混合模式的离线订单、链上订单簿不是相同负载。混合模式中 sender 可能是撮合者，用户订单签名与取消仍须另验；原生 nonce 不一定处于每次报价的关键路径。市场执行、价格与证明积压若是主要瓶颈，应由对应系统解决。

### C3｜近期用 7702 服务产品，长期选择原生 AA

**本页要讲什么**

给出阶段性结论和交付内容：近期不等待原生选型，通过 7702 组件让两个产品使用同一套受限账户能力；长期以真实成本、纳入改善和 EVM 兼容确定原生路线。功能验收延续到原生实现，不因换交易格式而丢掉权限与恢复。

**对应原文**

本轮用户明确的结论要求；[3.1–3.2](../references/chain-infra.mdx)，结合 P30B/P30C 的成本与兼容论据、P31–P31J 的机制与定位，以及 P32/P32A 的交付清单和验证方法。

**需要的图文及原因**

结论页用近期、长期两段路线，中间连接真实产品基线。近期列明两个产品和共用 AA 功能，长期列出选型目标与保留条件，不重复全部 EIP 流程，也不将 PoC 候选写成生产决定。

**上屏文案**

标题：近期用 7702 服务产品，长期选择原生 AA

近期：交付 Launchpad 与股票 Perps 共用的 AA 组件。

覆盖限权撤权、交易与提现分权、预算代付、批量执行、操作去重和失败恢复。

长期：选择符合 Mantle 路线的 Native AA，降低重复开销与纳入等待，并守住 EVM 兼容边界。

先用产品形成真实基线，再确定原生方案及升级时点。

**讲述补充（不上屏）**

7702 是实现这些组件的账户基础，不是自带全部功能的产品。近期成果包括钱包、SDK、服务及两产品适配；长期须维持同等安全与业务条件，实际测量收益，并承担客户端、工具与证明升级成本。当前没有锁定某项原生提案，也没有承诺未经测量的性能比例。

