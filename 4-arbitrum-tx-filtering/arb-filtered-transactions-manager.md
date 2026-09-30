# ArbFilteredTransactionsManager：Nitro 如何把交易过滤写进状态转换函数

面向 Mantle（OP Stack）devs / CTO 的技术深潜。Arbitrum 背景按「对照你们已经熟悉的东西」写，不按教材目录写。

一手源索引见同目录 `sources.md`。文内引用用 `D1`、`N6` 这类编号。

---

## 0. 先把结论钉死

`ArbFilteredTransactionsManager`（`0x0000…0074`）不是「sequencer 黑名单合约」。它是 ArbOS 60/61 加进 **状态转换函数（STF）** 的一张链上 `txHash` 表：授权 filterer 先把哈希写进去，之后任意节点（包括处理 L1 Delayed Inbox 强插进来的消息时）在进 EVM 之前就把这笔交易判失败（`N4` `N6`）。

合规产品（ArbOS 61 Elara，文档名 *Compliance Filtering*）是 **两层**：

1. **Sequencer 层**：用 TRM / Chainalysis 一类的加盐地址表 + 事件规则，模拟交易，命中则根本不打包（`D1`）。Robinhood 说的「看起来像从没发生过」指这一层（`R1`）。
2. **STF 层**：上面那张 `txHash` 表。Delayed Inbox 绕得过 sequencer，绕不过 STF（`D1` `R2`）。

两层必须拆开读。合在一起会得到错误结论：要么以为 OP Stack 开个 mempool 过滤器就算抄到了，要么以为链上预编译会让交易从历史上消失。

对 Mantle 的硬事实：

- OP Stack 的 `OptimismPortal.depositTransaction` **必须**被 derivation pipeline 收进 L2（`O2`）。默认 STF **没有**「按哈希作废」这一步。
- 因此：sequencer RPC 拒收挡得住日常流量，挡不住 L1 强插。Arbitrum 用预编译把强插也变成「进块但失败」。这是 fork STF，不是配节点。
- Arbitrum One / Nova **关掉**了这套能力；它是给 Orbit 专有链（Robinhood 这种）用的（`D2` `D3`）。
- L2Beat 因此把 Robinhood 的 sequencerFailure 从 Orbit 默认的 *Self sequence* 改成 *No mechanism*（`R2`）。这不是文案，是风险评级改写。

## 1. OP Stack 团队需要先补的 Arbitrum 背景

### 1.1 Nitro 不是「另一个 op-geth」

Mantle 熟悉的图是：`op-node` 从 L1 抽 `TransactionDeposited` 和 sequencer batches，`op-geth` 按 derivation 执行。执行引擎基本是 Geth，rollup 特异逻辑大量在 `op-node` 的 derivation 和 L1 合约（`OptimismPortal`、`SystemConfig`）。

Arbitrum Nitro 把特异逻辑塞进执行引擎内部，自称 **Geth sandwich**（`D5` `D6`）：

- **上片（ArbOS 预处理）**：把一条 `L1IncomingMessage` 变成恰好一个 L2 block；注入 `ArbitrumInternalTx`「start block」冻结 L1 区块号 / 时间戳 / base fee；定价；**做交易过滤**（`D6` 原文：*applies any transaction filtering*）。
- **馅（Geth core）**：EVM，行为对齐以太坊。
- **下片（ArbOS 后处理）**：结算费用、记 L2→L1 消息、收块。

一条消息对应一个 L2 block，是双射（`D6`）。欺诈证明要重放的就是这条 STF，不是「再跑一遍独立的 derivation 脚本」。所以 ArbOS 是 **编进 STF 的 Go 代码**，每个诚实节点和欺诈证明器必须确定性重放（`D6` `D10`）。

对照：

| | OP Stack | Nitro |
|---|---|---|
| 执行引擎 | op-geth，尽量少改 | Geth + ArbOS 夹心 |
| L2 特异逻辑主要在 | `op-node` derivation + L1 合约 | ArbOS 状态 + 预编译 |
| 一条 L1 消息 | 可能对应多个 L2 tx（deposit 列表） | 恰好一个 L2 block |
| 系统合约地址 | `0x42…` predeploys | `0x64`–`0x74` 预编译 |
| 升级单位 | 合约 + 节点软件 | **ArbOS version** 写在状态里，到点全网切语义（`D6`） |

ArbOS version 不是 Docker tag。行为变化绑在状态里的 `upgradeVersion` / `upgradeTimestamp` 上，所以 Elara 是一次 **STF 升级**，不是「新节点多几个 flag」。

Orbit（文档现称 *Arbitrum chains* / dedicated blockchains）是同一套 Nitro，给链所有者打开：出块时间、DA、gas token、以及本文的过滤（`D13`）。Robinhood Chain 是 Orbit L2，不是 Arbitrum One。

### 1.2 预编译：看起来像合约，跑的是 Go

以太坊预编译（`ecrecover` 等）是「固定地址、本地实现、省 gas」。Arbitrum 把这套扩成 **操作系统 API**：Solidity 只提供 ABI，Go 提供实现，启动时用 reflection 核对签名（`D6` `D11`）。

对过滤相关的几个地址（`N1` `N2` `N3`）：

| 地址 | 名字 | 谁能调 |
|---|---|---|
| `0x64` | `ArbSys` | 任何人（L2 block number、发 L2→L1 消息） |
| `0x6b` | `ArbOwnerPublic` | 只读：owner 列表、filterer 列表、`getTransactionFilteringFrom` |
| `0x70` | `ArbOwner` | **仅 chain owner**。授权 filterer、打开过滤开关、设资金接收地址 |
| `0x74` | `ArbFilteredTransactionsManager` | **仅已授权 filterer** 能写；`isTransactionFiltered` 只读开放 |

`0x70` 走 `OwnerPrecompile` 包装：非 owner 直接 `unauthorized`（`N7`）。`0x74` 走 `FreeAccessPrecompile`：filterer 调写入 **不收 storage gas**（`N7`），好让 sentinel 在延迟窗口内稳定写哈希，不被 L2 gas 市场卡住。

预编译存储不在普通合约账户。`FilteredTransactionsState` 开在专用系统地址上，key 就是 `txHash`，value 是 `bytes32(1)`（`N5`）。

### 1.3 Sequencer、Delayed Inbox、Force Inclusion

日常路径和 OP Stack 一样：用户把签过名的 L2 tx 交给中心化 sequencer，立刻从 sequencer feed 拿到 **soft finality**；稍后压缩进 batch，用 blob 或 calldata 打到 L1 `SequencerInbox`，继承 L1 数据可用性，再等断言确认才是 hard finality（`D7` `D8`）。

抗审查路径不一样。

**Arbitrum：**

1. 用户（或合约）在 L1 调 Delayed Inbox 的 `sendL2Message`，把签过名的 L2 tx 字节塞进去（`D8` `D9`）。
2. 正常 sequencer 会在几分钟内把这条 delayed message 编进 feed。
3. 若超过大约 **24 小时**还没编进去，任何人可调 `SequencerInbox.forceInclusion`，把 delayed 队列头部推进 sequencer inbox（`D8`）。BoLD 的 Censorship Timeout 可以在长时间停机时把这个窗口压到更短（`D7`）。

强插保证的是 **进入规范消息序列**，不是「EVM 里一定跑成功」。

**OP Stack：**

1. 用户在 L1 调 `OptimismPortal.depositTransaction`（多数经 `L1CrossDomainMessenger`）（`O2`）。
2. `op-node` 扫 `TransactionDeposited`，编成 deposit tx。
3. sequencer 最多可以拖：相对 L2 的 max time drift 约 **30 分钟**；sequencing window 约 **12 小时**。窗口过了，节点开始只出 deposit-only block（`O1`）。

OP 的强插保证更硬：一旦 L1 事件进了规范 L1，derivation **必须**把它变成 L2 tx。没有「先登记哈希再让 STF 失败」这一步。失败只能是 **EVM 执行 revert**（gas 不够、目标合约 revert），不能是「协议在进 EVM 前按政策作废」。

### 1.4 为什么「写进 STF」比「sequencer 拒收」硬一个数量级

Sequencer 拒收是 **节点政策**。换一个诚实节点、走 L1 入口，交易仍必须被处理。OP 和 Arbitrum 在这一层是同类：中心化 sequencer 都能在 mempool 里审查。

写进 STF 是 **共识政策**。每个验证者、每个欺诈证明器重放同一条消息时必须得到同一结果（`D10`）。如果规范历史里有一笔 delayed 消息，STF 要么执行它，要么按链上已提交的规则失败它；不能「这个节点跳过、那个节点执行」，否则欺诈证明会分叉。

Elara 的设计：

1. 把「这笔不该执行」编码成 **链上状态**（`txHash → 1`）。
2. 让 STF **读这份状态** 再决定要不要进 EVM。
3. 用一条 **更早的、由授权账户发出的 L2 交易** 来写这份状态，从而所有节点看到同一份表。

代价是：谁能写这张表，谁就能作废任意（包括强插的）交易。L2Beat 因此不再承认 Delayed Inbox 是 Robinhood 的 escape hatch（`R2`）。

## 2. Elara 把过滤产品化了，但 One / Nova 关掉了

Elara 把合规过滤做成专有链的 **opt-in 协议组件**，One 默认不开（`D3` `D4`）。
时间线（`D2`）：

| 日期 | 事件 |
|---|---|
| 2026-04-27 | 合规产品博文：政策由运营商定义，Arbitrum 提供 enforcement（`D4`） |
| 2026-06-29 15:00 UTC | ArbOS 61 在 Arbitrum Sepolia 激活 |
| 2026-08-01 | Constitutional onchain vote 截止 |
| 2026-08-20 17:00 UTC | ArbOS 61 Elara 在 One / Nova 激活。节点必须升到 Nitro `v3.11` |
| — | ArbOS **60** 因 gas refund 两个交互 bug **从未**在 One / Nova 激活；61 是修 STF 后的新版本（`D2`） |

接口和 Go 注释写的是 *Available in ArbOS version 60*（`N1` `N2`）。L2Beat 和 Elara 公告写 61。两边都对：代码闸门是 60，公共主网列车是 61。讨论生产链时说 Elara / ArbOS 61；读源码时会碰到 `ArbosVersion_TransactionFiltering` 和 60。

Elara 在 One / Nova 上实际打开的是 Stylus 96 KB 和 `BaseFeeManager`。过滤和 AltDA API **默认关**，要 chain owner 显式打开（`D2` `D3`）。官方原话：这套能力「不是打算给 Arbitrum One 用的」，放进 61 是为了 **canonicalize 进节点软件**，给有制裁义务的 Orbit 链用（`D1`）。

打开过滤还要过 `ArbOwner.setTransactionFilteringFrom(timestamp)`。`AddTransactionFilterer` 在 `enabledTime == 0 || enabledTime > now` 时直接 revert：`transaction filtering feature is not enabled yet`（`N8`）。二进制里有代码 ≠ 链上能写哈希。

文档建议 Orbit 链至少等 Elara 在 One 上线 **30 天**再采用（`D1`）。这是运维建议，不是协议约束。

## 3. 两套机制，不要合成一套

官方把它们画在一张「Compliance Filtering」页上（`D1`），工程上是两套进程、两份状态、两种失败外观。

```
用户 tx
        │
        ├─► Sequencer RPC / prechecker
        │       地址表（S3 加盐 sha256）+ 事件规则 + 模拟
        │       命中 → 拒绝打包，RPC："Transaction rejected by chain policy"
        │       链上无收据  ← Robinhood「像没发生过」
        │
        └─► Delayed Inbox / forceInclusion（绕过上一层）
                Delayed Sequencer 停住
                transaction-filterer 发 L2 tx：addFilteredTransaction(hash)
                STF 看到 hash 在表里 → 进块但失败
```

### 3.1 Sequencer 规则引擎：地址 / 事件，链下名单

名单不在链上。Chain owner 接外部合规商，产出 **加盐哈希列表**（`D1`）：

```
hashed_address = SHA256(salt16Byte || address20Byte)
```

JSON 里带 `salt`、`hashes`、`issued_at`、`hashing_scheme`（`sha256-stringinput` 或 `sha256-rawbytesinput`）。节点从 S3 拉取，放内存，周期性全量替换。明文地址不进状态树，是为了不让人从链上枚举制裁名单（`D1`）。

CLI 开关（`D1`）：

- `--execution.transaction-filtering.address-filter.enable`
- `--execution.transaction-filtering.address-filter.s3.{bucket,object-key,region,…}`
- `--execution.transaction-filtering.event-filter.path` / `.rules`

事件规则举例：监听 `Transfer(address,address,uint256)`（selector `0xddf252ad`），topic 1/2 当地址查表；`bypass` 允许 `to == 0x0` 的 burn，好让 Aave / Morpho / Paxos 一类协议清算或销毁受限账户余额（`D1`）。约束：

- 一笔 tx 里只要还有别的违规，整笔仍失败。
- 受限地址自己当 `tx.origin` 发起 burn：sequencer 层直接挡。

文档还列了 opcode 规则：对受限地址的 `CALL` / `CREATE` / `CREATE2` / `SELFDESTRUCT`（`D1`）。这是 **模拟阶段** 的策略，不是 EVM 里新加的 ban opcode。

这一层的失败外观：交易不进 block，没有 receipt，explorer / indexer 对不上「一笔失败的调用」。Robinhood 写的是：*a blocked transfer is never processed… appears as though the event never occurred, ensuring indexers remain synchronized with the actual state*（`R1`）。这句话只覆盖 sequencer 拒收，不要拿去描述 STF 路径。

### 3.2 STF Guardian：txHash，链上预编译

文档商品名：**Transaction Guardian Precompile**。代码名：`ArbFilteredTransactionsManager`。Delayed Inbox 专用配套进程文档叫 **Delayed Inbox Sentinel**，仓库里是 `cmd/transaction-filterer`（`D1` `N9`）。

为什么必须有这一层：Delayed Inbox 消息过了 force-inclusion 窗口就会进入规范 inbox。Sequencer 再拒收已经没用，所有节点都得处理这条消息（`D1`）。

Guardian 做的事非常窄：

1. 授权实体把 **交易哈希** 登记上链。
2. STF 处理该哈希时强制失败，并消耗所附 gas 防 spam（`D1`；实现见 §4.3）。

粒度是 **整笔交易的哈希**，不是「这个 from」或「这个 selector」。要拦一个地址的所有未来交易，sequencer 层靠地址表；STF 层必须对 **每一笔已签名的 tx** 各写一次哈希。哈希在签名完成时就确定，filterer 不必等它被打包（`N1` `N10`）。

Sentinel 的职责（`D1` `N9` `N10`）：

1. 监视 Delayed Inbox 里命中规则的消息。
2. 在 force-inclusion 窗口关之前，通过 filterer 账户调 `addFilteredTransaction`。
3. 然后才允许 delayed sequencer 把那条消息编进去，STF 按表失败。

系统测试把第 3 步写成硬等待：`WaitingForFilteredTx` 为真时 delayed sequencer **halt**，直到链上出现对应哈希（`N10`）。这是时序约束，不是协议多出来的挑战期。L2Beat 说的 *without delay* 指：哈希一旦在表里，STF **立刻**失败，没有 24 小时冷却（`R2` `R3`）。窗口用在 **写入必须发生在执行之前**。

## 4. `ArbFilteredTransactionsManager` 代码级机制

### 4.1 地址、ABI、存储

Solidity 接口三个方法、两个事件（`N1`）：

```solidity
interface ArbFilteredTransactionsManager {
    event FilteredTransactionAdded(bytes32 indexed txHash);
    event FilteredTransactionDeleted(bytes32 indexed txHash);

    function addFilteredTransaction(bytes32 txHash) external;
    function deleteFilteredTransaction(bytes32 txHash) external;
    function isTransactionFiltered(bytes32 txHash) external view returns (bool);
}
```

Go 实现几乎是薄封装（`N4` `N5`）：

```go
func (con ArbFilteredTransactionsManager) AddFilteredTransaction(...) error {
    if !con.hasAccess(c) { return c.BurnOut() }
    return filteredTransactions.Open(evm.StateDB, c).Add(txHash)
    // + emit FilteredTransactionAdded
}

func (s *FilteredTransactionsState) Add(txHash common.Hash) error {
    return s.store.Set(txHash, presentHash) // presentHash = bytes32(1)
}

func (s *FilteredTransactionsState) IsFilteredFree(txHash common.Hash) bool {
    return s.store.GetFree(txHash) == presentHash
}
```

`IsFilteredFree` 是 STF 热路径用的：读存储 **不收 gas**。普通 `IsFiltered` 给 `eth_call` / 预编译 view。`DeleteFree` 会从 trie 里真正删掉 key（不是写成零）；注释说过滤执行完后由外部服务清表，STF 里暂不删。retryable 路径还留着 `// May move to direct deletion here in future`（`N5` `N6`）。

存储账户是 `types.FilteredTransactionsStateAddress`，不是 `0x74` 自己的合约存储。预编译地址只是调用入口。

L2Beat 用 `FilteredTransactionAdded` 的 event count 当「运营商是否在真的审查」的指标（`R3`）。topic0：

- Added: `0xde91070c821fde128c9ad6c14fac600aa5d701f4f74997eb997cd5591414426f`
- Deleted: `0xdb54a829e2a086557fb16aeb8950816c19960c430bf7d36c791c0f8382b66d22`

### 4.2 权限：Owner 授权，Filterer 写哈希

两级，不要混：

```
Chain owner (0x70)
    setTransactionFilteringFrom(t)     // 功能开关，时间闸
    addTransactionFilterer(addr)       // 授权
    removeTransactionFilterer(addr)
    setFilteredFundsRecipient(addr)    // 0x0 = 回落到 networkFeeAccount

Filterer (任意 EOA / 合约，通常是 sentinel 热钱包)
    0x74.addFilteredTransaction(hash)
    0x74.deleteFilteredTransaction(hash)
```

`hasAccess` 只查 `c.State.TransactionFilterers().IsMember(c.caller)`（`N4`）。Owner 自己若没把自己加成 filterer，**不能**直接写 `0x74`。这是有意的职责分离：治理改名单，机器人写哈希。

未授权调用 `add`/`delete`：`BurnOut()`，吃光 gas 再失败。View `isTransactionFiltered` 不走这个检查。

`AddTransactionFilterer` 还要过时间闸（`N8`）。Genesis 可用 `params.ArbOSInit{TransactionFilteringEnabled: true}` 打开（`N10` 测试 builder）。

公开查询走 `0x6b`：`getAllTransactionFilterers()`、`getTransactionFilteringFrom()`、`getFilteredFundsRecipient()`（`N3`）。L2Beat 的 discovery 就是 call `0x6b.getAllTransactionFilterers()`；ArbOS < 61 的链上这个方法 revert，模板用 `expectRevert`（`R3`）。

### 4.3 STF 三条路径：普通 tx / deposit / retryable

热路径在 `arbos/tx_processor.go`（`N6`）。**不是**用户交易去 CALL `0x74`。ArbOS 在处理该交易类型时自己查表。

**A. 普通 L2 tx（含 Delayed Inbox 塞进来的 signed tx）**

走 `RevertedTxHook`。注释写明：delayed 消息被地址过滤器标过、再被写进链上表之后，这里 skip execution 但耗 gas。

```go
if p.state.FilteredTransactions().IsFilteredFree(txHash) {
    p.evm.StateDB.SetNonce(from, nonce+1, ...)
    usedGas := *gasRemaining
    *gasRemaining = 0
    return usedMultiGas, &core.ErrFilteredTx{TxHash: txHash}
}
```

效果：

- **进块**，有 receipt，`status = failed`（`ErrFilteredTx`）。
- nonce +1，防重放。
- 剩余 gas 全部烧掉（惩罚 / 防 spam，对应 `D1`「consuming any gas attached」）。
- **不调用** `to`，不改目标合约存储。
- 不是「从未发生」。Indexer 会看到一笔失败交易。

**B. `ArbitrumDepositTx`（L1 `depositEth` 一类）**

Deposit 的 `endTxNow=true`，到不了 `RevertedTxHook`，所以在类型分支里单独查表：

```go
if IsFilteredFree(txHash) {
    to = filteredFundsRecipient   // 或 networkFeeAccount
    txnErr = &core.ErrFilteredTx{TxHash: txHash}
}
MintBalance(from, value)
Transfer(from, *to, value)
return endTxNow=true, txnErr
```

ETH **仍从 L1 桥记到 L2**，只是贷记地址被改成合规接收账户。这和 `D1` 的警告不完全一样：文档说如果 child 侧 **丢掉** 这笔 credit，ETH 会锁在 L1 bridge 里。实现选择的是 **重定向** 而不是丢弃。锁仓仍可能发生（运营商另写逻辑不 mint），但默认 STF 不是那样。

**C. `ArbitrumSubmitRetryableTx`**

```go
if IsFilteredFree(ticketId) {
    tx.FeeRefundAddr = recipient
    tx.Beneficiary   = recipient
    isFiltered = true
    filteredErr = &core.ErrFilteredTx{TxHash: ticketId}
}
// 仍然创建 retryable；auto-redeem 被跳过
```

系统测试断言（`N10` `verifyCascadingRedeemFiltered`）：

1. Delayed sequencer 先 halt 在 `ticketId`。
2. Filterer 写入后 resume。
3. Submission receipt `status = failed`。
4. Ticket **仍在**（`GetTimeout` 不 revert），beneficiary 已改成 `filteredFundsRecipient`。
5. 同一 block 里 **没有** `ArbitrumRetryTx`（auto-redeem 没跑）。

`filteredErr` 传给 `PostTxFilter`，避免 delayed sequencer 把「已经按链上表处理过的 tx」再当成未登记而二次 halt。

三条路径的共同点：哈希必须 **先** 在表里。STF 不会自己去跑地址规则。地址规则是 sequencer / sentinel 的事；预编译只认哈希。

### 4.4 免费存储与 gas 惩罚

`FreeAccessPrecompile`（`N7`）：先把 call 转给真正的 `0x74` 实现，再看 caller 是不是 filterer。是：返回 `gasSupplied` 未扣、`gasUsed = 0`。不是：按普通计量。

再加上 `IsFilteredFree` / `DeleteFree`：STF 读表、清表也不走 gas 计量。否则「失败一笔」本身会改变 gas 会计，和欺诈证明的 gas 约束缠在一起。

用户侧相反：被过滤的普通 tx **烧光剩余 gas**。这是故意不对称：filterer 写入要便宜且确定性，被过滤的强插不能变成便宜的 L2 DoS。

## 5. Delayed Inbox：halt → register → fail

把 delayed 路径展开成时序。假设制裁地址 A 的 sequencer RPC 已被拒，A 改走 L1。

```
t0  A 在 L1 Inbox.sendL2Message(signed L2 tx)     // txHash 此刻已确定
t1  Sentinel 看见 delayed 消息，命中地址/事件规则
t2  Delayed sequencer halt：WaitingForFilteredTx([txHash]) = true
t3  transaction-filterer RPC Filter(txHash)
      → 热钱包发 L2 tx: 0x74.addFilteredTransaction(txHash)
      → 这笔必须先被 STF 执行，表里才有 1
t4  halt 解除，delayed 消息被编进 sequencer inbox / feed
t5  STF 处理 A 的 tx：IsFilteredFree → nonce++ / 烧 gas / ErrFilteredTx
```

`transaction-filterer` 是独立 binary（Makefile 和 Docker 镜像都打进去）。它起一个只暴露 `transactionfilterer` namespace 的 HTTP/WS 栈，队列长度 100，**单消费者** 串行发交易，避免 nonce 碰撞（`N9`）。Nitro 节点配 `execution.transaction-filtering.transaction-filterer-rpc-client.url`；changelog 写明：delayed-sequencing filtering 开着但 URL 没配，进程 **fail fast**（NIT-5012）。

Filterer 账户必须：

1. 已被 `ArbOwner.addTransactionFilterer`。
2. 有足够 L2 余额付普通 tx 的 L1 数据费（写入本身对 filterer 免 storage gas，但作为一笔 L2 tx 仍要付 batch 费）。
3. 在 24 小时（或 Censorship Timeout 压短后的窗口）内完成 t3。

窗口内 sentinel 挂了会怎样？Delayed sequencer 若一直 halt，规范延迟消息堆着。有人 `forceInclusion` 之后，消息进入 sequencer inbox，**所有节点必须执行**。此时若表里还没有哈希，STF 会 **真的执行** 那笔制裁交易。这是运维 SLA，不是协议自动补救。`D1` 把它列在 Security considerations 第一条：没有 STF + sentinel，受限用户等窗口结束就能绕过 sequencer。

`forceInclusion` 并不额外给 filterer 宽限。L2Beat 的 *without delay* 指 STF 失败没有挑战期（`R2`）。24 小时是 **写入窗口**，用完即过期。

还有一条 L1 桥风险（`D1`）：`depositEth()` 先把 ETH 锁进 L1 Bridge，再在 L2 记一笔 deposit tx。若 L2 侧既不贷记原地址也不重定向，ETH 会卡在桥里。默认实现走重定向（§4.3 B）。运营商仍要在用户沟通里写清：被过滤的充值不会到账原地址。

## 6. 对照 OP Stack：Mantle 默认做不到同等的强插拦截

| | Arbitrum Nitro + Elara 过滤 | OP Stack（Mantle 默认） |
|---|---|---|
| 日常审查 | Sequencer 模拟 + 地址/事件表，拒收不进块 | Sequencer / RPC 拒收。同样不进块 |
| L1 入口 | Delayed Inbox `sendL2Message` | `OptimismPortal.depositTransaction` |
| 强插延迟 | ~24h（Censorship Timeout 可压短） | max time drift ~30m；sequencing window ~12h |
| 窗口过后 | 任何人 `forceInclusion`，消息进规范 inbox | 节点出 deposit-only block，**必须**纳入 deposit |
| 强插之后能否按政策失败 | 能。STF 查 `0x74` 哈希表 | **不能**（不改 op-geth STF 的前提下） |
| 失败外观（强插） | 进块、failed receipt、nonce++、烧 gas / 改贷记 | 进块并执行；失败只可能是 EVM revert |
| 审查写在哪 | 链上状态 + 每个验证者的 STF | 节点政策；诚实 derivation 不能跳过 deposit |
| 欺诈证明 | 过滤是 STF 的一部分，重放必须一致 | deposit 执行也是 STF 的一部分，**跳过**才会不一致 |

OP 的抗审查比「没开 Elara 的 Arbitrum」更硬的一点：没有 24 小时排队，L1 事件一旦最终化，derivation 在窗口后必须消化（`O1` `O2`）。这正是合规团队不喜欢的性质。

Mantle 若只做 sequencer mempool 过滤：

- 日常 OFAC 流量能挡。
- 制裁地址付 L1 gas 走 `depositTransaction`，12 小时内 sequencer 可以拖；窗口后 **必须执行**。
- Explorer 会有一笔成功或 EVM-revert 的 deposit，取决于目标合约，**不是**协议级作废。

要在 OP Stack 上复制 Elara 的 STF 层，只能：

1. **改 op-geth / 自定义 deposit 预检查**，让 derivation 在执行前查一张链上表。这是 STF fork，要进欺诈证明 / 未来 ZK 证明的约束，等于自己做一次 Elara。
2. **L1 侧拦**。Portal 没有官方 filterer hook。要拦得改 `OptimismPortal` 或在前面加 gate 合约，用户可以绕过 gate 直接调 Portal（`O2` 也提醒别直接调 Portal，但那是安全建议不是权限）。
3. **不拦交易，拦资产**。ERC-20 hook / 权限池。强插的是「调了某个合约」；如果合约自己拒绝转账，EVM revert，状态不变。粒度在资产，不在 txHash。

1 的成本和 Nitro 已经付过的成本同类。3 才是 OP Stack 上不必 fork STF 的那条路。

ZK Stack / Linea 有另一类 *TransactionFilterer*：在 **L1 enqueue** 时按地址白名单拒绝进队列（L2Beat 对 Sophon、GRVT、Linea 的描述）。那是「强插入口都不让进」。Elara 是「入口放进，执行失败」。Robinhood 选择后者，所以合约部署仍然无许可，交易执行可被定点作废。

## 7. Robinhood 落地与 L2Beat 的评级改写

Robinhood Chain：Orbit L2，chainId 4663，2026-07-01 对公众去掉 transaction-access whitelist（`R2` milestones）。文档承认 sequencer-level screening，并链到 Arbitrum 的 Advanced Compliance Filtering（旧 URL；现文是 `D1`）（`R1`）。排序是 FCFS，没有优先费拍卖（`R1`）。Elara 给专有链加的 priority fee 是另一项，RH 没用它做审查。

L2Beat 对普通 Orbit 的默认假设是：sequencer 挂了或审查，用户可以 Delayed Inbox + `forceInclusion` 自助上链（*Self sequence*）。Robinhood 打破了这个假设，所以模板被 **nonTemplateRiskView 覆盖**（`R2`）：

> There is no guaranteed mechanism to have transactions included if the sequencer is down or censoring. Although users can enqueue messages in the L1 delayed inbox and call forceInclusion on the SequencerInbox, the chain runs ArbOS 61 transaction filtering: an authorized filterer can register any transaction hash in the `ArbFilteredTransactionsManager` precompile (`0x00…0074`), after which the state transition function forcibly fails that transaction, including force-included ones, without delay.

`forceTransactions` 技术段标题直接改成：*Force inclusion can be nullified by transaction filtering*。风险句：operator 把哈希登记进预编译，即使用户已经 L1 强插，STF 仍失败。

Discovery 还跟踪 One / Nova 上的同一个预编译，但注明 **no filterers and no filtered transactions**；模板存在是为了「一旦激活立刻被发现」（`R3`）。Robinhood 上 TransactionFilterer 角色 **当前在链上 active**（`R2` 注释）。

另外：Robinhood 的 validator whitelist 开着（`validatorWhitelistDisabled: false`），欺诈证明许可化，L2Beat 归类为 Other 而不是 Rollup（`R2`）。过滤和封闭证明是两件独立的中心化，不要并成一句「RH 很中心化」。CTO 讨论里应该分开：谁能排序、谁能作废交易、谁能提交断言。

Robinhood 文档对开发者的承诺是：`eth_call` / `eth_getLogs` / 余额查询不受影响；被挡的 **transfer** 不进状态，indexer 对齐（`R1`）。结合 §3.1 / §4.3：这在 sequencer 拒收时成立；走强插 + STF 失败时，indexer 会看到 failed receipt。如果合规审计要「链上无痕迹」，只能靠 sequencer 层；STF 层故意留痕迹（事件、失败收据、资金重定向）。

## 8. Mantle 该抄什么、不该抄什么

先分三层，再决定抄哪一层。Elara 把 1 和 2 焊在同一份节点软件里，不代表 Mantle 也要焊。

| 层 | 拦什么 | 粒度 | 强插 | 政治成本 | OP Stack 现状 |
|---|---|---|---|---|---|
| Sequencer / RPC | 日常 OFAC / 制裁地址 | 地址、事件 | 绕得过 | 中：要公开规则 | 能做，今天就能配 |
| STF / 预编译 | 强插进来的同一批政策 | txHash | 绕不过 | 高：L2Beat 会改评级 | **默认没有**；要 fork STF |
| 资产 / Hook | 某个 ERC-20 或某个池 | token / poolId | 交易进块，转账 revert | 低到中：只污染该资产 | 标准合约，不改 STF |

**该抄：**

1. **分层说话：**「链的使用无许可」和「链的验证 / 审查许可」必须分开写进对外材料。Robinhood 公开了 screening 的存在（`R1`）；藏着会被 L2Beat 和社区同时打。
2. **Sequencer 层只做制裁级、可公开的规则。** 加盐地址表 + 事件规则 + burn 例外，是 Elara 里最容易迁到 op-geth txpool / prechecker 的部分，且不碰 STF。
3. **股票腿的 KYC / 发行人限制放资产或池。** 用 txHash 做日常 KYC 等于每笔用户交易都要跑一遍 sentinel，而且会把整条链标成可审查。Launchpad 补遗里已经写过这层切法，这里只补工程理由：预编译粒度是整笔哈希，没有「只禁这个 selector」。
4. **失败外观要选。** 想让 indexer 「像没发生」：只能 sequencer 拒收。想要审计轨迹：必须进块失败。不要承诺两套同时成立。
5. **如果真要 STF 级拦截，按 Elara 的形状做，不要发明第三种。** 链上表、授权 filterer、写入必须先于执行、deposit 重定向而不是丢 credit、filterer 免 storage gas。缺任何一项都会在强插窗口或桥资金上出事故。

**不该抄：**

1. **不要为了 Launchpad KYC fork op-geth STF。** 欺诈证明、后续 ZK、与上游 OP 的 rebase 成本，换来的是 L2Beat *No mechanism*。Robinhood 愿意付，是因为它是持牌券商的证券场地；Mantle 主网不是。
2. **不要把 meme 和股票 quote 放进同一套链级过滤器。** RH 的结构问题正是：链层能杀交易，股票 ERC-20 却无 transfer hook，池子无 KYC。链级开关太粗，资产级太细，中间那层空着。
3. **不要把 One 上「Elara 已上线」读成「One 在过滤」。** One / Nova 关掉了（`D2`）。抄 Orbit 专有链的能力，不是抄 One。
4. **不要假设 24 小时窗口会救 sentinel。** 窗口是写入截止，不是执行冷却。

CTO 决策用一句话：Mantle 要的是 **可披露的制裁筛 + 按资产的合规**，不是 **可作废任意强插交易的 STF**。后者是持牌 Appchain 的制度选择，会改审查抗性评级。

## 9. 常见误区

1. **「预编译会让交易从历史上消失。」** Sequencer 拒收才会。STF 路径进块、失败、加 nonce（`N6`）。Robinhood 那句「像没发生过」只描述前者（`R1`）。
2. **「用户交易自己 CALL `0x74`。」** 用户交易甚至不必知道这个地址。ArbOS 在 `TxProcessor` 里查表（`N6`）。`0x74` 的调用者是 filterer 的 **另一笔** L2 tx。
3. **「Owner 能直接写哈希。」** Owner 只授权。没把自己加成 filterer 就 `BurnOut`（`N4` `N8`）。
4. **「ArbOS 60 和 61 是两套功能。」** 功能闸在 60；One 上的列车是 61，因为 60 因 refund bug 跳过（`D2`）。
5. **「Force inclusion 被关掉了。」** 入口还在。被作废的是 **执行结果**。L2Beat 改评级是因为 escape hatch 不再 *可靠*，不是因为函数被删（`R2`）。
6. **「过滤 = 地址黑名单上链。」** 链上只有 txHash。地址表在 S3 内存里（`D1` `N5`）。
7. **「OP Stack 开个 mempool 过滤器就等价。」** 等价的是 §3.1。§3.2 要改 STF。
8. **「Deposit 被过滤 ETH 会永远锁在桥里。」** 文档警告的是丢 credit 的情况（`D1`）。默认实现重定向到 `filteredFundsRecipient`（`N6`）。
9. **「Retryable 被过滤 = ticket 不存在。」** Ticket 在，beneficiary 被改，auto-redeem 跳过（`N10`）。
10. **「这是给 Arbitrum One 用户的审查。」** 官方反复写：One / Nova 关；给有制裁义务的专有链（`D1` `D2` `D3`）。

## 10. 一手来源

完整表见 `sources.md`。按阅读顺序：

1. **机制（先读）：** `D1` 官方 Compliance Filtering；`N1` `N4` `N5` `N6` 接口与 STF；`N9` `N10` sentinel 与系统测试。
2. **背景：** `D5` `D6` Nitro / ArbOS；`D8` `D9` Delayed Inbox；`D10` STF 与欺诈证明。
3. **产品边界：** `D2` `D3` Elara；`D13` Orbit。
4. **对照：** `O1` `O2` OP Stack 强插。
5. **落地与评级：** `R1` Robinhood 文档；`R2` `R3` L2Beat。

已知矛盾（不要在转述时抹平）：ArbOS 60 vs 61；`D1` ETH-lock vs `N6` 重定向；`R1`「从未发生」vs STF failed receipt；文档商品名 Guardian / Sentinel vs 代码名 `ArbFilteredTransactionsManager` / `transaction-filterer`。
