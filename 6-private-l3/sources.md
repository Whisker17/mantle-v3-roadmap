# Sources — Private L3 vs EVM account-model privacy

Collected 2026-09-17. Primary only. News recap / Gate Learn / CoinDesk 转述不作为事实。

本地 PDF 缓存（不入库）：`/tmp/private-l3-papers/{zerocash,privacypools,bonsai}.pdf`。Tornado 白皮书 PDF 因 `tornado.cash` TLS 失败未取到，改用 tornado-core README 与其白皮书路径声明。

## Lineage: Zcash / Tornado

| ID | Source | Why |
|---|---|---|
| Z1 | https://eprint.iacr.org/2014/349.pdf | Zerocash：DAP、隐藏 origin/destination/amount、note/nullifier 鼻祖 |
| Z2 | https://zips.z.cash/protocol/protocol.pdf | Zcash Protocol Specification 现行规范 |
| Z3 | https://zips.z.cash/zip-0224 | ZIP 224 Orchard shielded protocol：独立 shielded pool |
| T1 | https://github.com/tornadocash/tornado-core | Classic mixer 合约/电路；deposit/withdraw gas；白皮书路径 |
| T2 | https://github.com/tornadocash/docs | 官方 docs 源（Classic / Nova） |
| T3 | https://tornado.cash/audits/TornadoCash_whitepaper_v1.4.pdf | README 声明的白皮书（本次 TLS 失败，未核原文） |

## EVM 合约隐私：Railgun / Privacy Pools

| ID | Source | Why |
|---|---|---|
| R0 | https://docs.railgun.org/developer-guide/wallet/private-balances | 私有余额 = 加密 UTXO Merkle；「syncing process can take a few minutes」 |
| R1 | https://docs.railgun.org/wiki/learn/privacy-system | 官方：隐藏 sender/recipient/token/amount；UTXO Merkle 嵌在合约里 |
| R2 | https://docs.railgun.org/developer-guide/engine-1/state-structure | 链上状态 = Merkle accumulator + nullifier mapping |
| R3 | https://docs.railgun.org/wiki/assurance/private-proofs-of-innocence | Private POI：list provider、1h unshield-only standby |
| R4 | https://docs.railgun.org/developer-guide/wallet/transactions/cross-contract-calls | Relay Adapt：unshield → 公开 multicall → reshield，单笔原子 |
| R5 | https://docs.railgun.org/wiki/learn/integrating-railgun/adapt-modules | Adapt Module 把参数绑进 SNARK，不改核心合约 |
| R6 | https://github.com/Railgun-Community/private-proof-of-innocence | POI 节点实现 |
| P1 | https://privacypools.com/whitepaper.pdf | Buterin et al. 2023 论文 PDF（与 SSRN 4563364 同文） |
| P2 | https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4563364 | 同上 SSRN 记录 |
| P3 | https://docs.privacypools.com/ | 0xbow 官方：ASP / 部分提取 / ragequit |
| P4 | https://docs.privacypools.com/layers/contracts/privacy-pools | 合约：commitment/nullifier、ASP root、ragequit |
| P5 | https://github.com/0xbow-io/privacy-pools-core | 合约/电路/SDK 实现 |

## 可编程隐私对照（只为改变判断）

| ID | Source | Why |
|---|---|---|
| A1 | https://docs.aztec.network/developers/overview | Aztec L2：private/public 双执行、PXE；public 不能调 private |
| A2 | https://docs.aztec.network/developers/docs/foundational-topics/call_types | 私有可组合 vs enqueue 公开调用（无返回值） |

## Commonware private payments

| ID | Source | Why |
|---|---|---|
| C1 | https://commonware.xyz/blogs/private-payments | 指定博文：Out of Sight, Out of State；Bonsai 设计与吞吐数字 |
| C2 | https://eprint.iacr.org/2026/1987.pdf | Bonsai 论文：account commitment、无全局 nullifier、send/receive 分步 |

## 交易前隐私（只为划界，不展开）

| ID | Source | Why |
|---|---|---|
| M1 | `1-rfs-propamm/chain-infra/sequencing-and-cancel-priority.md` | 仓库已有：pre-transaction privacy ≠ 资产隐私 |
