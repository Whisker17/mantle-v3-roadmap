# Deeply Intents Ep. 47: propAMMs won, you just didn't notice - Katia Banina
## 全集双语对照转录全文 (Full Bilingual Transcript)

- **播客节目**：Deeply Intents (Ep. 47)
- **嘉宾**：Katia Banina（Bebop 联合创始人兼 CEO）
- **音频时长**：58 分 20 秒
- **音频文件**：
- **核心主题**：以太坊市场微观结构演进、专有做市 AMM（PropAMM）、区块预言机定价机制（BopAMM）、RFQ 与 RFS 的对比、LVR 与 MEV 治理、可交易链上预言机、做市商网络低延迟架构、金库可组合性（Vault Composability）与 DeFi 终局。

---

## 目录 (Table of Contents)
1. [Part 1 (00:00:00 - 00:20:00) 专有做市 AMM（PropAMM）的崛起与以太坊 BopAMM 的诞生](#part-1-000000---002000)
2. [Part 2 (00:20:00 - 00:40:00) 链下 RFQ 与链上可交易预言机（Transactable Oracle）的深度博弈](#part-2-002000---004000)
3. [Part 3 (00:40:00 - 00:58:20) 延迟优化、L2 排序器、OpSec 防御与资金乐高（Money Legos）的可组合性终局](#part-3-004000---005820)

---


## Part 1 (00:00:00 - 00:20:00)

# Deeply Intents Ep. 47 - PropAMMs Won, You Just Didn't Notice (Part 1)

**嘉宾：** Katia Banina（Bebop CEO）  
**主题：** 专有做市 AMM（PropAMM）的崛起与以太坊 BopAMM 的诞生  
**概要：** 本期播客探讨了 DeFi 市场微观结构与流动性执行的重大演进。Bebop 创始人兼 CEO Katia Banina 介绍了受 Solana 专有做市 AMM（propAMM）启发而推出的全新产品 BopAMM。对话深入分析了询价机制（RFQ）与报价流机制（RFS）在链上与链下的异同、以太坊 12 秒出块环境对做市商防范套利与毒性订单流（LVR）的挑战，以及如何通过与区块构建者（Block Builders，如 Titan、Flashbots BuilderNet）链下协同，将预言机价格更新置于块首（Top of Block），从而在以太坊主网为大额现货交易提供机构级的执行质量。

---

### [00:00:00 - 00:00:24]
**EN:** Katia, good afternoon, welcome back to the Deeply Intents podcast. It's great to have you on today. Oh, great. Great to be back. And what did I learn about you today? Can you start with this? Sure. What did you learn about me today? What's- I learned about you today that you have eight cats and yeah, you told me all about kind of how it happened and all, but yeah, that's, that's something I did not know about you and probably very few people do know.
**CN:** 主持人：Katia，下午好，欢迎回到《Deeply Intents》（深入意图）播客，非常高兴今天能邀请到你。  
Katia：很高兴再次回来。  
主持人：我今天了解到了关于你的什么事？你能先从这个聊起吗？  
Katia：当然可以，你今天了解到我什么了？  
主持人：我今天才知道你养了八只猫！对，你刚才还跟我讲了整件事是怎么来的等等。不过这确实是我之前不知道的，估计也很少有人知道。

### [00:00:24 - 00:00:48]
**EN:** So yeah, let's share it with the world. Uh, maybe just one fun cat story for you, for me to kick, kick this off with. One fun cat story. That's a good question. Sure. Um, I remember one. So my first cat, his name's Kawhi. I actually named him after the basketball player Kawhi Lettered. And the reason I did was I was actually doing basketball scouting at that time. So there was like a more direct lake.
**CN:** 主持人：所以向大家分享一下吧。要不你先讲一个有趣的猫咪故事来开个场？  
Katia：一个有趣的猫咪故事？问得好。当然，我记得有一个。我的第一只猫叫 Kawhi，我其实是用篮球球星科怀·伦纳德（Kawhi Leonard）的名字给他命名的。之所以这么起名，是因为我当时正好在做篮球球探工作，所以算是有个比较直接的渊源。

### [00:00:48 - 00:01:18]
**EN:** But anyway, he escaped one night. I had like a screen porch at that house I was living in. And, uh, the door to the screen porch, I guess we left open. He escaped, he left for maybe like two or three weeks and was looking for him all over the neighborhood. And at one point I was just like, I guess he's not coming back one night at maybe like 4AM, I hear this like meow and knock at the glass door that came leads out to the screen porch and it was him and he came back and it was really beautiful.
**CN:** 但不管怎样，有一晚他偷跑出去了。当时我住的房子有个带纱窗的门廊，我想我们大概是把门廊的门敞着了。他就溜跑了，整整失踪了大概两三周。我们在整个街区到处找他。一度我都觉得他肯定回不来了。直到有一天夜里大概凌晨 4 点，我听到了一声猫叫，还有抓挠敲打通向门廊玻璃门的声音——居然真的是他！他自己找回来了，那一刻真的太美好了。

### [00:01:18 - 00:01:48]
**EN:** He, uh, actually wound up going on. Like he had, um, various different issues and had surgery after that. So he had a little bit of dicey early part of his life, but since then he's, uh, been totally fine and healthy and been a great cat and a great companion and friend, so one of eight, one of eight. Well, uh, that's cool. And next, next time, next podcast, you start with like, I do have like a cat named after blockchain or something, but clearly, yes, no comment.
**CN:** 后来他其实经历了不少波折，比如患上过各种病症，后来还动了手术。所以他早年的猫生有点坎坷，但自那之后他就一直非常健康，是一只特别棒的猫，也是极好的伙伴和挚友——这就是八只猫中的第一只。  
主持人：哈哈，太酷了。下次播客开场你可得讲讲有没有哪只猫是用区块链命名的了，不过显然，你现在对此“不予置评”。

### [00:01:48 - 00:02:11]
**EN:** I think last time we were on, we talked a lot about intense. We talked about RFQs. We talked about solving. Different things going on with Bebop, but recently you guys put out an announcement of this new product called Bop AMM and there's a lot of really cool innovations surrounding it. I think you would argue that it's not that innovative, but I think it is. And we can talk about why.
**CN:** 主持人：我记得上次我们做节目时，聊了很多关于意图（Intents）、询价机制（RFQ）以及意图求解（Solving）的话题，探讨了 Bebop 的各种动态。但最近你们发布了一款名为 BopAMM（区块预言机定价 AMM）的新产品，围绕它有许多非常酷的创新。我想你可能会谦虚地认为它没那么颠覆，但我认为它确实极具创新性，我们可以深入探讨一下背后的原因。

### [00:02:11 - 00:02:35]
**EN:** And yeah, like I just give you the opportunity to tell us about Bop AMM. What is it and what do you guys aim into achieve? And then maybe we can unpack further. Bop AMM is a name, I suppose. So the product Bop AMM has been inspired by prop AMMs. And so for Bebop, Bop AMM sounded like a very cool, cool name to give to it. You know, our original home is on Ethereum.
**CN:** 主持人：对，我想借此机会先请你向大家介绍一下 BopAMM。它到底是什么？你们希望通过它实现什么目标？随后我们可以进一步展开拆解。  
Katia：BopAMM 首先是一个命名吧。这款产品很大程度上是受到了专有做市 AMM（propAMM，Proprietary AMM）的启发，因此对 Bebop 来说，BopAMM 听起来是一个非常贴切且酷炫的名字。你知道的，我们最初的大本营一直在以太坊上。

### [00:02:35 - 00:03:11]
**EN:** Most of our volume is on Ethereum. And generally we believe in Ethereum and like it. And Ethereum is the chain that has the largest amount of capital and capital needs good quality of execution. And especially as we see more and more of the institutional capital and just generally people who understand what those words mean, quality of execution, the more it's required. And however, the biggest or most recent innovation in the, in terms of quality of execution, we actually saw on Solana, which makes a lot of sense.
**CN:** 我们绝大部分的交易量都在以太坊上。总体而言，我们坚信以太坊的价值并深爱着它。以太坊是汇聚了最多资本的公链，而资本天生需要极高的执行质量（quality of execution）。尤其是当我们看到越来越多的机构资本，以及越来越多真正懂得“执行质量”内涵的用户涌入时，对高水准执行质量的需求就变得更加迫切。然而，近期在执行质量方面最大、最新的创新，其实率先发生在 Solana 上，这也非常合乎逻辑。

### [00:03:11 - 00:03:30]
**EN:** It's a fast chain. It's quite driven by purely financial use cases and things like that. Right. So it's, it kind of was a natural birth home to it. And we thought, well, is it possible to do in this situation where it's like a 12 second block environment? So we thought about it a little bit. We weren't the only ones thinking about it clearly.
**CN:** Solana 是一条高速链，主要受纯粹的金融用例驱动，因此它自然而然地成为了此类创新的天然温床。于是我们思考：在以太坊这种 12 秒出块时间的环境下，是否也能实现类似的效果？我们对此进行了深入探索，显然，我们并不是唯一在思考这个方向的人。

### [00:03:30 - 00:04:13]
**EN:** And I think we're going to talk about block builders and some others as well. Yeah. And so that's kind of, um, and we found the approach and the solution to that. Whereas prop AMMs are usually operated by, well, fully operated by the individual trading firms. We saw a role where people can come in and create a level of coordination layer for the market makers from one side to block builders on the other side and ultimately create not just good execution quality, but the kind of ensuring top level reliability for all the consumers downstream.
**CN:** 我想我们稍后也会聊到区块构建者（Block Builders）以及其他生态角色。是的，我们找到了解决这一难题的路径与方案。虽然专有做市 AMM 通常完全由单个量化做市机构独立运作，但我们发现了一个极具价值的生态位：可以有人出面搭建一个协调层，一端连接做市商，另一端连接区块构建者，最终不仅能实现卓越的执行质量，更能为下游的所有消费者提供最高等级的可靠性保障。

### [00:04:13 - 00:04:40]
**EN:** So that's anyone it's aggregators is direct users. It's, uh, wallets, whoever that is. That's what both payment is. It's in private beta. Currently a lot of things need to be done to fully productize it. But we, you know, we're definitely there. I'm happy to talk about, of course, like what, how prop AMMs themselves operate and sort of like what's inside there. But yeah, for those of you who do know prop AMMs, both AMM is the theorem version inspired by those primitives.
**CN:** 下游消费者包括聚合器、终端用户、钱包等任何需要流动性的参与方。这就是 BopAMM。目前它正处于内测（private beta）阶段，要实现完全的产品化还有很多工程要做，但我们确实已经走通了核心链路。我很乐意详细聊聊专有做市 AMM 本身是如何运作的、其内部机制细节是怎样的。但总的来说，对于了解 propAMM 的人而言，BopAMM 就是受这些金融原语启发而诞生的以太坊版本。

### [00:04:40 - 00:05:08]
**EN:** Beautiful. We'll talk about the Ethereum centric version in a second. The first thing I just want to ask you is maybe what did you see on Solana that was interesting? Like as I understand prop AMMs market maker has contracts that they can update in real time with their own custom pricing curve, so they can then go and integrate with a Jupiter or another aggregator and provide users maybe a better fill opportunity than what somebody else is quoting.
**CN:** 主持人：太棒了。我们马上来聊以太坊这个版本。首先我想先请教，你在 Solana 上看到的亮点究竟是什么？据我理解，在专有做市 AMM（propAMM）中，做市商部署了专属智能合约，他们可以用自己的自定义定价曲线实时更新这些合约，进而无缝集成到 Jupiter 或其他 DEX 聚合器中，为用户提供优于其他人报价的成交机会（fill opportunity）。

### [00:05:08 - 00:05:29]
**EN:** And this works really nicely because Solana has very fast blocks and low latency and it's also cheap to do state rights on Solana. And we saw, I think kind of maybe an explosion during Q4 of this trend last year start to happen. What was it that got you, like that popped out to you, I guess, that you said, like, this is an interesting innovation. We should start considering it.
**CN:** 主持人：这种机制在 Solana 上运转得极其顺畅，因为 Solana 出块极快、延迟极低，而且链上状态写入（state writes）成本非常便宜。我们看到去年第四季度这种趋势呈现出爆炸式增长。究竟是什么触动了你、让你眼前一亮，促使你认为“这是一项极具吸引力的创新，我们必须开始跟进研究”？

### [00:05:29 - 00:05:52]
**EN:** Was it like any kind of research that your team was doing? Was it just like looking for maybe a way to give, as you noted, different types of takers, better fills? What was the impetus to drive you to look into this direction? Well, first of all, charts and market shares and things like that. In general, I would say is the first thing that makes you notice. And then that's on the one hand, right?
**CN:** 主持人：是你们团队内部开展的某些专题研究？还是正如你所言，你们一直在探索为不同类型的吃单者（Takers）提供更佳成交价格的途径？推动你们深入该方向的核心驱动力是什么？  
Katia：嗯，首先最直观的当属各种数据图表和市场份额的变化。通常而言，这是最先抓住你眼球的信号，这是其中一方面，对吧？

### [00:05:52 - 00:06:25]
**EN:** So basically seeing what's going on, you know, reading what other people write and say and internalizing it. And the second part for me personally was really translating this into, into kind of, well, try to find for a forum, you might say, well, generally kind of in a forum that makes sense in global transactions, right? So I think by now everyone knows what RFQ is, plenty of various RFQ protocols and platforms on chain and at least Bebop.
**CN:** 也就是敏锐观察市场动态，阅读业界的深度文章与见解并将其内化。对我个人而言，第二部分是真正将其映射到一种在全球宏观金融交易中都能合理解释的形态。我想时至今日大家都非常清楚什么是 RFQ（询价机制）了，链上有许多不同的 RFQ 协议与平台，Bebop 便是其中之一。

### [00:06:25 - 00:06:55]
**EN:** So I think we've covered this pretty well, but I suspect few people have heard in crypto, have heard about RFS and you're shaking your head. Okay. So let me tell you what that is. RFS stands it's kind of like a celebration stands for request for stream. And what it means in practice, right? Is that instead of requesting for a quote that lives for that is fixed and lives for a certain period of time, you receive a continuous stream via WebSocket or something like that, right?
**CN:** 所以我觉得关于 RFQ 我们已经讨论得相当充分了，但我怀疑加密圈里很少有人听说过 RFS——我看你在摇头对吧？那我来告诉你这到底是什么。RFS 是一个缩写，代表报价流机制（Request for Stream）。在实际运作中，它的含义是：与请求一份在特定窗口期内锁定的固定报价不同，你是通过 WebSocket 等通道接收持续不断的价格流，对吧？

### [00:06:55 - 00:07:22]
**EN:** So you get constantly updating prices and then you basically place an order against those prices, right? And so how are they different? Well, an RFQ is a fixed price for a certain period of time. And so at this point, the market maker basically is on the hook to execute at the price they committed to and while RFS kind of much more up to date. And so whatever the latest price arrives, right?
**CN:** 这样你就能获取实时跳动更新的价格，然后你实际上是基于这些最新跳动的价格直接下单成交，对吧？那么两者的本质区别何在？RFQ 是在一段时间内锁定固定价格，此时做市商必须承担兑现其承诺报价的刚性履约义务（on the hook）；而 RFS 则要实时鲜活得多。因此无论最新推过来的价格是多少……

### [00:07:22 - 00:08:00]
**EN:** At the time of the order being placed, it executes at that same point. So now let's think about how market makers can today transact on chain, right? So there are with their own capital and with their own pricing, right? So one of them is RFQ, obviously. So they quote a price, they send the assigned call data that can then be submitted on chain by someone. Those quotes usually live for anything between on Bebop, for example, from five to 75 seconds, depending on how they're going to be used and use case there, but obviously within the context of DeFi, some quotes and you know, wallets and all that kind of stuff, right?
**CN:** 在订单下达的那一刹那，交易就按当时那个最新时间点的价格即时执行。现在让我们思考一下，做市商如今在链上利用自有资本和自主定价模型是如何交易的：显然，最主要的一种方式就是 RFQ。他们给出一个报价，发送一段带有签名的调用数据（signed calldata），随后由用户或结算方提交到链上。以 Bebop 为例，这些报价的有效期通常在 5 秒到 75 秒之间，取决于具体的业务场景。但在 DeFi 环境下，涉及到移动端钱包等诸多复杂交互……

### [00:08:00 - 00:08:30]
**EN:** The users sometimes need all this time to, you know, go and sign a transaction on the hardware wallet or whatnot, right? So those quotes can't be expiring very quickly, but for comparison, something like an RFQ in effects in the off-chain effects market lives for five seconds, right? And so we're comparing five seconds to 75 seconds, you know, those, those quotes get picked off and so you then start have to battling toxic flow and you know, all this kind of stuff that comes with it.
**CN:** 用户有时确实需要预留这么长的时间，去硬件钱包上翻页确认签名等等。因此这些链上报价不能失效得过快。但作为对比，传统链下外汇（FX）市场中的 RFQ 报价有效期通常只有短短 5 秒钟！试想一下，从 5 秒拉长到 75 秒，这些过时的报价极容易被外部套利者精准收割（picked off），从而迫使做市商不得不投入巨大成本去对抗毒性订单流（toxic flow）以及随之而来的一切逆向选择损失。

### [00:08:31 - 00:09:03]
**EN:** That's how RFQ works in general and how it works on chain. Now you can't quite stream anything like a price or something on chain unless you basically publish it in a transaction, right? And so this publishing of what we describe as pricing curves on chain is exactly that, right? You are basically streaming your updates and you're making sure that when the transaction happens in that block, whoever trades against you trades at your most up-to-date price.
**CN:** 这就是 RFQ 的通用逻辑以及它在链上的运转机制。然而在区块链上，你无法像在链下那样直接推送连续的数据流，除非你以交易的形式将价格发布上链，对吧？因此，我们所说的把定价曲线发布到链上，本质上正是如此：你实际上是通过高频交易流的方式将价格更新流式推送到链上，确保当该区块内发生撮合交易时，任何与你对冲交易的用户都在基于你最新鲜的价格成交。

### [00:09:03 - 00:09:30]
**EN:** And so from that perspective, this is exactly what it is. This is RFS and this request for stream and basically market makers are streaming their prices onto chain and that's what it is. So again, right? Like getting it, getting this to work on chain is what's interesting. But the principle itself is really not new at all. Coming back to your question about like, why did you think it was interesting? Right?
**CN:** 所以从这个角度来看，它的本质正是如此：这就是 RFS，即报价流机制（Request for Stream）。做市商本质上是在把他们的价格流式推送到链上。再次强调，让这套机制在区块链的约束下真正跑通才是最具工程挑战和魅力的地方，但这一金融原理本身在传统市场中早已有之，算不上什么新鲜事物。回到你刚才的问题：为什么我觉得它很有吸引力？

### [00:09:30 - 00:09:54]
**EN:** So like one thing, okay, it grasped our attention. The devs were excited to do this, right? But for me, it was this reflection. It was like, okay, why does it make sense? Because of that, because it exists and because it works. And then the difference is, and by the way, it doesn't mean that RFQ is bad, right? It means that like those things complement each other quite well and they have their place in the world in the global finance.
**CN:** 一方面，它确实迅速吸引了我们的注意，开发团队对构建这一系统感到极度兴奋；但对我个人而言，更多的是一种底层逻辑的审视：它为什么合理？正是因为在成熟金融体系中它早已存在并且被证明行之有效。顺便提一句，这绝不意味着 RFQ 不好，而是说这两种机制具备极强的互补性，它们在全球金融市场中各自拥有不可替代的生态位。

### [00:09:54 - 00:10:15]
**EN:** And that's why they will also have their place in the on-chain finance. And so yeah, for us, it was a very natural thing to do. So that's why we did it. It's funny because like in doing prep for the episode and just looking into prop AMMs, I didn't come across RFS. I did come across like a lot of binding quotes literature in the FX markets, but I actually missed RFS.
**CN:** Katia：这就是为什么它们在链上金融体系中也必然各自占据核心一席。因此对我们而言，顺应这一规律是极其自然的选择，这也是我们全力推进它的原因。  
主持人：非常有意思，因为在准备这期播客和调研专有做市 AMM（propAMM）时，我居然没有读到 RFS 这个概念。我确实研读了许多外汇市场中关于具有约束力的强制报价（binding quotes）的文献，但确实遗漏了 RFS。

### [00:10:15 - 00:10:41]
**EN:** So that's really interesting and it actually makes a lot more sense than as to why chain like Solana would be the first to adopt something like this or like a mega-eath, something that basically just has streaming blocks. Okay, so we have RFQs, binding commitment for the market maker. We have RFS, which is streaming quoted updates. One of the things that you noted, I think recently before the show, also even on Twitter, was that you learned a lot about block building recently.
**CN:** 主持人：这确实非常引人入胜，而且它也合理解释了为什么像 Solana 或是像 MegaETH 这种具备流式区块（streaming blocks）机制的高吞吐量链会率先采纳这种模式。好的，现在梳理清楚了：我们有作为做市商刚性履约承诺的 RFQ，也有流式实时报价更新的 RFS。而你在节目前以及最近在 Twitter 上也多次提到，你近期深入学习了大量关于区块构建（block building）的知识。

### [00:10:41 - 00:11:09]
**EN:** And we just talked about how Ethereum has 12 seconds. What component here is the block builder playing with the prop AMM system on Ethereum mainnet? Block building within the context of Ethereum, within the context of 12 second blocks, but also, you know, with the Ethereum roadmap, shortening the time of the block, you know, as relevant within the context of 10 second block, eight second block, six second block, right? Like it's, Ironence talks about micros, right?
**CN:** 主持人：我们刚才提到以太坊有 12 秒的出块时间。在以太坊主网上的专有做市 AMM 体系中，区块构建者（Block Builder）究竟扮演了什么关键角色？  
Katia：区块构建在以太坊 12 秒出块时间的约束下显得尤为关键。当然伴随以太坊路线图未来可能缩短出块时间，无论是在 10 秒、8 秒还是 6 秒出块的背景下也同样具有举足轻重的意义。你知道，传统金融/量化高频交易谈论的都是微秒（microseconds）级别……

### [00:11:09 - 00:11:38]
**EN:** So, and we're talking here in seconds. Like this is just completely. Orders of magnitude. It is orders of magnitude, exactly right. Multiple orders of magnitude as well, right? So anyway, within this context, super relevant. And within the context of proposable, the separation, just generally how Ethereum works, there's no dedicated fast lane for certain types of updates. There is no central sequencer. There's, you know, there's this market, right? That exists for blocks.
**CN:** Katia：而我们在以太坊上讨论的却是以“秒”计的时间单位，这完全是……  
主持人：差了几个数量级。  
Katia：没错，正是整整差了几个数量级，甚至可以说是差了好几个数量级！因此在这个背景下，区块构建极具相关性。而且在提议者-构建者分离（PBS, Proposer-Builder Separation）以及以太坊目前的底层架构下，既没有专门针对特定类型状态更新的“特权快速通道（fast lane）”，也没有中心化排序器（central sequencer），而是存在着一个由市场化竞价决定的区块拍卖市场。

### [00:11:38 - 00:12:02]
**EN:** And there's a level of supply chain, you know, of how a user's transaction actually, how it goes through different phases to ultimately get into a block that's validated. Going back to fast chains, like Solana, as a market maker, you want to trade on the latest price, on your latest price, the freshest price. You don't want to get picked off, right? You don't want to be front run.
**CN:** 这里存在着一条精密的 MEV 交易供应链：用户的交易必须经历不同阶段的路由与排序，最终才能被打包进一个完成验证的区块中。回到像 Solana 这样的高速链：作为做市商，你永远希望按照属于你自己的、绝对最新鲜的实时价格来成交。你绝不希望被套利者狙击收割（picked off），也绝不希望被抢跑（front run）。

### [00:12:02 - 00:12:29]
**EN:** And so from that respect, you want your Oracle update to be as high as possible towards the beginning of the block, top of the block, so that all the transactions listen to that price. Because as a market maker, as soon as you get that new information, you want to be able to quote that new information. It's a stream, right? And so you want your stream to arrive, to be there at the point of execution, right?
**CN:** 从这个角度来看，你必须确保你的预言机价格更新在区块内排得越靠前越好，也就是必须占据块首（top of the block），这样区块后续的所有交易都会采用该最新价格。因为做市商一旦接收到外部市场的最新信息，就必须立刻基于新信息重新报价。这是一个连续的数据流，你必须确保你的价格流在订单被撮合执行的当口已经准确就位，对吧？

### [00:12:29 - 00:12:57]
**EN:** And so not after. But because Solana blocks are so fast, again, still a lot of microseconds clearly, but because they're fast, you know, you still can be okay trading on the previous block price, right? So even if your Oracle update somehow didn't get included or like something happened, it is still okay. You can still price a quote quite tightly. Now, 12 seconds price is pretty old, right?
**CN:** 绝不能滞后于交易执行。但在 Solana 上，由于出块速度极快（虽然相较于微秒而言依然存在间隔），但因为足够快，哪怕按上一个区块的价格成交，做市商通常也还能承受。因此就算你的预言机更新偶尔没被打包进去或者出现小插曲，做市商依旧有底气报出非常紧凑的点差。然而在以太坊上，12 秒前的价格已经极其古老和陈旧了，对吧？

### [00:12:57 - 00:13:40]
**EN:** So looking about, we're talking about for the most liquid pairs, right? It uses C type thing. We're talking what like stub basis points, top of book spread to mid. I mean, clearly you will understand that the actual price of the assets of Ethereum moves a lot more than that at any given, you know, 12 second period. And so you can't, there's no way you're going to execute at the previous block price unless either you slip by really a lot or you just don't execute, right? But both of it is bad execution because either you get a shitty price or exactly, right?
**CN:** 试想一下，对于流动性最高的核心交易对（比如 ETH/USDC 这一类），盘口最优买卖价差到中间价（top of book spread to mid）通常只有几个基点乃至次基点（sub-basis point）。显然任何人都能理解，在任意一个 12 秒的窗口期内，以太坊等基础资产在外部市场的真实价格波动幅度都远远超出这几个基点。因此做市商绝对不可能以上一个区块的价格去成交，除非要么承担极大的滑点让步，要么干脆回滚拒绝成交。但这两种情况都意味着极度糟糕的执行体验——要么成交价格极烂，要么无法成交。

### [00:13:40 - 00:14:07]
**EN:** So basically both both outcomes are bad. So here it's imperative that your Oracle update gets top of block. And so with that, the role of block builders who actually build the blocks and order the transactions is it's imperative. So if you try to like just say naively update your Oracle in the public mempool and everybody was trying to do an Oracle update in the public mempool.
**CN:** 两种后果显然都不可接受。因此在以太坊上，将预言机价格更新精准插入块首（top of block）是至关重要的硬性前提。随之而来的，负责实际组装区块并对交易进行全局排序的区块构建者（Block Builders）就扮演了生死攸关的核心角色。试想，如果你天真地试图把预言机更新发送到公共内存池（public mempool）中，而所有做市商都在公共内存池里争夺预言机更新优先权……

### [00:14:07 - 00:14:43]
**EN:** People would have to probably pay higher fees to get those updates included. Right. And it would probably congest the public mempool and it would make it very difficult for the updates to land and there'd be a lot of competition. But because you're sending the updates directly to a block builder, you're streaming it. And if I understand the block builder is enforcing the Oracle update as the first transaction to touch that particular contract of that prop AMM, then they can guarantee that with every block that lands, the first transaction will be of that contract will be the Oracle update, which will then give the fresh quote where the execution is done against that price.
**CN:** 主持人：各方可能不得不支付极为昂贵的 Gas 费以确保更新能被优先打包，对吧？这不仅会严重拥堵公共内存池，导致价格更新极难精准着陆，还会引发激烈的恶性竞价。但由于你们是直接将报价更新流式推送到区块构建者那里，如果我理解无误的话，区块构建者会强制将该预言机更新作为触碰该专有做市 AMM（propAMM）合约的第一笔交易。这样他们就能绝对保证在每一个最终落地的区块中，该合约的第一笔交易必定是预言机价格更新，进而提供最新鲜的报价，后续的所有吃单交易都将严格基于该最新价格撮合。

### [00:14:43 - 00:15:08]
**EN:** Is that, is that right? That's right. So how did you think about this architecture because it's different than Solana and clearly like it requires maybe some off-chain coordination here. Was it like as simple as saying, okay, I know we need to stream quotes to the block builder faster just because like that's the only thing that's going to work or did you look at like any set of alternatives?
**CN:** 主持人：这是正确的理解吗？  
Katia：完全正确。  
主持人：那么你们当初是如何构想出这种架构的？因为它与 Solana 的运行机制截然不同，显然需要高度精密的链下协调。是因为你们一眼就看出“唯有以更低延迟向区块构建者流式推送报价才是唯一解”，还是你们此前也评估过一系列其他备选方案？

### [00:15:08 - 00:15:31]
**EN:** Did anybody say, oh, like we can just build like a side chain for this something like maybe like a Mev commit? Yeah, no, we didn't go the side chain routes. We do have a joke at Bebop that like, you know, whenever we hit a stumbling block, we say Bebop chain is the solution to this. Or at least my CTO does. And then I get like really upset about this and we move on.
**CN:** 主持人：当时是否有人提议说：“我们可以专门为此建一条侧链，比如类似于 MEV-Commit 之类的预确认方案”？  
Katia：没有，我们压根没有走侧链这条路。在 Bebop 内部我们有一个常开的玩笑：每当我们遇到绊脚石时，大家就会调侃说“做一条 Bebop 链就能解决这个问题”——至少我们的 CTO 总是这么打趣。然后我就会为此抓狂反驳，接着大家便收起玩笑继续攻关。

### [00:15:31 - 00:16:00]
**EN:** But consider that to be fair, but, and of course, because it's Bebop, it has to be the OP stack chain. And, you know, we have the whole lore about it in Bebop. So it's the, that kind of thing. But anyway, we want to do it on Ethereum, right? Again, the capital states on Ethereum. This capital needs to exact that good execution quality or great execution quality, if anything. So we want to do it on Ethereum.
**CN:** 不过平心而论，如果要发链，因为我们叫 Bebop，那必须得是一条基于 OP Stack 的 L2 链（笑），这在 Bebop 内部是个经久不衰的梗。但言归正传，我们坚定地要在以太坊主网上实现它。正如前面所强调的，体量庞大的资本沉淀在以太坊上，而这些巨资极度苛求优质、乃至顶级的执行质量。因此，我们必须扎根以太坊主网。

### [00:16:00 - 00:16:24]
**EN:** So we needed to work with the block builders, see if we can get enough coordination, see if they're interested and kind of figure that out. So, yeah, so that's what we did. What gave you the conviction to say like, oh, I know I can just go, you know, talk to a few of these block builders and get them to agree. Like, is that years of relationship building that you've done?
**CN:** Katia：因此我们必须与区块构建者（Block Builders）通力合作，去验证能否建立起足够顺畅的协同机制，确认他们是否对此感兴趣，并把全套流程彻底摸索清楚。是的，这就是我们的切入点。  
主持人：是什么给了你底气和信心，让你确信自己可以直接去找这些区块构建者并说服他们达成共识？这是依托于你们多年来积累的行业深厚人脉吗？

### [00:16:24 - 00:16:50]
**EN:** Is it you invite everybody to dinner and then you make a proposal? Like, what does that look like in practice? I didn't know any thing about block building or any block builders before quite recently, I had a Telegram contact from someone from Titan from like two years ago at some event, some maybe event or something like that in Denver, I think, like being them. Flashbots, we had some, some connections.
**CN:** 主持人：还是说你把大家都请去吃了一顿丰盛的晚宴，然后在席间抛出了合作提案？在实践中具体是怎样的过程？  
Katia：坦率地讲，在不久前我对区块构建以及各大区块构建者几乎一无所知。我手里唯一的联系方式，还是两年前在丹佛某场 MEV 活动上偶然加过的一位 Titan 团队成员的 Telegram。我直接在 Telegram 上敲了他们；至于 Flashbots，我们此前也保持着一些业务往来。

### [00:16:50 - 00:17:19]
**EN:** And so I asked like, who, who was looking at this? So, yeah, no, I mean, like there was, there was no dinner. We had a coffee with Titan and the first meeting, I suppose. And then, yeah, Flashbots will only meet face-to-face in Cannes already having worked with them for, for a while, because on the BuilderNet side, they have been working really actively on more generalized solutions for including Oracle updates into the execution sort of flow on Ethereum.
**CN:** 于是我四处打听：业内有谁正在探索这个方向？所以根本没有什么豪华晚宴，我们只是约了 Titan 团队喝了杯咖啡，完成了初步接洽。而对于 Flashbots，我们也是在深入合作了一段时间之后，才在戛纳（Cannes）第一次线下碰面。因为在 BuilderNet 方面，Flashbots 一直在极力推进一套更加通用的标准化解决方案，旨在将预言机更新原生纳入以太坊的执行流水线中。

### [00:17:19 - 00:17:54]
**EN:** And yeah, so we, more primarily, VBOP CTO has been contributing quite a lot to the design of that. That initiative overall is led by Flashbots and Uniswap. So these guys are obviously like super involved in this. And yeah, we just, we, I guess, contributed and shared the requirements and listened. And so, yeah, that worked out great. I did tweet about it, I think last week, super grateful to all the people who explained to me what, how these things work.
**CN:** 是的，Bebop 的 CTO 重点为该方案的底层机制设计贡献了大量力量。该倡议整体上由 Flashbots 和 Uniswap 牵头主导，他们在此方案中投入了极深精力。而我们则是分享了具体的实际业务诉求，贡献了技术思路，并认真吸纳各方反馈。这一合作推进得非常顺利。我上周还在 Twitter 上发文，由衷感谢那些向我耐心拆解这套底层复杂运行原理的技术专家们。

### [00:17:54 - 00:18:17]
**EN:** Have you talked to specifically like the market makers on your platform and they have opted into this new architecture and also have you talked to any of the other like competing DEX apps that are thinking about doing something similar here? Because I think one thing you noted in the release was that this can provide like a lot of coordination overall for the ecosystem, not just for VBOP.
**CN:** 主持人：你是否专门与 Bebop 平台上的做市商进行了深入沟通并让他们入驻接入了这一全新架构？此外，你是否与正在考虑推行类似机制的其他竞争性 DEX 应用有过交流？因为我注意到你们在发布公告中特别强调，这套方案不仅能赋能 Bebop，更能为整个去中心化金融生态提供深度的协同价值。

### [00:18:17 - 00:18:37]
**EN:** So I was just curious maybe if you could touch on that. Yeah, we started with one of our market makers on VBOP who I think some time ago, you know, who has always been telling me that, hey, like if you're doing something new, like we'd like to participate and kind of be with you early in that. So it's a 1010 is the name of the, of the market maker.
**CN:** 主持人：所以我很好奇你能否展开谈谈这方面？  
Katia：好的。我们最初是与 Bebop 上的一家核心做市商开始落地的。在很久之前他们就经常对我说：“如果你们打算探索任何新事物，我们都极其渴望尽早参与进来共同开拓。”这家做市商的名字叫 1010（1010 Trading）。

### [00:18:37 - 00:19:06]
**EN:** Fun fact, I have known them always as MX, MX3. Interesting. And then I realized two years ago, I don't know for however long I knew them, I've known them because MX of course is 1010. So we started with them and we have a few, a couple of makers integrating as well, and, you know, definitely there's been some good interest from various parties. So yeah, we are working through it like that.
**CN:** 一个有趣的冷知识：我以前一直以为他们叫 MX 或 MX3。后来我才恍然意识到——不管认识了他们多久，我终于想通了为什么他们叫 MX，因为在罗马数字里，X 就是 10，MX 对应的正是 1010（M=1000/X=10）这一谐趣代号。我们最先与他们达成了合作，目前还有几家做市商也正在紧密接入中。各方参与者对该架构表现出了浓厚兴趣，我们目前正是按照这个节奏扎实推进的。

### [00:19:06 - 00:20:00]
**EN:** Do you see this as being able to increase order flow for VBOP overall? Yes. And yes, I mean, I think some of the, the research that was put out, at least by the folks at Titan showed maybe like a six basis point improvement over $10,000 of trade went with doing some LVR Markout calculations for end users, and I think that clearly like is a good value proposition, but what would you say to, let's say maybe somebody who typically maybe trades with like a zero X or with maybe a, a Cal swap who has been using these products maybe for years, maybe trades pretty decent size regularly on a theory of main net when rebalancing positions and such, how do you think about maybe explaining this value proposition to some of the folks who are doing more of like the active spot trade?
**CN:** 主持人：你认为这是否能在总体上显著提升 Bebop 的订单流规模？  
Katia：是的。  
主持人：确实如此。Titan 团队发布的一项研究显示，在基于 LVR 价格偏离度（LVR Markout，做市商抗逆向选择的加价测算）对终端用户进行量化评估时，每 10,000 美元的交易额大约能实现 6 个基点（bps）的实质性执行质量改善。这显然是一个极具吸引力的核心价值主张。但你会如何向那些习惯使用 0x 或是 CoW Swap 的交易者阐述这一优势？比如那些长期使用这些产品、在以太坊主网上定期调仓并执行大体量交易的用户，你会如何向这些活跃的现货交易者讲清楚 BopAMM 的真正价值？


---

## Part 2 (00:20:00 - 00:40:00)

# Deeply Intents Ep. 47 - PropAMMs Won, You Just Didn't Notice (Part 2)

**嘉宾**：Katia Banina（Bebop CEO）  
**主题**：专有做市 AMM（PropAMMs / BopAMM）、链上流动性演进与可交易预言机（Transactable Oracle）  
**概要**：本部分对话深入探讨了 BopAMM（区块预言机定价 AMM）与传统 AMM、链下 RFQ 机制的技术与经济微观结构差异。Katia 与主持人剖析了 RFQ 在防范毒性订单流（toxic flow）和原子套利时的局限、链上合约可组合性与执行确定性的价值、以太坊 12 秒区块间隔导致的 LVR（再平衡损失）与 MEV 困境，以及成熟资产（如外汇稳定币、主流加密资产）与长尾代币在做市机制上的根本分流。最后，双方详细讨论了 BopAMM 如何通过流式报价（RFS）与区块构建者（Block Builders）协作实现毫秒级区块内聚合更新，以及这种“具备真金白银可交易性”的链上预言机价格对于以太坊生态清算、盯市与 EBBO 基准建设的深远意义。

---

### [00:20:00 - 00:20:29]
**EN:** So that's what we're shooting on mainnet, not just makers and maybe sophisticated takers. - Okay, so let me first talk about that, you know, six bebs improvement. I think that was the comparison to Unity 3. Comparing these two RFQ quotes, actually, it's tighter, but the RFQ markets on the theorem for ETSGC power specifically is incredibly tight, right? So yes, we do see improvements again, like we're going into like sub basis points.
**CN:** 这正是我们在主网上瞄准的目标，不仅面向做市商（makers），也服务成熟的吃单者（sophisticated takers）。——好的，那先谈谈你提到的 6 个基点的改善（6 bps improvement）。我认为那是对比 Uniswap v3（Unity 3）得出的。如果对比 RFQ 报价，实际上价差收得更紧；但在以太坊上，ETH/USDC 交易对（ETSGC power）的 RFQ 市场本身就已经极度收窄了，对吧？所以确实，我们再次看到了改善，甚至已经进入到了亚基点（sub-basis point，小于 1 bps）的微观竞争区间。

### [00:20:29 - 00:21:01]
**EN:** It's spread to mid in propAMM and maybe like BopAMM rather, or maybe like one and a half, something like that on RFQ. So those, it's close, right? So already RFQ improves compared to an AMM pull. However, RFQ is a fully off-chain, cut-off price discovery, if you will, because you need to receive a quote of chain and then the chain is the settlement layer, right?
**CN:** 专有做市 AMM（propAMM），更确切地说是 BopAMM 的中间价价差（spread to mid），与 RFQ 相比大约也是 1.5 个基点左右，两者非常接近。所以相比普通 AMM 流动性池，RFQ 已经有了明显改善。然而，RFQ 本质上是一种完全链下的报价发现机制（或者说链下截止定价），因为你必须在链下接收报价，而区块链仅仅充当最终的结算层。

### [00:21:01 - 00:21:32]
**EN:** And so that creates, for example, geographical latency, it creates basically a bunch of different things that kind of reduce the amount of flow you might be able to get, right? There's a lot of toxic flow in that as well from like atomic arbitrage and whatnot, right? And so generally, it's not the kind of flow you can serve within RFQ. Is that primarily because of the commitment that the RFQ makes? Of course. Okay. Of course.
**CN:** 这就会导致诸如地理网络延迟等瓶颈，产生诸多限制做市商捕获订单流规模的阻碍。其中还充斥着大量来自原子套利（atomic arbitrage）等行为的毒性订单流（toxic flow）。通常情况下，这类订单流是无法在 RFQ 机制内得到承接的。——这主要是因为做市商在 RFQ 中必须给出确定性的报价承诺（firm quote commitment）吗？——当然。没错，当然是因为这个。

### [00:21:32 - 00:22:07]
**EN:** So you can reverse the quote, sit on the quote, do some flash-long kind of thing, right? And it's not cool for us to deal with it. We're dealing with it. We're going to be dealing with it even better. So don't try this at home, but yeah, generally, generally, yeah. Because of that. However, at the same time, right, if a maker, for example, you know, is skewed the particular side, right, and they want actually to sell more or buy more, and they're pricing that side much better in the context of the stream, right, then, well, you know, if someone's orbiting that, like, it's actually, it might be good for that particular maker, right?
**CN:** 套利者可以对报价进行逆向操作、持价观望（sit on the quote，延迟执行以赚取免费期权收益），或者结合闪电贷（flash loan）进行套利。应对这种情况对我们做市商来说非常棘手。我们目前正在处理，并且后续会处理得更好。普通用户切勿轻易模仿，但大体上的确就是因为这个原因。然而与此同时，如果某个做市商的头寸出现单边倾斜（skewed），他们实际上非常希望能多卖或多买以再平衡仓位，并在流式报价中对该方向给出了极具吸引力的定价，那么一旦有人来对该价格进行套利（arbing），对该做市商而言反而是一件好事。

### [00:22:07 - 00:22:40]
**EN:** And so there are different things that occur when you can interact directly with a contract on chain via, on chain calls, rather than requesting quotes outside of that. But there are use cases, like you gave me an example of Xerox or Cowswap. So that is, I wouldn't call it as much trading. It's more of kind of like converting or maybe like, you know, taking a position, exiting the position. It's not constant trading.
**CN:** 因此，当你能通过链上合约调用直接在链上与合约交互，而不是在链外请求报价时，整个市场微观动态是截然不同的。但对于某些具体用例，比如你刚才提到的 0x 协议（ZeroX）或 CoW Swap，我认为它们并不算典型的高频交易，而更偏向于资产兑换（converting），或者单纯的建仓与平仓离场，并非那种持续不断的连续交易。

### [00:22:40 - 00:23:07]
**EN:** It's a different, it's not like watching the chart kind of trading, right? And so what matters there is not just the pricing, but also the reliability. So how do you make sure that if you want to trade, you actually get to trades? And how do you not slip by, you know, crazy amount and all that kind of stuff? And so that is what's important for, to those guys.
**CN:** 这种交易方式截然不同，不是那种盯着 K 线图的看盘交易。在这些场景中，关键不仅在于报价是否最优，更在于执行的可靠性（reliability）。如何确保只要你想交易，就一定能够成功撮合成交？如何确保交易不会遭遇离谱的极端滑点？这些才是对他们（聚合器与求解器）至关重要的核心考量。

### [00:23:07 - 00:23:32]
**EN:** And we know that very well, you know, cowswap solvers who work with all of them, they get penalized for failed transactions by the Cowswap auction rules. Xerox are, you know, in a best possible way, I mean, in a best possible way, completely obsessed about well-simulated quotes and no over-quoting and all that kind of stuff. And you know, it's great. So yeah, they definitely don't want failed transactions.
**CN:** 我们对此再清楚不过了。我们与所有的 CoW Swap 求解器（solvers）都有合作，根据 CoW Swap 的批次拍卖规则，交易失败会导致求解器被扣罚押金。而 0x 团队——在最积极的工程意义上说——极度痴迷于精准模拟报价（well-simulated quotes）和杜绝过度虚假报价（no over-quoting）。这非常棒。所以毫无疑问，他们绝对不能接受交易失败。

### [00:23:32 - 00:24:03]
**EN:** And that is where part of what Bob M.M. does is solving for that last mile kind of thing that they, you know, how do you make sure that actually the user who wants to transact gets to transact, even if their transaction never got on the lap of a block builder, even if their transaction got to the block builder who might only have like 0.5% market share, but they didn't order the Oracle thing direct.
**CN:** 这正是 BopAMM 所解决的核心痛点之一——攻克“最后一公里”问题：如何确保真正想要交易的用户必定能完成交易？哪怕他们的交易没有提交到头部主流区块构建者（block builder）手中，哪怕只是被一个市场份额仅有 0.5% 的小型构建者打包，且该构建者没有直接接入链下预言机数据流，用户的交易依然能够顺利执行。

### [00:24:03 - 00:24:28]
**EN:** That is a big part of what we're doing with Bob M.M. as well. So let's say you're wildly successful here, let's say that Bob M.M. really takes off in the best possible scenario and also probably just generally start taking off on Ethereum because people see the success of Bob M.M. and as usually with any kind of trends in Ethereum and crypto building, they, you know, people copy things and really start to compete.
**CN:** 这正是我们在推进 BopAMM 时的重要方向。——假设你们在此取得了巨大成功，在最理想的情况下，BopAMM 彻底爆发，并且由于行业看到了 BopAMM 的示范效应，专有做市 AMM（propAMM）开始在以太坊上蔚然成风——正如以太坊和加密开发中常见的那样，大家会迅速跟风模仿并展开激烈竞争。

### [00:24:28 - 00:24:50]
**EN:** What then happens to passive liquidity that is on Ethereum mainnet? Is that just like something toxic flow targets to trade against and rebalance? What becomes of those positions? Because like if all the flow then starts coming through proper AMMs, then, you know, passive LPs at some point I would think would say, well, this isn't really worth my time, so I'm going to start, you know, removing liquidity from some of these pools.
**CN:** 那么以太坊主网上的被动流动性（passive liquidity）将会面临怎样的命运？它会沦为仅仅被毒性订单流（toxic flow）针对套利与再平衡的牺牲品吗？那些流动性头寸最终会变成什么样？因为如果所有优质订单流都开始流经专有做市 AMM（propAMM），那么被动 LP 在某个时刻势必会意识到：“这根本不值得我继续投入”，并开始从这些传统流动性池中撤资。

### [00:24:50 - 00:25:20]
**EN:** Or do you think, do you just look at this as kind of striking an overall balance for the ecosystem and providing different types of liquidity for different users and different needs? I've never really quite understood the rationale behind providing completely possible liquidity. Like if you're managing Univi3, so you're managing it, you're probably like, you know, whether you're an individual or a company, you're a professional when it comes to managing those things, right? Like constraints liquidity.
**CN:** 抑或你认为，这其实是在为整个生态构建一种整体平衡，为不同用户和不同需求提供多元化的流动性类型？——我其实从来没有真正理解过提供“完全被动流动性（completely passive liquidity）”背后的商业逻辑。比如如果你在管理 Uniswap v3 的仓位，无论你是个人还是机构，只要你在管理集中流动性（concentrated liquidity），你就已经算是专业玩家了，对吧？

### [00:25:20 - 00:25:55]
**EN:** But when it comes to passive liquidity provision, well, that's like you get hit by the genius PR person came up with the impermanent loss, but also LDR, right? Because basically we're talking about, so, okay, so whenever someone says, hey, I've built something like some sort of mechanism that can only function if someone comes and orbs me or orbs this pool or like whatever, it's just, it blows my mind, right?
**CN:** 但谈到完全被动的流动性提供，你不仅会被天才公关发明的所谓“无常损失（impermanent loss）”所误导，还会实打实地承受 LVR（再平衡损失，Loss Versus Rebalancing）。因为归根结底，每当有人宣称：“看，我发明了一种机制，只有当外部有人跑来对我或者对这个池子进行套利（arbs me / arbs this pool）时，它才能正常运转”，这简直让我匪夷所思。

### [00:25:55 - 00:26:30]
**EN:** Because basically you say, hey, someone, like ULP, you put something in the pool or somewhere and someone comes and orbs you, you're losing out, right? Because clearly they're doing it for profit. But hey, this is so exciting. Now look, AMMs were incredibly excited because they made trading on chain possible when no one was interested, when there was no market maker, no profit. It is the innovation of DeFi alongside flash loads, right?
**CN:** 因为这种机制本质上是在说：“你作为 LP，把资产质押在池子里，然后套利者跑来割你，你遭受损失，因为对方显然是为了榨取利润而来，但大家居然还觉得这很酷很兴奋！”客观来说，AMM 最初确实令人无比激动，因为它在无人问津、没有专业做市商、缺乏利润动机的蛮荒时期，让链上交易成为了可能。它与闪电贷（flash loans）并列为 DeFi 最具突破性的底层原生创新。

### [00:26:30 - 00:26:57]
**EN:** Like RFQ and RFS, they're there, they can't be there, right? Like it's just like how you bring them in, they're not really innovations as much as I would want to claim those big words. But pools are, like the AMM pools, you know, UNI v2 kind of thing, of course they are, but what's important, right, like when it comes to, again, execution, quality, all of that, is price discovery.
**CN:** 像 RFQ 和 RFS，无论链上有没有，它们作为成熟机制早已存在。尽管我也想用宏大词汇来标榜，但把它们引入链上谈不上真正的底层范式创新。然而 AMM 资金池——比如 Uniswap v2 这类机制——确实是划时代的创新。但退回到最根本的问题，当重新衡量交易执行质量等核心指标时，最关键的生命线依然是价格发现（price discovery）。

### [00:26:57 - 00:27:37]
**EN:** So because if you operate a venue that does not dictate the price discovery, your venue operates in a way that's being arped to put you back in line with discovery, right? And so, and in Ethereum, this is exceptionally pertinent because the gap is 12 seconds. And because the state of the pools drifts so far away from the state, from the price discovery point, that this LVR problem just becomes, you know, really big.
**CN:** 如果你运营的交易场所不具备价格发现的主导权，那么你的场所就只能沦为被外部套利者（being arbed）纠偏的被动载体，强行被套利抹平与外部真实价格发现场所的差距。在以太坊上，这种情况尤为严峻，因为出块间隔长达 12 秒。流动性池的内部状态严重偏离外部真实的价格发现点，使得 LVR（再平衡损失）问题变得极其庞大且致命。

### [00:27:37 - 00:28:09]
**EN:** And it's very lucrative for MBB bots, but it's really damaging for the LPs, right? Coming back to your question, absolutely certainly any crypto native assets that we have seen much fewer of recently, admittedly, yeah, you start them with a pool, there is natural prices discovery with the basic curve or maybe with more advanced constraint liquidity management and whatnot. That's cool. So now let's take world currency stablecoins, Euro versus US dollar.
**CN:** 这对于 MEV 机器人（MEV bots）来说是极其丰厚的暴利，但对被动 LP 而言却是毁灭性的抽血。回到你的问题：毫无疑问，对于任何加密原生资产（尽管必须承认，最近新涌现的原生资产少了很多），你完全可以从普通资金池起步，通过基础的联合曲线或进阶的集中流动性管理来进行天然的价格发现，这非常合理。但现在让我们来看看全球法定货币稳定币，例如欧元兑美元（EUR/USD）。

### [00:28:09 - 00:28:38]
**EN:** Price discovery of that asset happens between a bunch of dealers, you know, large banks basically and trading firms in that market. You can't replicate it in a liquidity pool. That market is a trillion size a day market, there's no way you can replicate it in a liquidity pool. So of course, you need to bring that price discovery into it. And of course, those kinds of assets are not going to be sitting there as passive liquidity.
**CN:** 这类资产的价格发现发生在一群主交易商之间——本质上是外汇市场中的大型跨国银行与顶级自营交易机构。你根本不可能在一个链上流动性池中复制那种价格发现。那是一个日交易量数万亿美元的巨型市场，绝无可能仅靠流动性池完成定价。因此，你必须将外部真实的价格发现引入链上。显而易见，这类资产不可能作为被动流动性静止地躺在池子里。

### [00:28:38 - 00:29:08]
**EN:** So it depends on the asset, it depends on the use case, we have all the different tools now, right? And like more of them are coming on chain together with the non crypto assets, they're also coming on chain. So it's only natural. That makes a lot of sense. I actually really like that framing. I think it's really helpful. And it's a point that a lot of it's easy to miss, which is that for mature assets that are really well understood, constructions like a proper AMM make a lot more sense where the price discovery is happening in other venue where Ethereum is lagging.
**CN:** 所以这取决于资产性质和具体应用场景。我们现在拥有了多元化的工具，并且随着更多非加密传统资产上链，更多配套机制也在加速迁移，这种演化是必然的。——这非常有说服力，我非常赞同这个分析框架，它极具启发性且很容易被人忽视：对于那些已被市场透彻理解的成熟资产，价格发现发生在外部场所，而以太坊本身是存在时延滞后的，因此像专有做市 AMM（propAMM）这样的架构要合理得多。

### [00:29:08 - 00:29:38]
**EN:** So you're always going to be updating. But for an asset like Kawai coin that I just created for my cat, and I created a uni v3 pool against ETH, because it's episode that price discovery is going to happen on chain right there in that pool. And for that, you actually won AMM. And once that Kawai coin gets really well traded in is in demand, and likely some market makers will then in turn pick it up and then perhaps be quoting it via some some proper AMM construction.
**CN:** 因为你需要持续不断地向链上更新价格。但对于像我刚刚为我的猫发行的“卡哇伊币”（Kawai coin）这样的小币种，我在 Uniswap v3 上为它建了一个与 ETH 配对的池子，其初期的价格发现就必然直接发生在该链上池子里。对于这种场景，传统的 AMM 确实是不可或缺的。而一旦该代币交易活跃、需求旺盛，很可能会有专业做市商接手介入，进而通过某种专有做市 AMM（propAMM）架构开始为其提供深度报价。

### [00:29:38 - 00:30:07]
**EN:** And maybe like the liquidity also migrates to Binance or Coinbase or one of these other venues as well generally. So you have basically the long tail of assets, which AMMs are fantastic for. But for the high impact assets, this is really where you want the proper AMM construction to make sure that you're getting the tightest possible quotes, which are mirroring or close to mirroring whatever is happening in the price discovery venue, whether that's Binance or somewhere else.
**CN:** 随后其核心流动性也可能普遍迁移至币安、Coinbase 等中心化主流交易所。因此大体而言，传统 AMM 是长尾资产（long tail of assets）的绝佳温床；但对于高影响力、高交易量的主流资产（high impact assets），你真正需要的是专有做市 AMM（propAMM）架构，以确保能获取最紧凑收窄的极致报价，实时镜像或近乎镜像币安等外部价格发现场所的动态。

### [00:30:07 - 00:30:27]
**EN:** How much time does the do you have to actually update that quote? Is there like there's 12 second block time, but like is there like within that 12 seconds is there like a cutoff window where let's say Titan has to get the quote by otherwise they can't include it in in their next block. Excellent. And this was the question I was also asking.
**CN:** 你们实际上有多充裕的时间来更新这个报价？虽然以太坊出块时间是 12 秒，但在这 12 秒之内，是否存在一个明确的截止时间窗口（cutoff window），比如 Titan（区块构建者）必须在某个具体节点前拿到报价，否则就无法将其打包进下一个候选区块？——问得太精准了！这正是我在亲身探索过程中一直追问的核心问题。

### [00:30:27 - 00:31:09]
**EN:** This was one of the things that I learned as part of this journey and it blew my mind. So basically the answer to this is that you have to keep re updating this as fast as you can. So you literally stream it to them. Of course. It's RFS. See, like, it all makes sense now, right? Yes. It's okay, it's basically keep like sending the like streaming the most up to date prices and then, you know, block builders keep like rebuilding the blocks, which is also, you know, kind of cool, I guess, like from that sort of performance engineering thing.
**CN:** 这是我在这次探索历程中获知的最颠覆认知的机制之一。核心答案是：你必须以最快的速度毫秒级持续重推更新。所以你实际上是在向他们以数据流的方式推送报价（stream it to them）。当然，这正是 RFS（Request For Stream，流式请求报价）。你看，现在一切逻辑都串通了，对吧？做市商持续流式广播最新价格，而区块构建者（block builders）则在后台毫秒级持续重构区块（rebuilding blocks）。从极限性能工程的角度来看，这确实非常酷炫。

### [00:31:09 - 00:31:37]
**EN:** But they don't know which block is going to get picked up by a validator. There is no specific deadline, but it can be for what I understand, it can be like anything between sort of like two and one second, so you actually still have like a sort of one second old price, like once it hits the chain, but it's kind of like a bit less of a concern because everyone at the time has the same information.
**CN:** 但构建者事先无法获知验证者（validator/提议者）最终会采纳哪一个候选区块版本。这里并没有死板的硬性截止时限，据我了解，通常是在出块前大约 1 到 2 秒之间的某个动态节点。因此，当最新价格最终写入区块链状态时，它实际上仍然存在大约 1 秒钟的时滞；但这并没有那么令人担忧，因为在那个特定时刻，全网各方获取的信息是对等且一致的。

### [00:31:37 - 00:32:10]
**EN:** And so if someone's trying to like front run you or like argue in some way or go and, you know, I mean, the pools and things like this, everyone kind of has the same information, right? Like no one knows which like basically everyone keeps kind of sending their updates and their transactions having the same information. And then there is a like one second or so lag, but at least it's not that like someone knows the future basically and they buy that data kind of party gets left behind.
**CN:** 即使有人试图抢先交易（front-run）你、发起某种套利（arb），或者冲击资金池，所有市场参与者掌握的基准信息基本上是同质的。没有人能预知哪个区块最终会胜出打包，所有人都在基于相同的公开状态持续发送更新与交易。虽然存在约 1 秒的时差，但至少不会出现某一方拥有类似“预知未来”的信息不对称优势、从而让另一方遭受严重逆向选择被蒙在鼓里惨遭割肉的情况。

### [00:32:10 - 00:32:37]
**EN:** Doesn't matter where in the block the contract update happens, like let's say you have somebody else that's bidding for a top of block to update like some uni v3 pools and they're bidding like a really high priority fee because they want to take that opportunity. But then you also have all these prop AMM updates as well, like does it matter as long as the execution after that update is in the middle of the block, bottom of the block or should it be like in the top of the block?
**CN:** 合约更新具体落在区块的哪个位置有关系吗？比如，假设有竞争者正在支付极高的优先费竞价区块顶部执行（top-of-block），试图去更新或套利某些 Uniswap v3 资金池；但同时你们也有这些专有做市 AMM（propAMM）的报价更新。只要实际交易发生在更新之后，这笔交易是在区块中间、底部还是必须紧挨着置于区块顶部，影响大吗？

### [00:32:37 - 00:33:27]
**EN:** Kind of what Bob does, right, it brings together updates from all these different makers, right? And then puts them together in a single transaction update, which also kind of solve this problem of wasted Oracle updates, right? Like, first of all, it's shared so it's it can be optimized better for, you know, basically gas and all that kind of stuff. But also, I don't know, like, let's say if one maker is super wide or something, some assets, right, like you don't really and or someone like super tight right at this very moment that they can cover a very good depth without like this other maker without like a lot of depth from another maker, well, then great, like you don't really need to send the update of this other maker, right, so you don't need to have like every maker publishing their update.
**CN:** 这正是 BopAMM 的精妙之处：它汇集了所有不同做市商（makers）的报价更新，并将它们整合进单笔交易的批量原子更新中，这从根本上化解了预言机更新资源浪费（wasted Oracle updates）的难题。首先，它是多方共享的聚合更新，因此在 Gas 消耗等方面能够实现深度工程优化；其次，假设某个做市商在特定资产上的报价价差极宽，而另一家做市商此时此刻报价极窄且能独立提供充沛深度，那么系统就完全不需要上报那位报价宽泛做市商的数据，从而免除了每个做市商独立上链发布更新的冗余成本。

### [00:33:27 - 00:33:57]
**EN:** We basically bring this all together, right and say, well, okay, you know, this is the best possible pricing and this is the best possible book, basically for to transact on the theorem in this block, and it comes from a bunch of different makers. So that's one of the things actually that we do. Another way to look at it is so MEV, so like MEV transactions, generally, this Oracle updates has kind of, it has value and it doesn't have value.
**CN:** 我们将这些做市商的流动性统筹整合，向以太坊链上提供在当前区块内进行撮合的最优定价与最具竞争力的虚拟订单簿，它凝聚了多家做市商的合力。这是我们落地的核心创新之一。换个角度来看，对于 MEV 交易而言，这种预言机更新处于一种“既有价值、又没有价值”的微妙状态。

### [00:33:57 - 00:34:30]
**EN:** So for example, if you put the Oracle update, there is no transaction against it, like it doesn't really have any value. If the Oracle update is on the very top of the block, not just above all the transactions that sort of listen to the Oracle update, there's no difference. If the Oracle update comes before the MEV transaction that doesn't trade against this BobaMM, well, also has no like it doesn't do anything for better or for worse for that transaction.
**CN:** 例如，如果你在链上写入了预言机价格更新，但后续没有任何交易与之发生交互撮合，那这次更新本身并未产生直接经济价值；如果预言机更新被打包在区块最顶部，但只要它排在所有监听该预言机价格的交易之前，它具体在顶部的哪个微观位次并无实质区别；如果预言机更新排在某笔不与 BopAMM 产生交互的 MEV 交易之前，它对那笔交易也不会产生任何好坏影响。

### [00:34:30 - 00:34:54]
**EN:** So from this respect, like it's, it's quite benign, you know, kind of like this. Now what can happen afterwards and what matters is that ultimately the structure here is that it's an order book. It's not quite an order book, but it's, it means that, you know, you have tighter pricing on the top and then it widens kind of as you go deeper.
**CN:** 从这个角度来看，这种更新机制对外部是高度良性且中立无害的（benign）。真正关键的是更新落地后所发生的交易执行：其底层构架本质上类似于一个限价订单簿（order book）——虽然并不完全等同于订单簿，但它的经济结构意味着：最靠前触发的交易享受极其紧凑的极窄价差，而随着执行深度的递增，价差会呈阶梯状向外拓宽。

### [00:34:54 - 00:35:20]
**EN:** And so if you come first to this book, your spread is going to be tighter. And if you are like the last in that book, then yeah, then it's going to be wider, right? And so from that respect, there is value in kind of how your transactions are worded within there. And I wish it wasn't like this, but also kind of making RFS in a different way.
**CN:** 因此，如果你在当前区块中率先与该订单簿撮合，你所获得的价差就会更收窄；而如果你排在最后被执行，承受的价差就会变宽。正因如此，交易在区块内的微观排序位置（how your transactions are ordered within there）就蕴含了实实在在的经济价值。我个人其实希望机制不必如此残酷，但这与构建 RFS（流式报价）的另一种路径紧密相关。

### [00:35:20 - 00:35:43]
**EN:** That's not BobaMM and this is kind of coming soon as well. And that's kind of a problem too, but yeah, like that's the bit that I don't really like about it to be fair, because I think that, you know, if you want to transact like a hundred dollars, you should be able to transact a hundred dollars and not kind of at a million dollar spread type thing, but the same issue exists in, in AMM pools, right?
**CN:** 那是另一种非 BopAMM 的替代方案，很快也将面世。那套方案同样面临一些挑战。坦率地说，这也是我不太满意的一个细节：因为我认为，如果散户只想交易 100 美元，他就应当按 100 美元规模的极致窄价差成交，而不该被动承受宛如百万美元大单那样的深度折价与滑点。不过公平地讲，这一深度冲击成本在传统的 AMM 资金池中也同样客观存在。

### [00:35:43 - 00:36:04]
**EN:** And so from that perspective, like it's, it's, it's no different, but yeah, I still don't love it. So I'm also quite keen to fix that with something else. I think you maybe hinted at it in your article, and I think Titan did as well. What are some of the other use cases that you might envision for this Oracle update, which could be helpful in DeFi, if anything.
**CN:** 所以从宏观机制上看，二者并无二致，但我依然认为这不够完美，因此我也非常渴望探索其他机制来进一步修复这一痛点。——你在文章中可能暗示过这一点，Titan 团队似乎也曾提及。针对这种高质量的链上预言机更新，如果它能对 DeFi 生态产生积极助力，你还能预见到哪些潜在的应用场景？

### [00:36:04 - 00:36:28]
**EN:** Well, I guess like people like to talk about NDBL, which is the American national... Best bid and offer? Yeah, for on exchanges on like basically the single day from, like, I could take changes. It's not a perfect metric, whatever, but you know, it gives you like a good indication of where the liquidity of market is and kind of how, how the spreads are and whatnot.
**CN:** 人们经常谈论 NBBO（ASR 误作 NDBL），也就是全美统一最优买卖报价（National Best Bid and Offer）……——最优买卖报价（Best Bid and Offer）？——对，它整合了所有主流股票交易所的统一最优买卖基准报价。尽管它算不上尽善尽美的指标，但它能清晰反映全市场的流动性中枢所在以及实时的买卖价差分布。

### [00:36:28 - 00:36:48]
**EN:** I think Cowswap made an effort to create something they call EBB, Ethereum, Best Bid and Offer, which I applaud. I think it was really basic in kind of how it was, but maybe it was also quite, quite a few years ago, right? So like at the time, it probably would have been like one of the best ways to do this.
**CN:** 我记得 CoW Swap 曾做出过极富远见的尝试，他们提出了 EBBO（Ethereum Best Bid and Offer，以太坊最优买卖报价，ASR 误作 EBB），我对此非常敬佩赞赏。尽管在当时其实现机制相对简陋，但那是几年前的探索了，在当时的历史技术条件下，那可能已经是落地这一理念的最佳方式之一。

### [00:36:48 - 00:37:27]
**EN:** I think it was like some sort of kind of blended price from like some uni pools and curve and, you know, something else perhaps. But this is because of the 12 second blocks, because of LVR and MEV and all that kind of stuff, right? Like it's doesn't that price does not exist properly on chain, right? If you have a good quality price update, it then suddenly becomes the benchmark that you can use for, for example, your market to market for your quality of execution workouts for so many different things.
**CN:** 当时它更像是一种从 Uniswap、Curve 等流动性池中加权汇总的混合估值。但受制于 12 秒的漫长出块时间，以及 LVR、MEV 夹资套利等顽疾，真正的公允市场价格在以太坊链上根本无法稳定存在。而如果你拥有高频、高质量的实时链上价格更新，它就能一跃成为全生态的核心基准指标（benchmark），用于资产的盯市计价（mark-to-market）、交易执行质量分析（quality of execution workouts）等诸多关键领域。

### [00:37:27 - 00:37:54]
**EN:** And again, right? This comes back to, you know, if we're serious about serious capital on chain, right? This stuff is like what they do day in and day out, right? And so of course this is needed. Another thing is just general kind of oracles, right? So we see it's very expensive to send oracle updates for many assets on chain every block, right?
**CN:** 归根结底，如果我们严肃致力于推动大规模机构级资金（serious capital）上链，这些定价与清算基准正是传统金融机构日夜依赖的底层基础设施，因此其重要性不言而喻。另一个直接应用场景则是通用的链上预言机。我们都很清楚，在以太坊上为海量资产每个区块都写入一次预言机更新，其 Gas 成本是极其高昂且无法承受的。

### [00:37:54 - 00:38:19]
**EN:** And so that's why oracles only publish pricing if it moves by a certain amount or a certain amount of time has passed by, right? Well here, you know, we have an oracle that is there, it's reliable because it's transactable. So, you know, basically whoever sends the pricing, they're pushing their money where the mouth is, right? And so that's a very good, reliable quality of pricing with real transactions going through that.
**CN:** 这正是为什么传统预言机通常只在价格偏离达到特定百分比阈值、或经过固定的心跳时间间隔后才被动触发链上更新。但在我们的机制中，这个预言机不仅实时存在，而且坚如磐石——因为它是具备可交易性的预言机价格（transactable oracle）。提供该报价的做市商必须用真金白银为其背书（putting their money where their mouth is），在真实真金白银的交易撮合检验下，这沉淀出了极其可靠、高质量的定价数据源。

### [00:38:19 - 00:38:47]
**EN:** And yeah, and it's like super up to date. It's combined from multiple different pricing sources, which are market makers, which are the closest to the pricing. And so yeah, that's cool for anything else, right? Liquidations, whatever that thing. Yeah, I think that's pretty awesome because it actually provides like more, it lessens the trust assumption on like any kind of singular oracle that you might have, especially on pricing.
**CN:** 而且它是超实时的。它聚合了多个最贴近市场前沿的专业做市商源头报价。因此对于 DeFi 的其他应用——比如借贷协议的头寸清算（liquidations）等——具有无可比拟的赋能价值。——这确实非常硬核，因为它大幅削弱了 DeFi 对任何单一集中式预言机的信任假设，尤其是在价格馈送这一最脆弱的攻击面上。

### [00:38:47 - 00:39:12]
**EN:** So like, for instance, if you're getting some chain like update on, and you also have this now Bob AMM quote, or this aggregated market maker quote, as well as an oracle price. If there's a discrepancy between them, you have that information. If there's not discrepancy between them, or if one of them doesn't publish for whatever reason or fails to publish, you also have like the other one to fall back on.
**CN:** 比如，如果你正在接收来自 Chainlink 的预言机更新，同时链上又有来自 BopAMM 的聚合做市商可交易报价作为双重参考：一旦二者发生异常偏差，协议能立即捕获这一预警信号；如果二者紧密吻合，或者某一方因网络拥堵等任何偶发故障未能及时发布，协议还有另一方作为高可靠性的安全兜底（fallback）。

### [00:39:12 - 00:39:39]
**EN:** So I think actually it gives some interesting flexibility. And I wonder also too, if there's like something programmatic, like another application that could be built, which ingests the quote, once it's published, and then maybe execute some other action as a result of this, whether it's a liquidation or something else. Yeah, I think there's some optionality there. That's interesting. But you know, absolutely, like, I think it's just a really good sort of process of externality of this.
**CN:** 这赋予了架构极具想象力的灵活性。我还在思考，是否可以构建某种程序化智能合约，一旦该报价在链上发布，下游应用就自动消费该价格并触发级联动作，无论是自动化清算还是其他策略。——是的，这里蕴藏着广阔的可组合期权性（optionality）。这非常有意思，我认为这完全是这项技术为整个以太坊生态带来的极其可观的正外部性（positive externality）。

### [00:39:39 - 00:39:59]
**EN:** Now, obviously, it's like to, for it to become very meaningful, it has to be has to come into every block, and it has to have enough volume behind it, you know, so that it's hard to manipulate and whatnot, right. And so kind of on our, on our sides, you know, we operate as a neutral layer, we have operated with that.
**CN:** 当然，要让这种机制真正具备系统级影响力，它必须稳定嵌入到每一个区块中，并且背后必须有充足深厚的真实交易量作为锚定支撑，以此杜绝恶意操纵等风险。因此在我们这一端，我们始终坚守并践行着一个中立基础设施层的定位……


---

## Part 3 (00:40:00 - 00:58:20)

# Deeply Intents 播客第 47 期：PropAMMs 赢了，只是你尚未察觉（Part 3）

**嘉宾**：Katia Banina（Bebop 联合创始人兼 CEO）  
**主题**：区块预言机定价机制（BopAMM）、做市商网络延迟与东京节点、L2 扩展性、DeFi 协议与团队 OpSec 防御、AI 在工程与策略中的实际应用、场外衍生品（OTC Derivatives）与金库级原子可组合性

---

### 内容概览 (Overview)
在本期访谈的最后一部分（Part 3）中，Katia Banina 与主持人深入探讨了 propAMM 与 BopAMM（Block Oracle Priced AMM）落地的深层技术细节与市场展望：
1. **网络架构与延迟优化**：为何 Bebop 和头部加密做市商选择将核心服务器部署在东京（Tokyo servers），以及区块构建者（Block Builders）与验证者之间的时延博弈。
2. **L2 与排序器共存**：探讨将流式报价机制（RFS）直连 Rollup 排序器的可能性，以及以太坊主网与 L2 之间在流动性深度上的取舍。
3. **团队安全与 OpSec 挑战**：面对 AI 挖 0-day 漏洞和频发的社交工程/钓鱼攻击（假会议链接、SIM 劫持、Deepfake），无资金池（No-TVL）的 RFQ 架构如何保障协议安全，以及加密创始人的日常防范守则。
4. **AI/LLM 的实际效能**：AI 编写代码与系统优化中的优势，以及为何避免用 AI 从零起草文字内容以规避“AI 腔”和思路束缚。
5. **DeFi 的下一个前沿**：传统全球金融规模最大的场外衍生品（OTC Derivatives）向链上迁移的路径，以及 Bebop 即将推出的 RFQ/RFS 升级——支持在单笔原子交易中完成金库提取、还款、RWA 铸造与兑回的深度资金乐高可组合性。

---

### [00:40:00 - 00:40:41]
**EN:** Over a dozen market makers for a few years, and we definitely are neutral here, but at the same time, of course, it's kind of like a new way to think about verticals and data on chain. But generally, tradable data or traded data is something that's very valuable and generally trustworthy. And so that's why it's something that we're kind of thinking about how to really make a meaningful benchmark out of that and the more makers we have as part of the platform, the more of the quality of this benchmark we're going to be able to create.  
**CN:** 过去几年来我们连接了十几家做市商，在此我们绝对保持中立；但与此同时，这确实开启了一种审视链上垂直赛道和数据的新视角。总体而言，可交易数据或真实成交数据（tradable/traded data）具有极高的价值，且通常具备高度可信度。因此，我们一直在思考如何基于这些数据构建出真正有意义的价格基准（Benchmark）。随着加入平台的做市商越来越多，我们能够打造的这一基准的质量也会越来越高。

---

### [00:40:41 - 00:41:05]
**EN:** Were there any kind of a side question, but just intellectually curious, were there any kind of things that you learned about network or latency optimization with getting quotes to these block builders and also maybe with block builders even sharing quotes amongst each other? Because I think ideally what you want is not only the major builders, but the long tail of builders to all opt into this.  
**CN:** 插一个出于纯粹技术好奇的题外话：在将报价推送给这些区块构建者（Block Builders）的过程中，或者在构建者彼此共享报价方面，你们是否总结出了关于网络传输或延迟优化的关键经验？因为从理想状态来看，你们不仅希望顶级主流构建者参与进来，也希望长尾构建者都能接入这一机制。

---

### [00:41:05 - 00:41:29]
**EN:** So basically, anybody who is proposing a block is also proposing this Oracle update quote, right? Did you learn anything about networking? Yeah, I would just be curious in this line of thinking. Yeah, I was going to learn more about it. But then I realized that the latest update that gets picked up by the validators is like a second and a half or something, and I was like, wow, cool.  
**CN:** 所以从机制上讲，任何提交区块提案的人都会同时提交这一预言机更新报价，对吧？你们在底层网络传输方面有什么收获吗？我很想了解这方面的思考。  
是的，我之前也打算深入探究这一点。但我后来意识到，验证者最终打包采纳的最新报价更新，通常是在区块时隙截止前 1.5 秒左右。当时我的反应是：哇，这太有意思了。

---

### [00:41:29 - 00:42:00]
**EN:** But I mean, still there is performance optimizations on our side. I mean, in general, we have already been for quite a while, like our servers and everything in Tokyo, again, because a lot of price discovery of crypto is there because most market thinkers are there. So we want to be as close to market makers and obviously builders have also their points of presence in there, but then the validator can be anywhere, and then latency matters.  
**CN:** 不过在我们这边，依然存在大量的性能优化工作。实际上，我们的核心服务器节点很早以前就已经部署在东京（Tokyo servers），这主要是因为加密市场的绝大部分价格发现都在那里发生，多数主流做市商（此处原词 market thinkers 系做市商 market makers 之误）也都聚集于此。我们希望尽可能在物理上靠近做市商，而构建者显然在东京也设有网络接入点（PoP）。但问题在于，以太坊验证者可能分布在世界任何角落，这就让物理时延变得至关重要。

---

### [00:42:00 - 00:42:30]
**EN:** I mean, I know that there are plenty of improvements and optimizations that can be done there. But at the same time, we still have this unknown which one of them gets picked up and when. And is it like one second before, two seconds before, or I don't know, 500 million, we don't know. But yes, it's important for a number of reasons. Is it like more important than a bunch of other problems?  
**CN:** 我知道在网络层还有大量的改进和优化空间。但与此同时，我们依然面临一个核心未知数：究竟哪一笔报价会在什么精确时间点被采纳？是在区块截止前 1 秒、前 2 秒，还是前 500 毫秒？我们无法预知。出于多种原因，这确实很关键；但它是否比其他一系列工程问题更为紧迫？

---

### [00:42:30 - 00:43:01]
**EN:** And like this is imperative to solve before anything else to an extent, but not fully. I'm assuming you've probably already thought about L2s where you can integrate maybe directly into a roll up sequencer where market makers are able to stream these quotes and get these high latency updates and also to probably co-locate with some of these sequencers who maybe have like, I don't know, maybe a cluster in a particular jurisdiction, but not in others for legal reasons or something like that.  
**CN:** 在某种程度上需要优先解决，但也并非绝对压倒一切。我猜你们大概也考虑过 L2 生态，在那里你们或许能直接与 Rollup 排序器（Sequencer）集成，使做市商能够直接向排序器串流低延迟/高频报价；甚至可能与这些排序器进行物理机房同地托管（Co-location）——尽管出于合规或法律原因，某些排序器集群可能只会部署在特定司法管辖区。

---

### [00:43:01 - 00:43:25]
**EN:** Have you started to look into the L2 space at all? There's nothing that sort of prevents us from being on an L2. I heard from a couple of L2s who were kind of interested to explore this for sure. We kind of really focused on Ethereum for a few different reasons and it kind of comes back to my RFQ versus RFS.  
**CN:** 你们目前是否已经开始布局 L2 领域了？  
技术上没有任何障碍阻止我们部署到 L2。确实有好几家 L2 团队联系过我们，表达了探索合作的兴趣。但我们目前把核心精力聚焦在以太坊主网上，这背后有几个考量，归根结底又回到了我之前提到的 RFQ（询价）与 RFS（流式报价）的区别上。

---

### [00:43:25 - 00:43:54]
**EN:** You know, RFS needs to be there when markets are very liquid and there is a lot of demand. And so, you know, it just depends, right? Like what is the asset? What is the new chain and overall supply and demand and things like that. We clearly know that Ethereum is big enough to do this and that's where we started. But yeah, I mean, generally, of course, what you say is totally doable too.  
**CN:** 你知道，RFS 机制只有在市场流动性极高、交易需求极其旺盛的环境下才真正成立。所以这取决于很多维度的权衡：交易标的资产是什么？新公链的整体供需结构如何？我们非常清楚以太坊主网的体量足够大，足以支撑这一模式，所以我们选择从这里启航。但毫无疑问，你刚才提到的 L2 路径在工程和商业上也是完全可行的。

---

### [00:43:54 - 00:44:20]
**EN:** I'd like to transition a little bit. I think you gave us a great explanation on prop AMMs, the motivations. We dug into some technical questions about the architecture, talked about some of the goals that Bob AMM is trying to achieve and also just like the affordances overall for the ecosystem and some of the benefits. I'd be curious maybe more like on a just a company level for Bebop.  
**CN:** 我想稍微切换一下话题。刚才你对 propAMM 的底层逻辑与诞生动因做了非常精彩的阐述，我们深入剖析了系统架构的技术细节，探讨了 BopAMM（区块预言机定价 AMM）旨在达成的目标，以及它为整个 DeFi 生态带来的价值赋能。接下来，我很想从 Bebop 公司运营与团队安全的层面探讨一些问题。

---

### [00:44:20 - 00:44:42]
**EN:** What you guys have seen in the last six months or so, we've seen like a lot of DeFi hacks and we've also seen the announcement of different AI, quad mythos, different models that seem to be finding zero days and bugs and all kinds of different exploits and it feels like every week I'm reading about something, even if it's minor, $6 million here, $7 million there.  
**CN:** 纵观过去半年左右的行业动态，我们目睹了大量 DeFi 黑客攻击事件；同时，我们也看到诸如 Claude 等尖端 AI 模型能够挖掘出 0-day 漏洞、代码 Bug 以及各种攻击载荷。现在感觉几乎每周都能读到被黑的消息，哪怕涉案金额不算惊天动地，也是这里损失 600 万美元、那里损失 700 万美元。

---

### [00:44:42 - 00:45:24]
**EN:** Has this changed your approach to security at all or just how you guys are thinking about it in this environment or yeah, just like any thoughts that you've had recently on this topic? Yeah, of course. I mean, I guess human level is quite scary, right? It's just, yeah, it's really scary and you, of course, have to think about it. Well, kind of on Bebop level, generally kind of RFQ model is said to have fewer vectors of attack in general and that's what they have not been found for us just yet.  
**CN:** 面对这样的宏观环境，这是否改变了你们对安全防御的策略或思考方式？你们近期在这个议题上有什么心得？  
当然。从人的心理层面来说，这确实令人不寒而栗，你必须时刻绷紧神经。从 Bebop 协议本身的机制来看，通常认为 RFQ 架构的攻击向量（Attack Vectors）显著少于常规流动性池，至少到目前为止，我们还没有暴露出这类的智能合约漏洞。

---

### [00:45:24 - 00:45:58]
**EN:** So, and yeah, and so we don't hold TVL and all those things, they definitely matter and they make a difference in sort of how we think about protocol security. But what I would say scares me much more is the whole kind of OpSec part. I receive a lot of DMs and just outreach of any sort, like let's say in Telegram, right? And sometimes it's from people I don't know, right?  
**CN:** 是的，我们不沉淀或托管资金池总锁仓量（TVL），这些底层特性在决定我们如何构建协议安全防线时起到了决定性作用。但坦白讲，真正让我感到更加恐惧的是整个操作安全（OpSec）层面。我在 Telegram 等社交渠道上每天都会收到海量的私信和各种形式的商务联络，其中不少来自素未谋面的陌生人。

---

### [00:45:58 - 00:46:24]
**EN:** And sometimes just introductions or whatnot and it's totally cool, right? And sometimes it's like someone who I knew and who I connected with in person. And then like we have a normal human chat and when they agreed to catch up, right? And they send me some dodgy Teams link and I go like, "Oh, I'm sorry, it doesn't load for me. Can you join my Google Meet?" or something like that, right?  
**CN:** 有些只是普通的业务引荐或交流，这很正常；但有时发信人明明是我在线下亲身结识过的熟人。我们像往常一样寒暄，当约好线上详聊时，对方却突然发来一个形迹可疑的 Microsoft Teams 会议链接。我的第一反应通常是：“抱歉，我这边加载不出来，你能加入我的 Google Meet 会议室吗？”

---

### [00:46:24 - 00:46:47]
**EN:** And then kind of after a few back and forth, the person disappears and then the next day I read like about like exactly that whole kind of scam on Twitter. And I just know that this is constantly happening to me and I think so far I haven't joined the call like this, but it is scary from that perspective for sure.  
**CN:** 几个回合推拉之后，对方就彻底音信全无了。结果第二天，我就在 Twitter 上看到了针对这种特定钓鱼套路的曝光拆解帖。我知道这类攻击几乎无时无刻不在针对我发生。虽然迄今为止我从未误入过这种恶意会议，但从安全防御的角度来看，这确实令人心惊胆战。

---

### [00:46:47 - 00:47:32]
**EN:** And that is what worries me quite a lot, but I mean a lot of hacks we've seen with like DNS takeovers and things like that, right? So I'm worried about that, definitely not less than I'm worried about zero, they will not do it exactly. It's very hard to tell nowadays if somebody is reaching out for legitimate reasons or not. And as you noted, it could be somebody that has an account that you actually connect it with in real life and you remember connecting with them and then the conversation just kind of either gets a little bit weird or yeah, you get an invite for a link or something to click on that just seems a little bit sus and there's a lot of it.  
**CN:** 这正是我深感忧虑的隐患。此外，我们还见证了大量针对域名解析系统（DNS 劫持）的攻击事件。我对这类基础设施劫持的担忧，丝毫不亚于对 0-day 代码漏洞的担忧。如今，你极难辨别某人的沟通请求究竟是出于正常业务诉求还是恶意陷阱。正如你所言，那个账号可能确实属于你在线下打过交道的人，但聊着聊着对话画风就变得诡异，或者发来一个极其可疑的外部点击链接——这种攻击手段如今实在太泛滥了。

---

### [00:47:32 - 00:47:54]
**EN:** It's hard to tell I think what's real or not. Do you have like any kind of filter or protocol that you use? I mean, for me, usually my role on most things is like if I haven't engaged with you either in real life or on Twitter, then I'm not, I'm just going to assume that if you're reaching out to me, it's probably a scam.  
**CN:** 现如今确实真假难辨。你们在内部是否有某种过滤机制或安全流程？对我个人而言，我处理绝大多数事务的准则是：如果我们既没有在线下接触过，也没有在 Twitter 上公开互动过，那么只要你主动私信找我，我都默认将其视作潜在骗局。

---

### [00:47:54 - 00:48:30]
**EN:** I know that like cuts off some people who have legitimately reached out via telegram to me, but usually I find like if I don't know the person, then I would prefer to just interact with them in public on Twitter to like understand if they're real or not and also see like who else they interact with, see what mutuals they have before I engage further because I've just found like it's impossible to tell if I get a message from somebody on telegram and then like they say they're legit, but then like a month later I noticed they changed their account to like some other picture with some other name or something like this and they could have been SIM swapped obviously.  
**CN:** 我知道这种做法可能会误伤一部分通过 Telegram 正式寻求合作的人。但我通常觉得，如果不认识对方，我更倾向于先在 Twitter 上进行公开互动，借此判断对方是否真实可信，并在深入沟通前观察他们与谁互动、有哪些共同关注。因为现实中如果你在 Telegram 上收到消息，即便对方自称正规机构，一个月后你可能突然发现该账号换成了另一副头像和名字——很显然，他们极大概率遭遇了 SIM 劫持（SIM Swap）。

---

### [00:48:30 - 00:48:50]
**EN:** I mean, that happens to people pretty trivially and there's also like Google voice issues that people have had, right? But yeah, I'd just be curious if you have any kind of like best practices that you follow to keep yourself safe. For all I know, you might be a deep fake. No, but like I think the drift. That was really scary. Was really scary, right?  
**CN:** 这种安全事故发生得太频繁了，还有各种针对 Google Voice 虚拟号的安全漏洞。所以我很想知道，你们是否遵循某些最佳实践来确保自身安全？毕竟就我所知，眼前的你甚至可能是一个深度伪造（Deepfake）的数字人。  
不开玩笑，之前 Drift 协议遭遇的安全风波（社交工程/深度伪造诈骗）真的非常惊险。确实极其可怕，对吧？

---

### [00:48:50 - 00:49:20]
**EN:** Because yeah, like, you know, you just said, oh, I met these people face to face. Oh, you know, it and that is really scary. So look, I don't have a magic wand here and it scares me that I don't, I wish I did. I think I go by the rule that if I have like, you know, a tiny shadow of a doubt, I just err on the side of caution.  
**CN:** 因为就像你刚才说的：“我明明跟这帮人线下面对面聊过。”这种建立在真实信任之上的欺骗才最为致命。所以你看，在这件事上我并没有什么万灵药，正因没有万全之策才让我更加敬畏。我的基本信条是：只要心中升起哪怕一丝一毫的怀疑，我都宁可过度谨慎、直接终止危险操作。

---

### [00:49:20 - 00:49:45]
**EN:** But yeah, it's not a very bulletproof approach. Usually I would also go and like run it past someone and I'm like, do you think this looks sus and yeah, like, you know, sometimes people are like, oh yeah, I don't know, like, kind of, but I can't put my finger on it. And it's yeah, it's really scary. It's really scary.  
**CN:** 但客观来说，这算不上坚不可摧的万能解法。通常我还会找同事复核，问他们：“你觉得这个链接或沟通看着可疑吗？”有时候大家也会说：“呃，不好说，感觉确实哪里不对劲，但又说不上来具体问题出在哪。”面对这种未知的暗箭，确实让人感到脊背发凉。

---

### [00:49:45 - 00:50:15]
**EN:** The other question I wanted to ask you aside from this one was how in the last maybe year have you and your team transitioned to using AI more regularly in your day to day, maybe both from like an engineering perspective, but also maybe from a business strategy or a research perspective, have there been like anything that you have found that have worked really well or anything that maybe has slowed you down?  
**CN:** 除了安全话题，我还想问的是：在过去一年左右的时间里，你和你的团队是如何在日常业务中更常态化地引入 AI 工具的？无论是在工程开发端，还是在商业战略与市场研究端，有哪些实践你觉得成效显著？又有哪些尝试反而拖慢了团队的节奏？

---

### [00:50:15 - 00:50:51]
**EN:** Interesting about like slow slowing you down because I think actually, well, for me personally, I think I did find some things, some situations where it slowed me down to, for example, I don't really like writing and at one point I sort of tried to give some bullet points to AI and just have it like write something for me. But then I realized that you would like anchor me to structure that doesn't quite flow or doesn't feel natural to me or, you know, basically there's like too much editing and I now kind of go and write the thing myself.  
**CN:** 谈到“拖慢节奏”这点很有意思，因为对我个人而言，确实在某些场景下体会到了负面拖累。比如我本身并不热衷于撰写长篇文字，曾经我尝试给 AI 一些要点提纲，让它直接帮我扩写成文。但我很快发现，它生成的文章会把我禁锢在一种行文僵硬、不符合我个人自然表达习惯的思维框架里；最终我必须耗费大量精力去反复修改。现在我索性全部自己亲手撰写。

---

### [00:50:51 - 00:51:13]
**EN:** And then sometimes I would go to AI and go like, okay, can you like phrase this sentence better? And then maybe I would run the whole thing past it and say, please don't change the like, don't start sounding like AI. But then, so, you know, it's kind of more like using AI as an editor in a very deliberate way rather than editing the AI, right?  
**CN:** 随后我偶尔会把自己的草稿发给 AI，指令它：“能不能帮我润色一下这个句子的措辞？”或者把整篇文章让它通读一遍，并严正嘱咐：“请不要改动我的核心风格，千万别写出那种典型的 AI 腔调。”所以说，这更像是在非常克制、目标明确地把 AI 当作文字编辑来辅助，而不是被动地去给 AI 的产物擦屁股。

---

### [00:51:13 - 00:51:44]
**EN:** So like for me, that one thing that I found slowed me down and didn't give me the help I needed, right? And actually made it harder, right? And so now I use it in a different way and I think it does help with that. As a team, operationally, there's a bunch of AI gains for sure. That's one part that I think is kind of mostly a no-brainer and it's kind of quite easy to just kind of add into your workflow.  
**CN:** 所以对我来说，盲目依靠 AI 代笔是一件既不能提供实质帮助、反而拖慢工作效率的事情。在转变协作模式后，现在的用法确实高效得多。而在整个团队的运营协作层面，AI 带来的生产力提效是实打实的，在很多日常环节中引入它几乎是不需要纠结的基础操作，非常容易融入现有的工作流。

---

### [00:51:44 - 00:52:14]
**EN:** Maybe from a dev perspective, it's probably, it's been especially good in maybe describing some problem to an LLM and kind of getting solutions for how to fix it kind of like holistically or how to potentially approach it. So that was interesting. Not on the design level, more kind of like on an optimizations level and you know, things like that. That's I think quite useful.  
**CN:** 如果从软件研发的角度来看，大语言模型（LLM）的表现尤为亮眼：你可以向它详细描述一个具体工程痛点，它能够从系统全局的角度给出排错思路或潜在的破局路径。这非常令人惊艳。虽然它在宏观架构设计上尚不能独当一面，但在局部性能调优、代码优化等具体层面，我认为它展现出了极高的实用价值。

---

### [00:52:14 - 00:52:44]
**EN:** So I mean, we do have like plenty of things kind of running and going on. I think people do use it differently. It's not necessarily mandated, but everyone's using it, right? So yeah, that's kind of, it's just naturally there. Do you have your own custom like local model for the team or are you still more reliant on some subscriptions from like the frontier providers? I know everybody has frontier subscriptions.  
**CN:** 我们内部有很多相关的自动化流程在运转。团队成员对 AI 的使用方式各有不同，我们并未推行强制性的使用规范，但每个人都在自发地借助它提效。这已经演变成为一种潜移默化的常态。  
你们团队是在本地私有化部署了定制的开源模型，还是依然高度依赖顶级头部供应商的前沿模型订阅服务？我知道业内所有人都在订阅前沿模型服务。

---

### [00:52:44 - 00:53:07]
**EN:** So like a local model doesn't relinquish that need. They're not quite as capable just yet, but there are some great capabilities for like that you can like have maybe like some more private information locally that you might run either on the server or some hardware that you run yourself. Yeah, we have, I think some of the team have some stuff like running locally. I don't know a lot of detail on that to be fair.  
**CN:** 毕竟本地部署模型并不能完全取代前沿大模型，它们当前的综合能力还存在差距；但本地硬件或私有服务器部署在处理机密数据时确实具备天然的隐私隔离优势。  
据我所知，我们团队里确实有部分工程师在本地硬件上部署并运行了一些模型，但坦白讲我对具体细节没有做过多干涉。

---

### [00:53:07 - 00:53:50]
**EN:** I mean, generally I'm not particularly, I do care about privacy as a concept, but I also think that, yeah, like I don't think someone's gonna like, you know, someone cares basically about the code that we are writing and then putting on chain and in a verified state anyway type situation, you know? So yeah, like I think that's kind of the way we approach it and I think generally the sort of privacy and especially like AI privacy, LLM privacy is a much bigger question and issue than we bought code sitting there or like, you know, interacting with it.  
**CN:** 总体而言，虽然我从理念上非常重视隐私保护，但我也认为，反正我们编写的合约代码最终都是要部署上链并在区块链浏览器中开源验证的，我并不觉得有人会专门通过模型去觊觎这些即将完全公开的代码。这就是我们的务实态度。而且我认为，在宏观层面，AI 和大模型的隐私问题远比 Bebop 自身编写的代码与模型交互所引发的顾虑要宏大、复杂得多。

---

### [00:53:50 - 00:54:20]
**EN:** And so from that respect, I, yeah, like I'm not paranoid about running things locally for that reason at least. Last question I'll ask you is, do you think that there is any DeFi innovation that is on the horizon that people don't see coming yet is something that is going to be very apparent maybe in the next year or two? Well, I'm gonna be boring again, right? To say something like, DeFi is about finance, right?  
**CN:** 从这个角度出发，我至少不会为了这种代码涉密理由而陷入必须本地部署的偏执。  
我的最后一个问题是：你认为未来一到两年内，DeFi 领域是否会出现某种目前绝大多数人尚未预见、但随后会变得极为显而易见的颠覆性创新？  
那我可能又要给出一些看似“平淡无奇”的观点了：DeFi 的核心本质始终是金融。

---

### [00:54:20 - 00:54:52]
**EN:** And so to grow it, it needs to be able to do things that finance or global finance already does. And so the DeFi part of it will probably be like adapting it to this environment. So OTC derivatives is something that is the largest market out there in the world. It's bigger than anything exchange traded. It's bigger than, you know, anything spotted completely and utterly massive.  
**CN:** 要想推动 DeFi 真正发展壮大，它必须能够复现并支撑全球传统金融已经在成熟运转的业务。而 DeFi 的职责，就是将这些金融工具适配到区块链的原生环境中。比如场外衍生品（OTC Derivatives），它是当今全球体量最为庞大的金融市场，其规模远远超越任何场内撮合交易所的交易量，更非现货市场所能比拟，是一个体量不可思议的绝对庞然大物。

---

### [00:54:52 - 00:55:32]
**EN:** And right now the, like none of what we have in DeFi is equipped to support that. So yeah, so that. So whatever that is, it's a combination of multiple different things, right? But that's what it is. I really think that, you know, the innovation here is blockchain. You know, blockchain is the public ledger with its composability, with its, you know, so permissionless access, with its, you know, interrupt composability, it's got the same thing I suppose, right?  
**CN:** 但放眼当前的 DeFi 基础设施，没有任何一个现有协议具备承载这一规模级场外衍生品的能力。因此，无论未来的终极形态究竟是何种模块的协同组合，这都将是兵家必争之地。我坚信，这场变革真正的基石在于区块链本身——区块链作为公共账本，赋予了我们无需许可的准入机制、跨协议互操作性以及确定性的原子可组合性。

---

### [00:55:32 - 00:56:05]
**EN:** Like, that is, with the fungibility of, you know, your C20 tokens and that standard and all that kind of stuff, right? Like that is what the innovation is. And those are the things that can take whatever exists and create new use cases for it and make it shine in different ways. So that's real innovation. I think, you know, a lot of us, what we're doing is kind of marrying the best of both worlds together.  
**CN:** 再加上以太坊 ERC-20 等代币标准的同质化与标准化，构成了真正的底层技术飞跃。正是这些原生特性，能够将现实世界的既有资产模式移植过来，解锁前所未有的全新应用场景并绽放出别样生机。这才是货真价实的创新。我们这个行业中的很多人，所做的正是致力于将链上原语与传统金融的精髓完美融为一体。

---

### [00:56:05 - 00:56:32]
**EN:** And those of us who are going to do it best are going to be more successful. But that's, I think, what really the next innovation is. More money Legos for DeFi. Beautiful. Kati, it's been awesome catching up with you, where can people reach out to you if they want to talk about Bob A.M.M., do you have anything else that you want to tease on your way up? Our website, our docs have plenty of contact points.  
**CN:** 谁能在这场融合中做到极致，谁就能赢得最终的胜利。我认为下一波创新的实质，正是为 DeFi 带来更多威力强大的乐高积木式可组合性（Money Legos）。  
太精彩了。Katia，今天与你的深度交流非常畅快！如果大家想围绕 BopAMM 与你探讨，可以通过什么渠道联系到你？最后还有什么即将发布的重磅更新想要剧透的吗？  
我们的官方网站和开发文档中提供了丰富的联系渠道。

---

### [00:56:32 - 00:56:59]
**EN:** They're obviously on Twitter and places, so you can DM me, but please don't scam me. And if I don't reply, then that you sounded really sus, so I'm sorry in advance. Yeah, so that's teasing what's coming out. Well, yeah, I mean, I did talk about something that I don't particularly like, but this kind of propaganda stuff type kind of design, but the word rank.  
**CN:** 团队成员也都在 Twitter 等社交平台保持活跃，大家可以给我发私信——但拜托千万别是钓鱼欺诈。如果我没有回复你，那大概率是因为你的发信内容触发了我的防骗风控警报，在此我先提前致歉。  
至于未来的重磅剧透：虽然我前面提到了某些我个人并非全盘推崇的叙事——比如业内当前关于专有做市 AMM（此处原词 propaganda stuff 实指 propAMM 设计）的各种炒作噱头；

---

### [00:56:59 - 00:57:28]
**EN:** But yeah, I mean, generally, it has its place, right, totally investing in it and going to make it a success. But we also have RFQ, right? And we are bringing, we're kind of unleashing like the probably most exciting update to RFQ we've had, and we're doing kind of two things with it. One, some of this RFQ we're going to turn into RFS in a magical way.  
**CN:** 但客观来讲，这种模式确实有其用武之地，我们正在全力投入资源以确保其取得成功。但别忘了，我们同样拥有强大的 RFQ 引擎。我们即将推出 Bebop 创立以来最激动人心的一次 RFQ 架构升级，它主要聚焦两大方向：第一，我们将以一种极其精妙的链上机制，将部分传统的 RFQ 询价流转变为 RFS 连续流式报价。

---

### [00:57:28 - 00:57:55]
**EN:** And the second part is we're going to unleash a lot more composability with RFQ, basically allowing you to do a bunch of kind of cool things together with the quote that the executing is a swap. So, you know, minting or redeeming an RWA in the same transaction, or, you know, taking stuff out of a vault and putting it back in. So basically, you know, what it's telling you that, you know, the real innovation is stuff like composability.  
**CN:** 第二，我们将彻底释放 RFQ 的深度可组合性潜力——允许用户将做市商的承兑报价与多步骤复杂链上交易无缝捆绑。比如：在单笔交易中完成真实世界资产（RWA）的铸造或赎回，并同时与报价完成闪电兑换；抑或是在同一笔交易中实现金库可组合性（Vault Composability），从金库中闪电提取资产、执行兑换并将收益再存回金库。这再次印证了我刚才的观点：真正颠覆性的创新始终根植于底层的原子可组合性。

---

### [00:57:55 - 00:58:20]
**EN:** Like, that's what I mean, right? And like, it's up to us to kind of harness that, the power. And that's why we're doing that on the next situation of RFQ, which is coming out, I think really soon. Well, next month for sure. So it's almost there. It's been a pleasure chatting with you and hopefully we have more to talk about in a few months. Cool. Thank you so much.  
**CN:** 这就是我的核心构想，而充分驾驭并释放这种可组合性的力量，正是我们肩负的使命。这正是我们即将在新一代 RFQ 迭代版本中交付的核心功能——它很快就会上线，最迟下个月就会与大家见面，敬请期待。今天和你聊得非常尽兴，希望过几个月等新产品发布后我们再来录一期深度复盘！  
太棒了，非常感谢你！
