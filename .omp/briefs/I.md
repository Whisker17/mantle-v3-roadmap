# 轨道 I：RISEx 应用层机制 + 链-app 耦合点取证

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/I-risex-app-coupling.md`

主题：**RISEx 应用层机制，以及它与 RISE 链的耦合点** —— 即「链为这个 app 具体做了什么、app 又反过来吃到了什么链级红利」。

Shreds 的**底层实现**交给 H 轨道，你不要重复拆；你要拆的是 **Shreds 如何被用在下单 / 撮合 / 撤单路径上**。

## Change

### 1. RISEx 产品拆解
- 全链上 CLOB 的实现形态：订单簿数据结构是否在**合约存储**里？撮合是谁触发的？taker 主动撮合还是 keeper？
- **MarketCore 是什么**：是否链上共享撮合基础设施、是否 permissionless 部署市场、spot + perp 是否共用
- 保证金模型（逐仓 / 全仓 / 跨市场统一保证金）
- 清算机制（keeper 竞争？拍卖？ADL？保险基金？）
- funding rate 计算与结算频率
- 费率表（maker / taker / 清算 / 提现）
- 上线时间线、是否有代币与积分（Ignite points）
- **尽量读合约地址与源码 / ABI**（区块浏览器 verified source、官方 deployments 文档、GitHub），读到就标【实测】

### 2. oracle 路径
- Stork 的接入方式：push 还是 pull？谁付 gas？更新频率？是否走特权 / 系统交易？是否有链级 oracle 白名单或预留区块空间？
- 与 Hyperliquid 的 validator-oracle、dYdX 的 oracle module **各一句话**对照（深入对照交给 J 轨道）

### 3. 「链为 app 做的适配」逐项取证（本轨道最核心）
列成表格，逐条给出 **「有一手证据 / 有反证 / 无证据」三态判定**：

1. 下单 / 撮合是否吃到 Shred 级 pre-confirmation（官方给的确认延迟数字）
2. gas 赞助 / gasless 下单撤单 / session key / 一次签名多次下单
3. sequencer 级排序特权（RISEx 交易优先、oracle 先于撮合、cancel 优先于 fill）
4. 为撮合定制的 precompile（排序、定点数学、批量撮合）
5. 区块空间预留 / 专用 lane
6. 原生 sub-account / 权限委托（API-key 式交易权限）
7. revert protection / 失败订单不收费
8. 链级风控（pre-trade risk check 进入状态转换）
9. native USD / 统一保证金资产（是否 ETH 付 gas + USD 计价的双资产设计）
10. 提现 / 跨链的快速通道（fast withdrawal 还是依赖第三方桥）

### 4. 官方叙事取证
- RISE 官方是否**明确自我定位为 app-specific / 为 RISEx 服务的链**？找官方 blog、docs、创始人访谈 / 播客 / 推文的**英文原话引用**
- 考证 `docs.risechain.com` 站点标题变成「RISEx — The unified exchange」、首页大标题变成 "The trading chain" 的**时间点**（用 `web.archive.org` 对比历史快照，如 `http://archive.org/wayback/available?url=risechain.com&timestamp=2025`）与含义
- 是否存在「**从通用高性能 L2 转向交易专用链**」的战略转向？给证据（早期定位 vs 现在定位的原话对比最有力）

### 5. 数据表现
- RISEx 的成交量 / 未平仓 / TVL / 手续费 / 用户数（DefiLlama derivatives 端点、官方 API、Dune 若可读）
- 与 Hyperliquid / Lighter / dYdX / Aster / edgeX 对比排名（标日期与来源）
- 是否有激励驱动刷量迹象（积分活动、做市激励、交易大赛）—— 客观标注，不做道德判断

### 6. 可迁移性判断
把 RISEx 的每一项优势归因到三类：
- **(a) 只能靠链级改造实现**
- **(b) 应用层就能实现，与链无关**
- **(c) 靠中心化 sequencer 的运营特权实现**（换成去中心化就没了）

这个三分类是给委托方的**核心判断依据**。

## Acceptance

- 文件已创建，含上述 6 节
- 有一张 **「候选链级适配 × 证据三态（有一手证据 / 有反证 / 无证据）」** 表，覆盖第 3 节全部 10 个条目
- 有一张 **「优势归因三分类（链级改造 / 应用层可做 / 运营特权）」** 表
- 至少 **3 段** RISE / RISEx 官方英文原话引用，附 URL
- 「存疑清单」不少于 **8** 条
- **显式回答**：「整条链为 RISEx 服务」这个说法，**有多少是工程事实、多少是营销叙事**？给出有依据的判断
