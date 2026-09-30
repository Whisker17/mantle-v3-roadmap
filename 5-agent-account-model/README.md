# Mantle Agent 账户模型研究

本目录围绕 Meme Launchpad 与 Onchain Perps，研究 Agent 的受限授权、并发操作、费用支付和批处理需求。

## 综合研究报告与核心技术深度解析

- **[将 Agents 作为一等公民的 Mantle 账户模型研究报告](./agent-account-model-research-report.md)**：综合决策研报，覆盖产品场景需求对齐、全景 AA 方案横评及 Mantle 特性矩阵。
- **[EIP-8141 与 EIP-8130 底层机制与虚拟机执行内核深度解析](./eip-8141-and-8130-deep-dive.md)**：面向协议层与底层虚拟机的技术内幕指南，逐层拆解两项最新 Native AA 提案的 RLP 载荷结构、专用操作码（`APPROVE 0xaa` / `TXPARAM 0xb0`）、Keystore 32 字节槽位位域、2D Nonce 预编译、环形去重缓冲区及两级 Phase 提交状态机。

## 历史专题笔记

以下四篇保留为前期研究记录。它们包含尚未验证的性能估计、早期规范理解和架构设想；阅读账户机制与方案对比时，以综合报告的来源核查为准。尤其注意：7702 委托持久存在，4337 支持多 key nonce，7560 的非零 nonce 通道还涉及 RIP-7712。

| 文件 | 研究主题 |
|---|---|
| [01 EVM 账户限制](./01-evm-account-limitations-for-agents.md) | 私钥权限、nonce 阻塞、Gas 与其他账户限制 |
| [02 AA 技术路线](./02-aa-and-agent-account-landscape.md) | 早期 AA、Session Key、模块化账户与 TEE 调研 |
| [03 Meme / Perps 需求](./03-meme-and-perps-agent-requirements.md) | Tape 与股票 Perps 场景的账户需求建模 |
| [04 历史架构提案](./04-mantlev3-agent-account-architecture.md) | 前期自建设计设想，不是 Mantle 官方路线图或综合报告的选型结论 |
