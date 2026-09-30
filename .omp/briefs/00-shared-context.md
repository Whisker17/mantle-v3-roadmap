# 共享上下文（第二阶段：app-specific chain 研究）

项目根目录：`/Users/whisker/Work/research/work/launchpad-on-mantle`
已有第一阶段完整研究（`report/01-06` + `research/A-G`，中文，主题为 meme Launchpad 与 Mantle）。

## Goal

第二阶段主题：**「app-specific chain」模式 —— 整条链为一个旗舰应用做原生适配**。

以 **RISE Chain × RISEx（全链上 perps DEX）** 为主案例，对照 **OP Stack / Mantle 这类通用链模式**，最终推导出一种 **「资产发行原生（issuance-native）」的链 infra 模式**（第一落点是 meme / token Launchpad 的链上 UX，叙事上抬升为"资产发行"），并附带**为 perps 新增链原生支持**的设计空间。

委托方是 Mantle 生态方向研究者，核心疑问：**"与其去外部找项目方，不如把整条链做成 app-specific 的形式"** —— 需要证据支持或反驳，以及可执行的 infra 清单。

## Constraints（强制）

1. **输出语言：简体中文**；技术术语保留英文（Shreds、CLOB、precompile、pre-confirmation 等）。
2. **可信度标记制度**：
   - `[一手]` = 官方文档 / 官方 blog / 源码 / 白皮书 / L2Beat / 官方 API 原文
   - `[二手]` = 媒体 / 聚合器 / 研究机构 / KOL 转述，未经一手源交叉验证
   - `【实测】` = 你自己直接调 RPC / 合约 / 官方 API / 读源码所得，**必须附可复现方法**（endpoint + 请求体 + 返回摘要）
   - `⚠️存疑` = 源冲突或无法证实
3. **宁可标注"不确定"，也不编造收敛。** 文末必须有「存疑清单」，逐条列出无法证实项、冲突的源、以及"检索无果因此按不存在处理"的判断。**禁止**用"据了解 / 业界普遍认为"掩盖缺证。
4. 每个关键论断都要**内联 URL**。引用官方原话要给**英文原文**（可附中文翻译）。
5. **时间基准 = 2026-09-07**，数据写明取数日期。严格区分「已上线主网 / 测试网 / 仅路线图 / 仅提案」—— 这个区分是本研究的生命线，**禁止把 roadmap 写成既成事实**。
6. **优先一手源**：官方 docs 站点、GitHub 源码（读**具体文件**，可用 `raw.githubusercontent.com`，不要只读 README）、官方 blog、L2Beat、DefiLlama API。用 `read` 直接抓 URL，`web_search` 只用于发现源。
7. **不要跑任何 formatter / linter / 测试 / 项目级命令**（纯研究仓库）。
8. **只写你自己那一个文件**。不要改 README、不要碰 `report/` 目录、不要写其他 agent 的文件（并发写会导致工作丢失）。
9. 篇幅：**深度优先、禁止注水**。目标 500–1200 行，**表格优于散文**，不要用套话小结堆行数。
10. 若发现某个"预期存在的特性"其实**不存在**，这本身是**极高价值发现**，要显式写出来（第一阶段最有价值结论之一即"Mantle 完全没有 preconfirmation"）。

## Contract

- 必须用 `write` 工具**实际创建文件**，不要只在回答里贴内容。
- 文件头统一格式：

```
# <标题>
> 研究轨道：<字母> ｜ 取数日期：2026-09-07 ｜ 归属：app-specific chain 研究（第二阶段）
> 可信度标记：[一手] / [二手] / 【实测】/ ⚠️存疑
```

- 文末统一三段：`## 存疑清单`、`## 关键来源清单`（分类列 URL）、`## 可复现方法附录`（若有实测）。
- 跨轨道发现写成 `→ 交给 <轨道字母>` 标注，**不越界**研究别人的轨道。
- 术语统一：`RISE Chain`（链）/ `RISEx`（perps 应用，官方拼写全大写 X）/ `Shreds` / `app-specific chain` / `issuance-native`。
- 结论要**编号**、数据带**单位与日期**，便于主 agent 汇总进 `report/07` 与 `report/09`。

## 已知起点事实（供你校验，勿直接复述为结论）

- RISE 是 Ethereum L2，宣称 **1ms latency、50,000+ TPS**，核心机制叫 **Shreds**（无 state root merkleization 的 mini-block），并行 EVM（pevm）。
- **RISEx** 是其旗舰全链上 CLOB perps，底层共享撮合基础设施叫 **MarketCore**，oracle 用 **Stork**。
- `docs.risechain.com` 的站点标题（2026-09-07 实测）已经是 **「RISEx — The unified exchange」**，首页大标题是 **"The trading chain"** —— 链文档被应用文档收编，这个信号本身值得考证。

## 研究轨道分工（避免重复劳动）

| 轨道 | 文件 | 主题 |
|---|---|---|
| H | `research/H-rise-chain-infra.md` | RISE **链级** infra 全面拆解 |
| I | `research/I-risex-app-coupling.md` | RISEx **应用层**机制 + 链-app 耦合点取证 |
| J | `research/J-appchain-comparables.md` | 交易类 app-specific chain 横向对标（Hyperliquid / dYdX v4 / Injective / Aevo…） |
| K | `research/K-sequencing-latency-infra.md` | 排序 / 延迟 / 区块空间 / MEV 的链级零件目录 |
| L | `research/L-issuance-native-primitives.md` | 「资产发行原生」链级原语盘点 |
| M | `research/M-perps-native-chain-support.md` | 为 perps 新增链原生支持的设计空间 |
| N | `research/N-opstack-mantle-surface.md` | OP Stack / Mantle 可改造面与硬约束 |
