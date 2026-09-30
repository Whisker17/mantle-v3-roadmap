# 04. L3 方案：同一产品在 App-specific L3 上

> 架构基准：`0-narrative/09-native-l3-architecture-and-errata.md`。文档状态是路线比较提案，尚未选型。未定参数不写成已定。
> 产品语义与 `notes/02`、`notes/03` 相同：agent 发币 + 交易。换执行域，不换用户故事。

## 1. 为什么会想到 L3

09 §0.3：适合 L3 的产品内部闭环、可预存资金、不靠逐笔跨层原子调用。Launchpad 是否独立链取决于状态增长、交易负载、排序需求。高频 Perps 更像候选。

Agent meme 的压力在 **单池热点 + 高失败率 + 自定义开盘排序**，不在永续持仓。L3 有价值当且仅当这些压力在 L2 上已观察到，而不是设计阶段预设。

基础方案沿用 09 表：L2 有效性证明结算、原生状态机、独立 Sequencer、预存资产、Inbox/Outbox、可恢复数据发到 L2。不把远程授信、Solver 垫资、完整 JSON-RPC 写成必需。

## 2. 状态机（L3 内）

同一执行域，毕业不先退出到 L2（09 §3.1）。

账户：

- `Owner`（人类，L2 充值时绑定）
- `AgentSession{key, whitelist, budget, expiry, nonceKey}`
- `Balances[account][asset]`：USDC、各 meme token
- `FeeVault[token][recipient]`

发行：

- `Token{supply, quote, recipients[], phase, holderCap, taxSchedule}`
- `Pool{reserveQuote, reserveToken}` 或等价曲线参数
- `LaunchNonce` / 配额是 L3 状态，不是链下 Bankr 计数器——配额可证明

交易：

- 无完整 CLOB 也可：池 + 市价 swap 作为原生交易类型 `Swap` / `Deploy`
- 若做 FBA 开盘：`AuctionWindow` 内收集 `Bid`，窗口结束统一价成交。这是 L3 相对 L2 的真实增量

规则必须确定性：不用节点本地时钟；开盘窗口用 L3 区块高度。外部价格（若股票 quote，本产品默认不做）要先成为规范输入。本产品默认 quote = 预存 USDC。

## 3. 充提

### L2 → L3

按 09 §4.2：

1. 用户（人类或已在 L2 的 AA）把 USDC 锁进 **本 L3 的 Escrow**，Inbox 记目标账户、数量、消息 ID
2. L3 Observer 按确认策略纳入
3. L3 消费一次，增加 `Balances`
4. 证明把该充值绑到父链输入

Agent **不能**在 L2 无额度时在 L3 发币。预存是产品成本：新 agent 要先等充值确认。

充值确认策略、要不要在 unsafe 块上给暂定余额，09 要求能回滚；未验证前用保守策略。不写「N 秒可交易」。

### L3 内

deploy / swap / claimFee / revokeSession 本地执行。盈利来自对手方或池储备，不能在 L2 凭空增发。

### L3 → L2

按 09 §4.4：L3 先扣可提余额 → 进批次 Outbox → L2 Verify 后累加待领 → Claim 一次性支付。重复执行失败。

meme 持仓退出：提的是 L3 账本上的 token 映射资产。若 token 只存在 L3，L2 兑付的是「L3 凭证」或先在 L3 卖掉再提 USDC。推荐 **只允许提 USDC**（以及协议登记的 L2 资产）。L3 内 meme 不桥回 L2，避免无流动性的「L3 ERC-20」冒充可组合资产。

这是明确的产品选择：L3 闭环交易，毕业到 L2 Uni **不是**热路径。若需要毕业到 L2，那是异步：L3 锁仓 + 发消息 + L2 建池，**非原子**，有 MEV 窗口——正是 L3 为热点付出的可组合性代价。

## 4. 确认层级（四类时间，09 §4.1）

| 层级 | agent 看见 | 不是 |
|---|---|---|
| Sequencer 回执 | deploy/swap 暂定成交 | 不是 L2 已验证 |
| L3 执行确认 | 进入 L3 历史 | 不排除父链重组 |
| L2 接受证明 | 状态承诺已上 L2 | 不等于 L2 块最终 |
| 父层最终性 | 所选最终性条件 | 不替代 DA/升级检查 |

API 必须返回 `confirmationLevel`。禁止把签名或预确认叫结算。

延迟、证明间隔、提款等待：**不写数字**。09 禁止用别的系统的 15 分钟/1 小时冒充本方案。

## 5. Agent UX 在 L3 上怎么变

- Gas：用户免 L3 gas，费用走交易税或运营预算（09 §3.3）。必须有配额，否则零成本 DoS。
- 失败不扣费：Sequencer 可在入队前丢弃必然失败的 `Swap`（这是 L3 相对路线一的合法增量）。验证失败不上批次。
- Session nonce：原生二维 nonce 可做进状态机，不必 RIP-7560。
- 开盘 FBA / 分桶：独立排序器可做；FIFO 不能证明未隐瞒订单（09）。

人类次级：人类只在 L2 充提和强制退出；日常不碰 L3 API。

## 6. 故障退出（09 §6，按 Launchpad 改写）

| 故障 | 处理 | Launchpad 特有 |
|---|---|---|
| L2 重组撤销充值 | 回退共同父链历史 | 未稳定充值打出的币必须能回滚或该充值不得用于 deploy |
| Sequencer 停机/审查 | L2 强制请求，超时受控退出 | 退出兑付 USDC 权益，不兑付「按旧价算的 meme 市值」 |
| Prover 积压 | 限制未证明敞口，降接单 | 不把暂定成交当可提现 |
| DA 失败 | 不正式结算 | 公开数据能重建账户 USDC 与 token 余额 |
| Mantle L2 停机 | 停跨层；本地按敞口上限或停 | L3 不是独立结算层 |
| 业务规则缺陷 | 停新发行，版本恢复或退出 | 不能快照回滚已结算成交 |

Launchpad 退出比转账链简单、比 Perps 简单：无未平仓杠杆。仍不能「证明旧余额就提走全部」若其中包含未结算拍卖或未退款的失败发行。失败发行先退款或保留可领取承诺（09 §3.3）。

不把远程授信、Solver 快速充值写成闭环必需。它们改变风险承担，见 09 §7。

## 7. 与 L2 方案的产品差异（不是性能数字）

L3 得到：独立排序、失败丢弃、session nonce、热点不外溢到 Mantle DeFi、免 L3 gas。
L3 失去：与 L2 Uni / 借贷 / 任意合约原子组合；资金预存；独立运维与证明；meme 默认困在 L3。

09 拓扑图画了 Launchpad L3 作为可选项，不是已决定要建。
