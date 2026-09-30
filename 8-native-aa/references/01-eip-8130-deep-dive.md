---
title: "EIP-8130 Deep Dive：Keystore Accounts、受限 Actor 与交易生命周期"
description: "从普通 EOA 交易出发，解析 EIP-8130 的认证、权限、代付、二维 nonce、Phase 回滚与工程边界。"
last_verified: "2026-09-24"
status: "基于 Draft 规范的研究文档，不是生产实现或性能报告"
---

# EIP-8130 Deep Dive：Keystore Accounts、受限 Actor 与交易生命周期

> **核心问题：谁有权使用一个账户、使用什么密钥证明身份、能做什么，以及由谁支付费用？**
>
> EIP-8130 将这些问题拆成 Account、Actor、Authenticator、Keystore 和业务 Policy，再通过新的原生交易类型把它们连接起来。它标准化的是认证与基础权限入口，不是完整的交易策略或风险管理系统。[^spec]

**系列导航：** 本篇解析 8130；另见 [8141 Deep Dive](02-eip-8141-deep-dive.md) 和 [Mantle 产品能力覆盖与适配评估](03-mantle-aa-coverage-and-fit.md)。

**阅读建议：** 首次阅读先看 §1～§2 的交易前后对照；§3～§7 解释账户、认证、授权和 nonce；§8～§10 讨论回滚、费用与准入；§11～§12 汇总订单集成和原稿修正。

## 阅读口径与材料来源

本文以用户提供的《EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析》中的 **§3、§4，以及 §5 的相关比较**为主要组织依据，保留 Actor、Authenticator、Scope、Policy Manager、Epoch、2D nonce、nonce-free、Phase 和 Adoption Profiles 等术语及技术层次。[^intro]

在此基础上，使用 **2026 年 9 月 24 日访问的 EIP 官方文本**核对机制，新增交易前后对照、失败路径和实现边界。原稿标注的 8130 commit 为 `16390e1f`；本文没有对该 commit 及其全部依赖运行客户端或合约测试，也不将当前网页内容声称为该 commit 的逐字快照。

全文区分三类内容：

| 标记／位置 | 含义 |
|---|---|
| 机制正文与规范引用 | 原稿整理，并经当前官方规范核对；发生实质修正时在文末列出 |
| “设计示例”“示意代码” | 为解释生命周期而构造，不是 EIP 定义的 ABI，也不是 Mantle 已部署能力 |
| “待验证”“规范待澄清” | 尚需固定版本、实现或压测才能确定的事项 |

8130 当前仍为 **Draft**。官方明确指出：协议正文规定交易与节点行为，而 **Keystore、账户和认证器的精确存储、ABI、typehash 及函数实现，以 canonical contracts 仓库为权威**。参考仓库提示代码尚未审计、不应用于生产。[^spec][^repo]

---

## 1. 先看原本的一笔交易：EOA 是怎样买入代币的

### 1.1 基线设定

假设 Alice 用账户 `U`，花费 100 USDC 在 Meme Launchpad 买入代币 `MEME`。

这里的“原本”特指 **普通 EOA＋普通 EVM 交易**，不包含已安装的智能账户、ERC-4337、EIP-7702 委托钱包、Permit 或应用专用元交易。选择这个基线是为了看清协议结构变化，**不意味着只有 8130 才能实现批处理或代付**。[^e1559][^e7702]

先假设 USDC allowance 已经足够，一笔普通交易的生命周期是：

```text
① 前端／Agent 生成 Launchpad.buy 的 calldata
        ↓
② 钱包读取 U 的账户 nonce、估计 Gas、设置费用上限
        ↓
③ Alice 用 U 的 secp256k1 私钥签署普通交易
        ↓
④ RPC／节点接收：校验签名、nonce、费用与余额
        ↓
⑤ 排序器／区块构建者决定纳入顺序
        ↓
⑥ 执行交易：U 的 nonce 被消耗，U 支付 Gas
        ↓
⑦ Launchpad 看到 msg.sender = U，执行买入
        ↓
⑧ 输出回执；应用确认买入事件与资产余额
```

在这个基线中，账户身份、签名私钥与费用支付主体高度绑定。单个 EOA 的交易使用一个有序 nonce；业务执行失败后，交易 nonce 和已经消耗的费用不会退回为“未提交”。[^e1559]

### 1.2 没有 allowance、没有原生 Gas 资产时，业务流程不再是一笔交易

```text
先为 U 补充原生 Gas 资产
        ↓
交易 A：USDC.approve(Launchpad, 100)
        ↓
交易 B：Launchpad.buy(100, minOut, ...)
```

如果交易 A 已成功，而交易 B 因滑点失败，**A 中设置的 allowance 不会因为 B 的失败而撤销**。两笔独立交易不是一个共同回滚单元。支持 Permit、智能账户或专用入口的应用可以改造这条路径，但那是额外机制，不是普通 EOA 交易本身提供的能力。[^erc20][^e1559]

对于持续运行的 Agent，还会遇到另一种选择：要么持续请求 Owner 签名，要么把能够控制资产的密钥交给自动化进程。普通交易信封没有表达“这把键只允许在某个市场、某个额度内交易”的通用字段。

**8130 要改变的，正是认证主体、操作权限、重放保护、付款人与调用分组的表达方式。**

---

## 2. 换成 8130：同一个业务意图的完整生命周期

### 2.1 示例账户与权限

以下为**设计示例**，不是标准账户实现：

| 对象 | 示例配置 |
|---|---|
| 用户账户 `U` | 已安装经过审计的、理解 Policy 的账户代码 |
| Owner | 管理账户配置，不参与每一次自动交易签名 |
| Agent key `K` | 独立密钥，以 Actor 身份注册 |
| Agent Scope | `POLICY \| NONCE`，不授予 `OPERATOR` 或付款权限 |
| Manager | `U` 自身；账户代码必须进行完整 Policy 检查，不能信任任意 self-call |
| Policy | 允许指定 Launchpad、指定资产、固定资产接收人、每笔 100 USDC 上限和累计预算 |
| Sponsor `P` | 通过自己的付款授权承担原生 Gas 费用 |
| nonce 通道 | 为此策略使用 `nonce_key = 7`；通道分配不是协议强制的 Actor 所有权 |

采用 `manager = U` 是为了在示例中让账户代码直接发起后续 Token／Launchpad 调用，保持资产所有者身份。**账户必须拒绝受限 Actor 经其他入口执行任意动作；仅检查 `msg.sender == address(this)` 不安全。**第 5 节解释其他 Manager 架构。[^policy]

### 2.2 初始化与每笔交易分开理解

```text
初始化／授权阶段
Owner 安装或选择 Policy-aware 账户代码
    → Owner 授权 Actor K，并绑定 Manager、Policy、expiry
    → 为 Sponsor 建立付款授权与资金
    → Agent 开始工作

后续交易阶段
Agent 决定买入
    → 自己签署本次交易
    → Sponsor 审核并签署本次付款授权
    → 无需 Owner 再次确认
```

Owner 对“允许什么”的授权，和 Agent 对“本次具体做什么”的签名，是两份不同授权。

8130 也支持把 Owner 签好的配置变更附在首次 Agent 交易的 `account_changes` 中：节点先模拟配置变化，再依据**变化后的 Actor 状态**认证交易。不过，账户代码安装、Owner 签署配置、首次 Agent 操作能否合并，取决于使用 Create、Config change 还是 Delegation 路径，不能概括为“一切初始化都只需要 Agent 一次签名”。[^changes][^validation]

### 2.3 一笔 8130 买入交易的示意结构

下列字段使用符号值，`executePolicy` 是示例账户入口，不是 EIP 标准 ABI：

```text
AA transaction:
  type                = 0x79
  chain_id            = C
  sender              = U
  nonce_key           = 7
  nonce_sequence      = 42
  valid_after         = 0
  valid_before        = 短期截止时间
  gas_limit           = 用户侧 Gas 预算
  account_changes     = []  // 本例 Actor 已授权
  calls               = [
                          [
                            [U, executePolicy(
                                  operationId,
                                  policyParameters,
                                  [
                                    USDC.approve(Launchpad, 100),
                                    Launchpad.buy(100, minOut, recipient=U)
                                  ]
                                )]
                          ]
                        ]
  metadata            = 可选业务标注
  payer               = P
  sender_auth         = K 对 sender hash 的认证数据
  payer_auth          = P 的授权键对 payer hash 的认证数据
```

这里把两步业务操作放进账户的一次受限执行调用中。账户内部必须检查 allowance、目标、接收人、实际支出及调用结果，不能只检查传入的“预算声明”。

### 2.4 从签名到最终业务状态

| 阶段 | 发生什么 | 责任主体 |
|---|---|---|
| 1. 生成意图 | Agent 选择市场、金额、最低成交数量及 `operationId` | Agent／SDK |
| 2. 构造交易 | 选择 nonce 通道和序号、Phase、费用及有效期 | SDK |
| 3. 操作签名 | `K` 对 sender payload 签名 | Agent |
| 4. 付款签名 | Sponsor 审核业务、费用及预算，签署独立 payer payload | Sponsor |
| 5. 准入验证 | 解析、模拟配置变更、验证 Actor、scope、nonce、付款能力及时间条件 | 节点／排序器 |
| 6. 入块重验与协议处理 | 依实际前状态验证；预扣费用、消费 nonce／登记去重，应用账户配置 | 执行客户端 |
| 7. 入口门禁 | 对每个顶层 call 检查 `call.to == manager` | 8130 协议 |
| 8. 策略检查与执行 | Policy-aware 账户识别 Actor，校验权限与预算，执行授权及买入 | 账户／产品合约 |
| 9. 回滚与费用结算 | 成功则保留业务状态；失败按 Phase 回滚，仍结算已发生费用 | 协议＋EVM |
| 10. 业务确认 | 解码 Phase、账户变更、业务事件和最终余额；判断是否需要重试 | SDK／索引服务 |

上述是逻辑生命周期，不是可以直接替代客户端状态转换函数的伪实现。特别是配置变更的**模拟验证**与入块时的**实际应用**不能混为一谈；无效交易的临时模拟状态不会成为链上结果。[^validation][^execution]

### 2.5 前后到底改变了什么

| 维度 | 普通 EOA 基线 | 8130 示例 |
|---|---|---|
| 谁签本次交易 | EOA 主密钥 | 已被 Owner 授权的 Agent Actor |
| 链上账户身份 | EOA | 用户账户 `U`，不必等于签名密钥地址 |
| 权限限制 | 交易信封不表达受限密钥策略 | 协议强制 Manager 入口；账户执行细粒度策略 |
| Gas 付款 | Sender | 独立 Sponsor，或账户内部独立 Gas key |
| 多操作 | 常需应用额外机制／多笔交易 | 原生 Phase 分组＋账户内部批处理 |
| nonce | 单一有序序列 | 原生多通道或短时 nonce-free |
| 业务失败 | 当前普通交易回滚 | 当前 Phase 回滚，已提交前序 Phase 可能保留 |
| 用户体验 | 操作确认、准备 Gas、管理重试 | 一次设置权限后持续自动操作；仍需预算、撤权和恢复系统 |

---

## 3. 账户模型：Account、Actor、Authenticator、Keystore 分别是什么

### 3.1 Account 与 Actor 不是同一个东西

**Account 是资金与账户身份的容器；Actor 是某个已被授权的操作主体。**

同一个账户可以有 Owner、交易 Agent、专用 Gas key 等多个 Actor。一个 Actor 的权限来自该账户在 Keystore 中的配置，而不是来自“它能签出一个有效签名”这一事实。

```text
Account U
   │
   └── Keystore 中的授权
         ├── Owner Actor       → ADMIN
         ├── Launchpad Agent   → POLICY | NONCE
         ├── Perps Agent       → POLICY | NONCE
         └── Dedicated Gas Key → SELF_PAYER
```

Actor 不天然拥有独立资金、独立保证金或独立 PnL。**多个 Actor 共享一个 Account，不等于产品已经提供多个资金隔离子账户。**[^actors]

### 3.2 Authenticator 只回答“谁签了”，而不是“能做什么”

标准接口为：

```solidity
interface IAuthenticator {
    function authenticate(bytes32 hash, bytes calldata data)
        external view returns (bytes32 actorId);
}
```

协议先使用 Authenticator 获得 `actorId`，再读取该账户的 Actor 配置，检查认证器匹配、有效期与 scope。

Canonical 集合包含：

| 认证器 | 认证方式 | `actorId` |
|---|---|---|
| `k1` | secp256k1 | 恢复地址右对齐为 32 字节 |
| `p256` | P-256 | `keccak256(x \|\| y)` |
| `passkey` | WebAuthn／FIDO2 | `keccak256(x \|\| y)` |
| `delegate` | 另一个账户的签名授权 | 被委托账户地址右对齐为 32 字节 |

`p256` 与 `passkey` 的 Actor ID 派生相同，不代表两种验证语义相同；完整 WebAuthn 需要处理相应认证数据。协议还要求实际使用的认证器与存储的认证器匹配。[^auth]

Delegate 是**签名权限委托**，不要与 EIP-7702 的**账户代码委托**混淆。当前签名委托只允许一层，嵌套认证器必须属于 canonical 集合，且不能继续是 delegate。它也不自动提供任意层级的组织权限树。[^auth]

### 3.3 Actor 配置与存储

原稿的紧凑打包结构保留如下：

```text
actor_config: 32 bytes

authenticator (20) | expiry (6) | scope (2) | reserved (4)
```

`expiry` 是秒级 Unix 时间戳；`0` 表示无到期时间，入块时 `block.timestamp > expiry` 才失效。`scope == 0` 表示 ADMIN／无限制，而不是“没有权限”。

**单槽位只指基础 Actor 配置。**配置策略时还会有 `policy_manager` 和 `policy_commitment`；账户自身还有配置序号、锁定等状态。原生 secp256k1 self-actor 又是例外，保存在 packed account-state 的 inline 字段中。不要据此估算“每个账户总共只占一个槽”。[^storage]

`policyData` 只允许两种长度：

```text
空                  → 不带策略，并清除先前策略槽
20-byte manager
  + 32-byte commitment → 共 52 字节
```

**是否存储策略，由长度决定；是否启用 Manager 门禁，由 scope 决定。**只写了 commitment，却没有 `POLICY` 位，不会自动形成受限权限。反之，`POLICY` 没有有效 Manager，会被限制到零地址，并非自动降级为 unrestricted。[^policy]

---

## 4. 交易信封、签名域与付款模式

### 4.1 原生交易格式

```text
0x79 || RLP([
  chain_id,
  sender,
  nonce_key,
  nonce_sequence,
  valid_after,
  valid_before,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  gas_limit,
  account_changes,
  calls,
  metadata,
  payer,
  sender_auth,
  payer_auth
])

calls = [
  [call, call, ...],  // Phase 0
  [call, call, ...],  // Phase 1
  ...
]

call = [to, data]
```

与普通交易最明显的差别是：**显式账户身份、双维 nonce、有效期、账户变更、多组 calls、独立付款人及两组认证数据。**[^tx]

### 4.2 两种 sender 认证路径

| `sender` 字段 | `sender_auth` | 解释 |
|---|---|---|
| 空 | `r \|\| s \|\| v` | 通过 EOA 签名恢复 sender |
| 显式账户地址 | `authenticator(20 bytes) \|\| data` | 认证指定账户的 Actor |

不要把 8130 的 secp256k1 字节顺序直接用于 8141：8130 这里是 `r || s || v`，8141 标准签名为 `v || r || s`。SDK 需要按交易类型编码。[^tx][^e8141]

EOA self-actor 的隐式 Owner 路径还要检查 Keystore 中的撤销、scope 和 expiry 配置，不能理解为“只要恢复出了 EOA 地址，永远直接当管理员”。

### 4.3 操作签名与付款签名有不同域

```text
sender_hash =
  keccak256(0x79 || RLP(从 chain_id 到 payer 的所有字段))

payer_hash =
  keccak256(0x7A || RLP(从 chain_id 到 payer 的所有字段))
```

二者都不包含 `sender_auth` 和 `payer_auth` 本身。付款签名中的 sender 必须使用**解析后的真实 sender 地址**，不能在 EOA 路径继续使用空字段，否则会产生跨 sender 的付款授权重放问题。

两者都绑定费用、Gas 预算、有效期、操作和付款人。因此，**改费用或更换 Sponsor 通常都需要重新生成受影响的签名**；不能因为 `replay_id` 排除了费用，就认为旧签名可以授权新费用。[^sign]

### 4.4 三种付款模式

| 模式 | 关键字段 | 谁需要何种权限 |
|---|---|---|
| 同一个 Actor 自付 | `payer`、`payer_auth` 均为空 | Sender Actor 具有 `SELF_PAYER` 或 ADMIN |
| 同账户专用 Gas key | `payer = sender`，提供 `payer_auth` | 独立 Gas Actor 具有 `SELF_PAYER` |
| 外部 Sponsor | `payer = P ≠ sender`，提供 `payer_auth` | P 账户中的付款 Actor 具有 `SPONSOR_PAYER` |

第二种很有价值：交易 Agent 不必自身携带 `SELF_PAYER`，可以由另一把受服务控制的 Gas key 批准费用。

但**把权限拆成两把密钥，不会自动限制累计付款**。Gas key 的签名服务仍需要每笔费用上限、累计预算、并发预留和反滥用策略。[^payer]

### 4.5 时间单位不能混用

Actor expiry、账户锁定时间使用秒。交易的 `valid_after`／`valid_before` 接受秒或毫秒编码，按固定阈值 `10^11` 归一化为毫秒；`0` 表示没有对应边界。

实际生效判断仍依赖 `block.timestamp`。**毫秒字段不是毫秒级共识时钟，也不意味着毫秒级成交保证。**尚未生效或剩余时间过短的交易，节点也可能不接纳。[^tx]

---

## 5. Scope 与 Policy：8130 的权限边界在哪里

### 5.1 Scope 表

| 数值 | 名称 | 授予的能力 |
|---|---|---|
| `0x0000` | ADMIN | 无限制权限；配置管理的管理员判据 |
| `0x0001` | OPERATOR | 无 Manager 门禁的交易发起；默认通用账户签名的操作权限 |
| `0x0002` | SELF_PAYER | 支付本账户交易的 Gas |
| `0x0004` | SPONSOR_PAYER | 为其他 sender 支付 Gas |
| `0x0008` | POLICY | 交易顶层 call 只能到配置的 Manager |
| `0x0010` | NONCE | 受限 Actor 可以使用有序 nonce 通道，包括 key 0 |
| 其余位 | Reserved | 未知位不授予已知权限 |

Scope 是 **pure grants**：增加权限位通常扩大能力，不是增加限制。特别是：

```text
POLICY                 → 有 Manager 门禁
OPERATOR | POLICY      → 按更宽松的 OPERATOR 处理，没有该门禁
```

因此，“给 POLICY Agent 追加 OPERATOR，让订单签名接口通过”不是兼容性修复，而是权限升级。[^scope]

### 5.2 Manager Gate 是执行阶段检查

对于带 `POLICY`、不带 `OPERATOR` 的 Actor：

```text
allowedTarget = policy_manager(sender, actorId)

对每一个 Phase 中的每一个顶层 call：
    若 call.to != allowedTarget：
        不派发调用
        当前 Phase 失败
```

Manager 在 `calls` 开始时读取为快照；本次较早调用改变配置，不会把后续顶层门禁重新指向另一个 Manager。

违规错误为：

```solidity
error ActorPolicyViolation(bytes32 actorId, address target);
```

这是**业务执行失败，不是交易无效**。nonce／去重状态及费用仍按有效入块交易处理。[^execution]

### 5.3 协议只保证入口，业务策略由 Manager 执行

Manager 需要回答：

```text
参数是否对应 Owner 签署的 commitment？
允许的是哪个 token、哪个 market、哪个 selector？
接收人是否固定为用户账户？
本次 allowance 是否超过需要？
累计交易预算是否还有剩余？
仓位、保证金和风险条件是否仍满足？
是否允许升级、任意 delegatecall 或转移资产？
```

只允许某个 Router 地址，不等于安全：Router 的 calldata 可能包含任意接收人、任意 token 或扩展调用。限制“每笔输入金额”也不等于限制杠杆、累计损失或每日换手。

**Policy commitment 是被签署并存储的策略参数承诺，不是协议自带的风险规则解释器。**[^policy]

### 5.4 Manager 如何获得操作账户资金的权限

这是原稿需要补齐的一环：

```text
Account U → Manager M → DEX
```

普通 CALL 语义下，DEX 看到的是 `M`，不是 `U`。Manager 被写进 Keystore，不会凭空获得账户资产权限。可选择以下设计，但都需要独立安全验证：

| 设计 | 如何执行 | 主要边界 |
|---|---|---|
| `manager = account` | 账户自身校验 Policy，再调用业务合约 | 不能信任任意 self-call，所有危险入口都需封闭 |
| 受限模块 | Manager 经账户的受限执行接口调用 | 验证模块身份、Actor 上下文、参数和重入边界 |
| 产品充当 Manager | Launchpad／Perps 入口直接理解 Actor | 产品验证授权并管理其账本或预先允许的资金路径 |
| allowance／预存资金 | Manager 使用有限 allowance 或托管账本 | 额度、撤销、升级与资金退出风险 |

本篇第 2 节使用第一种。它不是“把一个通用钱包的 Manager 字段设成自己就完成了”，而是需要 **Policy-aware 账户实现**。[^policy][^security]

### 5.5 Policy 验证器示意

以下为**职责伪代码**，不是可编译或可直接部署的合约：

```text
executePolicy(operationId, policy, actions):
    require 当前处于可识别的 8130 执行上下文
    require TransactionContext.sender == 本账户
    actorId = TransactionContext.senderActorId

    cfg, manager, commitment = Keystore.getActorWithPolicy(本账户, actorId)
    require 当前 Actor 仍满足所需权限与业务有效性
    require manager == 本账户
    require hash(规范编码(policy)) == commitment

    require operationId 尚未被消费
    检查 actions 的全部目标、selector、token、amount、recipient、value
    检查累计预算；按防重入方式预留或更新额度
    逐项执行；检查 revert 和 token 返回值
    检查执行后的余额／仓位约束
    记录 operationId 与 Actor 业务事件
```

不同交易传输不能无条件共用这个上下文读取方法：没有 8130 预编译的链或其他提交路径，应显式采用独立、经过验证的认证适配器，而不是把空返回值当作授权成功。

### 5.6 三类预算必须分开

```text
交易投入预算：允许使用多少 token？
市场风险预算：允许建立多大仓位／损失或敞口？
Gas 预算：最多允许消耗多少原生费用？
```

`POLICY | SELF_PAYER` 的 Actor 可以反复发起最终失败的交易。Manager 的交易额度检查并不自动限制这些费用。

同理，`POLICY | SPONSOR_PAYER` 的**发起权限**受 Manager 限制，但其**赞助第三方交易的权限**不自动受该门禁约束。不能将交易 Policy 当成完整的账户损失上限。[^scope][^security]

---

## 6. 账户配置、首次使用、JIT 授权与撤权

### 6.1 三类 account_changes

| 类型 | 功能 | 关键约束 |
|---|---|---|
| `0x00 Create` | 创建账户并安装 initial actors | 至多一个，必须在最前；runtime code 直接放置，不执行构造器 |
| `0x01 Config change` | 授权、撤销、epoch、锁定等配置批次 | 带独立管理员认证与配置序号 |
| `0x02 Delegation` | 设置 EIP-7702 风格代码委托 | 至多一个；需要原生 secp256k1 self-actor 的 ADMIN 权限 |

新账户地址通过 CREATE2 规则派生，并绑定初始 Actor 集合及其权限，防止换掉初始密钥。初始 Actor 必须严格按 `actorId` 升序排列。

**initial_actors 不直接表达 expiry，也不能直接承诺尚未算出的 `manager = account`。**可以在创建后追加同笔配置变更来设置。初始化业务代码运行在后续 calls 中，所以账户代码应在未初始化时保持安全，不能依赖“创建后一定成功执行 initializer”。[^changes]

### 6.2 三种账户接入路径

EOA 可使用原有键发起 8130 交易；无代码且没有显式 Create／Delegation 的路径会自动委托到默认账户。已有智能账户需要显式调用 Keystore 的 import 流程；新无 EOA 账户走 Create。

**账户配置管理的权限迁移需要应用配合。**例如 import 后账户自身旧 Owner 逻辑与 Keystore 权威配置若不一致，会形成双重权限来源。当前参考仓库要求账户实现相应确认接口，并建议以专用 Owner-gated 入口处理 import。[^spec][^repo]

**还要区分“撤销 8130 身份”与“禁用旧 EOA 私钥”。**8130 保留普通 EOA 交易的兼容路径；对于从 EOA 演进的账户，Keystore 中禁用 implicit EOA Actor，不能被概括为该私钥在所有普通交易与代码委托路径中都已失去能力。恢复与密钥迁移方案必须审查账户的全部入口。新创建、没有对应 EOA 私钥的合约账户是另一种起点。[^spec][^e7702]

### 6.3 配置 nonce 与交易 nonce 是两套机制

不要把以下三类标识混为一谈：

| 标识 | 管理什么 |
|---|---|
| `nonce_key / nonce_sequence` | 原生交易重放保护 |
| 配置 `channel / sequence / local_epoch` | 授权、撤权等配置消息的顺序与重放 |
| `operationId / orderId / orderVersion` | 产品操作、订单、部分成交与取消 |

配置有两条通道：

```text
Multichain:
  chain_id = 0
  单调 uint64 sequence
  没有 Local epoch，也没有 JIT unsequenced

Local:
  chain_id = 当前链
  sequence = local_epoch(高 32 位) | local_sequence(低 32 位)
```

Local 的低 32 位为 `uint32.max` 时是 unsequenced／JIT：不消费有序配置计数，可以在同一个 epoch 内重复提交。它方便提前签好授权，但也意味着需要额外退休机制。[^epoch][^changes]

### 6.4 一次完整的 Agent 撤权，不只是删除一个槽

设 Owner 曾签出 JIT 授权 `J`，把 K 授权为交易 Actor：

```text
T0：J 被提交，K 成为有效 Actor
T1：Owner 发现 K 失陷
T2：Owner 只提交 RevokeActor(K)
T3：旧 J 仍在同一个 Local epoch 内且授权尚有效
    → J 可能再次被提交，使 K 恢复
```

因此，完整本地退休通常需要考虑：

```text
RevokeActor(K)
    + IncrementLocalEpoch
    + 产品层的订单失效／策略停用
```

前两者可按合规配置批次组合；第三者属于业务状态机，是否与配置变更同一事务执行，需要定义。

`IncrementLocalEpoch` **不会删除已经生效的 Actor，不会自动失效 Multichain 批次，也不会自动取消已签出的 Perps 订单**。撤销管理员也不会自动沿授权谱系级联删除其曾授权的其他 Actor。[^epoch][^actors]

### 6.5 Account Lock 的作用与代价

Lock 是为了让节点更容易相信账户授权集合在一段时间内稳定，从而**可能**给它更高 pending 额度；不是锁定资产盈利，也不是协议保证高吞吐。

锁定时拒绝 Actor 授权／撤销及代码委托变更。规范的锁定章节保留 `Unlock` 和 `IncrementLocalEpoch` 两类操作；Unlock 开始延迟释放，不等于即时解锁。Lock／Unlock 都是管理员授权的独立配置变更批次。[^lock]

所以：

> 用户交易账户的即时撤权，和通过冻结权限来换取更高准入额度，存在真实取舍。

对于 Sponsor，只有“账户锁定＋可识别、限制原生资产外流的代码＋余额预留”等条件结合，才能支撑更稳定的付款能力；普通 Lock 不能单独保证余额不被其他交易搬走。[^validation][^repo]

**规范待澄清：**锁定章节允许 Unlock／Epoch，但当前原生 `account_changes` 的准入／执行流程还有“locked 则拒绝 config entries”的概括表述。实现需要明确两条入口的例外处理，并测试 `applySignedAccountChanges` 的 EVM 路径，不能仅凭概括流程断言所有锁定管理操作都能经原生配置字段提交。[^lock][^validation]

---

## 7. 二维 nonce 与 nonce-free：去掉哪些依赖，保留哪些依赖

### 7.1 有序多通道

```text
(account U, key 7)  → 40 → 41 → 42 ...
(account U, key 8)  → 12 → 13 → 14 ...
```

不同通道没有同一个计数器造成的序号依赖。同通道仍然有序；一个通道内没有兑现的前序操作，仍可能影响后序操作接纳或执行。

只读预编译：

```solidity
interface INonceManager {
    function getNonce(address account, uint256 nonceKey)
        external view returns (uint64);
}
```

地址为 `0x813000000000000000000000000000000000aa01`，写入由协议处理。[^nonce]

### 7.2 Actor 并不拥有自己的通道

受限 Actor 没有 `NONCE` 位时，只能使用 nonce-free。授予 `NONCE` 后，允许使用**完整有序空间，包括 key 0**，不是仅授予某个 key。

因此：

```text
SDK 约定：Agent A 用 key 7，Agent B 用 key 8
不等于
协议保证：A 无法消费 key 8
```

失陷的 Actor 可能干扰其他策略或 Owner 约定使用的通道。仅在 Manager 的执行阶段拒绝它，不会退回已消费的协议 nonce。资金或可用性必须强隔离的策略，应评估独立账户，不能只分配不同 key。[^nonce][^execution]

### 7.3 nonce-free 的条件

选择：

```text
nonce_key      = 2^256 - 1
nonce_sequence = 0
valid_before   = 非零、位于链允许的短有效期窗口内
```

协议不读取、递增有序 nonce 槽，而是利用：

```text
seen[replay_id] → valid_before
固定容量 ring  → 记录需要保留的 live entries
```

这是**所有节点一致维护的有界共识状态**，不是节点本地缓存。缓冲区槽位中的旧记录尚未过期时不能驱逐；满载会拒绝新 nonce-free 交易。

容量与窗口必须一起配置：

```text
capacity ≥ 峰值已接纳 nonce-free 速率 × 最大有效窗口
```

实际工程还需要为突发、排序、时间边界与重组恢复预留空间，并验证状态承诺、数据恢复和降级路径。[^nonce]

### 7.4 replay_id 与业务 ID 不相同

```text
replay_id = keccak256(
  0x7901 || RLP([
    chain_id,
    resolved_sender,
    valid_after,
    valid_before,
    account_changes,
    calls,
    metadata,
    payer
  ])
)
```

费用、`gas_limit`、认证数据不包含在内，所以纯加价可以保持逻辑身份；payer 或有效期改变则得到新 replay_id。

这既有优点，也有重要边界：

| 操作 | replay_id 是否变化 | 产品处理 |
|---|---|---|
| 相同交易提高费用 | 不变 | 仍需有效的新签名及替换规则 |
| 更换 Sponsor | 变化 | 旧交易不能视为自动取消 |
| 延长截止时间 | 变化 | 必须通过业务 ID 防止重复操作 |
| 重新签名、逻辑不变 | 不变 | 不能以完整交易 hash 代替去重键 |

**nonce-free 不等于订单没有 nonce、不等于旧订单不能成交，也不自动支持撤单优先。**订单版本、成交累计量和取消状态依然属于产品。[^replay]

---

## 8. Phase：原子性发生在哪一层

### 8.1 三条规则

```text
Phase 内：所有 call 共同成功，否则当前 Phase 回滚。
Phase 间：顺序执行，先前成功 Phase 保留。
某 Phase 失败：后续所有 Phase 跳过。
```

各 Phase 使用同一个用户侧 Gas 池，没有 8141 那样独立的逐 Frame execution/state 预算。[^execution]

### 8.2 例子：报销保留、买入回滚

仍使用 Policy-aware `U` 作为 Manager：

```text
Phase 0:
  U.executePolicy(向 P 报销 0.2 USDC)

Phase 1:
  U.executePolicy(USDC.approve(Launchpad, 100))
  U.executePolicy(Launchpad.buy(...))

Phase 2:
  U.executePolicy(后续允许动作)
```

假设 Phase 0 成功，Phase 1 的买入因滑点失败：

| 状态 | 结果 |
|---|---|
| 报销 0.2 USDC | 保留 |
| 本次 Phase 1 新设置的 allowance | 回滚到 Phase 1 前的值，不是强制变成零 |
| MEME 买入 | 没有发生 |
| Phase 2 | 跳过 |
| nonce／nonce-free 去重 | 仍消费／登记 |
| 真实 Gas | 仍支付 |
| 同笔已经应用的 Actor 授权／代码委托 | 不因业务 Phase 失败而自动撤销 |

如 Phase 0 报销失败，则 Phase 1 根本不执行。**但 Sponsor 仍可能已经承担这次失败尝试的 Gas，因此“可以保留成功报销”不等于“保证得到报销”。**[^execution][^receipt]

若使用 POLICY Actor，不能把第一步写成绕过 Manager 的顶层 `Token.transfer`。代币返回 `false` 也不一定触发 EVM revert；账户／Manager 必须检查代币调用结果。[^erc20]

### 8.3 Perps：撤旧单与挂新单有两种不同正确性目标

```text
方案 A：同一个 Phase
  [cancel(old), place(new)]
```

新单失败时撤单也回滚：适合“完整替换，否则维持原状”，不适合所有风险降低场景。

```text
方案 B：两个 Phase
  Phase 0 = [cancel(old)]
  Phase 1 = [place(new)]
```

撤单成功后，新单失败不会恢复旧报价。更适合“宁可暂时没有报价，也不要恢复过期报价”的策略。

这属于产品对失败结果的选择，不是协议替产品决定哪一种更好。

### 8.4 回执不能只看 status

8130 的回执扩展包含 `payer`、总体 `status` 与 `phaseStatuses`。总体 `status == 0` 表示有业务 Phase 失败，**不是整笔交易无副作用**。

SDK 应区分：

```text
交易未接纳／无效
交易已纳入，但业务失败
部分 Phase 已保留
账户授权变更已生效
业务操作已完成
当前状态仍可能被重组影响
```

例如首次 JIT 授权后买入失败，前端应显示“授权已生效，买入失败”，而不是提示用户重复创建同一授权。[^receipt]

---

## 9. Gas 计量：原稿之外必须补齐的付款方成本

### 9.1 成本分解

```text
intrinsic_gas =
  AA_BASE_COST
  + tx_payload_cost
  + nonce_key_cost
  + bytecode_cost
  + account_changes_cost
  + auto_delegation_cost
  + sender_auth_cost
  + payer_auth_cost

sender_intrinsic_gas = intrinsic_gas - payer_auth_cost
calls 可用 Gas       = gas_limit - sender_intrinsic_gas
```

当前文本中的默认组件包括 `AA_BASE_COST = 15,000`；有序 nonce 槽首次写入约按 22,100 计价，复用为 5,000，nonce-free 为 13,000。**这些是特定规范计价项，不是整笔交易费用，也不是 Mantle 实测性能。**L1 profile 与 L2 profile 的定价权限不同，见第 10 节。[^gas]

### 9.2 为什么 payer_auth 在 gas_limit 外

付款方可以选择自己的认证数据。如果该认证成本消耗用户签署的 `gas_limit`，Sponsor 就可能通过更昂贵的认证方式挤掉业务调用所需的 Gas。

因此，8130 将付款方认证及其载荷成本单独计量：

```text
用户 gas_limit：
  用户侧 intrinsic + calls

额外付款认证成本：
  payer_auth 执行与 payer_auth 字节费用
```

在使用独立付款认证的路径，实际总 `gasUsed` 可以超过 `gas_limit`。规范以 `MAX_AUTHENTICATION_GAS` 约束额外开销，给出：

```text
effective_gas_limit = gas_limit + MAX_AUTHENTICATION_GAS
```

无独立 payer auth 的普通自付路径，额外成本为零，文本规定有效上限等于 `gas_limit`。显式 `payer = sender`、使用独立 Gas key 的路径仍有付款认证成本，不应与“无 payer_auth 的自付”混淆。[^gas]

### 9.3 数字示例：只解释预算，不声称基准测试

假设：

```text
用户签署 gas_limit     = 300,000
sender_intrinsic_gas   = 60,000
则 calls 预算         = 240,000

本次 calls 实际使用   = 170,000
payer_auth 实际计量   = 12,000
```

暂不考虑数据 floor 进一步抬高结算值，则总计量约为：

```text
60,000 + 170,000 + 12,000 = 242,000
```

若 calls 使用满 240,000，总计量会是 312,000，而不是 300,000。Sponsor 服务应基于实际协议上限和本链费用组成预留，不能只计算 `gas_limit × fee`。这些数值是教学假设，不是从客户端测得。

### 9.4 Calldata floor 与 L2 费用

当前 8130 的规范结构还对序列化交易字节计算 calldata floor：

```text
payload_tokens = zero_bytes + 4 × nonzero_bytes

sender_metered_gas = max(
  sender_intrinsic_gas + execution_gas_used,
  sender_intrinsic_gas - tx_payload_cost + 10 × payload_tokens
)
```

payer auth 字节在独立费用项中接受相应 floor 约束。当前 8141 文本采用另一组依赖与 floor 公式，不能直接共用计算器。

在 Mantle 适配时，还应单独映射 L2 数据发布费用、费用资产与费率接口。Ethereum 规范中的 ETH 扣费不能未经定义就当成 Mantle 完整最终账单。[^gas]

---

## 10. Adoption Profiles、Mempool 与工程实现

### 10.1 两个 Profile

| 维度 | L1 Profile | L2 Profile |
|---|---|---|
| Canonical 验证 | 原生、固定成本 | 原生、固定成本 |
| 非 canonical 认证器 | 可接受受限 `STATICCALL` | 原生交易热路径不接受 |
| 默认 Gas schedule | 规范常量，变更走硬分叉 | 可采用链特定 schedule，但节点必须共识一致 |
| 算法扩展 | 规范／硬分叉演进 | 可按链的升级治理扩展标准集合 |
| 普通 EVM 中的自定义认证器 | 可以使用 | 仍可用于配置、恢复或其他传输路径 |

**L2 Profile 不是让每台节点随意改费用或认证集合。**只要影响交易有效性或状态转换，就必须属于统一的链规则。[^profiles]

8130 不引入新的通用 EVM 指令，不代表无需改链。新交易解码、原生认证、系统状态、nonce／去重、费用、回执、RPC、数据派生和验证路径，都需要集成。

### 10.2 公共准入不只是验签

节点还需要处理：

```text
有效期／Actor expiry
配置变更模拟及失效
nonce／replay_id
Sponsor 余额与 pending 暴露
费用替换与重新认证
账户锁定状态
非 canonical 认证器的额外状态依赖（若允许）
```

原稿强调的“收窄验证热路径”是合理的设计方向，但不能写成“只有一个槽变化才会让交易失效”。费用、余额、时间和配置依赖同样要处理。[^validation]

### 10.3 高吞吐 Sponsor 与用户账户分开设计

一个平台可能服务很多独立用户账户，却通过一个 Sponsor 付款。即便用户 nonce 互不冲突，Sponsor 也可能成为共同准入瓶颈。

可验证的设计方向是：采用可识别的付款账户、余额预留、受约束提现和分区配额，而不是为追求额度把所有用户账户长期锁定。参考仓库提供 high-rate payer 思路，但生产安全和 Mantle 准入收益仍需测试。[^repo][^validation]

### 10.4 可移植的是哪些部分

Keystore、认证器及账户配置可以作为合约能力移植到其他 EVM 链；8130 原生交易信封、上下文和 nonce 预编译不会自动出现。

当前默认账户不应被当成已经实现了所有 ERC-4337 备用入口。备用传输还需账户实现、签名封装和上下文适配，且保持完全相同的 Policy 安全边界。[^repo][^spec]

---

## 11. Perps 离线订单：交易 Actor 与订单 Actor 必须分开

默认 ERC-1271 包装接受 ADMIN 或 OPERATOR。`POLICY | NONCE` 的 Agent 不自动获得通用 `isValidSignature` 通过资格。

8130 提供 Actor-aware 签名验证，例如：

```text
validateSignature(account, hash, auth)
    → (actorId, scope)
```

产品获得身份后，还要检查订单对应的策略、市场、子账户、仓位方向、数量、expiry、取消版本及部分成交额度。**返回 Actor 不等于该 Actor 有权签署任意订单。**[^sign]

```text
链下：
Agent K 签用户订单 O

链上：
撮合者 R 发交易，结算 O
    → 当前原生交易 sender／Actor 属于 R
    → 结算合约必须另外验证 O 的用户与 Actor K
```

这时用户的原生 2D nonce 不一定处在每一次报价路径上。订单重放、取消和保证金预留仍是 Perps 系统职责，详见第三篇。

---

## 12. 对原稿的保留、修正与新增

| 原稿位置／主张 | 本文处理 |
|---|---|
| §3：Account／Actor／Authenticator 解耦 | 保留，并补充配置与业务权限分层 |
| §3.1：单槽 actor_config | 保留；补充 self-actor 例外与额外 Policy／账户状态 |
| §3.2：NONCE 允许非零有序通道 | 修正为包括 key 0 的完整有序空间 |
| §3.2：`POLICY \| NONCE` 享有独立通道 | 明确为多通道能力，不是 Actor 对通道的独占权 |
| §3.3：Manager 驱动账户执行 | 补齐 Manager 的实际资产执行权限与 self-call 风险 |
| §3.4：Epoch 不注销 live Actor | 保留，并新增 JIT 重新授权的完整退休示例 |
| §4.2：nonce-free 共识环形去重 | 保留；补充 Sponsor 切换、业务幂等及容量验收 |
| §4.3：业务失败保留报销 | 保留条件性结论，删除“保证报销”的强解释 |
| §4.4：Canonical 验证与 L1／L2 Profile | 保留，不据此推导吞吐或安全性数量级 |
| 原稿没有展开的成本 | 新增 payer-auth 预算隔离、预扣与实际结算 |
| 原稿没有完整展示的生命周期 | 新增 EOA 对照、Agent 代付买入、撤权及 Perps 撤改单 |
| 原稿生产选型倾向 | 只作为 PoC 假设；实际比较放在第三篇 |

### 实现前必须固定和验收

固定 EIP 文本、canonical contracts、客户端及验证程序版本；验证配置签名与地址派生、scope 组合、所有账户执行入口、付款预算、nonce 干扰、JIT 退休、锁定管理、报销失败、回执恢复和重组。

当前规范中有效 Gas 上限、预扣与退款的文字还需要通过测试向量统一：尤其应区分“为额外认证预留的上限”和“实际计量的付款认证成本”，不能把示意公式直接当成生产费用估算器。

**本篇结论：8130 标准化了自动化账户的一组重要基础能力，但完整受限 Agent 仍是“协议＋Policy-aware 账户／产品＋Sponsor＋提交与恢复服务”的组合。**

---

## 来源

[^intro]: 用户提供：《EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析》，2026-09，重点为 §3、§4、§5。原稿列出的 EIP-8130 commit 为 `16390e1f`；本文保留来源归属，不声称已复现该版本。
[^spec]: [EIP-8130: Keystore Accounts](https://eips.ethereum.org/EIPS/eip-8130)，Overview、Specification、Portability，访问于 2026-09-24；状态 Draft。EIP 文本采用 CC0。
[^repo]: [base/eip-8130 canonical contracts repository](https://github.com/base/eip-8130)，README、Contracts、Import 与生产风险提示，访问于 2026-09-24。本文未运行其测试或安全审计。
[^e1559]: [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)，交易信封、验证与费用结算。
[^e7702]: [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)，EOA 代码委托；此引用用于界定基线，不表示 Mantle 当前部署状态。
[^erc20]: [ERC-20](https://eips.ethereum.org/EIPS/eip-20)，approve／transfer 与调用方处理 `false` 返回值的要求。
[^e8141]: [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)，Transaction Signatures。
[^actors]: [EIP-8130 — Keystore and Account Configuration](https://eips.ethereum.org/EIPS/eip-8130#keystore-and-account-configuration)，尤其 Actor Expiry 与授权不沿谱系级联的说明。
[^auth]: [EIP-8130 — Authenticators](https://eips.ethereum.org/EIPS/eip-8130#authenticators)。
[^storage]: [EIP-8130 — Storage Layout](https://eips.ethereum.org/EIPS/eip-8130#storage-layout)；精确实现以 canonical repository 为准。
[^scope]: [EIP-8130 — Actor Scope](https://eips.ethereum.org/EIPS/eip-8130#actor-scope)。
[^policy]: [EIP-8130 — Actor Policies](https://eips.ethereum.org/EIPS/eip-8130#actor-policies)。
[^tx]: [EIP-8130 — AA Transaction Type](https://eips.ethereum.org/EIPS/eip-8130#aa-transaction-type)，Field Definitions、Signature Format、Timestamp Normalization。
[^sign]: [EIP-8130 — Signature Verification](https://eips.ethereum.org/EIPS/eip-8130#signature-verification) 与 [Signature Payload](https://eips.ethereum.org/EIPS/eip-8130#signature-payload)。
[^payer]: [EIP-8130 — Payer Modes](https://eips.ethereum.org/EIPS/eip-8130#payer-modes)。
[^changes]: [EIP-8130 — Account Changes](https://eips.ethereum.org/EIPS/eip-8130#account-changes)，Create、Config Change Authorization、Delegation。
[^epoch]: [EIP-8130 — Epoch System](https://eips.ethereum.org/EIPS/eip-8130#epoch-system)。
[^lock]: [EIP-8130 — Account Lock](https://eips.ethereum.org/EIPS/eip-8130#account-lock)。
[^nonce]: [EIP-8130 — 2D Nonce Storage](https://eips.ethereum.org/EIPS/eip-8130#2d-nonce-storage)，Actor Nonce Scope、Nonce-Free Mode。
[^replay]: [EIP-8130 — Replay Identifier](https://eips.ethereum.org/EIPS/eip-8130#replay-identifier) 与 Mempool Replacement。
[^execution]: [EIP-8130 — Execution](https://eips.ethereum.org/EIPS/eip-8130#execution)，Call Execution、Call Phases、Transaction Context。
[^receipt]: [EIP-8130 — RPC Extensions](https://eips.ethereum.org/EIPS/eip-8130#rpc-extensions) 与 Block Execution。
[^gas]: [EIP-8130 — Intrinsic Gas](https://eips.ethereum.org/EIPS/eip-8130#intrinsic-gas)。
[^profiles]: [EIP-8130 — Adoption Profiles](https://eips.ethereum.org/EIPS/eip-8130#adoption-profiles)。
[^validation]: [EIP-8130 — Validation Flow](https://eips.ethereum.org/EIPS/eip-8130#validation-flow)，Mempool Acceptance、Block Execution。
[^security]: [EIP-8130 — Security Considerations](https://eips.ethereum.org/EIPS/eip-8130#security-considerations)。
