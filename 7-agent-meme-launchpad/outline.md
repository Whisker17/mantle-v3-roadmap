# Outline：Agent 第一公民 Meme Launchpad

> 一页规划。论据与双轨设计见 [agent-meme-launchpad-report.md](./agent-meme-launchpad-report.md)。独立于 Tape。

## 1. 产品是什么

一句话：Mantle 上 agent 发币、agent 买卖；人类只存款、签 session、撤销。

最小闭环：

1. 人类把 USDC 放进 AgentAccount，签一条受限 session（工厂 + router、预算、过期）。
2. Agent 调 `Deploy`：一笔得到代币、池、手续费路由。费的一腿进该 agent 的 runtime vault。
3. 其他 agent 调 `Buy` / `Sell`。模拟失败不上链。开盘有持仓顶和衰减税。
4. 人类随时 `revoke`。不必每笔确认，不必为发币付 gas。

不做什么：Tape 股票 quote；人连钱包再挂 bot；人格文件当发行条件；API key = 钱包根权限；NL 当抢开盘热路径。

## 2. Agent 第一公民怎么落地（账户，不是曲线）

主体：`Owner`（人）→ `AgentSession`（链上限额）→ 工厂 / router。

- Session：合约白名单、函数选择器、单块/累计花费、有效期、可撤销。泄漏不能转走 quote 本金，也不能改 fee recipient。
- 发币与交易同一 session。`claimFees` 只进 runtime vault，vault 再按策略拨推理/gas。
- 接口分层：结构化 `Deploy` / `Buy` / `Sell` / `Status` 是热路径；自然语言只给人类。
- 发行机制抄现成：Uni v4 单边池 + 永久锁 LP（Clanker/Bankr 家族）。曲线注意力若要，用同一池 hook 相变，不迁池。

## 3. Launchpad 要具备的东西

| 层 | 要有 |
|---|---|
| 协议 | 工厂、固定供应 ERC-20、v4 池、开盘持仓顶、短窗口衰减税、多 recipient 费路由 |
| 账户 | 4337/7702 session、Paymaster（发射赞助，交易可用 USDC 扣 gas） |
| API | 幂等 `clientRequestId`、模拟不占配额、未广播失败退回、机器可读失败码 |
| 发现 | `TokenLaunched` / `Swap` 事件订阅；执行不经过 LLM |
| 配额 | 应用层发射限额（对照 Bankr：滚动窗口、钱包年龄；数值待决） |

## 4. Onboarding（两条，不要合成一条）

**人（次级）**

- 入口：Bybit AI 对话层。心智对齐 AI Subaccount：隔离资金、上限、碰不到主账户。
- 映射：Bybit 子账户 USDT → 链上 AgentAccount session。不是把交易所子账户当成链上根钥。
- 允许：发现已上线的币、确认后买、给 agent 拨预算。
- 禁止：在 Bybit 聊天里给 8000 万 KYC 用户直接发 meme（合规会卡死产品）。

**Agent（第一公民）**

- 开户：拿到 session，不是拿到人格。
- 发现：自建类似 Moltbook 的 agent 论坛（帖子/评论/分区；人可围观、默认不发帖）。只做社交图谱和冷启动，不替代发射 API。
- 热路径始终是结构化 API。论坛、聊天、Bybit AI 都是发现或确认，不是撮合。

## 5. 对 Mantle chain infra 的额外需求

v1 维持现有 L2，不先开 L3。

必须有（应用层即可）：

- Session + Paymaster
- v4 hook：持仓顶、衰减税、费路由、LP 无 withdraw
- 事件可订阅

L2 基线做不到、不当 v1：协议级失败不入块、热点分桶、FBA、二维 nonce、独立费市场。

翻转才上 L3（需 Mantle 自己测到，不搬 Robinhood 数字）：

- **F1** 单次发射挤占全链，无关交易失败率 / base fee 持续抬升
- **F2** 做市 agent 因 revert 扣费或一维 nonce 退出
- **F3** hook 之后开盘仍被共址 bot 垄断，且其短窗利润超过创作者费

签名 ≠ 结算。预确认 ≠ L2 最终性。L3 不能覆盖 L2 状态根。
