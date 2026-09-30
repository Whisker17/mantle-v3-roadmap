# Credible Commitments: Block Building & PropAMMs on Ethereum - Kubi Mensah
## 全集双语对照转录全文 (Full Bilingual Transcript)

- **播客节目**：Credible Commitments
- **嘉宾**：Kubi Mensah（Gattaca / Titan Builder 联合创始人兼 CEO）
- **音频时长**：54 分 22 秒
- **音频文件**：
- **核心主题**：区块构建者视角下的以太坊交易供应链、PBS（提议者与构建者分离）架构演进、区块构建的经济学与优化挑战、专有做市 AMM（PropAMM）在以太坊上的实现机理与构建者协同、块首价格预言机注入、私有订单流（Private Order Flow）与订单流拍卖（OFA）。

---

## 目录 (Table of Contents)
1. [Part 1 (00:00:00 - 00:20:00) 从 YC 到 Titan Builder：以太坊区块构建者的崛起与 PBS 演进](#part-1-000000---002000)
2. [Part 2 (00:20:00 - 00:40:00) 区块构建微观经济学与 PropAMM：块首定价、LVR 消除与构建者协同机制](#part-2-002000---004000)
3. [Part 3 (00:40:00 - 00:54:22) 延迟军备竞赛、包含列表（IL）与以太坊交易执行的未来](#part-3-004000---005422)

---


## Part 1 (00:00:00 - 00:20:00)

# Credible Commitments 播客深度访谈：以太坊区块构建与 PropAMM（Part 1）

**嘉宾：** Kubi Mensah（Gattaca / Titan Builder 联合创始人兼 CEO）  
**主题：** 以太坊区块构建生态、市场演进与专有做市 AMM（Block Building & PropAMMs on Ethereum）  
**本期概要：** 在本部分访谈中，Titan Builder（市场份额占比超 50% 的顶级区块构建者）联合创始人兼 CEO Kubi Mensah 分享了他早期入选 YC（Y Combinator）并成功退出初创公司的经历，回顾了以太坊合并（The Merge）后提议者与构建者分离（PBS）架构下区块构建者角色的演进；深入剖析了 Titan 如何通过坚持“中立构建者”定位、拒绝自营交易冲突、深耕底层工程优化与高品质客户服务，在激烈的市场竞争中建立起核心壁垒与订单流飞轮效应。

---

### [00:00:00 - 00:00:59]
**EN:** Good morning, Kubi. Welcome to the podcast. Thank you for joining us today. Hey, good to be here and thanks for having me. Normally, I start the show by asking guests to give a brief description about their background, but I think you're pretty well known in the Ethereum ecosystem. You've done the podcast circuit a little bit. When I was doing some research for this episode, though, I noticed, correct me if I'm wrong, but you spent some time at YC very early on in your career, and you had some success as exiting a startup previously before you got into block building. I'd just be curious if you could talk about that experience and anything fun or anything that you learned out of that.  
**CN:** **主持人：** 早上好，Kubi。欢迎来到播客，非常感谢你今天加入我们。  
**Kubi：** 嗨，很高兴来到这里，感谢邀请。  
**主持人：** 通常在节目开始时，我会请嘉宾简要介绍一下自己的背景，但我想你在以太坊生态系统中已经非常知名了，也参加过不少播客节目。不过在为这期内容做功课时我注意到——如果我说错了请纠正我——你在职业生涯早期曾进过 YC（Y Combinator），并且在进入区块构建（block building）领域之前就已经成功退出过一家初创公司。我很好奇你能不能聊聊那段经历，分享一些趣事或者你从中汲取的经验。

### [00:00:59 - 00:01:33]
**EN:** Yeah. So I think probably like for most startup founders, especially if you're in tech, if you're not based in the US or not in Silicon Valley, you kind of see the big stars from afar. And so there's always a bit of admiration and like this fantasy of the interesting and intriguing tech world, and it all happens in Silicon Valley. So I would say, so I'm originally from Germany and have been living in London for a long time, so based in Europe.  
**CN:** **Kubi：** 好的。我想对于大多数初创公司的创始人来说，尤其是科技领域的创业者，如果你不在美国或者不在硅谷，往往只能在远方仰望那些耀眼的科技巨星。所以心中难免会带着一丝崇拜，对那个充满魅力、激动人心的科技世界怀揣着某种幻想，而这一切似乎都发生在硅谷。就我个人而言，我来自德国，后来在伦敦生活了很长时间，一直扎根在欧洲。

### [00:01:33 - 00:02:35]
**EN:** And so us getting into YC with my previous startup was sort of the opportunity to actually look behind a curtain and see what that world really looks like. That was super exciting. I would say the biggest revelation was that you start to realize that these very famous people often, at least in those circles, are just human beings. And very much like maybe if you have a pop idol or someone you admire an artist. So once you actually see them in the real world and you interact with them, so obviously they're still super smart and hardworking and all of that. But I think the biggest mental shift, which someone can tell you, but it's very different when you actually experience it. It just switches something in your brain that, okay, these are just normally humans and they're especially humans, but they're just humans. And so everything seems more possible. I think that was sort of the biggest takeaway for me.  
**CN:** **Kubi：** 因此，我和上一家初创公司入选 YC，就像是一个得以揭开帷幕、真正一窥那个世界全貌的契机。那段经历令人极其兴奋。我认为最大的感悟是，你开始意识到这些在圈内名声赫赫的人物，归根结底也只是普通人。这很像你追捧某位流行偶像或敬仰某位艺术家，一旦你在现实世界中真正见到他们并与之互动，尽管他们依然极其聪明、极其勤奋，但最大的心理转变在于——这种感受别人可以对你说上千百遍，可当你亲身经历时是完全不同的——你的大脑中仿佛被扳动了某个开关：原来他们也只是常人，虽有非凡之处，但终究是有血有肉的人。如此一来，世间万事似乎都变得更有可能实现了。我想这就是我当时最大的收获。

### [00:02:35 - 00:03:13]
**EN:** At what point did you feel like, I belong, I am an entrepreneur, I'm successful, I know what I'm doing? Right. I don't think that ever happened. And I would say once you're also in those circles, so we managed to raise a little bit of money, we exited the company, but in the grand scheme of things, we were just, we are not top of our YC badge. We are not on the front page of TechCrunch and all of that. So yeah, I don't think unless you are maybe one of the top startups in the world, you still never feel really like you made it or you belong.  
**CN:** **主持人：** 在哪个时间节点上，你开始觉得自己“真正属于这个圈子了，我是一名成功的创业者，我深谙此道”？  
**Kubi：** 是的，我觉得那种感觉其实从未真正出现过。而且当你身处那些精英圈子中时，尽管我们成功融到了一笔资金并完成了公司退出，但放眼更大的格局，我们并不是那期 YC 批次（YC batch）中最拔尖的，也没有登上 TechCrunch 的头版头条。所以，除非你成为全球最顶尖的几家初创公司之一，否则你永远不会真正觉得自己已经功成名就，或者彻底融入了那个殿堂。

### [00:03:13 - 00:03:42]
**EN:** Yeah. So for the listener today, it's a pleasure to have Kubeon and we're going to talk a little bit about prop AMMs in the meat of the show. But before we do that, I want to ask some questions just about block building and kind of the state of things just to get people up to speed. I think there's a lot of interesting developments and I think it would be great to hear from the number one builder himself leading the number one team. So start with a very basic question. What is a block and what is the role of a block builder in Ethereum?  
**CN:** **主持人：** 好的。对于今天收听节目的朋友们，我们非常荣幸能邀请到 Kubi。在稍后的核心板块中，我们将重点探讨专有做市 AMM（PropAMM / Proprietary AMM）。但在那之前，我想先就区块构建以及当前的行业现状提几个问题，帮助大家跟上背景节奏。目前行业内涌现了许多非常有趣的发展，能直接聆听带领排名第一团队的顶级构建者本人的见解，实在再好不过了。我们先从一个最基础的问题切入：什么是区块？在以太坊中，区块构建者（block builder）扮演着怎样的角色？

### [00:03:42 - 00:04:33]
**EN:** The way I think about it is the block is just the way blockchains essentially get updated. And if you think about it, there's limited compute capacity or processing capacity that a blockchain has. And that gets made available every, you know, in discrete time intervals on Ethereum in 12 seconds. And so there's a process of aggregating all the transactions, all the interactions with the chain and then batching them together and then making one big batch update. And so the block is the aggregation of, you know, transactions or interactions with the blockchain. And then there's a way of once that update has been locked in, there's a way to then transition everyone's data across the globe to and everyone agreeing.  
**CN:** **Kubi：** 在我看来，区块本质上就是区块链进行状态更新的载体。仔细想想，区块链所拥有的计算能力或吞吐处理容量是有限的。在以太坊上，这种处理能力是以离散的时间间隔释放的，也就是每 12 秒产生一个时隙（slot）。因此，这其中包含了一个将网络中所有的交易、所有与链的交互请求汇集在一起，将它们打包成批次，进而执行一次大规模批处理状态更新的过程。所以区块就是对链上交易和状态交互的聚合体。一旦这次状态更新被正式敲定锁定，全球所有节点的数据状态就能据此完成平滑转换，并让所有人达成一致共识。

### [00:04:33 - 00:05:16]
**EN:** So the block itself is basically like a batch update. And from that perspective, the block builder is, you know, just like the name says, the block builder is the entity that is then responsible for aggregating all the requests to interact with the chain and then trying to optimize to get as many of these requests fulfilled, such that the total fees that are being paid to interact with the blockchain are maximized. So that's sort of the objective function, so to speak. And then once that has been achieved, then there's an intricate process of, okay, what happens with those total fees and how do they get redistributed?  
**CN:** **Kubi：** 因此，区块本身基本上就是一次批处理状态更新。从这个视角来看，区块构建者（block builder）正如其字面意思，是一个专门负责汇总所有链上交互请求的实体；它的核心工作是通过全局算法优化来满足尽可能多的请求，从而最大化用户为了与区块链交互所支付的总手续费（total fees）。可以说，这就是构建者的目标函数（objective function）。而一旦达成这个目标，接下来就会进入一个极其精密的流程：这些汇总的手续费该如何处理？它们又是如何被重新分配给验证者及生态各方的？

### [00:05:16 - 00:07:55]
**EN:** Beautiful. So users have a lot of requests. They have some transactions, for lack of a better word, and these get batched together in discrete time intervals. And you have a global state update. The block builder is somewhat the coordinator of this process in an intricate supply chain, which we can or cannot get into. But, you know, back in 2021, pre the Ethereum merge, you still had proof of work, and you still had miners, and you still had a media guess as this client that I think like 90% of the miners had opted into at that point in time. So the mining pools were building a portion of the blocks. But then after the merge, things changed. And you had kind of this emergence of the need for a builder. And part of the reason you had this design, if I recall correctly, was because we wanted to isolate validators and keep them kind of dumb and pipey, and keep them decentralized, right? So anybody can spin up a validator, and they don't need this sophisticated algorithm to be running. And they don't need also any kind of incentive to have any kind of co-location or relation with a particular entity. And so you had this role with the block builder in merge. And I think the first time I saw you speak was at MEV Paris, and you were on the stage with Nathan and some others. You guys were really kind of getting into the meat of what this was looking like, you know, maybe a year into this process. How have things changed from when you started right after post merge, and maybe you can get some color on exactly when you started, and where you are now where you have a ton of water flow, a ton of clients and customers to satisfy as well. So maybe just like reflecting on how the role of the block builder maybe has evolved since that point in time.  
**CN:** **主持人：** 总结得太精辟了。用户有大量请求——姑且统称为交易——这些交易在离散的时间间隔内被打包批处理，从而完成一次全局状态更新。在这个复杂的交易供应链中，区块构建者在某种程度上扮演了总协调者的角色，关于这条供应链我们稍后可以深入探讨。但在 2021 年以太坊合并（The Merge）之前，网络采用工作量证明（PoW）机制，由矿工负责出块；当时 Flashbots 推出的 MEV-Geth 客户端据说有将近 90% 的矿工主动接入，由矿池自行构建一部分区块。然而合并之后机制彻底变了，提议者与构建者分离（PBS）架构下催生了对独立构建者（builder）的刚性需求。如果我没记错的话，这种架构设计的核心考量之一，就是为了将验证者（validator）完全隔离，让他们保持轻量、简单和“管道化”（dumb and pipey），从而维持整个网络的去中心化，对吧？这样任何人都能独立运行一个验证者节点，既不需要运行复杂的 MEV 提取算法，也不会产生将服务器托管在特定数据中心（co-location）或与特定垄断巨头绑定的利益驱动。因此，合并带来了区块构建者这一专业角色。我记得第一次看你演讲是在 MEV Paris 会议上，当时你与 Nathan 等嘉宾同台，深入探讨了合并运行约一年后这个生态的实际运转状况。从合并刚结束你们起步，到如今你们掌握着巨量的订单流（order flow）、需要服务海量客户，这期间发生了怎样的演化？能否谈谈你们具体是何时切入的，以及区块构建者的角色自那时起经历了怎样的演变？  
**Kubi：** 回答关于入场时机的问题：其实在合并发生的那一刻，我们并没有立即上线构建者服务。尽管在此之前的一场 MEV Day（我记得是在阿姆斯特丹）上，我们第一次听说了未来可能出现的区块构建者角色，当时便感到超级兴奋。原因在于，构建区块兼具了硬核底层工程与极限性能优化，对我们来说是非常引人入胜的技术挑战；但与此同时，它具备底层核心基础设施的属性，能让你真正扎根于整个庞大生态之中。而在那之前我们是一家自营交易公司，主要在链上做搜索（on-chain searching / 搜索者），纯粹是在链上捕获套利机会。因此转型做关键基础设施非常有吸引力。不过，我们当时花了一段时间去评估这项业务在商业模式上的可持续性，所以并未在合并当天立即上线。以太坊合并发生于 2022 年 9 月 15 日。

### [00:07:55 - 00:08:40]
**EN:** And then we launched our builder in January or February 23. So it's roughly, yeah, almost half a year after. So that's the thing in regards to timing. And the way we looked at it was, hey, this is super interesting, interesting technical problems, you are becoming part of the core infrastructure, which is intriguing. And there's more of a service element to it versus just capturing probably somewhere. But it was very not as professionalized, I would say, which, you know, three years later, the market structure has evolved a bit, but also just sophistication of everyone who's just part of the ecosystem or still around.  
**CN:** **Kubi：** 随后我们在 2023 年 1 月或 2 月才正式推出了我们的 Titan Builder，大概是在合并近半年之后，这是关于时间线的情况。我们当时的思考逻辑是：这是一件极具技术挑战的工作，解决这些底层工程问题能让你成为以太坊核心基础设施的一部分，这非常令人神往。而且与单纯在链上到处捕获套利收益相比，构建者业务具备更强烈的客户服务属性。但在当时，整个行业的专业化程度还相当初级；三年后的今天，不仅整体市场结构经历了长足演化，留在生态中的每一个参与者的成熟度与专业水准也都大幅提升了。

### [00:08:40 - 00:09:47]
**EN:** And so initially, we only looked at it as a function, you know, something to plug into like an existing system. So there's a mempool. And then there's a relay, you take construction from the mempool, and you submit a block, and you try to build the most valuable block. And it's basically just plug and play. But then as you actually start going into it, you get more familiar with intricacies. And even back then, you know, there were the large builders like, for example, Beaver Build was one of the top builders, the other one was Arsing, and there was one called Builder 69, which was, I think at the time, actually the largest builder. And so it was also a bit daunting because, you know, we came in with a zero market share, and we pretty quickly realized there was only a subset of blocks we could win if we only have access to mempool transactions. And so then that started unraveling into okay, so what kind of other transactions do they exist? What are the sources? What are the counterparties? And, you know, you start thinking more service oriented and targeting specific counterparties.  
**CN:** **Kubi：** 最开始的时候，我们纯粹把它当成一个功能模块，一个可以无缝嵌入现有体系的即插即用组件：网络中有公开内存池（mempool），下游有 MEV 中继器（relay）；你从内存池中提取待打包交易，组装出候选区块并提交竞标，力求构建出价值最高的一个。但当你真正切入实操后，才会逐渐看清其中复杂的暗流。即便在当时，市场上也已经矗立着几大巨头构建者，例如 BeaverBuild 是当时最顶级的构建者之一，另一个是 rsync-builder，还有一个叫 Builder0x69，我记得它在当时一度是全网体量最大的构建者。这让人感到相当具有压迫感，因为我们入场时市场份额是零，而且我们很快意识到：如果仅仅依赖公开内存池里的交易，我们能赢下的区块极其有限。于是我们开始抽丝剥茧地探索：市场上还存在哪些非公开交易？它们的源头在哪里？交易对手方都是谁？从那时起，你必须全面建立以服务为导向的思维，专门针对不同的细分对手方去量身定制解决方案。

### [00:09:47 - 00:10:14]
**EN:** I remember at MapParis, you specifically talked about, you know, this idea of being a neutral builder, and wanting to basically provide extra additional services to searchers, applications that might need special ordering services, in particular, and not necessarily doing your own trading, as some of the other block builders had done at the time. Is that still true today? Do you guys just really focus on the service oriented aspect?  
**CN:** **主持人：** 我记得在 MEV Paris 会议上，你特别强调过打造“中立构建者”（neutral builder）的理念，也就是专注于为搜索者（searchers）以及对交易排序有特定定制需求的应用提供额外的增值服务，坚决不下场做自营交易——而当时市场上其他一些区块构建者是兼做自营交易的。这个原则在今天依然成立吗？你们如今是否依然全心全意专注于服务导向？

### [00:10:14 - 00:10:46]
**EN:** Yeah, 100%. And I think this is also fundamentally what gives us a big edge, not just in terms of conflict of interest, which initially it mainly was, but also just in terms of company culture and focus. Because if you're trading from the DNA of the kind of people who are very, very good of trading, it's different to people who are more service oriented, and building something for a customer. And it's very tough to get the balance right.  
**CN:** **Kubi：** 是的，百分之百依然如此。我认为这从根本上铸就了我们巨大的竞争优势。这不仅体现在彻底规避了潜在的利益冲突（起初客户最在意的确实是这一点），更深层的原因在于团队文化与战略专注度。因为顶尖交易员的基因和那些以客户为导向、全心全意为客户打磨产品的工程团队完全不同。一家公司极其难以在自营交易与中立客户服务之间保持长久平衡。

### [00:10:46 - 00:11:33]
**EN:** Did you guys ever think, like, especially as you're seeing all these PerpDexes pop up, and then the success of Hyperliquid and Lighter, at any point did you guys ever think about, maybe we should like use our trading knowledge and get into Perps? I think everyone who's an infrastructure player that is looking at the success and insane amount of revenues these guys make, have probably, has probably thought about it. So answer is we've definitely thought about it. But I think we've always tried to stay very disciplined about what we are really good at, and try to expand scope, but always keeping the scope close enough to make sure we don't dilute focus. And so it was very tempting. But yes, we stayed disciplined and are still building.  
**CN:** **主持人：** 你们内部有没有动摇过？尤其是眼看着各类链上永续合约 DEX（Perp DEX）如雨后春笋般爆发，见证了 Hyperliquid 和 Lighter 取得如此巨大的成功，你们有没有在某个时刻想过：“也许我们应该利用自己深厚的交易知识积累，亲自下场去做永续合约”？  
**Kubi：** 我想任何一个基础设施领域的从业者，在目睹这些项目取得的现象级成功以及令人咋舌的庞大营收时，心里恐怕都闪过这个念头。所以坦白讲：我们确实认真考虑过。但我认为我们始终保持着极强的战略克制，清醒地聚焦于自己最擅长的领域；即便我们也在适度拓宽业务边界，但始终牢牢锁定在核心能力圈周围，确保绝不稀释专注度。诱惑确实巨大，但我们守住了纪律，至今依然专注于把基础设施深耕到底。

### [00:11:33 - 00:12:40]
**EN:** That totally makes sense. And it clearly, it has worked out well for you. I mean, every day I get a flash and I look at top builders, and it looks like almost every day Titan is the number one block builder between anywhere 45% to maybe 60% on a given day, depending on various different transactions and orders that come through. And also you have the Titan Relay, which is not vertically integrated. It's just a separate entity, if I understand correctly. But that's also up there in the leaderboards as well. What has been the edge for you guys in maintaining that number one spot? There's like a lot of rotation at number two right now. You see Quasar, some days it will be up at 25%. Other days may be down to 15%. Sometimes you'll see Buildernet with Nether Mind and Flashbots jump up. I'm not sure if B4Build is still participating or not. But the point is that you guys have been clearly at number one. What's kind of been that edge? I mean, is it just this philosophy that you've had about building out services? And then as a result of providing excellent customer service, you've been able to just generate more business on top of that and kind of leverage your network? Or is there something specific that you guys are doing some kind of edge?  
**CN:** **主持人：** 这完全说得通，而且事实证明这种专注换来了巨大的回报。我每天打开仪表盘查看顶级构建者数据时，几乎每一天 Titan 都稳坐全网第一大区块构建者的宝座，根据当天全网交易与订单流的情况，市场份额稳定在 45% 到 60% 之间。此外，你们还运营着 Titan Relay（MEV 中继器），据我理解它在业务上并没有与构建者进行纵向一体化绑定，而是一个独立的实体，但在中继器排行榜上也名列前茅。你们能够持续捍卫全网第一的核心壁垒究竟是什么？目前行业第二名的位置轮换非常剧烈：比如 Quasar 有时能冲到 25%，有时又跌到 15%；有时你能看到 Nethermind 与 Flashbots 联手打造的 Buildernet 份额跃升；我不太确定 BeaverBuild 目前是否还在全力竞逐。但核心事实是，你们始终稳居榜首。这种核心优势究竟是什么？仅仅是因为你们坚持打造中立服务的理念，并通过卓越的客户服务带来飞轮效应、拓展了人脉网络？还是说你们在底层算法或策略上拥有某种不为人知的硬核壁垒？

### [00:12:40 - 00:13:54]
**EN:** Yeah, so our obsessions are one, technical excellence. And then two, obsessing about the service we provide to the counterparty. And they feed into each other. So it is very true that if you don't have any transaction access or order flow access, that you don't have the building blocks to build blocks. But at the same time, if you have enough reputation over time, generally, you will speak to the same customers, the space of the ecosystem is not that large. And the differentiator very much is how good is the quality of your service? And then how responsive are you? And that's not just in terms of like someone having a request, debug something, something went wrong, but actually also shipping new features consistently improving. It's the cliche stuff on a high level. But then obviously, you go into the detail, there are specifics in this industry. But that is essentially our edge. And they're self-reinforcing because the faster you are at processing transactions and better services and sequencing, the better being able to include more transactions, then the better you are as a counterparty for people who want to land their transactions on chain.  
**CN:** **Kubi：** 是的，我们的核心执念归结为两条：第一是追求极限的技术卓越（technical excellence），第二是对我们向对手方所提供服务的极致关注。这两者是互为滋养、相互强化的。毋庸置疑，如果你缺乏交易入口或专属订单流（order flow）的接入通道，你就根本没有原材料来构建高价值区块。但与此同时，随着时间的推移与行业声誉的沉淀，大家接触的客户群体基本上是重合的——毕竟这个生态圈子的体量并不算庞大。此时真正的分水岭就在于：你的服务品质究竟有多硬？你的响应速度有多快？这种响应绝不仅限于某人提出技术支持、需要排查故障或系统报错时的反馈，更体现在持续交付新特性与稳定迭代系统的工程能力上。在宏观层面听起来似乎是老生常谈，但一旦切入微观细节，这个行业有着极其苛刻的特定工程挑战。这本质上构成了我们的护城河，而且它具备强烈的自我强化效应：你在交易处理速度、个性化服务和交易排序（sequencing）能力上越出色，打包容纳高价值交易的能力就越强；反过来，对于那些希望将交易捆绑包（bundles）精准上链的机构而言，你自然就成为了无可替代的最优对手方。

### [00:13:54 - 00:14:31]
**EN:** And it has a bit of a flywheel effect. Yeah, this actually, I was going to ask you this question about, you know, is order flow king or not? And it seems like you just answered that by saying something actually quite nuanced, which is the trust and reputation that you guys have built up over the years has allowed you to expand your business. And as a result of that, when you're going to potential customers or people see on the market, they have a need, there's like a natural potential answer for them, which is work with Titan to get this product built or get this thing shipped. How has that been the inbound for you versus maybe the outbound like at the start?  
**CN:** **主持人：** 这确实形成了一种强劲的飞轮效应。事实上，我原本正想向你抛出这个经典问题：“订单流是否真的为王（Is order flow king）？”但你刚才的回答极其细致且富有洞察：你们多年来建立的信任与声誉让业务得以不断扩张；因此，无论当你们接触潜在客户，还是市场上的机构产生特定需求时，他们脑海中自然浮现的答案就是——找 Titan 合作来开发这款产品或落地交付这项功能。那么，如今主动找上门（inbound）的合作与最初全靠你们向外开拓（outbound）相比，发生了怎样的变化？

### [00:14:31 - 00:15:45]
**EN:** So initially, it was all outbound. So no one really knew about us. But over the years naturally, as we came over a household name, there has been more inbound. But I must say, especially in the Ethereum community, there's always this myth of being super good at BD. And there's some truth to that. But like we basically hired our first BD person at the beginning of this year. That was our first non-technical hire, everyone else is an engineer. And so, you know, just being good, technically, being part of the community, like interacting with people, it was a lot more organic. So still a bit outbound. But then you are at the, you know, events at the workshops and you meet people and their requests and people are always thinking about improving the ecosystems or shipping new things. And if you're around and you're supportive and you're open enough to also explore things and not only do things that are a sure thing that is going to make you money, because some things will work out and some will be a waste of resources. I think that has just in the way of the way we approach it has been like a key factor. So I guess to answer your question, how much inbound versus outbound? It's more inbound these days than it used to, but it's also not that people are knocking on door all the time either.  
**CN:** **Kubi：** 刚起步时完全是纯粹的向外拓展（outbound），因为当时根本没人知道我们是谁。但这些年随着 Titan 逐渐成长为行业内的标杆品牌，主动找上门的合作确实越来越多。但我必须说明的是，尤其在以太坊社区内部，外界总是对所谓的“顶级商务拓展（BD）能力”有一种迷信。这固然有一定道理，但实际上我们直到今年初才招聘了第一名专职 BD 员工。那是我们团队第一个非技术岗位的成员，在此之前团队全员都是核心工程师。所以，依靠扎实过硬的技术实力、深度扎根社区、与同行真诚交流，业务增长其实要自然有机得多。虽然我们目前依然会主动向外沟通，但更多的是在各类线下峰会、研讨会现场与人面对面探讨实际需求。社区里的人总是在思考如何提升生态性能、落地全新机制；如果你始终在场、乐于提供技术支持，并且心态足够开放，愿意去前瞻性地探索新事物，而不是唯利是图只做稳赚不赔的事（毕竟有些探索会成功，有些探索注定会耗费资源）——我认为正是这种务实的做事风格，成为了推动我们发展的关键基石。所以回答你的问题：主动找上门的咨询确实比以往多了很多，但也绝非到了各路客户随时都在踏破门槛的地步。

### [00:15:45 - 00:16:36]
**EN:** Yeah, I've definitely seen you spend a lot of your personal time going to many of these events having like conversations with pretty much anybody who has something valuable to talk about with you, or just has questions. And I wonder, like, you also have this aura about you, that's very calm and open minded. And like, I wonder just how you maintain that, given that you have all this pressure every day of this business that you're running 24/7, it never stops. But you're still out there in the community with a lot of your personal time as a founder, having these conversations on the ground, why did you decide like this is important? And then maybe can you speak on like, what are some things that have helped you just kind of like maintain your balance and just making sure that you get out there and have those conversations even maybe like when you don't feel like going to the event, or maybe you have other things to press on.  
**CN:** **主持人：** 是的，我确实经常看到你投入大量的个人时间奔波于各大行业活动之间，几乎只要有人带着真知灼见想和你探讨，或者单纯有问题向你请教，你都会坐下来深入交流。而且我很佩服的是，你身上始终散发着一种极其从容、开明而沉稳的气场。我很好奇，面对这门 7x24 小时永不停歇、容不得半点差池的高压业务，你究竟是如何保持这种从容心态的？作为创始人，你依然在一线投入海量时间与社区面对面交流，你为什么觉得这至关重要？又有哪些习惯或心得帮助你在身心俱疲、或者手头事务堆积如山时，依然能走出办公室、亲临现场去展开那些深度对话？

### [00:16:36 - 00:17:10]
**EN:** I think just like most of us who have been in the space for a while, there's just this intrinsic interests and what we're actually building and what the whole like movement or the industry is trying to achieve. And especially Ethereum itself, which is obviously in terms of like the values we have as an ecosystem, like ahead of anyone else really. And so I think it's it's a mix of like genuine interests and you know, working on something that I like personally on and we as a team we like, and then building a business around that.  
**CN:** **Kubi：** 我想就像我们许多在这个行业深耕已久的人一样，内心深处始终对我们亲手构建的技术、对整个去中心化运动以及行业试图达成的远大愿景，保有一种纯粹且发自内心的热爱与好奇。尤其是以太坊本身，就整个生态所坚守的核心价值观而言，无疑走在所有人的最前沿。因此我认为，这是纯粹的求知欲、投身于我和团队发自内心认可的事业，并围绕着这一核心技术去构筑可持续商业模式的完美结合。

### [00:17:10 - 00:18:05]
**EN:** So there's quite a bit of overlap, just personal curiosity, like genuine interest and also wanting to move the industry forward. And then there's also the want to just be aware of what's going on, what people are thinking about, what are the new ideas, where things are going and especially in crypto, as you know, you know, things move really, really quickly compared to other industries. And so I think that is one of the drivers that just you know, just naturally makes me want to be out there. But absolutely, as you know, initially we didn't have as much responsibility, the team was much smaller. And so operationally it was much easier to be who we were and have more capacity to do these things. And so that's definitely more of a challenge now. And as you very well know in crypto, there are so many conferences as well. I feel like there's a conference every week throughout the year.  
**CN:** **Kubi：** 所以这里面有很多交集：个人的求知欲、由衷的兴趣，以及推动整个行业向前迈进的责任感。此外，我也渴望随时洞察前沿风向——大家在思考什么、涌现了哪些新想法、未来的叙事与技术会走向何方；尤其在加密领域，正如你所清楚的，其迭代演进的速度远超其他任何传统行业。我认为正是这种内在驱动力，让我自然而然地渴望亲临一线。不过确实如你所说，在创立初期我们的责任没现在这么重，团队规模也小得多，在运营层面上亲力亲为跑活动要轻松很多。现在随着业务规模成倍扩大，精力分配无疑面临着更大的挑战。而且大家都心知肚明，加密行业的峰会实在太密集了，感觉一年四季每周都有大会。

### [00:18:05 - 00:19:05]
**EN:** And so I think also being just selective about the events we go to, and we tend to generally go to the more technical Ethereum focused ones versus the more commercial ones. And so that makes it also more tolerable, especially as a technical person. Definitely. I've seen you at many of the MEV events and different pre-confirmation events and such. Recently, you guys put together a block space forum event, which I might ask you about later. But before I do that, I want to ask you, so like, let's say that you're you go to one of these events, you have great conversation, or there's an idea that emerges like something like maybe a commit boost or something like pre-confirmations or even this prop AMM idea. How do you guys internally sit down and decide, okay, let's allocate, you know, x time for prototyping this product for a customer and then, you know, having a conversation with them. What does that look like between going from like product ideation and prototyping to actually committing to the service and delivering it maybe on a consistent basis?  
**CN:** **Kubi：** 因此，我们必须对参会行程保持高度克制与筛选。相比于泛商业化的会议，我们通常更倾向于参加专注于以太坊底层技术的研讨会，这对于工程师背景的我来说，在体验上也更容易适应和享受。  
**主持人：** 确实如此。我在众多 MEV 闭门峰会和围绕预确认（pre-confirmations）的技术活动上都看到过你的身影。最近你们还主办了区块空间论坛（Block Space Forum），稍后我也会详细问问你。但在那之前我想先请教：假设你在参加某场活动时聊得非常投机，或者碰撞出了一个全新的技术灵感——比如类似于 Commit-Boost、基于 L1 的预确认机制，亦或是专有做市 AMM（PropAMM）这样的设想。你们团队内部是如何坐下来评估，并决定“好，让我们为这位客户分配 X 的研发时间来搭建产品原型，然后再找他们进一步对接”的？从最初的产品构思与原型验证，到最终正式承诺上线并保持稳定交付，你们的内部决策与研发流程是怎样的？

### [00:19:05 - 00:19:57]
**EN:** Yeah, so we tend to optimize for things that can have an impact and that can ship quickly. And so I think because the ecosystem itself has a lot of researchers that think more long term and more deeply and also more abstractly and theoretically, because we have an engineering background that is more like build something focused. And so if there's something where we feel we have a technical capability and we can ship something that's useful in a relatively short timeframe, that usually also gets a lot of excitement internally, whether it's just something that's open source, and then we put it out there or whether it's a product.  
**CN:** **Kubi：** 是的，我们的核心策略是优先选择那些能够产生实质性产业影响、并且能够迅速交付上线的项目。这是因为以太坊生态本身聚集了大量杰出的研究人员，他们往往偏向于进行更加长远、宏大、抽象且偏理论的研究；而我们团队拥有深厚的工程实战基因，更加专注于“动手造出实物”。因此，只要某些方向与我们的技术长项高度匹配，并且我们有把握在相对较短的时间窗口内交付出切实有用的成果，通常就会在团队内部激发极大的研发热情——无论它最终是作为一个开源公共产品发布到社区，还是打造成一项商业化服务。

### [00:19:57 - 00:20:00]
**EN:** And so I think that generally  
**CN:** **Kubi：** 所以我想，总体而言……


---

## Part 2 (00:20:00 - 00:40:00)

# 《Credible Commitments》访谈实录（Part 2）：以太坊区块构建与专有做市 AMM（PropAMM）深度解析

**嘉宾**：Kubi Mensah（Titan Builder / Gattaca 联合创始人兼 CEO）  
**主题**：以太坊区块构建（Block Building）与专有做市 AMM（PropAMM）微观结构  
**对话节点**：00:20:00 - 00:40:00  

---

### 内容概要（Executive Summary）
本部分访谈深入探讨了 Gattaca / Titan 团队的敏捷开发哲学以及专有做市 AMM（PropAMM）在以太坊生态的诞生与演进：
1. **工程哲学与客户群演化**：团队坚持 1~2 周的高速交付周期，在早期 MVP 的粗糙度与当前基础设施级服务的高可用性（uptime）之间取得平衡。客户从最初专注区块顶部（top-of-block）的 CEX-DEX 套利者拓展至 DApp、RPC 服务商与专业做市商。
2. **PropAMM 的微观结构革新**：传统 AMM 依赖恒定函数做被动流动性，面临严重的 LVR（损失与再平衡）和报价滞后问题。Solana 上的 PropAMM 证明了链上做市商主动报价与更优价差（Tighter Spreads）的可行性。
3. **以太坊上的 PropAMM 架构**：针对以太坊 12 秒离散槽（Slot）的特点，做市商将报价流式传输给区块构建者，构建者通过在区块顶部确保预言机更新交易优先执行，赋予做市商类似传统金融中的“最后审视权”（Last Look），免受有毒订单流（toxic flow）剥削，同时为普通用户提供优于 Uniswap v3 约 6.5 bps 的执行价差。
4. **宏观愿景与资产流通速度（Velocity）**：以太坊沉淀了巨量高价值资产，但链上周转活跃度不足。PropAMM 有望将价格发现（Price Discovery）拉回链上，大幅提升 ETH 资产的流动速度与经济密度。
5. **前沿扩展机制——ACE 与防作弊（Spoofing）**：针对预言机更新的 Gas 成本，通过应用控制执行（ACE, Application-Controlled Execution），做市商可实现“仅在区块内有真实匹配对手盘时才上链更新”，并可自定义交易规模门槛；同时构建者可在出块层直接惩戒恶意虚假挂单（Spoofing），保障市场稳健。

---

### [00:20:00 - 00:20:32]
**EN:** It's the right balance for us, so it's then not a thing where we have to go through this massive roadmap, what are all the priorities, right? It's something more pragmatic. In the next two weeks, we can do something that's meaningful, let's just do it. And that's actually also something, how we went about propAMMs, which I'm sure we're gonna talk about. Did that change for you when you guys were maybe like first started off building and maybe you didn't have as many customers, so the ideation cycles were a little bit longer.  
**CN:** **Kubi：** 对我们来说，这是最恰当的平衡点。这样我们就不用去纠结那种庞大繁杂的长期路线图、反复权衡所有的优先级。这是一种更务实的方式：如果未来两周内我们能做出一件有实质意义的事情，那就立刻动手做。事实上，我们当初涉足专有做市 AMM（PropAMM）也是遵循这个思路，我相信我们稍后会聊到这个话题。  
**主持人：** 在你们刚开始创业构建产品的时候，情况会有所不同吗？那时你们可能还没有那么多客户，构思和调研周期会不会更长一些？

### [00:20:32 - 00:20:59]
**EN:** You're willing maybe to entertain some more of the theoretical or like from the start, have you guys just been two week cycle? I've talked to some different builders and I've also tried different things as well. And everybody kind of has a different flavor, but what I have found from people who are successful in the space in Ethereum land is they have told me that this two week cycle or this one week product cycle seems to be something that is sticky, that works for them and allows them to ship at a fast cadence.  
**CN:** **主持人：** 你们那时会不会更愿意探讨一些理论层面的东西，还是说从一开始你们就一直保持两周一个周期的节奏？我和一些不同的区块构建者（Builders）交流过，我自己也尝试过各种模式。大家各有各的风格，但据我在以太坊领域观察到的成功团队来看，他们都告诉我，这种两周甚至一周的产品迭代周期似乎非常有效且具黏性，不仅行之有效，还能让他们保持极高的交付节奏。

### [00:20:59 - 00:21:30]
**EN:** I would say that has always been a thing and I would say that had more been a thing even early on because again, you can afford to also be scruffier because there's always a tradeoff between building something that's perfect versus building an MVP that's just good enough. And the larger you are as a company or an infrastructure provider, the more the services that people actually use obviously have to maintain uptime and so on and so forth, so you can't afford to be a scruffy.  
**CN:** **Kubi：** 我觉得我们一直都是这种节奏，甚至早期更是如此。因为在早期，你可以容忍做事更粗糙一些（scruffier），在追求尽善尽美与打造一个“刚刚够用”的 MVP（最小可行产品）之间永远存在权衡。而当你的公司或作为基础设施提供商的规模变得越来越大，用户实际在使用的服务显然必须保证高可用性（uptime）等等，你就再也无法承受粗制滥造了。

### [00:21:30 - 00:22:02]
**EN:** So I would say in the early days we were actually even more scruffier and now we try to balance it with being able to have enough uptime and redundancy and robust services. Who are some of your biggest customers today? Are they mainly MEV searchers, trading shops that are trying to land positions in a block? Are you also getting some order flow from applications or maybe RPC providers, et cetera? Yeah. So across all of those, I would say.  
**CN:** **Kubi：** 所以我想说，早期我们其实更为粗糙随性，而现在我们则努力在敏捷迭代与确保足够的服务正常运行时间、冗余备份以及健壮可靠的服务之间取得平衡。  
**主持人：** 如今你们最大的客户主要是哪些群体？他们主要是 MEV 搜索者（Searchers）、试图在区块中抢占交易位置的交易机构吗？你们是否也从去中心化应用（DApps）或 RPC 提供商等渠道获取订单流？  
**Kubi：** 是的，可以说涵盖了所有这些群体。

### [00:22:02 - 00:22:37]
**EN:** So initially when we first started, the core focus was searchers and there was both atomic and statistical arbitrage type searchers, so the sextex guys, but now we have just brought in the horizon to basically anyone that is consumer of block space and usually we don't go all the way to retail, so we don't have any retail facing services, but we work with either counterparties that then serve retail or large water flow generators themselves, like for example, the market makers.  
**CN:** **Kubi：** 刚起步的时候，我们的核心重点确实是搜索者（Searchers），包括原子套利（atomic arbitrage）和统计套利（statistical arbitrage）类的搜索者，也就是那些做中心化与去中心化交易所（CEX-DEX）套利的团队。但现在，我们已经把视野拓展到了几乎所有的区块空间消费者。通常我们不会直接触达终端散户，也就是说我们没有面向散户的业务，而是与那些服务散户的交易对手方合作，或者是那些体量庞大的订单流（order flow，转录误作 water flow）生成方本身，比如做市商（Market Makers）。

### [00:22:37 - 00:23:13]
**EN:** That sets the stage really nicely for us to actually get into Prop AMMs. So Prop AMMs sprouted up on Solana, I'd say maybe in Q3, 2025, and they became very popular as a way to integrate into these different DEX aggregators, Jupyter in particular, and a lot of flow started to go through them, and the nice thing about it was there's this song chain contract, market maker could update it with a new quote basically in real time and they have their own custom pricing curve, and it allowed them to be competitive in a way that maybe they couldn't have been competitive for flow previously.  
**CN:** **主持人：** 这正好为我们深入探讨专有做市 AMM（PropAMM）做好了极佳的铺垫。PropAMM 最初是在 Solana 上兴起的，大概是在 2024 年第三季度左右（转录为 2025 年 Q3），它们作为接入各类 DEX 聚合器（尤其是 Jupiter，转录为 Jupyter）的一种方式变得极为流行。大量订单流开始经由它们路由。其精妙之处在于存在一个链上合约（转录误作 song chain contract），做市商基本上可以实时用最新的报价去更新它，并且拥有自己定制的定价曲线，这使得他们能以一种以往无法企及的竞争力去争夺订单流。

### [00:23:13 - 00:23:35]
**EN:** Ethereum is a little bit different where you have block types that update every 12 seconds, so we have a different architecture here for Ethereum where market makers are streaming quotes to a block builder, and block builder is giving priority on updating their contract, making sure that that Oracle update is the first thing to touch that contract, first transaction, and then the other trade subsequently will touch it.  
**CN:** **主持人：** 以太坊的情况则略有不同，因为它的出块时间是每 12 秒一个区块。因此我们在以太坊上采用了不同的架构：做市商将报价实时流式传输给区块构建者（Block Builder），而构建者会给予其合约更新最高优先级，确保该预言机更新（Oracle update）是触达该合约的第一笔交易（即在区块顶部执行），随后其他交易才会相继触达它。

### [00:23:35 - 00:24:00]
**EN:** The nice thing about this is that it basically gives market makers a last look so they don't necessarily have to deal with toxic flow, and at the same time it allows for users potentially to get significantly better spreads than they would have gotten even trading in an RFQ system or against any other on-chain liquidity for some major pairs like ETH/USDC, ETH/USDT, and in the future more assets as well.  
**CN:** **主持人：** 这种机制的妙处在于，它基本上赋予了做市商“最后审视权”（Last Look），使他们不必被迫承受有毒订单流（toxic flow）。与此同时，在 ETH/USDC、ETH/USDT 等主流交易对上（未来还将扩展到更多资产），它还能让普通用户获得比以往在询价系统（RFQ）甚至任何其他链上流动性池中交易时显著更优的买卖价差（Spreads）。

### [00:24:00 - 00:24:33]
**EN:** Maybe if you could just give like a really high level, what is a prop AMM, and how did you guys come to the determination that this was the right microstructure to go forward with and to bring better pricing on some of these major pairs to Ethereum? An AMM is on-chain contracts that has constant function that determines based on relative liquidity of like a base and a quote asset what the price of a swap is depending on the amount of the swap.  
**CN:** **主持人：** 你能否从宏观层面为我们科普一下：究竟什么是专有做市 AMM（PropAMM）？你们团队又是如何判定这种市场微观结构（microstructure）是以太坊推进并为这些主流资产对带来更优定价的正确方向的？  
**Kubi：** 传统的 AMM 是一种链上合约，它拥有一个恒定函数，根据基础资产与计价资产之间的相对流动性，以及具体的兑换数量来决定兑换价格。

### [00:24:33 - 00:25:05]
**EN:** The problem with that is, well, it's good because you can have passive liquidity and all the benefits, so I'm not going to go into too much detail there, and it has been amazing to bootstrap this DeFi ecosystem and actually like amazing innovation. I think even with prop AMMs, there's a world and a place for AMMs, but they have this fundamental problem that you can't actually quote competitively, and the real world or the continuous world happens.  
**CN:** **Kubi：** 但这种机制的问题在于——当然，它的优点是允许被动流动性存在等等，这里我就不过多展开细节了，它在冷启动整个 DeFi 生态方面功不可没，确实是一项了不起的创新。即便有了 PropAMM，我认为传统 AMM 依然有其生存空间和地位。但传统 AMM 存在一个根本性问题：你无法进行具有竞争力的主动报价，而在现实世界或连续时间世界里，外部真实价格是在持续变动的。

### [00:25:05 - 00:25:51]
**EN:** Then as we talked about at the beginning, you have this discrete time winners during which you get to update the price if you wanted to. The innovation really with prop AMMs is that you still have an on-chain contract, which is really important, but you give a sophisticated entity like a market maker the ability to actually determine what the price of whatever liquidity you're willing to offer is. The biggest benefit really here is tighter spreads, and tighter spread means better prices, and so it incentivizes users not to just use centralized venues, but actually on-chain venues and so on and so forth.  
**CN:** **Kubi：** 然后正如我们一开始所讨论的，区块链存在离散的时间窗口（discrete time windows，转录误作 winners），你只能在这些窗口期内选择是否更新价格。PropAMM 真正的创新在于：它依然保留了链上合约（这一点至关重要），但赋予了像做市商这样高度成熟的专业实体自主决定其所提供流动性价格的能力。这带来的最大好处就是极窄的买卖价差（tighter spreads），更窄的价差意味着更优的成交价格，从而激励用户不再局限于中心化交易所，而是真正转向使用链上交易场所。

### [00:25:51 - 00:26:23]
**EN:** So that's broad strokes on how we think about prop AMMs. Yeah, you guys put together this really nice dashboard. The nice thing you can see on the dashboard is you can look at some existing prop AMMs and see what the spread difference is between that prop AMM and what you would get trading on UniV3. Right now, if I'm looking at BAP AMM, for example, you're looking at about a six and a half basis point improvement on execution versus what UniV3 is offering currently right now just as an idea for the listener.  
**CN:** **Kubi：** 以上就是我们对 PropAMM 的大致理解和思考。  
**主持人：** 是的，你们制作了一个非常出色的数据看板（Dashboard）。在这个看板上，最棒的一点就是可以查看现有的一些 PropAMM，并对比它们与 Uniswap v3 之间的价差差异。例如，如果我现在看 Bebop AMM（转录误作 BAP AMM），给听众一个直观概念，它的执行价格比目前 Uniswap v3 提供的报价优化了大约 6.5 个基点（bps）。

### [00:26:23 - 00:26:45]
**EN:** How much research did you guys do? I know you did some research, but how much time did you spend on the research to validate the idea before you said, "All right, engineering resources, two weeks, we're good to go." Yeah, so I think everyone on the team has been or had been looking at prop AMMs on Solana and just seeing what it had done for the market.  
**CN:** **主持人：** 你们做了多少前期调研？我知道你们做了一些研究，但在拍板决定“好了，调配工程资源，两周搞定上线”之前，你们花了多长时间验证这个想法？  
**Kubi：** 是这样的，我们团队的每个人其实一直都在关注 Solana 上的 PropAMM，观察它给整个市场带来了怎样的改变。

### [00:26:45 - 00:27:17]
**EN:** So seeing how much volume migrated from AMMs to prop AMMs on Solana, which was great validation. And then when we started seeing that the spreads were tighter than centralized venues for some pairs, especially Sol-USZ, actually not just was the spread tighter, but also sometimes it looked like price discovery started to happen on some of these prop AMMs on Solana versus on-chain venues, off-chain venues like centralized exchanges.  
**CN:** **Kubi：** 我们看到在 Solana 上有大量交易量从传统 AMM 迁移到了 PropAMM，这是一个极其有力的验证。接着，我们开始发现某些交易对的价差甚至比中心化交易所还要窄，尤其是 SOL/USDC（转录误作 Sol-USZ）。事实上，不仅价差更窄，而且某些时候，价格发现似乎已经开始发生在 Solana 的这些 PropAMM 上，而不是像以前那样完全由中心化交易所等链下场所主导。

### [00:27:17 - 00:27:46]
**EN:** That just was such a, I think that was a realization of what is possible on-chain. And everyone in crypto, I think, was talking about it. And within the Ethereum community, there was this assumption that this will never be possible in Ethereum just because Ethereum is too slow, right? And then it was, it's going to be, even if it was possible, it's too expensive to update it every block and so on and so forth, right?  
**CN:** **Kubi：** 这让人猛然意识到链上原来可以做到这种程度！当时加密圈里几乎每个人都在讨论这件事。而在以太坊社区内部，一直有一种固有假设，认为这在以太坊上绝不可能实现，因为以太坊太慢了，对吧？紧接着又有人说，即便技术上可行，每个区块都去更新一次预言机状态的成本也太高昂了，诸如此类。

### [00:27:46 - 00:28:14]
**EN:** And so we were just chatting internally and we actually started reasoning through some of these assumptions and we realized, "Wait a minute, look at gas prices today. Like what would an Oracle update cost?" And so on and so forth. So, okay, we can actually, this is not an issue anymore. And we just started working through all the assumptions that initially people had. And we realized actually this was possible.  
**CN:** **Kubi：** 于是我们在团队内部展开了讨论，逐一推演这些假设背后的逻辑。我们突然意识到：“等一下，看看现在的 Gas 价格，一次预言机状态更新到底要花多少成本？”推演之后我们发现，这根本不再是障碍了！我们梳理了大家最初抱有的所有刻板假设，最后确信这在以太坊上完全行得通。

### [00:28:14 - 00:28:46]
**EN:** And then I think one of the recent conferences, one of the recent Ethereum conferences, I don't know which one it was, we had also more and more people in the ecosystem starting to talk about it. We started sharing ideas and chatting to different people. And it seemed like this is something everyone could actually start picking up. And so when that realization was made, we actually talked, I think maybe less than half a day about it internally, like what would it actually take to build this into our existing infrastructure?  
**CN:** **Kubi：** 后来在最近的一次以太坊大会上（我记不清具体是哪一场了），生态里讨论这个方向的人也越来越多。我们开始与各方交流想法、深入探讨。看起来大家都意识到了这个方向并准备开始跟进。当意识到这一点后，我们在内部其实只花了不到半天时间讨论：“要把这个功能集成到我们现有的基础设施里，到底需要做哪些改造？”

### [00:28:46 - 00:29:17]
**EN:** And then we shipped it over a couple of days and that was that. And from that point onwards, the difficulty wasn't really the technical difficulty. The difficulty was more like getting people to actually start providing liquidity, right? And then takers starting to integrate and that's still happening and it's growing slowly. And obviously activity overall is also not the best in the current market environment. But it's slowly actually starting to get adoption more and more.  
**CN:** **Kubi：** 然后我们只用了几天时间就将它构建并发布上线了。从那时起，真正的难点就不再是技术本身了，真正的挑战在于如何推动大家真正开始在上面提供流动性，以及促使吃单方（Takers / 聚合器）开始接入集成。这个过程目前仍在推进，并且正在缓慢增长。显然，在当前的市场环境下整体活跃度不算太高，但它的采用率确实在逐步扩大。

### [00:29:17 - 00:29:46]
**EN:** Yeah, so the technical problem actually was really, really simple to solve. I think it was much harder, or it still is, just to scale the adoption across the ecosystem. Are you finding that you guys have to go to every market maker, have one-on-one conversation, kind of get them to buy in to start participating? Or are you focused more on talking to applications who want to work with market makers to get them to buy in? It's a mix of both.  
**CN:** **Kubi：** 是的，所以技术问题其实非常非常容易解决。我认为过去和现在更难的，是如何在整个生态系统中推动规模化采用。  
**主持人：** 你觉得你们是必须逐一拜访每一家做市商、进行一对一沟通以说服他们加入，还是说你们更多是在与那些希望与做市商合作的应用方（DApps）沟通，借此促成做市商的参与？  
**Kubi：** 两者兼而有之。

### [00:29:46 - 00:30:17]
**EN:** And so the other thing actually that we also realized, because as we talked about earlier in the conversation, we worked with counterparties that had very low latency execution requirements. So they're sextex guys and they always want to land at the top of the block. And from a latency perspective, that's the hardest place to be because everything else has to execute after. So often any state transition that happens at the top of the block then affects something later, right?  
**CN:** **Kubi：** 另一件我们意识到的事情是，正如我们在前半部分谈到的，我们一直都在与对执行延迟要求极低的交易对手方打交道。他们是做中心化与去中心化交易所（CEX-DEX）套利的团队，总想把交易打包在区块顶部执行（top-of-block）。而从延迟和构建的角度来看，区块顶部是最难处理的位置，因为区块内的其他所有交易都必须排在它之后执行。通常区块顶部发生的任何状态变迁（state transition），都会直接影响后续所有交易的执行状态。

### [00:30:17 - 00:30:46]
**EN:** And so we have spent a lot of resources optimizing that part of the block construction process. And so this is another reason why it only took us two days to ship this, is because we had all of these pipelines built in already. So instead of processing someone's transaction, slotting in top of blocking and executing everything else, we can do something very similar with Oracle updates. There just needs to be some difference in terms of scoring because obviously they're not paying the most priority fees.  
**CN:** **Kubi：** 因此，我们在优化区块构建管线中涉及区块顶部执行的这一环节上投入了海量资源。这也是为什么我们只用了两天就能上线这一系统的原因——因为底层的所有处理管线（pipelines）早已齐备。原本我们是将某位套利者的交易插入到区块顶部执行（top-of-block，转录作 top of blocking）然后再执行其余交易，现在我们对预言机更新（Oracle updates）采取完全相同的逻辑即可。唯一的区别在于排序评分机制（scoring），因为做市商显然不会支付极高的优先费（priority fees）。

### [00:30:46 - 00:31:17]
**EN:** So there's some custom logic there. So then going back to your question, because we have had to have worked with some of these market makers because they send transactions and then sometimes it's too slow or they send a cancellation and then it's still in a block that is submitted to Relay, so I've built a lot of optimized infrastructure. So we already had some relationships, so it was easier to go back to them and say, "Hey, this is very similar to what you're already doing and we can do something that's actually super competitive for the entire ecosystem."  
**CN:** **Kubi：** 因此这里需要一些定制逻辑。回到你的问题：因为我们之前就已经与这些做市商紧密合作过（比如他们发送交易时偶尔太慢，或者发送了撤单指令却依然被包含在提交给中继 Relay 的区块中，为此我们构建了大量高度优化的基础设施）。正因为我们已经建立了这种业务信任与合作关系，我们很容易回头找到他们说：“嘿，这跟你们平时做的事情非常相似，而且我们能一起做出对整个生态都极具竞争力的产品。”

### [00:31:17 - 00:31:48]
**EN:** And then at the same time, we also sometimes talk to RPC providers and aggregators. And so these are also the entities that do routing and other kinds of things. So again, we're easier to reach these people. And so we're having both these conversations and it's a chicken and egg thing. So it's quite important that both the liquidity and the takers scale more. And so it's a balance. What's the ultimate goal of bringing proper AMMs to Ethereum?  
**CN:** **Kubi：** 与此同时，我们有时也会与 RPC 提供商和 DEX 聚合器沟通。他们是负责交易路由等核心环节的主体，同样很容易触达。因此我们同时在推进这两方面的对话。这是一个“鸡生蛋还是蛋生鸡”的问题，流动性提供方与吃单方（takers）的同步扩张至关重要，必须在两者之间取得平衡。  
**主持人：** 将 PropAMM 引入以太坊的终极目标是什么？

### [00:31:48 - 00:32:33]
**EN:** To me, I guess one naive assumption that I would have is that you could potentially get price discovery for ETH asset on chain rather than price discovery happening on Binance or some other exchange in the future. Tell me about that intuition. How wrong is it? But then also, what's the ultimate goal here? Yeah, absolutely. So one of the more fundamental reasons that this is very important for Ethereum is that Ethereum is still by far the highest value chain in terms of the assets that are sitting on the chain, but far behind other chains that have maybe sometimes 10x less assets in terms of activity.  
**CN:** **主持人：** 就我个人而言，一个或许有些朴素的猜想是：未来我们有可能直接在链上完成 ETH 资产的价格发现（Price Discovery），而不是像现在这样由币安（Binance）或其他中心化交易所主导。这种直觉是否靠谱？如果不对，那么终极目标又是什么？  
**Kubi：** 完全切中要害！这对以太坊至关重要的一个根本原因在于：就链上沉淀的资产规模（TVL / 资本价值）而言，以太坊至今仍是无可匹敌的绝对第一；但在链上活跃度（交易频率与换手率）方面，却远远落后于那些资产体量可能只有以太坊十分之一的公链。

### [00:32:33 - 00:33:07]
**EN:** And so we have been thinking for a while about how can we increase the velocity of these assets on chain? This goes into conversations around shortening slot times and all of these things that have to be done in protocol and anything that happens in protocol also needs to be done very carefully because we need to make sure we don't sacrifice decentralization. And so when we looked at proper AMMs, that was a thing that would allow ETH to move a step closer to increasing velocity versus the other way around.  
**CN:** **Kubi：** 因此，我们长期以来一直在思考：如何提高这些链上资产的流通速度（Velocity）？这引出了诸如缩短以太坊槽时间（slot times）等讨论，但所有涉及协议层（in-protocol）的改动都必须极其谨慎，确保我们绝不牺牲去中心化。因此，当我们审视 PropAMM 时，发现它无需更改协议层，就能让以太坊在加速资产流通速度的道路上迈出坚实的一步。

### [00:33:07 - 00:33:37]
**EN:** The value of Ethereum as an ecosystem, as just a pure settlement layer, is going to be much lower than if you have the actual activity happening on chain. Proper AMMs has such a huge potential because there are so many counterparties that are comfortable keeping their assets on Ethereum. And so that was the most exciting thing more on a macro level from our perspective. That's awesome. More trading velocity is going to be beneficial for the ecosystem overall.  
**CN:** **Kubi：** 如果以太坊生态沦为纯粹的底层结算层（settlement layer），其生态价值将远低于真实经济活动直接发生在链上的价值。PropAMM 拥有如此巨大的潜力，正是因为有大量的机构和交易对手方非常放心地将资产长期存放在以太坊上。从我们的角度来看，在宏观层面上这才是最令人振奋的事情。  
**主持人：** 太棒了，更高的交易流通速度对整个生态无疑大有裨益。

### [00:33:37 - 00:34:06]
**EN:** What are, in your opinion, like some bogeyman or some downsides to watch out for for proper AMMs? Is it any kind of censorship, failed quotes, like maybe bad UX? What are some edge cases that could potentially come up? Yeah, so I think we've seen a bunch of them already on the other chains that launched them before Ethereum. So it's much easier for us to reason about them and try to minimize them as much as possible.  
**CN:** **主持人：** 在你看来，PropAMM 有哪些潜在隐患或负面风险需要警惕？会出现交易审查（censorship）、报价失效（failed quotes）或糟糕的用户体验吗？可能出现哪些极端边界情况（edge cases）？  
**Kubi：** 是的，在比以太坊更早推出该机制的其他公链上，我们已经见识过不少这类问题了。正因如此，我们更容易提前推演并尽可能去规避它们。

### [00:34:06 - 00:34:42]
**EN:** I think a big one is because you have 12 seconds block time on Ethereum versus on something like Solana, where it's more reliable to potentially make a decision based on the price you saw on the previous slots of block or shred on Solana, for example, versus on Ethereum. Because the update is so quick, is that correct? Because the delta between you being able to act on something that has happened been much smaller. So that was one big fundamental challenge.  
**CN:** **Kubi：** 我认为最大的挑战之一在于时间粒度差异：以太坊的出块时间是 12 秒，而在像 Solana 这样的链上，你可以基于前一个 slot 或 shred 中看到的价格更可靠地做出交易决策，以太坊则并非如此。  
**主持人：** 是因为 Solana 的更新速度极快，对吗？  
**Kubi：** 是的，因为在 Solana 上，事件发生与你对其做出反应之间的时间差（Delta）要小得多。因此在以太坊上，这是一个重大的根本性挑战。

### [00:34:42 - 00:35:18]
**EN:** So how do you actually let the takers know what the price is that they are going to get by swapping through a proper AMM? There are two ways to do this, one, which is the surest way to do it. And it's completely permissionless. And you don't have any trust guarantees by doing on chain routing. And doing on chain routing, well, maybe a query will cost you around 100 and 100 K gas, which I can't gas prices is tolerable as even more so now with scaling and all the next hard forks are going to make it even cheaper.  
**CN:** **Kubi：** 那么，你到底该如何让吃单方（Takers）预先知道通过 PropAMM 兑换能拿到什么价格？通常有两种途径：第一种是最稳妥可靠的方式，完全无许可，通过链上路由实现，不需要任何信任假设。进行链上路由查询大约会消耗 10 万（100k）左右的 Gas。在当前的 Gas 价格水平下（转录误作 I can't gas prices）这完全可以承受，尤其是随着扩容升级以及后续硬分叉的到来，执行成本还会进一步降低。

### [00:35:18 - 00:36:01]
**EN:** But it's obviously not the most optimized way. And the other way, which is a little bit more of a crutch is by basically the builders streaming the state updates that the market makers feed onto the proper AMMs. Some of the gotchas in the other ecosystem have been spoofing, so advertising one quote and then by the time it executes, right, which can be obviously avoided by routing on chain or the other thing is that builders also can police this, again, not the most robust solution, but it's actually something we, for example, on our blocks we enforce.  
**CN:** **Kubi：** 但这显然不是最极致的优化路径。第二种方式则略微依赖外力机制（crutch），即由区块构建者实时流式传输做市商注入 PropAMM 的状态更新。在其他生态中出现过的陷阱之一是虚假报价/虚假挂单（spoofing）——即做市商对外宣传一个优惠报价，但在实际执行时却变相抬价或撤回。通过链上路由自然可以杜绝这种情况；另一种防范手段是由区块构建者来实施监管（police），虽然这不是最根本的链上防线，但我们确实在自己的出块逻辑中严格执行了这一约束。

### [00:36:01 - 00:36:34]
**EN:** And so if we see, because it's very straightforward to see that as a pattern, we basically, we police that. These are sort of some gotchas, some crutches that, you know, we use to get around it. For sure. Would you analogize this or compare this to something in traditional finance markets called request for stream? Previously I was talking to another builder and they mentioned that this is kind of basically just an RFS system and we're reinventing it, rediscovering it in Ethereum.  
**CN:** **Kubi：** 如果我们监测到这种行为模式（识别起来其实非常直观），我们就会直接予以惩戒限制。这些就是我们用来解决某些隐患的应对手段。  
**主持人：** 明白。你会将这种机制类比为传统金融市场中的“连续报价请求”（Request for Stream, RFS）吗？此前我和另一位区块构建者交流过，他提到这本质上就是一个 RFS 系统，我们只不过是在以太坊上重新发明、重新发现了它。

### [00:36:34 - 00:37:02]
**EN:** Not exactly. I think this is something actually quite novel because it is a way for every liquidity provider to have some custom logic in how they want to express, how they want to provide liquidity based on who is swapping against them. So you can have whatever kind of logic in that property contract, right? So you can have any kind of order flow segmentation in that contract.  
**CN:** **Kubi：** 不完全一样。我认为这其实是一项相当新颖的创新，因为它允许每个流动性提供者根据“谁在与他们进行交易对手交易”来编写自定义逻辑，决定自己究竟如何表达流动性意图。你可以在这个 PropAMM 合约（转录误作 property contract）中嵌入任何自定义逻辑，对吧？也就是说，你可以在合约内实现任意形式的订单流细分（Order Flow Segmentation）。

### [00:37:02 - 00:37:30]
**EN:** So if you perceive that this is uninformed flow, you can quote it one way versus if this is toxic flow or something else you can quote it another way. Exactly, right. And it's completely permissionless, right? And because everything happens on one ledger and you have different proper AMMs existing on that ledger, anyone that routes across them basically has a view. It's something that is beyond an order book essentially, right?  
**CN:** **主持人：** 也就是说，如果你识别出这是无知情订单流（uninformed flow，散户流），你可以给出一个优惠报价；而如果这是有毒订单流（toxic flow）或套利流，你就可以给出另一种保护性报价。  
**Kubi：** 完全正确！而且它是完全无许可的。正因为所有事情都发生在同一本账本上，并且该账本上共存着不同的 PropAMM，任何跨池路由的实体基本上都能一览无余。从本质上讲，这已经超越了传统的订单簿（Order Book），对吧？

### [00:37:30 - 00:37:59]
**EN:** Because an order book is an expression of every market maker's view in terms of putting order at specific price levels with specific depth, but now you have a much more dynamic and expressive way of doing this holistically across the ledger, right? And so I think it's far more than just a request for stream and something actually truly novel and actually something that I'm really excited about. What other kind of innovations are available here?  
**CN:** **Kubi：** 因为订单簿仅仅是每位做市商在特定价格层级、挂出特定深度的订单意愿表达；但现在，你有了一种在整本账本上进行全局表达的、远为动态且极富表现力的方式。因此我认为它远不仅是所谓的连续报价请求（RFS），而是一项真正颠覆性的创新，也是让我感到非常振奋的方向。  
**主持人：** 在这个领域还有哪些潜在的创新空间？

### [00:37:59 - 00:38:33]
**EN:** The fact that you're getting this like really firm Oracle update at the top of the block, do you anticipate that this is something that's going to be maybe an input for liquidations and some vault protocols? Is there a way to leverage proper AMMs to make unique DeFi composability experiences maybe with like some kind of atomic trading or is there like anything else like application specific sequencing related where you could trade against the proper AMM and then do maybe like some yield strategy contains in a particular transaction.  
**CN:** **主持人：** 鉴于你们在区块顶部执行（top-of-block）获得了这种极度确定的预言机报价更新，你是否预计这可能会成为某些借贷清算或金库协议（Vault protocols）的输入源？是否有办法利用 PropAMM 创造出独特的 DeFi 可组合性体验？比如某种原子化交易，或者类似于应用专属排序（Application-Specific Sequencing / ACE）的场景——用户可以在一笔交易中先与 PropAMM 完成兑换，紧接着执行某种特定的收益策略？

### [00:38:33 - 00:39:03]
**EN:** What else is out there that could be built here? Yeah. So I think even just for proper AMMs and specifically addressing, for example, the cost thing. So currently you would feed your Oracle update in every block. So as part of ACE, we actually have a way for the market maker to say actually only include this Oracle update if there's someone who's going to trade against the proper AMM in this current block. Interesting.  
**CN:** **主持人：** 还有什么其他可以在此基础上构建的潜在创新？  
**Kubi：** 是的，单就 PropAMM 而言，特别是针对成本问题——目前做市商通常需要每个区块都推送一次预言机更新。而作为应用控制执行（ACE, Application-Controlled Execution）机制的一部分，我们实际上为做市商提供了一种方式，允许他们设定：“只有在当前区块内确实有人要与本 PropAMM 发生交易时，才打包包含本笔预言机更新”。  
**主持人：** 这太有意思了。

### [00:39:03 - 00:39:32]
**EN:** So again, more expressivity and this way we are very comfortable at the gas cost as well. So you can go even further. You can say, let's say gas prices are high only if the swap is high enough so that whatever if they capture maybe half a basis point on that trades in terms of P&L. So they can specify, for example, what the minimum order size should be for their Oracle updates to hit and then that order being executed.  
**CN:** **Kubi：** 这样一来就带来了更高的表达能力，做市商也能对 Gas 成本感到非常放心。甚至还可以更进一步：比如在 Gas 费飙升时，做市商可以要求只有当交易金额足够大、做市盈亏（P&L）能捕获哪怕半个基点（0.5 bps）的利润时才触发。也就是说，他们可以自定义设置触发预言机更新及成交执行所需的最低订单规模（minimum order size）。

### [00:39:32 - 00:40:00]
**EN:** So I think you can just go very fine granola and fine tune, which is like a beautiful thing. And this is the thing like this idea of ACE, but it actually happening practically. Other than that, I would say everything that you've mentioned, composability, which is just the beauty of it being an on-chain contract. No, that's awesome. How much work have you guys done on...  
**CN:** **Kubi：** 因此，你可以做到非常精细入微的粒度调节（fine-grained，转录误作 fine granola）与微调，这简直太美妙了。这就是应用控制执行（ACE）理念在实践中的真实落地。除此之外，正如你提到的可组合性，而这正是链上合约的魅力所在。  
**主持人：** 太棒了。那么你们在……方面做了多少工作？


---

## Part 3 (00:40:00 - 00:54:22)

# 《Credible Commitments》深度访谈：以太坊区块构建与 PropAMM（Part 3）

**访谈嘉宾**：Kubi Mensah（Gattaca / Titan Builder 联合创始人兼 CEO）  
**播客主题**：Block Building & PropAMMs on Ethereum（以太坊区块构建、交易供应链与 PropAMM 终局）  
**本节概要**：
本篇为深度访谈的第三部分（收官篇）。Kubi Mensah 与主持人探讨了交易供应链与区块构建在更广阔生态层面的落地延伸与终局推演：
1. **L2 与 Based Rollup 中的 PropAMM 机会**：分析了在 Arbitrum 等 L2 以及单一定序器环境下部署 PropAMM 的可行性与微观结构（类似 Solana 但定序环境更单一），以及为何现阶段多数 L2 团队受制于短期商业收益壁垒与定序器掌控权，尚未积极推进 PropAMM 或基于 L1 驱动的 Based Rollup。
2. **Taker 端基础设施与路由标准化**：指出 PropAMM 生态能够运转的两大支柱：除了由 Builder 保证的 Top-of-Block 严格排序外，下游 Taker 端的聚合路由同样至关重要。Titan 联合 LambdaClass 打造了标准化路由合约与通用接口，做市商只需一次集成即可自动接入各大聚合器，凭借极窄的买卖价差（Tighter Quote）获得全网交易流的分发。
3. **AI Agent 工具重塑底层系统工程研发**：Kubi 分享了 Gattaca 团队扁平、高上下文感知（High Context）与高主观能动性（High Agency）的硬核工程文化——坚决摒弃 Jira 工单与繁冗敏捷流程，借助前沿 AI 编程助手将核心架构师的大脑认知规模化放大，尤其在端到端 Telemetry（全链路遥测数据监控）与系统调试排障中，极度压缩迭代周期。
4. **AI Agent 链上自主交易与未来区块空间需求**：探讨了未来 AI Agent 通过免许可轨道自主发起微交易（Microtransactions）的终局趋势。尽管目前以太坊主网仍处于早期探索阶段，但随着区块空间大幅扩容与交易成本降低，智能体高频微交易将成为区块空间吞吐量增长的核心驱动力。
5. **BlockSpace Forum 的愿景与协议外协同**：详述了联合发起“区块空间论坛”（BlockSpace Forum）的初衷——PBS 虽在物理上解耦了提议者与构建者以保护协议层核心安全，但漫长的以太坊协议内硬分叉研发周期无法即时解决当前的中心化引力。全供应链核心参与者必须在“协议外”携手合作，采取务实、具有净正向收益（Net Positive）的渐进方案，共同捍卫以太坊去中心化与抗审查等底层基石。

---

### [00:40:00 - 00:40:23]
**EN:**
Looking into L2s, given that you've done a lot of work with Arbitrum, I know they've just changed their sequencing mechanism, but given that you've spent some time working with L2s, what does this look like there? Is it more straightforward, just like one market maker streaming directly to the sequencer? Are there other L2s that you know that are opted into this? How are you guys thinking about this?
**CN:**
主持人：转向 Layer 2 来看看，考虑到你们团队在 Arbitrum 上做了大量工作，而且我也知道他们最近刚刚调整了定序机制——既然你们在 L2 领域深耕了一段时间，那么 PropAMM 在 L2 上会呈现怎样的形态？它会变得更加简单直接吗，比如做市商直接将报价流式传输（Streaming）给单一定序器？目前据你所知，还有其他 L2 决定采用这种机制吗？你们对此是如何思考的？

### [00:40:23 - 00:40:53]
**EN:**
Because I'm sure it is at some point going to be a business surface, which will be monetizable. We have been thinking about L2s for a long time. We were quite excited about base rollups, so we actually invested quite a bit of resources, built some early stage reference implementations. And then I think, overall, as an ecosystem struggling a bit, that dialed things back. But specifically, if we think about propAMMs, the key part really is the sequencing part.
**CN:**
主持人：因为我相信在未来的某个节点，这必然会成为一个具有商业变现潜力的业务场景。  
Kubi：我们确实对 L2 进行了长期而深入的思考。此前我们对 Based Rollup（基于以太坊 L1 排序驱动的 Rollup）非常兴奋，甚至投入了相当多的工程资源构建了一些早期的参考实现（Reference Implementation）。但后来我认为，由于整个 L2 生态陷入了一些增长与竞争的困境，这方面的探索节奏有所放缓。但如果具体聚焦到 PropAMM 上，其最为核心的关键环节始终是定序（Sequencing）。

### [00:40:53 - 00:41:35]
**EN:**
And so on some of these, where the block times are short enough, it basically starts resembling somewhat the microstructure of Solana, but obviously much easier because you have a homogeneous sequencing environment because you just have one counterpart in your sequencer. So far, however, I haven't seen any of the sequencer operators actually adopting specific - so ACE4 propAMMs, so similar to Solana, you get your Oracle in, it might not land top of block, it's just a transaction that's going to be processed, and then the contract gets unlocked or not based on the Oracle making it in.
**CN:**
Kubi：所以在某些出块时间足够短的 L2 上，其微观结构基本上开始有点类似于 Solana；但由于定序环境是高度同构（Homogeneous）的，而且你的交互对手方仅有唯一的中心化定序器，因此实际实现要比 Solana 简单得多。然而到目前为止，我还没看到任何定序器运营方真正去采用针对性的专属机制——至于 PropAMM，类似于 Solana 的逻辑：你需要将预言机报价（Oracle）打包上链，它未必要处于块首（Top-of-Block），只需作为一笔普通交易被打包执行即可，然后智能合约再根据预言机更新是否成功被打包来决定是否解锁交易状态。

### [00:41:35 - 00:42:14]
**EN:**
But there's the same potential there, so I think it's really up to the sequencer teams. Do you think it's possible that, let's say, over the next six to 12 months, you guys are incredibly successful with propAMMs? In general, there's more activity flowing through propAMM execution on Ethereum. Do you think this will impact L2s and how they think about sequencing? It's definitely an incentive thing, and just based on the conversations we've had in the past, there had been this interest directionally, but then the short-term incentive being, like, we want to be in control of our sequencer, and we want to be in control of the funds.
**CN:**
Kubi：但这背后的潜力是完全一致的，所以归根结底取决于定序器开发团队的意愿。  
主持人：你认为是否存在这样一种可能：比如在未来 6 到 12 个月内，你们在以太坊主网上推进 PropAMM 取得了极大的成功，越来越多的交易流通过以太坊上的 PropAMM 执行，这是否会反过来倒逼 L2 重新审视其定序机制？  
Kubi：这本质上绝对是一个激励机制的问题。根据我们过去与各 L2 团队的交流沟通，他们在战略方向上确实抱有兴趣；但现实中的短期利益诉求却是：“我们希望牢牢掌控自己的定序器，并完全掌控产生的资金流/MEV 收益。”

### [00:42:14 - 00:42:55]
**EN:**
And it's something like, also, if you already have an ecosystem that is active and you have bootstrap liquidity and the different protocols, if you are ahead of everyone else, you kind of want to keep that in your garden, right? So an incentive question. That's totally fair, at least with the Strahl map, and I think, as you noted, with gas scaling and other improvements that are happening and research like E-Eazy and implementations there, I think there definitely is a path to some of this in the future, and I think some of these primitives that are being built now will certainly benefit there in the future.
**CN:**
Kubi：此外，如果你已经拥有一个高度活跃的生态系统，冷启动了可观的流动性并吸引了众多协议，如果你在竞争中处于领先身位，你自然倾向于把这一切圈留在自己的“围墙花园”内，对吧？所以这纯粹是一个利益驱动的博弈问题。  
主持人：完全理解，至少结合当下的 Rollup 路线图来看是这样。正如你所提到的，随着 Gas 扩容、底层性能优化以及类似 ePBS（协议内提议者-构建者分离）等前沿研究与工程实现的推进，未来一定会为这种范式开辟出一条清晰的演进路径，而今天正在构建的这些底层原语也必将在未来释放巨大价值。

### [00:42:55 - 00:43:29]
**EN:**
So yeah, thanks for providing some color on that. Is there anybody in particular in the trade supply chain that plays an outsized role in propAMMs that may be overlooked? So on Ethereum, I would say the most critical one is the builder, just because the ordering is what matters here, because the market makers interact directly with the builders. That is a big part. However, the other critical part is the taking.
**CN:**
主持人：非常感谢你对此的剖析。在交易供应链中，是否有哪一个特定角色在 PropAMM 体系中扮演着极其关键却又容易被外界忽视的角色？  
Kubi：在以太坊上，我认为最核心的角色无疑是构建者（Builder），因为这里的决胜点在于交易排序（Ordering），且做市商需要直接与构建者进行撮合与交互，这是极其关键的支柱。然而，另一个同等关键却鲜有人提及的组成部分是吃单端（Taking / 路由兑换端）。

### [00:43:29 - 00:44:07]
**EN:**
And the beautiful thing about propAMMs are that they are on the ledger, and then anyone that interacts with the chain and routes across what exists on the chain can discover best prices. And so the thing that routes, basically, is actually a critical infrastructure piece because now you no longer need to have everyone integrate with every new propAMM, right? So you just need the routers to be aware. We actually worked with the Lambda class team.
**CN:**
Kubi：PropAMM 的绝妙之处在于它们完全部署在账本（链上）上，任何与区块链交互并在链上现有流动性之间进行路由寻优的实体，都能实时发现最佳价格。因此，执行智能路由的组件实际上是至关重要的基础设施——因为你不再需要让每一个终端用户或前端应用都去单独适配每一个新上线的 PropAMM，对吧？你只需要让聚合路由器（Router）感知到它们的存在即可。为此，我们专门与 LambdaClass 团队展开了深度合作。

### [00:44:07 - 00:44:47]
**EN:**
So they built like a routing contract and like an interface that we are now also telling market makers to use so that there's basically a standardized interface. So when people integrate, it just has to be done once, and then maybe you add addresses, but everyone sort of has the same swap functions and so on and so forth. So I would say from the taker's perspective and getting this ecosystem going, that's actually like a critical part because then anyone can just plug in, provide liquidity, and as long as you provide like a tighter quote, you get distribution through the routers.
**CN:**
Kubi：他们打造了一个通用的路由合约与标准化接口，目前我们也正建议做市商统一接入该规范，从而建立起一套标准化的交互接口。这样一来，下游应用与聚合器只需要完成一次集成；后续即使有新的做市商资金池上线，也只需要添加合约地址，大家共用相同的兑换函数（Swap Functions）与接口规范。因此可以说，从 Taker 端以及推动整个生态运转的角度来看，这绝对是一个决定性的核心模块——任何机构都可以即插即用地提供流动性，只要你报出的买卖价差更紧凑（Tighter Quote），就能自动通过各聚合路由获得全网交易流的分发。

### [00:44:47 - 00:45:19]
**EN:**
I'd be curious. I've talked to a lot of different builders over the last two years about this. In kind of November, October, I think maybe a little bit earlier, we had Opus 4.6 come out and it kind of changed the game, right, with coding. And not only for non-technical folks, but also for technical folks who have a vision for architecture, they know exactly what they need. Basically being able to put together a workforce on demand, work on your production code base.
**CN:**
主持人：我对此非常好奇。过去两年我和很多不同的开发者与构建者聊过这个话题。大约在 10 月、11 月，或者更早一些，新一代前沿大模型（如 Claude 3.5 Sonnet / Opus 等）陆续推出，这彻底改变了编程的游戏规则。它不仅赋能了非技术人员，对于那些拥有宏观系统架构视野、清晰明确自身技术需求的高阶工程人员来说更是如此——你基本上可以按需随时组建一支虚拟的工程攻坚团队，在你的生产级代码库上协同作业。

### [00:45:19 - 00:45:58]
**EN:**
How have some of these tools, whether it's Opus, codecs, or any local models that you guys are running, how have these helped you in either like the product process or maybe even in the security process when you're looking for flaws? So I think overall, increased just productivity of our team. For us, it's quite important that everyone who's on the team is very high context and like high agency, so we absolutely hate the idea of Jira, tickets, having some product manager and sprint master or whatever the thing is, right?
**CN:**
主持人：无论是 Claude/Opus、Codex，还是你们本地部署运行的开源模型，这些 AI 工具在你们的产品研发迭代，或者在安全排查与代码审计挖掘漏洞的过程中，具体起到了怎样的助力？  
Kubi：我认为总体而言，它们极大地提升了我们团队的工程生产力。对我们团队来说，极为关键的一点是要求每位成员都必须具备“极高的全景上下文认知（High Context）”和“极强的主观能动性（High Agency）”。所以我们极度反感那种靠 Jira 工单流转、配备专职产品经理和敏捷教练（Sprint Master）的繁冗管理套路，对吧？

### [00:45:58 - 00:46:48]
**EN:**
And so because our engineers are generally just very high context and the bottleneck is so a lot of things happen in one brain and that brain has access and can make decisions autonomously having agents or using these tools basically just allows that source to have more scale in terms of shipping code. The key here is finding the right balance between keeping the scope clear enough, making sure that everything is reviewed, just using it as a way to extend your execution capabilities and making sure that the logic of thinking and how that gets translated into code is kept very tight.
**CN:**
Kubi：正因为我们的工程师普遍掌握着系统最深层的全局上下文，原本的瓶颈往往在于很多顶层架构思考只存在于某个核心工程师的大脑中，这颗大脑拥有全部系统权限且能高度自主决策。而借助 AI Agent 或这些前沿大模型，本质上就是赋予了这颗“核心大脑”在代码交付与工程实施上成倍放大的杠杆与规模化能力。这里的诀窍在于把握好平衡点：既要保持工程范围（Scope）足够清晰、确保所有代码均经过极其严格的 Code Review，同时将其纯粹作为延伸你执行能力的杠杆，确保底层思考逻辑与最终代码实现之间的映射严丝合缝。

### [00:46:48 - 00:47:23]
**EN:**
Has there been anything in particular that's changed in your workflow? Like for example, maybe before some members of the team were spending X percent of their time writing lines of code for a particular product feature, but now they're not necessarily writing the code as much, but they're spending more time reviewing code or working on technical documents historically, that's been a pain, right? You got to sit down and grind it out, but now you can have an LLM kind of go through that, generate it for you, but you spend your time editing maybe more so to make sure that it's up to snuff and that it didn't leave out or amid any key details.
**CN:**
主持人：在你们的具体工作流（Workflow）中有发生什么显著的转变吗？比如在过去，团队成员可能需要将 X% 的时间用于为某个具体功能手写代码；但现在可能不再需要频繁亲手敲代码，而是将更多精力投入到 Code Review 或者技术文档的撰写上——以往写文档往往是一件痛苦且耗时的苦差事，你必须硬着头皮一点点抠，但现在大语言模型可以替你草拟生成，你只需要花时间进行高阶精修，确保内容达到最高水准、没有任何遗漏或疏忽关键细节。

### [00:47:23 - 00:48:00]
**EN:**
Maybe you're not doing that, but yeah, I would just be curious like if there's anything in particular in the workflow that has changed. I would say debugging is a big one and data collection analytics. So we have telemetry across everything that hits our infrastructure at the edge, along the path to all different components until it exits, which helps us to keep an eye on things, have monitoring alerts and so on and so forth, but also understand what is happening and where the bottlenecks are and what needs to be improved.
**CN:**
主持人：也许你们的工作流并不是这样演变的，但我很想了解，到底有哪些流程发生了本质改变？  
Kubi：我认为系统调试（Debugging）以及数据采集与全链路分析是最显著的两个领域。在我们的底层基础设施中，从边缘节点（Edge）接收到流量开始，沿着数据流经各个核心组件直至最终出口，我们在每一处路径上都部署了细粒度的遥测监控（Telemetry）。这不仅能帮助我们全天候掌控系统运行状态、设置实时监控告警，更重要的是能够精准透视内部正在发生什么、瓶颈具体卡在何处，以及哪些环节亟需性能调优。

### [00:48:00 - 00:48:35]
**EN:**
Also keeping the cycles between pushing something, seeing how it behaves and then iterating on that so that it's much tighter. Have you seen much demand so far, at least from like transaction activity on various different networks that you're monitoring with these analytics where people are basically doing more agentic transactions with things like MPP or X402 or any of these types of protocols? There was a lot of hype conversation about that.
**CN:**
Kubi：同时，它极大压缩了“推送代码部署、观测系统线上行为、进而快速迭代”这一闭环的研发周期，让整个反馈环路变得前所未有的紧凑。  
主持人：基于你们这套分析系统所监控的各个不同网络，到目前为止，你是否在链上交易活动中观察到了明显的需求增长——即人们开始利用类似 MPP、HTTP 402（x402）或其他协议，发起更多的自主智能体交易（Agentic Transactions）？此前市场上围绕这个方向涌现了海量的热炒和讨论。

### [00:48:35 - 00:49:03]
**EN:**
So I guess the two part, are you seeing an activity right now beyond like what's on the dashboards and do you see this as potential demand for block space going forward? Where like maybe on Ethereum, let's say on a given day, I mean we do have already a lot of bot trading that is happening on a given day, it could be majority of the trading, but beyond that, do you see agentic use cases consuming more block space going forward?
**CN:**
主持人：所以这是一个由两部分构成的问题：第一，抛开那些宣传看板上的虚假数据，你们现在是否在链上看到了真实的智能体交易活动？第二，你是否看好这会成为未来区块空间的核心增量需求来源？在以太坊上，虽然在任意给定的日常交易中已经充斥着海量的 Bot 套利交易（甚至占到了交易流的大半），但除此之外，你是否预见未来会有更多基于自主智能体的真实用例去大量消耗区块空间？

### [00:49:03 - 00:49:40]
**EN:**
So directionally, absolutely yes. I think just fundamentally of thinking about what blockchains are and the fact that you have permissionless access and commerce is exactly kind of what you want. You don't want to go through the constraints of having to approve something or being blocked by gate kept and so on and so forth. So I guess the answer is seeing that today, not really and not on Ethereum, but I think directionally that is going to happen, absolutely yes, and a matter of time.
**CN:**
Kubi：从大趋势方向上看，答案是绝对肯定的。归根结底，如果你深入思考区块链的本质，它所提供的免许可访问（Permissionless Access）和无需中介的商业结算机制，恰恰是自主智能体所梦寐以求的运行土壤——它们绝不想受制于现实中繁琐的人工审批流程，或者被中心化机构设卡拦截。因此我的回答是：如果看当下现实，目前在以太坊上还没有真正成规模发生；但在长期发展趋势上，这必然会成为现实，这纯粹只是一个时间问题。

### [00:49:40 - 00:50:07]
**EN:**
The capacity is just going to be way higher and still actually having microtransactions for things that are super low cost is actually viable. Yeah, I agree with that. I think it's inevitable, but it's going to take some time. I love though that throughout this conversation, just referencing the scaling of block space on Ethereum mainnet and so I love that you guys are having that in mind as you're building for the future.
**CN:**
Kubi：未来的区块链吞吐容量将大幅提高，届时以极低成本进行高频微交易（Microtransactions）将完全具备经济可行性。  
主持人：我非常赞同这一点。这无疑是历史的必然，只是需要时间沉淀。在整场对话中，我非常欣赏你们始终紧扣以太坊主网区块空间的扩容演进来展开思考，我也很高兴看到你们在面向未来构建基础设施时，始终将这一点铭记于心。

### [00:50:07 - 00:50:32]
**EN:**
The last thing I'm going to ask you today and I appreciate you've given a lot of details about Prop AMMs and also your organization, how you guys think about building and different problems is you've recently done this collaboration called the BlockSpace Forum and I think it started in Buenos Aires. You had your first meeting there and then you did another one, DCC and CAN, and there's a video online and some excellent talks and panels.
**CN:**
主持人：今天我想向你探讨的最后一个问题是——首先非常感谢你详尽拆解了 PropAMM 以及你们团队攻克各种复杂问题的工程哲学——你们最近联合发起了一个名为“区块空间论坛”（BlockSpace Forum）的合作倡议。如果我没记错的话，这个论坛始于布宜诺斯艾利斯（Devcon 期间），在那里举办了首届闭门会，随后又在戛纳（EthCC 期间）举办了第二届，目前线上已经公布了活动视频，包含了许多极其精彩的专题演讲与圆桌研讨。

### [00:50:32 - 00:51:03]
**EN:**
People who are listening can check out, I recommend it. The information is still very relevant for today. How did that come about and what are the goals you're looking to get out of that? The reason why block builders exist was this realization that there's this fundamental problem around block space allocation that is very resource intensive and anything that is resource intensive and requires sophistication is a centralizing force.
**CN:**
主持人：收听本期播客的听众一定要去看看那些视频，我强烈推荐，其中的很多前沿认知至今依然极具含金量。能聊聊这个论坛是如何诞生的，以及你们希望通过它达成什么样的愿景与目标吗？  
Kubi：区块构建者（Block Builder）之所以会存在，根源在于人们深刻认识到：区块空间的分配与竞价机制存在一个极其根本的瓶颈，那就是它极度消耗计算与算法资源。而任何高度消耗资源且需要极其复杂的专业技术才能参与的环节，天然都是一股强大的“中心化引力”（Centralizing Force）。

### [00:51:03 - 00:51:47]
**EN:**
The point is that there is this centralizing force and this thing has a flywheel effect. The reason why we all here is because blockchains have these properties that are very attractive. So one way of making sure that the core protocol doesn't get degraded is by decoupling it. But then the question is if access to consuming this high value block space doesn't exhibit some of these properties and it doesn't have to be at the same level as the core protocol but it has to exhibit some of these properties, otherwise the overall properties get diminished.
**CN:**
Kubi：核心问题在于，这股中心化引力具有自我强化的飞轮效应。而我们所有人之所以齐聚在这个领域，是因为区块链具备去中心化、抗审查等极具吸引力的核心属性。因此，确保以太坊核心协议层不被中心化侵蚀的一种解法，就是通过解耦（如 PBS 提议者/构建者分离）。但紧接着带来的问题是：如果外部访问与消费这部分高价值区块空间的渠道，本身无法体现出区块链应有的这些核心属性——即便不需要达到核心协议同等的极高安全标准，也必须具备一定程度的抗审查与免许可保障——否则，整条公链引以为傲的整体属性都会被严重削弱。

### [00:51:47 - 00:52:17]
**EN:**
That is a problem for everyone and so basically the protocol itself is thinking about this in long time frames and there's only limited capacity in terms of upgrading the protocol and their priorities. But given that we are part of this supply chain and part of the stack that actually is faced with the intricacies of this, we obviously are aware of the problems and are thinking about how to solve them out of protocol.
**CN:**
Kubi：这关乎生态中每一个人的切身利益。然而，以太坊协议层核心开发者是以极长的时间跨度来考量这一问题的，协议内硬分叉升级的研发带宽与优先级排序极其有限。但作为身处交易供应链前线、每天都在直面这一技术栈各种复杂博弈与工程细节的参与者，我们深知这些痛点的紧迫性，并且一直在深入思考如何在“协议外”（Out-of-Protocol）找到破解之道。

### [00:52:17 - 00:52:46]
**EN:**
And so the block space forum was basically meant to bring together all the actors across the supply chain that think the same way and see what can be done in the meantime to make sure that the core protocol properties don't get degraded. I guess the key is similar to how we approach engineering. Let's do something that's net positive today, it might not solve the entire thing but let's keep improving and let's not wait around until the perfect solution is there and building it then.
**CN:**
Kubi：因此，创办“区块空间论坛”（BlockSpace Forum）的初衷，就是把整条交易供应链上志同道合的核心参与者（构建者、中继、搜索者、应用方）汇聚在一起，共同探讨在当下现阶段我们能做些什么，以确保以太坊核心协议的去中心化与抗审查等底层特质不被侵蚀退化。其精髓与我们的工程哲学如出一辙：让我们今天就去交付能带来“净正向效益（Net Positive）”的务实方案，它或许无法一次性彻底解决所有难题，但只要能持续演进就行，而不是消极等待遥不可及的“终极完美方案”出炉才开始动手。

### [00:52:46 - 00:53:18]
**EN:**
The other thing I will note, at least my observation, is that I think it's a net positive for the discourse having a venue where you can be open and talk about your own perspectives in maybe a way that's not crypto Twitter and it allows for people to actually collaborate and identify maybe what are some challenges that the other person has that I don't have or vice versa and to work together on solving a problem like decentralized block building for example or reducing spreads.
**CN:**
主持人：从我的观察来看，我也想补充一点：拥有一个能够让人畅所欲言、坦诚交换观点的平台，对整个行业的公共讨论是极具建设性的净收益。它摆脱了 Crypto Twitter（X 平台）上的口水战与噪音喧嚣，让从业者能够坐下来真正协同合作，深入了解对方正在面临哪些自己未曾遇到的工程困境，从而齐心协力去攻克诸如去中心化区块构建（Decentralized Block Building，如 BuilderNet 倡议）或进一步收窄做市价差等硬核行业难题。

### [00:53:18 - 00:53:48]
**EN:**
Major kudos on the venue and I think it will be excellent for the ecosystem discourse on this topic in particular. Thanks. It's been awesome chatting with you today. If folks want to get in touch with you and talk about Prop AMMs or they have some other business propositions, what's the best way to get in touch with the Gattaca team or yourself? Yeah, so easiest way probably just Twitter, so it could be meant on Twitter so you can just DM me and DM's are open.
**CN:**
主持人：向你们创办这一论坛致以由衷的敬意，我相信它对以太坊生态就此类议题的深度探讨具有巨大的推动价值。非常感谢，今天与你的对话干货满满、极其精彩！如果听众想与你取得联系，探讨 PropAMM 或者有其他商务合作构想，找到你或 Gattaca 团队的最佳渠道是什么？  
Kubi：最便捷的方式大概就是通过 Twitter（X）了，我的推特账号是 @kubimensah，欢迎随时私信我，我的私信通道是对全网开放的。

### [00:53:48 - 00:54:00]
**EN:**
Otherwise, we also have a Discord for Titan Builder and so anyone can just join and message in there as well. Sweet. Thank you for coming on the show today.
**CN:**
Kubi：此外，我们还为 Titan Builder 建立了官方 Discord 社群，大家也可以随时加入并在里面发消息与我们互动。  
主持人：太棒了。非常感谢你今天做客我们的播客节目！
