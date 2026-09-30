# 支柱 3：衍生品层 —— 股票 Perps 与链原生支持

> **产品定位**：代币化股票全天候杠杆与对冲市场（24/7 Equity Perpetual Derivatives）  
> **核心使命**：避开通用加密 Perps 面对 Hyperliquid 的正面死亡竞争，以代币化美股（Equity Perps）切入，依托 Mantle 现货 xStocks 与 Fluxion RFQ 提供的真实基差腿，将「股票周末休市」从流动性事故转化为独特交易产品。  
> **决策依据**：锁定决策 D2（股票 Perps 优先，Oracle-based 起步）。

---

## 1. 战略定位：为什么是股票 Perps？

### 1.1 通用 Crypto Perps 在 Mantle 的死局
- Mantle 链上通用 Perps 的 24 小时真实交易量长期为 **$0**。
- 正面对抗 Hyperliquid（HL 日均交易量数十亿美元、自主 L1 极致优化）毫无胜算。且 HL 的 HIP-3 正在扩展股票市场，Arbitrum 上亦有 Ostium 等 RWA 衍生品，单纯跟随必死。

### 1.2 Mantle 的绝对不对称优势：真实的现货基差套利腿
- **痛点**：纯合成或预言机驱动的 Perps 最怕资金费率失衡，缺乏现货深度导致套利者不敢入场搬砖。
- **Mantle 的底牌**：Mantle 拥有全球第二大代币化股票库（155 个 xStocks 标的，$633.7M 资产规模）以及 Fluxion Atomic RFQ 的直接 Mint/Redeem 兑换通道。
- **有机飞轮**：
  $$\text{Perp 溢价 (资金费率高企)} \iff \text{做空 Perp} + \text{买入现货 xStocks (Fluxion RFQ)} \implies \text{无风险基差套利}$$
  这为股票 Perps 带来了源源不断的有机做市与套利资金流，**不需要任何不可持续的代币补贴**。

---

## 2. 核心机制设计

### 2.1 Oracle-based 起步（轻量级、高抗压）
- 实测证明：纯 Solidity 链上 CLOB 撮合单笔消耗 **240 万–420 万 gas**。在 Mantle 当前 60M gasLimit 区块下，一个区块仅能承载 14–24 笔撮合，全链上订单簿在物理上不可行（详见本目录 `rise-appspecific-teardown.md`）。
- **v1 选型**：采用类似 Ostium / GMX 的 Oracle-based 模式，轻量高效，零撮合滑点，天然契合 RWA 标的。

### 2.2 休市定价机制（从「事故」变成「产品」）
- **Robinhood Chain 的教训**：现货 AMM 在周末处理美股交易时，发生 HIMS 4.6 倍、AMC 35 倍的荒谬脱锚事故（因库存无法在真实世界交割）。
- **Perps 的制度解法**：现金结算的永续合约是周末与夜间交易的最佳工具。
  - **指数锚定**：休市时 Index Price 冻结于上周五收盘价；
  - **Mark Price 浮动**：Mark Price 由链上多空资金博弈产生；
  - **资金费率钳位（Funding Clamp）**：设置最大 funding rate 上限，开市前 1 小时启动逐步收敛曲线，开市瞬间平滑回归真实市场。

---

## 3. 链级原生支持的三大机制空白

在协议与 Sequencer 层面，Mantle 为 Perps 引入三大行业首创原生支持（详见 `native-chain-support.md`）：

```
 ┌──────────────────────────────────────────────────────────────┐
 │ 1. 清算专用优先通道 (Dedicated Liquidation Lane)             │
 │    保留 10%–15% 区块空间专属配额，普通交易无论加多少 gas 均  │
 │    不可借用，彻底消除剧烈行情下清算交易被挤出的穿仓风险     │
 ├──────────────────────────────────────────────────────────────┤
 │ 2. 协议级「模拟-免费拒绝」(Revert Protection)                │
 │    Sequencer 执行前检测条件，未达成条件的爆仓/止损交易直接   │
 │    丢弃，不写入区块，避免 keeper 和用户因滑点承受无效 gas 损耗│
 ├──────────────────────────────────────────────────────────────┤
 │ 3. 预言机与清算原子绑定 (Oracle-Liquidation Atomicity)       │
 │    同一状态转换内，价格更新完成的纳秒瞬间直接触发依赖该价格  │
 │    的账户爆仓检查，杜绝抢跑套利 MEV 窗口                      │
 └──────────────────────────────────────────────────────────────┘
```

---

## 4. 北极星 KPI

- **未平仓合约量（Open Interest, OI）**：核心标的（NVDAx, TSLAx, SPYx 等）持仓规模。
- **基差收敛速度（Basis Convergence Rate）**：开盘后 Perp 价格向现货指数收敛的时效与摩擦。
- **休市成交占比（Weekend/After-hours Ratio）**：周末与非美股时段的交易量占比（衡量「休市定价即产品」的兑现度）。

---

## 5. 本目录文档索引

### 专属链原生支持（Dedicated Chain-Infra）
| 文档 | 类型 | 核心内容 |
|---|---|---|
| [`chain-infra/README.md`](chain-infra/README.md) | **【衍生品链优化总览】** | 为什么 CLOB 在 EVM 跑不动、Oracle-based 选型逻辑与三大无先例机制空白总述 |
| [`chain-infra/native-chain-support.md`](chain-infra/native-chain-support.md) | **核心报告** | 为 Perps 新增链原生支持的设计空间、十个链级痛点与三个行业首创机制空白（清算通道、模拟拒单、原子绑定） |
| [`chain-infra/perps-infra-notes.md`](chain-infra/perps-infra-notes.md) | **研究底稿** | Perps 链原生设计的工程细节、参数推导与设计空间矩阵 |
| [`chain-infra/rise-appspecific-teardown.md`](chain-infra/rise-appspecific-teardown.md) | **案例拆解** | RISE 为 RiseX 做了什么的工程真相（1.5 Ggas 暴力解、CLOB 真实开销与应用强耦合启示） |
| [`chain-infra/rise-chain-infra.md`](chain-infra/rise-chain-infra.md) | **底层实测** | RISE 节点参数实测、EVM 执行层改动盘点、吞吐与 gas 预算分析 |
| [`chain-infra/risex-app-coupling.md`](chain-infra/risex-app-coupling.md) | **应用拆解** | RISEx 撮合合约逆向、Latency Bumps、Operator 统一代付 gas 经济账 |
| [`chain-infra/appchain-comparables.md`](chain-infra/appchain-comparables.md) | **竞品对标** | Hyperliquid、dYdX v4、Injective、Sei、Berachain 等交易类专属链横向架构全景对比 |
