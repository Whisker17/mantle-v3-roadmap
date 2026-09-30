# 支柱 5：接口与分发层 —— Tape Terminal & Bybit Alpha（执行中枢）

> **产品定位**：资本市场的交易终端与超级流量入口（Trading Terminal & Distribution Hub）  
> **核心使命**：打破 Mantle 在专业交易工具链上「零覆盖」的死锁困境，以「自建原生 Terminal + Bybit Alpha 白标托管」双前端战略，抢占价值捕获最丰厚的接口层。  
> **关联规划**：参考第七部分 7.7 节接口层与 7.8 节 W1 白标工作流。

---

## 1. 为什么接口层是生死攸关的最高优先级？

### 1.1 行业实证：价值捕获排序「接口层 > 协议层 > 链层」
根据跨链实测与行业财务数据，交易接口层展现出惊人的流动性掌控力：
- **GMGN** 贡献了 Robinhood Chain 上 **40% 以上** 的 DEX 现货交易量；
- **GMGN / Photon / BullX** 的年度协议佣金收入规模，已与 pump.fun 等头部发行协议达到同一量级；
- **Bankr** 在 Base 上的交易撮合量是 Base 官方三大主流协议总和的 3.9 倍。

### 1.2 Mantle 的现实死穴：执行层全面「零覆盖」
全网主流的 Meme/DEX 专业执行工具对 Mantle 的支持度为 **0**：
- ❌ **GMGN、Photon、BullX、Axiom、Trojan、Banana Gun、Maestro 全部不支持 Mantle**；
- GMGN 的链列表甚至已收录 Monad、MegaETH、X Layer 与 Robinhood Chain，但**唯独没有 Mantle**；
- ❌ **Phantom** 等主流多链钱包不支持 Mantle 原生网络。

**结论**：等待第三方工具主动适配无异于坐以待毙。**自建专业 Terminal 是盘活 Mantle 交易生态唯一可行的自救路径。**

---

## 2. 双前端战略架构

```
                        ┌───────────────────────────────┐
                        │      终端用户分层与入口       │
                        └───────┬───────────────┬───────┘
                                │               │
                ┌───────────────┘               └──────────────┐
                ▼                                              ▼
 ┌─────────────────────────────┐               ┌─────────────────────────────┐
 │ 路径 A：自建 Tape Terminal  │               │ 路径 B：Bybit Alpha 白标端  │
 │ (面向 Web3 链上原生交易者)  │               │ (面向 Bybit 8000万 托管用户)│
 ├─────────────────────────────┤               ├─────────────────────────────┤
 │ - 发现：按股票 Quote 战壕板 │               │ - 免助记词：Bybit 现货账户 │
 │ - 安全：老鼠仓/Fee Key 审计 │               │ - 免 Gas 费：交易所全代付   │
 │ - 门禁：Spend-Gate 实时额度 │               │ - 体验：一键认购，后台结算 │
 │ - 体验：Passport + Paymaster│               │ - 角色：将 Tape 作为发行底座│
 └──────────────┬──────────────┘               └──────────────┬──────────────┘
                │                                              │
                └──────────────────────┬───────────────────────┘
                                       │ 统一接入链上核心层
                                       ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ Mantle L2 核心协议层 (Tape Launchpad 曲线 / Fluxion RFQ / 股票 Perps)      │
 └───────────────────────────────────────────────────────────────────────────┘
```

### 2.1 路径 A：自建 Tape Terminal 核心功能迭代
- **P0 基础执行（MVP）**：
  1. **资产发现**：实时新币发射流（New Pairs Stream）、按底层股票 Quote（NVDAx, TSLAx）分组的「战壕面板」；
  2. **安全透视**：流动性永久锁仓验证、Creator 持仓比例警报、老鼠仓（Insider）持仓分布分析；
  3. **快捷执行**：单键极速买卖（Snipe/Sell）、限价止盈止损单、主流代币一键兑换为 xStocks Quote；
  4. **门禁看板**：实时展示用户基于 mETH / xStocks / Bybit 等级计算的 Spend-Gate 剩余打新额度。
- **P1 移动化与无感体验**：
  - 深度集成 **Mantle Passport**（基于 Passkey 的社交/指纹登录无助记词钱包）；
  - 接入 **ERC-4337 Paymaster**，支持直接使用 USDT / USDC / MNT 或由协议完全补贴 Gas。

### 2.2 路径 B：Bybit Alpha 白标合作（W1 工作流）
- **商业原则**：坚决不试图强迫 Alpha 托管用户安装链上钱包「导流上链」（这逆向对抗产品漏斗，必败）。
- **落地形态**：Alpha 将 Tape 作为其专属的代币化资产发行引擎。用户在 Alpha 前端直接操作，底层交易在 Mantle 链上实时透明清算。

---

## 3. 北极星 KPI

- **Tape Terminal 日活跃地址（Terminal DAU）**：衡量链上原生专业交易者的留存。
- **白标端成交占比（Whitelabel Volume Share）**：Bybit Alpha 端贡献的曲线交易量占比目标 $\ge 50\%$。
- **生态聚合接入进度**：优先打通 Birdeye 深度供数（Birdeye 已为 Bybit Alpha 供数，具备现成商务基础）。

---

## 4. 本目录文档索引

### 专属体验链级优化（Dedicated Chain-Infra）
| 文档 | 类型 | 核心内容 |
|---|---|---|
| [`chain-infra/README.md`](chain-infra/README.md) | **【终端体验链优化总览】** | Mantle Passport（Passkey 社交/生物识别无助记词登录）、Session Key 免弹窗与 ERC-4337 Paymaster 全链免 Gas |
