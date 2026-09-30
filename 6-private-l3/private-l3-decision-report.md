# 为什么（不）一定要做 Private L3

> 研究轨道：6 ｜ 取数日期：2026-09-17 ｜ 归属：private L3 决策调研
> 可信度标记：[一手] / [二手] / ⚠️存疑
> 交叉引用：`0-narrative/07`、`0-narrative/09`（只引用，不改写）

## 0. 立场摘要

**推荐：只做支付型 private L3，且仅当产品目标是高吞吐隐私转账/结算。不要为了「隐私 DeFi 可组合性」去做 Commonware/Bonsai L3；也不要把「一定要上 private L3」当默认答案。**

负载论断三条：

1. **EVM 上的 Railgun / Privacy Pools / Tornado 并不是「在 account model 里做隐私」**。它们把 Zerocash 的 note + Merkle + nullifier 塞进 Solidity 合约。Account model 没有帮上隐私，只是把公开余额机和 Gas 账单送进来。
2. **EVM 隐私的真实加分项是：能跟已有公开 DeFi 在同一笔 EVM 交易里做夹心组合**。Railgun 的做法是 unshield → 公开 multicall → reshield。DeFi 本身仍是公开的。这个加分项 **不要求 L3**。
3. **Commonware/Bonsai 作为 L3 状态机，必然放弃同状态机原子 DeFi**。它是 account-based 隐私支付：send 与 receive 分两笔，receipt 打开要私有通道交接，验证者只存每账户一个 commitment。这个设计解的是全局 nullifier 增长与验证吞吐，不是可编程隐私。

若目标是隐私程序（私有 AMM / 隐私借贷），对照物是 Aztec 这类 private+public VM，不是 Bonsai。若目标是交易前隐私（不被夹），那是 sequencer / 加密 mempool，跟 L3 状态机无关。

## 1. 三类问题不要混：资产隐私 / 可编程隐私 / 交易前隐私

后文所有论证按这三个盒子分流。混在一起会把「为什么一定要 L3」说成三个不同问题的拼盘。

| 类 | 要隐藏什么 | 典型机制 | 不是什么 |
|---|---|---|---|
| **资产隐私** | 谁持有多少、付给谁、存取款链路 | shielded note / mixer / 账户 commitment | 不隐藏「我正在下一笔要买什么」 |
| **可编程隐私** | 合约调用栈、私有状态上的计算 | Aztec private fn + notes | 不等于「能跟 Uniswap 原子组合」 |
| **交易前隐私** | 待执行交易的 intent / 报价 | 加密 mempool、私有 RPC、TEE sequencer | 不改变链上已确认交易的可见性 |

Zerocash 明确定义的是第一类：DAP 让用户直接付款，「the corresponding transaction hides the payment's origin, destination, and amount」。[Z1]

Railgun 同样定义第一类：隐藏 sender / recipient / token type / amount。[R1]

Aztec 把第二类写进协议：`#[external("private")]` 在用户设备上证明，可 `self.call()` 私有调用其它合约；对公开函数只能 enqueue，「any return values or side effects are not available during private execution」。[A2]

第三类是排序器问题。仓库 `1-rfs-propamm/chain-infra/sequencing-and-cancel-priority.md` 已把 pre-transaction privacy 定义为：提议者必须经私有端点收交易，承诺区块前不得泄露。应用合约看不到、也管不了 sequencer 收到交易到打包前做了什么。**本报告不再论证第三类。它不能当作做 private L3 的理由。**

Bonsai 自己承认它在第一类上比 shielded notes 弱：「an external observer only learns that an account performed some action (send/receive) but never learns the amount or counterparty」；论文写「weaker on-chain privacy than shielded notes」。[C1][C2]

## 2. 谱系：Zcash → Tornado Cash → Railgun / Privacy Pools

### 2.1 Zerocash / Zcash：自带账本的 shielded notes

Zerocash (2014) 把 DAP 定义在任意 ledger 货币之上：coin 是 Merkle 树里的 commitment，花费时公开 nullifier，交易隐藏 origin / destination / amount。[Z1] 这个模型要求验证者保留两个不断增长的集合：coin ID 树与 spent nullifier 集。Privacy Pools 论文把同一逻辑写成教科书版本：「Both sets are ever-growing」。[P1]

Zcash 把 DAP 落到共识协议，并分池（Sprout / Sapling / Orchard）。ZIP 224：Orchard 只能花 Orchard notes，进出池走 `valueBalanceOrchard`。[Z3] **隐私资产的原子操作是池内 pour；跟透明资产或其它程序的接口是出入池。**

### 2.2 Tornado Cash：把 mixer 贴在 EVM 合约上

tornado-core README：用户生成 secret，把 commitment 连同固定面额存进合约；提款时证明自己拥有某个未花 commitment的 secret，不披露是哪一笔。「breaking the on-chain link between the recipient and destination addresses」。[T1]

Classic 的上限写在规格里：

- 固定面额（没有池内 split/merge）
- Deposit gas 1,088,354（`43381 + 50859 * tree_depth`）；Withdraw gas 301,233
- 证明时间 ~10s（`10213ms = 1071 + 347 * tree_depth`）[T1]

Privacy Pools 论文补了一句历史：「The first version of Tornado Cash did not have a concept of internal transfers, it only allowed deposits and withdrawals.」 Nova 才加任意面额与内部转账，且当时仍是 experimental。[P1]

**Tornado 没有 DeFi 可组合性。** 提款后资产回到公开地址，再去 Uniswap。隐私做完了，可组合从零开始。

### 2.3 Privacy Pools：用 association set 换可见的「清白」

论文 core idea：不再只证明「提款对应某个历史存款」，而是证明 membership in a more restrictive **association set**。用户对同一 coin ID 证两条 Merkle 路径：全局 coin ID 树根 R，以及 association set 根 RA。[P1]

分离均衡的教科书例子：Alice/Bob/Carl/David 排除 Eve（盗贼存款），Eve 不能排除自己。于是第 5 笔提款只能来自 Eve。**合规证明以缩小匿名集为代价。** [P1]

0xbow 实现把 ASP 写成第三层：合约层 / ZK 层 / ASP 层。提款要验证 `_proof.ASPRoot() == ENTRYPOINT.latestRoot()`。被 ASP 排除的存款可 **ragequit：公开退池**。[P3][P4]

这仍是 mixer 形状：存取款 + 部分提取。没有跟 Uniswap 的原子接口。

### 2.4 Railgun：把 Zcash UTXO 塞进 EVM，再用 Adapt 接 DeFi

官方隐私系统页：Private Balances 是加密 UTXO Merkle 树，「similar to Bitcoin and Zcash's spending system」，树在合约里。[R1] 状态结构页：accumulator + `nullifiers` mapping；花掉的 note 用 nullifier 做双花标记。[R2]

跟 Tornado 的差别不是「换了 account model」，而是：

- 池内私有转账（0zk → 0zk）
- 多 token、任意金额 JoinSplit
- **Relay Adapt 跨合约调用**（第 4 节）

它仍要在 EVM 上验证 Groth16、写全局 nullifier、同步 Merkle。Bonsai 的开篇就是在批这一类「每笔交易公布 nullifier」的部署系统。[C2]

## 3. EVM account-model 隐私协议的结构性问题

先纠正题目里的前提。**这些协议没有在 EVM account model 上做隐私；它们在 EVM 里仿真了一套 notes。** 公开账户余额、`msg.sender`、storage slot 仍然全透明。隐私发生在合约内部的 Merkle + nullifier，跟 Zcash 同一套原语。[R1][R2][P1]

下面五条限制针对资产隐私协议本身，不与交易前隐私混谈。

### 3.1 匿名集被「可见性」和「合规集合」同时咬

Railgun 自己把隐私水平写成三个函数：unique shield 数、TVL、DeFi/Private Send 量。稀有 meme token 的隐私弱于 USDC/DAI。[R1] 这是 mixer/shielded-pool 的共性：匿名集 = 你愿意与之不可区分的那些存款。

Privacy Pools 把这件事做成了显式政策。Association set 可以是「全部历史存款」到「只含自己」之间的任意子集。诚实用户的激励是 **做大集合但排除脏存款**。结果是：合规用户的匿名集严格小于全局池。[P1]

Tornado Classic 更硬：固定面额把池切成互不相通的桶。1 ETH 池帮不了 10 ETH 池。[T1]

### 3.2 合规层是旁路，不是密码学免疫

Railgun Private POI：**可选、与核心合约分离**，「does not make RAILGUN's privacy conditional」。但钱包侧有效果：新 shield 有 **Unshield-Only Standby Period（文档写 1 hour）**，期间只能退回原地址；Broadcaster 要求 token 完成 POI 才肯代发。List Providers 包括 Elliptic、Chainalysis Sanctions Oracle 等。[R3]

Privacy Pools 把合规写进提款电路：`ASPRoot` 必须等于 Entrypoint 最新根。被排除则 ragequit，**公开**拿回存款。[P4] 论文还警告：association set 的 Merkle 根至少应上链，否则恶意 ASP 可以对不同用户给不同集合以去匿名。[P1]

**上限：EVM 合约隐私可以加 POI / ASP，但不能既要全局匿名集又要交易所愿意收币。** 这是设计选择，不是实现缺陷。L3 不会 magically 消掉这个张力；最多换一个执行位置（排序器过滤 vs 电路 membership）。

### 3.3 Gas 与证明成本钉在 EVM 验证器上

Tornado Classic：存 ~1.09M gas，取 ~301k gas，证明 ~10s。[T1]

Railgun 开发文档：cross-contract 证明「Proofs can take 20-30 seconds on slower devices」。[R4] 链上仍要为 Groth16 verifier + Merkle 更新 + nullifier 写入付 gas。

Bonsai 指出结构原因：部署的隐私支付系统几乎都为每笔交易公布 nullifier；「At a million transactions per second, the nullifier set grows by a petabyte each year」。Nullifier 必须在快存储里并对每笔入账交易检查。[C2] **这不是 Solidity 写得差，是「全局 nullifier + EVM 存储」这条路的渐近。**

### 3.4 UX / 流动性：同步树、Broadcaster、出入池摩擦

Railgun 钱包要同步并解密每片 commitment 才能重建私有余额。官方原文：「This syncing process can take a few minutes。」[R0] 0zk 地址不能自己付 gas，Broadcaster 代发；没有 POI 时只能 self-broadcast 退回。[R1][R3]

出入池有费。Relay Adapt 文档写 unshield fee **0.25%**：想 swap 100 DAI，实际进入公开 DeFi 的是 99.75。[R4]

流动性被切成：公开池 / 私有余额 / 各 token 自己的 shield 噪声。Railgun 声称 DeFi 交互增加噪声，同等 TVL 下隐私强于纯 mixer。[R1] 这是他们的主张；它不改变「私有余额不能直接当 Uniswap 的 `msg.sender`」这个事实。

### 3.5 与普通 DeFi 组合的真实上限（机制，不是口号）

Railgun 跨合约调用的官方步骤：[R4]

1. unshield，token **暂时进入 Relay Adapt 合约**（公开）
2. 对该余额做任意 `{to, data, value}` multicall（公开，谁都能看见调用了 0x / Uniswap）
3. 把结果 shield 回 0zk
4. 三步在 **同一区块、同一笔交易**，失败则整笔 revert

Adapt Module 把执行参数绑进 SNARK，「proofs are only submittable from the specified contract」。[R5]

所以：

- **原子性**：有，但是 EVM 交易原子性，不是「私有状态机里的私有 AMM」。
- **隐私**：0zk 身份与私有余额被保护；**swap 路径、金额、对手协议在 Adapt 那一跳是公开的。**
- **可组合对象**：任何已有 EVM 合约。这是加分项，也是上限：你组合的是透明 DeFi，不是私有 DeFi。

Privacy Pools / Tornado Classic 连这一跳都没有：提款地址公开，之后是普通转账。

Aztec 作为对照（只为改变判断）：私有函数之间可以 `self.call()`，连 call stack 都可私有；对公开状态只能 enqueue，**拿不到返回值**，公开函数 **不能** 调私有函数。[A1][A2] 可编程隐私有，但「私有代码读公开预言机再继续私有计算」这条路是断的。

## 4. EVM 隐私的真实加分项及其上限（隐私 DeFi 可组合性）

加分项是真的。第 3 节不能被读成「EVM 隐私一无是处」。

### 4.1 加分项：同一结算域、同一资产、同一笔交易

相对 Zcash 或支付专用链：

- 资产已在 EVM（USDC、WETH、现有 LP token），不必先桥到隐私币。
- Relay Adapt 让「从私有余额出发、打到公开协议、再藏回去」成为 **一笔 EVM 交易**。失败原子回滚。[R4]
- Cookbook recipes 接 DEX / vault / LP / LST，不用重写结算层。[R4]
- 不新开一条 DA / 桥 / 退出协议。这与 `0-narrative/07` 路线一同构：保持现有执行层时，同链组合成本最低。

用户说的「在 evm 上做隐私的好处在于可以做隐私 defi 交易」，机制上应改口为：**私有身份 + 公开 DeFi 的原子夹心，不是私有 DeFi。**

### 4.2 上限 1：DeFi 那一跳是公开执行

Relay Adapt 的 multicall 跑在普通 EVM。金额、selector、router、pool 都在 calldata。观察者看不到 0zk，但看得到「有一笔 Adapt 交易刚在某池 swap 了 99.75 DAI」。[R4]

若攻击者的目标是 **策略/仓位** 而不是身份，这个加分项几乎不成立。挡策略要靠第 1 节第三类（交易前隐私），Railgun 不提供。

### 4.3 上限 2：私有状态不能成为其它合约的同步输入

公开 Uniswap 不能 `balanceOf(0zk)`。Railgun 核心只验证 JoinSplit，不跑私有 AMM。[R2] 要做私有订单簿，必须另写 Adapt 或另开私有 VM。

Aztec 能做私有合约之间的同步调用，但私有执行读不到「这一刻」的公开返回值。[A2] **可编程隐私 ≠ 与公共订单簿的同步组合。**

### 4.4 上限 3：费用、延迟、合规把高频交易踢出去

0.25% unshield fee、20–30s 证明、POI 1h 待机。[R3][R4] 这套 UX 适合偶发私有换仓，不适合 Tape 打新、Perps 撤单、RFS 流式报价。后者要的是排序与延迟（仓库已有轨道），不是 shielded note。

### 4.5 判决：EVM 加分项能不能替代 private L3

| 目标 | EVM 合约隐私够不够 |
|---|---|
| 偶发私有转账 + 偶发公开 DeFi | 够。Railgun / Privacy Pools 就是这个产品 |
| 合规可证明的 mixer | Privacy Pools / POI 就是这个产品 |
| 与现有 Uniswap/Aave 原子交互且隐藏身份 | Railgun Adapt 够；隐藏金额/路径不够 |
| 私有程序（私有 AMM、私有借贷状态） | 不够，需要 Aztec 类 VM |
| 百万 TPS 隐私支付、验证者状态不随交易数增长 | 不够，这是 Bonsai 的问题设定 [C2] |
| 交易前不被夹 | 不够，这是 sequencer 问题 |

**EVM 加分项很大，但大在「复用透明 DeFi」。它既不是做 private L3 的充分理由，也不是否定支付型 L3 的理由。**

## 5. 为何（不）一定要做 dedicated private L3

对照对象：L2 上的合约隐私（Railgun / Privacy Pools）vs 一条 dedicated 私有执行域（可以是 L3，也可以是 L2 上的 native 状态机）。「L3」本身不生产隐私。

### 5.1 L3（或任何专用执行域）多出来的能力

这些能力来自 **换状态机 / 换证明 / 换排序 / 换 DA**，不是来自「多一层 rollup」这个标签。

| 面 | 合约方案做不到或渐近做不到 | 专用执行域可以做 |
|---|---|---|
| 状态机 | EVM storage 里挂全局 Merkle + nullifier mapping [R2] | Bonsai：每账户一个 32-byte commitment，验证者不存 nullifier [C2] |
| 证明 | 每笔 Groth16 on-chain verify，batch 仍 O(B) pairing [C2] | ZK-PARI batch：论文测 2^16 笔摊到 11.7 µs/proof，18 核 ~1.05M ops/s [C1][C2] |
| 排序 | 跟公开 EVM 抢同一 gas 市场 | 支付专用 mempool、稳定 leader、多 proposer（Commonware 其它博文，非本报告范围） |
| DA | 每个 nullifier / commitment 都上 EVM calldata | 验证者只跟 frontier；收据可进 MMR 并修剪 [C1] |
| 合规 | POI/ASP 是电路旁路，匿名集缩小 [R3][P1] | 也可以做排序器过滤（见轨道 4），**这不是隐私，是审查面** |

Bonsai 的问题设定写得很窄：「How do we process one million private transactions per second on commodity hardware?」约束是验证者存储随账户数而非交易数增长，钱包可长期离线。[C2] **只有当你也有这个问题时，才「一定要」离开 EVM 合约。**

### 5.2 并不多出来的能力

- **并不自动得到私有 DeFi。** 支付状态机没有合约。Aztec 才有 private call stack，而 Aztec 是一条 privacy-first L2，不是「在 Mantle 上再叠一层 Bonsai」的免费升级。[A1]
- **并不自动强于 Zcash 的账本隐私。** Bonsai 公开「哪个账户在行动」。[C1][C2]
- **并不消掉出入池问题。** 资产若从 Mantle L2 EVM 进来，仍要 shield/unshield 或跨层桥。跨层不是原子组合（`0-narrative/07` 已写：路线三不得把异步跨层写成同链原子）。
- **并不提供交易前隐私。** 那是 sequencer 泄露模型。
- **并不提供法务免疫。** Privacy Pools 论文记载 Tornado 合约地址进入 OFAC SDN 名单；L3 另有 sequencer、桥、DA 发布者，攻击面不是自动更小。本报告不做法务结论。[P1]

### 5.3 直答「为什么一定要做 private L3」

**不一定要。**

- 若产品是「让 Mantle 用户偶尔私有转 USDC，并能原子地打到现有 DeFi」：在现有 EVM 上接 Railgun 类 Adapt，不要新链。
- 若产品是「合规 mixer」：Privacy Pools / POI，不要新链。
- 若产品是「私有程序」：那是 Aztec 类 VM，不是 L3 标签，更不是 Bonsai。
- 若产品是「高吞吐隐私支付，验证者用消费级硬件」：**要专用状态机**。放在 Mantle L2 之上做成 L3 只是结算/桥的位置选择（资产托管在 L2、执行在 L3），与 `0-narrative/09` 的 app-specific L3 同构。也可以做成 L2 内的 native 状态机（路线二），那不是「private L3」，是「MantleCore 但跑支付」。

把「专用状态机」说成「一定要 L3」，是把结算拓扑和执行语义绑死了。本报告反对这种绑死。

## 6. Commonware private payments 作为 L3 状态机，是否必然放弃 DeFi 可组合性

状态机以博文 [C1] 与论文 [C2] 为准。公开材料能核实的，写进正文；不能核实的进存疑清单。

### 6.1 状态机实际提供什么

账户公开状态是一个 hiding commitment：

`com = Com_acct(b, root_null)`

对余额 `b` 和该账户本地 nullifier 树根。验证者存 **每账户 32 字节 commitment**，外加收据 MMR 的 frontier。[C1][C2]

一笔支付不是一笔链上交易，是两步：

1. **Send.** 发送者发布 `(Sen, com'_Sen, ρ, π)`。`ρ = Com_rec(v, Sen, Rec)`。验证者更新 sender commitment，把 ρ 追加到 MMR。发送者再把 ρ 的 opening 和位置 `pid` **经私有通道**交给接收者。[C2]
2. **Receive.** 接收者把 `pid` 当作 nullifier 插入自己的树，发布 `(Rec, com'_Rec, root_ρ, π)`。验证者不公布 nullifier。[C2]

操作隐藏变体让 send/receive 在账本上看起来一样（dummy receipt），但仍是「某账户做了某事」。[C1]

没有：合约、跨账户同步调用、共享 AMM 状态、公开读取余额。论文 related work 把对照物写成 ecash / shielded notes / 凭证支付，**没有声称可编程**。[C2]

发送者可以扣住 opening 不交给接收者。论文的理想功能明确允许「a corrupted sender withhold a payment's opening from the receiver」。[C2] 钱已从 sender 余额扣掉，receiver 尚未 credit——这不是 DEX 能用的原子 settlement。

### 6.2 三种可组合性，分开回答

| 种类 | 定义 | Bonsai 作为 L3 SM？ |
|---|---|---|
| **同状态机原子组合** | 一笔规范状态转换里，支付与另一应用规则同时提交或同时失败 | **否。** 状态转换只有 Send/Receive（及 Register）。没有第二套应用规则可绑。 |
| **跨合约消息** | 支付 SM 向另一合约发调用/消息，等待或不等待返回 | **否。** 没有合约。Receipt 交接走链下私有通道，连跨合约消息都不是。 |
| **跨层异步** | L3 支付与 L2 EVM DeFi 分两笔，桥/unshield 衔接 | **可以。** 这是普通 rollup 出口。`0-narrative/07`：不得把异步跨层写成同链原子。 |

**直答：是的，采用这套状态机等于放弃同状态机原子 DeFi 可组合性。** 不是「以后加点合约就有了」——加合约就不再是 Bonsai，而是另一条协议（Aztec 或 EVM+notes）。

### 6.3 中间路线（仍要选一个主状态机）

1. **支付 L3 + L2 EVM（推荐的混合，若真要做支付 L3）**  
   Bonsai 跑转账/工资/订阅；需要 DeFi 时异步出口到 Mantle L2。身份隐私在 L3 内；出口后是公开地址。组合是跨层异步。

2. **EVM 上 Railgun，不做新执行域**  
   保留第 4 节加分项。吞吐与验证者状态走 Bonsai 的反面。适合低频。

3. **Aztec 类 private+public VM 作为 L3**  
   私有合约之间可原子组合；与公开状态仍是 enqueue、无返回值。[A2] 这是第三条产品，工作量与 Bonsai 无关。不要把 Commonware 博文当成这条路的设计文档。

4. **盾导资产出口（shielded wrapper）**  
   L3 发行「已屏蔽的 IOU」，L2 合约只认 wrapper。L2 DeFi 看到的是同质 wrapper，不是用户。原子性停在 L2；L3 支付与 L2 swap 仍异步。

没有第五条路可以把「Bonsai 的验证者状态」和「Uniswap 同步回调」焊在同一状态机里。那是两个状态机。

## 7. 推荐与推翻条件

**推荐（一条）：只做支付型 private L3，且仅当产品目标明确是高吞吐隐私转账/结算；默认不做。不要用 Commonware/Bonsai 去承载隐私 DeFi。不要把 Mantle 现有 Tape / Perps / RFS 叙事改写成隐私产品。**

执行含义：

- **默认：不做新的 private 执行域。** 若有低频隐私转账需求，在现有 Mantle EVM 上评估 Railgun 类 Adapt 或 Privacy Pools，当应用而不是当链。
- **仅当** 出现「验证者必须用消费级硬件跑远高于 EVM 合约隐私吞吐的隐私支付」这一产品约束时，才开支付型 L3，状态机用 Bonsai，出口到 L2 做 DeFi。
- **不要** 开 Aztec 类 L3，除非另立「可编程隐私」产品（本报告不推荐，因为与当前资本市场叙事无关，且私有↔公开同步组合仍断）。
- **交易前隐私** 继续放在现有 L3/排序轨道，不并入本轨道。

### 何种证据推翻该推荐

任何一条成立，就重开决策，而不是微调措辞：

1. **Bonsai 后续规范增加可编程状态转换**（合约、共享 AMM 状态、或与 EVM 的同步回调），且仍保持「验证者不存全局 nullifier」。公开博文与 2026/1987 论文目前没有这个。
2. **产品约束变成「必须隐藏策略/仓位，且必须原子打到公开订单簿」**——那时既不是 Bonsai 也不是 Railgun Adapt，需要另写；可能结论是「两边都不够，别做」。
3. **Railgun 类方案在公开基准上证明：验证者存储不再随交易数线性增长，且证明+链上 verify 达到支付产品的延迟预算。** 若 EVM 合约已经满足 Bonsai 的两条要求，支付型 L3 失去理由。
4. **Mantle 现有 L3 产品（Tape/Perps）的负载本身变成隐私支付。** 本报告按边界不把资本市场叙事改写成隐私产品；若业务自己变了，并入 `0-narrative/09` 而不是开第 6 轨。

### 明确拒绝的收尾

不接受「先做 L3 再看能不能加 DeFi」。状态机选 Bonsai，DeFi 就只能异步出口。不接受「EVM 隐私问题很多所以一定要 L3」——第 3 节的问题里，只有吞吐/nullifier 增长需要换执行域；匿名集、合规、公开 DeFi 夹心，换 L3 都不自动修好。

## 存疑清单

1. **TornadoCash_whitepaper_v1.4.pdf 原文未核。** `tornado.cash` TLS 失败。机制与 gas 数字以 tornado-core README 为准 [T1]。⚠️
2. **Railgun 2021 白皮书** 只找到第三方镜像（CryptoCompare PDF），未当一手。Adapt / POI / 状态结构以 docs.railgun.org 为准。
3. **Bonsai 的「完整支付系统」尚未存在。** 博文：下一步才是「carry that throughput through networking, consensus, and storage」。[C1] 吞吐数字是 M5 MacBook Pro 上的证明验证原型，**不是**已上线 L3。不得把 1.05M ops/s 写成生产 TPS。
4. **私有通道如何实现**（发送者 → 接收者的 receipt opening）：论文当假设，未规定反审查投递。若接收者离线，credit 延迟；这会进一步削弱「像传统支付」的口号。⚠️ 无法从公开材料核实投递层。
5. **Privacy Pools 主网规模与 ASP 治理。** 官方文档描述机制；TVL/匿名集大小未在一手文档给出可靠时序数据，故不引用二手媒体数字。
6. **Aztec 主网/测试网阶段。** 只引用 call types 规范。不把 Aztec 生态 DeFi 活跃度当事实。
7. 若后续 Commonware 代码仓出现与论文不一致的状态转换（例如多资产 AMM），以代码+规范为准，本报告第 6 节作废。

## 关键来源清单

完整表见 `sources.md`。正文引用的一手源：

- [Z1] Zerocash, ePrint 2014/349: https://eprint.iacr.org/2014/349.pdf
- [Z3] ZIP 224 Orchard: https://zips.z.cash/zip-0224
- [T1] tornado-core README: https://github.com/tornadocash/tornado-core
- [P1] Buterin et al., Privacy Pools 论文 PDF: https://privacypools.com/whitepaper.pdf
- [P3] Privacy Pools 官方 docs: https://docs.privacypools.com/
- [P4] PrivacyPool 合约: https://docs.privacypools.com/layers/contracts/privacy-pools
- [R0] Railgun Private Balances: https://docs.railgun.org/developer-guide/wallet/private-balances
- [R1] Railgun Privacy System: https://docs.railgun.org/wiki/learn/privacy-system
- [R2] Railgun State Structure: https://docs.railgun.org/developer-guide/engine-1/state-structure
- [R3] Private POI: https://docs.railgun.org/wiki/assurance/private-proofs-of-innocence
- [R4] Cross-Contract Calls: https://docs.railgun.org/developer-guide/wallet/transactions/cross-contract-calls
- [R5] Adapt Modules: https://docs.railgun.org/wiki/learn/integrating-railgun/adapt-modules
- [A1] Aztec Overview: https://docs.aztec.network/developers/overview
- [A2] Aztec Call Types: https://docs.aztec.network/developers/docs/foundational-topics/call_types
- [C1] Commonware, Out of Sight, Out of State: https://commonware.xyz/blogs/private-payments
- [C2] Bonsai, ePrint 2026/1987: https://eprint.iacr.org/2026/1987.pdf
