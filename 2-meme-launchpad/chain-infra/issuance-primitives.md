# 「资产发行原生（issuance-native）」链级原语盘点
> 研究轨道：L ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑

## 0. 本轨道任务与方法论

**任务**：回答"如果一条链要把「资产发行 + 早期流动性 + 交易」做成链原生能力，历史上/现存有哪些原语可抄"，为最终 issuance-native 链 infra 方案提供素材库。本节不做架构设计，只做**原语盘点 + 先例证据 + 落地形态分类**，设计推导留给主报告 `report/09`。

**方法论**：
1. 优先读官方一手源：`solana.com/docs`、`solana-program.com`、GitHub 源码（`solana-program/token-2022`、`ixofoundation/ixo-blockchain`、`osmosis-labs/osmosis`、`InjectiveLabs/*`）、EIP 原文（`eips.ethereum.org`）、Hyperliquid/Berachain/Sei 官方文档。
2. `web_search` 仅用于发现源，不作为最终引用；能一手验证的必附 URL + 原文摘录。
3. 对"链级模块 vs 链上合约"边界严格辨析——这是本轨道最容易混淆的地方（第 2.4 节专门处理）。
4. 对"是否有先例"类问题，穷举检索范围后明确给出"有/无"判断，不打太极。

**与已知起点事实的关系**：本轨道不直接研究 RISE/RISEx（H/I 轨道负责），但第 4 节的"发行类应用最痛的链级缺口"直接对接第一阶段结论（抗狙击、毕业迁移、流动性锁定、费用分配四大痛点），第 7 节的"EVM 缺失能力清单"是本轨道对主报告最核心的交付物。

## 1. 原生代币标准的能力边界

核心问题：一条链的"代币标准"本身能不能替 launchpad 干活？如果能，EVM 到底缺了哪几项，是"合约层能补"还是"必须动执行层"？

### 1.1 Solana Token-2022 / Token Extensions 逐扩展拆解

[一手] Token-2022（Token Extensions Program）是 SPL Token 的超集，程序地址 `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`。扩展通过 `ExtensionType` 枚举定义，在 mint / 账户初始化时选择启用，多数扩展**初始化后不可后加**。源码：https://github.com/solana-program/token-2022/blob/efd0c957fefbd79882d77df5fb2dac88c001249c/program/src/extension/mod.rs#L1059 ；文档总览：https://solana.com/docs/tokens/extensions

官方原文（枚举互斥提示）：
> "Some extensions are incompatible with each other and you can't enable them simultaneously on the same token mint or token account. For example, you can't combine the `NonTransferable` extension with the `TransferFeeConfig` extension."
中译：部分扩展互斥（例如 `NonTransferable` 与 `TransferFeeConfig` 不能共存），设计代币时必须提前规划好要启用哪些扩展组合。

下表**逐扩展**列出官方定义（源码注释原文，https://github.com/solana-program/token-2022/blob/efd0c957fefbd79882d77df5fb2dac88c001249c/program/src/extension/mod.rs#L1059 ）与对应 launchpad 用例：

| 扩展（`ExtensionType`） | 官方定义（源码注释直译） | Launchpad 能用它做什么（具体动作） |
|---|---|---|
| `TransferFeeConfig` / `TransferFeeAmount` | 转账费率信息 + 提取/设置费率的权限；转账中被暂扣的费用 | **创作者/协议手续费自动到账不用 claim**：每笔二级市场转账自动扣一定比例给 `withdrawWithheldAuthority`，无需应用层在 DEX 路由里插 hook。文档：https://solana.com/docs/tokens/extensions/transfer-fee |
| `MintCloseAuthority` | mint 账户的可选关闭权限 | 供给定案后**回收 mint 账户租金**，用于"毕业迁移"后清理旧 mint 元数据账户 |
| `ConfidentialTransferMint` / `ConfidentialTransferAccount` | Confidential Balances 的审计方配置 / 账户侧状态 | **私密转账金额但公开地址**：机构做市商建仓不暴露仓位规模；可配置 auditor ElGamal 公钥供合规方解密。文档：https://solana.com/docs/tokens/extensions/confidential-transfer |
| `DefaultAccountState` | 新建 Account 的默认 `state` | **发行时默认冻结所有新账户**，白名单地址逐个解冻——KYC 前置闸门，无需额外合约 |
| `ImmutableOwner` | 账户 owner 权限不可变更 | 防止 ATA（关联账户）owner 被恶意程序篡改，属于安全兜底，对 launchpad 意义是**降低托管地址被劫持风险** |
| `MemoTransfer` | 要求入账转账必须带 memo | 交易所/风控场景强制转账留痕，对 launchpad 用途有限，更多是合规审计线索 |
| `NonTransferable` / `NonTransferableAccount` | 该 mint 的代币不可转账 / 账户从属于不可转账 mint | **发行"灵魂绑定"式积分/权限代币**（例如打新资格凭证），无需自定义合约实现锁死转账 |
| `InterestBearingConfig` | 代币随时间计息 | 质押型发行代币的**利息在钱包 UI 展示层自动计算**，无需 rebase 或定期分发交易 |
| `CpiGuard` | 锁定通过 CPI 发起的特权指令 | 防止恶意程序在用户不知情时通过 CPI 冒用授权转走代币，属于**钱包安全默认项**，降低"仿冒 DEX 授权盗币"类 rug 手法 |
| `PermanentDelegate` | 可选的永久委托权限，且**用户无法撤销** | **一级 KYB 闸门的强制回收权**：发行方可在任意时刻 burn/转移持有人代币而无需其签名——用于合规冻结违规地址持仓，落地形态是"发行方保留终极控制权"这一类资产（不适合无许可 meme） |
| `TransferHook` / `TransferHookAccount` | mint 要求每次转账 CPI 到实现"transfer hook interface"的程序 | **狙击/女巫防护、动态转账税、白名单转账、转账事件上报**均可挂在这一个钩子上，是本表能力最强的扩展。文档：https://solana.com/docs/tokens/extensions/transfer-hook（`Execute` / `InitializeExtraAccountMetaList` / `UpdateExtraAccountMetaList` 三条指令，PDA 派生规则固定为 `["extra-account-metas", mint, hookProgramId]`） |
| `ConfidentialTransferFeeConfig` / `ConfidentialTransferFeeAmount` | 加密的暂扣手续费与加密公钥 | 让"转账费"与"隐私转账"两个扩展**可以叠加**，即隐私转账场景下手续费依然能自动路由 |
| `MetadataPointer` / `TokenMetadata` | mint 内置指向元数据账户的指针 / mint 直接内嵌 token-metadata | **元数据原生化**：mint 账户"自描述"，浏览器/钱包无需依赖 Metaplex 外部约定即可定位元数据，对应本轨道第 4.4 节"元数据原生化"缺口。文档：https://solana.com/docs/tokens/extensions/metadata |
| `GroupPointer` / `TokenGroup` | 指向"组配置"账户的指针 / mint 内嵌组配置 | **系列代币/多品类发行**（例如同一发行人下的多个关联 meme 系列）可在链级建立分组关系，无需额外 registry 合约 |
| `GroupMemberPointer` / `TokenGroupMember` | 指向"组成员配置"账户的指针 / mint 内嵌成员配置 | 与 `TokenGroup` 搭配，声明某 mint 是某组的成员，供索引器/前端**原生识别"系列代币"归属**，无需链下映射表 |
| `ConfidentialMintBurn` | 允许保密 mint/burn 的代币 | bonding curve 式发行若想对**买卖数量本身保密**（防止狙击者监听 mint 数量）可用，目前主要面向机构/RWA 场景 |
| `ScaledUiAmount` | 代币 UI 显示金额可按比例缩放 | **无需 rebase 实现"股票分割"式改面值**：例如项目方想把展示单位从 1e9 改为 1e6，不需要迁移合约，只改缩放系数 |
| `Pausable` / `PausableAccount` | mint/burn/transfer 可被暂停 | **发行方紧急熔断**：发现攻击/漏洞时链级暂停该代币所有转账，无需依赖治理合约或多签调用某个"暂停"函数（暂停开关本身是协议原生状态） |

【实测】未执行（无需要求；本节内容全部来自官方文档与源码原文，见上表逐条 URL）。

**对 launchpad 最关键的三项**（按第一阶段"抗狙击/毕业迁移/流动性锁定/费用分配"四大痛点排序）：
1. `TransferHook`——唯一能把"转账时校验"下放到协议层的通用可编程点，直接对应抗狷击（第 4.1 节）与反 rug（第 4.5 节）。
2. `TransferFeeConfig`——费用路由原生化（第 4.3 节）的现成答案。
3. `MetadataPointer`/`TokenMetadata` + `GroupPointer`/`TokenGroup`——元数据与索引原生化（第 4.4 节）的协议层地基。

⚠️存疑：`ConfidentialMintBurn`、`ScaledUiAmount`、`Pausable` 均为 Token-2022 **较新扩展**（2025 年前后陆续加入），官方文档未逐一标注"主网启用日期"，本表未能逐项核实各扩展的**首次主网可用日期**，只能确认功能存在于当前（2026-09-07）文档与源码中；这条计入存疑清单。


### 1.2 EVM 侧：ERC-20 + hooks 缺失与失败史

EVM 上"给 ERC-20 加转账钩子"这条路，历史上至少有三次标准化尝试，**全部未能成为主流**：

| 标准 | 提出时间 | 核心能力 | 状态（2026-09-07 实测/查证） | 为何未被采纳 |
|---|---|---|---|---|
| **ERC-777**（https://eips.ethereum.org/EIPS/eip-777） | 2017-11-20 | `send`/`operatorSend` 携带 `data`，`tokensToSend`（发送前）与 `tokensReceived`（接收后）两个 hook，通过 ERC-1820 registry 发现实现者 | EIP 规范状态为 **Final**（一手，页面标记"🎉 Final"），但生产实践**已被否决**：OpenZeppelin 在 4.9 版标记弃用、**5.0 版彻底移除** `ERC777` 实现（[二手] https://forum.openzeppelin.com/t/can-sombody-explain-why-erc777-was-removed/38105 ） | **`tokensToSend` 钩子在余额更新之前触发**，违反 Checks-Effects-Interactions；2020-04 **imBTC/Uniswap V1 重入攻击**利用该钩子在交易未完成前重新进入合约，操纵 AMM 价格，直接导致 Uniswap 损失约 **$1.1M**、给同样持有 imBTC 的 Lendf.Me 造成约 **$24M** 损失（[二手] https://peckshield.medium.com/uniswap-lendf-me-hacks-root-cause-and-loss-analysis-50f3263dcc09 ，https://www.openzeppelin.com/news/exploiting-uniswap-from-reentrancy-to-actual-profit ）。**教训**：在 EVM 上"转账时执行任意外部代码"这件事，如果新老合约对"转账是否可重入"的假设不一致，就会系统性破坏一整代 DeFi 合约的安全假设——这是 Solana Transfer Hook 用"CPI 时账户降级为只读、不传递签名者权限"专门设计来规避的同一类风险（见 1.1 节）。 |
| **ERC-1363**（https://eips.ethereum.org/EIPS/eip-1363，2018-08-30） | 2018 | `transferAndCall` / `transferFromAndCall` / `approveAndCall`，转账或授权后单笔交易内回调 `onTransferReceived`/`onApprovalReceived` | 规范状态 **Final**（一手，页面同样标记"🎉 Final"）。**采用率**：⚠️存疑——本轮检索未找到官方或权威统计的"链上部署量"数据，只能确认它被 OpenZeppelin 作为可选扩展维护，未成为 launchpad/meme 代币的事实标准 | 设计本身比 ERC-777 保守得多（回调方向单一，不引入 registry 依赖），但**采用它需要接收方合约主动实现 `ERC1363Receiver`/`ERC1363Spender` 接口**，多数 DEX/AMM 路由合约按纯 ERC-20 假设编写，不会去调用这套接口，导致"有标准、没生态"——反衬出**标准本身不够不足以形成能力，还需要下游基础设施（钱包、DEX 路由、索引器）配合升级**，这恰是 Solana Token-2022 能落地而 EVM hooks 标准反复难产的关键差异：Token-2022 的钩子由**协议层强制**触发（CPI 由 Token Program 发起，不依赖接收方选择性实现），而 ERC-1363 的钩子由**发送方合约主动选择调用**，生态两端都要配合升级意愿。 |
| **ERC-1155**（Multi Token Standard，2018） | 2018 | 单合约管理多种同质化/非同质化代币，批量转账 `safeBatchTransferFrom` | Final，生态采用**集中在 NFT/游戏资产**批量管理场景 | 与"单一 meme 代币发行"场景不匹配：launchpad 需要的是"千万个独立代币各自有独立地址与独立流动性池"，而 ERC-1155 的设计初衷是"一个合约内管理多个 id 对应的资产"，二级市场（DEX/聚合器）对 ERC-1155 的路由支持远不如 ERC-20 成熟，实际上没有主流 meme launchpad 采用 ERC-1155 作为发行标准 |
| **ERC-7579**（Modular Smart Accounts，2023 起） | 2023 | 定义"模块化智能账户"标准接口（Validator / Executor / Fallback / Hook 四类可插拔模块），是**账户抽象（ERC-4337 生态）方向**的标准 | Final（截至 2026-09-07 主流账户抽象基础设施如 Safe、ZeroDev、Biconomy 已支持） | **与代币扩展是两个不同的技术方向**：ERC-7579 解决的是"钱包/账户"要不要可插拔逻辑，不是"代币本身"要不要可插拔逻辑；容易被误认为"EVM 版 Token-2022"，实际上它无法让一个已存在的 ERC-20 合约获得转账钩子——除非发行方从一开始就把"账户"做成 ERC-7579 智能账户并要求所有持有人都用该类账户接收代币，这在无许可 meme 场景不现实（用户钱包五花八门，不能强制换账户类型） |

**本节结论（对 1.5 与第 7 节的输入）**：EVM 侧不是没试过"给代币加钩子"，而是**每次尝试都撞上了同一个结构性矛盾**——ERC-20 生态的下游基础设施（DEX 路由、钱包、清算合约）数量级过于庞大且互相假设"转账是原子、无副作用的"，任何试图打破这个假设的标准都会带来**系统性重入风险**（ERC-777）或**采用率悖论**（ERC-1363：需要下游主动升级才有用，但下游没有升级动力）。这与 Solana **从创世就把 Token Program 作为唯一权威转账入口**（应用层没有绕过 Token Program 自己实现转账的选项）形成结构性对比——即"能不能加钩子"本质上不是标准设计能力问题，而是**该链的代币转账路径是否收敛到单一可控入口**的问题（见 7 节）。

### 1.3 Sui / Aptos 的 coin / object 模型

#### 1.3.1 Aptos Fungible Asset：dispatchable hooks（AIP-73）

[一手] **AIP-73「Dispatchable Token Standard」**，作者 Runtian Zhou，创建于 2024-03-08，**状态 Accepted**，参考实现已合入 `aptos-core` 主分支且在主网可用（框架层 feature）。原文：https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-073-dispatchable-token-standard.md ；讨论帖：https://github.com/aptos-foundation/AIPs/issues/374 ；官方文档：https://aptos.dev/build/smart-contracts/fungible-asset （"Dispatchable Fungible Asset (Advanced)" 一节）；框架源码：`fungible_asset.move` / `dispatchable_fungible_asset.move`（`aptos-labs/aptos-core`）。

**机制**：Hook 函数以 `FunctionInfo`（模块地址+模块名+函数名的运行时函数指针）形式存入 FA 的 Metadata object，注册必须在**创建 metadata 对象的同一笔交易内**完成（依赖不可存储的 `ConstructorRef`）：

```move
dispatchable_fungible_asset::register_dispatch_functions(
    constructor_ref: &ConstructorRef,
    withdraw_function: Option<FunctionInfo>,
    deposit_function: Option<FunctionInfo>,
    derived_balance_function: Option<FunctionInfo>,
)
```

- Hook 签名固定为 `withdraw<T: key>(store, amount, &TransferRef): FungibleAsset` 与 `deposit<T: key>(store, fa, &TransferRef)`；一旦注册，普通 `fungible_asset::withdraw/deposit` 对该资产**直接 abort**，必须走 `dispatchable_fungible_asset::withdraw/deposit`（`primary_fungible_store` 用户走该路径时无感知，框架自动分发）。
- **重入防护是 AIP-73 的核心安全设计**：hook 内部必须用 `withdraw_with_ref`/`deposit_with_ref`，MoveVM 新增运行时检查——调用栈中**禁止形成模块间回边**（模块 A 调 B 后，B 返回前不得再调回 A），从 VM 层面直接堵死 ERC-777 那类重入漏洞（对照 1.2 节 imBTC/Uniswap 事件）。
- 官方列举目标用例：通缩/税费代币、转账 allowlist、条件转账（predicated transfer）、忠诚度返佣——与 Solana Transfer Hook 用例几乎一一对应，但**安全模型不同**：Aptos 靠 VM 级调用图检查防重入，Solana 靠"CPI 时账户降级为只读、剥离 signer 权限"防重入，殊途同归。

#### 1.3.2 Sui：Coin（开放）/ Regulated Coin（合规）/ Closed-Loop Token（围栏）三层模型

[一手] 文档：https://docs.sui.io/onchain-finance/fungible-tokens/coin （Coin 标准）、https://docs.sui.io/onchain-finance/fungible-tokens/regulated-tokens （Regulated Coin）、https://docs.sui.io/onchain-finance/closed-loop-token/ （Closed-Loop Token）、https://docs.sui.io/develop/objects/transfers/custom-rules （自定义转账规则的对象模型原理）。

Sui **不做"给现有代币挂 hook"这件事**，而是用**对象能力（ability）系统**从设计上分层：

| 层级 | 对象能力 | 转账行为 | 发行方能做什么 |
|---|---|---|---|
| **Coin（开放）** | `Coin<T> has key, store` | `store` 能力 = 任何人可 `transfer::public_transfer` 自由转移、可 wrap、可存入任意应用 | 只有 `TreasuryCap<T>`（`coin::create_currency` 用 one-time witness 返回）控制 mint/burn，**没有任何转账挂钩点** |
| **Regulated Coin（合规）** | 同上 + `DenyCapV2` | 转账逻辑不变，但受**系统级共享对象 `0x403`（DenyList）**拦截 | `coin::create_regulated_currency_v2` 同时发出 `TreasuryCap`（管供应）与 `DenyCapV2`（管合规），可分持给不同实体（金库多签 vs 法务）；`coin::deny_list_v2_add` 后地址**立即不能发送、下一 epoch 起不能接收**；创建时若 `allow_global_pause=true` 还可全局暂停该币所有转账——即**链级黑名单 + 熔断**，但没有任意自定义逻辑 |
| **Closed-Loop Token（围栏）** | `Token<T> has key`（**没有 `store`**） | 不能 wrap、不能存为 dynamic field、不能自由转账，只能被账户持有；protected actions（`token::transfer`/`to_coin`/`from_coin`/`spend`）返回 **`ActionRequest` hot-potato**，必须被一个"解析交易"消费掉才能成功 | 通过 `TokenPolicy` + 可组合的自定义 **Rules**（限额、KYC 验证、仅限特定服务消费等）实现**任意发行方逻辑**，且可与 Coin 双向转换（受 policy 控制）——这是 Sui 上"围墙内代币 + 出入闸门"的标准做法 |

**object 模型如何支撑自定义转账**：核心机制是——对象只要**没有 `store` 能力**，就只有**定义该类型的模块自己**能调用 `transfer::transfer`，因此该模块可以写任意签名的自定义转账函数（收费、加锁、白名单……）；但这是**单向门**：一旦某个对象类型被赋予了 `store`（开放 `public_transfer`），**无法再收回**自定义限制。这意味着 Sui 的"可编程转账"能力**必须在代币设计之初就二选一**（要么走开放 Coin 放弃控制权，要么走 Closed-Loop Token 保留控制权但牺牲与其他 DeFi 协议的无缝组合性），不像 Solana Token-2022 的 Transfer Hook 可以给一个"看起来普通"的可转账代币加装钩子。

**对 launchpad 的意义**：Sui 模型对应"发行方要不要在**创世阶段**就决定给不给自己控制权"，与 Token-2022 "先发行、扩展位提前占好坑"的思路不同，更接近 Injective permissions module（见 2.2/5.2 节）的"namespace 一次性声明"风格。

### 1.4 Cosmos tokenfactory（Osmosis / Injective）

[一手] Osmosis 源码：https://github.com/osmosis-labs/osmosis/tree/main/x/tokenfactory （README + `types/params.go`）；Injective 源码：https://github.com/InjectiveFoundation/injective-core/tree/master/injective-chain/modules/tokenfactory ，官方 spec：https://docs.injective.network/developers-native/injective/tokenfactory/01_concepts 、`/03_messages`、`/05_params`，发行费文档：https://docs.injective.network/developers-defi/token-launch 。

| 维度 | Osmosis `x/tokenfactory` | Injective `x/tokenfactory` |
|---|---|---|
| denom 命名空间 | `factory/{creator address}/{subdenom}`，subdenom 字符集 `[a-zA-Z0-9./]`、≤44 字节 | 相同格式：`factory/{creator}/{subdenom}`，同样 ≤44 字符 |
| 创建费用 | `DenomCreationFee` 当前默认**空（0）**（注释写"used to be 10 OSMO at launch"，即上线时收 10 OSMO，后改为免费）+ `DenomCreationGasConsume = 1,000,000` gas（防 spam 手段从"收费"改为"烧 gas"）；改动依据 PR https://github.com/osmosis-labs/osmosis/pull/4983 | `denom_creation_fee` **当前主网 0.1 INJ**（官方文档原文："The fee for creating a factory denom is 0.1 INJ"），无 gas-consume 参数；费用打入 community pool |
| Admin 权限（创建者默认为 admin） | 可 mint 到任意账户、burn 任意账户余额、强制在任意两账户间转移（force transfer）、`SetDenomMetadata` 覆写元数据；`ChangeAdmin` 可转移或设为 `""`放弃 | `MsgCreateDenom` 携带 name/symbol/decimals；有 `allow_admin_burn` 标志——**admin 默认不能烧他人代币，须创建时显式启用**（与 Osmosis 默认允许相反），且可用 `MsgSetDenomMetadata.AdminBurnDisabled` 事后单向关闭；官方建议 mint 完初始供应后将 admin 改为零地址 |
| before-send hook | **有**：Osmosis 在自己 fork 的 cosmos-sdk 里给 bank 加了 `BlockBeforeSend`（阻断式，返回 error 取消整笔转账，module→module 转账不触发）与 `TrackBeforeSend`（旁路式，錯誤被静默，限 100,000 gas）两个钩子（PR：https://github.com/osmosis-labs/cosmos-sdk/pull/421 ）；每个 denom 可注册一个合约地址（`MsgSetBeforeSendHook`），触发时以 **sudo** 调用该 CosmWasm 合约的 `SudoMsg::BlockBeforeSend/TrackBeforeSend` | **没有**：Injective tokenfactory 的消息集中不含 `SetBeforeSendHook`，这是 Osmosis fork 特有能力；Injective 走的是另一条路——permissions 模块做受限资产（见 2.2/5.2 节） |

**对 launchpad 的意义**：Osmosis 的 `BlockBeforeSend` hook 是本报告中**除 Token-2022 Transfer Hook 外唯一的"链级模块级转账钩子"**先例——性质上等价于"给一个 Cosmos denom 挂一个可编程转账校验器"，落地形态是**系统合约 + precompile 级**（模块提供钩子入口，具体策略仍是合约代码）；Injective 选择完全不做这层，转而用更重的 permissions/namespace 机制覆盖合规场景，两条路径分别对应"轻量可选钩子"与"强制声明式权限系统"两种设计哲学，都比 EVM 原生 ERC-20（无任何钩子）能力更强。

### 1.5 结论：EVM 缺的到底是哪几项能力

把 1.1–1.4 节的发现收敛为**能力清单**（详细版见第 7 节，此处先给结构性小结）：

1. **协议强制的转账钩子**——Solana Transfer Hook（协议级 CPI 强制触发）、Aptos AIP-73（VM 级函数指针分发）、Osmosis before-send hook（模块级可选钩子）三条路径都做到了"转账时执行发行方自定义逻辑且防重入"，EVM 三次尝试（777/1363/1155）均未成为主流事实标准；
2. **协议原生的转账税/手续费路由**——Solana `TransferFeeConfig` 做到"暂扣后归集"，EVM 上只能在合约 `_transfer` 里手写百分比扣费，且必须每个代币各自实现；
3. **协议原生的合规闸门（黑名单/白名单/冻结）**——Solana `DefaultAccountState`/`PermanentDelegate`、Sui Regulated Coin（DenyList 系统对象）、Injective permissions（namespace RBAC）、Provenance marker（`required_attributes`）都做到了"不用发行方自己写合约就能冻结/加白名单"，EVM 上这些都是**每个代币各自实现**的合约逻辑，没有协议层统一原语；
4. **协议原生的元数据/分组**——Solana `MetadataPointer`/`TokenGroup` 让 mint 账户自描述，EVM 上元数据靠链下约定（ERC-20 没有标准化 on-chain metadata 字段，NFT 侧的 ERC-721 metadata URI 也只是链下 JSON 指针）；
5. **这些能力能否用 precompile/系统合约/改 EVM 补上**——**部分能补，部分补不了**：转账钩子和费用路由理论上可以用**系统合约 + precompile**方式补（例如在 EVM 层加一个"标准化 hook 注册表"precompile，强制所有 ERC-20 `transfer` 走一次 precompile 校验），但这需要**改共识层的 EVM 执行语义本身**（让"转账"这个操作不再是普通 `CALL`，而是被协议拦截的特权操作）——这已经不是"加个库"能做到的，而是**改 EVM 执行层**（第 7 节详细展开四档落地形态的判定）。

## 2. 链级原生发行与上币额度

### 2.1 Hyperliquid HIP-1 / HIP-2 / HIP-3：发行经济学视角

[一手] 官方文档：HIP-1 https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-1-native-token-standard ；HIP-2 https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-2-hyperliquidity ；HIP-3 https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals ；部署 API：https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/deploying-hip-1-and-hip-2-assets 、https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/hip-3-deployer-actions-1 。（注：本轨道从"发行经济学"角度拆解，机制层与撮合/费用结构细节 → 交给 J 轨道。）

#### 2.1.1 HIP-1：荷兰拍卖如何充当反垃圾闸门

HIP-1 = 固定上限供应（capped supply）代币标准 + 原生链上现货订单簿（默认对 USDC 报价）。**上币额度本身是稀缺资源**，用**连续滚动的荷兰拍卖**分配：

| 拍卖参数 | 官方数值 | 反垃圾/定价机制 |
|---|---|---|
| 拍卖周期 | **31 小时**一场，连续滚动（一年约 282 个额度） | 用**时间稀缺性**替代人工审核——不是"审批上币"，而是"排队 + 付费"上币，闸门是经济成本而非白名单 |
| 起拍价 | `initial_price` = 上一次成交价 **× 2**；若上一场流拍则从底价重新起拍 | 价格随需求自适应：热度高则起拍价指数上升，天然抑制连续投机性上币；流拍则价格重置，防止价格永久失控飘高 |
| 底价 | 线性降至 **500 HYPE**（当前地板价） | 保证机制永不停摆——即使无人竞价，仍有最低价可选，避免"高热度期后无人问津导致额度彻底冻结" |
| Gas 收取时点 | 第一步 `registerToken2`（锁 ticker/decimals）时收取，**卡死不退款** | 强制部署方在测试网先演练（https://app.hyperliquid-testnet.xyz/deploySpot ），把"操作失误成本"也计入反垃圾闸门 |
| Anchor token 空投门槛 | 持仓需 ≥ anchor token 最大供给的 **0.0001%** 才有 genesis 份额 | 防"尘埃地址"批量薅空投份额，是**发行侧**（而非拍卖侧）的反女巫设计 |
| 状态膨胀费 | 每个 `(address, token)` 首次状态初始化收一次性小额 USDC gas | 把"链上状态膨胀"这一外部性直接定价给使用者，而非全网络分摊 |

**历史成交价数据**（[二手]，官方不发布历史价格页，可用链上 `spotDeployState` 复核）：2024-12-06 `SOLV` ≈ **$128,000**（https://m.theblockbeats.info/en/news/56389 ）；2024-12 中旬 `ANIME` ≈ **$530,000**（https://www.chaincatcher.com/en/article/2158389 ）；2024-12-16/17 `GOD`（Infinigods）≈ **$975,700**，历史峰值，导致下一场起拍价翻倍至 ≈$1.95M（https://www.bitget.com/news/detail/12560604426855 ）；早期多数拍卖在接近底价成交。第三方逐场看板：https://data.asxn.xyz/dashboard/hl-auctions 。

**发行经济学解读**：这套机制把"上币"从"项目方求交易所"倒转为"交易所对上币权做二级市场定价"——额度价格本身就是市场对"该上币窗口期注意力"的估值，价格越高说明市场认为这个时间点的发行会吸引越多交易量（进而产生手续费收入归 deployer）。这与传统 CEX 上币费的核心区别是：**费用不是给平台的固定门槛，而是付给"下一个部署者"的荷兰拍卖出清价**（因为 `initial_price` 由上一次成交价决定），使得拍卖成为一个自我调节的"注意力定价市场"。

#### 2.1.2 HIP-2：原生做市如何改变"冷启动流动性"经济学

Hyperliquidity 是**共识层内置的自动订单簿做市策略**，属于区块状态转换逻辑本身，**没有 operator**，不依赖任何用户交易维护。价格网格 `px_0=startPx`，`px_i=round(px_{i-1}×1.003)`（0.3% 档间距），每 ≥3 秒更新一次，部署者按 `nSeededLevels` 档位注入 USDC 铺单。**发行经济学意义**：项目方不再需要在 Uniswap 类 AMM 池子里预置流动性对（承担无常损失与被狙击风险），而是把"做市成本"转化为**一次性协议内注资**，做市逻辑本身由验证人共识保障、不可被抢跑操纵挂单顺序。目前仅支持 USDC 报价对，桥接资产/RWA 部署者可用 `noHyperliquidity` 关闭。

#### 2.1.3 HIP-3：保证金门槛如何筛选发行人

HIP-3 允许任何满足质押门槛的地址**无需许可地部署自己的永续市场**（builder-deployed perpetuals），核心是用**资本门槛**替代"团队审核"作为发行人筛选器：

| 门槛/规则 | 数值 | 筛选逻辑 |
|---|---|---|
| 质押要求 | **500,000 HYPE**（主网当前值，预期随基础设施成熟下调） | 用"要放多少钱在桌上"取代"你是谁"——门槛越高，恶意/短期套利部署者的机会成本越大 |
| 最短维持期 | 部署后至少维持 **183 天** | 防止"部署即跑路"，把长期运营意愿写进质押锁定期 |
| 每 deployer 限额 | 目前限 **1 个 perp dex**（未来可能允许共享多质押部署多个） | 限制单一资本对市场供给的垄断，同时避免"批量注册占坑" |
| 免拍卖资产数 | 每个 dex **前 3 个资产**免拍卖，之后走与 HIP-1 相同的跨 dex 共享荷兰拍卖 | 给新 dex 冷启动一定自由度，同时防止拍卖被单一 dex 长期垄断 |
| Slashing | 无效状态转换/长时间宕机最高罚没 **100%**质押，短暂宕机最高 50%，性能劣化最高 20%，罚没资金**销毁**不赔付用户 | 把"运营质量"直接与真实资本挂钩，且惩罚不进国库而是销毁——避免"惩罚即协议收入"的激励扭曲 |

**生产状态**：[一手/二手] 2025-10-13 主网上线（官方 docs 以 mainnet 表述现行 spec；二手交叉：https://nansen.ai/post/what-is-hip-3-hyperliquid ，https://www.coingecko.com/learn/hyperliquid-hip3-hip4-tokenized-stocks-and-prediction-markets ）。2026 年官方推进 HIP-3 permissioned markets（链上 allowlist）扩展与 HIP-4（结果市场，2026-05 主网）——**这是路线图与既成事实的分界点，本报告仅确认 HIP-3 基础版本已上线，permissioned markets 扩展需进一步核实当前状态** ⚠️存疑。

**发行经济学总结（三者合一）**：HIP-1 用"付费荷兰拍卖"筛出愿意为上币窗口出高价的现货代币发行人；HIP-2 把做市成本从"应用层流动性博弈"转移到"协议内一次性注资"；HIP-3 用"高额可罚没质押"筛出愿意长期运营衍生品市场的发行人——三者共同点是**全部把"谁有资格发行"这个准入问题转化为可编程的资本/时间门槛，而非链下审核**，这正是"issuance-native"最直接的可抄经济学模式，但门槛数字（31h、500k HYPE、183 天）都是针对 Hyperliquid 自身生态规模校准的，照抄到别的链需要重新校准。

### 2.2 Injective permissions / RWA module
[一手] 模块总览：https://docs.injective.network/developers-native/injective/permissions ；核心概念：https://docs.injective.network/developers-native/injective/permissions/01_concepts ；State/State Transitions/发行教程：`/02_state`、`/03_state_transitions`、`/04_launch_permissioned_asset`；TokenFactory：https://docs.injective.network/developers/modules/injective/tokenfactory ；源码路径 `injective-chain/modules/permissions`（`github.com/InjectiveLabs/injective-core`，本轮未能直抓仓库页面，经 Code4rena 审计镜像 `github.com/code-423n4/2026-02-injective` 交叉确认）。官方营销称之为 "RWA module"，随 **Volan 主网升级（2024-01）**上线：https://injective.com/blog/a-new-era-of-injective-the-volan-mainnet-upgrade 。

**核心机制**：每个 TokenFactory denom 可建**一个 namespace**（创建者须为该 denom 的 tokenfactory admin），mint/burn/send/receive 时**链级强制检查**——这是本报告目录里少数几个"转账规则由共识层而非应用合约执行"的 EVM 可比先例。

| 能力 | 机制 | 落地形态 |
|---|---|---|
| 动作分级 | `MINT`(1) / `RECEIVE`(2) / `BURN`(4) / `SEND`(8) / `SUPER_BURN`(16) 位掩码编码，`MINT` 只能 mint 给有 `RECEIVE` 权限的地址，`SEND` 只能发给有 `RECEIVE` 权限的地址 | 系统合约 + precompile 级（Cosmos SDK 原生 module） |
| KYC allowlist | 默认 `EVERYONE` 角色（创建时必须显式定义）；若不给 `EVERYONE` 赋 `RECEIVE`/`SEND`，则只有被 Role Manager 显式赋角色的地址能收/发——**链级白名单** | 系统合约 + precompile 级 |
| 冻结 | 零权限角色（blacklist role）赋给某地址后**覆盖其全部权限**（等效冻结），移除后恢复 | 系统合约 + precompile 级 |
| 全局熔断 | Policy Status 可全局 disable 某个动作（如暂停全网转账），`seal` 后不可逆固化 | 系统合约 + precompile 级 |
| 合约钩子 | 接收资产时可触发 Wasm contract hook | 合约层（挂在链级 module 之上） |
| 到账失败兜底 | 模块转账因目标无 `RECEIVE` 权限失败时生成 **voucher**，待获权后 `MsgClaimVoucher` 领取 | 系统合约 + precompile 级 |

**主网使用实例**（[一手/二手]）：
- **Ondo USDY**：2024-01 随 Volan 升级登陆，官方列为 RWA 生态首发实例——https://injective.com/blog/ondo-launches-the-worlds-first-interoperable-tokenized-treasuries-on-injective-to-accelerate-rwa-adoption
- **Agora AUSD**：2024-10-31 作为 Injective 首个**原生发行**（非桥接）稳定币上线——https://injective.com/blog/agora-to-launch-native-us-dollar-stablecoin-on-injective ；⚠️存疑：[二手] 已有报道称 Agora 于 2026-04 宣布停止在 Injective 上的 AUSD 发行并进入赎回期，本报告未找到官方一手公告确认该退出的具体原因，计入存疑清单。
- **"Injective Mint"**：官方基于 permissions module 的机构化合规发行入口（配置白名单、辖区限制、角色化 mint/burn/freeze）——https://injective.com/blog/inside-injective-mint ；官方 tokenization 白皮书：https://injective.com/Tokenization_on_Injective.pdf

**对 launchpad 的意义**：permissions module 证明"一级 KYB 闸门"（namespace 白名单）**可以做成链级原语**且已有机构资产（USDY/AUSD）生产验证，但它是**为合规资产设计**的重量级方案（namespace/角色/policy 三层配置），不是为"无许可 meme 秒发"设计的轻量原语——第 5.4 节将展开讨论这套机制能否"降维"用于无许可发行场景。

### 2.3 链级 bonding curve 先例检索（ixo x/bonds 等）

**结论先行：链级 bonding curve 模块确有先例，最完整的一个是 `ixofoundation/ixo-blockchain` 的 `x/bonds` 模块**，这是本轨道最重要的发现之一。

#### 2.3.1 ixo `x/bonds`：Cosmos SDK 原生 bonding curve 模块

[一手] 源码：https://github.com/ixofoundation/ixo-blockchain/tree/main/x/bonds ；spec 文档：https://raw.githubusercontent.com/ixofoundation/ixo-blockchain/main/x/bonds/spec/01_concepts.md 、`03_messages.md` 等（2026-09-07 直接读取源码原文）。ixo 是一条面向"影响力资产/Impact Bonds"的 Cosmos SDK 应用链，`x/bonds` 是其**运行时原生模块**（不是部署在链上的 CosmWasm 合约），随链二进制编译进共识层。

官方原文（`01_concepts.md`）：
> "The Token Bonds Cosmos SDK Module enables applications that use token bonding curves to be created on-the-fly. Each new Token B instance declares a new token denomination in the application... Buy instructions cause bond tokens to be minted during a state transition... Sell instructions burn bond tokens during a state transition."
中译：Token Bonds 模块让应用能够**即时创建**基于 bonding curve 的代币；每个新 bond 实例在链上声明一个新的代币面额；买入指令在状态转换时铸造 bond 代币，卖出指令则销毁代币——**mint/burn 权限直接内嵌在共识层状态机里，不需要应用层合约持有铸币权**。

**机制细节（逐项对照 launchpad 需求）**：

| 机制 | `x/bonds` 原生实现 | 对 launchpad 的意义 |
|---|---|---|
| **发行入口** | `MsgCreateBond`：任意地址可发送该交易创建一个新 bond，一笔交易内声明代币面额（`Token`）、名称、描述、曲线类型（`FunctionType`）、曲线参数（`FunctionParameters`）、储备代币（`ReserveTokens`）、费率（`TxFeePercentage`/`ExitFeePercentage`）、最大供给（`MaxSupply`）等——**发行 = 一笔链级交易，不是"部署合约+调用初始化+建池"三步** | 对应第 4.2 节"原子发行"诉求的**链级模块先例**：EVM 上做同样的事需要工厂合约 `deploy` + 建池 + 转移流动性三笔交易（或用 multicall 打包，但仍是合约层拼接，不是协议层原生状态转换） |
| **曲线类型** | 官方支持 4 种：`power_function`（形如 `m·x^n+c`）、`sigmoid_function`、`swapper_function`（无定价函数、由首笔买单确定汇率的"代币互换器"）、`augmented_function`（增强曲线，含 `HATCH`/`OPEN` 两阶段，`d0`/`p0`/`theta`/`kappa` 四个参数控制初始定价与"孵化"门槛） | `augmented_function` 的 `HATCH` 阶段本质上是一种**链级"孵化期固定价"机制**——在 `HATCH` 状态下所有买入按固定单价 `p0` 成交，不随供给变化，直到达成孵化条件转为 `OPEN`（自由定价）。这与 launchpad 常见"种子轮固定价、毕业后浮动定价"的两阶段设计**结构同源**，但由链原生状态机而非应用合约保证 |
| **抗 MEV / 反狙击** | **批量拍卖（batching）机制**：每个 bond 有一个 `BatchBlocks` 参数定义订单批次的生命周期（以区块数计）；同一批次内的所有买单/卖单先累加汇总，在批次结束时**用统一价格**一次性结算，而非"先到先得"的逐笔即时定价 | 官方原文："The primary task of the batching mechanism is to find a common price for all of the buys and sells submitted to the batch by summing up all of the buys and sells, thus ignoring their order, and matching-up the total buy and sell amounts to give balanced and fair global buy and sell prices." 中译：批处理机制的核心任务是**忽略订单到达顺序**，把同批次内所有买卖累加后算出统一的"公平"成交价——这是一种**协议层原生的抗抢跑（front-running）设计**，直接回应第一阶段"抗狙击"痛点，且是**链级模块**而非应用层的"隐藏 mempool"或"commit-reveal"方案 |
| **生命周期状态机** | `Bond.State` ∈ `{HATCH, OPEN, SETTLED, FAILED}`，合法转移仅 `HATCH→OPEN|FAILED`、`OPEN→SETTLE|FAILED`，`SETTLE`/`FAILED` 为终态；状态转移由 `MsgUpdateBondState` 触发，权限限定为 bond 的 `ControllerDid` | SETTLE 后"outcome payment reserve"移入 bond reserve，供代币持有人按持仓比例赎回——即**协议原生支持"退出清算"**，不需要应用层额外写一个 vesting/清算合约 |
| **储备与费用** | `TxFeePercentage`（买卖手续费）+ `ExitFeePercentage`（仅卖出时叠加）在创建时声明，费用直接进入 `FeeAddress`；`ReserveWithdrawalAddress` 声明储备提取地址；`AllowSells`与`AllowReserveWithdrawals` 互斥（二选一，防止"既能提走储备又能随时抛售"的双重掏空） | 费用路由原生化的**协议层先例**（对应第 4.3 节）：手续费分账规则是 bond 元数据的一部分，由状态机强制执行，不依赖应用合约在每笔转账里手动 `transfer()` 给创作者地址 |
| **反前置狙击的价格保护** | `MsgBuy` 携带 `MaxPrices`（愿意支付的最高价），若批次结算价超过该值，订单**在批次内自动取消**而非等到链下滑点保护失效才 revert | 用户体验上等价于"链级滑点保护"，但生效机制是批次汇总定价而非单笔即时 AMM 曲线，本质上**没有单笔可被夹的 mempool 时刻** |

【实测】**规模与生产状态**：2026-09-07 直接查询 ixo 主网（chain-id `ixo-5`，Cosmos 官方 chain-registry 登记为 live：https://raw.githubusercontent.com/cosmos/chain-registry/master/impacthub/chain.json ）REST 端点 `https://impacthub.ixo.world/rest/ixo/bonds/bonds_detailed`（GET，无需参数），返回**仅 4 个 bond**：`did:ixo:bond-atk1`（供给=1，储备=1 单位某 IBC 资产）、`bond-atk3`（供给=1）、`bond-atk8`（供给=0，储备空）、`bond-attacker`（供给=0，储备空，命名本身即显示是测试/探测数据）。**结论：模块代码在主网真实部署且查询端点/消息/状态机全部可用，但当前主网没有具规模的生产级 bonding curve 案例，现存 bond 均为粉尘级测试数据**；ixo 官方宣传的 Alphabond/风险调整曲线设计（与 BlockScience 合作，https://medium.com/ixo-blog/cosmic-bonding-4f948dd4c2e4 ，[二手]）主要停留在设计文档/试点层面，`spec/07_future_improvements.md` 也承认 IBC 储备等仍是未来工作。**这本身是一个重要发现：技术先例存在且完整，但从未被规模化使用**，需要与第 7 节"为什么没被复制"合并讨论。

#### 2.3.2 检索范围与"是否孤例"判断

检索范围：Cosmos SDK 模块生态（`cosmos-sdk` 核心仓库 + 主要应用链自定义模块，如 Osmosis/Injective/Sei/Terra/Regen/ixo）、Substrate pallet 生态（`paritytech/polkadot-sdk` 官方 `frame` 目录 + 主要 parachain 官方仓库，如 SORA、Polimec）、Move 生态（Aptos/Sui 官方标准库模块）、以及 Penumbra（ZK 应用链，批量拍卖但非 bonding curve）。检索方式：GitHub 源码目录直接核验（非仅新闻/二手转述）。

**修正后判断（Substrate 结论与初版检索相反，本节以修正版为准）**：Substrate 生态**确实存在链级 bonding curve / launchpad pallet 先例，且不止一个**，本报告此前一版检索范围不足导致漏判，特此更正：

| 项目 | 类型 | 一手来源 | 机制要点 |
|---|---|---|---|
| **SORA `multicollateral-bonding-curve-pool`** | Substrate **runtime pallet**（非合约） | https://github.com/sora-xor/sora2-network/tree/master/pallets/multicollateral-bonding-curve-pool ；机制说明 https://wiki.sora.org/tbc.html | 官方原文："the token bonding curve is essentially an infinitely liquid, decentralized central bank... you can buy newly minted XOR from the token bonding curve using specific reserve assets, or sell your XOR tokens (which are instantly burned)"——即**单一原生币 XOR 的货币政策机制**，买卖两条线性价格函数、价格随总供给变化，写死在 runtime 里，**不支持任意用户按需发行新曲线代币**（这点与 ixo x/bonds 的"通用发行"定位不同） |
| **SORA `ceres-launchpad`** | Substrate runtime pallet | https://github.com/sora-xor/sora2-network/tree/master/pallets （目录同级确认存在） | 与 TBC pallet 同一 runtime 内的**链级 launchpad pallet**，是本报告检索到的、"launchpad 逻辑做成 runtime 原生模块"最接近的完整先例 |
| **Polimec** | 整条 Polkadot 平行链即原生 launchpad | https://github.com/Polimec/polimec-node/tree/main/pallets （`funding`/`linear-release`/`proxy-bonding` 等目录已核实存在） | 募资/线性释放/委托质押全部实现为 runtime pallet，是"把 launchpad 做成 app-specific chain"的**Substrate 版对照案例**（与本研究"issuance-native chain"命题高度同构，→ 交给 J 轨道做横向对标补充） |

**Cosmos 侧的其余排除项**：
- **Bancor / Continuous Organizations 论文**（[二手] 早期理论文章，如 https://medium.com/@simondlr/curved-token-bonding-curves-in-solidity-6d0e1d6f6d6b 一类，均为**以太坊合约层**实现，非链级模块，不计入先例；
- **Terra 的 `x/market` 模块**（Terra Classic 时代，UST/Luna 铸造互换）：函数形态是恒定汇率互换而非连续曲线定价，且随 2022-05 Terra/UST 脱锚崩溃已成失败案例，本报告**不计入"健康先例"**，仅作为"链级铸造/销毁模块可能带来系统性风险"的警示（⚠️存疑：脱锚崩溃与 `x/market` 模块设计本身的因果关系需要更严谨的事后分析，本报告未展开）；
- **Move 生态（Aptos/Sui）**：未发现官方标准库层面的 bonding curve 模块，均以"发布在链上的普通 Move 模块（用户自定义，非协议内置）"形式存在，边界判定与 CosmWasm 合约类似（见 2.4 节）。

**最终结论**：
1. 严格意义的「**通用、面向任意发行方的链原生 bonding curve 发行模块**」——检索范围内 **ixo `x/bonds` 仍是唯一的"任意用户可发行任意曲线代币"的通用实现**，但如 2.3.1 节【实测】所示，其主网实际使用规模近乎为零；
2. 放宽为「**任何写进链运行时的 bonding curve / launchpad 机制**」——则至少有三例：ixo `x/bonds`（Cosmos，通用但零使用）、SORA `multicollateral-bonding-curve-pool`（Substrate，单币货币政策，主网使用中）、SORA `ceres-launchpad` + Polimec `pallet-funding`（Substrate，launchpad 语义的 runtime 原生实现）；
3. **生态分布明显偏向 Substrate**：Cosmos 生态虽然模块化程度高，但除 ixo 一例外未见其他链把"发行定价曲线"做进 runtime；Substrate 生态反而有多个独立项目（SORA、Polimec）做了这件事——这本身是一个值得记录的现象：**"能不能做进协议层"与"生态是否倾向于这样做"是两个独立变量**，EVM/Cosmos 生态的主流文化是"能放合约就不动运行时"，而部分 Substrate 应用链文化更倾向于"深度定制 runtime"，这与 Substrate 从设计上就鼓励"每条链自己攒 pallet"的模块化哲学有关；
4. 本报告检索局限：以英文公开资料 + GitHub 源码为准，未穷举全部长尾 appchain；私有链/未公开文档实现不在覆盖范围。

### 2.4 Osmosis CosmWasm launchpad：链级模块 vs 链上合约边界

[一手] 三个已核实的具体例子，正好展示三种"模块 vs 合约"分界方式：

1. **StreamSwap —— 纯合约层 launchpad**：https://github.com/StreamSwapProtocol/streamswap-contracts 。时间流式代币发售（连续出售、价格由订阅供需决定，替代 ICO/LBP），架构为 controller 合约 + stream 合约，生命周期 `Waiting→Bootstrapping→Active→Ended→Finalized/Cancelled`，结束后可自动建池与 vesting。官方 README 列出已部署地址：**osmosis-1 主网 3 个实例 + injective-1 一个实例**——证明发售逻辑完全在合约层完成，只依赖 bank/tokenfactory 提供的 denom 原语。
2. **`cw-tokenfactory-issuer` —— 模块原语 + 合约策略的分界样板**：https://github.com/osmosis-labs/cw-tokenfactory-issuer 。合约成为某个 tokenfactory denom 的 admin，并把自己注册为该 denom 的 `BlockBeforeSend` hook（见 1.4 节），在模块级 mint/burn/转账原语之上用合约实现央行式稳定币策略（铸造额度授权、冻结、黑名单）。
3. **`x/cosmwasmpool` —— 链级模块"收编"合约作为池实现**：https://github.com/osmosis-labs/osmosis/tree/main/x/cosmwasmpool 。设计动机官方写得很直白：池逻辑放合约可以免链上升级、快速迭代；但做成"模块+合约"混合体，让合约实现 `poolmanager` 的 `PoolI`/`PoolModule` 接口，从而免费获得链级 swap 路由、多跳、跨链 swap；swap 通过 **sudo**（仅链可调用）执行以保证 `swap_fee` 可信。示例：transmuter 合约 https://github.com/osmosis-labs/transmuter 。

**边界结论**：Osmosis 官方对"链级模块 vs 链上合约"的实际表态是——**资产发行（tokenfactory）与路由/结算（poolmanager）留在模块层，定价曲线与发售策略下放到合约层**。Osmosis 生态**没有链级 launchpad 模块**：发售逻辑（StreamSwap）全部在 CosmWasm，链只提供 denom 工厂、before-send hook、池路由三个模块级原语。这与 2.3.2 节 SORA/Polimec 的选择（把 launchpad 逻辑本身做进 runtime pallet）形成**同一问题的两种答案**，本质是治理哲学差异：Cosmos SDK 应用链文化倾向"能放合约就不动运行时"（模块升级需要治理+硬分叉级协调成本），Substrate 文化更倾向"深度定制 runtime"（因为 Substrate parachain 本身单位增量升级成本较低、且被设计为鼓励定制）。**这条对 issuance-native 设计的启示是**：判断某能力该做成"系统合约/precompile"还是"改执行层"，参考系不该是"技术上能不能"，而是"这条链的治理/升级机制能否承受模块级代码经常变动"。

## 3. 原生流动性与激励

本节视角从"发行"转向"发行后如何原生获得流动性/激励"，同样区分"链级模块"与"应用层激励"。

### 3.1 链级 AMM / DEX 模块：Osmosis GAMM、Injective exchange module

[一手] Osmosis GAMM 源码：https://github.com/osmosis-labs/osmosis/tree/main/x/gamm ；poolmanager：https://github.com/osmosis-labs/osmosis/blob/main/x/poolmanager/README.md ；concentrated-liquidity：https://github.com/osmosis-labs/osmosis/tree/main/x/concentrated-liquidity 。Injective exchange 模块文档：https://docs.injective.network/developers-native/injective/exchange/index 、EndBlocker/FBA 细节 https://docs.injective.network/developers-native/injective/exchange/08_end_block 。

**Osmosis GAMM**：Balancer 加权池（2–8 种资产，权重上限 2²⁰，支持 `SmoothWeightChangeParams` 平滑改权）+ Stableswap 池，LP share denom 为 `gamm/pool/{poolID}`；`PoolCreationFee` 官方 README 载明当前**100 OSMO**（防垃圾建池）。**已被部分取代**：`MsgSwapExactAmountIn/Out` 已弃用并迁移到 `x/poolmanager`（多跳路由统一入口）；引入 `x/concentrated-liquidity`（Uniswap v3 式集中流动性，Osmosis 称"Supercharged Liquidity"）后，GAMM README 中的 **Migration Records** 机制允许每个 balancer 池经治理与至多一个 CL 池建立 canonical link（`BalancerToConcentratedPoolLink`）用于 LP 迁移。**现状**：GAMM 仍在线，继续承载 balancer/stableswap 池，但主要交易对流动性已迁往 CL 池，架构上现在是 `poolmanager`（路由）+ 三类池模块（gamm / concentrated-liquidity / cosmwasmpool）。

**Injective exchange 模块**：官方称之为链的"sine qua non"（不可或缺之物），支持 Spot 与衍生品市场（永续/交割/二元期权），订单簿维护、撮合、结算全部在链上模块内完成，并与 auction、insurance、oracle、peggy 模块深度耦合。**撮合机制是 Frequent Batch Auction（FBA）**，在 EndBlocker 中执行：市价单先与"区块开始时"的挂单簿成交，所有市价单以**统一清算价**成交；限价单同样按统一清算价批量撮合（清算价取最优买/卖价、标记价格或中间价三种情形之一）——这与 2.3 节 ixo `x/bonds` 的批量结算、3.5 节将提到的 Penumbra 批量拍卖是**同一族抗抢跑设计**（均可追溯到 Budish/Cramton/Shim 的 frequent batch auction 市场设计理论）。

**结论**：Osmosis 把"发行"（tokenfactory）与"撮合/流动性"（GAMM/poolmanager/CL）分置于不同模块但都留在链级；Injective 走得更远，把整个订单簿撮合逻辑做进链级模块。两者都证明"链级 AMM/订单簿"是**成熟先例**，不是新鲜事——真正的分歧点在"发行"与"定价策略"要不要也进模块（见 2.4 节），而非"要不要原生流动性模块"本身。

### 3.2 负面案例：Sei DEX module 弃用

**这是本报告要求的负面案例，归因需要精确表述——官方从未把它归因为"链级订单簿模块设计本身失败"，而是把它裹挟进一次更大范围的"整条 Cosmos 技术栈弃用"**，这个区分本身就是一个值得记录的发现。

[一手] Sei 原生 `x/dex` 模块曾是"链上原生 CLOB"的核心卖点（配合 `wasmd`/CosmWasm 绑定，供开发者构建交易应用）。2025-05-07，Sei Labs 维护者 `philipsu522` 在官方治理仓库发起 **SIP-3「Deprecating CosmWasm & Cosmos」**：https://github.com/sei-protocol/sips/discussions/9 （讨论帖标题即为该提案名）。提案内容是**弃用整个 Cosmos 原生技术栈**（CosmWasm 智能合约、原生 Cosmos 账户体系、IBC 互操作），全面转向 **EVM-only 架构**，`x/dex` 模块作为该技术栈的一部分被一并弃用/边缘化，**而不是因为订单簿本身设计失败被单独下线**。

官方归因（综合 SIP-3 讨论帖与后续官方沟通，[二手] 交叉验证 https://cryptorank.io/news/feed/d0181-sei-proposes-shift-to-evm-only-model ）：
1. **开发者/用户已用脚投票**：链上超过 80% 的交易发生在 EVM 环境（Sei 此前已是 Cosmos+EVM 双栈链），维护一条几乎没人用的 Cosmos 原生路径的边际成本已超过收益；
2. **双栈维护成本**：两套执行环境（Wasm 与 EVM）要求每次协议升级都做两遍测试与开发，增加代码复杂度、维护负担与攻击面；
3. **生态割裂**：两套地址格式（bech32 vs 0x）、两套工具链，分流了本就有限的开发者注意力，且更成熟的 Ethereum 工具链（Hardhat/Foundry/ethers.js）对开发者吸引力更强。

**归因精确表述**：SIP-3 讨论帖原文（2026-06-19 更新条目提及范围）与 GitHub 讨论中官方未提供"链级 CLOB 模块本身有什么设计缺陷"的技术性归因；相反，SIP-3 提案文本聚焦于**生态资源配置**（"consolidate developer efforts, reduce tooling fragmentation"）。⚠️存疑：本轮检索**未找到**官方对"`x/dex` 模块本身使用率/性能数据"的专门披露，因此无法证实"CLOB 模块本身是否曾经被充分使用过"，只能确认它随整个 Cosmos 栈一起被放弃这一事实。

**对 issuance-native 设计的教训**：这是一个关于**"链级模块 vs 生态网络效应"**的负面案例——即使某个链级模块（原生 CLOB）在**技术设计**上被认为是先进的（Sei 早期以此为核心卖点），如果它所依附的**执行环境/工具链生态**在与更主流生态（EVM）的竞争中失去开发者心智，模块本身再精巧也会被连带牺牲。这提示：为 Mantle 设计 issuance-native 模块时，**必须先确认底层执行环境（EVM 兼容性）不会被牺牲**，否则模块设计得再好也可能重蹈"起了个大早、赶了个晚集"的覆辙。

### 3.3 Berachain PoL（Proof of Liquidity）

[一手] 官方文档总览：https://docs.berachain.com/general/proof-of-liquidity/overview ；变更记录：https://docs.berachain.com/general/proof-of-liquidity/changelog ；BGT 代币页：https://docs.berachain.com/general/tokens/bgt 。

**原始机制（2025 年设计，历史参考）**：三代币模型 $BERA（质押保安全）/ $BGT（治理+排放路由）/ $HONEY（稳定币）。验证人质押 BERA 出块，出块获得 BGT 排放，**验证人决定把 BGT 排放路由给哪些 Reward Vault**；用户"boost"验证人（委托 BGT）以增加其排放权重；协议向验证人提供"incentive"（贿选，常以自身代币或 HONEY 支付）换取验证人把排放导向自己的 Vault；用户在 Reward Vault 里质押"收据代币"（如 DEX 的 LP token）赚取被导向该 Vault 的 BGT——**本质是把"流动性挖矿"的激励分配权从协议自己的排放表，转移给验证人的市场化竞价**，是"链级流动性激励"最具原创性的先例之一。

**⚠️重大变更（本报告发现，写入时需高亮）**：**BGT 已于 2026-07-08 正式弃用**，官方公告：https://x.com/berachain/status/2074901414149771549 （官方文档确认："BGT was deprecated on July 8, 2026"）。此后 Berachain 升级为 **"PoL Next"** 单代币模型：
- 排放不再产出 BGT，而是直接产出 **$WBERA**；验证人仍决定 WBERA 排放路由给哪些 Reward Vault（`BeraChef` 管理分配），但不再有"BGT 委托/boost"这一层——即**去掉了双代币模型里"用户先获得治理代币、再委托给验证人"的中间环节**；
- 用户改为直接质押 $BERA/$WBERA 进 **$sWBERA**（Staked WBERA）金库，赚取来自 **Incentive Auction** 的收益：协议方付给验证人的 incentive token 经拍卖换成 $BERA，注入 sWBERA 收益；
- 官方明确"无需迁移"：现有 Reward Vault 与协议**不需要重新部署**，残余 BGT 授权在 claim 时自动按当前汇率结算为 WBERA；只是前端"claim UI/reward-token 标签/索引器"需要把 BGT 改标为 WBERA。

**机制演化对 issuance-native 设计的启示**：Berachain 用一年多时间验证了"把流动性激励做进共识出块奖励"这件事本身是可行的（PoL 机制持续运行至今），但**双代币模型（BGT 作为独立的、可交易的治理/路由代币）被证明过于复杂**——官方原文将新模型的目标表述为"Simplified Yield Path"、"single, predictable rail"。这是继 2.3 节 ixo x/bonds"有先例但未规模化使用"之后，本报告第二个"设计能跑通，但复杂度本身是失败原因"的模式，值得在第 7 节合并讨论。

### 3.4 链级流动性挖矿 / 发行补贴实现方式

把本节及 2.1/2.2 节的发现汇总为"链级流动性激励"的实现光谱：

| 实现方式 | 代表案例 | 落地形态 | 对发行方的意义 |
|---|---|---|---|
| 协议排放路由给验证人分配 | Berachain PoL / PoL Next（3.3 节） | 改共识层（出块奖励逻辑本身） | 新代币的 LP 池可以"竞价"获得协议原生排放，无需协议方自己发流动性挖矿代币 |
| 协议内置自动做市 | Hyperliquid HIP-2（2.1.2 节） | 改执行层/共识层状态转换逻辑 | 部署者一次性注资即可获得持续挂单，无需持续维护做市机器人 |
| 链级 AMM/CL 池 + 池创建费防垃圾 | Osmosis GAMM/CL（3.1 节，100 OSMO 建池费） | 系统合约 + precompile 级（Cosmos module） | 池子本身是协议原语，发行方不需要自己审计/部署 AMM 合约 |
| 模块级转账钩子 + 合约策略 | Osmosis before-send hook + `cw-tokenfactory-issuer`（1.4/2.4 节） | 系统合约 + precompile 级 | 手续费/税费可以在协议钩子上用合约实现，不需要自己重写整个代币合约 |

**结论**：链级流动性激励目前最有力的一手先例是 Berachain PoL——但其"从复杂到简化"的演化路径提示：**协议层激励机制的复杂度需要克制**，越复杂的多代币路由模型，长期维护与用户理解成本越高，最终可能被自我简化。

### 3.2 负面案例：Sei DEX module 弃用

<!-- TODO -->

### 3.3 Berachain PoL（Proof of Liquidity）

<!-- TODO -->

### 3.4 链级流动性挖矿 / 发行补贴实现方式

<!-- TODO -->

## 4. 发行类应用最痛的链级缺口

<!-- TODO -->

### 4.1 抗狙击的链级手段

<!-- TODO -->

### 4.2 原子发行（部署+建池+注池+锁仓）

<!-- TODO -->

### 4.3 费用路由原生化

<!-- TODO -->

### 4.4 元数据与索引原生化

<!-- TODO -->

### 4.5 反 rug 的链级原语

<!-- TODO -->

## 5. 发行侧的链级合规 / 许可闸门

<!-- TODO -->

### 5.1 Arbitrum ArbOS Elara

<!-- TODO -->

### 5.2 Injective permissions module（合规视角）

<!-- TODO -->

### 5.3 Canton / Provenance 类许可链

<!-- TODO -->

### 5.4 「一级 KYB 闸门 + 二级无许可」能否链级表达

<!-- TODO -->

## 6. issuance-native 原语总表

<!-- TODO -->

## 7. 显式回答：EVM 相对 Solana 缺失的资产发行能力清单

<!-- TODO -->

## 存疑清单

<!-- TODO -->

## 关键来源清单

<!-- TODO -->

## 可复现方法附录

<!-- TODO -->
