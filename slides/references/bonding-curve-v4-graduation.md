# Bonding Curve、毕业迁移与 Uniswap v4：内容核对

本稿为 Section 1 的 P09、P09A、P12A、P12B 提供机制依据。目标是讲清产品如何运行，不提供已审计的 Launchpad 合约实现，也不把某条链的 v4 部署或 EIP-1153 支持视为已经核验。

## 1. Bonding Curve 是定价规则，不是一种固定公式

用户直接按合约规则买卖，合约根据净售出量或储备状态计算应支付、应收到的数量，不需要先等另一位用户挂出相反订单。以下是三类教学示例：

| 类型 | 规则示例 | 如何向听众解释 |
| :-- | :-- | :-- |
| 线性 | `p(s) = a + b·s`，`a > 0, b > 0` | 净售出量越多，下一份的报价按固定斜率提高 |
| 指数 | `p(s) = a·exp(b·s)` | 后续报价提高得越来越快；形状由参数决定 |
| 虚拟储备恒定乘积 | `(x + v_x)(y + v_y) = k` | 买入改变用于计算的储备比，后续同样投入通常买到更少代币 |

`s` 表示净售出量，`p(s)` 是边际报价；连续曲线的大额买单应按区间积分或等价储备变化计算，不能拿买前报价直接乘整笔数量。第三类中，`x` 是真实计价资产储备，`y` 是该曲线的真实 Meme 库存，`v_x/v_y` 是加入定价公式的虚拟偏移。手续费、额外资金变更和毕业转换另行处理；`k` 在无费用、无外部注资的示例交换中保持不变。

虚拟储备不是资产余额。它能给初始定价提供非零参数、降低创作者预先配齐双边 LP 的需求，但不能凭空提供可提取的 mStocks，也不能允许卖出超过真实偿付能力或买走超过真实库存的代币。

### 一个可核对的小算例

教学设定：真实计价储备 `x=0`，真实 Meme 库存 `y=1000`，虚拟偏移 `v_x=10`、`v_y=0`，忽略手续费。计价单位可理解为 mStocks 单位，`k=10000`。

- 第一笔投入 1 单位计价资产：剩余 Meme 为 `10000/11`，用户收到约 `90.91`。
- 第二笔再投入 1 单位：剩余 Meme 为 `10000/12`，第二位用户收到约 `75.76`。
- 两笔投入相同，第二笔得到更少，因为曲线状态已被第一笔改变。若按规则卖回，储备会反向变化；价格不是随时间自动上涨。

这是解释公式的人工样本，不是 pump.fun、four.meme 或本次产品的真实参数与预期收益。代币精度、定点数取整和买卖可逆性需另行实现与测试。

## 2. 毕业迁移的对象是实际资产与交易状态

典型旧式路径为：达到门槛，停止发行曲线交易；确定扣费后的真实计价储备及预留 Meme；初始化目标池；把真实资产配置为 LP；开放二级交易并记录 LP 权益。

**迁入的只能是真实储备与可支配代币，虚拟储备不迁入。** 毕业价、池初始化价格、代币数量、LP 价格区间、剩余资产归属都要一致。

如果分成多笔交易或多个独立状态转换：

- 旧曲线已停、新池未就绪，用户遇到交易空窗；后续交易失败时资金和状态可能停在中间。
- 新池可能在配置完成前被抢先创建、初始化或交易，导致预期价格和实际池状态不同；是否可利用取决于具体权限和校验。
- Curve 末端报价与二级池配置若不匹配，会发生价格跳变和可被利用的价差。该问题即使在同一笔交易中也可能存在，原子性不能修复错误的比例与价格。

原子化的要求是将毕业相关的资产、价格、LP 和阶段状态作为同一组提交条件；任一步失败，相关变更一起回滚。**跨合约不等于非原子**：v2/v3 等同链合约也可以经正确编排在一笔 EVM 交易内迁移。v4 提供更便于共用池与结算的结构，不独占原子迁移能力。

Bonding Curve 帮助组织初始定价，但不天然消除抢跑、夹子或排序优势；这些还依赖价格限制、交易排序与访问规则。

## 3. 为什么 v4 更适合本方案的发行与毕业一体化

| 特性 | 对 Launchpad 的实际作用 | 不应夸大的地方 |
| :-- | :-- | :-- |
| Singleton / PoolManager | 多个池以不同 PoolId 管理，创建池时不再每池部署独立 pool 合约；有利于大量新币池和后续聚合 | 新 Meme 的代币合约仍可能需要部署，Hook 也有部署/存储开销；不是一次注册就完成发币和全部流动性配置 |
| Flash Accounting | 同一次解锁中先累计资产变化，多池操作可净额结算，减少逐跳中间转账 | 真实资产仍需交付；不是免转账、免费用或把不同币种按美元总额抵销 |
| Hooks / Custom Accounting | 允许在生命周期回调中检查权限、调整费用或以自定义曲线处理 swap，支持发行期与二级交易期的不同规则 | Hook 是开发与审计责任，v4 不会自动提供曲线、KYB、毕业或反 MEV 策略 |

v2 使用每对资产的 Pair 合约，v3 使用独立的 Pool 合约，不应把 v3 也称为 `UniswapV2Pair.sol`。v4 的池身份由 PoolKey 计算，包括两种资产、费率、tick spacing 和 Hook 地址；它们在一个 PoolManager 中有各自状态，不是所有池的资产权益可以任意混用。

单例与净额结算能减少部分部署和转账工作，但本次没有运行链上 gas 基准。因此不使用“几十万 gas 必然降到几万 gas”的固定承诺。

## 4. Flash Accounting 与 EIP-1153 的准确边界

`CurrencyDelta` 使用 `TLOAD/TSTORE` 存储按账户和币种划分的净变化。它们是**瞬态存储**，不是普通 EVM memory，也不是跨交易持久状态；同一交易内的相关调用可以共享，交易结束后清除。

流程是：外围合约调用 `PoolManager.unlock`，在 `unlockCallback` 中执行 swap、流动性调整及结算，返回 PoolManager 前完成余额处理。`PoolManager.unlock` 检查 `NonzeroDeltaCount == 0`，否则回滚。

因此“交易结束前平账”的教学说法应具体化为：**每次 unlock callback 返回时，所有未结的账户/币种 delta 必须清零**。不是把不同资产换算成同一个价值后看总和为零。`settle`、`take` 以及适用的 ERC-6909 claim 操作仍需符合实际资产与账务约束。

链环境需支持 EIP-1153，并核对所采用版本的编译和执行兼容性；这是 v4 适配前提，不是已经确认 Mantle 需要或已经完成某项升级。

## 5. Hooks 数量与 beforeSwapReturnDelta

本次核对版本有 10 个生命周期回调：

- `beforeInitialize` / `afterInitialize`
- `beforeAddLiquidity` / `afterAddLiquidity`
- `beforeRemoveLiquidity` / `afterRemoveLiquidity`
- `beforeSwap` / `afterSwap`
- `beforeDonate` / `afterDonate`

另有 4 个返回 delta 的权限标志，共 14 个权限位。`beforeSwapReturnDelta` 是其中一项权限，**不是独立回调函数或额外生命周期节点**。实际自定义逻辑写在 `beforeSwap` 中，并返回 `BeforeSwapDelta`。

PoolKey 内的 Hook 地址固定，所需回调与返回权限必须正确配置；不能假设毕业时可以随意换成另一个 Hook 地址而仍是同一 PoolId。鉴权或 KYB 可以作为 Hook 策略，但身份来源与授权需专门设计，不能直接相信未经认证的 `hookData` 用户地址；调用 Hook 的 sender 也可能是 Router。

## 6. 虚拟储备曲线怎样接管一次 swap

以固定输入买入为例，用户输入非零的计价资产数量 `q`：

1. 已初始化的 v4 池接到请求，`params.amountSpecified = -q`。
2. `beforeSwap` 用 `(x+v_x)(y+v_y)=k` 计算应交付的 Meme 数量 `b`，检查授权、真实储备、可售数量、费用与最小到手量。
3. 在启用 `beforeSwapReturnDelta` 权限时，Hook 返回的 specified delta 为 `+q`，相应 unspecified delta 可为 `-b`，具体映射要遵守输入输出币种和模式。
4. `Hooks.beforeSwap` 计算 `amountToSwap = amountSpecified + hookDeltaSpecified = -q + q = 0`。
5. `Pool.swap` 收到零剩余量，原生集中流动性 swap 部分返回零 delta，即通常所说的 NoOp 效果。外部请求本身不能直接以零输入调用 `PoolManager.swap`，入口会拒绝零指定量。
6. Hook 与调用者的自定义收付仍进入账本，依实际资产或合法 claim 完成结算，所有未结 delta 归零后才能结束 unlock。

**NoOp 不是“关闭整个 PoolManager”，也不免除结算。** 这里只是本次原生定价引擎没有剩余数量可处理。Hook 同时接管了报价和对应履约责任，不能只返回一张“结算单”而没有可交付资产。

## 7. 从发行期到公开 AMM：不能只切换一个布尔值

候选产品设计可以从开始就使用同一 v4 PoolKey / PoolId：

- **发行期**：Hook 用虚拟储备曲线报价，自定义 delta 承接 swap；原生集中流动性引擎不处理被完全接管的那部分数量。
- **毕业转换**：验证门槛，检查真实 mStocks 与 Meme 储备，处理费用和余额，配置可用 LP、目标价格/区间、LP 权益和阶段状态，并确保相关变更一致提交。
- **公开交易期**：Hook 的逻辑分支不再抵消正常 swap 数量，让真实 LP 支持原生集中流动性交易；需要的权限或费用规则可继续保留。

原生池在创建时已经初始化，自定义曲线期间不一定同步改变其 `sqrtPriceX96`。毕业时必须设计原生价格与曲线末端价格的衔接，通过允许的操作完成状态处理，不能直接覆盖 PoolManager 存储或对同一 PoolId 再次 initialize。没有真实 LP 和正确价格，简单切换阶段标志不能完成毕业。

同池生命周期是本方案优先验证的产品形态，仍需实现与审计。若某项参数必须改变 PoolKey，或同池转换不能满足价格/流动性要求，则需要明确目标 PoolId 并设计原子接续，不应假称“永远无需迁移”。用户的 mStocks 和 Meme 仍有实际收付与权益配置，只是可能减少旧池到新池之间的外部搬运。

## 8. 对前后页面的影响

- P09：定义曲线、列举类型，并在同页点出毕业的储备迁移与非原子风险。
- P09A：把“停曲线、转真实储备、初始化/加 LP、开放交易”的状态与失败窗口画清楚，说明同链跨合约也可原子化。
- P12A：用 Singleton、Flash Accounting、Hooks 解释本方案为什么优先评估 v4。
- P12B：说明 `beforeSwap → 自定义 delta → 原生剩余量为零 → 结算`，以及毕业后的规则切换与真实 LP 条件。
- P11/P11A：用户仍看到“达到条件后进入二级交易”，开发者需知道可研究同池转换；不是所有路径都必须先迁到外部 DEX。
- P13：补上 EIP-1153 与 Hook 集成验证。v4 的多个市场共用 PoolManager，热点配额不能仅按该合约地址粗分，需按 PoolId/业务资源域等方式评估。

## 9. 官方依据与固定代码版本

源码已核对固定提交 **`46c6834698c48bc4a463a86d8420f4eb1d7f3b75`**，提交时间为 2026-04-02。检查过的 6 个源码文件与本次读取的 main 版本内容一致。

- [Uniswap v4 Hooks](https://docs.uniswap.org/contracts/v4/concepts/hooks)
- [Flash Accounting](https://docs.uniswap.org/contracts/v4/concepts/flash-accounting)
- [Custom Accounting](https://docs.uniswap.org/contracts/v4/guides/custom-accounting)
- [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153)
- [Hooks.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/libraries/Hooks.sol)：权限位、`beforeSwap` 的数量调整与 `afterSwap` 的 delta 合并。
- [PoolManager.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/PoolManager.sol)：`initialize`、`unlock`、`swap` 与账户余额检查。
- [Pool.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/libraries/Pool.sol)：`amountSpecified == 0` 时的原生 swap 返回行为。
- [IHooks.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/interfaces/IHooks.sol)：生命周期回调签名及 `BeforeSwapDelta` 返回语义。
- [CurrencyDelta.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/libraries/CurrencyDelta.sol)：账户/币种瞬态账本。
- [BeforeSwapDelta.sol](https://github.com/Uniswap/v4-core/blob/46c6834698c48bc4a463a86d8420f4eb1d7f3b75/src/types/BeforeSwapDelta.sol)：specified / unspecified 两部分的编码。

本地 [Launchpad 基础机制材料](../../2-meme-launchpad/outputs/02-meme-launchpad-fundamentals.md) 用于场景背景。其“只改路由指针就完成毕业”“零迁移”“天然消除 MEV”等绝对说法不作为本次实现结论。
