# EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析

> **文档定位**：以太坊下一代 Native Account Abstraction（协议级原生账户抽象）源码级底层技术解析  
> **制定日期**：2026-09  
> **核心规范依据**：
> - **EIP-8141**：*Frame Transaction* (Status: Draft, commit `b75cbe61`)
> - **EIP-8130**：*Keystore Accounts* (Status: Draft, commit `16390e1f`)
> - **相关底层依赖**：EIP-2718 (Transaction Envelope), EIP-7702 (Set Code for EOAs), EIP-8037 (State Gas), EIP-7623 (Calldata Floor), EIP-7778 (Block Gas Accounting)
> **目标受众**：协议层工程师、虚拟机开发人员、高频系统架构师。拒绝高层抽象比喻，全篇聚焦数据结构、EVM 操作码、堆栈交互、存储布局、状态转换函数与 Mempool 准入算法。

---

## 1. EIP-8141 (Frame Transaction) 架构与执行内核

EIP-8141 的核心哲学是**「将交易解构为可编程的执行帧序列（A Sequence of Execution Frames）」**。它不再假设交易必须具备单一的发送者签名，而是将交易的**身份验证（Authentication）**、**Gas 费用支付（Payment Approval）**与**业务操作执行（Execution）**解耦为受虚拟机指令约束的独立执行单元。

### 1.1 交易信封与 RLP 载荷结构定义

EIP-8141 引入了新的 EIP-2718 交易类型 `FRAME_TX_TYPE = 0x06`。

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                         EIP-8141 交易载荷结构 (FRAME_TX_TYPE = 0x06)                        │
 ├─────────────────────────────────────────────────────────────────────────────────────────────┤
 │ 0x06 || RLP([                                                                               │
 │   chain_id:                 uint256,                                                        │
 │   nonce:                    uint64,                                                         │
 │   sender:                   address (20 bytes),                                             │
 │   frames:                   [ [mode, flags, target, limits, value, data], ... ],            │
 │   signatures:               [ [scheme, signer, msg, signature], ... ],                      │
 │   fees:                     [ max_priority_fee_per_gas, max_fee_per_gas, max_fee_blob_gas ],│
 │   blob_versioned_hashes:    [ bytes32, ... ]                                                │
 │ ])                                                                                          │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 1.1.1 核心字段与静态合法性约束（Static Constraints）

1. **`sender` (20 字节地址)**：
   - 显式声明交易发起方地址，不再依赖椭圆曲线公钥恢复隐式得出；
   - 长度必须为 20 字节。
2. **`nonce` (uint64 标量)**：
   - **关键架构约束**：EIP-8141 **保留了单一标量 Nonce**。在交易处理开始时，客户端必须严格校验：
     $$\text{tx.nonce} == \text{state}[\text{tx.sender}].\text{nonce}$$
   - Nonce 的递增不发生在交易入口，而是在 Gas 付款获批（`APPROVE` 指令执行）时由协议自动自增 1。
3. **`frames` 数组 (1 $\le$ len $\le$ 64)**：
   - 包含最多 64 个执行帧（Frame），每个 Frame 是一个六元组：
     `[mode, flags, target, limits, value, data]`
     - `mode` (uint8)：`0 = DEFAULT`, `1 = VERIFY`, `2 = SENDER`；
     - `flags` (uint8)：低 2 位（bit 0-1）为审批范围掩码 `APPROVE_SCOPE_MASK`；bit 2 为原子批处理标志 `ATOMIC_BATCH_FLAG (0x04)`；高位必须为 0；
     - `target` (address 或 Null)：目标地址。为 Null 时解析为 `tx.sender`；
     - `limits`：`[execution_gas_limit, state_gas_limit]`，定义单帧执行 Gas 与状态 Gas 硬顶；
     - `value` (uint256)：转移的 wei 数量。**注意**：只有 `mode == SENDER` 时 `value` 可以大于 0，其他模式 `value` 必须为 0；
     - `data` (bytes)：输入 Calldata。
4. **`signatures` 数组（协议级验签池）**：
   - 包含供交易与 Frame 调用的签名元组：`[scheme, signer, msg, signature]`；
   - **核心底层机制（重要修正）**：规范明确规定，`tx.signatures` 中的标准签名在**所有 Frame 执行之前由协议层完成验证，该验证并不发生在 EVM 解释器执行期间**（相关预编译 `ecrecover` 和 `P256VERIFY` 甚至不会被计入块级访问列表）。Frame 内部的 EVM 逻辑主要负责读取验签结果、校验业务授权并在满足条件时执行 `APPROVE`；
   - `scheme` 支持三种枚举：
     - `0x0 = ARBITRARY`：任意字节（仅提供见证数据，Gas 开销 100）；
     - `0x1 = SECP256K1`：经典以太坊椭圆曲线，协议级原生验签，签名编码 `v || r || s`（Gas 开销 2,800）；
     - `0x2 = P256`：WebAuthn / Passkey 曲线，协议级原生验签，签名编码 `r || s || qx || qy`（Gas 开销 6,700）；
   - `msg`：若长度为 0，代表签署整个交易的规范签名哈希 `compute_sig_hash(tx)`；若长度为 32，代表签署特定的自定义 32 字节摘要；其他长度非法。

---

### 1.2 Frame 运行模式与上下文切换矩阵

每个 Frame 执行时，EVM 执行环境根据 `mode` 进行严格的特权与上下文隔离：

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                                Frame 模式上下文矩阵                                          │
 ├──────────────┬───────────────┬─────────────────┬──────────────┬─────────────────────────────┤
 │ Frame Mode   │ CALLER        │ ORIGIN          │ 允许 Value?  │ 执行语义                    │
 │              │ (msg.sender)  │ (tx.origin)     │              │                             │
 ├──────────────┼───────────────┼─────────────────┼──────────────┼─────────────────────────────┤
 │ 0 = DEFAULT  │ address(0xaa) │ address(0xaa)   │ 否 (必须为0) │ 以协议 EntryPoint 身份调用  │
 ├──────────────┼───────────────┼─────────────────┼──────────────┼─────────────────────────────┤
 │ 1 = VERIFY   │ address(0xaa) │ address(0xaa)   │ 否 (必须为0) │ STATICCALL 静态只读模式执行 │
 │              │ (ENTRY_POINT) │ (ENTRY_POINT)   │              │ 仅 APPROVE 可变更授权上下文 │
 ├──────────────┼───────────────┼─────────────────┼──────────────┼─────────────────────────────┤
 │ 2 = SENDER   │ tx.sender     │ tx.sender       │ 是           │ 以 sender 身份执行业务调用  │
 │              │ (直接代表主号)│ (直接代表主号)  │              │ 要求 sender_approved == true│
 └──────────────┴───────────────┴─────────────────┴──────────────┴─────────────────────────────┘
```

#### 核心状态变量（Transaction-Scoped State）：
在整笔交易执行期间，EVM 维护两个全局状态指示器：
1. `sender_approved` (bool，初始为 `false`)：指示该交易是否已获得 `sender` 的正式授权，允许后续 `SENDER` 模式 Frame 执行；
2. `payer` (address，初始为 `None`)：指示已批准支付该交易全部最大 Gas 成本（`max_cost`）的账户地址。交易执行完毕后，如果 `payer == None`，整笔交易判定为非法（Invalid）。

---

### 1.3 核心专有指令：`APPROVE (0xaa)` 深入拆解

`APPROVE`（操作码 `0xaa`）是 EIP-8141 最核心的状态机跃迁指令。它的行为类似于 `RETURN`（退出当前调用帧并返回数据），但附带对交易上下文授权标记的更新。

#### 1.3.1 堆栈布局（Stack Layout）

```text
 Stack:
 [top - 0] offset:  返回数据在当前内存的起始偏移量 (与 RETURN 一致)
 [top - 1] length:  返回数据长度 (与 RETURN 一致)
 [top - 2] scope:   授权范围位掩码 (uint8 bitmask)
```

#### 1.3.2 授权操作数（Scope Operand）

`scope` 定义了当前调用的合约同意交出哪些权限：
- `0x0 = APPROVE_NONE`：不批准任何权限；
- `0x1 = APPROVE_PAYMENT`：批准支付该交易的最大 Gas 费用（本账户成为 `payer`）；
- `0x2 = APPROVE_EXECUTION`：批准后续 `SENDER` 模式以本合约身份发起外部调用；
- `0x3 = APPROVE_EXECUTION_AND_PAYMENT`：同时批准支付与执行。

#### 1.3.3 指令执行底层算法流程

```python
def op_approve(evm, offset, length, scope):
    # 1. 环境校验
    if not evm.is_frame_transaction:
        exceptional_halt()
    if evm.current_address != evm.current_frame.resolved_target:
        revert() # 只有当前帧的目标合约自身才有权执行 APPROVE
    
    # 2. 权限掩码合法性检查 (必须在当前 Frame 的 flags 允许范围内)
    allowed_scope = evm.current_frame.flags & APPROVE_SCOPE_MASK
    if scope == 0 or (scope & ~allowed_scope) != 0:
        revert()

    # 3. 授权状态转移
    if scope & APPROVE_EXECUTION:
        if evm.tx_context.sender_approved:
            revert() # 禁止重复批准执行
        if evm.current_frame.resolved_target != evm.tx.sender:
            revert() # 只有 sender 本身才能批准执行权限
        evm.tx_context.sender_approved = True

    if scope & APPROVE_PAYMENT:
        if evm.tx_context.payer is not None:
            revert() # 禁止重复批准付款 (单交易仅限一个 payer)
        if evm.current_balance(evm.current_frame.resolved_target) < evm.tx.max_cost:
            revert() # 余额不足以覆盖最大预扣费
        if not evm.tx_context.sender_approved:
            revert() # 协议铁律：必须先批准执行，才能批准付款！

        # 触发底层状态变更与 Nonce 递增
        if not account_exists(evm.tx.sender):
            charge_state_gas(STATE_BYTES_PER_NEW_ACCOUNT * CPSB)
        
        increment_nonce(evm.tx.sender) # 在此处消耗 sender 的 Nonce
        evm.tx_context.payer = evm.current_frame.resolved_target
        collect_funds(evm.tx_context.payer, evm.tx.max_cost) # 预扣款

    # 4. 退出当前调用帧，返回 [offset, offset+length) 内存数据
    evm.return_data = evm.memory[offset:offset+length]
    evm.exit_current_frame_success()
```

---

### 1.4 状态内省指令集（Introspection Opcodes）

为了使合约能够动态审计当前交易与 Frame 上下文，EIP-8141 新增了一组内省指令（仅在 `FRAME_TX_TYPE` 下有效，在传统交易中调用会直接 Exceptional Halt）：

1. **`TXPARAM (0xb0)` (Gas: 2)**：
   - 堆栈输入：`param` 索引编号；
   - 堆栈输出：返回交易级元数据（`0x00=tx_type`, `0x01=nonce`, `0x02=sender`, `0x03=priority_fee`, `0x04=max_fee`, `0x06=max_cost`, `0x08=sig_hash`, `0x09=len(frames)`, `0x0A=current_frame_index`, `0x0C=state_gas_left`）。
2. **`FRAMEDATALOAD (0xb1)` (Gas: 3)**：
   - 堆栈输入：`[top-0: offset, top-1: frameIndex]`；
   - 堆栈输出：从指定序号的 Frame 的 `data` 中读取 32 字节数据。允许跨 Frame 读取参数。
3. **`FRAMEPARAM (0xb3)` (Gas: 2)**：
   - 读取指定 Frame 的元数据（模式 `mode`、标志 `flags`、目标 `target`、限额 `limits` 等）。
4. **`SIGPARAM (0xb4)` 与 `SIGDATACOPY (0xb5)`**：
   - 读取 `tx.signatures` 数组中的签名元数据与任意签名原语（仅限 `ARBITRARY` scheme）。

---

### 1.5 默认代码（Default Code）机制

针对尚未部署合约代码的纯 EOA 账户，EIP-8141 实现了**原生无代码账户兼容（Default Code）**：
- 当一个 `VERIFY` 帧调用一个无代码地址时，EVM 并不直接返回空成功，而是原地执行固化的协议逻辑：
  1. 检查 Frame flags 是否允许 `APPROVE_SCOPE`；
  2. 在 `tx.signatures` 中查找序号匹配的 `SECP256K1` 签名，校验公钥是否恢复为当前目标地址；
  3. 若通过，直接调用 `APPROVE(allowed_scope)`。
- 这使得存量 EOA 无需部署合约，即可直接作为 EIP-8141 的 `sender` 或 `payer`！

---

## 2. EIP-8141 双维 Gas 计量、原子组回滚与 Mempool 规则

### 2.1 双维 Gas 计量模型（Two-Dimensional Gas Accounting）

EIP-8141 全面继承了 EIP-8037（State Gas）提出的双维 Gas 哲学，将网络开销明确划分为：
1. **执行 Gas（Execution Gas）**：衡量计算复杂度（CPU 周期）、临时数据吞吐和账户/存储项的热冷访问；
2. **状态 Gas（State Gas）**：衡量持久化状态膨胀（新账户创建、合约代码部署、新存储槽分配）。

与传统交易通过“蓄水池（Reservoir）模型”将一维 Gas 限制硬切给状态开销不同，EIP-8141 在 Frame 级别直接显式声明各自的独立限额：
`frame.limits = [execution_limit, state_limit]`

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                           EIP-8141 单帧双维 Gas 池运作模型                                   │
 ├─────────────────────────────────────────────────────────────────────────────────────────────┤
 │                                                                                             │
 │   Frame Entry:                                                                              │
 │   gas_left       = frame.limits.execution                                                   │
 │   state_gas_left = frame.limits.state                                                       │
 │                                                                                             │
 │   [计算与指令执行] ──消耗──► gas_left (若不足 -> Out of Gas Revert)                             │
 │   [新存储/新建账户] ──消耗──► state_gas_left (若不足 -> Out of State Gas Revert)                │
 │                                                                                             │
 │   * 两者完全正交：执行 Gas 用尽不会从 状态 Gas 借调；状态 Gas 耗尽也不会侵占 执行 Gas！            │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.1 状态存储计费与 CPSB 参数
在 EIP-8141 中，如果某个 Frame 创建了新账户（比如价值转移给了未初始化的地址），协议将直接扣除状态 Gas：
$$\text{Cost}_{\text{state}} = \text{STATE\_BYTES\_PER\_NEW\_ACCOUNT} \times \text{CPSB} = 120 \times 1,530 = 183,600 \text{ State Gas}$$
- `CPSB`（Cost Per State Byte）：每字节状态成本，固化为 1,530；
- `STATE_BYTES_PER_NEW_ACCOUNT`：新账户基础状态大小，固化为 120 字节。

#### 2.1.2 固有 Gas 与 Calldata Floor 算法（EIP-7623 / EIP-7976）
为了防止大交易载荷以极低成本泛洪网络，EIP-8141 引入了严格的 Calldata 下限（Calldata Floor）定价机制（下限系数 `TOTAL_COST_FLOOR_PER_TOKEN = 16` 源自其依赖的 **EIP-7976**）：

```python
# 1. 签名与固有开销
signature_verification_cost = sum(signature_gas(sig) for sig in tx.signatures)
frame_data_cost = sum(STANDARD_TOKEN_COST * tokens_in(frame.data) for frame in tx.frames)
signature_data_cost = sum(STANDARD_TOKEN_COST * tokens_in(sig.bytes) for sig in tx.signatures)

frame_tx_intrinsic_gas = (
    FRAME_TX_INTRINSIC_COST (12,000)
    + len(tx.frames) * FRAME_TX_PER_FRAME_COST (475)
    + frame_data_cost + signature_data_cost
    + signature_verification_cost + value_transfer_cost
)

# 2. Calldata Floor 计算 (EIP-7976 / EIP-7623 强制下限: 每 Token 16 Gas)
calldata_floor_gas = (
    FRAME_TX_INTRINSIC_COST + len(tx.frames) * FRAME_TX_PER_FRAME_COST
    + signature_verification_cost + value_transfer_cost
    + TOTAL_COST_FLOOR_PER_TOKEN (16) * calldata_floor_tokens
)

# 3. 交易最大 Gas 预约
max_gas = max(
    frame_tx_intrinsic_gas + sum(f.limits.execution + f.limits.state for f in tx.frames),
    calldata_floor_gas + sum(f.limits.state for f in tx.frames)
)
```

#### 2.1.3 状态 Gas 回补（Refill）vs 传统退款（Refund）
- **传统 Gas Refund**：受 `MAX_REFUND_QUOTIENT = 5`（最多退还 20% 执行 Gas）限制；
- **State Gas Refill**：根据 EIP-8037，如果交易在执行过程中清理了此前在同一交易中新建的存储槽位（如 `SSTORE` 槽位清零重置），协议执行**状态 Gas 回补（Refill）**。回补直接削减回执中的 `frame_receipt.gas_used.state`，**不受任何退款上限约束**，因为这些状态从未在链上持久化留存；
- **⚠️ 关键细节修正**：**`SELFDESTRUCT` 在 EIP-8037 中明确不会产生任何 State Gas Refill**，即使它销毁的是同一交易内刚刚新建的合约账户，也不会降低此前创建帧的 `gas_used.state`。跨帧回补仅发生在 `SSTORE` 将先前创建的槽位清零时。

---

### 2.2 原子批处理（Atomic Batching）与帧级回滚边界

EIP-8141 通过 Frame 的 `flags` 字段中的 `ATOMIC_BATCH_FLAG = 0x04` 原生实现原子多操作批处理。

#### 2.2.1 原子组的定义与判定
一个原子批次（Atomic Batch）被定义为连续的 Frame 序列 $[i, j]$：
- Frame $i$ 到 $j-1$ 的 `flags` 均显式设置了 `ATOMIC_BATCH_FLAG`；
- 终止帧 Frame $j$ **未设置** `ATOMIC_BATCH_FLAG`（标志着本原子批次闭合）；
- **静态规则**：`VERIFY` 模式的 Frame 绝对不允许携带 `ATOMIC_BATCH_FLAG`，批次内只能包含 `DEFAULT` 或 `SENDER` 模式。

#### 2.2.2 状态回滚与隔离铁律

```text
 Frame 0 (VERIFY): APPROVE(EXECUTION_AND_PAYMENT) ──► 状态落定！消耗 Nonce，锁定 Payer 预扣款
                                                              │
   ┌─────────────────────── ATOMIC BATCH ─────────────────────┼─────────────────────────┐
   │                                                          ▼                         │
   │ Frame 1 (SENDER, flag=0x04): USDC.approve(router, 100) ──► 成功                    │
   │                                                          │                         │
   │ Frame 2 (SENDER, flag=0x00): router.buy(...) ────────────► Revert! 触发原子组回滚    │
   └──────────────────────────────────────────────────────────┼─────────────────────────┘
                                                              │
                                                              ▼
               【回滚动作】：
                1. Frame 1 与 Frame 2 的所有业务状态变更全部丢弃！
                2. Frame 1 与 Frame 2 产生的所有日志 (Logs) 全部擦除！
                3. Frame 2 回执标记为 Reverted，该原子组内未跑的后续帧标记为 Skipped (status=0x02)
                4. ⚠️ 组外独立的后续 Frame (未带原子组标记的帧) 依然可以继续执行！
               【绝对不回滚的项】：
                • Frame 0 的 Nonce 消耗依然有效！
                • Frame 0 设置的 Payer 预扣款依然有效！
                • Frame 1 与 Frame 2 实际消耗的 CPU 执行 Gas 照常向 Payer 扣除！
```

**关键设计结论**：
1. **原子批次跳过范围限制**：当原子批次中某帧失败时，协议跳过的是**同一原子组内剩余的 Frame**，而不是无条件跳过整个交易的所有后续 Frame。组外独立的后续 Frame 仍会正常调度；
2. **“业务失败但代付报销保留”并非 8130 独有**：
   在 EIP-8141 官方给出的 **Example 3: Sponsored Transaction (Fee Payment in ERC-20)** 中：
   - Frame 0: `VERIFY` 批准用户执行权；
   - Frame 1: `VERIFY` 批准 Sponsor 付款权；
   - Frame 2: `SENDER` 用户向 Sponsor 转移 ERC-20 代币作为服务费报销（未设置原子批次标记）；
   - Frame 3~4: 用户的业务操作（带原子批次标记）；
   - **结果**：若 Frame 3~4 业务失败回滚，Frame 2 的 ERC-20 费用转移**不会被回滚**，Sponsor 依然可以稳妥收到报销。因此，两案在精细编排下均能做到“业务失败而报销保留”。但两案都无法防范用户在入块前恶意转走 ERC-20 余额的前置风险。

---

### 2.3 Mempool 准入规则、验证前缀与单 Nonce 瓶颈

由于 Frame 交易允许任意可编程逻辑，如果公共内存池（Public Mempool）允许节点模拟任意复杂的 Frame，网络将面临毁灭性的 DoS 洪泛攻击。为此，EIP-8141 制定了极其苛刻的准入限制。

#### 2.3.1 验证前缀（Validation Prefix）
交易的**验证前缀**是指：**从第 0 帧开始，到成功执行 `APPROVE` 并确立 `payer` 的最短 Frame 子序列。**
- 公共 Mempool 的准入规则**仅针对验证前缀执行静态与模拟审计**；
- 一旦 `payer` 确立，后续的所有 Frame 无论多复杂，都不受 Mempool 验证规则约束（因为付款人已确定，失败也能扣费）。

#### 2.3.2 节点准入三铁律
1. **严格的气量封顶**：
   - 验证前缀涉及的所有签名验证与模拟执行，累计 Gas 不得超过 `MAX_VERIFY_GAS = 100,000`；
   - 累计状态 Gas 不得超过 `MAX_VERIFY_STATE_GAS = 500,000`。
2. **零第三方可变状态依赖（Disallowed Mutable State）**：
   - 验证前缀在执行过程中，**严禁读取未授权的第三方可变存储槽（Storage Slots）**；
   - 规范严格定义了允许访问的依赖范围（合规状态集合）：
     1. 交易自身字段与规范签名哈希；
     2. 由 `EXPIRY_VERIFIER (0x8141)` 读取的区块时间戳；
     3. 发送方（Sender）自身的 Nonce、Code 与 Storage；
     4. 若包含工厂部署帧，被调用的 Factory 合约的代码；
     5. 若包含 Paymaster 帧，规范的 Canonical Paymaster 实例（带余额预约）或被不超过 `MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER (1)` 笔交易使用的非规范 Paymaster；
     6. 验证期间通过 `CALL*` 或 `EXTCODE*` 访问的其他既有非委托合约的代码（前提是不触碰非法可变存储）。
   - 任何读取外部 DEX 价格、全局可变变量的行为，将被公共 Mempool 判定为非法并直接拒绝传播。
3. **单 Sender 单待处理交易限制（Single In-Flight Transaction Per Sender）**：
   - 协议明确建议：**节点在公共 Mempool 中，同一个 `sender` 最多仅保留一笔待处理的 Frame 交易**！
   - 新交易进入 Mempool 只能通过标准的 `(sender, nonce)` 覆盖替换规则（加价至少 10%）。

#### 2.3.3 8141 独立规范的单 Nonce 瓶颈、伴生补丁 EIP-8250 与隐私定位

在单纯的 **EIP-8141 核心规范** 中：
- 交易载荷仅声明了单一标量 `nonce: uint64`；
- 公共 Mempool 限制单一 Sender 最多 1 笔待处理 Frame 交易。这确实使得纯 8141 在协议层面临串行线头阻塞（HoLB）问题。

**1. 伴生提案 EIP-8250（Keyed Nonces for Frame Transactions）的技术机制**
以太坊核心开发者（包括 Thomas Thiery, Toni Wahrstätter, lightclient, Vitalik Buterin）于 2026 年 4 月正式提出了 **EIP-8250**：
- **载荷替换**：将 8141 载荷中的 `nonce` 字段替换为 `nonce_keys`（最多 16 个升序 uint256 键）和 `nonce_seq`（uint64 序号）；
- **协议级 Nonce 管理器**：在 `0x0000000000000000000000000000000000008250` 引入 `NONCE_MANAGER`，每个非零 key 在底层拥有独立的存储槽：`slot = keccak256(sender || nonce_key)`；
- **并发重放独立性（Replay-Independence）**：当两笔 Frame 交易的非零 `nonce_keys` 集合互不相交时，二者在共识层面**完全相互独立，互不依赖**。
- **Mempool 规则与共识上限的区别**：EIP-8250 移除了协议层 Nonce 的阻塞障碍，但其 Draft 规范中为防 DoS，**在公共 Mempool 层面仍建议暂时维持单一 sender 仅 1 笔待处理交易的保守规则**，将多通道并发 Mempool 规则留待后续网络策略放开。但对于专有 L2（如 Mantle），公共 Mempool 规则不构成协议层共识上限。

**2. 8141 的隐私定位与一手出处边界**
EIP-8141 本身是纯正的 Native Account Abstraction 提案，并非脱离 AA 的专用隐私方案。但其通用 Frame 抽象确实被以太坊官方路线图深度纳入隐私基础设施规划：
- **一手出处一（EIP-8250 Motivation）**：8250 明确写道：*“The leading example is privacy protocols.”* 其核心动机是让隐私应用中的多个用户通过同一个公开 `sender` 提款，避免按地址暴露身份，而 Keyed Nonce 则派生自隐私 Nullifier，消除了单 Nonce 造成的用户间互相阻塞；
- **一手出处二（ethereum.org Privacy Roadmap）**：官方隐私路线图专门设有 *“How do frame transactions (EIP-8141) enable privacy?”* 章节，明确指出私密操作需要不被传统公开 EOA 签名和公开 Gas 付款束缚的解耦路径；
- **一手出处三（验证预算限制）**：2026 年 9 月 2 日的 Ethereum Research 研报 *《EIP-8141 and minimum required validation budget for privacy applications》* 指出，Groth16 隐私验证路径在许多场景下会超过公共池 `MAX_VERIFY_GAS = 100,000` 限制。
- **准确的评估结论**：EIP-8141 是通用型 Native AA 提案，其可编程验证和付款批准机制可为隐私应用提供交易基础设施；配套 EIP-8250 明确将共享 sender 的隐私协议列为主要动机。不过，这些提案本身不提供交易内容加密、隐藏仓位或完整的抗 MEV 保证，实际部署仍依赖隐私协议、证明成本与交易排序机制。

---

## 3. EIP-8130 (Keystore Accounts) 架构与执行内核

如果说 EIP-8141 是以“交易执行流”为中心的可编程脚本模式，那么 EIP-8130 的设计内核则是**「状态声明式的链上企业级 IAM（Identity and Access Management）与权限门禁系统」**。

它从根本上将**账户（Account，资金与状态容器）**、**操作者（Actor，谁在操作）**与**认证器（Authenticator，使用什么密码学算法验签）**三者正交解耦。

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                           EIP-8130 核心解耦架构 (The IAM Triad)                             │
 ├─────────────────────────────────────────────────────────────────────────────────────────────┤
 │                                                                                             │
 │   【Account 账户】  ◄──── (持有资产与状态，全局唯一地址，通过 7702 自动委托默认代码)            │
 │          │                                                                                  │
 │          ▼ (映射绑定)                                                                       │
 │   【Keystore 合约】 ────► 存储每个账户授权的 [Actor 列表] 与 [策略绑定]                     │
 │          │                                                                                  │
 │          ├──► Actor 1 (Owner):  Authenticator=k1 (secp256k1), Scope=0x0000 (Admin)          │
 │          ├──► Actor 2 (Agent):  Authenticator=p256, Expiry=t_exp, Scope=POLICY|NONCE        │
 │          │                      └── Gated to: PolicyManager (限额 500U, 限 MemePool)        │
 │          └──► Actor 3 (Sponsor):Authenticator=k1, Scope=SPONSOR_PAYER                       │
 │                                                                                             │
 │   【Authenticator 认证器】 ──► 纯函数/合约，负责入参验签并返回 actorId                         │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.1 链上 Keystore 规范与存储布局（Storage Layout）

在 EIP-8130 中，每个账户的授权配置直接固化在链上单例 Keystore 合约（`KEYSTORE_ADDRESS`）的存储中。协议执行层直接在原生代码中对该存储槽进行 SLOAD 读取，因此其字节打包格式是严格规范化的（Normative Packing）。

#### 3.1.1 `actor_config` 单槽位紧凑打包图解

每个 Actor 仅占用一个 32 字节的 Storage Slot：

```text
 32-Byte Word (actor_config slot):
 ┌───────────────────────────┬──────────────┬──────────────┬──────────────┐
 │ Byte 0..19 (20 Bytes)     │ 20..25 (6B)  │ 26..27 (2B)  │ 28..31 (4B)  │
 ├───────────────────────────┼──────────────┼──────────────┼──────────────┤
 │ authenticator             │ expiry       │ scope        │ reserved     │
 │ 认证器合约地址            │ uint48 时间戳│ uint16 位域  │ 预留版本位   │
 └───────────────────────────┴──────────────┴──────────────┴──────────────┘
```

- **`authenticator` (20 字节)**：指向负责该 Actor 验签的认证器合约地址；
- **`expiry` (uint48，6 字节)**：Unix 时间戳（以**秒**计）。`0` 表示永不过期；若 `block.timestamp > expiry`，该 Actor 瞬间失效；
- **`scope` (uint16，2 字节)**：权限位掩码（Bitmask Grants）。`0x0000` 为无限制 Admin 权限；
- **`reserved` (4 字节)**：预留供未来硬分叉扩展使用。

#### 3.1.2 策略关联存储槽（Policy Storage Slots）
如果在授权该 Actor 时附带了正好 **52 字节的 `policyData`**，Keystore 将在该 Actor 的存储命名空间下额外写入两个槽位：
```solidity
// policyData = manager (20 字节) || commitment (32 字节)
mapping(address account => mapping(bytes32 actorId => bytes32)) public policy_commitment;
mapping(address account => mapping(bytes32 actorId => address)) public policy_manager;
```
这两个槽位**在交易验证阶段完全不读取**（保持验证阶段仅需单次 SLOAD 的极致性能），仅在业务调用派发时由协议或 Manager 合约读取。

---

### 3.2 Actor Scope 权限掩码体系与排他性规则

`scope` 字段采用**纯白名单授权位（Pure Grants）**设计。如果读取到的位域中包含未知的保留位，协议一律 fail-closed（不授予任何未识别权限）。

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                           EIP-8130 Actor Scope 权限位掩码全景                               │
 ├─────┬────────┬───────────────┬──────────────────────────────────────────────────────────────┤
 │ Bit │ Value  │ Name          │ 权限语义与生效上下文                                         │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ -   │ 0x0000 │ ADMIN         │ 无限制根权限。唯一有权发起 authorize/revoke/delegation 的角色│
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 0   │ 0x0001 │ OPERATOR      │ 自由发起权限。允许作为 sender 发起针对任意 call.to 的交易；  │
 │     │        │               │ 统领 ERC-1271 签名验证权限                                   │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 1   │ 0x0002 │ SELF_PAYER    │ 自付 Gas 权限。当 payer == sender 时，允许从本账户扣取 Gas   │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 2   │ 0x0004 │ SPONSOR_PAYER │ 赞助 Gas 权限。允许作为 payer_auth 为其他第三方 sender 代付  │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 3   │ 0x0008 │ POLICY        │ 门禁发起权限。强制限制其发起的每一笔调用的 call.to 必须且    │
 │     │        │               │ 只能是已配置的 policy_manager 合约                           │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 4   │ 0x0010 │ NONCE         │ 并发通道权限。允许使用非零的有序 2D Nonce 通道 (nonce_key)   │
 ├─────┼────────┼───────────────┼──────────────────────────────────────────────────────────────┤
 │ 5-15│ -      │ (spare)       │ 预留未分配位                                                 │
 └─────┴────────┴───────────────┴──────────────────────────────────────────────────────────────┘
```

#### 权限组合与冲突仲裁铁律（Precedence Rules）：

1. **`OPERATOR` 绝对覆盖 `POLICY`（重大安全陷阱）**：
   - 规范规定：`OPERATOR` 是更宽松的无门禁权限。如果一个 Actor 同时被赋予了 `OPERATOR | POLICY`（`0x0009`），**协议将直接按 OPERATOR 处理，彻底无视 POLICY 门禁**！
   - **工程规范**：受限交易 Agent **绝对不能携带 `OPERATOR` 位**，否则门禁将荡然无存。
2. **合法的典型组合**：
   - **自主交易 Agent 标配**：`POLICY | NONCE`（`0x0018`）——只能调用指定的风控 Manager，且享有独立的 2D Nonce 并发通道；
   - **自费高频 Agent**：`POLICY | NONCE | SELF_PAYER`（`0x001A`）——在受限前提下，允许自主划扣本账户的原生代币付 Gas；
   - **机构第三方代付节点**：`SPONSOR_PAYER`（`0x0004`）——专门充当 Paymaster。

3. **⚠️ 极其关键的集成断层：`POLICY` 受限键不自动获得通用 ERC-1271 订单签名权限**：
   - 在 8130 官方参考实现及规范中，账户的通用 `isValidSignature(hash, signature)`（ERC-1271）默认调用的门禁判断是 `Scopes.isOperator`，它只接受 **`ADMIN`（`scope == 0x00`）或带有 `OPERATOR` 权限的 Actor**；
   - **直接影响**：一个被赋予 `POLICY | NONCE` 的受限交易 Agent，**无法直接通过通用 ERC-1271 验证去签署离线限价单（如链下撮合、链上结算的订单簿 Perps）**！
   - **不能靠简单追加 `OPERATOR` 修复**：因为一旦加上 `OPERATOR`，根据前述铁律，`OPERATOR` 会直接覆盖并摧毁 `POLICY` 门禁；
   - **工程结论**：如果业务涉及离线订单撮合，订单验证器必须专门感知 8130 的 Actor 策略并显式校验 Policy Commitment，不能直接假设通用 ERC-1271 校验器开箱可用。

---

### 3.3 策略管理器（Policy Manager）与门禁运行时校验

当交易由一个携带 `POLICY` 权限的 Agent 发起时，协议在虚拟机层面对其施加物理级调用拦截。

#### 3.3.1 协议级门禁拦截机制（The Manager Gate）

在进入 `calls` 执行阶段前，协议首先从 Keystore 快照中读取该 Actor 绑定的 `policy_manager`：
- **对于交易中的每一个调用 `call`**：协议强制校验：
  $$\text{assert}(\text{call.to} == \text{stored\_policy\_manager})$$
- **违规即时拦截**：若 Agent 试图调用非 Manager 地址（例如试图直接调用 `USDC.transfer(hacker)`），EVM 甚至**根本不会派发该外部调用**，而是原地触发共识级异常：
  `ActorPolicyViolation(bytes32 actorId, address target)`

```solidity
// 协议级回滚错误签名
error ActorPolicyViolation(bytes32 actorId, address target);
```

#### 3.3.2 费用与状态回滚结算：
- 这是一个**共识级执行错误，而非交易非法错误**；
- 违规调用所在的当前 Phase 状态全部回滚，后续 Phase 全部跳过；
- **但交易依然被成功打包上链，Nonce 正常递增，已消耗的固有 Gas 照常向 Payer 扣除**（彻底杜绝了 Agent 恶意尝试非法调用导致 Sequencer 白白耗费 CPU 的 DoS 漏洞）。

#### 3.3.3 安全边界澄清：`POLICY` 是入口目标限制，不是“本金固若金汤”
必须严格厘清协议层门禁的能力边界：
- 协议能够绝对保证的，仅仅是受限 Actor 的顶层 `call.to` 必须是指定的 Manager，防止其直接调用未经授权的外部恶意合约或直接发起原生/ERC-20 提现；
- **协议完全不理解最大杠杆、滑点、合理成交价、单日累计敞口或爆仓风险**：这些逻辑必须由 Manager 合约代码自身实现。若 Manager 内部逻辑有漏洞、受预言机操控或存在不良可升级性，资金依然面临风险；
- **“不能盗提”绝不等于“不会亏损”**。恶意或失控的交易策略在白名单池子内部依然可能通过高滑点交易或错误头寸造成本金严重损失，必须在应用层构筑独立风控规则。

#### 3.3.4 策略承诺（Commitment）的工作时序

```text
 1. 授权绑定:
    Owner 签名将 (Manager, Hash(Commitment)) 存入 Keystore。
    Commitment 内部声明: [预算 <= 100 USDC, 目标池 = MemePool, 截止时间 = t]

 2. 交易发起:
    Agent 发送 AA 交易 -> 协议校验目标确实是 Manager -> 派发调用至 Manager

 3. 业务校验:
    Manager 合约通过 TransactionContext 预编译读取 (sender, actorId)
    -> 从 Keystore 读取 Hash(Commitment)
    -> 比对 Agent 本次传入的参数是否符合 Commitment
    -> 若符合，Manager 驱动 Sender 完成业务兑换
```

---

### 3.4 纪元系统（Epoch System）与即时作废边界

为了支持离线批量授权的高效管理，EIP-8130 设计了**双层时间与版本机制**：
- **Actor Expiry（绝对时间戳）**：精确到秒。到期后节点在验证阶段直接判定该 Actor 死亡，交易无法进入 Mempool；
- **Local Epoch（纪元版本号）**：
  - Keystore 为每个账户维护一个 `local_epoch`；
  - 账户签署一条简单的 `IncrementLocalEpoch` 指令，即可将当前纪元递增；
  - **⚠️ 核心边界澄清：Local Epoch 不是全能的“紧急停机开关”**：
    - `IncrementLocalEpoch` 的真实作用是使**在 Local 通道中签署但尚未落地的账户配置变更批次（Unlanded Local Batches / JIT 签名）失效**；
    - **它绝对不会自动注销已经生效上链的 Actor**！已经写入 Keystore 存储的活跃 Actor 依然有效，必须通过显式的 `RevokeActor` 操作抹除其存储槽；
    - **它不影响 Multichain 跨链通道**（Multichain 批次仅由严格单调递增计数器维护）；
    - **它不会自动使链下已签出的业务订单（如 Perps 限价单）失效**。因此，撤销尚未落地的配置、注销已激活的交易键、取消已挂出的链下订单是三套独立的机制，不能混为一谈。

---

## 4. EIP-8130 2D Nonce、环形去重缓冲区与 Calls Phase 模型

### 4.1 二维 Nonce 预编译合约（Nonce Manager Precompile）

为了在保证重放保护的同时打破单调自增 Nonce 对高频并发的枷锁，EIP-8130 在协议层引入了专有的 **Nonce 管理预编译合约**：
- **固化预编译地址**：`NONCE_MANAGER_ADDRESS = 0x813000000000000000000000000000000000aa01`；
- **接口定义**：向 EVM 暴露只读状态查询：
  ```solidity
  interface INonceManager {
      function getNonce(address account, uint256 nonceKey) external view returns (uint64);
  }
  ```
- **执行内核直管**：该合约内部没有状态写入函数，写操作由协议层执行内核在处理 AA 交易时原生完成直接写入。

#### 4.1.1 通道空间划分与权限约束

交易显式携带两维 Nonce 标识：`nonce_key`（uint256 通道号）与 `nonce_sequence`（uint64 期望序号）。

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                           EIP-8130 Nonce Key 空间划分与调度策略                             │
 ├──────────────────────────┬─────────────────┬────────────────────────────────────────────────┤
 │ nonce_key 取值范围       │ 通道类别        │ 状态机行为与适用场景                           │
 ├──────────────────────────┼─────────────────┼────────────────────────────────────────────────┤
 │ 0                        │ 标准通道        │ 单调自增序列，Mempool 默认，兼容人类日常低频   │
 ├──────────────────────────┼─────────────────┼────────────────────────────────────────────────┤
 │ 1 .. NONCE_KEY_MAX - 1   │ 自定义并发通道  │ 独立序号映射 (account, key) => seq，多策略并行 │
 ├──────────────────────────┼─────────────────┼────────────────────────────────────────────────┤
 │ NONCE_KEY_MAX (2^256 - 1)│ 免 Nonce 模式   │ 不读不增 Nonce 计数器，走环形去重缓冲区        │
 └──────────────────────────┴─────────────────┴────────────────────────────────────────────────┘
```

- **权限约束与作用域**：
  - 只有持有 `ADMIN`（`scope == 0x0000`）或显式赋予了 `NONCE (0x10)` 的受限 Actor，才允许使用 `1 .. NONCE_KEY_MAX - 1` 的有序通道；
  - 若受限 Actor 未被赋予 `NONCE` 权限，它**只能使用 `NONCE_KEY_MAX`（免 Nonce 模式）**；
  - **⚠️ 关键通道隔离边界**：Nonce 存储槽按 `(account, nonce_key)` 映射，**不按 Actor 隔离**。这意味着同一账户下所有持有 `NONCE` 权限的 Actor 都可以使用同一个 `nonce_key`。将通道 1 分给策略 A、通道 2 分给策略 B 属于客户端约定，协议并不防止受限 Actor 干扰其他通道，失陷的 Key 仍可能污染共享通道。
- **Gas 计费标准**：
  - 首度开辟某个通道（首次写入）：`22,100 Gas`（冷 SLOAD + SSTORE 新建）；
  - 复用已有通道：`5,000 Gas`（冷 SLOAD + 热 SSTORE 重设）；
  - 免 Nonce 模式：`13,000 Gas`（覆盖环形缓冲区去重状态开销）。

---

### 4.2 免 Nonce 模式（`NONCE_KEY_MAX`）与共识环形缓冲区

对于高频微调报价、即时撤单或时效性极强的抢跑任务，维护有序自增的序号反而是累赘。EIP-8130 允许交易指定 `nonce_key = NONCE_KEY_MAX` 开启**免 Nonce 模式**。

#### 4.2.1 规则约束
当 `nonce_key == NONCE_KEY_MAX` 时：
1. `nonce_sequence` 必须严格强制等于 `0`；
2. `valid_before`（有效截止时间戳）必须非零；
3. `valid_before - now` 必须落在链参数 `NONCE_FREE_EXPIRY_WINDOW` 规定的时间窗口内（注：规范将时间戳归一化为毫秒，但底层共识时钟依然以区块秒级时间戳为准，不代表底层提供亚秒级共识时钟）；
4. 底层状态机**完全不读取、也不递增任何 Nonce 存储槽**。

#### 4.2.2 共识级环形去重缓冲区（The Consensus Circular Ring Buffer）
为了在没有 Nonce 的前提下防御双花与重放攻击，EIP-8130 在链上构建了一套**全网共识级环形去重结构**（注意：这是**有界的共识状态**，每个全节点计算出的状态根必须完全一致，而非单节点的链下缓存）。

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                         共识环形缓冲区 (Replay Ring Buffer) 防重放机制                       │
 ├─────────────────────────────────────────────────────────────────────────────────────────────┤
 │                                                                                             │
 │   seen 映射: mapping(bytes32 replay_id => uint64 valid_before)                              │
 │   ring 队列: 固定长度数组 [slot_0, slot_1, ..., slot_N-1] (容量 = REPLAY_BUFFER_CAPACITY)   │
 │                                                                                             │
 │   【入块检查流程】：                                                                        │
 │   1. 根据交易内容计算出业务逻辑指纹 replay_id                                               │
 │   2. 检查 seen[replay_id]:                                                                  │
 │      • 若存在且未过期 (valid_before >= block.timestamp * 1000) ──► 判定重放，直接拒绝！    │
 │   3. 读取环形队列当前游标指向的槽位:                                                        │
 │      • 若该槽位的旧条目尚未过期 (缓冲区已满且不可驱逐) ─────────► 交易直接被拒！            │
 │      • 若已过期 ──────────────────────────────────────────► 驱逐旧条目，写入新 (id, expiry) │
 │                                                                                             │
 │   * 特性：固定容量循环复用，但满载时会拒绝新交易，必须合理配置容量参数！                   │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.3 业务指纹算法（`replay_id`）与加价替换（RBF）
在 EIP-8130 规范及参考实现中，`replay_id` 的权威定义如下（注意字段排列与 `payer` 包含）：

```text
REPLAY_ID_TYPE = 0x7901

replay_id = keccak256(REPLAY_ID_TYPE || rlp([
    chain_id,
    resolved_sender,
    valid_after,
    valid_before,
    account_changes,
    calls,
    metadata,
    payer
]))
```

**关键设计与边界**：
- `replay_id` 显式排除了费用字段（`max_fee`, `priority_fee`）和认证签名数据，因此单纯提高小费加速交易（Fee-Bump）不会改变 `replay_id`，节点精准识别为同一逻辑交易并执行原位替换；
- **⚠️ 但是，`replay_id` 包含了 `payer` 和 `valid_before`**：如果更换了代付 Sponsor，或者顺延了有效期，`replay_id` 将发生变化，被视为全新的逻辑交易。因此它**不能作为长期业务订单的唯一 ID**，应用层改单和重试仍需业务幂等控制。

---

### 4.3 双层调用架构（`calls` 两级 Phase 执行模型）

在执行层面，EIP-8130 彻底摒弃了“单交易要么全成功要么全回滚”的粗糙模型，提出了 **两级调用架构（Sequentially-Committed Atomic Call Groups）**：
`calls = [ [call_0_0, call_0_1], [call_1_0, call_1_1], ... ]`

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                         EIP-8130 calls 两级 Phase 执行与提交模型                            │
 ├─────────────────────────────────────────────────────────────────────────────────────────────┤
 │                                                                                             │
 │   Phase 0: [ Call A (充值/转账给 Sponsor 作为 Gas 补偿) ]                                   │
 │            ├──► 执行成功！状态就地持久化提交 (Commit) ──────────────────────────┐           │
 │                                                                                 │           │
 │   Phase 1: [ Call B: USDC.approve,  Call C: Pool.buy ] (原子业务组)             │           │
 │            ├──► Call B 成功                                                     │           │
 │            └──► Call C 失败 (滑点超限) ──► 触发本 Phase 回滚！                  │           │
 │                                                                                 │           │
 │   Phase 2: [ Call D (后续操作) ] ──► 直接跳过 (Skipped)                         │           │
 │                                                                                 │           │
 │   【最终状态结算】：                                                            ▼           │
 │    • Phase 1 的业务状态变更 (Approve) 彻底回滚！                                            │
 │    • Phase 0 已经提交的状态 (Sponsor 拿到 Gas 补偿) 永久生效！不会被撤销！                  │
 │    • 整个交易宣告入块成功 (Nonce 消耗，Gas 结清)，收据中 phaseStatuses = [0x01, 0x00, 0x00]  │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 语义总结：
1. **Phase 内部原子性（Phase-Level Atomicity）**：一个 Phase 内部的多个调用属于强耦合原子组，任何一步失败，**当前 Phase 内部的所有状态全部回滚**；
2. **跨 Phase 顺序持久提交（Cross-Phase Sequential Persistence）**：早期 Phase 一旦成功，状态立即落定；**后续 Phase 的 Revert 绝对不回滚已提交的前序 Phase**；
3. **商业级价值**：完美解决代付商业模式中**“业务执行因市场滑点失败，但代付商不能白白垫付 Gas 必须拿到报销”**的长期痛点。

---

### 4.4 两种采用模式（Adoption Profiles）与验签路径

EIP-8130 充分考虑了 L1 主网与高性能 L2 之间的执行约束差异，设计了两种可选激活模式：

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                               两种采用模式 (Adoption Profiles)                              │
 ├────────────────────────┬───────────────────────────────────┬────────────────────────────────┤
 │ 维度                   │ L1 Profile (标准规范，宽容准入)   │ L2 Profile (高性能，仅标准集)  │
 ├────────────────────────┼───────────────────────────────────┼────────────────────────────────┤
 │ 准入策略               │ Permissive (宽容)                 │ Canonical-only (仅限标准集)    │
 │ Canonical 认证器       │ 原生固化验签，恒定成本            │ 原生固化验签，恒定成本         │
 │ 非 Canonical 认证器    │ 允许单次 STATICCALL，受限 Gas 预算│ 交易热路径完全禁止，只走普通EVM│
 │ 节点验证开销           │ 需跟踪任意状态依赖与失效条件      │ 路径收窄，依赖状态枚举清晰     │
 │ 适用场景               │ 以太坊 L1 主网 (最大化去中心化)   │ 高性能 Rollup L2 (高吞吐)      │
 └────────────────────────┴───────────────────────────────────┴────────────────────────────────┘
```

#### 规范认证器集合（Canonical Authenticator Set）：
**两个 Profile 都在验证路径上将 Canonical 认证器原生固化，提供恒定 Gas 成本**：
1. **`k1` (Native secp256k1)**：原生 ECDSA 验签；
2. **`p256` (P-256 / secp256r1)**：Apple / Android 硬件 Enclave 支持；
3. **`passkey` (WebAuthn / FIDO2)**：浏览器通行密钥；
4. **`delegate` (Signature Delegation)**：单层授权代理。

**性能对比客观性**：8130 的优势在于通过固化认证器和规范化存储槽收窄了热路径验证复杂度；而 8141 也在外层 `signatures` 提供了协议级原生验签。两者具体执行延迟与吞吐差异取决于节点实现与状态访问，不能直接在无基准测试的前提下断言“纳秒级”或“数量级领先”。

---

## 5. EIP-8141 vs EIP-8130 设计内核全景对比与 Mantle v3 选型评估

经过对两份规范底层细节与虚拟机执行内核的逐层推演，我们需要跳出纸面功能对比，结合真实业务架构进行严谨评估。

### 5.1 核心设计哲学的本质分歧

```text
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                               两种 Native AA 设计哲学的根本分歧                             │
 ├─────────────────────────────────────────┬───────────────────────────────────────────────────┤
 │ EIP-8141 (Frame Transaction)            │ EIP-8130 (Keystore Accounts)                      │
 ├─────────────────────────────────────────┼───────────────────────────────────────────────────┤
 │ 【核心视角】：以「交易执行流」为中心    │ 【核心视角】：以「账户状态与主体权限」为中心      │
 │ • 将交易抽象为一段动态执行脚本 (Frames)│ • 将账户抽象为企业级 IAM 门禁中心 (Actor & Scope) │
 │ • 协议层不强求绑定特定角色体系，        │ • 协议层强力监管身份、到期时间、门禁与 2D 通道    │
 │   只按顺序跑帧，靠 APPROVE 转移授权标记 │ • 认证器纯函数化，验证路径高度收窄与标准化        │
 └─────────────────────────────────────────┴───────────────────────────────────────────────────┘
```

- **EIP-8141（通用执行帧流）**：
  它赋予了开发者极高的组合自由度。任何授权逻辑、代付逻辑、多步调用都可以被放入 Frame 序列中。8141 同样是地道的 Native AA 方案，其通用 Frame 抽象能够作为隐私协议（结合 EIP-8250 Nullifier）和抗量子签名的协议级基础设施。其代价是验证端必须依赖严格的 Mempool 准入规则以防 DoS；
- **EIP-8130（声明式权限中心）**：
  它将复杂度沉淀在状态机（Keystore）中，交易本身高度规整。它通过标准化的 Actor 权限位域（`OPERATOR`, `POLICY`, `NONCE`）和原生 2D Nonce 预编译，收窄了常见交易的验证路径，更易于在特定工作负载下进行针对性优化。

---

### 5.2 业务场景判别力：Meme Launchpad vs Onchain Perps

在评估方案优劣时，必须分清**哪些场景具有判别力，哪些没有**：

#### 5.2.1 为什么 Meme Launchpad 并不具备核心判别力？
Meme Launchpad 的主要交互（首次进入、Passkey 体验、原子授权并买入、平台代付、受限策略打新）在两套方案中**均能良好实现**：
- **原子买入**：8141 靠原子批次帧，8130 靠 Phase 内原子，均可实现；
- **代付报销**：8141 靠独立报销帧，8130 靠已提交 Phase，均可保留费用；
- **原生币买入差异**：8141 的 `SENDER` 帧直接支持携带 `value`；而 8130 的顶层调用 `msg.value = 0`，若 Launchpad 允许原生 MNT 买入，需通过账户合约内部代码转账，SDK 调用方式略有不同；
- **并发特征**：Launchpad 的突发流量本质上是**“大量不同用户（Senders）同时涌入买入”**，并非“同一 Sender 发出海量并发交易”。因此，同 Sender 维度的 2D Nonce 优势无法直接转化为 Launchpad 的整体吞吐优势。

#### 5.2.2 为什么 Onchain Perps 是真正有判别力的分叉点？
Perps 衍生品场景对账户架构的要求极其严苛，存在五个关键技术分叉：

1. **分叉一：直发链上交易 vs 离线签单撮合结算**
   - **若每次下单/撤单都是用户直接发起的链上原生交易**：此时 8130 的 Actor 标准化、2D Nonce 通道与免 Nonce 模式处于核心高频路径，优势显著；
   - **若用户离线签订单，由撮合者或 Keeper 统一上链结算（如 dYdX 模式）**：用户的订单不等于原生 AA 交易，高频路径是订单签名校验与批量结算，交易 Sender 是撮合者。此时用户账户的 AA Nonce 并不在每次报价的瓶颈路径上，两案差距大幅收窄。
2. **分叉二：`POLICY` 受限键缺乏通用 ERC-1271 订单签名权限**
   - 8130 默认的 ERC-1271 校验器只接受 Admin 或 `OPERATOR`。若给 Agent 分配 `POLICY | NONCE`，该密钥**无法通过现有通用 ERC-1271 订单验证器**；
   - 不能加 `OPERATOR`（加了会覆盖门禁）。因此在 Perps 订单撮合场景下，应用层必须定制订单验证器来显式解析 8130 的 Policy Commitment。
3. **分叉三：`POLICY` 是入口门禁，不保证本金安全**
   - 协议保证 `call.to == manager`，但**完全不理解杠杆、滑点、Mark Price 和爆仓风险**。失控或遭恶意操控的 Agent 在合规池内依然可能造成穿仓，必须依赖独立风控合约。
4. **分叉四：Nonce 通道不等于 Actor 隔离与撤单优先级**
   - 8130 的 Nonce 是 `(account, nonce_key)`，多 Actor 共享通道，失陷密钥仍能干扰通道；
   - 撤单绕过 Nonce 阻塞，不等于撤单一定优先于成交入块，订单生命周期仍需独立状态机管理。
5. **分叉五：高吞吐账户锁定与紧急撤权的冲突**
   - 8130 通过账户锁定（Account Lock）换取更高的 Mempool 接纳额度，但**锁定期间授权变更和即时撤权受到严格限制**。不能盲目将所有用户账户长期锁定。

---

### 5.3 状态机生命周期、堆栈与回滚边界全景对比表

| 维度 | EIP-8141 (Frame Transaction) | EIP-8130 (Keystore Accounts) |
|---|---|---|
| **EIP-2718 交易类型** | `0x06` (`FRAME_TX_TYPE`) | `0x79` (`AA_TX_TYPE`) |
| **交易发起人 (`tx.origin`)** | `tx.sender` (在 `SENDER` 帧中) / `ENTRY_POINT` (在 `VERIFY` 帧中) | `tx.sender` (在所有 call 执行期间) |
| **授权认定机制** | **指令级动态跃迁**：执行专有操作码 `APPROVE (0xaa)` 标记 `sender_approved` | **状态声明式门禁**：Keystore 读取 `actor_config` 位域与 Policy Gate |
| **密码学验签路径** | **协议级原生验签**（外层 `signatures` 由协议在 Frame 执行前验证，非 EVM 模拟） | **双 Profile 原生验签**（L1/L2 Profile 均在验证路径原生固化 Canonical 集合） |
| **Nonce 并发机制** | ⚠️ **独立规范为标量 Nonce**；若搭配伴生提案 **EIP-8250** 则可支持 `(nonce_keys, nonce_seq)` 键控通道 |  **原生内置 2D Nonce** (`nonce_key + nonce_sequence`)，另有短时免 Nonce 模式 |
| **免 Nonce 重放保护** | ❌ 不支持 (8250 亦要求 nonce_seq 严格匹配) |  **共识级环形去重缓冲区** (`replay_id` 循环队列，有界共识状态) |
| **批处理与回滚边界** | `ATOMIC_BATCH_FLAG` 帧级原子回滚；仅跳过本批次剩余帧，不回滚前序 `APPROVE` 与组外独立帧 | **两级 Phase 架构** (`[[calls], ...]`)；Phase 内原子回滚，跨 Phase 持久化提交 |
| **状态 Gas 隔离** |  **EIP-8037 双维隔离** (`execution_gas` 与 `state_gas` 独立限额，支持 Refill；`SELFDESTRUCT` 不产生 Refill) | 单一 Gas 限制 + EIP-7623 / EIP-7976 Calldata Floor 计费 |
| **业务失败代付报销** |  **支持**（将报销帧置于原子批次之外即可保留，见 8141 官方 Example 3） |  **支持**（将报销置于 Phase 0 独立持久提交即可保留） |
| **公共 Mempool 限制** | 严格白名单状态依赖；即使使用 EIP-8250，当前 Draft 仍建议单 Sender 最多 1 笔待处理交易 | 允许按通道并发挂单；锁定账户（Locked Account）可获得更高准入频次 |

---

### 5.4 Mantle v3 优先工程验证方向与选型结论

#### 5.4.1 核心选型结论

> **对于以 Meme Launchpad 与 Onchain Perps 为首期核心产品的 Mantle v3，EIP-8130 值得作为优先工程验证方向，主要优势是账户权限、Canonical Authenticators、赞助和二维 Nonce 的标准化集成。这个优势在用户频繁直接提交链上订单交易时更明显，在离线签单、撮合者统一结算的架构中则需要重新评估。**
>
> **EIP-8141 同样是 Native AA 方案，其 Frame 抽象提供更通用的验证、付款和执行组合能力；配套 EIP-8250 明确考虑共享 sender 的隐私应用。不能以“8141 必须在 EVM 中验签”、“无法保留代付报销”或“只能串行执行”作为否定它的理由。最终选型应比较完整的产品与客户端方案，并验证订单授权、紧急撤权、Sponsor 接纳、状态成本及证明成本，而不是仅凭功能表推定性能、安全性与成熟度。**

#### 5.4.2 落地验证必备的四类基准测试（Pre-Production Workloads）

在正式采纳任何一种 Native AA 方案之前，必须基于相同硬件与真实流量模型执行以下四组工程基准测试，而非仅凭静态功能表决策：

| 测试工作负载 | 关键观察指标与评定基准 |
|---|---|
| **1. Launchpad 大量新用户首次打新涌入** | 首次建仓与委托成本、Sponsor Payer 接纳吞吐瓶颈、状态增长率、打新失败时的真实扣费与用户预期符合度 |
| **2. Perps 做市账户高频挂单/撤单/改单** | p95 / p99 交易接纳与撤单延迟、Nonce 通道干扰度、多策略共享保证金槽位冲突率 |
| **3. 极端安全演练：受限密钥失陷与订单流出** | 锁定账户状态下能否即时阻止后续成交；是否存在通过 `OPERATOR`、订单验证器或 Manager 绕过限制的攻击路径 |
| **4. 系统韧性测试：Sponsor 余额枯竭与重放攻击** | Nonce-free 环形缓冲区满载时的系统降级行为、网络重组（Reorg）下的重放恢复、SP1 ZK 证明系统开销与状态一致性 |




