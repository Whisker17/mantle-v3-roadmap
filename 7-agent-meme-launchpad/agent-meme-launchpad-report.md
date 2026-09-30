# 在 Mantle 上做 Agent 第一公民的 Meme Launchpad

> 轨道 7 ｜ 2026-09-17 ｜ 可独立阅读。细节与出处见 `notes/`、`sources.md`。
> 交叉引用 `2-meme-launchpad/`、`5-agent-account-model/`、`0-narrative/09`，不改写它们。
> 09 仍是「路线比较设计提案、尚未选型」。本报告不把未定延迟、TPS、证明费用写成已定。

## 0. 先讲结论

**推荐维持 L2。** 先在现有 Mantle L2 上做：Uni v4 工厂 + 链上 session 账户 + 结构化 agent API。不要为这个产品先开 App-specific L3。

理由很窄：市面上已经跑通「agent 发币 / agent 交易」的产品（Bankr、Clanker 工厂、Axiom/Trojan 终端）都在共享 L2 或 Solana L1 上，缺的不是独立状态机，缺的是 **身份与权限**——agent 能发币、能买卖，但不能拿根钥。这件事 4337/7702 session 和 Paymaster 就能起步。L3 能买到的失败不入块、热点隔离、自定义开盘排序，09 自己写过：Launchpad 要不要独立链，取决于已经出现的负载与排序需求，不是取决于叙事里有 agent。

至少三条可观察条件会翻转这条推荐，见第 7 节。

---

## 1. 这是什么产品（以及明确不是什么）

第一公民是 **agent**：它创建代币，它用受限资金买卖和做市。人类是 owner：存款、签发 session、撤销、停机。

最小闭环：

1. 人类把 quote（默认 USDC）放进 AgentAccount。
2. 人类签一条 session：只能打工厂和 router，有预算、有过期时间。
3. Agent 调 `Deploy`，一笔交易得到代币 + 池 + 手续费路由。费的一腿进该 agent 的 runtime vault，用来付推理和后续 gas。
4. 其他 agent 调 `Buy`/`Sell`。模拟失败不上链。开盘窗口有持仓顶和衰减税，不靠谁的 RPC 更近。
5. 人类不必每笔点确认，但随时 `revoke`。

**反目标**

- 不是 Tape。Tape 的 D3（股票 quote 情绪一级市场）不动。本轨道不寄生 xStocks，不用 Spend-Gate 当产品门。
- 不是「人连钱包，再挂 bot」。那是 `terminal/` 的事。
- 不是 Virtuals 式「agent 本身是标的」。有没有人格文件不能当发射条件。
- 不是再做一套 Bankr：API key 等于钱包根权限。

---

## 2. 市面 Agentic 产品怎么建

五件对照物不是一类东西。完整对照见 `notes/01-agentic-landscape.md`。这里只留对设计有约束的部分。

### 2.1 Clanker：发币工厂，agent 是吉祥物

人类是第一公民。`@clanker`、网站、`clanker-sdk` 部署固定供应 ERC-20，剩余供应单边进 Uniswap v4，LP 永久锁，手续费给创作者。[一手] SDK / DOCS；机制精度见仓库 `2-meme-launchpad/benchmarks/base-ecosystem.md`。

Droid 是附赠运营 agent：独立 Farcaster 账号，默认 10% LP 奖励（100–5000 bps）进 Base 上的 runtime 钱包付 USDC 推理。文档写明 **not autonomous with your money**——cast 要人确认，不能动主仓。[一手] `droids/README.md`、`droids/funding.md`

对 Mantle：**可抄工厂和「费养推理」；不可抄「agent 不许碰钱」。** Farcaster 门会把机器发币者卡在社交图谱外。

### 2.2 Virtuals：把 agent 当可融资标的

曲线配对 `$VIRTUAL`，42,000 VIRTUAL 毕业到 Uniswap V2，LP 锁 10 年。1% 税，70% 创作者 / 30% 金库。可选 99%→1% 狙击税。[一手] Capital Formation Layer、Launch Mechanics。

发币者仍是人类 founder。ACP 才是 agent 之间做生意（托管 + Evaluator），不是 Launchpad 热路径。[一手] Commerce Layer

对 Mantle：**可抄费分润、可选狙击税、Fee Delegation（代发、费留给 builder）。不可抄「先有人格再有币」。** ai16z 那条「AI 基金」叙事已经以诉讼和归零结束，见第 2.4 节。

### 2.3 Bankr：最像本产品，账户模型最不能抄

文档原句：发币是 *how agents fund themselves*。[一手] Token Launching Overview

入口有三：自然语言（聊天 / `@bankrbot`）、`bankr launch` CLI、REST。聊天默认 Robinhood Chain，CLI/网页默认 Base——同一产品两种默认链，说明 NL 不能当唯一发射接口。

EVM 现货走 Doppler / Uni v4：100B 固定供应，默认 85% 进池、15% 一年归属（可关），立刻可交易。开盘 5 分钟单钱包持仓顶 2%，另有约 10 秒反狙击费。零售滚动 24h 3 次发射、每分钟 1 枚；钱包满 24h 且有 0.002 native ETH。模拟不占配额；未上链失败退配额。Base 零售发币可赞助 gas，Robinhood Chain / Arbitrum 自付。[一手] 同上 + Bankr skills `token-deployment.md`

交易分两条：Agent API（`POST /agent/prompt`，LLM，慢）和 Wallet API（swap/sign/submit，快）。API key 分层：只读、IP 白名单、收款人白名单。官方警告：泄漏带 agent 权限的 key 等于丢全部资产。[一手] Agent API / Wallet API Overview

对 Mantle：**可抄三入口、配额与未上链退回、持仓顶、quote-only 费、发币赞助。不可抄托管根钥。** 链上 session 是本产品相对 Bankr 的全部差异化。

### 2.4 ElizaOS / ai16z：运行时有了，发射台没有

ElizaOS 是 TypeScript 框架。EVM/Solana plugin 把 `EVM_PRIVATE_KEY` / `SOLANA_PRIVATE_KEY` 放进环境变量，可选 TEE。交易是自然语言 → Jupiter / LiFi，不是抢 block 的 API。[一手] docs.elizaos.ai plugin 文档

ai16z 2024-10 在 daos.fun 募约 $75k，2025 改名并迁代币，2026-08 创始人宣布代币死亡。[二手] The Block、CoinDesk。框架还在，代币实验结束。

对 Mantle：**可抄 plugin 边界和可选 TEE。不可抄环境变量根钥，也不可把 Launchpad 寄托在框架生态自己接。** 发币必须是一等协议 API。

### 2.5 Trojan / Axiom（Photon 同类）：交易侧的 UX 合同

仓库已记录：GMGN / Photon / BullX / Axiom / Trojan 全部不支持 Mantle，接口层价值捕获高于协议层。[仓库] `0-narrative/02`、`terminal/README.md`

Trojan：self-custodial TG bot，1-click、sniper、migration sniper、copy trade、bundle、MEV 保护。[一手] trojanonsolana.com（其 $25B / 2M 用户为营销口径，未独立核实）

Axiom：非托管，Turnkey MPC；Pulse 发现 + Instant Trade。[一手] axiom.trade 产品页。官方 `/docs` 本次读取 403，策略细节待决。

它们几乎不发币。放进对照是因为 **agent 交易路径必须是这条延迟合同**：发现流订阅事件，执行路径无 LLM。Bankr 的 prompt 路径不能当热路径。

Photon 无公开一手 API，不杜撰。

### 2.6 横切：谁已经是发币者 / 交易者

| | 发币者是 agent？ | 交易者是 agent？ | 钱的权限 |
|---|---|---|---|
| Clanker | 否 | 外部 bot | Droid 沙箱，不动主仓 |
| Virtuals | 否 | 可选 ACP | founder EOA |
| Bankr | **是** | **是（慢 NL + 快 Wallet API）** | API key = 根 |
| ElizaOS | 否 | 是，但 NL | 环境变量根钥 |
| Trojan/Axiom | 否 | 半自动 | 整钥或 Turnkey |

结构结论只有五条：

1. 发币侧 agent 入口已存在（Bankr）；交易侧 agent 入口是终端 API，不是聊天。
2. 根权限几乎都在托管或环境变量。Mantle 不该再做一遍。
3. 「手续费养推理」被 Clanker / Bankr / Virtuals 同时验证。
4. 发行机制不必新发明：v4 单边池或 hook 相变，仓库已有。
5. 身份不能绑死 Farcaster/X。

---

## 3. Agent 第一公民的硬需求

展开见 `notes/02-agent-requirements.md`。

| ID | 需求 | 直接来源 |
|---|---|---|
| R1 | 发币是结构化 API（幂等、模拟不占配额、失败码），NL 只做人类入口 | Bankr 双默认链；Eliza 无发射台 |
| R2 | agent 有链上身份；session ≠ owner 根钥；fee 接收地址 ≠ 热钥 | Clanker 吉祥物 vs Bankr 根钥 |
| R3 | 花费白名单 / 单块与累计预算 / 过期 / 可撤销 | `5-agent-account-model` |
| R4 | 同一 session 能 deploy、buy、sell、claimFee，不能任意 transfer | 避免 Virtuals 式主体分裂 |
| R5 | 多标的并发，互不阻塞；交易路径无 LLM | 仓库 03 的 2D nonce 诉求；Axiom/Trojan |
| R6 | 模拟失败不上链不扣费；未广播发射退配额。执行失败在 L2 仍可能扣 gas，必须写进 UX | Bankr 配额；launchpad-chain-spec 高 revert 画像 |
| R7 | 发射时写死 feeRecipients，含 runtime 腿，只收 quote | Droid 10%；Bankr 0.665%；Virtuals 70% |
| R8 | 开盘持仓顶 + 短窗口衰减税或等价拍卖，写在 hook 里 | Clanker MEV 模块；Bankr 2% 顶；Virtuals 狙击税 |
| R9 | 人类能停机，不必每笔确认，不必为发币付 gas | 产品定义 |
| R10 | 发现与执行分离：事件订阅 + 无 LLM 下单 | 终端对照 |

---

## 4. L2 方案

基线 = 09 路线一：不改 Mantle 执行层、排序、证明。应用合约 + 链下 API + 既有 AA。细节 `notes/03-l2-design.md`。

### 4.1 发币

默认 **无 bonding curve**：固定供应、单边流动性进 Uni v4、LP 无 withdraw、立刻可交易。与 Clanker/Bankr 相同家族。Agent 需要一笔原子 `Deploy`，不需要「毕业直播」。

若以后要曲线注意力，用仓库已有的 **同一 v4 池 hook 相变**，不要四.meme 式迁池。[仓库] `2-meme-launchpad/chain-infra/launchpad-chain-spec.md` §4.2

Quote 默认 USDC（协议登记稳定币）。不默认 MNT，不默认 xStocks。

### 4.2 交易

Router 原子 approve+swap。Hook 执行开盘持仓顶和衰减税。不做链上订单簿。做市就是对同一池连续 swap。

Indexer 推 `TokenLaunched` / `Swap`。Agent 自己决策。协议不提供「跟单 LLM」。

### 4.3 账户、Gas、失败

MVP：ERC-7579 session 或 EIP-7702 委托 + 4337 Paymaster。发射赞助；交易可用 USDC 扣 gas。这是 `5-agent-account-model` Phase 1，不要求 RIP-7560。

Agent API 发送前模拟。模拟失败不提交。提交后超时标 `unknown`。

**L2 做不到（不是 TODO，是边界）：**

1. 协议级失败不入块、执行失败不扣费
2. 热点隔离：单币开盘与全链抢同一 blockspace
3. FBA / 按合约分桶 / cancel 优先（那会改 sequencer，不再是路线一）
4. 二维 nonce
5. 独立费市场
6. 把预当结算

Robinhood Chain 曾出现过热点推高 base fee 拖垮其他活动的画像。[仓库] launchpad-chain-spec。那是对照，不是 Mantle 上已测到的数。没测到之前，不拿它当必须上 L3 的证明。

---

## 5. L3 方案

基准：09。预存资金、异步跨层、独立 Sequencer / DA / 证明。远程授信、Solver 垫资、完整 JSON-RPC **不是**基础闭环。细节 `notes/04-l3-design.md`。

### 5.1 状态机

L3 内同一执行域持有：Owner、AgentSession（含独立 nonceKey）、Balances、Token、Pool 或曲线、FeeVault、开盘窗口。`Deploy` / `Swap` 是原生交易类型。毕业若发生，是 L3 内原子相变，资产不先退出到 L2。[09 §3.1]

Quote 只接受已从 L2 预存的 USDC。配额是可证明状态，不是 Bankr 那种链下计数器。

开盘 FBA、失败丢弃、二维 nonce 可以进状态机——这是相对 L2 路线一的真实增量。排序仍不能单凭私有 mempool 证明 Sequencer 未隐瞒订单。[09 §3.3]

### 5.2 充提

**充值。** 人类或 L2 AA 把 USDC 锁进 **本 L3 Escrow**，Inbox 记消息 ID。L3 消费一次后 agent 才有额度。不能在 L2 无锁定时在 L3 发币。[09 §4.2]

确认策略未测前走保守：不把 unsafe 父链输入当成可发币余额。暂定余额必须能回滚。

**L3 内。** deploy/swap/claim/revoke 本地完成。

**提现。** L3 先扣账，批次 Outbox，L2 Verify 后待领，Claim 一次性支付。[09 §4.4] **只允许提 USDC。** L3 meme 不桥回 L2，避免没有池的「L3 ERC-20」冒充可组合资产。若将来要毕业到 L2 Uni，那是异步消息 + L2 建池，非原子，有窗口——这是为热点隔离付的可组合性税。

### 5.3 确认层级

API 必须带 `confirmationLevel`：

| 层级 | 含义 | 不是 |
|---|---|---|
| Sequencer 回执 | 暂定执行 | 不是 L2 已验证 |
| L3 执行确认 | 进入 L3 历史 | 不排除父链重组 |
| L2 接受证明 | L2 记下该状态根 | 不等于 L2 块最终 |
| 父层最终性 | 所选最终性条件 | 不替代 DA 与升级检查 |

不写秒数。09 禁止用其他系统的证明等待期冒充本方案。[09 §4.1、§4.4]

签名、预确认、L3 执行、L2 验证是四件事。

### 5.4 故障退出

按 09 §6 改写 Launchpad（无杠杆，但仍不是纯转账链）：

- 父链重组撤销充值：回退；未稳定充值不得用于 deploy
- Sequencer 停机/审查：L2 强制请求，超时受控退出；兑付 USDC 权益，不兑付「按旧价估的 meme」
- 证明积压：限制未证明敞口；暂定成交不可提
- DA 失败：不正式结算；公开数据能重建 USDC 与 token 余额
- L2 停机：停跨层；L3 不是独立结算层
- 失败发行：先退款或保留可领取承诺，不能直接删余额 [09 §3.3]

不把 Solver 快速充值写进闭环。

---

## 6. 对比

数字栏只写结构，不写未测 TPS。

| 维 | L2 | L3 |
|---|---|---|
| 延迟与并发 | 共享出块；一维 nonce；4337 有 Bundler 额外跳 | 独立排序；原生 session nonce；回执仍不是结算 |
| 热点隔离 | 无。单币可挤占全链 | 有。外溢停在这条 L3 |
| 同链可组合性 | 与 L2 Uni、借贷、任意合约同一原子域 | 预存后闭环；回 L2 异步。meme 默认不桥回 |
| 账户模型 | 7579/7702 session，应用层 | 同一策略可做成原生交易类型 |
| 运维成本 | 工厂 + Paymaster + API + indexer | 再加 Sequencer、Observer、DA、Prover、退出 | 09 工作量按独立链计，本报告不报价 |
| 流动性割裂 | 无新割裂 | 热路径在 L3；L2 只见 USDC 托管 |
| agent 签名 | session 签 UserOp 或 7702 tx | session 签 L3 交易类型 |
| session | 有，EVM 成本较高 | 可更便宜，参数未测 |
| 赞助 gas | Paymaster；要原生或 USDC 通道 | L3 内用户免 gas，用税或运营预算，必须配额防滥用 |
| 失败不扣费 | 仅模拟阶段可靠；执行失败仍可能扣 | Sequencer 可丢弃必然失败交易；仍须防零成本 DoS |

---

## 7. 推荐与翻转条件

**推荐：L2 起步。** 产品缺口是 R1–R4、R7、R9（API、session、费路由、人类停机），不是独立状态机。Bankr/Clanker 已在共享 EVM L2 上证明发币工厂能撑起 agent 流量；它们的事故在权限和社交门，不在「没有 L3」。

L3 的好处（失败丢弃、热点笼子、FBA）是真的，但 09 把 Launchpad 写成「看负载再决定」，并且热路径一旦离开 L2，graduation 和任意 DeFi 组合立刻变成跨层问题。在没有 Mantle 上的热点测量之前，为 agent meme 先建一条链，是把未观测到的执行问题，用确定性的流动性割裂去换。

### 会翻转推荐的可观察条件（至少三条）

**F1 热点外溢。** 上线 L2 工厂后，单次发射窗口内无关交易失败率或 base fee 相对该窗口前 24h 中位数出现持续抬升，且能归因到该工厂地址的 gas 占比。对照是仓库对 Robinhood Chain 的描述，**必须用 Mantle 自己的 RPC 再测**，不能把 82 倍当本链参数。一旦测到外溢，优先评估 sequencer 分桶（那已超出路线一）；若治理不允许改共享 L2 排序，再开 L3。

**F2 失败扣费成为做市退出原因。** 统计 agent session 的 revert 率与浪费 gas 占其成交毛利的比例。若专业做市 agent（非 NL 路径）因执行失败扣费或一维 nonce 阻塞而停止挂单，且 4337 验证阶段丢弃 + 应用层子账户仍不够，则 L3 的「失败不入块 + 二维 nonce」变成产品条件，不是优化。

**F3 开盘排序被物理延迟垄断。** hook 衰减税和持仓顶上线后，若开盘短窗口的成交仍集中在可观测的共址 bot（同一 builder/RPC 集群），且创作者费捕获低于这些 bot 的短窗利润（用池成交与 fee locker 对账），则需要 FBA 或分桶。这两者在路线一里没有。若产品决定「开盘必须公平」而不是「税把狙击变贵」，推荐翻到 L3。

补充一条不会单独翻转、但会加强 F1–F3 的观察：**agent 从不把币毕业到 L2 Uni，预存 USDC 已被接受为默认入场。** 这时 L3 的可组合性代价变小。它仍不能单独成立，因为没有热点或失败问题就不必付独立运维。

### 不会翻转的东西

- 「叙事上是 agent」本身
- 尚未测量的 TPS 目标
- 想要零 MEV
- 想让签名等于结算
- 想让 L3 覆盖 Mantle L2 全局状态根（09 明确不能）

---

## 8. 假设与待决

- Quote 用 USDC：设计选择，不是链上已有登记。
- 发射配额、持仓顶百分比、衰减窗口：对照 Bankr/Clanker/Virtuals，**本产品数值待决**。
- Bankr 交易路径是否内部赞助 gas：Wallet API 概览未写死。
- Axiom 官方 docs 403；Turnkey 策略粒度待决。
- Photon 内部接口未知。
- Virtuals 曲线精确储备未读合约；只用白皮书 42,000 VIRTUAL 阈值。
- daos.fun 曲线公式官方未公开。
- 09 的证明间隔、DA 费用、提款等待：保持未定。
- L2 方案假设 Mantle 已有或可部署 Uniswap v4 与 4337 EntryPoint。若 v4 不可用，退回 v3 locker，损失 hook 相变，不因此改推荐。

未实现合约，未部署，未改 Tape，未改 09。
