# 轨道 H：RISE Chain 链级 infra 全面拆解

**目标文件**：`/Users/whisker/Work/research/work/launchpad-on-mantle/research/H-rise-chain-infra.md`

主题：**RISE Chain 的链级 infra 全面拆解**。不含 RISEx 应用层机制 —— 那是 I 轨道的事，**你只负责链**。

## Change

穷尽拆解以下 9 维，每一维都要区分「已上线 / 测试网 / 路线图 / 提案」：

### 1. 血缘与定位
- 是否 OP Stack fork？基于 reth 还是 op-geth？与 based rollup / 传统 rollup 的关系
- 团队（RISE Labs / RISE Chain）、融资（轮次 / 金额 / 领投 / 日期）
- 主网上线时间线、chainId、mainnet + testnet endpoint
- 查其 GitHub org（如 `risechain` / `riselabs-xyz` / `rise-labs`）**实际仓库清单**与主力语言

### 2. Shreds 的工程级细节
- 定义、产出频率、与 L2 block 的关系、是否含 state root、reorg 语义、如何被验证
- **给用户的确认语义是什么**（pre-confirmation 的信任假设是谁的签名？sequencer 单签？有无经济担保？）
- 找官方规范文档或**源码实现**
- 与 **Base Flashblocks / Unichain rollup-boost / Solana shreds** 逐项辨析（命名相似但机制可能完全不同，务必辨析清楚）

### 3. 执行层
- 并行 EVM（pevm / Block-STM 血统？）、状态存储引擎（自研 DB？MDBX？state commitment 优化？）
- **是否有非标准 opcode / 自定义 precompile / EVM 语义偏离** —— 极重要，列出 RISE 有哪些**标准 EVM 之外的能力**
- 区块 gas 上限、单笔 gas 上限

### 4. 交易接口层（对交易类应用最关键）
- 是否有 `rise_sendRawTransactionSync` 之类的**同步返回 RPC**？
- WebSocket 订阅 shred 的方法名与 payload 结构？
- revert protection？preconfirmed receipt？state override 模拟？
- 把**完整的非标准 JSON-RPC 方法表**列出（方法名 + 参数 + 返回 + 官方文档链接）

### 5. 费用市场
- base fee 机制（EIP-1559 与否）、gas token（ETH 还是自有币）
- 是否有 local fee market / per-account 或 per-contract 隔离
- priority fee 语义、排序规则（FCFS？PGA？是否有公共 mempool？）
- **是否为 RISEx 做了任何排序 / 费用特权**（系统交易、oracle 交易免费、cancel 免 gas）—— 本研究核心问题之一

### 6. DA / 结算 / 安全
- DA 层（Ethereum blob / 外部 DA？）、proof 系统（optimistic / zk / 无？）、fault proof 状态
- L2Beat 的 stage 评级与风险项、force inclusion 窗口
- sequencer 中心化程度、升级钥匙（多签构成、timelock）

### 7. 账户抽象与钱包层
- 原生 AA / session key / 协议级 paymaster / gas sponsorship
- 原生 embedded wallet 方案、EIP-7702 支持

### 8. 实测（必做）
直连 RISE 公共 RPC 做测量并标【实测】：
- `eth_chainId`、`eth_blockNumber`
- 连续采样出块间隔（≥30 个区块，给均值 / 中位数）
- `eth_getBlockByNumber` 的 gasLimit / gasUsed / **填充率**
- `eth_gasPrice` / `baseFeePerGas`、`eth_feeHistory`
- `web3_clientVersion`（暴露客户端血缘 —— 这一条极有价值）
- 尝试调用非标准方法（如 `rise_*`）看是否存在
- 若有 shred WebSocket，尝试订阅并记录一段时间内 shred 到达间隔

所有请求体与返回摘要写进「可复现方法附录」。

### 9. 生态构成实测
- RISE 上除 RISEx 之外还有多少真实应用？
- 用 DefiLlama（`https://api.llama.fi/v2/chains`、chain TVL、fees / dexs 端点）与区块浏览器取 TVL / DEX 量 / 活跃地址 / 日交易数（标日期）
- **量化 RISEx 占整条链活动的比例** —— 这是"整条链在为一个 app 服务"这一命题的**关键证据**

## Acceptance

- 文件已创建，含上述 9 节，每节都有一手 URL
- 有一张 **「RISE 的非标准 EVM / RPC 能力清单」** 表：能力 / 是否上线 / 一手来源 / 通用 OP Stack 是否具备
- 有一张 **Shreds vs Flashblocks vs rollup-boost vs Solana shreds** 机制对照表
- 实测数据至少覆盖：出块间隔、gasLimit、填充率、baseFee、clientVersion、chainId，且附可复现请求体
- 「存疑清单」不少于 **10** 条
- **显式回答**：RISE 有哪些能力是通用 OP Stack 链（含 Mantle）**今天做不到或没做的**？逐条列表，每条标一手依据
