# Deeply Intents: Tilting at PropAMMs - Markus Schmitt (Propeller Heads)
## 全集双语对照转录全文 (Full Bilingual Transcript)

- **播客节目**：Deeply Intents
- **嘉宾**：Markus Schmitt（Propeller Heads 创始人，旗下技术堆栈包含 Tycho、Fynd 与 Turbine）
- **音频时长**：66 分 09 秒
- **音频文件**：
- **核心主题**：求解器与路由视角下的 PropAMMs、Solana 与以太坊专有做市机制架构对比、Tycho 超高速状态索引、Fynd 跨流动性路由引擎、Turbine 执行协议、逆向选择与毒性订单流防范、去信任化去中心化交易基础设施的未来演进。

---

## 目录 (Table of Contents)
1. [Part 1 (00:00:00 - 00:20:00) 什么是 PropAMM：从 Solana 的参数高频更新到以太坊的主动做市合约](#part-1-000000---002000)
2. [Part 2 (00:20:00 - 00:40:00) 区块内协调难题：12 秒出块、Gas 成本与块首价格同步](#part-2-002000---004000)
3. [Part 3 (00:40:00 - 01:00:00) Propeller 核心技术堆栈拆解：Tycho 索引、Fynd 求解与 Turbine 结算](#part-3-004000---010000)
4. [Part 4 (01:00:00 - 01:06:09) AMM、RFQ 与 PropAMM 三足鼎立：以太坊流动性终局](#part-4-010000---010609)

---


## Part 1 (00:00:00 - 00:20:00)

# 《Deeply Intents》播客深度访谈：Tilting at PropAMMs（嘉宾：Markus Schmitt）

## Part 1 核心概要与微观结构背景

本部分为 Propeller Heads 创始人兼 CEO Markus Schmitt 与主持人 Patrick 对谈的第一部分。核心议题围绕**专有做市自动化做市商（PropAMM / Proprietary AMM）**的底层微观结构展开，深入剖析了以下关键问题：
1. **PropAMM 的定义与本质**：PropAMM 是传统 AMM 与做市商询价机制（RFQ）的混合体。它既是部署在链上、具备原子可组合性的自动化做市合约，又赋予了特定链下做市商（Off-chain Market Maker）独占的动态参数更新特权（如动态修改点差、公允价格及联合曲线形态）。
2. **Solana 与以太坊更新机制的根本分野**：在 Solana 上，得益于极短的出块时间与极低的状态写入开销，做市商可以直接在链上发起高频参数更新；而在以太坊 12 秒出块时间与高昂 Gas 成本的约束下，PropAMM 必须与区块构建者（Block Builder）建立点对点的私下点对点协同通道，在块首（Top-of-Block）以捆绑包（bundle）形式注入最新鲜的报价参数。
3. **消除 RFQ 的“免费期权”与“路径依赖”顽疾**：传统“吃单方最后确认（Taker Last-Look）”的 RFQ 机制实质上赋予了套利者一个 30 秒有效期的免费期权，使做市商面临巨大的逆向选择损失（Adverse Selection / LVR），不得不向全市场拉宽报价点差。而 PropAMM 采用指示性价格流（Indicative Quotes），实际成交严格沿着链上联合曲线推进，既消除了做市商在并发请求下的路径依赖风险，又使外盘对冲成本与链上成交规模保持严格正向线性匹配。
4. **求解器（Solver）生态与路由生态重塑**：PropAMM 的兴起不仅不会杀死原子路由求解器（Atomic Routing Solvers），反而可能帮助求解器夺回此前被做市商直接撮合（如 CoW Swap 私有结算）蚕食的市场份额。因为多 PropAMM 并存的碎片化市场格局，依然高度依赖多跳与拆单路由算法。
5. **被动流动性（Passive LP）的长期生存空间与“无套利区间”**：做市商将持仓代币视作负债与风险敞口，而长期被动 LP 本身就渴望持有底层资产的 Beta 暴露，资金成本最低；加之 AMM 链上客观存在“无套利区间（No-Arbitrage Band）”，在特定行情瞬间，Uniswap v3 等无许可池在点差单侧的报价甚至优于币安与各大 PropAMM。

---

### [00:00:00 - 00:00:22]
**EN:** Good afternoon, Markus. Welcome back to the Deeply Intense podcast. Thanks for joining us today. Hey, Patrick. Great to be back. Thanks for inviting me again. Yeah, it's honestly a pleasure to catch up with you and a privilege. You're one of the people in the space that I think always is thinking a step ahead and is up to date on the latest trends and changes in market microstructure.
**CN:** 主持人（Patrick）：下午好，Markus。欢迎回到《Deeply Intents》播客，非常感谢你今天能来参加我们的节目。
Markus：嗨，Patrick。很高兴能再次做客，感谢你的再次邀请。
主持人（Patrick）：能和你交流确实是一件非常荣幸且令人愉快的事。在加密行业中，我认为你始终是快人一步、走在前沿，并时刻洞悉市场微观结构最新趋势与变革的人之一。

### [00:00:22 - 00:00:48]
**EN:** One of the things I wanted to kick off this episode before we jump into all the cool innovations that you're building with propeller heads and some of the product lines we're going to get into is just this question about PropAMMs. They've gotten very popular in the last six to eight months, starting off in Solana land and then kind of now migrating over to the EVM with a slightly different architecture and we can get into that and the reasons for that.
**CN:** 主持人（Patrick）：在我们深入探讨你正在 Propeller Heads 构建的那些炫酷创新以及稍后将展开的产品线之前，我想先以关于专有做市 AMM（PropAMM）的讨论作为本期的开篇。在过去六到八个月里，PropAMM 变得非常热门——它最初起源于 Solana 生态，而如今正以稍显不同的技术架构迁移至 EVM 体系中，稍后我们可以深入拆解这背后的机制与原因。

### [00:00:48 - 00:01:10]
**EN:** But I'd really like to just sit with this for a moment and maybe try to unpack it. If you could, just for the audience, talk about what is a PropAMM and maybe how is it different from some of the things that exist today, like just a standard AMM or maybe like an RFQ type system? I think it's a super interesting development, especially that they're coming to the theorem so fast at the moment.
**CN:** 主持人（Patrick）：我真想先停下来好好拆解一下这个话题。能否请你先向听众们解释一下，究竟什么是 PropAMM？它与我们现有的机制（比如标准 AMM 或 RFQ 询价系统）相比有何不同？
Markus：我认为这是一个非常引人入胜的进展，尤其是眼下它们正以极快的速度向以太坊（Ethereum）渗透。

### [00:01:10 - 00:01:48]
**EN:** A PropAMM is a proprietary AMM and it inherits aspects from two different systems that existed before, which is an AMM, it's in the name, and Market Maker RFQ, and the PropAMMs currently are mostly run by the same teams who before were running RFQs or the overlap is very large and the PropAMM is a smart contract that sits on chain, similar like any other AMMs. So in a sense, it is an automated Market Maker that by itself quotes and you can settle against it, it holds liquidity.
**CN:** Markus：PropAMM 即“专有做市自动化做市商”（Proprietary AMM），它继承了先前存在的两种不同系统的核心特性：其一是 AMM（顾名思义），其二则是做市商询价机制（Market Maker RFQ）。目前运行 PropAMM 的主体大多正是此前运营 RFQ 系统的原班团队，两者的参与者重合度极高。PropAMM 本质上也是一个部署在链上的智能合约，与其它常规 AMM 类似。从某种意义上说，它就是一个自动化做市商：合约自身对外报价，你可以在其上完成交易结算，并且它持有真实的链上流动性。

### [00:01:48 - 00:02:25]
**EN:** In that sense, it is an AMM. It's proprietary in that there is special rights granted to one off-chain party who can change parameters on that AMM. Historically, AMMs change their parameters very slowly only, maybe by governance vote, or they only automatically change by people trading on it or adding or removing liquidity. But now on PropAMMs, you have one off-chain actor who changes maybe the price or the spread or the shape of the curve, however they want, and they're the only one who can do that.
**CN:** Markus：从这个维度看，它确实是一个 AMM。而之所以称其为“专有（Proprietary）”，是因为该合约赋予了某一个特定的链下实体独占特权，允许其随意修改该 AMM 的内部参数。在过去，传统 AMM 参数的变动极为缓慢，通常需要通过社区治理投票，或者只能在交易者发生交易、添加或移除流动性时，状态才会被动地自动演进。但现在的 PropAMM 则引入了一个链下主体，他们可以根据自身的意愿，随意调整当前价格、点差（spread）甚至是联合曲线的形态（shape of the curve），而且唯独他们一家独享这一修改特权。

### [00:02:25 - 00:02:52]
**EN:** And that's why in the name is proprietary to them to make these changes. Excellent. I think that was a great starting point. So you have a party who can set up their own AMM contract on chain and they can update that contract that users can then trade against the liquidity in that contract. So I'm assuming that these PropAMMs are not necessarily building their own front-end. I'm assuming a lot of them are integrating with different DEX aggregators.
**CN:** Markus：这也正是其命名中“专有”的由来——因为唯独他们拥有专属权限来进行这些调整。
主持人（Patrick）：太棒了，这真是一个非常清晰的切入点。也就是说，某个做市机构在链上部署了属于自己的专属 AMM 合约，并有权实时更新该合约的状态，普通用户则可以直接与该合约中的流动性进行兑换。那我推测，这些 PropAMM 大概率不会单独去搭建面向终端用户的前端界面，应该主要是接入并集成到各大 DEX 聚合器中，对吧？

### [00:02:52 - 00:03:17]
**EN:** But in addition to that, how is the updating happening on a lower latency chain like Solana and then like how is the updating happening on Ethereum? Because Ethereum has 12 second block times as we know. So there would seem to need to be some kind of maybe streaming or something that's going on there. Maybe if you could elaborate on that. Yeah. So I'm not much Solana boy, I have to trust the articles that I read on this.
**CN:** 主持人（Patrick）：除此以外，参数更新机制在像 Solana 这样的低延迟链上是如何运作的？而在以太坊上又是如何实现的？因为我们知道以太坊的出块时间长达 12 秒，看起来似乎需要某种链下推流（streaming）或类似的特殊机制。你能就此展开聊聊吗？
Markus：好的。坦白说我并不是深度钻研 Solana 的专家，这方面我主要参考阅读过的相关分析文章。

### [00:03:17 - 00:03:50]
**EN:** But yes, of course, block times are much faster on Solana. You can include updates much more frequently. And what's specific to Solana or what's specific in Solana in comparison to Ethereum is that these particular updates are very, very cheap. Two, three, maybe four, I'm not sure, orders of magnitude cheaper than on Ethereum. And that allowed for frequent updates or updates multiple times per block on Solana. On Ethereum, something like this would be prohibitive even today.
**CN:** Markus：不过显而易见的是，Solana 的出块时间要快得多，更新交易上链的频率也高得多。与以太坊相比，Solana 的一个鲜明特性在于：在链上执行这些参数更新操作的开销极其低廉，大概比以太坊便宜两、三甚至四个数量级。这使得做市商在 Solana 上进行高频更新、甚至单区块内多次刷新报价在经济上完全可行。而在以太坊上，哪怕在今天，如果采用这种高频更新模式，高昂的 Gas 开销也是不可承受的。

### [00:03:50 - 00:04:27]
**EN:** And you only can have the guarantees or similar guarantees as you have on Solana if you in some way collaborate with the builder. And so on Ethereum, prop AMMs need to directly collaborate with the builder in order to make it a profitable business as far as I understand. Okay, that makes sense. So the market maker then is basically going to, they communicate directly with the builder and the builder is going to make sure that that update that they communicate the latest quote is the one that's included at the top of their contract and then users will execute against that latest quote.
**CN:** Markus：在以太坊上，只有以某种方式与区块构建者（Block Builder）展开深度协同，你才可能获得类似于 Solana 上的那种执行确定性与时效保障。因此据我了解，在以太坊生态中，PropAMM 必须直接与 Builder 合作，才能使其成为一门有利可图的可持续生意。
主持人（Patrick）：明白，这很合理。这么说来，做市商基本上是直接与 Builder 保持通信，Builder 负责确保将做市商传递过来的最新报价状态注入到合约顶部（Top-of-Block / 块首优先执行），随后用户的交易再基于这一最新报价完成撮合结算。

### [00:04:27 - 00:04:53]
**EN:** Can you talk maybe about like, what's the difference between a system like this, as you understand it, and something like an RFQ? What I was thinking is in an RFQ system, usually, not in all of them, but usually the user gets some kind of guarantee with the quote they're getting and it makes the market maker commit to that quote versus in this type of system, it seems like the user would be setting their slippage and just trading against whatever the freshest quote would be.
**CN:** 主持人（Patrick）：你能结合你的理解，谈谈这种机制与 RFQ 询价系统之间的本质区别吗？我的思考是：在 RFQ 系统中，尽管并非所有场景都是如此，但通常用户拿到的是带有确定性保证的报价，做市商必须对其签署的报价做出刚性承诺；而在 PropAMM 这种体系中，用户似乎只需设定好自己的滑点保护阈值，直接按照链上当前最新鲜的现成报价去兑换撮合。

### [00:04:53 - 00:05:21]
**EN:** Maybe if you could elaborate on that. Yeah, exactly. So an RFQ, it doesn't have to be, but most RFQs practically that are in use today and were in use up until today are take a last look RFQs. So the maker signs a quote and that quote is then valid on Ethereum, for example, on people, I think the standard is 30 seconds and within these 30 seconds you can execute against that quote.
**CN:** 主持人（Patrick）：能否就这一点详细讲讲？
Markus：没错，完全正确。在传统的 RFQ 中——虽然理论上不一定非要如此设计，但目前实际投入运营以及此前绝大多数的 RFQ 实现中，基本上都是吃单方拥有最终确认权（taker last-look）的模式。做市商对报价进行密码学签名，该签名在以太坊上有固定的有效期（比如业内通行的标准通常是 30 秒）。在这 30 秒的窗口期内，吃单方可以随时凭该签名去链上执行交易。

### [00:05:21 - 00:06:02]
**EN:** So it's a free option to execute at a specific price. And that is problematic because if the taker happens to be an opportunistic taker, someone who maybe is looking for a profitable arbitrage and they're constantly pulling quotes and then in that one second where a finance price chumps and the quote is now very attractive and you know you can close an arbitrage against this quote, then you take it. And so this adverse takers who opportunistic you take quotes ruin the game for everyone else on an RFQ because the market maker constantly loses money to these adversarial takers and then needs to increase the spread overall for everyone in order to balance the books.
**CN:** Markus：这就相当于赋予了吃单方一个以特定价格执行成交的“免费期权”（free option）。这种机制存在极大的隐患：一旦吃单方是投机性套利者（opportunistic taker），他们会不断向做市商轮询拉取报价；而就在某一瞬间，币安（Binance）等中心化交易所的价格发生跳变（price jumps），使得链上原有的 RFQ 报价瞬间变得极其便宜可图，套利者发现可以借此完成无风险套利平仓，便会立即吃下这张订单。这类趁机套利的逆向选择吃单方（adverse takers）实际上把整个 RFQ 生态的体验毁掉了——做市商面对这类恶意套利者不断遭受剥削割肉，为了平衡自身整体账目，就不得不面向全市场所有交易者普遍拉宽报价点差（spread）。

### [00:06:02 - 00:06:43]
**EN:** And it's quite different than a prop M. You don't get a guaranteed quote. You do as a solver get a quote stream. So if you are a whitelisted solver taker or aggregator prop MMS do stream you their prices the same as they stream them to the block builder, but they stream you indicative prices. There's no guarantee that you will settle at exactly this price. What price you get depends very much on what your position in the block is like with any other AMM and if someone else traded on that prop AMM ahead of you in the block and used a lot of the liquidity, then that price is not available anymore.
**CN:** Markus：而 PropAMM 的底层逻辑则完全不同。在 PropAMM 中，你得不到确定性的刚性承诺报价。作为求解器（Solver），你确实会收到一个连续的报价流（quote stream）——如果你是被列入白名单的求解器吃单方或聚合器，PropAMM 会像向 Block Builder 推流一样，将实时价格流推送给你，但这些都只是“指示性参考价格”（indicative prices）。它绝不保证你最终一定能以这个完全精确的价格完成结算。你能拿到的实际成交价，与其他常规 AMM 完全一样，高度取决于你在当前区块中的执行次序（position in the block）；如果在区块内部排在你前面的其他交易抢先在该 PropAMM 上成交并消耗掉了大量深度，那么最初看到的那个价格就已经不复存在了。

### [00:06:43 - 00:07:08]
**EN:** So the prop AMM solves a lot of issues. It solves the issue that the maker doesn't have to commit to a fixed quote and it solves the issue that the maker can, has path dependency between multiple quotes, I think this is something not much talked about. In RFQ a maker might give the same quote to five traders and they don't know whether one, zero or five of these traders are going to take that quote.
**CN:** Markus：因此，PropAMM 一举化解了诸多顽疾：它不仅免除了做市商必须被死死绑定在刚性固定报价上的风险，还完美解决了多笔并发报价之间的“路径依赖（path dependency）”难题——我觉得这个关键点平时很少有人深入剖析。在传统 RFQ 机制下，做市商可能同时向 5 个交易对手给出了同一份报价，但他们根本无法预判最终会有 0 个人、1 个人还是全部 5 个人同时行权吃下该报价。

### [00:07:08 - 00:07:33]
**EN:** So they don't know if they're going to settle 10, 100 K or 2 million in the next block until they actually see the settlement. But on a prop AMM it's like on any other AMM you know exactly on what curve you will settle. If there's five takers settling, it will move out further in the book and your spread will increase thereby your hedging costs will stay proportional to that. So these are at least two important differences.
**CN:** Markus：这就意味着，做市商在真正看到链上结算结果之前，根本不知道下一个区块里自己究竟会被撮合成交 1 万、10 万还是整整 200 万美元。但在 PropAMM 上，情况与任何常规 AMM 一样：做市商确切知晓交易是沿着哪条数学联合曲线进行清算的。哪怕同一区块内有 5 个吃单方连续涌入结算，成交价也会沿着曲线不断向深度外侧推移（滑点自发增加），点差随之扩大，做市商在外盘对冲这部分头寸的风险成本自然就与成交量保持了严格的正向线性补偿。这至少是两者之间极其关键的两个差异。

### [00:07:33 - 00:08:28]
**EN:** Let's take a step back for a second. So just to emphasize this point, so this is a contract that a particular party of a market maker will have on chain and it has a curve that they've defined with some mathematics and there is liquidity on that curve. And so users can trade against that liquidity. What you just mentioned was that if somebody trades against that curve before you do, then you might get a different price than that person who traded first. So not all prices against that prop AMM are going to be the exact same. Can you maybe just unpack this and highlight this point a little bit more? Yeah, I think that's an important, easily maybe overlooked because we're talking so much about off-chain actors getting quotes where it's not really that it's off-chain actors setting parameters on a curve.
**CN:** 主持人（Patrick）：让我们先稍微退一步。为了加深对这一关键机制的理解：做市商团队在链上部署了这个专属合约，合约内嵌了一条由特定数学公式定义的联合曲线，并在曲线上沉淀了真实流动性，用户便可以与这些流动性进行对手交易。你刚才提到，如果排在你前面的某笔交易先触碰了这条曲线，你所获得的成交价就会与第一个人截然不同。也就是说，向同一 PropAMM 发起的交易并不享有完全相同的执行价格。你能否针对这一点再做进一步剖析和强调？
Markus：是的，我认为这是一个极为关键但又极容易被忽视的核心细节。因为大家平时总在讨论链下做市商如何向外“分发报价”，但 PropAMM 的本质根本不是分发静态报价，而是链下做市商在动态设定链上曲线的几何参数。

### [00:08:28 - 00:09:14]
**EN:** And that curve draws from liquidity that sits on chain. And many of these curves look a lot like concentrated liquidity curves on UNISOPP3 for example, and as you trade through that curve, the price versions towards you. I mean, if someone trades in the other direction, you might even have a better price than what you expected. It can also happen. You could also have positive slippage on a prop AMM. In that regard, it really behaves the moment the parameters are set. I think it's most adequate to model it in your head as any other AMM. The only difference being that the parameters of that AMM, they can change up until the very last moment.
**CN:** Markus：这条曲线调用的完全是静止在链上的真金白银流动性。其中许多曲线在数学结构上非常贴近 Uniswap v3 的集中流动性（Concentrated Liquidity）曲线。当你吃进该曲线上的深度时，价格会因价格冲击而对你变得更差（price worsens towards you）；但反过来，如果排在你前面的有人朝着反方向交易，你甚至可能获得优于预期的更好价格——这完全有可能发生，也就是说在 PropAMM 上同样存在“正滑点”（positive slippage）。就这一点而言，一旦当期参数设置生效，在你的认知模型中，把它完全当成普通 AMM 来理解是最为准确的。唯一的本质区别仅仅在于：这个 AMM 的参数可以被做市商一直调整修改，直到出块前最后一毫秒。

### [00:09:15 - 00:10:04]
**EN:** Now, as a result of this, what happens to more trades that are being routed on-chain through some of these passive liquidity pools? Like, take for instance, as I understand, some of the cow swap solvers might on a given batch route some flow through on-chain automated market maker pools. Could be a few, it could be 60, it could be any kind of amount that would be efficient. If there now are prop AMMs that are plugging into some of these aggregators and some of these auction as potential routes that can be taken, does this now start to, at least for some of the major pairs, like let's say ETH/USDC, USDC/USDT, and so on, does this start to siphon flow towards the prop AMMs? Is this going to change how trades are routed on-chain?
**CN:** 主持人（Patrick）：那么顺着这个逻辑推导，那些原本通过链上被动流动性池（如普通 AMM）路由撮合的交易会面临怎样的处境？举个例子，据我了解，CoW Swap 的求解器在给定的某个结算批次（batch）中，可能会将部分订单流路由到链上各大 AMM 池中撮合——可能是几个池子，也可能是多达数十个池子的组合，只要达到最优效率即可。如果现在各大 PropAMM 作为可路由通道直接接入聚合器和批次拍卖中，这是否意味着至少在主流交易对上（比如 ETH/USDC、USDC/USDT 等），它们会开始虹吸原本流向普通 AMM 的订单流？这是否会重塑链上交易的路由生态？

### [00:10:04 - 00:11:34]
**EN:** Yes, and I think there are at least two factors that are impacting here. One is that if an asset's price discovery clearly happens off-chain because off-chain is much deeper liquidity and prices update more frequently, such as, for example, for USDC ETH pair, then the on-chain AMMs on a slow chain like Ethereum are always going to be behind in what they know about the price. So it's going to be trivially easier for any off-chain actor to give quotes that are more up-to-date and thereby quotes that take less risk. The less time between the time that you change your price last and the time that a trade settles, the less risk you take because the less can happen in the market, the less volatility on your mark-out and the tighter you can quote. And so latency has a direct relation to how tight you can quote, especially on such large timescales as 12 seconds. On smaller timescales, it eventually doesn't matter anymore. Now, this already happened quite a while ago that RFQs or the market makers who run them have taken a lot of liquidity and have taken a lot of trade volume from venues like Cowswap, Uni X, or One Inch Fusion by simply quoting better prices and the best AMM route could quote on those trades.
**CN:** Markus：确实如此，我认为背后至少有两个决定性因素在起作用。其一，如果某种资产的价格发现（Price Discovery）高度集中在链下（因为中心化交易所等链下市场拥有深得多的流动性和秒级高频价格跳变，例如 USDC/ETH 交易对），那么像以太坊这样出块缓慢的公链上的传统 AMM，其链上价格感知永远是滞后落后的。因此，任何链下做市主体都可以轻而易举地报出最新鲜的价格，从而承担更低的做市风险。做市商最后一次更新价格与交易实际链上结算之间的时间差越短，其承担的无常波动风险就越小——因为在这期间市场发生剧烈异动的概率极低，盯市盈亏评估（mark-out）的波动率极小，他们便敢于报出极度收窄的点差。因此，延迟与做市点差宽度有着直接的对应关系，尤其是在以太坊 12 秒这种巨大的时间尺度下更是如此（在极小的时间尺度下延迟的边际效应才逐渐递减）。事实上，这种趋势早在很久之前就已经显现：运营 RFQ 的做市商通过报出优于最佳 AMM 路由的价格，早已从 CoW Swap、UniswapX 或 1inch Fusion 等交易场所中蚕食了巨额的流动性和成交份额。

### [00:11:34 - 00:12:09]
**EN:** And so many solvers and on Cowswap, we can easily see it, have seen that a lot of flow has gone to these RFQs or actually more precisely to market makers who directly quote into Cowswap. Because quoting directing to Cowswap, they had a lower risk than giving an RFQ quote to an up-earth taker. Thereby they could quote even tighter on Cowswap than they would have ever given a public quote. And so these quotes that RFQ providers gave to solvers were always worse than their own bits that they did directly into Cowswap.
**CN:** Markus：我们在 CoW Swap 上可以清晰地观察到，许多求解器都目睹了海量订单流流向了这些 RFQ，或者更准确地说是流向了直接向 CoW Swap 拍卖内嵌报价的专业做市商。原因在于，直接在 CoW Swap 的受控批次拍卖中私有做市，做市商所面临的逆向选择风险远低于公开发放给外部套利吃单方的 RFQ 报价；正因如此，他们在 CoW Swap 内部能够开出比任何外部公开 RFQ 都要紧窄得多的报价。以至于 RFQ 做市商提供给普通求解器的公共报价，往往始终劣于他们在 CoW Swap 竞标中自己直接提交的投标出价（bids）。

### [00:12:09 - 00:17:35]
**EN:** And that was not because they were malicious having, but that was because they take much higher risk on the quotes that have like 30 second expiry versus knowing they're going to settle in, you know, they send it in the last 500 milliseconds to the builder and then they know they have only this 500 millisecond window of volatility. So this already happened before, before Prop AMMs. And now Prop AMMs are an even, let's say, they're kind of like a marriage of both, right? I can update my price to the last moment and I can keep my liquidity composable. I can keep it composable without taking the risks that I took before. And so I'd say a lot of the flow that Prop AMMs are going to get, I would suspect, are actually going to come from what previously went to direct market maker settlements on Cowswap and not necessarily directly out of the AMM pool. In fact, it might even be that atomic solvers, that routing solvers, gain back share significantly perhaps versus market makers because now they have access to quotes through the Prop AMMs that are similarly competitive as the market makers direct quoting into those venues. So it might actually be that we see more routed trades because there isn't one Prop AMM, there is now already five and there will be many more maybe. And so routing stays very relevant. And so yeah, and we might actually see that more trades get routed, and the volume shifts away from the direct settlements to the Prop AMMs, but it's the same parties in the end, it's the market makers who were quoted before directly. So then what happens to the incentive for providing on-chain liquidity now? Is this just something that's going to die? Eventually, like aside from obviously like where you made the key point already, which was price discovery, right? So for a number of assets you're going to have that are native to Ethereum, you're going to have price discovery on chain for sure. So nothing's going to die there. But in terms of ETH/USDC, ETH wrapped Bitcoin, or even some of the liquid staking majors, is that just now going to move entirely to Prop AMMs? So many answers to this. Will all of that trade volume go to Prop AMMs? No, not necessarily. There are many reasons, and I think some are even not mentioned once even yet. AMMs can, and in many cases, even in the cases where price discovery is off-chain, will provide better prices to traders than a maker ever could. And I'll have to say a little bit in the abstract, but I'll give one complete example. In the abstract, makers and passive liquidity providers are two very different, they have two very different outlooks. Makers want to make a profit versus a quote currency, maybe US dollar in most cases. And so every token they hold is a liability. It's a risk. It's something that needs to be compensated by profits or needs to be hedged, which costs as well. Whereas someone who deposits ETH into a Uniswap E2 pool and just lets it sit there for two years, you can easily argue that they want to hold ETH or they want to hold ETH in that exposure that the Uniswap E2 pool gives them. For them, holding ETH is not a liability. It is what they want. It is the desire they have. They want to be exposed to ETH. And any profit they get on top is just extra profit. They don't need to be compensated for a risk. And so long term, the cheapest liquidity is always from those who want to hold the assets. And those are never the professional parties. So that's in the abstract, I think one of the key advantages of passive, any sort of passive design, because or permission does design, because it allows those who want to have exposure to the asset to make that asset more productive and earn on it. We will see, I think, still several surprises of how this affects the market structure. But one small one that you can also observe lies if you go on, I think it's PAMM.WTF. You can see. So what many see when they see this dashboard, I think first is that, oh wow, look at how tight the PAMM's quote, even tighter than Binance in many cases. But if you look and you wait for a little bit and you look at the dashboard, but you also see that there are many moments where the Uniswap E3 pool quotes more tightly on one side of the spread than any of the PAMMs and even Binance. It's inside the spread of all of them. Yes. And it's like, how is that possible? Why is Uniswap E3 quote? And so if you were to trade on Uniswap E3 in that moment, you would get a better price than anywhere else, including off-chain. And those are not missed opportunities.
**CN:** Markus：这并非由于做市商怀有恶意，纯粹是因为承担的风险层级完全不同：为一个有 30 秒有效期的公开报价兜底，要承担整整 30 秒的价格暴露；而如果是在出块倒计时最后 500 毫秒将报价打包发送给 Builder，做市商深知自己面临的价格波动窗口仅有这短短 500 毫秒。在 PropAMM 诞生之前，这种分化就已经确立了。

而如今的 PropAMM 则更进一步，可以说是二者的完美联姻：我既能在最后一刻实时更新价格，又能让我的流动性保持链上完全的可组合性（composable）。在无需承受以往那种巨大时间敞口风险的前提下，依然享有可组合流动性的红利。因此我预计，PropAMM 未来捕获的大量订单流，实际上大多会来自此前原本被做市商直接撮合结算（比如 CoW Swap 私有结算）吃掉的份额，而不一定直接是从传统 AMM 流动性池中硬生生剥离出来的。

甚至更有意思的是，那些依赖链上多跳路由的原子求解器（atomic routing solvers）反而可能会借此重新从单边做市商手中夺回大量市场份额！因为现在这些求解器能够通过各大 PropAMM 获得极具竞争力的价格源，其紧密程度足以匹敌做市商此前的直接私有报价。加之链上并非仅有一家 PropAMM，目前就已经有五家，未来还可能更多。因此，多池间的高效智能路由依然至关重要。我们很可能会见证更多跨池路由交易的复兴：交易量从做市商直接结算转移到各大 PropAMM 上，但在底层，提供流动性的最终还是同一批原本在链下直接报价的专业做市机构。

主持人（Patrick）：那么对于在链上提供流动性的常规激励机制而言，接下来会发生什么？被动提供链上流动性难道真的要走向消亡吗？除去你刚才点出的核心锚点——价格发现之外（显然，大量以太坊原生发行的长尾资产，其价格发现必然牢牢扎根在链上，那里的流动性绝不会消亡），但在像 ETH/USDC、ETH/WBTC 甚至是几大主流流动性质押代币（LST）等主流品种上，未来的交易量真的会彻底被 PropAMM 全面接管吗？

Markus：这个问题有非常丰富的解答层次。是不是所有交易量都会全面倒向 PropAMM？绝非如此。这背后有诸多深层原因，其中一些甚至此前从未有人公开点明过。

事实是：传统 AMM 完全有能力——而且在很多场景下，即使资产的价格发现在链下——也依然能够为交易者提供比专业做市商更优越的成交价格。这在宏观理论上可能略显抽象，但我可以举一个具体的现实案例。

从理论抽象层面来看，专业做市商与被动流动性提供者（Passive LP）两者的风险偏好和底色完全不同。做市商的目标是赚取计价货币（通常是美元）的纯利润，因此对于做市商而言，手中持有的任何现货代币敞口本质上都是“负债”与“风险”，必须由交易利润予以补偿，或者必须花费真金白银进行外盘对冲（这本身也是成本）。

相反，一个把 ETH 存入 Uniswap v2 池子并安安稳稳放上两年的普通用户，你完全可以认为他的初衷就是长期持有 ETH，或者说他乐于接受该池子赋予他的资产暴露（Exposure）。对他来说，持有 ETH 根本不是什么负债，这就是他的本意与愿望——他就是想持有该资产的 Beta。在此基础上从池子赚到的任何手续费分成，都是锦上添花的纯额外收益。他根本不需要为了承担所谓的现货风险而要求风险补偿。

因此从终局来看，全市场成本最廉价的流动性，永远来自于那些本身就主观意愿长期持有这些资产的人，而这类群体绝不是专业量化做市机构。这就是任何被动化、无许可（permissionless）机制设计的核心根基——它让那些本就渴望持有资产风险暴露的人，能够将沉淀的资产投入生产性用途并赚取收益。

我们未来还会看到很多微观结构层面的演变惊喜。如果你现在打开数据看板（比如 pamm.wtf），就能亲眼目睹一个非常微妙的小细节：很多人第一眼看到这个看板时，往往会惊叹于 PropAMM 的报价竟能做到如此之窄，许多时候甚至比币安现货还紧。但如果你耐下心观察片刻，就会惊讶地发现：在很多时刻，Uniswap v3 池子在买单或卖单的某一侧（单边点差），其报价竟然比所有 PropAMM 乃至币安还要窄！它直接插到了所有对手盘点差的内部（inside the spread）。

主持人（Patrick）：是的。

Markus：很多人会纳闷：这怎么可能？为什么 Uniswap v3 竟然能报出这样的神仙价格？如果你恰巧在那一瞬间在 Uniswap v3 上发起交易，你获得的成交价将优于全网任何渠道，包括所有链下中心化交易所。而且这绝非偶然遗漏的定价套利机会。

### [00:17:35 - 00:18:13]
**EN:** It's because there is a no arbitrage zone between zero difference to Binance mid and double the spread, I think, and Binance mid. And in that zone, there's going to be no profitable arbitrage so that the pool will just float freely. And it will float and will often float into the place where the spread is lower than anything else offered. That's not intentional by the by the AMM, but this is one reason why AMMs will always receive at the moment significant flow because there's there's many moments where it still is the best the best option.
**CN:** Markus：这是因为在链上 AMM 与币安中间价之间，客观存在一个“无套利区间”（No-arbitrage band / zone）——它大致位于与币安中间价零偏差到两倍 AMM 点差（手续费率）的边界之内。在这个区间内，由于不足以覆盖往返手续费和 Gas 成本，套利者无法实施任何有利可图的套利，因此池子内的定价处于完全自由漂移的状态。而在漂移过程中，池子的单边报价经常会恰好漂入一个点差比外界所有做市商给出的价格都更紧窄的位置。这虽然并非 AMM 本身的主观意图，但却正是目前传统 AMM 仍能持续捕获巨额交易流的核心原因之一——因为在无数瞬间，它客观上依然是全市场最优的成交选项。

### [00:18:13 - 00:20:00]
**EN:** So yeah, I think I don't have many more concrete examples on how this affects the market structure, but that's one example. That was great. I didn't anticipate that answer, so that was really helpful and I appreciate you describing it on the dashboard because just looking at it while you're explaining it, like you do see there are times with like on the bid side the Uniswap E3 pool is quoting tighter than Binance mid or any of the proper AMMs that are listed, which is pretty incredible, but you just broke down the reason why because there's a no arbitrage zone. And if there's going to be certain instances where Uniswap E3 is going to quote better, it's going to exist. And I think what you noted was like really interesting about the preferences of a passive LP, which is just holding the asset. And they might not be rebalancing, they just like want to buy ETH over time, right? So they can even do 50% in the pool and then just as the price shifts, they own more ETH. And they're usually okay with that. It's actually an interesting way to enter a trade over the course of time if you're thinking about it in those terms and modeling it that way. Last question I want to ask you on proper AMMs before we kind of get into the propeller stack and some of the things you guys are doing and also forward-looking, well which is very exciting is, you know, there's a obvious like low-hanging fruit question to ask is, are we going to see more centralization at the block builder level as a result of this? Different block builders have different like skill levels in terms of infra. So like me, hobbyist block builder in my basement, maybe I don't have the ability to, you know, connect to a market maker and provide this level of service, but if I'm one of the big three or four block builders, I certainly would. And we We already saw that with competitively some.
**CN:** Markus：关于这会对市场微观结构产生怎样的演进影响，我目前可能没有更多现成的具体例证了，但这无疑是一个极具代表性的生动范例。
主持人（Patrick）：这个解析太精彩了！我完全没想到会得到这样一个答案，真的极具启发。非常感谢你结合数据看板所做的生动阐述，因为听你讲解的同时看着面板，确实能亲眼看到某些时刻 Uniswap v3 池子在买方一侧的报价甚至比币安中间价以及列出的所有 PropAMM 都要紧窄，这确实不可思议。而你刚才透彻地拆解了这背后的根源——无套利区间的存在。既然在特定情境下 Uniswap v3 的报价客观上更加优越，被动流动性就必定有其生存空间。
而且我认为你关于被动 LP 资产偏好的洞察极为深刻：他们单纯只是想长期持有该资产，他们不一定刻意去再平衡，甚至本质上就想随着时间推移逢低买入更多 ETH，对吧？他们甚至可以在池子里配置 50% 的仓位，随着价格下跌变动，被动沉淀持有更多的 ETH，他们对这种结果完全泰然处之。如果你从这个角度去理解并对其建模，这实际上是在时间维度上逐步建仓入场的一种绝妙方式。
在转向探讨 Propeller 的底层技术栈以及你们正在做的一些极其令人兴奋的前沿布局之前，关于 PropAMM 我还想请教最后一个直击要害的问题：这是否会导致区块构建者（Block Builder）层面的进一步中心化？不同 Builder 在基建能力和技术水准上参差不齐。如果是像我这样在地下室捣鼓的业余 Builder，恐怕根本没有能力与专业做市商建立低延迟直连并提供这种级别的协同服务；但如果是排名前三或前四的头部 Builder 巨头，则显然轻而易举。我们在当下的白热化竞争中其实已经看到了一些端倪……


---

## Part 2 (00:20:00 - 00:40:00)

# 《Deeply Intents》访谈 Markus Schmitt：对抗专有做市 AMM（Tilting at PropAMMs）- Part 2

## 核心要点与章节概要

本期访谈的第二部分深入探讨了做市商（MM）向专有做市 AMM（PropAMM）迁移对以太坊区块构建者（Builder）竞争格局的深远影响，并详细拆解了 Propeller Heads 团队构建的“自主主权（Self-Sovereign）”开源交易技术栈：

1. **构建者竞争与透明度带来的网络效应**：
   - 针对市场关于 PropAMM 是否会加剧顶级构建者（如 Titan、BuilderNet 等）垄断地位的担忧，Markus 提出了极具反直觉的行业洞察：智能合约的透明度与去信任特性形成了高效的区块内协调机制（coordination mechanism）。
   - 以往做市商向求解器提供流式私有报价需要极高成本的商务谈判（BD），容易形成排他性黑盒集成；而 PropAMM 作为链上公开合约，促使做市商主动、迅速地（数天至数周内）向所有新兴构建者全面铺开，反而强化了去中心化与构建者层面的充分竞争。
2. **底层流动性索引与纳秒级模拟引擎：Tycho**：
   - 传统求解器在进行链上路由计算时，面临严重的 EVM 虚拟机运行开销与状态滞后问题。Tycho（转录文本误记为 Tyco/Tiger）通过 Rust 本地化重写各类恒定函数做市商（CFMM）与专有做市 AMM 的数学与执行逻辑，将 Uniswap v2 的单次模拟延迟压缩至 900 纳秒（每秒百万次级）。
   - Tycho 彻底打破了长期困扰新型 AMM 的“冷启动流动性瓶颈”，让新协议无需依赖与头部聚合器的闭门 BD，即可通过开源 PR 获得来自 CoW Swap、UniswapX、1inch 等全生态的订单流。
3. **本地可验证生产级聚合路由器：Find**：
   - 针对中心化聚合器 API 带来的“委托-代理风险”（黑盒报价、恶意限流、交易回滚与滑点磨损），Propeller Heads 开源了生产级 DEX 聚合寻路引擎 Find。
   - Find 基于 Tycho 构建了全局动态市场图谱（Market Graph），集成 Bellman-Ford 等多种图遍历算法，支持多算法并行竞速与硬件自动弹性扩展。
   - 本地化运行将路由求解耗时从远程 API 的 500ms~2000ms 降低至 20ms（提速 20~100 倍），在大幅降低延迟与 Gas 开销的同时，重塑了以太坊“无需信任、只需验证”的交易基础设施哲学。

---

### [00:20:00 - 00:20:31]
**EN:** Some players kind of opting into this structure and announcing as such. So, you know, this is a naive question and I know it's very nuanced, but yeah, I'd just be curious. Your take on is this leads to a market structure where we have more kind of a lock-in centralization at the builder level where we just have a few teams or does this actually lead to a different set of outcomes because the incentives are such that actually, no, you'll still have multiple block builders and this doesn't really put any weight on the scale.

**CN:** 主持人：一些市场参与者已经选择加入这种架构并公开宣布了合作。所以，你知道，这可能是一个比较天真的问题，而且我也知道这里面有很多微妙复杂的权衡，但我真的很想听听你的看法：这是否会导致一种市场结构——即在区块构建者（Builder）层面上出现更多的绑定式中心化，最终只剩下少数几支寡头团队？抑或是因为激励机制的作用，实际上会带来截然不同的结果——也就是市场上依然会存在多个区块构建者，而这种机制并不会对中心化天平产生实质性的倾斜？

### [00:20:31 - 00:21:18]
**EN:** I saw something interesting happening in the shift from market makers to PropAMMs. And it's this remarkable efficiency of transparency. The PropAMMs, just by the nature of being a contract and then wanting to be discoverable by solvers and also, there's also probably some other aspects I'm missing, but the effect is very clear. It is the time between one builder having an integration with a PropAMM and every builder have an integration with the same PropAMM is incredibly short.

**CN:** Markus：在做市商向专有做市 AMM（PropAMM）的转变过程中，我观察到了一件非常有趣的现象，那就是“透明度带来的惊人效率”。由于 PropAMM 本质上就是一个链上智能合约，并且它们本身就极度渴望被求解器（Solver）发现——当然可能还有一些我尚未注意到的因素——但其产生的效果是显而易见的：从一家构建者完成与某个 PropAMM 的集成，到所有构建者全部完成对该 PropAMM 集成的时间间隔，短得令人难以置信。

### [00:21:18 - 00:22:05]
**EN:** I'm observing it in real time with the PropAMMs pinging us on Telegram being like, "Okay, we're now on this builder. We're now on this builder. Please integrate on this as well. If you submit trades here, you can also get our reports." And it's happening in the span of days and weeks. If I compare that to previously where we tried to get market makers to directly stream calls to solvers, which is like the same parties to bring the same quotes to the same people, there was a remarkable BD effort necessary to get these relationships up and running and to get these integrations. But by having this coordination layer Ethereum where the interface is clear, you have a PropAMM, it's a contract, it fuels the coordination so much. It is incomparable.

**CN:** Markus：我正在实时见证这一切：PropAMM 团队在 Telegram 上不断私信我们：“好的，我们现在已经支持这家构建者了。我们也支持那家构建者了。请你们也在这个链路做集成吧。如果你把交易路由提交到这里，你也可以获得我们的报价与执行回执。”这种扩散完全是在几天到几周的极短时间内发生的。相比之下，在过去的模式中，我们曾尝试让传统做市商直接向求解器推送流式报价——本质上是完全相同的两方，把相同的报价传递给相同的人——但那时为了建立这种商务合作并跑通技术集成，需要耗费巨大的商务拓展（BD）精力。然而，有了以太坊这个接口明确的区块内协调机制（coordination mechanism），你面对的是一个标准化的 PropAMM 链上合约，这极大地加速了各方的协同运作，两者完全不可同日而语。

### [00:22:05 - 00:23:18]
**EN:** And I'd say before it was more true, based on just what I observe right now, it was more true that it was easier for builders to have secret integrations or not secret, but proprietary integrations with certain teams. Whereas now, simply by the transparency, the PropAMMs are much faster to integrate with other builders. And it's surprising. When you talk to the PropAMM teams, it's not like they don't want to be with another builder. It's often simply like, "Oh, I don't have the contact. I don't have a contact. Can you make me an intro and then I will integrate there as well." So from afar, with a certain distance, I do see that the distribution amongst builders has actually happened faster than it did with previous arrangements, it seems to me. And the second thing is that I am continuously surprised by how competitive new builders can be. We've had some chats with some newer builders or smaller teams, and I'm impressed by how competitive they are and can be. So that makes me positive about the future of decentralization.

**CN:** Markus：仅就我目前的观察而言，过去的情况确实更倾向于构建者更容易与特定团队建立秘密的、或者即使不公开也是排他专有的集成关系。而现在，仅仅凭借链上的透明度，PropAMM 与其他所有构建者建立集成的速度要快得多。这非常令人惊讶。当你和那些 PropAMM 团队交流时，并不是说他们不愿意接入其他构建者，往往只是因为：“噢，我没有对方的联系方式，你能帮我引荐一下吗？引荐之后我马上也跟他们集成。”因此退一步从宏观视角来看，在我看来，流动性在不同构建者之间的分发速度实际上比以往任何合作模式都要迅捷。第二点是，我时常对新兴构建者的竞争力感到惊喜。我们与一些较新的构建者或规模较小的团队进行过深入交流，他们在当前以及未来展现出的竞争力给我留下了深刻印象。这也让我对以太坊去中心化的未来保持乐观态度。

### [00:23:18 - 00:24:00]
**EN:** Sweet. I think that was actually a great point to note, especially because it's easy to kind of fund some of these new things. When you just kind of look at the charts and you see, you know, every day, there's like, you know, Titans building 54% of the blocks and BuilderNet maybe has like 24% key stars up there, and so on. There's a couple other builders that's, you know, on some days will do well. But no, that's great to hear. And it's great to also hear about the competition in particular is what you're excited about. Because as we've seen in the Ethereum space, one thing that has been consistent and I think unique is the level of competition.

**CN:** 主持人：太棒了。我觉得这一点非常值得关注，特别是当人们仅仅看着统计图表时很容易产生悲观情绪——比如你每天看到 Titan 构建了 54% 的区块，BuilderNet 大概占 24%，还有 BeaverBuild 等头部构建者名列前茅等等。当然还有其他几家构建者在某些日子里表现也很亮眼。但无论如何，听到你这么说真是令人振奋。尤其是听到你对这种竞争格局感到兴奋，因为正如我们在以太坊生态中所见证的那样，始终如一且尤为独特的特质，正是其无与伦比的竞争烈度。

### [00:24:00 - 00:26:16]
**EN:** This is a good positive outlook to have. All right, so we talked a little bit about Prop AMMs. And I think you really did a great job of unpacking what they are, how they work, and some of the impacts on the trade supply chain are going into the future. I'd like to now just talk about the propeller stack, because there's a lot of exciting things going on there. You know, last year you came on the show, we talked about Tyco. And I think Tyco's adoption has been pretty significant. I'd love to give you an opportunity to just remind folks of what you're doing there and unpack that a little bit. And then we could talk about Find, which you recently open sourced and needs Tyco in order to really to work well. And then last, I'd like to get into Turbine, which is a very exciting project. A lot of cool innovation there. So first up, you've done a lot with Tyco. Maybe just remind folks what it is and kind of the adoption curve that you've been on and some of the things that you've learned about it. So what drove us for the last three years, probably two and a half years, for sure, is we want to build a trading stack that actually matches Ethereum's philosophy, a self-sovereign trading stack. One that you don't have to trust, but you can verify and you can run and you can inspect and it's transparent and you can modify it to what you need. And I think that it's important that that exists. I get more into maybe why later. And the first step, we had to build it from the bottom up because if you want to build open source, you have to start at the bottom of the stack. And that's also the part of the stack that we knew very intimately as solvers. We started out as solvers. I think we were quite good at it and we used it as a way to understand what really are the hardest problems that need to be solved. And I think we found some novel ways of solving these problems. And our answer or our open sourcing of that was Tyco. And Tyco is a unified interface to all on-chain liquidity that is Texas, but also RFQs and also proper AMMs. So any liquidity that you can access on Ethereum, even if it's not permissionless, you even have liquidity where you need to get an API key from somebody.

**CN:** 主持人：这是一个非常积极向好的视角。好的，我们已经深入探讨了专有做市 AMM（PropAMM），我认为你非常出色地拆解了它们到底是什么、内在运作机制如何，以及未来对交易供应链可能产生的深远影响。现在我想把话题转向 Propeller 的技术栈，因为你们团队最近有很多令人振奋的进展。去年你来我们节目时，我们聊过 Tycho（转录文本误作 Tyco）。我认为 Tycho 的采用率已经相当可观。借这个机会，能否请你向听众重新介绍一下你们在 Tycho 上的工作，帮大家梳理一下；接着我们可以聊聊你们最近开源的 Find（路由求解器），它必须依赖 Tycho 才能发挥出极致性能；最后我还想深入聊聊 Turbine，这也是一个非常令人激动、充满硬核创新的项目。那么首先，你们在 Tycho 上做了大量工作，能否回顾一下它究竟是什么、目前的采用曲线如何，以及你们从中总结出的经验？

Markus：在过去三年（至少是过去两年半）里，驱使我们前进的核心动力，就是打造一套真正契合以太坊底层哲学的交易技术栈——一个“主权自主（self-sovereign）”的交易技术栈。它是一个“无需信任、只需验证”的系统，你可以亲自部署运行、审查代码，它完全透明，你甚至可以根据自身需求随意修改。我认为这样一套基础设施的存在至关重要，稍后我会详细阐述原因。第一步，我们必须自底向上进行构建，因为如果你想打造开源生态，就必须从技术栈的最底层切入。而这一层也恰好是我们作为求解器（Solver）时最熟悉不过的部分。我们最初就是做求解器起家的，我认为我们做得相当出色，正是通过这段实战经历，我们深刻理解了整个交易链条中最硬核的痛点到底是什么。我们找到了一些全新的解决路径，而我们的答案，或者说我们将该方案开源的成果，就是 Tycho。Tycho 是面向所有链上流动性的统一接口，它不仅囊括了恒定函数做市商等各类去中心化交易所（DEXes，转录文本误作 Texas），还覆盖了询价系统（RFQ）以及专有做市 AMM（PropAMM）。换言之，只要是以太坊上能触达的任何流动性——哪怕不是完全无许可的、甚至需要向特定对手方申请 API Key 才能调用的流动性——它都能实现标准化统一对接。

### [00:26:16 - 00:27:02]
**EN:** And this was a product that was first aimed at solvers, market makers, searchers, everyone who is directly trading on AMMs and not indirectly through a router, but they have an interest to directly trade and interface with the AMM. And so a solver, for example, their job is they get a code request. I want to trade one E for UCC and then they are in a competition to find the best path. And in the course of doing so, they need to basically look for every possible path of how you could solve this trade and find what is the amount of UCC I would get on this path.

**CN:** Markus：这个产品最初的目标受众是求解器（Solver）、做市商（Market Maker）、搜寻者（Searcher），以及所有直接在 AMM 上进行底层交易、而非通过第三方路由进行间接封装交易的专业参与者——他们有强烈的意愿直接与底层 AMM 进行交互。以求解器为例，他们的核心职责是接收报价请求（RFQ）：比如“我想用 1 个 ETH 换取 USDC（转录文本误作 UCC）”，随后他们便进入了一场寻找最佳成交路径的激烈竞标。在此过程中，他们必须遍历搜索这笔交易的所有可行路径，并精确计算出在每一条路径下到底能换得多少 USDC。

### [00:27:03 - 00:29:29]
**EN:** And to answer that question, you need to accurately simulate every single DEX. And that is harder than it sounds because every DEX is implemented differently. There is no standard interface to them. Every DEX has different logic, even in some cases different languages and different issues. And you need to have code running on your machine that 100% accurately reflects what you would get on chain on this DEX. If you make a mistake, the trade might not settle, the user gets less than what they expected, it's not acceptable. And you won't find the best path if you are not accurately simulating those DEXs. And you can only accurately simulate those DEXs if you have the latest state that's relevant. So that might be what is the reserve in the pool right now. And every trade changes the reserves. And so at the moment the trade happens in one block, you need to immediately update your state locally. You need to notice that and update it and then apply that to your calculations. And then of course, you also when you have a trade, you need to execute against that AMM and do so safely without exposing your users to risk more yourself. And for each one of these three steps, you need to basically do an integration, pulling the right state from a new block, holding the right logic about how that AMM trades in your system. Usually that means rewriting the solidity logic in Rust, because if you go and simulate over the EVM, then you're going to be way too slow to simulate everything, way too slow, right? You have the whole VM overhead completely unnecessary. Like if you have Unity V2 X times Y equals K, you're going to be orders, I don't know how many orders, but several orders of magnitude faster than just pinging your VM and asking for the contract return on that trade. On Unity V2, for example, we have 900 nanoseconds for a simulation. That is, if I'm not mistaken, 1 million simulations per second. And this is relevant to everyone who has done this job and who is doing this job. And so that's around 25 teams, maybe, that I would know of. And so maybe, maybe you would have heard of Tyco, because that's relevant precisely to these 25 teams. And if you're not one of them, then you wouldn't use it.

**CN:** Markus：要回答这个问题，你必须对每一个 DEX 进行极其精准的链下模拟。这听起来容易，实际做起来极难，因为每个 DEX 的底层实现都千差万别。它们之间根本没有统一的标准化接口。每个 DEX 都有不同的数学逻辑，甚至在某些情况下底层编写语言和边界漏洞都各不相同。而你的本地服务器上必须运行着一套能够 100% 精确复刻该 DEX 链上真实执行结果的代码。一旦出现哪怕一丝细微误差，交易就可能无法结算（回滚 revert），或者导致用户拿到的代币少于预期，这在生产环境中是绝对不可接受的。此外，如果你无法精确模拟这些 DEX，你也根本不可能计算出全局最优路径。而且，精准模拟的前提是你必须实时掌握最新的链上状态——比如池子里当前的真实代币储备量（Reserves）。链上的每一笔交易都会实时改变储备量，因此当某个区块内发生交易的瞬间，你必须立刻在本地更新状态。你必须感知到这种状态变动、更新本地缓存，并将其代入后续计算。最后，当你选定路径后，你还需要向该 AMM 执行交易，并且要确保执行的原子性与安全性，既不让用户承担滑点风险，也不让自己遭受资金损失。针对这三个步骤中的每一步，你基本上都必须做繁重的工作：从新区块中提取正确的状态、在系统内部维护该 AMM 的交易算法。通常这意味着必须用 Rust 将 Solidity 合约逻辑彻底重写一遍。因为如果你直接调用 EVM 节点来运行模拟，速度会极其缓慢，慢到根本无法在毫秒级内完成庞大的路径模拟——你背负了完全不必要的虚拟机整体运行开销。例如像 Uniswap v2（转录文本误作 Unity V2）这种遵循 $x \times y = k$ 的恒定函数做市商（CFMM），在 Rust 中的纯数学模拟速度，要比向 EVM 发起 RPC 请求去查询合约返回值快上好几个数量级。以 Uniswap v2 为例，我们单次模拟仅需 900 纳秒。如果我没算错的话，这相当于单核每秒可以完成 100 万次模拟。这对于所有从事或曾经从事这项底层交易工作的人来说都是生死攸关的。在业界，我所知道的这类顶级团队大概有 25 支左右。因此，你可能听说过 Tycho，正是因为它精准击中了这 25 支核心团队的刚需。如果你不在此列，你可能根本不会接触到它。

### [00:29:29 - 00:31:18]
**EN:** Let me ask you this, though, because, you know, this was a core insight that you guys had. And clearly, you identified this friction because you had to integrate all these pools yourself. And this takes engineering muscle and time. And it's frustrating and annoying as you kind of just laid out. Like, why open source that, though? Because if, you know, that could be an edge for you in, you know, winning solving competitions, and you can just keep winning because you have this edge, like, what about open sourcing? It is beneficial for the ecosystem. And the reason I'm asking this is because a lot of people have gone back and forth recently on the benefits of open sourcing things, as people are trying to financialize different layers of their product stacks. So, yeah, just be curious your thinking on this. I mean, there are many reasons to this. I think one interesting one is that, and I keep noticing that there are many factors that make software better when it is open source. I think off the top of my head, engineers probably take it more seriously if they know that it's open source. But that's probably a small factor. A bigger one is that it gets hardened so much, right? You have so many more teams using it. We had so many solvers using it in production. Their revenue depends on it, right? Their business depends on it. And so, they are going to tell us every little thing that needs to be fixed. And we will fix every little thing because we want them to keep using it. And so, the amount of improvements that happen over a short period of time, and Tiger has fully been live for around a year is incredible. It is faster than if you just used it yourself. Much faster. And so, your software hardens to a degree that is simply not possible. And there are at least these two factors, right? There's pride.

**CN:** 主持人：不过我想追问一句，这是你们团队的核心洞察，而且很明显，你们之所以能识别出这一摩擦痛点，是因为你们曾经不得不亲力亲为去集成所有这些流动性池。正如你刚才所言，这需要耗费极其强大的工程实力与漫长的时间，整个过程既繁琐又令人头疼。那你们为什么选择把它开源呢？因为如果保留闭源，这完全可以成为你们在求解器竞标（Solving Competition）中屡战屡胜的核心护城河（Edge），让你们凭借这种竞争壁垒持续获利。开源究竟能为生态带来什么？我之所以问这个，是因为随着人们试图将产品栈的各个层级金融化，最近很多人对“开源到底有何实际益处”展开了激烈交锋。所以，我很想听听你们在这个问题上的深层考量。

Markus：这背后的原因有很多。我认为其中一个非常有趣的维度是，我反复观察到：开源模式本身蕴含着许多能让软件变得更加强大的内在动力。首先直观的一点是，工程师一旦知道代码将要完全公开开源，往往会以更高规格的严谨态度来设计和实现它。不过这只是一个次要因素。更关键的核心原因在于，软件会因此得到极高强度的“实战淬炼（hardening）”。有海量的外部团队在运行它，我们有非常多的顶级求解器在生产环境中高频使用它，他们的真金白银和营收命脉都维系在上面，对吧？他们的商业大厦直接构建于此。因此，代码中只要出现任何一处细微瑕疵或性能回退，他们都会第一时间向我们反馈；而为了让他们持续使用，我们也会不遗余力地修复每一个细节。因此，在极短的时间内——Tycho（转录文本误作 Tiger）在生产环境中全面上线大概才一年左右——它所经历的质量飞跃与性能进化是惊人的。这种迭代速度远快于仅供内部闭门自用时的状态，快得多得多。你的软件在可靠性、鲁棒性与边界条件处理上，被淬炼到了内部闭源开发根本不可能企及的高度。这其中至少有两股强大的推动力：一是工程师的荣誉感（pride）——

### [00:31:18 - 00:32:48]
**EN:** You want it to be really, really well structured for the short and long term. And there is pressure, which is customers want it to be working. And that's one important aspect. But that alone is not enough. It's like, okay, where's the business? I think we wanted this infrastructure to be as efficient and stable as possible because we knew that there were other things to build on it that would be very valuable to have. That makes a lot of sense. I really like the argument about hardening the software because that actually allows you to then utilize it in a way where it can be more reliable. It can be a dependency that you feel good about, or at least that you don't feel bad about and worry about, which is nice because then that could potentially unlock some other products. So, that's interesting. That's another important factor here. For the longest time, we've seen AMMs be very frustrated by the whole integration process. It was a very big E-having process. You need to find the team. And if you were a new AMM, you didn't have connections. How are you going to reach out to all of these aggregators and convince them that they should integrate you? They're going to say, you have no volume on your AMM. I mean, I don't care. 10 other AMMs that want to integrate and they are paying me in some cases, maybe. And that is really frustrating. It's an important bottleneck for AMM innovation, this bootstrapping problem.

**CN:** Markus：一方面是你内心的工程师荣誉感，你希望无论从短期还是长远架构来看，代码都极其严谨稳健；另一方面是外部的真实压力——客户要求它必须稳定可靠，绝不能出故障。这是一个关键维度。但单凭这两点还不足以构成完整的商业闭环，大家会问：“好吧，那你们的商业模式到底是什么？”我们之所以希望这套底层基础设施达到极致的高效与坚固，是因为我们清楚地知道，在这套基座之上，我们还可以构建出其他更具巨大商业价值的高层产品。

主持人：这非常合情合理。我很认同你关于“软件淬炼”的论点，因为这反过来能让你们自己在更高可靠性的基石上复用它。它可以成为一个让你高枕无忧的底层依赖项，至少你不用整天提心吊胆担心它会崩掉，这太棒了，因为这样就能以此为跳板去解锁更高层级的产品。这确实很有意思。

Markus：这就引出了另一个极其重要的行业痛点。长期以来，我们看到各类新型 AMM 对极其繁复的集成对接流程感到无比沮丧。在以往，这是一个极度依赖商务拓展（BD-heavy，转录文本误作 E-having）的沉重流程。你必须到处托人找对方团队。如果你是一个刚刚创立的新型 AMM，在圈子里没有任何人脉背景，你该怎么去联系市面上所有的 DEX 聚合器并说服他们接入你的流动性？聚合器往往会傲慢地回应：“你的 AMM 上根本没有交易量，我凭什么在乎你？排队等着我集成的 AMM 还有十家，而且人家在某些情况下甚至还要付我高昂的集成通道费呢。”这让人感到无比挫败。这种冷启动（bootstrapping）难题，成为了阻碍 AMM 机制创新的核心绊脚石。

### [00:32:48 - 00:36:51]
**EN:** To get flow in the first place, you need to be in the aggregators. But how are you going to get it to the aggregators? You don't have flow. And so what was important about Tyco is that we did a lot of work to document and make the onboarding process as self-sufficient as possible so that anybody could come, read the docs, look at the examples, look at all the previous integrations, and do the entire integration themselves. Put a PR, and if the PR is well done, we'd review it and then they would go live. And that's simply something that's not possible. And we wanted it to be so that if you integrated to Tyco, you will get flow from everybody. Not just from one aggregator, but you would get flow from KAUSOP, UNIX, one intrusion, and some aggregators. Because enough of the market is using Tyco so that you're exposed to everywhere in one go. And that was an important motivation. And then, of course, sheer efficiency. It was a pain to watch everyone reinvent and do the same thing. And it's not something that teams enjoyed doing. And it's not something that added any value to the industry. It was simply just grunt work that everyone had to do again and again. That's also very frustrating to see. It also speaks to the ethos. For instance, a lot of people will say they care about decentralization open source. But when you actually build something that nudges people in that direction, you're contributing to the ecosystem with those values. So I think that's awesome that you guys did that. I also think what's interesting here is, okay, so Tyco allows you to index liquidity, but you still need to build your own route to decide how you're going to do this execution on behalf of the trader or the user or yourself, if you're the one doing the trading. Then you open source find. And if I understand correctly, find does this. Find defines routes for users to trade against. And this could be integrated by an application that wants to offer additional swaps, or it could be integrated by somebody that's doing some kind of on-chain solving, so on and so forth. Maybe if you can unpack find for us and how it fits together with Tyco. Yeah, so find wraps around Tyco, it uses Tyco and adds all of the pieces that were missing to turn Tyco into a production grade DEX aggregator. And so find most prominently implements algorithms, solving algorithms, it also implements a market graph. So it turns the individual pools that Tyco models correctly into a graph of interconnected pools. And then once you have that graph, you can run algorithms over it, something like, for example, Bell and Ford, or different graph algorithms, and find the optimal route. And then there's other things that you need to do. Of course, if you want to run a production solver, you should probably have vertical scaling, which find has auto scales to your infrastructure. You can have one worker and solve one trade at a time. You can have 10 workers and solve 10 trades at a time. You can even run four different types of algorithm in parallel and choose the one that gives you the best route for that trade. And that might actually be a different algorithm depending on the trade. It also gives you logs and metrics and benchmarking to monitor your algorithm. So it basically puts everything into a box that you would need to run a reliable local text aggregator. So everything that's happening behind in a black box with an aggregator API, you have locally transparently happening on your server. Yeah. And that gives you several significant advantages. What has the feedback been like so far as you rolled the product out? I'm assuming that you have a particular audience that you're trying to get some adoption from some initial potential customers there. Maybe if you could speak on those things.

**CN:** Markus：这就陷入了一个经典的先有鸡还是先有蛋的死循环：想要获得订单流（Order Flow），你首先得被主流聚合器接入；但如果你本身缺乏初始订单流，聚合器又根本不屑于理睬你。因此 Tycho 的核心革命性在于，我们投入了巨大的精力去完善开发者文档，让协议接入流程尽可能实现完全的“自给自足”。任何创新团队都可以直接查阅文档、参考代码样例、研读以往的所有集成案例，完全凭借自身能力独立完成整套协议的适配开发。随后他们只需提交一个 Pull Request（PR），只要代码规范达标，我们通过审核后便能直接合入上线。而在以往的生态里，这种无许可的自主接入是根本无法想象的。我们设定的愿景是：一旦你的协议完成了对 Tycho 的适配集成，你就能瞬间捕获来自全网的订单流。不仅限于某单一聚合平台，而是能够同时捕获来自 CoW Swap（转录文本误作 KAUSOP）、UniswapX（转录文本误作 UNIX）、1inch（转录文本误作 one intrusion）以及市场上其他各大主流聚合协议的真实交易流。因为市场上已经有足够多的核心参与者在深度运行 Tycho，只要完成一次集成，你的流动性就会在瞬间全方位暴露给整个行业。这是一个至关重要的出发点。当然，另一个纯粹的考量是效率：眼睁睁看着整个行业的团队不断重复造轮子、做一模一样的重复劳动，实在令人痛心。没有哪支优秀的工程团队会乐于做这种无休止的机械劳动，而且它对整个行业完全没有产生任何增量价值，纯粹是每个人都不得不周而复始去啃的脏活累活。这种资源内耗是非常令人沮丧的。

主持人：这也深度呼应了以太坊的核心精神。很多人嘴上常说自己在乎去中心化与开源，但只有当你真正构建出能够切实推动行业迈向这个方向的工具时，你才是在以实际行动回馈生态并践行这些价值观。你们能做到这一点真的非常了不起。而且我认为这里还有一个非常有意思的技术递进：Tycho 帮你完成了链上全量流动性的实时索引与状态建模，但你依然需要构建属于自己的路由算法（Router），以便替交易者、终端用户或者你自己（如果你是自营做市/套利者）计算出最佳执行路径。随后你们开源了 Find（路由寻路引擎）。如果我理解得没错的话，Find 正是用来攻克这一步的——它为用户的具体交易需求搜索并规划最优兑换路径。任何希望为用户提供 Swap 兑换功能的前端应用都可以集成它，链上求解器等专业做市套利团队同样可以无缝接入。能否请你为我们详细拆解一下 Find，以及它是如何与 Tycho 形成合力的？

Markus：没错，Find 是基于 Tycho 进行的上层封装。它以 Tycho 为底层引擎，补齐了将底层状态数据升级为一个“生产级 DEX 聚合器”所必需的所有核心算法与工程组件。在 Find 中，最核心的模块是各类寻路求解算法，同时它构建了全局市场图谱（Market Graph）。它把 Tycho 精准建模的一个个孤立流动性池，拼接组装成一个拓扑相连、动态互联的庞大图网络。一旦在内存中建立了这个市场图谱，你就可以在其上运行高效的图算法——例如贝尔曼-福特（Bellman-Ford，转录文本误作 Bell and Ford）算法或其各类变体与启发式搜索策略——从而毫秒级计算出全局最优的兑换路由。此外，如果你想运行一个真正能够抗住生产流量的求解器（Solver），系统还需要具备诸多硬核的工程特性：例如纵向扩缩容能力，Find 能够根据你的底层硬件配置实现自动化弹性扩展。你可以只启动一个 Worker，单线程处理一笔交易；也可以部署 10 个 Worker，高并发同时求解 10 笔交易。你甚至可以针对同一笔交易，在后台并行启动 4 种截然不同的寻路算法进行赛马，并动态选取能够提供最优结算路径的算法结果——针对不同规模和币种特征的交易，胜出的算法往往大相径庭。此外，它还内置了完善的结构化日志、性能指标看板以及端到端基准测试工具，便于你实时监控算法表现。简而言之，它把你运行一个高可用、高可靠的“本地 DEX 聚合器（转录文本误作 local text aggregator）”所需的所有工具链全部打包成箱。以往在中心化聚合器 API 背后黑盒运行的一切，现在都以 100% 透明、可审计的方式在你的本地服务器上实时运转。这带来了多项颠覆性的绝对优势。

主持人：到目前为止，随着产品的推出，外界的反馈如何？我猜你们应该有非常明确的目标受众画像，并正在努力从一些潜在的早期种子客户那里推动采用。能聊聊这方面的具体进展吗？

### [00:36:51 - 00:37:37]
**EN:** Yeah. So we've gotten more interest than we anticipated and also from some teams who we didn't expect to be. And I think you can come up with those yourself if you look at the advantages. So the advantage one is, first and foremost, why did we build this? We didn't come up with this completely by ourselves. We thought that that problem was kind of solved with aggregator APIs. It wasn't very pretty that you had to YOLO trust a third party API on the most essential thing, which is the call data for your transaction. But what really pushed us over the edge is the sheer frustration of the consistent frustration that developers express towards us.

**CN:** Markus：是的，我们收到的关注和集成意向远超最初的预期，甚至包括一些我们原本完全没想到的团队类型。如果你仔细拆解它的技术优势，其实很容易理解为什么会有如此强烈的需求。首先，第一大核心考量在于——我们当初为什么下定决心做这个系统？其实这并不是我们凭空闭门造车想出来的。在很长一段时间里，大家一度以为聚合器 API 已经把这个问题彻底解决了。尽管在最生死攸关的事情上——也就是为你即将签名的链上交易生成核心 Call Data——你不得不去盲目（YOLO）信任一个第三方的黑盒 API，这在系统设计和安全性上确实非常糟糕。但真正把我们推向临界点、促使我们动手去颠覆它的，是广大开发者持续不断向我们倾诉的巨大沮丧与挫败感。

### [00:37:38 - 00:39:59]
**EN:** Being rate limited at the wrong times, being over quoted all the time, trades reverting, that looking really bad on business. And it was very consistent. And it occurred to me that this is not because teams are malicious or incompetent. Definitely not. It's because the incentives are completely misaligned. There's a principle agent problem at work where the aggregator doesn't have skin in the game for the trade to go well. And the developer has no control. And that having no control and not having the same outcome or the same target then expresses itself in all this frustration. And it occurred to us that that's the same issue, the same problem that Ethereum is trying to solve. That it is not necessarily that the banks are evil. It is the incentives are laid out wrong. If you are blindly trusted and you are a black box, then you're incentives are to do whatever you can within the law or within the observable law to make a good profit. And that then eventually expresses itself as frustration on things aside. Just enough frustration that you don't switch away from the bank, but the maximal possible amount of frustration that your users will sustain. And that's just the predictable outcome of that system if you have no verifiability. And that made it very clear that we need to do this a different way and developers want it a different way. And then it happened along the way that we saw that there are advantages that are striking. If you run the router locally, and this only occurred to us when we were well into building the algorithms, you can be 100 times faster, at least 20 times faster. So you can solve a trade already with the simplest algorithm, not even optimized, 20 milliseconds on your MacBook. If you go through an API, it's 500 milliseconds. The fastest, fastest that's available right now, as far as I know, is something like 80 milliseconds, many out of two seconds. You have to go through the API, you increase your latency time, you also have to trust that the API is

**CN:** Markus：在最关键的行情时刻莫名其妙遭遇 API 限流（Rate Limit）、报价经常被注水虚报（over quoted）、链上交易频繁因状态过期而回滚（revert），这些灾难性体验对开发者的商业信誉造成了极其恶劣的打击。而且这种现象绝非个案，而是行业普遍的顽疾。我逐渐意识到，这并不是因为那些聚合器团队心存恶意或者技术无能，绝对不是这样；根本原因在于经济激励机制的彻底错配（incentive misalignment）。这里存在着经典的经济学委托-代理问题（principal-agent problem）：聚合服务商在最终交易是否能完美结算上，根本没有真正的切身利益绑定（skin in the game），而调用 API 的应用开发者却对此毫无底层控制权。这种“应用端毫无掌控力、双方利益目标严重割裂”的状态，最终必然以开发者一侧的极度沮丧与被动爆发出来。我们猛然意识到，这与以太坊诞生之初试图解决的深层痛点如出一辙：传统银行并不是天然邪恶，而是激励机制设计出了问题。如果你被外界盲目信任，且自身被封装在一个不可见的黑盒中，那么你的理性选择就是在法律许可（或在可被监管感知的边界内）尽最大可能攫取自身利益。这种博弈的结果，最终必然转化为用户的糟糕体验——只要这种痛苦程度恰好卡在“不足以让你愤怒地换一家银行”的临界点上，他们就会尽可能榨取用户所能承受的最大阻力。在一个缺乏可验证性的黑盒系统中，这是必然出现的制度性宿命。这让我们彻底确信，必须用一种完全不同的范式去重建这个体系，而这也正是开发者们梦寐以求的方向。与此同时，在我们深入研发寻路算法的过程中，我们还发现了极其震撼的性能代差：如果你在本地直接运行路由器（Router），你可以比远程 API 快 100 倍，最起码也能快 20 倍！即使用最基础的算法、未经深度微调，在你的 MacBook 本地也只需要 20 毫秒就能完美解出一笔复杂交易的路径。而如果你去调远程 API，往返网络延迟加上服务端排队通常就要 500 毫秒。据我所知，市面上最快、最极致的第三方 API 也需要大约 80 毫秒，很多甚至长达 2 秒以上。一旦必须经过外部 API，你不仅平添了延迟与 Gas 开销，还不得不盲目信任对方的黑盒 API 绝不会出错……


---

## Part 3 (00:40:00 - 01:00:00)

# 《Deeply Intents》访谈 Markus Schmitt：解构专有做市 AMM（PropAMM）与去信任交易架构（Part 3）

## 章节概要 (Overview)
在本部分（Part 3: 00:40:00 - 01:00:00）中，Propeller Heads 创始人 Markus Schmitt 深入剖析了本地路由求解器（Fynd）与专有做市执行引擎（Turbine）的核心设计哲学与底层微观结构：
1. **开源本地路由引擎 Fynd 的不可替代性**：分析了托管式聚合器 API 存在的虚报价格（Over-quoting）、API 限流与单点失效风险，阐释了开源本地运行如何从博弈论和激励机制上杜绝作恶；同时探讨了前端定制路由、加收手续费（如 Uniswap 模式）、以及将闪电贷与复杂路由原子化打包（如一键式金库循环杠杆 Looping Vault）带来的极端 Calldata 压缩与 Gas 优化。
2. **大额链上交易的困境与 Turbine 的破局之道**：揭示了加密对冲基金与大额交易者由于链上价格冲击（Price Impact）过高而无法满足“最佳执行（Best Execution）”要求的现实痛点；详细拆解了 Turbine 如何通过在可信执行环境（TEE，如 Intel TDX）中运行隐私批量求解器（Private Batch Solver）与内部订单簿，在单区块（12 秒）内实现零价差的“需求巧合（CoW, Coincidence of Wants）”撮合。
3. **消除委托-代理问题与 PropAMM 终极形态**：对比了传统意图拍卖中做市商因信息透明而提前撤单/扩大价差造成的隐性前跑，展示了 TEE 在仅保护订单流隐私而非托管资产时的完美安全边界；最后揭晓了基于 Uniswap v4 Hook 的 Turbine 专有资金池（PropAMM）形态——链上流动性透明存取，但由 TEE 内可验证的开源算法独家掌控动态定价权，从而构建出彻底去信任化的链上暗池撮合范式。

---

### [00:40:00 - 00:40:34]
**EN:** I'm just going to return the correct information to you and you also have to worry about potentially getting rate limited or basically losing access at a critical moment and then a trade reverting maybe for an end user. So you can solve these problems simply by running this API locally with Fynd. Yeah. If you run it locally, you have no trust issues because, well, we're not going to cheat the code in an open source software, that will be pretty obvious.  
**CN:** **Markus：** （如果是托管式 API，你只能寄希望于对方）会向你返回正确的信息，而且你还得时刻担心可能被 API 速率限制（Rate Limited），或者在关键行情时刻基本上失去访问权限，进而导致终端用户的交易发生回滚（Revert）。但如果你使用 Fynd 在本地运行这个 API，就能直接解决所有这些痛点。没错，在本地部署运行时，你根本不会面临任何信任危机，因为在开源软件的代码里做手脚是极其显而易见的，我们绝不可能作恶。

### [00:40:34 - 00:41:02]
**EN:** If an open source router is over-quoting, we would hear about it two hours later on Twitter. This sounds ridiculous, but this is exactly how incentives play out, right? We have no ability to over-quote, so we have no incentive to do so. So we will not ever over-quote because it is open source and so simply by having forced transparency through the open source code, we already have that problem out of the door.  
**CN:** **Markus：** 如果一个开源路由引擎胆敢虚报高价（Over-quoting，故意报出无法成交的虚高最优价以吸引流量），两个小时内推特（X）上就会骂声一片。这听起来可能有点夸张，但这正是博弈激励机制的运作规律。在开源模式下，我们根本没有能力作弊虚报价格，因此也就毫无动机去这么做。因为代码完全公开，开源带来的强制透明度在第一时间就将这一信任隐患彻底排除了。

### [00:41:02 - 00:41:31]
**EN:** And it doesn't matter what the moral quality of my character is, it is simply not rational to do anything like that in that open source router. And the second thing is control, right? One is reliability is very important and trust and the second is control. Is if anything goes wrong or if you want to do anything custom and surprisingly most people want to do something custom, they don't want to route through that pool. They don't want to route through that token.  
**CN:** **Markus：** 这与我的个人道德品质毫无关系，单纯只是因为在开源路由器里搞这种猫腻在经济理性上是完全说不通的。至于第二点，则是控制权（Control）。第一点是至关重要的可靠性与信任，第二点就是掌控力。一旦出现意外状况，或者你想进行任何自定义配置——出人意料的是，绝大多数团队都有定制化需求，例如他们不想路由经过某个资金池，或者不想穿过某种特定的代币。

### [00:41:31 - 00:41:53]
**EN:** They want to route through a particular token or a particular pool, but that's currently not supported by someone else because they don't consider that token safe. But you consider that token safe because it's your token. You want to trade through it. You want to offer a UI through it. Or you want to do something more elaborate like what Uniswap is doing now. They're integrating other AMMs but charging a fee on it.  
**CN:** **Markus：** 他们希望通过某个特定的代币或特定的池子进行路由，但第三方聚合器并不支持，因为对方判定该代币不安全。然而你很清楚它是安全的，因为那是你协议自己的原生代币，你希望通过它交易，并在自己的 UI 前端中提供支持；或者你想构建更复杂的商业模式，比如 Uniswap 目前正在做的策略：他们将其他 AMM 接入自己的前端，但在路由通过这些外部流动性时加收一笔界面费用。

### [00:41:53 - 00:42:29]
**EN:** So you can trade on Uniswap frontend, I think it's already live on tempo. You can trade on Uniswap frontend through non-Uniswap pools but then they're charging a fee on it. Now, that's an interesting concept. Other frontends might want to do that as well, right? If I hold an interesting protocol and I don't want my users to go to an aggregator API, so I want to actually also offer them very good quotes, I also want to integrate all of the other AMM pools but I maybe want to charge a small fee on it because ultimately the flow is coming from my frontend and from my community.  
**CN:** **Markus：** 所以你现在可以在 Uniswap 前端交易非 Uniswap 的资金池，但我记得他们对此会额外收取一笔费用。这是一个非常有趣的机制，其他 DApp 前端可能也想这么做对吧？如果我掌管着一个很有吸引力的协议，我不想让自己的用户流失到第三方聚合器 API，所以我既想为他们提供极其优异的报价，又想整合全链所有的 AMM 流动性池，同时我还可能想抽取少量手续费，因为归根结底，这些订单流（Order Flow）来自于我自己的前端和社区。

### [00:42:29 - 00:42:49]
**EN:** And my community is willing to pay that fee because they know that that fee is going into maintaining that community and that frontend and that team. So you can have the best of both worlds. Now as far as I know, no aggregator offers you that, but you can do that easily locally. If it's your router, you can encode these kind of things. You can also take fees if you want.  
**CN:** **Markus：** 我的社区用户也完全愿意支付这笔费用，因为他们知道这些资金将用于维护该社区、前端以及背后的核心团队，这样就能两全其美。据我所知，市面上没有哪家中心化聚合器 API 能为你提供这种自由度，但如果你在本地运行路由器（Fynd），就能轻而易举地实现。因为这是你自己的专属路由引擎，你可以把这类逻辑硬编码进去，随心所欲地捕获前端费用。

### [00:42:49 - 00:43:19]
**EN:** So this customizability is something that the people who came to us, they value a lot. Does the customizability also extend to how you write the call data? Can you, let's say, do something more efficient that might be cheaper because you know how to do that versus if you were just calling the API, maybe you have to pack the call data in a particular way? So if it comes to standard routes, then we do the best possible, right? In safety margins, right?  
**CN:** **Markus：** 因此，找到我们的合作团队非常看重这种高度的可定制性。  
**主持人：** 这种定制能力是否也延伸到了调用数据（Calldata）的构建层面？比如说，由于你们掌握深厚的底层优化技术，能否生成更加高效、Gas 更廉价的交易负载，而不是像调用常规 API 那样只能按照固定格式封装 Calldata？  
**Markus：** 如果是常规标准路由，我们在安全边际允许的范围内，已经做到了极致的最优解。

### [00:43:19 - 00:43:46]
**EN:** Between what is safe. And we use everything that we know and that people tell us, and that again goes back to hardening the open source stack to improve on these aspects. So encoding the traits so that they are as cheap as possible. We included a lot of tricks that they're very important to make the gas cheaper. Now gas is not so important anymore, but it does implement all of these tricks.  
**CN:** **Markus：** 在确保安全的前提下，我们倾尽所学并吸收社区的反馈，而这又进一步反哺并加固了开源技术栈在这些维度上的表现。在交易编码（Encoding the trades）方面，我们力求将其压缩到极致的廉价，融入了大量对于压降 Gas 必不可少的高级技巧。虽然如今在 L2 上 Gas 成本已经不再像以前那样敏感，但代码中确实完整实现了所有这些极致优化。

### [00:43:46 - 00:44:18]
**EN:** So when it comes to normal routing, I think that is within very close to what the most competitive teams are capable of doing. And the only things that we maybe didn't do are only because it makes the router significantly safer. Yes, but you can absolutely do things, for example, like wrap a flash loan and do arbitrage on it. Right? I wire coded what actually this morning that takes a flash loan and turns fine from a solver into an atomic arbitrage bot.  
**CN:** **Markus：** 因此在标准流动性路由层面，我认为我们的表现与业内顶尖团队的技术能力已十分接近；而我们极少数没有采纳的微优化，纯粹是为了确保路由器的绝对安全。此外，你完全可以基于它实现更复杂的进阶功能，比如封装闪电贷（Flash Loan）并执行原子套利。实际上，我今天早上刚刚随手写了一个脚本原型，接入了一笔闪电贷，瞬间把 Fynd 从一个单纯的 DEX 求解器变成了一台原子套利机器人。

### [00:44:18 - 00:44:38]
**EN:** Now I made it sound easy, but I've planned arbitrage bots for a long time and I've interviewed some arbitrators. So a lot of signal went into this, like into a project plan. But yes, it's absolutely possible to do that. I'm not openly recommending to wire code this, you know, do your own research on that. But this is absolutely possible or you batch calls, right?  
**CN:** **Markus：** 虽然我嘴上说得很轻巧，但我对套利机器人的架构已经构思了很久，也访谈过多位顶尖套利者（Arbitrageurs），因此在工程规划中沉淀了大量有价值的交易信号与经验。但这确实完全行得通。当然我并不是公开推荐大家随便写写就上线套利，大家要做好自己的调研（DYOR）。但这展现了无限的可能性，你还可以进行批量调用（Batch Calls/Multicall）。

### [00:44:38 - 00:45:06]
**EN:** You can take a flash loan, for example, a case that I hear a lot is you want to loop into a vault position. And if you do it really with looping, you need to work, you need to do like five transactions in a row and the user sits there and clicks a proof of sign and 2021 crazy. But if you were actually able to combine a flash loan with the route and you custom built this, then it's one transaction.  
**CN:** **Markus：** 举个例子，你可以结合闪电贷来处理我经常听到的一个高频需求：用户想要通过循环存贷加杠杆（Looping）进入某个收益金库（Vault）头寸。如果纯粹用传统方式手动循环，你得连续执行大约五笔链上交易，用户只能坐在那反复点击授权（Approve）与签名，像 2021 年 DeFi 狂潮时期那样繁琐得不可理喻。但如果你能将闪电贷与兑换路由定制化结合，就能在一笔交易内搞定。

### [00:45:06 - 00:45:32]
**EN:** You loan the money for the full position, you swap that you deposit. Now that's also possible if you control the router, because you put that into one call data and you let the user sign once and that's a much better UX as well. And lower fees, as another point I didn't see what before teams told us, also you pay much less fees because when you look, you're going to pay the AMN fee on every leg of that loop.  
**CN:** **Markus：** 你先通过闪电贷借出建立完整杠杆头寸所需的资金，通过路由兑换后直接存入金库。而只有当你完全掌控路由引擎时，这一切才成为可能，因为你可以将所有操作打包成单份 Calldata，用户仅需签名一次，用户体验（UX）会得到质的飞跃。此外还有大幅降低的规费成本——在其他团队告诉我之前我都没意识到这一点：当你手动循环时，在每个循环环节（Leg）都必须重复支付一次 AMM 的兑换手续费。

### [00:45:32 - 00:46:07]
**EN:** So you pay it five times, which is ridiculous, especially if you do stable funds. Yes. It almost defeats the purpose for a lot of users below a certain size threshold. No, this is super interesting because it sounds like with Find there's an opportunity to cater towards more sophisticated application teams that need this integration for the reasons that you just mentioned competitively. But then also for people that want to experiment that might be doing some on-chain searching or some solving, there's some cool products that you can build yourself and maybe even build like a business around potentially with these tools.  
**CN:** **Markus：** 结果你足足支付了五次手续费，这太荒谬了，尤其是在做稳定币池互换时更是如此。  
**主持人：** 确实，对于资金规模低于特定门槛的广大用户来说，这种损耗几乎让套利或加杠杆失去了意义。这非常有启发性，因为听起来 Fynd 不仅能迎合那些为了维持竞争优势而需要深度集成路由能力的高阶 DApp 团队，同时对于热衷于链上 MEV 搜索（Searching）或求解器（Solving）的开发者，也能基于这套工具搭建出非常硬核的产品，甚至可能围绕它们建立起一门独立的业务。

### [00:46:07 - 00:46:49]
**EN:** So it sounds like between Tyco and Find, you have everything that you need to build a local aggregator as you mentioned. How do these pieces fit with Turbine, which has been described in different ways over time, but I think the most recent one was that it's private batch solver on TEs that maximizes coincidence of wants. The definition might have changed or the description might have changed since I pulled that, but yeah, maybe if you could talk about first what Turbine is and then after how the other two components that you've built fit together and how Turbine leverages those pieces.  
**CN:** **主持人：** 听起来 Tycho（超高速索引引擎）加上 Fynd（最优流动性路由引擎），就已经涵盖了构建本地聚合器所需的全部拼图。那么这两大组件是如何与 Turbine 结合的？外界在不同时期对 Turbine 有过不同的描述，我记得最新的一种说法是：它是一个运行在可信执行环境（TEE）中、旨在最大化“需求巧合（Coincidence of Wants, CoW）”的隐私批量求解器（Private Batch Solver）。它的官方定义可能有所迭代，你能否先阐述一下 Turbine 到底是什么，然后再讲讲它如何与另外两个组件协同，以及它是如何调度前两者的底层能力的？

### [00:46:49 - 00:47:22]
**EN:** Maybe the simplest is to start from what the design is trying to achieve. One of the big motivations was conversations with liquid funds, traders who are trading large tickets, and they told us about their process and their frustrations. I was shocked. I didn't think that I was so hard and then confirmed many other times that there is simply no way to trade large size relative to the liquidity on-chain.  
**CN:** **Markus：** 最直观的切入点或许是先聊聊这项架构设计试图达成的终极目标。最大的核心动力之一，来自于我们与众多加密流动性对冲基金（Liquid Funds）和大额交易员的深入交流。他们向我们倾诉了自己的交易流程以及遭遇的重重挫折。我当时大为震惊，从未料到大宗交易会如此艰难；而后续的大量调研也反复印证了这一事实：相对于链上现有的流动性池深，你根本无法在链上顺畅执行大体量的订单。

### [00:47:22 - 00:48:00]
**EN:** If you have to liquidate one million hours, then it's simply not the best place to trade that on-chain, because you will get a worse price. There's like, I don't know the number particularly, but maybe 5% price impact and on a centralized exchange, maybe it's 1% or 0.5 or 0.1. Even if you want to trade on-chain, you wouldn't because you'd have to, well, in the case of liquid fund, answer to your LPs on did you do your best to get the best execution on this trade?  
**CN:** **Markus：** 假设你要在链上平仓出清 100 万美元的头寸，链上根本不是执行这笔交易的最优场所，因为你拿到的成交价会极其糟糕。我手头没有具体统计数字，但链上可能会产生高达 5% 的价格冲击（Price Impact），而如果在中心化交易所（CEX），价格冲击可能只有 1%、0.5% 甚至 0.1%。哪怕你主观上极度推崇链上交易，你也无法付诸行动，因为作为流动性基金管理者，你必须向你的出资人（LPs）履约交代：你是否真的在交易中竭尽全力实现了“最佳执行（Best Execution）”？

### [00:48:00 - 00:48:30]
**EN:** And you couldn't say that if you did on-chain. And this is, I think, also one of the reasons why so many large traders are not trading on-chain. Not necessarily because it's harder or because they're more on-boarded to centralized exchanges, but it's simply, in many cases, not rational to trade on-chain if you're looking for the best price. So that was one motivation. How can we have large trades that settle on-chain at a price that is as good or better than a centralized exchange?  
**CN:** **Markus：** 如果你直接走普通链上交易，你就绝不可能向 LP 交代自己做到了最佳执行。我认为这正是绝大多数巨鲸与大宗交易机构迟迟不愿进军链上的核心痛点。这并不一定是因为链上操作更复杂，也不是因为他们对 CEX 形成了路径依赖，而是因为在绝大多数情况下，如果你追求最优价格，在链上交易在财务逻辑上就是彻底不理性的。因此我们的初心很简单：如何让大单在链上结算时，获得与中心化交易所旗鼓相当、甚至更加优越的成交价格？

### [00:48:30 - 00:49:04]
**EN:** Because if we are able to do that, then a significant portion of centralized exchange traders and also maybe classified traders will come on-chain and they will trade there. And then I believe there's a lot of additional benefits that you get from trading on-chain, like the lower fragility itself, custody, transparency, and permissionlessness. And once you close the pricing gap, those factors will then do the deciding difference and get a lot of people on-chain. So that was the motivation.  
**CN:** **Markus：** 一旦我们成功做到了这一点，相当大比例的中心化交易所交易员以及传统金融机构交易者就会自然而然地迁移到链上。同时我相信，链上交易还蕴含着巨大的附加价值：更低的系统脆弱性、纯粹的资产自托管、无可争议的透明度以及无需许可性（Permissionlessness）。只要你抹平了价格差距这一致命硬伤，上述原生优势就会发挥决定性的转折作用，吸引大批资金涌入链上。这就是我们的底层驱动力。

### [00:49:04 - 00:49:36]
**EN:** How do we design an exchange that can settle large trades at arbitrarily low spread? Maybe even better than mid-price, which is something we figured out is possible. And it's possible in turbine now. And a lot of the inspiration came from different pieces. On the one hand, we're doing a lot of research into what are the issues with AMMs? What are the limitations of AMMs? And then we did research into what are the issues with TratFi designs and which researchers have done what kind of research on how to fix those issues.  
**CN:** **Markus：** 我们该如何设计一个能以极低乃至任意低的价差结算大宗交易的交易机制？甚至能否实现比市场买卖中间价（Mid-price）更优的成交？我们经过钻研发现这完全行得通，并且已经在当前的 Turbine 中变成了现实。这种灵感凝聚了多方面的思考：一方面，我们深度研究了传统恒定乘积 AMM 的缺陷与边界；另一方面，我们剖析了传统金融（TradFi）市场架构的弊端，并借鉴了前沿学术界为修补这些弊病所提出的各种理论模型。

### [00:49:36 - 00:50:16]
**EN:** And then out of that really, puzzle pieces emerged turbine. The turbine is a batching solver that runs on a TE. And this is also a private order book where you can put an order in and that order book sits side by side with that batching solver on the TE. And your order settles against all on-chain liquidity. All everything that FIND has access to and thereby the Tycho has access to, all AMMs, all RFQs that are permissionlessly accessible.  
**CN:** **Markus：** 正是在这些理论与工程拼图的交汇中，Turbine 破土而出。Turbine 是一个运行在可信执行环境（TEE）内部的批量拍卖求解器（Batching Solver）。与此同时，它也是一个隐私订单簿（Private Order Book），你可以将限价订单输入其中，订单簿与批量求解器在同一个 TEE 飞地内并肩紧密运行。你的订单既可以与全链上流动性进行结算撮合——涵盖 Fynd 和 Tycho 所能触达的全部底盘，包括所有的 AMM 流动性池以及无需许可访问的 RFQ（询价）网络。

### [00:50:16 - 00:50:45]
**EN:** But it also settles against each other. So the trades that sit in that order book, they don't settle one at a time, but they settle in batches. And the batches for simplicity right now, they happen once per block, once per theorem block. So once every 12 seconds. And if you batch traders, then theoretically, you can have a lot of moments where one trader settles against another trader and they, well, what price do they settle?  
**CN:** **Markus：** 但更关键的是，订单簿内部的订单还可以彼此撮合结算。保存在该订单簿中的交易绝不会单笔孤立地顺序执行，而是按批次（Batches）聚合结算。为了追求简洁与确定性，目前拍卖批次与以太坊的出块节拍严格对齐，即每个以太坊区块撮合一次，每 12 秒结算一批。而一旦你将不同交易者的订单打包成批，理论上就会频繁出现双方订单头寸精准互换的时刻，那么他们最终是以什么价格结算的呢？

### [00:50:45 - 00:51:11]
**EN:** They settle at, well, the market mid price somewhere in the middle, Naifi said. And that would mean that neither of the traders pays any spreads, there's no fees paid. So they basically settle exactly a hit price at zero spread, better than anywhere else you could settle, like you couldn't settle for this price on Binance or probably in them. But traditionally, batching hasn't resulted in many coincidence of once.  
**CN:** **Markus：** 从朴素博弈论的角度来看，他们直接在市场买卖中间价（Mid-price）成交。这意味着交易双方既不需要向市场做市商支付买卖价差（Spread），也完全省去了 AMM 的手续费。他们实际上是以零价差在公允的中间价完成结算，这种成交质量超过了全球任何现有的交易场所——你在币安（Binance）或 Coinbase 上绝不可能拿到这种零磨损的成交价。然而在传统设计中，批量拍卖往往很难促成大量的“需求巧合（CoW）”。

### [00:51:11 - 00:51:49]
**EN:** Like the one batching exchange that we do have on the theorem, do there hardly any coincidence of once? And there's a sequence of reasons for that. Most importantly is that we are addicted to, or we're stuck with so far, the idea that trades have to be limit orders. And I think that we inherit that from from Trafi from the most visible exchanges in Trafi, which are central limit order books like CME or NASA, where the limit order is the most common order type, because this is the order that you would want to use if you have an opinion about price.  
**CN:** **Markus：** 就像以太坊上现存的批量拍卖交易所（如 CoW Swap），其撮合成交中真正属于需求巧合（CoW）的比例其实相当微薄。背后有一系列深层原因，最关键的一点在于，我们过于迷恋、或者说至今仍被彻底禁锢在“交易必须以限价单（Limit Orders）形式呈现”的陈旧思维中。我认为这种观念完全继承自传统金融（TradFi）最显赫的中心化限价订单簿（CLOB）市场，比如 CME 或纳斯达克（Nasdaq）。在那些市场中，限价单是最常见的订单类型，因为只有当你对价格具有鲜明观点和预期时，你才会倾向于使用限价单。

### [00:51:49 - 00:52:24]
**EN:** Now who has an opinion about price? It's the market makers or the HFT traders. This is, in my opinion, what affected the design and what makes that design work for the cohort of traders. And we took that over, maybe not consciously, into how we design exchanges in centralized exchanges and in decentralized exchanges on the theorem. But as a user, and this is, I think, one important aspect of Turbine, who doesn't have a strong opinion on price, all I care about is fair treatment.  
**CN:** **Markus：** 那么究竟是谁对微观价格有主观观点？答案是专业做市商或高频量化交易机构（HFT）。在我看来，这深刻塑造了整个市场微观结构的设计，使得该设计天生只契合这批专业交易群体的利益。我们在构建中心化交易所乃至以太坊上的去中心化交易所时，不假思索地下意识沿袭了这套范式。但对于那些对微观盘口价格并不抱有强烈投机观点的普通用户和基金机构来说——这也是 Turbine 最核心立足点——我们真正在乎的，仅仅是获得“公平的对待（Fair Treatment）”。

### [00:52:24 - 00:52:51]
**EN:** I want a fair price. I don't want to be fleeced. I don't want to be rent-extracted. I want to get a price that is reasonable at the time when I settle, and I don't want to be sandwiched, or I don't want to be front-run. And I also don't want to pay twice the fee that I should have been paying on chain. Like if there's an AMM that gives me 10 bibs, then I want to be settling for 10 bibs and not 20.  
**CN:** **Markus：** 我想要的无非是一个公允的价格。我不想被暗中宰割，不想被寻租剥削（Rent-extracted）；我只希望在结算的那一刻拿到当时市场合理公允的公允价，我不想遭受三明治夹子攻击（Sandwiched），也不想被抢先交易（Front-run）。我更不想在链上平白无故多付一倍的手续费——比如明明有个 AMM 能提供 10 个基点（bps）的费率，我就希望按 10 个基点结算，凭什么要被迫承担 20 个基点？

### [00:52:51 - 00:53:25]
**EN:** And this is what, and that's the assumption behind it, this is what many traders really want. The liquid fund that settles their trade currently in a T-WAP over the period of 10 days, 10 different days, they don't care about executing this very second. They care about executing at a good mark-out at a low price. And so this is what Turbine is designed to do. You have the solving component of Turbine, which is using Find and Tyco, and it can calculate the most efficient route for any trade that comes in.  
**CN:** **Markus：** 而这，正是我们底层假设的基石所在——这才是绝大多数真实交易者梦寐以求的机制。那些正在通过时间加权平均算法（TWAP）分摊到 10 天乃至更长时间内去慢慢建仓平仓的对冲基金，他们根本不在意交易是否必须在当下这一秒内即时确认。他们最看重的是在长周期内拿到优异的交易后执行质量评估（Mark-out）以及低成本的公允均价。Turbine 的全部设计都是为了达成这一使命：它内嵌了由 Fynd 和 Tycho 驱动的求解计算组件，能够为涌入的每一笔交易实时演算出全网最高效的路由路径。

### [00:53:25 - 00:54:02]
**EN:** But you also have this private order book where users can send large orders, and they can be matched against each other or offset, so therefore they don't have to pay a spread, naively executing against the mid-price, so to speak. And a core thesis is that you have these liquid funds who care much more about the end result, the outcome of the trade, that they get a fair price or what they perceive to be a fair price versus they're getting fleeced to the point where they're keeping their capital on centralized exchanges or doing OTC deals for trading rather than actually executing on chain.  
**CN:** **主持人：** 与此同时，你们还拥有这样一个隐私订单簿，用户可以向其中提交超大额订单，订单之间可以直接实现内部对冲撮合，从而完全规避了买卖价差，可以说在纯理论层面上直接锚定市场中间价成交。你们的核心论点在于：这类流动性基金最关心的实际上是最终的执行结果，即他们是否获得了内心认可的公允价格，而不是被链上 MEV 肆意宰割，以至于此前只能被迫将资金滞留在中心化交易所或通过场外交易（OTC）解决，不敢在链上真实结算。

### [00:54:02 - 00:54:32]
**EN:** If I have that right, I'd like to ask you, like, what gave you confidence to build this with trusted execution environment, and what type of confidentiality affordances, or if any, are available to traders? What's important is that the trusted execution environment is not relied upon to be the store of value. So in our case, the trusted execution environment is only the store of the open order intents.  
**CN:** **主持人：** 如果我的理解没错的话，我想请问你：究竟是什么让你有底气选择基于可信执行环境（TEE）来构建这套系统？它能为交易者提供何种级别的隐私与机密性保障？  
**Markus：** 最至关重要的一点在于，我们绝不把可信执行环境作为资金资产的保管场所（Store of Value）。在我们的架构设计中，TEE 仅仅承担未决意图订单簿（Open Order Intents）的瞬时存储媒介。

### [00:54:32 - 00:55:01]
**EN:** And so worst case, if the TE gets broken into and all information exposed, what you would have is the view on the open orders. And that is exactly the status quo that you would have on any other intent auction. If you submit an intent to any intent auction today, every solver sees that trade, then every solver makes a quote request to every market maker, now every market maker knows about that trade. So it's completely transparent.  
**CN:** **Markus：** 因此在最极端的恶劣情况下，即便 TEE 硬件被攻破、内部全部机密数据遭到泄露，攻击者所能获取的也仅仅是当前未决订单的盘口视图。但这充其量只是回退到了当前任何常规意图拍卖市场的现状而已。今天只要你向任何意图拍卖协议提交一笔交易意图，所有的求解器（Solver）都能瞬间看光该订单，接着每个求解器都会向全网做市商发出询价请求（RFQ），所有做市商立刻对这笔大单了如指掌——其信息是完全全裸、毫无隐私可言的。

### [00:55:01 - 00:55:30]
**EN:** This transparency hurts, specifically large orders, because the market makers have a rational incentive to withdraw, increase their spread if they see a large order coming in. And this, in effect, means you pay more fees, and you have effectually been front run, even if nobody makes any kind of trade, which they also could, in some cases, might be incentivized to do so. Just by withdrawing their liquidity, you already pay more.  
**CN:** **Markus：** 这种信息裸露对于大额订单来说具有毁灭性的杀伤力。因为做市商具有高度理性的防御性激励机制：一旦窥见有天量大单正在路上，他们会立即撤单或骤然扩大买卖价差以防遭遇逆向选择（Adverse Selection）。这在实质上导致你需要承受更高的滑点与费用成本，哪怕没有任何人真正发起恶意交易，你在事实上已经被提前抢先交易（Front-run）了。仅仅是通过做市商撤出流动性这一个动作，你的实际交易成本就已经大幅飙升。

### [00:55:30 - 00:56:02]
**EN:** And so these are the guarantees that the TE gives you, is that your trade cannot be seen by anybody, including us, because also Turbine doesn't make any requests after you submit the trade. So it doesn't request a market maker for quotes. It doesn't request a solver for a solution. It only ingests information and doesn't share any information. In the worst case, if that information becomes transparent, it is really only the book that you see, but you cannot take funds from users.  
**CN:** **Markus：** 而 TEE 能够赋予你的硬核保证正是：包括我们 Propeller Heads 自身在内，全网没有任何实体能够提前窥探到你的交易意图。因为在你提交交易之后，Turbine 绝不会对外发起任何网络请求。它绝不会向外部做市商发起询价，也绝不会向第三方求解器索要路径解答。它只纯单向地摄取链上全域状态，绝不向外暴露半点敏感信息。退一万步讲，即便机密性彻底失效，攻击者看到的充其量也只是订单簿列表，却绝对无法掠夺或转移用户的任何资产。

### [00:56:02 - 00:56:24]
**EN:** I'm assuming this would be a TDX construction, so it would be running in cloud, so you get the security assumption of the data center and the guy with the Glock protecting it. How did you come to this? Because it seems like the way we're talking about Turbine is a little bit different than previously how we talked about Turbine as its own venue, as a DEX, so to say.  
**CN:** **主持人：** 我推测这应该采用了 Intel TDX 架构，这意味着它运行在云端机密虚拟机中，享有云数据中心以及“门口持枪保安”提供的物理与硬件安全假设。你是如何敲定这种技术路线的？因为感觉我们现在所探讨的 Turbine，与早先将其定义为一个独立的交易场所（Venue）、一个专属 DEX 的语境似乎有所微妙的不同。

### [00:56:24 - 00:56:50]
**EN:** But here we're talking about it more as a solver with a private order book running in a TEE, more or less like what's the difference between these framings and how did you come up with this being the correct architecture to take to market? Turbine is a mixture of many things and thereby it's difficult to put into a single box right now. When you said it's not an AMM or it's not a DEX, I shook my head because Turbine for example has its own pools as well.  
**CN:** **主持人：** 我们现在听起来更多把它视作一个在 TEE 中运行带隐私订单簿的求解器。这两种架构叙事之间有何本质区别？你又是如何确信这就是推向市场的最佳终局形态？  
**Markus：** Turbine 本身融合了多种微观结构构件，因此目前很难把它生搬硬套进某个单一的固有标签中。刚才你说它不是 AMM 或不是 DEX 时我之所以摇头，是因为 Turbine 实际上拥有完全属于自己的专属资金池。

### [00:56:50 - 00:57:24]
**EN:** They're turbine pools and these are Uniswap before hook pools on chain that only Turbine has access to. Now they are sort of proper AMM pools because they're proprietary to Turbine and they are AMM pools that are a Uniswap before pool. So you can deposit and withdraw from it and liquidity is transparently on chain, the curve is transparently on chain, but the pricing of that pool is exclusively set by Turbine, the proprietary price setter.  
**CN:** **Markus：** 这些就是 Turbine 资金池——它们是部署在链上的 Uniswap v4 Hook 资金池，唯独 Turbine 拥有对其路由兑换的排他性访问权。它们在本质上是真正意义上的专有做市 AMM（PropAMM）资金池，因为它们专属于 Turbine，同时又是标准的 Uniswap v4 资金池架构。任何人都可以透明地向其充提流动性，做市曲线也完全透明地锚定在链上；但该资金池的边际定价权，却百分之百由 Turbine 作为专属定价引擎（Proprietary Price Setter）独家裁定。

### [00:57:24 - 00:57:57]
**EN:** So you now have a proprietary pool run not by an off-chain team that can do anything but by an open source software that runs on a TEE. So where does that sit exactly? It's not proprietary off-chain black box, it's also not fully on-chain smart contract, it's somewhere in between I think. The logic is transparent and open source and you can verify that it runs, that this exact docker container with this exact deployment runs on this exact TEE.  
**CN:** **Markus：** 如此一来，你就拥有了一个专有资金池，但它的日常运作绝非掌控在一个为所欲为、不受约束的链下黑盒团队手中，而是由运行在 TEE 内的开源程序所严格调度。那么它究竟处于什么样的技术生态位？它既不是不透明的链下私有黑盒，也不是完全由简单链上智能合约固化的传统池子，我认为它恰好伫立于二者之间的黄金平衡点：其算法逻辑完全透明开源，而且你能够通过远程密码学证明进行核验——确保正是这个完全开源的 Docker 镜像和部署代码，分毫不差地运行在这台特定的硬件 TEE 飞地之中。

### [00:57:57 - 00:58:32]
**EN:** You can verify that in the Turbine frontend and that is essential to an exchange that skips these incentives that I've said earlier where you have this principle agent problem with a trusted agent. We don't want Turbine to be trusted, we want Turbine to be completely trustless. Also on the batching side, Turbine doesn't make assumptions about the market from its own order book. It reads the price from other exchanges, so Turbine doesn't have its own price discovery, in that sense it is not like other exchanges, in that sense it is much more like a tri-site dark pool.  
**CN:** **Markus：** 你直接可以在 Turbine 的前端完成这套密码学验证。这对于一个交易所而言至关重要，因为它彻底瓦解了我前面提到的、由于引入中心化代理人而导致的“委托-代理问题（Principal-Agent Problem）”。我们不希望 Turbine 建立在“受信任的人格担保”之上，我们要让 Turbine 成为一个纯粹的去信任化（Trustless）交易基础设施。此外在批量撮合层面，Turbine 绝不对自身订单簿做孤立的市场价格假设，它会实时读取全网各大交易所的主流价格基准。因此 Turbine 自身并不承担独立的价格发现功能，从这个意义上讲它迥异于传统交易所，而是更接近传统金融中的内部撮合暗池（Crossing Dark Pool）。

### [00:58:32 - 00:58:55]
**EN:** But yeah, I'm gonna stop here, there's a bit more time. But to your question, how do you become of this design, how did this design happen? It really just happened by us listing all of the issues that we had with exchanges on trainablewood and all of the different pieces that we knew about and then you just iterate until none of the problems remain or you make trade-offs that you're happy with.  
**CN:** **Markus：** 好的我先停一下，时间还剩不少。但回答你刚才的问题：这种架构设计究竟是如何推导出来的？这其实完全源于我们详尽列出了当下以太坊等链上交易平台暴露的所有痛点，结合我们所掌握的所有密码学与分布式组件，然后不断推演迭代，直到所有棘手问题都被逐一化解，或者找到了令我们由衷满意的权衡取舍（Trade-offs）。

### [00:58:55 - 00:59:21]
**EN:** For example, that you can't do fast trading on Turbine, it's not an HFT. You can't have fast trading and use trade protection, to my understanding you cannot have that. Turbine is very opinionated about that, it cares about price and not about speed. But it also, I mean it seems like with the customer that it's targeting specifically, like caring about price and not speed is the thing to care about. So I mean, it seems like you've built it specifically in that way.  
**CN:** **Markus：** 举个例子，在 Turbine 上你无法进行超高速即时交易，它绝不是高频交易（HFT）场所。因为在我的认知体系中，你绝不可能在追求亚秒级撮合速度的同时，还奢望享受到全方位的交易反 MEV 保护。Turbine 在这一立场上拥有极其坚定的价值主张（Opinionated）：它视最优价格为生命，而主动放弃对极速的执念。  
**主持人：** 但这也完全符合逻辑，对于它所瞄准的核心客群（大宗机构和对冲基金）而言，成交价格和执行质量才是他们命悬一线的关切点，速度根本无关紧要。所以你们这套架构完全是为他们量身定制的。

### [00:59:21 - 00:59:50]
**EN:** Do you ever plan on open sourcing everything with the Turbine logic and then maybe building something else? Because it seems like you have a track record now of building something to solve a problem and then open sourcing it and moving on to the next part of the stack. Or is this something that you think you're gonna spend a lot of time with? Oh, I mean, we are spending a lot of time with Taiko every day, there's a full team on it, it's not like we open source it that it takes care of itself for sure or not.  
**CN:** **主持人：** 你们是否计划最终将 Turbine 的全部核心逻辑彻底开源，然后再转头去攻关其他新赛道？因为纵观你们的发展历程，似乎一直在践行“为解决特定痛点研发产品 -> 全面开源技术栈 -> 转向更高阶基础设施”的轨迹。还是说你们打算在 Turbine 上长期深耕？  
**Markus：** 噢，其实我们每天依然在 Tycho（索引引擎）上倾注海量心血，由一个完整的核心团队全职负责。绝不是说一旦开源了，它就能自然而然地自我迭代，绝非如此。

### [00:59:50 - 01:00:00]
**EN:** No, these are all products that we expect to take care of for a long time. And we expect for them to be still polished for quite a while. So there's a full team on Taiko.  
**CN:** **Markus：** 事实上，这些都是我们抱定长期主义、打算深度维护打磨的核心支柱产品。在未来很长一段时间内，我们都将持续对其进行工程雕琢与性能优化，因此目前依然有一支完整的团队全情投入在 Tycho 的研发中。


---

## Part 4 (01:00:00 - 01:06:09)

# Markus Schmitt (Propeller Heads) - Tilting at PropAMMs (Part 4)

### 章节概要 (Overview)
在访谈的终章（Part 4），Propeller Heads 创始人 Markus Schmitt 与主持人深入探讨了团队旗下三大核心产品——**Tycho**（高性能全链流动性状态索引与结算引擎）、**Fynd**（链上聚合寻路与价格发现系统）与 **Turbine**（抗 MEV 的去中心化批量拍卖交易所）——之间的协同关系与架构蓝图，剖析了以太坊开源生态在基础设施领域的商业可行性，并给出了关于未来 5 至 10 年全球资产交易终局的重磅预测：
1. **三位一体的协同技术栈**：Tycho 负责毫秒级跟踪全链所有 AMM/流动性池状态并支持模拟结算；Fynd 依托 Tycho 进行极速链上路径计算与价格发现；Turbine 则利用 Fynd 撮合链上流动性并通过 Tycho 完成原子结算。Turbine 坚持彻底开源，确保拍卖机制与撮合逻辑的完全可验证性。
2. **开源哲学与正和商业模式**：Markus 坚决驳斥“开源不利于商业变现”的悲观论调，指出以太坊本身就是最具说服力的商业样板。优秀的基础设施创业不应陷入零和博弈去“抢夺技术栈（capture the stack）”，而应通过消灭低效（如消除 LVR 与潜伏套利）创造此前不存在的“增量经济剩余（Surplus）”，并在创造的增量价值中实现变现。
3. **交易供应链终局预测：批量拍卖（Batch Auctions）的必然胜利**：Markus 给出激进预测——未来 5 年内，链上绝大部分交易量将彻底从现行的“先到先得（FCFS）”连续搓合队列转向批次结算；在未来 10 年内，全球大多数金融资产（涵盖加密原生市场与受鲁棒系统赋能的传统资产）都将采用批量拍卖模式，从根本上解决高频潜伏套利与 MEV 损耗。

---

### [01:00:00 - 01:00:33]
**EN:** Full team on fines and a full team on turbine and I don't see any changes to that in the near future and we will also open source turbine. So turbine is also entirely open and it is part of the positioning of turbine that you can verify everything that's happening. This is not to the detriment of the exchange, I think similarly to AMMs it makes it more trustworthy and makes it easier to interface with and use.

**CN:** Markus：我们有专职的完整团队在负责 Fynd，也有另一支完整的团队在负责 Turbine，而且在可预见的未来我看不到这方面会有任何改变；不仅如此，我们还会将 Turbine 彻底开源。Turbine 将会完全透明开源，而且其核心定位之一就是让任何人都能对其链上链下发生的所有撮合与清算环节进行独立验证。开源丝毫不会损害交易所的竞争壁垒，我认为这正如 AMM 一样，透明开源反而让它更具公信力，也让外部系统与解算器（Solver）更容易与它对接和使用。

---

### [01:00:33 - 01:00:58]
**EN:** So I reject the framing of as soon as something is open source, somehow we don't need to take care of it and we need to move on to something else, absolutely not. I think open source and doing the right thing can be a very good, can be very good business, you know, it's not, it does not mutually exclusive, absolutely not. I mean, look at Ethereum, right, and this is in many ways our model is Ethereum.

**CN:** Markus：因此，我绝不认同那种“某样东西一旦开源了，我们就无需再悉心维护、该转向其他新项目”的论调，绝对不是这样。我认为坚持开源、做正确的事完全可以成为一门非常成功的商业生意，二者绝不是互斥的，绝非如此。看看以太坊就知道了，对吧？在许多层面上，以太坊本身就是我们的精神榜样与商业范本。

---

### [01:00:58 - 01:01:27]
**EN:** That's good, I didn't mean to paint you into a corner. I guess I was just more thinking about like, sometimes it's difficult to juggle so many balls in the air at the same time, but you know, clearly you guys have teams dedicated to each and then you also have some central leadership as well. And they wouldn't be possible without each other. So I think they're not separate things, so find is impossible without Taiko. And turbine is not possible without find.

**CN:** 主持人：这很好，我并不是想故意刁难你把你逼进死角。我当时主要是觉得，团队要同时在空中兼顾把玩这么多复杂的项目球，有时确实极具挑战性；不过很显然，你们为每个产品都配置了专注的专属团队，同时也有统一清晰的战略领导层。

Markus：而且，缺少了其中任何一个，其他产品都无法成立。我认为它们并非孤立割裂的组件——没有 Tycho，Fynd 根本不可能运行；而没有 Fynd，Turbine 同样无法运转。

---

### [01:01:27 - 01:02:08]
**EN:** Like turbine uses find to trade on chain, it uses find to discover prices on chain, and it uses Taiko to settle against all AMMs. Without Taiko, a find turbine would not be possible. So these are all, well, part of one thing. So we talked a lot about prop AMMs, and then you took us through Taiko, find, and turbine. And there's some things that were implied there. I'd be curious, what does the propeller product stack say now about what the trade supply chain is going to look like in the next couple of years?

**CN:** Markus：具体来说，Turbine 需要依靠 Fynd 来进行链上交易撮合，利用 Fynd 来完成极速链上价格发现，并借助 Tycho 与全网所有的 AMM 进行原子结算交互。没有 Tycho，Fynd 和 Turbine 根本无从谈起。所以说，它们实质上是一个有机整体的不同分工。

主持人：我们之前深入探讨了专有做市 AMM（PropAMM），接着你又带我们系统梳理了 Tycho、Fynd 和 Turbine 的全景。这背后其实蕴含了极其深厚的行业推演。我非常好奇，结合 Propeller 现在的整套技术栈与布局，你认为未来几年整个交易供应链（Trade Supply Chain）将会演变成怎样的形态？

---

### [01:02:08 - 01:02:33]
**EN:** Are there any kind of like spicy or bold predictions that you'd be willing to share? Or even if that's too much, just things that folks should pay attention to or some context with which they should process this going forward? Because it seems that a lot is changing very fast here and players like yourself have like a very good insight into where things are going. Spicy prediction.

**CN:** 主持人：你愿意分享一些颠覆性、大胆激进的行业预测吗？或者退一步讲，有哪些大家在未来演进中必须重点关注的微观结构趋势，或者梳理这一演变过程所需的核心认知框架？因为当前这个领域演进实在太快了，而像你们这样身处前线的团队，对未来的走向显然有着极其敏锐深刻的洞见。来个重磅预测？

Markus：重磅预测是吧。

---

### [01:02:33 - 01:03:08]
**EN:** Yeah, I'd say that a spicy prediction is that in DeFi, on chain, I think within five years, the majority of volume will be settled in batches, not first come, first serve. I think there's a natural equilibrium that tends in that direction. And we're doing our best with turbine to force that. But not only turbine, there are other ways to batch. And I think that that is also going to be, in 10 years, I'd say a majority of asset trading will be happening in batches worldwide.

**CN:** Markus：好，我的重磅预测是：在链上 DeFi 领域，我认为五年之内绝大部分交易量都将通过批次（Batch）进行结算，而不是采用当前的先到先得（FCFS）连续队列撮合。我认为市场本身存在一种自发趋向于批次结算的自然纳什均衡，而我们正在通过 Turbine 全力推动并加速这一平衡的到来。当然不仅限于 Turbine，市场上也还有其他实现批次化的路径。不仅如此，如果放眼未来十年，我认为全球范围内绝大多数金融资产的交易，最终都会采用批量结算（Batch Settlement）的方式进行。

---

### [01:03:08 - 01:03:41]
**EN:** And perhaps, in centralized markets, but perhaps a large part will also be happening on chain. Because we are now hardening these systems, and I think they can also serve that purpose for other assets. I think that's a spicy prediction. That's a spicy prediction to sit with. I think the question about batch auctions has been around a long time. And there's a lot of great research literature, Eric Budish and others and different papers that you have also put me onto, which I won't name here, but I appreciate it nonetheless.

**CN:** Markus：这可能发生在中心化市场中，但更大一部分极有可能会直接发生在区块链网络上。因为我们现在正在对这些去中心化系统进行极高标准的抗压强化与鲁棒性打磨（hardening），我认为这套架构未来完全足以承载并服务于其他各类主流金融资产的交易需求。

主持人：这确实是一个非常硬核、极具张力且值得反复咀嚼的激进预测！关于批量拍卖（Batch Auctions）的讨论在学术界与行业内其实由来源远，此前有许多极具价值的经典文献，比如 Eric Budish 等经济学家的先驱研究，还有很多是你之前推荐给我的论文（这里就不一一展开了），但我由衷感激你分享的这些底层学术资源。

---

### [01:03:41 - 01:04:02]
**EN:** I think this was an awesome episode. And one thing that I take away from just the overall message, and I really love how you highlighted it was you said, you know, we take a very much a similar approach to Ethereum. And the way that we think about open source and also being good for business. And I think you heard that throughout the entire episode, like it's very clear that you're not trying to capture the stack.

**CN:** 主持人：我认为今天这期播客内容实在是太硬核、太精彩了。如果要让我总结今天对话中最打动我的一点，就是你前面特别强调的那个理念——你们采取了与以太坊高度同频的发展哲学，深信开源生态与商业成功完全可以相辅相成。听完这整整一期访谈，大家能非常清晰地感受到：你们丝毫没有想过通过搞封闭花园来“侵占并垄断整个技术栈（capture the stack）”。

---

### [01:04:02 - 01:04:32]
**EN:** You're building tools that improve your own life, but also improve the lives of others. And there will be ways and paths to build monetization around certain angles. And there's many different ways if you're creative and you think about it. And I just really love hearing that because I think we're in a position right now where a lot of people are weighing with a lot of the questions about values of open source and is it worth it or not or building for decentralization, building for crops.

**CN:** 主持人：你们打造的工具不仅大幅改善了你们自己的解算与做市效率，同时也赋能并改善了整个生态中其他参与者的处境。只要保持创造力并深入思考，围绕特定业务维度完全能够探索出成熟多元的商业变现路径。听到你这番见解我真的非常欣慰，因为当前行业正处于一个迷茫期，很多人都在权衡开源的价值、质疑开源是否值得，或者在“为真正的去中心化构建”与“沦为纯粹追求垄断利润的企业机器（building for corps）”之间反复摇摆。

---

### [01:04:32 - 01:05:03]
**EN:** And I think hearing from a builder such as yourself, who's like very adamant about the direction and the way with which they're building, I think is awesome. And I think it just comes through listening to the last hours of our conversation. I think there's a strong misconception currently, specifically currently in our market. And that is that the only way to have a good business is to be more competitive is to be taking something that currently someone else has in the stack.

**CN:** 主持人：因此，能够听到像你这样对技术演进方向和构建范式抱有如此坚定信念的 Builder 发声，体验非常棒。在过去这一个多小时的深入长谈中，这种坚持贯穿始终。

Markus：我认为当前市场上——尤其是我们当前的加密货币市场中——存在一个根深蒂固的巨大认知误区：大家普遍认为，做好一门生意、提升竞争力的唯一途径，就是通过内卷竞争从现有技术栈中硬生生抢夺别人既有的蛋糕。

---

### [01:05:03 - 01:05:38]
**EN:** And I wholeheartedly disagree. I think the best way to have a good business is to add a surplus that's currently not present and then to charge on that surplus that is distinctly additive to what's currently possible. And that is the most fun and also, in my opinion, the best way to do business. Oh, yeah, Marcus, if folks are interested in using or testing out Tycho Find or Turbine, what are some ways that they can go about and interact with this technology?

**CN:** Markus：对此我发自内心地坚决反对，我绝不认同这种零和博弈思维。我认为打造一家优秀企业最健康、最强大的方式，是去创造整个系统中目前尚不存在的全新经济剩余（Surplus），然后针对这部分实打实带来的增量剩余收取费用，因为这为行业带来了原本无法企及的真正增量价值。这不仅是在技术研发中最有乐趣的事，在我看来也是最具生命力的商业经营之道。

主持人：说得太透彻了！Markus，如果听众、求解器开发者或做市团队对使用或测试 Tycho、Fynd 或 Turbine 感兴趣，他们可以通过哪些途径来体验和接入这些技术？

---

### [01:05:38 - 01:06:09]
**EN:** Is there anybody specific that they should reach out to? What's the down low? If you go to propellerheads.xyz, all the products are linked there. The docs are linked to find everything. And if you follow us @propellerswap or myself @tykane on Twitter, you will hear about every thing that we do. Sweet. It was great chatting with you today. Hopefully, we'll have something to talk about in the next six months or so. Thanks so much. This was fun. Smart, man.

**CN:** 主持人：有没有具体的联络人可以对接？能介绍下具体的参与方式吗？

Markus：大家直接访问我们的官网 propellerheads.xyz 即可，上面汇总了我们所有核心产品的跳转链接，官方开发文档也能在上面找到全部技术接入指引。此外，欢迎在 Twitter（X）上关注我们的官方账号 @propellerswap，或者关注我本人的推特 @tykane，我们会实时同步我们正在推进的所有最新技术进展与发布。

主持人：太棒了！今天和你交流非常痛快且受益匪浅，希望未来半年左右我们能再次相聚深聊。

Markus：非常感谢，今天聊得非常尽兴！

主持人：真有洞见，兄弟。
