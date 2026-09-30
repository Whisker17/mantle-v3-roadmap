# Deeply Intents: PropAMMs are eating DeFi - Quintus (Flashbots)
## 全集双语对照转录全文 (Full Bilingual Transcript)

- **播客节目**：Deeply Intents
- **嘉宾**：Quintus（Flashbots 机制设计与交易供应链核心研究员）
- **音频时长**：50 分 51 秒
- **音频文件**：
- **核心主题**：以太坊交易供应链的全面解构、订单流拍卖（OFA）与区块构建者中心化警示、为什么说 PropAMMs 正在吃掉传统 DeFi、LVR（损失与再平衡）与 CEX-DEX 套利机制、被动流动性与主动专有做市商的根本分野、Uniswap v4 Hooks 与 MEV 重新分配、意图与求解器网络的演化路线。

---

## 目录 (Table of Contents)
1. [Part 1 (00:00:00 - 00:20:00) 订单流拍卖（OFA）预言成真：交易供应链与构建者中心化](#part-1-000000---002000)
2. [Part 2 (00:20:00 - 00:40:00) 为什么说 PropAMMs 正在吞噬 DeFi：LVR 剖析与主动做市革命](#part-2-002000---004000)
3. [Part 3 (00:40:00 - 00:50:50) Uniswap v4 钩子（Hooks）、价值捕获与链上交易微观结构终局](#part-3-004000---005050)

---


## Part 1 (00:00:00 - 00:20:00)

# 《Deeply Intents》播客深度访谈：专有做市 AMM 正在吞噬 DeFi（Part 1）

**嘉宾：** Quintus（Flashbots 研究员）  
**主题：** PropAMMs are eating DeFi（专有做市 AMM 正在吞噬去中心化金融）  
**译者：** MEV 交易供应链与机制设计双语研究组  

---

### 【本期概要】
在本期访谈中，Flashbots 核心研究员 Quintus 与主持人深入剖析了以太坊交易供应链（Transaction Supply Chain）的演变历程，以及为什么专有做市 AMM（Proprietary AMM / PropAMM）正在成为 DeFi 市场微观结构（Market Microstructure）的下一代核心范式。
- **回顾 SBC 2022 的前瞻预言：** 讨论了 Quintus 早期关于订单流拍卖（OFA）与区块构建者（Builder）中心化格局的理论预测，以及当今排名前三的 Builder 占据 90% 市场份额的残酷现实。
- **理论机制到产品落地的鸿沟：** 探讨了研究员视角与市场微观现实的脱节，以及当前开发者从“将交易抽象视为通用数据包”向“聚焦真实交易点差、包含率与资金流动性”的务实转变。
- **现代交易供应链全貌与流动性演变：** 详细拆解了从用户发起交易、RPC/钱包分流、OFA 跟跑拍卖（Backrunning Auction）、Builder 组包到 PBS 提议者竞价的中继链路；对比了解释 AMM 链上流动性与链下做市商（如 Wintermute）确定性硬报价（Firm Quotes）的本质差异。
- **传统 AMM 结构性缺陷与 LVR 根源：** 深刻揭示了恒定函数做市商（CFMM）缺乏主动智能定价能力的问题——受限于以太坊 12 秒的离散出块时间，AMM 在每个区块之初不可避免地大幅落后于外部真实价格，从而沦为 CEX-DEX 统计套利的温床，导致被动 LP 面临巨大的逆向选择（Adverse Selection）与损失与再平衡（LVR）成本，最终催生了向 PropAMM 范式的根本性迁移。

---

### [00:00:00 - 00:00:04]
**EN:** Good afternoon, Quintus. Welcome to the Deeply Intents podcast. Thank you for joining us today.

**CN:** 主持人：下午好，Quintus。欢迎来到《Deeply Intents》播客，非常感谢你今天参加我们的节目。

### [00:00:04 - 00:00:29]
**EN:** GM, good to see you. I've been wanting to do this type of podcast with you for a long time. This is actually like really nice. Some contacts maybe for the listener. I think the first time I met you was at SBC in 2022. Back when, you know, some of the tornado cash sanctions had kind of just happened and people were getting very antsy, having a lot of anxiety around.

**CN:** 主持人：GM，很高兴见到你。我一直很想和你录一期这样的深度播客，今天能成行真的太棒了。先给听众们交代一些背景信息：我记得第一次遇到你是在 2022 年的斯坦福区块链大会（Stanford Blockchain Conference, SBC）上。当时针对 Tornado Cash 的制裁刚刚发生，行业里大家普遍坐立难安，产生了很多焦虑情绪。

### [00:00:30 - 00:00:45]
**EN:** The transaction supply chain. And I think around that time, there's a lot of great research that was published, things like builder commitments, proposed real separation. But one of the things that you worked on at that time was you wrote and you gave a presentation at SBC about order flow.

**CN:** 主持人：当时大家都在围绕交易供应链（Transaction Supply Chain）展开讨论。那段时间涌现出了大量非常优秀的研究成果，比如构建者承诺（builder commitments）、提议者与构建者分离（Proposer-Builder Separation, PBS）。而你当时所做的一项重要工作，就是在 SBC 上撰写并演讲了一篇关于订单流（order flow）的研究。

### [00:00:45 - 00:00:57]
**EN:** And think you titled it order flow auctions and centralizations and you called it a warning. Basically, it kind of came true, where you have the top three builders today have about 90% of the market share.

**CN:** 主持人：我记得你的演讲题目大概是《订单流拍卖与中心化：警告》（Order flow auctions and centralizations: a warning）。基本上，你的预言成真了——如今排名前三的区块构建者（builders）垄断了大约 90% 的市场份额。

### [00:00:57 - 00:01:12]
**EN:** And the builders that get the most private order flow oftentimes do win the auctions when they do receive it. So I just wanted to give you your flowers on that and just get some perspective because it's interesting. You did this research and a lot of it actually wound up playing out.

**CN:** 主持人：而且获得最多私有订单流（private order flow）的构建者，在收到这些订单时往往总能赢下区块竞拍。所以首先我想就此向你致敬，送上应有的赞誉，并听听你的看法。这非常有意思：你当年的研究，很大一部分最终都在现实中切实上演了。

### [00:01:12 - 00:01:29]
**EN:** Maybe I'm giving you too much credit because I know you're humble, but I think it was an accurate prediction. If you want to reflect on any of that, I think it'd be a good opportunity to. Thank you. I think at the time, there was definitely a bunch of people at Flashbots who were convinced that auto-solar dynamics would be very important.

**CN:** 主持人：我也许把你捧得有点高了，我知道你一向很谦逊，但我认为那确实是一次精准的预测。如果你想回顾一下当时的想法，现在正是个好机会。  
Quintus：谢谢。我想在当时，Flashbots 内部确实有一批人坚信，订单流微观动态（order flow dynamics）将会变得极其重要。

### [00:01:29 - 00:02:01]
**EN:** I think the exact way or the reason that they manifested, I think was slightly not misunderstood, but I think at the time there was multiple different factors we thought could contribute to these sort of order flow dynamics. And really what it ended up being was the way that blocks a bit to the proposer and how value is captured in that mechanism. And now when you look back, it's really clear that we can get into it.

**CN:** Quintus：至于这些动态具体是以何种方式或原因显现出来的，倒不能说我们完全理解错了，而是在当时，我们认为有多种不同因素促成了这类订单流动态。但最终起决定性作用的，其实是区块向提议者竞价出价（bid to the proposer）的机制，以及价值在该机制中是如何被捕获的。现在回过头来看，脉络已经非常清晰了，我们稍后可以深入展开探讨。

### [00:02:01 - 00:02:17]
**EN:** But at the time, we had lots of theories for why this may be the case, why people might pay for order flow and try to acquire it. And some of it I think was around network effects. If you have more order flow, then you can unlock more value because it's more coincidence of wants and these kinds of things.

**CN:** Quintus：不过在当时，关于为什么会出现这种情况、人们为什么会为订单流付费并千方百计去获取它，我们有各种各样的理论。其中一部分理论是围绕网络效应展开的：如果你掌握了更多的独占订单流（exclusive order flow），你就能解锁更大的价值，因为这样能产生更多的需求巧合（Coincidence of Wants, CoW）之类的情况。

### [00:02:17 - 00:02:45]
**EN:** And those kind of theories, I think, ended up being at least at the surface level reading, not really the best explanation for why we'd see the concentration. What's it been like for you basically doing a lot of research from a theoretical perspective? And I know you do have a background in the mechanism design game theory, but then also working on products related to some of these concepts. How has that journey been for you in the last few years?

**CN:** Quintus：但事实证明，这类理论至少从表面解读来看，并不是解释我们今天所看到的市场份额集中现象的最佳原因。  
主持人：从理论视角做了那么多深度的前沿研究，接着又参与开发与这些概念相关的实际产品，对你来说是一种怎样的体验？我知道你拥有机制设计和博弈论背景，但这几年将理论与产品结合的历程是怎样的？

### [00:02:46 - 00:03:07]
**EN:** Okay, so I think maybe to lay out that journey, I was doing much more research for like, let's say 2023, 2024. And in 2025, I spent a lot of it working on like, silicon security, a very different topic. And now I've come back in 2026 in a much more product oriented capacity.

**CN:** Quintus：好的，如果要梳理这段历程的话，大概在 2023 年和 2024 年，我主要聚焦在纯理论研究上。到了 2025 年，我花了大把时间研究芯片/硬件安全（silicon security，涉及 TEE/SGX 可信执行环境），那是一个截然不同的领域。而现在到了 2026 年，我又回过头来，以一种更加侧重产品落地的方式投入工作。

### [00:03:07 - 00:03:36]
**EN:** And so my view is very fresh on how these things differ. I think that a company's kind of like even any organization, especially FlashWatts is this machine that solves problems at like different time horizons, where researchers are thinking, like step back, not all research, a lot of the research is taking a step back, trying to understand some things, but the present maybe, but mostly because we're trying to think what's possible, or what are the longer term trends and these kinds of things.

**CN:** Quintus：因此，对于理论研究与产品实践之间的差异，我现在的视角非常鲜活。我认为一家公司或者任何组织——尤其是 Flashbots——就像是一台在不同时间跨度上解决问题的机器。研究人员做的是退后一步去思考（当然并非全部研究都是如此，但绝大多数研究都是跳出当下），试图去理解某些底层事物，虽然有时也关照当下，但主要是为了探索未来有什么可能性，或者推演长期趋势等等。

### [00:03:36 - 00:04:36]
**EN:** Whereas on the extreme end of product, there's like people who are responding to the day to day, like, oh, we have a little bit of like extra load on these machines, like let's move it over. You know, it's very much more short term response cycle. And if you iterate too greedily on just the next thing as a whole organization, you maybe end up in a local maximum and miss the big things. But at the same time, if you don't have that connection with reality, then you end up going away, you know, some are very interesting, but the probability of you actually like meeting the reality where it is, it's very low. So I guess the feedback is way tighter. And there's just more market reality, right? It's like, you can think of the most beautiful, interesting mechanism. But if, if it's like, requires a lot of code changes for someone to integrate, and, you know, the improvement you're making might be substantial within the mechanism.

**CN:** Quintus：而处于天平另一极的产品端，大家应对的则是日常琐碎的需求，比如“这些服务器负载有点高了，我们迁移一下负载吧”。这是一个反应周期非常短的循环。如果整个组织过于贪婪地只关注迭代眼前的下一件事，你可能会陷入局部最优（local maximum），从而错失真正的大方向。但反过来，如果你脱离了现实，你最终可能会走向另一个极端——研究方向或许非常引人入胜，但真正能落地契合现实市场的概率却极低。所以产品端的反馈回路要紧密得多，它面临着更多的市场现实。也就是说，你可以构想出最精美、最有趣的机制，但如果它需要对接方修改海量代码才能完成集成，即便你在该机制内部带来的提升非常可观——

### [00:04:36 - 00:05:05]
**EN:** It's almost 10% better or something like that. But if, you know, if the amount of the overall market that you're improving, Ethereum's transaction landing rates, but it turns out that like people who are integrating or landing transactions on thousands of blockchains, or maybe, you know, most of those don't matter, but like many blockchains and 10% improvement on Ethereum is, you know, 0.1% of everything they care about. And so, you know, you're way down the priority list. That doesn't work. So these are all of these practical realities you have to take into account.

**CN:** Quintus：——比如机制内性能提升了将近 10% 之类，但如果你在整个大盘中改善的只是以太坊上的交易落地率（landing rates），而集成的对手方其实在成百上千条区块链（或者说许多条链）上提交交易，对他们而言以太坊上 10% 的改善只占他们关心的业务总量的 0.1%，那么你的优先级就会被排到最末尾。这就行不通了。因此，所有这些现实痛点都是必须纳入考量的。

### [00:05:06 - 00:05:18]
**EN:** Did it help, like going into this year, at least kind of taking some time off from some of this theoretical research? And did it give you fresh perspective coming in, like this year, like where you feel energized to start to into prop AMMs?

**CN:** 主持人：那么，暂时离开这类纯理论研究一段时间，是否对你迈入今年有所帮助？它是否给你带来了全新的视角，让你精力充沛地开始深入专有做市 AMM（PropAMMs）这一领域？

### [00:05:19 - 00:06:17]
**EN:** Yeah, I think the feeling really is like, the Ethereum transaction execution pipeline has ended up in this position where it hasn't moved forward as much as it should have in many dimensions. And the world has, you know, progressed and it hasn't adapted. And so now there's like this potential energy where, okay, we have to adjust to its new world, these new dynamics, actually go beyond them. So it feels like there's a lot of potential energy, just has to come back to and I'll give an, you know, a concrete example. I went to eat Denver at the start of the year. And all of the conversations I was having with people were about tightening spreads, about improving inclusion rates, about, you know, reducing flow toxicity. And these are circles that were typically a bit more infrastructure oriented. They used to be focused more on auction design, and these kinds of things.

**CN:** Quintus：是的，我真实的感受是：以太坊的交易执行流水线（pipeline）目前陷入了一种停滞，在诸多维度上都没有达到应有的进展速度。外部世界在不断向前演进，而它却没有及时适应。因此现在积聚了一种巨大的“势能”——我们必须主动适应这个新世界、这些新的微观动态，甚至超越它们。我能强烈感受到这种蓄势待发的势能。举个具体的例子：今年年初我去了 ETHDenver，当时我和大家聊的所有话题，全都是关于收窄点差（tightening spreads）、提升打包率（inclusion rates）、降低订单流毒性（flow toxicity）。要知道，这些圈子里的人过去往往偏向底层基础设施，此前主要探讨拍卖机制设计等课题。

### [00:06:18 - 00:06:47]
**EN:** And now the framing is very different than language. And like people think about it as more trading oriented. And I think that's just kind of a natural evolution of the market. You see a lot more people taking risk, right? You have like market makers, more than you have solvers who are atomically routing trades. That still exists. But it's like a lot of the cow shop solvers now or like, it's basically just market makers. And there are a couple of atomic ones. But, and I think it's, it's a good thing, to some degree, because, like I said, it's a bit more in touch with reality.

**CN:** Quintus：而现在的探讨框架和专业语境截然不同了，大家更多是以交易为导向来思考。我认为这只是市场的一种自然演进：你会发现越来越多的人开始主动承担持仓风险（taking risk）。比如市场上的做市商（market makers），数量已经远远超过了纯粹做原子化路由交易的求解器（solvers）。原子路由当然仍然存在，但现在像 CoW Swap 上的很多求解器，本质上基本上就是做市商，只剩少数几家是纯原子路由的。在某种程度上我认为这是件好事，因为正如我所说，这更接地气、更贴近交易的商业现实。

### [00:06:47 - 00:07:30]
**EN:** to talk about spreads and you show things as opposed to the like properties of the market where it like lends itself to concentration. These are also important things that we need to talk about. And I think right now we probably aren't talking about the like bigger picture market design as much as we should. But there is just like a lot of inefficiency that has come from infrastructure kinds of people like Flashbots and others like us who have been thinking a bit more too much in terms of transactions as generic data objects, you know, as opposed to this is a swap that is swapping this many dollars and like it's routing through these pools. And that shift I think will bear fruits.

**CN:** Quintus：去切实讨论点差以及这些具体的执行表现，而不是空谈容易导致垄断集中的市场抽象属性。当然，后者也是我们需要探讨的重要命题，而且我觉得目前我们在更高维度的宏观市场机制设计上的讨论可能还不够深入。但是，过去很多低效恰恰是因为像 Flashbots 以及我们这类做基础设施的人，把交易过度抽象化为泛型的数据对象（generic data objects），而不是切中其经济实质——这是一笔涉及具体金额的代币兑换（swap），正在穿透路由过这些具体的流动性池。我认为向交易实质的认知转变一定会结出硕果。

### [00:07:30 - 00:07:57]
**EN:** I really like that framing. Yeah, I've spent time with a lot of people thinking about transactions as objects rather than specific discrete things like a swap or payment. So building on that, I think you mentioned the transaction supply chain. Maybe I think this would be a good point just for before we get into prop AMMs, if you could just kind of update people on what that looks like today in practice. And then after that, we can talk a little bit about like the different swap venues that exist and then we can get into prop AMMs.

**CN:** 主持人：我非常喜欢这个视角。是的，我也接触过很多把交易仅仅当成抽象对象、而非具体兑换或支付行为的人。顺着这个思路，既然你提到了交易供应链，在正式深入专有做市 AMM（PropAMMs）之前，或许可以请你先为大家梳理一下当今实际运行中的交易供应链全貌。之后我们再探讨现存的各种兑换场所（swap venues），进而切入 PropAMM。

### [00:07:57 - 00:10:04]
**EN:** Yeah, okay, complex topic. So okay, so let's walk it through from the user's perspective. There are two flows, a transaction based flow and a intent based flow, intent just meaning that the user's signature doesn't define all that much about the execution and a second party still required to sign to wrap that intent in a transaction for it to be executed. Thinking about the trading use case, users will sort of the front end of their choice and the mechanism by which they get directed to a front end is very web to advertising and apps and integrations and into wallets and all these kind of things. But anyway, the user ends up wanting to make a trade and the system will query in the transaction based flow will query for some call data from somewhere. If you go to the Uniswap front end, you will get call data from a system Uniswap runs and they actually have two flows, the others uni X system and we can leave that inside for now. We'll just do the simple flow that you sort of responds with a route that they calculated some call data for the user to sign and we can get into how that happens in a second. But okay, just to have this route user signs it and then that gets sent somewhere and who makes a decision of where that goes is an important question. In transactions, it's typically the wallet that decides where the transaction goes. Now, most wallets today use an order flow auction OFA provider. OFA providers are sort of this intermediary that sits between the user, the client and maybe the site, either client and then the wallet and the blockchain. And these OFA's will take transactions and optimize for value to the wallet usually or value back up the supply chain and for inclusion rate. And so usually how this happens, they'll do a backgrounding option. Through one mechanism or another, they'll say, Hey, guys, here's a transaction.

**CN:** Quintus：好的，这是一个非常庞杂的话题。让我们从终端用户的视角来完整梳理一遍。目前主要存在两条路径：基于交易的路径（transaction-based flow）和基于意图的路径（intent-based flow）。意图的意思是指，用户的签名本身并没有对具体执行路径进行过多强约束，仍需要第三方签名将该意图包装进一笔交易中才能被执行。  
以典型的交易场景为例，用户首先会打开他们选择的前端界面（用户被引流至该前端的机制往往充斥着 Web2 式的广告、DApp 推荐、集成以及钱包内置入口等）。但不管怎样，用户产生了交易需求。在基于交易的路径中，系统会向某个服务查询调用的 calldata 数据。如果你使用的是 Uniswap 前端，你会从 Uniswap 运营的后端系统中获取 calldata。实际上他们也有两套流，另一套是 UniswapX 系统，我们先把它放一边。先说简单的传统流程：系统返回计算好的最优路由以及相应的 calldata 供用户签名（稍后我们会讲路由是如何生成的）。用户对这条路由签名后，交易就会被发送出去。那么，由谁来决定将这笔交易发往何处？这是一个非常关键的问题。对于普通交易而言，通常是由钱包来决定路由目的地。  
如今，大多数主流钱包都会接入订单流拍卖（OFA）服务商。OFA 服务商充当着位于用户/客户端（或 DApp 站点/钱包）与底层区块链之间的中间层。这些 OFA 会接收交易，并针对钱包捕获的价值（或向交易供应链上游回馈的价值）以及交易打包率进行优化。其通常的运作方式是发起跟跑拍卖（backrunning auction）。通过某种竞价机制，他们会向外界宣告：“嘿，各位，这里有一笔交易——”

### [00:10:04 - 00:11:13]
**EN:** It's probably not going to execute optimally or often it won't. It'll create arbitrage opportunity. Here's this auction system filled with searches, more market makers, whatever term you want to use. Let's auction off the right to background this transaction. And then again, there's this whole dance that happens between order flow auctions and the builders, but eventually these bundles of user transaction and background or maybe even just user transaction get sent to some builder and or multiple builders usually. And in these these builders combine all the flow they've seen also through a relatively like involved process and form blocks, which they then if the block is more valuable than the previous one will form a bid for and send that off to relays, which is a pathway to this like PBS auction and winning block builder will get to include their block at which point the user's transaction lands. I feel like I gave a very long and winding explanation there, but I also feel like I left out a lot of important detail. But anyway, that's kind of how it works. The simple case, how it works right now.

**CN:** Quintus：“——这笔交易的执行很可能不是最优的，或者说往往并非最优，它会留下套利空间。这里有一个聚集了搜索者（searchers）或做市商的拍卖系统，大家来竞拍对这笔交易进行跟跑（backrun）套利的独占权利吧。”  
接下来，在订单流拍卖（OFA）与区块构建者（builders）之间还会展开一系列复杂的博弈与协调。但最终，这些包含了“用户交易 + 跟跑套利交易”的交易包（bundles，有时也可能只包含用户交易本身），会被发送给某一个或通常是多个区块构建者。这些构建者通过一套相当复杂的算法，将他们看到的所有订单流组装成候选区块。如果构建出的新区块价值高于前一个，他们就会生成相应的出价（bid）并提交给中继器（relays）。中继器连接着提议者与构建者分离（PBS）拍卖环节；最终获胜的区块构建者将成功打包并上链该区块，此时用户的交易便正式落地（lands）。  
我感觉自己的解释有点冗长曲折，同时也省略了不少关键细节。但总的来说，目前最基础的交易流程大致就是这样运转的。

### [00:11:13 - 00:11:34]
**EN:** One thing I want to zero in on though that you just mentioned was two things. One was the route. Who chooses the route? What does that look like? And then the other thing is like, what's the reason that there is an order flow auction? Why do you need that? Like, why does the background even exist? Okay, this is getting to the importance. So blockchains, true to the name operated blocks.

**CN:** 主持人：我想就你刚才提到的内容追问两点：第一是路由。究竟是谁在决定路由？实际流程是怎样的？第二是，为什么会存在订单流拍卖（OFA）？为什么我们需要它？或者说，为什么会天然存在跟跑套利（backrun）的机会？  
Quintus：好，这切中了问题的核心。区块链，名副其实，是以“区块”（blocks）为单位离散运行的。

### [00:11:34 - 00:13:27]
**EN:** And so the way information about the market is revealed to the public is in discrete discrete time intervals. And the way that basically the update happens is also to discrete time intervals. Okay, so firstly, that's important context for how the route is formed before we even talk about that. Also on the blockchain, there are multiple sources of liquidity, right? And these are like AMMs, like Uniswap and Curve that just like, will offer you a price token in token out. But it could also in some more extreme versions be liquidations, source of liquidity, typically doesn't use to get full users, but it's also on chain. Also private liquidity like a winter mute any kind of market maker will have a bunch of capital in their own smart contracts on chain. And because there are many of these different contracts and these contracts will change their price based on how people trade against them. There is an optimal way for the user to execute a trade, which will usually go through many of these liquidity pools on chain, many of these AMMs. And because they keep, you know, changing as other people trade through them, the exact prices and therefore the best route through these pools is not always changing. It's always changing. Well, it's changing in discrete intervals, but from the user's perspective, like unless you're landing at the top of the block, can you send your transaction in the last 12 seconds on Ethereum because blocks are 12 seconds apart, you don't know what the prices are going to be exactly. And so yeah, so it's always changing. Of course, if you trade some very rare, very like a low volume in liquid assets or maybe not, but generally speaking. And so anyway, so these routers have to look at, they have to take their best guess at what the on-chain prices are and then form a route through these on-chain liquidity sources and give that to the user.

**CN:** Quintus：因此，市场信息的公开披露是以离散的时间间隔进行的；状态的更新本质上同样发生在离散的时间区间内。在讨论路由是如何形成之前，这是首要的重要背景。  
其次，链上存在着多种流动性来源。比如像 Uniswap 和 Curve 这样的恒定函数做市商（AMMs），只要你输入代币就会按既定曲线输出代币；在一些极端场景下，清算也构成了一种流动性来源（虽然通常不面向普通用户，但同样在链上）；此外还有私有流动性，像 Wintermute 等各类做市商都会将大量资金部署在自己的链上智能合约中。由于存在大量不同的合约，且这些合约的定价会随着人们的交易交互而变动，因此客观上存在一种针对用户交易的最优拆单执行路径，通常会穿透多个链上流动性池和 AMM。  
而随着其他交易者不断在这些池子中交易，各池子的精确价格，以及穿透这些池子的最佳路径一直都在变动——虽然在区块链底层它是按离散区间变动的，但从用户的视角看，除非你能精准抢到区块最顶端（top of the block），否则你在以太坊上发出交易到出块的这 12 秒间隔内，根本不可能确切知道最终的成交价格。所以对用户而言，价格始终在动态变化（当然，如果你交易的是极低流动性、极冷门的长尾资产可能波动不大，但普遍而言皆是如此）。因此，各大路由算法必须尽可能准确地预测链上当前的价格分布，规划出穿透这些链上流动性源的路径并呈现给用户。

### [00:13:27 - 00:14:06]
**EN:** And there is a whole class of systems people use to get these quotes to the users. If a user just gets a quote from one specific router like Uniswap, that's kind of like, okay, there's a lot of new ones are leading up to Uniswap because I also have an auction. But if you just go from one router, you're just sort of saying, I assume these guys are going to give you a good route. Other systems have an auction for different routers to provide different quotes. But even then, at the end of the day, the user and the auction operator don't know exactly which route is going to give the best execution. Then user signs their transaction, it gets shipped off. There are sources of liquidity that give firm quotes.

**CN:** Quintus：行业里有一整套专用于为用户提供报价的系统架构。如果用户仅仅从某个单一路由器（比如 Uniswap）获取报价——这里面有许多细微差别，因为 Uniswap 自身也有拍卖机制——但如果你只信任一个路由器，就相当于默认他们给出的路由是优秀的。还有一些系统会举办多路由器聚合竞价，由不同的路由算法提供不同的报价。但即便如此，归根结底，在交易尚未打包之前，用户和拍卖运营方都无法百分之百确知究竟哪条路由会带来绝对最佳的执行结果。随后用户对交易进行签名并发送出去。  
另外，还有一类流动性能够提供确定性报价（firm quotes）。

### [00:14:06 - 00:14:52]
**EN:** And this is kind of like the private liquidity on-chain that I was mentioning. You have these RFQ systems that operate off-chain, where basically you can ping, I'll use Windermute as an example, because they're probably the best-known market maker in the space. You ping that server, they'll give you, usually a soft quote and say, hey, yes, you asked for 100 ETH, we'll give you 100 ETH at $5 an ETH or something like that. If you confirm, they'll send you a message, which you can execute on-chain and basically take ETH out of their private contract, provided you provide enough USDC at the specified price. And these are firm quotes, usually, because the market maker will have enough capital and the prices also will not be determined by on-chain logic, but rather by its messages they sign.

**CN:** Quintus：这就是我刚才提到的链上私有流动性。比如运行在链下的询价（RFQ）系统：你可以向做市商发起请求（以行业最知名的 Wintermute 为例），向他们的服务器发送请求后，他们通常会先返回一个软报价（soft quote），例如：“好的，你需要 100 枚 ETH，我们按每枚 5 美元（假设价格）给你成交。”如果你确认接受，他们会签署一条确定性消息发给你。你可以直接在链上执行这条签名消息，只要你在指定价格下提供足额的 USDC，就可以从他们的私有合约中提走 ETH。这通常是确定性的硬报价（firm quotes），因为做市商具备充足的自有资本支撑，而且成交价格不再由链上的 AMM 曲线逻辑决定，而是直接取决于他们签署的离线报价消息。

### [00:14:52 - 00:15:41]
**EN:** And the market maker is also in the background positioning off-chain, doing things in order books on centralized exchanges and other venues simultaneously, right? Yeah, exactly. But the market maker is doing a bunch of stuff that also actually affects the user's trade in other ways, because the trading against the liquidity the user will also trade against. But I think for this picture, you can just think of them as these providers of firm quotes. And that plays a big role in what kind of counterparty the user ends up executing against, because the on-chain liquidity is unpredictable, the market maker's liquidity is predictable, the prices are predictable. So depending on how the auction is structured, there'll be a systematic advantage for the market maker's firm quotes, because people discount uncertainty, maybe even just from a UX perspective.

**CN:** 主持人：而且做市商同时还在后台链下进行头寸管理，在中心化交易所（CEX）的订单簿以及其他交易场所同步对冲，对吧？  
Quintus：没错，完全正确。做市商在后台操作的一整套流程，其实也会在其他层面反过来影响用户的交易，因为他们对冲时消耗的流动性往往正是该用户原本也可以交易的流动性。不过在目前的宏观图景中，你可以简单将他们视作“确定性报价的提供者”。这在很大程度上决定了用户最终会与哪种类型的对手方撮合成交——因为链上 AMM 流动性具有不可预测性，而做市商提供的流动性及其价格是完全可预测的。因此，根据拍卖机制的设计不同，做市商的确定性硬报价往往会占据系统性优势，因为无论从系统本身还是用户体验（UX）的角度看，人们天然会对不确定性打折扣。

### [00:15:41 - 00:16:31]
**EN:** And so we've seen this kind of evolution, I think, over maybe the last few years, where originally you had a lot of concentration in on-chain liquidity. On-chain liquidity can't quote the tightest spreads oftentimes. And also at the same time, passive liquidity providers are getting picked off from sophisticated players. And so as a result of this, you started to see more of these RFQ systems pop up. And oftentimes the quotes are going to be better if they're trading maybe like a major pair, something that's really well-held by a market maker versus if they are trading something that's longer tail, which is like 99% of most of the stuff on-chain, then it's going to route through one of these complex flows where it touches maybe multiple AMM pools. How has that changed the trading environment? And then maybe we can use that as kind of a bridge to getting to prop AMMs.

**CN:** 主持人：过去几年我们确实目睹了这种演化：最初大部分流动性高度集中在链上 AMM 中；但链上 AMM 往往无法报出最窄的点差，与此同时，被动的流动性提供者（LP）不断被成熟的高频专业玩家套利收割（picked off）。于是，我们看到越来越多的链下 RFQ 系统涌现出来。通常如果用户交易的是主流交易对——即做市商重仓持有、深度极好的币种——RFQ 报出的价格往往更好；而如果交易的是长尾资产（占链上资产的 99%），则仍需穿透复杂的路由路径，触达多个 AMM 资金池。这种格局是如何重塑整个链上交易环境的？或许我们可以以此为桥梁，正式切入专有做市 AMM（PropAMMs）。

### [00:16:31 - 00:17:17]
**EN:** I guess there's a couple of different things to explain there. I think if you go back 18 months, the focus in the Ethereum DeFi trading community leading start lending was how we sustainably provide liquidity in AMMs. A market maker is a service provider in a market, and the service is basically like convenient access to liquidity. Because if the market maker wasn't there and you went to sell it, you'd have to wait for someone who will come around and buy it from you. And market makers are kind of just natural. It kind of just pops up as a solution to markets because they lend themselves to it, and they're also like a service provider to markets. So there's always someone to take the other side of your trade at some price.

**CN:** Quintus：这里面有几个不同层面的逻辑需要拆解。如果回溯到 18 个月前，以太坊 DeFi 交易社区关注的焦点（后来扩展到借贷等领域）主要是：我们究竟如何在 AMM 中可持续地提供流动性？  
做市商本质上是金融市场中的服务提供商，他们提供的核心服务就是便利的流动性获取渠道。因为如果没有做市商的存在，当你想要卖出资产时，你必须原地等待另一个想要买入的人恰好出现。做市商的诞生是金融市场演化的自然产物，因为市场本身具有这种撮合需求，做市商充当服务方，确保无论何时何地，总有人愿意在某个价格上作为对手方接下你的单子。

### [00:17:17 - 00:18:46]
**EN:** And the market maker's job is to know the price and to take a fee on the price. If they know in some sense the true price is going to be $100 right now. If they sell higher than that and buy lower than that, they're always making money. And they make money in exchange for providing the liquidity when you need it. But from a market maker's perspective, it's actually very hard to know the real price. The real price is actually getting hard to define here because the price is going to move in the next millisecond. And what ends up happening is you get informed trade. There's people who might know more about the price than the market maker. And trading against them, especially if they take too small of a fee, you can end up being expensive because the market maker thinks the price is X. The market maker knows the price is going to X plus one. The market maker is only selling at X plus a half. And that's actually a bad trade for the market maker. And so the informed taker will do that. And they'll do that consistently. And what happens with AMMs is we say, well, we need these market makers to exist. We don't want to rely on a specific central counterparty when it's like source these creatives to centralize market maker, basically, put it on a smart contract and allow people to pool capital to-- this is like what Eunice wrote with AMMs-- pool capital to act as a market maker and to collectively earn the rewards or stuff to the losses of this market maker entity. And sometimes people do that to make money.

**CN:** Quintus：做市商的核心职责就是洞察价格，并在公允价格的基础上收取一笔手续费/点差。如果他们在某种意义上知晓当前真正的公允价格是 100 美元，只要他们以高于 100 美元卖出、低于 100 美元买入，他们就始终能赚取利润；他们通过在你需要时提供即时流动性来获取报酬。  
但对做市商来说，知晓所谓的“真实价格”其实极度困难。甚至连真实价格本身的定义都变得模糊，因为下一毫秒价格就会发生位移。这样一来就会出现知情交易（informed trading）——市场上总有一些参与者掌握的信息比做市商更多。如果与这部分知情交易者交易，尤其是在做市商设定的手续费/点差太小时，做市商将付出沉重代价：假设做市商以为公允价是 X，知情交易者已经知道价格即将涨到 X+1，而做市商却依然在以 X+0.5 挂单卖出，这笔交易对做市商而言就是绝对亏损的劣质交易。知情吃单者（informed takers）会持续不断地这么做。  
而传统 AMM 的出现逻辑是：既然我们需要做市商，又不想依赖某个中心化的特定对手方，那不如将做市机制众包并去中心化——把算法写进智能合约中，允许大众聚集资金池（这正是 Uniswap 开创恒定函数做市商 CFMM 的核心理念），共同扮演做市商的角色，集体赚取收益或承担该做市实体的亏损。有时人们提供流动性是为了赚钱；

### [00:18:46 - 00:19:22]
**EN:** Sometimes people do that because they just want liquidity for a specific token. A foundation may do this. But the problem is that the automated market maker is doing no intelligent pricing of its own. It's relying on traders, on takers, to correct the market maker's price with every trade. And so there's a lot of adverse selection for the market makers. And there's a specific effect that makes it worse. The market makers only update their prices at every block. At the end of every 12 seconds at the moment. And so the price moves a lot in 12 seconds.

**CN:** Quintus：有时人们提供流动性则仅仅是为了给某个特定代币注入流动性，比如代币基金会。但核心症结在于：自动做市商（AMM）自身没有任何智能的主动定价能力！它完全被动地依赖外部交易者（吃单者）通过每一次交易来纠正 AMM 资金池的报价。  
这就导致做市商面临着极其严重的逆向选择（adverse selection，即知情交易毒性侵蚀，这也是 LVR 的核心来源）。更致命的是，还有一个特定机制让情况雪上加霜：在以太坊目前的机制下，链上做市商只能在每个区块打包时更新一次价格——也就是目前每 12 秒才更新一次。而在这漫长的 12 秒内，外部真实市场的价格早已发生了剧烈波动。

### [00:19:22 - 00:19:52]
**EN:** And so the market makers are very wrong at the top of every block. And so there's a very common trade that people do, which we call a sextex arbitrage, or statistical arbitrage. But it's just like, look at the price on Binance. Now people have way more sophisticated pricing models. But the market makers have some pricing model that takes into account information that arrived in the last 11.9 seconds. And then they trade the AMMs up and to that point.

**CN:** Quintus：因此，在每个区块的最顶端（top of the block），链上做市商的报价几乎注定是严重滞后且错误的。于是便诞生了一种极为普遍的套利策略——我们称之为 CEX-DEX 套利（CEX-DEX arbitrage）或统计套利。其最直观的形式就是盯住币安（Binance）等中心化交易所的价格。虽然现在套利者的定价模型要精细复杂得多，但本质上都是套利者结合过去 11.9 秒内到达的所有市场增量信息构建定价，然后在区块第一笔交易中将 AMM 的价格精准打到这个最新点位。

### [00:19:52 - 00:20:00]
**EN:** And there's a lot of complexity in executing this trade in reality, because you're competing with other people to do it. But because of that, the

**CN:** Quintus：在现实中执行这类套利交易极其复杂，因为你必须和其他顶尖高频套利者激烈竞速。但正因如此，这种机制导致——


---

## Part 2 (00:20:00 - 00:40:00)

# 《Deeply Intents》深度访谈：PropAMMs 正在重塑 DeFi 市场格局（Part 2）

**嘉宾：** Quintus（Flashbots 机制设计研究员）  
**主题：** 专有做市 AMM（Proprietary AMMs / PropAMMs）、LVR 消除机制、优先费排序不对称性与区块构建者（Block Builder）生态演变  

---

### 本期概要（Executive Summary）
在 Part 2 的对话中，Flashbots 研究员 Quintus 与主持人深入剖析了专有做市 AMM（PropAMM）在高性能 L1（Solana）及 EVM 生态（以太坊、Base）中的崛起逻辑与市场结构影响：
1. **传统 AMM 的致命困境与范式转移：** 传统恒定函数做市商（CFMM）中的被动流动性提供者（passive LPs）长期承受巨大的 LVR（损失与再平衡）损失与信息优势流剥削。与其在链上尝试复杂的动态费率花招，不如将价格发现交由链下做市商（作为服务提供者）在出块最后一刻更新。
2. **PropAMM 的两大解决路径：**
   - **零代理风险：** 专有做市商使用自有资金做市（Proprietary Capital），彻底消除了委托-代理问题。
   - **优先费排序与不对称性优势：** 在 Solana 及 Base 等支持优先费排序的链上，做市商通过编写极度轻量化的报价更新合约，以极高的单位优先费（Priority Fee Rate）排在区块前列，却仅消耗极少 Gas 成本，从而在经济学上压制了高 Gas 消耗的套利吃单者（Takers）。
3. **从 Gas 技巧到显式协议原语：** 市场正从“利用排序不对称”转向“显式区块拍卖原语”，如 Jito 在 Solana 上的 BAM（区块拍卖机制）以及以太坊 Builder 提供的顶部更新通道（Top-of-Block Priority）。
4. **相比传统 RFQ 的降维打击：** 彻底终结了 RFQ 机制下长达 30~60 秒的“陈旧期权风险（stale quote option）”与高昂的声誉维护成本；做市商得以流式更新毫秒级最新价格，提供更窄点差（Titan 看板显示为终端用户改善约 8 bps），并天然保留了链上可组合性。
5. **对交易供应链（MEV Supply Chain）的深远影响：** 验证者（Validators）暂未受波及，但区块构建者（Block Builders）面临极高的信任与声誉门槛——延迟做市商更新将导致灾难性的逆向选择；同时，吃单方也面临虚假报价（Spoofing）与滑点保护机制缺失的挑战。

---

### [00:20:00 - 00:21:11]
**EN:** Market makers lose a lot of money. This is an extremely long-winded explanation. The market makers lose a lot of money, passive LPs lose a lot of money in these AMMs. And what's happened with PropAMMs is that, you know, people have just said, like, you know, there's all of these design tricks that you're trying with the AMMs to try and dynamically price things so that you like minimize the, like, you know, the toxicity, the cost of trading against these informed takers while still making money off the like uninformed takers. But at the end of the day, like, you're just not making, you're just making or making money. It's way better if you just allow some other party to incorporate the information from 11.9 seconds, not through trading against you, but as like a service provider. And this wasn't something I mean, people theorize about this a lot, but I think there was a general rejection of the use of off-chain oracles. They don't want to rely on these off-chain parties. But then what happened on priority fee order chains is people found a way, okay, now we're getting into the auction. Anyway, let me pause there and then I can keep going. Yeah.

**CN:** **Quintus：** 做市商损失极其惨重。虽然这是一个非常冗长的解释：但在传统的 AMM 中，做市商亏损严重，被动流动性提供者（passive LPs）也承受了巨额损失。而专有做市 AMM（PropAMM）之所以诞生，是因为人们终于意识到：尽管大家在传统 AMM 上穷尽了各种精巧的机制设计，试图通过动态定价来最小化毒性订单流（toxicity）、降低与知情交易者（informed takers）对手交易的成本，同时又想从无知情交易者（uninformed takers）身上赚取手续费，但归根结底，你根本就赚不到钱。与其这样，远不如直接允许某个外部第三方在出块前的最后一刻（比如第 11.9 秒）把最新的外部市场信息整合进来——不是通过与池子进行套利交易把价差吃干抹净，而是将其作为一种专业信息服务提供方。这并不是什么凭空捏造的理论，学术界和行业内探讨了很久，但我认为此前加密世界普遍排斥使用链下预言机（off-chain oracles），大家极度抗拒依赖这些链下第三方。然而，随后在支持优先费用排序（priority fee ordering）的区块链上，人们找到了一条破局之路……好吧，我们这就涉及拍卖机制了。总之我先停在这里，待会儿继续展开。

---

### [00:21:11 - 00:21:57]
**EN:** Yeah. So I think you explained automated market making really well, some of the downsides and the negative externalities to providing liquidity there. You also explained how market makers make money and how they lose money and the importance of quoting accurately. And one thing that we did see is I think we saw this first innovation happen on lower latency chains first, right? Like we saw on Solana, kind of this emergence of what we're calling proprietary or prop AMMs. And as I understood this, I mean, please correct me if I'm wrong here, is that the market maker basically has a contract on chain where they can update the quote right at the last second, right at block time, to give the user the freshest quotes trade against. And they've integrated with a number of DEX aggregators.

**CN:** **主持人：** 没错。我认为你刚才把自动化做市机制（AMM）解释得非常透彻，包括在其上提供流动性的各种缺陷以及负外部性。你还讲清了做市商是如何赚钱与亏钱的，以及精准报价的重要性。而且我们确实看到，这项创新最先发生在更低延迟的公链上，对吧？比如在 Solana 上，率先涌现出了我们所说的专有做市 AMM（Proprietary AMM / PropAMM）。按照我的理解——如果说得不对请纠正我——做市商本质上是在链上部署一个自营合约，他们能够在出块的最后一秒、也就是出块时刻实时更新报价，从而让终端用户始终能与最新鲜的报价进行交易。并且他们已经与多家主流 DEX 聚合器完成了深度集成。

---

### [00:21:57 - 00:22:50]
**EN:** So when a DEX aggregator is then quoting to the end user on the front end, they are basing that off of the last quote that they will get maybe from a variety of different prop AMMs that exist, whether it's humidify, sulfide, or maybe something else that we don't know about. And this has actually been something on Solana that's been a trend, I think maybe starting in Q3, Q4 last year. And now we're seeing kind of this emergence in the EVM space with some announcements recently. And so I think it would be good maybe at this point to just talk about prop AMMs, maybe define them. You mentioned the soft chain Oracle, and maybe also then talk about kind of the trade off between that you need to make in order to incorporate it on an Ethereum in a 12 second slot time versus maybe like a Solana 400 milliseconds, or even as you noted, like a priority ordering chain, like that's using flash blocks or something like this.

**CN:** **主持人：** 因此，当 DEX 聚合器在前端向终端用户展示聚合报价时，他们依据的是从当前存在的各个专有做市 AMM（PropAMM）那里获取的最新报价，不管是 Humidi.fi、SolFi，还是其他我们尚不知晓的协议。在 Solana 上，这已经成为一股不可逆转的趋势，大概始于去年第三或第四季度。而现在，随着近期的一系列重磅发布，我们看到这种范式也开始在 EVM 生态中崭露头角。所以我觉得在此时深入探讨一下 PropAMM 并对其进行明确定义会非常有帮助。你刚才提到了链下预言机（off-chain oracle），或许还可以谈谈在以太坊 12 秒的时隙（slot time）与 Solana 400 毫秒的时隙之间整合它所需的权衡取舍，或者正如你所提到的，在基于优先费用排序、采用类似 Flashblocks（闪电区块/亚秒级预确认区块）的链上又是怎样的考量。

---

### [00:22:50 - 00:25:23]
**EN:** Yeah. So okay. So prop AMMs do solve two problems that AMMs had. So what happened with AMMs is people weren't willing to rely on external price sources. Why? Because now whoever's providing this price information, as it's like a service to the AMM, is a trusted party for the passive LPs who are putting their money in this on-chain AMM. And the other problem was that the way in which access to the blockchain was mediated was in many block changes, but I'm through a kind of an auction. And so even if you had an off-chain service provider who was willing to provide information every 12 seconds or every next block, they would have to outbid all of the takers, all of the arbitrageurs. And this is very costly to outcompete. If the market maker has to pay as much as the taker to be able to update their prices, then they have lost just as much as they would lose anyway, often very close to it. And so what happened with prop AMMs is some changes ended up implementing priority fee ordering, which is somewhat of a trust-based system, because you're relying on whoever's building the block to follow the rules. And you can verify that they order things by priority fee, but you can't verify that they included every transaction. And that ends up being an important assumption here. So assuming that whoever's building the block is just following the rules, taking all the transactions they receive and ordering them from highest to lowest fee, you can write the smart contracts so that updating on-chain prices ends up being much cheaper than trading against those prices. And so when I say priority fee ordering, I mean, your charged user pays a fee times how much on-chain compute they use, how much gas they use. And the amount of gas is defined by the smart contract, but the ordering is defined only by the fee, not the total amount paid. And so if you write a smart contract where it's very cheap to update quotes, you can pay a very large fee, and the total fee will still be quite small. Whereas the takers, they have to pay a lot of gas. And so even a small fee ends up being expensive. And so you have this asymmetry.

**CN:** **Quintus：** 好的。专有做市 AMM（PropAMM）确实解决了传统 AMM 长期存在的两大核心痛点。
传统 AMM 面临的第一个问题是：大家不愿意依赖外部价格源。为什么？因为向 AMM 提供价格信息的角色，在性质上变成了在链上存入资产的被动 LP 必须绝对信任的中心化/特权第三方。
第二个问题是区块链准入权的调解机制：在许多区块链中，交易打包与排序是通过某种竞价拍卖完成的。因此，即便你有一个链下服务提供者愿意每 12 秒或在每个新区块到来时更新一次价格，他也必须在竞价拍卖中击败所有的吃单方（takers）和套利者（arbitrageurs）。而要在竞价中胜出是极其昂贵的。如果做市商为了更新价格必须支付与套利吃单者一样高昂的 Gas 成本，那他们省下的钱就全作为手续费拱手送出了，最终损失往往与直接被套利所差无几（即 LVR 转移给了出块者）。
而 PropAMM 的破局点在于，某些区块链采用了“优先费用排序（priority fee ordering）”机制。这在一定程度上是一个基于信任的体系，因为你依赖区块构建者（block builder）遵守规则。你可以事后验证他们确实是按优先费率（priority fee rate）从高到低排序交易的，但你无法证明他们确实包含了收到的所有交易——这是一个非常关键的前提假设。
只要假设出块者遵守规则、将收到的交易严格按费率从高到低排列，你就可以通过巧妙编写智能合约，使得“在链上更新价格”的计算开销，远低于“与该价格对手交易”的计算开销。
所谓优先费排序，是指用户支付的实际费用等于“单位费率 × 消耗的链上 Gas 计算量”。Gas 消耗量由合约逻辑决定，但交易排布的先后顺序仅取决于单位费率（Gas Price / Priority Fee），而不是用户支付的总费用！
因此，如果你编写一个极度轻量、更新报价 Gas 极低的合约，做市商就可以出极高的单位费率抢占区块第一位，而实际支付的总 Gas 费用依然非常微小；相反，吃单套利者执行交易需要消耗大量 Gas，哪怕稍高一点的单位费率都会让其总费用变得极其昂贵。这样一来，你就在博弈中创造出了决定性的不对称优势。

---

### [00:25:23 - 00:26:46]
**EN:** So that solves the second problem of it being too expensive for market makers to update on-chain prices. But there's a second problem, which is reliance on this external party. And the way they solve this problem is the proper payment is for proprietary. It's a market maker using their own capital. There is no principal agent problem because there's no incentive for the market maker to quote a bad price because they would just be losing their own money. And so that's kind of what's happened. That's what started on Solana, it's started on base, because that's what priority fee ordering was, and that's what the users are, or at least some of them. That model ends up not being optimal for a variety of reasons. If you're going to do this anyway, then instead of having this weird asymmetry written into the block building rules, you could just encode that very explicitly and say, "Oh, here's an API for market makers, and here's an API for everyone else." And if you want to post prices, you get priority in the block, and everyone else becomes a taker. And we've seen that happen now in Solana with BAM, or I think within BAM, from the JITO guys. And that's also what's happening in Ethereum now. Block builders will take updates and put them at the top of the block, and then everyone else goes behind them, and then put them at the top of the block without requiring them to outbid toxic takers.

**CN:** **Quintus：** 这就解决了做市商在链上更新价格成本过高的第二个问题。
而针对第一个问题——即对外部第三方的信任依赖，PropAMM 的解法则在于其名字中的“Prop”（专有/自营，Proprietary）：这是做市商完全使用自有资本做市。这里根本不存在任何委托-代理问题（principal-agent problem），因为做市商绝对没有任何动机故意报出一个偏离市场的劣质价格，否则亏损的完全是他们自己的本金。
这就是它的演进脉络。这最初在 Solana 和 Base 上兴起，因为这些链采用了优先费用排序规则，并且聚集了大量的真实交易用户。
但由于种种原因，单纯依靠 Gas 不对称性的模型最终并不是最优解。既然大家横竖都要做这件事，与其依靠出块规则中这种怪异的 Gas 计算量不对称，不如在协议层直接将其显式编码出来，明确规定：“这是专门开放给做市商的接口，那是提供给其他所有人的接口。只要你是来发布更新报价的，你在区块中就享有绝对优先级；其余所有人则自动成为吃单者排在后面。”
我们看到 Solana 社区如今正是通过 Jito 团队的 BAM（区块拍卖机制，Block Auction Mechanism）实现了这一点。以太坊上现在的做法也如出一辙：区块构建者（block builders）直接接收做市商的价格更新并将其无条件置于区块顶部（Top of Block），所有其他交易跟在其后，完全不再需要做市商去出天价 Gas 竞价击败那些毒性套利吃单者。

---

### [00:26:46 - 00:27:37]
**EN:** So if you're a market maker, and I want to trade against your prop AMM, and we both have competing transactions, in the VM land on Ethereum in particular, the transaction will be placed to update the Oracle, or to update the quote will be placed always first before my transaction would come through and trade against that updated quote. Is that right? That's correct. And there are other flows as well, where the market makers provide flow segmentation that happens, basically. Where the market makers will find ways to discriminate between different traders, the different sources of flow, and give them different prices, basically based on how toxic they seem to be. But yes, it is correct.

**CN:** **主持人：** 这么说，如果你是一位做市商，而我想要与你的 PropAMM 合约进行交易，并且我们两人同时发出了互相冲突的交易，那么在 EVM 生态（尤其是在以太坊主网上），你更新预言机或更新报价的那笔交易永远会被排在最前列，随后我的交易才会进入，并必须严格以你更新后的最新价格进行撮合成交。是这样吗？

**Quintus：** 完全正确。此外还存在其他维度的机制，主要是做市商实现的订单流细分与隔离（flow segmentation）。做市商会通过各种手段对不同交易者和不同的订单流来源进行歧视性识别，并根据其实际表现出的毒性（toxicity）程度，动态提供差异化的报价。但就排序执行逻辑而言，你的理解完全正确。

---

### [00:27:37 - 00:28:42]
**EN:** So as then as a market maker, I'm now streaming quotes, basically, to a block builder to update my contract with the freshest quotes that I think my models are telling me is the price that I should offer to users and their segmentation based off of what price I'm going to offer to either toxic flow that I perceive or just maybe uninformed flow. So Titan just put out a dashboard yesterday. And don't remember what it's called offhand. I'll get into the sec. PAM.WTF, I think. There we go. PAM.WTF. One of the interesting things I've thought about the dashboard was they have this nice comparison showing you the spread on some of these prop AMMs versus what you would get on UniV3. And I haven't audited the information myself, so I can't do anything except trust what's on the page. But what's interesting about this is that it is suggesting that, at least initially, the user is getting better execution by about eight basis points just looking at that chart that they have there.

**CN:** **主持人：** 所以作为做市商，我现在本质上是在持续向区块构建者流式推送最新报价，用内部量化模型测算出的最优价格来实时更新我的链上合约，向用户报价；同时还会进行订单流分流，区分向毒性流（toxic flow）还是向无知情散户流（uninformed flow）提供何种定价。
Titan（Titan Builder）昨天刚刚上线了一个数据看板，我一时想不起确切名字了，等我查一下……对了，叫 PAM.WTF（pam.wtf）。我觉得这个看板非常引人入胜的一点在于，它对一些 PropAMM 与 Uniswap v3 上的实际有效点差（spread）进行了直观对比。我本人还没有亲自审计这些数据，所以只能先相信页面展示的内容。但非常亮眼的是，看板上的图表明确表明，至少在现阶段，终端用户在 PropAMM 上获得的成交执行价格大约能改善整整 8 个基点（8 bps）。

---

### [00:28:42 - 00:29:12]
**EN:** So that's one motivation, which is very clear. You're providing better execution for end users. And if you provide better execution for end users, you're probably going to get more trades against your prop AMM. And then if you get more trades, you can make more money as a service provider. This seems fairly logical. What else, though? What other affordances or what other benefits do market makers get by doing this service? You kind of touched on some of them, but maybe if you could just tease it out a little bit more.

**CN:** **主持人：** 这显然是一个极其明确的商业动力：你为终端用户提供了更优的执行价格。如果用户获得了更好的成交价格，路由聚合器就会向你的 PropAMM 引导更多交易；而一旦交易量上去了，你作为流动性服务提供商就能赚取更多手续费收益。这在商业逻辑上非常通顺。但除此之外呢？做市商通过提供这种自营做市服务，还能获得哪些机制便利（affordances）或额外优势？你之前已经稍微触及过一些，能否再进一步深入拆解一下？

---

### [00:29:12 - 00:30:02]
**EN:** So why do market makers prefer this system? Which is basically a similar question to why the users get pretty good prices in the system. Because it works out to market makers, and because market makers are competing against each other, they have to offer tight quotes to the users. And so that one of that service gets redirected to the user. Market makers like the system because, yes, they don't have the toxic flow that passive LPs would have by being stuck on chain. Another reason, which actually isn't unique to prop AMMs, it's just another feature that's come along at the same time. We saw a bit of it before prop AMMs, which is very prevalent in prop AMMs. They do auto flow segmentation, so they intentionally exclude the toxic takers.

**CN:** **Quintus：** 做市商为什么偏爱这套体系？这其实和“为什么用户在这套系统里能拿到极其优越的价格”是同一个问题的两面。因为这套机制在经济学上对做市商极度有利，而在多家做市商相互竞争的博弈下，他们被迫向终端用户报出极其收窄的点差（tight quotes），从而将这种制度红利直接让渡回馈给了普通用户。
做市商之所以拥抱这套系统，首要原因在于他们彻底摆脱了被动 LP 因资产困在链上而被迫吸收的那种毒性订单流（toxic flow）。
另一个原因（虽然并非 PropAMM 所独有，只是与之伴生普及开来）在于：它普及了自动化订单流分流（auto flow segmentation），做市商可以主动、有意识地将那些具有信息优势的毒性吃单者直接排除在外。

---

### [00:30:02 - 00:32:04]
**EN:** Another reason market makers prefer this relative to RFQ systems, which they were using before, is that in RFQ, you have to commit to a quote, often for 60 seconds. Because this quote ends up being sent to the user when the user has to click. And not all of this is inherent in the way RFQs work. It was just a consequence of the transaction-based supply chain. You could intend based on different. And so for whatever reason, they have to offer a price and stick to it for a long time. And they would still do this auto flow segmentation where they provide quotes to one actor. And then they realize this actor is always executing at the end of the window. And they're always basically only trading when it's at the cost of the market maker. Then they cut them off, or they give them wider spreads. And so they had to manage this. But it's like an effortful reputation management system. And you just end up having to quote wider and charge larger fees, because it's just the 60 seconds in the long term. But now, if you put your post your prices on chain, you can update them up into the last millisecond. And this affords a market maker the ability to really incorporate fresh prices. It also-- the AMM form factor is interesting. I wouldn't say that it's-- to me, it's not like the biggest driver in why this is interesting, why proper names work well. But it is another facet. AMMs allow this complex preference curve expression where the market maker can indicate what prices they're willing to offer given an amount of demand, which you also see on order books. But it wasn't how RFQs work. RFQs were fixing the price and the amount in advance. Those are the main differences.

**CN:** **Quintus：** 做市商青睐 PropAMM 的另一个关键原因，是相较于他们此前普遍依赖的链上 RFQ（询价系统）具有决定性优势。在 RFQ 系统中，做市商通常必须对报价做出具有法律或经济约束力的承诺，且时效往往长达 60 秒之久。因为该报价必须展示在用户前端，等待用户在钱包中确认点击。虽然这并不完全是 RFQ 机制的原生缺陷，更多是传统交易供应链架构下的产物（基于意图/Intent 的机制会有所不同），但无论如何，做市商过去都不得不报出一个价格并在极长的时间敞口内保持有效。
尽管做市商在 RFQ 中也试图进行订单流细分——例如向某个对手方报价后，发现该对手方总是在 60 秒窗口的最后一刻才发起交易，且每一次出手都恰好发生在对做市商不利的时刻（即外部市场价格已剧烈变动），做市商就不得不切断其白名单连接，或者向其提供极宽的点差。做市商必须花费巨大心力去维护这套声誉管理系统。面对长达 60 秒的陈旧报价风险（即赠予了对手方免费期权），做市商唯一的自我保护手段就是普遍拉宽点差并收取高昂的风险溢价。
但现在，如果你把报价直接发布在链上合约中，你就可以一直更新至出块前的最后一毫秒！这赋予了做市商完全吸收最新鲜市场价格信息的能力。
此外，AMM 的曲线形态本身也非常耐人寻味。对我个人而言，这虽然算不上 PropAMM 奏效的最核心驱动力，但它确实是另一个重要维度：AMM 允许做市商表达一条复杂的偏好函数曲线，能够针对不同规模的交易需求灵活提供不同档位的深层报价，这与中央限价订单簿（CLOB）的深度体验如出一辙。而 RFQ 完全无法做到这一点，RFQ 只能提前锁定离散的单一价格与成交数量。这些就是最本质的差异。

---

### [00:32:04 - 00:32:44]
**EN:** And I think there's interesting implications here for upstream actors, but at least for market makers, that's a layer of the land. And the other nice thing here is that you also get composability with the other pools that are on chain, right? Yes. I mean, this is a funny point because, again, there's no inherent technical reason why market maker liquidity, RFQ liquidity isn't composable with passive LPs in the RFQ model. It was. And you'd see this, like cow swap solvers would often take-- do often take RFQ liquidity and mix it with on chain routes.

**CN:** **主持人：** 这对交易供应链上游的各方参与者有着非常耐人寻味的影响，但至少对做市商而言，这清晰勾勒出了当下的整体竞争版图。而且 PropAMM 还有一个显著优势：你天然获得了与链上其他流动性池的可组合性（composability），对吧？

**Quintus：** 是的。不过这其实是个很有意思的现象，因为从技术底层来看，并没有任何内在原因导致 RFQ 模式下的做市商流动性无法与链上被动 LP 流动性进行可组合路由。实际上它完全可以组合——比如 CoW Swap 的求解器（solvers）就经常调取 RFQ 私人做市流动性，并将其与链上常规 AMM 路径混合在一起完成多跳路由。

---

### [00:32:44 - 00:33:15]
**EN:** But it's not that common. Well, it's not that common for a couple of reasons. One reason is that if you rely only on RFQ liquidity, you can offer firm quotes. And in some systems, users are punished for having variable quotes. In cow swap style systems, there's all variants of forcing the aggregator, the party who's combining the liquidity and offering the quote, to eat the inaccuracy of their quote.

**CN:** **主持人：** 但那种多跳混合路由在实际中并不算普遍。

**Quintus：** 确实不算普遍，原因有几点。首先，如果求解器完全依赖 RFQ 提供的流动性，他们就可以向用户提供绝对确定性的确定报价（firm quotes）。而在某些意图结算系统中，给用户提供浮动报价是会受到惩罚的。在类似 CoW Swap 的机制中，普遍存在某种惩罚规则，强制聚合器或求解器（即整合链上流动性并向用户最终报价的一方）自掏腰包承担报价偏差带来的滑点损失。

---

### [00:33:15 - 00:33:55]
**EN:** If they say, hey, on chain liquidity looks like it's offering this price at 100 bucks, and actually, the price is $5 worse by the time a transaction lands, that's the aggregator's responsibility. I think the other reason was just because-- I think there was just like general inefficiencies in how the systems work that are now being improved. But some of the auctions were just designed in a way where it was very hard to offer the aggregator market maker has to exactly honor the price they offered. And if they fail to do that more than 5% of the time, they will get kicked out of the system. Or 10% of the time, they will get kicked out of the system. And so this was just like disadvantage on chain liquidity.

**CN:** **Quintus：** 如果聚合器此前测算认为：“链上流动性看起来能以 100 美元成交”，结果等交易真正落块上链时，价格恶化了 5 美元，这笔差价必须由聚合器/求解器自行兜底承担。
另一个原因在于此前整个系统运行中存在的普遍低效，不过目前正在逐步改善。当时许多拍卖机制的设计极其严苛，做市商或聚合方必须 100% 严格兑现此前承诺的报价，一旦履约失败的比例超过 5% 或 10%，就会被系统直接踢出白名单。这种严酷的淘汰机制天然在制度上抑制并惩罚了极易波动的链上原生流动性。

---

### [00:33:55 - 00:34:52]
**EN:** They'd also hold these auctions way before the block time. And another inefficiency in these systems is that they will decide what the quote should be based on a top of block simulation. Basically saying, we don't know what's going to happen actually on chain. And so we're going to use this heuristic of assuming nothing changes in the next block before the user transaction lands. And we'll simulate against that state. But that's clearly not how it works. And so a lot of actors are incentivized to quote prices. This is not even specific to RFQs anymore. I'm now just ranting about why these meta-aggregator auctions, some would call them RFQ systems or whatever, why they don't work very well, given that we have a lot of variable passive liquidity on chain. Well, there's these nested implications and I'll pause there before I keep rambling.

**CN:** **Quintus：** 此外，这些意图拍卖往往在出块时间节点之前很久就已经提前举行了。这类系统的另一个低效根源在于：它们完全基于“区块顶部状态模拟（top of block simulation）”来决定向用户报出什么价格。其基本假设是：“由于我们无法预知稍后链上究竟会发生什么，因此只能采用一种启发式逻辑——假设在当前用户交易落块之前，下一个区块的状态不会发生任何改变，并基于该静态进行执行模拟。”但这显然完全脱离了真实的链上动态！因此，许多参与方在报价博弈上的激励被彻底扭曲了。这甚至已经不仅限于 RFQ 本身了，我现在其实是在吐槽：面对链上大量高度波动的被动流动性时，现有的元聚合器拍卖机制（有人称之为 RFQ 系统等等）为何普遍运转不良。总之，这里面包含许多层环环相扣的深层影响，在继续扯远之前我先暂停一下。

---

### [00:34:52 - 00:35:48]
**EN:** One thing I do want to ask you, because you mentioned it right before we went into that little tangent, was what are some of maybe the consequences of prop AMMs on the overall market structure? And some of the actors, how does it maybe change the incentives of the game a little bit? So for instance, you have block builders, they might be getting some feeds from some market makers. And so then people who integrate with that block builder then get the benefits. If you don't integrate with that block builder, or if certain block builders don't have an stream coming from a market maker, then they might be disadvantaged when they're building their blocks. And then you have other downstream consequences, maybe I'm speculating here, but maybe even a particular market maker might want to integrate directly with a validator, right, and sidestep the entire PBS auction based off of certain stake weight that they have access to. Yeah, I'd just be curious, kind of your thoughts, maybe zooming out kind of at the meta level.

**CN:** **主持人：** 在我们展开刚才那个小插曲之前你提到了一个关键点，我很想继续追问：PropAMM 对整个加密市场结构究竟会带来哪些潜在深远后果？它对参与博弈的各方主体带来了怎样的激励机制重塑？
举例来说，现在的区块构建者（block builders）可能会接收特定做市商独家推送的数据流，那么与该构建者集成的参与方就能坐享其成。而如果你没有与该构建者集成，或者某些构建者无法获取做市商的专属报价流，他们在打包构建区块时就会处于严重劣势。
此外还有更深远的下游链式反应——我这里可能纯属推测——甚至某个特定的做市商可能会想要直接与验证者（validators）达成私下协议，利用其掌握的一定质押权重直接绕过整个 PBS（提议者-构建者分离）竞价拍卖。我非常好奇，如果站在宏观全局视角，你对这些问题怎么看？

---

### [00:35:48 - 00:36:33]
**EN:** Yeah, so I think thankfully, at least in Ethereum land, the validator has been left alone. For everyone else, the game changes. So, okay, for market makers, I think we've covered that already. For, well, one difference in market makers we didn't cover is that now speaking, I guess we can speak about both Solana and Ethereum in this. So they now have to consider the infrastructure way more. Market makers were quoting through centralized servers before it was an RFQ server, or they used to trading on clubs and these kind of things. And now you have multiple different block builders or block building systems on Solana, on Ethereum.

**CN:** **Quintus：** 是的。值得庆幸的是，至少在以太坊的世界里，验证者（validators）目前基本没有被卷入或受到冲击。但对于供应链上的其他所有人来说，博弈规则已经彻底被改写了。
对于做市商而言，前面的部分我们已经讨论过了。但做市商还有一个我们尚未涉及的关键变化——在这里我们可以把 Solana 和以太坊放在一起讨论——那就是做市商现在必须对底层基础设施进行极其严肃的考量。过去，做市商主要是通过中心化服务器（例如 RFQ 服务器）对外报价，或者在中心化订单簿（CLOB）撮合交易。而现在，无论在 Solana 还是以太坊上，你面对的都是多个各不相同的区块构建者或构建系统架构。

---

### [00:36:33 - 00:37:38]
**EN:** On Ethereum, they're sort of like coexist in this PBS system and then they take turns in an unpredictable way landing blocks. In Solana, it's more predictable who's doing it, but there's still three things very differently. And so you have to just adapt to the different ways that those transactions get executed. And that means timelines are different, servers are different, geographies is all sorts of things that market makers need to attune their trading setup to. The second thing is, this is actually not a point for market makers because market makers are sort of used to trusting these exchange operators. But now, block builders are in a more explicitly trusted role where small differences in latency can have large costs for market makers. Like a block builder can delay or even... Okay, let's keep it a delay maker updates for a second, five seconds, depending on when they start sending them. That's a long time.

**CN:** **Quintus：** 在以太坊上，各家构建者共存于 PBS 竞价体系中，以一种高度不可预测的交替节奏轮流赢得打包出块权；而在 Solana 上，虽然由谁出块相对更具可预测性，但不同节点的处理逻辑依然大相径庭。因此，做市商必须主动适应这些截然不同的交易执行管道。这意味着时间线不同、服务器不同、地理部署节点不同——做市商必须对其量化做市集群进行全方位的精细调谐。
第二点在于信任机制的转移：虽然做市商在传统领域已经习惯了信任中心化交易所的运营方，但现在，区块构建者（block builders）被推上了一个更加显式的强信任角色：微秒级的延迟差异都会给做市商造成致命的巨额损失！比如，一个区块构建者如果故意恶意延迟做市商的价格更新——哪怕仅仅延迟 1 秒甚至 5 秒（取决于做市商何时开始发送报价）——这在瞬息万变的高频做市中已经是一段极其漫长且致命的敞口了。

---

### [00:37:38 - 00:39:59]
**EN:** Yeah, actually, I sort of don't know if market makers would send quotes at like five seconds before, but anyway, some largest time period. And this ends up meaning that the market maker just gets adversely selected and is profitable for some trader who picks them off or one of their other market makers. And so one implication of this is that the barriers to entry to being a block builder do increase over time. Because if you want to start as this permissionless builder running stuff out of your garage, you will have to compete with basically people who are trying to just exploit market makers who are saying, "Oh, I'm also spending up a block builder. I just want to join this ecosystem. Let me handle your quotes," because then they can also do these kinds of things. Thankfully, people do it once or twice and then you cut them off as a market maker. So maybe it's not so bad. But now there's this additional reputational angle that block builders have to gain to be able to participate in the market. I don't know how substantial that will be compared to just the current order flow barriers. I suspect the order flow barriers will remain dominant. But yeah, it's another consideration for actors higher up in the supply chain beyond block builders. I mean, obviously, the way blocks are built also changes. But I think while it's important to me, I think it's not like the broader implications are not so significant. Now, okay, generally speaking about takers, before we even talk about, you have to sort of segment between these sex facts arbitrageurs and sophisticated takers and retail. And then you have to think about how the user even chooses the quote without even specifying any of that. If you're not trading at the top of the block, or maybe even leaving aside the top of the block. If you're not sure what the maker is going to, how they're going to update their prices, you as a taker are vulnerable to being advertised a good price, deciding your route based on that. And then when you land your transaction, finding if the maker has updated the price they're offering to a much less attractive price, right? Which in traditional finance is called spoofing. And this is commonly seen on Solana. And the problem

**CN:** **Quintus：** 是的，实际上我不太确定做市商是否会提前 5 秒就发出报价，但总之哪怕只延迟了一段稍长的窗口，就意味着做市商会承受极度严重的逆向选择（adverse selection），任由外部抢先交易者或竞争对手做市商以陈旧报价无情吃单收割。
因此，这一现象带来的一个必然结果是：成为一名区块构建者（block builder）的准入门槛正在随时间显著攀升。如果你只是一个想在自家车库里跑一个无许可构建节点的新团队，你将面临极其严峻的竞争；而如果有人恶意建立一个构建节点声称：“嘿，我也启动了一个区块构建器，我也想加入这个生态，把你们的报价流交给我来处理吧”，他们完全有可能在暗中恶意延迟这些报价以趁机收割做市商。所幸的是，这种恶行只要暴露一两次，做市商就会立刻拉黑并切断连接，所以或许情况没有那么糟糕。但这意味着，区块构建者如今必须建立起极其深厚的声誉资本（reputational capital）才能被顶级做市商接纳。我不确定相比于当前既有的独家订单流壁垒，这一声誉门槛究竟有多大影响——我推测独家订单流的垄断壁垒依然会占据主导地位。但这确实是交易供应链上游构建者必须严肃面对的全新考量。
显然，区块的构建方式本身也随之重塑。
接下来谈谈吃单方（takers）。在深入分析前，你必须对吃单者进行分层：中心化与去中心化交易所套利者（CEX-DEX arbitrageurs）、高阶专业吃单者（sophisticated takers），以及普通散户（retail）。撇开这些细分不谈，单看用户究竟是如何做交易决策的：如果你无法在区块最顶部（top of the block）成交，或者姑且把区块顶部抛开不谈，只要你无法确定做市商究竟会如何更新报价，那么作为吃单方的普通用户就极易遭受欺诈——前端聚合器向你展示了一个极具吸引力的优异报价，你据此选择了该条交易路由；但当你的交易最终落块确认时，你却赫然发现做市商已经把报价更新成了一个极其恶劣的价格！这在传统金融中被称为“幌骗 / 虚假报价（spoofing）”。这种现象在 Solana 上屡见不鲜，而真正的问题在于……


---

## Part 3 (00:40:00 - 00:50:50)

# 《Deeply Intents》访谈：PropAMMs 正在吞噬 DeFi（Part 3 对照整理）

**嘉宾：** Quintus（Flashbots 研究员）  
**主题：** PropAMM 恶意幌骗与重定价机制、离散批次拍卖与原生价格发现、TEE 交易隐私保护、闭源智能合约反向工程、以及 DeFi 的密码朋克终局与中心化风险

---

### 章节概要与核心看点
在本次深度对话的第三部分（尾声部分），Quintus 与主持人重点剖析了 PropAMM 生态中的恶性博弈、市场微观结构与终极去中心化愿景：
1. **幌骗（Spoofing）与聚合器囚徒困境：** 深入解析了做市商利用区块边界（如 Flashblocks）或结算延迟动态拉大点差的恶意重定价行为。0x 联合创始人 Will Warren 将其精准定义为“囚徒困境”——聚合器为了最优展示报价往往被动接受劣质成交，最终伤害终端用户。
2. **以太坊链上价格发现（Price Discovery）之争：** 讨论了 PropAMM 是否能使链上现货成为真正的原生基准价格。Quintus 剖析了以太坊 12 秒区块时间所代表的“离散批次拍卖（Batch Auctions）”与币安等高速 CEX 连续交易撮合（Continuous Trading）之间的结构性鸿沟。
3. **交易前隐私与 TEE 信任锚点：** 探讨了预交易隐私（Pre-trade Privacy）为何对非固定价格动态交易至关重要，对比了传统金融中的“阳光交易”（Sunshine Trading），并阐述了 Flashbots 如何借助可信执行环境（TEE）为做市商报价提供防篡改保证。
4. **闭源合约与全自动逆向工程：** 驳斥了做市商试图通过将 PropAMM 链上合约闭源来保持长期竞争优势的“护城河幻想”，指出这不仅会被自动化反编译/LLM 分析秒破，还会严重阻碍求解器（Solvers）与聚合器的离线路由建模。
5. **DeFi 的未来终局——密码朋克还是传统中心化重演：** 警惕 DeFi 交易量向两三家量化巨头集中、以及出块构建者在法兰克福数据中心机房同地托管（Co-location）引发的中心化隐患；呼吁保护被动 LP 并探索超越简单复刻 TradFi 订单簿（CLOBs）的新一代链上原生设计（如暗池与新型高级订单机制）。

---

### [00:40:00 - 00:40:28]
**EN:** For all of the market makers is that once one market maker starts doing this, they can offer very attractive prices, meaning that all of the other market makers don't see any user flow. And so they're all sort of kind of forced into this dynamic. So Wolf Warren calls it a prisoner's dilemma, which I think is a good description. And this also sucks for the user because now they get way worse spreads than they should.  
**CN:** Quintus：对所有做市商而言，核心问题在于一旦某家做市商开始采用这种（幌骗/恶意重定价）手段，他们就能展示出极具吸引力的虚假低价，导致其他合规做市商完全分不到任何散户订单流。因此，其他做市商也被迫卷入这场博弈中。0x 的联合创始人 Will Warren（转录误作 Wolf Warren）将其称为一种“囚徒困境”（Prisoner's Dilemma），我认为这个描述非常贴切。同时这对于终端用户来说也是极其糟糕的体验，因为他们最终成交时的实际点差远比预期的要差得多。

### [00:40:28 - 00:40:53]
**EN:** And so there's a whole suite of solutions solving this. I can talk about some of the incentives for why it's not being solved, which I think Wolf Warren also covers really well. Let me ask you this question. You know, a lot of the sophisticated taking, as you noted earlier, comes from sextex or statistical arbitrage, which is going to bring prices back in line on chain with whatever the price should be, the real price, quote unquote.  
**CN:** Quintus：其实业界已经有一整套解决这个问题的方案。我稍后可以谈谈为什么至今还没被彻底解决背后的激励机制，Will Warren 在文章里也分析得很透彻。

主持人：我想请教你这个问题。正如你之前提到的，目前链上很多高阶吃单（Sophisticated Taking）都来自 CEX-DEX 跨市套利（转录误作 sextex）或统计套利，这些套利者会将链上价格重新拉回到所谓的“真实公允价格”。

### [00:40:53 - 00:41:28]
**EN:** Is there a world where with these prop AMM innovations on Ethereum in particular, that we actually could see ETH price discovery happen on chain against USDC or USDT rather and that be the reference point, rather than it being centralized exchange liquidity? Yeah, it's an interesting one. I think I think a very important structural barrier has been removed in that we now have really advanced takers putting prices on chain and that will, I think, improve volumes.  
**CN:** 主持人：那么是否存在这样一种未来：特别是伴随着以太坊上 PropAMM 的创新，我们能真正看到 ETH 对 USDC 或 USDT 的“原生价格发现”直接发生在链上，并使链上价格成为基准参考价格，而不再完全依赖中心化交易所（CEX）的流动性？

Quintus：这是一个非常有意思的设想。我认为目前一个非常关键的结构性障碍已经被打破了，那就是我们现在有极具实力的专业吃单者与做市机构开始把具有竞争力的价格搬到链上，我认为这确实会推动链上交易量的增长。

### [00:41:28 - 00:42:14]
**EN:** Certainly it'll bring volumes away from the RFQ systems and bring the actual execution on chain. There is a second and very important difference between Binance and other prices coming venues and Ethereum and that's what the block times mean, that effectively there's a weird kind of batch trading. And I think there's a bunch of different academic positions on this, which I don't feel sufficiently familiar with to really lay out, but it's unclear whether batch auctions can be like a price discovery mechanism in their own or if you just end up having to always seed price discovery to a continuous venue, to a faster venue.  
**CN:** Quintus：这无疑会将交易量从链下 RFQ 系统中抽离，促使真正的交易结算与撮合回归链上。但在币安等核心价格发现场所（转录误作 prices coming venues）与以太坊之间，还存在第二个至关重要的差异——那就是区块时间所带来的本质影响：以太坊实际上运行着一种特殊的离散批次交易（Batch Trading）。学术界对此有许多不同的观点与流派，我对此可能还没深入到能面面俱到的程度，但核心争议在于：批次拍卖（Batch Auctions）自身能否独立承担起高效的价格发现功能，还是说它注定只能将价格发现的主导权割让（cede，转录误作 seed）给撮合速度更快的连续交易市场（Continuous Venues）。

### [00:42:14 - 00:42:43]
**EN:** Yeah, I guess I think it makes sense to me that you want to have as much volume as you can on Ethereum, leaving all else equal. Whether you need price discovery on chain is a separate question. I don't know if price discovery necessarily implies the majority of volume or if there's some world in which you can still have the majority of volume happening in this discrete world.  
**CN:** Quintus：在其他条件不变的前提下，尽可能把更多的交易量留存在以太坊链上，这在逻辑上是完全合理的。但链上是否真正需要承担第一性的价格发现功能，则是另一个维度的议题。我不知道成为价格发现的中心是否必然意味着要占据绝大多数交易量，抑或是存在这样一种格局：即使大部分交易量发生在这个基于离散区块的世界里，价格发现依然由链下主导。

### [00:42:43 - 00:43:10]
**EN:** I don't have a strong view on that at all, I haven't thought about it that much. But I guess in the abstract, beyond volume, is there some motivation for price discovery happening on chain? You guys have done a lot of research on pre-trade privacy, post-trade privacy, private execution. You have teas in production today for some confidentiality, for some pre-trade privacy. How does this affect the quotes that users get and/or the price that they execute at?  
**CN:** Quintus：关于这一点我并没有特别强烈的定论，之前没有在这个问题上钻研太深。

主持人：但抽象来看，抛开交易量不谈，链上发生价格发现是否还有其他动机？你们 Flashbots 在交易前隐私（Pre-trade Privacy）、交易后隐私（Post-trade Privacy）以及私密执行方面做了大量研究。你们如今在生产环境中已经通过 TEE（可信执行环境，转录误作 teas）来实现一定程度的机密性与交易前隐私保护。这具体会如何改善用户获得的报价以及最终成交的价格呢？

### [00:43:10 - 00:43:49]
**EN:** Do they get strictly better execution because you get pre-trade privacy, or does that not matter here in this case, as it pertains to prop AMMs? Maybe a naive question. Underlying dynamics that motivated pre-trade privacy in the past are the same. Nothing changes. If a user hasn't received a fixed price and they still have trading as a dynamic price and that price doesn't change, typically revealing their intent to trade is only going to put them at a disadvantage, because be it through a price update or someone else trading ahead of you, there's some way in which the user can be taken advantage of, or the exposed trader.  
**CN:** 主持人：用户是否仅仅因为获得了交易前隐私，就一定能获得严格更优的执行？还是说在 PropAMM 这种情境下，隐私其实无足轻重？这可能问得有点外行。

Quintus：促使我们过去研究交易前隐私的底层驱动力如今依然完全适用，本质并没有改变。如果用户尚未锁定一个固定价格，而是在面对一个动态波动的报价，那么通常来说，过早公开暴露自己的交易意图（Trade Intent）只会让自己陷入劣势——因为无论是做市商顺势更新价格撤单，还是套利者提前抢跑（Front-run），公开暴露意图的交易者都会以某种形式被套利或剥削。

### [00:43:49 - 00:44:11]
**EN:** That's not true in all contexts. You get sunshine trading where people are trying to unload a huge block of stock and they can sort of-- everyone knows that uninformed, they're just trying to get rid of it. By telling people, hey, I'm about to sell a billion dollars of Bitcoin, you get more people to shore up liquidity, you're ready to take the other side of it. But generally speaking, pre-trade privacy is still as valuable as it is before.  
**CN:** Quintus：当然，这并非在所有场景下都绝对成立。传统金融中存在“阳光交易”（Sunshine Trading）的概念，比如某家机构试图出清巨量股票头寸，如果大家明确知道这是无毒的知情度外流动性（Uninformed Flow），仅仅是常规资产再平衡或套现，那么通过提前向全市场广播“嘿，我打算卖出 10 亿美元的比特币”，反而能吸引更多流动性提供者前来挂单，准备承接对手盘。但总体而言，在去中心化交易中，交易前隐私的价值依旧同以往一样至关重要。

### [00:44:11 - 00:44:47]
**EN:** And I think if anything, because of the dependence that market makers have on the central infrastructure providers to not fudge with their trades, having TE-based systems is better. Because it gives you better guarantees that you're not being abused. Tees are insufficient on their own, but they do move the needle. One follow up, I think we kind of danced around it. With Prop AMMs, the market maker oftentimes is not like having a GitHub repo with their open source code where anybody can just go read and fork it.  
**CN:** Quintus：甚至可以说，正因为如今做市商高度依赖中心化的基础设施提供商（如出块构建者）不去恶意篡改、延迟或审查自己的报价交易，基于 TEE（可信执行环境，转录误作 TE-based）的系统反而显得更加关键。因为它为做市商提供了更强的技术保证，确保自己不会被恶意操纵。虽然仅靠 TEE 本身还不足以解决所有信任问题，但它确实带来了实质性的改善。

主持人：我想跟进一个问题，之前我们其实已经稍微触及到了。在 PropAMM 模式下，做市商通常不会在 GitHub 上建立一个完全开源的代码库供任何人随意查看和分叉。

### [00:44:47 - 00:45:22]
**EN:** And yeah, you can obviously decompile code by looking on Jane, right? But do you worry at all about the impact on the open source nature of the Ethereum ecosystem or other ecosystems? No. I think honestly, it's a little bit silly, the closed source nature of these, because so many people have proved that one can consistently reverse engineer them. And really, the benefit of closed source contract now is that there's some time period it takes your competitor to reverse engineer it.  
**CN:** 主持人：当然，任何人都可以通过直接查看链上（on-chain，转录误作 on Jane）字节码来进行反编译。但你是否会担忧这种闭源趋势对以太坊或其他生态系统的开源基因造成负面冲击？

Quintus：不会。坦率地说，我认为将这些合约闭源的做法多少有点幼稚，因为已经有无数事实证明大家完全可以稳定地对其进行逆向工程。当下闭源合约唯一的实际收益，无非就是给竞争对手逆向破解它制造了一点时间差。

### [00:45:22 - 00:45:55]
**EN:** And that gives you an edge, maybe it's a week or two, but I think we're not far away from just having an automated element flow where like someone posted by code and then two hours later or like, I don't know how long it takes. The decompiled code is in the hands of the competitors, and it actually disadvantages the market makers because the way that routers and solvers and aggregated anyone who's trying to trade against these prices in some sophisticated way, route their trades as they build models in Rust or in some other language of how these contracts behave.  
**CN:** Quintus：这种闭源可能带来一两周的短期先发优势，但我认为我们很快就会迎来全自动化的逆向分析工作流（如通过大模型与自动化工具链）——某家机构刚把字节码（bytecode，转录误作 by code）部署到链上，可能两小时后反编译好的完整逻辑就已经摆在竞争对手案头了。而且闭源实际上反噬了做市商自己：因为路由器、求解器（Solvers）、聚合器以及任何希望通过高阶算法对接这些报价的参与方，在进行交易路由时，都需要在 Rust 或其他编程语言中为这些合约的行为建立高精度的执行与定价模型。

### [00:45:55 - 00:46:23]
**EN:** And without the contract being public, it's much harder for them to do that. And so the only way of defining good routes is to like simulate against this buy code and see what it does at different prices. But that's very frustrating, you know, it just slows things down. So I hope that we get to a equilibrium where market makers publish the open contracts. And honestly, from my understanding, the contracts themselves are not where most of the innovation is.  
**CN:** Quintus：如果合约源码不公开，求解器和路由协议要建模就会困难得多。寻找最优路由的唯一途径就变成了针对链上字节码（bytecode，转录误作 buy code）进行暴力仿真（Simulation），测试在不同价格输入下它的反应。但这种方式极其令人沮丧且会严重拖慢路由性能。因此我希望未来行业能达成一种新均衡——做市商主动公开其合约代码。而且说实话，据我所知，合约代码本身根本就不是 PropAMM 真正核心技术创新的所在。

### [00:46:23 - 00:46:56]
**EN:** Like in land, there's still some juice to squeeze out of them. But in Solana, there's not too much innovation adding doubt, you know, pricing and latency games and these kind of things. And I think I kind of hope that we come to view these on-chain contract as like order types, basically, order types that also encompass flow segmentation logic. I will also just, this is not related just before we completely move on to say that there are people who have good technical solutions for spoofing.  
**CN:** Quintus：可能在以太坊 EVM 生态中，合约逻辑层面还有一些可挖掘的潜力。但在 Solana 上，链上合约端并没有太多的创新空间，核心优势完全在于链下的自主定价策略和低延迟速度博弈。我其实希望大家未来将这些链上合约单纯视为一种新型的“高级订单类型”（Order Types）——一种原生内嵌了订单流分流/细分（Flow Segmentation）逻辑的订单形式。另外，在话题翻篇前我想顺便补充一句：针对前面提到的幌骗（Spoofing）问题，其实业内已经有团队提出了非常出色的技术解决方案。

### [00:46:56 - 00:47:23]
**EN:** We don't have to go there, but I just didn't want to let the conversation go public with people thinking this is like dire spoofing problem. Is there anything else that you want to say about prop AMMs and kind of like your research and where you see the space maybe going in the next six to 12 months in Ethereum land? I think that there are two things to say. One is the DeFi framing and one is the more cypherpunk-y framing.  
**CN:** Quintus：我们今天不必在这上面过分展开，但我只是不想让播客发布后让听众误以为这是一个无解的恶性幌骗绝境。

主持人：关于 PropAMM、你目前的研究，以及未来 6 到 12 个月以太坊生态在这方面的发展趋势，你还有什么想补充的吗？

Quintus：我认为主要有两个视角需要阐述：一个是 DeFi 实用主义视角，另一个则是更具密码朋克（Cypherpunk）色彩的意识形态视角。

### [00:47:23 - 00:47:52]
**EN:** I think in the DeFi framing, prop AMMs are an important innovation, or like maybe not even DeFi finance for everything important, like trading innovation on blockchains. And it's good to push the envelope. And there's a lot that meets in Ethereum because there's so many actors involved in the supply chain. It's going to take some time for people to adapt to this new paradigm. Definitely. So it's important that we adapt to it, and I think we should generally be excited about it.  
**CN:** Quintus：从 DeFi 的视角来看，PropAMM 是一项非常重要的突破；甚至不仅仅是 DeFi，它是整个区块链链上交易领域的一大关键创新，不断拓宽边界绝对是一件好事。在以太坊上这尤为复杂，因为交易供应链（Transaction Supply Chain）中牵涉到了太多相互博弈的角色。各方需要一定的时间来适应这种全新范式。我们积极适应这一范式至关重要，且总体上我们应当对此感到兴奋。

### [00:47:52 - 00:48:16]
**EN:** My question is, what's next? I think in the Ethereum community, prop AMMs are this like super normal thing. In Solana land, they're going to run for a while. People are launching PerpDexes and moving on to different kinds of venues. And I think like dark pools are another very interesting kind of thing that needs to be explored. We have to keep innovating. The rate at which we were like tweaking AMMs was too slow.  
**CN:** Quintus：但我真正思考的问题是：下一步是什么？在以太坊社区，PropAMM 正在逐渐成为常态；在 Solana 生态，这一模式还会持续演进一段时间。同时，人们正在将其推广到链上永续合约 DEX（Perp DEXes），并向不同类型的流动性场所迁移。此外我认为暗池（Dark Pools）也是另一个亟需深入探索的极具前景的方向。我们必须持续创新——过去我们只是在传统 AMM 曲线参数上微调修补，那种演进节奏实在太慢了。

### [00:48:16 - 00:48:46]
**EN:** And our prop AMMs have come along and we need like, there's a lot of tradify looking things out there. And I think if we don't innovate on the on-chain stuff enough, we're going to start just having the same venues, which maybe is, you know, we have clubs on-chain and that ends up being the optimal design. But I think they might be more interesting things and we need to keep pushing the envelope before everything collapses to the traditional equinibrium, assuming there exists more interesting better equilibriums.  
**CN:** Quintus：随着 PropAMM 的兴起，如今市场上涌现了许多越来越像传统金融（TradFi）的机制设计。如果我们不在链上原生机制上进行足够深刻的创新，链上最终很可能只会沦为对 TradFi 场所的简单复刻——也许最终大家全都在链上搞中央限价订单簿（CLOBs，转录误作 clubs），并且那被证明就是最优设计。但我坚信一定存在更具想象力、更有趣的架构。在一切最终不可逆地坍缩回传统金融均衡（equilibrium）状态之前，我们必须不断突破极限去探索，前提是链上确实存在更好、更有趣的新均衡。

### [00:48:46 - 00:49:16]
**EN:** Equilibria. And the type of thing to say is prop AMMs are great for Ethereum because they allow much better trading without compromising on the block time. And even if you were excited about lowering the block time to six seconds, that's still super long. But at the same time, they force us to ask this question, what happens to passive LPs? Is it a problem if we don't have decentralized market makers anymore? How can we preserve them? Okay. Different asset types.  
**CN:** Quintus：这种更优的均衡状态。另一方面需要强调的是：PropAMM 对以太坊来说非常契合，因为它在无需妥协以太坊现有区块时间的前提下，实现了质量高得多的交易执行。即便有人热衷于将以太坊出块时间缩短至 6 秒，在做市高频交易视角下 6 秒依然太漫长了。但与此同时，PropAMM 也逼迫我们面对一个尖锐的问题：被动流动性提供者（Passive LPs）该何去何从？如果我们不再拥有去中心化的普通做市群体，这会是一个严重的隐患吗？我们又该如何保护并留存他们？当然，不同资产类别的情况不可一概而论。

### [00:49:16 - 00:49:38]
**EN:** Also, there's different conversation. But what do we lose in DeFi if a lot of the volume is being facilitated by like three big firms with a bunch of quants? And then finally, I still think we should decentralize them. We're trying to decentralize the builder, you know, there's a lot of practical considerations at all that prevent that from happening and that are probably the bigger focus at the moment.  
**CN:** Quintus：不同资产需要另当别论。但如果整个 DeFi 的大部分交易量最终全由两三家坐拥顶尖宽客（Quant）的传统量化做市巨头所垄断，DeFi 究竟会丧失什么？最后，我依然坚信我们应当推动其去中心化。目前行业也在努力推进区块构建者去中心化（Decentralizing the Builder），虽然现实中存在许多极其繁重的落地阻碍导致其进展缓慢，而这也正是目前更受关注的焦点。

### [00:49:38 - 00:50:09]
**EN:** But we ideally don't want to have these trading setups like really just if all of the blocks in Ethereum are built in like one data center and Frank Ferdinand was co located around that, and there's a latency optimize, and that kind of loses a lot of the benefits of Ethereum. And so we want to make sure that, you know, the market structure improvements also are more compatible with something different from a big cluster in a data center somewhere.  
**CN:** Quintus：但从理想主义角度看，我们绝不希望链上交易演变成这样的格局：如果以太坊所有的区块都在同一个数据中心里被构建，所有做市商与套利者全都在法兰克福（Frankfurt，转录误作 Frank Ferdinand）同一个机房里进行物理机柜托管（Co-location），全网演变为纯粹极致的低延迟速度竞争——那以太坊引以为傲的去中心化与抗审查优势将荡然无存。因此，我们必须确保市场微观结构的演进，能够兼容某种超越“中心化数据中心大集群托管”的全新去中心化形态。

### [00:50:09 - 00:50:30]
**EN:** Hell yeah, I think that was quite eloquent. And yeah, we could do a whole nother episode on builder centralization. That's a very dense, meaty topic. Now, this has been really educational with respect to prop AMMs, and I think this will be a really good starting point for folks to listen and also folks can check out some other episodes that I have coming up on this topic as well.  
**CN:** 主持人：太精辟了，总结得非常深刻有力。确实，关于“区块构建者中心化”（Builder Centralization）我们完全可以再单独录一整期播客，那绝对是一个极其硬核、信息量巨大的大题目。今天这期关于 PropAMM 的讨论极具启发性和知识密度，我相信对于广大听众而言是一个极佳的认知切入点，大家也可以关注我接下来关于该话题的其他系列节目。

### [00:50:30 - 00:50:50]
**EN:** Grantis, if folks want to get in touch with you, talk about prop AMMs, or anything else that you're working on, what's the best way to reach out to you? Probably just Twitter, I'm a 0x contest on Twitter. Awesome. Well, it was a privilege to have you on today. Thank you for coming on, and we'll talk to you soon. Cheers. Good to catch up, man.  
**CN:** 主持人：Quintus（转录误作 Grantis），如果大家想联系你交流 PropAMM 或探讨你正在从事的其他研究，最佳的联络方式是什么？

Quintus：大概就是 Twitter（X）吧，我的推特账号是 @0xQuintus（转录误作 0x contest）。

主持人：太棒了。今天非常荣幸能邀请到你，感谢你的精彩分享，我们下次再聊！

Quintus：非常感谢，能和你畅聊真痛快！
