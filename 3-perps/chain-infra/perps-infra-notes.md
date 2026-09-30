# 为 perps 新增链原生支持的设计空间
> 研究轨道：M ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑
> 版式约定：`[已实现]` = 已有生产系统采用；`[本研究提案]` = 本文推导、尚无对应生产实现或仅部分实现

## 摘要

本文是**设计空间探索 + 可行性论证**底稿，不是现状盘点（现状对标见 → J 轨道 `J-appchain-comparables.md`）。核心方法：先用真实事故给 perps 的链级痛点立靶子（第 1 节），再列出已有解法的能力边界（第 2 节，交叉引用其他轨道），然后逐项推导「链还能做什么」（第 3 节，≥10 项，统一模板），用三轴给出优先级矩阵并挑出机会点（第 4 节），最后回答委托方最关心的战略问题——链级零件能否同时服务 perps 与资产发行（第 5 节）。

**版式约定重申**：`[已实现]` 表示本文写作时（2026-09-07）已有生产系统采用该机制（不论是否为 OP Stack / Mantle），并给出出处；`[本研究提案]` 表示本文推导的设计，目前没有任何生产系统完整实现（即便有部分构件先例，也在「是否有先例」字段中列出，不改变该项的提案属性）。若某项机制已被生产验证但尚未被 OP Stack 系链条采用，本文仍标 `[已实现]`，并在「在 OP Stack 上的改动量」中说明移植成本——这是区分「机制是否存在」与「机制是否已进入 OP Stack 语境」的关键。

## 1. perps 的链级痛点解剖

下表列出 perps 交易在链级面临的十类结构性痛点，每条都配真实事故 / 可验证证据，作为第 3 节设计提案的「靶子」。

### 1.1 oracle 延迟与可套利窗口

- **问题**：链上 oracle 更新有滞后（区块间隔、聚合延迟、多签/共识开销），在滞后窗口内，链下已知的真实价格与链上可读价格出现价差，形成确定性套利/操纵空间。
- **证据**：2022 年 10 月 **Mango Markets** 攻击（Solana 生态）。攻击者 Avraham Eisenberg 用 500 万美元 USDC 保证金在 Mango 上开出对冲的 MNGO 永续多空仓位，随后在**外部**交易所（FTX、AscendEX、Serum）拉盘 MNGO 现货价格（涨幅约 2300%），Mango 的 oracle 把这个被操纵的外部价格同步进链上保证金计算，使其虚增的仓位价值被用作抵押借出约 1.16 亿美元资产 `[一手]` https://www.justice.gov/archives/opa/pr/man-charged-110-million-cryptocurrency-scheme 、`[二手]` https://blockworks.com/news/mango-markets-mangled-by-oracle-manipulation-for-112m 。2025 年 5 月，美国联邦法官推翻了此前对 Eisenberg 的全部刑事定罪，理由是其行为是否构成「操纵」在法律上存在争议 `[一手]` https://www.trmlabs.com/resources/blog/breaking-federal-judge-overturns-all-criminal-convictions-in-mango-markets-case-against-avraham-eisenberg 。
- **链级含义**：oracle 更新与撮合/清算之间没有原子绑定，攻击者能在「链上价格滞后于外部真实价格」的窗口内完成操纵。→ 对应第 3 节提案 #1。

### 1.2 清算的 MEV 与 keeper 竞争外部性

- **问题**：清算奖励通常给「第一个提交清算交易」的地址，这天然是一场 gas 竞价/延迟竞赛；同时清算发生时刻往往与短时流动性枯竭重合，清算方与被清算方之间存在信息不对称。
- **证据**：2025 年 3 月 26 日 **Hyperliquid JELLY 事件**——攻击者用多账户在低流动性的 JELLY 永续合约上建仓（4.1M 美元空头 + 2.15M/1.9M 美元多头），故意撤出保证金触发强平；因订单簿深度不足，该空头仓位无法在市场上正常平仓，转由 Hyperliquid 的做市金库 HLP 被动接盘，随后攻击者在现货市场拉盘 JELLY 价格 400%+ 制造逼空，HLP 账面浮亏一度达 1000–1350 万美元 `[二手]` https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025 、`[二手]` https://oakresearch.io/en/analyses/investigations/hyperliquid-jelly-attack-context-vulnerability-team-solution 。Hyperliquid 验证者最终投票下架 JELLY 合约并用「oracle override」以固定价 0.0095 强制结算，此举因缺乏透明规则引发中心化质疑 `[二手]` https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025 。
- **链级含义**：清算失败后风险转嫁给谁（做市金库 / 保险基金 / ADL）是协议设计问题；清算触发和执行之间的时间窗口本身就是攻击面。→ 对应第 3 节提案 #2、#9。

### 1.3 下单撤单的 gas 摩擦

- **问题**：CLOB 做市商需要高频改价撤单（跟随中间价漂移），若每次撤单都要付 gas，等价于对报价行为直接征税，逼迫做市商放宽点差、降低挂单密度。
- **证据**：Hyperliquid 官方定位其自建 L1（HyperCore）的核心设计目标之一就是让下单/改价/撤单**免 gas**，官方文档明确「用户不为下单、修改、撤单支付网络 gas 费」，理由是消除其他链上「每次失败/更新订单都要计费」造成的费用流失（fee bleed），使高频做市策略在经济上可行 `[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore （HyperEVM 部分对 gas 结构的说明）、`[二手]` 综合技术介绍 https://phemex.com/academy/what-is-hyperevm-and-hyperliquid 。这从反面印证了「有 gas 的撤单」是通用链上做市商真实感知到的成本项，Hyperliquid 选择自建 L1 而非用现成 EVM 链正是为规避这个约束。
- **链级含义**：gas-free cancel 不是「优化」而是做市商愿不愿意来这条链的准入门槛之一。→ 对应第 3 节提案 #3。

### 1.4 撮合的状态争用（单一市场 = 单一热点账户）

- **问题**：一个永续合约市场的订单簿/持仓簿在状态层面是「单一账户」（或紧耦合的一组账户），所有该市场的下单、撤单、撮合、清算都必须对这个状态做写操作；并行执行引擎（无论是 Solana 的 Sealevel 还是 EVM 系的并行执行器）的加速原理是「不同交易的写集合不相交则可并行」，而同一市场撮合的写集合恒定相交，因此**撮合天然是单线程的，并行 EVM 在这里完全无效**。
- **证据（机制层面）**：Solana 的 Sealevel 运行时要求交易预声明读写账户集合，只读锁可并发、写锁互斥；当多笔交易竞争同一账户（「热点账户」，例如高热度 AMM 池或单一 oracle feed）的写锁时，运行时**被迫串行化**——这是 Solana 官方架构层面公开承认的限制，而非某次故障：热点账户处的竞争会造成交易被推迟、重试或丢弃，最终执行顺序在竞争下不确定 `[一手/机制说明]` 综合 Solana 官方文档与生态技术文章对 Sealevel 账户锁模型的描述（Solana Sealevel 设计文档体系，参见 https://solana.com/docs/core/transactions ，并可与生态分析交叉验证）。
- **链级含义**：这条痛点直接回应 brief 要求「论证清楚」的核心命题——不论 OP Stack 未来是否引入并行 EVM（如基于访问列表的乐观并行），**单一订单簿市场的撮合吞吐上限由该市场热点账户的单线程处理速度决定，与整条链的并行度无关**。这意味着"给 perps 加并行 EVM"本身不是有效的链级支持手段；真正有效的是第 3 节提案 #6（给热点账户一条确定性串行执行 lane，把它从"拖累全局并行度的瓶颈"变成"被显式承认并优化的独占通道"）和提案 #5（把撮合内层运算下沉到 precompile，用单次调用的常数时间换掉多次 SSTORE/SLOAD 的字节码开销，从"并行度"转向"单次执行效率"）。

### 1.5 爆仓级联与 ADL

- **问题**：极端行情下连环强平会造成「踩踏」：清算引擎为尽快平仓不断吃单造成价格进一步单边下跌，触发下一批清算；当保险基金无法覆盖穿仓亏损时，协议被迫用自动减仓（ADL）把亏损转嫁给对手方盈利仓位。
- **证据**：仍以 JELLY 事件为例——HLP 金库被动接盘后面临的就是「爆仓级联」的协议层最终兜底问题，Hyperliquid 选择「验证者投票下架 + oracle override 固定价结算」而不是标准 ADL 流程，本身说明其常规 ADL/保险基金机制在极端尾部场景下不足以应对，需要人工介入 `[二手]` https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025 。dYdX v4 把保险基金与 ADL 写成协议原生模块：保险基金优先吸收穿仓亏损，耗尽后才触发 ADL，按「盈利越多、杠杆越高优先被减仓」的规则强制平仓 `[一手]` https://docs.dydx.community/dydx/modules/governance/governance-adjustable-parameters/trading-core 、`[一手]` https://docs.dydx.xyz/concepts/trading/liquidations 。
- **链级含义**：ADL 触发规则、保险基金补充机制如果留在应用层合约里，治理延迟和合约升级摩擦会在极端行情中要命；协议化能让规则在链级不可绕过地执行。→ 对应第 3 节提案 #9。

### 1.6 funding 结算的 gas 成本

- **问题**：永续合约用 funding rate 让合约价格锚定现货，结算通常是"批量遍历所有持仓账户，计算并划转 funding payment"，账户数 N 越大，批量结算的计算/存储成本越高；如果由智能合约在 EVM 上做批量遍历，一次结算的 gas 会随 N 线性增长，最终必须收窄结算频率或改用"lazy settlement"（用户下次交互时才结算，代价是状态不一致窗口）。
- **证据**：dYdX v4、Hyperliquid 等把 funding 结算做成协议原生的周期性状态转换（不经过用户可读的"合约调用"，而是共识层在每个区块/每小时窗口自动应用），从而规避了"谁来支付这笔遍历 gas"的问题——但这是把成本从"显式 gas 账单"转移到"validator/sequencer 的隐性计算负担"，本质仍是 N 账户批量结算的成本，只是记账主体变了。⚠️存疑：本文未找到任何一手文档给出"funding 结算实际消耗多少计算资源 / 每个 tick 的账户遍历开销"的量化数据，这属于**已知未知**，列入存疑清单。
- **链级含义**：批量结算是否需要专门的链级原语（例如"惰性 funding 累加器 + 按需结算"，类似 Compound 的 cToken 利息累加器模式而非逐户遍历）是可行性问题，本文在第 3 节不单列一项，但在提案 #7（链级 margin）中一并讨论其数据结构含义。

### 1.7 跨保证金的原子性

- **问题**：交易者希望用同一份保证金抵押多个市场的仓位（cross margin），这要求"检查全体持仓的净风险"与"执行任意一腿的开平仓"必须原子发生，否则会出现"用同一份保证金被多个市场同时判定为充足"的双花式风险。
- **证据**：dYdX v4 的设计文档显式说明其默认是"跨保证金池"（markets 共享同一个抵押池和保险基金），并且专门推出了"隔离市场"（isolated margin）作为可选项，用来把高风险资产的保证金与主池物理隔离，这个"默认 cross + 可选 isolated"的产品分层本身就是在承认"跨保证金原子性 + 风险隔离"是两个互相冲突、需要工程权衡的目标 `[一手]` https://www.dydx.xyz/blog/introducing-isolated-markets-and-isolated-margin 。
- **链级含义**：如果保证金账本是原生状态（而非应用层合约里的一个 mapping），风险检查可以作为状态转换规则的一部分强制原子执行，而不依赖合约间调用的可重入性假设。→ 对应第 3 节提案 #7。

### 1.8 提现延迟与桥风险

- **问题**：Layer 2 到 Layer 1 的标准提现要等待挑战期（Optimistic Rollup 通常 7 天）或证明生成延迟（ZK Rollup），这段时间用户资金不可用；如果用第三方桥绕过延迟，就引入桥合约本身的安全假设。
- **证据（延迟侧）**：dYdX v3（基于 StarkEx）为解决这个问题推出"快速提现"：由做市商/流动性提供方在 L1 上垫付资金给用户，自己再走标准慢速提现从 L2 赎回，用户为此支付"gas 费与提现金额 0.1% 二者取高"的费用；这个机制的规模受限于 LP 垫付资金池是否枯竭 `[一手]` https://dydxprotocol.github.io/v3-teacher/ 、`[一手]` https://dydx.exchange/blog/fast-withdrawal-update 。dYdX v3 已于 2024 年 10 月 28 日停止交易/oracle 更新/funding 结算（v3 sunset）`[一手]` https://www.dydx.xyz/blog/v3-product-sunset ，本文引用其历史机制仅作设计参考，不代表现存产品。
- **证据（桥风险侧）**：2022 年 3 月 **Ronin 桥**（Axie Infinity 侧链的跨链桥）被朝鲜背景的 Lazarus Group 攻破，损失约 6.25 亿美元——9 个验证者节点中 5 个签名即可放行提现，攻击者拿到 Sky Mavis 的 4 个验证者私钥，外加一个此前因帮 Axie DAO 处理拥堵而遗留、且从未被撤销的第三方签名授权（allowlist），凑够 5 个签名后转走资金，事件发生后 6 天才被用户发现无法提现而曝光 `[二手]` 综合多方报道，含美国司法部相关通报背景 https://www.justice.gov/archives/opa/pr/man-charged-110-million-cryptocurrency-scheme （注：此 URL 为 Mango 案通报，Ronin 案主要信息源为链上分析与安全公司报告，本文未能定位到 Ronin 案的官方一手通报 URL，⚠️存疑，列入存疑清单）。
- **链级含义**：提现延迟和桥风险是一体两面——协议为缩短延迟引入的任何"信任外部签名者/LP"机制，都在重新引入桥的中心化风险敞口。→ 对应第 3 节提案 #11。

### 1.9 做市商的 latency 不公平

- **问题**：谁能更快看到订单簿变化、更快把交易打包进区块，谁就能在报价过期前完成有利可图的操作（抢先/狙击滞后报价）；物理距离、私有通道、与 sequencer/validator 的关系都会造成延迟不平等。
- **证据**：Hyperliquid 官方文档承认其订单簿微观结构专门做了"对做市商友好"的调整——共识层显式把撤单请求和 post-only（只挂单）订单的处理**优先于**普通 take 单，目的是防止"toxic flow"：即高频交易者在做市商的报价撤销生效前抢先"picked off"（吃掉）过期报价。文档说明这个设计让做市商不需要为了防止被狙击而报出更宽的点差 `[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore 、`[二手]` 补充说明 https://chainstack.com/hyperevm-evm-vs-nanoreth/ 。这本身是"cancel 优先于 fill"机制已经在生产环境解决 latency 不公平问题的实证，而非假设。
- **链级含义**：latency 公平不是抽象诉求，Hyperliquid 已经证明它可以被编码为排序规则的一部分。→ 对应第 3 节提案 #3。

### 1.10 失败交易成本

- **问题**：以太坊等 EVM 链上，交易执行失败（revert / out-of-gas）时消耗的 gas 不退还——协议设计上必须如此，否则恶意方可以免费发起会失败的高计算量交易做 DoS；但这意味着做市商/套利者的策略性挂单（例如条件性抢跑失败）要为"失败"本身付费。
- **证据**：以太坊正常网络条件下交易失败率通常报告在 2% 以下（即成功率 98%+），但在极端拥堵/热点铸造等场景下失败交易的绝对 gas 成本可观，单笔失败交易在拥堵时段被记录消耗数千至上万美元 gas `[二手]` 综合技术分析，未定位到单一可引用的一手统计源，⚠️存疑，列入存疑清单（本文未能找到官方/学术论文级别的"失败交易占比 / gas 浪费总量"权威统计，只有工程博客层面的定性描述）。
- **链级含义**：缓解手段主要落在"sequencer 侧模拟拒绝"（如 Flashbots Protect 类似私有 RPC 在进池前模拟交易，大概率失败者不转发）而非"protocol 退款"，因为退款在博弈论上等价于免费拒绝服务预算。→ 对应第 3 节提案 #12。



## 2. 已有链级解法的能力上限

本节只给能力边界的结论表，不重复深挖机制细节——机制细节与横向对标见 → J 轨道 `J-appchain-comparables.md`（Hyperliquid / dYdX v4 / Injective / Aevo 对比）、→ K 轨道 `K-sequencing-latency-infra.md`（排序 / MEV 零件目录）。

| 已有解法 | 解决的痛点（对应 1.x） | 能力上限 | 交叉引用 |
|---|---|---|---|
| 自建应用专属 L1（Hyperliquid HyperCore） | 1.3 gas 摩擦、1.9 latency 不公平 | 上限＝牺牲通用可组合性和 EVM 生态网络效应换取撮合性能；治理仍高度中心化（JELLY 事件的验证者投票+oracle override 即证据） | → J 轨道 |
| Cosmos 原生 exchange 模块（Injective） | 1.4 状态争用（部分）、1.9 latency | 把撮合下沉到共识层 Go 模块解决了"合约级"争用，但模块本身仍是链上单一状态机的一部分，热点账户问题在**该模块内部**依然存在，只是不再暴露给开发者；oracle 已转向外部 Chainlink Data Streams 而非纯原生模块 | → J、H 轨道 |
| Cosmos 原生 oracle 模块（vote extension，历史 Sei 模式） | 1.1 oracle 延迟 | 上限＝仍需要验证者在共识内投票达成价格共识，本质是"更快的多签"，不是"消除延迟"；且 Sei 已在 2025 年前后废弃原生 oracle 模块转向第三方 DON（Chainlink/Pyth/RedStone/API3），这是一次"预期存在的特性其实被放弃"的发现，值得记录 `[二手]` 综合 Sei 官方文档变更痕迹与生态报道，⚠️存疑（未定位到 Sei 官方"废弃公告"一手 URL，仅能确认当前文档已不再列出原生 oracle 模块），列入存疑清单 | → H、K 轨道 |
| StarkEx / Validium 快速提现（dYdX v3，已停用） | 1.8 提现延迟 | 上限＝LP 资金池容量，池子枯竭时退化为标准慢速提现；且该机制已随 v3 sunset（2024-10-28）不再运营 | 本轨道内部（历史参考） |
| Hyperliquid cancel 优先排序 + gas-free | 1.3、1.9 | 上限＝只解决"同一链内"的排序公平，不解决跨链/跨 venue 套利者的延迟优势；且优先规则本身由 Hyperliquid 团队设定，缺乏链下可验证的形式化保证 | 本节 1.3/1.9 已展开 |
| dYdX v4 保险基金 + ADL 协议模块 | 1.2、1.5 | 上限＝ADL 仍是"牺牲对手方盈利仓位"的零和转移，无法凭空消除穿仓亏损；极端尾部风险（如 JELLY 级别的单一低流动性资产被操纵）仍可能超出保险基金厚度，需要人工/治理介入 | 本节 1.5 已展开 |
| HyperEVM 读 precompile（`0x0800` 段） | 1.1（部分） | 只解决"EVM 合约同步读取 HyperCore 内部 oracle/仓位状态"，本质是给 EVM 开一个只读窗口，不改变 HyperCore 内部撮合与 oracle 更新的原生流程；这是"链级状态暴露给 EVM"而非"EVM 状态被链级消费"，方向是单向的 | → 第 3 节提案 #1 对比 |



## 3. 设计空间：逐项提出并论证

每项统一模板：机制描述 / 为什么必须在链级 / 实现形态 / 在 OP Stack 上的改动量 / 攻击面与失败模式 / 是否有先例。

### 3.1 `[本研究提案]` oracle 进入区块构建：价格先于撮合的确定性顺序

- **机制描述**：把 oracle 价格更新表达为一种特权系统交易（类似 OP Stack 的 L1Attributes deposit tx），由 sequencer/序列器在**每个区块的第一笔交易位置**强制插入，且该笔交易免 gas；同一区块内所有依赖该价格的撮合/清算逻辑只能读取「本区块已写入」的价格，不能读到「上一区块」的陈旧价格。清算和 oracle 更新在同一状态转换里原子发生（清算检查条件直接引用本区块 oracle 值，而不是通过独立合约调用二次读取，杜绝两次读取之间被插入的夹层交易）。
- **为什么必须在链级**：应用层合约当然可以在每次调用时"先读一次 oracle 合约再执行逻辑"，但这两次读之间的间隙本身就是 MEV 窗口——攻击者可以在"oracle 更新"和"依赖它的清算"之间插入交易（例如抢先补仓避免被清算，或抢先建仓吃清算价差）。只有把两者按同一笔状态转换原子打包，链级排序保证才能杜绝插入。这正是 Mango Markets 事件的根因：oracle 更新（通过外部价格）与保证金计算之间没有原子性，才给了操纵者窗口。
- **实现形态**：系统交易（比照 OP Stack `L1Attributes` deposit transaction 的模式，由 sequencer 在构建区块时插入，不经过公共 mempool）+ 状态转换层面的"先应用价格、再执行本区块其余交易"顺序约束。若要更强，可要求 oracle 更新交易与后续清算交易在**同一笔** L2 交易内完成（即清算 precompile 内部直接调用 oracle 更新，成功后立即执行清算判断），彻底消灭"两笔独立交易"的抢跑窗口。
- **在 OP Stack 上的改动量**：中高。需要修改 `op-node` 的区块构建规则（新增强制系统交易类型与插入位置约束）、修改 `op-geth` 执行层以识别该系统交易并使其对同区块内交易可见、需要 oracle 数据源（Chainlink/Pyth 等）与 sequencer 之间建立低延迟推送通道。不需要改共识（OP Stack 是单 sequencer，没有 validator 投票环节），比 Cosmos 系"vote extension"模式改动量小，但比纯应用层合约方案改动量大得多。
- **攻击面与失败模式**：sequencer 本身成为新的信任焦点——如果 sequencer 恶意延迟/篡改插入的价格，等价于 sequencer 自己操纵 oracle；这要求 oracle 推送本身有链下签名可验证性（sequencer 只是"转发者"而非"生成者"），且需要 fraud-proof/欺诈证明能覆盖到"sequencer 是否正确插入了 oracle 签名数据"这一断言。此外，免 gas 的系统交易若被滥用会造成 DoS 面（需要限制只有 sequencer 可插入，用户无法伪造）。
- **是否有先例**：部分构件有先例但完整方案无先例。Cosmos 系的 vote extension 模式（历史 Sei 原生 oracle 模块）实现了"价格必须先于区块提交达成共识"，但依赖的是验证者多签而非系统交易，且已被 Sei 放弃；HyperEVM 的只读 precompile（`0x0800` 段，`[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore ）让 EVM 合约能同步读到 HyperCore 内部已经处理好的 oracle 价格，但那是 HyperCore 原生撮合与 EVM 之间的单向窗口，不是"OP Stack 系统交易强制插入"的模式。本文未找到任何 OP Stack / Optimism 生态链把 oracle 更新做成系统交易的先例。

### 3.2 `[本研究提案]` 清算专用 lane / 优先区块空间

- **机制描述**：区块空间划出专用配额（例如每区块 gas 上限的固定比例，如 10–20%）只能被"清算类型"交易占用，且该配额在拥堵时**不参与**普通交易的 gas 竞价——清算交易按照"账户风险严重程度"（而非 gas price）排序进入这条 lane，保证无论主 lane 多拥堵，清算永远能在下一个区块被处理。
- **为什么必须在链级**：应用层合约无法为自己保留区块空间——区块空间分配是 sequencer/proposer 的特权，一旦拥堵，所有交易（不管是清算还是无关的 NFT mint）都在同一个 gas 竞价池里竞争，清算方要么加价（推高清算成本、间接传导给交易者）要么等待（造成穿仓风险敞口扩大）。这正是本文 1.2/1.5 节讨论的"清算级联"痛点的直接成因：Solend 等 Solana 借贷协议在网络拥堵时期确曾出现"清算机器人交易发不出去"的系统性风险描述（`[二手]` 综合报道，未找到官方一手事故报告，⚠️存疑，列入存疑清单）。
- **实现形态**：在 `op-geth` 的交易池（txpool）和区块构建器（block builder）中新增一个"交易类型标签"（类似 EIP-2718 的 TransactionType），清算类交易必须由风控 precompile 预先验证「此交易确实对应一个已越过清算阈值的仓位」才能打上该标签；区块构建规则里为该标签保留固定 gas 配额，普通交易的 gas 竞价逻辑不能挤占这部分配额，即使该区块清算交易数量为零，这部分配额也不能被普通交易借用（否则退化为"先到先得"，起不到保证作用）。
- **在 OP Stack 上的改动量**：中。核心改动在 `op-geth` txpool 与区块构建器新增交易类型/优先车道逻辑（可与 3.1、3.3 共享同一套"交易类型标签"基础设施），风控 precompile 需要读取当前保证金/仓位状态并做纯函数式判断（不修改状态），复杂度低于状态转移类改动；不需要动共识层。
- **攻击面与失败模式**：如果"打标签"的验证逻辑有漏洞，攻击者可以伪造清算交易骗取优先车道，事实上等价于免费获得优先出块权，可能被滥用做普通交易的抢跑通道；因此预验证 precompile 必须极其保守（宁可漏放真清算，不可误放假清算）。另外，固定配额在清算需求远超配额时（例如全市场级联爆仓，短时间内有几千个账户同时越过阈值）依然会拥堵，此时"lane 内部"又需要一个新的排序规则（例如按风险敞口大小），这本身是新的复杂度来源。
- **是否有先例**：本文未找到任何链在协议层面实现"清算专用、不可被挤占的区块空间保留"的先例。Solana 的优先费（priority fee）、Flashbots 的私有交易通道都只是"让愿意加钱的人排前面"，不是"保留配额"；这是本文认定的**无人实现的机制空白**之一，见第 4 节机会点。

### 3.3 `[已实现]` cancel 优先于 fill 的排序语义 + gas-free cancel

- **机制描述**：区块内交易排序时，撤单（cancel）与只挂单（post-only）类型的交易优先于吃单（taker/fill）类型交易被处理，且撤单/挂单/改价不收取 gas（只有实际成交的吃单方付费，通常通过 maker/taker 费率结构体现），从而保证做市商的报价撤回请求总能在被"捡漏"（picked off）之前生效。
- **为什么必须在链级**：这是 1.9 节已论证的"latency 公平"问题——应用层合约无法控制交易在区块内的相对顺序，只能接受 sequencer/validator 给定的顺序；如果 taker 单和 cancel 单同时到达 mempool，没有链级排序规则的话谁先被打包纯粹取决于谁的 gas price 更高或谁的网络延迟更低，这恰恰是对高频套利者有利、对做市商不利的默认状态。
- **实现形态**：Hyperliquid 的实现是把撮合引擎做成 L1 共识逻辑的一部分（Rust 原生，非 EVM），交易类型本身携带"cancel/post-only 优先级更高"的标签，HyperBFT 共识在排序阶段读这个标签。对 OP Stack 语境的等价实现：在 `op-geth` mempool 与 block-building 阶段新增交易类型区分（同 3.2 的 TransactionType 思路），并将"取消保证金/订单簿写操作"标记为免 gas 的系统调用类型（类似 3.1 的做法），排序优先级高于 taker 单。
- **在 OP Stack 上的改动量**：中。核心改动集中在 `op-geth` 交易池排序逻辑与 gas 计价规则（需要新增"该类型交易 gas 价格恒为 0 但仍计入区块 gas 上限，防 DoS"的例外规则），不需要改共识（单 sequencer），复用 3.1/3.2 已经引入的交易类型标签基础设施可以降低边际成本。
- **攻击面与失败模式**：免 gas 的撤单本身是 DoS 面——恶意方可以疯狂挂单再立刻撤单达到"刷屏占用区块空间但不用付费"的效果，Hyperliquid 的应对是限制每个地址的挂单/撤单频率配额（rate limit），OP Stack 移植时需要设计等价的防刷机制（例如按地址的历史成交量分配免费撤单配额，配额耗尽后退化为付费）。此外"cancel 优先于 fill"本身在撮合逻辑上要处理"cancel 和 fill 同时针对同一订单"的竞态——需要明确的 tie-break 规则（如"同一 slot 内 cancel 视为先于 fill 到达"）。
- **是否有先例**：`[已实现]`——Hyperliquid HyperCore 生产环境已验证该机制，官方文档明确描述"共识层优先处理 cancel 与 post-only 订单以防止 toxic flow"`[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore 。但该实现基于其自建非 EVM L1，本文未找到任何 OP Stack 系链条实现同等机制的先例——"移植到保留 EVM 兼容性的 OP Stack 派生链"这一具体路径本身仍是本研究提出的方案，需要新的 `op-geth` 层改动（见"在 OP Stack 上的改动量"）。

### 3.4 `[已实现]` 批量撮合 / frequent batch auction 作为链级原语

- **机制描述**：撮合不再是"订单到达即时连续撮合"（continuous limit order book），而是把一个时间窗口（如每个区块或每 N 毫秒）内到达的所有订单收集起来，在窗口结束时一次性求解出统一出清价格，同一窗口内所有成交都按这个价格结算，消灭窗口内部的"交易顺序"这个变量。
- **为什么必须在链级**：批量拍卖要生效，必须保证"提交订单的时刻"对最终成交价没有影响——如果这层批处理是应用层合约实现的，合约仍然要依赖底层链给交易排序，攻击者仍可以通过贿赂 sequencer 或利用私有交易通道在窗口内抢先获得有利位置（例如抢先知道其他订单流向后再提交）；只有共识层/排序层本身就不区分窗口内顺序（即窗口内所有交易按统一规则处理，物理上不存在"谁先谁后"的差异），批量拍卖的 MEV 免疫性才能被保证。
- **实现形态**：Penumbra 的做法是"每个区块结束时求解一次批量出清"，属于共识层内建的 DEX 引擎（不是 EVM 合约）；Injective 的 exchange 模块同样在原生模块层面做频繁批量拍卖（FBA），并通过 EVM 侧的 Exchange Precompile 把这个能力暴露给 Solidity 合约调用（合约只是"发起方"，撮合仍在原生模块完成）。对 OP Stack 语境：需要在 `op-geth` 之外新增一个"批量撮合原生模块"（类似 Cosmos SDK module 的思路，但要嫁接到 OP Stack 的账本模型上），或者退而求其次，用一个受保护的系统合约 + 排序层保证"该合约收到的所有调用在同一区块内按提交无关的规则处理"（弱于原生模块但改动更小）。
- **在 OP Stack 上的改动量**：高。真正的原生批量撮合模块需要在执行层新增一个"非 EVM 字节码路径"的原生模块（类似给 `op-geth` 加一个 Cosmos SDK 式的模块系统），这是架构级改动；退化版本（系统合约 + 排序保证）改动量中等，但保护力度弱于原生实现。
- **攻击面与失败模式**：批量拍卖仍需要一个"出清价格求解算法"，如果算法本身有 gaming 空间（例如允许"取消并重新提交"直到窗口关闭前一刻，变相恢复连续竞价的部分特性）就会削弱其 MEV 免疫性；另外，跨窗口的价格发现速度天然慢于连续撮合，对高频套利者不友好（这是設計取舍而非漏洞）。
- **是否有先例**：`[已实现]`——Penumbra 的 ZSwap 批量执行 `[一手]` https://protocol.penumbra.zone/main/dex.html 、`[一手]` https://guide.penumbra.zone/dex ；Injective 的原生 exchange 模块 + FBA `[一手]` https://injective.com/blog/understanding-injective-architecture-and-consensus 、`[一手]` https://injective.com/blog/injective-agents-the-platform-for-autonomous-ai-trading-agents （Exchange Precompile 部分）。两者均为生产系统，但都不是 OP Stack 架构；OP Stack 系链条实现原生批量撮合模块本文未找到先例。

### 3.5 `[本研究提案]` 撮合专用 precompile（定点数学、订单簿数据结构、批量匹配）

- **机制描述**：把限价单撮合的内层运算（价格-数量定点数比较、按价格优先/时间优先排序的 skip-list 或堆结构的插入/删除/查找、一次性批量匹配多笔对手单）实现为 EVM precompile（原生代码，而非 Solidity 字节码），Solidity 合约通过 `staticcall`/`call` 传入订单数据、拿到匹配结果，链上只需存储/更新最终状态而非在字节码层面执行排序算法。
- **为什么必须在链级**：Solidity 层面实现一个高效订单簿（例如红黑树或跳表）需要反复 `SLOAD`/`SSTORE`（每次 20000 gas 冷写入 / 2100 gas 热写入起），一次插入/删除操作往往涉及 O(log n) 次状态读写，n 是订单簿深度；而原生代码实现同样算法的时间复杂度相同但没有"每步都要过一次 EVM 解释器 + 状态树 trie 更新"的开销，理论上可以把同一操作的 gas 消耗降低一到两个数量级——这与"BLS12-381 配对运算加入 precompile 后从几百万 gas 降到几万 gas"是同一类杠杆（EIP-2537 背景）。
- **实现形态**：新增一个或一组 precompile 地址（比照 Hyperliquid HyperEVM `0x0800` 段的编址思路），暴露"insert order / cancel order / match batch"等接口；精确匹配结果仍然要写回 EVM 可读状态（供其他合约组合调用），因此 precompile 内部维护的订单簿数据结构需要有一份"状态根"或摘要写回普通存储槽，实现 EVM 合约与 precompile 状态的一致性绑定。
- **【量化估算，本研究自行推导，非实测】**：假设订单簿深度 n=1000（同价位若干笔），一次插入在纯 Solidity 跳表实现中估计需要 log2(1000)≈10 次比较 + 数次状态读写，按每次状态操作约 5000 gas（读热点存储 2100 + 写入摊销），保守估计一次插入/匹配约 50000–150000 gas；若通过 precompile 原生实现，按照现有 precompile gas 计价惯例（如 HyperEVM 的 `2000 + 65*(input_len+output_len)` 公式 `[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore ），同等复杂度操作大约落在 3000–8000 gas 量级，**估算降幅约 85–95%**。⚠️ 此估算基于类比推理与已知 precompile 计价公式外推，不是对任何已实现系统的实测结果，需在原型阶段用 benchmark 验证，列入存疑清单。
- **攻击面与失败模式**：precompile 是原生代码，其正确性不再有 EVM 字节码级别的"确定性由虚拟机保证"这层保护，任何实现 bug 都可能导致跨节点状态分裂（consensus failure）或直接被利用做资金安全漏洞——这是所有 precompile 类方案共同的高风险点，需要形式化验证或至少大规模模糊测试；此外 precompile 一旦上线就是协议的一部分，后续修复需要硬分叉，灵活性远低于可升级合约。
- **在 OP Stack 上的改动量**：高。新增 precompile 属于协议级改动，需要 `op-geth` 客户端硬分叉升级，且订单簿数据结构的正确性验证、gas 计价公式标定都需要新的测试基础设施；相比纯 Solidity 实现的合约可以随时升级，precompile 上线后修复 bug 需要再次硬分叉，运维成本显著更高。
- **是否有先例**：本文未找到任何 EVM 链把"订单簿匹配算法本身"做成通用 precompile 的先例。Injective 的 Exchange Precompile `[一手]` https://injective.com/blog/injective-agents-the-platform-for-autonomous-ai-trading-agents 本质是"EVM 合约调用原生 Cosmos 模块"的桥接层，撮合逻辑仍在原生 Go 模块里完成，precompile 只是调用入口，不是"把跳表/堆运算做成 precompile 指令"这个粒度；HyperEVM 的 `0x0800` 段是只读窗口，同样不涉及在 precompile 内执行匹配运算。这是本文认定的**机制空白**之一。

### 3.6 `[本研究提案]` 热点账户的确定性串行执行 lane

- **机制描述**：承认单一订单簿市场的撮合状态无法被并行化（1.4 节已论证），与其让它拖累全局并行执行引擎的调度效率（占用调度器反复检测冲突、重试的开销），不如显式声明"这个市场的账户是热点账户"，给它分配一条独占的、确定性串行的执行流水线，与其余可并行交易的执行完全解耦、并行运行，热点 lane 内部按到达顺序严格串行，不做任何冲突检测。
- **为什么必须在链级**：应用层合约无法声明"我需要独占执行資源"——执行调度（是否并行、并行度、冲突检测策略）是执行引擎的特权。如果不显式声明，通用并行执行引擎会对热点账户的每一笔交易都做一次"是否与其他交易冲突"的检测，这个检测本身有开销，且检测结果注定是"冲突"，等于白白浪费调度资源还没有获得并行收益——1.4 节已经用 Solana Sealevel 的账户锁模型证明了这一点。
- **实现形态**：执行引擎（`op-geth` 或未来 OP Stack 若采纳的并行执行框架）在交易分类阶段识别"目标地址属于已注册的热点账户集合"（如永续合约的订单簿系统合约地址），直接将其路由到一条单独的串行执行线程，不进入通用的并行调度器；这条 lane 可以运行在专用硬件资源（独立 CPU 核心）上，与其余交易执行完全解耦，只在写回全局状态树时做一次同步点。
- **攻击面与失败模式**：热点账户集合的"注册"机制本身是治理问题——谁能把一个地址注册为热点账户、享受独占执行资源？如果准入门槛过低，会被滥用为"抢占执行资源"的攻击手段（大量地址注册为热点，把独占 lane 挤爆，退化成比普通并行执行更差的吞吐）；此外热点 lane 与普通 lane 之间的状态依赖（例如清算需要同时读取热点账户的订单簿状态和普通账户的抵押品余额）需要额外的同步协议，设计不当会引入新的死锁或状态不一致窗口。
- **在 OP Stack 上的改动量**：高。这不是交易池/排序层的改动，而是执行引擎架构级改动——需要在 `op-geth`（目前是单线程 EVM 解释器）或其后继的并行执行框架中新增"lane 路由 + 独占调度"能力，如果 OP Stack 尚未采纳任何并行执行框架，这一项的前置依赖（先有并行执行引擎，再谈"给热点账户开后门绕开它"）本身就是一个大工程，需要与 N 轨道（OP Stack 可改造面）交叉确认现状 → 交给 N 轨道核实 OP Stack 当前执行模型是否已有并行化路线图。
- **是否有先例**：本文未找到任何链把"热点账户显式独占串行执行"做成协议原语的先例。最接近的类比是 Solana Sealevel 的写锁串行化（`[一手/机制说明]` https://solana.com/docs/core/transactions ）——但那是并行执行引擎的**副作用**而非**显式设计的独立 lane**，也是 Monad/Sei v2 等"乐观并行 + 冲突回退串行"架构试图缓解、而非主动利用的现象。本文的提案是把这个"副作用"正面转化为设计特征。

### 3.7 `[已实现]` 链级 margin / 风控：pre-trade risk check 进入状态转换

- **机制描述**：把"这笔交易是否会让账户跌破维持保证金率"这类风控判断做成状态转换规则的强制前置条件（类似 Cosmos SDK 的 `AnteHandler` 或 EVM 里的交易验证阶段），而不是等交易执行完再靠事后清算补救；保证金账本本身是协议原生状态（不是某个合约里的 mapping），任何改变仓位的交易都必须先通过这个前置检查才能进入区块。
- **为什么必须在链级**：如果风控只在应用层合约里做（`require` 语句），它仍然依赖交易被打包执行——恶意/极端行情下，一笔本该被拒绝的高风险交易仍然会消耗区块空间、可能在 revert 前对其他状态产生副作用（如果风控逻辑本身写得不够早）；把风控做成"进入状态转换之前"的前置门槛，可以在**交易验证阶段**就拒绝而不进入执行，减少无效计算和潜在的合约层竞态。
- **实现形态**：dYdX v4、Injective 等 Cosmos SDK 系统都是把保证金/风控做成链级模块（`x/clob`、`x/exchange` 等），而不是 CosmWasm/EVM 合约；EVM 语境的等价实现可以是 `op-geth` 交易池验证阶段调用只读 precompile 检查风险敞口，未通过检查的交易直接被拒绝进入 mempool（而不是进入区块后 revert，进一步节省该交易本可能消耗的区块 gas）。
- **在 OP Stack 上的改动量**：中。核心是在 txpool 准入规则里增加一个"风控 precompile 前置检查"钩子，这与 3.2 的清算标签验证复用同一套只读风险状态查询基础设施；不需要改共识。
- **攻击面与失败模式**：风控前置检查依赖的保证金状态如果与实际执行时的状态不一致（例如区块内前序交易已经改变了账户余额，但前置检查读的是区块开始时的快照），会出现"检查通过但执行时其实已经不满足"的竞态——需要保证前置检查与执行使用同一状态视图（要么严格串行处理同账户交易，呼应 3.6 的热点 lane）。
- **是否有先例**：`[已实现]`——dYdX v4 的保证金/清算模块 `[一手]` https://docs.dydx.xyz/concepts/trading/liquidations 、Injective 的 exchange 模块 `[一手]` https://injective.com/blog/technical-introduction-to-injective 均把风控做成协议原生状态转换的一部分。均为 Cosmos SDK 架构，非 OP Stack；OP Stack 系链条把 pre-trade risk check 做成 txpool 前置钩子（而非合约内 require）本文未找到先例。

### 3.8 `[已实现]` 原生 sub-account 与权限委托（API-key 式交易权限、session key）

- **机制描述**：主账户可以派生若干"仅限交易、不能提现"的子密钥（sub-account / agent wallet），子密钥可代表主账户下单、撤单、调整仓位，但任何转出资产的操作都必须由主账户签名；子密钥可设置有效期、每个子密钥有独立的 nonce 空间以支持多个自动化策略并发运行而不冲突。
- **为什么必须在链级**：如果权限委托只由应用层合约实现（例如一个"授权表"合约），撤销/过期逻辑仍然要通过一笔交易生效，存在"授权已被用户撤销但交易已经进入 mempool 尚未确认"的竞态窗口；把子账户权限模型做成协议原生的账户结构（而不是某个合约的内部状态），可以让"能否代表某地址下单"成为交易签名验证阶段就能确定的规则，而不是执行到一半才发现权限不足然后 revert。
- **实现形态**：Hyperliquid 的 Agent Wallet 需要主钱包先签一笔链上 `ApproveAgent` 交易来登记代理地址，此后代理地址可用自己的私钥签名下单请求，但没有转账权限，且信息类只读 API 请求仍需用主地址查询（代理地址查询返回空）`[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets 。dYdX v4 的 sub-account 模型允许一个母钱包下挂多个隔离的保证金子账户（编号 0-127），每个子账户风险互相隔离。EVM 语境等价实现可以是账户抽象（ERC-4337 风格的 session key）与本文场景专用化结合：把"session key 只能调用交易类方法、不能调用转账方法"做成协议层面强制的方法级 ACL，而不是依赖钱包合约自己实现的白名单逻辑（后者仍是应用层，有被绕过风险）。
- **在 OP Stack 上的改动量**：低中。如果基于 ERC-4337 账户抽象已有的基础设施（OP Stack 兼容 EVM，天然支持 ERC-4337），主要工作是在验证层为"方法级权限范围"提供协议层强制而非仅合约自愿遵守；如果从零做原生子账户结构（类似 dYdX 的 Cosmos SDK 模块），改动量上升到高。
- **攻击面与失败模式**：子密钥的 nonce 管理若被误用会导致签名重放——Hyperliquid 官方文档特别警示"注销后的代理地址不要重复使用，注销后的 nonce 剪枝可能导致此前签名的操作被重放"`[一手]` https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets ；权限范围定义过宽（例如"交易权限"是否包含"修改杠杆倍数"这种边界模糊的操作）也是常见设计缺陷来源。
- **是否有先例**：`[已实现]`——Hyperliquid Agent Wallet（`[一手]` 见上）、dYdX v4 sub-account（`[一手]` https://www.dydx.xyz/blog/introducing-isolated-markets-and-isolated-margin 提及子账户隔离概念）均为生产系统。OP Stack 系链条把"方法级权限"做成协议强制（而非 ERC-4337 合约自愿实现）本文未找到先例。

### 3.9 `[已实现]` 链级保险基金与 ADL 的协议化

- **机制描述**：保险基金作为协议原生的资产池（而非某个可升级合约里的余额），由清算费按固定比例自动注入；当穿仓亏损超过保险基金厚度时，自动触发 ADL（automatic deleveraging）——按照"盈利越多、杠杆越高"的规则强制平掉部分对手方仓位来对冲穿仓亏损，整个流程由协议状态转换规则强制执行，不依赖任何链下 keeper 或治理投票临时决策。
- **为什么必须在链级**：1.5 节已论证——如果保险基金/ADL 规则留在可升级合约里，极端行情下"要不要触发 ADL、按什么规则减仓"就可能变成一次治理提案或团队人工决策的时间窗口，而 JELLY 事件里 Hyperliquid 正是绕开常规流程、用验证者投票 + oracle override 处理，这恰恰暴露了"协议化程度不足、需要人工介入"的软肋。把规则写进协议状态转换（不可绕过、不需要额外治理），能保证极端行情下系统行为是**确定性**的，而不是"看当时团队怎么决定"。
- **实现形态**：dYdX v4 把保险基金/ADL 参数做成 `x/gov` 可调整参数、但触发逻辑本身是协议代码而非需要每次治理投票；保险基金资金来源为清算费的固定比例（governance 可调整比例，但资金流转本身自动执行）。
- **在 OP Stack 上的改动量**：中高。需要一个协议原生的资金池账户（不能是普通可升级合约，否则又退化为"合约层"方案，本项要求这个池子的扣款/注资规则本身写进状态转换代码，防止合约管理员绕过规则单方面支配资金）；ADL 的"按盈利/杠杆排序减仓"需要能高效遍历/排序全体持仓，这与 1.6 节"funding 结算 gas 成本"面临同样的"批量遍历 N 个账户"的数据结构问题，可与 3.7 的风控状态共享数据结构设计。
- **攻击面与失败模式**：ADL 规则的具体参数（阈值、减仓比例）如果被治理攻击者操纵，可能被用来定向让某些盈利仓位被优先减仓，这是治理攻击面而非技术漏洞；保险基金厚度设定过低仍会在极端尾部事件中被打穿，协议化不能消除"资金池不够用"这个根本约束，只能保证"打穿之后按规则处理"而非"打穿之后临时决定"。
- **是否有先例**：`[已实现]`——dYdX v4 `[一手]` https://docs.dydx.community/dydx/modules/governance/governance-adjustable-parameters/trading-core 。Hyperliquid 有 HLP 金库承担类似兜底角色，但 JELLY 事件表明其极端场景应对仍需人工介入，协议化程度低于 dYdX v4 的规则化 ADL。OP Stack 系链条本文未找到原生实现先例。

### 3.10 `[已实现]`（底层技术）／`[本研究提案]`（perps 场景组装） TEE / 加密 mempool 用于订单隐私

- **机制描述**：交易内容（订单价格、方向、数量）在离开用户客户端前先用链上可信执行环境（TEE，如 Intel SGX）的公钥加密，网络中继节点/sequencer 只能看到密文，只有进入 TEE 内部解密执行后，结果才重新加密写回状态；对做市商而言，这意味着报价意图在被执行前不会被其他参与者（包括 sequencer 自己）提前看到，从而防止抢跑。
- **为什么必须在链级**：应用层做订单隐私（例如链下暗池撮合、事后才提交结算）会牺牲"实时链上可验证性"，且暗池运营方本身成为新的信任中心；只有把加密-解密的边界设在共识/执行层（sequencer 和验证节点都无法看到明文），才能同时保留"链上可验证"与"执行前不可见"这两个目标。
- **实现形态**：Oasis Sapphire 的模式——客户端用 Sapphire ParaTime enclave 的公钥加密 calldata，交易在密文状态下传播，只有节点的 TEE（Intel SGX）内部才解密执行，执行结果重新加密后写回公共状态 `[一手]` https://docs.oasis.io/build/sapphire/ 。若要用于 perps 订单簿，需要把"下单"这个操作本身路由进一个类似的加密执行环境，撮合结果（成交与否、价格）在窗口结束后才解密公开，这本质是"TEE 版本的批量拍卖"（3.4 的加密变体）。
- **在 OP Stack 上的改动量**：高。需要节点硬件层面支持 TEE（不是所有验证节点/sequencer 硬件都有 SGX），且需要一整套加密 calldata 的客户端 SDK、enclave 远程认证（remote attestation）机制来让用户确信自己在跟真实的 TEE 而非伪造的软件模拟器通信；这是账本模型之外的基础设施投入，超出单纯的 `op-geth`/`op-node` 代码改动范畴。
- **攻击面与失败模式**：TEE 方案的信任假设从"代码公开可验证"转移到"硬件厂商的芯片没有后门/侧信道漏洞"——历史上 Intel SGX 已多次被学术界曝出侧信道攻击（如 Foreshadow、Plundervolt 等，本文未逐一核实每个 CVE 的现状，⚠️存疑，列入存疑清单，只做方向性提示：TEE 不是无条件安全）；此外 remote attestation 依赖芯片厂商的证书链，本质上引入了一个中心化的硬件供应商作为信任根。
- **是否有先例**：底层技术 `[已实现]`——Oasis Sapphire 是生产级的通用 TEE 加密 EVM `[一手]` https://docs.oasis.io/build/sapphire/ 、`[一手]` https://github.com/oasisprotocol/sapphire-paratime 。但**本文未找到任何主流 perp DEX 部署在 TEE 加密链上、专门用于订单隐私的生产案例**——这是一次值得记录的"预期存在但检索无果"的空白：TEE 技术本身成熟可用，但"perp DEX 订单隐私"这个具体应用场景尚无人把两者结合到生产环境，因此本条目标记为混合属性：底层原语 `[已实现]`，perps 场景的具体组装方案 `[本研究提案]`。

### 3.11 `[已实现]`（历史/停用）／`[本研究提案]`（协议原生化） 快速提现 / 结算通道

- **机制描述**：用户发起提现请求后立即（几秒到几分钟内）在目标链拿到资金，而不必等待标准的挑战期/证明生成延迟；实现方式通常是引入第三方流动性提供方垫付，再由 LP 自己走慢速通道把资金从源链赎回。
- **为什么必须在链级**：如果快速提现完全是链下 LP 与用户之间的场外协议（无协议层保证），用户要信任 LP 不会跑路，退化成一个新的信任第三方；把"垫付-赎回"流程的资金托管和清算规则写进协议状态转换（例如托管合约的资金流转规则由协议强制而非 LP 自由裁量），能降低但不能完全消除这层信任假设。
- **实现形态**：dYdX v3 历史上通过 StarkEx 生态的 LP 网络提供快速提现，用户支付"gas 费或提现金额 0.1% 取高"的费用，规模受 LP 池容量约束 `[一手]` https://dydxprotocol.github.io/v3-teacher/ 。这个模式本身没有把 LP 的行为约束写进协议层不可绕过的规则，仍然依赖 LP 的经济激励（赚取费用）自愿参与，本研究认为可以进一步协议原生化：把"LP 垫付 - 协议保证赎回优先权"做成协议原生的托管状态机，减少对 LP 自愿参与意愿的依赖。
- **在 OP Stack 上的改动量**：中。OP Stack 的标准 7 天挑战期是安全假设的核心（保证欺诈证明有足够时间被提交），任何"快速提现"方案本质上都是在这个 7 天窗口之外**叠加**一层做市商流动性层，而不能替代 7 天挑战期本身（除非改用不同的安全模型，如切换到 ZK 证明的即时终局性，那是完全不同量级的改动，不在本项讨论范围）。协议原生托管状态机的改动量集中在新增系统合约，不需要动 `op-node`/`op-geth` 核心逻辑。
- **攻击面与失败模式**：LP 池容量耗尽时快速提现退化为标准提现，用户体验不一致；协议原生托管如果设计不当，可能在 LP 与用户之间引入新的抢先/MEV 空间（例如 LP 可以选择性地只垫付对自己有利的提现请求）。
- **是否有先例**：历史先例 `[已实现]`（已停用）——dYdX v3 `[一手]` https://dydxprotocol.github.io/v3-teacher/ 、https://www.dydx.xyz/blog/v3-product-sunset （v3 已于 2024-10-28 sunset）。协议原生托管状态机（而非依赖 LP 自愿参与的场外协议）本文未找到先例，标记为 `[本研究提案]` 的部分。

### 3.12 `[本研究提案]` 失败交易免费 / revert protection 的链级实现

- **机制描述**：交易在真正被打包进区块、消耗 gas 之前，先经过一次协议层面的"模拟执行"，如果模拟结果显示必然 revert（例如滑点检查必然失败、清算目标已不存在），直接在 mempool 阶段拒绝，不消耗用户任何 gas；只有模拟通过的交易才进入区块并正常计费。
- **为什么必须在链级**：应用层无法拒绝已经上链的交易——一旦交易被打包，不论成功失败都已经消耗了区块空间和 gas，应用层合约的 `require` 只能决定"状态是否变更"，不能决定"这笔交易是否值得占用区块空间"；只有 mempool/sequencer 准入阶段的协议层规则，才能在"消耗资源"之前做拒绝。
- **实现形态**：类似 Flashbots Protect 这样的私有 RPC 已经在应用层做"提交前模拟，大概率失败者不转发到公开 mempool"，但这只是**某个 RPC 提供商的自愿服务**，不是协议强制规则，用户仍可以绕过 Protect 直接把必然失败的交易发给其他节点。本研究提议把"模拟-拒绝"做成 sequencer 的强制准入规则（而不是可选的第三方服务）：sequencer 收到交易后先在本地状态副本上模拟执行，只有模拟成功（或模拟失败但失败原因是"良性"如竞态而非用户自身错误）才允许进入区块构建候选池。
- **在 OP Stack 上的改动量**：中。OP Stack 目前是单一 sequencer 架构，天然具备"在打包前先模拟"的能力（不需要跨节点共识开销），核心改动是 txpool 准入规则新增"强制模拟 + 按失败原因分类"逻辑；比 Cosmos 系需要全体验证者达成共识的方案改动量小。
- **攻击面与失败模式**：强制模拟本身消耗 sequencer 计算资源，如果攻击者提交大量"模拟起来很慢但结果注定失败"的交易，会造成 sequencer 侧的计算 DoS（这类似于以太坊本身允许 revert 消耗 gas 的博弈论原因——protocol 层的"免费模拟"如果没有限速，本身就是新的攻击面）；因此需要给模拟阶段设置独立于正式 gas 计价的资源上限（例如"模拟计算预算"，超出预算直接拒绝而不管结果）。此外，"失败原因是否良性"的判定逻辑本身可能被针对性绕过，例如攻击者构造刚好卡在"良性失败"边界的交易反复试探。
- **是否有先例**：完整的协议层"强制模拟-免费拒绝"规则本文未找到先例。Flashbots Protect `[二手]` 综合公开文档描述，是最接近的构件先例，但其性质是**应用层/RPC 提供商自愿服务**，不是协议强制；Solana 即使交易失败仍收取基础费用（不属于"免费"）；NEAR 协议对未使用的 gas 有退款机制，但那是"部分执行后按用量计费"而非"执行前拒绝"。这是本文认定的**机制空白**之一。


## 4. 优先级排序

三轴打分（1=低，5=高），基于第 1、3 节的分析定性给出，非精确量化模型，用于排序而非绝对可比：

| # | 提案 | (a) 可感知收益 | (b) 实现成本 | (c) 去中心化代价 | 备注 |
|---|---|---|---|---|---|
| 3.1 | oracle 进入区块构建 | 5 | 4 | 3 | 收益直接对应 Mango 级事故防御，但 sequencer 成为新信任点，去中心化代价中等 |
| 3.2 | 清算专用 lane | 5 | 3 | 2 | 收益高（直接防清算失败级联），成本中（复用交易类型标签基础设施），去中心化代价低（不改变谁能验证，只改排序规则） |
| 3.3 | cancel 优先 + gas-free | 5 | 3 | 2 | 已有 Hyperliquid 生产验证，移植到 OP Stack 语境成本中等，去中心化代价低 |
| 3.4 | 批量撮合 / FBA | 4 | 5 | 3 | 收益高（MEV 免疫），但架构级改动（需要非 EVM 原生模块），成本最高 |
| 3.5 | 撮合专用 precompile | 4 | 4 | 4 | 做市商可感知收益中高（gas 降低），成本高（precompile 硬分叉运维），去中心化代价高（bug 修复灵活性差） |
| 3.6 | 热点账户串行 lane | 3 | 5 | 3 | 前置依赖"先有并行执行引擎"，成本最高且有依赖阻塞风险 |
| 3.7 | 链级 margin / 风控 | 4 | 3 | 2 | 已有多链验证，移植成本中等，去中心化代价低 |
| 3.8 | 原生 sub-account / session key | 4 | 2 | 1 | 可基于 ERC-4337 现有基础设施，成本最低之一，去中心化代价最低 |
| 3.9 | 链级保险基金 + ADL 协议化 | 4 | 4 | 3 | 收益高（防人工介入的信任危机），成本中高（需要原生资金池 + 批量遍历数据结构） |
| 3.10 | TEE / 加密 mempool | 3 | 5 | 4 | 底层技术成熟但组装到 perps 场景成本最高，且引入硬件信任根，去中心化代价高 |
| 3.11 | 快速提现通道 | 3 | 3 | 3 | 收益中（不影响交易时的核心体验，只影响出金），成本中，依赖 LP 流动性意愿 |
| 3.12 | 失败交易免费 / revert protection | 3 | 2 | 2 | 收益中（多数交易者对失败成本敏感度低于做市商），成本低（复用单 sequencer 架构），去中心化代价低 |

### 4.1 三个「高收益 + 中低成本 + 无先例」机会点

筛选标准：(a) 可感知收益 ≥4、(b) 实现成本 ≤3、(c) 第 3 节「是否有先例」字段明确写"未找到先例"（排除已被 Hyperliquid/dYdX 等生产验证的机制，即便分数达标，如 3.7、3.8、3.3 均因已有先例而不入选本清单，但仍是移植 OP Stack 的高优先级候选，只是不满足本节"机制空白"这一更严格的标准）。

1. **3.2 清算专用 lane**（(a)=5, (b)=3）——收益最高、成本可控，且是本文检索确认的机制空白：没有任何链在协议层保留"不可被普通交易挤占"的清算专用区块空间。直接命中委托方最关心的「无人实现的机制空白」标准。
2. **3.12 失败交易免费 / revert protection**（(a)=3, (b)=2）——成本最低（复用 OP Stack 单 sequencer 架构已有的"打包前可模拟"能力，不需要新增交易类型或改共识），且协议强制层面的"模拟-免费拒绝"规则本文未找到先例（Flashbots Protect 只是应用层自愿服务）。
3. **3.1 oracle 进入区块构建（弱化版）**——完整版（(a)=5, (b)=4）成本略高于"中低"门槛，但其中"清算与 oracle 更新的原子绑定"这一子机制单独抽出（不要求免 gas 系统交易，只要求两者在同一状态转换内完成），成本可降至中等（(b)≈3），仍是机制空白（历史 Sei 模式靠验证者投票而非原子绑定，HyperEVM precompile 是单向只读窗口），故以弱化版形式入选第三个机会点。

三者共同点：**都能在不改变 OP Stack 共识模型（仍是单 sequencer）的前提下，仅通过 sequencer/txpool/交易验证层的规则调整实现**，不需要引入原生并行执行引擎（3.6）、非 EVM 原生模块（3.4）或硬件级 TEE（3.10）这类架构级前置依赖，是"用最小改动换取最集中收益"的组合。

## 5. 与「资产发行原生」的共用零件

这是委托方最关心的战略问题：如果决定把整条链做成 app-specific，**是只服务 perps，还是能一套 infra 同时服务"资产发行"（→ L 轨道 `L-issuance-native-primitives.md`）**？下表逐项对照本文第 3 节的 12 项提案与 L 轨道盘点到的发行侧原语，标出哪些是同一个底层机制的两种应用、哪些是各自专用、无法复用。

### 5.1 共用零件（同一底层机制，两种应用）

| 链级零件 | perps 侧用途（本文提案） | 发行侧用途（→ L 轨道发现） | 共用程度判断 |
|---|---|---|---|
| **批量拍卖 / batch auction 原语**（3.4） | 消灭订单簿连续撮合的 MEV 窗口，统一出清价 | ixo `x/bonds` 模块的 `BatchBlocks` 机制——同一批次内所有买卖单汇总后按统一价格结算，"忽略订单到达顺序"，是发行侧的**抗狙击**核心手段（→ L 轨道 2.3.1 节，`[一手]` https://github.com/ixofoundation/ixo-blockchain/tree/main/x/bonds ） | **高度共用**——底层都是"批次汇总 + 统一出清价格"这一个数据结构与状态转换规则，perps 订单簿和 bonding curve 认购只是把同一个批量拍卖引擎接到不同的定价函数上。这是本文认定的"一套 infra 两个用途"最强证据。 |
| **优先车道 / 排序公平原语**（3.2 清算 lane、3.3 cancel 优先） | 保证清算不被挤占、做市商撤单不被狙击 | 发行侧的抗狙击同样需要"保证撤销认购/取消挂单不被抢跑"（HIP-1 荷兰拍卖的"卡死不退款"设计、ixo `MaxPrices` 价格保护均是同类需求的不同实现，→ L 轨道 2.1.1、2.3.1 节） | **中高度共用**——都需要"交易类型标签 + 区块内排序规则"这套基础设施（本文 3.1–3.3 已提议的 TransactionType 标签机制），具体标签定义（"清算"vs"认购取消"）不同，但排序引擎复用同一套代码。 |
| **原生费用路由**（本文 3.9 保险基金资金流；未单列但隐含于多项提案） | 清算费自动进保险基金、maker/taker 费率结算 | Solana Token-2022 `TransferFeeConfig`（转账费自动暂扣到 `withdrawWithheldAuthority`）、ixo `x/bonds` 的 `TxFeePercentage`/`ExitFeePercentage` 费用路由（→ L 轨道 1.1、2.3.1 节） | **高度共用**——都是"协议状态转换里自动划转一定比例给指定地址"这个原语，字段语义不同（清算费 vs 转账税）但底层数据结构（费率参数 + 目标地址 + 自动执行的转账触发点）相同。 |
| **原生 sub-account / 权限委托**（3.8） | Agent wallet 做市商 API 交易权限、dYdX 子账户风险隔离 | Injective permissions module 的角色分级（`MINT`/`RECEIVE`/`BURN`/`SEND`/`SUPER_BURN` 位掩码）、Sui Regulated Coin 的 `TreasuryCap`/`DenyCapV2` 分权（→ L 轨道 1.3.2、2.2 节） | **中度共用**——两者都是"给同一个资产/账户体系派生受限权限的子身份"，但 perps 场景的子账户是"交易 vs 提现"二元区分，发行场景的角色分级是"mint/burn/send/receive"多元位掩码，具体粒度不同，但都建立在"协议原生的账户/权限图"这个抽象之上，可以共享底层账户模型（一个通用的"能力位掩码"系统能同时表达两种场景的权限需求）。 |
| **原生 oracle 访问**（3.1） | 价格先于撮合的确定性顺序，清算原子绑定 | 发行侧对 oracle 的依赖弱于 perps（HIP-1 荷兰拍卖、ixo bonding curve 均为内生定价，不依赖外部价格），但**跨链资产/RWA 发行**（Injective permissions module 支持的 Ondo USDY、Agora AUSD 等）需要参考汇率或利率数据 | **低度共用**——perps 是 oracle 的重度刚需方，发行侧只有"锚定现实世界资产价格"这个子场景需要，多数原生发行（meme/bonding curve）完全不需要外部 oracle，因此这项零件的复用价值主要体现在"链已经有 oracle 系统交易基础设施"这一底座层面，而非"两边都同等需要"。 |
| **交易类型标签 / TransactionType 基础设施**（贯穿 3.1/3.2/3.3/3.7/3.12） | 免 gas 系统交易、优先车道、前置风控/模拟拒绝的技术地基 | 发行侧若要做"认购取消优先"、"KYC 前置检查"（Injective permissions 的 mint/receive 权限检查）同样需要这套地基 | **高度共用（基础设施层面）**——这不是某个具体业务机制，而是本文 3.1–3.12 多项提案共同依赖的底层能力（op-geth 交易分类 + txpool 准入规则扩展），一旦为 perps 建好，发行侧的抗狙击/KYC 闸门可以直接复用同一套交易类型标签系统，边际成本很低。 |

### 5.2 各自专用零件（无法复用）

| 链级零件 | 专用于谁 | 原因 |
|---|---|---|
| 撮合专用 precompile（3.5，定点数学/skip-list/heap） | perps 专用 | 订单簿撮合的数据结构（按价格-时间优先排序的堆/跳表）是 perps/CLOB 特有的计算模式，发行侧的拍卖/bonding curve 定价是"批量汇总求解"，不需要维护一个持续存在的、高频增删的订单簿数据结构，precompile 的具体指令集不通用。 |
| 热点账户串行执行 lane（3.6） | perps 专用（弱专用） | 发行侧的"热点"（如 HIP-1 荷兰拍卖的额度竞价）理论上也存在状态争用，但拍卖本身是**低频事件**（31 小时一场），热点持续时间和触发频率与 perps 撮合（每秒数千笔）不在一个量级，为发行场景单独建热点 lane 的收益不足以证明其工程成本；可以说这项零件"perps 场景才有必要建，发行场景就算能用也用不着"。 |
| 链级保险基金 + ADL 协议化（3.9） | perps 专用 | 保险基金/ADL 解决的是"杠杆头寸穿仓"这个 perps 特有的风险模型，发行侧没有杠杆和强平的概念（除非叠加杠杆代币/期权产品，那已经是另一个衍生品品类而非"发行"本身），此项完全不适用于纯发行场景。 |
| TransferHook / 转账钩子（→ L 轨道 1.1、1.2 节，Solana Token-2022、Aptos AIP-73） | 发行专用 | 转账钩子解决的是"发行方要不要在每次转账时校验/收税/限权"，perps 的仓位变化不是"转账"语义（是订单簿状态转换），这套原语在 perps 场景没有直接对应物。 |
| 荷兰拍卖式上币额度分配（→ L 轨道 2.1.1 节，Hyperliquid HIP-1） | 发行专用 | 这是"谁有资格发新币种"的准入机制，perps 场景的等价问题（谁有资格上新永续合约市场）理论上可以借用同一套拍卖原语，但本文与 L 轨道均未发现有链把"新增永续合约市场"做成拍卖制的先例（HIP-3 用的是质押门槛而非拍卖），暂不计入共用零件，列入存疑清单供后续验证。 |

### 5.3 战略含义

对委托方的核心疑问——"与其去外部找项目方，不如把整条链做成 app-specific 形式"——本节结论是：**如果目标是"一套 infra 同时服务 perps 与资产发行"，最具杠杆的投入顺序是先建批量拍卖引擎 + 交易类型标签基础设施 + 原生费用路由这三项高共用度零件，再在此基础上分别叠加 perps 专用的撮合 precompile/保险基金 和 发行专用的 TransferHook/拍卖式额度分配**。这意味着"app-specific chain"不必是非此即彼的单一应用专属，只要底层的排序公平 + 批量结算 + 费用路由基础设施设计得足够通用，"perps 优先"和"发行优先"两种路线在早期阶段的链级基础设施投入有实质性重叠，分叉点在于中后期是否追加撮合 precompile（perps 专精）还是 TransferHook（发行专精）。

## 存疑清单

1. **funding 结算的实际计算成本无量化数据**（1.6 节）：本文未找到任何一手文档给出"每个 funding tick 遍历 N 个持仓账户的实际计算/存储开销"的量化数字。dYdX v4、Hyperliquid 等把结算做成协议原生周期性状态转换后，这部分成本从"用户可见 gas 账单"转移到"validator/sequencer 隐性负担"，但没有找到任何论文或工程博客量化过这个隐性成本。**检索无果，按"无公开量化数据"处理，不代表成本不存在。**
2. **Ronin 桥事件缺官方一手通报 URL**（1.8 节）：本文对 Ronin 桥事件（2022-03，损失约 6.25 亿美元）的描述综合自多方二手报道与安全公司分析，未能定位到 Sky Mavis / Ronin Network 官方就此事件发布的一手通报页面 URL（司法部对该事件涉案人的正式起诉文件本文也未直接定位，仅确认存在美国政府对 Lazarus Group 的一般性制裁公告）。**结论方向（跨链桥的验证者签名集中度是系统性风险）可信，但具体细节数字建议读者与二手来源交叉核实。**
3. **以太坊失败交易 gas 浪费缺权威统计源**（1.10 节）：本文只找到"成功率通常 98%+"这类工程博客层面的定性描述，未找到学术论文或以太坊基金会官方发布的"失败交易占比 / 全网 gas 浪费总量"的权威统计数据。**检索无果，按"无官方权威统计"处理。**
4. **Solana/Solend 拥堵清算失败无官方一手事故报告**（3.2 节）：多篇二手技术文章描述"网络拥堵时期借贷协议清算机器人交易发不出去"的系统性风险，但本文未找到 Solend 或 Solana 基金会就具体某次事件发布的官方事故复盘文档。**这是本文用作论证"清算专用 lane 机制空白"的重要旁证，但证据强度低于一手源，读者应知悉。**
5. **Sei 原生 oracle 模块废弃缺官方一手公告**（第 2 节表格）：本文观察到 Sei 当前（2026-09-07）官方文档已不再列出原生 oracle 模块、转而推荐 Chainlink/Pyth/RedStone/API3 等第三方方案，但未定位到 Sei 官方发布的"正式废弃原生 oracle 模块"公告页面（可能随某次网络升级静默完成，或本文检索的文档版本恰好滞后/超前）。**这是"预期存在的特性其实已被放弃"的潜在发现，但因缺一手确认公告，标记为⚠️存疑而非确定结论，建议后续核实 Sei 官方升级日志（changelog）逐版本比对。**
6. **precompile 撮合 gas 降幅（85–95%）为类比推理，非实测**（3.5 节）：该估算基于"红黑树/跳表 Solidity 实现的状态读写次数"与"已知 precompile gas 计价公式（HyperEVM 的 `2000+65*len` 公式）"的外推类比，不是对任何已实现原型的 benchmark 结果。**原型阶段必须重新实测验证，当前数字仅作方向性参考，不可用于任何工程排期或成本预算的精确依据。**
7. **TEE 硬件侧信道漏洞现状未逐一核实**（3.10 节）：本文提及 Intel SGX 历史上曾被曝出侧信道攻击（如 Foreshadow、Plundervolt），但未逐一核实这些具体 CVE 截至 2026-09-07 的修复状态、是否仍适用于 Oasis Sapphire 当前部署的硬件型号。**只做方向性风险提示（TEE 不是无条件安全假设），不代表 Oasis Sapphire 当前生产环境存在已知未修复漏洞，也不代表其安全。**
8. **拍卖式永续合约市场准入是否有先例，检索未穷尽**（5.2 节）：本文与 → L 轨道均未发现任何链把"新增永续合约交易对的准入资格"做成荷兰拍卖或类似市场化机制（Hyperliquid HIP-3 用的是固定质押门槛 + 时间锁，而非价格发现型拍卖）。**检索范围限于英文公开资料，未穷举全部长尾 perp DEX 与 appchain，此判断可能被后续检索推翻。**
9. **OP Stack 当前执行引擎是否已有并行化路线图，本轨道未直接核实**（3.6 节"在 OP Stack 上的改动量"）：本文对"给热点账户开独占串行 lane"这项提案的成本评估，依赖一个前提假设——即 OP Stack 尚未采用任何并行执行框架（如果已经在路线图中，则本项的边际改动量会显著不同）。**此项需要 → 交给 N 轨道（`N-opstack-mantle-surface.md`）核实 OP Stack / op-geth 当前及路线图中的并行执行状态，本文不越界研究。**

## 关键来源清单

### 一手源（官方文档 / 源码 / 政府通报）
- Hyperliquid HyperEVM precompile 与 gas 结构：https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore
- Hyperliquid API 代理钱包（nonce 与权限管理）：https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets
- dYdX v4 治理可调参数（保险基金/ADL）：https://docs.dydx.community/dydx/modules/governance/governance-adjustable-parameters/trading-core
- dYdX v4 清算文档：https://docs.dydx.xyz/concepts/trading/liquidations
- dYdX 隔离市场/隔离保证金介绍：https://www.dydx.xyz/blog/introducing-isolated-markets-and-isolated-margin
- dYdX v3 快速提现（历史）：https://dydxprotocol.github.io/v3-teacher/ 、https://dydx.exchange/blog/fast-withdrawal-update
- dYdX v3 产品停用公告：https://www.dydx.xyz/blog/v3-product-sunset
- Injective 架构与共识介绍（含 FBA）：https://injective.com/blog/understanding-injective-architecture-and-consensus
- Injective Exchange Precompile：https://injective.com/blog/injective-agents-the-platform-for-autonomous-ai-trading-agents
- Injective 技术介绍（Exchange Module/CLOB）：https://injective.com/blog/technical-introduction-to-injective
- Penumbra DEX 协议文档（批量执行）：https://protocol.penumbra.zone/main/dex.html 、https://guide.penumbra.zone/dex
- Oasis Sapphire 开发文档（TEE 加密 EVM）：https://docs.oasis.io/build/sapphire/
- Oasis Sapphire 源码仓库：https://github.com/oasisprotocol/sapphire-paratime
- 美国司法部对 Mango Markets 案件通报：https://www.justice.gov/archives/opa/pr/man-charged-110-million-cryptocurrency-scheme
- TRM Labs 关于 Mango Markets 案定罪被撤销的报道（含法院文件信息）：https://www.trmlabs.com/resources/blog/breaking-federal-judge-overturns-all-criminal-convictions-in-mango-markets-case-against-avraham-eisenberg
- Solana 交易/账户模型官方文档（Sealevel 账户锁机制背景）：https://solana.com/docs/core/transactions

### 二手源（媒体 / 研究机构 / 技术博客）
- Hyperliquid JELLY 事件技术分析：https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025
- Hyperliquid JELLY 事件调查报告：https://oakresearch.io/en/analyses/investigations/hyperliquid-jelly-attack-context-vulnerability-team-solution
- Mango Markets 事件技术分析：https://blockworks.com/news/mango-markets-mangled-by-oracle-manipulation-for-112m
- HyperEVM 架构综合介绍：https://phemex.com/academy/what-is-hyperevm-and-hyperliquid 、https://chainstack.com/hyperevm-evm-vs-nanoreth/

### 本轨道引用的跨轨道发现（细节见对应轨道文件，本文不重复深挖）
- → J 轨道 `J-appchain-comparables.md`：Hyperliquid / dYdX v4 / Injective / Aevo 横向对标
- → L 轨道 `L-issuance-native-primitives.md`：Solana Token-2022 TransferHook/TransferFeeConfig、ixo `x/bonds` 批量拍卖抗狙击机制、Injective permissions module 角色分级、Hyperliquid HIP-1/2/3 发行经济学、Sui Regulated Coin 分权模型
- → H 轨道 `H-rise-chain-infra.md`、K 轨道 `K-sequencing-latency-infra.md`：排序 / MEV / oracle 延迟的链级零件目录（本文第 2 节仅给结论，不重复机制细节）
- → N 轨道 `N-opstack-mantle-surface.md`：OP Stack / op-geth 当前执行模型是否已有并行化路线图（3.6 节改动量评估的前置依赖，本文未核实，交给 N 轨道确认）

## 可复现方法附录

本轨道为**设计空间推导 + 一手文档研究**，未执行 RPC / 合约 / 官方 API 的实时数据采样（不涉及"链上状态实测"类工作，所有引用均为可通过上方 URL 直接复现的公开文档/源码/新闻）。

唯一的量化推导（3.5 节 precompile gas 降幅估算）复现方法如下：
1. 取 Solidity 跳表/堆实现的理论状态操作次数：`log2(n)` 次比较，n 为订单簿深度（本文取 n=1000 示例）。
2. 每次状态操作按 EVM 现行 gas 计价估算：热点 `SLOAD`=2100 gas，`SSTORE` 视冷热与是否清零在 2900–20000 gas 区间（本文取保守中位数估算，未做精确统计分布）。
3. 对比 precompile 侧的计价公式：以 HyperEVM 公开的 `2000 + 65 × (input_len + output_len)` 公式（`[一手]` 见上）作为同类系统的计价参照，代入典型订单数据的 `input_len`/`output_len` 字节数估算。
4. 两者相除得到降幅区间。**任何人可用上述步骤 + 更新后的 EVM gas 计价表（如未来 EIP 调整 SSTORE gas cost）重新推算，得到不同的具体数字；本文给出的 85–95% 只是当前（2026-09-07）计价规则下的粗估。**
