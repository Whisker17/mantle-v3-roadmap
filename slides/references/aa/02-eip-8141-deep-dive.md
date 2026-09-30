---
title: "EIP-8141 Deep Dive：Frame Transaction、授权状态机与交易生命周期"
description: "通过普通交易与 Frame 交易的逐步对照，解析验证、付款、执行、原子组、双维 Gas、公共准入及 EIP-8250。"
last_verified: "2026-09-24"
status: "基于 Draft 规范的研究文档，不是生产实现或性能报告"
---

# EIP-8141 Deep Dive：Frame Transaction、授权状态机与交易生命周期

> **核心问题：一笔交易能否先验证操作权限，再确定付款人，最后按明确边界执行一组动作？**
>
> EIP-8141 的答案是把交易组织为一串 Frame，并通过 `APPROVE` 显式批准执行与付款。它抽象的是交易处理流程，不是规定所有账户必须使用同一套角色、Session key 或策略存储。[^spec]

**系列导航：** 另见 [8130 Deep Dive](01-eip-8130-deep-dive.md) 与 [Mantle 产品能力覆盖与适配评估](03-mantle-aa-coverage-and-fit.md)。

**阅读建议：** 首次阅读先看 §1～§3 的 EOA 与 Agent 生命周期；§4～§8 解释信封、批准状态机和回滚；§9～§11 讨论费用、准入及 8250；§12～§13 汇总接入与原稿修正。

## 阅读口径与材料来源

本文主要依据用户提供的《EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析》**§1、§2 及 §5 的相关内容**，保留交易信封、Frame 模式、`APPROVE`、内省指令、双维 Gas、原子组和 Mempool 等组织层次。[^intro]

新增部分包括：普通交易与 8141 交易生命周期对照；Agent 代付买入；每一步的批准状态、nonce 和扣费；失败与回执解读；EIP-8250 的精确能力边界。

机制按 **2026 年 9 月 24 日访问的官方 Draft 文本**核对。原稿标注 8141 commit 为 `b75cbe61`，本文没有运行该 commit 的实现，也不声称当前网页与其逐字一致。8130／8141 原稿之外的修正与设计建议，在文末单独列出。

以下示例中的 `GuardedAccount8141`、`executePolicy`、`orderVersion` 等是**教学设计，不是标准 ABI 或 Mantle 已部署能力**。固定数字除规范常量外均为示意，不是性能测试结果。

---

## 1. 先从普通 EOA 买入说起

### 1.1 一笔普通交易的生命周期

Alice 使用账户 `U`，向 Launchpad 买入 100 USDC 的 MEME。假设 allowance 已经存在：

```text
① 生成 buy calldata
        ↓
② 读取 U 的 nonce，设置 Gas 和费用
        ↓
③ U 的私钥签署普通交易
        ↓
④ 节点校验签名、nonce、费用与余额
        ↓
⑤ 排序并纳入区块
        ↓
⑥ 以 U 为调用者执行 buy，U 支付 Gas
        ↓
⑦ 业务成功或回滚；结算实际费用
        ↓
⑧ 应用解析回执、余额与业务事件
```

该基线是普通 EOA 交易，不包括 ERC-4337、EIP-7702 钱包、Permit 或应用专用元交易。那些方案也能提供部分批处理和代付能力；这里仅用于看清 8141 改变了哪一层。[^e1559][^e7702]

若 allowance 不足，则普通路径可能是：

```text
交易 n：approve(Launchpad, 100)
交易 n+1：buy(...)
```

`approve` 成功后，后面的独立买入交易失败，不会把前一笔 allowance 回滚。[^erc20]

### 1.2 8141 不再把授权隐藏在一个外层 EOA 签名里

8141 把处理过程变成：

```text
验证签名材料
        ↓
执行一个或多个验证 Frame
        ↓
显式 APPROVE_EXECUTION
        ↓
显式 APPROVE_PAYMENT
        ↓
执行业务 Frames
        ↓
按原子组回滚／提交，结算费用
```

操作账户与付款人可以不同。对于已有 EOA，协议还提供 default-code 逻辑，使其不部署新账户合约，也能使用基本 Frame 批处理和独立付款机制。**这不等于 EOA 默认已经拥有受限 Session keys。**[^behavior][^default]

---

## 2. 最小完整示例：EOA 原子 approve＋buy

### 2.1 换成一笔 Frame 交易

假设 U 已存在并持有足够原生 Gas 资产，使用普通 EOA 密钥，没有自定义账户代码：

```text
type       = 0x06
chain_id   = C
nonce      = n
sender     = U

signatures = [
  [SECP256K1, U, empty_msg, U 对 canonical tx hash 的签名]
]

frames:
  F0: VERIFY, flags=0x3, target=null, value=0
      → default code → APPROVE_EXECUTION_AND_PAYMENT

  F1: SENDER, flags=0x4, target=USDC, value=0
      → approve(Launchpad, 100)

  F2: SENDER, flags=0x0, target=Launchpad, value=0
      → buy(100, minOut, recipient=U)
```

`target=null` 解析为 `tx.sender`，不是零地址。F1 的 `0x4` 表示“与后面的 Frame 组成原子组”，F2 不带该标志，结束这一组。

`APPROVE` 是协议指令；`USDC.approve` 是代币 allowance 函数，二者没有相同权限语义。[^tx][^batch]

### 2.2 一步步跟踪状态

| 时点 | `sender_approved` | `payer` | nonce | 业务状态 |
|---|---:|---|---|---|
| 开始 | false | None | n | 未变化 |
| 外层签名校验通过 | false | None | n | 未变化 |
| F0 default code 校验 U 的签名 | false | None | n | 未变化 |
| F0 执行 `APPROVE(0x3)` | true | U | n+1 | U 预付最大费用 |
| F1 approve 成功 | true | U | n+1 | 原子组内暂时产生 allowance |
| F2 buy 成功 | true | U | n+1 | 原子组业务效果保留 |
| 结算 | true | U | n+1 | 退还未计费部分，输出逐帧回执 |

如果 F2 买入失败，F1 与 F2 的业务状态一起回滚；F0 成功批准后产生的 nonce 和 Gas 费用不随业务组回滚。

**准确表述是“一个业务原子组”，不是“整笔交易的所有效果都原子回滚”。**[^approve][^behavior]

### 2.3 与原本流程的比较

| 维度 | 普通两笔交易 | 本例 8141 |
|---|---|---|
| 用户确认 | 通常两笔交易签名 | 一份覆盖完整 Frame 交易的签名 |
| nonce | 消费两个普通交易 nonce | 本例消费一次交易 nonce |
| 调用者 | 两次均为 U | 两个 SENDER Frame 均为 U |
| buy 失败 | 已成功 approve 保留 | 同原子组 approve 回滚到组前状态 |
| Gas | 两笔各自结算 | 一笔按协议规则与逐 Frame 预算结算 |
| 是否已有受限 Agent | 没有 | 仍没有；需要自定义账户授权逻辑 |

这个最小例子先解释 Frame 的价值；下一节再加入 Session key 和 Sponsor。

---

## 3. 完整 Agent 示例：受限授权、代付、报销与买入

### 3.1 前置条件与角色

以下为**设计示例**：

```text
U：用户的 Policy-aware 智能账户
K：Owner 授予的 Agent Session key
P：Sponsor／Paymaster
T：USDC
L：允许的 Launchpad
```

Owner 已在 U 的账户存储中安装 Session policy：

```text
K 可以：
  在 L 买入指定资产
  每笔最多使用 100 USDC
  所得资产只能进入 U
  使用约定的费用与授权额度

K 不可以：
  提现
  改 Owner
  升级账户
  调用任意外部合约
```

**这套 policy 是账户实现，不是 8141 原生角色表。**静态权限和签名授权可在 VERIFY 检查；累计预算、业务余额及市场动态约束通常还需要执行阶段检查。[^security][^mempool]

为避免把交易 expiry 与 Session expiry 混为一谈，验证器需要确认：本次交易的截止时间不晚于该 Session 被授权的到期时间。仅由 Agent 自己填一个 expiry，不构成 Owner 的期限限制。

### 3.2 示例 Frames

设 U 已安装代码、K 已授权。本例不包含账户部署：

| Frame | Mode | Flags | Target | 职责 |
|---|---|---:|---|---|
| F0 | VERIFY | `0x0` | `EXPIRY_VERIFIER` | 检查本次交易截止时间 |
| F1 | VERIFY | `0x2` | U | 验证 K、Session policy、完整 Frame 集合；批准执行 |
| F2 | VERIFY | `0x1` | P | 审核付款授权和费用；批准付款 |
| F3 | SENDER | `0x0` | U | `executePolicy`：向 P 支付约定代币报销 |
| F4 | SENDER | `0x4` | U | `executePolicy`：设置所需 allowance |
| F5 | SENDER | `0x0` | U | `executePolicy`：执行买入 |
| F6 | DEFAULT | `0x0` | P | 可选后处理／记录费用结果 |

F4 与 F5 是业务原子组。F3 不在该组内，因此**F3 已经成功提交的报销**不会因为 F5 失败而回滚。F6 作为组外后处理，可以在业务组失败后继续执行。[^batch][^examples]

账户 U 的所有可被 SENDER 模式触达的执行入口都必须安全。尤其 `msg.sender == U` 不应成为绕过 Session policy 的充分条件；F1 必须检查后续全部 SENDER Frames，执行入口仍需检查动态预算。

### 3.3 新生命周期

```text
Owner 预先配置 Session policy
        ↓
Agent 生成完整业务意图及 operationId
        ↓
SDK 固定全部 Frames、费用、expiry、签名元数据
        ↓
Agent／Sponsor 按约定签署相应授权
        ↓
协议先验证 signatures 中的标准签名
        ↓
公共准入模拟 F0～F2，检查可接受性与付款余额预留
        ↓
入块后按实际前状态重新执行验证前缀
        ↓
F1：APPROVE_EXECUTION
F2：APPROVE_PAYMENT → nonce 消费、P 预扣费用
        ↓
F3：报销
F4～F5：原子业务组
F6：组外后处理
        ↓
实际费用结算＋逐帧回执＋业务状态确认
```

公共池的具体 Sponsor 类型、代码识别与状态访问还要符合第 10 节。不能因为这组 Frames 在共识执行上表达得出来，就宣称它必然被任意公共节点传播。

### 3.4 失败路径必须逐个解释

| 失败点 | 是否是可正常入块的业务失败 | nonce／费用 | 后续执行 |
|---|---|---|---|
| 外层标准签名无效 | 否，交易无效 | 不产生有效链上交易的扣费与 nonce 结果 | 不执行 Frames |
| F0 过期 | 否，VERIFY 失败使交易无效 | 交易效果全部撤销 | 不继续作为有效交易执行 |
| F1 Session 验证失败 | 否，交易无效 | 同上 | 同上 |
| F2 付款批准失败 | 否，交易无效 | 前面临时批准不能保留为链上结果 | 同上 |
| F3 报销失败 | 可以是有效交易中的执行失败 | P 仍承担实际 Gas，nonce 已消费 | 独立 F4／F5 不会仅因 F3 失败自动停止 |
| F5 买入失败 | 是 | nonce 与费用保留 | F4／F5 回滚，组外 F6 可继续 |
| F6 后处理失败 | 是 | 仍结算实际费用 | 前面独立成功状态不自动回滚 |

这里最容易漏掉的是 F3：

> **“先把报销 Frame 排在前面”不等于“报销失败后自动停止业务”。**

业务必须需要报销成功时，账户或业务执行入口应显式检查对应已完成 Frame／原子组的结果，或在共同的受控调用中强制该前置条件。只检查“某个 Frame 没有 revert”还不一定证明 ERC-20 支付成功；必须处理 `false` 返回值以及实际到账语义。[^behavior][^erc20]

---

## 4. 交易信封与 signatures：8141 不是“所有验签都在 EVM”

### 4.1 完整载荷

```text
0x06 || RLP([
  chain_id,
  nonce,
  sender,
  frames,
  signatures,
  fees,
  blob_versioned_hashes
])

frames     = [[mode, flags, target, limits, value, data], ...]
limits     = [execution_gas_limit, state_gas_limit]
signatures = [[scheme, signer, msg, signature], ...]
fees       = [max_priority_fee_per_gas,
              max_fee_per_gas,
              max_fee_per_blob_gas]
```

当前 8141 **本体**的 `nonce` 是 `uint64` 标量；EIP-8250 会替换该字段，第 11 节单独展开，不能把两份规范混成一个既有功能。[^tx]

### 4.2 重要静态约束

Frames 数量为 1～64。mode 为 DEFAULT、VERIFY、SENDER 三种；flags 仅使用低三位。只有 SENDER 可以携带非零 `value`。

凡批准执行的 Frame，其 target 必须是 sender 或 null。原子组不能包含 VERIFY；组内所有 Frame 都不允许审批 scope，即终止帧也不能趁未带 `0x4` 而附带付款批准。最后一个 Frame 不得设置继续分组标志。[^constraints]

### 4.3 协议级签名集合

| Scheme | 编码 | 当前验证成本 | 含义 |
|---|---|---:|---|
| `0x0 ARBITRARY` | 任意字节 | 100 Gas 的结构性项 | 见证容器；密码学正确性由账户逻辑验证 |
| `0x1 SECP256K1` | `v \|\| r \|\| s`，65 字节 | 2,800 | 协议在 Frame 前验证 |
| `0x2 P256` | `r \|\| s \|\| qx \|\| qy`，128 字节 | 6,700 | 协议在 Frame 前验证 |

标准签名 `signer` 为空时，默认解析为 `tx.sender`；否则为显式 20 字节地址。P256 的 signer 地址是 `keccak256(qx || qy)[12:]`。

对 Agent 来说，K 通常不是 U，所以应显式表达 K 的签名身份，不能在签名元数据中无意使用“默认 sender”。[^signatures]

**P256 支持不等于已经完成 WebAuthn／Passkey 认证。**WebAuthn 的 challenge、authenticator data、origin 等语义，需要相应账户验证逻辑；不能把所有 Passkey 流程都当成对交易 hash 的裸 P256 签名。

### 4.4 Canonical signature hash

```text
对 signatures 中每个 msg 为空的条目：
    将其 raw signature bytes 置空用于哈希计算

sig_hash = keccak256(0x06 || RLP(上述规范化交易))
```

这么做是为了避免“签名的哈希包含签名自身”的循环。显式 32 字节 `msg` 的签名不造成这一循环，其原始签名字节会被 canonical transaction hash 绑定。

`msg` 只允许空或非零的 32 字节摘要，显式全零摘要无效。[^sighash]

因此：

```text
密码学签名有效
≠ 已经批准账户执行
≠ Agent 的业务权限允许交易中的全部动作
```

验证器仍需检查 signer 属于哪个角色、签了哪个消息、该消息绑定哪些交易字段，以及权限范围是否覆盖全部动作。

### 4.5 自定义签名的 witness 放在哪里

当自定义签名签的是 canonical transaction hash，见证应放进 `ARBITRARY` signature entry，而不是把签名字节直接塞进被签的 Frame data，造成自引用。

标准 SECP256K1／P256 的 raw signature 不向 EVM 暴露，合约读取其验证后的元数据。`ARBITRARY` 的原始字节可由 `SIGDATACOPY` 获取并自行验证；100 Gas 不是任意密码学算法的总成本。[^signatures]

---

## 5. 三种 Frame 模式与上下文

| Mode | 数值 | 顶层 caller | ORIGIN | value | 主要用途 |
|---|---:|---|---|---|---|
| DEFAULT | 0 | `ENTRY_POINT = address(0xaa)` | 同 caller | 必须为 0 | 部署、辅助动作、后处理 |
| VERIFY | 1 | `ENTRY_POINT` | 同 caller | 必须为 0 | STATICCALL 验证，允许 APPROVE 的特例效果 |
| SENDER | 2 | `tx.sender` | 同 caller | 可以非零 | 以用户身份执行操作 |

ORIGIN 在该 Frame 的所有调用深度内保持对应的 Frame caller。普通内部 CALL 的 `msg.sender` 仍按调用链变化，不能把顶层 caller 规则推广为所有下游合约都看到同一个 sender。[^behavior]

两个交易级状态是：

```text
sender_approved = false
payer           = None
```

SENDER Frame 开始时，`sender_approved` 必须已为 true；否则交易无效。执行完所有 Frames，若没有 payer，交易也无效。每笔交易只有一个最终付款账户。[^behavior][^approve]

**DEFAULT／VERIFY 中的 `address(0xaa)` 是协议入口身份，不是某个代替用户持有资产的 ERC-4337 Bundler。**

---

## 6. APPROVE：把授权变成明确的状态转换

### 6.1 操作码与堆栈

```text
opcode = 0xaa

stack top:
  offset
  length
  scope
```

它像 RETURN 一样成功退出当前 EVM 调用，并返回指定内存片段；同时更新交易批准状态。它不是一个执行后还继续运行下一行的普通函数调用。[^approve]

| scope | 含义 |
|---|---|
| `0x0` | 无可批准权限；不能执行 `APPROVE(0)` |
| `0x1` | 付款批准 |
| `0x2` | 执行批准 |
| `0x3` | 同时批准执行与付款 |

### 6.2 执行批准

检查的关键条件：

```text
当前交易必须是 Frame transaction
当前 ADDRESS 必须等于 resolved_target
scope 必须是 flags 允许的非零子集
批准执行的 resolved_target 必须等于 sender
sender_approved 尚未设置
```

成功后把 `sender_approved` 置 true。

“ADDRESS 等于 target”描述的是执行上下文，不能简化成“任意下游 helper 都能批准自己的 caller 代付”。使用库、DELEGATECALL 或账户模块时，也必须理解真正承担批准的是哪个账户上下文。[^approve]

### 6.3 付款批准

付款批准要求 sender 已经批准执行，payer 尚未设置，并有足够余额覆盖 `max_cost`。

成功时：

```text
必要时为 sender 首次存在分配 state gas
    ↓
消费 sender nonce
    ↓
payer = 当前批准账户
    ↓
预扣 max_cost
```

同时批准 `0x3` 则在一条一致的批准转换中完成这些步骤。若必要的状态预算不足，不应留下“执行已批准但付款半完成”的半状态。[^approve]

**nonce 不是在入口验证时立即消费，而是在成功付款批准时消费。**不过，当交易最终因 VERIFY 失败而被判无效，所有临时效果都必须撤销。

### 6.4 最大权限陷阱：批准覆盖后续全部 SENDER Frames

F1 验证器不能只检查一个允许的买入 Frame，然后就批准：

```text
F1：看到 F4 是允许的 buy → APPROVE_EXECUTION
F4：buy
F5：未检查的 transfer / upgrade / arbitrary execute
```

一旦批准，后面的所有 SENDER Frames 都能以 U 的身份发起顶层调用。因此必须检查：

```text
全部 SENDER targets 与 calldata
所有 value
账户自身 execute／upgrade 入口
预算、接收人、token 和 allowed selectors
费用、expiry、nonce 与签名域
相关 DEFAULT 辅助动作是否改变依赖或后置结果
```

Owner 亲自签署完整交易与受限 Agent 签署完整交易，是不同的授权模型。Agent 签得再完整，也不能代替 Owner 对该 Agent 的限制。[^security]

---

## 7. 内省指令：账户如何看见完整交易

### 7.1 指令集合

| 指令 | 操作码 | 主要功能 |
|---|---:|---|
| `TXPARAM` | `0xb0` | 交易级字段、签名哈希、费用上限、Frame 数量等 |
| `FRAMEDATALOAD` | `0xb1` | 从指定 Frame data 读取 32 字节 |
| `FRAMEDATACOPY` | `0xb2` | 拷贝指定 Frame data |
| `FRAMEPARAM` | `0xb3` | Frame target、mode、flags、value、预算和已完成状态 |
| `SIGPARAM` | `0xb4` | 签名方案、验证后 signer、msg 等 |
| `SIGDATACOPY` | `0xb5` | 读取 ARBITRARY raw witness |

这些指令只在 Frame 交易上下文有效；在其他交易类型执行会 exceptional halt。因此，一个同时支持普通交易／其他 AA 的账户需要显式分流，不应在通用入口无条件调用这些指令。[^introspection]

### 7.2 最有用的 TXPARAM 字段

| 参数 | 返回值 |
|---|---|
| `0x01` | 8141 本体的 nonce |
| `0x02` | sender |
| `0x06` | 最大交易费用 |
| `0x08` | canonical transaction signature hash |
| `0x09` | Frame 数量 |
| `0x0A` | 当前 Frame index |
| `0x0B` | 签名数量 |
| `0x0C` | 当前 Frame 剩余 state gas |

8250 在不冲突的索引中增加 nonce-key 相关内省，见第 11 节。

### 7.3 策略验证示意

以下是**逻辑伪代码**，不是 Solidity 可直接使用的库接口：

```text
validateSession():
    sessionSigner = 已验证的签名元数据
    session = 读取本账户的 session storage

    require signer 有效且未撤销
    require 签名绑定当前 canonical hash 或等价的完整授权对象
    require 交易 expiry 不晚于 session expiry
    require max_cost 不超过授权的单笔费用边界

    for 每个 Frame:
        检查 mode、target、flags、value、data
        拒绝不在允许模板中的 SENDER 动作
        检查 account.execute 等间接调用的内部 actions
        检查是否存在绕过权限的 upgrade、delegatecall、转移接收人
        检查批准／原子组结构符合预期

    APPROVE_EXECUTION
```

VERIFY 是只读路径，不能在其中随意写入“本日额度已消费”。预算累计的状态更新，要放在可执行写入的阶段，并正确处理并发与回滚。

### 7.4 查询已完成 Frame 不等于预测未来

`FRAMEPARAM` 可以读取过去 Frame 的 status／Gas；查询当前或未来 Frame 的结果会异常。

即使读取过去 Frame 的成功状态，也要结合其原子组是否后来回滚；第 8 节给出具体例子。不要用“F3 返回成功”作为不看上下文的永久到账证明。[^introspection][^receipt]

---

## 8. 原子组、逐帧失败与回执

### 8.1 原子组的精确定义

连续 Frames `[i, j]` 构成原子组：

```text
F_i ... F_(j-1)：设置 ATOMIC_BATCH_FLAG = 0x4
F_j：不设置 ATOMIC_BATCH_FLAG
```

组内任何一个 Frame 失败，组内此前业务效果回滚，组内尚未执行的 Frames 标为 skipped。**组外的后续独立 Frame 可以继续执行。**[^batch]

| 序列 | 结果 |
|---|---|
| flags `4, 0` | 一个两 Frame 原子组 |
| flags `4, 4, 0` | 一个三 Frame 原子组 |
| flags `0, 0` | 两个独立 Frame，不是原子组 |
| 最后一个 Frame flags `4` | 结构无效 |

### 8.2 业务组回滚，不会重开之前 Frame 的 Gas 预算

```text
F0 VERIFY：批准并付款
F1 SENDER：独立报销
F2 SENDER flag=4：approve
F3 SENDER flag=4：buy（失败）
F4 SENDER flag=0：后续组内动作（跳过）
F5 DEFAULT：组外后处理（可执行）
```

F2/F3/F4 的业务状态回到组开始之前。F0 的批准和付款、F1 的成功报销保留。F5 可以检查结果并处理失败，但没有权利凭空恢复已消耗的 Gas。

### 8.3 一个很重要的回执细节

当前规范的逐帧回执：

```text
ReceiptPayload = [
  cumulative_gas_used,
  payer,
  [frame_receipt, ...]
]

frame_receipt = [
  status,
  [execution_gas_used, state_gas_used],
  logs
]
```

没有原生的**交易级 status 字段**。兼容接口需要自己派生，不能假设某个 UI 的单一状态就是完整语义。[^receipt]

更细的一点是：**原子组回滚后，已经执行的 Frame 保留其执行状态与 execution gas；日志被清除，相关 state gas 回滚。**

以 F2 成功、F3 失败、F4 跳过为例：

| Frame | 回执 status | 状态是否最终保留 |
|---|---:|---|
| F2 approve | 1 | 不保留；所属原子组回滚 |
| F3 buy | 0 | 不保留 |
| F4 后续动作 | 2 | 未执行 |

所以 SDK 不能看到 F2 的 `status=1` 就告诉用户“allowance 已增加”。正确做法是根据 Frame flags 重建原子组、检查组的最终结果，并结合最终业务事件／状态。[^behavior][^receipt]

### 8.4 与普通 EVM 交易不完全相同的跨 Frame 状态

当前规范：

```text
warm / cold 访问记录跨 Frame 共享，并按回滚 journal 恢复。
TSTORE / TLOAD transient storage 在 Frame 之间清空。
state-gas 归属与 refill 随相应状态变更记录和回滚。
```

因此，不能用 F1 的 transient storage 向 F2 传递一个“已验证”标记。依赖临时存储的组合协议或防重入逻辑，应优先放进同一 Frame 的内部调用链，并针对真实上下文测试。[^crossframe]

### 8.5 对 Perps 撤改单的含义

```text
同组 [cancel, place]：
  place 失败 → cancel 也回滚

独立 cancel Frame + 独立 place Frame：
  place 失败 → 成功 cancel 保留
  cancel 失败 → place 不会自动停止
```

第二种要避免“撤单失败但仍继续挂新单”的双重暴露，就需要新单入口显式检查撤单结果或对应订单状态。与 8130 “前一 Phase 失败即停止后续 Phase”的行为不同，8141 的失败后续逻辑要主动编排。

---

## 9. 双维 Gas：执行预算与状态预算不是同一件事

### 9.1 每个 Frame 两个独立预算

```text
limits.execution：计算、数据和账户访问等执行预算
limits.state：持久状态增长预算
```

一个 Frame 的 state gas 不能拿来补 execution gas；前一个 Frame 没用完的预算也不能自动转给后一个 Frame。

例如某 Frame 还剩 50,000 state gas，但 execution gas 用完了，仍会异常。若 F1 剩余大量执行预算，而 F2 只声明 10,000 却需要 20,000，F2 仍可能失败。[^gas]

### 9.2 状态增长的成本

当前依赖参数：

```text
CPSB = 1,530
STATE_BYTES_PER_NEW_ACCOUNT = 120

创建新账户状态：
120 × 1,530 = 183,600 state gas
```

付款批准消费 nonce 时，如果使 sender 首次成为存在的账户，可能需要这一状态预算；SENDER value 转移到不存在的目标，也可能触发目标账户创建成本。

这意味着“无代码 EOA 不需要部署合约”不等于“首次使用没有状态成本”。[^gas][^approve]

### 9.3 Intrinsic、数据成本与 floor

当前规范计算：

```text
signature_verification_cost = Σ signature_gas(sig)

signature_data_cost =
  Σ [
    calldata_cost(sig.signer)
    + calldata_cost(sig.msg)
    + calldata_cost(sig.signature)
  ]

frame_tx_intrinsic_gas =
  12,000
  + 475 × Frame 数量
  + frame_data_cost
  + signature_data_cost
  + signature_verification_cost
  + value_transfer_cost
```

原稿的示意公式只统计了 signature bytes；生产估算还必须包括 signer 与 msg。

第 2 节的“三个 Frames＋一个 k1 签名”，仅固定部分就是：

```text
12,000 + 3 × 475 + 2,800 = 16,225
```

这**不包含数据、业务执行、状态增长及其他费用**，不能当成该买入交易的 Gas。

当前普通 calldata 计价使用 `zero_bytes + 4 × nonzero_bytes`；floor 的计数函数则是 `4 × len(data)`，对应系数为 16。不能把原稿的概念式 floor 直接复制成实际估算器。[^gas]

### 9.4 最大预算

```text
standard_gas_limit =
  intrinsic
  + Σ frame.execution_limit
  + Σ frame.state_limit

max_gas = max(
  standard_gas_limit,
  calldata_floor_gas + Σ frame.state_limit
)
```

付款人预扣 `max_cost`。若携带 blob，当前规范还计入本区块 blob base fee；`max_fee_per_blob_gas` 用于纳入检查。默认交易可以不带 blob。[^gas]

当前规范的双维预算不意味着不同应用有独立 basefee，也不保证 Perps 不受 Launchpad 热点影响。**Gas 预算隔离、应用资源隔离、局部费用市场是不同层次。**

### 9.5 Refill、Refund 与回滚

| 机制 | 含义 | 是否给后续 Frame 增加可用预算 |
|---|---|---|
| 未使用 Gas | 该 Frame 没有用到的预算 | 否，只影响最终收费 |
| State-gas refill | 撤回本交易内未形成持久状态的状态收费 | 否 |
| Storage refund | 按对应规则减少最终付费，存在上限 | 否 |
| 原子组回滚 | 撤销组内状态及状态费用归属 | 否；实际执行工作仍计费 |

后续 Frame 清理本交易早先新建的存储槽，可能降低较早 Frame 的最终 `gas_used.state`。当前 EIP-8037 语义下，`SELFDESTRUCT` 不产生该 state-gas refill。

逐 Frame 的两种 Gas 用量之和，也不一定等于交易最终 `gas_used`：intrinsic 在 Frame 预算之外，storage refund 和 calldata floor 在交易层应用。[^gas]

### 9.6 付款收费与区块容量又是两个口径

当前规范与 EIP-7778 集成，执行维度的区块容量不因 storage refund 而释放；用户最终支付可以受 refund 影响。state refill 则撤销的是没有形成持久状态的收费。

因此，评估 Mantle 时至少要区分：

```text
用户支付
Sponsor 最大预留
节点执行工作
持久状态增长
区块资源预留
数据发布／验证成本
```

不能只看用户回执里的最终 Gas，就宣称节点资源占用减少了相同比例。[^gas]

---

## 10. 公共 Mempool：可编程验证为什么仍然受约束

### 10.1 共识可执行与公共传播不是同一件事

8141 允许自定义验证逻辑，但公共 Mempool 必须限制验证工作和易失效依赖。否则大量交易可能因一个外部状态变化失效，迫使节点反复进行昂贵模拟。

不满足公共池规则的交易，可能通过本地／私有池提交，但仍必须满足共识有效性。这是一条部署路径选择，不是绕过协议。[^mempool]

### 10.2 Validation prefix

验证前缀是：从第一个 Frame 开始，到成功确定 payer 的最短前缀。

公共池主要识别：

```text
[self_verify]
[deploy, self_verify]
[only_verify, pay]
[deploy, only_verify, pay]
```

还允许按规则插入作为首帧的 `expiry_verify`；匹配前缀形状时跳过该到期验证帧。正文关于部署必须位于首位的表述，在与 expiry 同时使用时应由固定版本实现明确解释与测试。[^mempool]

前缀之后可有复杂业务与后处理，**但公共池不允许其后再出现 VERIFY Frame**：后续 VERIFY 若失败，会使整笔交易无效，付款已批准并不能为无效交易兜底。

### 10.3 验证预算

```text
Σ prefix.execution_limits
  + 外层 signatures 的验证成本
  ≤ 100,000

Σ prefix.state_limits
  ≤ 500,000
```

这里主要约束是声明的前缀预算总和及验证成本，不只是某次模拟恰好少用了一点 Gas。

state gas 上限限制可准入状态增长；计算验证工作主要仍受 execution 预算约束。自定义证明、复杂 Session policy、首次部署都要放进这一预算，而不能只比较一个签名函数。[^mempool]

### 10.4 可读取哪些状态

正常验证路径主要可依赖交易字段、sender 自身状态和合规代码；外部第三方可变状态受到严格限制。`SLOAD` 一般仅允许访问 sender 的存储，即便通过 helper／DELEGATECALL 间接访问也一样。

这产生实际架构差异：

| Session 设计 | 公共验证路径影响 |
|---|---|
| 密钥与 Policy 存在 sender 自身 storage | 更符合可追踪依赖模型 |
| 单独外部可变 Session Registry | 不能假设 VERIFY 可任意读取 |
| 纯代码 helper／库 | 可用，但不能引入不允许的状态访问 |
| 读取 DEX 价格、Perps 保证金 | 通常应移到执行阶段或选择其他准入方案 |
| 修改每日累计预算 | VERIFY 不能任意写状态，需另行执行 |

TIMESTAMP 一般被禁止，规范对 canonical expiry verifier 提供例外。不能把普通 Solidity 的 `block.timestamp <= sessionExpiry` 随手放进账户 VERIFY，就当作公共池兼容方案。[^trace]

### 10.5 Sponsor 是共享依赖

当前 draft 描述两种 Sponsor 路径：

**Canonical paymaster：**通过精确匹配 runtime code 识别，不只是使用某个项目自称的标准接口。它限制原生资产外流，并支持延迟提现。

```text
available_balance =
  链上余额
  - 本节点 pending 最大费用预留
  - 已申请的延迟提现金额
```

节点在接纳时预留，替换、驱逐、入块和重组时释放或调整。当前描述的 canonical 验证使用单个 secp256k1 signer，不等于支持任意合约签名、动态策略或任意多签。[^paymaster]

**Non-canonical paymaster：**文本另外给出保守的单个付款账户最多 1 笔 pending 的规则，避免共享可变状态扩大失效范围。

**规范待澄清：**同一草案还含有“只有 canonical paymaster 才有资格公共传播”的限制句，与 non-canonical 专节的准入描述存在张力。实现不应自行将二者合并成“任意自定义 Sponsor 都可公共高并发传播”。Mantle 应固定支持的路径、代码版本与传播策略，并形成测试向量。[^paymaster]

### 10.6 Sender pending 限制与替换

8141 本体的公共池建议每个 sender 最多保留 1 笔 pending Frame 交易。替换使用同 `(sender, nonce)`，并按节点规则同时抬高 max fee 和 priority fee；常规增幅示例为 10%。

替换可以改 payer，但需要有效的新授权，并原子调整旧／新付款方的预留。**公共 pending 限制不是共识规定“一个账户一块只能执行一笔交易”。**[^replacement]

对于 Mantle 自有提交路径，可以研究更宽松的 pending 策略，但必须补齐依赖追踪、反滥用、余额预留和重验证；这属于需要建设的系统，不能在功能表里直接勾选为已有吞吐能力。

---

## 11. EIP-8250：为 8141 增加 keyed nonces，但不等于 8130 nonce-free

### 11.1 为什么必须单独列出

8141 本体只有一个标量 nonce。8250 是另一份 Draft，将其替换为：

```text
nonce_keys：1～16 个严格升序的 uint256 key
nonce_seq ：本次共享的 uint64 序号
```

新的载荷是：

```text
[
  chain_id,
  nonce_keys,
  nonce_seq,
  sender,
  frames,
  signatures,
  fees,
  blob_versioned_hashes
]
```

这是同一交易类型在激活后的 schema 变化，不是可以随意附加的钱包元数据。SDK、签名哈希、解码和跨分叉重组都要一起适配。[^e8250]

### 11.2 key 0 与非零 key

```text
nonce_keys = [0]
    → 使用 sender 的 legacy account nonce

nonce_keys 不含 0
    → 使用 NONCE_MANAGER 中的协议状态
```

0 不可以和其他 key 混用。

非零槽位派生：

```text
slot = keccak256(
  left_pad_32(sender) || uint256_to_bytes32(nonce_key)
)
```

原稿的 `keccak256(sender || key)` 只是简写；精确编码必须包含 32 字节左填充地址。普通调用不能写这些协议管理的 nonce 槽。[^e8250]

### 11.3 一笔交易选择多个 key，不是每个 key 带自己的序号

有效性条件为：

```text
对每个所选 key：
    current_nonce_seq(sender, key) == tx.nonce_seq
```

付款批准时一次性把所有所选非零 key 更新为 `nonce_seq + 1`。

例如：

```text
key 7 当前为 4，key 9 当前为 4
    → [7, 9], seq=4 可匹配

key 7 当前为 4，key 9 当前为 8
    → 无法用同一个 nonce_seq 同时匹配
```

因此，多 key 提供的是重放域组合，不是对任意不同策略队列的自动同步器。[^e8250]

### 11.4 能消除什么依赖

```text
交易 A：keys=[7], seq=4
交易 B：keys=[9], seq=8
```

如果 key 集合不相交，它们不共享同一个 nonce 序号依赖。但同一账户余额、保证金、订单簿或 Sponsor 仍可能冲突。

**原稿的“共识层面完全相互独立”应收窄为“重放／nonce 域独立”。**

8250 仍保留 8141 公共池每 sender 一笔 pending 的保守建议。协议移除了放宽它的一个障碍，但没有同时交付并发公共池策略。[^e8250]

### 11.5 能否实现类似无序操作

可以为独立操作选择新的非零 key、以序号 0 首次消费。这样不依赖某个老通道的前序交易，也适合从 nullifier 派生重放域。

但每个新 key 会产生持久 nonce 状态及首次 state-gas 成本，和 8130 的短有效期＋有界 replay ring 不同。因此：

```text
不准确：8250 完全不能支持无序操作。
也不准确：8250 就是 8130 的无状态 nonce-free。
准确：可构造独立的一次性重放域，但状态成本模型不同。
```

首次 key 写入的 state gas 在付款批准 Frame 中承担；缺少 state 预算会使该批准失败。Sponsor 必须认识到该成本，而不能只给业务 Frame 配 state gas。[^e8250]

### 11.6 Session key 的通道权限可以怎样做

8250 新增内省：

| TXPARAM | 含义 |
|---|---|
| `0x0D` | 交易前的 legacy nonce |
| `0x0E` | key 数量 |
| `0x0F` | 规范化 key 集合的 hash |
| `0x10` | 第一个 key |

对于“一把 Session key 只允许一条通道”的设计，验证器可检查数量为 1、首个 key 是已授权值，再批准执行。多 key 可与预先授权集合 hash 匹配。

这是**可实现的账户策略**，不是 8250 原生提供的 Actor→key 所有权表。与仅在业务执行时拒绝非法通道不同，验证阶段拒绝可以避免形成有效入块交易中的通道消耗。[^e8250]

### 11.7 升级和取消语义

只消费 legacy nonce，不会自动取消非零 keyed-nonce 交易。替换或取消需要对应的键集合／序号，或明确消费重叠的重放域；业务订单取消仍是独立机制。

8250 激活改变 canonical signing payload，跨激活边界的旧待处理交易与相关授权需要重新生成。当前两份 Draft 的 calldata 成本变量命名还存在需要同步的地方，不能把组合文本未经核对直接粘贴为估算器。[^e8250]

---

## 12. Default code、首次使用与恢复路径

### 12.1 Default code 实际做什么

对无代码、也没有委托代码的账户：

```text
VERIFY：
  读取 flags 允许的 approval scope
  如包含 execution → 检查 signatures[0]
  如仅付款        → 检查 signatures[1]
  要求是该目标账户的 SECP256K1 签名
  要求 msg 为空，即覆盖 canonical tx hash
  然后 APPROVE(scope)

SENDER / DEFAULT：
  如普通空代码调用一样成功返回
```

所以无代码 EOA 可以做基本 sender 或 payer，但不能靠 default code 自行决定 Session key 权限。账户为智能账户时，不能误用“签名第 0／1 条自动正确”的 EOA 简化逻辑。[^default]

### 12.2 第一次部署与第一次使用

公共路径允许在验证前缀前部通过部署 Frame 安装 sender 代码，然后调用 sender 验证并批准付款：

```text
部署账户
  → 验证初始化权限
  → 批准执行
  → 批准付款
  → 首次业务
```

如果后续 VERIFY 失败使整笔交易无效，部署不应保留为有效链上结果。如果全部验证通过、仅业务 Frame 失败，部署作为前面成功的独立 Frame 可能保留。

**首次调用失败不等于账户没有创建。**这与 8130 初始配置可能在业务失败后仍生效的边界相似，但具体处理由各自规范定义。[^behavior][^mempool]

### 12.3 撤权仍需要产品设计

8141 不定义统一 Session storage、revoke API 或 epoch。账户可以设计 revoked flag、session version、有效期和 Owner 恢复入口，但这些不是采纳交易格式后自动产生的能力。

撤权交易生效后，使用相应账户状态验证的待处理交易应按依赖失效；已经执行的业务不能倒转。链下签出的 Perps 订单是否失效，要看订单结算时是否检查该 Session 的当前权限及订单版本。

不应将“撤销 Agent 交易权限”“取消订单”“关闭仓位”“释放保证金”合并成一个没有状态机定义的按钮。

---

## 13. 对原稿的保留、修正与新增

| 原稿位置／主张 | 本文处理 |
|---|---|
| §1.1：交易信封与标准原生验签 | 保留，并补充 signer、msg、显式零摘要与哈希排除规则 |
| §1.2／§1.3：模式与 APPROVE | 保留；补充批准是交易级、覆盖全部后续 SENDER Frames |
| §1.4：内省指令 | 保留；补充 FRAMEDATACOPY、跨交易类型限制与结果读取边界 |
| §1.5：默认代码兼容 EOA | 保留；明确没有自动获得 Session policy |
| §2.1：签名数据 Gas 简式 | 修正：还包括 signer 和 msg；floor 采用当前精确计数函数 |
| §2.2：原子组回滚 | 保留；补充组内已执行成功 Frame 的 status 仍可为 1 |
| §2.2：官方 Example 3 的 F3～F4 为业务原子组 | 修正：官方 F3 是业务，F4 是可选 DEFAULT 后处理；原例没有该原子组 |
| §2.3：payer 确定后所有 Frame 均不受验证规则约束 | 补充：公共路径此后不允许 VERIFY |
| §2.3：8250 使交易完全独立 | 修正为重放域独立，不代表共享状态或执行资源独立 |
| §5：8250 不支持任何无序操作 | 改为区分新 key 一次性消费与有界 nonce-free 去重 |
| 原稿未展开的操作生命周期 | 新增 EOA 买入、Session＋Sponsor、失败状态、撤改单与恢复 |
| 公共 Sponsor 与组合 Draft 的文字张力 | 单列待澄清项，不把不一致默认为已解决 |

### 实现前的重点验收

应固定 8141／8250 版本及所依赖的 Gas、部署与代码语义；对完整 Frame 授权、签名哈希、default code、付款预留、前缀状态访问、原子组回执、transient storage、首次状态增长、key 消耗、重组和失败恢复形成测试向量。

**本篇结论：8141 是可编排的原生 AA 交易基础。它可以承载受限 Agent，但受限账户逻辑、公共准入兼容和产品状态机必须由实现补齐；与 8250 组合后，才能公平讨论与 8130 的多通道能力对照。**

---

## 来源

[^intro]: 用户提供：《EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析》，2026-09，重点为 §1、§2、§5。原稿列出的 EIP-8141 commit 为 `b75cbe61`；本文没有运行该版本。
[^spec]: [EIP-8141: Frame Transaction](https://eips.ethereum.org/EIPS/eip-8141)，访问于 2026-09-24，状态 Draft，文本采用 CC0。
[^e1559]: [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)，普通交易结构、nonce 和费用规则。
[^e7702]: [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)，界定对照基线；不表示 Mantle 当前部署状态。
[^erc20]: [ERC-20](https://eips.ethereum.org/EIPS/eip-20)，allowance 与 `false` 返回值处理。
[^tx]: [EIP-8141 — Frame Transaction](https://eips.ethereum.org/EIPS/eip-8141#frame-transaction)，Payload Encoding、Field Definitions。
[^constraints]: [EIP-8141 — Constraints](https://eips.ethereum.org/EIPS/eip-8141#constraints)。
[^signatures]: [EIP-8141 — Transaction Signatures](https://eips.ethereum.org/EIPS/eip-8141#transaction-signatures) 与 Signature Validation。
[^sighash]: [EIP-8141 — Signature Hash](https://eips.ethereum.org/EIPS/eip-8141#signature-hash)，以及 Canonical signature hash rationale。
[^behavior]: [EIP-8141 — Behavior](https://eips.ethereum.org/EIPS/eip-8141#behavior)，Frame transaction 处理流程与回滚规则。
[^batch]: [EIP-8141 — Behavior](https://eips.ethereum.org/EIPS/eip-8141#behavior) 及 Constraints 中的 ATOMIC_BATCH_FLAG 规则。
[^approve]: [EIP-8141 — APPROVE Instruction](https://eips.ethereum.org/EIPS/eip-8141#approve-instruction-0xaa)。
[^introspection]: [EIP-8141 — Introspection](https://eips.ethereum.org/EIPS/eip-8141#introspection)。
[^receipt]: [EIP-8141 — Receipt Encoding](https://eips.ethereum.org/EIPS/eip-8141#receipt-encoding)。
[^crossframe]: [EIP-8141 — Cross-frame interactions](https://eips.ethereum.org/EIPS/eip-8141#cross-frame-interactions)。
[^gas]: [EIP-8141 — Gas Accounting](https://eips.ethereum.org/EIPS/eip-8141#gas-accounting)，Constants、Transaction settlement、Block gas accounting、Blob handling；具体依赖以该版本 Requires 和正文为准。
[^mempool]: [EIP-8141 — Mempool](https://eips.ethereum.org/EIPS/eip-8141#mempool)，Validation Prefix、Structural Rules、Expiry Verifier Frame。
[^trace]: [EIP-8141 — Banned Opcodes](https://eips.ethereum.org/EIPS/eip-8141#banned-opcodes) 与 Trace Validation。
[^paymaster]: [EIP-8141 — Paymasters](https://eips.ethereum.org/EIPS/eip-8141#paymasters)，Canonical／Non-canonical 与余额预留。
[^replacement]: [EIP-8141 — Replacement and Eviction](https://eips.ethereum.org/EIPS/eip-8141#replacement-and-eviction)。
[^default]: [EIP-8141 — Default code](https://eips.ethereum.org/EIPS/eip-8141#default-code)。
[^examples]: [EIP-8141 — Example 3: Sponsored Transaction](https://eips.ethereum.org/EIPS/eip-8141#example-3-sponsored-transaction-fee-payment-in-erc-20)。
[^security]: [EIP-8141 — Security Considerations](https://eips.ethereum.org/EIPS/eip-8141#security-considerations)，尤其 Execution Approval Authorizes All Subsequent Sender Frames。
[^e8250]: [EIP-8250: Keyed Nonces for Frame Transactions](https://eips.ethereum.org/EIPS/eip-8250)，访问于 2026-09-24，状态 Draft；Transaction payload、Nonce state、Stateful validity、Nonce consumption、Transaction introspection、Activation、Mempool。
