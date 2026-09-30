# Mantle v3 分享内容稿

这套文档说明每张幻灯片要讲什么、需要哪些图文，以及可以直接使用的上屏文案。全稿统一使用 [Mantle 深色制作规范](prompts/Mantle-dark-capital-markets.md)，接近官网的深绿近黑、白字与亮绿强调方向；原黄粉手绘提示词仅作历史参考。[核心叙事与产品评审](narrative-product-fit.md) 中保留 ASCII 生态总览。

## 阅读顺序

先看 [章节与页面规划](chain-infra-slide-design.md) 与 [深色制作规范](prompts/Mantle-dark-capital-markets.md)，再按下面的顺序读取内容稿。主题页 P00 是整场第 0 页，完整主题为“从资产供给走向可持续交易，以及产品驱动的 Chain Infra 演进”。

| 章节 | 页码 | 页数 | 结论重点 |
| :-- | :-- | :-- | :-- |
| [回顾与叙事](sections/01-section-0-review-and-narrative.md) | P00、S0、原有正文、C0 | 14 | 资产上链后，还需要好价格与使用场景 |
| [产品拆解](sections/02-section-1-product-breakdown.md) | S1、P08–P20 及后缀页、C1 | 27 | 用 Agent Launchpad 与小盘股 Perps 验证 mStocks 的用途 |
| [能力差距与改链边界](sections/03-section-2-capability-gap.md) | S2、P21–P27、C2 | 9 | 共建产品能力，再按明确缺口决定改链 |
| [AA 路线](sections/04-section-3-aa-roadmap.md) | S3、P28–P32 及后缀页、C3 | 22 | 近期交付 7702 组件，长期以成本、纳入与兼容性选择原生方案 |
| [Perps 执行路线](sections/05-section-4-fully-onchain-perps.md) | S4、P33–P42 及后缀页、C4 | 26 | 核对 RISE 全链路优化，推荐以 L3 承接长期定制与多业务扩展 |
| [验证与决策](sections/06-section-5-action-plan.md) | S5、P43–P44、C5 | 4 | 下一阶段交付产品流程与验证结果 |
| [问答附录](sections/07-appendix.md) | SA、A01–A03、CA | 5 | 在失败与恢复场景中判断可用性 |

共 **102 页主讲、5 页附录**。主讲包括 P00 总封面、六张章节分隔页、89 张内容页和六张总结页；附录包括 SA 入口、三张内容页和 CA 总结。阅读顺序为 `Sx → 本章正文 → Cx`，P00 仍为整场第 0 页。已有 Pxx/Axx 与后缀 ID 不变；S5/C5 只是收尾标识，不代表原文新增 Section 5。

新增 P00 给出主题页排版。Fluxion 由 P04 真实截图、P04A 的 TVL 局限、P04B 的资产孤岛与多协议用途连续讲述。P07 给生态总览，P07A/P07B 讲产品分工，P43 检查实际衔接；分类修正和 ASCII 图见 [评审稿](narrative-product-fit.md)。

Section 1 当前顺序：P09/P09A 解释 Bonding Curve 与毕业原子性；P10 复盘；P11/P11A 讲 100 USDC 任务的两种视角；P12 pitch 后接 P12A/P12B，解释 v4 的 Singleton、Flash Accounting、Hooks 与同池阶段转换，再由 P13 归纳 Infra。Perps 的 P14B–P14D 明确小市值币股定位。

参考笔记：[曲线与 v4 毕业核对](references/bonding-curve-v4-graduation.md)、[三篇 Perps 文章](references/perps-tokenized-assets-reading.md)、[Agent 实现方案](references/agent-launchpad-implementation-notes.md)。

Section 2 已回到原纲适配：P21 保留 2.1 的五行四列表；P22–P26 对应 2.2.1–2.2.5，每页先写两产品需求，再写对应优化；P27 保留 2.3 的两主线及两支线。图文用于解释原结构，不重组为另一套能力分类。

Section 3 从实例开始：P28 用 SimpleAccount 讲 4337 买入，P29 用 Simple7702Account 讲原 EOA 委托与自调用，P29A 再加入受限 Agent 和代付。P30B/P30C 用费用、Bundler 纳入路径和 EVM 兼容解释原生选型需求；P31 对照 4337 与 7560，P31H 解释 8250 如何扩展 8141。新增 P31J 讨论隐私、CROPS 和 8130 的 L1/L2 profile。P32 列近期 7702 组件功能，P32A 讲长期验收，C3 收束两阶段路线。

Section 4 当前为 26 页：P34 讲 Vault 实现、风险及过渡定位；P36 使用 RISE 截图换算，P36A 展示 CBP、Shreds、RiseDB、EigenDA 与挑战式证明，新增 P36B 解释 Solidity 撮合和 EVM 适配。路线一 P36–P37A，路线二 P38–P39B，路线三 P40–P41D，所有路线页均带完整眉标。P41A–P41D 分别 pitch 长期收益、三类业务扩展、实验与互操作空间、代价和落地条件；其中 P41B 重点讲 Launchpad 过期剪枝与 EIP-7736，以及 Bonsai 隐私支付原型的紧凑状态和百万级批量验证口径。C4 明确推荐 L3 为长期主线。原稿与新证据见 [执行路线核对](references/perps-execution-routes-evidence.md)。

## 每页怎样使用

- **章节分隔页 S0–S5 / SA**：说明本章将讨论什么，给出主旨和阅读路径。
- **章节总结页 C0–C5 / CA**：给出本章得出的判断、对下一步的含义及必要条件；制作说明中标明正文依据，不重复目录或新增结论。
- **本页要讲什么**：帮助讲述者和制作 agent 理解本页的重点。
- **对应原文**：给出具体小节，便于核对原意。
- **需要的图文及原因**：说明要展示的对象、关系、证据，以及它为什么有用。
- **上屏文案**：最终标题、正文、图中短标签和必要图注。制作时直接使用，不把前面的说明一并搬上屏。
- **讲述补充（不上屏）**：只在需要解释技术条件或事实口径时保留。

文案以清楚的短句为主。一般页尽量控制在约 120–180 个字符，流程或对照页可适当放宽；这只是密度提示，不能为了凑字数删掉主语、条件或必要解释。

## 依据与素材

- 主依据：[slides/references/chain-infra.mdx](references/chain-infra.mdx)。各页引用该文件的小节编号和标题；原文两个 `0.4` 通过标题区分。
- 本轮补充：[用户叙事导图](references/mantle-v3-narrative-mindmap.png)、[产品关系评审与资料取舍](narrative-product-fit.md)。扩展产品只补定位，不自动成为已承诺的开发任务。
- P06 必用图：[MegaETH TVL / Chain Fees](pics/megaeth_2026-09-26.png)、[RISE TVL / Chain Fees](pics/rise_2026-09-26.png)。两图用于说明产品驱动的 Infra 迭代；保留坐标与图例，指标使用“链费用”，不改为链净收入。
- Fluxion 案例：[本仓库必用截图](pics/fluxion-lp.png)、[本地 mStocks 材料](/Users/whisker/Work/research/work/mantle-research-base/knowledge-base/products/m-stocks.mdx)。P04 必须实际嵌入截图；P04A 取 META 数据说明 TVL 的局限；P04B 引用此前调研的单一场所与 Bybit 套利路径。截图时点未明确，不作为当前报价。
- P14A 必用图：[币股成交时段占比](pics/perps-after.png)。图中来源为 Blockworks，展示四条链各自的非 RTH / RTH 成交量占比；保留原图定义与白底配色，不作为 Perps 成交量或每小时交易强度的证据。
- P36 必用图：[RISE Gas / 挂撤单截图](pics/rise-gas-meter.png)。截图 1 GGas/s 不是单块 Gas Limit，挂单与撤单吞吐不可相加；按 [2026-09-27 Mantle RPC 样本](references/mantle-execution-rpc-2026-09-27.json) 推算约 9 次挂单/秒或 27 次撤单/秒，标明同开销、独占全链预算及非移植实测。
- Section 4 设计依据：[用户指定网页的本地源稿快照](references/execution-three-paths-source.mdx)、[容量、风险与证明核对](references/perps-execution-routes-evidence.md)。网页本次未取得正文，未假定本地版本与线上一致；采用详细设计及校正后的事实边界。
- Perps 立项论据：[三篇指定文章的阅读笔记](references/perps-tokenized-assets-reading.md)。已取得正文并归纳机制；文章数据不直接当成 Mantle 的增长或收益预测。
- Uniswap v4：[官方机制与固定源码核对](references/bonding-curve-v4-graduation.md)。包含曲线例子、非原子迁移、真实储备、Hook 返回权限、NoOp 效果与解锁结算边界。
- Section 3 主资料：[8130 Deep Dive](references/aa/01-eip-8130-deep-dive.md)、[8141 Deep Dive](references/aa/02-eip-8141-deep-dive.md)、[Mantle 能力覆盖评估](references/aa/03-mantle-aa-coverage-and-fit.md)。补充核对：[传统 AA、成本与纳入、EVM、隐私和近期组件](references/aa-comparison-and-evm-compatibility.md)。
- AA 机制核对：[ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)、[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)、[RIP-7560](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7560.md)、[EIP-8130](https://eips.ethereum.org/EIPS/eip-8130)、[EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)、[EIP-8250](https://eips.ethereum.org/EIPS/eip-8250)。8130 与 8141+8250 的多通道 nonce、Gas Sponsor、热点配额/LFM 和可靠重试的区别见 [实现核对笔记](references/agent-launchpad-implementation-notes.md)。提案内容会变化，具体实施需固定版本。

引用的项目用于解释机制或原文的判断，不据此推断 Mantle 已上线相关能力。Agent 获客、社区分发和新增资产用途保留为待验证方向；涉及首批任务、测试方法与分工时标明建议。
