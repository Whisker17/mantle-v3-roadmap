# 02. Agent 第一公民：产品需求与反目标

> 输入：`notes/01-agentic-landscape.md`，以及仓库 `2-meme-launchpad/`、`5-agent-account-model/`。不改写 Tape。

## 1. 产品是什么

Mantle 上一条 **agent 默认在场** 的 meme 发行与现货交易协议。

- 发币者：agent 调结构化 API / SDK 部署代币，手续费进入该 agent 的受限地址。
- 交易者：别的 agent 用同一账户模型并发买卖、做市、抢开盘。
- 人类：存款、设策略、看仓、必要时一键停机。人类不是默认点击者。

不是 Tape（股票 quote 情绪一级市场）。不是「人连钱包，再挂一个 bot」。

## 2. 硬需求

### R1 发币是一等 API，不是聊天彩蛋

Bankr 证明 NL 能发币，也证明聊天默认链和 CLI 默认链会分叉。Eliza 证明框架不会自动长出发射台。

必须同时有：

- `Deploy(name, symbol, quote, feeRecipients[], session)` → `token, pool, tx`
- 幂等 `clientRequestId`；模拟不占发射配额
- 机器可读回执：失败原因码（配额、余额、持仓顶、未上链）

NL / 社交只做人类次级入口，且必须声明目标链，禁止暗默认。

### R2 agent 有独立身份，但钱不是它的根钥

Clanker Droid 有身份不能动钱；Bankr / Eliza 能动钱但等于根权限。两者都不可用。

需求：

- 每个 agent 一个链上账户（AA 或 7702 委托），人类主账户是 owner
- agent 持有 **session**：合约白名单、函数选择器、累计/单块花费、有效期、可撤销
- 发币费接收地址 ≠ session 热钥；费进 vault，vault 再按策略拨推理预算

泄漏 session 不能转走 quote 本金，也不能改 fee recipient。

### R3 受限资金，而不是「先把私钥给进程」

复用 `5-agent-account-model` 的 session 结构，改成 Launchpad 语义：

- `target_whitelist`：工厂、池、router
- `max_spend_per_block` / `total_budget`（以 quote 计）
- `valid_until`
- 可选 `max_drawdown_bps`：触发则 session 失效

人类次级参与 = 签这类授权，而不是替 agent 点确认。

### R4 发币与交易同一账户模型

Virtuals 把「agent 标的」和「agent 钱包」拆开，结果发币仍是人类。本产品同一 session 必须能：

- `deploy`
- `buy` / `sell`（曲线或 v4 池）
- `claimFees`（仅 fee 角色）
- 不能 `transfer(quote, arbitrary)`

### R5 并发：多标的、多 agent、互不阻塞

仓库 03 已写 2D nonce。L2 若没有协议级 2D nonce，应用层也要：

- 每个 session 独立 nonce 空间，或每个 token 一个子账户
- 一个池 revert 不得卡住另一个池的 session

交易 API 禁止「先等 LLM 再发交易」。Bankr Agent API 是慢路径；Wallet API 才是交易路径。

### R6 失败尽量不扣费

Meme 开盘失败率高（仓库 launchpad-chain-spec：抢跑碰撞 50%+ revert 是画像，不是本产品测量）。Agent 以高频失败为常态。

- 模拟失败：不上链、不扣 gas、不占发射配额（Bankr 已做出发射侧）
- 执行失败：L2 上只能尽力（4337 验证失败不扣；执行失败仍扣）。必须在文档写清这条边界
- 发射配额：未广播失败退回

### R7 手续费能养运行时

Clanker Droid 默认 10% LP；Bankr 创作者约 0.665% 成交额；Virtuals 70% 的 1%。

- 发射时写死 `feeRecipients[]`（bps 合计 10000）
- 至少一类接收者标记为 `runtime`，只收 quote
- 人类不能事后把 runtime 份额改到自己 EOA（可改到新的 agent vault，需 owner 签名）

### R8 开盘不公平是协议问题，不是「请 agent 跑得更快」

可迁移的现成件：Clanker MEV 模块、Virtuals 99%→1% 税、Bankr 5 分钟 2% 持仓顶。本产品至少要有：

- 开盘窗口的持仓顶或单地址买入顶
- 短窗口衰减税或等价拍卖
- 不把「谁的 RPC 更近」当成产品能力

### R9 人类次级，但必须能停机

人类可：充值 quote、签发/撤销 session、领取自己那份费、紧急 `revokeAll`。
人类不必：每笔确认、为 agent 付 gas（赞助或从 quote 扣）、在 Farcaster 发帖才能发币。

### R10 发现与执行分离

Trojan/Axiom 的发现流可做索引；执行必须是无 LLM 的 RPC/HTTP。Agent 订阅 `TokenLaunched` / `Swap`，自己决定下单。

## 3. 反目标

| 不做 | 原因 |
|---|---|
| Tape 股票 quote 产品 | D3 仍属于 Tape；本轨道独立 |
| 人是默认用户再挂 bot | 那是 terminal 支柱，不是本产品 |
| 把人格文件当发行条件 | Virtuals/daos.fun 已证明这会变成 meme 皮 |
| API key = 根钥（Bankr 模型） | 与 agent 账户调研直接冲突 |
| 环境变量私钥（Eliza 默认） | 爆破半径 100% |
| 强制 Farcaster/X 才能发币 | 把机器第一公民卡在社交图谱 |
| 为「agent 币」做 ACP 商务托管 | 超出 Launchpad 范围 |
| 声称零 MEV / 签名即结算 | 约束 |
| 未测就把 L3 写成必需 | 09：Launchpad 是否独立链看负载与排序 |

## 4. 最小用户故事（教学，非生产参数）

1. 人类 H 在 Mantle L2 存 10,000 USDC 到 AgentAccount A。
2. H 签 session S：仅工厂+router，预算 500 USDC，2 小时，单块 ≤ 50 USDC。
3. Agent 调 `Deploy` 发 `FOO/USDC`，fee 80% runtime vault / 20% 协议。Base 路径赞助 gas。
4. 另一 agent B 用自己的 session 买 `FOO`，模拟失败不扣费；成交后持仓顶生效。
5. 费进入 A 的 runtime vault；A 只能按策略把 USDC 拨到白名单 paymaster/推理合约。
6. H 撤销 S，A 立即不能再买，已部署的 `FOO` 仍在，费仍按发射时路由。

## 5. 对链层的诉求（先产品后链）

产品不先要求 L3。它对执行环境的诉求按「缺了就会坏 agent 第一公民」排序：

1. 受限 session 与赞助 gas（应用层 AA 就能起步）
2. 结构化发币/交易 API 与未上链失败退配额（链下+工厂）
3. 开盘持仓顶 / 衰减税（v4 hook 或工厂）
4. 失败尽量不入块（要 sequencer 策略，L2 基线未必有）
5. 热点隔离与独立费市场（共享 L2 做不到协议级）
6. 自定义排序 / FBA（共享 L2 默认没有）

1–3 定义 MVP。4–6 是 L2「做不到什么」和 L3 翻转条件的输入。
