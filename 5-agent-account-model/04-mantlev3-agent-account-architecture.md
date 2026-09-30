# 04. Mantle v3 Agent Accounts 推荐架构与路线图落地设计

> **文档定位**：Mantle v3 Agent 账户模型（Agent Accounts / Wallets）前期技术预研（Part 4/4）  
> **核心命题**：如何依托 Mantle v3 的底层技术基线（OP Succinct + SP1 ZK + revm 执行层），打造专为 Tape Meme 与股票 Perps 服务的下一代 Agent 账户体系？  
> **制定日期**：2026-09-10

---

## 1. 架构总原则：拒绝抽象空转，分层渐进落地

构建 Agent 账户模型绝不能脱离具体产品陷入「通用 AA 基础设施」的自我感动。依据仓库整体设计原则：**「基础设施没有独立于产品的价值，必须为具体交易场景服务」**。

Mantle 拥有全生态独占的底层改造优势——自 2025-09 转向 **OP Succinct（SP1 ZK validity proof）**，拥有独立维护的 `op-geth`、`mantle-xyz/revm` 与 `kona` Rust 子树（详见 `0-narrative/04-mantle-opstack-surface.md`）。在 SP1 zkVM 路线下，扩展执行层与交易类型无需在链上编写 MIPS 解释器，只需在 Rust 内实现并编译入 guest ELF 即可由 SP1 统一证明。

基于这一事实，我们将 Mantle v3 Agent Account 划分为**三层立体架构**，并采取**双阶段渐进式落地策略**：

```
 ┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
 │                       Mantle v3 Agent Account 三层立体架构 (System Topology)                     │
 ├──────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ 【顶层：策略引擎与限权委托 (Scoped Policy Engine)】                                              │
 │  • ERC-7579 标准化模块 + Agent Session Keys                                                      │
 │  • 细粒度声明：合约白名单、函数选择器过滤、Spend-Gate 额度钳位、时效窗口与链上最大回撤熔断断路器│
 ├──────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ 【中层：极速路由与原生代付 (L1 Sequencer Routing & Sponsorship)】                               │
 │  • EIP-7966 (`eth_sendRawTransactionSync`) 零延迟同步回执                                        │
 │  • 200ms Flashblocks 预确认流与 Cancel 优先通道                                                  │
 │  • 业务专有 Paymaster (支持 Tape 打新免 Gas、股票 Perps 允许使用 mStocks / USDT 支付 Gas)        │
 ├──────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ 【底层：L2 协议级执行内核 (revm Native AA & 2D Nonce)】                                          │
 │  • RIP-7560 原生交易类型 (免 Bundler 中继，Sequencer 纳秒级直达验证)                             │
 │  • 二维并发 Nonce (`SequenceKey + SequenceNumber`)，彻底终结 Meme 抢买与做市清算间的线头阻塞     │
 │  • revm 级 Session Key 策略预编译加速 (Rust 原生验签与限额检查，单笔节省 40k+ Gas)               │
 └──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 三层体系核心技术设计

### 2.1 顶层：细粒度 Agent Session Key 策略引擎（Policy Engine）

针对 Agent 链下运行环境脆弱的特点，主账户（Human Owner / Treasury Vault）签发的不是私钥，而是**附带严格状态约束的临时 Session 凭据**：

```rust
/// Mantle revm 执行层可识别的 Agent 策略结构体定义
pub struct AgentSessionPolicy {
    pub session_key: Address,        // 临时 Agent 签名公钥 (可运行于普通云端或 TEE)
    pub target_whitelist: Vec<Address>, // 允许交互的合约白名单 (如: TapePool, PerpDEX)
    pub selector_whitelist: Vec<[u8; 4]>, // 允许调用的方法 (如: buy(), openPosition())
    pub max_spend_per_block: U256,   // 单块最大支出上限 (以 USDT/mStocks 折算)
    pub total_budget: U256,          // 本 Session 累计生命周期预算
    pub valid_after: u64,            // 生效时间戳
    pub valid_until: u64,            // 失效时间戳
    pub stop_loss_drawdown_bps: u16, // 最大允许累计回撤基点 (例如 500 = 5% 自动熔断)
}
```

#### 链上安全保障：
1. **零本金盗取可能**：Session Key **无法调用原生 `transfer` 或 ERC-20 `transfer` 向非白名单地址转账**。哪怕黑客通过 Prompt 注入完全控制了 Agent，其最大破坏力被锁死在 `total_budget` 范围内的正常买卖滑点中；
2. **时间炸弹机制（Time-Bomb）**：针对抢打新场景，Session Key 生效时间仅配置 15–30 分钟，开盘结束自动失效；
3. **链上断路器（Circuit Breaker）**：Hook 模块实时监控账户净值，若触发 `stop_loss_drawdown_bps` 异常回撤，策略引擎立即在底层标记该 Session 失效并冻结开仓权限。

---

### 2.2 中层：L1 极速路由、并发调度与原生代付

本层全部落在 OP Stack 的 L1 改造面（不改变状态转换，不破坏 EVM 等价性，不触发 SP1 ZK vkey 轮换）：

1. **RPC 零延迟同步往返（EIP-7966 `eth_sendRawTransactionSync`）**：
   - 传统 RPC 发送交易后需轮询 `getTransactionReceipt`，产生 1–2 秒无谓等待；
   - 引入 EIP-7966 后，Agent 发送交易即在单次 HTTP/WS 往返中直接获得 Sequencer 的包含执据与模拟执行结果，延迟压缩至 **10ms–30ms**。
2. **专属 Paymaster 赞助池与代币化结算**：
   - **Tape Meme 场景**：由 Tape 协议金库或项目方自营 Relayer 承担 Gas，实现「零 Gas 打新」；
   - **股票 Perps 场景**：允许 Agent 账户直接从已存入的 `USDT`、`NVDAx` 保证金中扣除等值 Gas，彻底终结「账户持有百万资产却因缺 0.1 MNT 被迫僵死」的系统性隐患。
3. **200ms Flashblocks 预确认流与 Cancel 优先**：
   - 与 `rfq-propamm/chain-infra/` 的 Flashblocks 预确认流保持一致，Agent 可以在 200ms 内确定交易挂单状态，做市商撤单请求走专用最高优先级通道。

---

### 2.3 底层：L2 执行层原生 AA 与二维 Nonce 引擎（revm 改动）

依托 Mantle 独立的 `mantle-xyz/revm` 与 SP1 ZK 架构，我们设计了最具突破性的执行层原生优化：

#### 1. 二维并发 Nonce（2D Multi-Channel Nonce）
协议层将账户 Nonce 拓展为两维映射：
$$\text{StateNonce} = (\text{SequenceKey} \in [0, 2^{64}-1], \ \text{SequenceNumber} \in [0, 2^{64}-1])$$

- **通道 0（`SequenceKey = 0`）**：保留为用户主账户的人类手动操作通道；
- **通道 1（`SequenceKey = 1`）**：分配给 **Tape Meme 极速打新 Agent**；
- **通道 2（`SequenceKey = 2`）**：分配给 **股票 Perps 网格做市 Agent**；
- **通道 999（`SequenceKey = 999`）**：分配给 **Perps 仓位自保紧急清算通道（Emergency Lane）**。

**彻底消除线头阻塞**：通道 1 中的 Meme 抢买即使因为 Spend-Gate 暂时 Pending，通道 2 的做市挂单与通道 999 的紧急补仓交易将**并行不悖地立即执行，彻底消除因抢 Meme 失败导致 Perps 仓位爆仓的系统级荒谬故障**。

#### 2. revm 级 Session Key 策略预编译加速
传统在 Solidity 智能合约中校验 Session Key 的白名单 Merkle Proof 或多重 Storage 读取，单笔通常消耗 **30,000–60,000 Gas**。
我们在 `mantle-xyz/revm` 中实现原生预编译加速：
- 策略哈希直接缓存在 Rust 状态机执行上下文中；
- 单笔 Session Key 策略校验在 Rust 原生层完成，仅需几十纳秒，Gas 消耗直接降至 **< 3,000 Gas**，为高频 Agent 交易抹平与 EOA 的性能差距。

---

## 3. 针对 Tape 与 Perps 业务的落地特化配置

```
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
 │                           两大支柱业务的 Agent 预设配置文件 (Profiles)                          │
 ├─────────────────────────────────────────┬───────────────────────────────────────────────────────┤
 │ 【Profile A: Tape Meme Sniper】         │ 【Profile B: Perps Guardian & Arbitrage】             │
 ├─────────────────────────────────────────┼───────────────────────────────────────────────────────┤
 │ • 绑定通道: SequenceKey = 1             │ • 绑定通道: SequenceKey = 2 (套利) / 999 (紧急自保)   │
 │ • 授权白名单:                           │ • 授权白名单:                                         │
 │   - Target: `TapeLaunchpadHook`         │   - Target: `MantlePerpRouter`, `FluxionRFQ`          │
 │   - Selectors: `buy()`, `claimFeeKey()` │   - Selectors: `openPosition()`, `addMargin()`, `fill`│
 │ • 额度限制:                             │ • 额度限制:                                           │
 │   - 单代币最多支出 500 NVDAx/TSLAx      │   - 最大杠杆率钳位: 10x                               │
 │   - 仅限当前开盘时段 (有效窗口: 30 mins)│   - 单日最大已实现亏损: 账户净值 5% (触发立即平仓)    │
 │ • Gas 策略:                             │ • Gas 策略:                                           │
 │   - 由 Tape 官方 Paymaster 100% 赞助    │   - 允许从 Perp 保证金 (USDT / xStocks) 中原生划扣 Gas│
 └─────────────────────────────────────────┴───────────────────────────────────────────────────────┘
```

---

## 4. 阶段性路线图与落地时间表（Roadmap）

为了平衡上线节奏、工程风险与生态交付，我们规划了清晰的「三步走」实施路线图：

```
 2026 Q3 (Phase 1)                     2026 Q4 (Phase 2)                     2027 Q1 (Phase 3)
 ┌───────────────────────────┐         ┌───────────────────────────┐         ┌───────────────────────────┐
 │ 【Phase 1: 合约与应用层】  │ ──►     │ 【Phase 2: RPC 与调度层】 │ ──►     │ 【Phase 3: revm 原生内核】│
 │ • ERC-7579 模块化智能账户 │         │ • EIP-7966 同步回执 RPC   │         │ • RIP-7560 Native AA 落地 │
 │ • Session Key 基础策略库  │         │ • 自营 Paymaster 代付中继 │         │ • 二维并发 Nonce 引擎     │
 │ • 适配 Tape 极速打新      │         │ • 200ms Flashblocks 接入  │         │ • SP1 ZK guest ELF 证明   │
 └───────────────────────────┘         └───────────────────────────┘         └───────────────────────────┘
```

### 阶段 1：合约与模块化应用层（2026 Q3，快速点火）
- **目标**：不改底层链，用成熟标准快速支持 Tape Meme Launchpad 上线；
- **交付内容**：
  1. 部署基于 **ERC-7579** 的轻量级模块化账户实现；
  2. 提供开箱即用的 `TapeSniperSessionValidator` 模块，支持用户在前端一键向 Agent 签发受限打新凭据；
  3. 搭建官方 ERC-4337 Paymaster，为符合 Spend-Gate 条件的用户提供全免 Gas 打新。

### 阶段 2：RPC 与 Sequencer 调度层（2026 Q4，高频提速）
- **目标**：优化节点通信与交易确认链路，满足 Perps 做市与套利时效；
- **交付内容**：
  1. 在 `op-geth` 中启用 **EIP-7966 `eth_sendRawTransactionSync`**，降低 Agent 往返延迟至数十毫秒；
  2. 落地 **200ms Flashblocks 软确认流**与做市商 Cancel 优先通道；
  3. 推出支持 USDT / mStocks 扣费的 L2 协议级自营 Relay，消除 Agent 的 MNT Gas 垫资依赖。

### 阶段 3：revm 原生内核与多维并发（2027 Q1，终极形态）
- **目标**：在 Mantle 执行层原生打通 RIP-7560，彻底消灭线头阻塞，达到全网最顶级的 Agent 基础设施；
- **交付内容**：
  1. 在 `mantle-xyz/revm` 中实现 **二维并发 Nonce（2D Nonce）**，彻底物理隔离打新与清算通道；
  2. 实现 **Session Key 策略引擎预编译加速模块**，将验签与限权 Gas 压缩至极限；
  3. 完成修改后的 `kona` 与 SP1 guest ELF 编译与 vkey 轮换，实现全链 ZK validity proof 闭环。

---

## 5. 总结与战略价值

通过这一全栈架构改造，Mantle v3 不仅解决了传统 EVM 账户对 Agent 的束缚，更在整个 L2 生态中建立了极具统治力的技术护城河：
1. **对 Meme 创作者与玩家**：提供了全网最安全的「一键托管抢打新」体验，既有极速爆发力，又杜绝了被盗币的后顾之忧；
2. **对专业做市商与量化机构**：消除了多账户资金碎片化，多通道 2D Nonce 确保紧急清算自保与做市挂单零冲突；
3. **对 Mantle 资本市场生态**：让链上沉淀的 $5.76 亿闲置资金能够安全、受控、高频地授权给链下 AI Agent 运转起来，真正盘活流动性，实现 **「Where Assets Go Public」** 的战略终局。
