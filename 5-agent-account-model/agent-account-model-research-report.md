# 将 Agents 作为一等公民的 Mantle 账户模型研究报告

> **报告定位**：Mantle v3 Roadmap · Agent 账户需求与现有 AA 方案研究。  
> **规范核查日期**：2026-09-20；引用的规范版本见文末来源。  
> **研究范围**：从 Meme Launchpad 与 Onchain Perps 的业务需求出发，解释 ERC-4337、EIP-7702、RIP-7560，以及 EIP-8141 / EIP-8130，并比较各方案能够满足哪些账户特性。本文不提出 Mantle 自有账户架构或实施路线图。  
> **证据边界**：产品需求来自仓库既有调研；协议机制以一手规范为准。草案能力、业务目标与链上已启用功能分别说明，本文没有执行性能基准测试或 Mantle 新交易类型的上线验证。

## 0. 背景信息：当交易主体从人转向 Agent

以太坊的传统 EOA 使用方式通常围绕人类的一次次交互展开：用户通过 MetaMask、硬件钱包等工具保管私钥，在屏幕前检查目标合约、转账金额和 Gas 费用，再签名提交交易。相对于持续运行的交易程序，人类操作频率较低，通常可以接受确认提示和失败后的人工重试。

这是一种常见使用模式，并非以太坊协议只允许人类或低频交易。套利 Bot、自动化做市与清算程序早已在 EVM 链上运行。这里要讨论的是：**当更多资金交由 Agent 持续操作，账户是否能明确表达“它可以做什么、使用多少资金、由谁付费，以及哪些操作不应互相等待”。** “Agent 成为链上最大消费者”是本文考察的业务情景，不作为已经得到统计验证的事实。

| 变化 | 人类交互中的常见安排 | Agent 场景中的账户要求 |
|---|---|---|
| 执行主体 | 用户检查并签署每笔交易 | 链下进程持续监听事件，自主签署受限操作 |
| 操作频率 | 低频、逐笔确认 | 多策略同时响应，可能出现高频突发请求 |
| 资本与策略 | 同一个人持有资产并决定交易 | 人类、DAO 或金库保留管理权，Agent 获得策略执行权 |
| 运行环境 | 钱包或硬件签名器 | 云端、容器、外部工具、AI Runtime 与依赖库构成更大的攻击面 |
| 异常处理 | 人工加速、撤销、补充 Gas | 需要可撤销授权、可预测的失败语义与持续运行能力 |

Meme Launchpad 的开盘买入与 Perps 的追加保证金，都可能对响应时间敏感。但应分开衡量事件发现、策略决策、签名、提交、预确认和最终结算。毫秒级的链下响应需求，并不意味着账户协议可以承诺毫秒级成交；LLM 参与策略制定，也不意味着每次交易必须等待一次模型推理。

## 1. 当前 EVM 账户模型存在的问题

以下三类问题主要针对**直接使用传统 EOA 根私钥发送普通交易**的模式。智能账户、代发交易以及 7702 委托可以改变其中一部分约束，不能将传统 EOA 的限制直接外推到所有 EVM 账户。

### 1.1 私钥安全与信任边界

EOA 地址取未压缩公钥坐标哈希的后 20 字节，即 `keccak256(pubKey)[12:]`，这里的 `pubKey` 不含格式前缀。普通 EOA 交易通过 secp256k1 签名恢复发送者；客户端还需要检查 nonce、余额、费用等有效性条件。它没有内建“这把钥匙只能调用某个池子、每天最多支出多少”的交易权限规则。

如果 Agent 持有根私钥，攻击者不必窃取密钥文件本身，也可能通过操纵签名请求获得资产控制权。风险包括云主机或容器被攻陷、Prompt Injection、工具调用越权、Skills 后门，以及 npm / pip 依赖投毒。把密钥放在 KMS 或 TEE 内能减少直接泄露，却仍需限制签名器接受的操作。

多签可以保护账户管理权。如果每笔交易都等待人类多签确认，协调过程通常不适合抢买和快速对冲；但**多签本身不必然带来固定的秒级延迟**，多签账户也可以预先安装受限执行模块。需要解决的是管理授权与日常交易之间的分工。

对 Agent 的授权至少要能表达目标、动作、资产、金额、有效期和撤销方式。只禁止 `transfer()` 仍不充分：无限 `approve()`、任意收款人、可升级 router、嵌套调用或恶意交易路径，都可能形成间接资产转移。Session Key 的作用是缩小授权范围，不能保证策略不亏损或账户绝不会失窃。

### 1.2 串行 nonce 与线头阻塞

普通 EOA 交易在执行位置必须满足 `tx.nonce == account.nonce`。有效交易被纳入并处理后，即使业务调用 revert，发送者 nonce 也会被消耗。问题发生在前序交易尚未入块时：同一发送者的后序 nonce 不能越过它执行。

```text
One EOA, one transaction nonce sequence

A: Meme buy       nonce=10   low fee / not yet included
                                  |
                                  v
B: Perps addMargin nonce=11  waits for nonce=10
   higher fee cannot remove this dependency

Another account's liquidation transaction has its own nonce sequence.
It may execute before B, exposing the position to liquidation risk.
```

比如一个 Agent 发出 `nonce=10` 的 Meme 买单后，又发出 `nonce=11` 的保证金追加请求。前者因费用或交易池调度未被纳入时，后者即使提高费用，也不能越过前者。若清算交易先发生，仓位可能被清算；这是一种风险路径，并不意味着每次阻塞都会造成全仓损失。

这里的 pending 不应解释为“EVM 合约内部锁竞争”。交易池决定是否纳入交易；合约滑点或额度检查失败通常表现为执行 revert，模拟服务也可能提前拒绝请求。两种情况都需要与尚未入块的 nonce 缺口区分。

常见处理办法有两类：

- **替换前序交易**：使用相同 nonce 和满足节点替换规则的费用重新提交。它需要处理传播、替换与回执竞态；正确替换本身不会自动制造 nonce gap，也不会回滚已经确认的其他交易。
- **多个 EOA 分担提交**：拆分发送者序列。直接把资产分放在多个钱包会增加资金调度成本；若这些地址只作为共享金库的受限执行者，则资产不一定需要等额分散，但授权与运维仍需管理。

评估 AA 时，要分别检查账户操作的 nonce、Bundler / Relayer 的外层 nonce，以及业务订单自己的序号。它们控制的依赖关系不同。

### 1.3 Gas Token 缺乏与费用支付角色绑定

普通 EOA 交易的发送者需要持有链原生代币支付网络费用，Mantle 上为 MNT。账户只有 USDT、USDC 或其他资产时，无法直接用这些资产支付普通交易的 Gas，因此新 Agent 的初始化和长时间运行都需要处理 Gas 补充。

代付把“谁执行操作”和“谁承担网络费用”拆开。Agent 可以不持有原生代币，由 Paymaster 或 Sponsor 支付；Sponsor 再向用户收取 ERC-20，或用业务收入补贴费用。**用户以 ERC-20 结算费用，不等于链的网络费用币种变成了 ERC-20。** 代付账户仍需要足够的原生代币及可用服务。

对于 Perps，还要区分钱包里的稳定币与已经锁入保证金合约的抵押品。后者不能被一个通用 Paymaster 任意扣走，必须经过业务协议允许的接口，并检查扣费后是否影响保证金安全。

### 1.4 次要能力与共同边界

替代签名算法和批量调用也有价值，但优先级低于受限授权、操作隔离与代付。传统 EOA 交易的签名算法固定，合约账户则可实现不同验证方法；具体计算成本取决于链上可用的预编译与实现。

普通 EOA 的顶层交易只有一个目标，但目标 router 可以继续调用多个合约，部分代币也支持 permit。不能断言所有 `approve + swap` 都必须等待两个区块。智能账户能更统一地提供批处理；原子批处理消除部分中间状态，却不会自动消除 MEV，也不能把跨链或异步成交变成原子结算。

## 2. Mantle 产品方向对账户的需求

仓库已有两个需要区分的 Meme 产品语境：Tape 研究包含股票 quote、Spend-Gate 等设定；后续 Agent Meme Launchpad 报告以 Agent 发币和交易为中心，采用默认 USDC quote。本文提取它们共同的账户需求，不把某个产品分支的计价资产、发行阈值或排序规则写成 Mantle 全链事实。[P1][P2]

Onchain Perps 的既有研究同时讨论 RFQ 与 Oracle-priced pool；不同结算模式决定哪些动作发生在链上、哪些仍由 keeper 或链下服务完成。账户能力不能替代撮合与清算系统。[P3]

### 2.1 Meme Launchpad 的业务场景

Agent 可能负责发币、开盘买入、卖出、做市和领取费用。人类或金库保留资金管理与授权撤销权，Agent 在策略范围内操作。

| 业务操作 | 账户需要提供的能力 | 能力边界 |
|---|---|---|
| 委托 Agent 发币或交易 | 限工厂、限 router、限资产与收款人，设置单笔和累计预算、有效期，可撤销 | 白名单与金额检查需要覆盖嵌套调用及授权路径 |
| ERC-20 授权后买入 | 将 `approve + buy + minOut` 检查置于同一原子调用组 | 授权仍发生；批处理不保证防夹，也不保证首块成交 |
| 多标的同时买入 | 不同任务不因无关的账户 nonce 缺口互相等待 | 仍竞争 Gas、账户余额和池子状态 |
| 新 Agent 启动 | 无 MNT 余额也能通过 Sponsor 发起允许的业务操作 | Sponsor 的余额、额度、反滥用检查与可用性仍是依赖 |
| 订单过时或策略停止 | 授权到期、交易截止时间、撤销与重试语义可识别 | 撤销需被执行；已成交操作无法靠撤销追回 |

用一个简化例子贯穿后文：Owner 授权 Agent 最多支出 100 USDC，通过指定 router 买入一个代币，且输出必须回到 Owner 账户。Agent 的交易包含 `approve(router, 100)` 与 `buy(..., minOut, recipient=account)`。这只是说明 AA 承载什么业务动作，金额和接口名都不是 Mantle 产品参数。

### 2.2 Onchain Perps 的业务场景

Perps 的账户要求围绕资产安全与仓位连续管理展开。RFQ 报价可能在链下频繁更新，Oracle-pool 的订单可能分为创建与 keeper 执行；高频报价不应直接等同于每个报价都要发送一笔链上交易。[P3]

| 业务操作 | 账户需要提供的能力 | 能力边界 |
|---|---|---|
| 追加保证金、减仓、取消未成交订单 | 与无关打新、开仓操作隔离的提交序列 | nonce 隔离不等于区块空间预留或清算前必达 |
| 现货与 Perps 双腿对冲 | 对同链、可同步完成的调用提供原子执行组 | keeper 延后执行、外部 RFQ 对冲或跨链腿不能仅靠账户原子化 |
| 做市与仓位守护由不同 Agent 执行 | 区分开仓、减仓、追加保证金与提现权限 | 守护权限不能因交易权限熔断而一并失效 |
| 累计亏损或杠杆超限 | 可承载风控检查，拒绝继续扩大风险 | 净值依赖预言机与业务状态；合约不会自行唤醒，也不能保证平仓有流动性 |
| 以稳定币承担费用 | 支持第三方支付网络费，并按规则向用户结算 | 从锁定保证金扣费还需 Perps 接口与健康度检查 |

独立 nonce 的目标是消除人为加入的顺序依赖。假设 Meme 买单和 `addMargin` 分别使用两个通道，买单的 nonce 缺口不应阻止补仓进入候选集合；补仓能否及时执行，仍取决于链负载、排序、价格更新和业务检查。

### 2.3 用于横向比较的 AA 特性

| 编号 | 需要比较的特性 | 主要场景 |
|---|---|---|
| A | 所有权与执行权分离，支持限权、到期、撤销 | Meme 与 Perps 的所有 Agent |
| B | 独立任务的 nonce / 重放保护不互相阻塞 | 多标的交易、Perps 自保 |
| C | Agent 无原生 Gas Token 时仍可交易 | 冷启动、持续交易、紧急补仓 |
| D | ERC-20 费用结算及保证金扣费的可组合性 | Launchpad 赞助、Perps 稳定币结算 |
| E | 明确的批处理与失败回滚边界 | 授权后买入、同步双腿交易 |
| F | 可承载支出、杠杆、净值等业务风控 | 防止失控交易扩大损失 |
| G | 低提交开销与可预测的验证工作 | 高频突发请求 |
| H | 明确回执、过期与失败付费语义 | Agent 重试、撤销、费用核算 |

另外单列两个所有方案都需要面对的条件：**账户抽象不保证交易优先级、抗 MEV 或亚秒级纳入；规范存在也不等于 Mantle 已启用。**

## 3. 现存 AA 方案综述与比较

ERC-4337 提供应用层执行入口；7702 让已有 EOA 地址执行委托代码；7560、8141 和 8130 则描述新的原生交易处理方式。它们并非五个互相排斥的钱包产品：7702 可与 4337 配合，8130 的账户合约也可在非原生支持链上通过 4337 使用。

```text
Where does the account capability live?

ERC-4337     UserOperation -> Bundler -> EntryPoint contract -> Account
EIP-7702     EOA address   -> persistent code delegation     -> Wallet logic
RIP-7560     Native tx     -> protocol-defined AA phases     -> Account/Paymaster
EIP-8141     Native tx     -> programmable frame sequence    -> Approval + calls
EIP-8130     Native tx     -> Keystore + actor authentication -> Call phases
```

本节以五份规范为主体。ERC-7579 是模块化账户接口，可用于组织 Validator、Executor、Hook 等能力；当前规范的验证流程依赖 4337，接入原生 AA 时仍需适配。它本身不定义新的原生交易类型，也不会自动赋予下面所有方案同一套 Session Policy。[S7]

### 3.1 应用层 AA：ERC-4337

#### 3.1.1 UserOperation 如何成为链上操作

ERC-4337 不要求更改链的共识规则。Agent 签署 `UserOperation`，Bundler 模拟验证并把它装入调用 `EntryPoint.handleOps()` 的普通交易。EntryPoint 调用账户的 `validateUserOp()`，可选地调用 Paymaster 验证，然后调用账户执行业务操作并结算费用。[S1]

```text
Owner -- installs limited session permission --> Smart Account
Agent -- signs UserOperation -----------------> Bundler
                                                  |
                                      simulate / select operations
                                                  |
                               ordinary tx: EntryPoint.handleOps(...)
                                                  v
                                  +-------------------------------+
                                  | Validation loop               |
                                  | Account.validateUserOp()      |
                                  | optional Paymaster validation |
                                  +---------------+---------------+
                                                  v
                                  +-------------------------------+
                                  | Execution loop                |
                                  | Account executes callData     |
                                  | optional Paymaster.postOp()   |
                                  +---------------+---------------+
                                                  v
                                     UserOperation result + fees
```

套用买入例子，UserOperation 的发送者是智能账户，签名来自受限 Agent，`callData` 请求账户完成 `approve + buy`。如果账户执行器把两次调用放在同一个全成或全退的调用中，买入失败会撤销这次授权。**单个 UserOperation 的业务原子性由账户执行代码决定；整个 bundle 不等于一组必须一起成功的用户交易。**

Paymaster 支付的是 UserOperation 对应的费用，外层普通交易仍由 Bundler 支付原生 Gas，再从 EntryPoint 的费用结算中获得补偿。网络里的 EntryPoint 有多个版本，账户、SDK 与 Bundler 必须对齐具体版本，不能只凭一个地址断言兼容。

#### 3.1.2 4337 的二维 Nonce：支持跨 Bundle 解耦，而非同 Bundle 并包

规范将 `uint256 nonce` 分成高 192 位 `key` 与低 64 位 `sequence`，由 EntryPoint 为每个 `(sender, key)` 维护递增序列，见 [S1] 的 Semi-abstracted Nonce Support。

这里必须严格区分**合约层 Nonce 机制**与**打包层（Bundler）规则**：

1. **同 Bundle 内默认不能包含同一普通用户的多笔 UserOp**：
   ERC-4337 的打包规则明确规定：*Bundler MUST exclude UserOperations that access any sender address of another UserOperation in the same bundle*。因为 `handleOps` 是“先批量验证、再批量执行（Two-loop）”架构，若同一个普通（未 Stake）账户的两笔操作放入同一个 Bundle，第二笔在验证时第一笔尚未执行，会引发严重的状态重叠与 DoS 风险。因此，标准 Bundler 在组装单个 Bundle 时，**同一个普通用户最多只会选入一笔 UserOp**。
2. **二维 Nonce 的真正价值是跨 Bundle / 跨区块的「消灭线头阻塞」**：
   在传统一维 Nonce 下，如果 Meme 买单（Nonce=10）因为费用低卡在 Mempool，即使有新的区块和新的 Bundle 空间，随后的追加保证金（Nonce=11）也绝对无法被任何 Bundler 打包上链（必须等 10 落块）。
   而在 4337 二维 Nonce 下，Perps 操作使用独立 key（如 `key=2, seq=1`），它**不需要等待 Meme 操作（`key=1, seq=10`）确认**。Bundler 可以完全跳过卡住的 Meme 操作，在下一个 Bundle 或由另一个 Bundler 立即将 Perps 操作打包上链。

```text
Same Smart Account, EntryPoint nonce state

key=Meme     seq=10 -> seq=11 -> ...
key=Perps    seq= 3 -> seq= 4 -> ...

Meme seq=10 pending in Mempool
   |
   +-- Bundler cannot put both in one bundle (same unstaked sender exclusion)
   |
   v
Bundler can immediately package Perps (seq=3) into a separate bundle/block,
without waiting for Meme (seq=10) to confirm!
```

因此，4337 的二维 Nonce 解决的是**任务流之间的依赖解耦**，不是“在单笔以太坊交易里同时打包同一个人的多笔 UserOp”。**账户 Nonce 隔离、Bundler 的单包选单规则（同一普通 Sender 仅一笔）以及链上执行吞吐是三个完全不同的概念。**

#### 3.1.3 对 Mantle 需求的覆盖与代价

4337 能承载受限 Session Key、有效期、批处理、Gas 赞助和 ERC-20 费用结算，但限权规则、金额记账和业务风控来自账户 / Paymaster 实现。Bundler 的模拟和接收规则还约束验证阶段可以访问哪些状态，复杂风控可能需要放在执行路径检查。

其代价包括 UserOperation 编码、EntryPoint 调用、验证与费用记账，以及 Bundler 的链下工作。规范没有规定“每笔必须等 100–500ms”或“固定多花 60,000–100,000 Gas”。这些指标随版本、账户实现、批大小、网络和服务策略变化，不能直接据此判定 4337 不适合所有高频交易。公开 UserOperation 也会暴露交易意图，4337 不提供内建 MEV 保护。

### 3.2 协议层账户能力与 Native AA

#### 3.2.1 EIP-7702：让存量 EOA 地址执行持久委托代码

7702 新增 `0x04` set-code 交易。EOA 所有者签署授权元组，协议在该地址写入 `0xef0100 || implementation` 委托标记，后续对该地址的调用就在该账户上下文执行实现合约代码。[S2]

**委托会持久保留，不会在交易结束后自动恢复。** 已设置好委托后，日常调用无需每次附带新的 `authorization_list`；更换或清除委托需要相应授权。即使设置委托的交易在后续业务执行中失败，已处理的委托标记也不会随业务 revert 撤销。

```text
Setup / delegation

Owner EOA A -- signs authorization for implementation W --+
                                                        |
Sponsor S -- sends type-0x04 tx, pays native gas ---------+
                                                        v
                                     code(A) = 0xef0100 || W
                                     persistent delegation

Subsequent operation

Agent -- signs a limited operation --> Relayer / Bundler
                                            |
                                            v
                                    call account A
                                            |
                               execute W using A's context
                                            |
                           verify session -> approve + buy
```

图中 `W` 验证 Session Key 的权限和业务签名，而不是把任意外部调用都当作可信授权。7702 的授权元组只批准代码委托，不自动批准图中的买入金额、收款地址或调用参数。

它对存量地址有直接价值：余额和应用身份可以留在原地址，账户代码可支持限权执行、批处理和被赞助调用。但要区分三种 nonce：

| 序号 | 由谁使用 | 7702 是否自动改变它 |
|---|---|---|
| 外层交易 nonce | 发送 set-code 或后续普通交易的 EOA | 保持传统顺序规则 |
| 委托授权 nonce | 授权代码变更的 EOA | 用于授权有效性检查，处理授权时递增 |
| 受限业务操作 nonce | 委托账户代码或 EntryPoint | 由实现定义；可采用多通道 |

表中前两行按使用场景区分，并不表示同一个 EOA 有两套独立计数器：外层交易与该 EOA 的委托授权都使用其系统 nonce。Owner 与 Sponsor 是不同地址时，才分别使用各自的系统 nonce。

如果用户仍亲自以自己的 EOA 发送普通交易，其发送序列仍然串行；如果 Relayer / Bundler 提交受限业务操作，账户代码可以采用独立 nonce，或使用 4337 的 key / sequence。**7702 本身未标准化二维 nonce，但不能据此断言所有 7702 账户都无法隔离任务。** 根 EOA 密钥仍能发交易和更换委托，因此根密钥不应交给 Agent。

#### 3.2.2 RIP-7560：把固定的 AA 阶段纳入协议

RIP-7560 定义新的 AA 交易，将账户部署、账户验证、Paymaster 验证、业务执行与后处理纳入协议规定的阶段。交易有效性取决于验证阶段成功及相应 approval callback；业务执行仍调用 EVM 合约，并不是把所有账户逻辑自动编译成原生客户端代码。[S3]

```text
Agent -- signs native AA transaction --> Node / Block builder
                                              |
                       precharge sender or Paymaster native balance
                                              |
                         +--------------------v------------------+
                         | Validation phase                      |
                         | deploy sender if needed               |
                         | sender.validateTransaction()          |
                         | optional Paymaster validation         |
                         | required approval callbacks           |
                         +--------------------+------------------+
                                              |
                               valid at this block position?
                                  /                     \
                                no                      yes
                                |                        |
                      cannot validly include             v
                                              +------------------+
                                              | Account execution|
                                              | Paymaster post-op|
                                              | fee settlement   |
                                              +------------------+
```

在买入例子中，交易明确携带 sender、执行数据、验证数据和可选 Paymaster。协议先检查账户授权和付费条件，再让账户执行 `approve + buy`。它不要求先将业务操作放进 Bundler EOA 的 `handleOps` 外层交易；已有 Bundler 仍可作为兼容基础设施参与，提案没有禁止它们存在，见 [S3] 的 Migration path。

**二维 nonce 需要连同 RIP-7712 阅读。** 7560 载荷已有 `nonceKey` 与 `nonceSequence`，但其字段说明把 `nonceKey != 0` 的用法交给 7712。7712 引入 NonceManager：非零 key 的序号在独立槽位校验和递增，不再检查或递增该笔交易的旧系统 nonce。[S3][S4]

```text
RIP-7560 with RIP-7712 enabled

Native tx(sender=A, key=Meme,  seq=10) --> NonceManager[A, Meme]
Native tx(sender=A, key=Perps, seq= 3) --> NonceManager[A, Perps]

Different keys: no cross-key sequence dependency
Same key:      still ordered by its sequence
Shared state:  balances, gas capacity and market state still interact
```

7560 和 7712 的草案对字段宽度等细节仍需在实现中统一，本文只用 key / sequence 表达机制，不把它们合并成已经冻结的交易编码。

7560 的原生验证可减少部分应用层封装工作，但节点仍需验证和防范 DoS，账户合约仍产生 Gas。验证无效的交易不能作为有效交易纳入；验证成功后，买入因滑点而 revert 仍会消耗费用。它不承诺“打新失败零费用”“纳秒验签”或“零排队”。

当前引用版本的 RIP-7560 状态为 Draft，交易类型字节尚未确定。不能将 EIP-7702 的 `0x04` 当作 RIP-7560 类型，也不能由 Mantle 使用 OP Stack 或 ZK 证明推出它已支持 7560。

#### 3.2.3 EIP-8141 与 EIP-8130：两种新的 Native AA 抽象

这两份 Core EIP 在本次核查版本中均为 **Draft**，放在一起看有助于理解新的设计分歧：8141 让交易以 frame 编排验证、付款与执行；8130 将身份认证与账户业务逻辑分开，通过 Keystore 中声明的 actor 配置让节点确定如何认证。它们不是必须一起启用的一套上下层协议，也没有仅凭编号就能确认的替代或上线关系。[S5][S6]

##### EIP-8141：Frame Transaction

8141 将一笔原生交易表示为一组 frame，每个 frame 指定模式、目标、Gas 预算、value 和 data。当前草案给出的 `FRAME_TX_TYPE` 是 `0x06`，并通过 `APPROVE` 指令显式批准执行或费用支付。[S5]

| Frame 模式 | 作用 | 关键限制 |
|---|---|---|
| `VERIFY` | 账户或付款方验证交易 | 以静态调用执行；`APPROVE` 是可改变批准上下文及相关状态的特例；失败使交易无效 |
| `SENDER` | 以 sender 身份调用业务目标 | 必须先取得 sender 的执行批准 |
| `DEFAULT` | 使用协议入口上下文执行调用 | 可用于部署或后处理，不能默认当作 sender 已授权的调用 |

以“平台付 Gas、Agent 买入”为例，可把责任分成四个 frame：

```text
Frame transaction: sender=A, nonce=n

F0 VERIFY A
   check Agent signature + permitted actions
   APPROVE(EXECUTION)
          |
F1 VERIFY Sponsor
   check sponsorship terms and native balance
   APPROVE(PAYMENT)
          |
          +---- atomic business group ------------------+
          | F2 SENDER -> USDC.approve(router, 100)       |
          |    ATOMIC_BATCH_FLAG set                    |
          | F3 SENDER -> router.buy(..., recipient=A)   |
          |    batch flag clear: end of group           |
          +---------------------------------------------+

If F3 reverts: F2 and F3 business changes roll back.
Payment approval, nonce consumption and used gas are not undone by that batch.
```

这里的 `APPROVE(EXECUTION)` 和 `APPROVE(PAYMENT)` 是作用域的简写，不是完整 opcode 调用编码。账户验证器必须检查全部后续获授权的 sender frames，覆盖目标、金额、收款人等条件；协议不会为 Agent 自动生成限权策略。

与 7560 固定账户 / Paymaster 阶段不同，8141 提供更通用的 frame 编排和显式批准机制。账户可用自定义代码验证签名，交易也提供签名列表与 introspection 指令供验证逻辑读取。原子组通过 flag 指定，**不是所有 frames 默认全成或全退**。使用者需要读取逐 frame 回执，不能只把“交易被纳入”当作“买入成功”。

对 Mantle 需求还有三项限制：

- **8141 核心规范自身仍是标量 Nonce（但存在伴生提案 EIP-8250 支持多维 Nonce）。** 8141 规范本身检查 `tx.nonce == state[tx.sender].nonce`；为了补齐并发短板，核心团队（含 Vitalik、lightclient 等）于 2026 年 4 月提出了伴生提案 **EIP-8250 (Keyed Nonces for Frame Transactions)**，在协议层引入 `NONCE_MANAGER` 合约将单一 Nonce 替换为 `(nonce_keys, nonce_seq)`，使非重叠的 key 集合在共识执行层面互不依赖。但需注意，EIP-8250 当前 Draft 仍建议公共 Mempool 暂时维持单 sender 仅 1 笔待处理交易。
- **公共 mempool 不是任意 frame 都接收。** 草案规定可识别的验证前缀、访问约束和 Gas 上限，并建议每个 sender 最多保留一笔待处理 frame transaction。链上可表达的交易范围与公共传播范围需要分别评估。
- **代付仍有余额和结算风险。** 官方示例可在业务前转 ERC-20 给 Sponsor，再以独立 frame 执行业务和后处理；Sponsor 最终支付原生网络费用，且要承担验证后用户余额改变等风险。

其变化在于原生授权、付款和调用编排，不应把 8141 的双维 Gas 预算误读成二维 nonce，也不能据此推导并发任务已完全隔离。

##### EIP-8130：Keystore Accounts

8130 当前标题是 **Keystore Accounts**，较早讨论中也称 Account Abstraction by Account Configurations。它引入链上 Keystore 与新的 AA 交易类型，把账户、操作者和认证算法分开。[S6]

- **Account** 是持有资金并执行操作的地址。
- **Actor** 是被账户授权的操作者，可对应 Owner、交易 Agent 或付费身份。
- **Authenticator** 验证签名并返回 `actorId`；它回答“是谁签的”。
- **Keystore** 保存 actor 的 authenticator、scope、expiry 及可选 policy 信息；它与协议检查一起回答“允许以什么身份发起操作”。

```text
Account A
   |
   +--> Keystore: actor configurations
          |
          +-- Owner actor    -> admin authority
          +-- Agent actor    -> authenticator + expiry + POLICY scope
          |                                      |
          |                              manager + commitment
          +-- Payment actor  -> permitted gas-payment role

Agent signs AA tx
   |
   v
Node authenticates -> resolves actorId -> checks scope / expiry / nonce
   |
   +--> resolves payer and verifies payment authorization
   |
   v
Protocol dispatches calls as Account A
   |
   v
Policy manager checks business limits -> executes permitted action
```

协议认证路径明确区分 canonical authenticator 与自定义 authenticator。当前草案的 L1 profile 接受在限定 Gas 内完成静态调用的非 canonical authenticator；L2 profile 默认只让 canonical 集合进入 8130 原生交易路径，以减少验证工作与状态依赖。非 canonical 合约仍可通过普通 EVM 调用或 4337 使用。这里的“无需修改 EVM”不代表零改链：新增交易类型、原生认证、状态处理与 RPC 支持仍需要客户端和协议集成。

**对 Agent 的限权有直接表达，但额度检查仍由策略合约执行。** 比如注册一个有到期时间的 actor，授予 `POLICY | NONCE`，并指定 policy manager；协议限制其顶层目标为该 manager，manager 检查 100 USDC 预算、router 与收款人，再执行买入。

```text
POLICY Agent -- native AA call --> configured manager only
                                      |
                              check commitment / limits
                                      |
                            +---------+---------+
                            |                   |
                         permitted           over budget
                            |                   |
                      execute buy            revert
```

这个例子里不能额外授予 `OPERATOR` 来“增强”权限：当前草案中 `OPERATOR` 允许不受 manager gate 限制的发起操作，`OPERATOR | POLICY` 会使 gate 不再生效。Policy target 检查在执行阶段发生，违反策略的交易可能已经纳入并付费，不能把“签名有效”当成“额度检查通过”。

8130 原生定义 `nonce_key` 与 `nonce_sequence`，并提供 nonce-free 模式：

```text
Account A, native Nonce Manager

key=Meme       seq=10 -> seq=11
key=Perps      seq= 3 -> seq= 4

key=NONCE_KEY_MAX, seq=0
   |
   +--> nonzero expiry + short validity window
   +--> consensus replay_id deduplication
   +--> no sequential nonce slot to consume
```

非零普通 key 可让无关任务使用独立序列；nonce-free 则用短有效期和共识状态中的 replay buffer 防止重复执行，并非放弃重放保护。受限 actor 需要 `NONCE` scope 才能使用有序 nonce key；该 scope 在当前草案中允许整个二维 key 空间，**不会自动把某个 Agent 限制在单一业务通道**。

8130 的 `calls` 由多个 phase 组成：**phase 内原子，phase 按顺序独立提交；一个 phase 失败后，后面的 phase 跳过，之前成功的 phase 保留。** 例如规范中的赞助模式可表示为：

```text
calls = [
  [ reimburse Sponsor ],          // phase 0: commit if successful
  [ approve router, buy token ]   // phase 1: atomic business group
]

phase 1 fails -> its approve + buy roll back
                 phase 0 reimbursement remains
                 nonce and network fees remain consumed
```

若使用 POLICY actor，图中的每个顶层调用也必须经过其 manager，不能为付款随意绕过 gate。Sponsor 和账户需要明确失败时是否退款，以及 manager 是否允许该结算动作。

8130 允许不同 payer 支付原生网络费用，账户合约也可通过 4337 在非 8130 链上运行；但后者不继承 8130 的原生 nonce manager、交易上下文和 phase 处理规则。当前 `AA_TX_TYPE=0x79` 是所引用草案的取值，不作为已分配给 Mantle 的交易类型。

##### 8141 与 8130 的差异

| 维度 | EIP-8141 | EIP-8130 |
|---|---|---|
| 核心抽象 | 一笔交易由不同模式的 frames 编排 | 账户通过 Keystore 声明 actor 与 authenticator |
| 验证灵活性 | 用户定义验证代码，受公共 mempool 规则约束 | canonical 集合可原生认证；其他算法的接收范围取决于 profile |
| 执行批准 | `APPROVE` 显式授权后续 sender frames | 根据认证 actor 的 scope 与 policy gate 授权调用 |
| 受限 Session | 账户验证 / 执行代码实现 | actor、scope、expiry 直接表达身份权限；预算由 manager 实现 |
| 跨交易 nonce | 核心规范为单一 sender nonce；搭配伴生 EIP-8250 可支持 (nonce_keys, nonce_seq) 键控通道 | 原生内置 2D nonce，另有短时 nonce-free 模式 |
| 批处理边界 | 通过 frame flag 组成原子组 | phase 内原子，已成功 phase 不随后续失败回滚 |
| 付费 | frame 中批准原生费用支付，可另行 ERC-20 结算 | `payer` / `payer_auth` 指定付款身份，可另行 ERC-20 结算 |
| 主要取舍 | 通用编排带来验证与 mempool 管理要求 | 可预测认证依赖明确的配置和算法集合，业务策略仍需合约 |

### 3.3 所有方案与 Mantle AA 特性的对比

下面的“支持”描述**机制能否表达需求**，不代表它已在 Mantle 启用，也不代表有现成账户实现。记号含义：**原生**为该方案直接定义；**实现**为需要账户、模块或业务合约完成；**条件**为依赖额外规范或提交路径；**不提供**为方案本身无法保证。

#### 3.3.1 功能覆盖矩阵

| Mantle 需要的特性 | ERC-4337 | EIP-7702 | RIP-7560 | EIP-8141 | EIP-8130 |
|---|---|---|---|---|---|
| **A：管理权与 Agent 执行权分离** | 实现：账户验证 Session | 实现：委托代码验证 Session，根 EOA 权限保留 | 实现：账户验证逻辑 | 实现：验证及执行代码检查 Agent 权限 | 原生 actor / scope / expiry；详细预算需 manager |
| **A：限额、有效期、撤销** | 实现限额和撤销；验证结果可带有效期 | 委托代码实现；代码委托本身持久存在 | 实现限额和撤销；验证可返回有效期 | 账户实现限权及撤销，可使用 expiry verifier | 原生 actor expiry / revoke；限额由 policy manager 检查 |
| **B：独立任务 nonce 通道** | 原生于 EntryPoint：192-bit key + 64-bit sequence；检查 Bundler 接收策略 | 条件：委托代码自定义或结合 4337；普通外层 nonce 仍串行 | 条件：须结合 RIP-7712 非零 key 支持 | 条件：核心规范为标量，需结合 EIP-8250 支持 Keyed Nonces | 原生 2D nonce；另有 nonce-free 与共识去重 |
| **C：Agent 无原生币也可交易** | Paymaster 存款支付，Bundler 发外层交易 | Sponsor / Relayer 支付外层费用 | 原生 Paymaster 余额支付 | 原生付款批准 frame | 原生 payer / payer_auth |
| **D：用 ERC-20 向付款方结算** | Paymaster / 账户实现 | 委托代码 / Relayer 实现 | Paymaster / 账户实现 | ERC-20 frame 与 Sponsor 实现，规范有例子 | call phases 与付款方实现，规范有例子 |
| **D：从已锁定保证金扣费** | 条件：Perps 允许的扣费接口 | 同左 | 同左 | 同左 | 同左 |
| **E：同链同步调用的原子批处理** | 账户执行器实现 | 委托执行器实现 | 账户执行器实现 | 原生 atomic batch flag | 原生 phase 内原子 |
| **F：支出、杠杆、净值风控** | 账户 / Hook / 业务合约实现 | 委托代码 / 业务合约实现 | 账户 / 业务合约实现 | 账户 / 执行合约实现 | policy manager / 业务合约实现 |
| **G：不依赖 Bundler EOA 封装** | 不提供：UserOp 由外层交易承载 | 条件：可普通调用；赞助时仍有外层发送者 | 原生交易路径提供 | 原生 frame transaction 提供 | 原生 AA transaction 提供 |
| **交易必定优先、亚秒纳入、抗 MEV** | 不提供 | 不提供 | 不提供 | 不提供 | 不提供 |

表中依赖关系依据各规范的验证、nonce、执行与付费章节。[S1][S2][S3][S4][S5][S6]

从功能覆盖看，**受限授权与代付并不要求先有 Native AA**，4337 和 7702 的账户实现已经能表达相当一部分需求。原生方案改变的是协议直接识别的角色与执行边界，其中 8130 明确提供多通道 nonce，7560 需结合 7712，而当前 8141 没有这一能力。

#### 3.3.2 失败语义、费用与提交开销

| 维度 | ERC-4337 | EIP-7702 | RIP-7560 | EIP-8141 | EIP-8130 |
|---|---|---|---|---|---|
| 主要提交工作 | UserOp 模拟、Bundler 选择、外层交易 | 设置委托后调用账户；取决于代发或 4337 路径 | 原生验证阶段与构块检查 | frame 验证前缀、签名和访问规则检查 | actor 配置、认证、nonce、payer 检查；复杂度受 profile 影响 |
| 业务失败回执 | 需检查 UserOperation 结果，不能只看 bundle 外层成功 | 检查账户执行结果；已处理委托不随业务失败撤销 | 独立 execution / postOp 状态 | 逐 frame 回执，原子组以外结果可保留 | 逐 phase 状态；后续失败保留之前成功 phase |
| 执行失败是否免费 | 否，已消耗 Gas 需结算 | 否 | 否 | 否 | 否，policy gate 失败也可能是付费执行失败 |
| 网络最终付费资产 | 链原生币 | 链原生币 | 草案按原生余额支付 | 草案按原生余额支付 | 草案按原生余额支付 |
| 能否单凭规范给出稳定 TPS / 毫秒数 | 不能 | 不能 | 不能 | 不能 | 不能 |

性能不能用“原生”二字排序。4337 的额外封装是真实成本，但 Bundler 不必遵循一个固定攒批延迟；8141 提供直接交易路径，但其公共 mempool 对单 sender 待处理数量有限制；8130 的 canonical 认证路径减少任意钱包代码验证，却仍要读状态、维护 nonce / 重放记录和执行策略。

规范中的基础费用常量也不等于业务交易总费用。比如所引版本的 7560 使用 15,000 基础 Gas，8141 使用 12,000 基础项加每 frame 475 等费用，8130 使用 15,000 基础项并另计 nonce 与认证成本；它们有不同的成本分项与草案依赖。把这些数值与 EOA 的 21,000 简单转账基础项直接相比，会遗漏业务执行、数据、签名和存储成本。[S3][S5][S6]

对 Meme 与 Perps 有意义的比较，应使用相同的买入 / 补仓操作、相同权限策略和成功 / 失败分布，分别记录提交至纳入的延迟、实际总费用及拥堵时的完成率。本文未提供此类实测数据，因此不声称某一方案可以固定节省多少 Gas 或提高多少 TPS。

#### 3.3.3 规范状态与 Mantle 可用性的区别

| 方案 | 本次引用的规范状态 | 在 Mantle 使用时需要确认的前提 |
|---|---|---|
| ERC-4337 | Final；应用层标准 | 具体 EntryPoint、账户、Bundler、Paymaster 版本与服务可用性；不需要新增原生交易类型 |
| EIP-7702 | Final；协议层委托能力 | Mantle 对 `0x04` 及委托执行语义的分叉支持，再检查具体钱包实现 |
| RIP-7560 | Draft | 新交易类型、验证阶段、费用与回执处理；二维通道还涉及 7712 |
| EIP-8141 | Draft | Frame transaction、新指令、mempool 与相关协议依赖的客户端实现 |
| EIP-8130 | Draft | AA 交易类型、Keystore、认证集合、nonce manager、phase 执行与 RPC 集成 |

本次研究没有取得足以证明 Mantle 已启用 7560、8141 或 8130 的链上与客户端证据，因此不把它们列为当前可直接调用的 Mantle 功能。规范为 Final 也不能证明某条 L2 已启用 7702；同样，部署 Keystore 合约或使用 4337 不等于启用了 8130 原生交易。

本文的比较停留在需求满足度：对 Meme，五条路径都能通过相应实现表达受限交易与代付；对 Perps，差异更集中在独立 nonce、提交策略、失败回滚与外部业务依赖。没有一份 AA 规范单独保证清算前补仓成功、从任意保证金扣费或阻止全部 MEV。

## 来源与版本

五个主体方案及 RIP-7712 的规范状态与机制以以下固定提交为准。在线页面可能继续更新，尤其是 8141 与 8130；文中的流程图是依据这些版本绘制的教学简化图，不是可直接广播的完整交易编码。

- **[S1] ERC-4337**：[规范页面](https://eips.ethereum.org/EIPS/eip-4337) · [固定版本 c8d8c210（2026-06-02）](https://github.com/ethereum/ERCs/blob/c8d8c2107f63c996fe8b60af20081d4d911e1f9d/ERCS/erc-4337.md)。重点：Smart Contract Account Interface、Semi-abstracted Nonce Support、EntryPoint functionality、Paymasters、Bundling。
- **[S2] EIP-7702**：[规范页面](https://eips.ethereum.org/EIPS/eip-7702) · [固定版本 bbc3f958（2025-10-07）](https://github.com/ethereum/EIPs/blob/bbc3f95844c37612a2f1b9e7477990bb717ecfa0/EIPS/eip-7702.md)。重点：Behavior、Persistence of code delegation、Gas Costs、Security Considerations。
- **[S3] RIP-7560**：[固定版本 0bc0739c（2025-05-31）](https://github.com/ethereum/RIPs/blob/0bc0739c305ca94524b02b21147998b339f8891e/RIPS/rip-7560.md)。重点：New Transaction Type、nonce 字段、Multiple execution frames、Execution layer transaction validation、Migration path。
- **[S4] RIP-7712**：[固定版本 4af5e521（2025-01-14）](https://github.com/ethereum/RIPs/blob/4af5e52142f7b81375d3a7bc588152349b385f3b/RIPS/rip-7712.md)。重点：Non-sequential nonce support、Nonce validation、NonceManager、Security Considerations。
- **[S5] EIP-8141**：[规范页面](https://eips.ethereum.org/EIPS/eip-8141) · [固定版本 b75cbe61（2026-09-01）](https://github.com/ethereum/EIPs/blob/b75cbe61150f09a44c38843be916417283d5b7bf/EIPS/eip-8141.md)。重点：Frame Transaction、Behavior、APPROVE、Mempool、Atomic batching、Examples。
- **[S6] EIP-8130**：[规范页面](https://eips.ethereum.org/EIPS/eip-8130) · [固定版本 16390e1f（2026-09-16）](https://github.com/ethereum/EIPs/blob/16390e1faee756757021e785f4e6ef35a7e6a0f5/EIPS/eip-8130.md)。重点：Adoption Profiles、Actor Scope / Policies、2D Nonce Storage、Call Phases、Portability、Validation Flow。
- **[S7] ERC-7579**：[Minimal Modular Smart Accounts](https://eips.ethereum.org/EIPS/eip-7579)。仅用于说明模块接口与交易类型的区别。
- **[S8] EIP-8250**：[Keyed Nonces for Frame Transactions](https://eips.ethereum.org/EIPS/eip-8250)。作为 EIP-8141 的伴生规范，为 Frame Transactions 引入多维 Nonce 键控。
- **[P1] Tape 场景的既有账户需求笔记**：[03-meme-and-perps-agent-requirements.md](./03-meme-and-perps-agent-requirements.md)。作为产品研究输入，旧笔记中的性能与安全绝对化表述不作为规范证据。
- **[P2] Agent Meme Launchpad 后续研究**：[agent-meme-launchpad-report.md](../7-agent-meme-launchpad/agent-meme-launchpad-report.md)。用于区分 Agent 发币产品与 Tape。
- **[P3] General EVM Perps 研究**：[general-evm-perps-on-mantle.md](../3-perps/general-evm-perps-on-mantle.md)。用于区分 RFQ、Oracle-pool、链下报价与链上结算。

[S1]: https://github.com/ethereum/ERCs/blob/c8d8c2107f63c996fe8b60af20081d4d911e1f9d/ERCS/erc-4337.md
[S2]: https://github.com/ethereum/EIPs/blob/bbc3f95844c37612a2f1b9e7477990bb717ecfa0/EIPS/eip-7702.md
[S3]: https://github.com/ethereum/RIPs/blob/0bc0739c305ca94524b02b21147998b339f8891e/RIPS/rip-7560.md
[S4]: https://github.com/ethereum/RIPs/blob/4af5e52142f7b81375d3a7bc588152349b385f3b/RIPS/rip-7712.md
[S5]: https://github.com/ethereum/EIPs/blob/b75cbe61150f09a44c38843be916417283d5b7bf/EIPS/eip-8141.md
[S6]: https://github.com/ethereum/EIPs/blob/16390e1faee756757021e785f4e6ef35a7e6a0f5/EIPS/eip-8130.md
[S7]: https://eips.ethereum.org/EIPS/eip-7579
[P1]: ./03-meme-and-perps-agent-requirements.md
[P2]: ../7-agent-meme-launchpad/agent-meme-launchpad-report.md
[P3]: ../3-perps/general-evm-perps-on-mantle.md
