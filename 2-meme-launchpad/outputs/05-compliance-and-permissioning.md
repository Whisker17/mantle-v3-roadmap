# 3.7 补遗：合规是第七个结构性缺陷（D7）

> **归属**：第三章缺陷图谱的补章。前六条（迁移、老鼠仓、碎片化、注意力、拥堵、休市耦合）都是产品机制问题。股票 Quote 一旦进来，**合规不再是法务附录，而是发行架构的约束条件**。  
> **文档位置**：`2-meme-launchpad/outputs/05-compliance-and-permissioning.md`

---

> ⚡ **【本节核心 TL;DR】**  
> 1. **D7：无许可发币 × 受监管 Quote。** pump.fun 可以假装合规不存在；Pons/LONG/PAIR 用美股代币当底池之后，OFAC、ATS、发行人抗议、一级 KYB 都会打到同一条发射管路上。  
> 2. **链层（ArbOS Elara / `ArbFilteredTransactionsManager`）**：按 **txHash** 在状态机里作废交易，连 L1 force-inclusion 也挡。这是「合约仍可乱部署，执行可被定点删除」。适合制裁地址，**不适合**做 Launchpad 的日常 KYC——粒度太粗、政治成本太高。  
> 3. **池层（Uniswap v4 Permissioned Pools）**：`beforeSwap` / `beforeAddLiquidity` 查 allowlist。因为 v4 资金在 `PoolManager` 内部记账，**ERC-20 自己的转账黑名单看不见池内换手**，所以必须用 Permissions Adapter + Hook，不能只改代币合约。  
> 4. **Mantle 不该二选一抄满。** 正确切法：链层只做制裁级过滤（可公开规则）；股票 Quote 的准入放在 **资产/池 Hook**；meme 本身保持无许可。Robinhood 把过滤做进 STF、池子却完全放任——这正是 AMC 事件和周末脱锚同时发生的结构原因。

---

## 1. 为什么前六条不够：合规是发行层的新约束

> ⚡ **TL;DR**：ETH/SOL Quote 的 Launchpad 活在灰色区。股票 Quote 把「谁能碰这笔流动性」变成协议必须回答的问题。前端 KYC 挡不住合约调用；代币黑名单挡不住 v4 内盘；Sequencer 拒收挡不住 L1 强插——三层各管各的，漏一层就等于没做。

普通 meme 的合规模型是：

- 代币是空气 ERC-20，没有证券属性；
- 用户自己连钱包，前端爱拦不拦；
- 出了事，平台可以说「我们只是合约工厂」。

股票代币作 Quote 之后，这条退路没了：

| 压力来源 | 具体形态（已发生或公开悬空） | 打在 Launchpad 哪一刀 |
|---|---|---|
| 制裁名单 | OFAC 地址买卖 `Meme/NVDAx` | 交易必须能被挡，且挡完索引器不能分叉 |
| 证券交易场所 | AMM 池交易 linked securities 是否构成 **Reg ATS / security-based swap** | 全行业零书面表态；RH 靠 Reg S + 泽西岛发行人隔离 |
| 发行人政治 | AMC CEO 2026-09-04 要求 cease and desist | 应用层放任，资产层（停预言机、停 AP 增发）才有手 |
| 一级 KYB vs 二级开放 | RH 股票代币：**mint/redeem 仅 Bitstamp，转账是裸 ERC-20** | 发射台用它当 Quote = 把合规资产送进无许可盘 |

三层拦截对不上，就会出现 RH 现在的结构：

```
链层：Sequencer / ArbOS 可以杀特定交易     ← 制裁级，已产品化
代币层：股票 ERC-20 无 transfer hook        ← 二级完全开放
池层：Pons/PAIR 的 v4 池无 KYC hook         ← meme 无许可，股票当燃料
```

合规缺陷（D7）就是：**这三层没有一张统一的权限图，Launchpad 却把受监管腿和空气腿焊在同一个池里。**

---

## 2. 链层：ArbOS Elara 与 `ArbFilteredTransactionsManager`

> ⚡ **TL;DR**：Elara（2026-08）把合规过滤做成预编译 `0x…0074`。授权 filterer 登记 txHash，**状态转换直接判失败**，包括 Delayed Inbox 强插进来的交易。对 Launchpad 的含义是：可以外科手术切掉一笔毕业/扫盘，但用它做日常 KYC 等于把整条链标成可审查。OP Stack 默认做不到同等的 force-inclusion 拦截。

### 2.1 机制：不是用户 tx 自己去读预编译，是 STF 先查表再决定跑不跑 EVM

> ⚡ **TL;DR**：用户签完 tx，哈希就定了。合规引擎在链下看到这笔 tx 之后，**由授权 filterer 另发一笔 L2 交易**，把该哈希写入 `0x…0074` 的存储。之后任意节点执行到「哈希 = 这笔」的交易时，**在进 EVM 之前就判失败**。所以：查表发生在 STF；**写入发生在更早的另一笔 filterer 交易里**。Sequencer 日常甚至连「打包再失败」都懒得做，直接不收录。

把它想成两段，不要合成一段。

```
【日常路径：根本不进区块】
用户广播 tx
  → Sequencer / 合规引擎链下扫描（from、to、calldata、模拟结果）
  → 命中制裁名单：直接不打包
  → 链上像没发生过（Robinhood 文档原意）

【硬路径：必须进规范历史时才用预编译】
尤其是 L1 Delayed Inbox 强插（force-inclusion）
  1. 用户已签名 → txHash = keccak(signed_tx) 在广播瞬间就确定
  2. filterer（链下机器人，ArbOwner 授过权）看见这笔哈希
  3. filterer 自己发一笔 L2 tx：ArbFilteredTransactionsManager.register(txHash)
     → 这笔必须先被 STF 执行，哈希才进预编译存储
  4. 稍后轮到被过滤的那笔（含 L1 强插进来的消息）
  5. STF：if (filtered[txHash]) fail;  不调用用户的 to 合约
```

**和「用户交易 revert」不是一回事。**  
Solidity `revert` 是 EVM 跑到一半失败，仍有收据、仍烧 gas。这里是 **ArbOS 在交易类型/校验阶段拒绝**：状态不改，索引器按「未处理」对齐。L2Beat 的表述是 STF *forcibly reject and fail*，包括 force-included 消息，且无额外延迟窗口。

**哈希何时进预编译：**

| 步骤 | 谁 | 何时 | 链上有没有 |
|---|---|---|---|
| 算出 txHash | 任何人（签名一完成就能算） | 用户签名的那一刻 | 无 |
| 写入 `filtered[hash]=true` | 授权 filterer 的 **另一笔 L2 交易** 调 `0x…0074` | 必须 **早于** 被过滤那笔被 STF 执行 | 有，可审计 |
| 查表并失败 | 每个验证者的 STF | 执行到被过滤那笔时 | 查的是上一步写下的存储 |

Sequencer 独占排序。**没有「用户那笔自己跑到预编译里登记自己」这回事。**

#### 2.1.1 「RH 不是 FCFS 吗？后到的 register 怎么可能排到前面？」

问得对。若对**所有进入区块的交易**都严格按到达顺序，filterer 必须先看到用户 tx，其 `register` 一定后到，FCFS 下必输。

解开这一句的关键：**广告上的 FCFS 是用户打用户（禁止加小费插队），不是用户打 Sequencer。** 过滤也不靠在同一条 FCFS 队列里抢跑。

```
FCFS 约束的对象：已经决定「要进这个区块」的普通用户 tx 之间的相对顺序
FCFS 不约束：
  ① Sequencer 可以根本不把某笔用户 tx 放进区块
  ② Sequencer / filterer 可以插自己的系统交易
  ③ L1 Delayed Inbox 到期后的消息，按 inbox 顺序消费，不跟 L2 mempool 抢到达时间
```

| 路径 | 来得及吗 | 为什么不跟 FCFS 打架 |
|---|---|---|
| **日常：入口丢掉** | 来得及 | 用户 tx **从未进入** FCFS 队列。看到 → 不打包。没有第二笔 tx 要抢位。Robinhood 文档「像没发生过」主要是这条 |
| **L1 强插** | 来得及 | Delayed Inbox 有等待期（小时级）。用户把签名消息打上 L1 之后，**在它有资格被 L2 消费之前**，filterer 有整段窗口在 L2 上 `register(hash)`。到期 STF 再消费这条消息时，表里已经有 hash。这是预编译真正存在的理由 |
| **同一 L2 块里先 register 再 fail** | 不靠 FCFS 赢 | 若非要把坏 tx 放进规范历史再失败，**出块的人就是 Robinhood**。它可以把 filterer 交易当块头系统交易插在最前。这已经不是用户间 FCFS，是运营者特权。[INFERENCE：Elara 源码是否把 register 做成强制块头交易，公开材料未写死] |

所以：**不是「检查完再发一笔普通用户 tx 去插队」。**  
日常路径甚至不发第二笔；硬路径要么用 L1 延迟换时间，要么用出块权把 register 放在前面。  
若 Sequencer 真把自己也锁进纯 FCFS、且必须收录每一笔到达的 tx，用户的质疑成立——那套来不及。RH 没有这样锁自己。

Robinhood 公开形态仍是：

> 合约部署无许可；交易执行可被许可过滤。

### 2.2 对 Meme Launchpad 的真实影响

**能做的：**

- 制裁地址的 buy/sell/graduate 交易，登记哈希后从历史上抹掉；
- 不需要暂停整个 TokenManager，也不需要给每个 meme 加黑名单；
- 和「无公开 mempool」叠在一起，链下看起来更干净，机构才敢把股票代币放上这条 L2。

**不能当日常产品开关的：**

- 过滤的是 **txHash，不是「这个人以后所有交易」**。地址级拦截要另做名单，或每笔新交易再登记——运营跟不上发射台的 TPS；
- 一旦用于「不喜欢的毕业 / 不喜欢的 meme」，就是任意审查。Degen 会用脚投票（L2Beat Stage 0 + force-inclusion 可杀，已经是现成叙事弹药）；
- **挡不住合约部署。** 工厂仍是 permissionless。脏发射照样上链，只是某些交互被掐；
- 与 meme 的 Bot 经济冲突：GMGN 类终端假设「发出去的 tx 要么上链要么在 mempool」。STF 级失败且「像没发生过」，索引器和机器人会短暂不同步。

### 2.3 和 OP Stack / Mantle 的差别

标准 OP Stack：`OptimismPortal.depositTransaction` 派生的 L2 交易，Fault Proof 要求 **无条件执行**。Sequencer 可以在 RPC 丢掉 OFAC 交易，但用户走 L1 强插，链必须认。

因此：

- Mantle 今天若只在 Sequencer 入口筛，**合规声明弱于 RH**；
- 若在 op-geth STF 里抄一套哈希黑名单，等于自己分叉出 Elara 同类能力，要公开规则 + 保留 L1 路径的例外说明；
- 更干净的切法是：**链层只承诺制裁哈希/地址，Launchpad 合规放在池/资产层**（下一节）。

---

## 3. 池层：Uniswap v4 Permissioned Pools

> ⚡ **TL;DR**：v4 的钱在 `PoolManager` 里内部记账，**代币合约的 `transfer` 黑名单看不到 swap**。官方 Permissioned Pools 用 **Permissions Adapter（真代币不出发行方白名单合约）+ Hook（`beforeSwap` / `beforeAddLiquidity` 查 allowlist）**。Launchpad 可以只对「股票那条腿」做许可，meme 腿保持开放——这是 ArbOS 做不到的细粒度。代价是聚合器、GMGN、共享水库全部要认这套 adapter，流动性会裂。

### 3.1 为什么「给 ERC-20 加黑名单」在 v4 上失效

v2/v3：swap 真的 `transfer` 代币，带 hook/fee-on-transfer/黑名单的币至少还能拦一刀（也因此 Recursion/重入事故一堆）。

v4：

- 所有池的资产进 `PoolManager`；
- 用户与池之间是 **delta / flash accounting**，不是每跳一次 ERC-20 `transfer`；
- 发行方在代币上写的 `onlyWhitelisted(to)`，**池内换手根本不经过 `to = 用户`**。

所以股票若是「一级 KYB、二级自由转账」的裸 ERC-20（RH / xStocks 现状），**毕业进 v4 之后，合规在池里等于零。** 这是 PAIR/Pons 今天的真实状态。

### 3.2 官方 Permissioned Pool 怎么接

Uniswap 文档（*Permissioned Pools on Uniswap v4*）三件套：

```
真证券代币 ──► Permissions Adapter（发行方 allowlist 过的托管合约）
                    │
                    ▼  进出池才 wrap/unwrap
              虚拟/内部记账资产  ← PoolManager 只碰这个
                    │
                    ▼
         Permissioned Hook
           beforeSwap        → 查 swapper 是否在 KYC/allowlist，否则 revert
           beforeAddLiquidity→ 查 LP 是否允许铸仓
           beforeInitialize  → 谁能建这种池
```

要点：

- 检查发生在 **Hook，不是前端**；
- Adapter 保证「真代币」只在发行方允许的地址间动；池子玩的是影子余额；
- allowlist 可以接链下 KYC 证明（attestation）或链上登记表。

### 3.3 对 Launchpad 的设计空间（不只是「整个池 KYC」）

股票 meme 不需要「用户买 meme 也要护照」。有用的切法是 **非对称许可**：

| 模式 | Hook 做什么 | 适合 | 代价 |
|---|---|---|---|
| A. 全池许可 | swapper + LP 都要 KYC | 代币化基金、合规 RWA 池 | 没有 degen，Launchpad 死 |
| B. **只许可 Quote 腿** | 入池/出池 NVDAx 必须经 Adapter；meme 自由 | 股票 Quote 发射台 | 实现复杂；聚合器要改 |
| C. 毕业后收紧 | 曲线期无许可，v4 相变后打开 allowlist | 想「先狂欢再合规」 | 相变瞬间流动性断裂，Bot 会提前退场 |
| D. 只拦 LP / 只拦大额 | `beforeAddLiquidity` 或金额阈值 | 防股票浮筹被单池吸干（HIMS 53%） | 挡不住散户扫盘 |
| E. 熔断式许可 | 休市或偏离过大时 `beforeSwap` revert 大额 | D6 休市脱锚 | 这是风控不是 KYC，但同一套 hook 位 |

Pons 的 singleton `MemeHook` 已经占了 `afterSwap` 收费位。v4 一个池一个 hook 地址，**收费 Hook 和合规 Hook 必须合成同一个合约**，不能并排插两个。这是工程约束，不是产品愿望。

### 3.4 Permissioned 池对「共享水库 / GMGN / 1inch」的伤害

一旦 Quote 走 Adapter：

- 共享流动性水库不能再假设「任意地址 `transferFrom` NVDAx」；
- 1inch / UniX 默认路由会 **跳过** 认不出 Adapter 的池；
- GMGN 一类终端要接 allowlist RPC，否则用户点了就 revert；
- 表面上修了 D7，实际上把 D3（碎片化）再撕一道。

所以：**合规 Hook 是可选模块，不能当默认毕业形态。** 默认池仍应无许可；只有「发行方要求股票不得进无许可 AMM」的标的，才走 Permissioned 毕业路径。

---

## 4. 三层对照：该放哪一层

> ⚡ **TL;DR**：制裁 → 链；证券身份 → 资产/Adapter；交易规则（KYC、熔断、限仓）→ v4 Hook。用 ArbOS 做 KYC、用代币黑名单做 v4 池，都是层放错了。

```
┌─────────────────────────────────────────────────────────────────┐
│ 层            │ 工具                         │ 粒度           │ 政治成本 │
├─────────────────────────────────────────────────────────────────┤
│ 链 / STF      │ ArbFilteredTransactionsManager、Sequencer 筛    │ txHash / 地址 │ 极高    │
│ 资产          │ Transfer hook、TIP-403、ERC-3643、Adapter       │ 某个 ERC-20   │ 中      │
│ 池 / Hook     │ v4 permissioned + 熔断 + 限仓                   │ 某个 PoolId   │ 低到中  │
│ 前端          │ Wallet KYC、Bybit 登录                         │ 会话           │ 低，可绕过 │
└─────────────────────────────────────────────────────────────────┘
```

Robinhood 的实际选择：**链层很重（Elara + 筛 Sequencer），池层很轻（无许可 v4），资产层二级完全开放。**  
结果：机构愿意上链（有 STF 杀手锏），但 Launchpad 把股票当空气烧，周末脱锚和发行人开战都发生在池层——正是他们没设防的那一层。

---

## 5. 给 Mantle Tape 的合规切法（不要抄 RH 的层放错）

> ⚡ **TL;DR**：公开写清两句话：「使用无许可，制裁级验证有许可」。股票 Quote 用 Adapter/Hook，不要用全局 tx 过滤器当产品功能。meme 默认无许可毕业。OP Stack 先做 Sequencer 筛 + 池 Hook；STF 级哈希过滤只有在法务认定「没有它就上不了 xStocks」时再上。

**默认路径（无许可发射）：**

- 曲线 + v4 相变，不接 Permissioned Pool；
- Sequencer 入口丢掉明确制裁名单（公开列表、可复现）；
- 不在 STF 里杀 force-inclusion，避免和 OP 证明系统打架，也避免「可任意审查链」标签。

**股票 Quote 路径（D7 真正要做的）：**

- xStocks 若发行方要求：进出共享水库 / v4 池走 **Permissions Adapter**；
- 同一个 Launchpad Hook 里叠：allowlist（可选）、休市熔断、单池持仓上限（防 53% 吸干）；
- 前端 Bybit/Passport KYC 只作为 UX，**不当事故后的抗辩**。

**明确不要：**

- 用 Elara 式过滤器按「这个 meme 不好看」删交易；
- 给 meme 本身加 ERC-3643（用户钱包会炸，DEX 会当邪教币）；
- 毕业后把池子从无许可改成许可（流动性与 K 线直接死亡）。

---

## 6. 写回缺陷表

把 D7 补进第三章图谱：

| ID | 缺陷 | 已有补丁 | 还洞开 | Mantle |
|---|---|---|---|---|
| D7 | 无许可发射 × 受监管 Quote | RH：链级过滤 + 一级 KYB；Uni v4：Permissioned Pool 标准 | 池层零 KYC；STF 过滤政治成本高；ATS 法律悬空 | 链层只做制裁；股票腿 Adapter+Hook；meme 默认无许可 |

合规不是「多一个开关」，是 **权限放在哪一层**。层放错，要么发射台没人用，要么股票发行人把你告上法庭——RH 这两周两件事都碰到了边。
