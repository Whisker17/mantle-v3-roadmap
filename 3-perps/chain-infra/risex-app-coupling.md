# RISEx 应用层机制 + 链-app 耦合点取证

> 研究轨道：I ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

---

## 0. 本轨道的七条核心结论

| # | 结论 | 证据强度 |
|---|---|---|
| **I-1** | **「整条链为 RISEx 服务」在组织、经济与流量三个层面都是工程事实，但在「链级技术特权」层面几乎为零。** 官方原话：*"RISE and RISEx are one team, one vision, one token"*、*"RISE is an exchange chain and RISEx is the core product"*、**100% 的 RISE 积分分配给 RISEx 用户**。但链没有为 RISEx 提供任何 precompile、系统交易、专用 lane 或排序特权。 | [一手] |
| **I-2** | **RISEx 的「链原生体验」几乎全部由「中心化 operator + 官方钱包 relay」合成，而非协议特权。** 用户不发交易 —— 用户签 EIP-712 消息发给 REST API，operator 代为上链并**代付 gas**。 | [一手] |
| **I-3** | **RISEx 用「latency bump」在应用层实现了做市商公平性**：taker 被人为延迟 100–300ms，maker 10–200ms，**cancel 恒为 0ms**。这正是「cancel 优先于 fill」的排序语义 —— 但它是**应用层的人为延迟**，不是链级排序规则。 | [一手] |
| **I-4** | **RISEx 用「TX 额度门禁」替代 gas 做反垃圾**：新地址免费 10,000 笔额度，每 $5 累计成交量 +1 笔，**撤单永久免费**。这与第一阶段为 launchpad 抗狙击选定的「额度门禁」范式是同一个机制。 | [一手] |
| **I-5** | **「fully onchain / no off-chain components」的表述需要限定**：*撮合与结算*确实在 EVM 内；但*订单接入、排序进交易、gas 支付、限流*全部在链下由 operator 与 Cloudflare 完成。 | [一手] |
| **I-6** | **RISEx 至今仍是 gated mainnet（邀请码准入）**，且 V1.1 的核心差异化功能（AutoYield、RIP-1 子账户、RIP-2 组合保证金、现货、Builder Codes）**全部未上线**。 | [一手] |
| **I-7** | 【实测】RISEx 30 个市场、24h 成交额 **$73.56M**、总未平仓 **$57.28M**。**已上线股票/ETF/贵金属永续**（SPY、QQQ、MSTR、SNDK、XAU、XAG），这是"tomorrow means equities, forex, commodities"的已兑现部分。 | 【实测】 |

---

## 1. RISEx 产品拆解

### 1.1 定位与状态 [一手]

| 项 | 内容 |
|---|---|
| 定位 | *"a fully onchain orderbook DEX built on RISE Chain, supporting perpetual futures (spot markets coming soon)"* |
| 上线状态 | **"RISEx is live now on gated mainnet."** 需要**邀请码**：*"To access RISEx, you will need an access code from a user currently trading on the platform."* |
| 登录方式 | 既可连 EVM 钱包，也可**用 Google / 邮箱**创建账户（托管式助记词） |
| 准入限制 | 测试网竞赛条款排除**美国公民**，以及白俄罗斯、波黑、希腊、科索沃、拉脱维亚、马其顿、**意大利、葡萄牙**公民；禁 VPN 绕过 |
| 审计 | *"RISEx contracts have undergone **internal audits** and are in the process of third-party audits."* ⇒ **第三方审计未完成** |
| 口号 | "Programmable Markets" / "Unified Exchange" —— *"Leverage anything, trade everything"* |

来源：<https://docs.risechain.com/docs/risex/faq>、<https://docs.risechain.com/docs/risex/onboarding/access>、<https://docs.risechain.com/docs/risex/onboarding/account-setup>、<https://docs.risechain.com/docs/risex/misc/eligibility>

### 1.2 合约架构 [一手] + 【实测】

主网 14 个核心合约（chainId 4153）：

| 合约 | 地址 | 职责 | 【实测】链上活跃度 |
|---|---|---|---|
| **RISExUniversalRouter** | `0xaaDDE0CeA454F2bcB26F46ED54C5709B7Bb34a7E` | 交易所动作的统一入口 | **占全链交易 82.5%**；字节码 1,074 字节 ⇒ **EIP-1967 代理** |
| **FundingRate** | `0x069eDF2C2A3c93b54640Ae142B9f5375fe4A207a` | perps 资金费计算 | **占全链交易 10.5%**；同为 1,074 字节代理 |
| OperatorHub | `0xf665AbA90b6ac7515D50b12FCB4f350136726734` | operator / keeper 动作枢纽 | 0.8% |
| OrdersManager | `0xE03C1D5081eb2d0E6bFd62A949C5b12eFa44F2cD` | 链上订单与生命周期 | 0.3% |
| PerpsManager | `0x53f10fAcFC8965750494E6965F5d6dA39B41d852` | 永续持仓管理（亦为 markets_config） | — |
| SpotManager | `0x1F92be734731e28F52C20AB0BAA73Db7cBf521F8` | 现货市场（**功能未上线**） | — |
| CollateralManager | `0x2C03C7d7e2974C6599b6B108879109281ef3F818` | 抵押品与保证金账本 | — |
| AccountRegistry | `0x1238991Cac4E65902C08213e79909A9c813Eebc3` | **交易账户与授权签名人注册表** | — |
| RISExAuthorization | `0x0D919DAA3f12AE715744Eb648c00066c5DBd66f0` | 受保护动作的授权校验 | — |
| AccessManager | `0x1BEe39C01907E3018b7ec2021Cf73F70541b36cC` | 角色与权限管理 | — |
| TokenManager | `0x07DCE641354bBbc93C785f86971ad9f78f676Bd5` | 支持代币与上币 | — |
| FeeManager | `0x11541dc387b9C307043ea732127DF92b80bab52b` | 费率配置与记账 | — |
| **RISExOracle** | `0x8fC4D0Cf74cdF595254cB763d4C05D38Df0e9503` | 聚合价格预言机 | **同时是全链的"Internal Oracle"公共原语**（见 H 轨道 §7.2） |
| RISExStork | `0x76A559C716c5B93b9d743e08D9E9f23f96a4f975` | Stork 价格源适配器 | — |
| （非文档表内）Stork 推送目标 | `0xacc0A0cF13571d30B4b8637996F5D6D774d4fd62` | 【实测】链上高频推送 | **占全链交易 2.1%，平均 gasUsed 1,345,593** |
| USDC | `0xe436820bA0c69702c1D3E601D421c0ef38262739` | 基础抵押资产 | 来自 `api.rise.trade/v1/system/config` |

来源：<https://docs.risechain.com/docs/risex/contracts/deployments>、<https://docs.risechain.com/docs/builders/mainnet-contract-addresses>、【实测】`https://api.rise.trade/v1/system/config`

> ⚠️ **两个占了 93% 流量的合约都是 1,074 字节的最小代理**，实现合约地址与其验证状态本轨道未逐一核实（前一轮 agent 线索称 OrdersManager 实现合约疑似未验证，**未复现，不予采信**）。

### 1.3 撮合与订单路径（本节是本轨道最重要的发现）[一手]

**官方声称**：
> *"RISEx executes all orders, matching, and settlement within the EVM."*
> *"All trading logic executes onchain and is verifiable... **There are no off-chain components for matching or settlement.**"*
> —— <https://docs.risechain.com/docs/risex/faq>

**但实际的下单路径是**（据 API 文档一手拼合）：

```
用户 → ① 一次性 registerSigner（注册 API 签名人，7 天有效期，权限可选 All/Perps/Spot/MoveFund）
     → ② 对 VerifyWitness 做 EIP-712 签名（bitmap nonce：nonceAnchor uint48 epoch + nonceBitmap uint8 bit，
          每 epoch 最多 256 个并发有效 nonce）
     → ③ HTTP POST 到 REST API（developer.rise.trade），经 Cloudflare 边缘
     → ④ RISEx operator 把签名消息打包成链上交易提交（OperatorHub / UniversalRouter）
     → ⑤ EVM 内撮合 + 结算
     → ⑥ 用户经 shred WebSocket 收到结果
```

来源：<https://docs.risechain.com/docs/risex/api/important-notes>、<https://docs.risechain.com/docs/risex/api>、<https://docs.risechain.com/docs/risex/api/register-signer>、<https://docs.risechain.com/docs/risex/api/order-rate-limits>

> **精确的判定**：
> - ✅ **撮合（matching）在链上** —— 这一点是真的，且【实测】印证：单笔 router 调用烧 240 万–420 万 gas，只有真在 EVM 里跑订单簿才会这么贵。
> - ✅ **结算（settlement）在链上** —— 真。
> - ❌ **「没有链下组件」不成立** —— 订单接入、身份限流、交易打包、gas 支付**全在链下**，由 RISEx 自营 operator 完成，且前置一层 **Cloudflare**（限流 200 请求/10 秒）。
> - ⇒ 正确表述应为：**「链上撮合引擎 + 中心化订单网关」**。用户签的是消息，不是交易。

### 1.4 限流机制：TX 额度门禁 [一手，对本研究高度可迁移]

| 层 | 限额 | 范围 |
|---|---|---|
| **边缘（Cloudflare）** | **200 请求 / 10 秒** | 所有 HTTP 端点，超限在到达 RISEx 前返回 429 |
| **订单额度（按地址）** | **1 TX / 每笔订单** | 应用层，绑定地址的**终身**交易额度 |

额度经济学：
- **新地址免费获得 10,000 TX**
- **每 $5 终身成交量 → +1 TX，永不重置**
- **下单 / TP-SL 单各扣 1 TX；撤单免费**（官方原话：*"Cancels are intentionally free so traders are never discouraged from cleaning up resting orders."*）
- 额度耗尽后进入软惩罚：**1 请求 / 10 秒**，不封号
- 响应头暴露余量：`X-Address-Quota-Earned`、`X-Address-Quota-Remaining`

来源：<https://docs.risechain.com/docs/risex/api/order-rate-limits>

> **为什么这一条对「资产发行原生链」极其重要**：
> RISEx 因为**替用户付了 gas**，就失去了 gas 这个天然的反垃圾闸门，于是不得不重新发明一个 —— **按地址的终身额度 + 用真实经济活动（成交量）赚取额度**。
> 第一阶段研究独立得出的结论是：Mantle 2 秒出块使「衰减税」失效，**抗狙击必须用「额度门禁」**（Flaunch Game Mode 式 spend-gate）。
> ⇒ **两条独立路径收敛到同一个机制。** 这大幅提高了「额度门禁」作为 issuance-native 原语的可信度。→ 交给 report/08。

### 1.5 Latency Bumps —— 应用层的排序公平性 [一手，重大发现]

官方明确说明动机（原文）：
> *"When prices move, takers with faster infra race to pick off stale quotes before makers can cancel. Makers eat the loss, widen spreads, books thin out. **By delaying takers, makers are able to pull their orders before being adversely selected.** This leads to tighter spreads and better execution for retail and a healthier venue."*

**人为延迟表**：

| 账户档 | Taker 延迟 | Maker 延迟 | **Cancel 延迟** |
|---|---|---|---|
| **API Trader** | **100 ms**（*见下方公告已临时改为 200ms*） | 10 ms | **0 ms** |
| **Click Trader**（新账户默认） | **300 ms** | 200 ms | **0 ms** |

- maker 必须是 **post-only 限价单**；其他限价单类型都算 taker
- API Trader 需在 Discord 开工单申请；**会被定期审查，若流量被判定为 toxic 可撤销快速通道**
- **官方公告（2026-08-23）原文**：*"Due to **congestion**, to protect makers, the taker latency for API Traders has been temporarily increased to **200 ms**. Congestion improvements are in progress and this will be reverted once improvements are live."*

来源：<https://docs.risechain.com/docs/risex/trading/account-types>

> **三个层次的含义**：
> ① **这是「cancel 优先于 fill」的实现** —— 但不是靠链级排序语义，而是靠**在 operator 里给不同动作插入不同长度的 sleep**。撤单 0ms、taker 100–300ms，等价于给撤单绝对优先权。
> ② **它是纯中心化的裁量权** —— 谁是 API Trader、谁的流量"toxic"、延迟设多少，全由 RISEx 单方面决定并可随时撤销。这是 app-specific 模式的**权力集中**面。
> ③ **2026-08-23 的拥塞公告是一记警钟** —— 一条 1.5 Ggas/s、填充率仅 10% 的链，其旗舰应用仍会因"congestion"而不得不把 taker 延迟翻倍。⇒ **瓶颈不在区块空间，在 operator 的单点处理能力。**（→ 这一条对 K/N 轨道"提升吞吐无用"的判断是强力佐证。）

### 1.6 保证金、清算与保险基金 [一手]

**保证金**
- 基础抵押资产：**USDC**（唯一）。*"Portfolio Margin will unlock any ERC20 as collateral **once available**."* ⇒ 多抵押品未上线
- 支持全仓（Cross）与逐仓（Isolated），均为 V1
- 健康度：`crossHealthFactor = crossMarginBalance / totalCrossMM`，≤1 触发清算

**四段清算瀑布**（官方原文结构）

| 阶段 | 触发条件 | 行为 |
|---|---|---|
| **1. Pre-liquidation** | `crossIMR > accountEquity ≥ crossMMR` | 仅允许不增加维持保证金/不减少权益的动作；**自动撤销所有非 reduce-only 挂单**；用户只能下 reduce-only 单 |
| **2. Partial liquidation** | `accountEquity ≤ maintenanceMargin` | ① 撤销全部挂单 ② **以 zero price 的 IoC 单把仓位直接打到订单簿上**，逐个仓位清算，账户一恢复健康立即停止 |
| **3. Full liquidation** | `accountEquity ≤ closeOutMaintenanceRequirement`（= 2/3 × 维持保证金） | **XLP vault 接管仓位**，同样按 zero price 以保证权益/维持保证金比率不恶化 |
| **4.**（文档续） | — | ⚠️ 本轨道未读完第 4 段，标为未完整核实 |

- **清算费**：若清算成交价优于 zero price，RISEx 抽取**最高 1%**，**该费用进入 XLP vault 以逐步积累保险基金**
- 设计取向：**先做部分清算**（"partial liquidations first, giving you the best chance to stay in the market"）

来源：<https://docs.risechain.com/docs/risex/trading/liquidations>、<https://docs.risechain.com/docs/risex/faq>

> **注意这里没有 keeper 竞争**：清算由风控引擎发起、以 IoC 单打到自家订单簿，**不存在外部 keeper 抢清算的 MEV 竞赛**。这是「自营 operator + 链上订单簿」组合的一个真实红利 —— 它把清算 MEV 内部化掉了。→ 交给 M 轨道。

### 1.7 资金费机制 [一手]

- **每小时支付一次，按 8 小时率计算**：`F = clamp((P + effectiveInterest) / 8, −4%, +4%)`
- 利息项：多数市场固定 `r = 0.01% / 8h`（≈ 0.00125%/hr ≈ **11.6% APR**）
- 溢价指数 `P = impactPrice / oraclePrice`，**每 5 秒采样、按小时平均**
  - `impactPrice = max(impactBid − oraclePrice, 0) − max(oraclePrice − impactAsk, 0)`
  - `impactNotional = 50 USDC / initialMarginFraction`
- **用 index price（非 mark price）计算名义规模**
- 部分市场有 per-market **interest rate dampener** `d`（对标 Binance 做法）：`P` 落在 `[r−d, r+d]` 稳定区内时资金费钉死在 `r/8`
- 【实测】`funding_interval = 3,600,000,000,000 ns = 3600 s = 1 小时`，与文档一致

来源：<https://docs.risechain.com/docs/risex/trading/funding>、【实测】`api.rise.trade/v1/markets`

> **FundingRate 合约占全链 10.5% 的交易量**就来自这里：每小时对全部持仓账户逐个结算，是链上第二大 gas 消耗源（【实测】selector `0x1747e9a6` 平均 160,559 gas、`0x5b4b886a` 平均 46,286 gas）。

### 1.8 费率表 [一手]

按**滚动 14 天成交量**分档：

| Tier | 14 天量门槛 | Taker (bps) | Maker (bps) |
|---|---|---|---|
| 1 | $0 | 3.00 | 1.00 |
| 2 | $5,000,000 | 2.50 | 0.75 |
| 3 | $25,000,000 | 2.10 | 0.50 |
| 4 | $100,000,000 | 1.70 | 0.25 |
| 5 | $500,000,000 | 1.55 | **0.00** |
| 6 | $1,000,000,000 | 1.50 | **0.00** |

- **无 maker rebate 计划**（可申请费率试用档）

来源：<https://docs.risechain.com/docs/risex/trading/fees>

> **单位经济学（本研究计算）**：Tier-1 taker 3 bps ⇒ 一笔 $1,000 名义的成交收 **$0.30** 手续费；而【实测】该笔交易的链上 gas 成本约 **$0.004**。
> ⇒ **手续费收入是 gas 成本的约 75 倍。**
> **这就是「operator 帮所有人付 gas」在商业上成立的原因**，也是「全链上订单簿」能跑通的真正财务前提。它依赖两个条件同时成立：**(a) base fee 近零、(b) 交易有 bps 级手续费**。
> ⚠️ **对发行类应用的直接推论**：launchpad 的手续费率（pump.fun 级 1%）远高于 3bps，**gas 代付的经济空间更大**。→ 交给 report/08。

### 1.9 未上线的功能（必须与已上线严格区分）[一手]

| 功能 | 状态 | 说明 |
|---|---|---|
| 永续交易、全仓/逐仓、自成交防护、TP/SL、RLP Vault | **V1 已上线** | — |
| **AutoYield**（闲置保证金生息） | **V1.1，未上线** | — |
| **Modular Sub-accounts（RIP-1）** | **V1.1；RIP 状态 = Draft** | — |
| **Permissionless Portfolio Margin（RIP-2）** | **V1.1；文档标 "Coming soon"** | 依赖 **Morpho Lite** 的无许可借贷市场 |
| **Spot Markets** | **V1.1，未上线** | SpotManager 合约已部署 |
| **Builder Codes** | **V1.1，未上线** | — |

来源：<https://docs.risechain.com/docs/risex/features/roadmap>、<https://docs.risechain.com/docs/risex/features/portfolio-margin>、<https://docs.risechain.com/docs/risex/rips/rip-1>

**RIP-1 规格要点**（值得单独记，因为它是"可编程市场"叙事的技术载体）：
- 两个原语：**`BaseSubAccount`**（封装下单/授权/存取的标准交互，builder 继承并 override hooks）与 **`SubAccountFactory`**（EIP-1167 最小代理部署 + `user → subAccount[]` 链上注册表 + 确定性地址）
- 权限模型：**Owner**（全权，可提款、可改 manager）/ **Manager**（可交易，不可提款）
- 授权集成：**session key 用于 API 的 gasless 交易**、operator 委托执行、`registerSigner` 用于 keeper 自动化
- 官方类比：*"Analogous to what browser extensions enable for browsing... The browser became a platform. RIP-1 does the same for perps."*

---

## 2. Oracle 路径

| 维度 | 内容 |
|---|---|
| 供应商 | **Stork**（独立第三方，<https://docs.stork.network/>） |
| **推送频率** | **每 500ms 推送到交易所**（官方架构页原话：*"Oracle — **Pushed to the exchange every 500ms** to reflect the most up-to-date pricing"*） |
| 模式 | **push**（不是合约内 pull） |
| index price | 直接取自 Stork，是**多家 CEX 现货价的中位数** |
| mark price | **三个价格的中位数** `mark = median(P1, P2, P3)`；P1 = index-anchored premium（用 `impactNotional = 50 USDC / initialMarginFraction`）；P2/P3 见文档 |
| mark 用途 | 计算未实现盈亏 + 判定清算 |
| **谁付 gas** | **operator / 推送方付**。【实测】推送目标 `0xacc0A0cF…fd62` 占全链交易 2.1%，**平均 gasUsed 1,345,593** —— 每次价格推送烧 134 万 gas |
| 是否走系统交易 / 特权 | ❌ **不是**。它是普通 EOA 发的普通交易，只是发得很频繁 |
| 是否有链级 oracle 白名单 / 预留区块空间 | ❌ **未找到任何证据** |
| **链级暴露** | ✅ `RISExOracle` 被列为**全链的 "Internal Oracle"**，任何合约可调 `getIndexPrice(marketId)` / `getMarkPrice(marketId)` |

来源：<https://docs.risechain.com/docs/risex/about/Architecture>、<https://docs.risechain.com/docs/risex/trading/oracle-pricing>、<https://docs.risechain.com/docs/builders/mainnet-internal-oracles>

**与其他 app-chain 的一句话对照**：
- **Hyperliquid**：validator 集体喂价，oracle 是**共识层职责**，无 gas 成本。
- **dYdX v4**：`x/prices` 是**链的原生模块**，价格更新由 validator 在 block 内完成。
- **RISEx**：oracle 是**普通合约 + 普通 EOA 高频推送 + 自付 gas**。
⇒ **RISE 在 oracle 这一维度上，恰恰是三者中最"不原生"的。** 它靠的是 gas 便宜到可以每 500ms 烧 134 万 gas。（深入对照 → 交给 J 轨道）

---

## 3. 「链为 app 做的适配」逐项取证（本轨道核心）

判定三态：**✅ 有一手证据** / **❌ 有反证** / **⭕ 无证据**（检索无果，按不存在处理）

| # | 候选链级适配 | 判定 | 依据 |
|---|---|---|---|
| 1 | 下单/撮合吃到 **Shred 级 pre-confirmation** | **✅ 有一手证据（但为通用能力）** | RISEx 架构页：*"Orderbook updates states every 1ms"*、*"E2E order latency is ~3ms"*；FAQ：*"RISE Chain has sub-3ms execution. **Cancel orders, for example, are sub-1ms.**"*。**但 shred 是链的通用能力，非 RISEx 专属** |
| 2 | **gasless 下单/撤单** | **✅ 有证据，但是应用层实现** | RISE Wallet 文档：**"RISEx trading is fully sponsored"**；实现在 **RISE Relay**（按 tier/日限额/合约白名单/函数白名单校验），**不是协议级 paymaster** |
| 3 | **session key / 一次签名多次下单** | **✅ 有证据，应用层 + 钱包层** | `registerSigner` 注册 API 签名人（7 天有效、权限枚举 All/Perps/Spot/MoveFund）+ bitmap nonce（每 epoch 256 并发）；RISE Wallet 侧另有 Porto/EIP-7702 session key |
| 4 | **sequencer 级排序特权**（RISEx 交易优先） | **⭕ 无证据** | 全站检索无任何"priority lane / reserved blockspace / privileged tx"表述；【实测】无 `rise_*` RPC、无 txpool 可见性；RISE 侧排序规则本身也无一手文档 |
| 5 | **oracle 先于撮合的确定性顺序** | **⭕ 无证据** | oracle 是普通 EOA 交易（见 §2）。**排序保证只能来自 operator 自己把两笔交易按序提交**，非链级语义 |
| 6 | **cancel 优先于 fill** | **✅ 有证据，但是应用层「人为延迟」** | Latency Bumps：cancel 0ms vs taker 100–300ms（§1.5）。**链层无此语义** |
| 7 | **为撮合定制的 precompile** | **❌ 有反证** | 【实测】单笔撮合烧 240 万–420 万 gas ⇒ **纯 Solidity 实现**。若有撮合 precompile，gas 会低 1–2 个数量级 |
| 8 | **区块空间预留 / 专用 lane** | **⭕ 无证据** | 无相关文档；且 2026-08-23 的"congestion"公告说明拥塞时的应对是**在应用层加延迟**，而非动用预留空间 |
| 9 | **原生 sub-account / 权限委托** | **✅ 有证据，但是合约层且未上线** | RIP-1（**状态 = Draft**，V1.1）；AccountRegistry + RISExAuthorization 已部署 |
| 10 | **revert protection / 失败订单不收费** | **⭕ 无证据（链级）** | `eth_sendRawTransactionSync` 只是**更快返回** `status: 0x0`；未找到 revert protection。**但因 operator 代付 gas，失败成本实际由 operator 承担** —— 用户侧等效免费，机制上不是链级 |
| 11 | **链级风控**（pre-trade risk check 进状态转换） | **❌ 有反证** | 风控是 `CollateralManager` / clearinghouse **合约**内的逻辑，属应用层；官方架构页把 Clearinghouse 列为 RISEx 组件而非链组件 |
| 12 | **native USD / 统一保证金资产** | **✅ 有证据（链级）** | **USDR** 是链级原生稳定币（M0/T-bill 背书），官方定位为"base unit of account across RISE"；但**RISEx 当前的抵押资产是 USDC**，USDR 主要用于 AMM/货币市场/AutoYield vault ⇒ 二者尚未统一 |
| 13 | **快速提现 / fast withdrawal** | **⭕ 无证据** | 【一手】提现最终性 **259,200 区块 ≈ 3 天**（标准 OP Stack 挑战期）。存款侧文档称"3–5 分钟"，提现侧未见加速通道 |
| 14 | **链级 VRF**（RISEx 未用，但链提供） | **✅ 有证据（链级公共品）** | `/docs/builders/vrf`，测试网 Coordinator `0x9d57aB45…8909`；主网状态未确认 |
| 15 | **链级 Internal Oracle** | **✅ 有证据（链级公共品 = RISEx 的 oracle）** | 见 §2 末与 H 轨道 §7.2 |

### 3.1 汇总

| 判定 | 数量 | 条目 |
|---|---|---|
| ✅ **有一手证据**（含"是链级公共品"与"是应用层实现"） | **8** | 1, 2, 3, 6, 9, 12, 14, 15 |
| ❌ **有反证** | **2** | 7（无撮合 precompile）、11（风控在合约内） |
| ⭕ **无证据** | **5** | 4, 5, 8, 10, 13 |

> **进一步拆解那 8 个 ✅**：
> - **真正的链级改造**：仅 **#1（Shreds）** —— 且它是通用能力，任何 RISE 上的应用都能用
> - **链级公共品（合约/运营层，零协议改动）**：#12 USDR、#14 VRF、#15 Internal Oracle
> - **应用/钱包层实现**：#2 gas 赞助、#3 session key、#6 cancel 优先、#9 子账户
>
> ⇒ **「链为 RISEx 做了专属技术特权」这一命题，取证结果是：没有。**

---

## 4. 官方叙事取证

### 4.1 三段决定性原话 [一手]

**① 组织与代币的一体化**（FAQ "What's the relationship between RISEx and RISE?"）：
> *"**RISE and RISEx are one team, one vision, one token.** RISE is the infrastructure layer that finally supports onchain orderbooks, and RISEx the flagship exchange built on that infrastructure. **RISEx will bootstrap activity on RISE** and fuel a thriving DeFi ecosystem on RISE."*
> —— <https://docs.risechain.com/docs/risex/faq>

**② 代币激励 100% 倾斜给旗舰应用**（Points 文档 "RISE airdrop allocation"）：
> *"**100% of RISE points are allocated to RISEx users**, including traders, LPs, and builder-code integrators.
> Why? **RISE is an exchange chain and RISEx is the core product.** Other apps built on the chain benefit from RISEx activity because they either: 1. Leverage RISEx liquidity and features through builder codes, or 2. RISEx is a user of their product, such as lending or AMM backstop liquidity, benefiting from the distribution and activity that trading generates."*
> —— <https://docs.risechain.com/docs/risex/rewards/points>

**③ 把订单簿定位为链的一等原语**（Vision）：
> *"But one primitive has been missing from the stack: **the orderbook**... **RISE brings the orderbook into the EVM as a first-class primitive.** Any Solidity contract can read the book, place orders, and settle in the same transaction. The deep liquidity on RISEx is not a walled garden. It's a shared infrastructure that any application on RISE can build on. **The same composable environment that made DeFi possible now has the performance to support real markets. That's what RISE was built for.**"*
> —— <https://docs.risechain.com/docs/risex/about/vision>

**④ 团队自述为何能同时做链和应用**：
> *"The RISE/x team has extensive experience in trading and in building high-performing, low-latency systems, **which is why the team was able to develop both the execution layer (RISE L2) and the onchain application (RISEx)**."*
> —— <https://docs.risechain.com/docs/risex/about/overview>

**⑤ 对 Hyperliquid 的定位差异化**：
> *"Hyperliquid showed the world what's possible with onchain orderbooks, but it's missing one key structural feature - **synchronous composability**. Hyperliquid runs on a custom L1 with a separate execution environment. RISEx runs on RISE, an EVM-compatible chain, which means it's synchronously composable with all other DeFi protocols."*
> —— <https://docs.risechain.com/docs/risex/faq>

> **判定**：**RISE 官方是明确、反复、书面地自我定位为「为 RISEx 服务的交易专用链」的。** 这不是外界解读，是官方文档原话。**"one team, one vision, one token"** 与 **"RISE is an exchange chain and RISEx is the core product"** 是本研究能找到的最强表述。

### 4.2 战略转向的证据

| 时间点 | 定位 | 来源性质 |
|---|---|---|
| **2025-06** | ⚠️ 前一轮 agent 报告 wayback 快照显示当时 docs 仍是通用 **"Gigagas L2"** 定位 | **⚠️ 本轨道未独立复现，仅记录为线索** |
| **2026-09-07（今）** | docs 站点标题 = **"RISEx — The unified exchange"**；首页大标题 = **"The trading chain"**；链文档（`/docs/rise-evm`）被收编在应用文档站点之下 | 【实测】 |

- **`/docs` 首页的措辞已经完全交易化**：*"Built for **CEX-grade performance**... enables builders, traders, and **institutions** to create and connect to **global orderbooks**"*。
- **点位证据**：`/docs/builders/mainnet-contract-addresses`（面向全体开发者的"链合约地址页"）里，**RISEx 的 14 个交易所合约与 OP Stack 系统合约、predeploy 并列**在同一页。→ 交易所合约在文档架构上已被当作链的基础设施。

> ⚠️ **存疑**：docs 站点改版的**确切时间点**本轨道未完成 wayback 考证（见存疑清单）。「从通用高性能 L2 转向交易专用链」的**方向**有充分现证，**时间线**待补。

### 4.3 代币与积分 [一手]

| 项 | 内容 |
|---|---|
| 积分项目 | **Season 1: Ignite**，**2026-07-20 00:00 UTC** 开始计分 |
| 首次分配 | **2026-07-28 14:00 UTC**，此后每周 |
| 每周额度 | **200k 点** |
| 预计结束 | **不晚于 2027 Q2** |
| 计分维度 | 订单活跃度（用户分层）、**总成本（手续费 + 滑点 + 负 markout）**、maker/taker 成交量、**未平仓与持仓时长**、推荐（额外 10%） |
| 方法论 | **不公开权重**（官方理由：防止被 game） |
| Season 0 | **Genesis Traders** 追溯积分，分配于 **2026-07-23 14:00 UTC** |
| 空投归属 | **100% 给 RISEx 用户**（见 §4.1②） |

来源：<https://docs.risechain.com/docs/risex/rewards/points>

> **注意「总成本」被计入积分**（手续费、滑点、负 markout 都算贡献）—— 这是**激励驱动交易量**的典型设计。评估 RISEx 数据时必须扣除这一层。

---

## 5. 数据表现【实测】

**取数：`https://api.rise.trade/v1/markets`，2026-09-07**

| 指标 | 实测值 |
|---|---|
| 市场数 | **30** |
| **24h 总成交额（quote volume）** | **$73,556,104** |
| **总未平仓（OI，按 mark price 折算）** | **$57,279,216** |
| 资金费间隔 | 3,600 s（1 小时） |
| 锁定市场数（`unlocked=false`） | **0**（全部开放） |

**成交额 Top 15**

| 市场 | 最大杠杆 | 24h 成交额 | 未平仓 |
|---|---|---|---|
| BTC/USDC | 25x | $24,386,470 | $12,956,456 |
| ETH/USDC | 25x | $8,097,462 | $6,821,471 |
| **XAU/USDC**（黄金） | 20x | $6,766,372 | $3,757,583 |
| HYPE/USDC | 20x | $4,976,766 | $3,033,041 |
| SOL/USDC | 20x | $3,640,514 | $1,685,124 |
| **XAG/USDC**（白银） | 20x | $2,716,970 | $1,860,002 |
| BNB/USDC | 20x | $2,644,818 | $1,341,415 |
| ZEC/USDC | 10x | $2,130,139 | $818,564 |
| **SPY/USDC**（标普 ETF） | 20x | $2,086,278 | **$6,936,965** |
| LIT/USDC | 5x | $1,628,379 | $1,392,339 |
| **MSTR/USDC**（股票） | 10x | $1,605,346 | $130,802 |
| PUMP/USDC | 10x | $1,540,878 | $900,528 |
| **SNDK/USDC**（股票） | 20x | $1,245,489 | $261,185 |
| DRAM/USDC | 20x | $1,215,519 | $334,782 |
| **QQQ/USDC**（纳指 ETF） | 20x | $1,214,900 | **$4,910,475** |

> **两个高价值观察**：
> ① **RWA 永续已经上线并有真实流量**：XAU、XAG、SPY、QQQ、MSTR、SNDK 合计 24h 成交 **$15.6M（占 21.3%）**，且 **SPY 与 QQQ 的未平仓（$6.94M / $4.91M）显著高于其成交额** —— 说明是**持仓型/对冲型资金**，不是刷量。这与第一阶段"meme × 代币化股票"的赛道判断在需求侧互相印证：**用户确实要在链上交易股票风险敞口**。
> ② 与 Hyperliquid 的量级差距：RISEx 24h $73.6M。⚠️ 本轨道未取到 Hyperliquid / Lighter / dYdX / Aster / edgeX 的同日数据用于严格排名（→ 交给 J 轨道），但按公开量级，Hyperliquid 日成交在**十亿美元级**，**RISEx 约为其 1%–数 % 量级**。**RISEx 仍处在 gated mainnet 的早期规模。**

### 5.1 激励驱动的客观标注

- Ignite 积分**每周 200k 点**、把**手续费与滑点计入贡献**、**100% 空投给 RISEx 用户**
- gated mainnet（邀请码）意味着用户基数被人为限制
- 【实测】12 秒窗口内全链仅 **37 个不同发送地址**、最活跃地址 12 秒 268 笔 ⇒ **链上流量高度集中于少数做市/operator 地址**
- ⇒ **$73.6M 的 24h 成交额应视为「激励期 + 邀请制 + 少数专业做市商」条件下的数字**，不能直接外推到开放状态。（不作道德判断，仅作口径说明。）

---

## 6. 可迁移性判断：优势归因三分类

| RISEx 的优势 | (a) 只能靠链级改造 | (b) 应用层即可 | (c) 靠中心化 operator 特权 | 依据 |
|---|---|---|---|---|
| **~3ms 端到端下单延迟** | ✅ **是**（Shreds + CBP + `eth_sendRawTransactionSync`） | — | — | 需自研执行层 |
| **撤单 sub-1ms** | ✅ 部分（shred） | — | ✅ 部分（latency bump 设 0ms） | 混合 |
| **全链上 CLOB 在经济上可行** | ✅ **是**（1.5 Ggas 区块 + 近零 base fee） | — | — | 【实测】单笔 240–420 万 gas |
| **下单/撤单对用户免 gas** | — | ✅（relay/paymaster 模式通用） | ✅（operator 代付） | RISE Wallet relay |
| **一次签名多次下单（session key）** | — | ✅（EIP-7702 / 4337 / 自建 signer 注册表皆可） | — | `registerSigner` |
| **cancel 优先于 fill 的公平性** | — | — | ✅ **纯运营裁量**（人为 latency bump，可随时改/撤销） | account-types 文档 |
| **反垃圾（TX 额度门禁）** | — | ✅ | ✅（额度由 RISEx 单方发放） | order-rate-limits |
| **清算无 keeper MEV 竞赛** | — | — | ✅（自营风控引擎发 IoC 单） | liquidations 文档 |
| **同步可组合性**（与 Morpho/AMM 同交易内组合） | ✅ **是**（必须同处一个 EVM 状态机） | — | — | 这是相对 Hyperliquid 的真差异 |
| **统一保证金 / 任意 ERC20 抵押** | — | ✅（RIP-2 + Morpho，纯合约） | — | **未上线** |
| **链级 Internal Oracle / VRF** | — | ✅（合约 + 链方运营） | ✅（谁能进"公共原语"名单由链方定） | 见 §2 |
| **USDR float 收益回投流动性** | — | — | ✅ **商业/法务安排** | USDR 文档 |
| **RWA 永续上币（SPY/QQQ/XAU…）** | — | ✅（有 oracle 即可） | ✅（上币权在 RISEx） | 【实测】markets |

### 6.1 归纳

| 归因 | 项数 | 含义 |
|---|---|---|
| **(a) 必须链级改造** | **4** | 低延迟确认、大 gas 预算、近零 base fee、同一 EVM 内的同步可组合性 |
| **(b) 应用层即可** | **6** | gas 代付、session key、额度门禁、统一保证金、公共原语、RWA 上币 |
| **(c) 中心化运营特权** | **7** | latency bump、清算内部化、额度发放、原语准入、USDR 收益、上币权、gated 准入 |

> **给委托方的核心判断**：
> **1. RISEx 的"体感优势"里，只有 4 项真的需要动链。** 其中 2 项（大 gas 预算、近零 base fee）**是纯参数**，Mantle 改配置即可；1 项（同步可组合性）**Mantle 天然就有**（它是通用 EVM 链）；**只有"毫秒级逐笔确认"需要真正的执行层硬工程。**
> **2. app-specific 模式的真正杠杆不在技术，在"同一个团队同时握有链、旗舰应用、钱包、oracle、稳定币、积分/代币"这六个控制点。** 技术只是其中最小的一块。
> **3. 但第 (c) 类的 7 项全部建立在中心化之上**，且 RISE 官方自己承认在走 progressive decentralization。**一旦 based sequencing 落地（Phase 2/3），latency bump、清算内部化、operator 代付这些机制都要重新设计。** 这是 app-specific 模式最大的未偿债务。

---

## 7. 显式回答：「整条链为 RISEx 服务」有多少是工程事实、多少是营销叙事？

| 层面 | 判定 | 证据 |
|---|---|---|
| **组织层** | **100% 工程事实** | *"one team, one vision, one token"*；同一团队做 L2 + 应用 |
| **经济/代币层** | **100% 工程事实** | **100% 的 RISE 积分给 RISEx 用户**；USDR 储备收益回投生态流动性；gas 币是 ETH ⇒ 链不靠 gas 挣钱，靠交易所与稳定币挣钱 |
| **流量层** | **100% 可量化事实** | 【实测】**93.5% 的链上交易直达 RISEx 合约**，加 oracle 推送达 **96.2%** |
| **文档/产品层** | **事实** | 链文档被收编进 "RISEx — The unified exchange" 站点；RISEx 合约与 OP predeploy 并列在"链合约地址"页；全链 Internal Oracle 就是 RISExOracle |
| **链级技术特权层** | **⭕ 基本是叙事** | **无 precompile、无系统交易、无专用 lane、无排序特权、无协议级 paymaster**。所有"特权感"来自**通用能力（Shreds）+ 参数（1.5 Ggas / 近零 base fee）+ 中心化 operator 运营** |
| **去中心化承诺层** | **⚠️ 尚未兑现** | based sequencing 三阶段全在路线图；PEVM 未上线；dispute game 为 Permissioned；DA 在 EigenDA 而非以太坊 |

**一句话结论**：
> **RISE 是一条「治理与经济上完全 app-specific、但技术上仍是通用 OP Stack 链」的 L2。**
> 它证明的不是"要为一个应用改 EVM"，而是**"要为一个应用把链的参数、公共原语、代币激励、钱包与稳定币全部对齐"**。
> 对 Mantle 的直接含义：**这条路的技术门槛远低于预期，组织与经济门槛远高于预期。**

---

## 存疑清单

1. ⚠️ **docs 站点改版的确切时间点未考证完成**。前一轮 agent 线索称 2025-06 wayback 快照仍为通用 "Gigagas L2" 定位，**本轨道未独立复现**，故"战略转向"只有方向证据、无精确时间线。
2. ⚠️ **RISEx 主网上线日期未找到官方公告**。只知积分 Season 1 自 2026-07-20 起、Season 0 追溯分配于 2026-07-23，且至今仍为 gated mainnet。
3. ⚠️ **`0xacc0A0cF13571d30B4b8637996F5D6D774d4fd62` 的确切身份未由官方文档确认**。本轨道依据「高频推送 + 与 `api.rise.trade/v1/system/config` 中 `stork` 字段前缀一致」判定为 Stork 推送目标，**属推断**。
4. ⚠️ **两个占 93% 流量的代理合约（router / FundingRate）的实现合约地址与验证状态未逐一核实**。前一轮线索称 OrdersManager 实现合约疑似未验证，未采信。
5. ⚠️ **清算瀑布第 4 阶段未读完**（文档 §Stage 3 之后的内容截断），ADL 是否存在、保险基金耗尽后的处理未确认。
6. ⚠️ **XLP vault 与 RLP Vault 的关系未厘清**：清算费进入"XLP vault"，而 roadmap 把"RLP Vault"列为 V1 已上线，文档另有 `/docs/risex/xlpvault/*` 章节。**两者是否同一物、命名是否变更中，未确认。**
7. ⚠️ **同业对标数据未取**：Hyperliquid / Lighter / dYdX / Aster / edgeX 的同日成交额与排名未取得（→ J 轨道）。DefiLlama 上 RISEx 的收录状态亦未确认。
8. ⚠️ **`api.rise.trade` 是否为官方主网 API 未由文档明确背书**：docs 给出的 base URL 是 `developer.rise.trade`；`api.rise.trade` 由本研究实测可用且返回的合约地址与官方 deployments 页**完全一致**，故判定为官方，但属推断。
9. ⚠️ **Latency bump 的实现位置未确认**：是在 REST API/operator 层还是别处？官方只说"artificial delay injected after an order or cancel is processed"。本轨道判定为应用层（因链上无此机制），属推断。
10. ⚠️ **"congestion"的确切成因未知**（2026-08-23 公告）。在填充率仅 10%、容量 1.5 Ggas/s 的链上出现拥塞，本轨道推断瓶颈在 operator 单点吞吐，**无一手证据**。
11. ⚠️ **RISEx 是否有原生代币、与 RISE 代币的关系**：官方只说"one token"，且积分 100% 给 RISEx 用户。RISE 代币本身的发行状态、`GovernanceToken` predeploy（`0x4200...0042`）是否已启用，均未确认。
12. ⚠️ **mark price 的 P2、P3 具体算法未读全**（oracle-pricing 文档在 P1 之后被截断）。

---

## 关键来源清单

**RISEx 官方文档（一手）**
- 介绍：<https://docs.risechain.com/docs/risex> ｜ 概览：<https://docs.risechain.com/docs/risex/about/overview> ｜ 愿景：<https://docs.risechain.com/docs/risex/about/vision> ｜ 技术架构：<https://docs.risechain.com/docs/risex/about/Architecture>
- **FAQ（含"one team, one vision, one token"）**：<https://docs.risechain.com/docs/risex/faq>
- 部署地址：<https://docs.risechain.com/docs/risex/contracts/deployments>
- 费率：<https://docs.risechain.com/docs/risex/trading/fees> ｜ 清算：<https://docs.risechain.com/docs/risex/trading/liquidations> ｜ 资金费：<https://docs.risechain.com/docs/risex/trading/funding> ｜ 预言机定价：<https://docs.risechain.com/docs/risex/trading/oracle-pricing> ｜ 市场规格：<https://docs.risechain.com/docs/risex/trading/markets>
- **账户档与 Latency Bumps**：<https://docs.risechain.com/docs/risex/trading/account-types>
- **订单额度限流**：<https://docs.risechain.com/docs/risex/api/order-rate-limits>
- API 要点（EIP-712 / bitmap nonce / registerSigner）：<https://docs.risechain.com/docs/risex/api/important-notes> ｜ <https://docs.risechain.com/docs/risex/api> ｜ <https://docs.risechain.com/docs/risex/api/register-signer>
- **RIP-1 模块化子账户（Draft）**：<https://docs.risechain.com/docs/risex/rips/rip-1> ｜ RIP-2 PPM：<https://docs.risechain.com/docs/risex/rips/rip-2>
- 组合保证金（Coming soon）：<https://docs.risechain.com/docs/risex/features/portfolio-margin> ｜ 路线图：<https://docs.risechain.com/docs/risex/features/roadmap>
- **积分 / 空投分配**：<https://docs.risechain.com/docs/risex/rewards/points>
- 准入 / 账户设置 / 存款 / 资格：<https://docs.risechain.com/docs/risex/onboarding/access> ｜ <https://docs.risechain.com/docs/risex/onboarding/account-setup> ｜ <https://docs.risechain.com/docs/risex/onboarding/deposits> ｜ <https://docs.risechain.com/docs/risex/misc/eligibility>
- 链侧 Internal Oracle：<https://docs.risechain.com/docs/builders/mainnet-internal-oracles> ｜ 主网合约全表：<https://docs.risechain.com/docs/builders/mainnet-contract-addresses>
- RISE Wallet（gas 赞助 / session key）：<https://docs.risechain.com/docs/rise-wallet/how-it-works>

**外部**
- Stork 文档：<https://docs.stork.network/>
- Morpho（RIP-2 依赖）：RIP-2 文档内引用
- EIP-1167 最小代理（RIP-1 使用）：<https://eips.ethereum.org/EIPS/eip-1167>

---

## 可复现方法附录

### A. RISEx 官方 API 实测

```bash
# 系统配置（链参数 + 全部合约地址 + USDC 地址）
curl -s https://api.rise.trade/v1/system/config

# 全市场行情（30 个市场：成交额、OI、mark/index price、杠杆、维持保证金因子、资金费间隔）
curl -s https://api.rise.trade/v1/markets
```

聚合口径：
- 24h 总成交额 = Σ `markets[].quote_volume_24h`
- 总未平仓（USD）= Σ (`markets[].open_interest` × `markets[].mark_price`)

### B. 链上流量归属实测

见 H 轨道「可复现方法附录 §B」。方法：`eth_getBlockByNumber(…, true)` 取 12 个连续区块的全部交易，按 `to` 分组计数，再用官方 deployments 表把地址映射回合约名。

### C. 文档语料本地留存

```bash
curl -sL https://docs.risechain.com/llms-full.txt -o rise-llms-full.txt   # 21,195 行
```
RISEx 章节位于该文件第 1784–1878、13569–16182、19481–20546 行区间（按 `^# .*(/docs/risex` grep 定位）。
