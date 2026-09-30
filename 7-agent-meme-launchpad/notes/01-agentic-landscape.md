# 01. 市面 Agentic 产品如何构建

> 取数日期：2026-09-17。可信度：`[一手]` 官方文档 / SDK / 合约仓库；`[仓库]` 本仓库既有调研；`[二手]` 媒体或第三方综述。未知标为假设或待决。
> 问题：若第一公民是 **发币者 + 交易者** 的 agent（人类次级），现有产品各自把账户、发币、交易、Gas 和权限放在哪一层。

## 0. 读法

五件对照物不是同一类产品。不能把「agent 币」和「agent 发币」混成一件事。

| 对照物 | 实际卖的是什么 | 第一公民是谁 |
|---|---|---|
| Clanker | 链上发币工厂 + 社交触发器；Droid 是发币附赠的运营 agent | 人类创作者；Droid 不能动资金 |
| Virtuals | 把 agent 做成可融资标的（曲线 + 毕业 + ACP 商务） | 被代币化的 agent；发币者仍是人类 founder |
| Bankr | 托管式 agent 运行时：自然语言 / API / CLI 发币和交易 | 绑定 Bankr 钱包的 agent；人类是账户主人 |
| ElizaOS / ai16z | 开源 agent 框架 + 一次失败的「AI 基金」代币实验 | 运行时是 agent；密钥是人类环境变量 |
| Photon / Trojan / Axiom | 高频交易终端，几乎不发币 | 人类交易员；bot 是加速器 |

对 Mantle 新产品有用的，是它们的 **接口形状**，不是它们的叙事。

---

## 1. Clanker

### 1.1 谁是第一公民

人类创作者。发币入口是 Farcaster `@clanker`、网站 `clanker.world/deploy`、或 `clanker-sdk`。[一手] Clanker 文档把 Droid 定义为「随代币附带的 AI agent」：有自己的 Farcaster 账号、用 LP 奖励支付推理，但 **「not autonomous with your money」**——每条 cast 要人确认，钱包被沙箱。[一手] `clanker-devco/DOCS/droids/README.md`

Moltbook 那一波（仓库 `2-meme-launchpad/benchmarks/base-ecosystem.md`）说明 **机器社交图谱可以当冷启动**，不等于 Clanker 把 agent 当成资金主体。

### 1.2 账户 / 身份

- 发币身份：Farcaster FID 或 EOA。早期限制「有 Neynar 分数的 FID 每天 1 枚」；SDK 路径是 `PRIVATE_KEY` + viem wallet。[一手] SDK README；[二手] 旧 Farcaster bot 文档
- Droid 身份：独立 Farcaster 账号 + Base 上的 runtime 钱包。runtime 只收 USDC 付推理，不持有主仓。[一手] Droids funding
- 没有 session key、没有 7579 模块。agent 不是链上账户。

### 1.3 发币接口

内核从 v0 到 v4.1 不变：**固定供应 ERC-20（100B）→ 剩余供应作单边流动性进 AMM → LP 永久锁 → 手续费按 bps 分给创作者**。[仓库] `2-meme-launchpad/benchmarks/base-ecosystem.md`

- 无 bonding curve、无毕业。v4 用 Uniswap v4 hook。
- SDK：`clanker.deployToken(config)` / CLI `clanker-sdk deploy --name … --chain base`。[一手]
- REST：公开检索 + Partner `x-api-key` 部署 + Bearer（Farcaster / Privy）。[一手] DOCS API
- Droid 可与代币同发；LP 奖励默认 10%（100–5000 bps）切到 droid runtime。[一手] droids/funding.md
- 仅 Base 主网 + USDC 配对才允许 Droid。[一手]

### 1.4 交易接口

Clanker 自己不是交易终端。交易走 Uniswap v4；兼容列表在文档 `compatible-trading-platforms`。MEV 模块（2-block delay、sniper auction、descending fees）写在 hook 里，上限约 2 分钟。[仓库] base-ecosystem.md

### 1.5 Gas 与权限

- 部署者付 ETH gas。没有官方 Paymaster。
- LP locker 无 withdraw：创作者拿不到本金，只能拿费。[仓库]
- Droid 权限被明确裁掉：不能动用户资金。[一手]

### 1.6 链上 vs 链下

链上：工厂、hook、locker、FeeLocker、MEV 模块。
链下：Farcaster 解析、网站、Droid 推理与 Farcaster 发帖、Partner API。

### 1.7 可迁移 / 不可迁移

可迁移：单边流动性 + 永久锁 LP + 可插拔 MEV hook；SDK/CLI 作为 agent 发币原语；「手续费养 agent 推理」这条资金回路。
不可迁移：Farcaster / Neynar 身份门；Droid 硬编码 Base+USDC；把 agent 做成不能动钱的吉祥物——这正好是本产品要反过来的点。

---

## 2. Virtuals Protocol

### 2.1 谁是第一公民

被代币化的 agent。白皮书 Capital Formation Layer：agent 被当成可融资经济主体，曲线配对 `$VIRTUAL`，42,000 VIRTUAL 毕业到 Uniswap V2，LP 锁 10 年。[一手]

发币动作仍是人类 founder 在 `app.virtuals.io/create` 点模块。ACP 才是 agent↔agent 商务层。

### 2.2 账户 / 身份

- 发射：人类钱包。
- 注册：Console / ACP SDK / CLI 给 agent 建钱包，权限事后配。[一手] app.virtuals.io/acp/registry
- Base MCP 插件：SIWE 拿约 1 小时 JWT，之后卡片、邮箱、agent 运维走 Virtuals 后端，不经 Base MCP。[一手] 曾见于 Base agents 文档；页面已迁移，细节待复核
- ACP v2：Account / Job / Memo / Payment 模块 + LayerZero 跨链。[一手] `Virtual-Protocol/agent-commerce-protocol`

### 2.3 发币接口

- 创建免费；Capital Formation / SOL Launch 等模块 10 VIRTUAL，Launch Radar 100 VIRTUAL。[一手]
- 曲线立刻可交易；无预售、无白名单（模块另开除外）。
- 1% 交易税：70% 创作者、30% 金库。
- 可选 99%→1% 狙击税，窗口 0s / 60s / 10min / 98min；税回购并 3+9 月归属团队。[一手] Launch Mechanics
- 毕业：曲线凑满 42,000 VIRTUAL → Uniswap V2，LP 锁 10 年。
- Fee Delegation：别人可代发，创作者费留给 builder。[一手]

### 2.4 交易接口

平台内买卖；毕业后任意 DEX / 聚合器 / bot。ACP 是另一条路：Request → Negotiation → Transaction → Evaluation，资金进托管，Evaluator 对照 PoA。[一手] Commerce Layer

### 2.5 Gas 与权限

发射方付 gas。没有面向 agent 的 session spend limit。Pre-buy 最多 100% 供应，默认 1 月 cliff + 12 月归属。[一手]

### 2.6 链上 vs 链下

链上：曲线、毕业、LP 锁、ACP escrow。
链下：模块配置 UI、agent 运行时、Evaluator、部分 MCP 会话。

### 2.7 可迁移 / 不可迁移

可迁移：手续费养主体；狙击税窗口可选；Fee Delegation（agent 代人发币、费留给 builder）；毕业锁 LP。
不可迁移：强制 `$VIRTUAL` 计价与 42k 阈值；「agent = 标的」而不是「agent = 发币/交易客户端」；ACP 评估器市场对本 Launchpad 过重。
**反例价值**：Virtuals 证明「给 agent 发币」很容易变成「给一个叫 agent 的 meme 发币」。本产品禁止把「有没有人格文件」当成发行条件。

---

## 3. Bankr

### 3.1 谁是第一公民

绑定 Bankr 钱包的 agent。文档原话：发币是 **「This is how agents fund themselves」**。[一手] `docs.bankr.bot/token-launching/overview`

人类是账户所有人：存钱、发 API key、设 allowlist。日常发币/买卖走自然语言、CLI、`POST /agent/prompt`、或 Wallet API。

### 3.2 账户 / 身份

- 每用户一套托管钱包：至少 EVM + Solana 地址。[一手] `GET /wallet/me`
- 登录：邮箱 / X / Farcaster / Telegram + Privy JWT。
- API key 分层：读任意 key；写要 `walletApiEnabled`；`readOnly`、`allowedIps`、`allowedRecipients`。[一手] Wallet API Overview
- Agent API 与 Wallet API 分开：前者 NL→LLM→执行（慢），后者直接 sign/submit/swap（快）。[一手]
- 安全模型是 **API key = 钱包**。官方警告：泄漏等于丢该账户全部资产。[一手] Agent API Overview
- 没有链上 session key。限额是服务端策略，不是 7579。

### 3.3 发币接口

三层入口同一套限额：

1. 聊天 / 社交：`launch a token called FROG`；X 上 tag `@bankrbot`。聊天默认 **Robinhood Chain**，要 Base 需说 `on base`。[一手]
2. CLI：`bankr launch`；CLI/网页默认 Base。[一手]
3. REST：`POST /token-launches/deploy`。[一手] skills token-deployment.md

机制（EVM，Doppler / Uniswap v4）：

- 固定 100B，不可增发。默认 85% 进池、15% 创作者归属（30 天 cliff，1 年线性，可 `--no-vesting`）。[一手]
- 立刻可交易，无 bonding curve（EVM）。Solana 走 Raydium LaunchLab 曲线再迁 CPMM。[一手]
- 5 分钟内单钱包持仓硬顶 2%（非 partner）。另有约 10 秒反狙击费衰减。[一手]
- 零售限额：每钱包滚动 24h **3 次计入尝试**、每分钟 1 枚；钱包须满 24h 且链上至少 0.002 native ETH。模拟 20 次/日。失败若未上链可退配额。[一手]
- Base 零售发币可赞助 gas；Robinhood Chain / Arbitrum 零售自付。[一手]
- 费：池 0.7% 的 95% 给创作者（= 成交额 0.665%），加上 hook 的协议费 / BNKR 回购 / LP 复投，新币全包约 1.75%。[一手] 旧币费率冻结在发射时。
- 旧路径曾用 Clanker；现在 fee claim 自动识别 Doppler vs Clanker。[一手]
- 可选：degen 起始市值 $2,500；股票 quote（RH Stock Token / Base B20）；quote-only fees。[一手]

Builder 不能用普通 swap 出货自己抽成的币，必须走 Glidepath。[一手]

### 3.4 交易接口

- NL：swap / limit / stop / DCA / TWAP。
- Wallet API：`/wallet/swap`、`/wallet/transfer`、`/wallet/sign`、`/wallet/submit`。sign 禁止 `eth_signTransaction` / typed data（无法从 calldata 核验收款人）。[一手]
- 多链：Base、Ethereum、Polygon、Solana、Unichain、World、Arbitrum、BNB、Robinhood Chain（产品页声明）。[一手] bankr.bot
- x402：用钱包付 API 费用。[一手]

### 3.5 Gas 与权限

- 发币：Base 赞助（在 3 次限额内）；RH/Arb 自付。
- 交易：钱包里要有 gas 或走其内部代付（细节未在 Wallet API 概览写死，标待决）。
- 权限：服务端 API key 策略，不是链上策略。这是 Bankr 对 agent 最不「第一公民」的地方——agent 的手等于全钱包。

### 3.6 链上 vs 链下

链上：Doppler/v4 池、归属合约、Clanker 遗留 locker。
链下：意图解析、配额、反女巫、赞助、Glidepath、LLM gateway。发币是「NL → 服务端组交易 → 托管钱包签」。

### 3.7 可迁移 / 不可迁移

可迁移：发币=agent 自我供血；NL + CLI + 结构化 API 三入口；配额与「未上链失败退配额」；2% 开盘持仓顶；手续费 quote-only；agent 默认链与 CLI 默认链分开。
不可迁移：托管钱包 + API key = 根权限；强制依赖 Bankr 服务器；股票 quote 与 Tape 抢叙事（本产品明确不是 Tape）；Clanker 手续费一次扫全部 token 的 claim 语义。

Bankr 是五件里 **最接近「agent 发币 + agent 交易」** 的，也是 **账户模型最不该抄** 的。

---

## 4. ElizaOS / ai16z / daos.fun

### 4.1 谁是第一公民

两层不要混：

- **ElizaOS**：TypeScript agent 框架。人格文件 + plugin。运行时是第一公民，链只是 plugin。[一手] docs.elizaos.ai
- **ai16z 代币**：2024-10 在 daos.fun 募约 $75k，叙事是 AI 跑风险投资。2025-01 改名 elizaOS，后迁 ELIZAOS（1:6）。2026-08 创始人宣布代币死、基金会关。[二手] The Block 2025-09-25；CoinDesk 2026-08-05
- **daos.fun**：Solana 上「给 agent/DAO 发币」的曲线发射台。ai16z 从这儿长出来，不是 Eliza 框架的一部分。

本对照取 **ElizaOS 作为 agent 交易运行时**，daos.fun/ai16z 作为「agent 叙事发币」失败样本。

### 4.2 账户 / 身份

- EVM plugin：`EVM_PRIVATE_KEY`；可选 `TEE_MODE` + `WALLET_SECRET_SALT`。[一手] plugin-evm examples
- Solana plugin：`SOLANA_PRIVATE_KEY` 或只读 `SOLANA_PUBLIC_KEY`。[一手] plugin-solana
- 身份 = 环境变量里的密钥。没有协议级 session、没有花费上限，除非自己写 plugin。

### 4.3 发币接口

框架不提供 Launchpad。发币要自己接 Clanker SDK / Bankr CLI / 任意工厂。
daos.fun：填名字、人格、符号，连钱包发射。曲线细节官方页几乎不写，标待决。

### 4.4 交易接口

- EVM：自然语言 → 识别 action → 抽参数 → 执行。Swap 走 LiFi / Bebop；桥接多步。[一手] defi-operations-flow
- Solana：Jupiter swap；Helius / Birdeye 可选。[一手]
- 流程是对话式，不是抢 block 的 API。没有官方 snipe / 2D nonce / revert protection。

### 4.5 Gas 与权限

Agent 钱包自己付 gas。私钥在进程里。TEE 是可选包装，不是默认。
失败交易照常耗 gas（普通 EVM）。

### 4.6 链上 vs 链下

几乎全在链下：LLM、plugin、RPC。链上只看到普通 EOA 转账/swap。

### 4.7 可迁移 / 不可迁移

可迁移：plugin 边界（发币 plugin / 交易 plugin / 策略 plugin）；TEE 作为密钥托管选项；性格与密钥分离。
不可迁移：环境变量私钥；把「AI 基金」当发币理由（诉讼与归零已说明风险）；daos.fun 与框架脱节——**发币协议必须是一等 API，不能指望框架社区自己接**。

---

## 5. 高频交易终端：Trojan / Axiom（Photon 作同类）

仓库已有结论：GMGN / Photon / BullX / Axiom / Trojan **全部不支持 Mantle**；接口层收入与发行协议同量级。[仓库] `0-narrative/02-gap-analysis.md`、`terminal/README.md`

### 5.1 谁是第一公民

人类交易员。Bot/终端是执行加速器。不把 agent 当发币者。放进来是因为 **agent 交易侧的 UX 合同是这批产品写的**，不是 Bankr 的 LLM。

### 5.2 账户 / 身份

- Trojan：自称 self-custodial；TG bot 可导入/导出 Solana 密钥；后来有 Terminal 和 OEX。[一手] trojanonsolana.com；[二手] Solana Compass
- Axiom：非托管，密钥在 Turnkey MPC / air-gapped infra；也可连已有钱包。[一手] axiom.trade 产品页（docs 路径 403，未读到逐步文档）
- Photon：闭源 Solana 终端。架构以仓库既有描述为准，不杜撰 API。

### 5.3 发币接口

基本没有。它们吃 pump.fun / 毕业迁移，不办发射台。

### 5.4 交易接口

Trojan 产品页声明（数字为营销口径，不作生产参数）：[一手] trojanonsolana.com

- 1-click 买卖、auto-sniper、migration sniper、copy trade、bundle（最多 10 钱包）、limit / DCA
- 价格刷新声称 0.04s
- MEV protection、backup bots、多钱包
- 声称 lifetime $25B 量 / 2M 用户 / 140M 笔——**未独立核实**

Axiom：Pulse 发现、Instant Trade / 热键、limit、migration actions、钱包追踪、Tweet monitor、Hyperliquid perps。[一手] 营销站 docs 目录；官方 docs 403

共同形状：

1. 发现流（新池 / 毕业 / KOL 钱包）
2. 预签名或 session 钱包，点击即发
3. 优先费 / tip / MEV 模式可调
4. 失败要尽快知道，好改滑点再发

### 5.5 Gas 与权限

用户钱包里的 SOL 付优先费和 Jito tip。没有「失败不扣费」的协议保证——Solana 失败仍可能丢 tip。权限 = 整把私钥或 Turnkey 策略（Axiom 细节待决）。

### 5.6 链上 vs 链下

链下：发现、路由、打包、私有 RPC、bundle。
链上：普通 swap / pump buy。终端不改发行协议。

### 5.7 可迁移 / 不可迁移

可迁移：1-click；热键；发现流与执行流同一表面；多钱包；session/MPC 代替把主钥交给 TG bot；agent 交易 API 必须是这条延迟合同，不能是「等 LLM 想完再 swap」。
不可迁移：Jito / 400ms slot / pump.fun 迁移狙击；Solana 小费市场；「人类盯着 Pulse 点一下」——本产品默认调用方是脚本。

---

## 6. 横切对照

| 维 | Clanker | Virtuals | Bankr | ElizaOS | Trojan/Axiom |
|---|---|---|---|---|---|
| 第一公民 | 人类创作者 | 被代币化的 agent | Bankr 钱包上的 agent | 运行时进程 | 人类交易员 |
| 发币者是 agent？ | 否（Droid 不动钱） | 否（founder 点发射） | **是**（NL/CLI/API） | 否 | 否 |
| 交易者是 agent？ | 外部 bot | 可选 ACP | **是** | 是（慢、NL） | 半自动 |
| 账户 | EOA / FID | EOA + 后配 agent 钱包 | **托管 + API key** | 环境变量私钥 | TG 钥 / Turnkey |
| 花费上限 | 无 | 无 | 服务端 allowlist | 无 | 终端风控 |
| 发币 API | SDK / Partner REST | 网页模块 | **三入口 + 配额** | 无 | 无 |
| 交易 API | 无（Uni） | 平台 + ACP | Wallet API + NL | plugin NL | 终端热键 |
| Gas | 用户 ETH | 用户 | Base 发币可赞助 | 用户 | 用户 SOL |
| 失败不扣费 | 否 | 否 | 未上链发币退配额 | 否 | 否 |
| 链上结算 | Uni v4 工厂 | 曲线→V2 | Doppler v4 / LaunchLab | 任意 EOA | DEX/pump |
| 供血 | LP 费给创作者/Droid | 1% 税 70% 创作者 | 0.665%+ 创作者费 | 无 | 终端抽佣 |

---

## 7. 对 Mantle 的五条结构结论

1. **发币侧已经有 agent 入口，交易侧的 agent 入口是终端不是 LLM。** Bankr 证明 NL 能发币；Photon/Axiom 证明抢 block 不能经过 LLM。产品要 **结构化发币 API + 结构化交易 API**，NL 只做人类次级入口。
2. **现有 agent 产品几乎都把根权限交给托管或环境变量。** 这和仓库 `5-agent-account-model` 的 session / 限额 / 熔断是反的。Mantle 的差异化应在 **链上受限资金**，不要再做一套 Bankr。
3. **「手续费养推理」是唯一被多处验证的 agent 经济回路。** Clanker Droid 10% LP、Bankr 创作者费、Virtuals 70% 税。发币协议必须能把费路由到 **agent 运行时地址**，且与人类创作者地址分开。
4. **发行机制可以抄 EVM 现货，不必再发明曲线。** Clanker/Bankr 用 Uni v4 单边池；Virtuals 用 42k 曲线。仓库已有泵曲线和 v4 相变材料。新产品的新东西是主体，不是曲线数学。
5. **身份不能绑死在 Farcaster/X。** Clanker/Bankr 靠社交图谱反女巫，也因此把机器第一公民卡住。Mantle 要用 **链上账户年龄、质押、session 声誉**，社交只做可选发现，不当发射前提。

---

## 8. 待决

- Axiom 官方 docs（axiom.trade/docs 403）；Turnkey 策略能否表达「只许买某工厂的币」。
- Bankr 交易路径是否内部赞助 gas。
- Photon 无公开 API；接口形状以仓库既有观察为准。
- Virtuals 曲线精确储备参数未读合约；42,000 VIRTUAL 毕业阈值来自白皮书，不引用第三方反推。
- daos.fun 曲线公式官方未公开。
- Clanker Partner 部署是否赞助 gas：文档未写。
