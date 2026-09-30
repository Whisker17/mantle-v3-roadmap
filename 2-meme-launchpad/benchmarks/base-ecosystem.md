# 专题 C：Base 生态 meme / creator launchpad 深度拆解

> 数据截止：**2026-09-06**。所有数字均标注时点与来源；一手数据（我直接调用 DefiLlama API / clanker.world API / Zora API 拉取）标记为 `[一手]`；无法核实的标记为 **待核实**。
>
> 本文服务于 Mantle 生态 launchpad 设计建议，因此**机制精度优先于叙事**：能落到合约名、函数名、常量、bps 的地方一律落到位。

---

## 0. 执行摘要（先看这一页）

### 0.1 三条主线

| | Clanker | Zora | Flaunch |
|---|---|---|---|
| 定位 | Farcaster 原生「@ 一下就发币」 | 「每个 post 即币」的创作者经济 | Uniswap v4 hook 金融工程实验室 |
| 上线 | 2024-11-08（v0） | Coins 协议 2025 年初，Creator Coins 2025-06-20 | 2025-01（Base） |
| AMM | v0–v3.1 用 Uniswap **v3**；v4.0/v4.1 用 Uniswap **v4 hook** | Uniswap **v4**（`ZoraV4CoinHook`），Doppler 提供初始多曲线流动性 | 全程 Uniswap **v4**（`PositionManager` 即 hook） |
| 有无 bonding curve | **无**。100B 全供应量一次性投成单边 LP | **无**。1B 供应量直接进 v4 池（多曲线） | **无**。固定价格 Fair Launch → 全区间 AMM 两段式 |
| 核心创新 | 单边流动性 + 永久锁 LP + 模块化 MEV 模块 | 内容币↔创作者币↔ZORA 的**货币层级**；99%→1%/10s 狙击税 | **Progressive Bid Wall**（用手续费在现价下方挂单）+ 收益权 NFT 化 |
| 累计手续费 `[一手]` | **$90.82M** | **$10.43M**（Coins，Base） | **$3.59M** |
| 协议净收入 `[一手]` | **$15.15M** | $7.70M（Base） | $3.07M（主要来自 flETH 的 Aave 收益，非抽成） |
| 累计发币数 | **737,210**（Base 711,358 = 96.5%）`[一手]` | 约 160 万枚（2025-07 口径，**待核实**） | 151,995（官方 Dune） |
| 近 30 天手续费 `[一手]` | $283,465 | $15,067 | $389.64 |
| 峰值 vs 现在 | 2026-02 月费 $22.39M → 2026-08 $0.24M（**−98.9%**） | 2025-08 月费 $2.51M → 2026-08 $0.015M（**−99.4%**） | 2025-01 月费 $1.12M → 2026-08 $283（**−99.97%**） |
| 平台币 | CLANKER（`tokenbot`）ATH $142.84 (2025-10-26) | ZORA −94.1% 距 ATH，mcap $38.4M | FLAY −98.3% 距 ATH，mcap $2.64M |

### 0.2 六条核心判断

1. **「社交图谱做冷启动」在 Base 上被验证有效，但只在冷启动那一段有效。**
   Base App 于 2025-07-16 上线当天把 Zora 接进社交流，Zora 日发币量从 <5,000 跳到 **单日约 54,000（2025-07-27）**，日活交易钱包从 <5,000 到 >50,000，月协议收入从约 $600K 跳到约 $2M。这是全行业最干净的一次「分发即冷启动」实验。但一年后（2026-09-05→06）Zora 的 24h 新建币只有 **422 枚** `[一手]`，相比峰值 **−99.2%**；月成交额从 2025-08 的 $113M 掉到 2026-08 的 **$0.5M**（−99.6%）`[一手]`。

2. **Base 官方已在 2026 年公开否定这条路线——这是本专题最重要的新事实。**
   2026-07-15/16，Jesse Pollak 交出 Base App 给 Cobie（Jordan Fish），并写道：*"the entire social side of the market that many of us had been building towards - farcaster, zora, miniapps, and yes, creator coins - disintegrated completely… **i was definitively wrong**."* 又说 *"the collateral damage was pretty bad… and this year has been an exercise in eating shit."* 2026 年三大优先级改为 **"winning trading, payments, and agents"**。Brian Armstrong 同期表态 content coins **"didn't work"**，Coinbase「今年年初就转向了」。Base App 的 Creator Rewards 项目已于 **2026-02-18 关停**，7 个月共发出约 **$450,000 给约 17,000 名创作者**（平均每人约 $26）。
   来源：<https://thedefiant.io/news/people/base-creator-jesse-pollak-hands-app-to-cobie-says-social-bet-was-definitively-wrong>（2026-07-16，直接引用 <https://x.com/jessepollak/status/2077427261586997745>）

3. **Base 上真正跑出第二波、且规模超过第一波的，不是人类创作者经济，而是 AI agent 社交图谱。**
   Clanker 月手续费：2024-12 $13.76M（第一波）→ **2026-02 $22.39M（第二波，历史最高）**；2026-02 单月部署 **261,047 枚**（日均 9,323 枚）`[一手]`。诱因是 **Moltbook**（只允许 AI agent 互动的社交网络）+ Clawstr 带来的 agent↔memecoin 反馈回路。2026-02 单月 Clanker 手续费占**整个 Base 链应用层手续费的 19.7%**（$22.39M / $113.40M）`[一手]`。
   → 结论要改写为：**Base 的 launchpad 冷启动来源是「任何高频、低摩擦、带身份的社交图谱」，人类的（Farcaster/Base App）和机器的（Moltbook）都算，机器那一波甚至更猛。**

4. **单币经济质量塌陷比总量塌陷更能说明问题。** Clanker「每枚币产生的手续费」：2024-12 约 **$1,371/枚** → 2026-02 约 **$86/枚** → 2026-08 约 **$53/枚** `[一手]`。发币量能靠机器人 / agent 刷回来，单币价值捕获回不来。这是所有「无门槛发射」模式的共同宿命。

5. **Base 全链 DEX 量只有 Solana 的 1/3，应用层手续费只有 1/7，但 TVL 已基本追平。**
   2026-09-06 `[一手]`：30 天 DEX 量 Base **$24.28B** vs Solana **$65.03B**（37%）；30 天应用层手续费 Base **$51.50M** vs Solana **$339.63M**（15%）；TVL Base $5.669B vs Solana $5.925B（96%）。→ Base 是「资金停放地」，Solana 是「换手场所」。meme launchpad 天然长在换手场所上。

6. **Uniswap v4 hook 是 Base 系 launchpad 唯一真正的技术护城河，而且它是可移植的。**
   Clanker / Zora / Flaunch / Doppler 四家的差异化 100% 建立在 hook 之上：动态费（狙击税）、LP 硬锁（`beforeAddLiquidity` revert）、单边流动性、自定义曲线（`beforeSwapReturnDelta`）、手续费自动路由。任何有 Uniswap v4 的 EVM 链（含 Mantle）都能复制这套设计——**说明 Base 的技术优势不可持续，其真实优势始终是 Coinbase 分发**。而分发这件事，Base 自己已经承认没转化成留存。

---

## 1. Clanker：v0 → v4.1 完整拆解

**官方文档**：<https://clanker.gitbook.io/documentation>（llms.txt 索引：<https://clanker.gitbook.io/documentation/llms.txt>）
**代码**：<https://github.com/clanker-devco>（`v4-contracts`、`v4-pool-extensions`、`clanker-sdk`、`v3.1-contracts`）
**合约地址全表**：<https://clanker.gitbook.io/documentation/references/deployed-contracts.md>

### 1.1 版本演进表

不变的内核（v0 一直到 v4.1）：**部署固定供应 ERC-20（100,000,000,000 枚 / 18 位小数，不可增发，可 `burn()`）→ 把（扣除 vault/airdrop/devbuy 之后的）全部剩余供应作为「单边流动性」投入 AMM 池 → LP 仓位永久锁死 → 交易手续费按 bps 永久分给创作者**。分发入口是 Farcaster 上 @clanker 或网站 clanker.world。

| 版本 | 状态 | AMM | Base 主网关键合约 | 关键变化 |
|---|---|---|---|---|
| **v0.0.0** | 最早 | Uniswap v3 | `SocialDexDeployer` `0x250c9FB2b411B48273f69879007803790A6AeA47`；`LockerFactory` `0x515d45F06EdD179565aa2796388417ED65E88939` | Farcaster「tag the bot」发射器。2024-11-08 上线（**待核实**：创始人 Jack Dishman / @proxystudio.eth 仅来自二手聚合） |
| **v1.0.0** | 废弃 | Uniswap v3 | `Clanker` `0x9B84fcE5Dcd9a38d2D01d5D72373F6b6b067c3e1`；`LockerFactory` `0x18db5Fce22bE8814B7E31FBDA2f6488d607A1172` | — |
| **v2.0.0** | 废弃 | Uniswap v3 | `Clanker` `0x732560fa1d1A76350b1A500155BA978031B53833`；`LPLockerv2` `0x618A9840691334eE8d24445a4AdA4284Bf42417D` | **Quantstamp 审计，2025-01** |
| **v3.0.0** | 废弃 | Uniswap v3 | `Clanker` `0x375C15db32D28cEcdcAB5C03Ab889bf15cbD2c5E`；`ClankerPreSale` `0x71cDc0bDF30F5601fb0ac80Cf1d20B771342C035`；`LPLockerv2` `0x5eC4f99F342038c67a312a166Ff56e6D70383D86` | 首次引入 presale 合约 |
| **v3.1.0** | legacy（仍可领费） | Uniswap v3 | `Clanker` `0x2A787b2362021cC3eEa3C24C4748a6cD5B687382`；`LpLockerv2` `0x33e2Eda238edcF470309b8c6D228986A1204c8f9`；`ClankerVault` `0x42A95190B4088C88Dd904d930c79deC1158bF09D` | 5 个核心合约（ClankerToken / Clanker / ClankerDeployer / ClankerVault / LpLockerv2）。**最多 30% 供应进创作者 vault（最短锁 30 天）**；创作者 dev buy（前端限 1 ETH，合约层不限）。默认起始市值 **10 WETH**。**0xMacro 审计，2025-03** |
| **v4.0.0** | 被 v4.1 取代 | **Uniswap v4 hooks** | `Clanker` `0xE85A59c628F7d27878ACeB4bf3b35733630083a9`；`ClankerFeeLocker` `0xF3622742b1E446D92e45E22923Ef11C2fcD55D68`；`ClankerLpLockerFeeConversion` `0x63D2DfEA64b3433F4071A98665bcD7Ca14d93496`；`ClankerVault` `0x8E845EAd15737bF71904A30BdDD3aEE76d6ADF6C`；`ClankerAirdrop` `0x56Fa0Da89eD94822e46734e736d34Cab72dF344F`；`ClankerUniv4EthDevBuy` `0x1331f0788F9c08C8F38D52c7a1152250A9dE00be`；`ClankerMevBlockDelay` `0xE143f9872A33c955F23cF442BB4B1EFB3A7402A2`；`ClankerSniperAuctionV0` `0xFdc013ce003980889cFfd66b0c8329545ae1d1E8`；`ClankerHookDynamicFee` `0x34a45c6B61876d739400Bd71228CbcbD4F53E8cC`；`ClankerHookStaticFee` `0xDd5EeaFf7BD481AD55Db083062b13a3cdf0A68CC` | **2025 年 6 月中上线**（至 2025-08 已部署 7,819 枚）。彻底重构为「工厂 + 4 类可插拔模块」。**Cantina + Macro 审计，2025-06~07** |
| **v4.1.0** | **当前** | Uniswap v4 | `ClankerHookDynamicFeeV2` `0xd60D6B218116cFd801E28F78d011a203D2b068Cc`；`ClankerHookStaticFeeV2` `0xb429d62f8f3bFFb98CdB9569533eA23bF0Ba28CC`；`ClankerSniperAuctionV2` `0xebB25BB797D82CB78E1bc70406b13233c0854413`；`ClankerAirdropV2` `0xf652B3610D75D81871bf96DB50825d9af28391E0`；`ClankerPoolExtensionAllowlist` `0xaa12bb11E9876FCAFc7c46dBEB985d3fA23832c9` | `ClankerHookV2` 支持 **Pool Extensions** + 允许 MEV 模块动态改 LP 费；新增 `ClankerMevDescendingFees`、`ClankerSniperAuctionV2`。**Macro 审计，2025-08** |

**多链**：Base(8453) / Arbitrum One(42161) / Unichain(130) / Ethereum(1) / BNB / **Monad(143)** / Base Sepolia，另有 **Solana**（在 Farcaster @clanker 或 X @clanker_world 触发，发的是 **Meteora** 池）。
**安全事件**：未发现针对 Clanker 自身合约的资金损失型漏洞。0xMacro 在 v3.1 审计中标记过 `FeesSwapped` 事件误发、`distributeLoop` 奖励数学边界问题（低/中危，已修）——**待核实**（未读到原始 PDF）。Clanker 代币的「跑路风险」来自单边流动性 + 无审核模型本身，而非代码 bug。

### 1.2 Uniswap v4 Hook 架构（v4 / v4.1）——模块化是关键

v4 的设计哲学是「3 个不可升级核心 + 4 个可插拔接口」。

**不可升级核心（3 个）**
- `Clanker`（工厂）：`deployToken()` 部署 ERC-20 → 触发 v4 池创建 → 触发 locker 放置流动性 → 触发 extensions。
- `ClankerFeeLocker`：**单一全局合约**，承接 v4.0.0+ 所有代币的手续费，用户 pull 式领取。
- `ClankerToken`：superchain 兼容 ERC-20，每次部署新建一份。

**4 个接口（实现必须被团队白名单）**
| 接口 | 职责 | 现有实现 |
|---|---|---|
| `IClankerHook` | Uniswap v4 hook：协议费 + 自动收费 + 触发 MEV 模块 | `ClankerHook`(abstract) → `ClankerHookStaticFee` / `ClankerHookDynamicFee`；v4.1 的 `ClankerHookV2` → `...StaticFeeV2` / `...DynamicFeeV2` |
| `IClankerLpLocker` | LP 仓位放置 + 收费 + 路由到 FeeLocker | `ClankerLpLockerFeeConversion`（支持 **7 个** 奖励接收人 + **7 个**初始 LP 仓位，每人可选收费币种） |
| `IClankerExtensions` | 部署流程之外的附加功能 | `ClankerVault`、`ClankerAirdrop`/`V2`、`ClankerUniv4EthDevBuy`、`ClankerPresaleEthToCreator`、`ClankerPresaleAllowlist` |
| `IClankerMevModule` | 按链定制的 MEV 缓解 | `ClankerMevModule2BlockDelay`、`ClankerSniperAuctionV0`/`V2`、`ClankerMevDescendingFees` |

**`ClankerHook` 抽象基类实际承担的四件事**（原文：<https://clanker.gitbook.io/documentation/references/core-contracts/v4/clankerhook.md>）
1. **每笔 swap 自动收取初始 LP 仓位的累积手续费**，经 `IClankerLpLocker` 路由进 `ClankerFeeLocker`。
2. **DEX 层协议费**，与创作者 LP 费分开计，**永远以配对代币计价**（默认 WETH），**固定为 active LP fee 的 20%**。
3. **触发 MEV 模块**，直到模块自行 disable 或过期（**上限 2 分钟**）。
4. **LP fee 上限 = 单笔 swap 的 30%**（文档明确不建议设这么高，会破坏路由器兼容性）。

> ⚠️ 工程细节（值得抄）：**两套收费机制都滞后一笔 swap**——因为 Uniswap v4 的 `PoolManager` 只在 swap 完成之后才把该笔的手续费记入池子。所以第 N 笔 swap 里收到的是第 N−1 笔的费。

**开放初始化路径**：`initializePoolOpen(clanker, pairedToken, tickIfToken0IsClanker, tickSpacing, poolData)` 任何人可调，但这样建的池**没有自动收费、没有 MEV 模块**；且 `clanker == WETH` 时 revert（协议费只在非 WETH 侧收，希望配对侧是 WETH）。

**实际用到的 hook 回调**：`beforeInitialize` / `afterInitialize`（建池 + 起始 tick）、`beforeSwap` / `afterSwap`（费用记账 + 自动领取 + MEV 模块）；**MEV 模块运行期间 `beforeAddLiquidity` 被封**——原文理由是「要保留用 `donate()` 只给原始 LP 仓位受益人付款的能力」，即**狙击窗口内第三方不能加竞争性流动性**。（精确 `Hooks.Permissions` bitmask 需读源码，**待核实**）

### 1.3 单边流动性 / 无 bonding curve 模型

这是 Clanker 与 pump.fun 最根本的分歧点。

1. 总供应固定 **100B**，一次铸造，之后不可增发。
2. 可选 vault 预留（v3.1 最多 30%，最短 30 天；v4 `ClankerVault` 最短 7 天，受 extensions 合计 90% 上限约束）。
3. 池子用「代币 + 配对代币（默认 WETH）」初始化，把**扣除 vault/airdrop/devbuy 后的全部剩余供应**作为**单边流动性**存入——池子开局是 **100% 代币 / 0% 配对代币**，起始 tick 决定隐含起始价（v3.x 默认起始市值 10 WETH；v4.1 的 MEV 文档提到**默认起始市值 $40k USD**）。
4. **因为 100% 供应已经在池子里、且价格由 tick 定死，所以不需要 bonding curve、不需要「毕业」**。价格发现完全靠买方把配对代币换进来沿集中流动性曲线推价。**Clanker 代币从第 0 个区块就是 DEX 原生资产。**
5. v4 把这一点泛化了：`LockerConfig` 允许把供应拆到多组 `(tickLower, tickUpper, positionBps)`（**最多 7 组，positionBps 必须合计 10,000**）。区间**可以不连续、可以重叠**，但**全部必须 ≥ `tickIfToken0IsClanker`**——这在结构上保证池子永远单边开局，形态上等于一道**阶梯式卖墙**，而不是平滑的 bonding curve。
6. LP 仓位 NFT 交给 locker，**locker 没有 withdraw 函数** → LP 永久锁死。原文：*"The LP Locker has no withdraw function, so LP NFTs that are deposited into it can never be withdrawn and are effectively locked forever."* 这是唯一的反 rug 机制：创作者永远拿不回本金流动性，只能拿手续费流。
7. dev buy 在 vault 计算之后执行，前端限 1 ETH，合约层不限。

**给 Mantle 的启示**：单边流动性 + 永久锁 LP 是「零成本发射 + 反 rug」的最简组合，**不需要写 bonding curve，也不需要毕业逻辑**，把复杂度从「两套定价系统 + 迁移」降到「一次 tick 计算」。代价是**没有「毕业」这个天然的注意力事件**，也没有 bonding curve 阶段的抗夹优势——Clanker 只能靠 MEV 模块来补这个洞（见 1.4）。

### 1.4 MEV / 狙击对抗模块（Base 系最成熟的一套）

| 模块 | 版本 | 机制 |
|---|---|---|
| `ClankerMevModule2BlockDelay` | v4.0 | 新池**前 2 个区块完全不可交易**，防同块抢跑 |
| `ClankerSniperAuctionV0` | v4.0 | **利用 Base 的 priority ordering**：最多 5 轮拍卖、每 2 个区块 1 轮，狙击者用 gas price 竞价「下一笔 swap 的执行权」；付款额 = (中标 gas price − 抬高后的 gas peg) × `paymentPerGasUnit` 的 WETH。**拍卖收入按 80/20 分给代币奖励接收人 / Clanker 工厂**。某轮无人出价即关闭拍卖，恢复正常交易。设计文档：<https://hackmd.io/@lobstermindset/rkwlyMpkgl> |
| `ClankerMevDescendingFees` | v4.1 | 部署者选 **startingFee（≤80%）、endingFee、secondsToDecay（≤120s）**，费用**抛物线**衰减：`fee = endingFee + feeRange × (timeDecay / timeToDecay)²`。官方推荐配置：**80% 起、30 秒衰减到 5%**。文档明确算术：默认起始市值 $40k + 80% 起始费 = **狙击者的实际入场市值 $200k** |
| `ClankerSniperAuctionV2` | v4.1 | V0 的拍卖 + V1 的衰减费复合：中标 swap 付起始高费，拍卖窗口结束后才开始抛物线衰减 |

**正常交易开放时点**（官方 `token-deployments.md`）：拍卖 5 轮跑完（**约部署后 22 秒**）或某轮无人出价立刻开放；**最坏情况延迟到 n+11 区块**。

> ⚠️ 文档里的一个安全警告值得抄：**不要让 EOA 直接给 `ClankerSniperAuctionV0` 授权 WETH**——任何人都可以把该 EOA 地址作为 `auctionData` 传进 `beforeSwap` 来花掉它的 WETH。官方提供 `ClankerSniperUtilV0` 作为「不中标就 revert」的代理示例。

**三种狙击对抗范式的取舍（这是 Mantle 设计时的核心选择题）**
- **硬延迟**（2 block delay）：最简单，但把 MEV 价值直接烧掉了，谁都没拿到。
- **拍卖**（SniperAuction）：把 MEV 价值**回流给创作者**（80%），但依赖链的 priority ordering，且需要精确落块能力（文档自己吐槽 Base 上没有 bundle 支持，"please complain on twitter to the Flashbot / Base people"）。
- **衰减费**（DescendingFees / Zora 的狙击税）：最通用、不依赖链的排序语义，把 MEV 价值转成 LP 费**分给所有受益人**。**移植性最好，Mantle 应优先选这条。**

### 1.5 费用分成——精确数字（推翻「60/40」「80/20」传闻）

官方唯一给出的协议级数字（<https://clanker.gitbook.io/documentation/general/creator-rewards-and-fees.md>）：**协议费 = 创作者 LP 费 × 20%，加在上面（不是从里面切）**。

| 创作者 LP 费 | 协议费 | 交易者实付总费 | 协议占总费比 |
|---|---|---|---|
| 1% | 0.2% | **1.2%** | 16.67% |
| 2% | 0.4% | **2.4%** | 16.67% |
| 3% | 0.6% | **3.6%** | 16.67% |

- 协议那 20% **永远以配对代币（WETH）计价**。
- **创作者那一侧**由 `LockerConfig.rewardBps` 拆给**最多 7 个** 接收人（创作者、接入方/前端、推荐人…），`rewardBps` 必须合计 10,000 且**部署后不可改**；但每个 slot 的 `rewardAdmin` 可以随时改自己的 `rewardRecipient` 和 `feePreference`。
- `ClankerLpLockerFeeConversion` 的 `FeeIn` 枚举：`Both` / `Paired` / `Clanker`——每个接收人可独立选择「收 WETH / 收该 meme 币 / 两者都收」。这是很实用的设计：创作者可以选择只收 WETH 避免自己砸盘。
- **所谓 60/40、80/20 都不是协议常数**，而是某个前端（Bankr / clanker.world / ClankFun / Native / Tab）在自己那个 rewardBps slot 内部的分法。**待核实**具体前端的分成。
- **2025-11-13 政策变更**：创作者拿回「以 CLANKER 计价的那部分手续费」的完全控制权（可领可烧）。来源：<https://www.kucoin.com/news/flash/clanker-to-return-fee-control-to-creators-starting-november-13>

### 1.6 CLANKER 代币与回购

- CoinGecko id 已改名为 **`tokenbot`**（symbol 仍是 CLANKER），Base 合约 `0x1bC0c42215582d5A085795f4baDbaC3ff36d1Bcb`。**注意 CoinGecko 上有 5 个以上同名山寨币，`clanker` / `clanker-2` 都不是它**。
- **ATH $142.84（2025-10-26）**，隐含 ATH 市值约 $141–142M（聚合器交叉一致，非一手，**待核实**）。
- **回购机制（已确认）**：Clanker 用 **协议费的 2/3** 在公开市场持续买入 CLANKER。2025-11-08 的链上实例：当日买入 2,233 CLANKER（其中 1,644 枚 = $133,047 来自 2/3 协议费，589 枚来自流动性费），团队/金库持仓达 10,349 CLANKER（约占供应 1%）。
- **DefiLlama 把这块单列为 "Holders Revenue"，累计 $5.62M** `[一手]`——即 Clanker 历史上一共约 **$5.62M 的协议收入被用于回购 CLANKER**，占累计协议收入 $15.15M 的 **37.1%**，与「2/3 协议费」的口径大体吻合（DefiLlama 的 methodology 原文：*"CLANKER tokens bought back and distributed to holders."*）。
- **Clanker Ecosystem Fund (CEF)**：2026-04 前后宣布把「相当比例的协议费重新导向创作者与社区」——**待核实**（crypto.news 页面正文未取到）。
- 「Clanker 被 Farcaster（通过 Neynar）收购」的说法**未经核实**，只在 AI 搜索合成结果里出现。
- CLANKER 总供应 / 分配 / 解锁表、是否有 staking：**待核实**。

### 1.7 Droids（2026 新增功能，很值得注意）

<https://clanker.gitbook.io/documentation/droids/funding.md>

**Droid = 随代币一起发射的 AI agent**：自带 Farcaster 账号、可自定义人格、**用代币自身 LP 收益的一个分成来支付自己的推理算力**。

资金流：
```
Token LP rewards（配对代币侧）
   → 最大的那个 paired-token reward admin slot
   → 切出 carve-out（默认 1000 bps = 10%；下限 100 bps，上限 5000 bps = 50%）
   → Droid 运行时钱包（Base）
   → 支付算力（USDC）
```
- carve-out 从**最大的 paired-token reward admin** 里扣，然后新增一条指向 droid 运行时钱包的 reward-admin。如果那个 slot 给不出请求额度，carve-out 会**静默降到可用额度**。
- 部署表单支持最多 **6 个** 表单 reward admin + droid 那条 = 链上 **7 条**（正好用满 `rewardBps` 的 7 个 slot）。
- Droid 面板显示 **USDC runway**；runway < 3 天、上次推理失败、或 runtime link 过期 → 状态变 "Needs attention"；USDC 花光后下一次推理抛 `Insufficient USDC runway`，droid 停止发帖/回复直到有人充值。
- **硬绑 Base 主网**（其他链的 droid 部署会被 API 拒绝：`Launch with Agent requires Base mainnet`）。
- 进阶：droid 的 reward-admin slot 可以指向任意合约（金库、splitter、Juicebox multiterminal），v4 和 **v5** 都支持（→ Clanker v5 已存在或在路上，**待核实**）。

**这是「代币现金流反哺自身运营成本」的第一个生产级实现**，比「代币赋能」这种空话具体得多，对 Mantle 的 mStocks launchpad 有直接借鉴价值（例：把手续费分成的一部分自动路由去支付预言机/做市/合规成本）。

### 1.8 Clanker 数据（全部一手）

**累计发币数**（clanker.world `/api/tokens`，`total` 字段，2026-09-06）：
- 全链累计 **737,210**；仅 Base **711,358（96.5%）**

**近期发币速率** `[一手]`（`startDate` 过滤）：
| 窗口 | 发币数 | 日均 |
|---|---|---|
| 24h | 337 | 337 |
| 7d | 1,080 | 154 |
| 30d | 4,628 | 154 |
| 90d | 40,458 | 450 |
| 180d | 106,004 | 589 |
| 365d | 519,602 | 1,424 |

**逐月发币数 vs 逐月手续费**（发币数来自 clanker.world API 月度边界差分；手续费来自 DefiLlama `totalDataChart`）`[一手]`

| 月份 | 发币数 | 手续费 (USD) | 每枚币产生的费用 |
|---|---|---|---|
| 2024-11 | 2,407 | $3,282,423 | $1,364 |
| **2024-12** | 10,039 | **$13,761,579** | **$1,371** |
| 2025-01 | 11,144 | $5,868,495 | $527 |
| 2025-02 | 8,510 | $3,437,875 | $404 |
| 2025-03 | **75,180** | $3,359,458 | $45 |
| 2025-04 | 19,835 | $898,398 | $45 |
| 2025-05 | 12,924 | $1,129,170 | $87 |
| 2025-06 | 15,029 | $1,260,895 | $84 |
| 2025-07 | 15,323 | $1,383,163 | $90 |
| 2025-08 | 38,155 | $2,841,217 | $74 |
| 2025-09 | 15,971 | $2,227,837 | $139 |
| 2025-10 | 27,687 | $6,048,779 | $218 |
| 2025-11 | 37,119 | $3,973,519 | $107 |
| 2025-12 | 15,464 | $965,324 | $62 |
| 2026-01 | 39,083 | $11,286,528 | $289 |
| **2026-02** | **261,047** | **$22,394,092** | **$86** |
| 2026-03 | 34,935 | $1,464,207 | $42 |
| 2026-04 | 11,712 | $794,324 | $68 |
| 2026-05 | 33,429 | $3,324,242 | $99 |
| 2026-06 | 20,307 | $507,253 | $25 |
| 2026-07 | 20,025 | $313,385 | $16 |
| 2026-08 | 4,614 | $242,379 | **$53** |
| 2026-09（6 天） | 1,001 | $58,783 | $59 |

**累计财务** `[一手]`（DefiLlama `/summary/fees/clanker`，2026-09-06）
- 累计手续费（全链全时）**$90,823,325**（Base $90.74M / Ethereum $60.3K / Arbitrum $17.1K / Unichain $4.6K / Robinhood Chain $2.2K / Monad $0）
- 累计协议收入 **$15,151,222**
- 累计 Holders Revenue（CLANKER 回购）**$5.62M**
- 24h / 7d / 30d 手续费：$8,965 / $70,750 / $283,465
- 24h / 7d / 30d 协议收入：$1,492 / $11,792 / $47,248
- **单日手续费峰值：2024-12-03，$4,790,738**

> ⚠️ **数据口径冲突（必须标注）**：DefiLlama 给出的单日峰值是 2024-12-03 的 $4.79M；而 Cointelegraph（2025-08）引用某 Dune dashboard 说峰值是 **2024-11-26 的 $1.1M**。两者相差 4 倍，说明「fees」的定义不同（DefiLlama 明确算 creator fee + Clanker 的 20%，Dune 可能只算其一，或只算 Base 上某个 factory）。同理，二手报道的「累计 $27M（2025-04）/ $34.4M（2025-08）」与 DefiLlama 的 $90.8M 也差得很远——**DefiLlama 口径更宽且包含了 2026 年那一大波，本文一律以一手 API 数据为准，二手数字仅作时间点参照。**

**历史时点（二手，各自标注）**
- 2025-04-04（The Block）：累计手续费近 **$27M**，团队收入 **$13M**，>200,000 枚代币，>**$2.7B** 链上 swap 量，Clanker 系代币总市值约 $150M；联创 Alex 称「第一天就盈利」。<https://www.theblock.co/news/business/2025-04-04-clanker-team-earns-13-million-in-revenue-from-over-200000-tokens-on-base-in-just-five-months-349549>
- 约 2025-08（Cointelegraph）：累计手续费 **$34.4M**，**355,179** 枚存活 clanker，生态市值 $172.3M；日均手续费从 2025-06 的约 **$65K** 升到 2025-07 的约 **$89K**（+37%）；v4 自 2025-06 中旬上线已部署 **7,819** 枚。
- **2026-01-30/31**：日发币量突破 **13,000 枚/日**（2025-03 中以来最高）；单日手续费突破 **$600,000**，前后连续几天累计 >**$3M**；累计交易量到 2026-02 初超 **$7.62B**，仅 1/30–1/31 两天就 >**$300M**。**诱因：Moltbook（只面向 AI agent 的社交网络）与 Clawstr 引爆的 AI agent 浪潮**，agent 用 Clanker 发币/交易形成反馈回路。来源：<https://www.kucoin.com/news/flash/clanker-token-creation-surpasses-13-000-per-day-near-previous-high>、<https://thedefiant.io/news/tokens/base-ai-agent-ecosystem-surges-with-rise-of-moltbook>

**存活率**：「>95% 的 Clanker 代币 48 小时内失去流动性」「只有约 1% 达到 $100k 市值」——两条都**无法追溯到具体研究/看板，低可信，待核实**。但从 1.8 表格的「每枚币手续费」从 $1,371 掉到 $53 可以间接确认：**长尾币的经济价值已趋近于零**。

### 1.9 分发：Farcaster bot 的 UX 与冷启动机制

- 主通道：Farcaster 上 @clanker（或 X 上 @clanker_world）→ bot 部署；bot 还支持 creator-buy 的 Frame 流程（回复一个预填参数的 clanker.world 链接）。
- clanker.world：官方前端；`clanker.world/clanker/<TOKEN>/admin` 领创作者奖励；有 by-FID 查询页。
- 第三方接口（官方文档明确承认）：**Bankr**（自然语言交易 agent，DefiLlama 上 Bankr 累计手续费 **$33.34M**、30 天 **$1.17M** `[一手]`——是目前 Base 上最活跃的发射接口）、ClankFun、Native、Tab。
- **为什么冷启动成立（三条，值得抄）**：
  1. 部署 = **一次社交动作**（一条 mention），零技术摩擦；
  2. 代币**立刻可交易**（单边流动性，无 bonding curve 毕业等待）；
  3. 创作者奖励**自动、且按社交图谱归属**（默认发到创作者 Farcaster 的 verified address / custody address）。
  → 「分发」和「变现」都原生在 Farcaster 身份图谱里，不是另开一个 connect-wallet 流程。
- **Farcaster vs 网站 vs 第三方的发币占比**：无公开拆分。v4 的 `TokenConfig.context` 字段（不可变，记录「谁部署的」）理论上可以用来还原，需要自定义 Dune 查询。**待核实**。

---

## 2. Zora：「每个 post 即币」

**官方文档**：<https://docs.zora.co/coins>｜Agent 友好版：<https://docs.zora.co/skill.md>（含 SDK 版本 `@zoralabs/coins-sdk@0.5.2`）
**ZORA 代币（Base）**：`0x1111111111166b7fe7bd91427724b487980afc69`
**ZoraFactory**：`0x777777751622c0d3258f214F9DF38E35BF45baF3`（**待核实**，未在 BaseScan 上二次确认）

### 2.1 三类币：Creator / Content / Trend

所有币**固定 1,000,000,000（1B）供应**。注意这与 ZORA 代币本身（10B）无关。

| | **Creator Coin** | **Content Coin** | **Trend Coin**（2026 新增） |
|---|---|---|---|
| 粒度 | **一个 profile 一枚**，ticker = $username | **一个 post 一枚** | 一个话题/梗一枚 |
| 进池供应 | 500M（50%） | 990M（99%） | **1,000M（100%）** |
| 创作者分配 | 500M，**5 年线性解锁**（`claimVesting()`） | 10M，**即时到账** | **0** |
| 交易费 | **1%**（10,000 pips） | **1%** | **0.01%（1 bps）** |
| 狙击税 | 99% → 1% / 10 秒 | 99% → 1% / 10 秒 | 99% → 0.01% / 10 秒 |
| LP remint | 手续费的 20% | 20% | **无** |
| 配对资产 | **ZORA** | **该创作者的 Creator Coin** | ZORA |
| 费用受益人 | 创作者 / 平台推荐 / 交易推荐 / Doppler / 协议 / LP | 同上 | **协议 100%** |
| ticker 唯一性 | 否 | 否 | **是（大小写不敏感，链上强制）** |
| 部署函数 | `deployCreatorCoin()` | `deploy()` | `deployTrendCoin()` |

**Trend Coin 的几个精巧点**（<https://docs.zora.co/coins/contracts/trend-coins>）
- ticker 的 hash **就是 CREATE2 salt** → 地址完全由 ticker 决定，`trendCoinAddress(symbol)` 可提前预测。重复 ticker `revert TickerAlreadyUsed`。
- metadata 自动生成：name = ticker，tokenURI = `https://trends.theme.wtf/trend/{symbol}`。
- 池子用**预配置的 Doppler 多曲线**：3 条曲线，供应占比 **5%（宽发现区间）/ 12.5%（中区间）/ 20%（窄区间）**——注意合计只有 37.5%，剩余 62.5% 的分布未在文档披露（**待核实**）。
- 部署参数只要 3 个（`symbol`, `postDeployHook`, `postDeployHookData`），**没有 payoutRecipient、没有 platformReferrer**。

### 2.2 合约架构

<https://docs.zora.co/coins/contracts/architecture>

- `BaseCoinV4`（abstract）：所有币型基类，持 `IPoolManager` + `PoolKey`，负责 hook 版本间的流动性迁移（`migrateLiquidity()`）与 `getPayoutSwapPath()`。
- `CreatorCoin` / `ContentCoin` / `TrendCoin`：分别继承，各带 `currency`（ZORA / CreatorCoin / ZORA）与 `vestingSchedule`。
- `ZoraFactoryImpl`：确定性地址部署 + 自动建 v4 池 + 支持 post-deploy hook + 地址预测。
- **`BaseZoraV4CoinHook`（abstract）→ `ZoraV4CoinHook`**：实现 `afterInitialize` / `beforeSwap` / `afterSwap`，内部方法 `collectFees()` / `mintLpReward()` / `_calculateLaunchFee()` / `_distributeMarketRewards()`。
  > **版本史（重要）**：v2.3.0 之前是两个 hook（`ContentCoinHook` + `CreatorCoinHook`），之后**合并为统一的 `ZoraV4CoinHook`**。

### 2.3 1% 交易费的精确分配（v2.2.0 起）

<https://docs.zora.co/coins/contracts/rewards>

**第一层拆分**：收到的 1% 手续费 → **20% 归 "LP Rewards"** + **80% 归 "Market Rewards"**。

**LP Rewards 那 20% 不是发给 LP，而是重新铸成永久单边流动性**（这是 Zora 最独特的设计）：
- token0 侧的费 → 在**当前价格之上**新建仓位（价格不涨上去就不激活）
- token1 侧的费 → 在**当前价格之下**新建仓位
- 这些仓位**被锁死/烧掉，不再赚未来的费**，纯粹为了让池子越交易越深。原文：*"Instead of extracting 20% of fees from the pool, this mechanism keeps them as permanent liquidity, ensuring the pool becomes deeper and more liquid over time."*

**第二层：Market Rewards 那 80% 的分配**

| 受益人 | 占 Market Rewards | **占总手续费** |
|---|---|---|
| Creator（创作者） | 62.5% | **50%** |
| Platform Referral（推荐创作者来发币的平台，发币时一次设定，**永久生效**） | 25% | **20%** |
| Trade Referral（推荐该笔交易的平台，**逐笔通过 `hookData` 传入**） | 5% | **4%** |
| Protocol（Zora 金库） | 6.25% | **5%** |
| Doppler | 1.25% | **1%** |
| —— LP Rewards（独立桶） | — | **20%** |

**2025 年中的费率重构（v2.2.0）：总费率从 3% 降到 1%，创作者占比从 33.33% 升到 50%**

| 受益人 | Content Coin 改前（3%） | 改后（1%） | Creator Coin 改前（3%） | 改后（1%） |
|---|---|---|---|---|
| Creator | 33.33% | **50%** | 33.33% | **50%** |
| Platform Referral | 10% | **20%** | 无 | **20%** |
| Trade Referral | 10% | **4%** | 无 | **4%** |
| Doppler | 3.33% | **1%** | 无 | **1%** |
| Protocol | 10% | **5%** | 33.33% | **5%** |
| LP Rewards | 33.33% | **20%** | 33.33% | **20%** |

净效果：**交易者实付费率降到 1/3，创作者拿到的比例提高 50%，Creator Coin 首次引入推荐分成**。（v2.2.0 的确切上线日期**待核实**）

**Trade Referral 的实现细节（很值得抄）**：
```solidity
// 发币时设定永久的 platform referral
address coin = factory.deploy(payoutRecipient, owners, uri, name, symbol,
    poolConfig, YOUR_PLATFORM_ADDRESS /* ← 永久拿 20% */,
    postDeployHook, postDeployHookData, coinSalt);

// 逐笔交易的 trade referral：直接编码进 v4 的 hookData
bytes memory hookData = abi.encode(YOUR_PLATFORM_ADDRESS);
```
→ **v4 的 `hookData` 是天然的归因通道**，不需要额外的 registry。但要注意 `hookData` 是调用者可控且核心协议不校验的，任何特权效果必须自行验签（见 §7.5）。

### 2.4 狙击税：99% → base fee，10 秒线性衰减

<https://docs.zora.co/coins/contracts/rewards#sniper-tax-early-launch-fee>

**公式（文档原文）**：
```
fee = 99% - (elapsed_seconds / 10) × (99% - base_fee)
```

| 距创建时间 | Creator/Content Coin | Trend Coin |
|---|---|---|
| 0 s | **99%** | 99% |
| 2.5 s | ~74.5% | ~74.5% |
| 5 s | ~50% | ~50% |
| 7.5 s | ~25.5% | ~25.5% |
| ≥10 s | **1%**（10,000 pips） | **0.01%**（100 pips） |

**两条豁免（文档明确）**
1. **initial supply bypass**：部署时带的 initial supply purchase 走正常 1%，不吃狙击税。
2. **legacy coins**：v2.5.0 之前创建、不支持 `IHasCreationInfo` 接口的币，一律按正常 1% 计。

**实现**：统一的 `ZoraV4CoinHook`，内部 `_calculateLaunchFee()`。
**狙击税收入去哪了**：文档**没有**给单独的受益人表 → 推测走 §2.3 的标准分配（创作者 50% / 平台推荐 20% / …）。**这是推断，非文档明示，待核实**。若成立，则意味着**创作者能吃到狙击者付的 99% 税的一半**，这是极强的创作者激励设计。
**v2.5.0 的日历日期**：**待核实**。

**与 Clanker `ClankerMevDescendingFees` 对比**：
| | Zora 狙击税 | Clanker DescendingFees |
|---|---|---|
| 起始费 | 99%（硬编码） | ≤80%（部署者可选） |
| 衰减曲线 | **线性** | **抛物线**（`endingFee + range×(t/T)²`） |
| 衰减时长 | 10 秒（硬编码） | ≤120 秒（部署者可选，推荐 30 秒） |
| 可配置性 | 无 | 高 |
| 抛物线的意义 | — | 前期跌得慢（多榨狙击者），后期跌得快（快点恢复正常） |

→ **抛物线 + 可配置是更成熟的形态**；Zora 选硬编码是为了「每个 post 都能一键发币」的零决策 UX。**Mantle 若做 mStocks 发射，应选可配置抛物线**（不同标的的狙击风险差异极大）。

### 2.5 货币层级与多跳换汇——「结构性买盘」不是「回购」

**层级**：Content Coin ←配对→ Creator Coin ←配对→ **ZORA**

**每一笔交易**，hook 自动做多跳 swap 把所有 market rewards 换成最终计价货币再付出去：
```
Content Coin 交易 → 收费 → Content Coin → Creator Coin → ZORA → 付给所有受益人
```

> ⚠️ **必须纠正一个流传很广的说法**：**Zora 没有传统意义上的「协议收入回购 ZORA」项目**。我们在 docs.zora.co、zora.co/blog、zora.co/writings 都**没有找到**任何官方回购公告，第三方 tokenomics 追踪器（tokenomist.ai）也没有 ZORA 的 buyback 条目。
> 存在的是**强制多跳换汇机制**：所有以任何币收到的手续费，在支付之前**必须在公开市场换成 ZORA**。这确实产生**持续的结构性买盘**，功能上类似回购；但换来的 ZORA 是**立刻付出去的**（给创作者/推荐人/协议/Doppler），**不进金库、不销毁**。
> → 严格说：这是**「以 ZORA 计价的手续费」而不是「用手续费买 ZORA」**。设计上更聪明（买盘规模 = 手续费规模，自动、无需治理），但也更脆弱（手续费一塌，买盘立刻归零——2026 年正是如此）。
> DefiLlama 上 ZORA 有一条累计 **$1.5M 的 "Holders Revenue"**，与文档的 Creator 分成线**无法对上**，口径**待核实**。

### 2.6 Base App 集成时间线

| 日期 | 事件 |
|---|---|
| 2025-02-04 | Coinbase 宣布把 Coinbase Wallet 重建为更宽的 App。<https://blog.base.org/evolving-coinbase-wallet-to-bring-the-world-onchain> |
| 2025-03-03 | ZORA 代币预告（"The Ticker is $ZORA"），空投第一次快照 09:00 EST / 14:00 UTC |
| **2025-03-11** | ZORA **TGE**（官方 support 文章口径，附 BaseScan tx `0x0825dac3…`） |
| 2025-04-16 | Base 官方账号发 "Base is for everyone" → 自动生成 content coin → 争议（见 §2.9） |
| 2025-04-23 | 二手来源称 ZORA「公开领取/开放交易」日（与 3-11 的 TGE **口径冲突，待核实**，很可能是两个不同事件） |
| **2025-06-20** | **Creator Coins 上线**。官方原话："Earn 1% on every trade of your creator coin and posts"；"50% of the creator coin supply streams into your wallet over 5 years" |
| **2025-07-16** | **Base App 上线（"A New Day One"）**，Coinbase Wallet 正式更名。社交流跑在 **Farcaster 协议**上，**Zora 原生驱动 post→币、profile→币**。同日 **Flashblocks 主网上线**。<https://blog.base.org/a-new-day-one> |
| 2025-07-27 | **单日新建币峰值约 54,000 枚**；日活交易钱包 >50,000；日成交峰值约 $41M；7 月月协议收入约 $2M（前高约 $600K，>3x） |
| 2025-12-17/18 | Base App 退出 beta，开放 140+ 国家（二手，**待核实**） |
| **2026-02-18** | **Base App Creator Rewards 关停**。7 个月共发出约 **$450,000 给约 17,000 名创作者** |
| 2026-07-15/16 | Jesse Pollak 交出 Base App，公开承认 creator coins 路线「definitively wrong」 |
| 2026-09 | Base 新推「Creator Grant Program」——**传统现金 grant，不再是代币奖励** |

Base 官方原文（2025-07-16，值得完整引用，因为它就是「创作者经济做冷启动」这条论点的最强证据）：
> *"We believe the next chapter of the internet won't come from big platforms. It'll come from creators. That's why we rebuilt the Base app from the ground up, not just as a wallet, but as a new kind of open social network. One where you own what you post. Where your work earns. Where your feed is yours."*
> *"Earn from your content: **Every post is a coin that can be bought, powered by Zora.** Creators can earn with no follower minimum, brand deals, or geo restrictions."*

**「通过 Base App 发的币 vs 通过 zora.co 发的币」的拆分**：无公开数据。两个入口共用同一套合约，所有公开看板（DefiLlama / Dune）只报协议级总量。**待核实/可能无法回答**。

### 2.7 ZORA 代币

- 总供应 **10,000,000,000（10B）**，固定，Base 上。
- **治理权：明确没有**。官方原文：*"ZORA is for fun only and does not entitle its holders to any governance rights or a claim on any equity ownership in Zora or its products."*
- 解锁（官方 support 文章文本表）：Treasury 自 TGE 后 6 个月起、**48 个月**按月释放；Team 6 个月起、**36 个月**按月；Investors 6 个月起、**36 个月**按月；Incentives / Airdrop / Liquidity **无锁定**。
- 分配比例（Airdrop 10% / Treasury 20% / Team 18.9% / Investors 26.1%）——官方页面是**图片**，无法提取文本，聚合器数字**未经核实**。
- 市场数据 `[一手]`（CoinGecko，2026-09-06）：价格 **$0.0086**，市值 **$38.36M**，FDV **$85.81M**，**ATH $0.145579（2025-08-11）**，距 ATH **−94.1%**，近 1 年 **−87.8%**（近 30 天 +64.6%，属反弹）。

### 2.8 Zora 数据（一手）

**Zora Coins 逐月手续费与 DEX 成交额** `[一手]`（DefiLlama `zora-coins`，2026-09-06）

| 月份 | 手续费 (USD) | DEX 成交额 (USD, 百万) |
|---|---|---|
| 2025-02（起） | $228,384 | 0.9 |
| 2025-03 | $181,961 | 1.9 |
| 2025-04 | $588,102 | 7.0 |
| 2025-05 | $250,011 | 0.4 |
| 2025-06 | $52,807 | 0.0 |
| **2025-07** | **$2,348,233** | **101.8** |
| **2025-08** | **$2,510,233** | **113.0** |
| 2025-09 | $779,787 | 32.5 |
| 2025-10 | $2,116,116 | 90.1 |
| 2025-11 | $742,203 | 27.0 |
| 2025-12 | $199,604 | 7.2 |
| 2026-01 | $191,525 | 7.7 |
| 2026-02 | $57,811 | 2.4 |
| 2026-03 | $30,471 | 1.2 |
| 2026-04 | $28,861 | 1.2 |
| 2026-05 | $58,652 | 2.5 |
| 2026-06 | $19,025 | 0.8 |
| 2026-07 | $28,158 | 1.1 |
| **2026-08** | **$14,768** | **0.5** |
| 2026-09（6 天） | $3,108 | 0.1 |

- 累计手续费 **$10,429,820**（Base）；累计 DEX 成交额 **$399,441,758**
- **单日手续费峰值：2025-07-27，$646,017**（正是 Base App 上线后 11 天）
- 24h / 30d 手续费：**$300 / $15,067**
- 24h / 30d DEX 成交额：$14,212 / $555,280
- **新建币速率 `[一手]`（我直接分页 Zora API `explore?listType=NEW`）：2026-09-05→06 的 24 小时内 422 枚，全部为 `CONTENT` 类型，0 枚 Creator Coin、0 枚 Trend Coin。**
  → 对比 2025-07-27 的约 54,000 枚/日：**−99.2%**。**Creator Coin 已实质停止新增。**

**前置时代基线**（2025-03，官方 blog，NFT 协议时代累计）：2.4M+ collectors、618K+ creators、$27.7M+ rewards、$376M+ 二级成交额。
**2025-07 底口径（二手，待核实）**：约 160 万枚币、>200,000 独立创作者、>$445M 累计成交额。

### 2.9 两场争议

**争议一：「Base is for everyone」事件（2025-04-16）**
Base 官方账号发了一条本来是品牌 slogan 的 post，Zora 自动为它生成 content coin。市场解读为「Base/Coinbase 官方代币」，市值冲到约 **$14–18M** 后 **暴跌约 95%**，引发 rug / pump-and-dump 指控。Coinbase 随后声明：Base 没有发官方代币、该 content coin 不是 Base/Coinbase 官方资产、Base 的份额不会卖出、手续费将用于开发者 grant。Jesse Pollak 辩称这是「content coin」实验，2025-04-23 承认「执行方式本可以更好」。
来源：<https://fortune.com/crypto/2025/04/19/coinbase-zora-base-content-coin-jesse-pollak>、<https://cointelegraph.com/news/coinbase-distances-base-criticized-memecoin-drops-15-million>、<https://blockworks.com/news/jesse-pollak-base-content-coin>

**争议二：结构性批评（"financial nihilism"）**
- content coin 是纯投机/娱乐资产，对现金流无索取权；
- 创作者/内部人向散户砸盘；
- 流动性太薄，滑点巨大；
- 严重的生存者偏差（少数病毒级赢家被大量报道，长尾迅速丧失流动性）。
「30 天留存死亡区」的说法广泛流传但**找不到一手量化数据集**（需自己跑 Dune 查询）。不过 §2.8 的一手数据已经从另一个角度证实了：**成交额 −99.6%、新建币 −99.2%，且 Creator Coin 新增归零**。

---

## 3. Flaunch：Uniswap v4 hook 的金融工程实验室

**文档**：<https://docs.flaunch.gg>（llms.txt：<https://docs.flaunch.gg/llms.txt>）
**代码**：<https://github.com/flayerlabs/flaunchgg-contracts>（MIT / Foundry）
**团队**：Flayer Labs（2024-09 由 NFTX + FloorDAO 合并而来，**待核实**）
**审计**：EnigmaDark、Omniscia
**治理代币**：$FLAY（ERC20Votes，Tally 治理）

### 3.1 Progressive Bid Wall（PBW）——本专题最值得抄的一个机制

**文档**：<https://docs.flaunch.gg/features/auto-buybacks.md>
**合约**：`src/contracts/bidwall/BidWall.sol`

合约自己的 docstring：
> *"a single sided liquidity position (Plunge Protection) that is placed 1 tick below spot price, using the ETH fees accumulated. After each deposit into the BidWall the position is rebalanced to ensure it remains 1 tick below spot."*

**接线方式（重要：BidWall 本身不是 hook）**
- `PositionManager` 才是真正的 Uniswap v4 hook；`BidWall` 是卫星合约，只有持 `ProtocolRoles.POSITION_MANAGER` 角色的 `PositionManager` 能调（`onlyPositionManager` modifier）。
- `PositionManager` 在 **`afterSwap`** 的费用分配路径里调 `BidWall.deposit(poolKey, ethSwapAmount, currentTick, nativeIsZero)`，传入的是**社区侧（非创作者）那部分 1% 手续费，且已折算为 ETH（flETH）**。
- `PositionManager` 在 **`beforeSwap`** 里调 `BidWall.checkStalePosition(...)`，若 **`staleTimeWindow`（默认 7 天）** 内没有 BidWall 交易，强制提前重定位（把累积价值取出来）。

**触发阈值**
- `deposit()` 累加 `pendingETHFees` 与 `cumulativeSwapFees`。
- 常量 **`_swapFeeThreshold = 0.1 ether`**（构造函数默认，owner 可通过 `setSwapFeeThreshold` 调；基类的 `_getSwapFeeThreshold(cumulativeSwapFees)` 是 `virtual`，留了「随累计费缩放」的口子）。文档口径一致：*"A new PBW is created for every 0.1 ETH of trading fees it receives."*
- `pendingETHFees >= threshold` 时执行 `_reposition()`。

**挂单位置计算（`_addETHLiquidity`）**
```
TickFinder.TICK_SPACING = 60                       // 池子固定 tickSpacing
baseTick = nativeIsZero ? currentTick + 1 : currentTick - 1   // 现价外 1 个 raw tick，ETH 买侧
newTickLower = TickFinder.validTick(baseTick)      // 对齐到 60 的倍数
newTickUpper = newTickLower + 60                   // 恰好一个 tickSpacing 宽
liquidity = LiquidityAmounts.getLiquidityForAmount0/1(...)    // 只用 ETH/flETH 计量
```
→ **这是真正的单边限价买单，不是对称 LP 仓位。**

**「棘轮上移」的实现**（这是精髓）
每次 `_reposition()`：
1. **先完全移除**旧 BidWall 仓位（`_removeLiquidity`，把记录的 tickLower/tickUpper 上的流动性全部 burn），收回其中的 ETH **和/或** memecoin；
2. **再重建**一个新仓位，规模 = `ethWithdrawn + newFees`，位置按**当时的** currentTick 重新算。

因为价格涨过之后旧仓位的 ETH 已被吃成 memecoin，收回的 ETH + 新手续费会被部署到**新的、更高的**现价下方一个 tickSpacing 处 → **买墙价位单调不降**。它自己永远不会往下走，只在新手续费到账或 stale 触发时**向上**重定位。

**被吃单之后的钱去哪了**
- `memecoinWithdrawn`（旧区间里积累的 memecoin）→ **直接转给该代币的 `MemecoinTreasury`**（不烧、不卖），发 `BidWallRewardsTransferred` 事件。
- `ethWithdrawn` → 直接循环进新仓位。

**创作者能不能提走**
- BidWall 流动性属于池内协议持有，不直接归创作者。但创作者（`IMemecoin(_memecoin).creator()` 验证）可以调 `setDisabledState(poolKey, true)` → `PositionManager.closeBidWall()` → `BidWall.closeBidWall()`：**完全移除所有 BidWall 流动性，把 ETH 和 memecoin 全部转给该代币的 `MemecoinTreasury`**（不是创作者个人钱包），之后由挂在上面的 `TreasuryActionManager` 白名单动作（如 `BuyBackAction.sol`、`BurnTokensAction.sol`）来支配。重新启用需要从零攒新手续费。
- 文档提到 memecoin 收益可以去驱动 "Full Stack Churchills"（一个白名单化的市价买入动作）。

**为什么这比「用手续费市价回购」更优**
文档的论点（whitepaper）：
> *"PBWs have the effect of supporting price, without risk of loss to MEV or bots via market buys, achieving a more effective result for memecoin holders."*
市价回购是一笔可预测的大买单，必然被三明治夹；**挂成限价单则是被动成交，MEV 无从下手**，且提供了真实的挂单深度。这一点对 Mantle 尤其重要——**在流动性薄的链上，被动挂单比主动市价买入的滑点损耗小一个数量级**。

### 3.2 Fixed-Price Fair Launch（固定价格公平发射窗口）

**文档**：<https://docs.flaunch.gg/features/fixed-price-fair-launch.md>

- **默认时长 30 分钟**（whitepaper：*"Every coin starts with a 30 minute Fair Launch period"*），可按次配置。
- **机制**：把总供应（100,000,000,000）的一个百分比放在**单一 tick** 上 → **窗口期内所有人同价**，degen、bot、KOL 一视同仁。
- **只能买不能卖**：*"While a Fair Launch is active, coins that are purchased cannot be sold."* 但窗口结束后，**Fair Launch 期间买入的可以按同价卖出（扣手续费）**——即**价格风险为零**。
- **窗口结束时（这是设计上最漂亮的一步）**：
  1. Fair Launch 募到的**全部 ETH → 立刻做成一道位于现价下方的买墙**（即 PBW），保证 Fair Launch 买家能按入场价原价退出；
  2. **未卖出的 Fair Launch 额度 + 未投入 Fair Launch 的剩余供应 → 全部投成从当前现价起的全区间（full range）仓位**，价格发现正式开始。
  3. LP 仓位**从发币那一刻起就等效于被烧掉**，收益流永久归 dev + 持币人。
- **超买警告**：可以买超过剩余 Fair Launch 额度，但**超出部分不受固定价格保护**。
- **创作者预买（premine）**：创作者可以在公开之前、以同价买下一部分 Fair Launch 额度（`FlaunchPremineZap`；`FlaunchParams.premineAmount`）。
- **实现位置**：现版本在 `PositionManager` 内部（`flaunchesAt` mapping 门禁 + `initialPoolTick`）。**早期有独立的 `FairLaunch.sol`，其 Base / Base Sepolia 部署地址在 repo README 里已标记 `[Deprecated]`**，逻辑已折进 PositionManager v1.1+。
- 默认起始市值常被引为 **$10,000**，但**未在取到的官方页面中确认**（**待核实**；文档只说 Starting Market Cap 可配）。Fair Launch 默认占总供应的百分比同样**待核实**。

**与 bonding curve 的本质区别**：Flaunch **不用**连续定价曲线。价格在整个 Fair Launch 窗口内**完全平坦**（只受固定供应额度限制），然后**跳变**到全区间 AMM 交易——是**离散两段式**，不是平滑曲线。

**额外的 anti-bot 层**
- **Sniper Protection**（<https://docs.flaunch.gg/features/sniper-protection.md>）：把 web2 验证嵌进 AMM——Fair Launch 期间（**默认 5 分钟**，注意这个 5 分钟是 CAPTCHA 门禁子窗口，不是 30 分钟的 Fair Launch 本身）必须先过 **CAPTCHA** 才能 swap；还可以设 **per-wallet cap**。可自定义实现，用任意链下数据授予准入。技术路径见 <https://docs.flaunch.gg/references/spend-gate.md>（Trusted Signer + v4 hook 内强制执行）。
- **Game Mode（2026 新，见 §3.6）**：从「你是不是人」升级到「你凭什么配得上这个额度」。

### 3.3 创作者收益 0–100% 可调 + 收益权 NFT 化

**文档**：<https://docs.flaunch.gg/features/creator-revenue.md>、<https://docs.flaunch.gg/features/royalty-nft.md>

- **swap 费固定 1%，买卖双向**（FAQ 原文："1% on both buys and sells"）。注意**池子的 v4 原生 `fee` 参数被设为 0**，所有真实收费都在 hook 内部由 `FeeDistributor._captureSwapFees` 执行：`swapFee_ = swapAmount * baseSwapFee / 100_00`，再用 `poolManager.take` 取走。→ 这样才能实现「按创作者自定义比例 + 瀑布式分配」。
- **创作者份额：发币时一次选定 0%–100%**，存在 `creatorFee[poolId]`，**发币后不可改**（文档："Revenue split is immutable and cannot be changed after the coin has launched"）。
- **默认 UI 分法：创作者 80% / 社区回购 20%**。
- **创作者没拿的部分自动全进 PBW**："Whatever the dev doesn't take in fees goes to the PBW. For instance, if the dev share was 20%, then 80% would go to the PBW."
- **结算币种：ETH（技术上是 flETH）**，逐笔 swap 实时流入 `FeeEscrow`，累计超过 **`MIN_DISTRIBUTE_THRESHOLD` = 0.001 ETH** 即可领。
- **Royalty NFT / "Memestream"**：发币时 `Flaunch.sol`（ERC-721）给创作者 mint 一枚 NFT，代表**该代币创作者手续费流的独占权利**。可自由转让（FAQ：Memestream 二级交易无版税），从而解锁：
  - **卖掉未来现金流**（一次性变现）
  - **以未来收入做抵押借贷**
  - **碎片化 / DAO 化 / 多签共管**
  - **利率互换**（把浮动收益换成固定收益）
  - **真正的 CTO（Community Take Over）市场**：创作者跑路后，社区可以在二级市场买下它的收入流 + 管理权，把文化续下去并且**因此获得报酬**
  - NFT 持有者还获得「Meme Management」权：市价买回、销毁买回的代币等（经 `TreasuryActionManager` 白名单）

### 3.4 费用瀑布与协议费开关

**文档**：<https://docs.flaunch.gg/community/governance/protocol-fee-switch.md>

**瀑布顺序**（`FeeDistributor.sol` 代码注释）：`swapfee → referrer → protocol → creator → bidwall`

- **`MAX_PROTOCOL_ALLOCATION = 10_00`** → 协议费**链上硬上限 10%**，2 位小数表示（`550` = 5.5%）。
- **当前协议费设为 0%**。FAQ 原文：*"Does the team take any fees? No… FLAY governance can choose to turn on a fee switch that can take a maximum of 10% of the fees."*
- 官方给的**瀑布算例**（假设 100 ETH 成交、协议费开到 5%、创作者 80%）：

| 受益人 | 金额 | 计算 |
|---|---|---|
| Swap Fee 总额 | 1 ETH | `100 / 100 * 1` |
| Protocol | 0.0475 ETH | `(1 - 0.05) / 100 * 5` |
| Creator | 0.722 ETH | `(1 - 0.05 - 0.0475) / 100 * 80` |
| Bidwall | 0.1805 ETH | 剩余 |

- **一个不可逆的历史包袱（对 Mantle 是很好的反面教材）**：**第一个** `PositionManager` 合约的 protocolFeeRecipient 被写死为 **Flayer Foundation 多签，不可更改**。文档写明「截至撰写时该 `PositionManager1` 上有 **4,755** 枚已 flaunch 的代币」，并说明基金会**在法律上没有义务**把这些费用用于回购。之后所有 PositionManager 的费用才由 $FLAY 持有人链上治理支配。
  → **教训：把「收款人」写成不可变的会永久绑定一个可能过时的治理主体。设计时务必留可治理的 recipient 指针。**
- 地址：`FeeEscrow` `0x72e6f7948b1B1A343B477F39aAbd2E35E6D27dde`；`ProtocolFeeRecipient` `0x1150c53eB4cE3aDE47808D1D1Ac9636b774eE079`；`PositionManager1` `0x6A53F8b799bE11a2A3264eF0bfF183dCB12d9571`；`PositionManager2` `0xB4512bf57d50fbcb64a3adF8b17a79b2A204C18C`（Base 8453）。
- $FLAY 治理地址：Ethereum Governor `0x8BA5eA8c8b1Aafe9dbcb7a36737AcfAd6afa5D38`；Token `0xF1A7000000950C7ad8Aff13118Bb7aB561A448ee`；Timelock `0x6c4c0CD7E0E5eeFfbd77AAfe1820d3b9B1ef27b0`；Base `L2Owner 0x000000000Bb63D5c070d0D5791517886a4d8C545`。

### 3.5 AMM 细节（Flaunch 用 hook 的深度是四家里最高的）

**PoolKey 统一形态**：`{ currency0/1: flETH & memecoin（按地址排序）, fee: 0, tickSpacing: 60, hooks: PositionManager }`

**hook 回调逐个用途**（官方 <https://docs.flaunch.gg/references/hooks.md> 原文整理）

| 回调 | 用途 |
|---|---|
| `beforeInitialize` | **阻止外部合约用这个 hook 初始化池子** |
| `afterInitialize` | 向 Notifier 订阅者与 subgraph 发 PoolState 更新 |
| `beforeAddLiquidity` | **Fair Launch 窗口内禁止外部加流动性** |
| `afterAddLiquidity` | 发 PoolState 更新 |
| `beforeRemoveLiquidity` | **Fair Launch 窗口内禁止外部撤流动性** |
| `afterRemoveLiquidity` | 发 PoolState 更新 |
| **`beforeSwap`** | ① 若代币是定时发射的，仅在有 premine 调用时允许 swap；② 若 Fair Launch 窗口已结束但仓位还开着 → **关闭仓位**；③ **尝试用 FairLaunch 仓位吃掉这笔 swap**，若超出额度或窗口已过则同时关闭仓位并建立新区间；④ **用手上的 token1 手续费库存在 swap 打到 Uniswap 池之前先填单**——原文：*"This frontruns Uniswap to sell undesired token amounts from our fees into desired tokens ahead of our fee distribution. This acts as a partial orderbook to remove impact against our pool."* |
| **`afterSwap`** | ① 捕获本笔手续费，分配或送进 ISP；② 向 LP 分费并发价格更新事件；③ 若挂了 `feeCalculator`，记录 swap 数据供动态计算；④ 发 PoolState 更新 |
| `beforeDonate` | 未使用 |
| `afterDonate` | 发 PoolState 更新 |

**关键组件**
- **`PositionManager.sol`**：真正的 v4 hook（继承 `UnlockingHook` / `FeeDistributor`）。核心入口 `flaunch(FlaunchParams)`，参数结构：
  ```solidity
  struct FlaunchParams {
      string name; string symbol; string tokenUri;
      uint initialTokenFairLaunch;   // 作为单边 fair launch 流动性的代币量
      uint premineAmount;            // 创作者自己先买的量
      address creator;               // 收 ERC721 与 premine 代币的地址
      uint24 creatorFeeAllocation;   // 创作者从 BidWall 手里拿走的费用百分比
      uint flaunchAt;                // 定时发射时间戳
  }
  ```
  另有 `getFlaunchingFee()`（发币需付的 ETH 费）、`getFlaunchingMarketCap()`、`poolKey(token)`（`tickSpacing == 0` 即为空）。
  可插拔的 `IFeeCalculator`：`StaticFeeCalculator` / `TrustedSignerFeeCalculator` / `PauseCalculator`；另有 `FeeExemptions.sol` 白名单。
- **`InternalSwapPool.sol`**：docstring 原文 *"Frontruns Uniswap to sell undesired token amounts from protocol fees into desired tokens ahead of fee distribution, acting as a partial orderbook that removes impact against the pool."* 当手续费收到的是**非原生币**时，不去公开 AMM 砸盘，而是**用真实用户的 swap 在内部对冲掉**，定价用 **`oracle.twapTick()`（绝不是 spot）** 经 `SwapMath.computeSwapStep` 计算 → **原子内操纵不可行**。
- **`Oracle.sol`**：TWAP 环形缓冲，建池时 `oracle.recordObservation` 种子化。
- **flETH**：Base 上 `0x000000000d564d5be76f7f0d28fe52605afc7cf8`，1:1 ETH 背书的 ERC-20 包装（像 WETH 一样 deposit/withdraw）。**闲置在 Flaunch 池里的 flETH 背书资产会被 `flETHHooks` 扫进 `AaveV3Strategy`（`FlAaveV3WethGateway`）吃 Aave v3 借贷收益（FAQ 称「ultra low risk 2% yield」）**。这份收益归 **Flayer Foundation**，不归创作者或交易者——**这就是 DefiLlama 上 Flaunch "Revenue" 那一行的来源，而不是手续费抽成**。
  > **这是本专题第二个最值得抄的机制**：把 AMM 里躺着的 quote 资产做生息包装，收益归协议。对 Mantle 尤其契合——Mantle 有 mETH/cmETH 这套生息 ETH 基础设施，**「launchpad 的 quote 资产默认是生息资产」在 Mantle 上是天然优势**。
- **`MemecoinTreasury`** + **`TreasuryActionManager`**（白名单动作：`BuyBackAction.sol`、`BurnTokensAction.sol`，协议级另有 `BuyBackAndBurnFlay.sol`）
- **`InitialPrice` / `MarketCappedPrice` / `AnyMarketCappedPriceV3`**：从目标市值反算发射 `sqrtPriceX96`（后两者源码**待核实**）
- **`FlaunchZap` / `FlaunchPremineZap`**：前端入口，处理 premine/prebuy

### 3.6 Revenue Manager / Treasury Manager：让别人在你身上建 launchpad

**文档**：<https://docs.flaunch.gg/managers/revenuemanager.md>

Flaunch 的协议设计**天然偏向代币创作者**，这对想在上面建 launchpad 的第三方协议不友好。解法是 `RevenueManager`——一个**中间件 escrow**，先截住手续费再按任意规则分。

```solidity
struct InitializeParams {
    address payable protocolRecipient;  // 外部协议的收款地址
    uint protocolFee;                   // 外部协议抽成（2 位小数）
}
```
关键调用：`balances(recipient)`、`claim()`、`claim(FlaunchToken[])`、`creator(flaunch, tokenId)`、`creatorTotalClaimed(creator)`、`deposit(flaunchToken, creator, data)`、`getProtocolFee(amount)`、`protocolTotalClaimed()`、`tokens(creator)`、`tokenTotalClaimed(flaunch, tokenId)`；owner-only：`rescue()`、`setCreator()`、`setProtocolRecipient()`（可设 0 地址来跳过抽成）、`transferManagerOwnership()`。

文档给外部协议的四条卖点（原文）：
1. 建自己的 launchpad，业务模型完全自主；
2. 把自己协议的功能接进 Flaunch 的白名单 treasury actions 来做 TVL；
3. 用「代币化收益流」做金钱游戏并盈利；
4. **builder 最多可拿 100% 的手续费**，剩余归代币创作者与社区。

Manager 家族：`RevenueManager` / `StakingManager` / `AddressFeeSplitManager` / 自定义（<https://docs.flaunch.gg/managers/custom-managers.md>）。它们可以**代持 Flaunch ERC-721**（代表创作者/DAO/多签）并施加自定义分账逻辑。

> **战略含义**：Flaunch 已经从「一个 launchpad」转型成「**launchpad 的底层协议**」。这是量级最小但战略位置最好的一家——因为它把自己变成了别人的基础设施。Mantle 若要做 launchpad，**这是最值得对标的定位**（做 Mantle 上所有 launchpad 的共同底层，而不是做第 N 个前端）。

### 3.7 Game Mode（2026 新增）：play-to-enter bonding curve

**文档**：<https://docs.flaunch.gg/game-mode/game-mode.md>｜技术细节 <https://docs.flaunch.gg/references/spend-gate.md>

**问题定义（说得非常准）**：
> *"A fair launch is meant to give everyone the same shot at a coin. In practice the first block goes to whoever has the best infrastructure. Bots watch the mempool, land their buys in the same second the pool opens, and sell into the people who arrived moments later. Technically nothing stopped you from buying; practically the good price was gone before you saw the coin."*

**机制**：把「抢跑竞赛」换成「窗口」——窗口开着的时候，**唯一上曲线的方式是玩游戏赚额度**。
1. 代币带窗口发射，创作者选一个游戏守门。**窗口打开前，池子里每一笔 swap 都 revert——这是池子自己的规则，不靠游戏服务器执行。**
2. 所有人共享一个窗口、一个排行榜。**计分在服务器端，服务器根据玩家输入重放每一步动作**（不信任浏览器）。
3. **分数换成 ETH 花费额度**。分越高买越多，上限是对所有人一致的 per-wallet cap。
4. 领取额度会产生一个**签名授权**，池子在每一笔 swap 上校验签名与金额。
5. 窗口关闭，门禁解除，代币变成普通代币。

**池子（而非服务器）保证的四件事**：
- 窗口前不能买（开启时间写进池子）
- 不能买超过赚到的（每张授权带 max spend，池子按 swap 的真实 ETH 输入量比对）
- 不能买超过份额（per-wallet cap **累计**强制）
- **代币一定会按时进入公开市场**——门禁带一个写进池子的 expiry，过期后门禁自行停止执行，**不需要游戏服务器、创作者或 Flaunch 发任何交易**。原文强调这一点：*"Without an expiry, a coin would need our servers to still be alive to ever trade freely."*

**一次真实 Game Mode 轮次的分发数据**（官方，来自 <https://x.com/flaunchgg/status/2085701754864189580>）：
| 时点 | 已售供应 | 持有钱包数 |
|---|---|---|
| 10 秒 | 0.72% | **2** |
| 30 秒 | — | 51 |
| 60 秒 | — | 138 |
| 窗口关闭 | **35.55%** | **194** |

**对照组**：一个 15 钱包的 bundler 集群，**窗口内买入 0.00%**（其中 1 个尝试玩了一轮，没有人买），窗口结束后在公开市场买了 9.27%，**价格是玩家们已经定好的**。官方总结：*"For the first time, the players front-ran the bundlers."*

**经济设计**：游戏本身**永不代币化**（游戏没有自己的币，做游戏不涉及发币）；**官方游戏库里的游戏，可以从经它发射的每一枚币的交易手续费里分成**。首个游戏 `Split the Arrow`（three.js 3D 射箭，带风、有限箭数、可观战的排行榜）。**Game Mode 目前在 Robinhood Chain 上线**（不是 Base）。

> **给 Mantle 的直接启示**：Game Mode 是「**用链下可验证的努力换取链上额度**」的通用模板——把 `spend-gate`（trusted signer 签名 + hook 内强制 max-spend / per-wallet cap / expiry）抽出来，游戏可以换成任何东西：**Mantle 生态任务、mETH 持仓时长、Bybit KYC 等级、mStocks 的持仓证明**。这比单纯的 CAPTCHA 或白名单强得多，因为**执行在 hook 里，不依赖服务器活着**。

### 3.8 Flaunch 数据

**官方 Dune**（<https://dune.com/flaunch/flaunch-protocol-dashboard>，2026-09-06 读取）
- 累计发币 **151,995**
- 累计交易量 **$472,864,098**
- 累计手续费 **2,072.6898 ETH**
- 累计创作者收入 **792.6695 ETH**
- 累计回购（PBW）量 **239.6914 ETH**
- 独立交易者 **127,749**；独立部署者 **22,513**

**DefiLlama** `[一手]`（2026-09-06）
- TVL **$1.79M**（100% Base，+21.9% / 30d）
- 累计手续费 **$3,588,635**；累计 revenue **$3,073,917**（= flETH 的 Aave 收益，不是抽成）
- 24h / 7d / 30d 手续费：**$28.93 / $158.39 / $389.64**
- **单日手续费峰值：2025-01-29，$871,388**
- Launchpad 类目里按 TVL 排 **#11 / 249**（类目总 TVL $253.04M，Flaunch 占 0.7%）

**逐月手续费** `[一手]`（USD）
| 2025-01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **1,118,116** | 629,390 | 23,714 | 5,323 | 42,318 | 14,028 | 45,889 | 533,314 | 7,277 | **671,836** | 5,809 | 931 |

| 2026-01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09(6d) |
|---|---|---|---|---|---|---|---|---|
| 5,826 | 38,137 | 56,670 | **230,945** | 29,794 | 108 | **128,773** | 283 | 153 |

→ Flaunch 的形态很特殊：**极度脉冲式**（2025-01、2025-08、2025-10、2026-04、2026-07 各有一根尖峰，其余月份趋近于零）。这说明它的收入完全由**个别病毒级代币**驱动，没有 base load。这与 Bags 的形态一致（见 §4.1）。

**FLAY 代币** `[一手]`（CoinGecko，2026-09-06）：$0.0044，市值 **$2.64M**，FDV $4.40M，**ATH $0.257272（2025-01-31）**，距 ATH **−98.3%**，近 1 年 −88.4%。

**多链扩张**：原生 Base，现文档/SDK 已列 **Ethereum 主网、Unichain、Robinhood Chain**（部分 Base 专有功能未支持，**待核实**具体地址表）。2025-12 changelog 有「Solana Imports」「Flaunch Wrap: Solana → Base Bridging for Token Creators」（**待核实**机制）。
**AI 入口**：**Flaunchy**（X 上 @ 它就发币）、**Flaunch MCP**、AI Skills（`npx skills add https://github.com/flayerlabs/flaunch-skills`）。
**融资**：**未找到任何可信一手来源确认 Flaunch / Flayer Labs 有 VC/seed 轮**——按未确认处理（**待核实**）。

---

## 4. 其他玩法（Bags / Believe / Virtuals / Doppler / Wow.xyz 等）

### 4.0 横向数据对照表（全部 `[一手]`，DefiLlama，2026-09-06）

| 项目 | 链 | 类别 | 累计手续费 | 近 30 天 | 近 24h | 峰值单日 / 峰值月 | 状态 |
|---|---|---|---|---|---|---|---|
| **pump.fun** | Solana | Launchpad | **$1,210,677,613** | $47,041,841 | $679,806 | — | **活跃，绝对霸主** |
| **PumpSwap** | Solana | DEX | $811,258,622 | $89,627,649 | $2,677,316 | — | 活跃 |
| **four.meme** | BSC | Launchpad | $98,046,027 | $290,434 | $9,578 | — | 大幅冷却 |
| **Clanker** | Base+5 链 | Launchpad | **$90,823,325** | $283,465 | $8,965 | 2024-12-03 $4.79M / 2026-02 $22.39M | 大幅冷却 |
| **Virtuals Protocol** | Base(+Sol) | AI Agents | $73,849,436 | $665,753 | $11,202 | 2025-01-02 $1.59M / 2025-01 $16.59M | 冷却但仍有量 |
| **BONK.fun** | Solana | Launchpad | $66,994,970 | $181,559 | $19,627 | — | 冷却 |
| **Bags** | Solana(+RH) | Launchpad | $64,002,249 | **$1,807,694** | $30,603 | 2026-01-16 $4.80M / 2025-08 $25.70M | **Base 系之外最活跃的创作者分成盘** |
| **Believe(LaunchCoin)** | Solana | Launchpad | $40,910,390 | **$214** | $0.57 | 2025-05 峰值 | **已实质死亡** |
| **Bankr** | Base(+RH) | Interface | $33,344,633 | **$1,172,755** | $10,985 | — | **Base 上最活跃的发射/交易接口** |
| **ZORA Coins** | Base | Launchpad | $10,429,820 | $15,067 | $300 | 2025-07-27 $646K / 2025-08 $2.51M | 接近归零 |
| **pump.fun Mobile** | Solana | Interface | $8,732,012 | **$6,056,344** | $134,059 | — | **唯一逆势增长的** |
| **Flaunch** | Base | Launchpad | $3,588,635 | $389.64 | $28.93 | 2025-01-29 $871K / 2025-01 $1.12M | 接近归零 |

> **一句话读表**：pump.fun 累计手续费（$1.21B）≈ Clanker + Zora + Flaunch + Virtuals 四家 Base 系之和（约 $178.7M）的 **6.8 倍**；只看近 30 天则是 **$47.0M vs $0.965M ≈ 49 倍**。**Base 系 launchpad 的衰减速度远快于 Solana 系。** 而 Base 上目前唯一还有可观现金流的，是 **Bankr（接口层）** 而不是任何 launchpad 本身。

### 4.1 Bags（bags.fm）—— ⚠️ **链 = Solana，不是 Base**

**为什么必须放进本专题作为对照**：Bags 是「创作者永续分成」这一机制的最干净对照实验。它跑在 Solana（Raydium/Meteora 轨道）上，**证明这套机制是链无关、可移植的** → **推论：Base 的优势从来不在 fee-sharing 这个原语，只在分发（Coinbase Wallet / Farcaster 社交图谱）。**

**机制**（bags.fm/how-it-works、docs.bags.fm）
- 任何人可为一个人/梗/想法发 SPL 代币（"Token Launch v2"）。
- **创作者拿全部交易量的 1% 永续版税**，不是一次性发射费。
- 发射时可配置最多 **100 个钱包**的版税分账。
- 两段流动性：bonding curve / pre-migration 阶段（抽成更高，最大化早期收入）→ 迁移进 **DAMM v2（Meteora dynamic AMM）**池，后续费率更低。
- 不开 fee compounding 时大致协议/创作者 **50/50**；开启 compounding（**25–50% 手续费回灌池内流动性**）时，剩余部分再对半分。
- **"Get Bagged"**：如果代币是**未经本人同意**用某个真人的名字/形象发的，该真人事后可以验证社交账号所有权，把**已经累积的版税流重定向到自己钱包**（版税按身份托管，可追溯领取）。

**争议（多来源一致）**："Get Bagged" 既是功能也是争议——它明确允许**任何人用他人身份发币**、积累真实版税，再拿着「免费的钱」去引诱当事人公开互动/背书，从而在名人下场时进一步拉盘、早期持有者出货。有报道提到创始团队此前与曾受 FTC 关注的「欺骗性变现」类 App 有关联。**没有找到单一的、有名有姓的司法案例**——这是系统性/结构性批评，不是某一次头条事件（**待核实**）。

**规模纠偏**：「Bags 在 2025-08 超越 pump.fun / 日收入 $1M」的说法**不成立**——流传的 $1M 是**创作者在平台上募到的资金**，不是协议收入。按 30 天手续费看，Bags 甚至在 Solana 内部都不是龙头。但 `[一手]` 数据显示 Bags 是**本表里衰减最慢的**（近 30 天 $1.81M，且 2026-01-16 还创下 $4.80M 的单日峰值）。形态同样是**脉冲式**（依赖个别币暴涨，如 GAS 涨约 700% 带动平台活跃度）。

**判定**：**存活，但声誉受损、生态位边缘**。分析价值在于：**证明「创作者分成」可移植，也证明这套机制在缺乏「同意门禁」时会被滥用。** → Mantle 若做创作者分成，**必须内置「身份同意」环节**（对 mStocks 这种涉及真实公司/资产的标的尤其致命）。

### 4.2 Believe（believe.app，原 Clout）—— ⚠️ **链 = Solana**

**为什么放进来**：它是「社交图谱做冷启动」最极端的对照——社交层**不在链上、也不是 Farcaster**，而是**直接寄生在 X（Twitter）**上。这直接反驳了「Farcaster 的链上社交图谱是结构性护城河」的说法。

**时间线与机制**
- 原名 **Clout**，**2025-04 更名 Believe**，转向「Internet Capital Markets」叙事（任何一条推文/回复都能变成可投资资产）。
- 原生代币 **PASTERNAK 于 2025-05 更名 LAUNCHCOIN**。
- 机制：在某条推文下回复并 @**@launchcoin**，bot 就为该帖/该账号部署一枚 Solana 代币；部分手续费路由给触发发射的创始人/发帖者。

**数据（有日期）**
- **峰值：2025 年 5 月中**。周协议收入从 2025-05 初的约 **$30,500** 暴涨到 5 月第二周的 **超过 $400 万**（约 130 倍，1–2 周内）。LAUNCHCOIN 市值峰值 **>$2.5 亿**。
- **2025-06**：周收入从 5 月峰值**下跌约 94%**（热度冷却 + 2025-06-08 的反滥用更新打击了刷量者）。
- **终局：2025-10** 宣布「代币升级」，LAUNCHCOIN 迁移到新代币 **BELIEVE**，兑换窗口 **2025-10-29** 关闭。公告当天 LAUNCHCOIN 大跌，部分持有人视其为事实上的软 rug。
- 2026 年有报道称项目因「误导投资者、未履行回购承诺」面临法律审查（**待核实**）。
- `[一手]`：累计手续费 **$40,910,390**，**近 30 天仅 $214**。

**判定**：**已实质死亡**。教训极其清晰：**纯靠病毒回路、没有收入地板的 launchpad，热度一冷立刻归零，而代币「升级/迁移」是信任的终点。**

### 4.3 Virtuals Protocol —— 链 = **Base**（+ Solana）

**机制（工程级）**
- **bonding curve → 毕业**：每个 AI agent 代币在内部 bonding curve 上以 **$VIRTUAL** 计价发行。curve 内累积到 **42,000 $VIRTUAL** 的流动性时，自动触发「毕业」。
- **迁移**：毕业时把累积的 $VIRTUAL + agent 代币迁进 Base 上的 **Uniswap V2** 池（agent-token / $VIRTUAL 对）。
- **LP 锁定：LP token 被质押并锁 10 年**（协议强制，防抽池）；LP 仓位名义所有权仍归创作者钱包，但底层流动性锁死。
- **生命周期分层**：**Prototype**（毕业前，仍在 curve 上）→ **Sentient**（毕业后，进入开放市场）。
- **交易税 1%**：毕业前全额进协议金库；毕业后拆分去**支付 agent 自身的推理/GPU 成本**、创作者激励、affiliate 奖励。
  > 注意：这与 Clanker Droids（§1.7）是同一个思路的两代实现——**代币现金流反哺 agent 运营成本**。Virtuals 是把它做在协议层，Clanker 是做在 reward slot 层（更灵活、更可组合）。
- **Agent Commerce Protocol (ACP)**：位于代币化层之上的商业/协调层，让自治 agent 之间订立服务协议、链上可验证结算 —— 让毕业后的 agent 代币成为**生产性经济主体**，而非纯投机品。这是本专题里**唯一有真实「发射后效用」叙事**的项目。
- **Genesis Launches / 积分制配额**：旗舰级 agent 发射改用「质押 $VIRTUAL 累积积分 → 积分决定 Genesis 认购配额」，而非纯无许可 bonding curve —— 一层反 bot / 守门机制。
- **$VIRTUAL 代币学**：固定供应 **10 亿**。$VIRTUAL 是**每一笔 agent 代币交易、每一个新 agent 的 curve 流动性的强制基础/路由货币** → 结构性买压。配合用税收资助的 **buyback-and-burn**（针对表现好的 agent 代币，**不是**直接销毁 $VIRTUAL 本身——这个细节常被误传）。

**数据 `[一手]`**（CoinGecko + DefiLlama，2026-09-06）
- VIRTUAL 价格 **$0.6833**，市值 **$449.76M**，FDV $683.12M，**ATH $5.07（2025-01-01）**，距 ATH **−86.5%**，近 1 年 −38.3%
- 累计手续费 **$73,849,436**；近 30 天 $665,753；近 24h $11,202
- **峰值单日 2025-01-02 $1,594,093；峰值月 2025-01 $16,585,887**
- 逐月（USD，节选）：2024-12 **$14.68M** → 2025-01 **$16.59M** → 2025-02 $2.18M → 2025-05 $5.26M → 2025-10 $5.08M → 2026-07 $2.23M → 2026-08 $0.64M
  → 形态与 Clanker 类似：**两波（2024-12/2025-01 与 2025-05/2025-10），2026 年仍有余温**，是 Base 系里衰减最慢的 launchpad。
- 「2025-01 峰值市值约 $50 亿」的说法**无法证实**，应视为夸大（**待核实**）；价格 ATH 是确认的。

**判定**：**存活，已过投机峰值，但结构上是 Base 系最成熟的一家**——唯一拥有「发射后真实用途（ACP）」的项目。

### 4.4 Doppler / Whetstone Research —— 「plumbing 层」的赢家

**公司/产品区分**：**Whetstone Research** 是团队/公司，**Doppler** 是协议（一个链上发射基础设施原语，**本身不是面向 C 端的 launchpad**，由 App 建在其上）。

**核心机制：Dutch-auction Dynamic Bonding Curve，实现为 Uniswap v4 hook**
- bonding curve 的定价逻辑跑在 v4 池的 `beforeSwap` 里 —— **没有独立的托管合约持有资金，价格发现原生发生在 AMM 内部**。
- 关键参数：
  - `numTokensToSell`：本次出售的代币总量
  - `duration = endingTime − startingTime`：拍卖总时长
  - `epochLength`：单个 epoch 秒数；协议在每个新 epoch 的**首笔 swap 之前**重新平衡曲线，epoch 长度同时决定流动性 "slug" 的放置位置
  - `gamma (γ)`：陡度/增长参数，`tickUpper = tickLower + gamma`；不设时协议可自动算「最优 γ」
  - `tickSpacing`：v4 强制的 tick 粒度 —— **这就是价格衰减是离散阶梯而非连续曲线的原因**
- 动力学：跟踪一个 `tickAccumulator`，随时间移动原点 tick τ_t：`bc(t) = γ·(t/t_max) + τ_t`，价格 `p(t) = 1.0001^bc(t)`（标准 Uniswap tick↔price）。
- **未售出供应导致价格下移（Dutch auction 的精髓）**：如果实际销量**落后于**预设的销售曲线（`numTokensToSell` / `duration`），bonding curve **每个 epoch 向下平移**一个「脉冲」，必须由真实买压来抵消 —— 即**未售库存会持续压低价格直到出清**，而不是静静挂着。
  > **这对狙击者是致命的**：操纵曲线要花钱，因为曲线会持续向操纵者的反方向移动。这是与「高费衰减」完全不同的第三条抗狙击路线：**用供需机制而非税收惩罚**。
- **流动性 "slug"（三类仓位，围绕当前价格放置）**：
  - **lower slug**：让持有者能卖回给曲线
  - **upper slug**：按本 epoch 预期销量定尺，吸收买单
  - **price-discovery slug**（一个或多个，数量由接入方配置）：放在 upper slug 之上

**Airlock 架构**
- **`Airlock`** 是中心工厂/编排合约 —— 接入方唯一需要对接的入口。
- 它协调模块化子工厂：**Token Factory**（部署标准化 ERC-20 字节码，避免恶意代币实现）、**Liquidity / Bonding-Curve 模块**（跑上面的荷兰拍曲线）、**Migration Factory / Migrator**（销售目标达成后把流动性迁进永久 AMM 仓位，v2 或 v4）、**Timelock / Vesting 模块**（用解锁计划阻止发行方售后 rug）。
- **治理**：文档里称为 "no-op" / 极简治理（**DopplerDN**）—— 协议刻意接近不可变/参数化而非 DAO 治理，减小攻击面、彰显可信中立（与 Uniswap 核心合约同一哲学）。
- **hook flag 用法**：`afterInitialize`（放置初始 curve 仓位）、`beforeSwap`（若落后进度则重平衡曲线）、`afterSwap`（记录累计售出/收款，排除手续费）、**`beforeAddLiquidity`（revert 掉所有外部加流动性 —— bootstrap 期间池子是「无第三方 LP」的）**。

**谁在用**
- **Zora**：用 Doppler 配置/优化 Coins 的**初始多曲线流动性**（官方文档明确：*"Deposits initial liquidity using Doppler protocol (doppler.lol) for optimized multi-curve positioning"*），并且 **Doppler 直接分走每笔交易的 1%（总费的 1%，market rewards 的 1.25%）**。
- **Pure Markets**：Unichain 上第一个基于 Doppler hook 拍卖的发射前端。
- **Bankr** 已把发射基础设施迁到 Doppler 上运行（→ 直接证据：Doppler 正在赢下「管道层」）。
- **Paragraph、FxHash、Base App** 也被列为接入方/合作方。
- 部署链：**Base、Ink、Unichain**。（「Doppler 用在 Ronin」的说法**查无实据，应视为错误**）

**融资**：pre-seed **$1.3M**（Variant 领投，2025，Uniswap Ventures 参与）；**seed $9M**（**Pantera Capital 领投**，Variant / Figment Capital / Coinbase Ventures 跟投，**2026-01** 官宣），定位「成为链上资产发射的默认基础设施」。
**与 Uniswap Foundation 的关系**：建**在** v4 之上、有 Uniswap Ventures 早期投资、与 Unichain 生态叙事紧密，但**不是** Uniswap Foundation 的直接 grant 受助方 —— 不要把两者混为一谈（**待核实**已澄清）。

**判定**：**本专题里唯一「活着且在增长」的一家**。刚拿 $9M（2026-01），有真实接入（Zora / Bankr / Pure Markets），把自己定位成**中立的协议级管道**而不是又一个竞争性的 C 端 launchpad —— **意味着无论哪个月哪个前端品牌流行，它都能赢。**

> **这是给 Mantle 最重要的战略参照：不要做第 N 个 launchpad 前端，要做 Mantle 上所有 launchpad 的共同底层（Doppler / Flaunch RevenueManager 的定位）。**

### 4.5 Wow.xyz（Zora 出品）—— 链 = Base，已被自家产品取代

**已核实**：Wow.xyz 确实是 Zora 团队做的 Base memecoin launchpad，用 **bonding curve → Uniswap v3 毕业**模型，基于 Zora 的 **ERC-20z** 标准做生命周期切换。

**机制**
- 新币初始 **8 亿**供应在内部 bonding curve 上交易（合约作自动对手方：买推价涨、卖推价跌）。
- **毕业阈值：卖出 8 亿枚代币**（约 **$69,000 市值** —— 就是那个著名的「69k」梗数字）。
- 毕业时：**额外铸造 2 亿枚**（总供应达到 **10 亿**），并从 bonding curve 池里抽出**约 $12,000 等值 ETH** 存入 Base 上的 **Uniswap v3 池**做初始流动性；**产生的 LP token 被销毁**，永久锁定流动性。
- **费用模型：每笔买/卖 1% 协议费（ETH 计价）**，拆分为 **50% 代币创作者 / 20% 协议 / 15% 平台 / 15% 订单推荐人**。
- **纠正一个常见误记**：毕业阈值是 **800M 代币 / 约 $69K 市值**，**不是**「4 ETH」。「4 ETH」在现行文档中查无实据。

**兴衰与关系**：Wow.xyz 是 Zora 早期的独立 memecoin 发射实验（团队做的另一个品牌），**早于** Zora 后来的第一方 Coins/Creator Coin 产品；后者直接在 Zora 主 App 内吸收并超越了 Wow 模型（而 Coins 本身又转去依赖 Doppler 做流动性引导，见 §4.4）。**判定：已被取代/休眠**，机制活在 Zora 主线产品里，2026 年不再作为独立品牌运营。

> **Zora 的产品路径本身就是一条完整的教训线**：`Wow.xyz（bonding curve + 毕业）` → `Zora Coins（无 curve、v4 hook、多曲线）` → `Creator Coin（profile 币做配对资产）` → `Content Coin（每 post 一币）` → `Trend Coin（0.01% 费、无创作者分配、纯高频）`。
> 注意最后一步：**Trend Coin 已经完全放弃了「创作者经济」——没有创作者分配、没有推荐分成、手续费 100% 归协议、费率砍到 1 bps。这本质上是承认「高频投机」才是真实需求，而创作者分成不是。** 这是 §6 结论的一个强证据。

### 4.6 简要条目

**Aerodrome Finance（Base）—— 流动性场所，不是 launchpad**
- Base 的主导 DEX / **ve(3,3)** 流动性中枢：锁 AERO → veAERO → 投票决定每周 AERO 排放去哪个池（bribe 飞轮）。
- 有一个真实的发射工具 **"Aero Launch"**（无许可地引导池子/价格区间/锁定流动性）→ 它同时是结算层**和**一个次要 launchpad 竞争者。
- 现实中，无论 meme/creator 代币从哪个 launchpad（Zora / Virtuals / Streme）发出来，**最终流动性大多迁进 Aerodrome 池** → 它是共享管道，不是独立玩法。
- `[一手]`（2026-09-06）：AERO $0.5493，市值 **$542.04M**，ATH $2.32（2024-12-07），距 ATH −76.3%；Base 上 Aerodrome TVL $334.96M、24h 手续费 $213,442（对比 Uniswap on Base：TVL $423.27M、24h 手续费 $404,937）。

**Bankr（Base，@bankrbot on Farcaster/X）—— Base 上最活跃的发射接口**
- AI / 自然语言交易 + 发币 agent：用户用大白话命令 @bankrbot 在 Base 上部署或交易代币。
- **反狙击设计**：swap 费从**约 80% 起、约 10 秒衰减**（与 Zora 狙击税同族）。
- 常态 swap 费约 **0.7–1.2%**（视接入方），在创作者 / 协议 / 流动性锁定资金之间分账；部署即自动锁流动性。
- **已把发射基础设施迁到 Doppler 上运行**。
- `[一手]`：累计手续费 **$33,344,633**，**近 30 天 $1,172,755**，24h $10,985 —— **是 Base 上所有 meme 相关产品里近 30 天现金流最高的一个，超过 Clanker（$283K）+ Zora（$15K）+ Flaunch（$390）之和的 3.9 倍。**
  > **这个事实本身就是结论**：Base 上真正在赚钱的不是「发射协议」，而是**用户接触点（interface / agent）**。价值在分发层，不在协议层。

**Streme.fun（Base）**
- Farcaster 上 `@streme` 触发的 AI agent launchpad，部署 **Superfluid "Super ERC-20"** 代币，自带**逐秒实时流式奖励**（而非静态 claim）。
- 部署时：**80% 供应立刻做 Uniswap v3 池；剩余 20% 进质押合约**，奖励通过 Superfluid agreement 连续流出。
- 费用模型：**40% 交易费回流给代币创作者**。
- 规模很小（白名单准入、hackathon 阶段），机制有意思但不是主要玩家。

**Noice（Base，Farcaster mini-app）—— 社交打赏，不算 launchpad**
- 把 Farcaster 社交动作（like / recast / comment）自动转成链上 **USDC 微额打赏**，是直接打赏工具而不是发射场所。
- 试过创作者专属代币做互动门禁，但那是次要功能。归入本表仅为完整性。

**Moonwell（Base）—— 借贷，不是 launchpad**
- 非托管借贷市场（WELL 治理代币），位于 launchpad 活动的**下游**（交易者用发射出来的代币做杠杆/抵押），不争夺发射量。**不属于 launchpad 类目**，此处仅按要求列出作流动性场所背景。

**pump.fun 登陆 Base —— 确有其事，且是 2026 年最新变量**
- pump.fun（Solana 霸主，累计手续费 $1.21B `[一手]`）通过 **2026-05-26** 的 "frictionless multi-chain trading" 升级**扩张到 Base**（以及 Ethereum、BNB Chain），此前在 2026-03 已有子域名注册/品牌改动等信号。
- 采用无 gas UX + 自动多链钱包生成，**结算主要仍走 Solana 后端轨道** —— 即 **Base 变成 pump.fun 的又一个前端界面，而不是真正的 Base 原生部署**。
- **这是 Base launchpad 战争最新、也最讽刺的一笔**：在 Base 官方放弃 creator coin 路线（2026-07）之前两个月，Solana 的 launchpad 龙头反向把 Base 收成了自己的一个分发渠道。
- 另注 `[一手]`：**pump.fun Mobile App 是全表唯一逆势增长的产品**（累计 $8.73M，但近 30 天 $6.06M —— 即 69% 的累计收入发生在最近 30 天）。**移动端原生分发 > 链上社交图谱分发**，这是 2026 年最强的一条信号。

**Uniswap 自己的 launchpad / Unichain 上的 Doppler**
- Uniswap Labs / Foundation **没有**自营 C 端 launchpad；生态打法就是 §4.4 的 Doppler-on-Unichain（旗舰前端 Pure Markets）。不存在独立的「Uniswap 品牌 launchpad」。

**Zora 在 Base 上的「竞争者」核查结果**
- **Sound.xyz：已确认关停**，**2026-01-16** 完全下线；团队转做新产品 **Vault**；已有 NFT 保留在链上，艺术家仍可通过 Splits 合约提取资金。
- **Party（party.app）、Rodeo、Phi**：**无法证实**它们在 2025–2026 年是仍在运营、有实质活跃度的 Base 创作者代币平台。按「查无实据即丢弃」处理。
- **「Buttery」**：**不存在**这个 Base launchpad（该词只作为 UX 营销用语或不相关项目 "Butter Network" 出现）。已丢弃。

---

## 5. Base 本身的策略：分发 + 基础设施

### 5.1 Base App（Coinbase Wallet 更名）

**发布**：**2025-07-16**，"A New Day One" 直播（anewdayone.xyz）。官方公告 <https://blog.base.org/a-new-day-one>

**打包了什么**
- Coinbase Wallet → **Base app**：社交流 + 交易 + 聊天 + mini apps + 支付，一个 App。
- **社交流跑在 Farcaster 协议上**。官方原话：*"The new social feed in the Base app is powered by Farcaster, which means creators can own their content and earn directly from their success... **Every post is a coin that can be bought, powered by Zora.**"*
- **创作者付款**：*"Top creators who post and engage with the community on the Base app will earn weekly rewards for a limited time... Payouts occur weekly directly in the app."*
- **Mini apps**：*"Hundreds of mini apps are available on day one"*（Remix、Noice、Decentralized Pictures 等），通过 base.dev / Base Build 面板变现。
- **聊天**：XMTP 驱动，端到端加密，集成 AI agent（**Bankr**、Mamo）。
- **Base Account**：通用链上身份/智能钱包（"Sign in with Base"）。
- **Base Pay**：USDC 极速结算；**Shopify 合作**（2025-06 宣布）把 USDC 支付带给数百万商户；美国用户 1% 返现（计划）。
- **USDC 奖励**：最高 **4.1% APY**（仅美国，EU/加拿大不可用）。
- **Flashblocks 同日主网上线**：*"reducing effective block times from 2 seconds to just 200 milliseconds, making Base Chain 10x faster."*

**推出节奏**
- 2025-07-16：base.app 候补名单 beta 开放
- **2025-12-18**：beta 结束，开放到 **140+ 国家**（二手来源，**待核实**）

**用量数字（关键：官方从未披露 Base App 的 MAU/DAU）**
- 截至本次研究（2026-09-06），**Coinbase 没有发布过 Base App 自身的 MAU/DAU**。公开报道在 2025-12 GA 时只说「数十万用户」（二手估计，不是 Coinbase KPI）。
- Base **链**（与 App 不同）`[一手]`（DefiLlama，2026-09-06）：**24h 活跃地址 238,687**、24h 新增地址 41,532、**24h 交易 859 万笔**。2026 年初有「约 372,000 日活地址」的二手估计。

**Creator Rewards 项目的完整生命周期（最重要的单一数据点）**
- 2025-07 随 Base App 重启上线。
- **2026-02-18 停止**。
- 约 7 个月共发出 **约 $450,000 给约 17,000 名创作者** → **人均约 $26，月均总额约 $64K**。
- Jesse Pollak 在关停时称 Base App 是一个「imperfect」的 Farcaster 客户端，产品需要重新聚焦交易。
- **2026-09** 新推「Creator Grant Program」——**传统现金 grant，不再是基于互动的代币奖励**，明确标志路线转向。

> **量级对比（这一条极具说服力）**：Base 官方给创作者的**全部**直接补贴 = **$450,000**。同期 Clanker 一家在 **2026 年 2 月单月**产生的手续费 = **$22,394,092**，是前者的 **49.8 倍**。
> → **官方补贴从来不是 Base 创作者经济的驱动力；投机是。** 一个靠投机驱动的「创作者经济」，在投机退潮时不会剩下创作者。

### 5.2 Coinbase 分发漏斗：「1 亿验证用户」这个数字必须打折

- 「100 million verified users」源自 Coinbase **2021 年 IPO** 注册材料，并在其后到约 2023 年的文件/营销中反复引用。
- **定义问题**：「verified user」= 任何验证过邮箱/手机号的人，**包含不活跃与重复账户**，不是独立活跃用户。
- Coinbase **约在 2023 年停止报告该指标**，改用 "monthly transacting users"（MTUs）。
- **2025-05**：有报道称 SEC 正在调查 Coinbase 是否在 2021 年 IPO 披露中错误陈述用户数；Coinbase CLO Paul Grewal 表示该指标早已自愿停用。
- **结论：不存在一个新的、与 Base App 挂钩的「1 亿+」数字。任何「Base App 通过 Coinbase 漏斗触达 1 亿人」的说法都是营销框架，不是可核实统计。**
  → **这直接削弱了「Coinbase 分发是 Base 的决定性优势」这一命题的可量化基础。** 真实可核实的分发结果就是上面的 238,687 日活地址与 $450,000 创作者补贴。

### 5.3 Flashblocks：200ms 预确认

**机制**
- Base 的**规范区块时间仍是 2 秒**。Flashblocks 由 block builder 在这 2 秒窗口内**流式推送 10 个 200ms 的 sub-block**。
- 应用因此获得近实时的「预确认」包含信号，**底层 2 秒终局性不变**。
- 建于 **Flashbots 的 Rollup-Boost** 之上（OP Stack 的模块化出块 sidecar）；使用 **TEE（Flashtestations）**做更快、可验证的出块。
- **今天 Base 的每个区块默认都由 Flashblocks builder 构建**（不是 opt-in）；RPC 方法仍可用于取标准 2 秒终局性。
- **对 MEV/交易的影响（对 Mantle 很关键）**：把 MEV 竞争从「gas 费喊价拍卖（bot 灌 mempool）」转向**延迟竞赛**——谁跟 sequencer 的连接更快、延迟更低，谁就在每个 200ms 窗口内取得排序优势。内建的 revert protection 降低了交易者/bot 的失败交易摩擦。

**时间线（官方）**
| 日期 | 事件 |
|---|---|
| **2025-02-27** | Flashblocks 上 **Base Sepolia 测试网**，有效确认从 2s → 200ms，观测到**交易包含时间最多降低 10 倍**。<https://blog.base.org/building-for-the-long-term-making-base-faster-simpler-and-more-powerful>；主网原定 Q2 2025 |
| **2025-07-16** | Flashblocks **主网上线**（与 Base App 同日）。<https://blog.base.dev/flashblocks-deep-dive>、<https://blog.base.dev/accelerating-base-with-flashblocks> |
| **2026 年下半年** | 内部提案 **"Denim"** 讨论中：把每个 200ms 区间变成**规范区块**（退役当前的 Flashblocks 预确认层，改成原生 200ms 出块）。截至 **2026-09** 仍是 request-for-feedback，**没有确定时间表**。<https://docs.base.org/upgrades/denim/migrate-from-flashblocks> |

**实测效果**：Flashblocks 上线后峰值 TPS **约 1,267**（理论上限约 1,429）。

### 5.4 Gas limit / 吞吐量扩容路线

**「北极星」目标：1 Ggas/s**（官方，base.dev 工程博客 2025-02-07：<https://blog.base.dev/scaling-base-in-2025>）

| 时点 | 里程碑 |
|---|---|
| 2023-08 上线 → **2025-02** | Base 达 **24 Mgas/s**（「接近一年前的 10 倍」）。官方原话：*"Our 2025 goal requires us to go exponentially further to scale throughput more than 10x beyond where we are today, and 100x the scale of Base when we initially launched."* |
| 2025-02 宣布 | 2025 年底目标 **250 Mgas/s**（北极星 1 Ggas/s） |
| — | **客户端瓶颈**：`op-geth` 出块速度软上限约 **40–50 Mgas/s**；**Reth**（Paradigm）实测 p999 出块速度提升约 70%，上限约 2–3 倍（约 100 Mgas/s），低状态访问负载下最高可达约 **500 Mgas/s** |
| — | **Fault Proof System (FPS)**：原系统支持约 60 Mgas/s 上限（30 Mgas/s target）；MT+64 Cannon 升级后约 100 Mgas/s 上限（约 50 Mgas/s target） |
| **2025 H1** | 近乎每周提升 gas limit；中位手续费从约 **$0.30/笔**降到不到 1 美分 |
| **2025-06** | 持续跑到 **约 1,500 TPS**，中位手续费 < 5 美分。<https://blog.base.dev/scaling-base-sustain-10x-growth> |
| 2025 Q2 中 | 暂停提升，专注 Reth 迁移 / TrieDB / 稳定性 |
| **2025-10-28** | 官方博客「30 天内翻倍」：Base 在 **75 Mgas/s**，Q4 2025 目标翻倍到 **150 Mgas/s**；Reth 迁移「已使提升到至少 150 Mgas/s 变得安全」，认为 **2026 年初可达 400–500 Mgas/s**。<https://blog.base.dev/scaling-base-doubling-capacity-in-30-days> |
| **2025-11-06** | gas limit 提到 **125 Mgas/s**，年底目标 150 Mgas/s |

**以太坊 L1 DA（blob）配套**：Pectra 硬分叉（2025-05-07）把 blob target/limit 翻倍（3/6 → 6/9）；Fusaka 硬分叉 **2025-12-03**，随后 Blob-Parameter-Only 分叉 **2025-12-09**（target 10 / limit 15）与 **2026-01-07**（target 14 / limit 21）—— 预计到 2026 年初生态 blob 容量翻倍以上。

**吞吐记录**
- **历史最高单日交易量：2026-06-05，20,767,612 笔**（basescan.org）
- Flashblocks 后峰值 TPS 约 **1,267**（理论上限约 1,429）
- 实时（2026-09-06）`[一手]`：24h **859 万笔**交易，24h 活跃地址 **238,687**

### 5.5 Base vs Solana：硬数据对照

**全部 `[一手]`（DefiLlama API / 页面，2026-09-06）**

| 指标（2026-09-06） | **Base** | **Solana** | Base / Solana |
|---|---|---|---|
| TVL | $5.669B | $5.925B | **96%** |
| 稳定币市值 | $4.991B（USDC 占 85.06%） | $16.367B（USDC 占 44.63%） | 30% |
| 链手续费（24h） | $91,115 | $381,125 | 24% |
| 链收入（24h） | $90,861 | $73,212 | **124%** |
| **应用层手续费（24h）** | **$1.48M** | **$9.06M** | **16%** |
| **应用层手续费（30d）** | **$51.50M** | **$339.63M** | **15%** |
| **DEX 量（24h）** | **$567.22M** | **$1.961B** | **29%** |
| **DEX 量（30d）** | **$24.28B** | **$65.03B** | **37%** |
| DEX 量（历史累计） | $830.02B | $3,084.06B | 27% |
| DEX vs CEX 占比 | 2.74% | 8.38% | 33% |
| 永续量（24h / 7d） | $146.1M / $1.439B | $551.07M / $9.616B | 27% / 15% |
| 活跃地址（24h） | 238,687 | 2.03M | **12%** |
| 交易数（24h） | 8.59M | 77.57M | **11%** |
| 跨链/原生 TVL | $14.076B（bridged） | $28.82B（native） | 49% |
| 单日 DEX 量峰值 | **2025-10-10 $3.02B** | **2025-01-18 $38.18B** | 8% |
| 主要 DEX | Uniswap、Aerodrome | Pump、Raydium、Meteora、Jupiter | — |

**逐月 DEX 量（USD 百万）** `[一手]`

| 月份 | Base | Solana | Base 占比 |
|---|---|---|---|
| 2024-11 | 35,598 | 158,522 | 22% |
| 2024-12 | 48,325 | 135,280 | 36% |
| **2025-01** | **49,997** | **313,913** | 16% |
| 2025-02 | 29,603 | 149,145 | 20% |
| 2025-03 | 15,508 | 80,037 | 19% |
| 2025-06 | 24,834 | 86,218 | 29% |
| 2025-08 | 47,654 | 134,826 | 35% |
| **2025-10** | **53,393** | 168,437 | **32%** |
| 2025-12 | 27,405 | 109,518 | 25% |
| 2026-02 | 27,918 | 108,888 | 26% |
| 2026-05 | 29,166 | 57,658 | **51%** |
| 2026-06 | 32,001 | 67,107 | **48%** |
| 2026-07 | 22,670 | 53,139 | 43% |
| **2026-08** | **23,851** | **64,033** | **37%** |

> **读法**：Base 的 DEX 量绝对值在 2025-10 见顶（$53.4B/月），之后在 $23–32B/月区间横盘；Solana 从 2025-01 的 $313.9B 掉到 2026-08 的 $64.0B（−80%）。**Base 的相对份额从 2025-01 的 16% 升到 2026 年的 37–51%，但这是 Solana 掉得更快造成的相对改善，不是 Base 的绝对增长。**

**逐月应用层手续费（USD）** `[一手]`
| 月份 | Base | Solana | Base 占比 |
|---|---|---|---|
| 2025-01 | 141,703,587 | 1,657,190,097 | 8.5% |
| 2025-07 | 68,998,773 | 625,897,405 | 11.0% |
| **2025-10** | **124,750,178** | 484,287,582 | 25.8% |
| **2026-02** | **113,403,952** | 265,164,625 | **42.8%** |
| 2026-08 | 48,594,273 | 334,385,100 | 14.5% |

**Base launchpad 在 Base 自己体内的占比** `[一手]`（关键指标）
| 月份 | Clanker 手续费 | Base 应用层总手续费 | Clanker 占比 |
|---|---|---|---|
| 2025-01 | $5,868,495 | $141,703,587 | 4.1% |
| **2026-02** | **$22,394,092** | **$113,403,952** | **19.7%** |
| 2026-08 | $242,379 | $48,594,273 | **0.50%** |

加上 Zora Coins + Flaunch，**2026-08 三家 Base 原生 launchpad 合计 $257,430，占 Base 应用层手续费的 0.53%** → **Base 的 launchpad 赛道在 Base 自己体内已经从 20% 的支柱变成 0.5% 的边角料。**

### 5.6 无代币立场、sequencer 收入、Coinbase 财报

- Base **没有原生代币**，这是长期刻意立场（不同于 Optimism 的 OP、Arbitrum 的 ARB）。与 Coinbase 的合规姿态一致，把 Base 定位为「开放基础设施」而非投机资产。
- Coinbase **不在正式财报中单列「Base sequencer 收入」**，它被合并进 "other" transaction revenue。
- **2025 Q3**：Base sequencer 收入约 **$68M**（分析师二手估计，非 Coinbase 官方标注）。
- **2026 Q2 财报**：Coinbase "other" transaction revenue **环比下降 11% 至 $47.4M**，公司把降幅「主要」归因于 **Base 收入下降**。
- Coinbase 的公开逻辑：把手续费压到 1 美分以下是刻意取舍，用交易量/采用换直接抽成，把价值导向 USDC 与更广生态。Base sequencer 收益曾被转给 Coinbase 母公司用于「安全与监督」，部分再投入以太坊生态。
- 精确季度数字建议查 investor.coinbase.com 的股东信/财报 deck，交叉 Dune 的 sequencer profit 看板。

### 5.7 生态冷启动杠杆

- **Base Batches**：全球 builder 孵化器 —— $10K 启动 grant + 导师 + 通过 **Base Ecosystem Fund** 追加最高 $50K 投资（常有 Coinbase Ventures 参与），以 Demo Day 收尾。
- **Base Builder Rewards**：面向持续交付链上基础设施/工作的 ETH 计价激励。
- **Base Builder Grants**：对已上线项目的追溯性 ETH 资助（奖励「已发布」而非「提案」）。
- **Onchain Summer**：与主网首发同步的多周社区活动，核心是 **Onchain Summer Buildathon**（大型全球黑客松）。
- **Base Build / base.dev 面板**：2025-07-16 随 Base App 发布，给 builder 提供「构建、增长、变现 mini app」的工具。

---

## 6. 核心结论

### 6.1 原命题成立：Base 确实把创作者经济/社交图谱当作发射的冷启动来源

**证据链（全部有据可查）**

1. **官方战略宣言（2025-07-16，blog.base.org）**
   > *"We believe the next chapter of the internet won't come from big platforms. It'll come from creators. That's why we rebuilt the Base app from the ground up, not just as a wallet, but as a new kind of open social network."*
   > *"**Every post is a coin that can be bought, powered by Zora.** Creators can earn with no follower minimum, brand deals, or geo restrictions."*

2. **产品层面把「社交图谱」写进了协议**
   - Clanker：**@clanker 一条 Farcaster mention 即发币**；创作者奖励**默认发到其 Farcaster verified address / custody address** → 变现原生绑定社交身份，不是另开 connect-wallet 流程。
   - Zora：**profile → Creator Coin（ticker = $username）**，**post → Content Coin**，且 **Content Coin 的配对资产就是创作者自己的 Creator Coin** → 一个创作者的所有内容币的价值**在协议层强制回流到他的个人币**。这是「社交图谱即资产结构」的最激进实现。
   - Base App：把 Farcaster 社交流 + Zora 发币 + 交易 + 支付**装进同一个 App**。

3. **数量级证据：冷启动确实奏效**
   - Base App 上线（2025-07-16）后 11 天：Zora **单日新建币约 54,000 枚**（vs 6 月 <5,000/日，约 **11 倍**）；日活交易钱包从 <5,000 到 **>50,000**；日成交峰值约 **$41M**；月协议收入从约 $600K 跳到约 **$2M**（>3 倍）。
   - `[一手]` Zora Coins 月手续费：2025-06 **$52,807** → 2025-07 **$2,348,233**（**44.5 倍**）→ 2025-08 **$2,510,233**。
   - `[一手]` Zora Coins 月 DEX 量：2025-06 **$0.0M** → 2025-07 **$101.8M** → 2025-08 **$113.0M**。
   → **这是全行业最干净的一次「把交易所级分发接上发币协议」的 A/B 实验，结论是：分发确实能在几周内把发射量拉高 1–2 个数量级。**

4. **Clanker 也是同一个故事**：2024-11-08 从 Farcaster bot 起步，靠社交图谱在 5 个月内做到 200,000+ 枚代币、$2.7B 交易量、$27M 手续费（The Block，2025-04-04），**累计 737,210 枚代币** `[一手]`。

### 6.2 但命题的后半段失败了：分发能买到冷启动，买不到留存

| 指标 | 峰值 | 现值（2026-08/09） | 变化 |
|---|---|---|---|
| Zora 日新建币 | 约 54,000（2025-07-27） | **422** `[一手]` | **−99.2%** |
| Zora 月 DEX 量 | $113.0M（2025-08） | $0.5M | **−99.6%** |
| Zora 月手续费 | $2.51M（2025-08） | $14,768 | **−99.4%** |
| Zora 新建 Creator Coin | — | **0（24h 内 422 枚全是 Content）** `[一手]` | 实质停更 |
| Clanker 月手续费 | $22.39M（2026-02） | $242,379 | **−98.9%** |
| Clanker 每枚币手续费 | $1,371（2024-12） | **$53** | **−96.1%** |
| Flaunch 月手续费 | $1.12M（2025-01） | $283 | **−99.97%** |
| ZORA 代币 | ATH $0.1456（2025-08-11） | $0.0086 | **−94.1%** |
| FLAY 代币 | ATH $0.2573（2025-01-31） | $0.0044 | **−98.3%** |
| Base 三大原生 launchpad 占 Base 应用层手续费 | 19.7%（2026-02，仅 Clanker） | **0.53%** | — |
| Base App Creator Rewards | 运行中 | **2026-02-18 关停**（7 个月 $450K / 17,000 人，人均 $26） | 终止 |

### 6.3 Base 官方自己已经宣告这条路线失败（这是 2026 年最重要的事实）

**2026-07-15/16，Jesse Pollak 交出 Base App 给 Cobie（Jordan Fish）**，并在 X 上写道（<https://x.com/jessepollak/status/2077427261586997745>，经 <https://thedefiant.io/news/people/base-creator-jesse-pollak-hands-app-to-cobie-says-social-bet-was-definitively-wrong> 报道，2026-07-16）：

> *"[我做了一个] two pronged bet：builder 会驱动下一波 crypto 采用；采用会来自链上原生的社交体验。第一个赌对了，第二个赌错了。"*
> *"**the entire social side of the market that many of us had been building towards - farcaster, zora, miniapps, and yes, creator coins - disintegrated completely… i was definitively wrong.**"*
> *"**the collateral damage was pretty bad… and this year has been an exercise in eating shit.**"*
> *"I thought for a long time that social was the only thing that could drive the sort of viral growth to get crypto to a billion people. **It's clear that better money is more than enough** - we are seeing this live with stablecoins, predictions, perpetuals, tokenization."*
> 2026 年三大优先级：**"winning trading, payments, and agents"**；Base 要成为 *"the place that the world's money settles over the next century"*，点名 **Robinhood 和 Stripe** 为竞争者。

**Cobie**（<https://x.com/cobie/status/2077694974443876800>）：*"I am responsible for trading products at Coinbase (CB app / Pro / Baseapp / etc)... I cant explain why I did this except I like the opportunity to make something actually good more than I like playing Factorio."*（Coinbase 去年以约 **$3.75 亿**现金+股票收购其融资平台 Echo，把他招进来）

**Brian Armstrong（Coinbase CEO，同期）**：Base 的 content coins **"didn't work"**；Coinbase「今年年初就转向了」，优先级是 **"trading, payments, and agents（按此顺序）"**。<https://thedefiant.io/news/blockchains/coinbase-ceo-says-base-s-content-coins-didn-t-work>

**转向时的背景数据**（同一篇报道）：Base TVL **$4.54B**，全链第 5、以太坊 L2 第 1（Arbitrum $1.23B）；ZORA 距 2025-08 峰值 **−约 95%**，约半美分，市值约 **$3,100 万**；Base 24h DEX 量约 $886M、30 天约 $25.6B。Pollak 称 Base 的 DEX 市占与支付量有季度环比增长，**但未提供数据支撑**。

**时点也很关键**：报道明确指出这一转向发生在 **Robinhood 上线自己的以太坊 L2 之后一周**（围绕代币化股票与 meme 交易构建）——这与本项目的 mStocks 主题直接相关。

### 6.4 最反直觉的发现：Base 上真正跑出的第二波，是**机器**社交图谱，不是人类创作者经济

| | 第一波 | 第二波 |
|---|---|---|
| 时间 | 2024-11 ~ 2025-01 | **2026-01 ~ 2026-02** |
| 载体 | Farcaster（人类社交图谱） | **Moltbook（只允许 AI agent 互动的社交网络）+ Clawstr** |
| Clanker 峰值月手续费 | $13.76M（2024-12） | **$22.39M（2026-02）** |
| Clanker 峰值月发币量 | 10,039（2024-12） | **261,047（2026-02，日均 9,323）** |
| 占 Base 应用层手续费 | 4.1%（2025-01） | **19.7%（2026-02）** |
| 单日发币峰值 | — | **>13,000（2026-01-30/31）** |
| 单日手续费 | — | **>$600,000**；前后连续数日累计 >$3M |
| 累计交易量 | — | 2026-02 初 >**$7.62B**；1/30–1/31 两天 >$300M |

来源：`[一手]` DefiLlama + clanker.world API；诱因见 <https://www.kucoin.com/news/flash/clanker-token-creation-surpasses-13-000-per-day-near-previous-high>、<https://thedefiant.io/news/tokens/base-ai-agent-ecosystem-surges-with-rise-of-moltbook>

**因此命题应当改写为：**
> **Base 的 launchpad 冷启动来源不是「创作者经济」，而是「任何高频、低摩擦、带持久身份的社交图谱」。人类的（Farcaster / Base App）算，机器的（Moltbook / AI agent）更猛。**
> 这也解释了为什么 Pollak 把 2026 年的三大优先级定为 trading / payments / **agents** —— 官方也观察到了同一件事：**agent 是比人类创作者更高频的发币与交易主体。**

### 6.5 六条可迁移的结构性教训（写给 Mantle）

1. **分发能买到冷启动，买不到留存。** Base 用 Coinbase Wallet 的全部分发力量，把 Zora 的日发币量在两周内拉高 11 倍——然后在 13 个月内跌掉 99.2%。**不要把 launchpad 的成败押在「我们有分发」上。** Mantle 有 Bybit，这跟 Base 有 Coinbase 是同一种牌；Base 已经打过这张牌，结果记录在案。

2. **「无门槛发射」必然导致单币价值捕获趋零。** Clanker 每枚币手续费 $1,371 → $53（−96%）。发币量能靠 bot/agent 刷回来，**单币经济回不来**。→ **发射摩擦不是缺陷，是特性。** Flaunch 的 Game Mode（用可验证的努力换额度）、Virtuals 的 Genesis 积分制、Doppler 的荷兰拍衰减，都是在**重新加回摩擦**。mStocks 这种有真实标的的资产**更应该加摩擦**。

3. **价值在接口层，不在协议层。** `[一手]` 近 30 天：**Bankr（接口）$1.17M > Clanker + Zora + Flaunch 三个协议之和 $0.30M 的 3.9 倍**；pump.fun Mobile App 累计 $8.73M 里有 69% 发生在最近 30 天。**移动端原生分发 > 链上社交图谱分发。**

4. **要做底层，不要做第 N 个前端。** 本专题里唯一「活着且在增长」的是 **Doppler**（2026-01 拿 $9M，Zora / Bankr / Pure Markets 都在用它）；Flaunch 也已把自己重定位成 `RevenueManager` 底座（「builder 最多可拿 100% 手续费」）。**Mantle 应该做 Mantle 上所有 launchpad 的共同底层协议，而不是官方自营一个前端。**

5. **抗狙击有四条路线，Mantle 应选「可配置抛物线衰减费」+「可验证额度门禁」组合。**
   | 路线 | 代表 | 优点 | 缺点 |
   |---|---|---|---|
   | 硬延迟 | Clanker `2BlockDelay` | 最简单 | MEV 价值被烧掉，谁都没拿到 |
   | 首块拍卖 | Clanker `SniperAuctionV0/V2` | MEV 价值 80% 回流创作者 | **依赖链的 priority ordering + 精确落块能力**（Base 上都没 bundle 支持，官方自己吐槽）→ **在 Mantle 上可行性最差** |
   | 衰减高费 | Zora（99%→1%/10s 线性，硬编码）、Clanker `MevDescendingFees`（≤80%→抛物线，≤120s，可配置）、Bankr（约 80%/10s） | **不依赖排序语义，移植性最好**；MEV 价值转成 LP 费分给所有受益人 | 会把真实早期买家也一起罚到 |
   | 供需机制 | **Doppler**（未售库存持续压低价格） | 操纵曲线要真金白银亏钱 | 实现复杂度最高 |
   | 可验证额度门禁 | **Flaunch Game Mode / spend-gate** | 从「你是不是人」升级到「你凭什么配得上」；**expiry 写进池子，服务器死了也能自动开放交易** | 需要链下签名服务 |

6. **两个可以直接抄进 Mantle 的机制**
   - **Progressive Bid Wall**：用累积手续费在现价下方 1 个 tickSpacing 挂单边限价买单（阈值 0.1 ETH，`staleTimeWindow` 7 天），被吃单后把 memecoin 转进代币金库、ETH 循环进更高位的新挂单 → **单调不降的价格地板，且完全不给 MEV 下手的机会**。在流动性薄的链上，**被动挂单比主动市价回购的滑点损耗小一个数量级** —— Mantle 的流动性深度正是这种情形。
   - **quote 资产默认生息**：Flaunch 用 **flETH**（1:1 ETH 包装）做池子的原生币，闲置背书资产被扫进 **Aave v3** 吃约 2% 收益，归基金会。**Mantle 有 mETH / cmETH 这套生息 ETH 基础设施 —— 「launchpad 的 quote 资产默认是生息资产」在 Mantle 上是天然的、别人抄不走的优势。**
   - （加分项）**Clanker Droids 式的「现金流反哺运营成本」**：从 reward slot 切出 10%（100–5000 bps 可配）自动路由去支付某项持续成本。对 mStocks 可以是**预言机费、做市成本、合规审计费** —— 让资产自己养活自己的基础设施，这比「代币赋能」具体得多。

### 6.6 一句话总结

> **Base 的实验证明：把交易所级分发 + 社交图谱 + Uniswap v4 hook 拼在一起，可以在两周内把发币量拉高一个数量级；但也证明，这套组合无法把投机需求转化为留存需求。Base 官方已在 2026 年 7 月公开承认这条路线「definitively wrong」，转向 trading / payments / agents。对 Mantle 的含义是：不要重演「分发 + 无门槛发射」的剧本，而应把 Base 系最好的工程件（v4 hook 抗狙击、单边流动性 + 永久锁 LP、Progressive Bid Wall、收益权 NFT 化、可验证额度门禁）与 Mantle 独有的资产（mETH 生息 quote、mStocks 真实标的、Bybit 分发）组合成一个「有摩擦、有真实标的、quote 资产自带收益」的发射底层协议。**

---

## 7. Uniswap v4 hooks 对 launchpad 设计的意义

> 这一节是全专题技术上最可复用的部分：**Base 系四家 launchpad 的全部差异化都建立在 v4 hook 之上，而 v4 hook 是任何 EVM 链（含 Mantle）都能部署的。**

### 7.1 架构回顾

**单例 `PoolManager`**：v4 用**一个合约**保存**所有**池子状态（v3 是每池一合约）。池子由 `PoolId = keccak256(abi.encode(PoolKey))` 标识，**不是地址**。这是 v4 建池 gas 下降约 99% 的主因（状态写入 vs 合约部署）。

**Flash accounting / `unlock`**：基于 **EIP-1153 瞬态存储**的锁模型。任何要 swap / 加减流动性 / donate 的调用者必须先调 `poolManager.unlock(data)`，`PoolManager` 回调 `IUnlockCallback.unlockCallback(data)`。在回调内可以任意串联 `swap` / `modifyLiquidity` / `donate`，每一步净出一个 `int256` currency delta（按 `(locker, currency)` 存在瞬态存储）。回调返回前，每个非零 delta 必须归零：
- `settle()` —— 付进去（ERC-20/ETH 转账，或 `sync` + transfer）
- `take()` —— 取出来
- `mint()` —— 把 credit 转成 ERC-6909 claim
- `burn()` —— 用已有 claim 抵扣 debit（免转账）
顶层 `unlock` 返回时若还有非零 delta → **整笔交易 revert（`CurrencyNotSettled`）**。
→ **这就是多跳 swap、hook 内嵌套 swap、JIT 流动性能在一个原子帧内完成、且中间步骤零 ERC-20 转账的原因。** Zora 的「手续费多跳换成 ZORA 再分发」、Flaunch 的 `InternalSwapPool` 内部填单，全部依赖这个能力。

**ERC-6909 claims**：`PoolManager` 内部实现 ERC-6909 多代币，currency ↔ token id。hook / router 可以把工作余额留在 `PoolManager` 内跨池跨交易复用，不用来回搬真代币 —— **对反复复投手续费的 hook（如 bid-wall）至关重要**。

**`PoolKey` 与 hook 地址 flag**
```solidity
struct PoolKey {
    Currency currency0;
    Currency currency1;
    uint24 fee;
    int24 tickSpacing;
    IHooks hooks;      // ← 是 key 身份的一部分
}
```
`PoolManager` **通过读 hook 合约地址的低位 bit** 来决定调哪些回调，**不是**每次 swap 去做接口探测（更省 gas，且在 initialize 时就固定，无法中途篡改）。

### 7.2 14 个 hook flag（精确 bit 位）

来源：`Hooks.sol` —— `uint160 internal constant ALL_HOOK_MASK = uint160((1 << 14) - 1);` → **flag 占 hook 合约地址的低 14 位**。

| # | Solidity 常量 | Bit | 值 | 回调 |
|---|---|---|---|---|
| 1 | `BEFORE_INITIALIZE_FLAG` | 13 | `0x2000` | `beforeInitialize(sender, key, sqrtPriceX96)` |
| 2 | `AFTER_INITIALIZE_FLAG` | 12 | `0x1000` | `afterInitialize(sender, key, sqrtPriceX96, tick)` |
| 3 | `BEFORE_ADD_LIQUIDITY_FLAG` | 11 | `0x0800` | `beforeAddLiquidity(sender, key, params, hookData)` |
| 4 | `AFTER_ADD_LIQUIDITY_FLAG` | 10 | `0x0400` | `afterAddLiquidity(sender, key, params, delta, feesAccrued, hookData)` |
| 5 | `BEFORE_REMOVE_LIQUIDITY_FLAG` | 9 | `0x0200` | `beforeRemoveLiquidity(sender, key, params, hookData)` |
| 6 | `AFTER_REMOVE_LIQUIDITY_FLAG` | 8 | `0x0100` | `afterRemoveLiquidity(sender, key, params, delta, feesAccrued, hookData)` |
| 7 | **`BEFORE_SWAP_FLAG`** | 7 | `0x0080` | `beforeSwap(sender, key, params, hookData)` |
| 8 | **`AFTER_SWAP_FLAG`** | 6 | `0x0040` | `afterSwap(sender, key, params, delta, hookData)` |
| 9 | `BEFORE_DONATE_FLAG` | 5 | `0x0020` | `beforeDonate(sender, key, amount0, amount1, hookData)` |
| 10 | `AFTER_DONATE_FLAG` | 4 | `0x0010` | `afterDonate(sender, key, amount0, amount1, hookData)` |
| 11 | **`BEFORE_SWAP_RETURNS_DELTA_FLAG`** | 3 | `0x0008` | 允许 `beforeSwap` 返回非零 `BeforeSwapDelta` |
| 12 | `AFTER_SWAP_RETURNS_DELTA_FLAG` | 2 | `0x0004` | 允许 `afterSwap` 返回非零 unspecified-currency `int128` |
| 13 | `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` | 1 | `0x0002` | 允许 `afterAddLiquidity` 返回非零 `BalanceDelta` |
| 14 | `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` | 0 | `0x0001` | 允许 `afterRemoveLiquidity` 返回非零 `BalanceDelta` |

**`Hooks.validateHookPermissions` / `isValidHookAddress` 强制的不变量**
- 4 个 `*_RETURNS_DELTA_FLAG` 只有在对应基础 flag 也置位时才有意义（如只设 `beforeSwapReturnDelta` 而不设 `beforeSwap` → 拒绝）。
- 非零地址的 hook 必须**至少置位 14 个 flag 之一**，**或**池子使用 `DYNAMIC_FEE_FLAG`，否则 `initialize` revert `HookAddressNotValid`。
- 地址的 flag 位必须与 hook 的 `getHookPermissions()` 返回**完全一致**（构造函数中检查），否则构造即 revert。

来源：<https://github.com/Uniswap/v4-core/blob/main/src/libraries/Hooks.sol>、<https://developers.uniswap.org/docs/protocols/v4/concepts/hooks>

### 7.3 动态费（launchpad 的狙击税全靠这个）

**`LPFeeLibrary`**
```solidity
uint24 public constant DYNAMIC_FEE_FLAG    = 0x800000;   // uint24 fee 的最高位
uint24 public constant OVERRIDE_FEE_FLAG   = 0x400000;   // 次高位
uint24 public constant REMOVE_OVERRIDE_MASK = 0xBFFFFF;
uint24 public constant MAX_LP_FEE = 1_000_000;           // 100%，单位 pips（万分之一 bp）
```
- 池子是「动态费池」的条件：`PoolKey.fee` 在 initialize 时**恰好等于** `0x800000`（`isDynamicFee()` 检查的是 `self == DYNAMIC_FEE_FLAG`，**不是**位测试）。真实费率单独存在池子状态里，**初始为 0**。
- 这个 flag 与 14 个地址 flag **正交** —— 它在 `PoolKey.fee` 里，不在 hook 地址里。

**`updateDynamicLPFee`**
`IPoolManager.updateDynamicLPFee(key, newDynamicLPFee)`，**守卫条件：`msg.sender == address(key.hooks)` 且 `key.fee.isDynamicFee()`**。hook 通常在：
- `afterInitialize` 里设初始非零费（动态池初始是 0）
- `beforeSwap` / `afterSwap` 里按波动率、距发射时间、成交量档位调整

**逐笔覆盖（狙击税的真正实现方式）**
与 `updateDynamicLPFee` 独立：动态费池的 hook 可以从 `beforeSwap` 返回一个 **OR 上 `OVERRIDE_FEE_FLAG`（0x400000）** 的费率值，**只对这一笔 swap 生效**。核心里：
```solidity
if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();
```
→ **这就是「对刚好在第 1 个区块内成交的那一笔收 90% 税」而不需要把存储费率一直挂高的机制。** Zora 的 99%→1%/10s、Clanker 的 80%→5%/30s、Bankr 的 80%/10s 全部走这条路。

**LP 费 vs 协议费的拆分（重要区别）**
- 总 swap 费 = **LP 费**（动态或静态，上限 `MAX_LP_FEE` = 1,000,000 pips = **100%**）+ **协议费**（先扣）。
- **`ProtocolFeeLibrary.MAX_PROTOCOL_FEE = 1000` pips = 0.1%**，按方向打包成 uint24 的两个 12-bit 半（`zeroForOne` / `oneForZero` 各一）。
- 协议费先从输入额扣，LP 费再从剩余里扣：`swapFee = protocolFee + lpFee - protocolFee*lpFee/1e6`（向上取整，`ProtocolFeeLibrary.calculateSwapFee`）。
- 协议费由 `protocolFeeController`（核心 `PoolManager` 上的治理地址，**与 launchpad hook 无关**）通过 `ProtocolFees.setProtocolFee` 设定，只有该 controller 能 `collectProtocolFees`。

> **launchpad 必须理解的关键点**：0.1% 上限**只约束协议费**（Uniswap 治理层）。**hook 可以把整个 LP 费（最高 100%）路由到任何地方** —— 只要 hook（或它控制的 `PositionManager` 仓位）本身就是那个赚手续费的 LP。核心协议**不会自动把 LP 费路由到任何地方**，它只累积给「在区间内的流动性仓位」。**所以「手续费归创作者」的实现前提是：hook 自己（或它独占控制的仓位）就是唯一的 LP** —— 这直接推导出 §7.5 的 LP 门禁设计。

### 7.4 自定义曲线 / no-op hook（`beforeSwapReturnDelta`）

**`BeforeSwapDelta` 的精确打包**
```solidity
type BeforeSwapDelta is int256;
// 高 128 位 = SPECIFIED 币种的 delta（amountSpecified 指的那个）
// 低 128 位 = UNSPECIFIED 币种的 delta（与 afterSwap 的签名对齐）
function toBeforeSwapDelta(int128 deltaSpecified, int128 deltaUnspecified) ...
```

核心 `Hooks.beforeSwap` 的处理逻辑：
```solidity
int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();
if (hookDeltaSpecified != 0) {
    bool exactInput = amountToSwap < 0;
    amountToSwap += hookDeltaSpecified;
    if (exactInput ? amountToSwap > 0 : amountToSwap < 0) revert HookDeltaExceedsSwapAmount();
}
```
→ **hook 可以在内建集中流动性数学跑之前，吃掉用户指定输入/输出的一部分或全部。** 如果 hook 吃掉全部（把 `amountToSwap` 降到 0），底层 v3 式曲线执行一笔**零规模 swap**，**hook 就成了这笔交易的全部定价机制** —— bonding curve、固定价格销售、荷兰拍、订单簿撮合都可以这样实现。`afterSwap`（配 `AFTER_SWAP_RETURNS_DELTA_FLAG`）可以事后再调整 unspecified 币种的 delta（补舍入、多收一层费）。

**结算原语**：`poolManager.take(currency, to, amount)`、`currency.settle(...)`、`poolManager.donate(key, amount0, amount1, hookData)`（**不经 swap 直接给区间内 LP 记 fee growth** —— hook 用它把回购收益/协议奖励当「收益」注入现有 LP；Clanker 文档就明确提到「保留用 `donate()` 只给原始 LP 仓位受益人付款的能力」）。所有这些都汇入同一个 per-locker delta 账本，收尾必须归零。

**launchpad 应用**
- **bonding curve / 固定价格 fair launch**：hook 从自己存的曲线状态（如已售数量）算价，完全替换 `beforeSwap` 的 delta；底层池子的流动性甚至可以是名义的，或只在毕业后作为兜底。
- **荷兰拍**：hook 跟踪时间/区块，算递减起始价，同时调整 `beforeSwap` 的 delta 和/或移动 LP 仓位的 tick（**Doppler**）。
- **CLOB 式撮合**：配合 `afterSwap` 与存储的挂单状态，hook 可以在动用 AMM 流动性之前先撮合静止的限价单（Zora 有公开的 `IZoraLimitOrderBook` 接口）。

### 7.5 流动性锁定 / 单边流动性（launchpad 的「反 rug」基石）

**hook 持有 + 锁定的 LP**
因为 v4-periphery 的 `PositionManager` 仓位是 ERC-721，且 `beforeAddLiquidity` / `beforeRemoveLiquidity` 对**每一次** modify-liquidity 都会被调用，launchpad hook 可以：
1. 在建池时（`afterInitialize`）**自己 mint 掉全部初始流动性**（作为仓位所有者，或让 `PositionManager` mint 给 hook 自身/一个 burn 地址）→ 仓位 NFT 永不转出，除了 hook 自己的逻辑没人能抽走流动性（从交易者视角就是「LP 锁死」，因为不存在任何 EOA 控制的提取路径）。
   - Clanker 的做法更直接：**locker 合约根本没有 withdraw 函数**。
2. **门禁掉所有第三方 `modifyLiquidity`**：在 `beforeAddLiquidity` 里无条件 `revert`（除非 `sender == address(this)`）→ 池子变成 **permissionless-LP-free**，除了 hook 自己的逻辑谁都不能加流动性。**Doppler 就是这么做的**（bootstrap 期间禁止外部加流动性）；**Flaunch 在 Fair Launch 窗口内做同样的事**；**Clanker 在 MEV 模块运行期间做同样的事**。

**单边流动性（用一侧的 tick 区间实现）**
v4 保留了 v3 式集中流动性数学 → 一个仓位可以**完全位于当前 tick 之上或之下**，这样的仓位 mint 时**只持有两种代币之一**，行为上就是一张**限价单**，直到价格穿进它的区间。
- **现价之上的单边仓位** = 待分发的**卖方库存**（不需要同时提供 quote 资产）→ **Clanker 的 100% 供应单边投放、Zora 的 LP remint 上半部分**
- **现价之下的单边仓位** = **买墙 / 回购库存** → **Flaunch 的 PBW、Zora 的 LP remint 下半部分**

> ⚠️ **一个容易踩的坑**：hook 持有的单边仓位在价格**处于**其区间内时正常累积手续费（标准 v3 `feeGrowthInsideX128` 记账），但**价格在区间之外时手续费为零**。**这正是 Flaunch 的 bid wall 和 Zora 的发射区间必须由 hook 主动重新居中、而不能静态放着不动的根本原因。**

**手续费归集机制**
`afterAddLiquidity` / `afterRemoveLiquidity` 会把 `feesAccrued`（一个 `BalanceDelta`）返回给调用者；如果 hook 置了对应的 `*_RETURNS_DELTA_FLAG`，它还可以**截流并重定向**这部分（在到达名义调用者之前抽一刀）。由于按 §7.5 的门禁设计，hook 通常是该池**唯一**能加减流动性的实体，**该池实现的全部 swap 费收入实际上 100% 流向 hook 自己的仓位**，然后由 hook 按程序再分配（创作者份额、回购、协议份额），而不是被任意 LP 领走。

### 7.6 before/afterSwap 的 launchpad 模式清单

| 模式 | 常用 flag | 机制 |
|---|---|---|
| **首块/首 N 块拍卖、抗狙击** | `beforeSwap`（+ 动态费） | initialize 后前若干区块内高税或直接 revert / 限速，之后按块或按时衰减 |
| **按块限速** | `beforeSwap`（+ `afterSwap` 记账） | hook 存「某地址上次触碰的区块」或「本块累计量」，超限则 revert 或加税 |
| **黑白名单 / KYC 门禁** | `beforeSwap`、`beforeAddLiquidity` | 检查 `sender`（注意：这是 `PoolManager` 看到的 **router/caller** 地址，**不一定是 EOA** —— 通常要用 `hookData` 或上游 permit 签名来证明真实交易者） |
| **手续费收割 → 回购（bid-wall）** | `afterSwap` + 持有一个现价之下的单边 LP 仓位 | 累积费越过阈值（Flaunch = **0.1 ETH**）后转成现价略下的买单，并重新居中单边仓位 |
| **空投/解锁门禁** | `beforeSwap` 或 `beforeRemoveLiquidity` | 检查解锁计划 / merkle proof 后才允许卖出或撤 LP |
| **推荐归因（`hookData`）** | 任意阶段 | `hookData` 是调用者透传到 hook 的 `bytes calldata`，**核心不校验**。launchpad 把推荐人地址编进去（`abi.encode(referrer)`），hook 在 `afterSwap` 解析并记账（**Zora 的 Trade Referral 4% 就是这么实现的**）。⚠️ **`hookData` 是攻击者可控且未认证的，任何由它触发的特权动作必须 hook 自己独立验证**（ECDSA 验签、merkle proof），盲信是常见踩坑点 |
| **预言机更新（truncated oracle）** | `afterSwap`（有时 `afterInitialize`） | hook 每笔 swap 后写 TWAP 观测数组；"truncated oracle" 额外**钳制单块最大价格移动**以抗单块操纵（**Flaunch 的 `Oracle.sol` + `InternalSwapPool` 用 `twapTick()` 而非 spot 定价，正是这个思路**） |
| **限价单** | `afterSwap`、`beforeSwap` | hook 按 tick 存挂单；`afterSwap`（或 `beforeSwap` 先撮合）检查价格是否穿过有挂单的 tick，用 `take`/`settle` 执行结算，该腿完全绕过 AMM 曲线 |

### 7.7 实践约束、风险、以及 hook mining（非显然但必须懂）

**（1）hook 按池不可升级 —— 这是最重要的架构约束**
`hooks` 在 `initialize` 时被烙进 `PoolKey`/`PoolId`，**永不可改，没有 setHook**。→ **launchpad 无法升级一个已上线池子的 hook 逻辑。** 任何不能通过**现有 hook 自己的可变存储**参数化的改动（费率表、税曲线、门禁逻辑），都必须**部署新池子**（新 `PoolKey`）并迁移流动性。
→ **因此绝大多数 launchpad hook 都是重度参数化的：一个 hook 合约服务很多池子，每个池子的具体参数按 `PoolId` 存在 hook 自己的存储里，在 `beforeInitialize`/`afterInitialize` 时从创建参数写入。**
- **Clanker 的应对**：核心 3 合约不可升级，但把 hook / locker / extension / MEV module 做成**4 个可白名单替换的接口** → 新逻辑 = 新模块 + 新池子，老池子不受影响。**这是本专题里对「hook 不可升级」最成熟的工程回答，强烈建议 Mantle 照抄这个分层。**
- **Zora 的应对**：`BaseCoinV4` 里带 `migrateLiquidity()`，支持在 hook 版本间迁移流动性（v2.3.0 把两个 hook 合并成 `ZoraV4CoinHook` 时就用到了）。
- **Flaunch 的应对**：多个 `PositionManager` 并存（`PositionManager1` / `PositionManager2`）—— 代价见 §3.4 那个不可变收款人的教训。

**（2）gas 开销**
- 单例 + flash accounting 让**建池** gas 降约 99%（状态写 vs 合约部署）；**多跳 swap** 的节省温和一些（第三方估算约 **23–38%**，视路径复杂度），来自净额结算免掉每跳的 ERC-20 转账。
- 但 hook 本身是**额外的外部调用**（`beforeSwap`/`afterSwap` 都是对另一个合约的 `CALL`）→ hook 越多回调、逻辑越重（外部调用、存储写、TWAP 更新），每笔 swap 的边际 gas 越高。
- **没有找到 Uniswap 官方对「hook 执行的边际 gas」的基准测试**（**待核实**）；只有针对**核心**多跳节省的第三方估算。
  → **对 Mantle 的含义**：Mantle 的 gas 成本结构与 Base 不同，**必须自己基准测试 hook 开销**，尤其是像 Flaunch 那样在一个 `beforeSwap` 里做 4 件事的重 hook。

**（3）安全面：恶意/有 bug 的 hook 可以偷钱或冻结资金**
hook 在同一调用上下文里运行，能用几乎任意逻辑回调 `PoolManager`（`take` / `settle` / `donate` / `mint` / `burn`），所以恶意或有 bug 的 hook 可以：
- 从 `beforeSwap`/`afterSwap`/`afterAddLiquidity`/`afterRemoveLiquidity` 返回被操纵的 delta（只受 `Hooks.beforeSwap` 里「delta 不能把 exact-in 翻成 exact-out」这一检查和 `Pool.swap` 的范围检查约束）→ 实质上错误定价或抽干一笔 swap
- 拒绝交出累积手续费，或人为阻塞 `beforeRemoveLiquidity` 来困住存款人的资金
- 在自己的回调里抽水、抢跑、审查交易 —— 它是一个完全通用的合约，且**能看到 pending calldata（`hookData`）和池子状态**

Uniswap periphery 提供 `BaseHook`（v4-periphery）作为经审计的脚手架（实现权限声明样板与分发模式），**但 `BaseHook` 不会让 hook 的业务逻辑变安全** —— **每一个 launchpad hook 都是独立的审计面**。
**实例警示**：2025-09 的 **Bunni v2 漏洞（约 $840 万）** 根因是 hook 相关的记账 bug（LDF / 反复微额提取下的舍入），**不是 v4 核心的缺陷** —— 说明**在重 hook 的池子里，主导风险面已经从 `PoolManager` 的正确性转移到 hook 的正确性**。<https://blog.bunni.xyz/posts/exploit-post-mortem/>

**（4）`hookData` 的大小与信任**
`hookData` 从顶层调用者（router/periphery）原样透传到 hook，**核心不做任何解释**。协议层没有大小上限（只受 calldata/gas 经济约束），但大 `hookData` 花 calldata gas，典型用法是打包结构体（推荐人地址、deadline、签名）而非大载荷。**因为它是攻击者提供且核心不认证的，hook 从 `hookData` 推导出的任何特权效果（推荐记账、白名单绕过、KYC 证明）都必须由 hook 自行验证。**

**（5）Hook mining / `HookMiner` —— CREATE2 salt 搜索（非显然但必须做）**

因为**权限是从已部署地址的低 14 位读出来的**，而以太坊地址不由部署者直接选定 → **launchpad 不能用普通方式部署 hook 然后期待那些 bit 刚好对**（结果地址相对于那些位基本是随机的）。

解法是 **`CREATE2`**：地址 = `keccak256(0xff ++ deployer ++ salt ++ keccak256(initcode))[12:]`，给定 `(deployer, salt, initcode)` 完全确定。**`HookMiner`**（v4-periphery 工具）在**链下**暴力搜索 `salt`（一个简单循环，不是链上计算），直到结果地址 `& Hooks.ALL_HOOK_MASK`（`0x3FFF`，低 14 位）**恰好等于**目标 flag 组合：
```solidity
uint160 flags = uint160(Hooks.AFTER_ADD_LIQUIDITY_FLAG | Hooks.AFTER_SWAP_FLAG);
bytes memory constructorArgs = abi.encode(POOLMANAGER);
(address hookAddress, bytes32 salt) =
    HookMiner.find(CREATE2_DEPLOYER, flags, type(PointsHook).creationCode, constructorArgs);
```
部署脚本随后用挖到的 `salt` 通过标准 `CREATE2` deployer（如固定地址的 deterministic-deployment-proxy，或 Foundry 自己的 CREATE2 工厂）部署，并断言实际地址 == 挖到的地址。

> ⚠️ **挖矿只搜索 `salt`，不搜索构造参数或字节码 → 改动任何构造参数或编译器设置都会改变 `initcode`，之前挖到的 salt 立即失效。** 这就是为什么 hook 部署脚本要把 salt 钉死到某个具体 build。
> 本地测试可用 Foundry 的 `deployCodeTo` cheatcode 把字节码直接放到任意地址、完全绕过挖矿（**仅测试可用**，真链上不行，因为它不走 CREATE2）。
> **这是 launchpad 集成里最常见的 bug**：把 hook 部署到 bit 与 `getHookPermissions()` 不符的地址，会导致 hook 构造期的 `Hooks.validateHookPermissions` 直接 revert；若跳过该检查，则 `PoolManager.initialize` 会**静默地永不调用 hook「以为自己有」的那些回调**（或以 `HookAddressNotValid` revert）。
> **给 Mantle 的操作性提醒**：确认 Mantle 上有标准的 deterministic CREATE2 deployer（`0x4e59b44847b379578588920cA78FbF26c0B4956C`）部署到位，否则整个 hook 生态无法按 Base 的方式落地。

### 7.8 launchpad → hook flag 映射表

| Launchpad | 使用的 hook flag | 用途 |
|---|---|---|
| **Clanker v4 / v4.1** | `beforeInitialize`、`afterInitialize`、`beforeSwap`、`afterSwap`；MEV 模块运行期封 `beforeAddLiquidity`；动态费池（`ClankerHookDynamicFeeV2`）/ 静态费池（`ClankerHookStaticFeeV2`） | 每笔 swap 自动收取初始 LP 仓位手续费并路由进 `ClankerFeeLocker`；**DEX 层协议费 = active LP 费的 20%，永远以配对代币计**；触发 MEV 模块（上限 2 分钟）；LP 费上限 30%。hook 必须被 Clanker 工厂白名单。（精确 bitmask **待核实**） |
| **Flaunch** | **全部 9 个实际回调都用了**：`beforeInitialize`（禁外部初始化）、`afterInitialize`、`beforeAddLiquidity` / `beforeRemoveLiquidity`（Fair Launch 期禁外部加/撤流动性）、`afterAddLiquidity` / `afterRemoveLiquidity`、`beforeSwap`（Fair Launch 填单/关仓 + ISP 内部填单）、`afterSwap`（捕获并分配手续费 → BidWall）、`afterDonate` | 池子 `fee=0`、`tickSpacing=60`，所有收费在 hook 内做（`_captureSwapFees`）；Fair Launch 固定价格；PBW 单边买墙；`InternalSwapPool` 用 TWAP 定价内部填单避免砸盘。**四家里 hook 使用深度最高的** |
| **Zora（Coins v4 / `ZoraV4CoinHook`）** | `afterInitialize`（种下初始多曲线流动性）、`beforeSwap`（99%→base 的衰减发射税，`_calculateLaunchFee()`）、`afterSwap`（`collectFees()` → `mintLpReward()` → `_distributeMarketRewards()`，多跳换汇成 ZORA 后分发） | 抗狙击发射税；hook 持有的单/多区间初始流动性；20% 手续费 remint 成永久单边流动性；创作者奖励来自 **Zora 自己的市场流动性仓位赚到的手续费**（不是按成交量），由 `CoinRewardsV4` 计算，经 `ICoin` 的 `CoinMarketRewards` 事件暴露 |
| **Doppler v4（Whetstone）** | `afterInitialize`（放置初始 bonding-curve 仓位）、`beforeSwap`（若落后销售进度则通过 tick 下移重平衡曲线）、`afterSwap`（跟踪累计售出/收款，排除手续费）、**`beforeAddLiquidity`（revert 所有外部加流动性）** | 荷兰拍式动态 bonding curve：按 `duration`/`epochLength` 计划出售 `numTokensToSell`；销量落后则按落后幅度自动下移曲线；bootstrap 期完全排除第三方 LP |
| **Bunni v2**（非 launchpad，但是曲线替换的范式样本） | `beforeSwap`/`afterSwap`（自定义流动性密度函数定价 + TWAP 驱动的费率逻辑）；通过 LDF 实现单边/非对称流动性 | tick 上的流动性密度由可编程 **LDF** 定义而非平坦 v3 区间；TWAP 预言机驱动 LDF 重塑与波动率敏感费率；可把闲置储备再抵押到外部借贷市场（此项在 2025-09 漏洞中是次要因素而非根因） |
| **EulerSwap**（仅备注，非 launchpad） | 用 v4 hook 接口作为借贷协议支撑的 AMM 的集成面（自定义曲线，用「信用额度感知」定价替换恒定乘积）。精确 flag **待核实** | 仅作为 launchpad 领域之外另一个 `beforeSwapReturnDelta` 级完整曲线替换的例子 |

### 7.9 v4 hook 对 launchpad 设计的六条意义（总结）

1. **动态费把「抗狙击」从链层问题变成应用层问题。** 在 v3 时代，抗狙击只能靠链的排序语义（priority ordering / private mempool / bundle）。v4 的 `OVERRIDE_FEE_FLAG` 让 launchpad 可以**在应用层用一行返回值实现任意的时间/区块相关税曲线**。→ **这是 Mantle 最应该优先落地的能力**：它不要求 Mantle 有 Solana 式的本地费用市场或 Base 式的 priority ordering，只要求有 Uniswap v4。

2. **`beforeAddLiquidity` revert 是「反 rug」的最强原语。** 「LP 锁定」有三种实现强度：把 LP token 烧掉（Wow.xyz / Virtuals，最弱，池外仍可加流动性竞争）→ locker 合约无 withdraw（Clanker，强）→ **`beforeAddLiquidity` 无条件 revert 掉一切外部加流动性（Doppler / Flaunch fair launch 期 / Clanker MEV 期，最强）**。最后一种同时解决了「有人开竞争池/加竞争流动性稀释创作者手续费」的问题。

3. **单边流动性让 bonding curve 变成可选项。** Clanker 证明了：只要有集中流动性 + 单边 tick 区间，**根本不需要写 bonding curve、也不需要毕业迁移**。这把系统复杂度从「两套定价系统 + 一次原子迁移」降到「一次 tick 计算」，**消除了整整一类漏洞面**（毕业迁移是 Solana 系 launchpad 的经典事故点）。代价是失去「毕业」这个天然注意力事件与 curve 阶段的抗夹优势——用动态费补上即可。

4. **`beforeSwapReturnDelta` 意味着「AMM」和「发行机制」不再需要是两个系统。** 想要 bonding curve、固定价格窗口、荷兰拍、订单簿？全部都可以作为**同一个 v4 池上的一个 hook** 存在，共享同一套流动性、路由、预言机基础设施。这是 v4 相对 v3 最深的架构差异，也是 Doppler 能把自己做成「中立管道」的技术前提。

5. **`hookData` 是免费的归因层。** Zora 的 Trade Referral（4%）纯粹靠 `abi.encode(referrer)` 塞进 `hookData` 实现，不需要任何 registry、不需要链下索引。**任何 launchpad 都应该从第一天就把这条通道留出来**（Mantle 的场景：Bybit 前端 / 第三方 bot / KOL 归因）。但记住它未认证，特权动作必须验签。

6. **代价是：hook 不可升级 + hook 成为主导风险面 + 必须做 hook mining。** 这三件事共同要求一种特定的工程组织方式：**「不可升级的薄核心 + 可白名单替换的厚模块 + 每池按 PoolId 参数化」**。Clanker v4 的 `3 核心 + 4 接口` 是这个模式最干净的实现，**建议 Mantle 直接照抄这个分层，而不是从零设计。**

---

## 8. 附录

### 8.1 一手数据方法说明

标记 `[一手]` 的数字由本次研究直接调用 API 取得（2026-09-06）：
- **DefiLlama**：`https://api.llama.fi/summary/fees/<slug>?dataType=dailyFees|dailyRevenue`（取 `totalDataChart` 后按月聚合、找单日峰值）；`https://api.llama.fi/overview/fees?...`（协议清单）；`https://api.llama.fi/overview/dexs/<chain>`（链级 DEX 量）；`https://api.llama.fi/overview/fees/<chain>`（链级应用层手续费）；`https://api.llama.fi/summary/dexs/<slug>`（协议级 DEX 量）
- **clanker.world 公开 API**：`GET https://www.clanker.world/api/tokens?limit=1&startDate=<unix>` —— 响应里的 `total` / `tokensDeployed` 字段就是「该时点之后部署的代币数」；用月度边界差分得到逐月发币量。`chainId=8453` 得到 Base 单链数。（注意：`page` 参数**无效**，实际分页要用 `cursor` 或 `offset`；`limit` 上限 20）
- **Zora API**：`GET https://api-sdk.zora.engineering/explore?listType=NEW&count=100&after=<cursor>` —— 分页遍历 `exploreList.edges`，按 `node.createdAt` 统计 24 小时内新建币数与 `node.coinType` 分布
- **CoinGecko**：`/coins/markets?ids=...&price_change_percentage=30d,1y`（价格/市值/ATH）

**已知口径冲突（务必标注）**
| 项目 | 冲突 | 处理 |
|---|---|---|
| Clanker 累计手续费 | DefiLlama **$90.82M** vs The Block $27M（2025-04）vs Cointelegraph $34.4M（2025-08） | 以 DefiLlama 为准（口径更宽 + 含 2026 年那一大波），二手数字仅作时点参照 |
| Clanker 单日手续费峰值 | DefiLlama **2024-12-03 $4.79M** vs Dune「2024-11-26 $1.1M」 | 两者相差 4 倍，说明 fee 定义不同（DefiLlama 明确 = creator fee + Clanker 的 20%）。**两个都列，标注冲突** |
| Zora TGE 日期 | 官方 support 文章 **2025-03-11** vs 二手 **2025-04-23**（公开领取/开放交易） | 很可能是两个不同事件，**未完全对齐，待核实** |
| ZORA "Holders Revenue" | DefiLlama 累计 $1.5M，与文档的 Creator 分成线**对不上** | 口径**待核实** |
| Flaunch 手续费 | Dune 的 **2,072.69 ETH** vs DefiLlama 的 **$3.59M** | 方法/窗口不同，**不要直接换算比较** |
| Base App 用户数 | 官方从未披露 MAU/DAU；「1 亿 verified users」是 2021 IPO 的遗留指标且 2023 年已停用 | **不使用任何「Base App 触达 1 亿人」的说法** |

### 8.2 待核实清单（汇总）

**Clanker**
1. v0 精确上线日与创始团队（Jack Dishman / proxystudio.eth）—— 仅二手
2. 0xMacro v3.1 审计的精确措辞与严重级别 —— 未读原始 PDF
3. `ClankerHook` / `ClankerHookV2` 的精确 `Hooks.Permissions` bitmask —— 需读 `v4-contracts` 源码
4. 各前端（Bankr / clanker.world / ClankFun / Native / Tab）在自己 rewardBps slot 内部的分成
5. CLANKER 总供应 / 分配 / 解锁表 / 是否有 staking
6. CLANKER ATH $142.84（2025-10-26）与当前市值 —— 仅聚合器
7. 「Clanker 被 Farcaster/Neynar 收购（2025-10）」—— **查无一手来源，很可能不实**
8. Clanker Ecosystem Fund (CEF) 细节 —— crypto.news 正文未取到
9. Clanker **v5** 是否已存在（Droids 文档提到「v4 和 v5 都支持」）
10. Clanker 代币存活率（「>95% 48h 内死」「约 1% 达 $100k」）—— 低可信、无出处
11. Farcaster vs 网站 vs 第三方的发币占比 —— 需针对 `TokenConfig.context` 做自定义 Dune 查询
12. Top Clanker 代币（ANON / NATIVE / BNKR）的市值排名与日期

**Zora**
13. ETH → ZORA 配对资产切换的精确日期与机制（是离散迁移还是自 6-20 上线即如此）
14. `ZoraFactory` `0x7777777516…` 与 `ZoraV4CoinHook` 地址 —— 未在 BaseScan 二次确认
15. **狙击税收入是否走与普通手续费完全相同的分配表** —— 这是推断，非文档明示（若成立则创作者能吃到 99% 税的一半，影响很大）
16. 协议 v2.2.0（费率重构）与 v2.5.0（狙击税 legacy 分界）的日历日期
17. ZORA 各类别分配比例 —— 官方页面是图片，聚合器数字未核实
18. **ZORA 是否存在真正的回购项目** —— 本文结论是**不存在**，只有强制多跳换汇产生的结构性买盘。若有官方公告请以其为准
19. Trend Coin 三条曲线只覆盖 37.5% 供应，剩余 62.5% 的分布
20. Base App 全球开放日（2025-12-17/18）—— 仅二手
21. 「通过 Base App 发的币 vs 通过 zora.co 发的币」拆分 —— 可能无法回答
22. Content Coin 30 天留存/存活率的一手量化数据 —— 需自跑 Dune

**Flaunch**
23. 默认起始市值（常引 $10,000，未在官方页面确认）
24. Fair Launch 默认占总供应的百分比
25. `MarketCappedPrice.sol` / `AnyMarketCappedPriceV3.sol` 的精确机制
26. Ethereum / Unichain / Robinhood Chain 的完整合约地址表
27. **Flaunch / Flayer Labs 是否有 VC/seed 轮** —— 多次检索无果，按「无」处理
28. Flayer Labs = NFTX + FloorDAO 合并（2024-09）—— 仅二手
29. 「Solana Imports」/「Flaunch Wrap: Solana→Base」的机制 —— 仅见 changelog 标题
30. Top Flaunch 代币排名

**其他**
31. Bags「Get Bagged」的具体司法案例 —— 系统性批评已确认，单一案例未确认
32. Virtuals「2025-01 市值约 $50 亿」—— **应视为夸大**，价格 ATH $5.07 已确认
33. Doppler 用于 Ronin —— **查无实据，应视为错误**
34. Believe 2026 年的法律审查
35. Party.app / Rodeo / Phi 作为活跃 Base 创作者代币平台 —— **查无实据，已丢弃**
36. 「Buttery」launchpad —— **不存在，已丢弃**

**Uniswap v4 / 基础设施**
37. `HookMiner.sol` 与 `BaseHook.sol` 在 v4-periphery 的当前精确路径/API
38. **hook 执行的边际 gas 开销的官方基准** —— 未找到 Uniswap 官方数据，只有针对核心多跳节省的第三方估算（约 23–38%）
39. Doppler v4 / Bunni v2 / EulerSwap 的精确 flag 集合（地址位 vs 文档叙述）
40. Base "Denim" 硬分叉（原生 200ms 区块）的时间表 —— 截至 2026-09 仍是 RFC
41. Base sequencer 收入的精确季度数字 —— 需查 investor.coinbase.com 原始股东信

### 8.3 主要来源索引

**官方文档 / 代码**
- Clanker：<https://clanker.gitbook.io/documentation>｜<https://clanker.gitbook.io/documentation/llms.txt>｜<https://github.com/clanker-devco>｜合约表 <https://clanker.gitbook.io/documentation/references/deployed-contracts.md>｜费用 <https://clanker.gitbook.io/documentation/general/creator-rewards-and-fees.md>｜v4 <https://clanker.gitbook.io/documentation/references/core-contracts/v4.md>｜部署配置 <https://clanker.gitbook.io/documentation/references/core-contracts/v4/deployment-config.md>｜ClankerHook <https://clanker.gitbook.io/documentation/references/core-contracts/v4/clankerhook.md>｜MEV 模块 <https://clanker.gitbook.io/documentation/references/core-contracts/v4/mev-modules/clankersniperauctionv0.md>、<https://clanker.gitbook.io/documentation/references/core-contracts/v4/mev-modules/clankermevdescendingfees.md>｜Droids <https://clanker.gitbook.io/documentation/droids/funding.md>｜狙击技术文 <https://paragraph.com/@clankerworld/clanker-v4_1-sniper-tech>｜拍卖设计文 <https://hackmd.io/@lobstermindset/rkwlyMpkgl>
- Zora：<https://docs.zora.co/coins>｜<https://docs.zora.co/skill.md>｜奖励 <https://docs.zora.co/coins/contracts/rewards>｜架构 <https://docs.zora.co/coins/contracts/architecture>｜Trend Coins <https://docs.zora.co/coins/contracts/trend-coins>｜推荐奖励 <https://docs.zora.co/coins/contracts/earning-referral-rewards>｜代币学 <https://support.zora.co/en/articles/4797185>｜<https://github.com/ourzora/zora-protocol>
- Flaunch：<https://docs.flaunch.gg>｜<https://docs.flaunch.gg/llms.txt>｜hooks <https://docs.flaunch.gg/references/hooks.md>｜PBW <https://docs.flaunch.gg/features/auto-buybacks.md>｜Fair Launch <https://docs.flaunch.gg/features/fixed-price-fair-launch.md>｜创作者收益 <https://docs.flaunch.gg/features/creator-revenue.md>｜Royalty NFT <https://docs.flaunch.gg/features/royalty-nft.md>｜协议费开关 <https://docs.flaunch.gg/community/governance/protocol-fee-switch.md>｜RevenueManager <https://docs.flaunch.gg/managers/revenuemanager.md>｜Game Mode <https://docs.flaunch.gg/game-mode/game-mode.md>｜spend-gate <https://docs.flaunch.gg/references/spend-gate.md>｜whitepaper <https://docs.flaunch.gg/community/whitepaper.md>｜<https://github.com/flayerlabs/flaunchgg-contracts>
- Uniswap v4：<https://developers.uniswap.org/docs/protocols/v4/concepts/hooks>｜`Hooks.sol` <https://github.com/Uniswap/v4-core/blob/main/src/libraries/Hooks.sol>｜`LPFeeLibrary.sol` <https://github.com/Uniswap/v4-core/blob/main/src/libraries/LPFeeLibrary.sol>｜`ProtocolFeeLibrary.sol` <https://github.com/Uniswap/v4-core/blob/main/src/libraries/ProtocolFeeLibrary.sol>｜`BeforeSwapDelta.sol` <https://github.com/Uniswap/v4-core/blob/main/src/types/BeforeSwapDelta.sol>｜`PoolKey.sol` <https://github.com/Uniswap/v4-core/blob/main/src/types/PoolKey.sol>｜flash accounting <https://developers.uniswap.org/docs/protocols/v4/concepts/flash-accounting>｜ERC-6909 <https://developers.uniswap.org/docs/protocols/v4/concepts/erc-6909>｜truncated oracle <https://blog.uniswap.org/uniswap-v4-truncated-oracle-hook>
- Base：<https://blog.base.org/a-new-day-one>（2025-07-16）｜<https://blog.base.dev/scaling-base-in-2025>（2025-02-07）｜<https://blog.base.dev/scaling-base-doubling-capacity-in-30-days>（2025-10-28）｜<https://blog.base.org/building-for-the-long-term-making-base-faster-simpler-and-more-powerful>（2025-02-27）｜<https://blog.base.dev/flashblocks-deep-dive>｜<https://blog.base.dev/accelerating-base-with-flashblocks>｜<https://blog.base.dev/scaling-base-sustain-10x-growth>（2025-06）｜<https://docs.base.org/upgrades/denim/migrate-from-flashblocks>｜<https://blog.base.org/evolving-coinbase-wallet-to-bring-the-world-onchain>（2025-02-04）

**关键新闻 / 一手引用**
- **Pollak 反转（2026-07-16）**：<https://thedefiant.io/news/people/base-creator-jesse-pollak-hands-app-to-cobie-says-social-bet-was-definitively-wrong>；原推 <https://x.com/jessepollak/status/2077427261586997745>；Cobie <https://x.com/cobie/status/2077694974443876800>
- Armstrong「content coins didn't work」：<https://thedefiant.io/news/blockchains/coinbase-ceo-says-base-s-content-coins-didn-t-work>
- Moltbook / AI agent 浪潮：<https://thedefiant.io/news/tokens/base-ai-agent-ecosystem-surges-with-rise-of-moltbook>；<https://www.kucoin.com/news/flash/clanker-token-creation-surpasses-13-000-per-day-near-previous-high>
- Clanker 早期规模：<https://www.theblock.co/news/business/2025-04-04-clanker-team-earns-13-million-in-revenue-from-over-200000-tokens-on-base-in-just-five-months-349549>；<https://www.tradingview.com/news/cointelegraph:258c043c5094b:0-ai-bot-clanker-racks-up-34m-in-swap-fees-launching-base-memecoins/>
- Clanker 回购与费用政策：<https://www.kucoin.com/news/flash/clanker-to-return-fee-control-to-creators-starting-november-13>
- Base App 上线报道：<https://www.theblock.co/news/business/2025-07-16-coinbase-unveils-base-app...-362713>
- Zora × Base App 效应：<https://www.theblock.co/news/business/2025-07-21-zora-token-base-app-363518>；<https://thedefiant.io/newsletter/defi-daily/are-zoras-content-coins-just-high-brown-memecoins>
- 「Base is for everyone」争议：<https://fortune.com/crypto/2025/04/19/coinbase-zora-base-content-coin-jesse-pollak>；<https://cointelegraph.com/news/coinbase-distances-base-criticized-memecoin-drops-15-million>；<https://blockworks.com/news/jesse-pollak-base-content-coin>；<https://www.forbes.com/sites/clorischen/2025/04/30/the-controversies-around-zora-and-base-explained>
- Bunni v2 事故复盘：<https://blog.bunni.xyz/posts/exploit-post-mortem/>
- Flaunch Game Mode 数据线程：<https://x.com/flaunchgg/status/2085701754864189580>

**看板**
- <https://defillama.com/chain/base>｜<https://defillama.com/chain/solana>｜<https://defillama.com/protocol/clanker>｜<https://defillama.com/protocol/zora-coins>｜<https://defillama.com/protocol/flaunch>｜<https://defillama.com/protocol/bags>
- <https://dune.com/flaunch/flaunch-protocol-dashboard>（Flaunch 官方）｜<https://dune.com/clanker_protection_team/awesome-clanker>（Clanker 社区）｜<https://dune.com/zorateam/coins>
- <https://growthepie.xyz>｜<https://tokenterminal.com/explorer/projects/base>｜basescan.org/chart/tx
