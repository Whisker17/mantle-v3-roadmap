# Mantle 双 L3 与 L2 跨层可组合性与资金结算架构规范

> **文档定位**：技术架构规范（Engineering Specification）  
> **核心使命**：彻底拆解并解决「Mantle L2 资产信贷总行 + Launchpad L3 资产发行专属链 + Perps L3 衍生品专属链」三者之间的跨层可组合性（Cross-Layer Composability）、远程资金授信与安全结算难题。  
> **存放路径**：`0-narrative/06-cross-layer-interop-and-composability.md`  
> **制定日期**：2026-09-11

---

## 0. 架构全景拓扑与问题定义

在确立「一核双星（Hub & Dual-Spokes）」架构后，系统面临的最严苛挑战是：**“如何让分散在 L2、L3-A、L3-B 的资产与功能模块产生无缝的化学反应，而不让用户和 Agent 感知到跨链的存在，同时避免引入第三方多签桥的黑客攻击面？”**

```
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │                      MANTLE L2：中央资产、信贷与清算总行 (Hub)                          │
 │  • mStocks 现货总金库 ($6.33 亿) ｜ $5.76 亿稳定币沉淀底池 ｜ Mantle Lending 信贷中心   │
 └───────────────────────────┬────────────────────────────┬───────────────────────────────┘
                             │                            │
             【通道 ① 意图闪电垫付通道】                    │ 【通道 ② 远程保证金与授信通道】
             (Intent Fast Relay / ERC-7683)               │ (Remote Margin & Credit Line)
                             ▼                            ▼
 ┌──────────────────────────────────────────────┐ ┌──────────────────────────────────────────────┐
 │       L3-A: Launchpad 专属链 (Tape)          │ │       L3-B: Perps 专属链 (Equity Perps)      │
 │ ──────────────────────────────────────────── │ │ ──────────────────────────────────────────── │
 │ • 50ms 极速 Bonding Curve 发行               │ │ • 24/7 股票永续合约撮合                      │
 │ • 0 Gas 体验 ｜ 状态物理剪枝                 │ │ • 毫秒级仓位风控与专属清算通道               │
 │ • ★【内嵌毕业 AMM DEX】：原地相变二级市场    │ │ • ★【内嵌 RFS 流式做市】：毫秒级紧点差报价   │
 └──────────────────────────────────────────────┘ └──────────────────────────────────────────────┘
```

---

## 1. 模块一：L2 Lending 远程授信机制 (Remote Margin & Credit Delegation)

### 1.1 设计哲学：资产不离 L2，信用映射至 L3
传统跨链模式要求用户把真实 USDT 或 mStocks 从 L2 充值（Bridge）到衍生品链上。这会带来三大死穴：
1. **资金碎片化**：用户在 L2 的抵押品无法再参与 L2 的借贷生息；
2. **提现延迟**：平仓后想回 L2 需要漫长的等待期；
3. **安全风险**：衍生品链的智能合约一旦被黑，存入的全部本金全损。

**Mantle 解法：采用「远程信用额度（Credit Line Delegation）」**：
* 用户的 mStocks 和 USDT **100% 物理存放在 Mantle L2 的安全金库合约**；
* 用户质押资产后，L2 授信合约（`L2CreditHub`）向 L3-B 的 Perps 引擎签署**虚拟信用额度凭证**；
* 真实本金从未离开 L2，L3-B 仅负责高频记账与盈亏（PnL）跟踪。

### 1.2 核心合约交互规范

```solidity
// ==================== L2 侧：信用签发与金库合约 ====================
interface IL2CreditHub {
    // 1. 用户质押 mStocks / USDT 申请 L3-B 信用额度
    function lockAndDelegateCredit(
        address asset,          // 如 mNVDAx 或 USDT
        uint256 amount,         // 质押本金
        uint32 targetChainId,   // L3-B 链 ID
        address agentRecipient  // L3 上的交易账户或 Agent 地址
    ) external returns (bytes32 creditId, uint256 creditUsdValue);

    // 2. 接收来自 L3-B 经过 Sequencer 签名的周期性扎差结算回传
    function settleNetPnL(
        bytes32 creditId,
        int256 netPnL,          // 盈亏金额（可正可负）
        bytes calldata sequencerProof
    ) external;

    // 3. 紧急清算接管（L3 爆仓后划扣 L2 抵押品）
    function executeLiquidationSeizure(
        bytes32 creditId,
        uint256 debtToCover,
        address liquidator
    ) external;
}

// ==================== L3-B 侧：信用额度承接与清算 ====================
interface IL3PerpMarginEngine {
    // 1. 接收来自 L2 的信用额度凭证 (无需资产跨链)
    function creditAccount(
        bytes32 creditId,
        address trader,
        uint256 creditUsdValue
    ) external;

    // 2. 开仓时占用该信用额度作为初始保证金 (IM)
    function openPositionWithCredit(
        bytes32 creditId,
        bytes32 marketId,       // 如 NVDA-PERP
        bool isLong,
        uint256 size,
        uint256 leverage
    ) external;
}
```

### 1.3 跨层清算与扎差流程 (PnL Netting Cycle)
1. **日内高频运转**：
   - 交易者在 L3-B 上以 10 倍杠杆买入 NVDA 股票永续，由于有 L2 授信额度支持，撮合引擎在 **10ms 内直接成交**；
   - 浮动盈亏（Unrealized PnL）全部在 L3-B 内存状态中实时更新。
2. **净额扎差回传（Netting Settlement）**：
   - 每 1 小时（或交易者申请结算时），L3-B 汇总该账户的净盈亏：
     - 若**盈利 $5,000**：L2 结算金库直接给用户的 L2 账户记入 $5,000 利润；
     - 若**亏损 $3,000**：L2 授信合约扣除用户在 L2 的对应抵押品保证金。
3. **穿仓防护**：
   - 当账户保证金率触及维持保证金（MM）阈值，L3-B 极速清算通道（Liquidation Lane）在 50ms 内强制平仓；
   - 若出现极端滑点穿仓，由 L2 保险基金与做市商金库（Omni-Vault）优先填补，绝不波及 L2 大盘。

---

## 2. 模块二：Launchpad L3 内嵌毕业 AMM 机制 (Embedded Graduation AMM)

### 2.1 为什么必须内嵌？严禁“毕业即跨链”
历史上的跨链 Launchpad 往往犯下一个致命错误：**打新在一条链，毕业后把流动性跨到另一条链开盘**。
这会导致：
* **致命真空期**：跨链需要等待 5–15 分钟确认，玩家打新的 FOMO 情绪在等待中彻底冷却；
* **操作割裂**：用户打完新，还要去切换钱包网络、准备新链的 Gas 币才能参与二级买卖，散户流失率高达 70% 以上。

### 2.2 同区块相变流水线 (In-Block Phase Transition)
在 L3-A（Launchpad 专属链）中，毕业机制被设计为**同一个区块内的原子相变（Atomic Phase Shift）**：

```
 [阶段 1: Bonding Curve 内盘]
 散户与 Agent 抢购 Meme (以 mStocks 计价)
                 │
                 │ 累计募集达到 $12,000 mStocks 阈值 (Trigger Point)
                 ▼
 [阶段 2: 状态机原子相变 (Same Block Transition)]
 ① 冻结 Bonding Curve 曲线合约；
 ② 自动提取曲线内全部 $12,000 mStocks 与对应代币储备；
 ③ 原生调用内嵌 AMM 核心工厂 (Agni/Fluxion V4 Core)；
 ④ 部署全区间流动性池，并将 LP 凭证永久锁死/销毁至黑洞地址；
 ⑤ 铸造该池的唯一治理凭证：Fee Key NFT，派发给代币创建者。
                 │
                 ▼
 [阶段 3: 内嵌 AMM 二级外盘开市 (Zero Latency)]
 用户无需换网、无需等待，在同一界面直接使用 AMM 深度进行二级买卖！
```

### 2.3 Fee Key NFT 现金流永续分润
内嵌 AMM 产生的手续费（标准 1%）分配规则由合约在相变时固定：
* **45% 永续流向 Fee Key NFT**：代币创建者或社区持有该 NFT，可直接在 L3-A 提领每日分红，亦可将该 NFT 在市场上质押折现；
* **25% 注入 L3-B 的 RFS 现货深度金库**：做厚底层美股代币的做市底池；
* **20% 强制回购 MNT**：定期汇总跨层至 L2 触发市价回购 MNT；
* **10% 协议运维储备**。

---

## 3. 模块三：Perps L3 内嵌 RFS 流式做市机制

### 3.1 做市商流式签名与毫秒级保护 (Streaming Quotes)
为解决高频永续合约中做市商（MM）的报价滞后与被狙击痛点，**RFS（Request for Stream）被直接内嵌在 L3-B 引擎层**：
1. **流式软报价（Streaming Soft Quotes）**：
   - Bybit 机构做市商通过全双工 WebSocket 长连接向 L3-B 节点广播带毫秒级时效的 EIP-712 签名报价：
     $$\text{Quote} = \{\text{Symbol}: \text{NVDA}, \text{Bid}: 118.20, \text{Ask}: 118.25, \text{ValidUntil}: t + 500\text{ms}, \text{Nonce}: N\}$$
2. **做市商撤单绝对优先排单（Cancel Priority）**：
   - 当纳斯达克现货出现剧烈跳水，做市商发出的撤单（Cancel）交易在 L3-B Sequencer 拥有**无条件最高排单权重**；
   - 彻底杜绝套利 Bot 利用 100ms 延迟差吃掉做市商的陈旧亏损单（Stale Quotes）。

### 3.2 跨层基差套利管道 (Basis Arbitrage Flow)
Perps L3 的最大核心竞争力，是与 **Mantle L2 现货总金库中的 mStocks 形成确定性无风险套利**：

```
 [美股财报发布：市场狂热看多 NVDA]
                 │
                 ▼
 ┌─────────────────────────────────────────────────────────────┐
 │  Perps L3：NVDA-PERP 溢价，资金费率飙升至 +120% APR           │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                 ┌──────────────┴──────────────┐
                 ▼ 套利 Bot 触发双腿对冲策略   ▼
 ┌──────────────────────────────┐ ┌─────────────────────────────┐
 │ 腿 1 (在 L3-B 卖出开空)：    │ │ 腿 2 (在 L2 买入现货)：      │
 │ • 在 Perps L3 做空 NVDA-PERP │ │ • 在 L2 现货池买入 mNVDAx   │
 │ • 每 8 小时吃满 +120% 资金费 │ │ • 对冲现货敞口，实现 Delta 0│
 └──────────────────────────────┘ └─────────────────────────────┘
                                │
                                ▼
 无论现货暴涨还是暴跌，套利者实现完全市场中性，稳赚确定性年化！
 这为整个生态提供了海量、健康的真实有机交易量 (Volume)。
```

---

## 4. 模块四：跨层 Fast Intent 意图闪电通道 (Intent Fast Relay)

### 4.1 用户无感操作流（基于 ERC-7683 标准）
散户和普通 Agent 资金在 L2，如何在 100ms 内直接买到 L3-A 的 Meme 或在 L3-B 开单？

我们采用 **ERC-7683 跨链意图标准 + 做市商/Relayer 毫秒垫付模型**：

```
 [步骤 1: 客户端单次签名]
 用户在前端点击「买入 $500 NVDA-MEME」，钱包弹窗签署一段 EIP-712 意图：
 "授权在 L2 锁定 500 USDT，换取在 L3-A 获取对应 Meme 代币，滑点 ≤ 1%"
                 │
                 ▼
 [步骤 2: Solver 毫秒级抢单垫付 (50ms)]
 L3-A 的官方 Fast Relayer (或外部 Solver) 监听到该签名；
 Relayer 立即在 L3-A 本地出资 500 USDT 替用户代为买入 Meme 并存入用户 L3 地址；
 用户在 100ms 内看到交易完成并拿到代币！
                 │
                 ▼
 [步骤 3: 异步提款与批量结算]
 Relayer 凭用户的有效签名，去 L2 的 `L2DepositVault` 合约领取属于它的 500 USDT 本金。
```

**散户体验评价**：用户完全不需要打开什么跨链桥界面、不需要等 10 分钟确认，**感知上与在同一条链内 Swap 没有任何区别**。

---

## 5. 模块五：安全防线与风控硬约束 (Rate-Limiter Circuit Breaker)

### 5.1 彻底封死跨链攻击面：速率限流熔断器
历史跨链桥被盗的根本原因，是攻击者利用合约漏洞在子链凭空伪造巨额资金，然后瞬间将 L1/L2 存管金库全部提空。

在 Mantle L2 的中央出金金库中，我们强制部署 **硬编码级速率限制器（Rate Limiter）**：

```solidity
contract L2SafetyVault {
    uint256 public constant MAX_HOURLY_OUTFLOW_PERCENT = 2; // 单小时最多流出 2%
    uint256 public constant HOURLY_VOLUME_CAP = 5_000_000 * 1e6; // 单小时提现绝对硬顶 $5M
    
    uint256 public currentHourOutflow;
    uint256 public lastHourTimestamp;

    // 提现检查
    function withdraw(address recipient, uint256 amount) external onlyCanonicalBridge {
        if (block.timestamp >= lastHourTimestamp + 1 hours) {
            currentHourOutflow = 0;
            lastHourTimestamp = block.timestamp;
        }

        require(currentHourOutflow + amount <= HOURLY_VOLUME_CAP, "RATE_LIMIT_EXCEEDED");
        require(amount <= (totalReserve * MAX_HOURLY_OUTFLOW_PERCENT) / 100, "HOURLY_PERCENT_CAP");

        currentHourOutflow += amount;
        // 执行放款...
    }
}
```

### 5.2 安全性收益评估
1. **隔离爆破半径**：即使某一条 L3 遭遇了极端零日漏洞（Zero-day），黑客在单个小时内最多只能提走总池的 2%；
2. **充足的人工干预时间窗口**：监控探针检测到异常大额流出会立即向安全委员会报警，风控团队有整整 60 分钟时间排查并触发全局暂停合约；
3. **Mantle L2 的 25 亿国库与 5.76 亿稳定币底池拥有不可撼动的物理级安全保障**。

---

## 6. 总结与落地标准

通过上述五大模块的精密配合，Mantle 彻底破解了多链系统的三大经典矛盾：
* **速度 vs 安全**：高频换手在 50ms 的专有 L3 肆意狂飙，大额资产在以太坊 ZK 结算的 L2 固若金汤；
* **分散 vs 聚合**：Launchpad 毕业资产就地消化，Perps 享受 L2 统一保证金，零资产碎片化；
* **用户体验 vs 架构解耦**：底层是精巧的“一核双星”，表层是对用户完全无感的一键意图流转。
