# Mantle v3 路线图：CeDeFi Capital Markets 叙事重构与全栈架构

> **项目名称**：`mantlev3-roadmap`  
> **核心叙事**：**CeDeFi Capital Markets（链上资本市场）**  
> **战略 Tagline**：**Where Assets Go Public（万物上市）**  
> **核心破局点**：承认历史，完成闭环 —— 将 Mantle 从沉淀了 $576M 资金却无处交易的「链上银行」，重构为集**一级发行、二级做市、现货交易、全天候衍生品、杠杆借贷与最终清算**于一体的完整「链上资本市场」。

---

## 1. 架构总览：三位一体的模块化体系

本仓库按照「**核心宏观叙事总纲 ──► 产品矩阵与专属链优化 ──► 交易所动力挂斗（Sidecar）**」三层结构进行系统性规划：

```
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │                                   【母目录 1】宏观叙事与总路由                             │
 │   narrative/ (CeDeFi Capital Markets 核心战略总纲，阐明从链上银行到资本市场，并路由至各产品)│
 └────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                              │
                     ┌────────────────────────┴────────────────────────┐
                     │ 全景产品矩阵 (每个产品均内嵌专属定制的 Chain-Infra 优化)│
                     ▼                                                 ▼
 ┌──────────────────────────────────────────┐      ┌──────────────────────────────────────────┐
 │ 支柱 1：发行层                           │      │ 支柱 2：现货交易层                       │
 │ meme-launchpad/                          │      │ rfq-propamm/                             │
 │ ├─ Tape (股票 quote 情绪一级市场)        │      │ ├─ Fluxion RFQ (全资产私有库存报价)      │
 │ ├─ benchmarks/ (Solana/BSC/Base/RH)      │      │ └─ chain-infra/ (无公开mempool/Cancel优先│
 │ └─ chain-infra/ (转账钩子/出块解耦)      │      │                 Flashblocks预确认流)     │
 └──────────────────────────────────────────┘      └──────────────────────────────────────────┘
                     │                                                 │
                     ▼                                                 ▼
 ┌──────────────────────────────────────────┐      ┌──────────────────────────────────────────┐
 │ 支柱 3：衍生品层                         │      │ 支柱 4：融资借贷层                       │
 │ perps/                                   │      │ lending/                                 │
 │ ├─ 股票 Perps (休市定价即产品/基差套利)  │      │ ├─ Margin Lending (xStocks/mETH 质押借贷)│
 │ └─ chain-infra/ (清算通道/模拟免拒/原子化│      │ └─ chain-infra/ (市场日历/周末动态折价)  │
 │                 RISE 1.5Ggas CLOB实测)   │      └──────────────────────────────────────────┘
 └──────────────────────────────────────────┘                          │
                     │                                                 │
                     └────────────────────────┬────────────────────────┘
                                              ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ 支柱 5：接口与分发层                                                                     │
 │ terminal/                                                                                │
 │ ├─ Tape Terminal (自建原生战壕面板/安全审计) + Bybit Alpha 白标端                        │
 │ └─ chain-infra/ (Mantle Passport Passkey 登录 + ERC-4337 Paymaster 全免 Gas)             │
 └────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                              │
                                              ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │                                   【母目录 2】外部动力系统挂斗                           │
 │   sidecar/ (Bybit CeDeFi 全栈协同：W1 经纪商 + W2 做市商 + W3 自营发行 + W4 UTA 保证金)  │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心叙事弧线：从「链上银行」到「链上资本市场」

### 2.1 过去三年：建成了买方，却缺少市场
过去三年，Mantle 投入巨资（EcoFund $2 亿、Methamorphosis、Journey 2,500 万 MNT）建成了完备的买方资产库：
- **全球第 2 大代币化股票资产库**：155 个 xStocks 标的，AUM 达 **$633.7M**（是 Robinhood 链上规模的 4.8 倍）；
- **雄厚资金沉淀**：**$576M 闲置稳定币**（为链上 DeFi TVL 的 5.9 倍），mETH / cmETH 生息资产，UR 新银行入口；
- **历史归因**：过去的激励机制全部为质押与锁仓，**「奖励不动的钱」，从未建立起「奖励交易的飞轮」**，导致链上有钱却无市场，日均 DEX 交易量长期低至 **$0.94M**。

### 2.2 Mantle v3：建设卖方与场内，让钱动起来
> **「银行聚集了钱，资本市场让钱动起来。」**

新叙事不推翻历史，而是完成商业闭环：
- **过去建买方**：银行入口、资管产品、生息沉淀；
- **现在建卖方与场内**：一级发行（Tape）、二级做市（Fluxion RFQ）、全天候衍生品（股票 Perps）、杠杆融资（Margin Lending），并由 Bybit 作为 Sidecar 提供承销、做市与全球 8,000 万用户分发。

---

## 3. 基础设施哲学：拒绝抽象空转，Chain Infra 必须为具体产品服务

本仓库彻底摒弃将基础设施作为独立孤岛的传统模式：
- **基础设施没有独立于产品的价值**：脱离了具体交易场景的改链是工程师的自我感动；
- **各产品定制专属链优化**：
  - **Launchpad 需要**：转账钩子（Transfer Hook）、出块时间解耦的 Spend-Gate 门禁（`meme-launchpad/chain-infra/`）；
  - **RFQ 现货需要**：做市商撤单优先通道（Cancel Priority FIFO）、无公开 Mempool 天然抗夹、200ms Flashblocks 预确认（`rfq-propamm/chain-infra/`）；
  - **股票 Perps 需要**：清算专用保留通道（Dedicated Liquidation Lane）、模拟-免费拒绝（Revert Protection）、预言机与清算原子绑定（`perps/chain-infra/`）；
  - **Lending 借贷需要**：市场日历预编译（Market Calendar Precompile）、周末动态折价（Weekend Haircut）（`lending/chain-infra/`）；
  - **Terminal 终端需要**：Passkey 无助记词登录、ERC-4337 Paymaster 全链免 Gas（`terminal/chain-infra/`）。
- **底层改造技术底气**：Mantle 已经转向 **OP Succinct（SP1 ZK validity proof）**，摆脱了 Optimism 传统 MIPS 单指令仲裁约束，魔改执行层（op-geth / revm）的技术成本与治理风险完全可控（详见 `narrative/04-mantle-opstack-surface.md`）。

---

## 4. 四大锁定决策（Decision Log）

| # | 决策点 | 选定方案 | 核心依据 |
|---|---|---|---|
| **D1** | **主叙事** | **CeDeFi Capital Markets**；Tagline: **Where Assets Go Public** | 覆盖全部产品支柱、机构友好、对 Bybit 组织变动鲁棒；明确不推泛化的"交易所链" |
| **D2** | **Perps 路线** | **股票 Perps 优先，Oracle-based 起步** | 通用 crypto perps 面对 Hyperliquid 必败；Mantle 独有现货 xStocks + Fluxion RFQ 真实基差腿；休市定价做成产品而非事故 |
| **D3** | **Launchpad 范围** | **股票 Quote 专注（Tape 方案）** | 通用 meme 在 Mantle 经历两次关停；meme × 代币化美股经 Robinhood PAIR/LONG 与 BNB bStocks 验证为真实赛道 |
| **D4** | **Bybit 合作深度** | **全深度：白标 + 做市 + 自营发行 + UTA 打通** | 拆分为四条可独立验收工作流（W1–W4）；主线仅依赖 W1+W2，W3/W4 作为重大战略 Upside |

---

## 5. 全仓库目录与文档完整导航

### 🏛️ 母目录 1：宏观叙事与产品总路由（[`narrative/`](narrative/)）
- [`narrative/README.md`](narrative/README.md)：**【高层母篇】** 宏观叙事总纲、各产品矩阵系统介绍与跳转路由、基础设施哲学与 MNT 价值捕获。
- [`narrative/01-repositioning-and-roadmap.md`](narrative/01-repositioning-and-roadmap.md)：**【决策定稿】** 四大锁定决策全景推导、五支柱业务闭环、三轨路线图与开放问题。
- [`narrative/02-gap-analysis.md`](narrative/02-gap-analysis.md)：**【现状诊断】** 链空事实、执行层零覆盖、两次失败复盘与六张独有底牌。
- [`narrative/03-mantle-baseline.md`](narrative/03-mantle-baseline.md)：**【底层实测】** RPC 数据实测、TVL 崩塌序列、28 个 DEX 活跃度审计。
- [`narrative/04-mantle-opstack-surface.md`](narrative/04-mantle-opstack-surface.md)：**【底层改造面】** OP Succinct（SP1 ZK proof）架构审计与 Treasury 治理预算。
- [`narrative/05-infra-requirements-framework.md`](narrative/05-infra-requirements-framework.md)：**【需求框架】** 14 维链级基础设施需求框架与 LFM in EVM 阶梯。

---

### 🚀 产品矩阵（每个产品内嵌专属 `chain-infra/`）

#### 支柱 1：发行层（[`meme-launchpad/`](meme-launchpad/)）
- [`meme-launchpad/README.md`](meme-launchpad/README.md)：代币化股票情绪一级市场总览、Tape 核心机制、北极星 KPI 与反目标。
- [`meme-launchpad/tape-design.md`](meme-launchpad/tape-design.md)：Tape 协议完整设计方案（xStocks bonding curve、Spend-Gate、双创新 Hook、费用路由）。
- [`meme-launchpad/xstocks-inputs.md`](meme-launchpad/xstocks-inputs.md)：xStocks 可组合性约束与代币化 IPO 输入。
- [`meme-launchpad/chain-infra/`](meme-launchpad/chain-infra/)：**【发行层专属链优化】**
  - [`meme-launchpad/chain-infra/README.md`](meme-launchpad/chain-infra/README.md)：为什么 Launchpad 需要专属链优化。
  - [`meme-launchpad/chain-infra/issuance-primitives.md`](meme-launchpad/chain-infra/issuance-primitives.md)：资产发行原生链级原语盘点（Token Extensions、转账钩子）。
- [`meme-launchpad/benchmarks/`](meme-launchpad/benchmarks/)：**【行业竞品全面调研】**
  - Solana pump.fun 机制与实测、BSC four.meme bStocks 拆解、Base 生态反思、Robinhood Chain Pons/PAIR 深度拆解（共 7 篇文献）。

#### 支柱 2：现货交易层（[`rfq-propamm/`](rfq-propamm/)）
- [`rfq-propamm/README.md`](rfq-propamm/README.md)：做市商私有库存报价层总览、零补贴做市、28 个 DEX 收敛战略。
- [`rfq-propamm/fluxion-propamm-spec.md`](rfq-propamm/fluxion-propamm-spec.md)：Fluxion RFQ 泛化架构规格书、EIP-712 签名协议、Bybit W2 库存接入规范与开闭市双模状态机。
- [`rfq-propamm/chain-infra/`](rfq-propamm/chain-infra/)：**【现货做市专属链优化】**
  - [`rfq-propamm/chain-infra/README.md`](rfq-propamm/chain-infra/README.md)：做市商专属链优化总览。
  - [`rfq-propamm/chain-infra/sequencing-and-cancel-priority.md`](rfq-propamm/chain-infra/sequencing-and-cancel-priority.md)：做市商撤单优先队列（Cancel Priority FIFO）与 Flashblocks 预确认。
  - [`rfq-propamm/chain-infra/mechanism-notes.md`](rfq-propamm/chain-infra/mechanism-notes.md)：做市机制与无公开 mempool 天然抗夹实测数据。

#### 支柱 3：衍生品层（[`perps/`](perps/)）
- [`perps/README.md`](perps/README.md)：股票 Perps 优先、Oracle-based 起步、现货基差套利与休市定价机制总览。
- [`perps/chain-infra/`](perps/chain-infra/)：**【衍生品专属链原生支持】**
  - [`perps/chain-infra/README.md`](perps/chain-infra/README.md)：衍生品专属链原生支持总览。
  - [`perps/chain-infra/native-chain-support.md`](perps/chain-infra/native-chain-support.md)：三大机制空白（清算专用通道、模拟免拒单、预言机原子绑定）。
  - [`perps/chain-infra/perps-infra-notes.md`](perps/chain-infra/perps-infra-notes.md)：衍生品链级参数设计底稿。
  - [`perps/chain-infra/rise-appspecific-teardown.md`](perps/chain-infra/rise-appspecific-teardown.md)：RISE 为 RiseX 做了什么的工程真相（1.5 Ggas 暴力解 CLOB 实测）。
  - [`perps/chain-infra/rise-chain-infra.md`](perps/chain-infra/rise-chain-infra.md) & [`risex-app-coupling.md`](perps/chain-infra/risex-app-coupling.md)：RISE 节点与应用层拆解。
  - [`perps/chain-infra/appchain-comparables.md`](perps/chain-infra/appchain-comparables.md)：Hyperliquid、dYdX、Sei 等横向架构对标。

#### 支柱 4：融资层（[`lending/`](lending/)）
- [`lending/README.md`](lending/README.md)：证券抵押借贷、融资融券业务闭环、激活 $576M 闲置稳定币总览。
- [`lending/chain-infra/`](lending/chain-infra/)：**【借贷风控专属链优化】**
  - [`lending/chain-infra/README.md`](lending/chain-infra/README.md)：市场日历预编译（Market Calendar Precompile）、周末动态折价（Weekend Haircut）与预言机休市陈旧度防护。

#### 支柱 5：接口与分发层（[`terminal/`](terminal/)）
- [`terminal/README.md`](terminal/README.md)：自建 Tape Terminal MVP、Bybit Alpha 白标双前端、打破执行层零覆盖。
- [`terminal/chain-infra/`](terminal/chain-infra/)：**【终端体验专属链优化】**
  - [`terminal/chain-infra/README.md`](terminal/chain-infra/README.md)：Mantle Passport（Passkey 社交/生物识别无助记词登录）、Session Key 免弹窗与 ERC-4337 Paymaster 全链免 Gas。

---

### 🏎️ 母目录 2：外部动力挂斗（[`sidecar/`](sidecar/)）
- [`sidecar/README.md`](sidecar/README.md)：**【Sidecar 协同总纲】** 承销商、做市商、经纪商三位一体架构。
- [`sidecar/strategy-and-robustness.md`](sidecar/strategy-and-robustness.md)：商务推进策略、稳健性降级预案（主线仅依赖 W1+W2）与对 Byreal 的分工口径。
- [`sidecar/w1-alpha-whitelabel.md`](sidecar/w1-alpha-whitelabel.md)：W1 经纪商工作流（Bybit Alpha 白标发行引擎接入，8000 万用户直通）。
- [`sidecar/w2-prop-inventory.md`](sidecar/w2-prop-inventory.md)：W2 做市商工作流（自营库存接入 Fluxion RFQ 与股票 Perps 种子流动性）。
- [`sidecar/w3-native-bstocks.md`](sidecar/w3-native-bstocks.md)：W3 发行方工作流（bStocks 路线原生自营股票发行，ADGM 招股书与 Backed 双轨制）。
- [`sidecar/w4-uta-margin.md`](sidecar/w4-uta-margin.md)：W4 主经纪商工作流（UTA 统一交易账户跨链保证金打通，CeDeFi 终极形态）。

---

## 6. 三轨推进路线图（Timeline）

```
阶段        轨道 A：市场结构 (链 + 现货)          轨道 B：发行与衍生品               轨道 C：Bybit Sidecar 挂斗
──────────────────────────────────────────────────────────────────────────────────────────────────────────
P0 核验     Fluxion RFQ 可编程接口确认            Backed 三项核验 (Hook/Multiplier/法务) 四条工作流书面意向; W3 预研
(0-6周)

P1 底座     Prop AMM 泛化 + 聚合器对接;            曲线合约 + Spend-Gate 审计;        W2 做市库存上线;
(4-16周)    预确认流 (Flashblocks); Terminal MVP                                    W1 接口打通

P2 点火     做市商撤单优先上线                   Tape Launchpad 上线 (Tier 1 标的);   W1 Alpha 白标上线
(12-26周)                                        股票 Perps Beta (Oracle-based)

P3 护城河   触发式热点分桶待命                   双创新 Hook (休市熔断 + 公司行动);   Alpha 专区晋级制度化
(24-38周)                                        xStocks 抵押借贷上线

P4 全栈     —                                    代币化 IPO 曲线; Fee Key 二级市场  W3 自营股票首批标的;
(36-52周)                                                                           W4 UTA 保证金打通
```

---

## 7. 北极星 KPI 纪律与反目标

### 🚫 坚决不用的虚假指标与反目标
- ❌ **坚决不用「日发币量」**：four.meme 发币量仅跌 14% 但日收入暴跌 99%，纯发币量是欺骗性虚荣指标；
- ❌ **坚决不用「补贴买来的虚假 TVL」**：Aave 在 Mantle 经历的 $137M → $704M → $62M 暴跌已彻底证伪雇佣兵资本；
- ❌ **不做通用 Meme 曲线**：通用 Meme 在 Mantle 经历 Funny Money / Printr 两次同一周关停，第三次尝试绝无借口；
- ❌ **不做通用 Crypto Perps**：正面硬刚 Hyperliquid 必败；
- ❌ **不为 Meme 单开独立 Appchain**：杜绝割裂与主网闲置稳定币及 xStocks 的流动性连接。

### 🎯 真实的北极星 KPI
1. **发行层**：`每周毕业代币数 × 毕业后 7 天存活率`（P2 验收：毕业 $\ge 3$/周，7天存活率 $\ge 30\%$）；
2. **现货层**：主流资产对 CEX 买卖点差（$\le 3-5\text{bps}$）与 7×24 报价在线率（$\ge 99.5\%$）；
3. **衍生层**：未平仓合约规模（OI）与周末/休市时段成交占比；
4. **融资层**：$576M 稳定币的链上真实借贷利用率；
5. **接口层**：Tape Terminal DAU 及 Bybit Alpha 白标端贡献成交占比（$\ge 50\%$）。
