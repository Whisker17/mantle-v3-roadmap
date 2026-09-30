# 03. L2 方案：现有 Mantle L2 上的 agent meme Launchpad

> 基线：不改 Mantle 执行层、排序规则、证明程序（09 路线一）。应用合约 + 链下 agent API + 既有 AA/Launchpad 原语。
> 复用：`2-meme-launchpad/` 的 v4 hook / 永久锁 LP / 衰减税；`5-agent-account-model` 的 4337/7702 session 与 Paymaster。不把 Flashblocks、2D nonce、sequencer 分桶写成已有能力。

## 1. 组件

```
人类 owner
  │ 存款 / 签 session / 撤销
  ▼
AgentAccount (ERC-7579 或 EIP-7702 委托)
  │ UserOp 或 7702 tx；Paymaster 可赞助
  ▼
Launch Factory ──► Token + Uni v4 Pool + Hook + FeeVault
  │
  ├── Agent API（链下）：deploy / quote / buy / sell / status
  └── Indexer：TokenLaunched / Swap 事件 → bot 订阅
```

没有独立 sequencer。没有 Inbox/Outbox。资产始终在 L2。

## 2. 发币

推荐 **Clanker/Bankr 型：无曲线、单边流动性、立刻可交易**。理由：仓库已指出曲线毕业迁移是 MEV 窗口；agent 发币要一笔原子完成，不要「再迁一次」。

若要坚持曲线注意力事件，用既有 **v4 hook 相变**（`2-meme-launchpad/chain-infra/launchpad-chain-spec.md` 4.2）：同一池 Phase1 曲线 / Phase2 CL，不跨合约搬钱。两条都合法；默认无曲线，因为 agent 自我供血不依赖「毕业直播」。

发射交易（原子）：

1. 部署 ERC-20（固定供应，不可增发）
2. 建 v4 池，quote = USDC（或协议登记的稳定币；不默认 MNT，避免 agent 再多一条 gas 腿）
3. 剩余供应作为单边 LP 锁进无 withdraw 的 locker
4. 写 `feeRecipients`（含 runtime vault）
5. 可选：开盘 T 秒持仓顶、衰减税模块

`msg.sender` 必须是 AgentAccount。工厂检查 session 允许 `deploy`。

配额（应用层，学 Bankr，数字为设计假设）：每账户滚动 24h 发射次数上限；模拟不占配额；未广播失败退回。假设待产品定，不写成已测。

## 3. 交易

Router：`buyExactQuote` / `sellExactToken`，内部 uni v4 swap。Agent 一笔完成 approve+swap（7702 批处理或 4337 executeBatch）。禁止先 approve 再等下一笔。

开盘保护在 hook，不在 bot：

- 短窗口衰减税或 sniper auction（Clanker 模块形态）
- 窗口内地址持仓顶（Bankr 2% 是对照，本产品百分比待决）

做市：agent 对同一池连续 swap，不提供链上订单簿。L2 不做 CLOB。

## 4. 账户与 Gas

MVP（零改链）：

- Owner EOA / 已有 AA
- Session 模块：白名单工厂+router，预算，过期
- Paymaster：协议赞助发射；交易允许用 USDC 扣 gas（`5-agent-account-model` 中层；4337 即可，不必 RIP-7560）

已知缺陷（01、02 已写，L2 基线消不掉）：

- EOA/7702 仍受一维 nonce 阻塞
- 4337 Bundler 额外延迟与 EntryPoint gas
- 执行失败仍扣 gas（验证失败才不扣）

## 5. 排序与失败

现有 L2：共享区块、普通 mempool/sequencer 策略。Agent API 必须：

- 发送前 eth_call / UserOp 模拟
- 模拟失败不提交
- 提交后超时标记 `unknown`，不假装失败不扣费
- 不提供「失败不入块」保证——那是 sequencer 特权，09 路线一不包括

## 6. 人类次级

前端可以没有「连接钱包买币」作为主路径。人类用：

- 存款到 AgentAccount
- 仪表盘签 session
- 紧急撤销

零售若要从网页买，走同一 router，不另做一套权限。

## 7. 复用清单

| 原语 | 来源 | 用法 |
|---|---|---|
| Uni v4 hook / 永久锁 LP | `2-meme-launchpad` + Clanker | 发行与反 rug |
| 衰减税 / 持仓顶 | 同左 + Bankr | 开盘 |
| Fee Key / 多 recipient | Clanker locker、Bankr | runtime 供血 |
| 4337/7702 session、Paymaster | `5-agent-account-model` Phase 1 | 受限资金与赞助 |
| Bot 发现流 | `terminal/` 需求，不实现终端 | Indexer 事件 |

不复用 Tape Spend-Gate、xStocks quote、公司行动 hook。

## 8. L2 做不到什么

1. **协议级失败不入块 / 不扣费。** 只能模拟后少发垃圾。
2. **热点隔离。** 单币开盘与全链抢同一 blockspace；Robinhood Chain base fee 外溢是仓库已有反例，不是本产品测量。
3. **独立排序（FBA、cancel 优先、按合约分桶）。** 除非改 sequencer——那就不再是路线一。
4. **2D nonce。** 无 RIP-7560 / 原生 AA。
5. **独立费市场。** agent 微利做市与人类转账同一 gas 拍卖。
6. **把「预确认」当成结算。** 即使以后加 Flashblocks，仍不是 L2 最终性。

这些是 L3 方案要接的缺口，不是 L2 的实现 TODO。
