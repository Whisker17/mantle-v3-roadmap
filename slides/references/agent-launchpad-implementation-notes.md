# Agent Meme Launchpad：实现方案与提案核对

本稿支撑 Section 1 的 P11、P11A、P13，区分产品目标、可以先用的账户/服务实现，以及需要改链的候选方案。方案是设计方向，不表示 Bybit、Moltbook 或 Mantle 已完成对应集成。

## 1. 同一个 $100 USDC 任务，两种阅读方式

用户流程固定为六步，P11 与 P11A 使用相同的步骤编号与连接关系：

1. **下指令**：用户在拟议 Bybit App 入口提交“用 $100 USDC 去 Mantle 打新 Meme”的目标。
2. **钱包接单**：已经创建的 Agentic Wallet 是 AA 账户，接收经过用户确认的任务授权与可用预算。
3. **购买 Meme**：通过可执行路径将 USDC 换成所选 mStocks，再用 mStocks 买 Meme；多标的任务仍受总预算约束。
4. **社区传播**：Agent 在拟议 Moltbook 接入中介绍资产和自己的参与情况，其他 Agent 独立判断是否参与。
5. **达到条件后毕业**：真实买盘满足发行规则时，合约推进毕业并接续 mStocks/Meme 二级交易；优先评估 v4 同池阶段转换，但必须完成真实储备、价格和 LP 配置。详见 [曲线与 v4 核对](bonding-curve-v4-graduation.md)。
6. **卖出与汇报**：按用户策略卖出 Meme，得到 mStocks，按需换回 USDC，汇报实际盈亏和剩余资金。

$100 USDC 是示例预算，不是利润目标。是否包含兑换费用、gas 代付回收费用以及如何处理预算不足，需要在任务确认时明确。Bybit 内部余额不会自动变成链上可支配资金，充提或其他授权资金路径要实际完成；不要把 Agent 的 App 指令画成获得交易所主账户私钥。

社区传播是链下信息与分发动作，不等于链上买盘。真实参与者是否购买、能否毕业、卖出能否盈利是分别发生的结果。毕业只改变交易阶段，盈亏在实际买卖及费用结算后确定。

## 2. 权限与资金：Session keys + 账户策略

- Owner 保留根权限，Agent 使用限时、可撤销的 Session Key；限制合约、函数、资产、接收地址、额度和成交条件。
- 预算应跨并发任务累计检查，不能给每条 nonce 通道重复分配整笔 $100。
- 每个任务先记录授权与资金到账状态，再进入执行。授权撤销以实际生效状态为准。
- 可通过智能账户或 7702 委托代码实现这些策略；7702 本身不规定完整 Session Key 行为，委托也不会因一笔交易结束而自动过期。

## 3. 多任务：2D / keyed nonce 的候选路径

| 方案 | 核对到的能力 | 本次可写入 slide 的结论 |
| :-- | :-- | :-- |
| ERC-4337 | EntryPoint 支持 key + sequence 的分组 nonce | 可先在账户与提交服务路径验证独立操作队列 |
| EIP-8130：Keystore Accounts | 草案的 2D Nonce Storage 定义 `nonce_key` 选择通道、`nonce_sequence` 为通道内序号 | 是协议原生多通道 nonce 的候选路线 |
| EIP-8141：Frame Transaction | 本体仍检查 sender 的标量 nonce | 不能把多帧或二维 gas 预算直接说成 2D nonce |
| EIP-8141 + EIP-8250：Keyed Nonces for Frame Transactions | 8250 用 `nonce_keys` 与 `nonce_seq` 扩展 8141；不重叠非零 key 集合之间没有同一 nonce 序列的重放依赖 | 可以作为原生多任务操作流的另一条候选路线 |

**需要同时说明**：EIP-8250 的当前草案仍保留公共 mempool 对同一 sender 最多一笔待处理 frame 交易的保守指导，多 pending 的接纳策略需另行设计。不同 nonce 通道也可能访问同一余额、代付预算或市场状态，所以它们减少账户级顺序依赖，不保证物理并行执行、即时纳入或业务状态完全隔离。

本次读取的 8130、8141、8250 均标为 Draft。Mantle 启用原生交易类型需要客户端、排序接纳、验证、RPC 和钱包配套；这与部署一个账户合约不是同一项工作。

官方依据：

- [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)，Nonce。
- [EIP-8130](https://eips.ethereum.org/EIPS/eip-8130)，2D Nonce Storage。
- [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)，Behavior 中的 sender nonce 检查。
- [EIP-8250](https://eips.ethereum.org/EIPS/eip-8250)，Abstract、Nonce State、Mempool、Rationale 与 Security Considerations。

## 4. Gasless 体验：明确谁实际付费

Gasless 表达的是用户不必另外准备原生 gas 资产，不是网络执行没有成本。可用 Paymaster / Gas Sponsor / Relayer 承担或垫付链费用，按规则赞助或向用户回收。

这能让持有 USDC 或 mStocks 的用户完成任务，但代付方仍需预算、限流和拒绝无效请求的规则。Moltbook 发帖本身也不应被含糊写成链上 gasless 交易，社区接口和链上账户分别处理。

## 5. 高峰隔离：合约 gas 配额是 LFM 方向的第一层

用户提出的目标是让一场 Meme 热点尽量少影响其他应用。可以分两步描述：

1. **容量隔离**：排序器在纳入前对业务合约/资源域设置每区块 gas 上限，给其他操作保留容量，配合限流、排队和预算。仅在合约执行后通过 revert 限额，会继续消耗区块 gas，不能达到同样目的。
2. **费用隔离**：进一步研究 Local Fee Market，即按相关资源的争用程度形成局部费用规则。若底层仍共用全局 base fee，单个合约的 gas 上限不能保证其他交易手续费不涨。

配额不能只按交易 `to` 地址粗分：AA 的 EntryPoint、聚合器 Router 或共享 PoolManager 可能服务多个产品。需识别实际业务资源，处理代理合约绕过和共享状态，否则可能把无关业务一起限流。v4 集成时尤其需要按 PoolId/业务域评估，而不是对整个 PoolManager 一刀切。具体归集与计量方案是链设计工作，本次不指定百分比或宣称已实现完整 LFM。

因此 P13 使用 **“合约 gas 配额 / LFM”** 作为递进研究方向，并在注释中保留“配额先隔离容量，费用隔离另需定价”的区别。

## 6. 自动提交与重试：幂等任务队列 + nonce + 回执恢复

建议使用一个可持久化的任务状态服务，配合链上去重与执行结果，而不只是在超时后重新发一笔买单。

- **任务标识**：为每个业务动作生成稳定的 `requestId / operationId`，绑定账户、授权、资产、金额和操作内容；重复 API 请求返回同一任务状态。该 ID 必须与签名/授权内容关联，避免相同 ID 被用于不同操作。
- **交易关联**：保存 `requestId → nonce 通道 / txHash 或 userOpHash`。同一逻辑动作的重广播或替换沿用相应 nonce 语义；重新分配 nonce 之前先确认旧操作状态。
- **链上去重**：nonce 阻止同一签名交易重复生效。若一个业务动作可能通过不同交易被再次提交，还需要账户或应用层的已处理 `operationId` 检查；只有数据库去重不够。
- **执行回执**：普通交易查询 `eth_getTransactionReceipt`；4337 路径可查询 `eth_getUserOperationReceipt`。调用被接收、交易被纳入和买入成功必须分别记录，批量操作需检查实际调用结果。
- **断线补查**：订阅事件后保留游标，断线后用历史日志补齐；遇重组按确认级别回退本地状态。事件重放是恢复已发生的事实，不是再执行一次买入。
- **受控重试**：区分未确认、失败、过期、已成交与已撤销；重新报价后仍要检查用户授权、总预算和最小到手量。重试有次数与退避限制，失败需归因。

P13 上屏只保留“请求 ID + nonce + 回执恢复”，细节留在本稿或讲述备注。这里不需要为整套业务可靠性发明一个新的 EIP。
